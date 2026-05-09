# ADR 0026: Structured RFx Ingestion as a First-Class Peer to Document Ingestion

## Status

Accepted (2026-05-07). Slice 70 (`docs/slices/slice-70-structured-rfx-adapter-ddq-rows.md`) shipped the first implementation: `packages/structured-rfx-adapters/` houses the `StructuredRfxAdapter<TInput>` contract, the in-process registry, the orchestrator (with injectable persistence sink), and the three concrete adapters (CAIQ v4, SIG-Lite, SIG-Core). Migration `0043_slice_70_structured_rfx_adapters.sql` adds the `requirement_standard_schema` extension, the `produced_by_adapter_*` provenance discriminator on `requirement` / `opportunity` / `evaluation_criterion`, the `content_asset` standard-schema tag, and the `structured_rfx_adapter_run` per-run audit row. The orchestrator is wired into the slice-21 `parseQuestionnaireDocument` as a schema-recognition pre-step.

Slice 71 (`docs/slices/slice-71-structured-rfx-adapter-sam-gov-opportunity-fields.md`) shipped 2026-05-09 as the first `public_structured` adapter — `sam-gov/structured-opportunity/v1`. It composes with the slice-02 SAM.gov source adapter (which now delegates field mapping) and the slice-59 binary-capture pipeline; an opportunity-flavoured orchestrator sink at `apps/api/src/modules/opportunities/structured-opportunity-sink.ts` runs alongside the slice-70 requirement-flavoured sink. Migration `0046_slice_71_sam_gov_structured_opportunity.sql` adds the `opportunity_source_metadata` source-only extension table (§3 case-2; generic across adapters) and the `opportunity_field_extraction_disagreement` forensic side table for §10 structured-vs-extracted disagreements. `AdapterContext.tenantId` widens to `string | null` so `public_structured` runs can dispatch through the orchestrator without a tenant context. Subsequent structured-target slices (slice 73 Ariba, slice 79 OCDS) extend the same contract.

## Context

The platform's intake pipeline today has two distinct paths:

- **Source-driven opportunity ingestion** (slice 02 + ADR 0007): a `SourceAdapter` discovers opportunities from a procurement source (SAM.gov, TED, etc.), fetches raw records, persists them, and a normaliser maps them onto the canonical opportunity model. The substrate is `raw_source_record` → `opportunity` (with `opportunity_version`).
- **Document-driven artifact ingestion** (slices 53 / 54 + slice 21 / 22): xlsx, docx, and PDF documents are uploaded or attached, run through extractors (`packages/document-extractors/`), parsed for structure (compliance matrices, questionnaire rows, requirements), and projected into the canonical model.

Both paths terminate at the same canonical model (ADR 0002): `opportunity`, `requirement`, `evaluation_criterion`, `compliance_matrix_row`, etc. That convergence is right; ADR 0002 is what makes the downstream work (qualification, drafting, compliance matrix, submission package) source-agnostic.

What both paths share is an implicit assumption that procurement input is _unstructured_ — it lives in a PDF cover sheet, in a Word RFP narrative, in an Excel questionnaire, in scraped HTML. Document extraction is the bridge from unstructured to canonical; the platform invests heavily in making that bridge robust (slice 54 text extraction, slice 61 LLM-assisted requirements extraction, slice 22 questionnaire-from-PDF routing).

That assumption is decaying. The strategy synthesis frames it directly:

> "RFP as document" is a decaying assumption. […] The long-term destination is less `PDF → extraction → review → draft` and more `structured RFx object → canonical requirement model → policy/evidence response → approval/submission`.

Several procurement input shapes are _already_ structured at source and do not need extraction at all:

- **DDQ row schemas** (CAIQ, SIG-Lite, SIG-Core) — published spreadsheets with a known column layout where every row is a structured question with a stable identifier across versions. The buyer sends a CAIQ xlsx and every row maps unambiguously to a canonical requirement.
- **SAM.gov structured opportunity fields** — alongside the cover-sheet PDF that slices 54/59 already consume, SAM.gov publishes opportunity metadata as structured fields (NAICS codes, set-aside, response date, points of contact, attachment manifest). Today the platform extracts much of this from the PDF; the structured fields are authoritative and free.
- **Open Contracting Data Standard (OCDS)** — JSON-shaped publication standard for public procurement, used by a growing number of jurisdictions.
- **SAP / Ariba / Coupa procurement objects** — enterprise procurement platforms expose procurement events through APIs as structured objects with declared schemas.
- **Peppol / UBL** — XML-shaped procurement messages used in EU and other jurisdictions for transactional procurement.

Today the platform has no first-class place for these. A DDQ xlsx is treated as "an excel questionnaire" by slice 21 and parsed generically; the structured shape is rediscovered with every parse. SAM.gov structured fields are not consumed at all; the cover-sheet PDF is. There is no architectural seat for "this input is already structured at source; don't extract, map."

