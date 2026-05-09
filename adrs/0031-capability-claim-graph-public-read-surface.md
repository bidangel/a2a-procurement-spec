# ADR 0031: Capability Claim Graph Public Read Surface for Authorised Buyer Agents

## Status

Accepted (2026-05-04 — slice 85 ship). Authored alongside slice 85 (`docs/slices/slice-85-capability-claim-graph-public-read-endpoints.md`); slice 85 was the first implementation of the principles recorded here and shipped them: the v2 `external_buyer_credential.scope` extension with the optional `claimGraph` clause, three buyer-facing read endpoints under `/api/v1/buyer-facing/claim-graph/*`, two new audit event types (`claim_graph_read_completed`, `claim_graph_read_rejected`), per-response JWS signing via the slice-74 `signTenantEnvelope` helper, and the marked-conflict redaction discipline (per ADR 0025 §5). Builds on ADR 0029 (`docs/adr/0029-public-manifest-read-surface-for-authorised-buyer-agents.md`) by extending the buyer-facing read surface from per-package manifest reads to direct claim-graph queries that do not require a pursuit or submission package.

The Phase B counterparty-gate amendment below records the operational gating posture; the durable architectural design recorded here was implemented as written.

Open follow-ups (tracked separately, do not affect this ADR's principles): the per-tier credential allowance enforcement (`enforceCredentialQuota` precheck) and the agent-ops-meter wiring for claim-graph reads at the discount weight (per `project_pricing_two_flow_levers.md`). Both are billing-side, not architectural.

**Amended 2026-05-05 by ADR 0032** (M2M Roadmap Pause). Two amendments apply; the durable design is unchanged:

- **Directory carve-out.** ADR 0032 §3 narrows Alt F's rejection ("Cross-supplier discovery via a platform-hosted directory — rejected per pushback #5") and the corresponding entry under "Things this ADR explicitly does not commit to" to a narrower rule: no platform-aggregated cross-supplier search, scoring, or ranking; supplier-controlled opt-in directory is permitted under ADR 0033. The claim graph remains supplier-credentialled and per-resource-scoped exactly as recorded here; the directory is a separate, non-cryptographic surface — buyers who arrive via the directory and want claim-graph access still obtain a `bbk_*` credential per §1 / §2 below.
- **Phase B counterparty gate.** ADR 0032 §1 gates slice 85's ship on a written buyer-side counterparty commitment. This ADR's design remains the eventual shape; only the ship date is gated.

## Context

ADR 0029 closed the long-standing buyer-facing read deferral by exposing **signed submission package manifests** to authorised buyer agents under supplier-issued credentials. That surface serves Flow 1 (RFx response): the buyer publishes an RFx, the supplier assembles a signed package, the buyer agent reads and verifies it.

The strategic synthesis behind the M2M roadmap names a second flow that ADR 0029 does not yet serve. The pitch deck (`docs/pitch/slide-06-how-it-works.md`) and the marketing site's two-flows section name it directly: a buyer-side agent forming an intent — capability shortlist, pre-RFx market intelligence, direct vendor discovery — and querying the supplier's published capability graph **before any RFx is written**. The supplier graph that Flow 1 produces (claims, evidence links, freshness, jurisdiction coverage, category mappings) is the substrate Flow 2 queries.

The architecture today is asymmetric. The capability graph exists (ADR 0025 / slice 68). The graph is signed when serialised into a submission package's manifest (slice 69). The credential model exists (ADR 0029 / slice 82). But the only path a buyer agent has to read the graph is to fetch a package's manifest — which means the graph is only legible **after** a pursuit has been run and a package assembled. A buyer agent that has not yet seen the supplier in any RFx context has no read path. That is exactly the asymmetry Flow 2 needs to close.

The decision is whether to extend ADR 0029's surface with a second read mode (claim-graph queries scoped per-credential, bounded per-resource, signed per-response, redaction-aware in the same shape as manifest reads) or to ship a separate read surface with its own credential model. This ADR records the former and pins the durable consequences. It is warranted because the credential-model decision binds every later buyer-facing slice and an inconsistent split between manifest reads and claim-graph reads would force buyer-side integrators to manage two parallel auth surfaces.

The strategy synthesis user-pushback points apply directly:

**Pushback #2 (cautious buyer-side expansion):**

> Do not build a broad buyer-side product yet. Build buyer-side-compatible primitives. […] preserve seller-side optionality, do not get pulled into buyer-side commitments before the architecture is ready.

ADR 0031 stays inside the disciplined buyer-side-compatible-primitives posture. It does **not** publish supplier capabilities to the world; it does **not** expose cross-supplier search; it does **not** introduce platform-side aggregation. It exposes a per-supplier, supplier-credentialled, scoped read surface — the same posture ADR 0029 holds, applied to a different resource shape.

**Pushback #5 (no platform-issued reputation scores):**

> Tenant-controlled trust packets, not platform-generated public reputation scores.

ADR 0031 commits to: the supplier controls which credentials see which slices of the claim graph, with what redaction policy, at what freshness floor, with what rate envelope. The platform operates the read surface but never publishes, scores, indexes, ranks, or aggregates the claim graph. There is no path to discover suppliers other than via a supplier-issued credential.

The reserved-slot discipline that ADR 0022 §2 codified, and that ADR 0029 §"Forward-compatibility consequences" extended, anticipated this ADR explicitly:

> A future "scoped clarification posting from buyer-side" […] layers on a new buyer-facing write endpoint with stricter authorisation.

ADR 0031 is the read-side analogue: a new buyer-facing read endpoint family that layers on the same credential model, scope shape, redaction model, audit attribution, and bearer-prefix routing already established by ADR 0029.

## Decision

### 1. Claim-graph reads are a new scope mode on the existing buyer credential

The `external_buyer_credential` entity from ADR 0029 §1 is **extended**, not replaced. A new optional field is added to the `scope` jsonb: `claimGraph` (described in §2). A credential whose scope contains only `submissionPackages` / `pursuits` / `opportunities` continues to authorise only manifest reads; a credential whose scope contains `claimGraph` authorises claim-graph reads; a credential containing both authorises both.

The shape:

```jsonc
{
  "v": 2, // bumped from 1; v1 credentials are auto-upgraded with no claim-graph access
  "submissionPackages": ["uuid", "..."],
  "pursuits": ["uuid"],
  "opportunities": ["uuid"],
  "publicKeyAccess": "any | restricted_to_referenced_packages",
  "revocationListAccess": "any | restricted_to_referenced_packages",
  "fieldRedactionPolicy": "full | redacted_audit_excerpt | redacted_evidence_manifest", // applies to manifest reads only
  "claimGraph": {
    // present iff claim-graph access is granted; see §2
  },
}
```

The `bbk_*` bearer prefix from ADR 0029 §7 is unchanged. The bearer-header sniffer routes `bbk_*` to the buyer-facing surface as today; the new claim-graph endpoints sit under `/api/v1/buyer-facing/claim-graph/*` and resolve the credential via the same auth helper. The supplier-side UI (Settings → Buyer-facing access) is extended to issue, edit, and rotate credentials with a `claimGraph` scope clause.

The reason to extend rather than fork: a credential carries a single buyer relationship. A buyer agent that needs both to verify a signed package and to query the supplier's graph holds one credential, not two. Splitting the surface across two credential families would force suppliers to issue and manage parallel credentials and would drift the audit attribution across two event families. Extension preserves the single credential per buyer relationship.

### 2. Claim-graph scope is declarative and bounded

The `scope.claimGraph` clause declares exactly which slices of the graph are accessible:

```jsonc
{
  "v": 1,
  "kinds": [
    "certification",
    "past_performance",
    "capability_statement",
    "jurisdiction_coverage",
    "other",
  ],
  // explicit allowlist of claim kinds; default = empty (deny all)
  "categoryFilters": {
    "naics": ["541512", "541519"], // only claims tagged with these NAICS codes
    "unspsc": ["80101500"], // only claims tagged with these UNSPSC codes
    "tenantCustom": ["security-compliance"], // only claims with these tenant-defined tags
  },
  "jurisdictionFilters": {
    "countries": ["US", "CA"], // ISO 3166-1 alpha-2 country codes
    "regulatoryRegimes": ["FedRAMP", "ITAR"], // free-form; matched exact-string against claim's regulatory_regime
  },
  "freshnessFloor": "current | aging | stale",
  // claims with freshness worse than this floor are excluded from reads;
  // default = "stale" (excludes only `expired`)
  "evidenceDereferencing": "redact_all | redact_asset_id_only | reveal_metadata",
  // controls what the evidence manifest looks like in claim responses; see §3
  "specificClaimIdAllowlist": ["uuid", "..."],
  // optional explicit allowlist; if present, only these claim ids are readable
  // regardless of the kinds/categoryFilters/jurisdictionFilters above
  "specificClaimIdDenylist": ["uuid", "..."],
  // optional explicit denylist; takes precedence over all other filters
  "responseSizeCap": {
    "maxClaimsPerResponse": 100, // pagination is mandatory; cap default 100
    "maxEvidenceLinksPerClaim": 10, // cap on per-claim evidence link expansion
  },
}
```

The filter composition rule is **AND across filter families, OR within a family**: a claim is readable iff it matches at least one entry in `kinds` AND at least one entry in `categoryFilters.naics` (if `naics` is non-empty; or `categoryFilters.unspsc`, etc.) AND at least one entry in `jurisdictionFilters.countries` (if non-empty) AND its computed freshness is `freshnessFloor` or better AND it is not in `specificClaimIdDenylist` AND (if `specificClaimIdAllowlist` is non-empty) it is in that allowlist.

Empty filter families (e.g. `categoryFilters.naics: []`) mean "no NAICS constraint" — they do not constrain. The empty-family-means-unconstrained rule mirrors the well-trodden firewall pattern; the pre-issuance UI surfaces a warning when the operator's filters would expose more than they likely intend.

A claim that is internally marked-conflicting (per ADR 0025 §5) is **never readable by a buyer credential** regardless of scope. Marked conflicts are a supplier-internal review state; exposing conflicting pairs to a buyer would propagate internal disagreement into the trust packet. The buyer-side query result includes a `conflictsRedacted: bool` flag at the response level (not per-claim) when at least one matching claim was excluded due to a marked conflict, so the buyer agent knows the result is not exhaustive without learning which claim was affected.

### 3. Evidence dereferencing has three modes, mirroring ADR 0029's redaction discipline

Each claim has linked evidence (`content_asset` rows). The three `evidenceDereferencing` modes control how evidence appears in the response:

- **`redact_all`** — claims appear with no evidence references at all. The buyer sees the claim's structured fields (kind, jurisdiction, freshness, category mappings, valid_until) and the evidence-link **count** but not the asset ids or metadata. Use case: maximum non-disclosure; the buyer learns the supplier has the capability but not how it is documented.
- **`redact_asset_id_only`** — claims appear with evidence references replaced by stable opaque tokens (the same `evidence-token-{sha256(asset_id)[0:8]}` pattern slice 82 uses for `redacted_evidence_manifest`), plus per-link metadata (the `evidence_role: primary | supporting | corroborating` from ADR 0025 §3). The buyer can reason about evidence cardinality and roles but cannot dereference to specific assets. Use case: typical pre-RFx discovery posture.
- **`reveal_metadata`** — claims appear with full per-evidence metadata: title, kind, attestation date, attestor, with the asset id still represented as a stable opaque token (asset ids are tenant-internal and never exposed). The buyer can verify that the evidence exists and was attested at a plausible time. Use case: late-stage qualification where the buyer needs to assess evidence sufficiency.

In **no mode** does the buyer-facing claim-graph response expose `content_asset` ids, asset binary contents, or the `evidence_manifest.json` slice-65 produces. Asset bytes remain inside the supplier tenant. The buyer agent that needs to inspect the underlying evidence must obtain it through a separate channel (typically a manifest read on a related submission package, governed by ADR 0029's redaction policy).

### 4. Claim-graph responses are signed; the JWS envelope mirrors ADR 0027's discipline

Every claim-graph read response carries a signed JWS envelope covering the canonical bytes of the response payload. The signature uses the tenant's signing key (ADR 0027), produced at read time by `signTenantEnvelope` (slice 74's helper). The envelope shape:

```jsonc
{
  "v": 1,
  "tenantId": "uuid",
  "responseKind": "claim_graph_query | claim_detail",
  "credentialId": "uuid",
  "credentialFingerprint": "sha256(secret_prefix + scope_canonical_bytes)[0:16]",
  "responseSha256": "sha256(canonical bytes of response payload, with signature: null)",
  "issuedAt": "2026-05-04T12:34:56Z",
  "ttl": "PT5M",
  "supplierLabel": "string",
  "scopeFingerprint": "sha256(canonical bytes of credential scope at sign time)[0:16]",
}
```

Rationale for each field:

- `responseKind` lets a buyer-side verifier reject envelopes used in the wrong context (a claim-detail response presented as a query response, etc.).
- `credentialFingerprint` and `scopeFingerprint` let a buyer-side audit reason about "the supplier signed this response under the same credential we believe we are using" without exposing the credential secret.
- `responseSha256` is the load-bearing field; it covers the response payload bytes and is what the verifier recomputes to validate.
- `issuedAt` and `ttl` are advisory only — JWS envelopes do not expire cryptographically; the supplier signals freshness so a buyer agent caching responses knows when to re-read. The platform does not enforce TTL on the read surface; the buyer agent enforces it on its own cache.

The signature is over the JWS envelope, not over the response payload directly. This mirrors ADR 0027 §3's "sign an envelope referencing the artifact by hash, not the artifact itself" discipline. The reason: response payloads can be very large (up to `responseSizeCap.maxClaimsPerResponse` claims plus evidence metadata), and signing the envelope-with-hash keeps signing operations small (one KMS call per response, regardless of response size).

The verifier path (buyer-side):

1. Fetch the response with `Authorization: Bearer bbk_...`.
2. Parse the response. The payload includes `signature` (the JWS Compact Serialization) and `signatureEnvelope` (the JWS payload).
3. Compute `sha256(canonical bytes of response payload with signature: null and signatureEnvelope: null)`. Compare to `signatureEnvelope.responseSha256`.
4. Verify the JWS signature using the supplier's public key (fetched separately via the existing `/api/v1/buyer-facing/signing-keys` endpoint per ADR 0029 §"Buyer-facing read endpoints"). The public key is keyed by tenant id; the buyer caches it.
5. Optionally check `issuedAt` / `ttl` against the buyer's freshness policy.

If any step fails, the response is unverified and the buyer's policy decides what to do (typically: discard).

The signing happens on every claim-graph read. The cost is one KMS sign per response (Ed25519 over a small envelope; sub-millisecond plus network). The slice-85 NFR absorbs this; high-traffic claim-graph credentials may want monitoring.

### 5. Read endpoints

Three endpoints under `/api/v1/buyer-facing/claim-graph/*`. All require a valid `bbk_*` bearer credential whose scope contains a `claimGraph` clause. Responses are JSON with the signed envelope embedded; `Content-Type: application/vnd.bidangel.capability-claim-graph+json; version=1`.

- `GET /api/v1/buyer-facing/claim-graph/claims` — list / paginate claims matching the credential's scope filters. Query parameters refine within the scope (e.g. `?kind=certification&jurisdictionCountry=US`); query parameters cannot **expand** beyond the credential's scope, only narrow within it. Pagination via opaque cursor; max page size = `scope.claimGraph.responseSizeCap.maxClaimsPerResponse`. Returns 200 with the signed response payload; 403 with `claim_graph_not_in_scope` if the credential lacks claim-graph access; 401 / 429 / 412 per the ADR 0029 envelope.
- `GET /api/v1/buyer-facing/claim-graph/claims/:claimId` — fetch a single claim's detail with evidence dereferencing per the credential's mode. Returns 404 (not 403) if the claim id does not exist in the supplier tenant (this is consistent with "the buyer knows or guesses the claim id from a list response"); 403 with `claim_not_in_scope` if the claim exists but is not within the credential's scope.
- `POST /api/v1/buyer-facing/claim-graph/query` — structured query with a request body more expressive than query strings can carry. Body shape:

```jsonc
{
  "v": 1,
  "kinds": ["certification"],
  "naicsAny": ["541512"],
  "unspscAny": ["80101500"],
  "jurisdictionCountriesAny": ["US"],
  "regulatoryRegimesAny": ["FedRAMP"],
  "freshnessFloor": "current",
  "validOnDate": "2026-05-04",
  "categoryHierarchyExpansion": "exact", // MVP only supports exact; "ancestors" / "descendants" are future
  "cursor": "opaque",
  "pageSize": 50,
}
```

The body's filters compose with the credential's scope using the same AND-across-families / OR-within-a-family rule. The body cannot widen the scope; it can only narrow within it. This endpoint is the workhorse for buyer-agent shortlisting workflows.

The discovery endpoint from ADR 0029 §4 is extended to advertise these new endpoints when the discovering tenant has at least one credential with a `claimGraph` scope clause:

```jsonc
{
  "v": 1,
  "supplierTenantId": "uuid",
  "supplierLabel": "string",
  "endpoints": {
    "manifest": "https://api.bidangel.com/api/v1/buyer-facing/submission-packages/{packageId}/manifest",
    "publicKeys": "https://api.bidangel.com/api/v1/buyer-facing/signing-keys",
    "revocationList": "https://api.bidangel.com/api/v1/buyer-facing/signing-keys/revocation-list",
    "claimGraphList": "https://api.bidangel.com/api/v1/buyer-facing/claim-graph/claims",
    "claimGraphDetail": "https://api.bidangel.com/api/v1/buyer-facing/claim-graph/claims/{claimId}",
    "claimGraphQuery": "https://api.bidangel.com/api/v1/buyer-facing/claim-graph/query",
  },
  "authentication": "bearer-bbk-prefix",
  "supportedManifestVersions": ["v1"],
  "supportedClaimGraphVersions": ["v1"],
}
```

The discovery endpoint remains publicly accessible (no credential required) per ADR 0029 §4. It does not list which claims exist, only which endpoints are available; the per-credential scope determines what each buyer agent can actually read.

### 6. Rate limit and quota mirror ADR 0029, with claim-graph-specific weights

`external_buyer_credential_usage` (slice 82) is the same rolling-window table; no new schema is introduced. The `weighted_cost` model from ADR 0029 §5 is extended with claim-graph weights:

- Claim-graph list / query (paginated, up to 100 claims): weight 3.
- Claim-graph detail (single claim with evidence dereferencing): weight 1.
- Manifest read: weight 1 (unchanged).
- Manifest read + bundle: weight 5 (unchanged).
- Public key set read: weight 1 (unchanged).
- Revocation list read: weight 1 (unchanged).

The weighting reflects that claim-graph queries do more database work than manifest reads (the matcher executes joins across `capability_claim`, `capability_claim_category`, `capability_claim_jurisdiction`, and `capability_claim_evidence`) but the supplier-side cost is bounded by the response size cap.

Per-credential rate limit defaults at issuance for claim-graph credentials: 30 requests/minute (lower than ADR 0029's 60 because each claim-graph read is heavier), 500 weighted operations/day, 15000/month. Tenants can adjust per credential.

### 7. Audit attribution extends ADR 0029's event family

The `external_buyer_credential.*` event family from ADR 0029 §6 is extended with claim-graph-specific event types. The lifecycle events (`issued`, `rotated`, `revoked`) are unchanged. New per-read events:

- `external_buyer_credential.claim_graph_read_completed` — payload: `{ credentialId, buyerOrganisationLabel, requestedKind: 'list' | 'detail' | 'query', filterFingerprint: 'sha256(canonical bytes of effective filters)[0:16]', claimsReturned: number, conflictsRedacted: bool, evidenceDereferencingMode: 'redact_all' | 'redact_asset_id_only' | 'reveal_metadata', latencyMs, weightedCost }`.
- `external_buyer_credential.claim_graph_read_rejected` — payload: `{ credentialId (when known), reason: 'unauthenticated' | 'unauthorized' | 'claim_graph_not_in_scope' | 'claim_not_in_scope' | 'rate_limited' | 'quota_exceeded' | 'invalid_query', latencyMs }`.

The `filterFingerprint` is the supplier's audit handle for "what did this buyer query?" without exposing the buyer's specific filter values to anyone but the supplier (the supplier sees the canonical bytes; the audit log sees the fingerprint). A future enrichment could expose the full canonical bytes to the supplier-side audit detail view.

The supplier's audit log thus shows every buyer-agent query, including which kinds / categories / jurisdictions were searched and how many claims came back. The buyer agent operating outside the platform does not see the audit (consistent with ADR 0029 §6).

### 8. Versioning: claim-graph is `v1`; future versions ship at `/api/v2/buyer-facing/claim-graph/*`

The discovery endpoint's `supportedClaimGraphVersions` field is the shape under which forward compatibility is preserved. The MVP is `v1`. A future `v2` (e.g. supporting category-hierarchy expansion, semantic-similarity matching, or W3C Verifiable Credentials wrapping) ships under `/api/v2/buyer-facing/claim-graph/*` with a new response payload shape and is advertised additively. Buyer agents pinned to `v1` continue to work; those that opt into `v2` get the new shape.

ADR 0022 §2's reserved-slot discipline applies to the response payload: schema additions to `v1` are additive (new fields default to absent for older consumers); breaking changes ship as `v2`.

## Consequences

### Direct consequences (slice 85 implements)

- The `external_buyer_credential.scope` schema gains a `v: 2` and an optional `claimGraph` clause. Existing v1 credentials are auto-upgraded with no claim-graph access (`claimGraph` absent → no change in behaviour).
- Three new buyer-facing read endpoints under `/api/v1/buyer-facing/claim-graph/*`.
- The discovery endpoint advertises claim-graph endpoints when the supplier has at least one credential with a `claimGraph` scope clause.
- The `signTenantEnvelope` helper from slice 74 is invoked on every claim-graph read.
- The `external_buyer_credential.*` audit family gains two new event types.
- The supplier-side UI gains a `claimGraph` scope clause editor (kinds, category filters, jurisdiction filters, freshness floor, evidence dereferencing mode, response size cap, claim-id allow/deny lists).
- One new Class A read tool for the supplier to inspect claim-graph read activity (`listClaimGraphReadActivity`).

### Forward-compatibility consequences (this ADR's load-bearing reason)

- **Category-hierarchy expansion** (NAICS / UNSPSC ancestor / descendant matching) is forward-compatible via the `categoryHierarchyExpansion` field, currently fixed to `"exact"`.
- **Semantic-similarity matching** (free-text query against capability statements) is forward-compatible as a `v2` shape; not in MVP.
- **W3C Verifiable Credentials wrapping** of the claim-graph response is a `v2` shape.
- **Federated buyer identity** (one credential, many suppliers) — same forward-compatibility posture as ADR 0029.
- **Webhooks for buyer-agent queries** ("notify the supplier when a buyer searches for X") — not in MVP; layers on the audit data.
- **Cross-tenant claim-graph search** is **declined** per pushback #5 — not forward-compatible by design.

### Costs and trade-offs accepted

- **Sign-per-read cost.** Claim-graph reads pay one KMS sign each. At AWS KMS pricing this is sub-millisecond and well below a tenth of a cent per read; the cost is negligible at MVP traffic levels but worth monitoring at scale. Caching responses at the supplier side is forward-compatible if cost forces it.
- **No supplier directory.** A buyer agent cannot ask "find me all suppliers with FedRAMP Moderate certification." This forecloses some discovery use cases but preserves the supplier-controlled posture per pushback #5. The buyer agent must already know the supplier's tenant id; the supplier shares it with their authorised buyers.
- **Conflicts are not exposed.** A claim with a marked conflict is never returned to a buyer credential, even if it would otherwise match. The `conflictsRedacted` response flag tells the buyer that some matching claims were withheld. This costs transparency for internal-disagreement protection; the trade is accepted because conflicts are a review-state signal, not a public claim shape.
- **Filter expressivity is bounded.** No free-form regex, no full-text search, no nested boolean expressions. The MVP filter is the union of structured fields from ADR 0025. Richer query shapes are a `v2` decision, not a v1 hack.
- **Evidence asset bytes never leave the supplier tenant.** A claim-graph read's most expansive evidence dereferencing (`reveal_metadata`) shows metadata only; the asset bytes themselves are never readable through this surface. A buyer agent that needs to inspect an asset must obtain it through a manifest read on a submission package that cites the asset, governed by ADR 0029's redaction policy. This is intentional asymmetry: the claim graph is the surface of capability assertions; the manifest is the surface of bundled bid responses; assets are tenant-internal and only ever leave via signed bundle exports.

### Things this ADR explicitly does not commit to

- **A supplier directory or marketplace** (per pushback #5).
- **Cross-supplier claim-graph search** (per pushback #5).
- **Aggregate platform-side analytics on buyer reads** (privacy-sensitive; deferred).
- **Buyer-side write access of any kind** (read-only by design; write capabilities — clarification posting, etc. — go through ADR 0030 and a future buyer-facing scoped-write ADR).
- **Federated identity** for buyer agents (one credential, many suppliers).
- **OAuth flows** for buyer authentication (bearer-only in MVP, mirroring ADR 0029).
- **Bulk download** of all claims in a buyer's scope (paginated reads only).
- **Webhook notifications** to buyers when new claims are approved (pull-only in MVP).
- **Supplier-hosted discovery URLs** at the supplier's own DNS (forward-compatible mirror per ADR 0029).
- **Cross-tenant claim-graph queries** via federated identity (not in any future roadmap).
- **Auto-generated supplier "trust scores"** (declined permanently per pushback #5).
- **Live freshness streaming** (server-sent events / WebSockets for "claim X just refreshed") — pull-only in MVP.
- **Differential reads** ("show me what changed since my last read") — full reads only in MVP; forward-compatible via `cursor` enrichment.

## Alternatives considered

### Alt A — A separate buyer credential family (`bbg_*` for buyer claim-graph)

Rejected. The two surfaces serve the same buyer relationship and benefit from a single credential. Forking would duplicate every audit event, every UI affordance, and every rotation flow. The scope-extension model preserves the single-credential-per-buyer property and keeps the bearer-prefix routing simple.

### Alt B — Use the manifest endpoint with a synthetic "graph manifest" package

Considered. Every supplier could maintain a "capability passport" pseudo-package whose manifest is just the claim graph. Buyer agents could read it via the existing manifest endpoint. Rejected because:

- The manifest schema is anchored to a pursuit (slice 65 §"Schema"); pseudo-packages without pursuits would force schema drift.
- The redaction modes for manifests (`redacted_audit_excerpt`, `redacted_evidence_manifest`) are oriented around audit and evidence presentation in a bid context; they do not map cleanly to claim-graph access controls (kinds, categories, jurisdictions).
- The query expressivity needed for buyer-agent shortlisting (filter on category, freshness, jurisdiction) is not a manifest-read access pattern.

### Alt C — Public-by-default claim graph with optional supplier opt-out

Rejected per pushback #5. Same reasoning as ADR 0029 Alt A: default-on disclosure surfaces every supplier's capabilities to anyone who guesses a URL, which the supplier-credentialled posture forbids.

### Alt D — Sign individual claims rather than the whole response

Considered. Each claim could carry its own signature, so a buyer agent could verify subsets without re-fetching the whole response. Rejected because:

- The supplier's claim approval flow already signs individual claims at approval time (a future enrichment of slice 68; not in MVP). Re-signing the whole response is the single point of trust the buyer needs.
- Per-claim signature verification multiplies the buyer-side verification cost; one envelope per response is cheaper.
- The response-level envelope carries the response context (`responseKind`, `credentialFingerprint`, `scopeFingerprint`) that per-claim signatures cannot.

### Alt E — Allow the buyer to specify which claims to fetch by id (without filtering)

Considered. A buyer agent that has previously read the supplier's claim graph might want to re-fetch specific claim ids cheaply. Rejected as a primary mode because it reverses the "supplier-controlled scoping" posture: a buyer with a credential scoped to "FedRAMP certifications in CONUS" should not be able to fetch a SOC 2 claim by id even if they happen to know the id. The detail endpoint (`GET /claims/:claimId`) **is** in scope but its 403 response on out-of-scope claims preserves the posture; the buyer must hold a scope that includes the claim, not just the claim id.

### Alt F — Cross-supplier discovery via a platform-hosted directory

Rejected per pushback #5. Same reasoning as ADR 0029 Alt G.

### Alt G — Auto-derive scope from supplier-marked "public" claims

Considered. A supplier could mark certain claims as "public" and the credential model could auto-grant access. Rejected because:

- It blurs the per-credential discipline. ADR 0029's strength is that disclosure is per-buyer, not platform-wide.
- A "public" claim is structurally identical to "a claim accessible to a credential issued to anyone who asks" — the latter is more honest, more revocable, and more auditable.
- A future supplier-facing affordance ("issue a public-discovery credential") is forward-compatible via the existing scope shape; no schema or policy change required.

### Alt H — Per-buyer-organisation aggregate quotas (not just per-credential)

Considered. A buyer organisation operating multiple credentials (e.g. one per business unit) might want a single aggregate quota. Rejected for MVP because:

- ADR 0029's `buyer_organisation_label` is free-text only; promoting it to a structured entity is a deferred decision.
- Per-credential quotas are already protective; aggregate quotas can be modelled later if real demand emerges.

## Follow-on implications

- **Slice 85** is the first implementation. Schema migration (scope `v: 2` upgrade), three new endpoints, signing wiring, audit family extension, supplier-side UI extension, one new Class A read tool, M2M-roadmap composition test extension.
- **Future slice for category-hierarchy expansion** in claim-graph filters (NAICS / UNSPSC ancestor / descendant).
- **Future slice for semantic-similarity claim-graph queries** (free-text against capability statements; v2 shape).
- **Future slice for buyer-facing scoped writes** (clarifications under slice-78 model; would extend the bearer-prefix routing again).
- **ADR 0009 amendment** registers the two new event types in the buyer-credential family.
- **ADR 0022 follow-up** — none. The submission package manifest schema is unchanged; this ADR exposes a separate read surface.
- **ADR 0025 follow-up** — none. The capability claim model is unchanged; this ADR exposes a read surface over the existing schema.
- **ADR 0027 follow-up** — `signTenantEnvelope` is now invoked in a new context (per claim-graph read); the helper is target-agnostic so no change is needed, but the operational profile (signs per minute) shifts.
- **ADR 0029 amendment** — this ADR extends the `external_buyer_credential.scope` schema (v1 → v2) and the audit event family. ADR 0029 is updated by reference to ADR 0031 in its forward-compatibility consequences section; the original v1 scope shape is preserved for backwards compatibility.
- **M2M strategy roadmap** — adds Track 8 (capability claim graph public read surface) covering ADR 0031 + slice 85.

## References

- Slice 85 — `docs/slices/slice-85-capability-claim-graph-public-read-endpoints.md` (first implementation)
- ADR 0005 — shared vs tenant-owned data
- ADR 0009 — audit and event taxonomy
- ADR 0012 — external agent credentials and MCP adapter (the `bak_` prefix this ADR's `bbk_` companion was extended from)
- ADR 0018 — audit-trail immutability
- ADR 0022 — submission package as first-class artifact (the manifest's `claims[]` slot this ADR's read surface complements)
- ADR 0023 — agent persona model
- ADR 0024 — agent authority policy engine
- ADR 0025 — supplier capability claim model (the claim noun this ADR exposes)
- ADR 0027 — tenant-scoped signing keys (the keys signing claim-graph responses)
- ADR 0028 — commitment ladder + autonomy
- ADR 0029 — public manifest read surface (the credential, scope, redaction, discovery, audit, and bearer-prefix model this ADR extends)
- ADR 0030 — clarification exchange
- Slice 68 — capability claims schema (the underlying claim graph)
- Slice 69 — claims in submission package manifest (the prior buyer-visible surface for claims)
- Slice 74 / 75 — signing key infrastructure + signed manifests (the signing helpers reused)
- Slice 77 — buyer-agent query sandbox (the internal simulation that has been validating matcher behaviour; slice 85 productionises a subset of this)
- Slice 82 — public manifest read surface (the credential model this ADR extends)
- RFC 5785 — well-known URIs
- RFC 7515 — JSON Web Signature
- RFC 8785 — JSON Canonicalization Scheme
