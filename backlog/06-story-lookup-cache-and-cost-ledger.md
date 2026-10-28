---
id: ZBI-BE-01
epic: ZBI-EPIC
type: story
title: Retrieve business details with a 24-hour cache and auditable lookup costs
status: interview-draft
priority: P1
owner: Backend
delivery_scope: supporting-backend-dependency
depends_on: [provider-contract-and-credentials, existing-draft-ownership]
contract: 08-shared-contracts.md
schema_version: 1
---

# BE-01: Retrieve business details with a 24-hour cache and cost ledger

## Context

Supporting backend work makes the frontend contract implementable and fulfils the requested cost controls. The assumed ABR integration uses identifier-specific current-detail methods; the real paid provider contract remains unconfirmed. [Source notes](10-source-notes.md) separate documented behaviour from proposed adapter policy.

## Story statement

**As a Zeller onboarding team member, I want reusable business lookup results and an attributable record of each provider attempt, so that merchants progress quickly and we can measure lookup volume and estimated spend.**

## Deterministic acceptance criteria

1. **Preflight:** Authenticate, authorise ownership, check safe request shape, and apply the same versioned normaliser as the client before identifier validation or use. Accept supported prefixes/separators, preserve zeros, reject conflicting labels, then validate length/checksum. Canonical digits/type drive business-payload equality, cache, dispatch, saved identity, and later fields. Equivalent labelled/spaced values share a key. Failure makes zero provider calls; client validation is never trusted as a substitute.
2. **Dispatch:** For valid uncached ABN use the ABN method; for ACN use the ASIC-number method. Request current data only. Keep the GUID server-side. Apply an 8-second outbound deadline within the frontend's 10-second budget; do not automatically retry.
3. **Mapping:** Produce the shared normalised result or reason. Select current identity/status using provider semantics, not array order. Preserve provenance and allow absent optional attributes. Confirm supported business type separately from ACN classification: the ASIC method also handles other registration kinds. Use verified provider/company-type semantics; non-ACN or unconfirmed kind returns UNSUPPORTED_IDENTIFIER_TYPE with ABN/support recovery, without another automatic lookup. Contradictory current values are an invalid/ambiguous result, not a guess.
4. **Cache TTL:** Cache valid returned entities, including inactive or incomplete entities, at `retrieved_at`. At `now < retrieved_at + 24h` reuse the cached result; at equality or later fetch anew. Reads do not extend TTL. Errors, malformed payloads, and NOT_FOUND are not cached; stale values cannot approve a save during an outage.
5. **Cache key:** Scope by environment, provider, method/schema version, history flag, identifier type, and normalised identifier. Public register data may be shared across authorised users; per-user drafts, answers, and lookup references are never shared. Verified ACN↔ABN aliases reuse the same snapshot and original expiry. Never infer an alias arithmetically.
6. **Concurrency:** For simultaneous misses of the same key, use a bounded shared in-flight operation so exactly one outbound call occurs. Other users' requests get their own authorised `lookup_id` and request record referencing that call. Cache failure must not create a fan-out of duplicate calls; fail safely if single-flight ownership cannot be established.
7. **Idempotency:** A re-delivered lookup operation with the same owner/draft, key, and payload returns its original result while retained, without another call. A reused key with a different payload is rejected. A new explicit retry gets a new attempt ID. An expired lookup replay remains expired for saving; it cannot extend cache freshness.
8. **Cost ledger:** Before dispatch durably record a unique provider-call intent with `user_id`, `temporary_business_id`, attempt ID, and timestamps. Finalise with outcome/latency even when the browser has left. If durable intent recording fails, do not dispatch. A crash between intent and confirmed dispatch leaves an explicit uncertain state for reconciliation, never a fabricated success or a hidden call.
9. **Every request measurable:** Separately record cache-hit, cache-miss/coalesced, preflight rejection, and provider-backed request outcomes. Cache/coalesced requests do not count as extra provider calls; initiation is attributed to the initiating user and consuming users are linked separately.
10. **Estimated cost:** Count each confirmed dispatched attempt once at AUD $0.60 under the exercise assumption, including failed/timeout attempts pending billing confirmation. A cache hit costs $0 incremental provider spend. Two actual dispatched attempts estimate $1.20. Preserve unknown-dispatch states and reconcile them instead of silently counting them as zero.
11. **Save protection:** Issue opaque user/draft-bound lookup references. Authenticate/authorise and compare the idempotent business payload first; replay committed saves before new-mutation TTL/catalogue/revision checks. Pending replay returns SAVE_IN_PROGRESS without a second mutation. Only a new mutation validates snapshot freshness/revision and stores official provenance plus separate merchant answers. The backend alone durably schedules completion publication and deduplicates it by completion ID. A user cannot save another user's reference or alter provider identity.
12. **Event context:** Bind the onboarding attempt from the owned draft/lookup. Retain optional `client_context` from the initiating request for backend events, including after browser departure. Missing/invalid visit/app context becomes explicit nulls plus a status; it never blocks lookup/save or changes business-payload equality. Do not invent client IDs or overwrite original operation context on replay.

## Required ledger fields

`provider_call_id`, initiating `user_id`, `temporary_business_id`, `onboarding_attempt_id`, `lookup_attempt_id`, provider/method/version, identifier type plus protected identifier reference, intent/dispatched/completed timestamps, dispatch certainty, normalised outcome, latency, `estimated_unit_cost_minor: 60`, `currency: AUD`, billing assumption version, reconciliation status. Each application request has its own `request_id`, requester IDs, `source`, and optional shared `provider_call_id`.

## Implementation & architectural guardrails

- Durable operational ledger is separate from best-effort product analytics. Unknown outcomes are visible; transport abort cannot prove no charge.
- Use an approved shared cache and concurrency primitive. Do not prescribe a new database/queue technology without the existing architecture.
- Store only necessary normalised public fields; encrypt/restrict operational data, redact credentials and original request echoes, and define retention with existing policy owners. Do not expose another user's ledger fields through a cache hit.
- Map HTTP failures, application exceptions, and unknown payloads explicitly. Provider configuration failures trigger an internal alert and generic user feedback.
- Reuse the snapshot when saved; a save never initiates a paid lookup. Only a new mutation with an expired snapshot returns LOOKUP_EXPIRED; an already committed save replays its original outcome.

## Out of scope / non-goals

Rate limiting, bot challenges, invoicing automation, negative caching, stale-on-error approval, bulk/name searches, live AML checks, or circumventing suppressed register data.

## Verification / handover

Use a fake clock and stub provider, not charged live calls: invalid checksums → 0 calls; fresh hit → 0 additional calls; exact expiry → 1 new call; simultaneous misses → 1 call; errors → no cached error; two users → separate request records and one cost record; changed idempotency payload → rejection; timeout/browser exit → completed or uncertain ledger record. Test ledger-write failure prevents dispatch and cross-user lookup references cannot save.

Also compare client/server normalisation outputs; prove labelled/hyphenated equivalents reuse cache/replay, conflicting prefixes make zero calls, non-ACN ASIC results stop safely, and missing telemetry is explicit. Save replay after expiry/newer revision returns the original outcome with one completion event; pending replay starts no duplicate mutation.
