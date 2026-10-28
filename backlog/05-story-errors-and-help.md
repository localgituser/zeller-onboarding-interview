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
2. NOT_FOUND includes a “Why can't I find my business?” link to the in-app help content specified below. Errors remain on the originating entry/details URL; errors, retry, and manual-mode transitions add no history entries. Help follows the shared navigation contract: an overlay closes back to the form with trigger focus restored, and full-page help has its own history entry so browser Back restores the originating form. Help opens without losing same-session form state or making a lookup call; a fallback link remains available if rich help fails to render.
3. Client and server normalise permitted labels/separators before format/checksum validation. Prefix/length/type conflicts use FE-01's feedback and make zero provider calls. A known offline browser state shows offline feedback without dispatching a lookup; a connection lost after dispatch may still incur a charge.
4. For recoverable lookup failures, show primary “Enter details manually” and secondary “Try again” only while the two-attempt budget remains (after connection returns when offline); keep identifier editing and relevant help available. Manual entry opens FE-03 without another request, retains the validated identifier, invalidates the old UI attempt, and can save through FE-04 without a lookup reference. Retry is always explicit. NOT_FOUND retry starts a new attempt; transient transport re-delivery uses the original operation ID. Once both server-admitted attempts have been used, remove all lookup-retry/recheck actions for that canonical identifier/type in this onboarding attempt; a second recoverable failure uses the retry-limit copy below. Manual entry and help remain available for recoverable failures. A server `LOOKUP_RETRY_LIMIT_REACHED` response stops pending state without another provider call. There are no background retries, refresh-on-focus requests, or retry-on-keystroke requests.
5. Manual details are never presented as register-confirmed. Missing/suppressed identity and unclassified ASIC-number results may use manual entry with unverified provenance. Explicit inactive/unsupported entity, replaced identifier, ambiguous/mismatched identity, unknown status, and confirmed non-ACN results keep their correction/support action; manual entry must not silently bypass an explicit adverse register result. An unclassified ASIC result cannot be labelled a confirmed ACN. Offer ABN entry/support for a confirmed non-ACN identifier; no automatic second lookup. Do not silently follow replacements or invent official fields.
6. Session expiry returns through the existing login recovery with an allowlisted internal return route and restores the last saved owned draft. Restore the requested details route only when its required state exists; otherwise replace it with identifier entry. Recovery never automatically restarts a lookup or save, and unsaved edits across sign-in are not guaranteed; never leak another draft's data. Forbidden draft access displays generic access feedback and dispatches zero provider calls.
7. LOOKUP_EXPIRED offers manual entry as primary recovery, retaining displayed details as editable user-entered values requiring review, or an explicit recheck as secondary recovery while budget remains. An exhausted budget leaves manual entry and help; it does not block manual save. Recheck uses a new lookup attempt and retains typed answers only if the returned canonical ABN is unchanged; otherwise clears them. Changed official identity is shown for review before save.
8. A save conflict offers reload of the latest saved draft; it does not overwrite it. The action warns that unsaved edits will be replaced.
9. Browser Back from a details-step error returns to identifier entry with the submitted ABN/ACN retained; Forward restores safe matching state without a lookup. Keep the retry budget across help/authentication/history navigation. If lookup was interrupted with a retained validated identifier but no usable result, show “Your business lookup was interrupted. Enter your details manually or try again.”; offer manual entry and explicit retry only while budget remains, otherwise the retry-limit feedback.
10. Each error emits a reason-coded error-displayed event; help clicks emit a target ID. No identifier, name, provider exception, or free-text error enters analytics.

## Error catalogue

All lookup retry/recheck actions below are conditional on the shared two-attempt budget. After the second recoverable failure, use `LOOKUP_RETRY_LIMIT_REACHED` copy/actions even when the underlying reason is TIMEOUT, NOT_FOUND, or another recoverable failure; retain that underlying safe reason for diagnostics. Save-operation retries are unaffected.

`{type}` is “ABN” or “ACN” from validated input, not provider text.

| Adapter/UI reason | Exact customer copy | Recovery action |
| --- | --- | --- |
| NOT_FOUND | We couldn’t find your business. Check your {type} or enter your business details manually to continue. | Enter details manually (primary); edit/retry and help link |
| OFFLINE | You’re offline. Enter your business details manually, then save when you’re back online. | Enter details manually (primary); retry lookup after connection returns |
| LOOKUP_RETRY_LIMIT_REACHED | We couldn’t retrieve your business details. Enter them manually to continue. | Enter details manually (primary); existing help/support link; no further lookup for this identifier in this onboarding attempt |
| LOOKUP_INTERRUPTED (frontend restoration) | Your business lookup was interrupted. Enter your details manually or try again. | Enter details manually (primary); explicit retry only while budget remains, otherwise retry-limit feedback; no automatic lookup |
| TIMEOUT | We couldn’t look up your business. Enter your business details manually to continue. | Enter details manually (primary); Try again (secondary) |
| PROVIDER_UNAVAILABLE / PROVIDER_CONFIGURATION / INVALID_RESPONSE | We couldn’t look up your business. Enter your business details manually to continue. | Enter details manually (primary); Try again (secondary); internal alert for configuration/schema failures |
| INACTIVE | This ABN isn't active. Check the number or contact us for help. | Edit identifier; help/support |
| INCOMPLETE_RECORD | Some business details aren’t available. Enter your business details manually to continue. | Enter details manually (primary); optional retry/help; do not mark missing or entered fields register-confirmed |
| UNSUPPORTED_ENTITY_TYPE | We can't continue with this business type online. Contact us for help. | Existing support route; apply only to the Product-approved exercise eligibility policy |
| UNSUPPORTED_IDENTIFIER_TYPE / UNCLASSIFIED_KIND | We couldn’t confirm your business details. Enter them manually to continue. | Enter details manually (primary), with unverified provenance; optional ABN correction/help; no automatic lookup |
| UNSUPPORTED_IDENTIFIER_TYPE / NON_ACN | We couldn't confirm this as an ACN. Enter your ABN or contact us for help. | Edit identifier to ABN or support; no automatic lookup |
| REPLACED_IDENTIFIER | This ABN has been replaced. Enter your current ABN and try again. | Edit identifier; no automatic second lookup |
| AMBIGUOUS_RESULT / IDENTIFIER_MISMATCH / UNKNOWN_STATUS | We couldn't confirm these business details. Check the number or contact us for help. | Edit identifier or support |
| LOOKUP_EXPIRED | Your business lookup has expired. Enter your details manually or check your business again. | Enter details manually (primary); explicit “Check business again” (secondary) |
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

Building a support platform or manual review queue, restoring government records, implementing general traffic rate limits or bot challenges beyond the specified lookup-attempt budget, and promising a registration publication SLA.

## Verification / handover

Fixture every catalogue row, check copy and action, complete manual entry and save after recoverable failures, verify explicit adverse/access results cannot be bypassed, verify the second recoverable failure removes retry/recheck without blocking manual entry/save, verify help preserves form values and browser Back/overlay close restores its origin and focus, verify sign-in return-route guards and interrupted-lookup recovery without automatic dispatch, and prove technical errors cannot become NOT_FOUND. See the broader [unhappy-path register](09-unhappy-paths-and-decisions.md).
