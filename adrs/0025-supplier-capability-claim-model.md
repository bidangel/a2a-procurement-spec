# ADR 0025: Supplier Capability Claim as a First-Class Noun, Distinct from Content Asset

## Status

Proposed. Authored alongside slice 68 (`docs/slices/slice-68-capability-claims-schema-approval-and-freshness.md`); slice 68 is the first implementation of the principles recorded here. Expected to be updated by the slice-68 implementation as edge cases are discovered.

## Context

The platform's evidence model today is `content_asset` (slice 07): a tenant-owned, approval-tracked, freshness-tracked file or text snippet that proposal teams cite when answering RFP requirements. Slices 18–20 added the Answer Library on top of `content_asset`. Slice 24 cites them from the compliance matrix. Slice 65's submission package references them through the evidence manifest. The model is well-shaped for "here is a piece of evidence; we can cite it from a draft."

It is the wrong shape for "here is a _claim_ the supplier makes about its capabilities, backed by one or more pieces of evidence, valid in certain jurisdictions, mapped to certain procurement categories, approved by a specific role, and valid until a specific date." A claim is not an asset. The asset is the proof; the claim is the assertion.

The strategic synthesis frames the requirement directly:

> **Supplier Capability Passport**: a tenant-controlled, machine-readable, evidence-backed profile of what the seller can credibly claim, where, for whom, under what approvals, and with what supporting proof.
>
> A claim is `(claim, evidence_refs[], jurisdiction, category_mappings[], approval, valid_until, freshness, conflicts)`.

ADR 0022 §2 already reserved a `claims[]` slot in the submission-package manifest. That slot has been empty in every package the platform has produced since slice 65, and the schema commits to a forward-compatible shape ("claim, evidence, freshness, approval entries"). Filling that slot — and, later, exposing it to buyer agents (ADR 0029, slice 82) — requires a noun the current data model does not have.

The decision is whether to extend `content_asset` with claim semantics or to introduce a separate `capability_claim` entity. This ADR records the latter and pins the durable consequences. The decision is warranted because almost every M2M-roadmap slice that touches the seller-side manifest cites the claim noun, and getting the noun wrong means re-shaping the manifest contract that ADR 0022 §2 froze for forward compatibility.

The strategy note is the explicit framing:

> Build the Evidence Graph that outputs signed trust packets. […] This becomes extremely valuable when buyer agents need to distinguish real evidence from agent-generated fluff.

The capability claim is the structured truth; the content asset is the evidence; the trust packet (slice 75 / ADR 0027) is the signed extract. Each is a different noun. This ADR establishes the first.

## Decision

### 1. The capability claim is a first-class, tenant-owned noun, distinct from content asset

A new `capability_claim` row is introduced. It is tenant-owned (ADR 0005), independent of any pursuit, opportunity, or buyer. It is the **assertion** the supplier makes about its capabilities. The `content_asset` row remains the **proof**. The two are linked through a many-to-many junction (§3).

The decision to introduce a separate noun rather than extending `content_asset` is load-bearing. The alternatives considered (Alt A) make this concrete; the short version is:

- A claim has structure that no asset has: jurisdiction, category mappings, validity window, kind-specific structured fields, and approval that is _independent of_ the underlying evidence's approval.
- A claim and an asset have different lifecycles. An asset can be uploaded and approved long before any claim cites it; a claim can be drafted and approved against existing approved evidence; a claim can be superseded without touching the evidence; an asset can be retired without retiring the claims that cite it (the claims become evidence-stale, which is a freshness signal, not a structural failure).
- The cardinality is many-to-many: one claim ("we maintain SOC 2 Type II compliance") cites many evidence pieces (audit report, scope statement, attestation letter); one evidence piece (the same audit report) backs many claims ("SOC 2 Type II compliant," "annually externally audited," "operates a SOC-controlled environment"). Conflating the two nouns forces 1:1 or invents a parallel junction inside the asset model.
- The noun in the manifest is "claim" (ADR 0022 §2). The noun on a buyer-agent's query interface is "claim." The mismatch between the user-facing noun and the data model's noun creates exactly the kind of drift this codebase generally avoids.

`content_asset` is not modified by slice 68. It continues to mean what it means. The claim sits adjacent to it.

### 2. Five claim kinds, fixed at creation, structured kind-specific fields via typed jsonb

A claim's `kind` is one of:

