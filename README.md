# Automated BI Migration to Omni — Guardrails

**Status:** Draft
**Audience:** Teams building automated migration tooling that targets Omni.
**Scope:** Any engagement where tooling migrates content from another BI tool into Omni. Tool-agnostic and customer-agnostic.

## Purpose

This document lists the conditions a migration must satisfy to be accepted. It states requirements, not implementation approaches.

Where a requirement depends on a specific Omni mechanism, the mechanism is named so the requirement can be tested. Where Omni does not enforce a requirement itself, that is stated; several of the gates below are procedural only.

Two constraints shape most of the rules:

- The migrating party has no way to determine the intent behind existing model objects. Rules that forbid modifying existing objects follow from this.
- Content that renders correctly can still return wrong numbers. Structural rules are necessary but not sufficient, and parity (§15) is a separate gate.

The document is in four parts: preconditions the customer owns, standards the output must meet, hygiene the development process must follow, and the gates that determine acceptance.

---

# Part I — Alignment, preconditions, and discovery

## 1. Customer alignment

Agreements to reach before any migration work begins. These are organizational rather than technical, and the gates in Part IV cannot be evaluated without them.

### 1.1 Model freeze

**The customer freezes the target shared model for the duration of the migration.** For the freeze window: no changes to views, topics, relationships, or model-level settings; no schema refreshes; no connection-scope or connection-environment changes.

This is required by Omni's branch semantics, not by convention:

- Files the migration branch has touched shadow the base. A base change to the same file is invisible from the branch and is overwritten when the branch merges — no conflict, no warning, no failure (§10).
- Files the branch has not touched resolve live from the base. A field renamed or removed on the base breaks migrated content mid-flight, and the breakage presents as a migration defect.
- Both validators are baselined against main. If main moves, the branch-versus-main delta changes for reasons unrelated to the migration and the §14 gates stop being meaningful.

**Define the exception path.** Where a change cannot wait, require notification before it lands, a re-baseline of both validators, and re-validation of any migrated content touching the affected files. Record each exception in the manifest (§12).

### 1.2 Source system freeze

The in-scope source content is frozen for the same window. Parity (§15) compares migrated output against the source tool, which requires the source to hold still.

### 1.3 Named owners

Name these roles before work begins:

- **Parity sign-off** per migrated dashboard — the business owner of the source content (§15).
- **Workbook-model exceptions** — a single approver, or one per domain (§8).
- **Connection schema-scope changes** — a data-team approver (§2).
- **Content inventory and triage** — the owner of the in-scope decision (§2).
- **Access gating** — the source content owner who confirms user-attribute mappings (§9.2).

### 1.4 Agreed scope and tolerances

- **The in-scope content list is agreed and recorded** before work begins. Excluded content is recorded with a reason (§12).
- **Numeric parity tolerance is agreed**, including whether it varies by metric class (§15).

---

## 2. Preconditions

The following are prerequisites owned by the customer's data team. Migration tooling must not perform them.

- **Schema refresh.** Run a schema refresh before the migration branch is opened. Many "not found" validation errors are stale-schema artifacts rather than real breakage, and a baseline taken against a stale schema cannot distinguish the two. Refresh requires Connection Admin. Some connections reject branch-scoped refresh, so a shared refresh is the only option on those connections, which is why this cannot sit inside the migration's branch.
- **Connection schema scope.** If a required database view is not in scope for the connection, the connection must be edited to bring its schema into scope. Widening scope exposes every table in that schema to every modeler on that connection, so this requires data-team approval.
- **Virtual schemas.** Where the customer intends to use dbt virtual schemas, they must exist before the migration begins (§6).
- **Model access for the migrating identity.** Branching requires full-model access. Confirm with `omni whoami whoami --modelid <modelId>` before work begins.
- **Content inventory and triage.** A named owner determines what is in scope. Without an explicit in-scope list, dead and duplicated source content is reproduced along with everything else.
- **Target model.** Decide which model receives the migration's output (§10).

---

## 3. Discovery