This ADR establishes that seat. It records the contract for `StructuredRfxAdapter`, the relationship to existing `SourceAdapter` and document extractors, the canonical-model mapping discipline, and the provenance model. Each subsequent structured-target slice (DDQ in slice 70, SAM.gov in slice 71, Ariba in slice 73, OCDS in slice 79) implements the contract; this ADR fixes the contract once.

The ADR is warranted because the contract decisions made here bind every future structured-input slice. Getting the boundary wrong — between SourceAdapter and StructuredRfxAdapter, between StructuredRfxAdapter and document extractors, between adapter output and canonical model — forces re-modelling the intake pipeline every time a new structured shape lands. Getting it right means each new shape is a contained adapter slice with no architectural questions outstanding.

The user's strategic note is the explicit framing:

> Demotes PDF extraction from "moat" to "one of N input shapes." […] The moat is the canonical procurement model, approved evidence graph, agent-operable tool surface, governance, audit, and ability to expose machine-readable seller responses without losing human governance.

PDF extraction is not the moat. The canonical model is. This ADR widens the substrate the canonical model can absorb without changing the model itself.

## Decision

### 1. Structured RFx ingestion is a first-class peer to document ingestion, not a special case

A new package `packages/structured-rfx-adapters/` houses the contract and the per-shape implementations. It is a _peer_ to `packages/source-adapters/` (which discovers opportunities) and to `packages/document-extractors/` (which extracts structured content from unstructured documents). All three packages produce inputs to the canonical model; none subsumes the others.

The boundary distinction is operational, not theoretical:

- `SourceAdapter` discovers and fetches. Its input is "the world"; its output is `raw_source_record` rows.
- `StructuredRfxAdapter` maps known structured shapes to canonical. Its input is a structured object whose schema the adapter knows; its output is canonical rows (opportunity, requirement, evaluation criterion, etc.).
- Document extractors derive structured content from unstructured documents. Their input is bytes (PDF, docx, xlsx); their output is structured intermediate forms that may then be mapped to canonical.

A single procurement input may pass through all three. SAM.gov is a worked example: a `SourceAdapter` discovers the listing and fetches the bundle (slice 02 + slice 59). A `StructuredRfxAdapter` for SAM.gov structured fields (slice 71) maps the structured opportunity metadata directly to `opportunity`. A document extractor (slices 54 + 61) extracts requirements from the cover-sheet PDF where they aren't in the structured fields. The three composite-but-do-not-overlap.

Treating structured RFx as a special case of document ingestion (e.g. "an xlsx with a known fingerprint is just an xlsx with extra parsing") was considered and rejected (Alt A). The cost of that conflation is that every structured shape's adapter has to live inside the document-extractor abstraction whose contract assumes "input is unstructured bytes" — which forces every structured adapter to declare itself as a "smart parser" and inherits the document-extractor's lifecycle (extraction → structure detection → structure mapping) for inputs where the first two steps are no-ops.

### 2. The adapter contract: `StructuredRfxAdapter<TInput, TOutput>`

Each adapter declares:

- `name: string` — stable identifier for registration and provenance recording (e.g. `caiq/v4`, `sam-gov/structured-opportunity/v1`, `ariba/questionnaire/v2`).
- `version: string` — semver-shaped; bumped on schema-breaking changes to the adapter's mapping logic.
- `inputSchema: ZodSchema<TInput>` — Zod-validated input shape. The schema is the source of truth for "does this input belong to this adapter?"
- `inputClass: 'public_structured | tenant_credentialled | tenant_uploaded'` — the tenancy posture of the input (see §6).
- `outputKinds: CanonicalOutputKind[]` — declares which canonical entity types this adapter produces (e.g. `['opportunity', 'requirement']`). Used by the registry to route outputs and to validate that adapter output matches declared output.
- `idempotencyKey(input: TInput): string` — a deterministic key derived from the input; re-running the adapter against the same input must produce the same canonical output (modulo timestamps and audit bookkeeping). Used by the upsert path to avoid duplicate canonical rows.
- `mapToCanonical(input: TInput, context: AdapterContext): Promise<AdapterOutput<TOutput>>` — the load-bearing method. Returns a structured envelope containing the produced canonical-fragment rows plus per-row provenance metadata.

The adapter is a _pure mapper_: no DB writes, no network calls, no side effects. The orchestration layer (`packages/structured-rfx-adapters/src/orchestrator.ts`) takes the adapter's output, performs the upsert against the canonical tables, and emits provenance and audit events. This is the same separation `packages/source-adapters/` uses (the adapter fetches; the orchestrator persists).

The contract being purely typed and purely functional means a new adapter can be unit-tested against fixture inputs without touching the database. It also means a future adapter that produces canonical fragments not anticipated today extends `outputKinds` without needing to change the orchestrator.

### 3. The canonical model is the substrate; adapters do not introduce new canonical types

ADR 0002's canonical procurement model is unchanged by this ADR. Structured RFx adapters map onto the existing canonical entities; they do not define new ones.

