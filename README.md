# Automated BI Migration to Omni — Guardrails

**Status:** Draft
**Audience:** Teams building automated migration tooling that targets Omni.
**Scope:** Any engagement where tooling migrates content from another BI tool into Omni. Tool-agnostic and customer-agnostic.

## Purpose

This document lists the conditions a migration must satisfy to be accepted. It states requirements, not implementation approaches.

Where a requirement depends on a specific Omni mechanism, the mechanism is named so the requirement can be tested. Where Omni does not enforce a requirement itself, that is stated; several of the gates below are procedural only.

Two constraints shape most of the rules:

- The migrating party has no way to determine the intent behind existing model objects. Rules that forbid modifying existing objects follow from this.
- Content that renders correctly can still return wrong numbers. Structural rules are necessary but not sufficient, and parity (§14) is a separate gate.

The document is in four parts: preconditions the customer owns, standards the output must meet, hygiene the development process must follow, and the gates that determine acceptance.

---

# Part I — Alignment and preconditions

## 1. Customer alignment

Agreements to reach before any migration work begins. These are organizational rather than technical, and the gates in Part IV cannot be evaluated without them.

### 1.1 Model freeze

**The customer freezes the target shared model for the duration of the migration.** For the freeze window: no changes to views, topics, relationships, or model-level settings; no schema refreshes; no connection-scope or connection-environment changes.

This is required by Omni's branch semantics, not by convention:

- Files the migration branch has touched shadow the base. A base change to the same file is invisible from the branch and is overwritten when the branch merges — no conflict, no warning, no failure (§9).
- Files the branch has not touched resolve live from the base. A field renamed or removed on the base breaks migrated content mid-flight, and the breakage presents as a migration defect.
- Both validators are baselined against main. If main moves, the branch-versus-main delta changes for reasons unrelated to the migration and the §13 gates stop being meaningful.

**Define the exception path.** Where a change cannot wait, require notification before it lands, a re-baseline of both validators, and re-validation of any migrated content touching the affected files. Record each exception in the manifest (§11).

### 1.2 Source system freeze

The in-scope source content is frozen for the same window. Parity (§14) compares migrated output against the source tool, which requires the source to hold still.

### 1.3 Named owners

Name these roles before work begins:

- **Parity sign-off** per migrated dashboard — the business owner of the source content (§14).
- **Workbook-model exceptions** — a single approver, or one per domain (§6).
- **Connection schema-scope changes** — a data-team approver (§2).
- **Content inventory and triage** — the owner of the in-scope decision (§2).
- **Access gating** — the source content owner who confirms user-attribute mappings (§7.3).

### 1.4 Agreed scope and tolerances

- **The in-scope content list is agreed and recorded** before work begins. Excluded content is recorded with a reason (§11).
- **Numeric parity tolerance is agreed**, including whether it varies by metric class (§14).

---

## 2. Preconditions

The following are prerequisites owned by the customer's data team. Migration tooling must not perform them.

- **Schema refresh.** Run a schema refresh before the migration branch is opened. Many "not found" validation errors are stale-schema artifacts rather than real breakage, and a baseline taken against a stale schema cannot distinguish the two. Refresh requires Connection Admin. Some connections reject branch-scoped refresh, so a shared refresh is the only option on those connections, which is why this cannot sit inside the migration's branch.
- **Connection schema scope.** If a required database view is not in scope for the connection, the connection must be edited to bring its schema into scope. Widening scope exposes every table in that schema to every modeler on that connection, so this requires data-team approval.
- **Model access for the migrating identity.** Branching requires full-model access. Confirm with `omni whoami whoami --modelid <modelId>` before work begins.
- **Content inventory and triage.** A named owner determines what is in scope. Without an explicit in-scope list, dead and duplicated source content is reproduced along with everything else.
- **Target model.** Decide which model receives the migration's output (§9).

---

# Part II — Modeling and content standards

## 3. Topics

- **Net-new topics must be identifiable as migration output.** This allows migration-driven work to be separated from pre-existing work after the merge. Carry provenance as metadata — a tag, plus topic-level `owners:` naming an accountable person. Do not use `description` or `ai_context` for this; the migrating party cannot characterize a topic's intended use, and an inaccurate characterization is worse than none.
- **Notes on how the migrated content uses a topic belong in the topic as comments.** These record what the migration did, which the migrating party does know.
- **Existing topics must not be modified.**
- **Topics must not duplicate existing topics.** If an existing topic can serve the query, use it.
- **Consolidate net-new topics by base view, not by query.** One topic per base view, covering the union of join paths the migrated queries require. A topic's join graph costs only what a query references, so unused joins in a consolidated topic add no query-time cost.
  - *Exception:* a join-path or grain conflict — two different paths to the same view, or the same view needed at two grains — forces a split.
  - *Accepted cost:* a wider field list and greater fan-out exposure. See §4 for fan-out.
