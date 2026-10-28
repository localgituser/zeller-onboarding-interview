---
id: ZBI-CONTRACT
type: shared-contract
status: proposed-for-interview
schema_version: 1
updated: 2026-10-28
---

# Shared implementation contract

This is a proposed contract for deterministic fixtures and frontend/backend agreement. It does not assert these endpoints or fields already exist at Zeller. Use the existing application's equivalent routes and infrastructure once confirmed. [Epic assumptions](00-epic-business-information.md) apply throughout.

## URLs and browser navigation

Use the existing application router with a distinct URL for each navigable step. Proposed paths below illustrate the contract; map them to the host application's conventions before integration. Loading/error states and story boundaries do not each create a new page.

| Step / state | Proposed URL | Browser-history behaviour |
| --- | --- | --- |
| FE-01 identifier entry | `/onboarding/business-information/identifier` | Stable entry step; invalid input stays here without a history entry |
| FE-02 lookup pending or lookup failure | `/onboarding/business-information/details` | A deliberate valid Next/Enter pushes this step exactly once before showing progress or immediate failure |
| FE-03 retrieved or manually entered details; FE-04 fields/save pending/save error | Same details URL | Update the current step without adding history entries for results, mode changes, retries, validation, or saves |
| Server-acknowledged continuation | Existing next-step URL | Push once only while the originating details screen is still active; use the existing approved destination |
| FE-05 help/sign-in recovery | Existing application surfaces | Help overlay closes back to its origin; a full help page uses normal history. Sign-in returns to an allowed origin route after checking draft ownership |

1. **Back parity:** In the normal entry → details journey, browser Back and the form's Back return to identifier entry in one action from lookup, error, retrieved details, or manual details. Retain the entered ABN/ACN for editing and preserve same-identifier answers in the current onboarding session. Form Back uses the matching prior flow entry when present; for a direct details link with no such entry, route to identifier entry with replacement. Native browser Back still follows actual history; do not trap it or add synthetic entries to prevent leaving. From identifier entry, Back follows the existing previous-step navigation.
2. **Leaving pending work:** Any route departure, including browser Back/Forward, invalidates that screen's active lookup/save response handler and aborts the browser request where possible. A late response cannot change the destination, restore the old screen, overwrite answers, or advance onboarding. Dispatched provider work and committed saves can still finish server-side; retain their ledger/idempotency semantics. Never reset lookup budgets on route changes.
3. **Forward and route restoration:** Browser Forward, refresh, and direct route entry are read-only restoration actions: no automatic provider call, save, or operation replay. Restore matching in-memory form state first within the current session, otherwise read the owned server-saved draft and current retry budget. Restore details only for the currently selected identifier; an edited/different identifier must not reveal old-business details through Forward. Replace an unavailable/mismatched details route with identifier entry, retaining any current identifier input. If a validated identifier remains but a lookup was interrupted without usable details, show actionable “Your business lookup was interrupted. Enter your details manually or try again.” on the details route, with manual entry and explicit retry only while budget remains. When exhausted, use FE-05's retry-limit feedback. Do not restart loading or accept an invalidated late result.
4. **Save reconciliation:** Saving is an in-place state, not a history entry. In-form submission/navigation controls can be disabled while saving; native browser navigation remains available. If the user leaves, ignore late UI success and reconcile the saved draft/revision through an authenticated read on return before accepting a changed save. If the operation outcome is still uncertain, keep the original save ID and offer an explicit same-key retry; do not automatically resubmit on route entry. A successful save pushes the next-step route only if its originating screen/operation remains active. Back from that next step returns to details and reads the latest saved state without another lookup/save/completion event.
5. **Help and authentication:** Lookup/save errors remain on their originating step. An in-app help overlay restores focus to its trigger on close; browser Back closes a history-aware overlay before leaving the form. Full-page help pushes its own route and Back restores the originating form. Help must not consume lookup attempts. After sign-in, restore only an owned draft to an allowlisted internal return route; if details cannot be restored, replace with identifier entry. Never restart an interrupted lookup/save automatically.
6. **State and privacy:** URLs, query strings, fragments, and browser history state must not contain ABN/ACN, business names, provider payloads, or operation credentials. Use the existing authenticated draft context and minimal route metadata. Keep unsaved same-session answers in the onboarding state store; this is not autosave. Refresh/browser closure/sign-in redirects may lose unsaved answers under the current persistence scope; only server-saved drafts are guaranteed across those boundaries. Router restoration never bypasses authentication, ownership, identifier matching, lookup expiry, or server-owned retry limits.
7. **Accessibility and measurement:** Give entry, details, and help pages meaningful titles. On entering identifier entry via Back, focus its labelled identifier input; on details restoration focus the heading (or the first manual field when explicitly entering manual mode). Announce loading/error changes without creating navigation. Track browser Back/Forward as navigation, not a lookup submission or an additional completion; stage transitions follow DATA-01.

