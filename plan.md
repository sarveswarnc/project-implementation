

# Core Table Comparison Engine — Technical Specification

**Suggested document name:** `CORE_COMPARISON_ENGINE_SPEC.md`  
**Version:** 1.0  
**Date:** 25 October 2026  
**Purpose:** Replace the comparison engine inside an existing enterprise product.

This document consolidates the comparison rules discussed in the planning conversation. It covers inputs, configuration, profiling, column mapping, content verification, subset discovery, row comparison, counting, feedback and engine outputs.

Hosting, deployment, UI construction, CI/CD, database provisioning and infrastructure architecture are outside this document. The implementing AI must integrate with the company product’s existing services.

**Important clarification:** The engine does not ignore data types. Different physical types are allowed when a defined, lossless comparison is possible.

`MUST` and `MUST NOT` indicate mandatory requirements.

---

## 1. Objective and terminology

Given two dynamically selected Snowflake tables, the engine must:

1. Inspect every input column.
2. Identify column pairs that may represent the same concept.
3. Independently verify their current content overlap.
4. Determine whether enough columns qualify to proceed.
5. Discover different compatible combinations of accepted mappings.
6. Compare rows jointly within each discovered combination.
7. Report directional overlap, repeated occurrences, comparison methods and limitations.
8. Reuse applicable feedback without bypassing current verification.

The inputs may contain any business domain, millions of rows and hundreds of columns. The engine must not depend on predefined customer, product, patient or transaction schemas.

### 1.1 Distinct concepts

| Term | Meaning |
|---|---|
| Semantic equivalence | Evidence that two columns represent the same concept |
| Column content overlap | Overlap between the current values of two columns |
| Accepted mapping | A column pair eligible for comparison under the current policies |
| Projection or subset | A compatible combination of accepted column mappings |
| Row relationship | A particular source occurrence and target occurrence compared together |
| Covered occurrence | An input row occurrence with at least one qualifying counterpart for a finding |
| Reconciled occurrence | An occurrence allocated against a corresponding occurrence for multiplicity comparison |
| Finding | Results for one projection, including its membership and evidence |

Semantic equivalence does not prove that two records describe the same real-world entity. Shared identifier literals can also occur in unrelated identifier systems.

The engine reports verified overlap under its stated policies. It must not silently promote that evidence into universal business identity.

---

## 2. Mandatory invariants

The engine must preserve all of these rules:

1. **Input immutability:** Never modify either input table’s data or schema.
2. **Automatic column discovery:** Users are not required to select corresponding columns.
3. **No mandatory review loop:** Runtime execution must not wait for a business user to approve mappings.
4. **Position independence:** Column ordinal positions and physical row positions must not determine alignment.
5. **Separate verification stages:** Metadata assessment and content verification cannot compensate for each other through an averaged score.
6. **Joint row verification:** All fields in a selected projection must qualify against the same counterpart row.
7. **Subset-specific comparison:** Fingerprints and results apply to the selected projection, not automatically to the complete input row.
8. **Safe normalization:** Preserve original values and prohibit lossy transformations that manufacture equality.
9. **Hash verification:** A matching hash is a candidate, not proof of equality.
10. **Directional accounting:** Source and target coverage must be calculated independently.
11. **Occurrence preservation:** Repeated rows remain separate input occurrences.
12. **No Cartesian inflation:** Join-pair counts must not become covered-row counts.
13. **No transitive fuzzy equality:** Direct fuzzy links must not create unverified relationships.
14. **No hidden input truncation:** Default execution considers all rows and all columns.
15. **Honest completeness:** Exact verification of a discovered pattern does not establish exhaustive discovery.
16. **Fresh feedback verification:** Saved corrections never bypass current content verification.
17. **AI data boundary:** Raw values and record samples must not enter model prompts or model tracing payloads.
18. **No fabricated success:** Failed, skipped and incomplete stages must remain distinguishable from completed comparisons with zero matches.

These are requirements, not optional feature switches.

---

## 3. Configuration contract

The names below are proposed handoff names. They do not claim that corresponding configuration files already exist.

The company product may use its existing configuration mechanism, provided these meanings and defaults are preserved.

### 3.1 Agreed adjustable settings

| Configuration key | Initial value | Meaning |
|---|---:|---|
| `metadata_assessment_min_score` | `70` | Minimum AI metadata assessment on a 0–100 scale |
| `content_overlap_min_pct` | `30` | Minimum current distinct-value overlap in either direction |
| `table_column_coverage_min_pct` | `30` | Minimum mapped-column coverage required independently on both tables |
| `descriptive_text_similarity_min_score` | `90` | Minimum descriptive-text similarity score |
| `source_row_limit` | `null` | Optional limit; null means all source rows |
| `target_row_limit` | `null` | Optional limit; null means all target rows |

Example representation:

```yaml
comparison:
  metadata_assessment_min_score: 70
  content_overlap_min_pct: 30
  table_column_coverage_min_pct: 30
  descriptive_text_similarity_min_score: 90

  source_row_limit: null
  target_row_limit: null
```

### 3.2 Configuration validation

