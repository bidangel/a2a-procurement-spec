# Governance

## Current stewardship

The A2A Procurement Profile is currently stewarded by
[BidAngel](https://github.com/bidangel), the originating platform. The
upstream authoring repository is private; published artifacts in this
repository are derived and provenanced via `PROVENANCE.md`.

## Standards-liaison posture

This profile is explicitly designed to **layer on existing procurement
semantics**, not to duplicate them. The crosswalks under `crosswalks/`
demonstrate the field-by-field alignment with UBL 2.3, Peppol BIS
Pre-Award, OCDS 1.1.5, and EU eForms. Novel territory is bounded and
documented (see ADR 0034).

We intend to engage the following bodies as the profile matures:

- **OpenPeppol Pre-Award Domain Community** — the natural home for the
  pre-award structural work this profile builds on. Liaison to be
  initiated alongside v0.1 publication.
- **AAIF (Agent-to-Agent Interoperability Forum) TSC** — for the
  agent-protocol layering specifically.
- **Open Contracting Partnership** — for OCDS-aligned crosswalk review,
  particularly around the Requirements extension shape.

The intent over time is to either (a) propose this profile, or its
successor, as a contribution to an existing standards body's pre-award
workstream, or (b) place stewardship under a neutral foundation. Until
that transition, BidAngel maintains the spec under the dual license
declared in the README.

## Decision-making

- **Editorial changes** (typos, clarifications, broken links): handled
  via PR.
- **Substantive technical changes** (schema shape, new fragment types,
  crosswalk amendments): require an RFC issue first. See
  `CONTRIBUTING.md`.
- **Profile-version bumps**: cut from a tagged release commit; release
  notes summarize the delta and any breaking changes.

## Conflict-of-interest disclosure

BidAngel operates a commercial platform that implements this profile.
The profile is published with the explicit intent of enabling
interoperable implementations by other parties. We will not introduce
profile constraints whose only purpose is to differentiate the BidAngel
implementation against a competing one.

If a future stewardship transition occurs, the conflict-of-interest
question becomes moot at that point.
