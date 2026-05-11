# Crosswalk audit — `procurement.*` clause vocabulary for the A2A `RejectionRecord` artifact

## Status

Draft (2026-05-12). Audit-only — written ahead of any commitment to the
`procurement.*` clause enumeration in the three-way `RejectionRecord`
proposal under discussion at
[a2aproject/A2A#1737](https://github.com/a2aproject/A2A/discussions/1737)
(Concordia / BidAngel / A2CN). Concordia ratifies the `mandate.*`
namespace prefix; this audit covers what BidAngel would be ratifying
under `procurement.*`.

The audit applies the ADR-0034 §1 layered-on-UBL discipline (exact /
partial / novel classification, 20 % novel-share ceiling) to a clause
enumeration rather than to a fragment's field shape. Vocabulary
sources surveyed:

- **UBL 2.3** — `cac:TenderResult`, `cac:TenderingProcess`,
  `cac:ProcessJustification`, `cbc:RejectionNote` (note: scoped to
  `OrderResponseSimple`, not procurement), `cac:WinningParty`,
  `cbc:TerminatedIndicator`.
- **OCDS 1.1.5** core schema (`awards.status` closed codelist) plus the
  **Bids extension** (`bids.details.status` codelist).
- **EU eForms** (Implementing Regulation 2019/1780) — codelists
  `non-award-justification` (BT-144), `exclusion-ground` (BT-67),
  `winner-selection-status`, `received-submission-type`.
- **Peppol BIS Pre-Award 1.0** — ESPD profile (consumes eForms
  `exclusion-ground` codelist; adds no independent bid-rejection
  vocabulary).

Re-pin trigger inherits from ADR-0034 §2.

Sources cited inline:

- UBL 2.3 `cac:TenderResult` — https://www.datypic.com/sc/ubl23/e-cac_TenderResult.html
- UBL 2.3 `cac:TenderingProcess` — https://www.datypic.com/sc/ubl23/e-cac_TenderingProcess.html
- UBL 2.3 `cac:ProcessJustification` — https://www.datypic.com/sc/ubl23/e-cac_ProcessJustification.html
- OCDS Bids extension codelist — https://extensions.open-contracting.org/en/extensions/bids/master/codelists/
- OCDS `awardStatus` codelist — https://standard.open-contracting.org/latest/en/schema/codelists/
- eForms `non-award-justification` codelist (BT-144) — `OP-TED/eForms-SDK/codelists/non-award-justification.gc`
- eForms `exclusion-ground` codelist (BT-67) — `OP-TED/eForms-SDK/codelists/exclusion-ground.gc`
- eForms `winner-selection-status` codelist — `OP-TED/eForms-SDK/codelists/winner-selection-status.gc`

## Bottom line

**AMBER — proceed with a scoped clause set, but the load-bearing
clarification below changes Erik's sketch shape.** The bid-level
rejection vocabulary in existing standards is thinner than the
evaluation-criterion / requirement / opportunity surfaces this profile
has crosswalked so far. The most cohesive existing taxonomy
(eForms `non-award-justification`, 10 codes) is **lot-level**, not
bid-level — it answers "why was the whole procurement not awarded?"
not "why was this specific bid rejected?". The cleanest per-bid
status discriminator (OCDS Bids extension's `bids.details.status`,
5 values: `invited` / `pending` / `valid` / `disqualified` /
`withdrawn`) lacks a structured **reason** axis; the reason currently
lives in unstructured `statusDetails` prose.

Implications for the A2A `RejectionRecord` design:

1. **`RejectionRecord` should be scoped to bid-level rejection only.**
   Lot-level non-award (procurement cancelled / no winner / insufficient
   funds) is a different artifact — eForms already names it
   distinctly via BT-144 + `winner-selection-status`. Conflating the
   two in one enum leads to the kind of duplicative-ontology objection
   ADR-0034 §1 calls out.
2. **The `procurement.*` clause enum primarily *structures the reason
   axis* that OCDS leaves unstructured.** Most clauses align to
   `bids.details.status='disqualified'` at the status level; the clause
   gives the reason that today lives in `statusDetails` free-text. This
   is layered-on, not parallel — but the partial-match share is high
   for the same reason.
3. **The eForms `exclusion-ground` codelist (24 codes) is the single
   strongest existing reason vocabulary.** Any `procurement.*` clause
   that maps to eligibility / exclusion should round-trip to a specific
   `exclusion-ground` sub-code; treating it as a single canonical
   clause loses information. Recommend a hybrid carrier (canonical
   clause + verbatim eForms exclusion-ground code) for that subspace,
   following the `criterionTypeCode` / `cpvCode` precedent from the
   evaluation-criterion crosswalk.

Bucket distribution on the recommended 8-clause set: **1 exact /
6 partial / 1 novel = 12.5 % novel**, comfortably below the
ADR-0034 20 % ceiling.

## Scope clarification — what `RejectionRecord` covers

This audit treats `RejectionRecord` as **bid-level**: an instance is
produced when a buyer-side agent rejects a specific economic
operator's submission. Out of scope (separate artifacts, flagged in
Findings):

