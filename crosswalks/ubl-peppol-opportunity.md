# Crosswalk — `opportunity` to UBL / Peppol / OCDS / eForms

## Status

Probe (2026-05-08). One-page feasibility check ahead of slice 71 (SAM.gov)
defining the typed `CanonicalOpportunityFragment` shape; mirrors the
2026-05-07 evaluation-criterion probe pattern and rides on ADR 0034's
layered-on-UBL discipline. Pinned to:

- UBL 2.3 (`CallForTenders` document, with `cac:TenderingProcess` /
  `cac:ProcurementProject` / `cac:ContractingParty`).
- Peppol BIS Pre-Award (call-for-tenders transactions T004 / T005).
- OCDS 1.1.5 (`release.tender` object).
- EU eForms (Implementing Regulation 2019/1780) — Contract Notice (F02 in
  pre-eForms numbering).

Re-pin trigger is event-driven per ADR 0034 §2 (UBL 2.4 / Peppol BIS
Pre-Award 4.0 / OCDS 1.2 / eForms codelist revisions).

Sources:

- UBL 2.3 — https://docs.oasis-open.org/ubl/UBL-2.3.html
- `cac:TenderingProcess` element — https://www.datypic.com/sc/ubl23/e-cac_TenderingProcess.html
- OCDS 1.1.5 schema reference — https://standard.open-contracting.org/latest/en/schema/reference/
- TED eForms codelists — https://docs.ted.europa.eu/eforms/latest/reference/code-lists/index.html

## Bottom line

**GREEN.** Mapping is feasible at high alignment. Of ~22 proposed fields on
`CanonicalOpportunityFragment`, the bucket distribution is **6 exact / 9
partial / 1 novel / 4 platform-internal** (the platform-internal four —
`regime`, `sourceFamily`, `procurementRegime`, `sourceConfidence` — do not
count toward the novel-share ceiling because they are platform-pipeline
metadata not emitted to UBL). On procurement-meaningful fields the ratio is
**6 / 9 / 1 of 16** (≈ 94 % UBL-aligned). Novel territory is one field
(`primaryCategoryScheme`) and is well below the 20 % ceiling. The probe
recommends shipping the typed shape into `contract.ts` ahead of slice 71's
implementation, same shape as the evaluation-criterion path. Three open
questions to resolve (below).

## Proposed canonical fragment shape (slice 71 input)

`packages/structured-rfx-adapters/src/contract.ts:109` currently ships
`CanonicalOpportunityFragment` as a stub. The crosswalk implies the
following typed shape (mirroring the existing canonical `opportunity` table
columns at `packages/db/src/schema/index.ts:144` plus a small set of
crosswalk-driven additions):

```ts
export interface CanonicalOpportunityFragment {
  perRowKey: string;

  // Identity
  externalId: string | null; // source-system stable id (SAM noticeId, OCDS ocid, Ariba eventId)

  // Display (canonical opportunity columns)
  canonicalTitle: string;
  summary: string | null;

  // Buyer reference — the orchestrator resolves/creates the Buyer row from this
  buyerRef: {
    canonicalName: string;
    countryCode: string | null;
    parentName: string | null; // for federal sub-agencies / org-unit chains
  };

  // Procurement classification
  opportunityType: CanonicalOpportunityType; // 7-value enum (below)
  opportunityTypeCode: string | null; // hybrid path — verbatim source code (eForms procurement-procedure-type, OCDS procurementMethod)
  contractVehicle: string | null; // framework agreement / IDIQ / BPA / SDP / DPS

  // Lifecycle
  status: CanonicalOpportunityStatus; // canonical 5-value lifecycle (below)
  publishAt: Date | null;
  dueAt: Date | null;

  // Value (UBL has only a single estimated total; we keep low/high to absorb OCDS minValue + value)
  estimatedValueLow: number | null;
  estimatedValueHigh: number | null;
  currency: string | null;

  // Geography (high-level country/region; subdivision detail flows through opportunity_source_metadata)
  geography: string | null;

  // Primary commodity classification (CPV / UNSPSC / NAICS / PSC — see Q3)
  primaryCategoryCode: string | null;
  primaryCategoryScheme: 'cpv' | 'unspsc' | 'naics' | 'psc' | 'tenant_custom' | null;

  // Platform-pipeline metadata (NOT emitted to UBL; canonical opportunity row carries these for routing/QA)
  regime: string; // 'us_federal' | 'uk_public' | 'eu_oj' | 'enterprise_global' | ...
  sourceFamily: string; // 'gov_portal' | 'enterprise_procurement' | 'open_contracting'
  procurementRegime: string | null;
  sourceConfidence: number | null;

  // Adapter-specific spillover
  fields: Record<string, unknown>;
}

export type CanonicalOpportunityType =
  | 'goods'
  | 'services'
  | 'works'
  | 'mixed_goods_services'
  | 'concession'
  | 'framework'
  | 'other';

export type CanonicalOpportunityStatus = 'planned' | 'active' | 'closed' | 'awarded' | 'cancelled';
```

