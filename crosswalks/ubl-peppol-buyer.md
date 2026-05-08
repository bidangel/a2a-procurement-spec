# Crosswalk — `buyer` to UBL `cac:Party` / OCDS `parties` / eForms

## Status

Probe (2026-05-08). Third per-fragment crosswalk under the ADR 0034
layered-on-UBL discipline; companion to the opportunity crosswalk.
Authored as the Opportunity probe identified the Buyer crosswalk as the
"second-most-load-bearing UBL crosswalk after this one." Pinned to:

- UBL 2.3 — `cac:Party` (with `cac:ContractingParty`,
  `cac:PartyName`, `cac:PartyIdentification`, `cac:PostalAddress`,
  `cac:PartyLegalEntity`, `cac:Contact`).
- OCDS 1.1.5 — `parties[]` array with `roles[]`, `identifier`,
  `additionalIdentifiers[]`, `name`, `address`, `contactPoint`,
  `details`.
- EU eForms — Buyer party identifier (BT-500 series) and Contracting
  Authority identification (BT-505 family).

Re-pin trigger is event-driven per ADR 0034 §2.

Sources:

- UBL 2.3 `cac:Party` — https://www.datypic.com/sc/ubl23/e-cac_Party.html
- OCDS `parties` — https://standard.open-contracting.org/latest/en/schema/reference/

## Bottom line

**GREEN.** Mapping is feasible. The current canonical `Buyer` (9
columns at `packages/db/src/schema/index.ts:126`) is a small,
intentionally lean entity; UBL `cac:Party` is much richer (23 child
elements). The ratio inverts the previous crosswalks: instead of
"how do we map our rich shape to UBL," it's "how much of UBL do we
absorb into our canonical Buyer." On the 9 existing canonical fields the
distribution is **6 exact / 2 partial / 0 novel / 1 platform-internal**
out of 9 (≈89% UBL-aligned on procurement-meaningful fields). The
relevant question is which of the ~14 UBL/OCDS Party fields we **don't**
have are worth promoting to canonical Buyer columns vs. leaving in the
`fields` spillover or a future side-table.

**Recommendation: ship the canonical `Buyer` mostly as-is, with two
small additions** to close obvious gaps for the layered-on-UBL emit
path: a stable `legalIdentifier` (UBL `cac:PartyIdentification` /
OCDS `identifier.id`), and a structured `address` extension (UBL
`cac:PostalAddress` / OCDS `address`). Both are nullable; existing
rows remain unchanged.

## Existing canonical `Buyer` shape

```ts
{
  id: uuid;                    // platform-internal
  canonicalName: text;         // primary display name
  buyerType: text;             // 'federal_agency' | 'state_authority' | 'enterprise' | ...
  regime: text;                // 'us_federal' | 'uk_public' | 'eu_oj' | 'enterprise' | ...
  parentBuyerId: uuid | null;  // self-referencing for sub-agency hierarchy
  countryCode: text | null;    // ISO 3166-1 alpha-2
  regionCode: text | null;     // ISO 3166-2 (sub-national)
  sourceConfidence: numeric;   // platform-internal
  createdAt, updatedAt: timestamptz;
}
```

Unique on `(canonicalName, regime)`.

## Proposed two additions

```ts
{
  // ... existing 9 columns ...
  legalIdentifier: text | null; // UBL cac:PartyIdentification/cbc:ID, OCDS identifier.id
  legalIdentifierScheme: text | null; // UBL listID, OCDS identifier.scheme — e.g. 'GLN' | 'DUNS' | 'EU-VAT' | 'US-DUNS' | 'US-CAGE' | 'US-UEI'
}
```

The structured address belongs on a separate `buyer_address` extension
table (1:N for buyers with multiple offices); see the §Future
follow-ons section. Adding it to canonical `buyer` directly would
collapse the multi-office case the moment a buyer has more than one
contracting office.

## Field-by-field crosswalk