- **`certification`** — third-party attested credentials (SOC 2, ISO 27001, FedRAMP, ISO 9001, sector-specific licences). Carries structured fields: certifying authority, certificate identifier, attested scope, valid_from, valid_until, attestation kind (Type I / Type II / equivalent).
- **`past_performance`** — completed engagements offered as evidence of capability. Carries structured fields: customer name, customer sector, engagement value range, period of performance, role (prime / sub), reference contact gating ("contact only with permission" / "approved as public reference" / "approved as anonymised").
- **`capability_statement`** — a free-text capability assertion ("we operate 24×7 SOC-controlled environments in three regions"). Carries no kind-specific structured fields beyond the prose claim and a category set; intended for capabilities that do not naturally have a third-party attestation.
- **`jurisdiction_coverage`** — geographic / regulatory scope assertions ("authorised to deliver to US federal customers under SAM-registered status," "VAT-registered in Germany"). Carries: jurisdiction, regulatory regime, registration identifier (where applicable), valid_from, valid_until.
- **`other`** — escape hatch for tenant-specific claim shapes that do not map to the above. Carries no kind-specific structured fields. Tenants are encouraged to request promotion of common `other` patterns to a dedicated kind via product feedback.

The kind is **fixed at creation**. Mutating the kind changes the meaning of the structured fields and the conflict semantics; treating a `certification` row as a `past_performance` row retroactively would break attestation history and any cited references. If a claim's nature changes, the operational pattern is to supersede it (§6), not edit the kind.

Kind-specific structured fields live in a typed `kind_specific_data` jsonb column on `capability_claim` in MVP, with per-kind validation enforced at the service boundary by Zod schemas. The MVP-jsonb-with-typed-validation approach is a deliberate scope cap: promoting any of the per-kind structured fields to dedicated columns or extension tables is a follow-up that does not require an ADR. Keeping them as typed jsonb today avoids five extension tables before any tenant has used the feature in anger.

### 3. Claim ↔ evidence is many-to-many

A junction table `capability_claim_evidence` links claims to `content_asset` rows. Each link carries:

- `capability_claim_id`, `content_asset_id` (composite PK)
- `evidence_role` — enum: `primary | supporting | corroborating`. The primary evidence is what the claim most directly relies on; supporting and corroborating are subsidiary.
- `link_note` — free text, optional, why this evidence backs this claim
- `linked_at`, `linked_by_user_id`

The cardinality is enforced at the schema level. A claim with no linked evidence is permitted in `draft` status (the claim is being authored before its evidence is attached) but is rejected at the `pending_approval → approved` transition by the reviewer workflow: a claim cannot be approved without at least one linked evidence row, and the link must point at a `content_asset` whose `approvalState='approved'`.

This is the load-bearing approval invariant: **an approved claim always has at least one linked, approved evidence asset.** It is not the only invariant (§4 covers freshness, §6 covers versioning); it is the one that distinguishes a claim from a free-floating capability statement.

### 4. Claim approval is independent of evidence approval; freshness is computed

A claim's `approvalState` (`draft | pending_approval | approved | rejected | retired`) is independent of the `approvalState` on its linked content_assets. An approved claim can sit on top of evidence that was approved months earlier; an evidence asset can be re-approved (a new revision uploaded, re-reviewed) without re-approving every claim that cites it; a claim can be approved and then later become _evidence-stale_ if the underlying evidence is retired or expires.

Freshness is the dimension that captures the composed reality. A claim's `freshnessState` (`current | aging | stale | expired`) is **computed, not stored**, from four inputs:

1. **Calendar validity** — if `valid_until` is set and is in the past → `expired`. If within 60 days of `valid_until` → `aging`. (Thresholds are tenant-tunable in a future slice; pinned at 60d for MVP.)
2. **Evidence freshness floor** — the minimum `freshnessState` across all linked evidence. If any linked evidence is `expired`, the claim is at minimum `stale`. If any is `stale`, the claim is at minimum `aging`. Evidence freshness comes from `content_asset.freshnessState` (slice 53/54) and from per-asset expiry where present.
3. **Last review** — if more than 365 days since `last_reviewed_at` (set on approval and on explicit re-review) → at minimum `aging`.
4. **Linked evidence approval** — if any linked evidence has `approvalState != 'approved'` (e.g. retired), the claim is at minimum `stale`.

The composition rule is **lowest-bucket-wins**: the claim's freshness is the worst (least-fresh) of the four inputs.

