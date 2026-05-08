# Crosswalk — `requirement` to UBL / Peppol / OCDS / eForms

## Status

Probe (2026-05-08). Fourth per-fragment crosswalk under the ADR 0034
layered-on-UBL discipline. Authored ahead of slice 73 (Ariba) producing
canonical `requirement` rows from Ariba's structured-question shape;
follows the same pattern as the evaluation_criterion and opportunity
probes. Pinned to:

- UBL 2.3 — `cac:TenderingCriterionProperty` (the structured-property
  noun within `cac:TenderingCriterionPropertyGroup`) and the prose
  surfaces (`cbc:Description` on various tender entities).
- Peppol BIS Pre-Award — corresponding profiles around
  TenderingCriterionProperty.
- OCDS 1.1.5 with the **OCDS Requirements extension** —
  `tender.criteria[].requirementGroups[].requirements[]`.
- EU eForms — BT codes around requirement description (BT-749) and
  associated value constraints.

Re-pin trigger is event-driven per ADR 0034 §2.

Sources:

- UBL 2.3 `cac:TenderingCriterionProperty` —
  https://www.datypic.com/sc/ubl23/e-cac_TenderingCriterionProperty.html
- OCDS Requirements extension —
  https://extensions.open-contracting.org/en/extensions/requirements/master/
- Existing canonical `Requirement` —
  `packages/domain/src/types/requirement.ts:100`,
  `packages/db/src/schema/index.ts:330`.

## Bottom line

**GREEN-with-a-load-bearing-clarification.** The field-by-field mapping
is feasible at high alignment: of ~14 proposed fragment fields, the
distribution is **6 exact / 6 partial / 1 novel / 1 platform-internal**
on procurement-meaningful fields (≈92 % UBL-aligned; novel territory
one field, well below the 20 % ceiling).