Findings the migration must produce before topics are authored or content is built. Each one settles a decision that is expensive to reverse once work is under way. Producing them belongs to the migrating party and to the export side of their tooling, since it depends on source-system expertise.

### 3.1 Platform and model context

- **dbt usage and virtual schema intent.** Whether dbt is integrated and whether the customer intends to use virtual schemas. Determines what everything is built against (§6); the schemas must exist before work begins (§2).
- **Connection dialect**, read from the connection rather than inferred from its name. Determines the SQL available for custom dimensions and for aggregates with no built-in equivalent (§4.4, §4.5).
- **Whether the target model is git-connected.** Determines the ship path and the merge semantics (§10).
- **Existing model coverage** — the topics and views already present that could serve migrated queries. Without this the rule against duplicating existing topics (§5) cannot be honoured.
- **The current validation baseline** — errors and warnings already present on the target model, so that net-new ones can be distinguished (§14).

### 3.2 Source content

- **The in-scope content list** and the owner of each item (§1.3, §1.4).
- **Attribute-filtered content.** Which source content is gated or pre-filtered by user attributes, the gating expression, and the population it applies to. For each in-scope source object, record whether it is gated and how. There is no established practice for this among migration vendors today, so it should be scoped explicitly rather than assumed. Feeds §9.
- **Multi-fact content.** Which content combines measures from more than one fact table at different grains. Tableau's relationships model handles this natively, so it is not apparent from the workbook without looking for it. Determines composite topic versus a single topic (§5).
- **Source access model** beyond attribute filtering — who can see each item today. Required to show that migrated content does not broaden access (§9.3).
- **Grain of each source data source**, which determines primary key selection (§4.1).
- **Calculation semantics** — which source calculations are model-level and which are presentation-level (§7).
- **Source model descriptions** on views, fields, and their equivalents, for porting (§4.7).
- **Parity inputs** — filter defaults, relative date expressions, row limits, timezone, fiscal calendar, and week start. Parity cannot be assessed without them (§15).

---

# Part II — Modeling and content standards

## 4. Views, dimensions, and measures

### 4.1 Primary keys and aggregation correctness

- **Every view used in a migration-generated topic must have a declared primary key**, whether database view or query view. Omni keeps `sum`, `count`, and `avg` correct under one-to-many joins by deduplicating on the primary key; a view without one cannot be made symmetric, and a row-count measure on it inflates whenever a join duplicates its rows.
  - A single unique column is declared with `primary_key: true` on that dimension.
  - Where no single column is unique, use the built-in compound key: `custom_compound_primary_key_sql: [field_a, field_b]` at the view level, with no `primary_key: true` dimension. Do not hand-build a concatenated key expression. The same parameter accepts a single column where that is more convenient.
  - Where no combination of columns is row-unique, do not declare a key at all. Fix the grain or do not join the view. A declared key that is not actually unique is worse than none.
- **Validate fan-out and symmetric aggregation explicitly.** A declared primary key is necessary but not sufficient. Any measure reachable across a one-to-many join in a consolidated topic requires explicit validation. The consolidation rule in §5 increases exposure to this. A measure exceeding a measure it should be a subset of is the signature of fan-out from a non-distinct count on a view with a weak key.

### 4.2 View provisioning and discovery

- **Database views must not be provisioned in the shared model.** Resolve a missing view by bringing its schema into connection scope (§2).
- **Establish that a view is missing before acting on it.** A model-wide `yaml-get` returns only views from currently-loaded schemas; views in offloaded or inactive schemas are absent from the response while remaining available. Run `omni models get-schemas` and, if the schema is listed, load it with `--includeschemas <SCHEMA>` (one schema per call) before concluding a view does not exist. The remedy for a genuinely missing view is a connection-scope change, so a false negative here widens data access unnecessarily.

### 4.3 Query views