- **Lot-level non-award.** The whole procurement (or a lot within it)
  closed without an award. eForms BT-144 + `non-award-justification`
  codelist owns this.
- **Bid withdrawal.** Bidder-initiated, not buyer-initiated. OCDS
  bids `withdrawn` covers it as a status; semantically not a rejection.
  Included here as `procurement.bid_withdrawn` only because the
  Concordia/A2CN sketch may want unified state telemetry; flagged as
  category-debatable in Findings.

## Proposed `procurement.*` clause enumeration

Eight clauses. The first three are Erik's sketch (refined); the last
five are additions justified below.

```ts
export type ProcurementRejectionClause =
  | 'procurement.requirement_unmet'
  | 'procurement.eligibility_failed'
  | 'procurement.specification_violated'
  | 'procurement.late_submission'
  | 'procurement.documentary_evidence_missing'
  | 'procurement.eligibility_sanctions_match'
  | 'procurement.bid_withdrawn'
  | 'procurement.bid_not_most_economically_advantageous';
```

## Field-by-field crosswalk

| # | Clause | UBL 2.3 | OCDS 1.1.5 (+ Bids ext.) | EU eForms | Classification | Recommended action |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | `procurement.requirement_unmet` | `cac:TenderingCriterion` describes the criterion (positive side); no structured per-bid "criterion-unmet" code. Narrative would land in `cac:TenderResult/cbc:Description`. | Bids ext. `bids.details.status='disqualified'` ("did not meet the qualification criteria"). Reason axis unstructured (`statusDetails` prose). | n/a structured at bid level. Adjacent: `received-submission-type` flags inadmissible submissions but at lot level. | **partial** | Keep. Aliases to OCDS `disqualified` status; this clause adds the reason taxonomy OCDS leaves to prose. Document in the per-clause emit table that on OCDS export the clause maps to `bids.details.status='disqualified'` + a fixed `statusDetails` stem. |
| 2 | `procurement.eligibility_failed` | n/a structured. ESPD criteria type `EXCLUSION_GROUNDS` (Peppol BIS `CriteriaTypeCode` codelist, EU-COM-GROW listAgencyID) sits on the *criterion* side, not the per-bid result. | Bids ext. `bids.details.status='disqualified'`. | **exclusion-ground** codelist (BT-67) — 24 codes spanning criminal-conviction, fiscal, professional-misconduct, and national-grounds families (`exg-crim-*`, `exg-pmt-*`, `exg-mis-*`, `exg-natl-*`). | **partial** | Keep, but add a sibling `eligibilityGroundCode` hybrid carrier (nullable string holding the verbatim eForms code like `exg-mis-misrepresent`). Same precedent as `criterionTypeCode` / `cpvCode` / `requirementTypeCode` in the existing crosswalks. The canonical clause stays one value; the codelist axis preserves round-trip fidelity. Consider renaming to `procurement.exclusion_ground_applied` to align with eForms naming — judgment call; "eligibility_failed" is more agent-readable, "exclusion_ground_applied" is more standards-aligned. |
| 3 | `procurement.specification_violated` | n/a structured. UBL splits this from `requirement_unmet` only implicitly (a criterion of type `TECHNICAL` failed vs `SELECTION`). | Bids ext. `bids.details.status='disqualified'`. Indistinguishable from #1 in OCDS without prose. | n/a structured. | **partial** | Keep, but call out the high overlap with #1 in the emit table. Differentiator: #1 fires when a criterion's pass/fail axis fails (yes/no); #3 fires when technical/format/spec content of the bid is non-conformant (regulated by spec doc rather than evaluation rubric). Tight definition is required to avoid two-codes-one-meaning. |
| 4 | `procurement.late_submission` | n/a. UBL `cac:TenderSubmissionDeadlinePeriod/cbc:EndDate` defines the deadline (positive side); a late bid is simply rejected with no structured code. | Bids ext. `bids.details.status='disqualified'`. | n/a structured at bid level. `received-submission-type` codelist categorises submissions but doesn't carry a "late" code in the current version. | **partial** | Keep — common rejection reason in every jurisdiction; absent from standards because the deadline itself is structured but its violation isn't. Round-trip is OCDS-only (`disqualified` + prose). |
| 5 | `procurement.documentary_evidence_missing` | `cac:TemplateEvidence` describes the *required* evidence (positive side); no structured "evidence missing" rejection element. | Bids ext. `bids.details.status='disqualified'`. | exclusion-ground code `exg-mis-misrepresent` — "Misrepresentation, withheld information, **unable to provide required documents** or obtained confidential information". | **partial** | Keep. Strong eForms alignment via `exg-mis-misrepresent`; round-trip emits to OCDS `disqualified` + (when applicable) eForms exclusion-ground code via the same `eligibilityGroundCode` carrier from #2. |
| 6 | `procurement.eligibility_sanctions_match` | n/a. | Bids ext. `bids.details.status='disqualified'`. | exclusion-ground codes `exg-crim-*` (criminal-conviction family, 6 codes including `exg-crim-corrpt`, `exg-crim-laund`, `exg-crim-terror`) and `exg-mis-sanction` ("Early termination, damages, or other comparable sanctions") and `exg-mis-unrel-sec` ("Lack of reliability to exclude risks to the security of the country"). | **partial** | Keep — separate from #2 because sanctions matches typically trigger different operator-side audit/escalation paths than generic eligibility failures (and we want telemetry to discriminate them). The codelist axis (`eligibilityGroundCode` from #2) is the round-trip carrier. |
| 7 | `procurement.bid_withdrawn` | n/a structured. (UBL captures the positive side via `cac:WinningParty` / `cac:TenderResult` only.) | Bids ext. `bids.details.status='withdrawn'` — "The submitted bid or expression of interest was withdrawn by the tenderer(s)." | n/a structured. | **exact** | Category-debatable: bidder-initiated, not buyer-initiated, so semantically not a "rejection." See Findings; recommendation is to **drop** from `procurement.*` clauses and route to a separate `BidStatusRecord` (or whatever the three-way protocol names the lifecycle artifact). If kept, exact alignment to OCDS `withdrawn`. |
| 8 | `procurement.bid_not_most_economically_advantageous` | n/a. UBL encodes the winning side (`cac:WinningParty`, `cac:TenderResult/cac:AwardedTenderedProject`) but has no first-class element for "valid but unsuccessful tenderers" carrying a per-bid loss reason. The narrative lives in `cac:TenderResult/cbc:Description` and the evaluation report doc. | n/a. OCDS Bids `status='valid'` says the bid qualified; there is no per-bid "lost on scoring" status. The award level offers `awards.status='unsuccessful'` but at the award unit, not the loser-bid level. | n/a. `winner-selection-status='selec-w'` says a winner was chosen for the lot but doesn't carry per-loser reasons. Bid-level loss data lives in evaluation reports (`evaluationReports` documentType) as unstructured text. | **novel** | Keep — this is genuine novel territory. MEAT (Most Economically Advantageous Tender) decisions are the dominant award model in EU public procurement, and "your bid was valid but scored lower than the winner's" is the most common reason a supplier-side agent receives a rejection. None of the four standards model it at the per-bid level; we should. Document the absence-of-equivalent in the per-clause rationale; on OCDS emit this is the bid-side annotation that accompanies the corresponding `awards.status='active'` (winner) decision. |

