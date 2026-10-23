

# Enterprise Snowflake Table Comparison
## Requirements, Architecture, and Implementation Handoff

**Version:** 1.0  
**Prepared:** 24 October 2026  
**Implementation location:** Company-managed systems only  
**Source control:** Company-hosted Bitbucket  
**Preferred application hosting:** Snowflake  

## 1. Purpose and authority

Build an enterprise application that accepts two dynamically selected Snowflake tables and automatically discovers overlapping data across matching subsets of their columns and rows.

The application must support arbitrary business domains. It must not require the user to identify whether the data represents customers, products, transactions, patients, or another subject.

This document consolidates the latest decisions. Where earlier discussions differed, the rules here take precedence.

In particular:

- Column-content verification passes when either directional overlap reaches its threshold.
- Table-level column coverage passes only when both directions reach their threshold.
- Exactly 30% passes a threshold configured to 30%.
- All input columns are examined by default.
- Optional size limits apply to rows, rather than introducing a column-count or subset-width cap.
- Discovery is heuristic; exhaustive enumeration of every possible subset is not promised.
- Feedback never bypasses current content verification.

“MUST” identifies a mandatory requirement. “Initial default” identifies a configurable starting value. Company deployment settings are explicitly identified where their values remain unknown.

## 2. Mandatory boundaries

### 2.1 Read-only input data

The application MUST leave selected source and target objects unchanged.

It MUST NOT perform inserts, updates, deletes, merges, truncation, schema changes, or persistent hash-column additions against input tables.

This boundary MUST be enforced through effective Snowflake privileges, including inherited privileges—not merely application conventions.

Comparison transformations operate through query expressions and application-owned working data.

Do not enable change tracking or create other source-side infrastructure as an incidental implementation step.

### 2.2 Application-owned writes

The application requires separate writable storage for:

- Configuration.
- Comparison requests and job state.
- Global feedback and its history.
- Column profiles and mapping evidence.
- Working snapshots or comparison copies.
- Findings, matching groups, member rows, and export artifacts.
- Application migration history.

These writes MUST remain within explicitly configured application-owned objects.

Original values may be copied into the application’s working/report storage for consistent verification and complete exports. This creates additional copies while leaving inputs unchanged. Their retention and access policies are company deployment settings.

### 2.3 Company implementation boundary

Application implementation, company-data access, deployment, and execution MUST occur on company-managed systems.

Company records, credentials, private keys, database exports, and generated business reports MUST NOT be committed to Bitbucket.

Development fixtures MUST use synthetic data unless the organization explicitly authorizes another dataset.

### 2.4 AI input boundary

Runtime AI models may receive:

- Database, schema, table, and column names.
- Available descriptions and comments.
- Types and inferred format characteristics.
- Aggregate statistics.
- Relevant saved equivalence relationships.
- Table-level and relationship context.

They MUST NOT receive:

- Raw source or target records.
- Representative value samples.
- Frequent-value lists containing actual business values.
- Free-text data snippets.
- Raw record exports or screenshots of business records.

Profile serialization MUST use an explicit allowlist. A field named “profile” does not automatically make its contents safe for a model prompt.

Content comparison remains inside Snowflake.

## 3. Business meaning and comparison meaning

The application must distinguish:

1. **Column compatibility:** evidence that columns represent the same concept.
2. **Value agreement:** values satisfy the configured comparison policy.
3. **Subset agreement:** all selected fields qualify together on the same source–target row relationship.
4. **Entity identity:** evidence that records describe the same real-world person, organization, product, or other thing.

The required output measures data overlap under an explicit policy.

Similar column names, shared ID values, or matching names alone do not establish real-world entity identity. For example, independently generated IDs may share literal values across systems.

The application MUST NOT silently present partial value overlap as proof of complete record or entity identity.

## 4. Selected architecture

Use:

- **Streamlit in Snowflake, container runtime:** user interface.
- **Snowflake Cortex AI functions:** semantic candidate generation and interpretation.
- **Snowflake SQL/Snowpark:** profiling, normalization, grouping, hashing, similarity checks, and verification.
- **Snowflake Tasks and stored procedures:** durable background execution.
- **Application-owned Snowflake tables:** configuration, feedback, working data, and results.
- **Company-hosted Bitbucket and corporate CI:** source control and releases.
- **Snowflake CLI:** application deployment.