- **Query views must either be modeled on a topic** (using the `query` parameters) **or reference all database tables, dimensions, and measures as `${database_view__table}` and fields as `${foo_bar__qux.zip}`.** No direct-table or direct-field references. In a query view's `sql:` block this means `${view_name}` rather than a hard-coded `CATALOG.SCHEMA.TABLE` path. This preserves the path to dbt virtual schemas and keeps the definition correct if the table moves.

### 4.4 Field and column references

- **No direct column references in migration-authored code.** Reference a field on the same view as `${zip}`, and a field on another view as `${foo_bar__qux.zip}`. Never reference a database column directly.
- **Determine the SQL dialect from the connection**, not from the connection's name, and use dialect-appropriate functions where needed for custom dimensions.

### 4.5 Measures

- **Do not re-create built-in aggregates.** Declare the aggregate and keep `sql` to the value being aggregated: `aggregate_type: sum` with `sql: ${zip}`, not `sql: SUM(${zip})`.
- **Use a measure `filters:` block rather than `CASE WHEN` for filtered aggregates.** In filter blocks, use the bare field name for fields on the measure's own view and fully qualify fields from a joined view. Booleans use the same `{ is: … }` operator form as every other field, not a bare scalar.
- **Determine the SQL dialect from the connection**, not from the connection's name, and use dialect-appropriate functions for any aggregate with no built-in equivalent.

### 4.6 Changes to existing views

- **Adding fields to existing views is permitted.** Tag migration-added fields with the same migration tag used for topics (§5) so they remain identifiable after the merge.
- **Placement is determined by join dependency.** A field whose `sql` references another view belongs in the topic's `views:` block, because not every topic containing the host view carries the required join (§5). A field that depends only on its own view's columns belongs on the view.
- **Existing fields must not be changed.** No label, `hidden`, format, or `sql` changes. These alter something already in use, and the effect reaches every topic and every dashboard referencing the field — content the migration does not own and cannot assess. Adding a description where none exists is covered by §4.7 and is not a change in this sense.

### 4.7 Descriptions

- **Port descriptions from the source model.** Where the incumbent tool carries descriptions on its model objects — Looker's `description` on views, dimensions, measures, and explores, for example — migrate them to the Omni equivalent. This metadata is costly to recreate and is silently lost if the migration ignores it. The rule covers topics as well as views and fields.
- **Never overwrite a description that already exists.** Where the Omni object already carries one, leave it. It reflects a decision made in Omni that the migrating party cannot evaluate, and the source description is not necessarily the more current of the two.
- This is distinct from the prohibition in §5 on using `description` for provenance. Porting a description written by the source model's author carries existing metadata forward; authoring one means the migrating party characterizing an object it does not understand.

### 4.8 Naming

- **Treat names as permanent.** Content breaks on rename. Labels are cosmetic; names are not. Check net-new names for collisions, including collisions that appear only after view-name scoping.

## 5. Topics

- **Net-new topics must be identifiable as migration output.** This allows migration-driven work to be separated from pre-existing work after the merge. Carry provenance as metadata — a tag, plus topic-level `owners:` naming an accountable person. Do not use `description` or `ai_context` for this; the migrating party cannot characterize a topic's intended use, and an inaccurate characterization is worse than none.
- **Notes on how the migrated content uses a topic belong in the topic as comments.** These record what the migration did, which the migrating party does know.
- **Existing topics must not be modified.**
- **Topics must not duplicate existing topics.** If an existing topic can serve the query, use it.
- **Consolidate net-new topics by base view, not by query.** One topic per base view, covering the union of join paths the migrated queries require. A topic's join graph costs only what a query references, so unused joins in a consolidated topic add no query-time cost.
  - *Exception:* a join-path or grain conflict — two different paths to the same view, or the same view needed at two grains — forces a split.
  - *Accepted cost:* a wider field list and greater fan-out exposure. See §4 for fan-out.
