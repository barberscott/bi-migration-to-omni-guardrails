# Automated BI Migration to Omni — Guardrails

**Status:** Draft
**Audience:** Teams building automated migration tooling that targets Omni.
**Scope:** Any engagement where tooling migrates content from another BI tool into Omni. Tool-agnostic and customer-agnostic.

## Purpose

This document lists the conditions a migration must satisfy to be accepted. It states requirements, not implementation approaches.

Where a requirement depends on a specific Omni mechanism, the mechanism is named so the requirement can be tested. Where Omni does not enforce a requirement itself, that is stated; several of the gates below are procedural only.

Two constraints shape most of the rules:

- The migrating party has no way to determine the intent behind existing model objects. Rules that forbid modifying existing objects follow from this.
- Content that renders correctly can still return wrong numbers. Structural rules are necessary but not sufficient, and parity (§10) is a separate gate.

---

## 1. Preconditions

The following are prerequisites owned by the customer's data team. Migration tooling must not perform them.

- **Schema refresh.** Run a schema refresh before the migration branch is opened. Many "not found" validation errors are stale-schema artifacts rather than real breakage, and a baseline taken against a stale schema cannot distinguish the two. Refresh requires Connection Admin. Some connections reject branch-scoped refresh, so a shared refresh is the only option on those connections, which is why this cannot sit inside the migration's branch.
- **Connection schema scope.** If a required database view is not in scope for the connection, the connection must be edited to bring its schema into scope. Widening scope exposes every table in that schema to every modeler on that connection, so this requires data-team approval.
- **AI quarantine configuration.** Configure the model-level `ai_chat_topics` property once, before the migration begins (§8). Migration tooling must not write this property.
- **Content inventory and triage.** A named owner determines what is in scope. Without an explicit in-scope list, dead and duplicated source content is reproduced along with everything else.
- **Target model.** Decide which model receives the migration's output (§4).

---

## 2. Topics

- **Net-new topics must be identifiable as migration output.** This allows migration-driven work to be separated from pre-existing work after the merge. Carry provenance as metadata — a tag, plus topic-level `owners:` naming an accountable person. Do not use `description` or `ai_context` for this; the migrating party cannot characterize a topic's intended use, and an inaccurate characterization is worse than none.
- **Notes on how the migrated content uses a topic belong in the topic as comments.** These record what the migration did, which the migrating party does know.
- **Existing topics must not be modified.**
- **Topics must not duplicate existing topics.** If an existing topic can serve the query, use it.
- **Consolidate net-new topics by base view, not by query.** One topic per base view, covering the union of join paths the migrated queries require. A topic's join graph costs only what a query references, so unused joins in a consolidated topic add no query-time cost.
  - *Exception:* a join-path or grain conflict — two different paths to the same view, or the same view needed at two grains — forces a split.
  - *Accepted cost:* a wider field list and greater fan-out exposure. See §3 and §8.

---

## 3. Views, dimensions, and measures

- **Every view used in a migration-generated topic must have a declared primary key**, whether database view or query view.
- **Database views must not be provisioned in the shared model.** Resolve a missing view by bringing its schema into connection scope (§1).
- **Establish that a view is missing before acting on it.** A model-wide `yaml-get` returns only views from currently-loaded schemas; views in offloaded or inactive schemas are absent from the response while remaining available. Run `omni models get-schemas` and, if the schema is listed, load it with `--includeschemas <SCHEMA>` (one schema per call) before concluding a view does not exist. The remedy for a genuinely missing view is a connection-scope change, so a false negative here widens data access unnecessarily.
- **Query views must either be modeled** (using the `query` parameters) **or reference all database tables, dimensions, and measures as `${database_view__table}` and fields as `${foo_bar__qux.zip}`.** No direct-table or direct-field references. This preserves the path to dbt virtual schemas.
- **No direct column references in migration-authored code.** All column references use `${foo_bar__qux.zip}` notation.
- **No net-new fields on existing views.** No relabeling, no `hidden` changes, no format changes on existing fields. Migration-authored calculations go in a migration-owned extension view or query view.
- **Use a measure `filters:` block rather than `CASE WHEN` for filtered aggregates.** Keep `sql` limited to the value being aggregated.
- **Validate fan-out and symmetric aggregation explicitly.** A declared primary key is necessary but not sufficient. Any measure reachable across a one-to-many join in a consolidated topic requires explicit validation. The consolidation rule in §2 increases exposure to this.
- **Treat names as permanent.** Content breaks on rename. Labels are cosmetic; names are not. Check net-new names for collisions, including collisions that appear only after view-name scoping.

---

## 4. Branch and model state

- **The model must return to the state it was in when the branch was opened.** No net-new errors. §5 defines how this is measured.