Where a structured input has fields the canonical model has no place for, the choice is, in order of preference:

1. The field is irrelevant to downstream workflow — discard.
2. The field is useful only to the adapter's own consumers (e.g. the source-specific identifier needed for round-trip) — record on a source-side metadata table, not on the canonical row.
3. The field is genuinely a new canonical concern that the model should grow to absorb — this is an ADR-level decision, not an adapter implementation choice. The adapter slice flags it; a follow-up ADR amends ADR 0002.

The discipline is "canonical-model-first." A DDQ adapter that wants to record a CAIQ-specific question identifier (`q_id: SEF-08`) on every requirement does so via a `requirement_standard_schema` extension table, not by adding a `caiq_question_id` column to `requirement`. The extension shape is generic across structured schemas; the column never accumulates source-specific names.

### 4. Multiple canonical outputs per input; per-output provenance

Some structured inputs produce a single canonical entity (an OCDS lot → one opportunity). Most produce multiple — an Ariba RFP includes opportunity metadata, evaluation criteria, requirements, attachment manifest, possibly a structured questionnaire. The adapter's `mapToCanonical` returns an envelope:

```ts
type AdapterOutput = {
  opportunities?: CanonicalOpportunityFragment[];
  requirements?: CanonicalRequirementFragment[];
  evaluationCriteria?: CanonicalEvaluationCriterionFragment[];
  questionnaireRows?: CanonicalQuestionnaireRowFragment[];
  // per output kind, with a per-row provenance entry
  provenance: {
    adapterName: string;
    adapterVersion: string;
    sourceFingerprint: string; // hash of the input
    perRow: Record<string, RowProvenance>;
  };
};
```

Each canonical row written by the orchestrator carries a denormalised `produced_by_adapter` discriminator (`adapter_name`, `adapter_version`, `source_fingerprint`) so downstream consumers can answer "which adapter produced this row" without joining through a provenance ledger. The discriminator is small (three text columns) and indexed for "all rows produced by adapter X" queries (used by the slice-70 UI's "DDQ-derived requirements" filter, and by future migration tooling that needs to re-run an adapter's output after a version bump).

Provenance is not the same noun as `raw_source_record`. The latter records the raw input ingested by a `SourceAdapter`; the former records which adapter mapped a structured input to a canonical row. A canonical row may have both: the SAM.gov example produces a `raw_source_record` (the fetched bundle) and a `produced_by_adapter` of `sam-gov/structured-opportunity/v1`.

### 5. Idempotency and re-running

Adapters declare `idempotencyKey(input)` returning a deterministic string. The orchestrator uses this key to detect "have we already mapped this input?" and to upsert rather than insert.

Re-running an adapter against the same input is permitted (and is in fact the intended behaviour when an adapter version bumps): the orchestrator produces the new canonical rows, computes a diff against the previously-produced rows for the same idempotency key, and applies the diff under the existing slice-02 versioning discipline (`opportunity_version` for opportunities; analogous for other entities). This is the same upsert-and-diff pattern source-adapter normalisation uses today.

A re-run that produces _no_ changes is a no-op at the canonical layer (no new versions, no new audit events). A re-run that produces changes mints new versions and emits the existing canonical-model events (`opportunity_version.created`, `requirement.updated`, etc.). The downstream workflow surfaces (qualification, drafting, compliance matrix) consume these events without needing to know that the change came from an adapter re-run rather than a fresh discovery.

### 6. Three input classes with different tenancy postures

The contract distinguishes three input classes:

- **`public_structured`** — input is published openly (OCDS feeds, SAM.gov structured fields, public procurement portals). Output is **shared market data** (ADR 0005). The adapter operates against the platform's own discovery/fetch path and the produced canonical rows live in the shared partition. A future tenant accessing the same opportunity sees the same shared row.
- **`tenant_credentialled`** — input requires tenant-specific credentials to access (Ariba buyer-portal credentials; Coupa supplier-network credentials). Output is **tenant-scoped** because the procurement event was disclosed only to the credential-holder. The shared partition does not absorb tenant-credentialled outputs even when two tenants happen to receive the same RFP through their respective Ariba accounts (each tenant's view is its own canonical row).
- **`tenant_uploaded`** — input is provided directly by the tenant (a DDQ xlsx the buyer sent the supplier; an Ariba export the operator downloaded and uploaded). Output is **tenant-scoped** for the same reason: the platform did not discover the input.

The class is declared on the adapter and is enforced by the orchestrator at write time. A `public_structured` adapter that tries to write a tenant-scoped row, or vice versa, is rejected. This is the structural enforcement that ADR 0005's shared-vs-tenant distinction relies on.

The credential model for `tenant_credentialled` adapters reuses the slice-08 / ADR 0012 credential infrastructure where applicable. A future Ariba adapter (slice 73) brings its own credential wiring (Ariba credentials are not API keys; they are typically OAuth or X.509). Per-adapter credential storage is the responsibility of the per-adapter slice, not this ADR.

### 7. Composition with `SourceAdapter` and document extractors

A `StructuredRfxAdapter` may be invoked:

- **Standalone** — operator uploads a known structured input (DDQ xlsx); the upload pipeline routes to the matching adapter by fingerprint.
- **By a `SourceAdapter`** — a SAM.gov source adapter fetches the bundle and invokes the SAM.gov structured-opportunity adapter on the structured fields it includes; in the same fetch, attached PDFs are routed to document extractors. The source adapter's job is to discover and decompose; each piece is routed to the appropriate downstream.
- **By a document extractor** — a document extractor parses an xlsx and recognises a known schema (CAIQ fingerprint, etc.); it invokes the matching structured adapter on the parsed rows. This is the bridge case: the document extractor is doing the unstructured-to-structured conversion, and the structured adapter is doing the structured-to-canonical mapping.

The orchestration layer routes these compositions; adapters do not invoke each other directly. The routing logic lives in `packages/structured-rfx-adapters/src/orchestrator.ts` and is keyed off `inputSchema` validation: the orchestrator hands a candidate input to each registered adapter's `inputSchema.safeParse` until one matches. The first matching adapter wins. (Schemas are designed to be mutually exclusive; an ambiguous match is a registry-construction error.)

A future "richer router" — e.g. ML-driven schema detection — is not foreclosed; it would replace the linear safeParse loop with a smarter dispatcher that produces the same `(input, adapter)` tuple. The adapter contract is unchanged.

### 8. The canonical requirement model gets a small extension for standard-schema metadata

Many structured inputs (DDQ rows above all) carry a _standard schema identifier per row_ — the CAIQ question SEF-08 has the same identifier across CAIQ versions and across every CAIQ a tenant has ever received. Recording this identifier on the canonical requirement is what unlocks cross-pursuit answer reuse: "we answered SEF-08 in March; auto-suggest the same answer next time."

A new `requirement_standard_schema` extension table is introduced (single-row-per-requirement; foreign key on `requirement.id`). Fields: `requirement_id`, `standard_schema_id` (e.g. `caiq`), `standard_schema_version` (e.g. `v4`), `standard_question_id` (e.g. `SEF-08`), `standard_question_category` (e.g. `Security Engineering — Filtering`), `expected_answer_type` (enum: `yes_no | yes_no_with_explanation | multiple_choice | free_text | numeric | date`). The extension is populated only when an adapter produces the structured metadata; document-extracted requirements have no `requirement_standard_schema` row.

This extension is the discipline-§3 case 3: a genuinely new canonical concern that the model grows to absorb. The shape is generic across all structured schemas (every structured schema with stable per-question identifiers can populate it), so it does not become source-specific. ADR 0002 is amended in a follow-up to record the extension.

### 9. Adapter registration and discovery

Adapters register at module load via a registry pattern (mirroring `packages/source-adapters/`):

```ts
import { registerStructuredRfxAdapter } from '@bidangel/structured-rfx-adapters';
import { caiqV4Adapter } from './caiq/v4';

registerStructuredRfxAdapter(caiqV4Adapter);
```

The registry is process-local (in-memory); adapter registration happens at API process startup. A future change to make adapters dynamically configurable per tenant (e.g. "tenant X has enabled the Ariba adapter") would extend the registration with tenant-scoped activation; the contract is forward-compatible.

The registry is queryable: `listAvailableAdapters()` returns a tenant-scoped list of adapters available given the tenant's credentials, used by the UI for the "what structured shapes can I upload" picker.

### 10. Extraction is preferred to be lossless; structured is preferred to extracted where both are available

When both a structured input and a document extraction are available for the same logical entity (the SAM.gov example: structured opportunity fields + cover-sheet PDF), the **structured input is authoritative** and the document extraction is supplementary. The orchestrator's diff-on-re-run discipline (§5) means that when a structured field and an extracted field disagree, the structured field wins on the canonical row, and the extracted field is recorded on a side table for forensic comparison if needed (a future "extraction quality" metric).

The reason for this asymmetry is provenance honesty: a structured field came from the source as-published; an extracted field came from the platform's interpretation of unstructured bytes. Trusting the source over the interpretation is the conservative call. Where a tenant disputes a structured field's value (e.g. "the buyer's structured NAICS is wrong; the PDF cover sheet has the right code"), the operator can manually override the canonical row and record the override; the override surface is owned by the slice that needs it (likely a slice-03 inbox enhancement), not by this ADR.

## Consequences

### Direct consequences (slice 70 implements; later structured-target slices extend)

- A new `packages/structured-rfx-adapters/` package ships with the contract, the orchestrator, and the registration pattern.
- A new `requirement_standard_schema` extension table ships, populated by adapters that produce per-row standard identifiers. The extension is generic across all structured schemas.
- Canonical entities gain a denormalised `produced_by_adapter` discriminator (three text columns) for provenance.
- Slice 70 ships the first three concrete adapters (CAIQ v4, SIG-Lite, SIG-Core).
- Subsequent slices (71 SAM.gov, 73 Ariba, 79 OCDS) implement the contract for their respective structured shapes.
- Document extractors (slices 54, 22) gain a schema-recognition step: when a parsed xlsx matches a known structured-RFx adapter's `inputSchema`, the parsed rows are routed to that adapter rather than processed generically.
- The Answer Library (slices 18–20) gains a "auto-link by standard schema identifier" affordance — answers to CAIQ SEF-08 in one pursuit auto-suggest as candidate answers for CAIQ SEF-08 in another pursuit.

### Forward-compatibility consequences (this ADR's load-bearing reason)

- New structured shapes (Peppol, UBL, OAGIS, jurisdiction-specific procurement schemas) ship as additional `StructuredRfxAdapter` implementations with no architectural changes.
- Adapter version bumps (e.g. CAIQ v5 ships and supersedes CAIQ v4) are handled by registering a new adapter version and bumping the canonical rows' `adapter_version` column on re-run; the version-aware idempotency key keeps old and new adapter outputs distinguishable.
- A future "structured RFx output target" — when an `AssemblyTarget` (slice 65 / ADR 0022 §4) ships that produces a structured response object (e.g. Ariba questionnaire xlsx with answers populated) — composes with structured adapters: the input adapter mapped buyer-side structured input to canonical; the output target maps canonical responses to buyer-side structured output. The two are mirror operations under the same contract style.
- A future "structured RFx through MCP" — where a buyer-side agent submits a structured procurement object directly to the platform via the existing MCP surface (slice 17 / ADR 0012) — uses the same adapter chain. The MCP adapter receives a structured input; the orchestrator routes it to the matching adapter; canonical rows materialise. ADR 0029's eventual buyer-facing read surface is the symmetric read.
- Per-tenant adapter activation (e.g. "this tenant has paid for the Ariba adapter") is a registry extension, not a contract change.

### Costs and trade-offs accepted

- **A new package and a new noun** (`StructuredRfxAdapter` distinct from `SourceAdapter` distinct from document extractors). The taxonomy must be learned by anyone touching intake. The mitigating choice is the operational distinction in §1: the boundary is mechanical (discover vs map vs derive-structured) and each path's contract is small.
- **A small canonical extension** (`requirement_standard_schema`) and a denormalised provenance discriminator on every adapter-produced row. The schema growth is bounded; the indices are inexpensive.
- **Two paths can produce the same canonical entity** (structured + extracted). The §10 asymmetry rule (structured wins) prevents ambiguity at the canonical layer; the operator-override surface (deferred) handles the disputed-source case.
- **Adapter mutual-exclusivity** is a registry-construction discipline, not a runtime check. A registry whose adapter input schemas overlap is a configuration bug. The cost is real but contained — adapter authors must understand schema-mutual-exclusivity at design time.
- **Per-adapter credential wiring** is owned by per-adapter slices, not by this ADR. Means a follow-up slice that introduces a new credential model (Ariba OAuth) does its own credential storage. The cost is duplicated wiring across credentialled adapters; the benefit is each credential model is properly fit to its source.
- **Standalone adapters cannot discover** (only `SourceAdapter` discovers). A user who wants the platform to _fetch_ a DDQ from a buyer's portal needs a `SourceAdapter` for that portal that invokes the DDQ adapter on the fetched payload. The two-step composition is the right factoring; building "smart" structured adapters that also discover would conflate concerns and dilute the boundary.
- **Document extractors gain a schema-recognition pre-step.** A small additional check on every uploaded xlsx ("does this fingerprint match a known structured adapter?"). The check is cheap and has clean fallback (no match → generic parsing).

### Things this ADR explicitly does not commit to

- **A specific structured adapter's implementation.** Slice 70 ships CAIQ/SIG; slice 71 ships SAM.gov; slice 73 ships Ariba; slice 79 ships OCDS. Each has its own slice and contained scope.
- **Per-adapter credential models.** Each credentialled adapter brings its own; this ADR does not dictate a credential abstraction.
- **A buyer-facing structured response surface.** The `AssemblyTarget` discipline from ADR 0022 §4 is the input-side mirror; whether and how a buyer-side agent reads structured outputs is ADR 0029.
- **Cross-tenant sharing of tenant-credentialled adapter outputs.** Two tenants with their own Ariba credentials each see their own canonical row even for the same RFP. Cross-tenant deduplication is not in scope.
- **Automatic schema-detection via ML.** Mutual-exclusive Zod schemas only in MVP. A richer dispatcher is forward-compatible.
- **A new canonical entity type.** ADR 0002's model is unchanged; only the small `requirement_standard_schema` extension is added.
- **Operator override of structured-vs-extracted disagreement.** The §10 asymmetry rule applies; operator override surface is a future slice.

## Alternatives considered

### Alt A — Treat structured RFx as a subclass of document ingestion

Rejected. Forces every structured shape to live inside the document-extractor abstraction, whose contract assumes "input is unstructured bytes." Structured shapes (e.g. an Ariba object received via API) have no bytes; they have a typed payload. Subclassing forces awkward "fake bytes" wrappers and inherits a lifecycle (extract → detect → map) for inputs where the first two steps are no-ops. The maintenance cost compounds with every new structured shape.

### Alt B — Make `StructuredRfxAdapter` a kind of `SourceAdapter`

Rejected. SourceAdapter discovers and fetches; StructuredRfxAdapter maps. Conflating the two means every structured adapter inherits a discovery contract that most don't need (a DDQ adapter's input is uploaded, not discovered; an Ariba adapter is invoked by a SourceAdapter for Ariba). The clean separation in §1 lets each path's contract stay small and the composition (§7) be explicit.