- **Curate fields on wide topics.** Consolidation produces broad topics, and AI accuracy degrades as a topic approaches roughly 550 fields. Where a consolidated topic is that wide, curate with `ai_fields`.
- **Cross-view fields belong in the topic's `views:` block, not in a view file.** A dimension or measure whose `sql` references `${other_view.field}` depends on that view being joined, which is not guaranteed in every topic containing the host view. Defining it in the view file produces validator errors in every topic that includes the host view without the referenced view, cascading across topics that are otherwise correct. Define it in the topic, where the join context is explicit. Only put a cross-view field in a view file when the referenced view is joined in every topic that includes the host view.
- **Check the global relationships file before authoring a topic-scoped relationship.** If a join between the same two views already exists in either direction with the same `on_sql`, the topic-scoped copy is redundant — use `joins` alone. If the `on_sql` differs, use the extended-views pattern rather than silently overriding the global join.
- **Join the same view multiple ways with the extended-views pattern**, not `join_to_view_as`. The error `relationship alias duplicates view name` indicates the wrong pattern was used.
- **Before adding a topic-scoped field to an existing view, confirm the field does not already exist.** A field of the same name with different SQL is an override: queries through that topic use the topic-scoped definition while every other topic keeps the shared one. Overrides require explicit approval.

## 4. Views, dimensions, and measures

- **Every view used in a migration-generated topic must have a declared primary key**, whether database view or query view. Omni keeps `sum`, `count`, and `avg` correct under one-to-many joins by deduplicating on the primary key; a view without one cannot be made symmetric, and a row-count measure on it inflates whenever a join duplicates its rows.
  - A single unique column is declared with `primary_key: true` on that dimension.
  - Where no single column is unique, use the built-in compound key: `custom_compound_primary_key_sql: [field_a, field_b]` at the view level, with no `primary_key: true` dimension. Do not hand-build a concatenated key expression. The same parameter accepts a single column where that is more convenient.
  - Where no combination of columns is row-unique, do not declare a key at all. Fix the grain or do not join the view. A declared key that is not actually unique is worse than none.
- **Database views must not be provisioned in the shared model.** Resolve a missing view by bringing its schema into connection scope (§2).
- **Establish that a view is missing before acting on it.** A model-wide `yaml-get` returns only views from currently-loaded schemas; views in offloaded or inactive schemas are absent from the response while remaining available. Run `omni models get-schemas` and, if the schema is listed, load it with `--includeschemas <SCHEMA>` (one schema per call) before concluding a view does not exist. The remedy for a genuinely missing view is a connection-scope change, so a false negative here widens data access unnecessarily.
- **Query views must either be modeled** (using the `query` parameters) **or reference all database tables, dimensions, and measures as `${database_view__table}` and fields as `${foo_bar__qux.zip}`.** No direct-table or direct-field references. In a query view's `sql:` block this means `${view_name}` rather than a hard-coded `CATALOG.SCHEMA.TABLE` path. This preserves the path to dbt virtual schemas and keeps the definition correct if the table moves.
- **No direct column references in migration-authored code.** All column references use `${foo_bar__qux.zip}` notation. There is no `${TABLE}` construct; `${TABLE}.column` fails at validation and query time with `Column "__omni_scoped" not found`. A raw column auto-maps by name and needs no `sql:` at all.
- **No net-new fields on existing views.** No relabeling, no `hidden` changes, no format changes on existing fields. Migration-authored calculations go in a migration-owned extension view or query view.
- **Use a measure `filters:` block rather than `CASE WHEN` for filtered aggregates.** Keep `sql` limited to the value being aggregated. In filter blocks, use the bare field name for fields on the measure's own view and fully qualify fields from a joined view. Booleans use the same `{ is: … }` operator form as every other field, not a bare scalar.
- **Determine the SQL dialect from the connection**, not from the connection's name, and use dialect-appropriate functions.
- **Validate fan-out and symmetric aggregation explicitly.** A declared primary key is necessary but not sufficient. Any measure reachable across a one-to-many join in a consolidated topic requires explicit validation. The consolidation rule in §3 increases exposure to this. A measure exceeding a measure it should be a subset of is the signature of fan-out from a non-distinct count on a view with a weak key.
- **Treat names as permanent.** Content breaks on rename. Labels are cosmetic; names are not. Check net-new names for collisions, including collisions that appear only after view-name scoping.

