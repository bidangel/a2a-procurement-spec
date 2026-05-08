# Cross-map — `CanonicalOpportunityStatus` ↔ OCDS / UBL / SAM.gov / Ariba

## Status

Authored 2026-05-08 alongside the typed `CanonicalOpportunityFragment`
shape and the opportunity crosswalk. Pinned to:

- OCDS 1.1.5 — `tender.status` closed codelist (`tenderStatus`).
- UBL 2.3 — `cac:TenderingProcess/cbc:TerminatedIndicator` (binary) plus
  the document-type lifecycle (CallForTenders → Tender → ContractAwardNotice).
- SAM.gov v2 Opportunities API — `type` plus `active` boolean.
- SAP Ariba Sourcing API — event status lifecycle (`Preview`,
  `Open`, `Pending Selection`, `Completed`, `Cancelled`).

Sources:

- OCDS `tenderStatus` codelist — https://standard.open-contracting.org/latest/en/schema/codelists/#tender-status
- UBL `cac:TenderingProcess` — https://www.datypic.com/sc/ubl23/e-cac_TenderingProcess.html

## Why a status cross-map

Each source ecosystem expresses opportunity lifecycle with its own
codelist. Naive emit-side translation (e.g. OCDS `complete` → "we mark
canonical as `closed`") loses the win-or-lose distinction and breaks
downstream qualification reporting. The canonical 5-value lifecycle
preserves the meaningful distinctions that downstream surfaces consume,
and the cross-map keeps round-trip emission deterministic.

## Canonical lifecycle

`CanonicalOpportunityStatus` (5 values, defined in
`packages/structured-rfx-adapters/src/contract.ts`):

| Value       | Semantics                                                                                |
| ----------- | ---------------------------------------------------------------------------------------- |
| `planned`   | Opportunity announced or pre-published; not yet open for tenders. (PIN, advance notice.) |
| `active`    | Open for tender submission; the deadline has not passed.                                 |
| `closed`    | Submission deadline has passed; awaiting evaluation / award.                             |
| `awarded`   | Contract has been awarded. (One or more awards published.)                               |
| `cancelled` | Procurement was cancelled or withdrawn before award; no contract exists.                 |

Lifecycle transitions are monotonic in the normal case (`planned →
active → closed → awarded`) with `cancelled` terminal-from-anywhere.
The orchestrator does not enforce these transitions; the source-of-truth
for state changes is the source adapter's idempotency-and-diff path.

## Forward direction — canonical → source

When emitting to a source-flavoured surface (OCDS export, eForms F03
Contract Award Notice, Ariba response submission, etc.), the adapter
picks the source code from the table below.

### Forward to OCDS `tender.status`

| Canonical   | OCDS        | Note                                                                                                                                                       |
| ----------- | ----------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `planned`   | `planning`  | OCDS distinguishes planned procurement (PIN) from active tendering.                                                                                        |
| `active`    | `active`    | 1:1.                                                                                                                                                       |
| `closed`    | `active`    | OCDS keeps `active` until the award is published; `closed` (canonical) → still `active` (OCDS) with a populated `tender.tenderPeriod.endDate` in the past. |
| `awarded`   | `complete`  | OCDS uses `complete` once the procurement is concluded with an award.                                                                                      |
| `cancelled` | `cancelled` | 1:1. (`unsuccessful` and `withdrawn` are also OCDS values; map to `cancelled` on reverse but emit as `cancelled` for canonical → OCDS.)                    |

### Forward to UBL `cac:TenderingProcess`

UBL's lifecycle is split between an indicator (`cbc:TerminatedIndicator`)
and the document type. The relevant emit pattern:

| Canonical   | UBL pattern                                                                                                                   |
| ----------- | ----------------------------------------------------------------------------------------------------------------------------- |
| `planned`   | Emit a `PriorInformationNotice` (PIN) document; `cbc:TerminatedIndicator=false`.                                              |
| `active`    | Emit a `CallForTenders` document; `cbc:TerminatedIndicator=false`.                                                            |
| `closed`    | `CallForTenders` document with `cac:TenderSubmissionDeadlinePeriod/cbc:EndDate` in the past; `cbc:TerminatedIndicator=false`. |
| `awarded`   | Emit a `ContractAwardNotice` document; `cbc:TerminatedIndicator=true` on the parent `CallForTenders`.                         |
| `cancelled` | `cbc:TerminatedIndicator=true`; emit `ContractAwardNotice` with `cbc:NoneAwarded=true` indicating cancelled procurement.      |

### Forward to SAM.gov

SAM.gov's structured opportunity exposes `type` (e.g. `Solicitation`,
`Combined Synopsis/Solicitation`, `Award Notice`, `Cancellation`) and an
`active` boolean. The canonical → SAM.gov mapping is asymmetric: BidAngel
is not a SAM.gov publisher, so this direction is documentation-only
(used by future write-side `AssemblyTarget` work, deferred per ADR 0022).
Recorded for completeness:

| Canonical   | SAM.gov `type`                                     | `active` |
| ----------- | -------------------------------------------------- | -------- |
| `planned`   | `Sources Sought` or `Special Notice`               | `true`   |
| `active`    | `Solicitation` or `Combined Synopsis/Solicitation` | `true`   |
| `closed`    | `Solicitation` (with response deadline past)       | `false`  |
| `awarded`   | `Award Notice`                                     | `false`  |
| `cancelled` | `Cancellation`                                     | `false`  |

### Forward to Ariba Sourcing event status