Freshness is computed on read by a single helper (`computeClaimFreshness`) and surfaced through the claim's API shape and on every materialised view (the future buyer-agent query sandbox in slice 77, the manifest population in slice 69). Computed-not-stored is the right call because the inputs change independently of the claim row: a content_asset's freshness can shift overnight (an evidence document past its review date) without any write to the claim row itself. Storing the freshness on the claim would require a sweep job; computing it on read is cheap (the inputs are joined in the same query) and always-current.

A future slice may add a denormalised cache column for query-performance reasons; that cache would be invalidated on any input change. ADR 0025 commits only to the computation rule, not to the cache.

### 5. Conflicts are manually marked, not auto-detected

Two claims may contradict each other ("we serve healthcare clients" plus "we do not operate in regulated industries"). The platform does **not** auto-detect contradictions in MVP. It provides a `capability_claim_conflict` mutual junction so an admin can explicitly mark a pair of claims as conflicting, with a `conflict_note` explaining why.

The reasons for marking-not-detection in MVP:

- Auto-detection requires structured semantic understanding of the claim text or kind-specific fields. The claims that most need conflict detection (capability_statement free text) are the ones where auto-detection is hardest.
- A false positive (the engine declares a conflict that is not real) is more damaging than a false negative: the operator second-guesses every approved claim and the policy surface becomes noisy.
- Manually-marked conflicts are precise and explainable. They are also the right input to a future auto-suggestion engine that learns from prior manual markings.

Marked conflicts surface in three places: the claim editor (a banner: "this claim conflicts with X — review before approving"), the future buyer-agent query sandbox (slice 77, "your passport contains internally-conflicting claims; here is the pair"), and (eventually) the manifest's `claims[]` payload as a `conflictsMarked: bool` flag per claim entry (slice 69 will decide whether the marked-conflicting peer's id is included or only the boolean flag, balancing transparency against leaking internal claim structure).

### 6. Versioning is via supersession, not in-place edits, for material changes

A claim's _minor_ fields (display label, description prose, link notes, category re-tagging that does not change semantic meaning) can be edited in place. A claim's _material_ fields (the prose claim itself, kind, kind-specific structured data, jurisdiction, valid_until, evidence link set composition) are **immutable on an approved claim**. To change them, the operator supersedes:

1. Create a new claim with the new fields (`status='draft'`, with `supersedes_claim_id` pointing at the prior approved claim).
2. Submit for approval and approve.
3. On approval of the successor, the prior claim transitions to `retired` automatically and gets `superseded_at`, `superseded_by_claim_id` set.

The supersession chain is preserved indefinitely. A historical attestation citing claim version 1 (now retired, superseded by version 2) remains interpretable: the chain `v1 → v2 → v3` is queryable, and the manifest's per-claim entry records the claim id at the moment of assembly (slice 69), so the audit answer "what was the claim shape when this submission package was assembled" is precise.

The reason for supersession-rather-than-edit on material fields is the same reason ADR 0008 codifies advisory-vs-approved separation at the workflow level and ADR 0018 codifies audit-trail immutability: an approved claim is a commitment. Mutating it in place would mean a buyer reading a one-month-old submission package's manifest and a buyer querying the live capability surface today see the same claim id with different semantics. Supersession makes the claim id stable per-revision and the chain queryable.

What counts as "material" is enumerated explicitly on the schema (a list of column names whose UPDATE on an approved row is rejected by a trigger or service-layer guard). This list is part of the load-bearing contract and should be amended only via ADR.

### 7. Category mappings are many-to-many across multiple schemes

A claim is tagged with category codes from one or more schemes. MVP supports two standard schemes plus tenant-defined custom categories:

- **NAICS** — North American Industry Classification System codes. The most common procurement category surface in US federal and state RFPs.
- **UNSPSC** — United Nations Standard Products and Services Code. The most common in enterprise procurement (SAP/Ariba).
- **`tenant_custom`** — tenant-defined category labels for internal organisation ("our SOC 2 cert is part of our 'Security & Compliance' bundle").

A `capability_claim_category` junction carries `(claim_id, scheme, code, label, mapped_at, mapped_by_user_id)` with composite uniqueness on `(claim_id, scheme, code)`. The scheme is an enum; adding a third standard scheme (e.g. CPV for EU procurement) is an enum extension, not a schema change.

The category mappings are the seller's _self-declared_ mappings. They are not authoritative — a future slice may cross-check them against the canonical taxonomies (e.g. flag that a claimed NAICS code is invalid or has been retired) — but in MVP they are accepted as written. The future buyer-agent sandbox uses the mappings as the discoverability surface ("buyer is querying for NAICS 541512 capabilities; here are the matching claims").

