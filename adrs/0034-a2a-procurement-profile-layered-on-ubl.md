# ADR 0034: A2A Procurement Profile is Layered on UBL / Peppol Semantics, Not Parallel to Them

## Status

Accepted (2026-05-08). Companion to ADR 0032 §7 (the standards-watch /
translate-and-comply commitment) and ADR 0031 (capability claim graph
public read surface). First load-bearing artifacts: the typed
`CanonicalEvaluationCriterionFragment` shape in
`packages/structured-rfx-adapters/src/contract.ts` and the crosswalk
documents under `docs/crosswalks/`. ADR 0026's 2026-05-08 amendment is the
ADR-0026-local consequence of this decision; this ADR is the cross-cutting
posture that binds every future canonical fragment that has a UBL
counterpart.

## Context

ADR 0032 §7 commits the platform to a quarterly standards-watch (W3C VC,
eIDAS 2.0, OCDS response-side, MCP, SAP/Ariba) and a translate-and-comply
posture: when a recognised standards body publishes a format the M2M
roadmap depends on, the platform adopts the standard rather than persisting
with the transitional ADR 0027 JWS envelope. The standards-watch is
_general_ and points at multiple bodies; this ADR pins the _specific_
posture for **EU public procurement semantics** because the EU stack
(UBL → Peppol → eForms / ESPD) is uniquely load-bearing for the
government-first wedge and uniquely cohesive — UBL is ratified as ISO/IEC
19845, EU eForms (Implementing Regulation 2019/1780, mandatory across all
27 member states from October 2023) consumes UBL 2.x, and OpenPeppol BIS
profiles are the operational subset every contracting authority and
economic operator exchanges.

Two recent slice-level events forced the question:

1. **Slice 70 shipped with a stub `CanonicalEvaluationCriterionFragment`**
   (`{ perRowKey, fields: Record<string, unknown> }`). Slice 73 (Ariba) was
   set to define the real shape from Ariba's input alone. Doing that
   without a UBL crosswalk would have ossified an Ariba-shaped canonical
   fragment that, when the platform later attempts to emit EU-conformant
   output, would need to be retrofitted under standards-track timing
   pressure.
2. **The M2M strategy synthesis named UBL/Peppol overlap as a
   first-tier risk** for the prospective LF AAIF A2A Procurement Profile
   submission: "map field-by-field or get rejected in EU." A profile that
   introduces fields (`evaluation_criterion`, `requirement`,
   `capability_statement`) without a normative mapping back to UBL's
   `cac:TenderingCriterion`, `cac:RequirementResponse`, `cac:QualifyingParty`
   fails three ways — silent EU adoption failure, loud standards-review
   objection on duplicative ontology, and EU-anchored co-author refusal.

The 2026-05-07 probe (`docs/crosswalks/ubl-peppol-evaluation-criterion.md`)
validated that the mapping is feasible: the typed
`CanonicalEvaluationCriterionFragment` shape lands at ~92 % UBL-aligned
(5 exact / 6 partial / 1 novel out of 12 fields). Novel territory is well
below the 20–40 % red line that would trigger a duplicative-ontology
objection. The probe's conclusion was that the platform should _commit
to layering on UBL_ as a posture, not negotiate it case-by-case per
fragment.

This ADR records that commitment, the maintenance discipline it implies,
and its scope boundary against the broader translate-and-comply commitment
in ADR 0032 §7.

## Decision

### 1. Every canonical procurement fragment with a UBL counterpart is layered on UBL semantics

When a canonical fragment kind (today: `opportunity`, `requirement`,
`evaluation_criterion`, `questionnaire_row`; future: claim graph entries,
buyer credential scope, manifest variants) has a corresponding UBL element
or Peppol BIS profile structure, the canonical fragment's typed shape is
designed to alias UBL terms for **exact** field equivalents, narrow with a
documented constraint for **partial** equivalents, and explicitly carry
**novel** fields only with a written rationale. The 20 % novel-share ceiling
is the planning constraint; a fragment whose draft shape exceeds it is
re-scoped before it ships, not after.

