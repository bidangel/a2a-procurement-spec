# A2A Procurement Profile

Public spec repository for the **A2A Procurement Profile** — a draft
agent-to-agent procurement interoperability profile layered on existing
UBL 2.3 / Peppol BIS Pre-Award / OCDS 1.1.5 / EU eForms semantics.

> Status: **Draft v0.1 (in flight).** This repository is the public
> publishing surface for design notes, ADRs, field-by-field crosswalks,
> and (forthcoming) JSON-LD `@context` files and reference schemas.
> Authoritative authoring lives upstream in the BidAngel platform
> repository; published artifacts here are derived and provenanced
> (see `PROVENANCE.md`).
>
> **Intent to author** the Extension URI was recorded on 2026-05-10 —
> see [`correspondence/2026-05-10-aaif-intent-to-author.md`](correspondence/2026-05-10-aaif-intent-to-author.md).
> Active TSC discussion: [a2aproject/A2A#1832](https://github.com/a2aproject/A2A/discussions/1832) (posted 2026-05-11).

## Why this profile exists

The procurement-document standards landscape has strong, well-maintained
semantic vocabularies for human-readable structured documents (UBL, OCDS,
eForms, Peppol Pre-Award), but no widely-adopted interoperability profile
for **agent-to-agent procurement traffic** — the machine-readable,
session-bounded exchanges between buyer-side and supplier-side agents
working an opportunity through qualification, response drafting,
clarification, and submission.

This profile takes the position that the right move is to **layer on
existing semantics, not duplicate them**. The canonical types defined
here align field-by-field with UBL/Peppol/OCDS/eForms wherever those
vocabularies have a mapping, and introduce novel territory only where
the agent-to-agent workflow surface genuinely lacks an existing fit
(currently <10% of fields per fragment, per the published crosswalks).

## What's in this repository

| Directory          | Contents                                                                |
| ------------------ | ----------------------------------------------------------------------- |
| `adrs/`            | Architecture Decision Records that establish the profile                |
| `crosswalks/`      | Field-by-field mappings to UBL / Peppol / OCDS / eForms                 |
| `contexts/`        | JSON-LD `@context` files, versioned (`v0.1/requirement.context.jsonld` shipped; remaining fragments forthcoming) |
| `schemas/`         | JSON Schema files derived from the canonical fragments (forthcoming)    |
| `correspondence/`  | Standards-track correspondence (intent-to-author letters, TSC threads)  |
| `PROVENANCE.md`    | Per-file source SHA mapping back to the upstream platform               |
| `GOVERNANCE.md`    | Stewardship model and standards-liaison posture                         |

## Roadmap

- **v0.1** (90-day window from first AAIF TSC acknowledgement)
  - Stable JSON-LD `@context` for the requirement, opportunity,
    buyer, and evaluation_criterion fragments
  - JSON Schema files for each canonical fragment
  - At least one round-trip CI test (canonical fragment ↔ UBL XML)
- **v0.2 / Phase B**
  - Wire-protocol envelope (counterparty-gated; see ADR 0032)
  - Single-variant signing manifest (ADR 0027 / slice 82)

## Status disclosure

This is a draft profile. Nothing here should be relied on as a frozen
standard. Implementations should expect breaking changes until v1.0.

## Contributing

See `CONTRIBUTING.md`. Issues and discussion are welcome; substantive
changes go through an RFC pattern documented there.

## License

- **Code, schemas, JSON-LD `@context` files**: Apache License 2.0
  (see `LICENSE-CODE`).
- **Prose (ADRs, crosswalks, this README)**: Creative Commons
  Attribution 4.0 International (see `LICENSE-DOCS`).

## Maintainer

Currently stewarded by [BidAngel](https://github.com/bidangel). See
`GOVERNANCE.md` for the standards-liaison posture and intended
evolution.