The clarification: **the existing `requirement` canonical table already
carries _both_ subType=narrative and subType=questionnaire rows**
(slice 21's discriminator); there is no separate `questionnaire_row`
canonical table today, even though `CanonicalQuestionnaireRowFragment`
is declared in `contract.ts` as a stub and `'questionnaire_row'` is in
the `CanonicalOutputKind` enum. ADR 0026's 2026-05-08 amendment said
"UBL `TenderingCriterionPropertyGroup` routes to
`CanonicalQuestionnaireRowFragment`," but that fragment kind currently
has no consumer — slice 70 produces requirement fragments
(subType=questionnaire) and slice 73 plans to do the same. **Q1 below
asks the platform to decide whether `questionnaire_row` ever becomes
its own thing or whether the ADR amendment should be revised to route
property groups to `requirement` (subType=questionnaire) directly.**
The recommendation is the latter — collapse the planned distinction —
and the rest of this probe assumes that resolution.

If Q1 resolves the other way (questionnaire_row promoted to its own
canonical entity later), this probe still applies to the
narrative-side `requirement` mapping, with the questionnaire-side
fields lifting cleanly to the future questionnaire_row crosswalk.

## Proposed canonical fragment shape (slice 73 input)

`packages/structured-rfx-adapters/src/contract.ts:91` currently ships
`CanonicalRequirementFragment` as the slice-70 minimal-DDQ shape (6
fields: `perRowKey`, `title`, `body`, `sectionRef`, `mandatoryFlag`,
`standardSchema`). Slice 73's Ariba payloads carry value constraints,
data types, and expected-answer typing that won't fit. The crosswalk
implies the following extended shape (additive — slice-70 adapters
keep working with existing fields nullable on the new ones):

```ts
export type CanonicalRequirementType =
  | 'administrative'
  | 'technical'
  | 'pricing'
  | 'legal'
  | 'security'
  | 'staffing'
  | 'certification'
  | 'submission_format'
  | 'attachment_requirement'
  | 'clarification_or_q_and_a'
  | 'other';

export type CanonicalRequirementSubType = 'narrative' | 'questionnaire';

export type CanonicalRequirementValueDataType =
  | 'boolean'
  | 'integer'
  | 'number'
  | 'string'
  | 'amount'
  | 'quantity'
  | 'date'
  | 'period'
  | 'uri';

export interface CanonicalRequirementValueConstraints {
  dataType: CanonicalRequirementValueDataType;
  pattern: string | null; // regex (OCDS Requirements `pattern`)
  unit: string | null; // UBL ValueUnitCode (e.g. 'kg', 'EUR')
  minNumeric: number | null;
  maxNumeric: number | null;
  minAmount: { amount: number; currency: string } | null;
  maxAmount: { amount: number; currency: string } | null;
  expectedValue: string | number | boolean | null; // OCDS expectedValue, UBL Expected* family
  applicablePeriodStart: Date | null;
  applicablePeriodEnd: Date | null;
}

export interface CanonicalRequirementFragment {
  perRowKey: string;
  externalId: string | null; // UBL cbc:ID, OCDS requirement.id

  // Display
  title: string | null;
  body: string; // canonical description (single-string per ADR 0034)
  sectionRef: string | null;

  // Classification (Q2)
  requirementType: CanonicalRequirementType;
  requirementTypeCode: string | null; // hybrid path — verbatim source code (Ariba section code, eForms BT, etc.)
  subType: CanonicalRequirementSubType;

  // Mandatory + scheduling
  mandatoryFlag: boolean | null;
  dueAt: Date | null;

  // Slice 70 extension (already populated by CAIQ/SIG)
  standardSchema: RequirementStandardSchemaExtension | null;

  // Slice 73 Ariba prep — questionnaire-only structured response shape (Q3)
  // Null for subType=narrative; populated for subType=questionnaire when the
  // source surfaces value constraints. UBL: cac:TenderingCriterionProperty fields;
  // OCDS: tender.criteria[].requirementGroups[].requirements[].
  valueConstraints: CanonicalRequirementValueConstraints | null;

  // Evidence prompt
  evidenceNeededSummary: string | null;

  // Adapter-specific spillover
  fields: Record<string, unknown>;
}
```

This is a strict superset of the existing shape; the slice-70 DDQ
adapters become trivially compliant by populating the new fields with
sensible defaults (`requirementType='security'`,
`subType='questionnaire'`, `valueConstraints=null` since CAIQ/SIG carry
no value constraints — yes/no questions only).

## Field-by-field crosswalk

| #   | Our field                                        | UBL `cac:TenderingCriterionProperty`                                                                                                                       | OCDS Requirements ext. (`requirements[]`)                                  | eForms                 | Status                | Note                                                                                                                                                                                                                                                                                                                                               |
| --- | ------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------- | ---------------------- | --------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | `externalId`                                     | `cbc:ID`                                                                                                                                                   | `requirement.id`                                                           | OPT-200                | **exact**             | Stable identifier within the source document. CAIQ adapter populates with the per-question id (e.g. `SEF-08`).                                                                                                                                                                                                                                     |
| 2   | `title`                                          | `cbc:Name`                                                                                                                                                 | `requirement.title`                                                        | n/a                    | **partial**           | Single-string per ADR 0034. UBL `cbc:Name` is `[0..1]` here (singular within property), so multilang collapse is mild.                                                                                                                                                                                                                             |
| 3   | `body`                                           | `cbc:Description` `[0..*]`                                                                                                                                 | `requirement.description`                                                  | BT-749                 | **partial**           | Same single-string policy as `title`. UBL allows multiple `Description` for multilang.                                                                                                                                                                                                                                                             |
| 4   | `sectionRef`                                     | n/a (UBL is structurally nested, no string section pointer)                                                                                                | n/a                                                                        | n/a                    | **partial**           | Our flat string section reference; UBL/OCDS encode hierarchy structurally (parent `TenderingCriterion` / `requirementGroup`). Adapters reconstruct hierarchy from source as needed.                                                                                                                                                                |
| 5   | `requirementType`                                | n/a (no canonical type-of-requirement codelist on Property)                                                                                                | n/a (extension's classification lives elsewhere)                           | n/a (BT-507 buyer-cat) | **novel-ish**         | Our 11-value enum is procurement-flavoured (`administrative`, `technical`, etc.). Closest UBL analog is the parent `TenderingCriterion`'s `cbc:CriterionTypeCode`, but that's evaluation-side, not requirement-side. The enum stays our own; it serves classification/routing on our side and emits to the parent criterion's typeCode when known. |
| 6   | `requirementTypeCode`                            | (verbatim source code carried in `fields` if surfaced)                                                                                                     | (verbatim source code in `fields`)                                         | n/a                    | **exact**             | Hybrid carrier — verbatim source classification (Ariba section code, eForms BT, etc.). Same pattern as `criterionTypeCode` and `opportunityTypeCode`.                                                                                                                                                                                              |
| 7   | `subType`                                        | n/a — UBL splits at the document level (prose vs property)                                                                                                 | n/a (extension is questionnaire-only)                                      | n/a                    | **platform-internal** | Slice 21's `narrative` vs `questionnaire` discriminator. UBL handles the same split structurally; we surface it as a flat field. Q1 resolution preserves this.                                                                                                                                                                                     |
| 8   | `mandatoryFlag`                                  | `cbc:ValueRequiredIndicator` (Property doesn't have this directly; lives on the parent group's `cbc:FulfilmentIndicatorTypeCode` semantics)                | n/a (implicit in extension)                                                | BT-748                 | **partial**           | UBL doesn't carry per-property mandatory flag directly; the indicator lives on the criterion / property-group level. Round-trip: emit at criterion level when a group of properties is uniformly mandatory; spill exceptions via per-property `cbc:Description` annotation.                                                                        |
| 9   | `dueAt`                                          | `cac:ApplicablePeriod/cbc:EndDate` (when the property has its own deadline)                                                                                | `requirement.period.endDate`                                               | various BT             | **partial**           | UBL splits Period into Start/End; we carry a single `dueAt`. `applicablePeriodEnd` on `valueConstraints` is the broader carrier when the source provides both.                                                                                                                                                                                     |
| 10  | `standardSchema` (extension)                     | n/a — UBL has no schema-level question id                                                                                                                  | (carried via OCDS-Requirements `id` per CCCEV)                             | n/a                    | **novel-ish**         | The slice-70 cross-pursuit-reuse extension. Survives because it's the platform's discipline for cross-source standard-question identification (CAIQ SEF-08 = CAIQ SEF-08 across pursuits); no UBL counterpart by design.                                                                                                                           |
| 11  | `valueConstraints.dataType`                      | `cbc:ValueDataTypeCode`                                                                                                                                    | `requirement.dataType`                                                     | n/a                    | **exact**             | UBL/OCDS share the type axis. Our 9-value enum (`boolean`/`integer`/`number`/`string`/`amount`/`quantity`/`date`/`period`/`uri`) is the union of OCDS examples and UBL `Expected*` carriers.                                                                                                                                                       |
| 12  | `valueConstraints.pattern`                       | n/a (UBL has no regex pattern)                                                                                                                             | `requirement.pattern`                                                      | n/a                    | **partial**           | OCDS-Requirements supports regex; UBL doesn't. Round-trip emits to OCDS only; for UBL emit the pattern lives in `cbc:Description` annotation.                                                                                                                                                                                                      |
| 13  | `valueConstraints.unit`                          | `cbc:ValueUnitCode`                                                                                                                                        | (implicit in dataType + description)                                       | n/a                    | **partial**           | UBL has the typed unit; OCDS uses description text. We carry as a string (typically a UN/ECE unit code or ISO 4217 currency).                                                                                                                                                                                                                      |
| 14  | `valueConstraints.minNumeric` / `maxNumeric`     | `cbc:MinimumValueNumeric` / `cbc:MaximumValueNumeric`                                                                                                      | `requirement.minValue` / `maxValue`                                        | n/a                    | **exact**             | Numeric bounds.                                                                                                                                                                                                                                                                                                                                    |
| 15  | `valueConstraints.minAmount` / `maxAmount`       | `cbc:MinimumAmount` / `cbc:MaximumAmount`                                                                                                                  | (carried via `requirement.minValue/maxValue` with currency in description) | n/a                    | **partial**           | UBL types amounts with `currencyID`; OCDS lacks the currency dimension on amounts. Adapter populates both fields when the source carries currency-typed bounds.                                                                                                                                                                                    |
| 16  | `valueConstraints.expectedValue`                 | `cbc:ExpectedID` / `ExpectedAmount` / `ExpectedIndicator` / `ExpectedCode` / `ExpectedValueNumeric` / `ExpectedDescription` / `ExpectedURI` (all `[0..1]`) | `requirement.expectedValue`                                                | n/a                    | **partial**           | UBL splits the "expected response" axis across 7 elements typed by data type; OCDS collapses to one. We carry one `expectedValue` (typed at runtime by `dataType`) and round-trip to the right UBL Expected\* family on emit.                                                                                                                      |
| 17  | `valueConstraints.applicablePeriodStart` / `End` | `cac:ApplicablePeriod/cbc:StartDate` / `cbc:EndDate`                                                                                                       | `requirement.period.startDate` / `endDate`                                 | n/a                    | **exact**             | Period for the constraint to apply (e.g. "certification valid for the next 12 months").                                                                                                                                                                                                                                                            |
| 18  | `evidenceNeededSummary`                          | `cac:TemplateEvidence/cbc:Description`                                                                                                                     | n/a (not in extension; lives in `requirementGroup.description`)            | n/a                    | **partial**           | UBL's TemplateEvidence is structured (carries id, name, description, format codes); we keep a flat summary string that rolls up the source's evidence prompts. Future structured `requirement_evidence_template` extension is an open follow-on if a downstream consumer surfaces.                                                                 |

## UBL elements we deliberately omit

| UBL element                                                                                                           | Why we omit                                                                                                                                                                                                                                                            | Where it lives in our model instead                                                                                      |
| --------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| `cac:TenderingCriterionProperty/cbc:CertificationLevelDescription` `[0..*]`                                           | Niche EU-procurement concept (e.g. ISO certification level required); rare in our adapter set.                                                                                                                                                                         | `fields.certificationLevels` when an adapter surfaces; promote on demand.                                                |
| `cbc:CopyQualityTypeCode`                                                                                             | Niche (paper-copy quality requirements); pre-electronic-procurement legacy.                                                                                                                                                                                            | Out of scope.                                                                                                            |
| `cbc:TranslationTypeCode`                                                                                             | Translation-type requirement (e.g. "submit in English and French").                                                                                                                                                                                                    | `fields` spillover.                                                                                                      |
| `cac:TenderingCriterionProperty/cbc:TypeCode`                                                                         | Property typeCode codelist; rare on the Ariba/SAM/OCDS adapter set.                                                                                                                                                                                                    | `fields.propertyTypeCode` when surfaced.                                                                                 |
| `cac:TemplateEvidence/cbc:DocumentTypeCode`                                                                           | Structured evidence-template-type metadata.                                                                                                                                                                                                                            | Future `requirement_evidence_template` extension when slice 73's evidence-prompt surface needs structuring.              |
| `cac:TenderingCriterion` (parent) — full structure                                                                    | Owned by the `evaluation_criterion` fragment (already crosswalked). Requirement carries only `sectionRef` as a back-pointer; the relationship between `requirement` and its parent `evaluation_criterion` is a downstream linking concern handled by the orchestrator. | Existing `requirement.opportunity_id` link + future explicit `evaluation_criterion_id` FK if a slice needs it.           |
| Free-form prose tender-document fields (`cac:Tender/cbc:Description`, `cac:ProcurementProject/cbc:Description`, etc.) | Where narrative requirements come FROM, not where they map TO; these are extraction-side concerns handled by slice-22 (Word/PDF) and slice-61 (LLM extraction), not adapter-side.                                                                                      | Slice 22 / 61 produce `subType=narrative` rows from prose; this crosswalk documents the post-extraction canonical shape. |

## Open questions to resolve

1. **`requirement` vs `questionnaire_row` boundary.** Today the
   canonical `requirement` table carries both subType=narrative and
   subType=questionnaire rows; there is no separate `questionnaire_row`
   table. ADR 0026's 2026-05-08 amendment said
   `TenderingCriterionPropertyGroup` routes to a (still-stub)
   `CanonicalQuestionnaireRowFragment`. Three viable resolutions:
   - **(A)** Collapse the planned distinction. Revise the ADR
     amendment so property-group-shaped sources route to
     `CanonicalRequirementFragment` (subType=questionnaire) directly,
     and remove `CanonicalQuestionnaireRowFragment` /
     `'questionnaire_row'` from `CanonicalOutputKind`. Single canonical
     entity for both flavors; simpler.
   - **(B)** Keep the distinction. Promote `questionnaire_row` to its
     own canonical table at slice 73 implementation time; migrate
     existing slice-70 questionnaire-flavored requirement rows over.
     More faithful to UBL's structural split; bigger schema change.
   - **(C)** Defer. Keep the stub fragment kind in `contract.ts`,
     produce no questionnaire_row fragments today, decide at slice 79
     (OCDS) when the structural-vs-flat tension would be most
     concrete.
   - **Recommendation: (A).** The slice-21 `subType` discriminator
     already does the work the structural split would do, and the
     existing `requirement_questionnaire` extension table adds the
     questionnaire-only fields. Two canonical tables for a distinction
     a discriminator already captures is over-modelling. The ADR
     amendment lift is a small follow-up commit.

2. **`requirementType` on the fragment vs orchestrator default.**
   Slice 70's CAIQ/SIG adapters today implicitly default
   `requirement_type='security'` (the orchestrator's job, since the
   existing fragment doesn't carry the field). Slice 73 Ariba can't
   default — Ariba sections classify questions as administrative /
   technical / pricing / past-performance variably.
   - **(A)** Promote to a required field on `CanonicalRequirementFragment`,
     with a sibling nullable `requirementTypeCode` for the verbatim
     source code (hybrid path).
   - **(B)** Keep optional; orchestrator defaults from the parent
     evaluation_criterion's `criterionType` when null.
   - **Recommendation: (A).** Same hybrid path as `criterionType` and
     `opportunityType`; consistent across the three crosswalks.
     Slice 70 adapters set `requirementType='security'` explicitly at
     the adapter, no implicit defaulting.

3. **How rich should `valueConstraints` be?** UBL
   `cac:TenderingCriterionProperty` has 7 expected-value carriers
   (Amount, ID, Indicator, Code, ValueNumeric, Description, URI) plus
   6 bounds carriers (Min/Max for Amount/ValueNumeric/Quantity) plus
   ApplicablePeriod plus dataType plus unit plus pattern. OCDS
   Requirements collapses to a leaner shape (dataType, pattern,
   expectedValue, minValue, maxValue, period). Slice 73 Ariba's
   question types (text / numeric / choice / attachment / table) are
   leaner still.
   - **(A)** Full UBL-aligned shape (lossless round-trip on all
     UBL emit paths).
   - **(B)** OCDS-aligned shape (lean; covers OCDS / Ariba / SAM
     comfortably; UBL-emit lossy on the Quantity dimension).
   - **(C)** Defer entirely; spill constraints to `fields` until a
     downstream surface needs them.
   - **Recommendation: (B).** OCDS's shape is a documented superset
     of every adapter we'll ship in the near term; the Quantity loss
     is acceptable (none of our MVP adapters surface quantity-typed
     requirements). The `expectedValue` field is union-typed at
     runtime by `dataType`, mirroring OCDS's pattern.

4. **`requirement_questionnaire` extension and structured response
   shape.** Slice 21's `requirement_questionnaire` extension table
   (`packages/db/src/schema/index.ts:1756`) captures the per-row
   answer state for questionnaire requirements (response value, state
   machine: `unanswered` / `drafting` / `answered` / `approved`). On
   the fragment side, this is response-side, not definition-side, and
   stays out of `CanonicalRequirementFragment`. Confirming for the
   record: nothing changes on the extension table from this
   crosswalk; it's downstream of the canonical requirement and
   downstream of the buyer-asking shape this probe describes. No
   recommendation needed.

## What this probe says about the rest of the work

- Bucket distribution at ≈92 % UBL-aligned on procurement-meaningful
  fields keeps us comfortably below the 20 % novel-share ceiling. The
  one novel-ish field (`requirementType`) is internally meaningful
  (drives reviewer-queue routing) and absent from UBL by design;
  same shape as the other crosswalks' canonical enums.
- **Q1's resolution affects ADR 0026 §Amendments and the
  `CanonicalOutputKind` enum.** If (A) — recommended — we add a
  third 2026-05-08 amendment retiring the stub
  `CanonicalQuestionnaireRowFragment` and route property-groups to
  `CanonicalRequirementFragment` (subType=questionnaire) directly.
  The ADR-0034 §3 future-crosswalks list shrinks by one accordingly.
- **Slice 73's Ariba question-type mapping** (already named in slice
  73 doc) gains a typed home: each Ariba question becomes a
  `CanonicalRequirementFragment` (subType=questionnaire) with
  `valueConstraints` populated per Ariba's question-type metadata
  (text → dataType=string, numeric → dataType=number with optional
  unit, choice → dataType=string with `expectedValue` carrying the
  chosen-from set, attachment → dataType=uri, table → dataType=string
  with a flag note).
- **Subsequent crosswalks** become smaller. After this one, the only
  "active-trigger" remaining in ADR 0034 §3 is `questionnaire_row`,
  which Q1=A retires. The others (`capability_claim`,
  `external_buyer_credential` scope) are Phase B per ADR 0032 and
  wait for the counterparty gate.

## Deltas — resolved 2026-05-08 (Q1=A, Q2=A, Q3=B, Q4 no-action)

| #   | Delta                                                                                                                                                                                                                                                                                                                                                                                                       | Status                   |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------ |
| 1   | `packages/structured-rfx-adapters/src/contract.ts` — extended `CanonicalRequirementFragment` to the typed shape; added `CanonicalRequirementType`, `CanonicalRequirementSubType`, `CanonicalRequirementValueDataType` enums and `CanonicalRequirementValueConstraints` interface. Removed the stub `CanonicalQuestionnaireRowFragment` and dropped `'questionnaire_row'` from `CanonicalOutputKind` (Q1=A). | **shipped** (2026-05-08) |
| 2   | `packages/structured-rfx-adapters/src/index.ts` — exported the new types; removed `CanonicalQuestionnaireRowFragment`.                                                                                                                                                                                                                                                                                      | **shipped** (2026-05-08) |
| 3   | ADR 0026 §Amendments — third 2026-05-08 amendment retiring the stub fragment kind and revising the property-group routing rule. ADR 0034 §3 future-crosswalks list dropped `questionnaire_row`.                                                                                                                                                                                                             | **shipped** (2026-05-08) |
| 4   | Slice 73 doc — adopted the typed shape; explicit `valueConstraints` mapping for Ariba's five question types.                                                                                                                                                                                                                                                                                                | **shipped** (2026-05-08) |
| 5   | Slice 70 adapters (CAIQ v4, SIG-Lite, SIG-Core) — populate `requirementType='security'` and `subType='questionnaire'` explicitly. The API sink (`apps/api/src/modules/structured-rfx-adapters/sink.ts`) now reads from the fragment instead of defaulting `RequirementType.OTHER` / `RequirementSubType.QUESTIONNAIRE`.                                                                                     | **shipped** (2026-05-08) |
| 6   | (Optional) Cross-map artifact — `CanonicalRequirementValueDataType` ↔ UBL Expected\* family ↔ OCDS dataType.                                                                                                                                                                                                                                                                                                | deferred                 |

## Out of scope for this probe

- **`requirement_questionnaire` extension** (Q4 above) — response-side
  state, not definition-side.
- **`evaluation_criterion` ↔ `requirement` link** (sectionRef as
  back-pointer vs explicit FK) — a future schema decision when slice
  73's parent-criterion linkage becomes concrete.
- **Round-trip CI test** against a TED F02 fixture's
  TenderingCriterionProperty subtree — separate slice; this probe
  asserts feasibility.
- **Capability-claim crosswalk** (ADR 0025) — Phase B per ADR 0032;
  reuses the Buyer crosswalk's `cac:Party` mapping for `subjectParty`.
- **`external_buyer_credential` scope crosswalk** — Phase B per ADR 0032.