- Scores and percentages must be finite numeric values on their declared 0–100 scales.
- A setting must not silently disable a mandatory verification stage.
- Row limits, when provided, must be positive integers.
- Zero, negative, fractional and malformed row limits must be rejected.
- Threshold comparisons are inclusive: a value exactly at the threshold passes.
- Apply thresholds to underlying values before presentation rounding.
- Reject invalid configuration before comparison begins.
- Freeze effective configuration for the run.
- Later changes must not alter an in-flight run or historical results.

For example, a measured overlap below 30% must not pass merely because its displayed value rounds to `30.00%`.

### 3.3 Fixed policies

The initial engine contract includes:

| Policy | Required behaviour |
|---|---|
| Ordinary mapping | Metadata evidence followed by current content verification |
| Content acceptance | Either directional percentage may satisfy the threshold |
| Table acceptance | Both directional column-coverage percentages must satisfy the threshold |
| Data scope | All rows unless an explicit side-specific limit applies |
| Column scope | Inventory and profile all columns |
| Minimum supported finding width | Single-column findings are permitted |
| Maximum column count | No agreed fixed cap |
| Maximum subset width | No agreed fixed cap |
| Numeric comparison | Lossless normalization, including agreed leading-zero equivalence |
| Missing values | Excluded from column-content evidence; shared nulls may qualify within accepted row projections |
| Descriptive text | Permitted case/space normalization and configured similarity comparison |
| Identifiers and codes | Protected from descriptive typo matching |
| Feedback | Optional, reusable, attributed and freshly content-verified |
| Search | Heuristic discovery with disclosed limitations |
| Final equality | Verify actual values after hash-based retrieval |

Do not introduce an undisclosed minimum subset width, minimum matching-population threshold, column limit or default sample size.

Constant columns and low-information findings must be identified as weak evidence. No numerical entropy or uniqueness threshold was agreed for automatically excluding them.

### 3.4 Required versioned integration settings

The existing product must supply or register:

- Approved runtime AI model identity.
- Metadata prompt and structured-output schema versions.
- Metadata/embedding retrieval policy, if used.
- Supported native comparators.
- Normalization and numeric-parser policies.
- Descriptive-text preprocessing and comparator versions.
- Subset-search and row-candidate retrieval policies.
- Fingerprint algorithm and tuple-serialization version.
- Applicable feedback revision.
- Any execution budgets, timeouts or search parameters.

No numerical defaults were agreed for sketch size, candidate-retrieval breadth, batching, runtime budgets or pruning parameters.

Such values must come from a documented, versioned product policy. They must not act as hidden business rules or silently truncate the input.

---

## 4. Core input and output interfaces

These are logical contracts. Adapt them to the product’s existing interfaces rather than rebuilding its infrastructure.

### 4.1 Engine request

The request must identify:

- Source table.
- Target table.
- Effective configuration.
- Authenticated initiating user.
- Stable input scope or a mechanism to capture it.
- Approved AI/comparator/search policies.
- Feedback repository access.
- Result/evidence persistence access.
- Cancellation and execution-budget context, where supported.

Table references must preserve account context, database, schema, table identity and quoted-identifier case.

A column name alone is not a globally unique identity.

### 4.2 Logical engine dependencies

The engine needs capabilities to:

1. Resolve and read approved Snowflake objects.
2. Capture or reference stable input versions.
3. Execute profiling and comparison operations inside Snowflake.
4. Obtain structured metadata assessments.
5. Read applicable feedback.
6. Persist findings, memberships and diagnostics.

The product decides how these capabilities are implemented.

### 4.3 Engine result

The result must contain:

- Run identity and outcome.
- Source/target identities and analysed scope.
- Effective configuration and policy versions.
- Column inventory and profiles.
- Mapping candidates, acceptance decisions and exclusions.
- Table-gate measurements and outcome.
- Discovered projections.
- Verified groups and direct relationships.
- Source/target membership and coverage metrics.
- Comparison-method evidence.
- Feedback provenance.
- Search-completeness diagnostics.
- Errors, warnings and completed-stage information.

---

## 5. Ordered execution flow

### Step 1 — Resolve and validate inputs

The engine must:

1. Resolve both fully qualified table references.
2. Verify that the existing product permits reading them.
3. Confirm they are accessible.
4. Read their current object and schema identities.
5. Validate effective configuration.
6. Record the initiating user.

The engine must not assume that matching table names, schema names or column names imply shared meaning.

An access or resolution failure is an explicit failure, not an empty comparison result.

### Step 2 — Freeze the run context and input scope

Create an immutable run context containing:

- Table identities.
- Schema signatures.
- Configuration.
- Model and policy versions.
- Feedback revision used.
- Capture/version boundaries.
- Actual analysed row counts.

Every stage must operate on the same captured scope.

The engine must not profile one table version and compare rows from a later version without making that change explicit.

Captured rows must have stable occurrence references within the run. Identical values must not collapse separate occurrences.

If a row limit applies:

- Use the product’s documented, reproducible selection policy.
- Record the selection policy and actual selected count.
- Label the scope as limited.
- Use analysed-row denominators.