## Standards elements deliberately omitted

| Element / code | Why we omit from `procurement.*` clauses | Where the concept lives in our model instead |
| --- | --- | --- |
| UBL `cbc:RejectionNote` | Scoped to `OrderResponseSimpleType` (e-commerce order rejection), not procurement. Despite the name, this is a category-violating false friend — adopting it would inherit ordering-domain semantics into procurement. | Use OCDS Bids `bids.details.statusDetails` for the human-readable narrative; the structured reason axis is the `procurement.*` clause enum. |
| eForms `non-award-justification` (BT-144) | Lot-level, not bid-level. Belongs in a separate artifact (`NonAwardRecord` or equivalent) — see Findings. | Future sibling artifact crosswalk. |
| eForms `winner-selection-status` | Lot-level lifecycle indicator (`clos-nw` / `open-nw` / `selec-w`), not a bid-level rejection reason. | `CanonicalOpportunityStatus` lifecycle (already crosswalked in `canonical-opportunity-status-lifecycle-map.md`). |
| UBL `cbc:TerminatedIndicator` | Binary; can't carry rejection-reason granularity. | Already in the opportunity-status crosswalk's UBL emit table. |
| UBL `cac:ProcessJustification/cbc:PreviousCancellationReasonCode` and `cbc:ProcessReasonCode` | These justify the *choice of procurement procedure* (e.g. why direct award vs open competition), not bid-level rejection. | Out of scope for this artifact; would belong on the canonical opportunity fragment if we ever needed it. |
| OCDS `awards.status='unsuccessful'` and `awards.statusDetails` | Award-unit level ("this award could not be successfully made"), not bid level. Maps to a different artifact. | Future award-status crosswalk if needed; for now lives implicitly in `CanonicalOpportunityStatus='cancelled'`. |
| OCDS Bids `bids.details.status='invited'` and `pending` | Pre-decision states (invitation issued / submitted-not-yet-evaluated); not rejection reasons. | Bid lifecycle (separate from rejection). |