- **Source content spanning multiple fact tables migrates to a composite topic.** Where the source combines measures from two or more fact tables at different grains against shared dimensions, joining them physically in one topic multiplies each fact's rows against the others before aggregation. Given correct primary keys and relationship types the results are still right — symmetric aggregates deduplicate on the key — but the query does far more work than the question requires, and the cost compounds with each additional fact. A composite topic aggregates each constituent topic independently in its own subquery and stitches the much smaller aggregated results on the conformed dimensions with a full outer join.
  - **Build a topic per fact domain first, then stitch them.** A composite topic is assembled from constituent topics, so multi-fact content produces one topic per domain plus the composite that joins them — not one topic, and not a topic per query.
  - **Consolidation applies to each constituent topic.** Every domain topic follows the rule above: one per base view, covering the union of join paths its own queries require. The composite is a layer on top of that, not a replacement for it.
  - **Declare the conformed dimensions with `mappings:`**, pointing each constituent topic at its own field.
  - **Composite topics are authorable through the model API** as a `.composite_topic` file, so this is implementable by tooling rather than IDE-only. Content querying one addresses constituent fields as `@<topic>.<view>.<field>` and shared dimensions as `@_shared_dimensions_.<name>`.
  - **Detection is a discovery obligation** (§3.2). Tableau's relationships model handles multi-fact analysis natively, so source content relying on it is not apparent from the workbook without looking for it. Identify content whose measures originate in more than one fact table before topics are authored, not at parity testing.
- **Cross-view fields belong in the topic's `views:` block, not in a view file.** A dimension or measure whose `sql` references `${other_view.field}` depends on that view being joined, which is not guaranteed in every topic containing the host view. Defining it in the view file produces validator errors in every topic that includes the host view without the referenced view, cascading across topics that are otherwise correct. Define it in the topic, where the join context is explicit. Only put a cross-view field in a view file when the referenced view is joined in every topic that includes the host view.
- **Check the global relationships file before authoring a topic-scoped relationship.** If a join between the same two views already exists in either direction with the same `on_sql`, the topic-scoped copy is redundant — use `joins` alone. If the `on_sql` differs, use the extended-views pattern rather than silently overriding the global join.
- **Join the same view multiple ways with the extended-views pattern**, not `join_to_view_as`. The error `relationship alias duplicates view name` indicates the wrong pattern was used.

  The pattern creates a named alias view that `extends` the original, relabels its fields for the role it plays, and joins on its own condition. Define it in the topic, so the alias, its join condition, and its use are all visible in one file. Where `order_items` needs `users` as both buyer and seller:

  ```yaml
  # .topic file
  base_view: order_items

  views:
    sellers:
      extends: [users]
      dimensions:
        name:
          label: Seller Name

  relationships:
    - join_from_view: order_items
      join_to_view: sellers
      on_sql: ${order_items.seller_id} = ${sellers.id}
      relationship_type: many_to_one
      join_type: always_left

  joins:
    sellers: {}
    users: {}
  ```

  Promote the alias to a standalone `.view` file with a global relationship only once several topics need the same role. Until then the topic-scoped form keeps the definition next to its use.

- **Before adding a topic-scoped field to an existing view, confirm the field does not already exist.** A field of the same name with different SQL is an override: queries through that topic use the topic-scoped definition while every other topic keeps the shared one. Overrides require explicit approval.

## 6. dbt

- **If dbt is integrated, everything must be built off virtual schemas.**
- **Virtual schemas must be in place before the migration begins.** Where the customer intends to use them, standing them up is a precondition (§2) rather than migration work. Authoring against physical schemas and converting afterwards means reworking every view, query view, and topic the migration produced.

---

## 7. Content

- **All queries must be based on topics**, including single-view topics.
- **Dashboard filters must bind to modeled fields**, with default values and control types matching the source.
- **Window-shaped result columns are table calculations, not model fields.** Running totals, moving averages, period-over-period and month-over-month percentages, percent-of-total, and rank are computed per query. Modeling them as fields is incorrect.
- **Migrated content must be placed deliberately.** Folder, owner, and permissions are assigned as part of the migration (§9).

## 8. Workbook model and ad hoc content

Anything placed in the workbook model must be negotiated by exception. Nothing goes there by default.

