---
id: ZBI-FE-05
epic: ZBI-EPIC
type: story
title: Recover from lookup and save problems with useful feedback
status: interview-draft
priority: P1
owner: Frontend-and-Content
delivery_scope: supporting-recovery
depends_on: [ZBI-FE-02, ZBI-FE-04, ZBI-BE-01]
contract: 08-shared-contracts.md
schema_version: 1
---

# FE-05: Recover from lookup and save problems

## Context

The original brief asks for an unhappy-path list; this supporting story specifies the user's requested errors and help. The frontend consumes stable adapter reason codes and never renders raw provider exceptions. These recovery paths supplement the four core frontend tickets.

## Story statement

**As a merchant whose business information cannot be retrieved or saved, I want a clear explanation and a useful next action, so that I can fix the problem without guessing or losing my progress.**

## Deterministic acceptance criteria

1. For each reason below, show the exact corresponding copy and action; stop pending state and announce the error. Keep the submitted identifier. Clear any unrelated prior lookup result.
2. NOT_FOUND includes a “Why can't I find my business?” link to the in-app help content specified below. Help opens without losing the form or making a lookup call; a fallback link remains available if rich help fails to render.
3. Client and server normalise permitted labels/separators before format/checksum validation. Prefix/length/type conflicts use FE-01's feedback and make zero provider calls. A known offline browser state shows offline feedback without dispatching a lookup; a connection lost after dispatch may still incur a charge.
4. Retry is always explicit. NOT_FOUND retry starts a new attempt; transient transport re-delivery uses the original operation ID. There are no background retries, refresh-on-focus requests, or retry-on-keystroke requests.
5. Ineligible, suppressed/incomplete, ambiguous, replaced, mismatched, unknown-status, or non-ACN/unclassified ASIC-number records cannot proceed as a confirmed business in this branch. Offer ABN entry/support for an unsupported identifier type; no automatic second lookup. Do not silently follow replacements or invent official fields.
6. Session expiry returns through the existing login recovery and restores the last saved draft; never leak another draft's data. Forbidden draft access displays generic access feedback and dispatches zero provider calls.
7. LOOKUP_EXPIRED asks the merchant to recheck explicitly. Recheck uses a new lookup attempt and retains typed answers only if the returned canonical ABN is unchanged; otherwise clears them. Changed official identity is shown for review before save.
8. A save conflict offers reload of the latest saved draft; it does not overwrite it. The action warns that unsaved edits will be replaced.
9. Each error emits a reason-coded error-displayed event; help clicks emit a target ID. No identifier, name, provider exception, or free-text error enters analytics.

## Error catalogue

`{type}` is “ABN” or “ACN” from validated input, not provider text.

| Adapter/UI reason | Exact customer copy | Recovery action |
| --- | --- | --- |
| NOT_FOUND | {type} not found. Please check and try again. | Edit and retry; show help link |
| OFFLINE | You're offline. Check your connection and try again. | Retry after connection returns |
| TIMEOUT | This is taking longer than expected. Please try again. | Retry |
| PROVIDER_UNAVAILABLE / PROVIDER_CONFIGURATION / INVALID_RESPONSE | We couldn't look up your business right now. Please try again shortly. | Retry or existing support link; internal alert for configuration/schema failures |
| INACTIVE | This ABN isn't active. Check the number or contact us for help. | Edit identifier; help/support |
| INCOMPLETE_RECORD | We found your business, but some details aren't available. Contact us for help. | Existing support route; no fabricated details |
| UNSUPPORTED_ENTITY_TYPE | We can't continue with this business type online. Contact us for help. | Existing support route; apply only to the Product-approved exercise eligibility policy |
| UNSUPPORTED_IDENTIFIER_TYPE | We couldn't confirm this as an ACN. Enter your ABN or contact us for help. | Edit identifier to ABN or use support; no automatic lookup |
| REPLACED_IDENTIFIER | This ABN has been replaced. Enter your current ABN and try again. | Edit identifier; no automatic second lookup |
| AMBIGUOUS_RESULT / IDENTIFIER_MISMATCH / UNKNOWN_STATUS | We couldn't confirm these business details. Check the number or contact us for help. | Edit identifier or support |
| LOOKUP_EXPIRED | Your business lookup has expired. Please check your details again. | Explicit “Check business again” |
| SAVE_FAILED | We couldn't save your details. Please try again. | Retry with answers retained |
| SAVE_IN_PROGRESS | Your details are still being saved. Please try again shortly. | Retain answers; explicit retry of the same save operation; no second mutation or automatic polling |
| OPTION_INVALID | This option is no longer available. Please choose another. | Reload choices; focus affected field |
| DRAFT_CONFLICT | Your details changed in another window. Reload to continue. | “Reload saved details”; warn before discarding unsaved edits |
| SESSION_EXPIRED | Your session has expired. Sign in to continue. | Existing sign-in recovery |
| FORBIDDEN | We couldn't open this application. Return to your onboarding to continue. | Existing onboarding entry |

## Help content: “Why can't I find my business?”

Content belongs in the existing help surface; proposed internal content ID is `business-lookup-help`. The exact route is an integration dependency, not a newly asserted Zeller URL.

> Check you've entered an 11-digit ABN or a 9-digit ACN, and that it belongs to the business you're signing up.
>
> You can paste a number with an ABN or ACN label, spaces or hyphens. We'll format it for you. If you include a label, make sure it matches the number type.
>
> Newly registered or updated details may take time to appear. Check that your registration is active, then try again later. You can check the public register or get help if your active ABN still isn't showing.
>
> An ACN may exist even when a business isn't listed in ABN Lookup. Try your ABN if you have one.
>
> Missing trading name? You can leave this field blank or type the name you trade under. It doesn't have to appear in the suggestions.

Include actionable links to [ABN Lookup](https://abr.business.gov.au/), [registration/application help](https://abr.business.gov.au/FAQ/ABNBasics), [missing-name help](https://abr.business.gov.au/FAQ/Names), [active ABN missing guidance](https://www.abr.gov.au/general-information/scheduled-site-maintenance), and the existing Zeller support route. External links preserve the form and have accessible labels. The proposed 24–48-business-hour estimate stays in epic A13 until provider-specific timing is confirmed.

## Implementation & architectural guardrails

Use stable internal codes, not raw exception strings. Treat HTTP success with an exception payload as an error. Missing public details can be legitimate suppression and must not be guessed from other customers' data. The support handoff uses a safe correlation reference, not credentials or raw XML.

## Out of scope / non-goals

Building a support platform or manual review queue, restoring government records, implementing rate-limit/bot challenges, and promising a registration publication SLA.

## Verification / handover

Fixture every catalogue row, check copy and action, verify help preserves form values, and prove technical errors cannot become NOT_FOUND. See the broader [unhappy-path register](09-unhappy-paths-and-decisions.md).