## Open questions to resolve

1. **`procurement.bid_withdrawn` — keep in `RejectionRecord` or split
   into a separate lifecycle artifact?** Bidder-initiated withdrawal is
   semantically distinct from buyer-initiated rejection. OCDS Bids
   treats both as `bids.details.status` values (one codelist, mixed
   semantics), which is a known wart of the extension. The three-way
   protocol can either replicate the OCDS pattern (cleaner round-trip,
   muddier semantics) or split (cleaner semantics, additional artifact
   type). Recommendation: **split** — `RejectionRecord` is buyer-side;
   add a separate `BidStatusRecord` or reuse Concordia's mandate
   lifecycle artifact for withdrawal. Drops `bid_withdrawn` from the
   `procurement.*` clause enum, lowering count to 7 and increasing
   novel-share to ~14 % — still well below ceiling.

2. **`eligibility_failed` vs `exclusion_ground_applied` naming.** The
   eForms-aligned name (`exclusion_ground_applied`) round-trips
   verbatim; the agent-readable name (`eligibility_failed`) is more
   intuitive in the supplier-side agent's UX. Recommendation: keep
   `eligibility_failed` as the canonical clause and document the
   round-trip mapping in the per-clause emit table — same trade-off
   already accepted in `CanonicalCriterionType` (canonical enum
   distinct from EU-COM-GROW `CriteriaTypeCode`).

3. **Hybrid `eligibilityGroundCode` carrier — fragment-level or
   clause-level?** Adding a verbatim eForms `exclusion-ground` code is
   only meaningful for clauses #2 (`eligibility_failed`), #5
   (`documentary_evidence_missing`), and #6
   (`eligibility_sanctions_match`). Two viable shapes:
   - **(A)** One nullable string on the `RejectionRecord` artifact;
     populated only when the clause is in the eligibility family.
   - **(B)** A typed union: each eligibility-family clause carries the
     codelist sub-axis as a required typed field; non-eligibility
     clauses don't carry the field.
   - **Recommendation: (A).** Simpler; matches the existing hybrid-field
     pattern across the other four crosswalks (`criterionTypeCode`,
     `cpvCode`, `requirementTypeCode`, `opportunityTypeCode` — all
     nullable single-string carriers). The discipline-cost of a typed
     union for one sub-axis isn't worth it.

4. **Single-string `clauseDescription` text alongside the enum?** OCDS
   leaves the rejection reason as unstructured `statusDetails`; if we
   alias to that on emit, we should also accept *receiving* prose from
   buyer-side adapters that didn't classify into the clause enum. The
   pragmatic answer is yes — keep a nullable `clauseDescription`
   string for prose carryover; populated when the source provides
   prose, nullable otherwise. Same partial-match policy as
   `requirement.body` and `requirement.evidenceNeededSummary` in the
   requirement crosswalk.

## Findings

### Layer-on candidates the sketch should rename / align