Forbidden absent an approved exception:

- Workbook-level ad hoc dimensions and measures
- Workbook-level joins and relationships
- Raw-SQL tiles
- Any SQL-authored field at the workbook layer

**Table calculations are permitted** and are outside this rule. They live in the query rather than the workbook model, and per §7 they are the correct mechanism for presentation-layer computation. Using a table calculation to reimplement a metric that belongs in the model is not permitted.

**This must be enforced by a check.** Content that does not fit the shared model still renders from the workbook model, so drift there is invisible in review. A document's identifier is also its workbook model identifier: read each migrated document's workbook model and assert it is empty as a pipeline step.

**Exceptions require a named approver**, the reason recorded in the migration manifest (§12), and a tracked follow-up to model it properly.

## 9. Access and row-level controls

Row-level security in Omni is topic-only. A net-new topic has nothing to inherit: it starts as an unfiltered surface, and the migration must reconstruct the gating.

### 9.1 Routing

Each discovered gate routes to one of:

- **`access_filters`** — row-level filtering keyed to a user attribute. Use where the source gate resolves to "this user sees their own rows."
- **`always_where_sql`** or **`always_where_filters`** — a non-removable filter, where the constraint is not attribute-keyed or cannot be expressed as an attribute mapping.

Record the routing decision per source object in the manifest (§12), with the source expression alongside the Omni expression.

### 9.2 Required validations

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

### 9.3 Other access requirements

- **Migrated content must not broaden access.** Where source and target access models do not map cleanly, escalate (§13).
- **Content permissions are assigned, not inherited.** Migration carries no access. Folder placement, ownership, and permits are deliverables.
- **Tooling must not modify connection permissions.** Schema scope changes are a §2 precondition.

# Part III — Development hygiene

## 10. Branch and model state

- **The branch is the isolation boundary.** Work on a branch is not visible to production AI chat, the topic picker, or any user who has not opened that branch. Review happens before merge, so no additional quarantine mechanism is required — merging is the act that exposes the work.

- **Scope each branch to a coherent subset of work** with a logical affiliation: a department, a content owner, or a model owner. The branch should have one reviewer who is competent to judge everything in it. This also narrows the set of files a branch touches, which is the only surface where a concurrent base change can be silently overwritten (below), and it shortens the window each freeze has to cover.

- **The model must return to the state it was in when the branch was opened.** No net-new errors and no net-new warnings. §14 defines how this is measured.

- **Every file the migration writes must be a file the migration owns.** The pull request reports the files it touched, and none may be a pre-existing model file. Global model settings — `alwaysScopeViewNames`, model-level defaults, connection bindings, `ai_chat_topics` — are out of bounds.

  The file set is discovered during the work, so this is enforced as an invariant on the completed pull request rather than as a pre-declared allowlist. It also supplies the definition of "my files" that the `yaml_path` gate in §14.2 depends on.

- **Model YAML must not be hand-committed to the git repository.** All writes go through Omni so that normalization happens on write. Hand-edited YAML produces corrective commits and unreviewable diffs. On a git-connected model, Omni regenerates the default branch from its own model state and deletes git-only model files that state does not contain.

- **An Omni branch is a live overlay on the current base, not a snapshot.** Verified on a scratch model:
  - Files the branch has **not** touched resolve from the base as it stands now. A topic added to the base *after* the branch was created resolves on that branch immediately. `--mode extension` returns only the branch's deltas; `--mode combined` returns current base plus those deltas.
  - Files the branch **has** touched shadow the base. Later base changes to the same file are invisible on the branch, and merging the branch overwrites them. In the test, the base was updated to a newer version of a file the branch had already modified; the merge replaced it with the branch's version, with no conflict, warning, or failure.

  This is why there is no rebase operation — for untouched files none is needed. The exposure is limited to the intersection of files the branch touched and files the base changed, and within that intersection the branch wins silently. Keep migration branches short-lived, and before merging, diff the set of files the branch touched against the base's current version of those same files.

