---
id: ZBI-FE-04
epic: ZBI-EPIC
type: story
title: Complete revenue and category and save before continuing
status: interview-draft
priority: P1
owner: Frontend
delivery_scope: core-happy-path
depends_on: [ZBI-FE-03, existing-option-catalogues, existing-draft-save]
contract: 08-shared-contracts.md
schema_version: 1
---

# FE-04: Complete revenue and category and save before continuing

## Context

Completes screen 3's Expected Annual Business Revenue and Business Category fields. A Next action below these fields is assumed because the screenshot is cropped. The brief does not supply option values or the following step; use existing catalogues and navigation, confirmed before integration. See epic A11–A12.

## Story statement

**As a merchant, I want to add my expected annual revenue and business category and save my information, so that I can continue onboarding with accurate business details.**

## Deterministic acceptance criteria

1. **Initial state:** Show labelled revenue and category selects with no preselected answer, plus optional Trading Name, Back, and Next. Render approved catalogue labels; store stable option IDs and catalogue version, not labels as identity.
2. **Required answers:** On Next, missing revenue shows “Select your expected annual business revenue.” Missing category shows “Select your business category.” Show both when applicable, focus the first invalid field, and make zero save calls.
3. **Valid save:** With a successful current lookup, valid optional trading name, and both valid option IDs, submit one save operation containing the shared fields and available `client_context`. Enter `save_pending`, announce “Saving your business information…”, and disable submission and form navigation until completion or the 10-second UI deadline.
4. **Success:** Advance only after server acknowledgement of the saved draft revision. The backend alone emits `business_info.completed`, once per `completion_id`; frontend acknowledgement/navigation does not emit it again. A Next click alone is not completion and cannot make a new provider lookup.
5. **Failure/deadline:** Retain every displayed answer, stop pending state, show “We couldn't save your details. Please try again.”, and allow Retry. Do not navigate on failure or a late response from an invalidated UI operation.
6. **Duplicate/uncertain outcome:** Re-delivery after timeout uses the same save operation ID and business payload. After authentication/ownership and payload comparison, backend replays a committed outcome before checking new-mutation lookup expiry, catalogue validity, or revision. This includes an expired lookup or a revision since advanced by another save. If the operation is still pending, return SAVE_IN_PROGRESS without a second mutation; show “Your details are still being saved. Please try again shortly.” and allow explicit same-key retry. Changed business answers use a new operation ID. Late earlier responses cannot overwrite edits. Client telemetry context is excluded from business-payload equality.
7. **Restore:** Back from the next step or a reload fetches the server-saved draft without a lookup call. Unsaved edits may be lost on reload. A changed identifier invalidates earlier business answers as defined in FE-03.
8. **Server validation:** If a catalogue selection is no longer valid, preserve other answers, reload the catalogue, and show “This option is no longer available. Please choose another.” If a lookup has expired before save, retain the form and use FE-05's recheck path; never silently buy another lookup.
9. **Accessibility:** Both selects have accessible names and keyboard operation; errors are associated with fields and announced. Pending state has visible text as well as animation.

## Implementation & architectural guardrails

- Save a server-owned `lookup_id` reference, not pasted identifier strings or editable legal identity. Backend checks ownership and resolves existing idempotent outcomes first; only new mutations validate lookup freshness, allowed option IDs, and revision. A replay of an older committed revision confirms that operation, then reloads current draft state without restoring older values.
- Reuse the existing save service if it meets the shared semantics. The proposed route names are illustrative and must align with the host application.
- Never overwrite a newer draft silently after a multi-tab conflict. Never send save requests while validation errors remain.
- Build against test catalogues (`revenue_test_1`, `category_test_1`) until approved production catalogues are supplied; those IDs must never ship as real choices.
- Server emits completion; frontend emits selection, save intent/failure, and navigation events through DATA-01. Analytics delivery must not block the form.

## Out of scope / non-goals

Defining Zeller's revenue bands or industry taxonomy, collecting actual financial statements, payments activation, next-step implementation, autosaving unsaved edits, and AML approval.

## Verification / handover

Verify required-field errors, optional name, stale options, duplicate Next, rejection, reload, and conflicts. Commit a save with a lost response; retry after lookup expiry and after draft revision advances: return the original outcome, reload current draft, emit exactly one server completion, and make no provider call. A pending replay returns SAVE_IN_PROGRESS without a second mutation. Test missing/changed client context never duplicates a committed business operation.