### 8. Jurisdiction is a structured field, not free text

A claim's jurisdiction is a structured value: `{ country: ISO-3166 code, subdivision?: ISO-3166-2 code, freeText?: string }`. The country code is required for `certification` and `jurisdiction_coverage` kinds (where jurisdiction is load-bearing); optional for `capability_statement`, `past_performance`, and `other`. The `freeText` field exists for nuance the structured fields cannot capture ("authorised in EU member states except for Hungary pending pending appeal").

Multiple jurisdictions per claim are supported via a junction table (`capability_claim_jurisdiction`); a single `capability_claim` can carry zero, one, or many jurisdiction rows. The structured representation is what makes a future buyer-agent query like "show me suppliers authorised to deliver in Germany under FedRAMP-equivalent regimes" mechanically answerable.

### 9. The claim is the noun for the manifest's `claims[]` slot, but slice 68 does not populate it

Slice 68 ships the internal model only. The manifest's `claims[]` population happens in slice 69, which serialises the approved claims relevant to the assembling pursuit's opportunity (matched on NAICS / UNSPSC overlap with the opportunity's categorisation; the matching algorithm itself is owned by slice 69). This split is deliberate:

- Slice 68 is data-modelling and approval-workflow. Slice 69 is serialisation and matching. Each slice has a contained scope.
- The manifest schema slot is already declared (ADR 0022 §2). Slice 68 changes nothing about the schema; slice 69 changes only the serialiser to produce non-empty `claims[]` arrays.
- Tenants can use the claim model (author claims, approve them, see the freshness surface) before slice 69 ships. The model becomes useful internally — for the future buyer-agent sandbox (slice 77), for the answer library (slice 18) which can suggest claims as candidate answers — without waiting for the manifest population.

### 10. Claims are tenant-owned (ADR 0005); buyer-facing exposure is a separate decision (ADR 0029)

A claim is the supplier's assertion about themselves. It is tenant-scoped and lives in the tenant-owned partition. It is **not** shared market data (claims describe a single tenant's capabilities, not the procurement landscape).

This ADR makes no commitment about whether claim contents are ever surfaced outside the tenant. The manifest's `claims[]` payload (slice 69) is internal to the platform's existing artifact distribution: the tenant downloads the bundle and shares it with buyers through their own channel. A _direct_ buyer-agent query interface against the live claim set is the subject of ADR 0029 (deferred until the credential model, governance, audit, and rate-limit story exist for buyer-side reads).

The deliberately-conservative posture is the same one ADR 0022 took for the manifest itself: the _shape_ of the claim is forward-compatible with eventual buyer-agent exposure, but the _exposure_ decision belongs to a future ADR.

## Consequences

### Direct consequences (slice 68 implements)

- A new `capability_claim` table ships with the five kinds, kind-specific typed jsonb, claim text, validity window, jurisdiction set, supersession chain, and approval state.
- New junction tables for evidence (`capability_claim_evidence`), categories (`capability_claim_category`), jurisdictions (`capability_claim_jurisdiction`), and conflicts (`capability_claim_conflict`).
- The claim approval flow reuses the slice-28 reviewer surface with claim-specific entry into the queue.
- A `computeClaimFreshness` helper composes the four-input freshness rule.
- Two new Class A read tools (`listCapabilityClaims`, `getCapabilityClaim`) join the Layer A registry.
- The Answer Library (slices 18–20) gains a "this answer corresponds to claim X" affordance allowing the tenant to link existing answers to newly-authored claims (or auto-suggest claim creation from frequently-cited answers).

### Forward-compatibility consequences (this ADR's load-bearing reason)

