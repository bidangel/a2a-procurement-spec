# Crosswalk — `evaluation_criterion` to UBL / Peppol ESPD

## Status

Probe completed (2026-05-07). Open questions resolved 2026-05-08; the canonical
shape below is now shipped in `packages/structured-rfx-adapters/src/contract.ts`
and is the authoritative input for slice 73 (Ariba), slice 71 (SAM.gov), and
slice 79 (OCDS). Pinned to UBL 2.3 (`cac:TenderingCriterion`) and Peppol BIS
ESPD 1.0 (`ccv:Criterion`, `CriteriaTypeCode` listAgencyID `EU-COM-GROW`,
listVersionID `1.0.2`). When Peppol BIS Pre-Award 4.0 / UBL 2.4 land, re-pin
and re-run the diff.

Sources:

- UBL 2.3 — https://docs.oasis-open.org/ubl/UBL-2.3.html
- `cac:TenderingCriterion` element — https://www.datypic.com/sc/ubl23/e-cac_TenderingCriterion.html
- Peppol ESPD `ccv:Criterion` — https://docs.peppol.eu/pracc/espd/syntax/request/ccv-Criterion/

## Bottom line

**GREEN.** Mapping is feasible. After resolving the three open questions
(below) the canonical shape is **12 fields, distribution 5 exact / 6 partial /
1 novel** (≈ 92 % UBL-aligned). Novel territory is well below the 20–40 % red
line called out in the M2M strategy note. The shape is now committed in
`contract.ts` and is the input slice 71 / 73 / 79 must consume.

## Canonical fragment shape (shipped)

The typed shape now lives in `packages/structured-rfx-adapters/src/contract.ts`
alongside `CanonicalCriterionType` and `CanonicalEvaluationMethod`. Slice 73
(Ariba), slice 71 (SAM.gov) and slice 79 (OCDS) consume it as their canonical
output for evaluation criteria. The 12 fields are:

`perRowKey`, `externalId`, `name`, `description`, `criterionType`,
`criterionTypeCode`, `evaluationMethod`, `weightNumeric`,
`weightingConsiderationText`, `lotRef`, `parentCriterionPerRowKey`, `cpvCode`,
`legislationRef`, plus the `fields` spillover map.

## Field-by-field crosswalk