- **Every file the migration writes must be a file the migration owns.** The pull request reports the files it touched, and none may be a pre-existing model file. Global model settings — `alwaysScopeViewNames`, model-level defaults, connection bindings, `ai_chat_topics` — are out of bounds.

  This is stated as an invariant rather than a pre-declared allowlist because the file set is discovered during the work. It restates the prohibitions in §2 and §3 as a single checkable assertion, and it supplies the definition of "my files" that the `yaml_path` gate in §5.2 depends on.

- **Model YAML must not be hand-committed to the git repository.** All writes go through Omni so that normalization happens on write. Hand-edited YAML produces corrective commits and unreviewable diffs.

- **Branch currency behaves differently on each ship path.**
  - *Git-connected:* `omni models commit` merges the base into the git feature branch before creating or updating the pull request. The git artifact is current with main automatically. The Omni branch model that was validated is not — it remains forked from the point the branch was created. Validation ran against the pre-merge state; the merged state is what ships. Re-validate after any commit that pulled main in.
  - *Not git-connected:* there is no rebase or pull operation for an Omni branch model. `create-branch`, `delete-branch`, and `merge-branch` are the entire surface. A stale branch cannot be made current, and merging promotes its authored files over anything the shared model has since acquired for the same files.
  - *Requirement:* keep migration branches short-lived and detect drift by comparing against main. Branch age is a risk measure.

- **Choose the target model up front.** If a hub-and-spoke arrangement exists, choose the spoke. A dedicated migration model extending the shared model is a third option; see the appendix. It has an unresolved exit path and should be evaluated separately rather than adopted by default.

---

## 5. Validation gates

### 5.1 Severity

Model validation returns a JSON array of issue objects carrying `is_warning`, `message`, and `yaml_path`. Only `is_warning: false` is an error.

- Errors are blocking.
- Warnings are a reported delta requiring a written waiver. They are not automatically blocking. Warnings such as `No join path from …` are common on production models, so a rule of "no net-new errors or warnings" is stricter than Omni's own bar and will either block every pull request or be ignored.

### 5.2 Gate on `yaml_path`

No pre-branch snapshot is needed. Both validators accept an optional branch, and main can be queried at any time:

```
omni models validate <modelId>                                       # baseline
omni models validate <modelId> --branchid <branchId>                 # branch
omni models content-validator-get <modelId>                          # baseline
omni models content-validator-get <modelId> --branch-id <branchId>   # branch
```

The two commands use different flag spellings (`--branchid` and `--branch-id`). Pin both in the tooling spec.

Filter issues by `yaml_path` against the files the migration owns (§4) rather than diffing branch against main. This is unaffected by main moving under a long-running branch, and it prevents new breakage being attributed to pre-existing noise.

### 5.3 Topic exposure gate

A topic-scoped `relationships` entry written without the corresponding view `joins` entry validates clean, but the joined view's fields are not exposed and queries against them return empty. Model validation does not detect this.

For every net-new topic, confirm the resolved field set:

```
omni models get-topic <modelId> <topicName> --branch-id <branchId>
```

The response must contain every field the migrated content references.

Ordering constraint: a topic marked `hidden: true` returns 404 from this endpoint while still validating clean. Run this gate before applying the quarantine flag in §8.

### 5.4 Content validator behavior

The content validator reports broken references in saved content. It does not evaluate whether results are correct.

- Requiring "no content-validator errors in newly generated content" achieves little on its own, since content generated from the branch's own topics rarely carries broken references. The load-bearing half of the rule is no net-new breakage in pre-existing content, measured as the branch-versus-main issue delta.
- Default `--content-filter-mode` is `ALL`, which returns every document with at least one query rather than only documents with breakage. Tooling that treats returned rows as errors will flag the entire instance. Use `WITH_ISSUES` or parse the issue arrays.
- Issues appear in two arrays: `queries_and_issues[]` and `dashboard_filter_issues[]`. Reading only the first misses broken dashboard filters, which bears on the filter-parity requirement in §10.
- `--branch-id` does not filter the document set. It sets the validation context; the same documents are returned either way.
- Drafts are returned alongside published documents with `type: "draft"`. Migration output held in drafts is therefore in scope. Only documents cleared to zero query presentations are omitted.
- Consume the API response rather than the user interface. The validator's search filters on field, view, and topic references parsed from query definitions and does not read issue text. Breakage originating in the model, such as a broken `always_where` on a topic, attaches an error to every tile while the document contains no matching reference, so those rows cannot be reached through the interface filter. The errors are present in the API response.

### 5.5 Sequencing

Verify written YAML with a read-back before interpreting any validator output. A successful write response means the write was accepted, not that it landed in the intended file. Validator results against a branch whose contents differ from expectation are not meaningful.

### 5.6 No enforcement at merge

