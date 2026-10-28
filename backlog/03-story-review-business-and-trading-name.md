---
id: ZBI-FE-03
epic: ZBI-EPIC
type: story
title: Review registered details and optionally provide a trading name
status: interview-draft
priority: P1
owner: Frontend
delivery_scope: core-happy-path
depends_on: [ZBI-FE-02]
contract: 08-shared-contracts.md
schema_version: 1
---

# FE-03: Review registered details and optionally provide a trading name

## Context

Implements the top part of screen 3. The screenshot's “Business Name” is the entity's registered identity, not whichever trading name appears first. The adapter supplies a display-ready entity name and type. Current registered business names provide optional suggestions; legacy ABR trading-name fields are excluded under epic A09. See [source notes](10-source-notes.md).

## Story statement

**As a merchant, I want to review my registered business and optionally type the name I trade under, so that the business information reflects how I operate.**

## Deterministic acceptance criteria

| ID | Given / when | Then |
| --- | --- | --- |
| AC01 | Eligible ABN result arrives | Show the read-only submitted canonical identifier formatted `NN NNN NNN NNN`, “Business Name”, and “Business Type”. Names/types match the adapter result exactly. Never reuse pasted labels, hyphens, or the original raw string in later fields. |
| AC02 | Eligible confirmed ACN result arrives | Show the submitted canonical ACN formatted `NNN NNN NNN` and the resolved ABN formatted `NN NNN NNN NNN` in a labelled summary. Do not silently replace one with the other or call an unconfirmed ASIC number an ACN. |
| AC03 | The entity is an individual | Display the adapter's current legal-name presentation. For a non-individual, use its current main organisation name. Missing name/type follows FE-05 rather than creating editable official fields. |
| AC04 | Zero, one, or multiple registered business names exist | Show an optional free-text Trading Name input, initially blank. While focused, offer current deduplicated names as secondary suggestions that typing filters; never select one automatically, require a match, or show historical names. Zero matches leaves free-text entry available. |
| AC05 | I leave Trading Name empty or enter only whitespace | Store `trading_name: null`, `trading_name_source: none`; I can continue if the other required fields are valid. |
| AC06 | I select a suggestion | Store its text and source `register_selection`. I can clear it or edit it. Editing to a different value changes source to `user_entered`. |
| AC07 | I enter a custom name of 1–200 Unicode code points after trimming | Accept it regardless of register matching or registration age, with source `user_entered`. Unicode letters and punctuation are allowed; no extra lookup occurs. |
| AC08 | Custom name exceeds 200 code points or contains disallowed controls | Show the matching error below, retain the text, and prevent save until corrected. Do not silently truncate or strip the name. |
| AC09 | I select Back to change the identifier | Return to entry with identifier retained. An accepted lookup for a newly submitted identifier clears old details, trading name, revenue, and category, even if the previous form was complete. |
| AC10 | I use keyboard/screen reader | Read-only values remain readable and labelled. Suggestions support keyboard selection, dismissal, and custom text; no mouse-only interaction is required. |

## Succinct feedback

- Helper: “Optional. Enter the name you trade under.” Suggestions are a convenience, not a required choice.
- Overlength: “Use 200 characters or fewer for your trading name.”
- Control characters: “Enter your trading name on one line.”
- Name absent from suggestions: no error; a missing suggestion does not invalidate a custom name.

## Implementation & architectural guardrails

- Preserve official entity identity and merchant-entered trading name as separate fields. Custom text is not marked register-confirmed or AML-verified.
- Suggestions must not trigger paid requests. Sort/deduplicate using the shared contract; render strings as text, never HTML.
- Business Type may look like a dropdown in the reference; read-only behaviour is an explicit assumption pending Product confirmation, preventing client edits to provider-derived identity.
- On returning from a later step without a new lookup, hydrate server-saved answers. Do not erase an existing saved optional name merely because it is absent from current suggestions.
- Record name mode (`none`, `register_selection`, `user_entered`) only in analytics, never the name itself.

## Out of scope / non-goals

Editing register details, checking ownership or legal entitlement to a name, registering a business name, multiple selected trading names, and document comparison.

## Verification / handover

Fixtures cover sole trader, company, no business names, multiple current names, duplicates, historical names, and a custom recently registered name. Verify blank and custom names both progress, limits count Unicode code points, and markup-like text displays literally. No interaction here makes a lookup call.