## 5. Content

- **All queries must be based on topics**, including single-view topics.
- **Dashboard filters must bind to modeled fields**, with default values and control types matching the source.
- **Window-shaped result columns are table calculations, not model fields.** Running totals, moving averages, period-over-period and month-over-month percentages, percent-of-total, and rank are computed per query. Modeling them as fields is incorrect.
- **Migrated content must be placed deliberately.** Folder, owner, and permissions are assigned as part of the migration (§7).

## 6. Workbook model and ad hoc content

Anything placed in the workbook model must be negotiated by exception. Nothing goes there by default.

Forbidden absent an approved exception:

- Workbook-level ad hoc dimensions and measures
- Workbook-level joins and relationships
- Raw-SQL tiles
- Any SQL-authored field at the workbook layer

**Table calculations are permitted** and are outside this rule. They live in the query rather than the workbook model, and per §5 they are the correct mechanism for presentation-layer computation. Using a table calculation to reimplement a metric that belongs in the model is not permitted.

**This must be enforced by a check.** Content that does not fit the shared model still renders from the workbook model, so drift there is invisible in review. A document's identifier is also its workbook model identifier: read each migrated document's workbook model and assert it is empty as a pipeline step.

**Exceptions require a named approver**, the reason recorded in the migration manifest (§11), and a tracked follow-up to model it properly.

## 7. Access and row-level controls

Row-level security in Omni is topic-only. A net-new topic has nothing to inherit: it starts as an unfiltered surface, and the migration must reconstruct the gating.

### 7.1 Discovery

Identifying which source content is gated or pre-filtered by user attributes in the incumbent system is a discovery deliverable produced before any topic is authored.

For each in-scope source object, discovery must produce: whether it is user-attribute gated, the gating expression, and the population it applies to.

This work belongs to the migrating party and to the export side of their tooling, since it depends on source-system expertise. There is no established practice for it among migration vendors today, so it should be scoped explicitly rather than assumed.

### 7.2 Routing

Each discovered gate routes to one of:

- **`access_filters`** — row-level filtering keyed to a user attribute. Use where the source gate resolves to "this user sees their own rows."
- **`always_where_sql`** or **`always_where_filters`** — a non-removable filter, where the constraint is not attribute-keyed or cannot be expressed as an attribute mapping.

Record the routing decision per source object in the manifest (§11), with the source expression alongside the Omni expression.

### 7.3 Required validations

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

Cover at least one user per distinct attribute value plus one user with no value set.

Automated checks that omit `userId` do not exercise this. Running under the API key's own identity returned zero rows and a `COMPLETE` status where a real user with the same missing attribute got a hard failure. Access-filter tests must name a real `userId`.

### 7.4 Other access requirements

- **Migrated content must not broaden access.** Where source and target access models do not map cleanly, escalate (§12).
- **Content permissions are assigned, not inherited.** Migration carries no access. Folder placement, ownership, and permits are deliverables.
- **Tooling must not modify connection permissions.** Schema scope changes are a §2 precondition.

## 8. dbt

- If dbt is integrated, everything must be built off virtual schemas.

---

# Part III — Development hygiene

## 9. Branch and model state

- **The branch is the isolation boundary.** Work on a branch is not visible to production AI chat, the topic picker, or any user who has not opened that branch. Review happens before merge, so no additional quarantine mechanism is required — merging is the act that exposes the work.

- **Scope each branch to a coherent subset of work** with a logical affiliation: a department, a content owner, or a model owner. The branch should have one reviewer who is competent to judge everything in it. This also narrows the set of files a branch touches, which is the only surface where a concurrent base change can be silently overwritten (below), and it shortens the window each freeze has to cover.

- **The model must return to the state it was in when the branch was opened.** No net-new errors and no net-new warnings. §13 defines how this is measured.

- **Every file the migration writes must be a file the migration owns.** The pull request reports the files it touched, and none may be a pre-existing model file. Global model settings — `alwaysScopeViewNames`, model-level defaults, connection bindings, `ai_chat_topics` — are out of bounds.

  The file set is discovered during the work, so this is enforced as an invariant on the completed pull request rather than as a pre-declared allowlist. It also supplies the definition of "my files" that the `yaml_path` gate in §13.2 depends on.

