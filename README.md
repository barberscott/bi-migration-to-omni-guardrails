# Automated BI Migration to Omni — Guardrails

**Status:** Draft
**Scope:** Any engagement where automated tooling migrates content from another BI tool into Omni. Written to be tool-agnostic and customer-agnostic; applies to Block, but is not specific to Block.

## How to read this document

These are the conditions a migration must satisfy to be accepted. They are written as constraints on the migrating party, not as implementation guidance.

Rules are grouped by what they protect. Where a rule depends on a specific Omni mechanism, the mechanism is named so the requirement is testable rather than aspirational. Where a rule cannot be enforced by Omni itself, that is stated explicitly — several of the most important gates here are procedural, and a reviewer who assumes the platform is enforcing them will be wrong.

Two principles underpin most of what follows:

1. **An outside migrating party cannot divine intent.** It cannot know why an existing topic is shaped the way it is, what an existing field is used for, or which of two plausible readings of a source metric is correct. Every rule that forbids modifying existing objects derives from this.
2. **Rendering is not migrating.** A dashboard that loads, passes validation, and shows numbers can still be wrong. Hygiene rules are necessary and not sufficient; parity is a separate and equally binding gate.

---

## 1. Preconditions — human-owned, completed before the migration branch opens

These are not migration tasks. They are prerequisites, owned by the customer's data team, and the tooling must never perform them.

- **Schema refresh.** A large class of "not found" validation errors are stale-schema artifacts rather than real breakage. Baselines taken against a stale schema are noise, and the migration will either be blocked by errors it did not cause or will be granted cover for errors it did. Refresh first, then baseline. Note that this requires Connection Admin, and that some connections reject branch-scoped refresh outright, leaving a shared refresh as the only option — which is precisely why it cannot sit inside the migration's branch.
- **Connection schema scope.** If a required database view is not in scope for the connection, the connection is edited to bring its schema into scope. This is a security-relevant act: widening scope exposes every table in that schema to every modeler on that connection. It requires data-team approval and is never performed by the tooling.
- **AI quarantine configuration.** The model-level `ai_chat_topics` property is configured once, up front, per §7. The tooling must never write this property.
- **Content inventory and triage.** A named owner decides what is in scope. A migration is an opportunity not to carry junk forward; without an explicit in-scope list, dead and duplicated source content gets faithfully reproduced.
- **Target model decision.** See §2.

---

## 2. Branch, model state, and where migration output lives

- **The model must be returned to the state it was in when the branch was opened.** No net-new errors. See §3 for how this is measured and why "and warnings" is the wrong bar.
- **Every file the migration writes is a file the migration owns.** Stated as an invariant rather than a pre-declared list, because the set of files is discovered during the work, not known up front: the pull request reports the files it touched, and none of them may be a pre-existing model file. Global model settings — `alwaysScopeViewNames`, model-level defaults, connection bindings, `ai_chat_topics` — are out of bounds outright.

  This is not an independent rule. It is the mechanical form of the prohibitions in §4 and §5 against modifying existing topics, views, and fields, collapsed into one checkable assertion — and it is what makes the `yaml_path` gate in §3.2 possible, since "issues in files that are mine" requires a definition of *mine*. If migration output lives in its own model or its own directory (above), the invariant is satisfied by construction and the check is trivial.
- **Model YAML is never hand-committed to the git repository.** Writes go through Omni so that normalization happens on write. Hand-edited YAML produces corrective-commit churn and diffs that cannot be reviewed.
- **Branch currency cannot be assumed, and on one path it cannot be achieved.** Main moves underneath long-running migration branches, and the two ship paths behave differently:
  - **Git-connected.** `omni models commit` merges the base into the git feature branch before creating or updating the pull request. The *git artifact* is therefore current with main automatically. The **Omni branch model you validated is not** — it is still forked from whenever the branch was created. Validation ran against the pre-merge state; the merged result is what ships. Re-validate after any commit that pulled main in.
  - **Not git-connected.** There is no rebase or pull operation for an Omni branch model. `create-branch`, `delete-branch`, and `merge-branch` are the whole surface; nothing refreshes a branch from main. A long-running branch cannot be made current, and merging it promotes its authored files over whatever the shared model has since acquired for the same files.
  - **Therefore:** keep migration branches short-lived, and detect drift by comparing against main rather than by rebasing. Treat branch age as a risk measure in its own right.
- **Decide which model receives the migration's output,** and if a hub-and-spoke arrangement is already in place, which spoke. A third option — a dedicated migration model that extends the shared model — is set out in the appendix; it has real advantages and an unresolved exit path, so it warrants separate evaluation rather than adoption by default.