| #   | Our field                    | UBL `cac:TenderingCriterion`                                          | Peppol ESPD `ccv:Criterion`                                                  | Status      | Note                                                                                                                                                                                                                       |
| --- | ---------------------------- | --------------------------------------------------------------------- | ---------------------------------------------------------------------------- | ----------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | `externalId`                 | `cbc:ID` `[0..1]`                                                     | `cbc:ID` `[1..1]`                                                            | **exact**   | Language-independent token. Our value MUST be unique within the source document.                                                                                                                                           |
| 2   | `name`                       | `cbc:Name` `[0..*]`                                                   | `cbc:Name` `[1..1]`                                                          | **partial** | UBL allows multiple `Name` for multilang; we ship single-string (resolved Q1). Round-trip emits the canonical string only; non-preferred translations are not preserved. Revisit at profile-lift time.                     |
| 3   | `description`                | `cbc:Description` `[0..*]`                                            | `cbc:Description` `[1..1]`                                                   | **partial** | Same single-string policy as `name`.                                                                                                                                                                                       |
| 4   | `criterionType`              | `cbc:CriterionTypeCode` `[0..1]`                                      | `cbc:TypeCode` `[1..1]` (codelist `CriteriaTypeCode`, `EU-COM-GROW` `1.0.2`) | **partial** | Our 7-value enum, mapped to the EU-COM-GROW `CRITERION.{EXCLUSION,SELECTION,AWARD,...}.*` hierarchy via the cross-map (forthcoming artifact). Hybrid path: see `criterionTypeCode` below for the verbatim EU code carrier. |
| 5   | `criterionTypeCode`          | `cbc:CriterionTypeCode` `[0..1]` (verbatim)                           | `cbc:TypeCode` `[1..1]` (verbatim)                                           | **exact**   | Hybrid path (resolved Q2). Carries the source-system EU-COM-GROW code unchanged when present; preserves round-trip fidelity for EU adapters without forcing non-EU adapters onto the codelist.                             |
| 6   | `evaluationMethod`           | `cbc:EvaluationMethodTypeCode` `[0..1]`                               | n/a (ESPD is selection-side, not award-side)                                 | **partial** | UBL codelist for evaluation method. Our 4-value enum maps cleanly; document the cross-map alongside the `criterionType` map.                                                                                               |
| 7   | `weightNumeric`              | `cbc:WeightNumeric` `[0..1]`                                          | n/a (ESPD is binary fulfilment)                                              | **exact**   | UBL is `xsd:decimal`; we use `number` 0..100 normalised. Constrain in profile.                                                                                                                                             |
| 8   | `weightingConsiderationText` | `cbc:WeightingConsiderationDescription` `[0..*]`                      | n/a                                                                          | **exact**   | Free-text rationale. Single-string per Q1 resolution.                                                                                                                                                                      |
| 9   | `lotRef`                     | `cac:ProcurementProjectLotReference` `[0..*]`                         | n/a                                                                          | **partial** | UBL holds a complex lot reference (id + scheme + agency); we carry an opaque id string. Profile-narrowing decision.                                                                                                        |
| 10  | `parentCriterionPerRowKey`   | `cac:SubTenderingCriterion` `[0..*]` (inverse)                        | n/a (ESPD is flat under `RequirementGroup`)                                  | **novel**   | UBL nests sub-criteria; we flatten with a parent pointer. Round-trip is lossless via topological reconstruction. Document explicitly.                                                                                      |
| 11  | `cpvCode`                    | `cac:CommodityClassification` `[0..*]` (CPV `ItemClassificationCode`) | n/a                                                                          | **exact**   | Resolved Q3. Single 8-digit CPV code per fragment; multi-CPV inputs spill the secondary codes into `fields.cpvSecondary`. EU adapters populate; SAM.gov keeps NAICS / PSC in `fields` until justified separately.          |
| 12  | `legislationRef`             | `cac:Legislation` `[0..*]`                                            | `ccv:LegislationReference` `[0..n]`                                          | **partial** | UBL has a full `Legislation` entity (article number, jurisdiction, URI). We carry a single ref string today. Future enrichment.                                                                                            |

## UBL elements we deliberately omit

| UBL element                                                      | Why we omit                                                                                                                                                                          | Where it lives in our model instead                                                          |
| ---------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------- |
| `cbc:FulfilmentIndicator`                                        | Response-side, not definition-side.                                                                                                                                                  | Per-pursuit response state; not on the canonical criterion.                                  |
| `cbc:FulfilmentIndicatorTypeCode`                                | Same.                                                                                                                                                                                | Same.                                                                                        |
| `cac:TenderingCriterionPropertyGroup` (≈ `ccv:RequirementGroup`) | Conflates _evaluation criterion_ with _structured sub-questions_. We route those through `CanonicalQuestionnaireRowFragment` instead, with `parentCriterionPerRowKey` pointing back. | `questionnaire_row` fragment with parent pointer. Boundary worth documenting in ADR 0026 §3. |

## Open questions — resolved 2026-05-08

1. **Multilang policy — RESOLVED: single-string.** Canonical fragment carries one
   string per multilang field (`name`, `description`,
   `weightingConsiderationText`). Round-trip emits the canonical string only;
   non-preferred translations are not preserved. Promotion to
   `Array<{ lang: string; text: string }>` is deferred to the
   standards-submission slice when EU adoption is the load-bearing test.

2. **`criterionType` enum strategy — RESOLVED: hybrid.** Keep the 7-value
   `CanonicalCriterionType` enum as the primary classification, plus a sibling
   `criterionTypeCode: string | null` carrying the verbatim EU-COM-GROW
   `CriteriaTypeCode` value when the source provides one. One extra nullable
   field; preserves both readability and round-trip fidelity. The enum
   cross-map table to `CRITERION.{EXCLUSION,SELECTION,AWARD,OTHER}.*` is the
   next crosswalk artifact.