Omni provides no dry-run and no server-side gate. A branch carrying blocking validation errors can be merged. Every gate in this section is procedural and depends on the review process.

---

## 6. Content

- **No content-validator errors in newly generated content, and no net-new content-validator errors in pre-existing content.** Measured per §5.4.
- **All queries must be based on topics**, including single-view topics.
- **Dashboard filters must bind to modeled fields**, with default values and control types matching the source.
- **Migrated content must be placed deliberately.** Folder, owner, and permissions are assigned as part of the migration (§9).

---

## 7. Workbook model and ad hoc content

Anything placed in the workbook model must be negotiated by exception. Nothing goes there by default.

Forbidden absent an approved exception:

- Workbook-level ad hoc dimensions and measures
- Workbook-level joins and relationships
- Raw-SQL tiles
- Any SQL-authored field at the workbook layer

**Table calculations are permitted** and are outside this rule. They live in the query rather than the workbook model, and they are the correct mechanism for presentation-layer computation such as running totals, percent-of-total, and period-over-period. Using a table calculation to reimplement a metric that belongs in the model is not permitted.

**Enforce this with a check, not a policy statement.** Migration tooling tends to drift into the workbook model, because content that does not fit the shared model still renders from there. A document's identifier is also its workbook model identifier, so each migrated document's workbook model can be read and asserted empty as a pipeline step.

**Exceptions require a named approver**, the reason recorded in the migration manifest (§12), and a tracked follow-up to model it properly.

---

## 8. AI surface and discoverability

Net-new topics are quarantined from AI and from the topic picker until reviewed and approved.

**Mechanism**

1. Every migration-created topic carries a tag, for example `migration_pending_review`.
2. Model-level `ai_chat_topics: [all_topics, -tag:migration_pending_review]`.
3. Topic-level `hidden: true` on each net-new topic keeps it out of the workbook picker.
4. Approval consists of removing the tag and the `hidden` flag from that topic's file.

**Rationale**

- With no `ai_chat_topics` property present, all topics are available to AI chat. Taking no action means the migration's topics are live in AI on merge day, so the quarantine must be an affirmative step.
- Negation by tag keeps the approval action inside the migration-owned topic file. An explicit allow-list would require editing the model file on every approval, which adds churn and creates opportunities to break AI for pre-existing topics.
- The tag also serves as the provenance marker required by §2: greppable, machine-readable, and carrying no semantic claim about the topic.

**Constraints**

- **Tooling must not write `ai_chat_topics`.** An empty list disables AI chat for the whole model. Configuration is a one-time human prerequisite (§1).
- If the model already carries an explicit `ai_chat_topics` list, append the negation rather than replacing the list.
- **Hiding a topic does not break content built on it.** `hidden: true` removes the topic from the workbook picker; saved content continues to query it.
- **Apply `hidden` last.** A topic carrying `hidden: true` returns 404 from `get-topic` while still validating clean, so the §5.3 exposure gate cannot inspect it. Run the exposure check first.
- **This is not a security control.** It governs AI chat and discoverability only. Access control is §9.
- **Consolidated topics are wide.** AI field curation becomes necessary as a topic approaches roughly 550 fields, so `ai_fields` curation belongs in the approval review.

---

## 9. Access and row-level controls

Row-level security in Omni is topic-only. `access_filters`, `always_where_sql`, and `always_where_filters` are topic parameters; views do not carry them. A net-new topic therefore has nothing to inherit — it starts as an unfiltered surface, and the migration must reconstruct the gating.

Object-level gating differs. `required_access_grants` exists at view and field level as well as topic level, so grants on an existing view carry into any topic using it. Grants control whether a user sees an object; they do not filter rows.

### 9.1 Discovery

Identifying which source content is gated or pre-filtered by user attributes in the incumbent system is a discovery deliverable produced before any topic is authored.

For each in-scope source object, discovery must produce: whether it is user-attribute gated, the gating expression, and the population it applies to.

This work belongs to the migrating party and to the export side of their tooling, since it depends on source-system expertise. It is called out here because there is no established practice for it among migration vendors today and it should be scoped explicitly.

### 9.2 Routing

Each discovered gate routes to one of:

- **`access_filters`** — row-level filtering keyed to a user attribute. Use where the source gate resolves to "this user sees their own rows."
- **`always_where_sql`** or **`always_where_filters`** — a non-removable filter, where the constraint is not attribute-keyed or cannot be expressed as an attribute mapping.

Record the routing decision per source object in the manifest (§12), with the source expression alongside the Omni expression.

### 9.3 Required validations

**1. The attribute is the correct one.** The chosen Omni user attribute must be the semantic counterpart of the source gate. This is an intent question and requires the source content owner to confirm it.