## Identifier validation

Use the same versioned normaliser and fixture set on client and server. **Normalise before identifier validation or use** in cache keys, idempotency business-payload comparison, provider requests, saved identity, and later fields. The backend authenticates/authorises and checks safe request shape first; it never trusts a client's claimed validation. Normalisation/formatting does not prove existence, ownership, status, or eligibility.

### Normalise first, then validate

1. Require string input; never convert to a number. Trim outer whitespace, then recognise at most one leading case-insensitive `ABN` or `ACN`, optionally followed by whitespace and one colon. A separator is not required before the digits: `ABN51824753556` is accepted. Remove only this recognised prefix; retain its type hint. Do not accept repeated labels, arbitrary surrounding words, URLs, or multiple labelled identifiers.
2. In the remaining value, remove only Unicode White_Space characters and supported hyphens: ASCII `-` plus `U+2010`, `U+2011`, `U+2012`, `U+2013`, `U+2014`, and `U+2212`. Pin whitespace behaviour to `U+0009–000D`, `U+0020`, `U+0085`, `U+00A0`, `U+1680`, `U+2000–200A`, `U+2028`, `U+2029`, `U+202F`, `U+205F`, `U+3000`. Thus non-breaking spaces and line-wrapped single identifiers are accepted consistently. Reject every other non-ASCII-digit character; never use a blanket “remove all non-digits” rule. No canonical identifier is usable from a malformed paste.
3. The result is a digits-only **string**, retaining leading zeros. Empty (including label/separators only) → required error. Prefix and explicit request `identifier_type` must agree, else `INVALID_FORMAT` / `TYPE_CONFLICT`. A prefix or declared type fixes expected length (ABN 11, ACN 9); wrong length → `INVALID_FORMAT` / `LABEL_LENGTH` with FE-01's type-specific copy. Without either, infer candidate ABN at exactly 11 digits or candidate ACN at exactly 9; other lengths fail `INVALID_FORMAT`. All zeros fail the product sanity rule. Do not truncate, pad, reinterpret a conflicting prefix, or derive one identifier from another.
4. ABN: subtract one from the first digit; multiply by `[10,1,3,5,7,9,11,13,15,17,19]`; sum modulo 89 must be zero. [ABR validation specification](https://abr.business.gov.au/Help/AbnFormat). ACN: first eight digits × `[8,7,6,5,4,3,2,1]`; ninth equals `(10 - (sum modulo 10)) modulo 10`. [ASIC specification, Appendix D](https://download.asic.gov.au/media/nvnh1kjr/msg_spec.pdf). Failure → `INVALID_CHECKSUM`; nine-digit checksum success still establishes only a candidate ACN, not registration kind.
5. Only validated canonical digits/type may leave the entry stage on explicit Next/Enter. Re-normalising a canonical identifier must preserve it (idempotence). Canonically equivalent raw/labelled/formatted values with the same declared type are the same business payload and cache key. Conflicting labels fail before cache/replay/provider access. A valid but different number under the same idempotency key is rejected.

### Live display formatting and placeholders

Formatting is a presentation function, separate from checksum validation. For a known ABN hint, group digits left-to-right as `2-3-3-3`; for a known ACN hint, `3-3-3`. For unlabelled partial input, use groups of three through nine digits; at ten or eleven digits reflow to ABN grouping. Therefore `123456789` displays `123 456 789`, `1234567890` displays `12 345 678 90`, and `12345678901` displays `12 345 678 901`. The reflow never changes canonical digits or submits a request. Prefix hints remain in field state after the label is removed visually; clear or full replacement with an unlabelled value resets the hint. A full replacement with another label replaces it.

Insert spaces after each group only when another digit follows; no trailing spaces. Preserve overflow using further groups of three rather than dropping digits, and reject on submission. If unsupported text remains, retain it visibly for correction rather than displaying a seemingly valid extracted number. Preserve selection/caret by counting digits before the edit; Backspace/Delete beside an inserted separator removes the previous/next digit respectively. Support selection replacement and undo/redo; format after input-method composition ends. No format/normalise action makes a lookup request or emits input contents to analytics.

The labelled input's default placeholder is `e.g. 12 345 678 901`; with a retained ACN hint it is `e.g. 123 456 789`. Always keep the visible helper `ABN: 11 digits. ACN: 9 digits. You can paste labels, spaces or hyphens.` A placeholder is an example, not a label, a prefilled value, or a validity guarantee. Use “Select your expected annual business revenue” and “Select your business category” as empty select prompts; leave optional Trading Name blank with its existing helper rather than adding an unnecessary example name.

| Input | Expected result before lookup |
| --- | --- |
| `51824753556` / `51 824 753 556` / `51-824-753-556` | Canonical ABN `51824753556`; display `51 824 753 556`; passes checksum |
| `ABN: 51-824-753-556` / `abn 51824753556` / `ABN51824753556` | Same canonical ABN/display; passes checksum |
| `ACN: 004-085-616` / `acn004085616` | Canonical ACN `004085616`; display `004 085 616`; passes checksum |
| ABN with non-breaking/narrow spaces, tabs/newlines, or supported Unicode hyphens between groups | Same canonical digits/type/display as the equivalent ASCII-spaced identifier |
| `12345678901` / `ABN: 12-345-678-901` | Canonical `12345678901`; display `12 345 678 901`; invalid checksum; zero calls |
| `51 824 753 557` | ABN checksum failure; zero provider calls |
| `004 085 616` | ACN `004085616`, display `004 085 616`, leading zeros retained, passes checksum |
| `004 085 617` | ACN checksum failure; zero provider calls |
| Blank / whitespace / `ABN:` with separators only | Required input error; zero provider calls |
| `ABN: 004085616` / `ACN: 51824753556` | Label-length error; zero calls; do not switch type |
| Request type ABN plus raw `ACN: 004085616` | Type-conflict error; zero calls |
| `ABN ABN 51824753556` / `ABN ACN 51824753556` / `My ABN is 51824753556` / two labelled numbers | Unsupported text; zero calls; no substring extraction |
| `12345678`, `123456789012`, `51A24753556`, all zeros, full-width digits, slash/dot/comma inside digits, zero-width spaces | Format/sanity failure; zero calls; retain input for correction |

These are format fixtures, not live assertions about registration. The screenshot's identifier is illustrative and must not be assumed to be a valid integration fixture.

## Lookup operation

Proposed `POST /business-drafts/{temporary_business_id}/lookups`, authenticated, with `Idempotency-Key: lookup_attempt_id`:

```json
{
  "identifier_type": "ABN",
  "identifier": "51824753556",
  "onboarding_attempt_id": "attempt-fixture-1",
  "lookup_attempt_id": "lookup-attempt-fixture-1",
  "client_context": {"screen_visit_id": "visit-fixture-1", "app_version": "fixture-1"}
}
```

User ID comes from authentication. Request and idempotency key must agree. Frontend sends canonical digits; server also accepts the defined raw forms and normalises independently before validation, business-payload equality, cache or dispatch. `identifier_type` must agree with any pasted prefix and canonical length. No provider request happens during typing, name entry, catalogue selection, save, or passive screen restoration.

### Lookup deadline and manual recovery

The frontend caps the entire lookup at 3,000 ms from deliberate submission using a monotonic clock. At exactly the deadline, timeout wins over a response observed at or after it. Successful earlier results display immediately. The backend uses a 2,500 ms total budget from request receipt and limits any outbound call to the remaining time; no dispatch after expiry, automatic retry, or polling. The frontend cap remains independent of backend/transport timing. Dispatched work can still incur cost; late completion may update operational records/cache but never the failed UI attempt or a manual draft.

For recoverable failure, FE-05 offers primary “Enter details manually” and optional “Try again” while the lookup-attempt budget below remains. Entering manual mode invalidates the active UI attempt, retains the validated identifier, and makes no extra lookup. A successful lookup is not a prerequisite for saving user-entered identity. Explicit adverse register results and access failures retain FE-05’s specific recovery.

### Lookup-attempt budget

Allow **two server-admitted lookup operations total: the initial attempt plus one explicit retry**, per authenticated owner, server-bound `onboarding_attempt_id`, and canonical `identifier_type`/digits. This is an operation limit, not just a paid-call counter: cache hits, coalesced operations, successful results, and errors/timeouts after admission consume one slot each. A recheck is a new operation under the same budget. Invalid/unauthorised preflight and a known-offline click with no server request consume none. Once admitted, cancellation, timeout, or failure does not refund a slot.

Order: authenticate/authorise and normalise/validate; resolve a retained matching idempotency operation first; otherwise atomically reserve the new operation and one budget slot before cache/coalescing/dispatch. A duplicate key attaches/replays without another slot or provider dispatch even when exhausted; a changed payload under that key is rejected. Concurrent distinct keys cannot admit more than two operations. A third new operation returns HTTP 409 `LOOKUP_RETRY_LIMIT_REACHED`, without consulting cache or calling the provider. No retry/limit rejection triggers an automatic call.

Persist counts for the onboarding attempt’s lifetime, independently of 24-hour operation replay retention. Refresh, Back, sign-in recovery, concurrent tabs, label/spacing edits, clearing/re-entering, or changing away from and back to an identifier do not reset its bucket. A genuinely different canonical identifier/type has its own bucket; only the existing server-owned onboarding workflow can create a new onboarding attempt. Clients cannot reset the counter by supplying a new attempt ID. Broader traffic quotas and bot controls remain separate.

Lookup responses after validated operation handling include `lookup_budget: {"attempt_limit": 2, "attempts_used": 1, "attempts_remaining": 1}` (example after initial admission; exhausted is 2/0). On idempotent replay, preserve the original lookup outcome but return current budget metadata. The owned-draft read restores budget for the retained identifier without a lookup call. A response from an older UI attempt cannot reduce the latest observed count. After response loss/timeout, reconcile uncertain admission through that read before enabling a new operation; if budget cannot be obtained, keep manual entry/help available and withhold retry. Browser-only click counts are not authoritative.

After the second recoverable failure, show FE-05’s `LOOKUP_RETRY_LIMIT_REACHED` feedback with manual entry and help, and remove “Try again”/recheck. A success on the second operation still displays normally; budget exhaustion alone does not replace success or an explicit adverse/access error. The retry limit never blocks manual save and does not apply to save-operation retries.

## Request context for backend analytics

Both lookup and save requests carry optional `client_context` with `screen_visit_id` and `app_version`, bounded/shape-checked as telemetry. The server binds lookup/save operations to its authoritative draft `onboarding_attempt_id`; the lookup request's attempt ID must match that association. A save resolves the attempt from its owned lookup/draft, never by guessing from the browser. User ID, draft ID and environment come from trusted server context.

At operation creation, retain the available client context for later backend events even if the browser leaves. These fields describe the initiating screen/app, not the server version or a later retry. Legacy/missing/invalid context is explicitly `null` with `client_context_status: missing` or `invalid`; provided valid fields use `present`. Telemetry defects alone do not block business operations. Do not fabricate client IDs. Server schema permits nullable client fields; FE events have valid local context. Context is excluded from idempotency business-payload comparison: a retry from a new visit does not create another save or rewrite the original event context.

An eligible result has this shape (all entity data below are synthetic fixtures, not actual registration details for the sample number):

```json
{
  "lookup_id": "opaque-owner-bound-reference",
  "lookup_attempt_id": "lookup-attempt-fixture-1",
  "outcome": "ELIGIBLE",
  "source": "provider",
  "submitted_identifier": {"type": "ABN", "value": "51824753556"},
  "business": {
    "abn": "51824753556",
    "acn": null,
    "status": "ACTIVE",
    "entity_name": "Example Merchant",
    "entity_name_kind": "legal_name",
    "entity_type_code": "IND",
    "entity_type_label": "Individual (Sole Trader)",
    "registered_business_names": ["Example Cafe", "Example Catering"]
  },
  "provenance": {
    "provider": "assumed-abr-adapter",
    "method": "SearchByABNv202001",
    "retrieved_at": "2026-10-28T00:00:00Z",
    "expires_at": "2026-10-29T00:00:00Z",
    "register_last_updated_at": null,
    "record_last_updated_at": null
  }
}
```

Only `ELIGIBLE` results populate the register-derived review form. Manual mode is a separate user-entered branch, never a fabricated eligible lookup result. An ACN lookup echoes the submitted ACN and returns the matched ABN as well. Entity name/type may be absent in a valid register response: the adapter returns INCOMPLETE_RECORD rather than fabricating them. ABN Lookup is not a full ASIC company register; a company without a listed ABN may not resolve through this service. [Lookup coverage](https://abr.business.gov.au/FAQ/ABNLookup).

## Adapter rules and provenance

- Assumed methods: `SearchByABNv202001` for ABNs and `SearchByASICv201408` for ACNs, with historical details excluded. [Method guide](https://abr.business.gov.au/Documentation/WebServiceMethods).
- Distinguish individual legal name from non-individual main name. Use current records according to provider semantics; preserve display punctuation/casing. No name/type guessing when details are suppressed. [Response guide](https://abr.business.gov.au/Documentation/WebServiceResponse).
- Identify requested ABN against the returned ABN. For candidate ACN, require the returned ASIC identifier to match; missing/mismatched linkage is INCOMPLETE_RECORD/IDENTIFIER_MISMATCH. The method also handles ARBN/ARSN/ARFN: matching digits/checksum alone cannot confirm ACN. Use authoritative registration-kind information if the actual provider supplies it, or an Engineering/Risk-confirmed company-type classification mapping supported by fixtures. ABR's documented `ASICNumber` alone contains no registration-kind discriminator; do not invent one. If the result is non-ACN or ACN classification cannot be established, return UNSUPPORTED_IDENTIFIER_TYPE with safe `reason: UNCLASSIFIED_KIND` or `NON_ACN`. Offer manual entry as primary for unclassified kind, or ABN entry/support for confirmed non-ACN, with no automatic second lookup. Classification mapping and the separate product-supported-type allowlist must be agreed before release. A replacement is REPLACED_IDENTIFIER; do not auto-follow it. [Broader ASIC method](https://abr.business.gov.au/Documentation/WebServiceMethods).
- Normalise status to ACTIVE, INACTIVE, or UNKNOWN. ACTIVE alone does not imply product eligibility; for this exercise, eligibility also requires a current identity and a supported business type. Missing required fields or unknown type → INCOMPLETE_RECORD; recognised but unsupported type → UNSUPPORTED_ENTITY_TYPE; unrecognised status → UNKNOWN_STATUS; conflicting current records → AMBIGUOUS_RESULT. The supported-type list is a Product/Risk dependency, not the complete ABR type list.
- Current `businessName` values feed suggestions. Exclude legacy main/other trading names and historical business names. Trim suggestion edges, collapse whitespace only for comparison, and deduplicate case-insensitively; preserve the first current display form and sort by a pinned `en-AU` case-insensitive collator. Text filtering uses the same comparison form and allows zero matches. Manual trading-name input remains available at all times; manual identity entry follows the recovery rules above.
- `retrieved_at` is our server's receipt time; `expires_at` is exactly 86,400 seconds later. Provider register/record timestamps are different fields and do not reset TTL. Preserve the snapshot and version used at save for later evidence comparison. See [data dictionary](https://abr.business.gov.au/Documentation/DataDictionary).

## Errors and HTTP semantics

Proposed error body: `{ "code": "NOT_FOUND", "lookup_attempt_id": "…", "request_id": "…" }`. Do not return credentials, raw provider request echoes, or raw exception text. Reason-to-copy mapping lives in FE-05.

| HTTP | Internal codes |
| --- | --- |
| 400 | INVALID_FORMAT, INVALID_CHECKSUM, IDEMPOTENCY_MISMATCH |
| 401 / 403 | SESSION_EXPIRED / FORBIDDEN |
| 422 | NOT_FOUND, INACTIVE, INCOMPLETE_RECORD, UNSUPPORTED_ENTITY_TYPE, UNSUPPORTED_IDENTIFIER_TYPE, REPLACED_IDENTIFIER, AMBIGUOUS_RESULT, IDENTIFIER_MISMATCH, UNKNOWN_STATUS, OPTION_INVALID, INVALID_TRADING_NAME, INVALID_BUSINESS_NAME |
| 409 | DRAFT_CONFLICT, LOOKUP_EXPIRED, SAVE_IN_PROGRESS, LOOKUP_RETRY_LIMIT_REACHED |
| 502 / 503 / 504 | INVALID_RESPONSE / PROVIDER_UNAVAILABLE or PROVIDER_CONFIGURATION / TIMEOUT |

OFFLINE, LOOKUP_INTERRUPTED, and SAVE_FAILED may be frontend-derived safe codes. Provider HTTP success does not guarantee business success: inspect application exceptions. Map an explicit no-records result to NOT_FOUND; timeouts, credentials, unknown descriptions, invalid XML, and unknown responses never map to NOT_FOUND. Provider `WEBSERVICES` is broad and needs classification; do not treat the code alone as a user typo. [Exception guide](https://abr.business.gov.au/Documentation/Exceptions).

UNSUPPORTED_IDENTIFIER_TYPE is a 422 outcome with safe `reason: UNCLASSIFIED_KIND` or `NON_ACN`; its copy/actions are in FE-05. This distinction lets coverage uncertainty offer manual entry without treating a known non-ACN as an ACN. INVALID_FORMAT includes safe reasons `UNSUPPORTED_TEXT`, `INVALID_LENGTH`, `LABEL_LENGTH`, `TYPE_CONFLICT`, or `ALL_ZERO`; FE-01 owns their concise feedback. Field normalisation failures never dispatch a paid request.

## Saving and restoration

Proposed `PUT /business-drafts/{temporary_business_id}/business-information`, authenticated, with `Idempotency-Key: save_operation_id` and expected draft revision:

```json
{
  "identity_source": "register",
  "lookup_id": "opaque-owner-bound-reference",
  "trading_name": null,
  "trading_name_source": "none",
  "annual_revenue_band_id": "revenue_test_1",
  "business_category_id": "category_test_1",
  "catalogue_version": "fixture-only-v1",
  "expected_draft_revision": 3,
  "save_operation_id": "save-fixture-1",
  "client_context": {"screen_visit_id": "visit-fixture-1", "app_version": "fixture-1"}
}
```

The save body has two mutually exclusive identity branches. The example above uses `identity_source: register`: require an owned `lookup_id`, reject client-supplied official identity, and resolve identity/provenance from its snapshot. A manual save replaces those two identity fields with:

```json
{
  "identity_source": "user_entered",
  "lookup_id": null,
  "manual_business": {
    "identifier_type": "ACN",
    "identifier": "004085616",
    "entity_name": "Example Merchant Pty Ltd",
    "entity_type_code": "approved-company-type-fixture"
  }
}
```

All remaining save fields are identical. The type code above is a fixture; use approved supported-type catalogue IDs in production. Require `manual_business` only for `user_entered`; reject mixed/missing branches. Server normalises/validates the canonical identifier under the same rules as lookup, requires an entered entity name of 1–200 Unicode code points after trimming with no controls/line breaks, and validates supported type/options and draft revision. Return field-specific `INVALID_BUSINESS_NAME` (`REQUIRED`, `TOO_LONG`, `CONTROL_CHARACTERS`) or `OPTION_INVALID` errors using FE-03 copy. User-entered identity requires no successful lookup, lookup reference, resolved ABN for ACN-only input, or provider call. From server-known results associated with this draft/identifier, enforce FE-05’s explicit adverse-result recovery; do not trust a client claim that the earlier lookup failed. An unknown/failed lookup alone is never an adverse eligibility result.

Persist `identity_source` for both branches. The server sets `verification_status: unverified` for manual identity and stores no fabricated register status, retrieved timestamp, cache expiry, or provider provenance. Register-derived identity retains its snapshot provenance, without implying ownership verification or AML clearance. A manual save completes this information-capture step and advances through the existing next route; verification/payment activation remain later work. Restore manual identity as editable fields with its source/status intact. A late lookup cannot attach itself to or replace it. Converting an expired displayed snapshot to manual entry requires user review and an explicit manual save; retain prior lookup history separately, never relabel typed answers as register data.

Blank optional name → null/none. Otherwise trim outer whitespace, require 1–200 Unicode code points and no C0/C1 control characters, line/paragraph separators, or newlines; retain interior characters and punctuation. Selection gives `register_selection`; custom text gives `user_entered`. On save, server verifies a claimed `register_selection` matches this snapshot's suggestions; user-entered text need not match anything. Manual identity has no register suggestions and accepts only `none` or `user_entered` trading-name source. On conversion from an expired snapshot, preserve any trading-name text as `user_entered`. All names are rendered as text.

Server field errors include a field ID and safe reason enum: INVALID_TRADING_NAME uses `TOO_LONG` or `CONTROL_CHARACTERS` and the matching FE-03 copy; missing/invalid revenue or category uses the matching FE-04 field copy. A claimed register selection that does not match is rejected with instructions to review the field or submit it as user-entered text. Preserve all other values and focus the affected field. Retrieved identity remains read-only; manual recovery saves a distinct user-entered identity branch, never edits the provider snapshot.

### Save idempotency ordering

1. Authenticate and verify current user/draft ownership and safe request shape. Compare business payload under the owner/draft/key (exclude `client_context`). Reused key with different business answers → IDEMPOTENCY_MISMATCH. Serialise concurrent deliveries with the same key.
2. If this operation is already committed, return its original acknowledgement **before** rechecking lookup TTL, catalogue changes, or expected revision. A retry after lookup expiry or a newer draft revision must not turn a previously committed save into LOOKUP_EXPIRED/DRAFT_CONFLICT. Return the original completion ID; the frontend then reads the latest draft before continuing, so old answers never overwrite newer ones. Do not rerun mutation or re-emit completion.
3. If the same operation is still pending, return 409 SAVE_IN_PROGRESS with no second mutation. The UI retains answers and offers an explicit same-key retry; no polling or provider call. A terminal rejected operation replays its original rejection; after correction/recheck, use a new operation ID. Do not replay transient failures with unknown commit status as a definitive rejection: reconcile the transaction first.
4. Only a **new** mutation checks branch-specific identity, catalogue IDs/version, expected revision, and name rules. Register-sourced saves check the user-owned lookup’s unexpired snapshot; manual saves validate entered identity and require no lookup/TTL. Idempotency payload equality includes identity source and every identity/business field, so switching to manual mode uses a new save operation ID. Atomically commit answers, draft revision, completion identity, and durable completion-event intent. Backend alone publishes `business_info.completed`; idempotent delivery/deduplication by completion ID ensures one logical event, including lost responses and retrying event transport.
5. Return `draft_revision`, `completion_id`, and existing-contract `next_step`. Retain idempotency outcomes for at least 24 hours from operation creation, matching lookup replay retention. Once retention ends, reconcile the saved draft before issuing a new uncertain save. Operation/event context retains the original initiating client values, with explicit nulls when missing.

Restore reads the last saved draft without a lookup. Store canonical identifiers from the selected identity branch, keep register identity and merchant-entered identity/answers distinct, and use display formatters in all later fields. Expired saved register provenance may be shown as historical saved information, but a new changed register-sourced save requires explicit recheck or reviewed manual entry. Manual saved identity remains editable and does not expire with lookup TTL. For a previously register-derived identity, recheck retains answers only for an unchanged canonical ABN and displays the refreshed identity. A retry from manual mode uses FE-03's same-submitted-identifier review before any source switch.

## Fixture set for integrated verification

Entry → lookup → details with one browser Back to entry; manual/error/save states without extra history entries; browser Forward without dispatch; interrupted lookup restoration; refreshed/deep-linked details with and without owned state; changed identifier followed by Forward; Back during save then late commit; return after an uncertain save; help overlay/full-page Back; session-expired route restoration; URL/history privacy; eligible sole trader; eligible company via confirmed ACN; known non-ACN ASIC registration; ASIC registration with unconfirmed kind; missing client context; original context preserved after browser exit/retry from another visit; no business names; current/historical names; blank/custom/overlength name; suppressed identity; inactive/replaced/reissued/conflicting statuses; mismatched identifier; not found; transport/application/configuration/schema errors; success at 2,999 ms; timeout at exactly 3,000 ms; late success during manual entry; immediate manual recovery after offline/provider/no-match/incomplete failures; backend deadline exhaustion before dispatch; initial plus one retry; third new operation rejected; concurrent distinct/same-key admissions at the cap; replay with current budget metadata; budget restoration after reload/Back/another tab; canonical-equivalent inputs and changing away/back; budget survival after idempotency retention expiry; unknown admission after response loss; second-attempt success and failure; expired lookup with exhausted budget into manual save; manual ABN and ACN-only save/restore with no lookup ID; manual name/type validation; branch switching with new save key; explicit adverse-result protection; exact cache expiry; equivalent labelled/spaced cache and replay keys; same-key concurrency; cross-user lookup access; stale catalogue; save conflict; committed save replay across expiry and across a newer revision; pending save replay; successful commit with lost response and exactly one logical server completion. Reissued identifiers use current record semantics; never revive historical identity from an older cache key/version.
