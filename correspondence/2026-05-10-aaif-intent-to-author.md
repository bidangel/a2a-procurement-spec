# Intent to Author — A2A "Procurement Profile" Extension URI

- **To:** A2A Project TSC / `a2aproject/A2A` GitHub Discussions
- **CC:** Mazin Gilbert (Executive Director, AAIF); SAP Joule / Ariba product leads
- **From:** Brian P., BidAngel (brianpasf@gmail.com)
- **Subject:** Intent to author an A2A Extension URI for procurement (RFx, supplier capability claims, buyer scope) — UBL / Peppol / OCDS / eForms aligned
- **Date:** 2026-05-10
- **Public spec repository:** [github.com/bidangel/a2a-procurement-spec](https://github.com/bidangel/a2a-procurement-spec)

---

Dear A2A maintainers,

We intend to author an A2A Extension URI scoped to B2B procurement — specifically the solicitation, supplier-capability, and evaluation segments of the source-to-award lifecycle. This note is to surface the work early, invite collaboration before we publish a draft URI, and ask whether anything materially overlapping is already in flight that we should join rather than parallel.

## Why this profile, why now

A2A's Extensions mechanism explicitly invites vertical work, and procurement is a domain where (a) buyer and supplier agents need to exchange evidence-backed assertions across organizational boundaries, (b) document formats are heterogeneous (CAIQ, SIG, Ariba DDQ, SAM.gov solicitations, OASIS UBL 2.3, Peppol BIS Pre-Award 1.0, OCDS 1.1.5, EU eForms per Implementing Regulation 2019/1780), and (c) no public extension or working group has yet staked a profile.

With SAP Ariba's Bid Analysis Agent reaching GA this quarter and Coupa's "A2A collaboration" entering product vocabulary, the ground is shifting from product-level toolkits toward a need for shared semantics. We would rather contribute one than ship a private dialect.

## What we propose to contribute

Four concrete artifacts from BidAngel's open architecture (durable ADRs in our repository), each procurement-neutral once tenant-scoping is removed:

1. **Capability-claim noun.** A typed, evidence-linked supplier assertion with kind discriminator (certification, past-performance, capability-statement, jurisdiction-coverage, other), supersession-not-edit, and freshness composed at read time from calendar validity, evidence freshness, review age, and approval state. Reference: BidAngel ADR-0025, Slice 68 (shipped).

2. **Buyer-scope filter algebra.** A declarative scope shape an agent can advertise on its AgentCard and a counterparty can narrow within: kind allowlist, AND-across-families / OR-within categoryFilters, jurisdictionFilters, freshnessFloor, supplier-veto denylist, and a three-tier evidence-dereferencing mode (`redact_all` / `redact_asset_id_only` / `reveal_metadata`). Reference: BidAngel ADR-0031, Slice 85 (shipped).

3. **Structured RFx adapter contract.** A format-agnostic orchestrator producing **three canonical fragment kinds** — `opportunity`, `evaluation_criterion`, `requirement` — with per-row provenance and per-row subtype discrimination (the `requirement` fragment carries `narrative` vs `questionnaire` as a typed `subType`, replacing an earlier separate `questionnaire_row` kind that was retired as duplicative). Each fragment ships as a typed TypeScript shape with documented round-trip semantics; the orchestrator pattern composes naturally with both A2A AgentCard advertisement and an MCP server pattern. Reference: BidAngel ADR-0026, packages/structured-rfx-adapters/, with **Slice 70 (CAIQ / SIG DDQ rows → `CanonicalRequirementFragment`) and Slice 71 (SAM.gov structured opportunities → `CanonicalOpportunityFragment`) shipped to production**; Slice 73 (SAP Ariba sourcing events → all three fragment kinds) is in flight under ADR-0032 Phase A.

4. **UBL / Peppol / OCDS / eForms field-by-field crosswalks under a layered-on-UBL discipline.** Six durable artifacts (ADR-0034) covering `evaluation_criterion`, `opportunity`, `buyer`, `requirement`, plus supporting cross-maps for criterion-type codelist (EU-COM-GROW `CriteriaTypeCode`) and opportunity status lifecycle. Each fragment is bucketed exact / partial / novel; the four shipped sit at **≈92% UBL-aligned with novel-share comfortably below the 20% ceiling we set ourselves** (ADR-0034 §1). Pinned to UBL 2.3 / Peppol BIS ESPD 1.0 / OCDS 1.1.5 / current eForms codelists, with documented event-driven re-pin commitment when UBL 2.4 / Peppol BIS Pre-Award 4.0 / OCDS 1.2 land. This is the work that absorbs OpenPeppol Pre-Award and OASIS UBL TC review surface; we'd rather contribute it to AAIF as part of the Extension URI proposal than have it discovered as a duplicative-ontology objection during review.

We are willing to host the draft Extension URI on a domain we control for v1 and migrate it to an AAIF-blessed namespace later if the foundation prefers that pattern. The public spec repository at [github.com/bidangel/a2a-procurement-spec](https://github.com/bidangel/a2a-procurement-spec) already contains the four ADRs above, the six crosswalks, and a v0.1 JSON-LD `@context` for the `requirement` fragment, dual-licensed under Apache 2.0 (code, schemas, contexts) and CC BY 4.0 (prose). Provenance is source-SHA-pinned to our upstream repository.

## Explicit invitation to SAP as co-author

SAP is an A2A founding member, and the Joule + next-gen Ariba roadmap is the most credible enterprise procurement-agent surface in market. We would rather co-author with SAP than publish a parallel profile that drifts from how the largest existing buyer-side workflow is being built.

Our specific ask: review of the typed canonical fragment shapes (`CanonicalEvaluationCriterionFragment`, `CanonicalOpportunityFragment`, `CanonicalRequirementFragment` — full TypeScript declarations available in the spec repo, with two of three already in production via Slice 70 [tenant-uploaded DDQ] and Slice 71 [public-structured government opportunity]) against Ariba's sourcing-event payload, to identify field-level divergences before slice 73 implementation lands. Co-listing as authors on the resulting Extension specification is on the table from our side. We are happy to begin from a Joule-side draft if one already exists internally.

## What we are not asking for

No changes to A2A's core protocol, AgentCard schema, or task model. No new SEP. We are submitting under the public Extensions mechanism precisely because vertical work belongs there — and because the layered-on-UBL discipline above means our canonical shapes round-trip to UBL / Peppol / OCDS / eForms without modifying any horizontal A2A primitive.

## What we are asking the TSC for

1. Confirmation that vertical procurement profiles are in scope for the Extensions mechanism today.
2. Pointers to any in-flight overlapping proposals (procurement, B2B vendor assertions, RFx ingestion, UBL / Peppol mapping work) we should join rather than parallel.
3. A preferred forum for ongoing discussion — GitHub Discussion thread, an Interest Group convening, or another channel.

## Timeline

We aim to publish a v0.1 Extension URI with reference schemas (markdown specification, JSON-LD `@context` for all canonical fragments, JSON Schema files, at least one round-trip CI test against a real EU eForms F02 fixture) within 90 days of TSC acknowledgement. The spec repository linked above is the publishing surface; product-side implementation continues in our private codebase and is referenced rather than included.

Thank you for the work on A2A and on the AAIF. We are glad the shared substrate exists, and we want to contribute to it rather than around it.

— Brian P., BidAngel