**2. The attribute's `default_value` is safe.** The default, not per-user population, is the exposure vector. Verified on a two-country dataset (USA 123,933 rows / UK 25,907):

| Attribute state for the querying user | Result |
|---|---|
| Value set, matching rows | Filtered correctly (UK user saw 25,907) |
| No value, attribute has no default | Query fails, `error_type: PLAN` |
| No value, attribute has a default | Returns the default's rows (unprovisioned user received 123,933, 83% of the data) |

A null attribute fails closed: the user gets an error rather than data. A defaulted attribute does not — a user who was never provisioned receives whatever the default grants, with no error and nothing in validation output to indicate it.

- Audit `default_value` on every attribute used in an access filter. A null default is the safe configuration. Do not set a default that is a real, data-bearing value.
- Audit `values_for_unfiltered`. It is an explicit bypass list; any value in it that a user can hold is a gap.
- Per-user population is an availability concern rather than a security one, provided the default is null. Unprovisioned users get a failing dashboard.
- `omni user-attributes list` confirms that the definition exists and reports `default_value`. Per-user values require a SCIM read-back.

**3. The filter restricts results.** Run the query as the target user:

```
omni query run --body '{ "query": { ... }, "userId": "<target-user-uuid>" }'
```

Cover at least one user per distinct attribute value plus one user with no value set. All three validations are acceptance criteria.

Automated checks that omit `userId` do not exercise this. Running under the API key's own identity returned zero rows and a `COMPLETE` status where a real user with the same missing attribute got a hard failure. Access-filter tests must name a real `userId`.

### 9.4 Other access requirements

- **Migrated content must not broaden access.** Where source and target access models do not map cleanly, escalate (§11).
- **Content permissions are assigned, not inherited.** Migration carries no access. Folder placement, ownership, and permits are deliverables.
- **Tooling must not modify connection permissions.** Schema scope changes are a §1 precondition.

---

## 10. Parity

Structural rules do not establish that the numbers are correct. Parity is a separate gate.

- **Numeric parity is an acceptance criterion.** Compare every migrated tile against the source tool's output within a stated tolerance, and commit the comparison artifact with the pull request.
- **The business owner of the source content signs off on parity**, not the migrating party.
- **Verify date and time semantics explicitly:** timezone (connection, user, and source tool), week start day, fiscal calendar, and relative-date handling. Omni's relative-date expressions are calendar-based rather than rolling, so a migrated "last 2 years" filter can shift its boundaries.
- **Verify aggregate semantics explicitly:** count versus count-distinct, averages of averages, weighted metrics, null-versus-zero handling, and sort order.
- **Verify row limits.** Source tools often apply default caps. A migrated tile that inherits a different limit returns a different number.
- **Verify formatting** — rounding, currency, and number format.

---

## 11. Unsupported constructs

Define the behavior for constructs the tooling cannot reproduce faithfully: unsupported visualization types, calculations with no Omni equivalent, source features with no analog, and access models that do not map.

Unsupported constructs must halt and be logged for human escalation. The tooling must not substitute an approximation. A gap is visible in the not-migrated list; an approximation is not.

---

## 12. Manifest, idempotency, and rollback

- **A committed machine-readable manifest** mapping source object to target object, per tile, topic, view, and field.
- **Re-running the migration must not duplicate anything.** Object names must be deterministic functions of the source object rather than generated per run.
- **An explicit not-migrated list with reasons**, covering both what the tooling could not do (§11) and what triage excluded (§1).
- **An assumptions log** recording every judgment call the tooling made.
- **A rollback plan** describing how to un-merge once content depends on the new topics.

---

## 13. dbt

- If dbt is integrated, everything must be built off virtual schemas.

---

## Open questions

- What tolerance is acceptable for numeric parity (§10), and does it vary by metric class?
- Who is the named approver for workbook-model exceptions (§7), and is that one person or one per domain?

---

## Appendix — A dedicated migration model

An alternative to writing migration output into the shared model: give the migration its own model that extends the shared model, and place everything it creates there.

**Advantages**

- The provenance requirement in §2 becomes structural rather than a tagging convention.
- The ownership invariant in §4 holds by construction, since the migration is not writing to the shared model at all.
- The migration is separable and revertible as a unit.
- Content that does not belong in the shared model has an obvious home, which reduces pressure toward the workbook model.

**Costs**

- **Backporting.** Moving a proven migration-created topic into the governed shared model is manual. There is no promotion path between models.
- **Document portability.** Content built against the migration model is bound to it. Moving those documents to the shared model is an export/import exercise with its own failure modes, not a reassignment.
- **Two surfaces to govern.** Access grants, AI configuration, and validation gates exist in both models.

The exit path is the unresolved part. Adopt this only where the migration output is expected to remain a distinct layer, or where the backport cost is accepted at the outset.