The discipline is operational, not theoretical:

- **Exact** equivalents alias to UBL via JSON-LD `@context` entries when
  the fragment is emitted to wire / standards surfaces (AAIF Extension URI
  payloads, EU-conformant outputs).
- **Partial** equivalents document the narrowing constraint in the
  per-fragment crosswalk under `docs/crosswalks/`.
- **Novel** fields document the absence of a UBL equivalent and the
  reason. This is where the platform's IP lives — but it has to be a
  small, clearly-flagged set, not a parallel ontology.

ADR 0026 §3 ("the canonical model is the substrate; adapters do not
introduce new canonical types") is unchanged. This ADR adds a perpendicular
discipline: the canonical _fragment shapes_ themselves are designed to
round-trip to UBL / Peppol where a counterpart exists.

### 2. Pinned reference versions and re-pin trigger

The platform's UBL / Peppol crosswalks are pinned to specific versions and
the pinning is part of the contract:

- **UBL** — 2.3 (`docs.oasis-open.org/ubl/UBL-2.3.html`).
- **Peppol BIS ESPD** — 1.0, codelist `CriteriaTypeCode` listAgencyID
  `EU-COM-GROW`, listVersionID `1.0.2`.
- **EU eForms** — Implementing Regulation 2019/1780, codelist set at the
  current TED publication (`docs.ted.europa.eu/eforms/latest`).

Re-pin trigger is event-driven, not date-driven: when any of UBL 2.4,
Peppol BIS Pre-Award 4.0, ESPD-EDM 3.x, or an eForms codelist revision
lands, the next quarterly standards-watch tick (per ADR 0032 §7) re-runs
the diff against every crosswalk under `docs/crosswalks/` and produces a
re-pin commit. Crosswalk documents carry the pinned version in their
status header so the diff is mechanical.

### 3. Per-fragment crosswalks are the maintenance surface

Every canonical fragment with a UBL counterpart has a markdown crosswalk
under `docs/crosswalks/` that:

- Pins the UBL / Peppol versions (per §2).
- Cites the authoritative source URLs.
- Reports the bottom-line bucket distribution (exact / partial / novel)
  and the implied verdict.
- Lists the field-by-field mapping in a table with status and rationale
  per row.
- Names UBL elements deliberately omitted and where they live in our
  model instead.
- Records resolved open questions inline.
- Captures the deltas-to-commit checklist (and tracks shipped vs pending).

Shipped crosswalks (as of 2026-05-08):

- `docs/crosswalks/ubl-peppol-evaluation-criterion.md` — the
  `CanonicalEvaluationCriterionFragment` mapping (probe + resolutions).
- `docs/crosswalks/canonical-criterion-type-eu-com-grow-map.md` — the
  `CanonicalCriterionType` enum × EU codelist cross-map (forward and
  reverse directions).
- `docs/crosswalks/ubl-peppol-opportunity.md` — the
  `CanonicalOpportunityFragment` ↔ UBL `cac:TenderingProcess` /
  `cac:ProcurementProject` / OCDS `release.tender` / eForms F02 mapping
  (probe + Q1=B / Q2=A / Q3=A resolutions).
- `docs/crosswalks/canonical-opportunity-status-lifecycle-map.md` —
  status lifecycle cross-map (canonical 5-value ↔ OCDS `tenderStatus` ↔
  UBL `cbc:TerminatedIndicator` + document type ↔ SAM.gov ↔ Ariba) with
  round-trip fidelity guarantees.
- `docs/crosswalks/ubl-peppol-buyer.md` — `Buyer` ↔ UBL `cac:Party` /
  OCDS `parties[]` (probe + Q1=A resolution; Q2/Q3 deferred).
- `docs/crosswalks/ubl-peppol-requirement.md` — the
  `CanonicalRequirementFragment` ↔ UBL `cac:TenderingCriterionProperty`
  / OCDS Requirements extension `requirements[]` mapping (probe +
  Q1=A / Q2=A / Q3=B resolutions). Q1=A retired the
  `'questionnaire_row'` canonical kind; the original 2026-05-08
  ADR 0026 routing rule for property groups was revised in the
  third 2026-05-08 amendment.

Future crosswalks (committed but not yet drafted):

- `capability_claim` (ADR 0025) ↔ UBL `cac:QualifyingParty` /
  `cac:Certificate` / `cac:Evidence` (reuses the Buyer crosswalk's
  `cac:Party` mapping for `subjectParty`). **Phase B per ADR 0032 —
  waits for the counterparty gate.**
- `external_buyer_credential` scope (ADR 0029 / 0031) ↔ ESPD
  selection-criteria filter algebra and EU eForms exclusion grounds.
  **Phase B per ADR 0032 — waits for the counterparty gate.**

### 4. The crosswalk document is the source of truth; code mirrors it

When a crosswalk and a canonical fragment shape disagree, the canonical
fragment shape is the bug. Crosswalks are reviewed and approved
independently of the slice that ships the fragment shape; a slice that
introduces a fragment kind covered by a future crosswalk is allowed to
proceed only after the crosswalk has been authored or explicitly
deferred-with-rationale in the slice doc.

A future machine-readable artefact (e.g. a JSON-LD `@context` file
generator under a new `packages/ubl-peppol-mapping/`) will derive its
shape from these crosswalks; the human-readable markdown stays the source
of truth so that reviewers (internal and external — OpenPeppol Pre-Award
liaisons, OASIS UBL TC observers, AAIF reviewers) can audit the mapping
without a tool.

### 5. Maintenance commitment and out-of-band events

The maintenance commitment is small and non-negotiable:

- **Quarterly standards-watch re-pin diff** (per ADR 0032 §7), as above.
- **Out-of-band: any new canonical fragment kind triggers a crosswalk
  authoring task before the fragment ships.** A fragment whose typed
  shape lands without a crosswalk is treated as a slice-level discipline
  violation, surfaced at PR review.
- **Out-of-band: any change to a canonical fragment's typed shape that
  affects a mapped field re-runs the relevant crosswalk's verification.**
  The crosswalk's bucket distribution must remain at or below the 20 %
  novel-share ceiling.

### 6. Scope boundary against the AAIF / A2A Extension URI submission

This ADR is the precondition for the AAIF Extension URI submission, not the
submission itself. The submission is its own future slice (the
"standards-submission slice" referenced in ADR 0032's standards-watch and
in the crosswalk follow-ups). Specifically:

- This ADR commits to the crosswalk discipline _internally_ — the maps
  exist, the canonical shapes layer on UBL, the maintenance is
  committed.
- The AAIF / A2A Extension URI submission slice will publish the
  resulting JSON-LD `@context`, the supplementary normative documents,
  and the round-trip CI test against a real anonymised TED contract
  notice. Slice timing is gated by ADR 0032's Phase B counterparty
  commitment (≥ 1 named buyer-side organisation), not by this ADR.

The discipline in this ADR pays off whether or not the AAIF submission
ever happens: layered-on-UBL canonical shapes also unblock direct EU
adopter integrations (a contracting authority that consumes UBL XML
directly), Peppol-network participation if the platform ever offers it,
and EU customer trust signalling. The submission is a downstream
consequence.

### 7. What this ADR does **not** commit to

- **It does not commit to UBL as the wire format.** The platform's
  internal canonical model remains TypeScript objects + JSONB; UBL is
  an interop target, not the on-disk shape.
- **It does not commit to Peppol-network participation.** Joining the
  Peppol network is an operational and commercial decision, deferred
  until customer demand justifies the access-point overhead.
- **It does not commit to round-tripping non-UBL EU formats** (e.g.
  TED's pre-eForms XML schemas). UBL 2.x is the eForms-era substrate;
  TED's legacy schemas are not in scope.
- **It does not extend to non-EU procurement standards** by default. The
  ADR is EU-scoped because the EU stack is uniquely cohesive and
  uniquely load-bearing. SAP / Ariba and SAM.gov / FAR alignment are
  separate concerns: they layer on the same canonical shapes but do not
  obligate UBL emission or pin to UBL versions. ADR 0032 §7's
  standards-watch covers SAP-Ariba; this ADR does not.
- **It does not introduce a new canonical entity or extension table.**
  ADR 0002 and ADR 0026 §3 are unchanged.

## Consequences

### Direct (this ADR's load-bearing reason)

- The typed `CanonicalEvaluationCriterionFragment` is the first canonical
  fragment shape designed against this discipline; ADR 0026's 2026-05-08
  amendment ratifies it.
- Future fragment kinds (claim graph, buyer credential scope, manifest
  variants) draft their shape with a UBL crosswalk authored before the
  shape ships.
- The crosswalks under `docs/crosswalks/` become a reviewable artefact
  for OpenPeppol / OASIS UBL TC liaisons and AAIF reviewers without
  shipping any code.
- The `criterionTypeCode` and `cpvCode` fields in the
  `CanonicalEvaluationCriterionFragment` set the precedent for hybrid
  (canonical + verbatim source code) fields where round-trip fidelity
  matters.

### Forward-compatibility consequences

- **Adopting UBL 2.4 / Peppol BIS Pre-Award 4.0** is a documented diff,
  not a migration project. The crosswalk re-pin trigger names the work
  bound; the maintenance discipline keeps it small.
- **AAIF / A2A Procurement Extension URI submission** has the
  precondition cleared. Submission slice ships when ADR 0032's Phase B
  gate clears.
- **Direct EU contracting authority integration** — a future slice that
  emits UBL XML for a tenant's canonical opportunity / evaluation
  criteria has a documented, version-pinned mapping to draw from rather
  than reverse-engineering it case-by-case.
- **Peppol network participation** (deferred per §7) becomes an
  operational decision rather than a re-modelling project if customer
  demand justifies it.

### Costs and trade-offs accepted

- **Crosswalks must be maintained.** New fragment kinds + UBL / Peppol
  version drift create authoring work. The cost is bounded by §3's
  scope discipline (only fragments with UBL counterparts) and §5's
  cadence (quarterly diff + per-shape gate).
- **Slice authoring cadence picks up a precondition.** A slice that
  introduces a new canonical fragment kind needs its crosswalk
  authored or deferred-with-rationale before merge. The cost is real
  but is the same investment ADR 0026's "canonical model is the
  substrate" discipline already requires; this ADR widens the
  discipline from "respect the canonical model" to "respect UBL when
  there's a counterpart."
- **The 20 % novel-share ceiling is judgment-laden.** A fragment that
  lands at 25 % novel may be defensible; a fragment at 35 % may not.
  The crosswalk's per-row rationale is what makes the judgment
  reviewable.
- **EU-scoping is asymmetric** vs SAP / Ariba / SAM.gov posture.
  Standards-watch in ADR 0032 §7 is broader; this ADR is narrower and
  more committal. The narrower commitment pays off where EU adoption
  matters most (government-first wedge); broader commitments to other
  ecosystems are deferred until the same load-bearing case exists.
- **Hybrid fields (canonical enum + verbatim source code) cost one
  nullable column per axis.** The
  `CanonicalEvaluationCriterionFragment` adds two
  (`criterionTypeCode`, `cpvCode`); future fragments will add their
  own. Storage and indexing cost is bounded; readability and
  round-trip fidelity gains are the offsetting benefit.

### Things this ADR explicitly does not commit to

(See §7.)

## Alternatives considered

### Alt A — Parallel ontology

Rejected. Define canonical fragments from each adapter's input alone, then
build a one-way emit-to-UBL projection later. Cost: every later projection
discovers a different mapping; the canonical shapes drift further from UBL
each slice; the eventual translate-and-comply step (ADR 0032 §7) becomes
re-modelling rather than emission. The probe's bucket-distribution argument
makes this concretely worse — without the discipline, novel-share would
exceed 30–40 % within two slices.

### Alt B — UBL as the canonical wire format

Considered. Make UBL XML the on-disk and on-wire shape; serialise / parse
at every boundary. Cost: every canonical entity becomes a UBL document; the
TypeScript object model is a thin facade; debugging, indexing, and
performance become an EU-procurement problem rather than a software
problem. The platform has many non-UBL canonical concerns (DDQ rows, claim
graph, attestation history); forcing them through UBL is a category
violation. Rejected.

### Alt C — Crosswalks as a JSON-LD `@context` only, no markdown

Considered. Skip the human-readable crosswalk; ship only the
machine-readable `@context`. Cost: standards reviewers (OpenPeppol, OASIS,
AAIF) cannot audit the rationale; the per-row "why exact / why partial /
why novel" reasoning lives nowhere; future contributors cannot pick up the
mapping without re-deriving it. Rejected. (The `@context` ships eventually
per §4, generated from the markdown.)

### Alt D — EU-scope only, no broader standards posture

Rejected by ADR 0032 §7's existence. Standards-watch is broader than EU;
this ADR is the EU-specific narrowing. The broader posture is the parent
commitment.

### Alt E — Defer the discipline until the AAIF submission slice

Rejected. The probe surfaced that doing it later is much more expensive:
slice 73's `CanonicalEvaluationCriterionFragment` would have shipped from
Ariba's input alone, and the standards-submission slice would then
retrofit. Doing the work now (1 day's probe + this ADR + the typed shape)
saves the retrofit cost across every fragment that lands between now and
the eventual submission.

## Follow-on implications

- **Per-fragment crosswalks** (per §3) authored alongside future fragment
  shapes (`requirement`, `opportunity`, `questionnaire_row`,
  `capability_claim`, buyer-credential-scope).
- **Machine-readable `@context` generator** under a new
  `packages/ubl-peppol-mapping/` (or equivalent), derived from the
  markdown crosswalks. Ships with the AAIF submission slice.
- **Round-trip CI test** against a real anonymised TED F02 contract
  notice + ESPD response, covering the full canonical fragment set,
  validating lossless round-trip on the supported subset. Lives in the
  AAIF submission slice's acceptance criteria.
- **OpenPeppol Pre-Award liaison** as the preferred path for the
  per-fragment crosswalks' external review. Contractor liaison is the
  fallback if OpenPeppol cadence does not match the AAIF submission
  timeline.
- **Slice 79 (OCDS) and slice 73 (Ariba)** consume the typed
  `CanonicalEvaluationCriterionFragment` per ADR 0026's 2026-05-08
  amendment.

## References

- ADR 0026 — Structured RFx Ingestion as a First-Class Peer to Document
  Ingestion (the canonical-fragment substrate this ADR layers on).
- ADR 0032 — M2M Roadmap Pause: Counterparty-Gated Cryptographic Stack
  and Discovery Carve-Out (§7 standards-watch / translate-and-comply
  commitment, the parent of this ADR).
- ADR 0031 — Capability Claim Graph Public Read Surface (the M2M-roadmap
  surface that ultimately rides on this discipline once Phase B clears).
- ADR 0002 — Canonical Procurement Model (the substrate that this ADR
  preserves while adding the layered-on-UBL discipline).
- `docs/crosswalks/ubl-peppol-evaluation-criterion.md` — first per-fragment
  crosswalk; `CanonicalEvaluationCriterionFragment` ↔ UBL / Peppol ESPD.
- `docs/crosswalks/canonical-criterion-type-eu-com-grow-map.md` —
  `CanonicalCriterionType` enum × EU codelist cross-map.
- `packages/structured-rfx-adapters/src/contract.ts` — the typed shape
  this ADR pins.
- UBL 2.3 specification — https://docs.oasis-open.org/ubl/UBL-2.3.html
- Peppol BIS ESPD `CriteriaTypeCode` codelist —
  https://docs.peppol.eu/pracc/espd/codelist/CriteriaTypeCode/
- TED eForms codelists index —
  https://docs.ted.europa.eu/eforms/latest/reference/code-lists/index.html
- EU Implementing Regulation 2019/1780 (eForms mandatory date October 2023) — referenced for context; canonical text on EUR-Lex.
