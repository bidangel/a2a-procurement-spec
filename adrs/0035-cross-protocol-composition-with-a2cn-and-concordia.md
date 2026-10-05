# ADR 0035: Cross-Protocol Composition — A2CN Owns Session, Concordia Owns Envelope, BidAngel Owns Procurement Payload Semantics

## Status

Proposed (2026-05-12). Authored in response to the three-way coordination
emerging on the AAIF Extension URI thread
([a2aproject/A2A#1737](https://github.com/a2aproject/A2A/discussions/1737))
between BidAngel (this profile), the A2CN authority/session protocol, and
the Concordia envelope/receipt protocol. Companion to ADR 0034 (the
layered-on-UBL discipline that defines what "procurement payload semantics"
actually means in this profile). Cross-cuts ADR 0032 (M2M roadmap pause —
counterparty-gated wire envelope) in the upstream BidAngel monorepo;
that ADR's Phase B wire-envelope scope is reduced as a consequence of
this composition and is flagged as a follow-up reconciliation.

This ADR is **proposed**, not accepted, because adoption depends on
confirmation in the discussion thread (Erik Newton's Draft B is still
open) and on at least one round-trip composition exercise against the
Concordia v0.5+ and A2CN v0.3+ reference implementations. The shape
recorded here is the shape we intend to commit to absent material
objections.

> **Amended 2026-10-06.** The composition exercise has been run and the
> adjacent maintainers have narrowed what they agree to. The amendment
> below records both. Where it conflicts with the text that follows, the
> amendment governs.

## Amendment (2026-10-06): what is agreed, and what the composition exercise found

The decision in §1 stands: this profile composes with A2CN and Concordia
and does not author a session, an envelope or a receipt format. What has
changed is how much of that composition the adjacent maintainers have
endorsed, and which Concordia artifacts exist to compose with. This ADR
stays **Proposed**.

### A. What the adjacent maintainers have agreed

- **Concordia.** Erik Newton's position, stated in review of
  [concordia-a2cn/procurement-patterns#2](https://github.com/concordia-a2cn/procurement-patterns/pull/2#issuecomment-5999253749)
  on 2026-10-05, is limited to one statement: "Concordia receipts can
  point at these fragments through the existing `references[]`". He
  states that it does not change Concordia, does not attest the
  crosswalks, the JSON-LD contexts, the Extension URI or any later
  filing, and is not an A2A registration. He does not speak for A2CN.
- **A2CN.** No statement. The A2CN maintainer confirmed the joint
  repository and path on 2026-06-17 and has not commented since.
- **A2A project.** The intent-to-author thread
  ([a2aproject/A2A#1832](https://github.com/a2aproject/A2A/discussions/1832))
  has had no reply from a member of the A2A Technical Steering
  Committee. Nothing is registered or on file with the A2A project.

The profile is therefore authored by BidAngel alone. It is not
co-authored or co-signed, and it must not be described that way.

### B. Concordia artifacts this ADR assumed that do not exist

- **`RejectionRecord` was never shipped.** Erik Newton corrected this on
  2026-06-18
  ([#1737](https://github.com/a2aproject/A2A/discussions/1737)) and
  again on 2026-10-05. Concordia ships `ApprovalReceipt`,
  `FulfillmentAttestation` and `RevocationRecord`; the last is a
  different artifact, not a rename.
- **No umbrella Registered Vocabulary Profile has been filed.** The
  plan that an umbrella registration would reference this profile is
  withdrawn.

Consequences for the text below:

- **§2, `procurement.*` clause namespace.** There is no Concordia
  artifact for it to plug into. The reservation stands as this
  profile's own convention, and
  `docs/crosswalks/procurement-rejection-clause-audit.md` stands as a
  statement of what a typed rejection artifact would need to carry. It
  is not a registry any other protocol has agreed to.
- **§2, relationship verbs.** `satisfies` has not been added to
  Concordia's vocabulary. It travels as an unknown relationship value,
  which Concordia preserves as an opaque string (Concordia SPEC
  §11.5.5). `approves` is defined by Concordia §9.6.4b.
- **Open questions 1 and 2.** Both ask how `RejectionRecord` should be
  shaped. Neither is under discussion. They are kept as the positions
  this profile would take if the artifact is ever proposed.
- **Consequences, "Credible co-authoring slot".** Withdrawn; see A.

### C. What the composition exercise found

The worked example proposed in
[bidangel/a2a-procurement-spec#4](https://github.com/bidangel/a2a-procurement-spec/issues/4)
is built as
[concordia-a2cn/procurement-patterns#3](https://github.com/concordia-a2cn/procurement-patterns/pull/3):
a best-and-final-offer round in which a human approves an acceptance
that crosses a mandate threshold. It was run against the Concordia
reference implementation at `847729c` (0.10.0) and the A2CN reference
implementation at `c7ff3f1` (0.3.0).

- **The split holds.** A procurement fragment rides inside the A2CN
  offer's terms, the A2CN act hash covers it, and a Concordia
  `ApprovalReceipt` binds to that same digest and references the
  session, the mandate and the opportunity. Concordia's verifier
  accepts the receipt; A2CN pauses the session as specified.
- **The pause belongs to the accepting party.** In A2CN the approval
  pause triggers on the act of the party whose mandate carries the
  threshold, so the buyer's approval attaches to the buyer agent's
  acceptance, not to the supplier's offer as issue #4 proposed.
- **A2CN does not accept the Concordia receipt as issued.** Three
  differences between the two reference implementations:
  1. A2CN compares the receipt's offer hash to its own base64url
     encoding; Concordia's schema requires `sha256:<hex>`. Same digest.
  2. A2CN reads the approver from a top-level `approver_did`; Concordia
     carries `approver.identity`.
  3. Concordia recommends `urn:a2cn:session:<id>` for the session
     reference; A2CN accepts the bare id or `a2cn:session:<id>`.

  Concordia's maintainer has confirmed that Concordia keeps the first
  two as specified. None of the three is this profile's to resolve.

### D. Changes on this profile's side

- The JSON-LD contexts bind the prefix `bapp:` to the profile namespace
  `https://spec.bidangelai.com/v0.1/`. They previously used `a2a:`,
  which a reader could take for a namespace owned by the A2A project.
  The namespace and term IRIs are unchanged.
- All four fragments now have a v0.1 context.
- The fragments, contexts and crosswalks are proposed for the joint
  repository
  ([procurement-patterns#2](https://github.com/concordia-a2cn/procurement-patterns/pull/2))
  as payload fragments, under that repository's licence, not as a
  registration.

ADR 0032 §10 in the upstream monorepo already records the Phase B
consequence of this composition. It relies on Concordia `references[]`,
which exists, and is not changed by this amendment.

## Context

The AAIF Extension URI submission (intent-to-author filed 2026-05-10 per
`correspondence/2026-05-10-aaif-intent-to-author.md`; TSC discussion at
[a2aproject/A2A#1832](https://github.com/a2aproject/A2A/discussions/1832))
opened a parallel coordination thread at
[a2aproject/A2A#1737](https://github.com/a2aproject/A2A/discussions/1737)
in which two adjacent profiles — A2CN (authority chain / session) and
Concordia (negotiation envelope / receipt) — converged on a substrate
split that places this profile's scope cleanly above theirs.

The relevant comments in the thread:

- [`#discussioncomment-16874987`](https://github.com/a2aproject/A2A/discussions/1737#discussioncomment-16874987)
  — **Draft A.** BidAngel posted an early sketch in which the A2A
  Procurement Profile carried its own envelope, signing manifest, and
  session shape (consistent with the original ADR 0032 §1 plan to ship
  a counterparty-gated BidAngel-owned wire envelope in Phase B).
- [`#discussioncomment-16882987`](https://github.com/a2aproject/A2A/discussions/1737#discussioncomment-16882987)
  — **Erik Newton (Concordia) substrate-split proposal.** Erik observed
  that A2CN already covers session establishment and authority/mandate
  verification, that Concordia already covers the negotiation envelope
  and receipt format with cross-protocol `references[]`, and that the
  A2A Procurement Profile's distinctive contribution is the **payload
  semantics** (canonical procurement fragments) and a small set of
  procurement-specific relationship verbs. Erik asked for a worked
  example of the composition before either side commits.
- [`#discussioncomment-16883023`](https://github.com/a2aproject/A2A/discussions/1737#discussioncomment-16883023)
  — **Draft B.** Erik's revised proposal pinning the substrate split
  with two open questions left to the working group: closed-per-version
  versus open-ended `clause` namespace for typed RejectionRecord
  enumerations, and whether counteroffer-time and acceptance-time
  rejections should share the same `RejectionRecord` shape.
- [`#discussioncomment-16521773`](https://github.com/a2aproject/A2A/discussions/1737#discussioncomment-16521773)
  — **Earlier A2CN signing-suite reply.** A2CN signs with ES256 (RFC 8785
  canonicalization) and accepts Ed25519 as an alternate suite. Concordia
  signs with Ed25519 with ES256 as an alternate. The two protocols are
  algorithm-aligned in the intersection (both accept both), which means
  this profile does not need to mediate.

The pre-coordination plan was the one ADR 0032 §1 records: the A2A
Procurement Profile would, in Phase B, ship its own counterparty-gated
wire envelope and its own signed manifest (anchored to the slice 82 /
ADR 0027 single-variant signing manifest in the upstream BidAngel
monorepo). That plan was sized for a world in which no adjacent profile
covered session or envelope in a way the procurement domain could compose
with. Erik's Draft A → Draft B sequence demonstrates that world is no
longer the operative one: A2CN is mature enough (v0.3+) to own session
and authority-chain semantics; Concordia is mature enough (v0.5+) to own
the negotiation envelope, receipt format, and the cross-protocol
`references[]` mechanism. Both have running reference implementations
the LF AAIF working group has been reviewing.

The strategy synthesis behind ADR 0034 already named the ceiling: a
profile with a high novel-share burden (UBL alignment alone targets
≤20% novel; cf. ADR 0034 §1) is hard to defend in standards-review.
Bringing additional novel surface — a wire envelope, a session state
machine, a receipt signing format — into a profile whose distinctive
contribution is procurement payload semantics inflates the novel-share
numerator with infrastructure that is not procurement-specific. Two
mature adjacent profiles already cover that infrastructure cleanly.
The economically and politically correct move is to compose, not to
duplicate.

The decision recorded here is the substrate split itself: positively
fixing what this profile owns, and explicitly declining what it does
not own, so that the AAIF submission, the eventual ADR 0032 amendment
in the upstream monorepo, and the per-fragment crosswalks under
`crosswalks/` all reference the same composition shape.

## Decision

### 1. Adopt the three-way substrate split as Erik proposed it

This profile composes with A2CN (authority + session) and Concordia
(envelope + receipt + references) along the boundaries Erik's Draft B
draws. The split is:

- **A2CN owns** session establishment and the authority chain — mandate
  verification (does this agent have the authority to act on behalf of
  this principal?), bilateral transaction record (the durable
  transcript both parties retain), session-state lifecycle.
- **Concordia owns** the wire envelope (the message frame two agents
  exchange), the negotiation receipt (the signed acknowledgement of an
  exchange step), and the cross-protocol `references[]` mechanism with
  its relationship verbs (`fulfills`, `approves`, `enforces`, `rejects`).
- **This profile owns** the procurement payload semantics that ride
  inside the Concordia envelope, plus the `procurement.*` namespace
  reservation (for typed RejectionRecord clause enumeration) and a
  small initial set of procurement-specific relationship verbs added
  to Concordia's `references[]`.

The split is not arbitrary; it follows the maturity gradient of each
profile. A2CN's mandate-verification semantics generalise across
domains and are well-shaped for the procurement use case (a buyer-side
agent acting on behalf of a contracting authority is the canonical
mandate scenario). Concordia's envelope and receipt format are
similarly domain-neutral and were designed with cross-protocol
composition in mind from inception. This profile's contribution is
the part that is **procurement-specific**: what fragments ride in
the envelope, what relationship verbs the references mechanism uses
when the parties are negotiating a procurement event, what clause
types a rejection enumerates.

### 2. What this profile DOES own, positively

> Amended 2026-10-06: the clause namespace has no Concordia artifact to
> attach to, and `satisfies` is not part of Concordia's vocabulary. See
> the amendment, part B.

The owned surface is restricted to procurement-specific semantics. In
order:

- **Canonical procurement payload fragments.** The four canonical
  fragments ADR 0034 already pins to UBL — `opportunity`,
  `requirement`, `evaluation_criterion`, `buyer` — plus future
  fragments per ADR 0034 §3. These are the JSON-LD-shaped objects
  that ride as the payload inside a Concordia envelope. The
  per-fragment crosswalks under `crosswalks/` remain the maintenance
  surface; the layered-on-UBL discipline of ADR 0034 is unchanged.
- **The `procurement.*` namespace** for typed RejectionRecord clause
  enumeration. Concordia's `RejectionRecord` shape carries a `clause`
  field whose namespace is open-ended (per Erik's anticipated Draft B
  position; cf. Open Questions §1 below). This profile reserves
  `procurement.*` for procurement-domain clauses — for example,
  `procurement.evaluation_criterion.unmet`,
  `procurement.requirement.non_responsive`,
  `procurement.compliance.exclusion_ground.applies`. A future
  registry document under `crosswalks/` (or a sibling
  `clauses/procurement.md`) will enumerate the initial set, version
  it, and pin the registry to the layered-on-UBL crosswalks where
  applicable (`exclusion_ground.applies` aliases ESPD exclusion
  grounds; `evaluation_criterion.unmet` aliases the
  `CanonicalEvaluationCriterionFragment` of ADR 0034).
- **Procurement-specific relationship verbs added to Concordia's
  `references[]`.** Initial set: **`satisfies`** (target: an OCDS
  opportunity identifier). The `satisfies` verb says "this
  procurement payload [a supplier response, a clarification answer,
  a capability claim] satisfies the requirement identified by the
  referenced OCDS opportunity ID." Future verbs (anticipated, not
  yet committed): `evidences` (capability claim → underlying
  evidence; pending alignment with ADR 0025 in the upstream
  monorepo), `cites` (response section → requirement ID),
  `supersedes` (clarification round N+1 → round N answer). Each
  future verb ships in its own ADR amendment to this one, after at
  least one composition test against Concordia's reference
  implementation.

The discipline that bounds this owned surface is the same one
ADR 0034 §1 already commits to: novel territory only with a written
rationale, in a small clearly-flagged set, not as a parallel
ontology.

### 3. What this profile does NOT own, explicitly

The negative space is as load-bearing as the positive surface. This
profile does not own, and will not author, any of:

- **The wire envelope.** That is Concordia's. This profile's
  payload fragments ride inside the Concordia envelope; this
  profile defines no parallel envelope shape, no parallel
  serialization, no parallel framing.
- **The session state machine.** That is A2CN's. Session
  establishment, mandate verification, bilateral transaction
  record, session lifecycle — out of scope.
- **The negotiation-receipt signing format.** That is Concordia's
  (Ed25519 with ES256 alternate). This profile does not define a
  receipt signing manifest; the Concordia receipt covers the
  exchange step. Note that this is **distinct from claim signing**
  (see Consequences §3 below): the upstream BidAngel monorepo's
  ADR 0027 / Slice 82 single-variant signing manifest signs
  capability claims, which is a different concern from signing a
  negotiation receipt.
- **Mandate verification.** That is A2CN's. Whether the buyer-side
  agent presenting itself in a session is in fact authorised by
  the contracting authority is an A2CN mandate-chain question, not
  a procurement-payload question. This profile's payloads assume
  the session is already established and the mandate is already
  verified.

If a future requirement surfaces that does not fit cleanly into
one of A2CN's, Concordia's, or this profile's owned surfaces, the
preferred resolution is to push it into the appropriate adjacent
profile (or onto the LF AAIF working group as a cross-profile
question), not to absorb it here. Absorbing it here would
re-inflate the novel-share burden the substrate split exists to
prevent.

### 4. Algorithm and signing notes (informational, not normative)

This profile does not define a signing suite of its own at the
envelope/receipt layer; that is Concordia's. For reference:

- **A2CN** signs with ES256 (RFC 8785 canonicalization) and
  accepts Ed25519 as an alternate suite (per
  [`#discussioncomment-16521773`](https://github.com/a2aproject/A2A/discussions/1737#discussioncomment-16521773)).
- **Concordia** signs with Ed25519 with ES256 as an alternate.
- The intersection (both accept both) means this profile does not
  need to mediate signing-suite negotiation. A composition stack
  picks one suite per session and both adjacent profiles
  interoperate.

The upstream BidAngel monorepo's ADR 0027 (single-variant signing
manifest, claim-signing) is a separate concern at a separate layer
and is not affected by this paragraph.

## Consequences

### Positive

- **Phase B scope reduction in the upstream monorepo.** ADR 0032 §1's
  counterparty-gated BidAngel-owned wire envelope is no longer in
  scope — the wire envelope is Concordia's. The procurement-specific
  Phase B work shrinks to: payload fragments (already pinned by ADR
  0034), the `procurement.*` clause registry, the initial relationship-verb
  set, and a composition test against Concordia + A2CN reference
  implementations. The remaining novel surface is small enough to
  ship inside the Phase B counterparty-gate window without further
  ADR-level scope.
- **Leverages mature negotiation primitives.** A2CN and Concordia are
  not abstract specifications; both have running reference
  implementations the LF AAIF working group has been reviewing. This
  profile picks up the operational benefit (running code, debugged
  edge cases, completed security review of the envelope/receipt
  layers) without authoring it.
- **Credible co-authoring slot.** _(Withdrawn 2026-10-06; see the
  amendment, part A.)_ A profile that brings procurement
  payload semantics to a session+envelope substrate two adjacent
  protocols already maintain has a clear distinctive contribution and
  a clear collaboration story. Standards-review pressure is materially
  reduced when the novel surface is small and the composition shape is
  worked out in advance with the adjacent profile authors.
- **Reduces novel-share burden under ADR 0034.** ADR 0034 §1 commits
  to a 20% novel-share ceiling per fragment against UBL. The substrate
  split moves the envelope, the session machinery, and the receipt
  format out of "novel surface this profile must defend in
  standards-review" entirely — they are not this profile's surface to
  defend.

### Negative

- **Release cadence couples to Concordia v0.5+ and A2CN v0.3+.** This
  profile cannot ship interoperable demonstrations without compatible
  versions of both adjacent profiles. A regression or breaking change
  in either reference implementation propagates into this profile's
  composition tests. The mitigation is the same one ADR 0034 §2
  applies to UBL/Peppol pinning: pin the adjacent-profile versions in
  the composition test fixture, re-pin event-driven (not date-driven)
  when the upstream lands a compatible new version.
- **Inherits operational issues in the adjacent reference
  implementations.** A bug in Concordia's envelope serializer or
  A2CN's mandate-verification path surfaces as a bug in the composed
  stack. The mitigation is upstream contribution and bug filing
  through the AAIF working group; the cost is non-zero coordination
  overhead. This is the standard cost of composition over
  re-implementation and is judged to be smaller than the cost of
  carrying a parallel envelope/session implementation.
- **Cross-profile change coordination.** Adding a new relationship
  verb (per §2) requires Concordia working-group ratification, not
  unilateral commit. This is the right cost for a public surface but
  is slower than the in-monorepo cadence the upstream BidAngel
  platform would otherwise expect.

### Neutral

- **Claim signing is unaffected.** The upstream BidAngel monorepo's
  Slice 82 / ADR 0027 single-variant signing manifest signs
  **capability claims** — the supplier's signed assertions about its
  own capabilities. That is a different concern from signing a
  **negotiation receipt** (Concordia's responsibility). Algorithm
  alignment with Concordia's Ed25519 is desirable for operational
  consistency but is not blocking; ADR 0027's signing-suite choice
  remains an upstream-monorepo decision.
- **Discovery and read-side surfaces are unaffected.** The upstream
  ADR 0029 (public manifest read surface) and ADR 0031 (capability
  claim graph public read surface) operate on a different axis (HTTP
  GET endpoints under supplier-issued credentials) than the A2A
  negotiation surface this composition addresses. Erik's earlier
  endorsement of the three-tier evidence-dereferencing as composing
  cleanly into typed `references[]` per tier (cf. Implications below)
  applies if and when a buyer agent inside an A2A session wants to
  surface claim-graph references; it does not require any change to
  the read-surface design.

### Things this profile explicitly does not commit to (under this composition)

- **Authoring a competing wire envelope** "in case Concordia stalls."
  If Concordia stalls or fragments, the resolution is upstream
  coordination, not unilateral re-authoring.
- **Locking Concordia's relationship-verb registry shape.** The
  `procurement.*` namespace is reserved here; the registry's
  meta-shape (open-ended vs closed-per-version, prefix discipline,
  publication mechanism) is Concordia's call (cf. Open Questions §1).
- **Mandating ES256 vs Ed25519 at the envelope/receipt layer.** This
  profile does not pick the suite; the composed stack does, per the
  intersection both adjacent profiles accept.
- **Defining a session-resumption protocol.** Sessions are A2CN's; if
  procurement workflows need long-lived or resumable sessions
  (clarification rounds spanning days, BAFO sequences spanning weeks),
  the requirement goes to A2CN as a use-case input, not to this
  profile as an extension.

## Open questions

> Amended 2026-10-06: Concordia has no `RejectionRecord`, so neither
> question is under discussion. See the amendment, part B.

Two open questions from Erik's Draft B
([`#discussioncomment-16883023`](https://github.com/a2aproject/A2A/discussions/1737#discussioncomment-16883023))
are still under discussion. This ADR records anticipated positions but
defers ratification to the working group.

### 1. Closed-per-version vs open-ended `clause` namespace for `RejectionRecord`?

The question is whether Concordia's typed `RejectionRecord.clause`
field should enumerate a closed set of clause types per Concordia
version (with version bumps to add new types) or an open-ended
namespace where each composing protocol publishes its own prefix and
registry.

**Anticipated position: open-ended with prefix discipline plus a
published clause registry.** Each protocol owns its prefix
(`procurement.*` for this profile, `tos.*` for terms-of-service
clauses, etc.) and publishes a registry document that enumerates
the clauses, their semantics, and their version. Cross-protocol
composition wins under this model: a procurement rejection citing a
ToS clause does not require a Concordia version bump; the
RejectionRecord just carries `clause: "tos.acceptable_use.violated"`
and the receiving party dereferences via the published registry.

The closed-per-version alternative makes Concordia the bottleneck for
every clause-type addition across every composing protocol; that
scales poorly and pushes coordination overhead upstream.

The risk in the open-ended model is registry-quality drift: prefix
namespaces with no published registry, or registries that are
stale or unauthenticated. The mitigation is the discipline already
operative in this profile and ADR 0034: the registry document is
under version control in the publishing protocol's spec repository,
the authoritative URL is part of the composition contract, and a
quarterly diff (cf. ADR 0034 §5's standards-watch cadence) catches
drift.

### 2. Counteroffer-time vs acceptance-time rejection — same `RejectionRecord` shape?

The question is whether a rejection issued during the counteroffer
phase of a negotiation and a rejection issued at the acceptance phase
should share the same `RejectionRecord` shape, or whether they
warrant different shapes (different fields, different signing
discipline, different audit semantics).

**Anticipated position: same `RejectionRecord` shape with a
`stage` discriminator.** A single shape with a typed `stage` field
(open-ended namespace per Question 1, with a `procurement.*`
prefix for this profile's stages) keeps the implementation simple
and the cross-protocol composition story clean. Procurement adds
three stages naturally: `procurement.bid_evaluation` (rejection
during the buyer's evaluation of submitted bids),
`procurement.clarification_round` (rejection during a clarification
exchange — the buyer rejects an answer as non-responsive, or the
supplier rejects a clarification as scope-expanding), and
`procurement.bafo` (rejection during a Best-And-Final-Offer round).

The alternative — different shapes per stage — multiplies the
contract surface and forces every receiving implementation to handle
N parallel rejection types. The shared-shape-with-stage-discriminator
is the cheaper and more composable model.

The cost accepted is that the `RejectionRecord` shape carries some
fields that are stage-specific (e.g. a `procurement.bafo` rejection
might carry a "final price gap" field that a `procurement.bid_evaluation`
rejection does not). The mitigation is the same typed-jsonb pattern
ADR 0025 §2 (in the upstream monorepo) uses for kind-specific fields:
a `stage_specific_data` field validated per-stage by the receiving
implementation.

## Implications for existing ADRs

This composition has cross-cutting implications. Per the prompt
constraint, this ADR does not modify the affected ADRs in this run;
the cross-links are recorded one-way and follow-up reconciliation
is flagged where required.

- **ADR 0034 (this repo, layered-on-UBL discipline):** The
  composition reduces this profile's novel-share burden under
  ADR 0034 §1. Envelope, receipt, and session are no longer this
  profile's surface to defend, so the 20% novel-share ceiling
  applies to a smaller numerator. ADR 0034 will be amended in a
  follow-up to record the composition (no change to the
  fragment-level discipline; only a clarification of scope).
- **ADR 0025 / Slice 68 (capability claims; upstream monorepo):**
  Unaffected directly. The capability claim model continues to live
  in the upstream monorepo as a tenant-owned noun. This composition
  opens an optional path to surface a capability claim as a
  Concordia `references[]` target with a relationship verb like
  `evidences` (pending Concordia working-group ratification of the
  verb). No change to the claim model itself.
- **ADR 0031 / Slice 85 (capability-claim graph public read surface;
  upstream monorepo):** Unaffected. Erik explicitly endorsed the
  three-tier evidence-dereferencing (`redact_all` /
  `redact_asset_id_only` / `reveal_metadata`) as composing cleanly
  into typed `references[]` per tier when an A2A session needs to
  surface graph references. The HTTP read surface remains
  independently load-bearing for the buyer-agent flows that do not
  use an A2A session.
- **ADR 0032 (upstream monorepo, Phase B counterparty-gated wire
  protocol):** **Amendment required.** ADR 0032 §1's plan to ship a
  BidAngel-owned wire envelope in Phase B is superseded: the wire
  envelope is Concordia's, not this profile's. The Phase B
  counterparty-gate is unchanged; what ships under it changes.
  Marked as a follow-up reconciliation; this ADR does not modify
  ADR 0032 in the present commit (per the prompt constraint and
  because ADR 0032 lives in a different repository).
- **ADR 0027 / Slice 82 (single-variant signing manifest; upstream
  monorepo):** Unaffected. Claim signing (the upstream concern) is
  a different layer from receipt signing (Concordia's concern). The
  algorithm-alignment observation in §4 is informational; ADR 0027
  is not amended by this composition.

## References

- A2A Discussion #1737 — three-way coordination thread:
  [a2aproject/A2A#1737](https://github.com/a2aproject/A2A/discussions/1737)
  - Draft A:
    [`#discussioncomment-16874987`](https://github.com/a2aproject/A2A/discussions/1737#discussioncomment-16874987)
  - Substrate-split proposal + worked-example ask:
    [`#discussioncomment-16882987`](https://github.com/a2aproject/A2A/discussions/1737#discussioncomment-16882987)
  - Draft B (substrate split with two open questions):
    [`#discussioncomment-16883023`](https://github.com/a2aproject/A2A/discussions/1737#discussioncomment-16883023)
  - A2CN signing-suite reply (ES256 + Ed25519 alternate):
    [`#discussioncomment-16521773`](https://github.com/a2aproject/A2A/discussions/1737#discussioncomment-16521773)
- ADR 0034 (this repo) — A2A Procurement Profile is Layered on UBL /
  Peppol Semantics, Not Parallel to Them. The discipline this
  composition rides on; the novel-share ceiling whose numerator this
  composition reduces.
- ADR 0031 (this repo) — Capability Claim Graph Public Read Surface
  for Authorised Buyer Agents. The HTTP read surface that composes
  cleanly with Concordia `references[]` per Erik's endorsement.
- ADR 0026 (this repo) — Structured RFx Ingestion as a First-Class
  Peer to Document Ingestion. The substrate that produces the
  canonical fragments this profile carries as Concordia payloads.
- ADR 0025 (this repo) — Supplier Capability Claim Model. The claim
  noun whose optional surfacing through Concordia `references[]` is
  forward-compatible under this composition.
- AAIF intent-to-author letter (2026-05-10) —
  `correspondence/2026-05-10-aaif-intent-to-author.md`.
- AAIF TSC discussion (2026-05-11) —
  [a2aproject/A2A#1832](https://github.com/a2aproject/A2A/discussions/1832).
- RFC 8785 — JSON Canonicalization Scheme (the canonicalization A2CN
  uses for ES256 signing; informational).
- RFC 7515 — JSON Web Signature (the framing both A2CN and Concordia
  use for signed envelopes; informational).