| #   | Our field                     | UBL `cac:Party`                                                                 | OCDS `parties[]`                            | eForms                   | Status                | Note                                                                                                                                                                                                                           |
| --- | ----------------------------- | ------------------------------------------------------------------------------- | ------------------------------------------- | ------------------------ | --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1   | `canonicalName`               | `cac:PartyName/cbc:Name` `[0..*]`                                               | `name`                                      | BT-500                   | **partial**           | UBL allows multiple party names (multilang); we ship single-string per ADR 0034.                                                                                                                                               |
| 2   | `buyerType`                   | `cac:PartyLegalEntity/cbc:CompanyTypeCode` (closest analog)                     | `details.partyClassification` (extension)   | BT-508 (Buyer Type Code) | **partial**           | Our taxonomy is procurement-flavoured (`federal_agency`, `enterprise`, etc.); UBL/eForms use legal-entity codes. Cross-map deferred (low load on emit-side).                                                                   |
| 3   | `regime`                      | n/a                                                                             | n/a                                         | n/a                      | **platform-internal** | Procurement regime, not a Party attribute.                                                                                                                                                                                     |
| 4   | `parentBuyerId`               | `cac:AgentParty` (loose analog) or `cac:PartyLegalEntity/cac:HeadOfficeAddress` | `parties[].roles` + relationship extension  | BT-633 (rare)            | **partial**           | UBL has no clean "parent agency" pointer; we keep the self-ref FK and document it as a partial mapping. EU multi-buyer notices use `cac:AdditionalContractingParty[]` rather than parent pointers; reverse map collapses both. |
| 5   | `countryCode`                 | `cac:PostalAddress/cac:Country/cbc:IdentificationCode`                          | `address.countryName`                       | BT-514                   | **exact**             | ISO 3166-1 alpha-2 on our side; UBL allows multiple country codelists; OCDS uses country name (we keep code, reverse-map names).                                                                                               |
| 6   | `regionCode`                  | `cac:PostalAddress/cbc:CountrySubentityCode`                                    | `address.region`                            | BT-727 (NUTS for EU)     | **partial**           | ISO 3166-2 subdivision; eForms uses NUTS codes for EU. Cross-map collapses to ISO 3166-2 with NUTS preserved in `legalIdentifier` only when source provides it. Future: dedicated NUTS column if EU adoption demands.          |
| 7   | `sourceConfidence`            | n/a                                                                             | n/a                                         | n/a                      | **platform-internal** | Confidence on canonical resolution; not emitted.                                                                                                                                                                               |
| 8   | `createdAt` / `updatedAt`     | n/a                                                                             | n/a                                         | n/a                      | **platform-internal** | Audit timestamps.                                                                                                                                                                                                              |
| 9   | `legalIdentifier` (new)       | `cac:PartyIdentification/cbc:ID` `[0..*]`                                       | `identifier.id` + `additionalIdentifiers[]` | BT-501 (organization id) | **exact**             | Stable cross-source identifier (DUNS, GLN, EU-VAT, UEI, CAGE). Promotes once; secondary identifiers spill to `fields` per adapter.                                                                                             |
| 10  | `legalIdentifierScheme` (new) | `listID` attribute on `cbc:ID`                                                  | `identifier.scheme`                         | BT-502                   | **exact**             | Codelist discriminator.                                                                                                                                                                                                        |

## UBL elements we deliberately omit

| UBL element                                                                                | Why we omit                                                                                                                                                                       | Where it lives in our model instead                                                                      |
| ------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| `cbc:WebsiteURI`, `cbc:LogoReferenceID`, `cac:AdditionalWebSite`, `cac:SocialMediaProfile` | Marketing / metadata; not procurement-meaningful for our consumers.                                                                                                               | `fields` spillover when an adapter surfaces them.                                                        |
| `cbc:EndpointID` (Peppol participant id)                                                   | Peppol routing; only relevant if BidAngel ever joins the Peppol network (ADR 0034 §7 explicitly defers).                                                                          | `fields.peppolEndpointId` when the participation case lands.                                             |
| `cbc:IndustryClassificationCode`                                                           | Buyer's own industry classification — distinct from the opportunity's commodity classification.                                                                                   | `fields` spillover; promote if a downstream consumer surfaces the need.                                  |
| `cac:PartyTaxScheme`                                                                       | Tax registration; relevant for invoicing, not for opportunity discovery.                                                                                                          | Out of scope; would land on a future Invoicing slice.                                                    |
| `cac:Contact` (primary contact for the party)                                              | Buyer-side contact information — falls into the `opportunity_source_metadata` `point_of_contact` kind on slice 71 since contacts are usually opportunity-scoped not buyer-scoped. | `opportunity_source_metadata` `metadata_kind='point_of_contact'`.                                        |
| `cac:Person` `[0..*]`                                                                      | Multiple persons associated with a party.                                                                                                                                         | Same as `cac:Contact`.                                                                                   |
| `cac:AgentParty`, `cac:ServiceProviderParty`                                               | Procurement-agent / outsourced procurement-service relationships.                                                                                                                 | Out of scope; modelled via `parent_buyer_id` only when the agent IS the contracting authority of record. |
| `cac:PowerOfAttorney`, `cac:PartyAuthorization`                                            | Power-of-attorney relationships.                                                                                                                                                  | Out of scope (legal-procedural; not load-bearing for our consumers).                                     |
| `cac:FinancialAccount`                                                                     | Financial account; relevant for invoicing, not procurement discovery.                                                                                                             | Out of scope.                                                                                            |
| `cac:PostalAddress` (full structure with street / building / room)                         | Useful but multi-office buyers are common; collapses if put on canonical buyer.                                                                                                   | Future `buyer_address` extension table; see §Future follow-ons.                                          |

## Open questions to resolve

