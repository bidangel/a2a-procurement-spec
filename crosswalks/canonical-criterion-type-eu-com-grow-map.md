# Cross-map — `CanonicalCriterionType` ↔ EU `CriteriaTypeCode` and eForms `award-criterion-type`

## Status

Authored 2026-05-08 alongside the typed `CanonicalEvaluationCriterionFragment`
shape (`packages/structured-rfx-adapters/src/contract.ts`) and ADR 0034. Pinned
to:

- Peppol BIS ESPD 1.0, codelist `CriteriaTypeCode`, listAgencyID
  `EU-COM-GROW`, listVersionID `1.0.2` (the ESPD selection / exclusion
  codelist).
- EU eForms `award-criterion-type` codelist (the award-side codelist used by
  TED / Implementing Regulation 2019/1780).
- UBL 2.3 `cbc:CriterionTypeCode` is the abstract carrier; the codelist used
  in any given message depends on whether the criterion is selection-side
  (ESPD) or award-side (eForms).

When listVersionID bumps land, re-run the diff — entries marked _present in
1.0.2_ are conservative.

Sources:

- Peppol ESPD `CriteriaTypeCode` codelist — https://docs.peppol.eu/pracc/espd/codelist/CriteriaTypeCode/
- TED eForms codelists index — https://docs.ted.europa.eu/eforms/latest/reference/code-lists/index.html

## Why two codelists

The 7-value `CanonicalCriterionType` enum was designed for **award-stage**
criteria (price / quality / technical / sustainability / past performance /
social value / other) — what a buyer scores tenders against. EU procurement
uses:

- The **`award-criterion-type`** codelist (eForms / TED) for award-stage
  criteria. Small flat enumeration (`price`, `cost`, `quality`).
- The **`CriteriaTypeCode`** codelist (ESPD, `EU-COM-GROW` `1.0.2`) for
  **selection-stage** criteria — qualification grounds (exclusion /
  selection / other supplier-data). Hierarchical: `CRITERION.EXCLUSION.*`,
  `CRITERION.SELECTION.*`, `CRITERION.OTHER.*`.

The same UBL `cbc:CriterionTypeCode` element carries either codelist value
depending on context. Round-trip fidelity therefore requires that
`criterionTypeCode` on our fragment carries the verbatim codelist value AND
the codelist identifier alongside (`listID` + `listAgencyID` per UBL norms).

In practice MVP adapters (Ariba, SAM.gov, OCDS) almost always emit
**award-stage** criteria. Selection-side criteria appear when an adapter
ingests a published ESPD request — slice 79 (OCDS) is the most likely first
caller. The cross-map covers both directions so neither side is a future
hazard.

## Forward direction — `CanonicalCriterionType` → EU code

When emitting EU output (UBL XML, Peppol BIS Pre-Award message, eForms F02),
the adapter or assembler picks the EU code from the table below. If the
source did not provide a `criterionTypeCode`, the adapter uses the canonical
enum to pick a default; if the source did provide one, the verbatim code
takes precedence (hybrid path, ADR 0026 §Amendments).

### Award-side defaults (eForms `award-criterion-type` codelist)

| `CanonicalCriterionType` | Default eForms code                                               | Note                                                                                                                                                                                                                                                                            |
| ------------------------ | ----------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `price`                  | `price`                                                           | Lowest-price-only award.                                                                                                                                                                                                                                                        |
| `quality`                | `quality`                                                         | Quality-only or quality-weighted award.                                                                                                                                                                                                                                         |
| `technical`              | `quality`                                                         | eForms folds technical-merit under `quality`; the criterion's `name` / `description` carries the technical specificity.                                                                                                                                                         |
| `sustainability`         | `quality`                                                         | eForms has no dedicated sustainability code at the award-criterion-type level; use `quality` and let `name` / `description` carry the environmental / sustainability framing. CPV codes on the criterion (`cpvCode` field) carry the green-procurement signal where applicable. |
| `past_performance`       | `quality`                                                         | Award-stage past-performance scoring is rare in EU procurement (most past-performance lives in selection); `quality` is the conservative landing.                                                                                                                               |
| `social_value`           | `quality`                                                         | Same shape as `sustainability` — eForms folds social-value into quality at the codelist level.                                                                                                                                                                                  |
| `other`                  | `cost` (if non-price economic dimension) or `quality` (otherwise) | Falls back to `cost` only when the criterion is explicitly a non-price economic factor (lifecycle cost, total cost of ownership). Default to `quality` otherwise.                                                                                                               |