### Alt C — One adapter contract that handles both structured input and structured output

Considered. A bidirectional adapter shape is appealing for symmetry: the same Ariba contract reads buyer-side requirements and writes supplier-side answers. Rejected for slice 70 because input adapters and output targets have meaningfully different concerns (input cares about idempotency and provenance; output cares about authoring and approval-state filtering). ADR 0022 §4 already established `AssemblyTarget` as the output-side contract. Future symmetry between them is possible (a per-adapter "two-faced" wrapper) but is not the right starting shape.

### Alt D — No new package; put adapters in `packages/source-adapters/`

Rejected. SourceAdapters and StructuredRfxAdapters do different things (§1, Alt B). Co-locating them means the source-adapters package's dependencies (HTTP clients, scrapers, scheduling) leak into structured adapters that don't need them, and vice versa. A separate package is cheap and the boundary stays clean.

### Alt E — Map structured inputs directly to the canonical model without an adapter abstraction

Rejected. Without an abstraction, every new structured shape's mapping logic lives somewhere bespoke (in the upload route handler, in the source-adapter that fetched it, in a service module). The boundary erodes; the same per-shape logic gets duplicated; the registry of "what shapes does this platform absorb" is implicit. The adapter abstraction is the same investment `packages/source-adapters/` made for ingestion sources, and pays off the same way.