- **`eligibility_failed` (#2)** is essentially "eForms
  `exclusion-ground` codelist code applied at the per-bid level."
  Recommendation: keep the agent-readable name, add the verbatim
  codelist carrier — round-trip fidelity is preserved without
  trading away the readable enum. (Standards reviewers care about the
  carrier; supplier-side agents care about the enum.)
- **`documentary_evidence_missing` (#5)** maps cleanly to eForms
  `exg-mis-misrepresent` — same hybrid-carrier path as #2.
- **`bid_withdrawn` (#7)** is an exact OCDS `withdrawn` alias and
  arguably shouldn't be in this enum at all (Open Question 1).

### Genuine novels (and why)

- **`bid_not_most_economically_advantageous` (#8)** is the one
  unambiguous novel. None of UBL / OCDS / eForms / Peppol BIS Pre-Award
  carries a per-bid "lost on evaluation scoring" code. The closest
  proxies are inverse (the *winner's* award status is `active`; the
  *losers'* state is mute) or indirect (evaluation reports as a
  document type, unstructured). The clause is novel in the same sense
  that `requirementType='security'` and `criterionType=*` were novel
  in earlier crosswalks — it serves classification/routing on our
  side, captures a real and very common rejection reason, and has no
  UBL / OCDS / eForms counterpart by design (the standards model the
  winner positively, not the losers negatively).

### Rejected-bid concepts present in standards but absent from the sketch

- **The eForms `exclusion-ground` codelist's 24 sub-codes** are not
  individually surfaced — we collapse them under three clauses (#2,
  #5, #6) plus the hybrid carrier. Acceptable trade-off: the canonical
  clause stays narrow; the codelist axis preserves the full vocabulary
  on round-trip. The alternative (24 `procurement.*` clauses) would
  blow up the enum and ship a near-parallel ontology.

- **eForms `received-submission-type` codelist** (codes for tender
  categories like inadmissible / abnormally-low / etc.) is adjacent
  and lot-level. The "abnormally low tender" rejection reason is a
  real EU-procurement-specific concept absent from this sketch; we
  could add `procurement.abnormally_low_tender` later if a
  buyer-side adapter surfaces it. Not in the initial v0.1 enum.

### Lot-level non-award reasons — separate artifact

This is the load-bearing finding: the most cohesive existing
codelist in this space (`non-award-justification`, 10 codes) is
**not bid-level**. It captures *why the whole procurement did not
result in an award*: `all-rej` (all tenders inadmissible),
`chan-need` (buyer's needs changed), `ins-fund` (insufficient
funds), `no-rece` (no submissions received), `no-signed` (winner
refused to sign), `one-admis` (only one admissible — competition
failed), `rev-body` (review body decision), `rev-buyer` (buyer
review reversal), `tch-pr-error` (technical/procedural error),
`other`. These are exact-match codes for a different artifact —
recommend the three-way protocol add a sibling
`NonAwardRecord` (or `LotNonAwardRecord`) artifact carrying a
parallel `procurement_lot.*` clause enum that round-trips to
eForms BT-144 1:1. Exact-share for that enum would be 9/10
(everything except `other`) — much higher than this audit's
bid-level enum, because the lot-level vocabulary has a single
cohesive standards owner.

### Novel-share for `procurement.*` clause vocabulary: 12.5 % (1 of 8 clauses is a genuine novel)

If Open Question 1 resolves to drop `procurement.bid_withdrawn` from
the enum (moves to a separate lifecycle artifact), the recomputed
distribution is **0 exact / 6 partial / 1 novel = 14.3 % novel** on
a 7-clause set. Still comfortably below the ADR-0034 §1 20 % ceiling
in either resolution.

## What this audit says about next work

- **The clause enum can be locked at 7–8 values** with high
  confidence; the round-trip story is documented; the novel-share is
  within ceiling.
- **The hybrid `eligibilityGroundCode` carrier** (Open Question 3,
  resolution A) is the single typed-shape decision that needs to land
  in the artifact's TypeScript / JSON-Schema definition. Same pattern
  as four prior crosswalks.
- **A sibling `NonAwardRecord` artifact** for lot-level non-award is
  the strongest follow-on — exact-share would be ~90 % against eForms
  BT-144, making it the highest-leverage standards-aligned artifact
  the three-way protocol could add after this one. Recommend filing as
  a discussion comment under
  [a2aproject/A2A#1737](https://github.com/a2aproject/A2A/discussions/1737)
  noting the bid-level / lot-level distinction.
- **The `procurement.abnormally_low_tender` candidate clause** is
  parked pending a buyer-side adapter that surfaces it.
- **Round-trip CI test** against an anonymised TED F03 (Contract Award
  Notice) fixture's unsuccessful-supplier section is a candidate for
  the AAIF submission slice's acceptance criteria, once the artifact
  ships.

## Out of scope for this audit

- **JSON-LD `@context` for `RejectionRecord`** — produced from this
  crosswalk once the enum is locked; aliases the OCDS Bids
  `bids.details.status='disqualified'` term and the eForms
  `exclusion-ground` codelist term where applicable.
- **JSON Schema for `RejectionRecord`** — derived from the clause enum
  + the hybrid carrier; deferred until the three-way protocol's
  artifact-shape ratification stage.
- **`NonAwardRecord` lot-level artifact** — separate crosswalk;
  flagged above.
- **`BidStatusRecord` lifecycle artifact** (depends on Open Question 1
  resolution) — separate crosswalk if the split is accepted.
- **A2CN-side ratification of any namespace prefix** — out of
  BidAngel's lane; this audit only commits the `procurement.*`
  namespace.