1. **Should `legalIdentifier` be unique on the buyer table?** A buyer
   with a known DUNS / EU-VAT / UEI is uniquely identified by that
   number across regimes; allowing duplicates is a data-quality
   problem. But not every buyer carries a stable identifier (small
   municipalities often don't), so a hard uniqueness constraint
   conflicts with NULL-tolerant ingestion.
   - **(A)** Add a partial unique index on `(legalIdentifier,
legalIdentifierScheme)` WHERE `legalIdentifier IS NOT NULL`.
     Strong data quality; tolerates NULLs.
   - **(B)** No uniqueness; rely on application-level deduplication.
     Simpler; weaker quality.
   - **Recommendation: (A).** Same shape as the
     `(canonicalName, regime)` unique constraint already on the table;
     the partial index covers the NULL case cleanly.

2. **Multi-office buyers — promote `buyer_address` now or defer?**
   The opportunity probe noted that finer geographic detail flows
   through `opportunity_source_metadata`. But buyer-side addresses
   (the contracting authority's mailing address) live nowhere
   structured today.
   - **(A)** Add a `buyer_address` extension table now (1:N), with
     `kind` ('headquarters' | 'mailing' | 'invoicing' | 'physical')
     and structured fields (`street`, `city`, `postalCode`,
     `regionCode`, `countryCode`).
   - **(B)** Defer until a downstream surface needs structured buyer
     address; today it can live in `fields` spillover for adapters
     that ingest UBL.
   - **Recommendation: (B).** No current downstream consumer demands
     structured buyer address (the canonical row's `countryCode` /
     `regionCode` already covers the geographic-routing use case).
     Promote when the consumer lands.

3. **Tenant-scoped buyer overrides — out of scope or surface here?**
   Some tenants want to record their own canonical name for a buyer
   that disagrees with the platform's (e.g. "we always call Department
   of X 'DoX'"). Today no such surface exists.
   - **(A)** Add a `tenant_buyer_alias` table now.
   - **(B)** Defer entirely; this is a UI / display-layer concern
     downstream of the canonical row.
   - **Recommendation: (B).** Tenant-scoped display preferences are
     downstream of the canonical model and should not pollute the
     shared-market `buyer` table. Closes per ADR 0005's shared / tenant
     boundary.

## What this probe says about the rest of the work

- The Buyer crosswalk is the smallest of the per-fragment crosswalks
  authored to date — most of UBL `cac:Party`'s 23 child elements are
  deliberately out of scope for the procurement-discovery use case
  this platform serves.
- The two-column addition (`legalIdentifier` +
  `legalIdentifierScheme`) is the only schema change recommended; the
  rest of UBL Party richness lives in `fields` or future side-tables.
- The Party crosswalk applies symmetrically to the **supplier** side
  too (capability_claim's `subjectParty`, ADR 0025); the supplier
  crosswalk reuses the same `cac:Party` mapping and inherits this
  document's UBL element coverage.

## Deltas — resolved 2026-05-08 (Q1=A, Q2=B deferred, Q3=B deferred)

| #   | Delta                                                                                                                                                                                                                                                                                                                                                                                          | Status                   |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------ |
| 1   | Migration `0045_buyer_legal_identifier.sql` — adds `legal_identifier` + `legal_identifier_scheme` columns to `buyer`, with partial unique index on `(legal_identifier, legal_identifier_scheme)` WHERE NOT NULL (Q1=A).                                                                                                                                                                        | **shipped** (2026-05-08) |
| 2   | Updated `packages/domain/src/types/opportunity.ts` (the `Buyer` interface) and `packages/db/src/schema/index.ts` (the `buyer` pgTable) to expose the new columns. Also extended the `buyerRef` structure on `CanonicalOpportunityFragment` (`packages/structured-rfx-adapters/src/contract.ts`) so adapters can pass the legal identifier through to the orchestrator's buyer-resolution step. | **shipped** (2026-05-08) |
| 3   | Slice 71 (SAM.gov UEI / CAGE → `legalIdentifierScheme = 'US-UEI'` / `'US-CAGE'`) and slice 73 (Ariba ANID → `'Ariba-ANID'`) doc references added. Slice 79 (OCDS) inherits via the documented `parties[].identifier` mapping in this crosswalk; explicit slice-79 reference deferred until that slice surfaces concrete OCDS adapters.                                                         | **shipped** (2026-05-08) |
| 4   | This probe doc's deltas table — see this section.                                                                                                                                                                                                                                                                                                                                              | **shipped** (2026-05-08) |

## Future follow-ons (not in this probe's scope)

- Buyer-side `buyer_address` extension table (Q2=B, deferred).
- Tenant-scoped buyer alias surface (Q3=B, deferred).
- Buyer-type cross-map (eForms BT-508 codelist ↔ canonical `buyerType`).
- Supplier-side reuse of the Party crosswalk for `capability_claim.subjectParty`.

## Out of scope for this probe

- Supplier-side party (ADR 0025 `capability_claim.subjectParty` ↔
  UBL `cac:Party`). The same crosswalk applies but the consumer set is
  different; document that in a separate artifact when slice 85's
  buyer-facing read of the supplier graph activates.
- `cac:Contact` / `cac:Person` mapping — handled by
  `opportunity_source_metadata` `point_of_contact` per slice 71.
- Round-trip CI test against an EU eForms F02 buyer party fixture.