Streamlit supports owner-rights execution and container runtime. Cortex supports structured model output suitable for validated mapping proposals. [Streamlit runtimes](https://docs.snowflake.com/en/developer-guide/streamlit/app-development/runtime-environments), [owner’s rights](https://docs.snowflake.com/en/developer-guide/streamlit/object-management/owners-rights), [structured AI output](https://docs.snowflake.com/en/user-guide/snowflake-cortex/complete-structured-outputs).

```text
Company user
    |
    v
Streamlit: source/target selection and configuration
    |
    v
Persistent comparison request
    |
    v
Profile columns
    |
    v
AI interpretation + saved equivalence lookup
    |
    v
Current column-content verification
    |
    v
Table-level column-coverage gate
    |
    +---- Fail ----> Report mappings, exclusions, and skip reason
    |
    +---- Pass ----> Discover matching column subsets
                         |
                         v
                    Verify row relationships
                         |
                         v
                    Persist findings and evidence
                         |
                         v
                    Report, paging, exports, optional feedback
```

Cortex Code/CoCo may assist development and Snowflake-specific investigation. The runtime pipeline uses explicit application orchestration; it does not depend on an autonomous coding agent deciding how to execute arbitrary SQL.

Cortex Agents may be considered later for conversational exploration. They are not required for the initial comparison workflow.

## 5. Access model and user interface

### 5.1 Owner-rights execution

Use a dedicated application owner role.

Its effective privileges must provide:

- Read access to approved input objects.
- Required compute and Cortex access.
- Write access only to application-owned objects.

The input catalog reflects the application owner role’s approved readable catalog. It must not be described as the individual viewer’s personal Snowflake access.

Snowflake application grants determine who may use the application.

Record the authenticated viewer independently from the execution role for run attribution, feedback, and audit events.

### 5.2 Input selection

Provide independent source and target selectors:

1. Database.
2. Schema belonging to that database.
3. Table belonging to that schema.

Changing a parent selection clears incompatible child selections.

Catalog enumeration must be lazy or paginated where necessary. Do not load every table across the enterprise into the browser at once.

Use fully qualified, correctly quoted identifiers. Identical table or schema names in different databases must remain distinct.

Both inputs must be queryable from the hosting Snowflake account. Shared/imported databases can participate where permissions and supported capabilities permit. Additional cross-account arrangements are deployment prerequisites.

### 5.3 User workflow

The user:

1. Selects source and target.
2. Reviews or overrides permitted configuration.
3. Starts a comparison.
4. Receives a run identifier and progress.
5. Opens the completed, skipped, partial, failed, or cancelled report.
6. Optionally submits column-equivalence feedback.

The user MUST NOT be required to:

- Select matching columns.
- Supply a business domain.
- Approve mappings during execution.
- Understand the correct mapping before obtaining a report.

Optional feedback is a post-report correction mechanism, not a mandatory runtime approval loop.

## 6. Configuration and threshold rules

Keep these settings separate.

| Configuration | Initial default | Rule |
|---|---:|---|
| Metadata assessment threshold | 70/100 | AI score must be at least this value |
| Column-content overlap threshold | 30% | Either direction must meet this value |
| Table-level column coverage | 30% | Both directions must meet this value |
| Descriptive-text similarity | 90/100 | Eligible descriptive values must meet this value |
| Source row limit | Unset | Unset means all source rows |
| Target row limit | Unset | Unset means all target rows |

Additional rules:

- Thresholds are configurable within 0–100.
- A blank/unset row limit means unlimited input rows.
- Explicit row limits must be positive integers.
- All columns are profiled; there is no initial user-facing column-count or subset-width cap.
- Save an immutable effective configuration snapshot for each run.
- Changing defaults or feedback must not retroactively change historical reports.
- Global defaults and per-run overrides must be distinguishable.
- Operational settings such as compute concurrency and retry budgets are separate from input-data limits.

AI metadata scores and string-similarity scores are different quantities. Neither should be displayed as a calibrated probability of business correctness.

Zero overlap and insufficient evidence must not become accepted mappings simply because a threshold is configured to zero.

## 7. Run scope, snapshots, and profiling

### 7.1 Consistent run inputs

Profiling, row verification, and report exports must refer to consistent run inputs.

Materialize the selected input scope in application-owned working storage, preserving:

- Original values.
- Side: source or target.
- Run-local row identity.
- Original column metadata.
- Capture/snapshot references.

Where a common supported Snowflake snapshot timestamp is available, use it for both inputs. Record actual capture boundaries. If inputs cannot be captured consistently, report the condition instead of claiming a consistent comparison.

A row cap must use a deterministic selection method whose parameters are recorded. Do not imply that a capped run represents full-table overlap.

### 7.2 Column profiles

Collect appropriate statistics for each column, including:

- Type and nullability.
- Missing-value count/rate.
- Distinct count and whether it is exact or estimated.
- Uniqueness characteristics.
- Length and format patterns.
- Numeric/date-like parseability where relevant.
- Value-frequency information.
- Compact distinct-value sketches.
- Descriptions and table context.

Low-cardinality columns can use full distinct-value/count calculations inside Snowflake.

High-cardinality columns must not be collected into giant Python objects or model prompts. Use warehouse-side operations and compact summaries.

### 7.3 Fingerprint distinction

Maintain two different concepts:

- **Column sketch:** summarizes values across a column to assist candidate discovery.
- **Subset row fingerprint:** summarizes selected mapped values together for one row.

A column sketch is not proof that complete row tuples match.

MinHash-style sketches represent distinct-value similarity; frequencies require separate calculations. Approximate statistics must be labeled.

Do not assume that Snowflake maintains a reusable content fingerprint for arbitrary column subsets. Stream row identifiers track identity/change history, rather than proving equal content between independently loaded tables.

## 8. Automatic column mapping

### 8.1 Metadata assessment

Use metadata, aggregate profiles, contextual table information, and saved relationships to identify candidate mappings.

Context may include other attributes and relationships in the table. It MUST NOT depend on physical adjacency or matching column positions.

A source column at position 5 may map to a target column at position 100.

Return structured proposals containing:

- Fully qualified source and target column identities.
- Inferred roles and business concepts.
- Assessment score and explanation.
- Relevant type compatibility.
- Permitted comparison transformations.
- Alternative candidates and unresolved conditions.
- Whether descriptive-text matching is eligible.

Validate model output against the actual catalog and supported transformation policy.

AI output must not directly execute arbitrary SQL.

### 8.2 Ordinary mapping rule

An ordinary mapping must satisfy:

1. Metadata assessment threshold.
2. Current column-content verification.

High metadata confidence does not bypass the second check.

High content overlap does not automatically rescue an opaque or low-confidence metadata mapping under the initial policy.

### 8.3 Opaque and unresolved columns

Exclude opaque or uninterpretable names from automatic row comparison unless applicable saved feedback supplies the missing equivalence evidence.

Report them explicitly.

Keep separate reasons for:

- Opaque metadata.
- Low assessment score.
- Ambiguity.
- Unsupported comparison.
- Insufficient content evidence.
- Low observed content overlap.
- Processing failure.

“Excluded from this run” must not be presented as proof that two columns have different business meanings.

### 8.4 Column-content verification

Use distinct, nonmissing comparison values under the approved normalization policy.

For source value set S and target value set T:

- Source coverage is the proportion of distinct source values having at least one qualifying target value.
- Target coverage is the proportion of distinct target values having at least one qualifying source value.

Accept the content check when either direction reaches the configured threshold and positive overlap exists.

Count each distinct comparison value at most once per direction. Multiple matching counterparts must not inflate coverage.

Frequency distributions remain separate evidence. For example, source frequencies `A=90, B=10` and target frequencies `A=10, B=90` have complete distinct-value overlap despite different frequencies.

Missing values do not establish column-content overlap. Empty/all-missing populations require an insufficient-evidence outcome.

Use the same permitted numeric and descriptive-text policies that apply during row comparison.

Report exact normalized overlap separately from additional overlap found through descriptive-text similarity. Fuzzy candidate pruning can miss relationships; disclose this limitation.

Approximate sketches may rank candidates. They must not be presented as exact content-verification counts.

## 9. Table-level column-coverage gate

After accepting column mappings, calculate:

**Source column coverage = distinct accepted source columns ÷ all source columns**

**Target column coverage = distinct accepted target columns ÷ all target columns**

Include opaque, unsupported, and unmatched columns in the total column counts.

Count distinct mapped columns, not mapping edges. Several mappings involving the same column must not inflate the numerator.

Proceed to subset row comparison only when both percentages meet the configured threshold.

At the default 30%:

| Source coverage | Target coverage | Outcome |
|---:|---:|---|
| 40% | 20% | Skip row comparison |
| 20% | 40% | Skip row comparison |
| 30% | 30% | Proceed |
| Below 30% | Any percentage | Skip |
| Any percentage | Below 30% | Skip |

When skipped:

- Still generate the profiling and mapping report.
- Explain the failed percentage and effective threshold.
- Mark row comparison as not performed.
- Do not show zero row matches as if comparison had run.

The initial example of ten matching columns among hundreds falls below this default gate. Lowering the configurable gate can enable that sparse-overlap case.

If there are no accepted mappings, do not compare empty tuples even when the configured threshold is zero.

## 10. Subset discovery and row verification

### 10.1 Discovery contract

Use all selected input rows and columns by default.

Automatically discover useful mapped subsets, such as:

- `A, B, C`.
- `B, C, D`.
- Other accepted combinations with different matching populations.

Single-column findings may be reported, but must clearly identify their subset size and information strength.

The search uses candidate retrieval, informative shared values, composite anchors, and recurring agreement patterns. Compress redundant patterns while retaining findings with meaningfully different support.

The application does not promise to enumerate every possible subset. Its report must identify:

- Heuristic discovery mode.
- Search-policy version.
- Candidate mappings evaluated.
- Patterns verified.
- Pruning or approximation methods.
- Incomplete stages and limitations.

There must be no silent input truncation or silent report truncation.

### 10.2 Joint comparison

A subset qualifies only when all selected fields agree under policy on the same source–target row relationship.

For example:

| Source A | Source B | Target X | Target Y |
|---|---|---|---|
| 1 | Red | 1 | Blue |
| 2 | Blue | 2 | Red |

Individual column value sets overlap, but the two-column tuples do not match.

Do not combine values from different target rows to manufacture agreement.

Row order is irrelevant. Column order is determined by the saved mapping alignment, independently of physical position.

### 10.3 Subset fingerprints

Compute fingerprints dynamically for the selected mapped subset.

- `A, B, C` and `B, C, D` require different subset fingerprints.
- Use consistent field ordering and representations on both sides.
- Preserve boundaries between values.
- Preserve missing/empty distinctions.
- Verify actual normalized values after a hash match.
- Do not use whole-row fingerprints as the primary method for discovering a small shared subset.

Snowflake hash behavior differs across some physical types, including numbers and strings, so compatible representation must precede hashing. [Snowflake HASH](https://docs.snowflake.com/en/sql-reference/functions/hash).

### 10.4 Matching policies

**Exact values**

Values agree without required numeric normalization, descriptive-text correction, or shared missingness.

**Numeric normalization**

Accept compatible numeric/string representations, including leading zeros, such as:

- Numeric `123` and text `'123'`.
- Numeric `123` and text `'00123'`.
- Equivalent numeric-text representations supported without precision loss.

Preserve original values and record the transformation.

Do not introduce rounding or numeric tolerances. Do not convert failed parses into matching nulls.

**Missing values**

Two missing values count as equal within an accepted subset, and the result receives a missing-data flag.

A missing value and a populated value do not match. Empty strings remain distinct from missing values unless a future explicit policy changes that rule.

**Descriptive text**

Allow descriptive-text variations and possible spelling errors using the configured score, initially 90/100.

Use case and surrounding-space normalization for eligible descriptive text, followed by deterministic similarity evaluation where necessary.

Identifiers and codes are excluded from typo matching. The agreed numeric normalization still applies to compatible numeric representations.

Do not use the LLM to judge millions of individual value pairs.

**Other transformations**

Do not invent unit conversion, code lookup, concatenation, splitting, translation, or date-format rules without an explicit supported policy.

Potential relationships requiring unsupported transformations must be reported rather than silently guessed.

### 10.5 Fuzzy relationship integrity

Text similarity is not transitive.

If A qualifies against B, and B qualifies against C, this does not automatically establish A–C agreement.

Preserve directly verified group-to-group relationships in `MATCH_LINKS`.

Every selected field must satisfy its own comparison predicate. A high score on one field must not compensate for failure on another.

## 11. Counts and report behavior

### 11.1 Directional metrics

For each finding, report independently:

- Source rows covered.
- Target rows covered.
- Source coverage percentage.
- Target coverage percentage.
- Distinct matching tuples/groups.
- Repeated occurrences within each input.
- Uncovered rows or rows with no match found.
- Comparison categories and evidence.

Use the analyzed row scope as the denominator. Clearly distinguish limited runs from full-table runs.

Use “no match found under this search and policy” where exhaustive uniqueness has not been established.

### 11.2 Repeated occurrences

Do not count raw join rows as duplicated records.

For an exact/canonical tuple occurring s times in source and t times in target:

- Covered source rows: s when t is positive.
- Covered target rows: t when s is positive.
- Reconciled occurrences: minimum of s and t.
- Excess source occurrences: maximum of s − t and zero.
- Excess target occurrences: maximum of t − s and zero.

These are different metrics.

For fuzzy relationships involving several counterpart groups, do not sum per-edge counts as a one-to-one occurrence reconciliation. Report distinct covered members and verified relationships.

### 11.3 Four result views

Provide separate views for:

1. Exact matches.
2. Numeric-normalized matches.
3. Missing-data matches.
4. Descriptive-text variations/possible typos counted as matches.

A relationship may carry several flags—for example, numeric normalization and shared missingness.

Preserve all applicable flags. Explain overlapping views.

Overall source/target coverage must use distinct row identities across categories and subsets. Category totals and finding totals must not be blindly added.

“Full match under policy” must not be presented as 100% certainty of business identity.

### 11.4 Complete evidence access

Provide:

- Group summaries.
- Every matching source and target member row through deterministic paging.
- Full authorized matching-row exports.
- Original values alongside relevant normalized values and scores.

One thousand matching occurrences on each side should expose both member sets without expanding one million redundant displayed pairings.

Pagination is a presentation mechanism, not an undisclosed data limit.

### 11.5 Required report sections

Every report includes:

- Run identity and authenticated requester.
- Complete source/target identities.
- Input capture and analyzed scope.
- Effective configuration and model/policy versions.
- Profiling results.
- Accepted and excluded column mappings.
- Opaque/unresolved/unsupported columns.
- Saved-feedback provenance and current validation outcome.
- Table-level gate calculations.
- Subset findings and their mapped fields.
- Four matching views and directional quantities.
- Search limitations and exact/approximate distinctions.
- Skips, errors, partial stages, and elapsed times.
- Paging/export access and retention status.

## 12. Global feedback learning

### 12.1 Scope and identity

A saved equivalence must identify both columns through:

- Snowflake account.
- Database.
- Schema.
- Table.
- Column.

Preserve quoted identifier case and relevant object/schema signatures. Similar names in other locations must not inherit the correction.

Feedback is global across authorized users of that application environment.

Production, test, and development registries remain isolated by their environment storage.

### 12.2 Attribution and history

Record:

- Feedback identifier.
- Complete source and target identities.
- Reporting user ID and username.
- Submission time.
- Equivalence status.
- Revision/supersession/revocation history.
- Originating comparison run.
- Applicable schema signatures.

Obtain the viewer’s identity from authenticated session information. Do not use a freely entered username or mistake the execution owner for the reporting person.

### 12.3 Reuse behavior

Saved equivalence supplies first-stage semantic evidence on later runs, including reversed source/target selection.

It does not fabricate an AI confidence score or establish duplicate values by itself.

Fresh content verification remains mandatory.

If current overlap is zero or below the configured threshold:

- Exclude the mapping from the current row comparison.
- Preserve the saved suggestion and its history.
- Show who originally suggested it.
- Show current measured overlap and the rejection reason.

A content failure affects the current run; it must not silently erase the historical correction.

Schema/object changes require re-evaluation and explicit stale-evidence handling.

New feedback applies to subsequent runs. Historical and already-running configurations remain stable.

Learning here means retrieval and reuse of attributed corrections. Model fine-tuning is not required.

## 13. Snowflake object inventory

Use one application database per environment.

Example names:

- `TABLE_COMPARISON_DEV`
- `TABLE_COMPARISON_TEST`
- `TABLE_COMPARISON_PROD`

These are deployment naming examples, not existing objects.

### 13.1 Schemas and tables

| Schema | Table | Responsibility |
|---|---|---|
| CONTROL | CONFIGURATION | Versioned application defaults and policies |
| CONTROL | COMPARISON_RUNS | Requests, queue state, progress, effective configuration, versions, and outcomes |
| CONTROL | COLUMN_EQUIVALENCE | Active global equivalence relationships |
| CONTROL | FEEDBACK_HISTORY | Attributed feedback revisions and revocations |
| CONTROL | SCHEMA_MIGRATIONS | Applied application-schema migrations |
| WORK | RUN_INPUT_ROWS | Scoped original input copies and stable run-local row IDs |
| WORK | COLUMN_PROFILES | Per-run profiles, sketches, and model-safe profile projections |
| WORK | COLUMN_CANDIDATES | AI/feedback proposals, current content evidence, acceptance and exclusion reasons |
| REPORTS | SUBSET_FINDINGS | Verified patterns, mappings, metrics, and verification status |
| REPORTS | MATCH_GROUPS | Repeated tuple/value groups and their occurrence counts |
| REPORTS | MATCH_LINKS | Directly verified source-group/target-group relationships and match flags |
| REPORTS | MATCH_MEMBERS | Group membership linked to run-local source/target rows |

`APP` contains runtime objects rather than ordinary data tables:

- Streamlit application.
- Stored procedures/functions.
- Tasks.
- Deployment/export stages as appropriate.

### 13.2 Counts

Per environment:

- One application database.
- Four schemas.
- Twelve shared application tables.

Across development, test, and production:

- Three application databases.
- Twelve schemas.
- Thirty-six application tables.

This is the selected starting design, not a Snowflake platform minimum.

Do not create a new database or a new table for every user, subset, or comparison. Store run-scoped records using identifiers.

Per-run mapping evidence belongs in `WORK.COLUMN_CANDIDATES`. Do not introduce an unplanned extra `MAPPING_EVIDENCE` table while still claiming a twelve-table inventory.

### 13.3 Retention

Working inputs and member references must remain available for the lifetime of their downloadable detailed reports.

Do not purge working rows while a retained report still depends on them.

Feedback/history retention and report/working-data retention are separate company settings.

Expired reports must be identified as expired. Do not silently reconstruct an old report from changed live inputs.

## 14. Enterprise code organization and interfaces

Use a modular Python application, with Streamlit separated from comparison logic.

```text
src/table_comparison/
  ui/
  application/
    catalog/
    profiling/
    mapping/
    content_verification/
    subset_discovery/
    comparison/
    feedback/
    reporting/
  domain/
    models/
    policies/
    comparators/
  infrastructure/
    snowflake/
    cortex/
    repositories/
  workers/

sql/
  migrations/
  procedures/
  tasks/

config/
tests/
  unit/
  integration/
  acceptance/
  performance/

deployment/
docs/
```

Include:

- Typed domain contracts.
- Dependency locking.
- Centralized configuration and validation.
- Versioned prompts, model selection, normalization, and search policies.
- Structured errors and logs.
- Automated CI checks.
- Environment isolation.
- Architecture, deployment, operations, and troubleshooting documentation.
- A requirement-to-test traceability matrix.

Minimum operations:

- List databases/schemas/tables.
- Start a comparison.
- Read status/progress.
- Cancel a run.
- Retrieve report summaries and findings.
- Page matching members.
- Export complete matching data.
- Save/revise/revoke equivalence feedback.
- Read feedback history.

A comparison request contains source/target identities and permitted configuration overrides.

Responses identify the run, execution state, analyzed scope, findings, verification method, and limitations.

Validate identifiers against the catalog, quote them correctly, and use bound values where applicable. Query execution must come from validated application templates and supported operations, rather than unrestricted model-generated SQL.

## 15. Durable execution and failure behavior

Use persistent background jobs independent of browser sessions.

At minimum, distinguish:

- Queued.
- Running.
- Completed under the selected discovery policy.
- Skipped because of insufficient column coverage.
- Partial.
- Failed.
- Cancelled.

Record stage progress, timestamps, query identifiers, checkpoints, policy versions, commit/release identity, and diagnostics.

Requirements:

- Browser closure must not end the comparison.
- Retried stages must not duplicate findings or feedback.
- Transient failures use bounded, configurable retries.
- Cancellation must stop pending work and request cancellation of active queries.
- Preserve completed evidence when a run becomes partial.
- Processing errors must not be reported as zero duplicates.
- Unavailable Cortex processing must not silently become “columns do not match.”
- Stage completion must not be labeled exhaustive subset discovery.

Cache reusable metadata interpretation only when schema/model/prompt identities remain applicable.

Recompute current content evidence for each new run. Do not reuse old overlap percentages merely because column names are unchanged.

Keep million-row operations in Snowflake. Do not collect full input tables into Streamlit/Python memory.

## 16. Bitbucket, deployment, and execution

### 16.1 Source control

The repository is company-hosted Bitbucket.

Commit:

- Application source.
- Tests and synthetic fixtures.
- Prompts and policy definitions.
- Nonsecret configuration templates.
- Dependency manifests/lockfiles.
- SQL migrations and runtime definitions.
- Deployment manifests and corporate CI configuration.
- Requirements and operations documentation.

Do not assume Bitbucket Cloud Pipelines. Use the company’s approved CI system, such as Jenkins or Bamboo.

### 16.2 Deployment flow

```text
Company-hosted Bitbucket
    -> corporate CI runner
    -> automated checks
    -> versioned release package
    -> Snowflake CLI
    -> development/test/production Snowflake application objects
```

Snowflake CLI supports deployment from CI environments. [Snowflake CI/CD](https://docs.snowflake.com/en/developer-guide/snowflake-cli/cicd/integrate-ci-cd).

The runner requires approved access to both internal Bitbucket and Snowflake.

Use a dedicated deployment service identity. Credentials are injected through approved secret management and are not stored in the repository.

Deploy the same tested release through environments, with environment-specific configuration.

### 16.3 Runtime location

After deployment:

- Streamlit runs inside Snowflake.
- SQL/Snowpark comparison work runs on Snowflake compute.
- Cortex executes runtime AI requests.
- Tasks/procedures execute background work.
- Feedback and reports persist in Snowflake.

The corporate CI runner is not the application web server.

### 16.4 Deployment safety

Application releases must:

- Apply versioned application-schema migrations.
- Preserve feedback and historical results.
- Update/publish the Streamlit application and worker definitions.
- Apply required application grants.
- Record release identity.
- Support compatible rollback of code.

Do not recreate or empty the application database on every deployment.

Do not roll back a code release by deleting user feedback.

All migration targets must be fully qualified and validated against the configured application database.

## 17. Cost, latency, and operational controls

Measure independently:

- Cold and warm application startup.
- Catalog navigation.
- Profiling.
- AI assessment.
- Column-content verification.
- Subset discovery.
- Exact and fuzzy verification.
- Result paging and export.
- Concurrent runs.

Cost components include application hosting, warehouse execution, Cortex usage, working/report storage, and export handling.

Do not claim a fixed completion time or a universal cheaper hosting choice before representative measurement.

Keep AI calls at metadata/candidate level. Do not call an LLM for every row or row pair.

Resource concurrency, batching, retry settings, retention, and cost controls are operational settings. Any resulting partial execution must be visible.

External hosting is technically possible through Snowflake APIs/connectors, but is not the initial selected deployment. It must not become an unapproved fallback.

A future custom UI can also evaluate Snowflake App Runtime without changing the core comparison contracts.

## 18. Company-specific requirements and prerequisites

The implementation must represent these as explicit deployment settings or blockers rather than inventing values:

- Actual Snowflake account, cloud, and region.
- Available Streamlit/container capabilities.
- Approved Cortex models and inference-routing policy.
- Owner, viewer, administrator, and deployment identities/roles.
- Approved input databases and effective privileges.
- Corporate CI product and runner.
- Warehouse and compute-pool sizing.
- Secret-management integration.
- Working/report/feedback retention.
- Export access controls.
- Production performance and operating-cost targets.
- Applicable company security and regulatory controls.

No specific pharmaceutical, privacy, or regulatory certification was established during requirements gathering.

Implement the technical audit and access requirements here, but do not claim regulatory compliance without the organization supplying the applicable framework and validation requirements.

Do not change account-wide inference-routing or security settings by copying defaults from examples.

Missing deployment settings may block integration or production activation while synthetic implementation/testing continues.

## 19. Acceptance tests

The implementation must provide automated coverage for these scenarios.

| ID | Scenario | Required outcome |
|---|---|---|
| AT-01 | Same concept, different names and positions | Mapping can succeed independently of ordinal position |
| AT-02 | Numeric column versus numeric text | Agreed equivalent values match |
| AT-03 | `'00123'` versus `123` | Match appears in numeric-normalized results |
| AT-04 | Numeric parse failure | Does not become a matching null |
| AT-05 | Opaque name without feedback | Excluded and reported |
| AT-06 | Applicable saved equivalence | Supplies semantic evidence; fresh content check runs |
| AT-07 | Saved equivalence with zero overlap | Excluded for that run; author and history shown |
| AT-08 | Low AI score, high content overlap, no feedback | Ordinary mapping does not bypass metadata gate |
| AT-09 | Distinct values match but frequencies differ | Distinct overlap and frequencies are reported separately |
| AT-10 | Content coverage passes only one direction | Column-content gate can pass |
| AT-11 | Table coverage 40%/20% | Row comparison skipped |
| AT-12 | Table coverage 30%/30% | Row comparison proceeds |
| AT-13 | One table below 30% | Row comparison skipped |
| AT-14 | Threshold lowered for sparse overlap | Sparse accepted subsets become eligible |
| AT-15 | Crossed `(1, Red)`/`(1, Blue)` example | Selected two-column tuples do not falsely match |
| AT-16 | `ABC` and `BCD` patterns | Independently verified findings and populations |
| AT-17 | Repeated tuples on both sides | Counts avoid Cartesian multiplication |
| AT-18 | Shared missing values | Accepted under row policy and flagged |
| AT-19 | Missing versus populated value | Does not qualify for that field |
| AT-20 | Descriptive capitalization/typo variation | Qualifies according to score and appears in fourth view |
| AT-21 | Similar identifiers/codes | No typo matching |
| AT-22 | Score immediately below/at/above 90 | Correct inclusive threshold behavior |
| AT-23 | A–B and B–C fuzzy links only | No invented A–C match |
| AT-24 | Hash candidates with unequal actual tuples | Rejected after value verification |
| AT-25 | Several categories/subsets cover the same row | Overall coverage counts the row once |
| AT-26 | Empty/all-missing content | Explicit outcome without division by zero |
| AT-27 | Optional row limit | Deterministic scope and labeled scoped metrics |
| AT-28 | Same names in different databases/schemas | Feedback does not cross unintended identities |
| AT-29 | Reversed source/target | Applicable equivalence reused; directional counts remain correct |
| AT-30 | Schema/object changes | Saved evidence re-evaluated and stale state reported |
| AT-31 | Browser closes | Background job remains durable |
| AT-32 | Retry/cancellation/failure | Correct state, no duplicated output, no false zero-match report |
| AT-33 | Multiple viewers | Correct authenticated attribution and approved access |
| AT-34 | Full export across many pages | All matching members are retrievable without truncation |
| AT-35 | Application deployment | Existing global feedback remains intact |
| AT-36 | Million-row, wide-table fixtures | No full-table collection into Python; measured resource behavior |

Verify source protection through grant inspection and controlled synthetic permission fixtures. Do not test it by attempting destructive operations against real company input tables.

Measure discovery recall separately from verification correctness using planted overlaps of different sizes, rarity, and column positions.

## 20. Implementation sequence and completion criteria

Implement in these stages:

1. Enterprise package structure, configuration, domain contracts, and synthetic tests.
2. Twelve-table application schema and versioned migrations.
3. Owner-rights catalog and dynamic selectors.
4. Profiling, AI adapter, saved-equivalence lookup, and column-content verification.
5. Table-level gate and skipped-run reporting.
6. Exact/numeric/missing subset comparison and counts.
7. Descriptive-text similarity and direct fuzzy relationships.
8. Complete report views, paging, exports, and feedback history.
9. Durable job execution, cancellation, recovery, observability, and CI deployment.
10. Company-environment integration, performance measurement, and release validation.

Completion requires:

- Requirement-to-test traceability.
- Passing relevant automated checks.
- Explicitly documented discovery limitations.
- Verified source-read-only privileges.
- Persistent, attributed feedback.
- Complete accessible matching-row evidence.
- Working deployment artifacts for the selected corporate CI system.
- No placeholder implementation presented as production functionality.
- A list of company prerequisites still blocking activation, if any.

## 21. Coding chat and model guidance

Use a company-approved coding agent that can read/edit the repository, run tests, and inspect relevant documentation.

Recommended division:

- **Primary implementation:** VS Code Agent mode with Claude Opus 5.5 or GPT-5.6 Sol.
- **Snowflake-specific inspection and validation:** CoCo through the Snowflake extension.
- **Routine bounded changes:** Sonnet 5.5 or GPT-5.6 Terra.
- **Small scoped tasks:** GPT-5.6 Luna, with normal review and test requirements.

This is a workflow recommendation, not a benchmark guarantee.

CoCo is also a valid primary implementation environment if it is the organization’s approved option. Its actual model list is authoritative; check the model picker or the CLI’s `/model` command rather than assuming Sonnet 4.5 is the only choice. [CoCo in VS Code](https://docs.snowflake.com/en/user-guide/cortex-code/cortex-code-in-your-editor), [CoCo model selection](https://docs.snowflake.com/en/user-guide/cortex-code/cortex-code-cli).

Development-agent model selection is separate from the application’s configured Cortex runtime models.

Use one primary writer per working branch. Other chats may review the committed requirements and code, but should not independently make conflicting changes to the same working copy.

## 22. Starter prompt for the company implementation chat

> Read `docs/requirements/snowflake-table-comparison-requirements.md` completely and treat it as the authoritative functional and architectural specification.
>
> Implement this project only within this company-managed repository and environment. First inspect the repository and available company tooling. Identify required deployment settings that have not been supplied; do not invent company account names, roles, regions, CI products, credentials, retention rules, or regulatory requirements.
>
> Build the enterprise application in the implementation sequence specified in the document. Maintain a requirement-to-test checklist and continue through implementation and relevant verification.
>
> Preserve these invariants: read-only source/target objects; writes restricted to the application database; metadata/profiles only to AI; automatic two-stage mapping; separate content and table-coverage gates; subset-only row comparison; correctly attributed global feedback; fresh content verification; four matching views; deduplicated totals; and explicit discovery limitations.
>
> Use synthetic fixtures for development and tests. Do not query raw company records into AI prompts or modify company input objects. Validate source permissions through metadata/grant inspection and controlled fixtures.
>
> Produce deployable Streamlit/Cortex/SQL-Snowpark components, versioned application migrations, corporate CI deployment assets, tests, and operations documentation. Preserve feedback across releases.
>
> If integration is blocked by missing company configuration, finish independent implementation and synthetic verification, then report the precise blocker. Do not replace required functionality with a success-returning placeholder or claim unexecuted checks passed.