### Selection-side mapping (ESPD `CriteriaTypeCode`, listVersionID `1.0.2`)

For canonical fragments that originated from an ESPD ingestion, the adapter
SHOULD set `criterionTypeCode` to the verbatim ESPD code and use the
canonical enum below as the in-product classification.

| `CanonicalCriterionType` | Representative ESPD branches                                                                                                                                                        | Note                                                                                                                                                                                                  |
| ------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `price`                  | n/a                                                                                                                                                                                 | ESPD is selection / exclusion; price is award-stage.                                                                                                                                                  |
| `quality`                | `CRITERION.SELECTION.TECHNICAL_PROFESSIONAL_ABILITY.CERTIFICATES.*`, `CRITERION.SELECTION.TECHNICAL_PROFESSIONAL_ABILITY.QUALITY_ASSURANCE.*`                                       | Quality-management certifications (ISO 9001 etc.) and equivalent.                                                                                                                                     |
| `technical`              | `CRITERION.SELECTION.TECHNICAL_PROFESSIONAL_ABILITY.*` (broad)                                                                                                                      | The widest selection branch; covers technical capacity, equipment, education / qualifications, supply-chain management.                                                                               |
| `sustainability`         | `CRITERION.SELECTION.TECHNICAL_PROFESSIONAL_ABILITY.TECHNICAL.ENVIRONMENTAL_MANAGEMENT_MEASURES`, `CRITERION.SELECTION.TECHNICAL_PROFESSIONAL_ABILITY.CERTIFICATES.ENVIRONMENTAL_*` | Environmental-management commitments and certifications (ISO 14001, EMAS).                                                                                                                            |
| `past_performance`       | `CRITERION.SELECTION.TECHNICAL_PROFESSIONAL_ABILITY.REFERENCES.*`                                                                                                                   | Past-performance references (works / supplies / services).                                                                                                                                            |
| `social_value`           | `CRITERION.OTHER.EO_DATA.SHELTERED_WORKSHOP`, `CRITERION.OTHER.EO_DATA.RESERVED_FOR_PROGRAMME`                                                                                      | Social-value reservations (sheltered workshop, social-economy reservations).                                                                                                                          |
| `other`                  | `CRITERION.EXCLUSION.*` (any), `CRITERION.OTHER.EO_DATA.*` (residual)                                                                                                               | Exclusion grounds (corruption, bankruptcy, tax compliance, sanctions, etc.) all land on `other` because they are not award-stage classifications and do not fit the seven product-meaningful buckets. |

## Reverse direction — EU code → `CanonicalCriterionType`

When ingesting EU input, adapters set `criterionTypeCode` to the verbatim EU
code AND set `criterionType` per the rules below. The mapping prefers
specificity (the most narrow canonical bucket that fits the EU code) and
defaults to `other` only when no specific bucket applies.

### From eForms `award-criterion-type`

| eForms code | `CanonicalCriterionType` | Rationale                                                                                                                                                                                                                                                                                                                |
| ----------- | ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `price`     | `price`                  | 1:1.                                                                                                                                                                                                                                                                                                                     |
| `cost`      | `other`                  | Cost ≠ price (cost includes lifecycle / TCO factors); no canonical equivalent, set `other` and let `description` carry the specifics.                                                                                                                                                                                    |
| `quality`   | `quality`                | 1:1. The adapter MAY refine to `technical` / `sustainability` / `social_value` / `past_performance` if `criterionTypeCode` itself is `quality` but the criterion's text or referenced legislation indicates a more specific bucket. Refinement is allowed only when the source signal is explicit; default is `quality`. |

### From ESPD `CriteriaTypeCode`

The ESPD codelist is large (≈ 100 codes across 1.0.2). Mapping is by code
**prefix**:

| ESPD prefix                                                                                            | `CanonicalCriterionType`                                                               |
| ------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------- |
| `CRITERION.EXCLUSION.*`                                                                                | `other`                                                                                |
| `CRITERION.SELECTION.SUITABILITY.*`                                                                    | `other` (suitability is registration / authorisation, not a product-meaningful bucket) |
| `CRITERION.SELECTION.ECONOMIC_FINANCIAL_STANDING.*`                                                    | `other` (financial standing — economic indicator without a product-meaningful bucket)  |
| `CRITERION.SELECTION.TECHNICAL_PROFESSIONAL_ABILITY.REFERENCES.*`                                      | `past_performance`                                                                     |
| `CRITERION.SELECTION.TECHNICAL_PROFESSIONAL_ABILITY.CERTIFICATES.QUALITY_ASSURANCE.*`                  | `quality`                                                                              |
| `CRITERION.SELECTION.TECHNICAL_PROFESSIONAL_ABILITY.CERTIFICATES.ENVIRONMENTAL_*`                      | `sustainability`                                                                       |
| `CRITERION.SELECTION.TECHNICAL_PROFESSIONAL_ABILITY.TECHNICAL.ENVIRONMENTAL_MANAGEMENT_MEASURES`       | `sustainability`                                                                       |
| `CRITERION.SELECTION.TECHNICAL_PROFESSIONAL_ABILITY.*` (residual, after the more specific rules above) | `technical`                                                                            |
| `CRITERION.OTHER.EO_DATA.SHELTERED_WORKSHOP`                                                           | `social_value`                                                                         |
| `CRITERION.OTHER.EO_DATA.RESERVED_FOR_PROGRAMME`                                                       | `social_value`                                                                         |
| `CRITERION.OTHER.EO_DATA.*` (residual)                                                                 | `other`                                                                                |
| `CRITERION.OTHER.*` (residual)                                                                         | `other`                                                                                |

The reverse mapping deliberately flattens many EU codes onto `other` rather
than over-fitting the canonical enum to ESPD's structure. EU adapters that
need the original ESPD distinction always have it on
`criterionTypeCode`.

## Round-trip fidelity guarantees

- **EU → canonical → EU**: lossless on `criterionTypeCode` (verbatim
  passthrough). The `criterionType` enum value may be a coarsening of the
  source code, but the EU code is recoverable verbatim from
  `criterionTypeCode`. The forward-direction default in §Forward direction
  is used only when the source did not provide a code.
- **Non-EU source → canonical → EU**: lossy on the EU code dimension —
  the adapter picks the eForms / ESPD code per the §Forward direction
  defaults. Loss is bounded: the picked code is a valid codelist member,
  so the emitted UBL passes EU schema validation. The original
  source-system code (Ariba's `criteriaType`, SAM.gov's evaluation factor
  type, etc.) is preserved in `fields` for forensic round-trip.
- **Multi-codelist sources**: when a source provides BOTH an award-stage
  and a selection-stage criterion in the same payload, the adapter emits
  separate `CanonicalEvaluationCriterionFragment` entries (one per
  criterion). The canonical enum value reflects the criterion's product
  meaning; the codelist value reflects the source's stage classification.

## Open follow-ups

- **JSON-LD `@context` generator.** The cross-map should ship as a
  generated `@context` JSON file alongside the AAIF Extension URI submission
  (ADR 0034). Today the cross-map is documentation; the machine-readable
  shape is the next artifact.
- **Round-trip CI test.** A real anonymised eForms F02 contract notice +
  ESPD response, run through `structured-rfx-adapters` (when slice 79's
  OCDS adapter ships), emitted back to UBL XML, diffed against the
  original on the supported subset. The cross-map's correctness is what
  that test ultimately validates.
- **eForms `award-criterion-type` codelist version pinning.** The eForms
  codelists evolve with TED / IR 2019/1780 revisions; the next minor
  revision may add or rename values. The maintenance commitment in ADR
  0034 covers this; document version-shift impact in the changelog when
  it lands.
- **ESPD listVersionID drift.** Peppol BIS ESPD has been at `1.0.2` for
  some time; an ESPD-EDM 3.x revision is the expected forward move. When
  the codelist version changes, re-pin and re-run the diff (this doc).
