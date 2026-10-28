---
id: ZBI-DATA-01
epic: ZBI-EPIC
type: story
title: Measure the business-information funnel and time at each stage
status: interview-draft
priority: P1
owner: Data-with-Frontend-and-Backend
delivery_scope: supporting-measurement
depends_on: [existing-analytics-pipeline, ZBI-BE-01]
contract: 08-shared-contracts.md
schema_version: 1
---

# DATA-01: Measure the funnel and time at each stage

## Context

Measure progress toward faster onboarding and the cost of lookups without collecting business details in behavioural analytics. All events use the authenticated user ID and temporary business ID. A stable onboarding attempt can include several lookups, retries, and revisits.

## Story statement

**As a product manager, I want to see where merchants progress, pause, or fail and how long each stage takes, so that I can improve onboarding speed, completion, and lookup cost.**

## Deterministic acceptance criteria

1. Every event contains `schema_version`, unique `event_id`, UTC `occurred_at`, `user_id`, `temporary_business_id`, authoritative `onboarding_attempt_id`, environment, `screen_visit_id`, `app_version`, and `client_context_status`. FE events use their local visit/app context with status `present`. Backend events use retained initiating-request `client_context`; unavailable/invalid client fields are explicitly nullable with status `missing`/`invalid`. The server derives user/draft/environment/attempt association; the lookup-supplied attempt must match the owned draft, and save derives it from the owned lookup/draft. Add lookup IDs when available; no email address substitutes for user ID.
2. Emit the events below at their stated trigger. Frontend reports UI behaviour; the backend alone produces `business_info.completed` on durable save commit, regardless of acknowledgement reaching the browser. Frontend success/navigation never duplicates that event. Join via operation IDs, not raw ABN/ACN. Persist the original operation's client context so backend events remain attributable after navigation; missing telemetry does not block business operations.
3. A new visible mount of the business-information step gets a new `screen_visit_id`; each stage entry gets a new `stage_visit_id`. Keep the same onboarding attempt across Back, refresh, and retries until the existing onboarding workflow starts a genuinely new application attempt.
4. Time stages `identifier_entry`, `lookup_pending`, `business_details`, and `save_pending`. Record stage start/end, `elapsed_ms`, `active_ms`, and exit reason. Use a monotonic clock within a visit. Pause active time while the page is hidden; elapsed time continues. Example: 10 seconds visible + 20 hidden + 5 visible gives 15,000 active ms and 35,000 elapsed ms.
5. On Back, timeout, failure, or navigation, end the active stage once with its reason; on re-entry create a new stage visit. Best-effort page exit may be missing; mark such durations incomplete and exclude them from complete-duration percentiles rather than inventing an end timestamp.
6. “Abandoned” is a derived diagnostic after 30 minutes with neither completion nor further activity, not a reliable browser-close event. A later return/completion reclassifies the attempt. A separate 24-hour cohort completion metric includes eventual completion within that window.
7. Deduplicate by `event_id`; server completion also deduplicates by `completion_id`. Atomically commit the save and durable completion-event intent; retry event delivery with the same identity. Replayed saves produce no new completion. Retried uploads preserve event ID. A lost browser response followed by save replay after expiry/revision changes still yields one logical server completion. Context changes on a retry do not change idempotency identity or overwrite the original event's initiating visit.
8. Product analytics failure never blocks lookup or save. Respect existing consent/collection policy; report excluded or missing telemetry separately. Operational cost records remain governed by the existing operational logging policy.
9. Analytics contains no ABN/ACN, raw or hashed business name, custom text, registration documents, credentials, provider payload, or raw exception descriptions. Error/name-mode/category interactions use approved enums/IDs only; avoid replay tools capturing input values.
10. Demonstrate the metrics in the epic from fixture journeys, and prove that UI retries do not inflate completion or provider-cost totals.

## Event catalogue

| Event | Trigger / owner | Additional properties |
| --- | --- | --- |
| `business_info.step_viewed` | Visible screen entry / FE | entry source, restored draft flag |
| `business_info.stage_started` | Stage entered / FE | stage, stage visit ID |
| `business_info.stage_ended` | Stage left / FE | stage, stage visit ID, elapsed/active ms, exit reason, timing complete flag |
| `business_info.validation_failed` | Submitted invalid form / FE | field ID, safe reason code |
| `business_info.lookup_submitted` | Valid deliberate lookup / FE | identifier type, lookup attempt ID |
| `business_info.lookup_resolved` | Request result resolved / BE | source (`provider`, `cache`, `coalesced`, `replay`), outcome, request latency, provider call ID if applicable |
| `business_info.details_viewed` | Current result actually displayed / FE | lookup ID, cache age ms, identifier type |
| `business_info.field_changed` | Committed changed selection or trading-name blur / FE | field ID; name mode or approved option ID, never name text |
| `business_info.error_shown` | Actionable error displayed / FE | stage, reason code |
| `business_info.help_opened` | Help or support link activated / FE | reason code, approved target ID |
| `business_info.navigation_clicked` | Back/clear/recheck activated / FE | action enum, stage |
| `business_info.save_submitted` | Valid save submitted / FE | save operation ID, name mode |
| `business_info.completed` | Durable save committed / BE | save operation ID, completion ID, draft revision |

`lookup_resolved` can exist without `details_viewed` when the merchant navigates away. Join operational ledger records to results for cost, not frontend clicks. Completion means this step's information was saved, not AML passed or payments enabled.

## Implementation & architectural guardrails

Use the existing event platform and naming conventions where equivalent. Both lookup/save requests carry the shared optional `client_context` envelope. Distinguish validated telemetry shape from trusted user identity; clients cannot set the server's ownership context. Keep schemas versioned. Aggregate durations without overlapping-tab double counts; initially flag concurrent-tab attempts as timing-incomplete. Do not log keystrokes, raw/normalised/formatted identifiers, or clipboard text. Access and retention follow existing policies, confirmed before rollout.

## Out of scope / non-goals

Building a new warehouse, session replay, identifying users from business numbers, defining whole-onboarding payment-readiness events, and guaranteed telemetry on abrupt browser termination.

## Verification / handover

Inspect captured events for happy path, local validation failure, cache hit, timeout, Back/revisit, help, save timeout with committed server outcome, and hidden-tab timing. Validate schema, privacy exclusions, deduplication, and the 15-second-active/35-second-elapsed example. Reconcile two actual provider calls to the AUD $1.20 estimate independently of event upload retries.

Test absent/malformed `client_context`, completion after browser exit, retry from a different screen visit, and committed-save replay across lookup expiry/newer revision. Expect explicit nullable client fields, correct server-owned attempt IDs, preserved original operation context, and exactly one logical backend completion.