---

## 3. Validation gates

### 3.1 Split the bar by severity

Model validation returns a bare JSON array of issue objects carrying `is_warning`, `message`, and `yaml_path`. Only `is_warning: false` is an error.

- **Errors are hard-blocking.**
- **Warnings are a reported delta requiring a written waiver, not an automatic block.** Warnings such as `No join path from …` are endemic on real models. A rule of "no net-new errors or warnings" is stricter than Omni's own bar and will either block every pull request or be quietly ignored.

### 3.2 Gate on `yaml_path`, not on a captured snapshot

No pre-branch snapshot is required. Both validators accept an optional branch, and main is queryable at any time:

```
omni models validate <modelId>                                  # baseline
omni models validate <modelId> --branchid <branchId>            # branch
omni models content-validator-get <modelId>                     # baseline
omni models content-validator-get <modelId> --branch-id <branchId>   # branch
```

Note the flag inconsistency between the two commands (`--branchid` versus `--branch-id`); pin both in the tooling spec.

Because the migration already declares a file allowlist (§2), the stronger gate is to filter issues by `yaml_path` against that allowlist rather than to diff branch against main. It is insensitive to main moving under a long-running branch, and it cannot be gamed by attributing new breakage to pre-existing noise.

### 3.3 Clean validation is not evidence that the topics work

The failure mode that matters most for automated tooling: a topic-scoped `relationships` entry written without the corresponding view `joins` entry **validates clean**, while the joined view's fields are silently not exposed. Queries against them return empty.

**Add a per-topic exposure gate.** For every net-new topic:

```
omni models get-topic <modelId> <topicName> --branch-id <branchId>
```

The returned resolved field set must contain every field the migrated content references. This is the check that catches a migration shipping empty tiles, and model validation will never catch it.

Note the ordering constraint with §8: a topic marked `hidden: true` returns 404 here while still validating clean. Run this gate before the quarantine flag is applied.

### 3.4 Content validator — what it does and does not tell you

The content validator finds **broken references in saved content**. It has no opinion about whether numbers are correct.

- The "no content validator errors in newly generated content" half of the rule is close to vacuous on its own. Content freshly generated from the branch's own topics will rarely carry broken references. The rule's real value is the second half: **no net-new breakage in pre-existing content**, measured as the branch-versus-main issue delta.
- **Default `--content-filter-mode` is `ALL`** — every document with at least one query, not breakage-only. Tooling that counts returned rows as errors will flag the entire instance. Use `WITH_ISSUES`, or parse the issue arrays.
- **Issues live in two arrays:** `queries_and_issues[]` and `dashboard_filter_issues[]`. Reading only the first misses broken dashboard filters, which matters directly for the filter-parity requirement in §9.
- **`--branch-id` does not filter which documents are returned.** It sets the validation context. The document set is the same either way.
- **Drafts are enumerated** alongside published documents, carrying `type: "draft"`. Migration output sitting in drafts is therefore in scope. The validator misses only documents explicitly cleared to zero query presentations.
- **Consume the raw API response, not the user interface.** The validator's search filters on field, view, and topic references parsed out of query definitions, and never reads issue text. Breakage originating in the model — a broken `always_where` on a topic, for instance — attaches an error to every tile while the document itself contains zero matching references, making it unreachable through the interface filter. The errors are present in the API response.

### 3.5 Sequencing

Content-validator results against a branch are meaningless if the branch YAML is not what it is assumed to be. Verify written YAML with a read-back before interpreting any validator output. A successful write response means the write was accepted, not that it landed in the intended file.

### 3.6 Nothing in Omni enforces any of this at merge

There is no dry-run and no server-side gate. A branch carrying blocking validation errors can be merged. Every gate in this section is procedural, and the review process is the only thing enforcing it.

---

## 4. Topics

- **All net-new topics are identified as created by the migration.** This allows post-merge separation of migration-driven work from anything pre-existing or created elsewhere. Provenance is carried as *metadata*, not prose: a tag, plus topic-level `owners:` naming an accountable human. Descriptions and `ai_context` are not the vehicle — it is not reasonable to expect the migrating party to characterize a topic's full breadth or intended use, and a wrong characterization is worse than none.
- **Notes on how a topic is used by the migrated content are carried in the topic itself,** as comments. These describe what the migration did, which the migrating party does know, rather than what the topic means.
- **Existing topics are never modified.** There is no way for an outside party to determine whether a modification is valid.
- **Topics must not duplicate existing topics.** If an existing topic can serve the query, it is used.
- **Net-new topics are consolidated by base view, not by query.** One topic per base view, covering the union of join paths the migrated queries require. A topic's join graph costs only what a query actually references, so unused joins in a consolidated topic are free at query time — which is what makes consolidation the right default rather than a compromise.
  - **The exception that forces a split** is a join-path or grain conflict: two different paths to the same view, or the same view needed at two different grains. These cannot coexist in one topic.
  - **The accepted cost** is a wider field list and increased fan-out exposure. Both are managed, not eliminated — see §5 and §8.