- Slice 69 serialises the approved-claim subset matching a pursuit's opportunity into the manifest's `claims[]` slot. The manifest schema does not change; the slot was reserved.
- Slice 75 / ADR 0027 signs the manifest including `claims[]`; the per-claim payload includes claim id, claim version, evidence-link integrity, freshness state at assembly time, and category mappings.
- Slice 77 (buyer-agent query sandbox) reads from the same internal model to simulate buyer queries — gaps, stale claims, missing categories surface against the live claim set, not against the manifest snapshot.
- ADR 0029 (public manifest read surface) decides what subset of claim fields is buyer-visible vs internal-only. The structured representation makes selective field exposure straightforward.
- Future kind-specific extension tables (e.g. `capability_claim_certification` if `certification`'s structured fields outgrow the typed jsonb) are additive and require no ADR.
- A future cross-tenant capability discovery surface (out of scope for the M2M roadmap) would build on the same claim noun.

### Costs and trade-offs accepted

- **Two parallel evidence-adjacent nouns** (`content_asset` for proof; `capability_claim` for assertion). Tenants must learn the distinction. The mitigating choice is the Answer Library bridge: existing answers in the library (which already cite content_assets) gain a "promote to claim" affordance, so the migration path is incremental rather than greenfield.
- **Many-to-many junctions** (evidence, categories, jurisdictions, conflicts) increase the schema surface. The cardinality is real and irreducible; collapsing any of them to 1:N would lose modelling fidelity.
- **Computed freshness costs a join on every read.** The inputs (linked evidence rows, last_reviewed_at, valid_until) are small and indexed. A future cache column is a straightforward addition if the join cost becomes meaningful.
- **Supersession-not-edit on material fields is a UX cost.** Operators editing a typo in approved-claim prose are forced through the supersession ceremony. The mitigating choice is a clear distinction in the UI between minor and material edits, with the latter labelled as "create new version."
- **Kind-specific structured data in jsonb** trades some type safety for a smaller initial schema surface. Per-kind Zod validation at the service boundary keeps the typing strict at the API layer; the database accepts any jsonb. Promoting common patterns to dedicated tables remains a no-ADR follow-up.
- **Manual conflict marking** has a discoverability cost — if no one marks a conflict, the platform cannot surface it. The buyer-agent sandbox (slice 77) is the safety net: even un-marked conflicts may surface there as buyer-query oddities. Auto-detection is a follow-up.
- **Self-declared category mappings are accepted as written.** A tenant who tags a claim with an invalid NAICS code will not be told. A future validation slice can cross-check against canonical taxonomies; the mapping shape does not need to change.

### Things this ADR explicitly does not commit to

- **Manifest population.** Slice 69 owns the serialisation and matching algorithm.
- **Auto-detection of conflicts.** Manual marking only in MVP.
- **Cross-validation against canonical category taxonomies** (NAICS / UNSPSC). Self-declared in MVP.
- **Cross-tenant claim discovery.** All claims are tenant-scoped.
- **A claim DSL.** Claim prose is free text; structured fields are typed; no DSL.
- **Auto-promotion of frequently-cited answers to claims.** Affordance to manually promote, no auto-promotion.
- **Per-buyer claim suppression** ("hide this claim from buyer X"). The claim is either approved-and-eligible or it is not; per-buyer audience controls are a future selective-disclosure slice that wants its own ADR.
- **Claim-level signing.** Signing is per-package (ADR 0027); per-claim signing is not foreclosed but is not in scope here.
- **A federated claim model** (claims mirrored from an external supplier-master-data system). All claims are authored in-platform.
- **Translation / multilingual claim text.** A claim's prose is single-language in MVP. Slice 64a's multilingual extraction work is the precedent for a future multilingual extension.

## Alternatives considered

### Alt A — Extend `content_asset` with claim semantics rather than introduce a new noun

Rejected. The reasons (§1) are extensive; the short version is that one-to-many cardinality, independent approval lifecycles, structured kind-specific data, jurisdiction, category mappings, and supersession chains all belong on a noun whose meaning is "the assertion," not "the proof." Conflating produces a row that means both, which makes every downstream consumer (manifest serialiser, buyer-agent query, freshness composer) check a discriminator and operate on a subset of fields per case. The ergonomic and integrity costs compound.

### Alt B — Make the claim a generated view over `content_asset` plus tags

Rejected. A claim has fields no asset has (jurisdiction, valid_until, kind-specific structured data, supersession chain). A view cannot store fields that don't exist on its base table; a base-table extension is Alt A.

### Alt C — Per-kind tables (`capability_claim_certification`, `capability_claim_past_performance`, etc.) instead of typed jsonb

Considered. The per-kind table approach gives strict types at the database level and clean indices on per-kind fields. It also produces five tables for an MVP feature, with five separate API surfaces and five separate UI editors. Rejected for slice 68 in favour of typed jsonb with Zod validation at the service boundary; the per-kind extension is a no-ADR follow-up if any kind's structured-field surface outgrows jsonb readability.

### Alt D — Single jsonb column for _everything_ (claim + evidence links + categories + jurisdictions)

Rejected. A claim with no relational projection of its links cannot be efficiently queried for "all claims that cite this evidence" or "all claims tagged NAICS 541512" — the queries the manifest population (slice 69) and the buyer-agent sandbox (slice 77) most need. The junction tables are load-bearing for those queries; the ergonomic cost of authoring through structured forms is small.

### Alt E — Auto-detect conflicts via embedding similarity / language-model classification

Deferred. Not rejected — a future slice could add an opt-in conflict-suggestion engine. But MVP commits to manual marking only, for the false-positive-cost reasons in §5. The marked-conflict junction's shape is forward-compatible: an auto-suggestion engine writes to the same junction with a `marked_by = 'system'` discriminator (currently `marked_by_user_id` is non-null; adding a system marker is additive).

### Alt F — Allow material-field edits on approved claims with audit-only tracking

Rejected. The whole point of the approved-state distinction (ADR 0008) is that approved means committed. Mutating an approved claim's material fields means a buyer reading a month-old submission package and a buyer querying the live capability surface today see the same claim id with different semantics. Supersession preserves the chain and makes the manifest's per-claim entry stable.

### Alt G — Use only one evidence-link role (not `primary | supporting | corroborating`)

Considered. A simpler junction would carry only `(claim_id, content_asset_id)`. The role discriminator was added because it surfaces in the buyer-agent sandbox and (possibly) in the manifest's per-claim payload — "the primary evidence is X, the supporting evidence is Y" carries meaningful information that "evidence is {X, Y}" does not. The one-byte discriminator is cheap.

### Alt H — Reuse `requirement_coverage` (slice 05) as the claim noun

Rejected. `requirement_coverage` is per-pursuit: it maps a _specific RFP requirement_ to evidence. The claim is per-tenant: it asserts a _capability_ irrespective of any pursuit. Conflating them would either tenant-scope `requirement_coverage` (breaking its per-pursuit semantics) or pursuit-scope the claim (breaking its reusability). The right relationship is the future "this pursuit's compliance matrix can auto-cite claim X to cover requirement Y" affordance, owned by a slice that links the two nouns at the matrix level.

## Follow-on implications

- **Slice 68** is the first implementation. It ships the tables, the approval workflow integration with slice 28, the freshness-composition helper, the claim editor UI, and the two new Class A read tools.
- **Slice 69** serialises approved claims into `manifest.claims[]` and ships the matching algorithm (claim ↔ opportunity overlap on category mappings).
- **Slice 75 / ADR 0027** signs the manifest including `claims[]`; per-claim integrity at signing time is a precondition.
- **Slice 77 (buyer-agent query sandbox)** reads from the live claim set to simulate buyer queries and surface gaps, stale claims, and conflicts.
- **ADR 0008 follow-up** — the advisory-vs-approved frame is unchanged but extended in scope: "approved" now applies to a third entity (claim), not just drafts and requirements. The ADR's general principle holds.
- **ADR 0009 amendment** — register the `capability_claim.*` event family.
- **Slices 18–20 follow-up** — the answer library gains a "promote to claim" affordance. Existing answers can become evidence-backed claims without re-authoring.
- **A future ADR for selective-disclosure / per-buyer claim audience controls** if and when the use case materialises.
- **A future ADR for cross-validation of category mappings** against canonical NAICS / UNSPSC.

## References

- Slice 68 — `docs/slices/slice-68-capability-claims-schema-approval-and-freshness.md` (first implementation of this ADR)
- Slice 69 — capability claims in submission package manifest (planned; serialisation slice)
- ADR 0005 — shared market data vs tenant-owned workflow data (claims are tenant-owned)
- ADR 0008 — advisory vs approved workflow state (the approval model claims inherit)
- ADR 0009 — audit and event taxonomy (the `capability_claim.*` event family)
- ADR 0011 — semantic retrieval for content memory (the embedding model that may eventually power claim ↔ evidence auto-suggestion)
- ADR 0018 — audit-trail immutability (the discipline supersession-not-edit echoes)
- ADR 0022 — submission package as first-class artifact (the manifest's `claims[]` slot this ADR fills the noun for)
- ADR 0029 — public manifest read surface (the future decision about buyer-facing claim exposure)
- Slice 07 — content memory and evidence mapping (the `content_asset` noun the claim is distinct from)
- Slice 18–20 — answer library (the existing answer noun the claim relates to but does not replace)
- Slice 24 — compliance matrix (the surface that may eventually cite claims to cover requirements)
- Slice 28 — reviewer workflow (the approval surface the claim's approval reuses)
- Slice 53 / 54 — content-asset extraction and freshness (the freshness-floor input the claim composes)
- Slice 65 — submission package assembly (the manifest the claims will eventually populate)
