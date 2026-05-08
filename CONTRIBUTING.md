# Contributing

Thanks for your interest in the A2A Procurement Profile.

## Issues

Open an issue for:

- Questions about how the profile maps to a specific source or
  standard (cite the standard version and the section)
- Concerns about the crosswalks (cite the field and the source URI)
- Requests for clarification on an ADR
- Reports of breakage in the published JSON-LD `@context` or schemas
  (when those land — currently in flight for v0.1)

## Pull requests

PRs are welcome for:

- Editorial fixes (typos, broken links, formatting)
- Crosswalk-evidence additions (e.g., a real-world counter-example
  that contradicts a current mapping)
- Test fixtures and round-trip evidence

For substantive technical changes, please open an RFC issue first
(see below).

Note that the prose ADRs and crosswalks in this repository are
derived from an upstream private platform repository. Editorial PRs
are happily accepted and round-tripped upstream; structural PRs may
require coordination with the upstream maintainer.

## RFC pattern (substantive changes)

For changes to:

- Canonical fragment shape
- ADR-level decisions
- Crosswalk recommendations (e.g., a Q1=A → Q1=B flip on a probe)
- Profile-version-affecting decisions

…open an issue tagged `rfc` with the following structure:

```
## Summary

One paragraph: what changes, in what direction.

## Motivation

What problem does this address? Cite the source/standard/adapter
or real-world workflow that surfaced the issue.

## Proposed change

Concrete: which fragment, which field, which mapping, which ADR
section.

## Alternatives considered

At least two; explain why they're inferior.

## Compatibility impact

Breaking? Additive? Profile-version-affecting?
```

The maintainer will respond within a reasonable window. Approved RFCs
are implemented upstream and republished here via the export pipeline.

## Code of conduct

Be substantive and respectful. We expect contributors to assume good
faith and to engage with the technical merit of arguments. Personal
attacks, harassment, or bad-faith engagement will be removed and the
contributor blocked.