---

## 5. Views, dimensions, and measures

- **Every view used in a migration-generated topic — database view or query view — must have a declared primary key.**
- **Database views are never provisioned in the shared model.** A missing view is resolved by bringing its schema into connection scope (§1).
- **"Missing" must be established correctly before anything is done about it.** A model-wide `yaml-get` returns only views from currently-loaded schemas; views in offloaded or inactive schemas are absent from it while remaining perfectly available. An existence check must therefore run `omni models get-schemas` and, if the schema is listed, load it explicitly with `--includeschemas <SCHEMA>` (one schema per call) before concluding a view does not exist. This matters because the remedy for a genuinely missing view is a connection-scope change, which is a security-relevant act — a false negative here causes an unnecessary widening of data access.
- **Query views must either be modeled** (using the `query` parameters) **or reference all database tables, dimensions, and measures using `${database_view__table}` and fields using `${foo_bar__qux.zip}` notation** — never direct-table or direct-field references. This preserves the path to dbt virtual schemas.
- **No direct column references anywhere in migration-authored code.** All column references use `${foo_bar__qux.zip}` notation.
- **No net-new fields on existing views.** No relabeling, no `hidden` flips, no format changes to existing fields. This is the same intent argument as the rule against modifying existing topics, applied one layer down. Migration-authored calculations belong in a migration-owned extension view or query view, never appended to a shared view file.
- **Prefer a measure `filters:` block over `CASE WHEN` for filtered aggregates.** Keep `sql` focused on the value being aggregated.
- **Fan-out and symmetric aggregation must be validated explicitly.** Declared primary keys are necessary but not sufficient. Any measure reachable across a one-to-many join in a consolidated topic requires explicit validation. This is the highest-severity silent failure mode in an automated migration, and the consolidation rule in §4 increases exposure to it.
- **Names are permanent API contracts.** Content breaks on rename; labels are cosmetic and names are not. Net-new names require a collision check, including collisions that only appear after view-name scoping is applied.

---

## 6. Content

- **No content-validator errors in newly generated content,** and **no net-new content-validator errors in pre-existing content** — measured per §3.4.
- **All queries are based on topics,** even where the topic is a single-view topic.
- **Dashboard filters bind to modeled fields,** with default values and control types matching the source.
- **Migrated content is placed deliberately:** folder, owner, and default permissions are assigned as part of the migration, not inherited by accident. See §8.

---

## 7. Workbook model and ad hoc content

Anything that goes in the workbook model must be negotiated by exception only. Nothing goes there by default.

Stated explicitly, the following are forbidden absent an approved exception:

- Workbook-level ad hoc dimensions and measures
- Workbook-level joins and relationships
- Raw-SQL tiles
- Any SQL-authored field at the workbook layer

**Table calculations are permitted and are not part of this rule.** They live in the query rather than the workbook model, and they are the correct home for presentation-layer computation — running totals, percent-of-total, period-over-period. What is not permitted is using a table calculation to reimplement a metric that belongs in the model.

**Make the rule detective, not merely declarative.** The workbook model is where automated tooling drifts, because it is the path of least resistance: anything that does not fit the shared model lands there silently and still renders. Since a document's identifier is also its workbook model identifier, each migrated document's workbook model can be read and asserted empty as a pipeline check. This converts the policy into something that fails the build rather than something discovered in review.

**Exceptions require teeth:** a named approver, the reason recorded in the migration manifest (§11), and a tracked follow-up to model it properly. Without this, the exception path becomes the default path.

---

## 8. AI surface and discoverability

Net-new topics are quarantined from AI and from the topic picker until a human has reviewed and approved them.

**Mechanism:**

1. Every migration-created topic carries a tag, for example `migration_pending_review`.
2. Model-level `ai_chat_topics: [all_topics, -tag:migration_pending_review]`.
3. Topic-level `hidden: true` on each net-new topic keeps it out of the workbook picker.
4. **Approval consists of a human removing the tag and the `hidden` flag** from that topic's file.

**Why this shape:**