| Canonical   | Ariba               |
| ----------- | ------------------- |
| `planned`   | `Preview`           |
| `active`    | `Open`              |
| `closed`    | `Pending Selection` |
| `awarded`   | `Completed`         |
| `cancelled` | `Cancelled`         |

## Reverse direction — source → canonical

When ingesting source data, adapters set the canonical `status` per
the rules below.

### From OCDS `tender.status`

| OCDS                       | Canonical                                                                                                                | Note                                                                                                                                                          |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `planning`                 | `planned`                                                                                                                | 1:1.                                                                                                                                                          |
| `planned` (informal usage) | `planned`                                                                                                                | Treat as `planning`.                                                                                                                                          |
| `active`                   | `active` if `tender.tenderPeriod.endDate` ≥ now (or absent); `closed` if the deadline is past and no award is published. | OCDS does not distinguish "open for tender" from "deadline past, awaiting award"; the adapter computes the canonical distinction from `tenderPeriod.endDate`. |
| `complete`                 | `awarded`                                                                                                                | OCDS conflates "awarded" with "complete"; we resolve to `awarded`.                                                                                            |
| `cancelled`                | `cancelled`                                                                                                              | 1:1.                                                                                                                                                          |
| `unsuccessful`             | `cancelled`                                                                                                              | OCDS `unsuccessful` (no winning bid) collapses to canonical `cancelled` since no contract exists.                                                             |
| `withdrawn`                | `cancelled`                                                                                                              | OCDS `withdrawn` (notice retracted) collapses to canonical `cancelled`.                                                                                       |

The `unsuccessful` vs `cancelled` distinction is preserved on the
adapter side via `fields.ocdsTenderStatus` for forensic round-trip.

### From UBL document + indicators

| UBL pattern                                                                                      | Canonical   |
| ------------------------------------------------------------------------------------------------ | ----------- |
| `PriorInformationNotice` document                                                                | `planned`   |
| `CallForTenders` with `cbc:TerminatedIndicator=false` AND submission deadline ≥ now              | `active`    |
| `CallForTenders` with `cbc:TerminatedIndicator=false` AND submission deadline < now              | `closed`    |
| `ContractAwardNotice` with `cac:AwardedNotificationDescription` populated                        | `awarded`   |
| `ContractAwardNotice` with `cbc:NoneAwarded=true` OR `cbc:TerminatedIndicator=true` AND no award | `cancelled` |

### From SAM.gov

| SAM.gov                                                                               | Canonical                                                                              |
| ------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| `Sources Sought` / `Special Notice` (`active=true`)                                   | `planned`                                                                              |
| `Presolicitation` / `Solicitation` / `Combined Synopsis/Solicitation` (`active=true`) | `active`                                                                               |
| `Solicitation` (`active=false`) AND no award notice yet                               | `closed`                                                                               |
| `Award Notice`                                                                        | `awarded`                                                                              |
| `Cancellation`                                                                        | `cancelled`                                                                            |
| Justification / Fair Opportunity / Intent to Bundle                                   | `fields.samNoticeKind` only; canonical status driven by parent solicitation lifecycle. |

### From Ariba

| Ariba                                       | Canonical   |
| ------------------------------------------- | ----------- |
| `Preview`                                   | `planned`   |
| `Open`                                      | `active`    |
| `Pending Selection`                         | `closed`    |
| `Completed`                                 | `awarded`   |
| `Cancelled`                                 | `cancelled` |
| `Closed` (Ariba's own pre-completion state) | `closed`    |

## Round-trip fidelity guarantees

- **OCDS → canonical → OCDS**: lossy on the `unsuccessful` / `withdrawn`
  distinction — both map to canonical `cancelled` and emit back as
  `cancelled`. The original OCDS value is preserved in
  `fields.ocdsTenderStatus` for forensic round-trip on the same adapter.
  The lossy emit is acceptable because OCDS readers treat `cancelled`,
  `unsuccessful`, and `withdrawn` as terminal-no-contract on the
  consumer side.
- **UBL → canonical → UBL**: lossy on the `awarded` (with award) vs
  `cancelled` (no award) distinction in some `ContractAwardNotice`
  shapes; canonical preserves the distinction directly so emit is
  deterministic.
- **SAM.gov → canonical**: SAM.gov's pre-solicitation kinds (Sources
  Sought, Special Notice) all collapse to `planned`. The original
  SAM.gov `type` is preserved in
  `opportunity_source_metadata.metadata_kind='other'` with
  `metadata_payload.samNoticeType` for forensic round-trip.
- **Ariba → canonical → Ariba**: 1:1 reversible on all five canonical
  values.

## Open follow-ups

- **Status-transition validation.** The current model does not enforce
  monotonicity. A future slice could add a `status_transition` audit
  trail keyed on `(opportunity_id, prev_status, next_status,
transitioned_at)` so adapter-driven status changes are auditable.
  Out of scope for this cross-map.
- **Re-issued vs amended.** Some sources distinguish "this is a re-issue
  of the prior cancelled procurement" vs "this is an amendment to an
  active procurement." Today both flow through `opportunity_version`
  (slice 02). A future enhancement could add a `re_issue_of_opportunity_id`
  pointer.
- **Pre-award vs award-notice fragments.** The canonical opportunity
  represents the procurement notice; an award notice is currently
  treated as a status change on the same canonical row. If award notices
  grow their own canonical entity (separate `award` table), the cross-map
  splits across both nouns. Defer until ADR 0022 §4 (`AssemblyTarget`)
  surfaces the need.