- **Choose the target model up front.** If a hub-and-spoke arrangement exists, choose the spoke. A dedicated migration model extending the shared model is a third option; see the appendix. It has an unresolved exit path and should be evaluated separately rather than adopted by default.

## 11. Model write mechanics

These are properties of the YAML write API that silently produce wrong results when a tool assumes otherwise.

- **Writes are whole-file read-modify-write.** `yaml-create` replaces a file's authored content; it does not merge field by field. To change one field on an existing view, `yaml-get` the file, edit it, and write the complete file back. Anything omitted from the payload is dropped. Schema-layer base columns are unaffected, since they live outside the authored file.
- **`fileName` is an exact path on write, not a pattern.** Reuse the full-path key returned by `yaml-get`, including any folder prefix. A non-matching `fileName` does not error — Omni creates a new file at that path and returns `success: true`, producing a duplicate view at the repo root.
- **`success: true` means accepted, not correct.** After every write, re-list files and confirm the object does not now exist at two paths, and read the file back to confirm the intended content landed.
- **`branchId` must be a server-issued UUID.** Passing a branch name returns `400 Unrecognized key: "branchName"`.
- **Topic file names normalize to the repository root**, and the topic name is the filename stem rather than the view-scoped name. Pass that stem as both `topicName` and `join_paths_from_topic_name`.
- **Know which layer you are reading.** `yaml-get` returns the extension layer by default — the authored deltas only. `--mode combined` returns the composed result, schema base included.
- **Never write with `mode: merged`.** Write modes differ in what the posted YAML is deduplicated against, and this is the source of authored-layer bloat. Posting one 646-byte view body — a combined read differing from the schema base by a single label — produced these authored layers:

  | Write mode | Authored layer |
  |---|---|
  | `combined` (default) | 83 bytes — the label only |
  | `extension` | 83 bytes |
  | `staged` | 83 bytes |
  | `merged` | **646 bytes — every schema base dimension materialized** |

  The reason is the pruning pass. A save in `extension` or `staged` is followed by an automatic re-save in combined mode that strips properties already supplied by the schema layer; `combined` needs no such pass because it prunes on the way in. `merged` is excluded from it, so whatever was posted persists as authored content. The default is correct; select a mode deliberately or not at all.
- **Assert the authored layer after every write.** Read back with `--mode extension` and confirm it contains only the intended delta and not a materialized copy of the schema base. Dedup depends on the base being resolvable, so a write against a view whose base does not resolve — an offloaded or inactive schema, a table no longer present — persists the whole posted body as authored content. Model bloat accrues silently this way and is expensive to unwind later.

## 12. Manifest, idempotency, and rollback

- **A committed machine-readable manifest** mapping source object to target object, per tile, topic, view, and field.
- **Re-running the migration must not duplicate anything.** Object names must be deterministic functions of the source object rather than generated per run.
- **An explicit not-migrated list with reasons**, covering both what the tooling could not do (§13) and what triage excluded (§2).
- **An assumptions log** recording every judgment call the tooling made.
- **A rollback plan** describing how to un-merge once content depends on the new topics.

## 13. Unsupported constructs

Define the behavior for constructs the tooling cannot reproduce faithfully: unsupported visualization types, calculations with no Omni equivalent, source features with no analog, and access models that do not map.

Unsupported constructs must halt and be logged for human escalation. The tooling must not substitute an approximation. A gap is visible in the not-migrated list; an approximation is not.

---

# Part IV — Validation and acceptance

## 14. Validation gates

### 14.1 Severity

Model validation returns a JSON array of issue objects carrying `is_warning`, `message`, and `yaml_path`. `is_warning: false` is an error.

**The bar is no net-new errors and no net-new warnings.**

Warnings are fixed, not waived. The most common class, `No join path from …`, is produced by authoring a cross-view field in a view file rather than in the topic's `views:` block (§5). Correct placement eliminates it.

### 14.2 Gate on `yaml_path`

No pre-branch snapshot is needed. Both validators accept an optional branch, and main can be queried at any time:

```
omni models validate <modelId>                                       # baseline
omni models validate <modelId> --branchid <branchId>                 # branch
omni models content-validator-get <modelId>                          # baseline
omni models content-validator-get <modelId> --branch-id <branchId>   # branch
```

The two commands use different flag spellings (`--branchid` and `--branch-id`). Pin both in the tooling spec.

Filter issues by `yaml_path` against the files the migration owns (§10) rather than diffing branch against main. This is unaffected by base changes during the branch's life, and it prevents new breakage being attributed to pre-existing noise.

### 14.3 Query-level checks

For every migrated query, confirm in the response that `summary.missing_fields` is `[]` and that `summary.invalid_calculations` is empty. Note that `invalid_calculations` returns as `{}` rather than `[]`.

A non-empty `missing_fields` is the catch-all for a field that did not resolve: a model file dropped or renamed during a write or merge, or a topic whose joined view was declared in `relationships` without a corresponding `joins` entry — a combination that validates clean while leaving the joined view's fields unexposed.

**Fallback for query-restricted instances.** Where queries cannot be executed, assert exposure at the topic level instead:

```
omni models get-topic <modelId> <topicName> --branch-id <branchId>
```

The response must contain every field the migrated content references. This is weaker than running the queries, since it tests a field list rather than execution.

### 14.4 Content validator

The content validator reports broken references in saved content. It does not evaluate whether results are correct.

Two requirements:

- **No content-validator errors in newly generated content.** A document built early on the branch can be broken by model changes made later on the same branch — a field renamed, a topic restructured, a view scoped differently. Run the validator against the branch after the final model write, not at the point each document is created.
- **No net-new content-validator errors in pre-existing content**, measured as the branch-versus-main issue delta.

Mechanics:

- Default `--content-filter-mode` is `ALL`, which returns every document with at least one query rather than only documents with breakage. Tooling that treats returned rows as errors will flag the entire instance. Use `WITH_ISSUES` or parse the issue arrays.
- Issues appear in two arrays: `queries_and_issues[]` and `dashboard_filter_issues[]`. Reading only the first misses broken dashboard filters, which bears on the filter-parity requirement in §15.
- `--branch-id` does not filter the document set. It sets the validation context; the same documents are returned either way.
- Drafts are returned alongside published documents with `type: "draft"`. Migration output held in drafts is therefore in scope. Only documents cleared to zero query presentations are omitted.
- Consume the API response rather than the user interface. The validator's search filters on field, view, and topic references parsed from query definitions and does not read issue text. Breakage originating in the model, such as a broken `always_where` on a topic, attaches an error to every tile while the document contains no matching reference, so those rows cannot be reached through the interface filter. The errors are present in the API response.

### 14.5 Sequencing

Verify written YAML with a read-back (§11) before interpreting any validator output. Validator results against a branch whose contents differ from expectation are not meaningful.

### 14.6 No enforcement at merge

Omni provides no dry-run and no server-side gate. A branch carrying blocking validation errors can be merged. Every gate in this section is procedural and depends on the review process.

## 15. Parity

Structural rules do not establish that the numbers are correct. Parity is a separate gate. The access-filter validations in §9.2 are acceptance criteria alongside these.

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

- The provenance requirement in §5 becomes structural rather than a tagging convention.
- The ownership invariant in §10 holds by construction, since the migration is not writing to the shared model at all.
- The migration is separable and revertible as a unit.
- Content that does not belong in the shared model has an obvious home, which reduces pressure toward the workbook model.

**Costs**

- **Backporting.** Moving a proven migration-created topic into the governed shared model is manual. There is no promotion path between models.
- **Document portability.** Content built against the migration model is bound to it. Moving those documents to the shared model is an export/import exercise with its own failure modes, not a reassignment.
- **Two surfaces to govern.** Access grants, AI configuration, and validation gates exist in both models.

The exit path is the unresolved part. Adopt this only where the migration output is expected to remain a distinct layer, or where the backport cost is accepted at the outset.