## Field-by-field crosswalk

| #   | Our field                | UBL (`CallForTenders` / `cac:TenderingProcess` / `cac:ProcurementProject`)                                            | OCDS `release.tender`                                                                             | eForms F02                       | Status                | Note                                                                                                                                                                                                                                                                                                                          |
| --- | ------------------------ | --------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- | -------------------------------- | --------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | `externalId`             | `cac:NoticeDocumentReference/cbc:ID` (or root `cbc:UUID` of CallForTenders)                                           | `tender.id` (commonly the `ocid`)                                                                 | F02 OPT-200                      | **exact**             | Language-independent token; uniquely identifies the procurement notice.                                                                                                                                                                                                                                                       |
| 2   | `canonicalTitle`         | `cac:ProcurementProject/cbc:Name`                                                                                     | `tender.title`                                                                                    | F02 BT-21                        | **partial**           | UBL `cbc:Name` is `[1..*]` (multilang allowed); we ship single-string per ADR 0034's standing multilang policy.                                                                                                                                                                                                               |
| 3   | `summary`                | `cac:ProcurementProject/cbc:Description`                                                                              | `tender.description`                                                                              | F02 BT-24                        | **partial**           | Same single-string policy as `canonicalTitle`.                                                                                                                                                                                                                                                                                |
| 4   | `buyerRef.canonicalName` | `cac:ContractingParty/cac:Party/cac:PartyName/cbc:Name`                                                               | `tender.procuringEntity.name` / `parties[].name`                                                  | F02 BT-500                       | **partial**           | Buyer canonical name. The orchestrator resolves to a `Buyer` row by `(canonicalName, countryCode)` rather than carrying the full UBL party shape on the fragment; the full crosswalk for `Buyer` ↔ `cac:Party` is a separate artifact (committed in ADR 0034 §3).                                                             |
| 5   | `buyerRef.countryCode`   | `cac:ContractingParty/cac:Party/cac:PostalAddress/cac:Country/cbc:IdentificationCode`                                 | `parties[].address.countryName`                                                                   | F02 BT-505 (sub)                 | **partial**           | ISO 3166-1 alpha-2 on our side; UBL allows multiple country codelists.                                                                                                                                                                                                                                                        |
| 6   | `buyerRef.parentName`    | `cac:ContractingParty/cac:Party/cac:PartyLegalEntity/cac:HeadOfficeAddress` (proxy)                                   | `parties[].roles` + relationship extension                                                        | F02 BT-633 (rare)                | **novel-ish**         | UBL has no clean parent-agency pointer; we use it for federal sub-agency chains (US-specific). Future Buyer crosswalk will record this as a partial mapping, not novel — for now keep on the fragment.                                                                                                                        |
| 7   | `opportunityType`        | `cac:ProcurementProject/cbc:ProcurementTypeCode`                                                                      | `tender.mainProcurementCategory`                                                                  | F02 BT-23                        | **partial**           | Our 7-value enum vs UBL's `ProcurementTypeCode` (typically `goods` / `services` / `works`) and OCDS's `mainProcurementCategory`. Hybrid path applies via `opportunityTypeCode` (#8).                                                                                                                                          |
| 8   | `opportunityTypeCode`    | `cac:TenderingProcess/cbc:ProcedureCode` OR `cac:ProcurementProject/cbc:ProcurementTypeCode` (depending on dimension) | `tender.procurementMethod`                                                                        | F02 BT-105 / BT-23               | **exact**             | Hybrid carrier (resolved Q2 from the evaluation-criterion probe applies here too). Conflict resolution: when the source provides both codes, we prefer the procedure code (BT-105 / `procurementMethod`); the type code lives in `fields.opportunityProcurementTypeCode` if needed.                                           |
| 9   | `contractVehicle`        | `cac:TenderingProcess/cac:FrameworkAgreement`                                                                         | `tender.contractTerms` (extension)                                                                | F02 BT-765                       | **partial**           | UBL `FrameworkAgreement` is a complex element (with terms, duration, max-amount); we carry an opaque label string. Profile-narrowing decision.                                                                                                                                                                                |
| 10  | `status`                 | `cac:TenderingProcess/cbc:TerminatedIndicator` (binary) + lifecycle-by-document-type                                  | `tender.status` (`planning` / `active` / `cancelled` / `unsuccessful` / `complete` / `withdrawn`) | F02 various BT codes             | **partial**           | Our 5-value lifecycle vs OCDS's 6-value vs UBL's binary indicator. Cross-map deferred to a future artifact (analogous to the EU-COM-GROW map).                                                                                                                                                                                |
| 11  | `publishAt`              | Root `cbc:IssueDate` of the CallForTenders document                                                                   | `release.date`                                                                                    | F02 publication (BT-738)         | **exact**             | UBL's `IssueDate` is the publish date; OCDS's `release.date` is the same concept.                                                                                                                                                                                                                                             |
| 12  | `dueAt`                  | `cac:TenderingProcess/cac:TenderSubmissionDeadlinePeriod/cbc:EndDate` (+ `EndTime`)                                   | `tender.tenderPeriod.endDate`                                                                     | F02 BT-130                       | **exact**             | Submission deadline. UBL splits into `EndDate` + `EndTime`; we carry a single `Date` (full timestamp).                                                                                                                                                                                                                        |
| 13  | `estimatedValueLow`      | n/a (UBL has only a single `EstimatedOverallContractAmount`)                                                          | `tender.minValue.amount`                                                                          | F02 BT-271 (low range)           | **partial**           | Our model carries low/high; UBL collapses to single. On UBL emit we publish `min(low, high)` as the single value (or the `estimatedValueHigh` if low is null) and document the loss.                                                                                                                                          |
| 14  | `estimatedValueHigh`     | `cac:ProcurementProject/cac:RequestedTenderTotal/cbc:EstimatedOverallContractAmount`                                  | `tender.value.amount`                                                                             | F02 BT-27                        | **partial**           | Same single-vs-range concern as #13.                                                                                                                                                                                                                                                                                          |
| 15  | `currency`               | `currencyID` attribute on `cbc:EstimatedOverallContractAmount` or root `cbc:DocumentCurrencyCode`                     | `tender.value.currency`                                                                           | F02 BT-5                         | **exact**             | ISO 4217 three-letter code.                                                                                                                                                                                                                                                                                                   |
| 16  | `geography`              | `cac:ProcurementProject/cac:RealizedLocation/cac:Address/cac:Country/cbc:IdentificationCode` (or higher)              | `tender.deliveryAddresses` (extension)                                                            | F02 BT-727 (NUTS-coded, partial) | **partial**           | Country/region high-level only; finer subdivision detail (NUTS code, US state, etc.) goes through `opportunity_source_metadata` per slice 71.                                                                                                                                                                                 |
| 17  | `primaryCategoryCode`    | `cac:ProcurementProject/cac:MainCommodityClassification/cbc:ItemClassificationCode`                                   | `tender.items[].classification.id`                                                                | F02 BT-262 (CPV)                 | **exact**             | Resolved Q3. Single primary code per fragment; secondary CPVs spill to `fields.categoriesSecondary`. EU adapters populate as CPV; SAM.gov populates as NAICS or PSC (see #18).                                                                                                                                                |
| 18  | `primaryCategoryScheme`  | (carried as the `listID` attribute on `ItemClassificationCode`)                                                       | `tender.items[].classification.scheme`                                                            | F02 codelist `cpv`               | **novel**             | Codelist discriminator (`cpv` / `unspsc` / `naics` / `psc` / `tenant_custom`). UBL carries this on the codelist attributes; OCDS exposes it as a field. We surface it as a typed enum field for round-trip predictability — ours is novel only in that we elevate the discriminator from an attribute to a first-class field. |
| 19  | `regime`                 | n/a                                                                                                                   | n/a                                                                                               | n/a                              | **platform-internal** | Our routing dimension (`us_federal` / `uk_public` / `eu_oj` / `enterprise_global` / ...). Not emitted to UBL; canonical row carries it for downstream surfaces.                                                                                                                                                               |
| 20  | `sourceFamily`           | n/a                                                                                                                   | n/a                                                                                               | n/a                              | **platform-internal** | Coarse source taxonomy (`gov_portal` / `enterprise_procurement` / `open_contracting`).                                                                                                                                                                                                                                        |
| 21  | `procurementRegime`      | n/a                                                                                                                   | n/a                                                                                               | n/a                              | **platform-internal** | Explicit procurement regime tag (e.g. `us_far`, `uk_public_contracts_regulations_2015`).                                                                                                                                                                                                                                      |
| 22  | `sourceConfidence`       | n/a                                                                                                                   | n/a                                                                                               | n/a                              | **platform-internal** | Numeric confidence on source-record-to-canonical mapping. Not a procurement concept.                                                                                                                                                                                                                                          |

## UBL elements we deliberately omit

| UBL element                                                                                                       | Why we omit                                                                             | Where it lives in our model instead                                                                                                               |
| ----------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| `cac:TenderingProcess/cbc:UrgencyCode`, `cbc:ExpenseCode`, `cbc:PartPresentationCode`, `cbc:SubmissionMethodCode` | Process-tuning codes that downstream workflow doesn't act on.                           | `fields` spillover; promote to canonical column only when a consumer surfaces.                                                                    |
| `cac:TenderingProcess/cac:NoticeDocumentReference` (multiple)                                                     | Multiple-notice references (corrections, amendments) are a slice-02 versioning concern. | `opportunity_version` table (existing).                                                                                                           |
| `cac:TenderingProcess/cac:AdditionalDocumentReference`                                                            | Attachment manifest.                                                                    | `opportunity_source_metadata` `metadata_kind='attachment_manifest'` (slice 71).                                                                   |
| `cac:TenderingProcess/cac:ProcessJustification`                                                                   | Procedure-rationale text.                                                               | `fields.processJustification` (rare; promote on demand).                                                                                          |
| `cac:TenderingProcess/cac:EconomicOperatorShortList`                                                              | Restricted-procedure shortlist.                                                         | Future qualification slice; out of scope for canonical opportunity.                                                                               |
| `cac:TenderingProcess/cac:AuctionTerms`, `cac:FrameworkAgreement` (full structure)                                | Highly specific procedural content.                                                     | `contractVehicle` carries the label; full structure deferred.                                                                                     |
| `cac:ContractingParty/cac:BuyerProfile`                                                                           | Buyer profile metadata.                                                                 | Future Buyer crosswalk's territory; not on the opportunity fragment.                                                                              |
| `cac:ProcurementProject/cac:ContractExtension`                                                                    | Optional contract-extension shape.                                                      | `fields` spillover.                                                                                                                               |
| `cac:ProcurementProjectLot` (multi-lot)                                                                           | Multi-lot tendering.                                                                    | Out of scope for MVP fragment (canonical opportunity is single-lot in current model); a future canonical extension when slice 79 (OCDS) needs it. |

## Open questions to resolve

1. **Multi-lot opportunities.** UBL / OCDS / eForms all natively support
   multi-lot procurements (a single notice with N independent lots, each
   with its own value, deadline, evaluation criteria). Our current
   canonical opportunity is single-lot. Three viable answers:
   - **(A)** Keep canonical single-lot; an N-lot UBL document materialises
     N canonical opportunities. Loses the "these lots share a notice"
     relationship; gains simplicity.
   - **(B)** Add a `lotRef: string | null` to the fragment + a `notice_id`
     column to canonical opportunity, so N opportunities share a
     `notice_id` and are correlated. Carries the relationship; minor
     schema cost.
   - **(C)** Promote opportunity to a parent + lots schema with a new
     `opportunity_lot` table. Most faithful to UBL/OCDS; biggest schema
     change.
   - **Recommendation: (B).** Cheap, preserves the multi-lot relationship
     where it matters, and the existing `evaluation_criterion.lotRef`
     field already foreshadows this. Resolve before slice 79 (OCDS).

2. **`opportunityType` enum vs `mainProcurementCategory` strategy.**
   OCDS's `mainProcurementCategory` is a closed 4-value codelist
   (`goods` / `services` / `works` / `consultancyServices`). UBL doesn't
   pin a procurement-type codelist (the `ProcurementTypeCode` codelist is
   referenced but values are profile-specific). Our 7-value enum is
   richer (adds `mixed_goods_services`, `concession`, `framework`,
   `other`). Two viable answers:
   - **(A)** Stay with our 7-value enum + the hybrid `opportunityTypeCode`
     field (matches the evaluation-criterion `criterionType` resolution).
   - **(B)** Adopt OCDS's 4-value closed codelist as the canonical enum;
     `concession` / `framework` collapse into `services`; `mixed` collapses
     into the dominant kind.
   - **Recommendation: (A).** Same reasoning as the evaluation-criterion
     hybrid resolution: preserves source-system specificity without
     forcing non-EU adapters onto the EU codelist; round-trip lossless via
     `opportunityTypeCode`.

3. **`primaryCategoryCode` on the fragment vs. on a side table.** The
   canonical opportunity row today has no commodity-classification
   column. CPV / NAICS / PSC / UNSPSC codes are critical for downstream
   routing (qualification matching, regime detection, reviewer routing).
   Two viable answers:
   - **(A)** Add `primary_category_code` and `primary_category_scheme`
     columns on the canonical opportunity row (and on the fragment).
     Single primary; secondaries spill to `opportunity_source_metadata` /
     `fields.categoriesSecondary`.
   - **(B)** Keep classification entirely in
     `opportunity_source_metadata` with a new `metadata_kind='primary_classification'`.
     Avoids the schema change; readers must always join.
   - **Recommendation: (A).** Same reasoning as `cpvCode` on
     evaluation_criterion: ~5 LoC schema cost; meaningful downstream win
     (single-query "primary CPV/NAICS/PSC" filter). The `scheme`
     discriminator is the only field on the fragment that earns the
     **novel** classification; adding it now legitimises that novel-share
     spend.

## What this probe says about the rest of the work

- **Bucket distribution at ~94% UBL-aligned** on procurement-meaningful
  fields (16) confirms the layered-on-UBL discipline is the right shape
  for this fragment. Platform-internal fields (4) sit cleanly outside
  the UBL-emit surface and don't pressure the novel-share ceiling.
- **Slice 71's source-metadata extension is the right home for
  source-only fields** that don't have UBL counterparts (solicitation
  number, points of contact, attachment manifest). The probe doesn't
  re-litigate that decision.
- **The Buyer crosswalk** (`cac:Party` ↔ `Buyer` table) is the next
  most-load-bearing follow-on artifact. Today the fragment carries a
  lean `buyerRef` triple; the full UBL `Party` mapping happens at the
  orchestrator's buyer-resolution step.
- **`status` lifecycle cross-map** (canonical 5-value ↔ OCDS 6-value ↔
  UBL `TerminatedIndicator` + document type) is a smaller follow-on
  artifact, similar in shape to the `criterionType` cross-map.
- **Multi-lot promotion (Q1)** is a real schema decision and should be
  resolved before slice 79 (OCDS) since OCDS routinely publishes
  multi-lot tenders.

## Deltas — resolved 2026-05-08 (Q1=B, Q2=A, Q3=A)

| #   | Delta                                                                                                                                                                                                                                                                                                            | Status                   |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------ |
| 1   | `packages/structured-rfx-adapters/src/contract.ts` — replaced stub `CanonicalOpportunityFragment` with the typed shape; added `CanonicalOpportunityType`, `CanonicalOpportunityStatus`, `CanonicalCategoryScheme` enums.                                                                                         | **shipped** (2026-05-08) |
| 2   | `packages/structured-rfx-adapters/src/index.ts` — exported the new enum types.                                                                                                                                                                                                                                   | **shipped** (2026-05-08) |
| 3   | ADR 0026 §Amendments — recorded the typed shape and the three resolutions (Q1=B multi-lot correlation, Q2=A hybrid `opportunityType`+`opportunityTypeCode`, Q3=A `primary_category_code`+`primary_category_scheme`).                                                                                             | **shipped** (2026-05-08) |
| 4   | Slice 71 doc — adopted the typed shape; named SAM.gov-specific population (NAICS → `primaryCategoryCode` + `primaryCategoryScheme='naics'`, award-type → `opportunityTypeCode`, `noticeId` direct).                                                                                                              | **shipped** (2026-05-08) |
| 5   | Migration `0044_opportunity_ubl_alignment.sql` — adds `notice_id`, `lot_ref`, `primary_category_code`, `primary_category_scheme` columns to canonical `opportunity` plus partial indices for notice and primary-category lookups. Schema in `packages/db/src/schema/index.ts` updated; `_journal.json` extended. | **shipped** (2026-05-08) |
| 6   | `docs/crosswalks/canonical-opportunity-status-lifecycle-map.md` — status cross-map covering OCDS / UBL / SAM.gov / Ariba in both directions with round-trip fidelity guarantees.                                                                                                                                 | **shipped** (2026-05-08) |
| 7   | `docs/crosswalks/ubl-peppol-buyer.md` — Buyer ↔ `cac:Party` per-fragment crosswalk authored as a probe (its own three open questions surfaced for resolution).                                                                                                                                                   | **shipped** (2026-05-08) |

## Out of scope for this probe

- **Buyer crosswalk** (`cac:Party` ↔ `Buyer` table) — separate artifact;
  out of scope here.
- **Round-trip CI test** (TED F02 contract notice → canonical → UBL XML
  diff) — separate slice; this probe just asserts feasibility.
- **`opportunity_version`** mapping — versioning is a slice-02 concern;
  the canonical fragment is pre-version (a single snapshot).
- **Multi-lot full-promotion (Q1 option C)** — not in scope; the
  open-question recommendation is (B).
- **Source-metadata extension fields** (solicitation number, attachment
  manifest, etc.) — those have their own fragment type
  (`OpportunitySourceMetadataFragment`) per slice 71, not the canonical
  opportunity fragment.