- **The default is AI-visible.** With no `ai_chat_topics` property present, all topics are available to AI chat. Doing nothing means the migration's topics go live in AI on merge day. The quarantine must be an affirmative act.
- **Negation by tag keeps the approval action out of shared files.** Approving a topic edits only the migration-owned topic file. An explicit allow-list would force an edit to the model file on every approval — churn, plus an opportunity to break AI for pre-existing topics.
- **The tag doubles as the provenance marker** required by §4: greppable, machine-readable, and making no semantic claim about the topic.

**Guardrails:**

- **The tooling must never write `ai_chat_topics`.** An empty list disables AI chat for the entire model; a tooling bug that emits one is a model-wide outage. Configuration is a one-time human prerequisite (§1).
- If the model already carries an explicit `ai_chat_topics` list, the negation is **appended**, not substituted.
- **Hiding a topic does not break content built on it.** Topic-level `hidden: true` removes the topic from the workbook picker while saved content continues to query it. The quarantine therefore costs nothing at runtime.
- **Apply `hidden` last.** A topic carrying `hidden: true` returns 404 from `get-topic` while still validating clean, so the exposure gate in §3.3 cannot inspect it. Run the exposure check first and set the quarantine flag afterwards, or the gate silently has nothing to look at.
- **This is not a security control.** It governs AI chat and discoverability only. Access control is §9.
- **Consolidation interaction:** consolidated topics are wide, and AI field curation becomes necessary as a topic approaches roughly 550 fields. `ai_fields` curation is part of the approval review, not a later cleanup.

---

## 9. Access, security, and row-level controls

**Row-level security in Omni is topic-only.** `access_filters`, `always_where_sql`, and `always_where_filters` are topic parameters. Views do not carry them. So a net-new topic is not failing to *inherit* protection — there is nothing to inherit. **Every net-new topic over sensitive data starts as an unfiltered surface by construction**, and the migration must positively reconstruct the gating rather than preserve it.

Object-level gating behaves differently. `required_access_grants` is available at view and field level as well as topic level, so grants on an existing view do carry into any topic that uses it. Grants govern whether a user sees an object at all; they do not filter rows. A grant is not a substitute for a row filter.

### 9.1 The discovery problem

The hard part sits upstream of Omni: **identifying which source content is gated or pre-filtered by user attributes in the incumbent system.** This is a first-class discovery deliverable produced before any topic is authored, not a modeling detail handled during authoring.

For each in-scope source object, discovery must produce: whether it is user-attribute gated, the gating expression, and the population it applies to.

**This discovery is the migrating party's responsibility**, and it belongs in the export side of their tooling — it depends on source-system expertise that only they have. It is called out here because there is no known-good practice for it among partner migrators today, so it should be scoped explicitly as work rather than assumed to be handled.

### 9.2 Routing

Each discovered gate routes to exactly one of:

- **`access_filters`** — row-level filtering keyed to a user attribute. The default where the source gate resolves to "this user sees their own rows."
- **`always_where_sql`** or **`always_where_filters`** — a non-removable filter, where the constraint is not attribute-keyed or cannot be expressed as an attribute mapping.

The routing decision is recorded per source object in the manifest (§12), carrying the source expression alongside the Omni expression.

### 9.3 Three validations, all required

1. **The right attribute.** That the chosen Omni user attribute is the correct semantic counterpart of the source gate. This is an intent question, so it requires the source content owner to confirm it. The migrating party cannot determine it.
2. **The attribute's `default_value` — not merely whether users have values.** This is the exposure vector, and it runs opposite to intuition. Verified empirically on a two-country dataset (USA 123,933 rows / UK 25,907):

   | Attribute state for the querying user | Result |
   |---|---|
   | Value set, matching rows | Correctly filtered (UK user saw 25,907) |
   | No value, attribute has **no default** | Query **fails** — `error_type: PLAN` |
   | No value, attribute **has a default** | Silently returns the **default's** rows (unprovisioned user received 123,933 — 83% of the data) |

   So a null attribute is safe-but-broken: it fails closed, and the user gets an error rather than data they should not see. **A defaulted attribute is the danger.** A user who was never provisioned silently receives whatever the default grants, with no error and no signal anywhere in the model or in validation output.

   Therefore:
   - **Audit `default_value` on every attribute used in an access filter.** A null default is the safe configuration. Never set a default that is a real, data-bearing value.
   - **Audit `values_for_unfiltered`.** It is an explicit bypass list; any value in it that a user can actually hold is a hole.
   - **Per-user population is an availability check, not a security one.** Unpopulated users get a broken dashboard, which arrives as a support ticket rather than a breach — provided the default is null.
   - **`omni user-attributes list` confirms only that the definition exists,** and shows `default_value`. Per-user values require a SCIM read-back.