### Alt F — Adapters write to canonical model directly (no orchestrator)

Rejected. Same reasoning as `packages/source-adapters/`'s normaliser/orchestrator separation. Writes have cross-cutting concerns (provenance recording, idempotency upsert, audit emission, version diffing) that should not be re-implemented in every adapter. The orchestrator is the single site for those concerns.

### Alt G — Lossy mapping to canonical for structured fields the canonical model doesn't cover

Considered. A structured input often carries fields the canonical model has no place for. The discipline in §3 (extension table for genuine new concerns; side table for source-only metadata; discard for irrelevant) is the right factoring. A lossy map without a side-table fallback would lose the source-only metadata that some tenants legitimately need (e.g. an Ariba-specific event id needed to round-trip back to Ariba on submission).

### Alt H — Structured-vs-extracted disagreement resolution in code, not in the canonical row

Rejected. The §10 asymmetry rule (structured wins) is simple, predictable, and matches provenance honesty (source-as-published trumps platform-interpretation). Encoding the disagreement on the canonical row (e.g. a `structured_value` and `extracted_value` pair) would force every downstream consumer to choose between them; far worse than a single authoritative value plus an audit trail of the disagreement.

## Follow-on implications

- **Slice 70** is the first implementation. CAIQ v4, SIG-Lite, SIG-Core adapters; the orchestrator; the `requirement_standard_schema` extension; document-extractor schema-recognition pre-step.
- **Slice 71** shipped (2026-05-09) the SAM.gov structured-opportunity adapter, composed with the existing slice-59 binary capture and slice-02 source-adapter machinery. First `public_structured` adapter; introduced the `opportunity_source_metadata` extension (§3 case-2), the §10 disagreement-detection helper, and the opportunity-flavoured orchestrator sink alongside slice-70's requirement sink.
- **Slice 73** ships the Ariba adapter (likely 73a/b/c given Ariba's surface area), with its own credential wiring.
- **Slice 79** ships the OCDS adapter.
- **ADR 0002 amendment** records the `requirement_standard_schema` extension.
- **ADR 0007 amendment** records the relationship between `SourceAdapter` and `StructuredRfxAdapter` (peer, composing through the orchestrator).
- **ADR 0009 amendment** registers the `structured_rfx_adapter.*` event family (`adapter_run_started`, `adapter_run_completed`, `adapter_run_failed`, `adapter_output_diff_applied`).
- **A future ADR for operator override of structured-vs-extracted disagreements** when the use case becomes concrete.
- **A future ADR for buyer-side structured input over MCP** (the symmetric read for ADR 0029).

## Amendments

### 2026-05-08 — Canonical `evaluation_criterion` fragment shape (UBL-aligned) and `evaluation_criterion` vs `questionnaire_row` boundary

`CanonicalEvaluationCriterionFragment` now ships as a typed 12-field shape in
`packages/structured-rfx-adapters/src/contract.ts`, replacing the slice-70
stub. The shape was derived from the UBL 2.3 `cac:TenderingCriterion` and
Peppol BIS ESPD 1.0 `ccv:Criterion` field set by the crosswalk in
`docs/crosswalks/ubl-peppol-evaluation-criterion.md`. Slice 73 (Ariba), slice
71 (SAM.gov, when it eventually adds award criteria) and slice 79 (OCDS, when
it consumes `awardCriteria[]`) MUST emit fragments conforming to that typed
shape; ad-hoc per-adapter `fields: Record<string, unknown>` evaluation-criterion
output is no longer permitted.

The crosswalk surfaced one boundary question that this ADR amendment now pins:

**UBL `cac:TenderingCriterionPropertyGroup` (≈ Peppol `ccv:RequirementGroup`)
routes to `CanonicalQuestionnaireRowFragment`, not
`CanonicalEvaluationCriterionFragment`.** UBL nests structured sub-questions
under each tendering criterion via `TenderingCriterionPropertyGroup` →
`TenderingCriterionProperty`. We deliberately do not absorb that nesting
onto the `evaluation_criterion` row, because (a) it conflates "what is being
evaluated" (the criterion) with "what evidence is asked for" (the questions /
properties), and (b) the latter is what the existing
`CanonicalQuestionnaireRowFragment` already models. Adapters whose source
exposes `TenderingCriterionPropertyGroup`-shaped data emit one
`CanonicalEvaluationCriterionFragment` per criterion plus N
`CanonicalQuestionnaireRowFragment` entries per property, with each
questionnaire row carrying `parentCriterionPerRowKey` pointing back at the
owning criterion. Round-trip to UBL XML reconstructs the nesting via that
pointer.