A successfully completed limited-scope run is different from a run interrupted before completing its selected scope.

### Step 3 — Extract every column’s metadata

For each source and target column, extract:

- Fully qualified identity.
- Exact name and quoted-case information.
- Physical data type and relevant precision/scale.
- Available description/comments.
- Nullability and other available metadata.
- Original ordinal position for lineage only.
- Object/schema signature information.

Column ordinal position must never be used as a required matching condition.

Unsupported and opaque columns must remain in the inventory.

### Step 4 — Profile every column

Calculate, where supported:

- Total analysed rows.
- Null and non-null counts.
- Null percentage.
- Distinct non-null count.
- Uniqueness.
- Repeated-value frequencies.
- Value lengths.
- Format/pattern distributions.
- Numeric parseability.
- Other applicable parseability.
- Aggregate distributions.
- Compact value sketches.
- Information-strength indicators.

Mark estimated statistics as estimates.

For unsupported types, retain metadata and explicitly identify unavailable profiling operations. Do not silently drop the column.

Categorical distinct values and frequencies may be processed inside Snowflake. They must not be sent as actual-value lists to AI.

High-cardinality columns must not cause millions of values to be collected into application memory.

### Step 5 — Build column-level discovery evidence

Use profiles, metadata and compact sketches to identify promising column pairs.

Maintain the distinction between:

- Set overlap.
- Frequency similarity.
- Semantic evidence.
- Type compatibility.

None is a substitute for all the others.