3. **CPV coverage — RESOLVED: add now.** Single `cpvCode: string | null` field
   on the canonical fragment. EU adapters populate from
   `cac:CommodityClassification`; SAM.gov keeps NAICS / PSC in `fields` until
   a separate canonical column is justified. Multi-CPV inputs spill secondary
   codes into `fields.cpvSecondary` so the lossy single-code shape stays
   reversible at parse time.

## What this probe says about the rest of the work

- The `evaluation_criterion` mapping is the highest-risk fragment (most structured, most EU codelist surface) and it lands at ≈ 90 % UBL-aligned with one clean addition (`parentCriterionPerRowKey`). The `requirement` and `opportunity` fragments will almost certainly be cleaner.
- Effort to produce the full crosswalk (all four canonical fragment kinds + Capability Claim Graph + Buyer Credential Scope) is bounded: ~80–120 rows total, similar mix of exact/partial/novel, similar codelist cross-maps. **3–4 weeks for a UBL/Peppol-fluent contractor; 6–8 weeks via an OpenPeppol Pre-Award liaison.**
- The contractor path is faster and lower-prestige. The liaison path is slower but produces an implicit endorsement that materially de-risks the AAIF Extension URI review. Recommend the liaison path unless standards-track timing constraints force the contractor option.
- The round-trip CI test (eForms F02 → canonical → UBL XML → diff) is buildable today on the proposed fragment shape. Use a public TED contract notice as the fixture. This is the proof point that travels with the AAIF submission.

## Deltas

| #   | Delta                                                                                                                                                                                                                         | Status                   |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------ |
| 1   | `packages/structured-rfx-adapters/src/contract.ts` — replace stub `CanonicalEvaluationCriterionFragment` with the typed shape; add `CanonicalCriterionType`, `CanonicalEvaluationMethod`, `criterionTypeCode`, `cpvCode`.     | **shipped** (2026-05-08) |
| 2   | `packages/structured-rfx-adapters/src/index.ts` — export the two new enum types.                                                                                                                                              | **shipped** (2026-05-08) |
| 3   | ADR 0026 §Amendments — record the `evaluation_criterion` vs `questionnaire_row` boundary (UBL `TenderingCriterionPropertyGroup` routes to questionnaire_row, not evaluation_criterion).                                       | **shipped** (2026-05-08) |
| 4   | Slice 73 doc (`slice-73-structured-rfx-adapter-sap-ariba-procurement.md`) — adopt the typed shape as the contract slice 73's Ariba adapter must produce. Slice 71 (SAM.gov) and slice 79 (OCDS) inherit the same shape.       | **shipped** (2026-05-08) |
| 5   | `docs/crosswalks/canonical-criterion-type-eu-com-grow-map.md` — table mapping `CanonicalCriterionType` to ESPD `CRITERION.{EXCLUSION,SELECTION,OTHER}.*` and to eForms `award-criterion-type`, plus reverse mapping.          | **shipped** (2026-05-08) |
| 6   | ADR 0034 (`docs/adr/0034-a2a-procurement-profile-layered-on-ubl.md`) — declares the A2A Procurement Profile is _layered on_ UBL semantics, pins UBL 2.3 / Peppol ESPD 1.0 reference versions, commits to maintenance cadence. | **shipped** (2026-05-08) |

## Out of scope for this probe

- Capability Claim Graph crosswalk (UBL `cac:QualifyingParty`, `cac:Certificate`, etc.) — separate document.
- Buyer Credential Scope crosswalk (ESPD selection-criteria filter algebra, EU eForms CTA codes) — separate document.
- Full EU-COM-GROW `CriteriaTypeCode` enum cross-map — separate artifact, depends on resolving open question 2 first.
- Round-trip CI test against a real eForms F02 fixture — separate slice; this probe just asserts feasibility.