This amendment does not relax any §3 discipline: `evaluation_criterion` and
`questionnaire_row` remain canonical types established by ADR 0002, not new
ones. The amendment clarifies the routing rule when a UBL element could
plausibly land on either.

**Resolved companion questions (recorded for posterity, full rationale in the
crosswalk doc):**

- Multilang fields (`name`, `description`, `weightingConsiderationText`)
  ship single-string. `Array<{lang, text}>` promotion is deferred to the
  standards-submission slice when EU adoption is the load-bearing test.
- `criterionType` follows a hybrid path: a 7-value `CanonicalCriterionType`
  enum plus a sibling `criterionTypeCode: string | null` carrying the
  verbatim EU-COM-GROW `CriteriaTypeCode` value when the source provides
  one. Lossless round-trip without forcing non-EU adapters onto the EU
  codelist.
- `cpvCode: string | null` joins the canonical fragment so
  `cac:CommodityClassification` has a home; secondary CPV codes spill to
  `fields.cpvSecondary`.

ADR 0034 (A2A Procurement Profile layered on UBL semantics) is the durable
home for the cross-cutting standards posture; this amendment is the
ADR-0026-local consequence.

### 2026-05-08 (continued) — Canonical `opportunity` fragment shape (UBL-aligned)

`CanonicalOpportunityFragment` now ships as a typed shape in
`packages/structured-rfx-adapters/src/contract.ts`, replacing the slice-70
stub. Companion entry to the evaluation-criterion amendment above; same
ADR 0034 layered-on-UBL discipline. Slice 71 (SAM.gov), slice 73 (Ariba)
and slice 79 (OCDS) MUST emit fragments conforming to the typed shape.

Three opportunity-specific resolutions are pinned alongside the shape:

- **Multi-lot correlation (Q1 = B).** UBL / OCDS / eForms natively support
  multi-lot procurements. Our canonical opportunity remains single-lot;
  the N opportunities materialised from an N-lot UBL/OCDS notice are
  correlated via two new columns on canonical `opportunity`: `notice_id`
  (shared across the lot opportunities) and `lot_ref` (per-row lot
  identifier). Migration `0044_opportunity_ubl_alignment.sql` adds both.
  Promotion to a parent + lots schema (Alt C in the probe) was rejected
  on cost-of-change grounds; the lightweight correlation pattern matches
  the existing `evaluation_criterion.lotRef`.
- **`opportunityType` enum (Q2 = A).** Hybrid path mirrors the
  `criterionType` resolution: 7-value canonical
  `CanonicalOpportunityType` enum (`goods`, `services`, `works`,
  `mixed_goods_services`, `concession`, `framework`, `other`) plus a
  sibling `opportunityTypeCode: string | null` carrying the verbatim
  source-system code (eForms `procurement-procedure-type`, OCDS
  `procurementMethod`, SAM.gov award-type). Lossless round-trip without
  forcing non-EU adapters onto the EU codelist.
