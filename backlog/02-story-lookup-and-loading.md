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

Implements screen 2 and the transition into screen 3. Each paid lookup must follow deliberate user intent. The frontend calls the authenticated Zeller adapter described in [the contract](08-shared-contracts.md); it never calls ABR directly. Backend availability is an integration dependency; fixtures allow frontend work now.

## Story statement

**As a merchant, I want to know that my business lookup is in progress, so that I can wait confidently without submitting it again.**

## Deterministic acceptance criteria

1. **Start:** Given a normalised, validated submission, issue one request with canonical digits/type, a new `lookup_attempt_id`, and available `client_context` per the shared contract; immediately enter `lookup_pending`. Labels/spacing never enter business identifiers downstream. Disable editing, clear, and Next; replace Next's label with a spinner and accessible “Looking up your business…”. Keep Back available, as illustrated.
2. **Repeated clicks:** While pending, further Next/Enter events produce no extra requests. Network re-delivery of the same operation uses the same idempotency key.
3. **Success:** Given an eligible result for the current attempt before the deadline, stop loading and show FE-03 using that result and its `lookup_id`. A cache hit follows the same user journey without an artificial minimum spinner time.
4. **Deadline:** At 10,000 ms without a completed response, stop loading and show FE-05's timeout action. A response observed at or after the deadline cannot change this failed attempt's UI. No automatic retry occurs.
5. **Back or navigation:** Leaving pending state invalidates that attempt for UI purposes and aborts the browser request where possible. A late response does not reopen this step. Backend calls already dispatched can still complete and be logged/cached.
6. **Race protection:** If a later attempt has started, a previous attempt's response cannot populate, clear, save, or emit a displayed-success event for the later attempt, even if its response arrives last.
7. **Errors:** An adapter error stops loading and routes to FE-05. A provider failure is never displayed as a registration-not-found result. No prior business details remain under a new identifier.
8. **Accessibility:** Mark the region busy and announce loading once, then announce success or the actionable error. On success, move focus to the returned-details heading; do not trap keyboard focus in the spinner.

## Feedback and interaction

Loading copy is “Looking up your business…”. Error copy is owned by [FE-05](05-story-errors-and-help.md). Retry is explicit and uses a new attempt ID; a transport re-delivery of an existing operation reuses its ID. Leaving the page is not a guarantee that an incurred lookup charge was cancelled.

## Implementation & architectural guardrails

- Keep a single authoritative state machine and active attempt reference; avoid independent loading flags that can disagree with displayed business data.
- Server caching, coalescing, validation, and durable call logging belong to BE-01. Browser cancellation and disabled buttons are not sufficient cost controls.
- Emit lookup submission and result-displayed events per DATA-01. Backend cost/result records are authoritative for dispatched calls, including abandoned ones.
- Do not store credentials, provider XML, or complete results in browser persistence or error-reporting breadcrumbs.

## Out of scope / non-goals

Background polling, automatic retries, name search, browser-managed provider cache, rate limiting, and bot detection.

## Verification / handover

Use a controlled clock and fixtures for immediate cache success, slow provider success, failure, and timeout. Double-click/Enter produces one request; resolve request A after navigating Back and starting B to prove A cannot replace B. Confirm a timeout then a late success remains an error until explicit retry.