MinHash-style sketches can assist set-overlap discovery, but they do not preserve occurrence frequencies. A Jaccard estimate must not be substituted directly for the directional overlap gate defined later. [Snowflake MINHASH documentation](https://docs.snowflake.com/en/sql-reference/functions/minhash)

### Step 6 — Retrieve applicable feedback

Look up corrections using the exact pair of column identities and their applicability information.

Applicable feedback may supply semantic evidence for that pair.

It must not:

- Manufacture a model assessment of 100.
- Bypass content verification.
- Override an unsupported or unsafe comparator.
- Apply merely because another table has similarly named columns.

The feedback rules are specified in Section 11.

### Step 7 — Generate and assess semantic mapping candidates

Use approved native AI capabilities for metadata interpretation.

The model may receive:

- Column names.
- Approved comments/descriptions.
- Physical types.
- Aggregate profile statistics.
- Format summaries.
- Relevant table/schema context.
- Applicable semantic feedback.

The model must not receive:

- Raw rows.
- Sample values.
- Actual frequent-value lists.
- Free-text cell contents.
- Raw-value payloads copied into prompts, traces or model logs.

Candidate retrieval may use embeddings or metadata search. Retrieval scores must remain distinct from:

- The metadata assessment score.
- Content overlap.
- Row coverage.
- Descriptive-text similarity.

The structured assessment should identify:

- Proposed source and target column.
- Whether semantic evidence is sufficient.
- Assessment score.
- Explanation.
- Inferred comparison role.
- Potential comparator/normalization compatibility.
- Ambiguity or contradiction.

The engine must validate the response against the actual catalog and allowed policies.

Names, descriptions and model output are data, not executable instructions. The model must not supply unrestricted executable SQL.

### Step 8 — Apply the semantic gate

Ordinary semantic eligibility requires:

```text
Sufficient interpretable semantic evidence
AND
metadata_assessment_score >= configured minimum
```

A valid saved equivalence may provide semantic evidence instead.

The engine must not average a failed metadata assessment with high content overlap to produce acceptance.

Opaque columns without qualifying feedback remain excluded, even if their values resemble another column.

An unresolved or contradictory mapping must be reported with its reason. No user approval is required to continue with other eligible mappings.

### Step 9 — Select permitted comparison policies

For each semantically eligible candidate:

1. Determine whether a supported common comparison domain exists.
2. Select permitted native comparison and normalization policies.
3. Record their versions.
4. Assess conversion failures and limitations.

**Different physical types are not an automatic rejection.**

However, semantic confidence does not make an unsafe cast acceptable.

For example:

```text
VARCHAR '00123'
NUMBER 123
```

may qualify through the agreed lossless numeric-normalization policy.

By contrast, the engine must not automatically infer:

- Currency or unit conversion.
- Encoded-value translations.
- Locale-specific numeric interpretation.
- Arbitrary date-format conversion.
- Time-zone assumptions.
- Split/concatenate transformations.
- `first_name + last_name → full_name`.
- Transliterations or language translations.

Those require an explicitly registered additional policy. They are not established first-release features from this discussion.

### Step 10 — Verify current column content

For each eligible mapping, compare current values under its selected policies.

Define:

- \(D_s\): distinct non-null source values represented under the selected policy.
- \(D_t\): distinct non-null target values represented under the selected policy.
- \(M_s\): source distinct values with at least one verified qualifying target counterpart.
- \(M_t\): target distinct values with at least one verified qualifying source counterpart.

Then:

\[
O_s = 100 \times \frac{|M_s|}{|D_s|}
\]

\[
O_t = 100 \times \frac{|M_t|}{|D_t|}
\]

Content passes when:

```text
O_s >= content_overlap_min_pct
OR
O_t >= content_overlap_min_pct
```

Requirements:

- Count each distinct value once, regardless of frequency.
- Exclude nulls from these denominators.
- Form normalized distinct sets consistently with the selected policy.
- Preserve conversion-failure statistics.
- Do not silently discard failed conversions in a way that improves the reported overlap.
- Do not convert failed parsing into matching nulls.
- Empty/all-null inputs provide insufficient non-null evidence.
- A zero denominator produces an unavailable measurement and an explicit reason.
- Similar frequencies are separate evidence, not an additional acceptance gate.
- A sketch estimate alone must not establish final acceptance.

For exact/native/normalized comparisons, verify the relevant value overlap against the current input scope.

Where descriptive fuzzy links participate, count only directly verified qualifying links. Report fuzzy coverage as observed coverage when retrieval is heuristic.

A high observed fuzzy coverage can satisfy the gate. A low observed coverage does not prove that no additional qualifying links exist.

Keep exact/normalized overlap and additional fuzzy-qualified coverage separately inspectable.

### Step 11 — Finalize the accepted mapping graph

An ordinary mapping is accepted only when:

```text
Semantic gate passes
AND
supported comparison policy exists
AND
current content gate passes
```

Feedback changes the source of semantic evidence; it does not remove either remaining requirement.

Retain legitimate alternative mappings rather than forcing an early global assignment.

Each candidate must have an outcome such as:

- Accepted.
- Insufficient semantic evidence.
- Below metadata threshold.
- Opaque.
- Ambiguous or contradictory.
- Unsupported comparison.
- Insufficient non-null content.
- Below observed content threshold.
- Stale feedback.
- Processing error.

These reasons must remain distinct.

### Step 12 — Apply the table-level matched-column gate

Define:

- \(C_s\): all source column identities.
- \(C_t\): all target column identities.
- \(A_s\): unique source columns participating in accepted mappings.
- \(A_t\): unique target columns participating in accepted mappings.

Calculate:

\[
G_s = 100 \times \frac{|A_s|}{|C_s|}
\]

\[
G_t = 100 \times \frac{|A_t|}{|C_t|}
\]

Proceed only when:

```text
G_s >= table_column_coverage_min_pct
AND
G_t >= table_column_coverage_min_pct
```

Requirements:

- Count unique physical column identities, not mapping edges.
- Include unmatched, opaque and unsupported columns in total-column denominators.
- Multiple accepted mappings referencing one source column count that source column once.
- At exactly 30%, the default gate passes.
- Source 40% and target 20% fails the default gate.
- Failure stops row-subset comparison but preserves profiles and mapping evidence.

The accepted default can exclude sparse overlap. For example, ten accepted columns among hundreds may fail this gate. The engine must not silently lower the configured threshold to recover that finding.

### Step 13 — Discover compatible projections

A projection is a set of accepted source-to-target mappings.

Within one projection:

- Each selected physical source column appears at most once.
- Each selected physical target column appears at most once.
- Its mapping alignment is explicit.
- Competing compatible projections may remain eligible.

Discovery must consider separate patterns such as:

```text
A+B+C
B+C+D
A+B
B+C
```

They may cover different row populations.

The search must:

1. Use accepted mappings and profile evidence.
2. Generate promising row/group candidates through multiple evidence paths.
3. Record which fields qualify for the same source–target relationship.
4. Discover compatible co-agreement patterns.
5. Consider reduced or alternative patterns that add different coverage.
6. Verify proposed projections against the complete selected input scope.

Do not rely exclusively on identifier equality. That would miss relationships where another subset matches while identifiers differ.

Single-column findings are permitted, but must show their width and information strength.

Heuristic pruning is permitted. Its policy and limitations must be recorded.

No exhaustive claim is permitted unless a particular search actually establishes it.

### Step 14 — Verify each discovered projection

For each projection:

1. Order fields by the explicit mapping.
2. Apply the selected comparison policies.
3. Build subset-specific comparable tuples.
4. Group repeated tuples while preserving occurrence membership.
5. Retrieve candidate counterpart tuples.
6. Verify actual field values.
7. Execute permitted descriptive-text candidate checks separately.
8. Retain direct qualifying relationships.
9. Calculate directional membership and counts.

Fingerprinting and row predicates are specified below.

---

## 6. Value comparison and normalization

### 6.1 Native exact comparison

An exact field relationship requires compatible native values without needing numeric normalization, descriptive variation handling or shared missingness.

For text, warehouse collation behaviour must not silently classify case or whitespace variations as pure exact matches.

The exact comparator must preserve the intended distinction between:

```text
'Anita Rao'
'ANITA RAO'
' Anita Rao '
```

The latter two may qualify through the descriptive-text policy, but their equality must not be mislabelled merely because a warehouse collation ignores the difference.

### 6.2 Numeric normalization

Required examples include:

```text
'123'   ↔ 123
'00123' ↔ 123
```

Rules:

- Use a lossless representation.
- Do not pass identifiers through floating-point conversion when precision could be lost.
- Do not round or introduce numeric tolerance.
- Do not partially parse a string.
- Do not strip unrelated characters to manufacture a numeric value.
- Preserve original values and conversion evidence.
- Record invalid or unsupported representations.
- A failed conversion cannot become shared missingness.

The registered parser must explicitly define accepted lexical formats and precision boundaries. Formats outside that policy are not guessed.

No additional locale, currency, exponent-rewriting or unit-conversion behaviour was agreed.

The agreed leading-zero equivalence must remain available; do not silently revoke it while implementing identifier typo protection.

### 6.3 Descriptive-text comparison

Descriptive text may use:

- Permitted case normalization.
- Permitted whitespace normalization.
- A deterministic configured similarity comparator.

The comparator initially discussed is Snowflake `JAROWINKLER_SIMILARITY`, with an inclusive threshold of 90.

Snowflake’s function returns an integer from 0 to 100, is case-insensitive and is sensitive to formatting characters. Its result must therefore remain distinct from raw exactness and from AI assessment confidence. [Snowflake Jaro–Winkler documentation](https://docs.snowflake.com/en/sql-reference/functions/jarowinkler_similarity)

Requirements:

- Apply fuzzy comparison only when the inferred/registered role permits descriptive matching.
- Protect identifiers and codes from typo matching.
- Version preprocessing, including whitespace treatment.
- Do not infer unregistered accent removal, transliteration, token reordering or abbreviation expansion.
- Record the actual comparator score for verified fuzzy relationships.
- Do not issue an LLM request for every row pair.

If a role is uncertain, do not automatically enable typo matching.

### 6.4 Missing values

At column-verification time:

- Nulls are excluded from content-overlap denominators.
- Shared nulls do not establish semantic or non-null content evidence.

At row-verification time, within an accepted projection:

| Source | Target | Result |
|---|---|---|
| NULL | NULL | May qualify; flag shared missingness |
| NULL | Populated | Fails that field |
| Empty string | NULL | Not shared missingness |
| Literal `"NULL"` | NULL | Not shared missingness |
| Failed numeric parse | NULL | Not shared missingness |

Do not add an undisclosed requirement that every matching row must contain a populated selected field. Shared-null patterns must remain visible and identified as weak evidence where appropriate.

### 6.5 Unsupported transformations

If comparison requires a transformation not defined by a registered policy:

- Do not guess.
- Do not ask a user to approve it during the run.
- Exclude that comparison path with a reason.
- Continue other eligible comparisons.

---

## 7. Fingerprints and joint row matching

### 7.1 Column sketches versus row fingerprints

| Artifact | Purpose |
|---|---|
| Column sketch | Helps discover possible value overlap across columns |
| Projection row fingerprint | Helps retrieve matching tuples for one selected projection |

They are not interchangeable.

### 7.2 Projection fingerprints

For mappings:

```text
Source A → Target X
Source B → Target Y
Source C → Target Z
```

the comparable tuples are:

```text
Source: (normalize(A), normalize(B), normalize(C))
Target: (normalize(X), normalize(Y), normalize(Z))
```

The fingerprint must depend only on those selected fields and their applicable policies.

It must not include unrelated original columns.

`A+B+C` and `B+C+D` require separate tuple definitions and fingerprints.

Requirements:

- Preserve field boundaries.
- Preserve type/common-domain information.
- Preserve explicit missingness.
- Avoid unsafe concatenation.
- Associate fingerprints with their mapping/normalization versions.
- Verify actual tuples after hash agreement.

Snowflake `HASH` accepts selected expressions, but numeric and string representations do not automatically share hashes. Its finite hash space also means it cannot serve as proof of uniqueness. [Snowflake HASH documentation](https://docs.snowflake.com/en/sql-reference/functions/hash)

Do not assume Snowflake already stores an application-ready content fingerprint for arbitrary selected subsets.

### 7.3 Joint row predicate

For projection \(P\), a source occurrence \(s\) and target occurrence \(t\) qualify only when:

\[
Match_P(s,t)
=
\bigwedge_{m \in P} Compare_m(s,t)
\]

Every field must qualify for the same relationship.

There is no rule allowing an average field score to compensate for a failed field.

Example:

```text
Source:
(1, Red)
(2, Blue)

Target:
(1, Blue)
(2, Red)
```

Both individual columns overlap completely.

The two-column projection has zero matching tuples.

### 7.4 Fuzzy retrieval path

A full tuple hash does not detect a spelling difference.

The engine must have a separate permitted candidate path, for example:

1. Retrieve a promising pair using other agreeing fields or descriptive candidate search.
2. Calculate the actual descriptive similarity.
3. Check every remaining selected field.
4. Retain the direct relationship only if all selected predicates pass.

Retrieval limits must be reported separately from verification correctness.

### 7.5 Non-transitive relationships

If:

```text
A qualifies against B
B qualifies against C
A does not qualify against C
```

the engine must not invent `A ↔ C`.

Connected components may assist browsing, but are not unconditional equivalence classes.

---

## 8. Result categories and evidence

Provide all four result views:

| View | Meaning |
|---|---|
| Exact | The selected relationship qualifies without necessary conversion, descriptive variation or shared missingness |
| Numeric-normalized | At least one selected field requires permitted numeric normalization |
| Shared missing | At least one selected field qualifies through null–null handling |
| Descriptive-text variation | At least one selected field requires permitted descriptive normalization or similarity |

Rules:

- Evidence belongs to the actual relationship and selected projection.
- Do not classify a match using fields outside that projection.
- Numeric, missing and descriptive flags can overlap.
- A pure exact relationship does not require those other methods.
- A row can participate in different methods through different counterpart relationships.
- View totals must not be added to obtain overall covered rows.

Preserve raw-value subgroups when canonical grouping combines different representations.

For example, a normalized group may contain both:

```text
'123'
'00123'
```

Its members may participate in different exact versus normalized relationships. One group-wide label is insufficient to classify every member correctly.

Record per-field evidence sufficient to explain:

- Original value references.
- Selected comparator.
- Applied transformation.
- Conversion result.
- Missingness.
- Similarity score where applicable.
- Why the field passed or failed.

Original values remain inside authorised Snowflake/result access paths, outside model payloads.

---

## 9. Counting rules

### 9.1 Directional occurrence coverage

For a finding:

```text
Covered source occurrences =
distinct source occurrence IDs with at least one qualifying target

Covered target occurrences =
distinct target occurrence IDs with at least one qualifying source
```

Calculate:

\[
SourceCoverage =
100 \times \frac{CoveredSourceOccurrences}{AnalysedSourceOccurrences}
\]

\[
TargetCoverage =
100 \times \frac{CoveredTargetOccurrences}{AnalysedTargetOccurrences}
\]

Use the analysed scope, not an unrelated full-table count.

For empty input denominators, return unavailable percentages with an explanation.

### 9.2 Exact tuple multiplicity

For an exactly verified comparable tuple with source frequency \(s\) and target frequency \(t\):

```text
Covered source occurrences = s, when t > 0
Covered target occurrences = t, when s > 0

Reconciled occurrences = min(s, t)
Excess source occurrences = max(s − t, 0)
Excess target occurrences = max(t − s, 0)
```

Example:

```text
Source occurrences: 2
Target occurrences: 3
```

Correct output:

| Metric | Value |
|---|---:|
| Matching comparable tuple | 1 |
| Covered source occurrences | 2 |
| Covered target occurrences | 3 |
| Reconciled occurrences | 2 |
| Excess source occurrences | 0 |
| Excess target occurrences | 1 |

A six-row join expansion is not six covered source records.

All three target occurrences can be covered while one remains an excess occurrence relative to source multiplicity.

### 9.3 Fuzzy multiplicity

For fuzzy graphs:

- Covered-source and covered-target counts use distinct membership.
- Direct links remain explicit.
- Do not sum `min(s,t)` across overlapping fuzzy links.
- Do not report one-to-one reconciled fuzzy occurrences unless a separately specified allocation policy exists.

No such allocation policy was agreed. The initial engine must not invent that metric.

### 9.4 Overall coverage across findings

Calculate overall coverage using unions of occurrence IDs.

Do not sum:

- Subset counts.
- Category counts.
- Group counts containing overlapping members.
- Source and target counts.

Example:

- `A+B` may cover a record with a changed city.
- `B+C` may cover another record with a changed identifier.

Their union may cover every input row even though `A+B+C` does not.

Label that metric as participation in at least one qualifying finding. Do not call it complete-table duplication.

### 9.5 Distinguish reported units

Results must distinguish:

- Covered row occurrences.
- Distinct comparable tuples.
- Direct verified relationships.
- Reconciled exact occurrences.
- Excess occurrences.
- Repetitions within a selected tuple.
- Input rows with no discovered counterpart.

“No match found” must be qualified by the selected projection, scope and search policy.

---

## 10. Search completeness and scale

The engine must support large inputs without:

- Downloading complete datasets into Python/application memory.
- Sending row values to AI.
- Materialising every repeated source–target pair.
- Comparing every possible row pair by default.
- Enumerating all subsets of hundreds of columns by default.

The implementation may use Snowflake-side:

- Aggregation.
- Distinct-value operations.
- Value indexes/postings.
- Blocking.
- Tuple grouping.
- Hash retrieval.
- Descriptive candidate retrieval.
- Co-agreement pattern discovery.
- Redundancy compression.

Separate these completeness dimensions:

| Dimension | Required disclosure |
|---|---|
| Input scope | All rows or explicitly limited scope |
| Column inventory | All columns inventoried, with unsupported operations identified |
| Mapping discovery | Candidate-generation policy and exclusions |
| Subset discovery | Examined/discovered patterns and pruning |
| Exact verification | Whether the selected projection was fully verified over its scope |
| Fuzzy retrieval | Verified links and candidate-retrieval limitations |
| Execution completion | Completed, interrupted, failed or cancelled stages |

A heuristic policy finishing normally does not become exhaustive.

Conversely, heuristic discovery does not prevent exact full-scope counts for an exact pattern that was discovered and fully verified.

No numerical runtime, cost or discovery-recall SLA was agreed. The implementing team must measure these against the company product’s targets.

---

## 11. Feedback rules

Feedback remains part of the engine contract even though the host product provides persistence.

### 11.1 Persistence and identity

Corrections must persist in Snowflake through the product’s feedback repository.

They must not exist only in:

- Model conversation history.
- Process memory.
- A local developer file.
- An unversioned prompt.

Store:

- Both fully qualified column identities.
- Account context.
- Quoted-case-safe names.
- Object/schema signatures.
- Equivalence status.
- Creating and revising actor identities.
- User ID and username.
- Timestamps.
- Revision/history.
- Originating run.
- Applicability and revocation information.

### 11.2 Reuse

Feedback is reusable:

- Across authorised users of the same product environment.
- When source and target are selected in reverse order.
- Only for the applicable exact column pair and valid schema context.

Do not generalise it to unrelated tables or infer a third-column equivalence transitively.

### 11.3 Verification

For every reused correction:

1. Check applicability.
2. Check schema compatibility.
3. Supply semantic evidence.
4. Select a supported comparator.
5. Recalculate current content overlap.
6. Apply the configured content threshold.
7. Record the outcome and provenance.

If current overlap is insufficient:

- Exclude the mapping for the current run.
- Explain the current verification result.
- Identify the saved suggestion and its author.
- Preserve historical feedback.

Low overlap does not automatically erase the semantic correction.

### 11.4 Revisions and invalidation

- Revoked feedback cannot supply affirmative evidence.
- Schema drift requires reassessment.
- Conflicting or stale records must not be resolved through an invented model score.
- Use the repository’s versioned active revision.
- Changes affect subsequent runs, not completed results or frozen in-flight contexts.

Feedback is optional. Normal execution must complete without asking the user to validate mappings.

“Learning” here means retrieving applicable corrections. It does not mean foundation-model fine-tuning.

---

## 12. Required evidence and output structure

Every run must expose enough evidence for another system to reproduce and interpret its decisions.

### 12.1 Run evidence

- Input identities.
- Capture/version boundaries.
- Analysed and limited scope.
- Initiating user.
- Effective configuration.
- Model/prompt/comparator/search versions.
- Feedback revision.
- Stage outcomes and timings.
- Available execution/query references.

### 12.2 Column evidence

- Complete inventory.
- Profiles.
- Supported/unsupported operations.
- Candidate mappings.
- Semantic evidence and score.
- Comparator policy.
- Directional content measurements.
- Exact versus fuzzy contribution.
- Acceptance/exclusion reason.
- Feedback provenance.

### 12.3 Finding evidence

- Explicit source-to-target projection.
- Subset width.
- Comparison-method flags.
- Verified membership.
- Group frequencies.
- Direct fuzzy links where applicable.
- Directional counts and percentages.
- Applicable exact multiplicity metrics.
- Information strength.
- Verification completeness.
- Search limitations.

### 12.4 Member retrieval

The host product must be able to retrieve all authorised members of a finding through deterministic pagination or export.

Do not present a few samples as if they represent the complete membership.

Do not require a raw Cartesian expansion to provide those members.

Expired underlying evidence must be identified explicitly. Historical reports must not silently rerun against live inputs and present those results as the original run.

---

## 13. Errors, caching and lifecycle behaviour

### 13.1 Outcome semantics

Support logical outcomes equivalent to:

- Completed under the recorded policy.
- Skipped.
- Partial.
- Failed.
- Cancelled.

The host product may map these to its existing status names.

Examples:

| Condition | Required outcome |
|---|---|
| Input inaccessible | Failure with reason |
| No usable input evidence | Explicit insufficient-data outcome |
| Mapping below a threshold | Excluded candidate with evidence |
| Table gate fails | Skip row comparison |
| Unsupported column comparator | Explicit exclusion |
| AI response invalid | Stage error; no fabricated mapping |
| Query fails | Failed/partial stage; no invented zero |
| Budget/cancellation interrupts verification | Incomplete outcome with completed evidence preserved |

Unavailable metrics must remain unavailable, rather than becoming numeric zeros.

### 13.2 Retry and cancellation

- Retried work must not duplicate findings, memberships or feedback.
- Preserve the frozen input/configuration context.
- A retry against changed inputs must be identified as a new scope/run.
- Cancellation must stop further work where supported and record what completed.

### 13.3 Caching

Metadata interpretations may be reused only when their dependencies remain valid, including relevant schema/model/prompt/policy versions.

Cached semantics must still be evaluated against the current configured threshold.

Current content verification remains mandatory. A prior run’s overlap measurement is not a substitute.

---

## 14. Mandatory acceptance tests

Implement tests against controlled fixtures. Source immutability checks must not attempt destructive operations against business tables.

### 14.1 Metadata and configuration

1. Renamed equivalent columns can become candidates.
2. Repositioned columns produce the same alignment.
3. Identically named columns in different objects remain distinct identities.
4. Opaque columns without qualifying feedback are excluded.
5. High content overlap cannot rescue a failed ordinary metadata gate.
6. High metadata assessment cannot bypass content verification.
7. Values just below, exactly at and above every threshold behave correctly.
8. Presentation rounding cannot change acceptance.
9. Invalid row limits are rejected.
10. Configuration changes do not alter frozen runs.

### 14.2 Content verification

11. Distinct frequencies do not affect distinct-set overlap.
12. `{A}` versus `{A,B,C,D}` reports 100% source and 25% target overlap.
13. Either-direction acceptance applies to the content gate.
14. Nulls do not inflate content overlap.
15. All-null/empty evidence does not generate a passing percentage.
16. Normalization is applied consistently before comparable-set accounting.
17. Conversion failures remain visible and cannot become matching nulls.
18. Sketch estimates are not treated as exact threshold evidence.
19. Exact and fuzzy-qualified coverage remain distinguishable.

### 14.3 Table gate

20. Three accepted columns out of ten pass the 30% boundary.
21. Source 30%, target 20% skips row comparison.
22. Multiple mappings involving one column count that column once.
23. Opaque/unsupported columns remain in denominators.
24. Sparse ten-column overlap among hundreds respects the configured gate.

### 14.4 Value comparators

25. `'123'` versus `123` follows numeric normalization.
26. `'00123'` versus `123` qualifies under the agreed numeric rule.
27. Large exact numbers do not lose precision through floating point.
28. Invalid numeric strings do not manufacture equality.
29. Null–null qualifies only through shared-missing handling.
30. Empty string and literal `"NULL"` remain distinct from null.
31. Case/space variations are not silently labelled pure exact.
32. Descriptive similarity at 90 passes; below 90 fails.
33. Identifiers/codes cannot qualify through descriptive typo matching.
34. Unsupported transformations are excluded explicitly.

### 14.5 Joint matching and discovery

35. Crossed tuples `(1,Red)/(2,Blue)` versus `(1,Blue)/(2,Red)` have zero joint matches.
36. Matching fields from different target rows cannot create one relationship.
37. `ABC` and `BCD` can produce separate populations.
38. A lower-width subset can cover rows excluded by a wider subset.
39. Single-column findings remain clearly labelled.
40. An identifier-only candidate path cannot be the sole discovery mechanism.
41. Deliberate hash collisions are rejected by actual value verification.
42. Fuzzy A–B and B–C links do not create A–C.

### 14.6 Classification and counts

43. Two source occurrences and three target occurrences report coverage 2 and 3, not 6.
44. Exact multiplicity reconciliation reports 2 and one excess target occurrence.
45. Numeric, missing and text flags can coexist without multiplying overall coverage.
46. Raw subgroups inside canonical groups retain correct classifications.
47. Overlapping subsets use occurrence unions for overall coverage.
48. Fuzzy links do not produce invented one-to-one reconciliation.
49. Zero-row denominators return unavailable percentages.
50. “Any finding” coverage is not labelled complete-table equality.

### 14.7 Feedback and lifecycle

51. Saved feedback is reused for the exact pair.
52. Reverse selection resolves the same applicable correction.
53. Current content failure still excludes a saved mapping.
54. Feedback attribution and revisions remain inspectable.
55. Revocation and schema drift prevent invalid reuse.
56. Feedback is not generalised to similarly named unrelated columns.
57. Closing the client does not change the frozen comparison semantics.
58. Retry does not duplicate results.
59. Partial failures are not reported as successful zero-match comparisons.
60. Limited scopes use their own denominators and labels.

### 14.8 Scale and integration

61. Million-row/hundred-column fixtures do not require full collection into application memory.
62. Model payload inspection confirms no raw values, samples or frequent-value lists.
63. Planted subset overlaps test discovery recall separately from verified-result correctness.
64. All discovered exact projections marked fully verified reconcile against independent fixture calculations.
65. Existing product API and persistence adapters preserve these semantics.
66. Input records and schemas remain unchanged.

A heuristic search need not find every planted pattern unless its declared policy promises that coverage. The test report must expose discoveries and omissions rather than conceal them.

---

## 15. Instructions for the implementing AI

Use this document as the behavioural contract for replacing the existing core engine.

1. Inspect the company repository’s existing engine interfaces, configuration, Snowflake access, AI adapter, feedback repository and result contracts.
2. Map this specification onto those interfaces.
3. Preserve the agreed defaults: **70, 30, 30, 90**, with both row limits unset.
4. Implement every mandatory invariant and verification stage.
5. Keep comparison semantics separate from hosting and orchestration details.
6. Reuse company-approved, versioned policies where they conform to this specification.
7. Do not invent unsupported conversions, new business thresholds or hidden scope limits.
8. Require explicit integration settings where this document identifies a versioned policy without an agreed numerical default.
9. Preserve current-content verification even when metadata or feedback is cached.
10. Maintain requirement-to-test traceability.
11. Report actual test execution and remaining integration failures accurately.
12. Do not claim exhaustive discovery, business identity, regulatory certification or production validation that the implementation has not established.

The engine is complete only when its decisions, counts, memberships and limitations are reproducible from the frozen inputs, effective configuration and recorded policy versions.