3. **The filter actually filters.** Test by running the query as the target user and confirming the result set is restricted:

```
omni query run --body '{ "query": { ... }, "userId": "<target-user-uuid>" }'
```

Run this for at least one user per distinct attribute value, plus one user with no value set. All three are acceptance criteria, not spot checks.

**Automated checks that omit `userId` do not test this.** Running under the API key's own identity returned zero rows and a `COMPLETE` status where a real user with the same missing attribute got a hard failure. A migration pipeline validating its own output as a service identity will not reproduce what a real user experiences — every access-filter test must name a real `userId`.

### 9.4 Remaining access rules

- **Migrated content must never broaden who can see what.** Where source and target access models do not map cleanly, that is an escalation (§11), not a judgment call for the migrating party.
- **Content permissions are assigned, not inherited.** Migration carries no access. Folder placement, ownership, and permits are explicit deliverables.
- **Connection permissions are never modified by the tooling.** Schema scope changes are a §1 precondition.

---

## 10. Parity and correctness

Hygiene rules do not establish that the numbers are right. Parity is a separate, binding gate.

- **Numeric parity is an acceptance criterion.** Every migrated tile is compared against the source tool's output within a stated tolerance, and the comparison artifact is committed with the pull request. Without this, "migrated" means "renders."
- **Parity is signed off by the business owner of the source content,** not by the migrating party. This is the same argument that governs §4: an outside party cannot determine intent.
- **Date and time semantics** require explicit verification: timezone (connection versus user versus source tool), week start day, fiscal calendar, and relative-date handling. Omni's relative-date expressions are calendar-based rather than rolling — a migrated "last 2 years" filter can silently shift its boundaries.
- **Aggregate semantics** require explicit verification: count versus count-distinct, averages of averages, weighted metrics, null-versus-zero handling, and sort order.
- **Row limits.** Source tools frequently apply silent default caps. A migrated tile that inherits a different limit produces a different number.
- **Formatting** — rounding, currency, and number format — is part of parity, not cosmetics.

---

## 11. Escalation: fail loudly, never approximate

There must be a defined behavior for constructs the tooling cannot faithfully reproduce: unsupported visualization types, calculations with no Omni equivalent, source features with no analog, and access models that do not map.

**Unsupported constructs halt and are logged for human escalation. The tooling never silently substitutes an approximation.** The default behavior of a generative tool is to produce something plausible, and a plausible substitute is materially worse than a gap, because a gap is visible and a substitute is not.

---

## 12. Manifest, idempotency, and reversibility

- **A committed, machine-readable manifest** mapping source object to target object — per tile, per topic, per view, per field. This answers "where did this come from" after the merge, and it is a stronger provenance mechanism than comments.
- **Re-running the migration must not duplicate anything.** Object names must be deterministic functions of the source object rather than generated per run.
- **An explicit not-migrated list, with reasons,** covering both what the tooling could not do (§11) and what triage decided not to carry forward (§1).
- **An assumptions log** recording every place the tooling made a judgment call.
- **A rollback plan** describing what un-merging looks like once content depends on the new topics.

---

## 13. dbt

- **If dbt is integrated, everything is built off virtual schemas.**

---

## Open questions

- What tolerance is acceptable for numeric parity (§10), and does it vary by metric class?
- Who is the named approver for workbook-model exceptions (§7), and is that a single person or per-domain?

---

## Appendix — A dedicated migration model

An alternative to writing migration output into the shared model: give the migration its own model that `extends` the shared model, and put everything it creates there.

**What it buys**

- The provenance requirement in §4 becomes structural rather than a tagging convention. Everything in the model is migration output by definition.
- The ownership invariant in §2 is satisfied by construction — the migration cannot modify a pre-existing file, because it is not writing to that model at all.
- The migration is separable and revertible as a unit.
- Most of the pressure that pushes output into the workbook model disappears, because there is now an obvious place for work that does not belong in the shared model.

**What it costs**

- **Backporting.** When a migration-created topic proves itself and should join the governed shared model, moving it is a manual exercise. There is no promotion path between models.
- **Document portability.** Content built against the migration model is bound to it. Bringing those documents into the shared model is an export/import exercise with its own failure modes, not a reassignment.
- **Two surfaces to govern.** Access grants, AI configuration, and validation gates all now exist in two places.

The ergonomics of the exit path are the open question. Adopt this only where the migration output is expected to remain a distinct layer, or where the backport cost is understood and accepted at the outset.