- **Model YAML must not be hand-committed to the git repository.** All writes go through Omni so that normalization happens on write. Hand-edited YAML produces corrective commits and unreviewable diffs. On a git-connected model, Omni regenerates the default branch from its own model state and deletes git-only model files that state does not contain.

- **An Omni branch is a live overlay on the current base, not a snapshot.** Verified on a scratch model:
  - Files the branch has **not** touched resolve from the base as it stands now. A topic added to the base *after* the branch was created resolves on that branch immediately. `--mode extension` returns only the branch's deltas; `--mode combined` returns current base plus those deltas.
  - Files the branch **has** touched shadow the base. Later base changes to the same file are invisible on the branch, and merging the branch overwrites them. In the test, the base was updated to a newer version of a file the branch had already modified; the merge replaced it with the branch's version, with no conflict, warning, or failure.

  This is why there is no rebase operation — for untouched files none is needed. The exposure is limited to the intersection of files the branch touched and files the base changed, and within that intersection the branch wins silently. Keep migration branches short-lived, and before merging, diff the set of files the branch touched against the base's current version of those same files.

- **Choose the target model up front.** If a hub-and-spoke arrangement exists, choose the spoke. A dedicated migration model extending the shared model is a third option; see the appendix. It has an unresolved exit path and should be evaluated separately rather than adopted by default.

## 10. Model write mechanics

These are properties of the YAML write API that silently produce wrong results when a tool assumes otherwise.

- **Writes are whole-file read-modify-write.** `yaml-create` replaces a file's authored content; it does not merge field by field. To change one field on an existing view, `yaml-get` the file, edit it, and write the complete file back. Anything omitted from the payload is dropped. Schema-layer base columns are unaffected, since they live outside the authored file.
- **`fileName` is an exact path on write, not a pattern.** Reuse the full-path key returned by `yaml-get`, including any folder prefix. A non-matching `fileName` does not error — Omni creates a new file at that path and returns `success: true`, producing a duplicate view at the repo root.
- **`success: true` means accepted, not correct.** After every write, re-list files and confirm the object does not now exist at two paths, and read the file back to confirm the intended content landed.
- **`branchId` must be a server-issued UUID.** Passing a branch name returns `400 Unrecognized key: "branchName"`.
- **Topic file names normalize to the repository root**, and the topic name is the filename stem rather than the view-scoped name. Pass that stem as both `topicName` and `join_paths_from_topic_name`.
- **Read back with the right mode.** `yaml-get` returns the extension layer by default — the authored deltas only. Use `--mode combined` to see what the model actually resolves to.

## 11. Manifest, idempotency, and rollback

- **A committed machine-readable manifest** mapping source object to target object, per tile, topic, view, and field.
- **Re-running the migration must not duplicate anything.** Object names must be deterministic functions of the source object rather than generated per run.
- **An explicit not-migrated list with reasons**, covering both what the tooling could not do (§12) and what triage excluded (§2).
- **An assumptions log** recording every judgment call the tooling made.
- **A rollback plan** describing how to un-merge once content depends on the new topics.

## 12. Unsupported constructs

Define the behavior for constructs the tooling cannot reproduce faithfully: unsupported visualization types, calculations with no Omni equivalent, source features with no analog, and access models that do not map.

Unsupported constructs must halt and be logged for human escalation. The tooling must not substitute an approximation. A gap is visible in the not-migrated list; an approximation is not.

---

# Part IV — Validation and acceptance

## 13. Validation gates

### 13.1 Severity

Model validation returns a JSON array of issue objects carrying `is_warning`, `message`, and `yaml_path`. `is_warning: false` is an error.

**The bar is no net-new errors and no net-new warnings.**

Warnings are fixed, not waived. The most common class, `No join path from …`, is produced by authoring a cross-view field in a view file rather than in the topic's `views:` block (§3). Correct placement eliminates it.

### 13.2 Gate on `yaml_path`

No pre-branch snapshot is needed. Both validators accept an optional branch, and main can be queried at any time:

```
omni models validate <modelId>                                       # baseline
omni models validate <modelId> --branchid <branchId>                 # branch
omni models content-validator-get <modelId>                          # baseline
omni models content-validator-get <modelId> --branch-id <branchId>   # branch
```

The two commands use different flag spellings (`--branchid` and `--branch-id`). Pin both in the tooling spec.

