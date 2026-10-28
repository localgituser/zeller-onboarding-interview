---
id: ZBI-FE-01
epic: ZBI-EPIC
type: story
title: Enter and validate an ABN or ACN before lookup
status: interview-draft
priority: P1
owner: Frontend
delivery_scope: core-happy-path
depends_on: [existing-onboarding-shell]
contract: 08-shared-contracts.md
schema_version: 1
---

# FE-01: Enter and validate an ABN or ACN

## Context

Implements screen 1 on page 2 of the brief. The merchant is authenticated and the current draft belongs to them. Free local validation prevents avoidable paid lookups. Read [epic assumptions A01–A04](00-epic-business-information.md) and [validation rules](08-shared-contracts.md#identifier-validation).

## Story statement

**As a merchant, I want to enter my ABN or ACN and correct simple mistakes immediately, so that I can find my business without unnecessary delays.**

## Deterministic acceptance criteria

| ID | Given / when | Then |
| --- | --- | --- |
| AC01 | I enter the step | Show “Your business information”, the existing explanatory text, selected “I have an ABN or ACN” option, labelled “ABN or ACN” input, clear control, Back and Next. The no-ABN branch is excluded from this delivery. |
| AC02 | I type raw digits or paste a labelled/formatted identifier | Apply the shared normalisation rules before validation or downstream use. Accept a single leading `ABN`/`ACN` label (case-insensitive, optional colon), spaces and supported hyphens; retain leading zeros. Keep canonical digits separate from the spaced display. Use a numeric keyboard hint on a text input, while permitting clipboard labels. |
| AC03 | I select Next or press Enter with empty/invalid input | Show the matching error below, focus the input, remain on this screen, and make zero lookup requests. |
| AC04 | I submit a valid 11-digit ABN or 9-digit ACN | Emit one valid submission intent with normalised identifier and detected type to FE-02; validation alone does not assert registration exists. |
| AC05 | I edit after an error or activate clear | Remove the stale error; clear empties and focuses the input. No lookup occurs on typing, paste, blur, or clear. |
| AC06 | I select Back | Return to the existing previous step with the identifier retained in onboarding memory. No lookup occurs. |
| AC07 | I use a keyboard or screen reader | Label, clear control, buttons, and inline error are accessible; error is linked to the input and announced. At 320 CSS pixels wide there is no horizontal form scrolling. |
| AC08 | I enter `12345678901`, or paste `ABN: 12-345-678-901` | Display `12 345 678 901` and normalise to `12345678901`. Formatting is independent of checksum validity; this example still fails checksum on submission and makes zero requests. |
| AC09 | I paste `acn: 004-085-616` | Display `004 085 616`, retain type ACN and canonical `004085616`, and pass its checksum. No lookup occurs until Next/Enter. |
| AC10 | I insert/delete digits in the middle, replace a selection, undo/redo, or backspace beside a generated space | Preserve logical caret/selection by digit position; deleting beside a separator deletes the intended adjacent digit. Do not jump to the end, trap deletion, lose digits, or trigger lookup. Defer formatting during input-method composition. |
| AC11 | The identifier field is empty | Show the persistent label “ABN or ACN”, placeholder “e.g. 12 345 678 901”, and visible helper “ABN: 11 digits. ACN: 9 digits. You can paste labels, spaces or hyphens.” If an ACN type hint is retained, use placeholder “e.g. 123 456 789”. Placeholder examples are formatting guidance, never default values. |
| AC12 | A label conflicts with digit length or declared request type, or the paste includes extra words/multiple identifiers | Keep an actionable error and make zero requests. Never extract an arbitrary valid-looking substring, truncate overflow, or silently relabel an ACN as an ABN. |

## Validation and succinct feedback

| Condition | Exact customer copy |
| --- | --- |
| Empty after normalisation | Enter your ABN or ACN. |
| Unsupported characters, incorrect unlabelled length, or all-zero identifier | Enter an 11-digit ABN or 9-digit ACN. |
| ABN-labelled input has wrong length | An ABN has 11 digits. Check the label or number. |
| ACN-labelled input has wrong length | An ACN has 9 digits. Check the label or number. |
| Prefix and declared request type disagree | The label and number type don't match. Check and try again. |
| ABN checksum failure | This ABN looks incorrect. Please check and try again. |
| ACN checksum failure | This ACN looks incorrect. Please check and try again. |

Next remains enabled in the idle state so submission can reveal a clear error; it is disabled while a lookup is pending. Do not show errors before the first submission. Error text and analytics reason codes are separate.

## Implementation & architectural guardrails

- Reuse existing field/button and responsive layout components. Do not reproduce the screenshot's light-grey text at the expense of readability.
- Client and backend run the same versioned normalisation before identifier validation, cache keys, idempotency comparison, provider access, or persistence. Use canonical digits/type in later fields and payloads; format on display only. Never convert to a number or derive an ABN from an ACN.
- Do not set a raw 11-character input limit: labelled/spaced pastes are longer. Keep overflow visible for correction. Reject unsupported text without silently stripping it. The exact partial-entry spacing and caret rules live in the shared contract.
- Keep `user_id` from trusted authentication context; do not ask the user to enter it.
- Record stage entry, failed validation, valid submit, clear, and Back through DATA-01. Never send input contents in analytics.

## Out of scope / non-goals

Provider access, cache storage, business-name search, no-ABN flow, and any assertion that a checksum-valid number belongs to the merchant.

## Verification / handover

Use the shared normalisation/formatting fixtures, including prefixes, Unicode whitespace/common hyphens, leading zeros, conflicting labels, malformed pastes, and partial entry. Verify invalid characters, overflow, and checksum failures produce zero network lookups; one explicit valid submission produces one lookup intent. Test mid-string edits, separator deletion, selection paste, undo/redo, accessible placeholder/helper text, keyboard submission, and error announcement. Compare client/server outputs for every fixture.