- **Primary commodity classification (Q3 = A).** Canonical `opportunity`
  gains two new first-class columns: `primary_category_code` and
  `primary_category_scheme` (`cpv` / `unspsc` / `naics` / `psc` /
  `tenant_custom`). Single primary code per opportunity; secondary codes
  spill to adapter-side `fields.categoriesSecondary` or
  `opportunity_source_metadata`. Migration `0044` adds both columns plus
  a partial index for primary-category lookups.

The cross-cutting status-lifecycle cross-map
(`docs/crosswalks/canonical-opportunity-status-lifecycle-map.md`) and the
follow-on Buyer crosswalk (`docs/crosswalks/ubl-peppol-buyer.md`) ship
alongside this amendment as their own artifacts.

### 2026-05-08 (third) — Canonical `requirement` fragment shape (UBL-aligned) and retirement of the `questionnaire_row` canonical kind

`CanonicalRequirementFragment` now ships as the typed UBL-aligned shape
in `packages/structured-rfx-adapters/src/contract.ts`, replacing the
slice-70 minimal-DDQ shape. Slice 73 (Ariba), slice 71 (SAM.gov, when
it eventually surfaces structured requirements) and slice 79 (OCDS)
MUST emit fragments conforming to the typed shape. The shape adds
`externalId`, `requirementType` + `requirementTypeCode` (Q2=A hybrid
path), `subType`, `dueAt`, `valueConstraints` (Q3=B OCDS-aligned shape),
`evidenceNeededSummary`, and a `fields` spillover map; existing slice-70
adapters (CAIQ v4, SIG-Lite, SIG-Core) populate the new fields with
explicit values (`requirementType='security'`, `subType='questionnaire'`,
`valueConstraints=null` since CAIQ/SIG carry no value constraints).

This amendment also **retires the `'questionnaire_row'` canonical
kind** (Q1=A from the requirement crosswalk). The original 2026-05-08
amendment said UBL `cac:TenderingCriterionPropertyGroup` routes to a
(stub) `CanonicalQuestionnaireRowFragment`. That fragment kind had no
consumer — slice 70 produced `requirement` (subType=questionnaire)
rows, slice 73 plans the same, and the slice-21 `subType` discriminator
already does the work a separate canonical kind would have done. Two
canonical entities for the same distinction is over-modelling.

The amendment therefore:

- Removes `'questionnaire_row'` from `CanonicalOutputKind`.
- Removes the stub `CanonicalQuestionnaireRowFragment` from
  `contract.ts` and its `questionnaireRows?: ...` slot from
  `AdapterOutput`.
- Re-pins the routing rule for property-group-shaped sources (UBL
  `cac:TenderingCriterionPropertyGroup` ≈ Peppol ESPD
  `ccv:RequirementGroup`): each property within the group emits one
  `CanonicalRequirementFragment` with `subType='questionnaire'` and a
  back-pointer to the parent evaluation criterion via `sectionRef` (or
  a future explicit FK if a slice needs it). The
  `parentCriterionPerRowKey` field on
  `CanonicalEvaluationCriterionFragment` continues to model the
  evaluation_criterion → sub-criterion nesting; that's a separate
  axis from the requirement → criterion linkage and is unaffected.

The cross-cutting requirement crosswalk
(`docs/crosswalks/ubl-peppol-requirement.md`) ships alongside this
amendment.

## References

- Slice 70 — `docs/slices/slice-70-structured-rfx-adapter-ddq-rows.md` (first implementation of this ADR)
- Slice 71 — SAM.gov structured-opportunity adapter (shipped 2026-05-09; `docs/slices/slice-71-structured-rfx-adapter-sam-gov-opportunity-fields.md`)
- Slice 73 — Ariba structured procurement adapter (planned)
- Slice 79 — OCDS adapter (planned)
- ADR 0002 — canonical procurement model (the substrate adapters map to)
- ADR 0005 — shared market data vs tenant-owned workflow data (the §6 tenancy posture rule)
- ADR 0007 — source-adapter pattern (the contract pattern this ADR mirrors and composes with)
- ADR 0008 — advisory vs approved workflow state (unchanged; adapter outputs are advisory until approved)
- ADR 0009 — audit and event taxonomy (the new `structured_rfx_adapter.*` family)
- ADR 0012 — external agent credentials and MCP adapter (the credential model some tenant-credentialled adapters reuse)
- ADR 0022 — submission package as first-class artifact (the symmetric output-side contract `AssemblyTarget`)
- Slice 02 — opportunity ingestion and canonical persistence (the existing source-adapter machinery this composes with)
- Slice 21 / 22 — questionnaire ingestion (Excel and Word/PDF; the document-extractor entry points)
- Slice 53 / 54 — content-asset ingestion and text extraction (the document-extractor surface)
- Slice 59 — SAM.gov binary capture (the per-source companion to slice 71's structured adapter)
- Slice 18 / 19 / 20 — answer library (the consumer of the new standard-schema metadata for cross-pursuit answer reuse)