Filter issues by `yaml_path` against the files the migration owns (§9) rather than diffing branch against main. This is unaffected by base changes during the branch's life, and it prevents new breakage being attributed to pre-existing noise.

### 13.3 Query-level checks

For every migrated query, confirm in the response that `summary.missing_fields` is `[]` and that `summary.invalid_calculations` is empty. Note that `invalid_calculations` returns as `{}` rather than `[]`.

A non-empty `missing_fields` is the catch-all for a field that did not resolve: a model file dropped or renamed during a write or merge, or a topic whose joined view was declared in `relationships` without a corresponding `joins` entry — a combination that validates clean while leaving the joined view's fields unexposed.

**Fallback for query-restricted instances.** Where queries cannot be executed, assert exposure at the topic level instead:

```
omni models get-topic <modelId> <topicName> --branch-id <branchId>
```

The response must contain every field the migrated content references. This is weaker than running the queries, since it tests a field list rather than execution.

### 13.4 Content validator

The content validator reports broken references in saved content. It does not evaluate whether results are correct.

Two requirements:

- **No content-validator errors in newly generated content.** A document built early on the branch can be broken by model changes made later on the same branch — a field renamed, a topic restructured, a view scoped differently. Run the validator against the branch after the final model write, not at the point each document is created.
- **No net-new content-validator errors in pre-existing content**, measured as the branch-versus-main issue delta.

Mechanics:

- Default `--content-filter-mode` is `ALL`, which returns every document with at least one query rather than only documents with breakage. Tooling that treats returned rows as errors will flag the entire instance. Use `WITH_ISSUES` or parse the issue arrays.
- Issues appear in two arrays: `queries_and_issues[]` and `dashboard_filter_issues[]`. Reading only the first misses broken dashboard filters, which bears on the filter-parity requirement in §14.
- `--branch-id` does not filter the document set. It sets the validation context; the same documents are returned either way.
- Drafts are returned alongside published documents with `type: "draft"`. Migration output held in drafts is therefore in scope. Only documents cleared to zero query presentations are omitted.
- Consume the API response rather than the user interface. The validator's search filters on field, view, and topic references parsed from query definitions and does not read issue text. Breakage originating in the model, such as a broken `always_where` on a topic, attaches an error to every tile while the document contains no matching reference, so those rows cannot be reached through the interface filter. The errors are present in the API response.

### 13.5 Sequencing

Verify written YAML with a read-back (§10) before interpreting any validator output. Validator results against a branch whose contents differ from expectation are not meaningful.

### 13.6 No enforcement at merge

Omni provides no dry-run and no server-side gate. A branch carrying blocking validation errors can be merged. Every gate in this section is procedural and depends on the review process.

## 14. Parity

Structural rules do not establish that the numbers are correct. Parity is a separate gate. The access-filter validations in §7.3 are acceptance criteria alongside these.

- **Numeric parity is an acceptance criterion.** Compare every migrated tile against the source tool's output within a stated tolerance, and commit the comparison artifact with the pull request.
- **The business owner of the source content signs off on parity**, not the migrating party.
- **Verify date and time semantics explicitly:** timezone (connection, user, and source tool), week start day, fiscal calendar, and relative-date handling. Omni's relative-date expressions are calendar-based rather than rolling, so a migrated "last 2 years" filter can shift its boundaries.
- **Verify aggregate semantics explicitly:** count versus count-distinct, averages of averages, weighted metrics, null-versus-zero handling, and sort order.
- **Verify row limits.** Source tools often apply default caps. A migrated tile that inherits a different limit returns a different number.
- **Verify formatting** — rounding, currency, and number format.

---

## Appendix — A dedicated migration model

An alternative to writing migration output into the shared model: give the migration its own model that extends the shared model, and place everything it creates there.

**Advantages**

- The provenance requirement in §3 becomes structural rather than a tagging convention.
- The ownership invariant in §9 holds by construction, since the migration is not writing to the shared model at all.
- The migration is separable and revertible as a unit.
- Content that does not belong in the shared model has an obvious home, which reduces pressure toward the workbook model.

**Costs**

- **Backporting.** Moving a proven migration-created topic into the governed shared model is manual. There is no promotion path between models.
- **Document portability.** Content built against the migration model is bound to it. Moving those documents to the shared model is an export/import exercise with its own failure modes, not a reassignment.
- **Two surfaces to govern.** Access grants, AI configuration, and validation gates exist in both models.

The exit path is the unresolved part. Adopt this only where the migration output is expected to remain a distinct layer, or where the backport cost is accepted at the outset.
