---
id: ZBI-FE-02
epic: ZBI-EPIC
type: story
title: Look up the business with clear loading feedback
status: interview-draft
priority: P1
owner: Frontend
delivery_scope: core-happy-path
depends_on: [ZBI-FE-01, ZBI-BE-01]
contract: 08-shared-contracts.md
schema_version: 1
---

# FE-02: Look up the business with clear loading feedback

## Context

Implements screen 2 and the transition into screen 3. Each paid lookup must follow deliberate user intent. The frontend calls the authenticated Zeller adapter described in [the contract](08-shared-contracts.md); it never calls ABR directly. Lookup is a convenience for prefilling details, not a prerequisite for continuing. Backend availability is an integration dependency; fixtures allow frontend work now.

## Story statement

**As a merchant, I want a quick business lookup with clear feedback and an easy manual fallback, so that I can continue onboarding without waiting on the lookup service.**

## Deterministic acceptance criteria

1. **Start:** Given a normalised, validated submission, issue one request with canonical digits/type, a new `lookup_attempt_id`, and available `client_context` per the shared contract; push the details-step URL exactly once and immediately enter `lookup_pending`. This history transition also applies to immediate recoverable failure such as known offline state. Labels/spacing never enter business identifiers downstream. Disable editing, clear, and Next; replace Next's label with a spinner and accessible “Looking up your business…”. Keep Back available, as illustrated.
2. **Repeated clicks:** While pending, further Next/Enter events produce no extra requests. Network re-delivery of the same operation uses the same idempotency key.
3. **Success:** Given an eligible result for the current attempt before the deadline, stop loading and show FE-03 using that result and its `lookup_id`. If manual answers already exist for this identifier, follow FE-03's explicit review before replacing them. A cache hit follows the same user journey without an artificial minimum spinner time.
4. **Deadline:** Start a monotonic 3,000 ms deadline when the user submits, covering the entire lookup request. At the deadline without a completed response, stop loading and show “We couldn’t look up your business. Enter your business details manually to continue.” with primary “Enter details manually” and secondary “Try again” only while the retry budget remains. When both attempts have been used, show FE-05's retry-limit copy/actions instead. Abort the browser request where possible. A response observed at or after the deadline cannot change this failed attempt's UI. No automatic retry occurs.
5. **Back or navigation:** Leaving pending state invalidates that attempt for UI purposes and aborts the browser request where possible. Browser Back returns to the identifier URL in one action with the identifier retained; form Back follows the same destination. Browser Forward restores matching available state or interrupted-lookup recovery without automatically dispatching or restarting a spinner. A late response does not reopen this step. Backend calls already dispatched can still complete and be logged/cached.
6. **Race protection:** If a later attempt has started, a previous attempt's response cannot populate, clear, save, or emit a displayed-success event for the later attempt, even if its response arrives last.
7. **Errors and fallback:** Stop loading immediately on failure; do not wait out the deadline. FE-05 offers manual entry as the primary recovery for timeout, offline, provider/response failure, no match, or incomplete details. “Enter details manually” opens FE-03's editable form with the validated identifier retained, without another lookup or a `lookup_id` requirement. Invalidate the active attempt before entering manual mode so no late result can populate or overwrite it. Explicit confirmed ineligibility/mismatch and access errors keep their specific recovery in FE-05. A provider failure is never displayed as registration-not-found. No prior business details remain under a new identifier.
8. **Retry limit:** Allow the initial lookup plus one explicit retry: two server-admitted lookup operations per canonical identifier/type per onboarding attempt. After the second recoverable failure, remove “Try again” and show FE-05’s retry-limit feedback with manual entry and help. The same limit applies to entry-screen resubmission and explicit recheck. Restore the server-owned budget on refresh/Back; changing formatting or returning to an earlier identifier cannot reset it. Same-key re-delivery does not consume an attempt. If a timeout leaves admission uncertain, reconcile the budget through the owned-draft read before enabling a new retry; if that read fails, manual entry remains available. See the shared contract for counting and concurrency.
9. **URL/history:** Success, failure, timeout, retry, and entering manual mode update the same details URL/history entry. Refresh or direct details entry restores owned state or follows the shared entry fallback; it never initiates lookup. Do not put the identifier, names, or provider response in the URL/history state.
10. **Accessibility:** Mark the region busy and announce loading once, then announce success or the actionable error. On success, move focus to the returned-details heading; do not trap keyboard focus in the spinner.

## Feedback and interaction

Loading copy is “Looking up your business…”. Error copy is owned by [FE-05](05-story-errors-and-help.md). Manual entry is the primary fallback; retry is optional, explicit, limited to one after the initial lookup, and uses a new attempt ID; a transport re-delivery of an existing operation reuses its ID. Leaving the page is not a guarantee that an incurred lookup charge was cancelled.

## Implementation & architectural guardrails

- Keep a single authoritative state machine and active attempt reference; avoid independent loading flags that can disagree with displayed business data.
- Server caching, coalescing, validation, and durable call logging belong to BE-01. Browser cancellation and disabled buttons are not sufficient cost controls.
- Emit lookup submission and result-displayed events per DATA-01. Backend cost/result records are authoritative for dispatched calls, including abandoned ones.
- Do not store credentials, provider XML, or complete results in browser persistence or error-reporting breadcrumbs.

## Out of scope / non-goals

Background polling, automatic retries, name search, browser-managed provider cache, general traffic rate limiting, and bot detection. The per-identifier two-attempt lookup budget is in scope.

## Verification / handover

Use a controlled clock and fixtures for immediate cache success, slow provider success, failure, and timeout. Double-click/Enter produces one request; resolve request A after navigating Back and starting B to prove A cannot replace B. Check success at 2,999 ms, timeout at exactly 3,000 ms, and responses after the deadline. Confirm immediate failures expose manual entry without delay; choosing it makes zero extra lookup calls, and late success cannot replace typed details. Verify manual completion through FE-04 and one optional explicit retry. After both attempts, confirm no third lookup from Retry, Next/Enter, Back/re-entry, refresh, or another tab; manual entry/help remain available. Check same-key replay at the limit and budget reconciliation after a lost response. Use real router history to check one Back to entry from pending/success/error/manual states, Forward after an interrupted lookup, refresh/deep-link fallback, late-response suppression, and no provider dispatch on restoration.
