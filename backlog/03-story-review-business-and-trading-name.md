---
id: ZBI-FE-03
epic: ZBI-EPIC
type: story
title: Review retrieved details or enter business details manually
status: interview-draft
priority: P1
owner: Frontend
delivery_scope: core-happy-path
depends_on: [ZBI-FE-02]
contract: 08-shared-contracts.md
schema_version: 1
---

# FE-03: Review retrieved details or enter business details manually

## Context

Implements the top part of screen 3. The screenshot's “Business Name” is the entity's registered identity, not whichever trading name appears first. The adapter supplies a display-ready entity name and type. Current registered business names provide optional suggestions; legacy ABR trading-name fields are excluded under epic A09. When lookup cannot supply usable details, the same screen supports manual business name/type entry with the validated ABN/ACN retained. See [source notes](10-source-notes.md).

## Story statement

**As a merchant, I want to review retrieved details or enter them myself when lookup fails, so that I can continue onboarding with my business information.**

## Deterministic acceptance criteria

| ID | Given / when | Then |
| --- | --- | --- |
| AC01 | Eligible ABN result arrives | Show the read-only submitted canonical identifier formatted `NN NNN NNN NNN`, “Business Name”, and “Business Type”. Names/types match the adapter result exactly. Never reuse pasted labels, hyphens, or the original raw string in later fields. |
| AC02 | Eligible confirmed ACN result arrives | Show the submitted canonical ACN formatted `NNN NNN NNN` and the resolved ABN formatted `NN NNN NNN NNN` in a labelled summary. Do not silently replace one with the other or call an unconfirmed ASIC number an ACN. |
| AC03 | The entity is an individual | Display the adapter's current legal-name presentation. For a non-individual, use its current main organisation name. Missing name/type follows FE-05 into an explicitly user-entered form, rather than presenting editable fields as register-confirmed. |
| AC04 | Zero, one, or multiple registered business names exist | Show an optional free-text Trading Name input, initially blank. While focused, offer current deduplicated names as secondary suggestions that typing filters; never select one automatically, require a match, or show historical names. Zero matches leaves free-text entry available. |
| AC05 | I leave Trading Name empty or enter only whitespace | Store `trading_name: null`, `trading_name_source: none`; I can continue if the other required fields are valid. |
| AC06 | I select a suggestion | Store its text and source `register_selection`. I can clear it or edit it. Editing to a different value changes source to `user_entered`. |
| AC07 | I enter a custom name of 1–200 Unicode code points after trimming | Accept it regardless of register matching or registration age, with source `user_entered`. Unicode letters and punctuation are allowed; no extra lookup occurs. |
| AC08 | Custom name exceeds 200 code points or contains disallowed controls | Show the matching error below, retain the text, and prevent save until corrected. Do not silently truncate or strip the name. |
| AC09 | I select form Back or browser Back to change the identifier | Return to the identifier URL in one step with identifier retained and editable. Submitting a different identifier clears old details, trading name, revenue, and category before either a new lookup or manual entry, even if the previous form was complete. Back with the same identifier retains manual answers; retrying alone does not erase them. |
| AC10 | I use keyboard/screen reader | Read-only values remain readable and labelled. Suggestions support keyboard selection, dismissal, and custom text; no mouse-only interaction is required. |
| AC11 | I choose “Enter details manually” after a recoverable lookup failure | Open the details form immediately, retain the validated identifier as a labelled summary, and focus “Business Name”. Show required editable “Business Name” and “Business Type”, plus optional Trading Name and FE-04’s revenue/category fields. Explain “Enter your registered business details. We’ll verify them later.” No lookup or lookup reference is required. |
| AC12 | I enter manual business identity | Require a business name of 1–200 Unicode code points after trimming, with no controls or line breaks, and a type from the same approved supported-type catalogue. Preserve punctuation/casing and render as text. Do not invent an ABN for an ACN-only submission, register suggestions, active status, or provider provenance. Store identity source `user_entered` and verification status `unverified`. |
| AC13 | A late lookup result arrives after manual entry begins | Keep all typed values and manual mode unchanged. If I deliberately retry from Back, preserve same-identifier manual answers; if that retry succeeds, show the returned identity for review and require explicit “Use retrieved details” before replacing manual identity. Reconcile trading name/revenue/category only for the same submitted identifier; otherwise clear them. Never silently switch identity source. |
| AC14 | I reach retrieved details, choose manual entry, or use browser Forward | Retrieved/manual modes share the details-step URL; mode changes and edits do not add history entries. Forward restores matching same-session answers without a lookup; mismatched/unavailable state follows the shared identifier-entry fallback. Reload/direct entry uses owned server-saved state, with no guarantee for unsaved edits across reload. |

## Succinct feedback

- Helper: “Optional. Enter the name you trade under.” Suggestions are a convenience, not a required choice.
- Overlength: “Use 200 characters or fewer for your trading name.”
- Control characters: “Enter your trading name on one line.”
- Name absent from suggestions: no error; a missing suggestion does not invalidate a custom name.
- Missing manual name: “Enter your registered business name.”
- Manual name overlength: “Use 200 characters or fewer for your business name.”
- Manual name controls: “Enter your business name on one line.”
- Missing manual type: “Select your business type.”

## Implementation & architectural guardrails

- Preserve register-derived identity, user-entered identity, and merchant-entered trading name with explicit sources. Manual details are unverified; saving them does not establish registration, eligibility, or AML approval.
- Suggestions must not trigger paid requests. Sort/deduplicate using the shared contract; render strings as text, never HTML.
- Retrieved name/type remain read-only. Manual mode uses an editable name and an approved supported-type select; catalogue IDs/labels are an integration dependency, not invented production values.
- Follow [URLs and browser navigation](08-shared-contracts.md#urls-and-browser-navigation) for Back/Forward, direct links, same-session state, focus, and identifier matching. On returning from a later step without a new lookup, hydrate server-saved answers. Do not erase an existing saved optional name merely because it is absent from current suggestions.
- Record name mode (`none`, `register_selection`, `user_entered`) only in analytics, never the name itself.

## Out of scope / non-goals

Editing register details, checking ownership or legal entitlement to a name, registering a business name, multiple selected trading names, and document comparison.

## Verification / handover

Fixtures cover sole trader, company, no business names, multiple current names, duplicates, historical names, and a custom recently registered name. Verify blank and custom names both progress, limits count Unicode code points, and markup-like text displays literally. No interaction here makes a lookup call. Verify manual entry and save after timeout/offline/no match/incomplete responses, name/type validation, ACN-only identity, source/status persistence on restore, same-identifier retry review, and late-result protection. Test one browser Back to editable ABN/ACN from retrieved/manual forms, Forward with retained same-session answers, changed-identifier guards, no extra history entries for mode changes, and no lookup on refresh/direct entry.
