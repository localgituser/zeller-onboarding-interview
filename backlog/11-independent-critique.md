---
id: ZBI-REVIEW
type: independent-critique
reviewed: 2026-10-28
status: findings-addressed-not-independently-rereviewed
review_scope: original-12-markdown-files
---

# Independent critique of the interview backlog

## Fixes applied after review — 28 October 2026

The original assessment below describes the pre-fix pack. The primary agent applied these changes at the user's request; this is a resolution record, not a new independent approval. Historical line labels refer to the original version; file links now open the revised appendices without stale line targets.

| Finding | Applied change / evidence |
| --- | --- |
| 1: Completion ownership | [FE-04](04-story-save-business-information.md) and [DATA-01](07-story-onboarding-analytics.md) specify backend-only logical completion and lost-response/replay checks. |
| 2: Backend event context | [Shared request context](08-shared-contracts.md#request-context-for-backend-analytics) defines optional visit/app transport, nullable missing/invalid values, original-operation retention, and trusted server attempt association. |
| 3: Save replay ordering | [Save ordering](08-shared-contracts.md#save-idempotency-ordering) replays committed outcomes before new TTL/catalogue/revision checks; pending operations cannot duplicate a mutation. |
| 4: Broad ASIC method | [Adapter rules](08-shared-contracts.md#adapter-rules-and-provenance) distinguish ACN classification from supported-type policy and stop non-ACN/unconfirmed results with an ABN/support fallback. Epic A10 now uses supported-type wording. |
| 5: Presentation size | [Short walkthrough](INTERVIEW-WALKTHROUGH.md) is the interview view; [README](README.md) indexes the preserved detailed appendices. |

Additional requested UX changes: FE-01/shared contract cover live spacing, labelled and separated pastes, canonical identity reuse, client/server parity, caret handling, and useful placeholders. The reference fixtures are checked separately; no application implementation or live API behaviour is claimed.

## Verdict

The backlog meets the exercise's main behavioural requirements and incorporates the user's additions carefully. The ABR research is broadly accurate: the selected methods, distinction between entity and business names, optional/suppressed fields, and application-error handling agree with the official guides. There is no fundamental API misunderstanding that requires rewriting the happy path.

I found no definite behavioural conflict that blocks the core happy path. Before handing the tickets to an engineer, clarify completion wording, supply server-event context, and make save-replay ordering explicit. Before the interview, make the presentation substantially lighter than the reference pack. The original 12 files contain approximately 10,553 words; the four core tickets, epic, and shared contract alone contain about 5,505. The extra detail is justified by the user's broader request, but it should support the discussion rather than become the discussion.

## Exercise criteria scorecard

| Criterion | Assessment | Evidence / qualification |
| --- | --- | --- |
| Cover the three illustrated frontend states | Meets | FE-01 covers entry, FE-02 loading, FE-03 returned details. FE-04 adds the remaining visible fields and explicitly assumes the cropped final action. |
| Ignore the no-ABN branch | Meets | It is consistently excluded; the tickets do not attempt to implement it. |
| Engineer-readable tickets ready to start | Mostly meets | Clear behaviours, ownership, scope, and fixture expectations. Real integration remains conditional on catalogues, navigation, eligibility, and endpoint agreement, as the epic acknowledges. |
| Make assumptions clear | Strong | A01–A14 distinguish exercise inputs, proposals, and decisions needing confirmation. |
| List unhappy paths | Strong | The register is comprehensive and the error catalogue is actionable. It exceeds the brief's minimum without claiming all paths are core happy-path work. |
| Control avoidable lookup cost | Strong | Client/server preflight, deliberate submission, positive cache, duplicate suppression, and a separate operational ledger are specified. |
| Match current ABR documentation | Meets with boundary clarification | Method versions and major response constraints are correct. Clarify how the broader ASIC-number method is restricted to the intended ACN journey. |
| Measure funnel and step duration | Mostly meets | Stage timing, server-confirmed completion, and telemetry limitations are well considered; event ownership/context need the fixes below. |
| Fit a 30-minute exercise / 15-minute walkthrough | Presentation risk | The README provides a route, but the amount of material and repeated rules could obscure prioritisation. Keep the comprehensive pack as reference. |
| Explain AI use and candidate judgment | Meets | Source notes record actual changes and identify proposals for candidate review, rather than claiming the candidate has already endorsed every AI decision. |

## Findings to fix before engineering handover

### 1. Low: completion emission wording is ambiguous

[FE-04 line 30](04-story-save-business-information.md) says to emit completion after acknowledgement inside a frontend story. Its [guardrail at line 43](04-story-save-business-information.md), and [DATA-01's event definition at line 54](07-story-onboarding-analytics.md), explicitly assign completion to the server. Reading the documents together resolves the intended owner; this is a wording clarification, not proof of a duplicate-emission requirement. An engineer following the isolated acceptance criterion could nevertheless add an unnecessary frontend emission.

**Recommended change:** make the server the sole producer of `business_info.completed`. FE-04 should navigate after acknowledgement and, only if useful, emit a separately named continuation event. Add one acceptance test where the save commits, the response is lost, and a replay produces exactly one completion.

### 2. Medium: server analytics require context the API contract never carries

[DATA-01 line 27](07-story-onboarding-analytics.md) requires every event to contain `screen_visit_id` and app version, including backend-produced lookup and completion events. However, the [lookup request at line 39](08-shared-contracts.md) and [save request at line 109](08-shared-contracts.md) do not carry that context or define a shared header/enrichment mechanism. The save request also omits `onboarding_attempt_id`; it may be derivable through the lookup/draft, but that is not stated.

**Recommended change:** define a small analytics-context envelope or existing request headers containing the client visit/app fields, plus how the server resolves the onboarding attempt. Keep user/draft ownership server-verified. Alternatively, define a common server-event schema with explicit optional client fields. Do not invent or silently omit required fields when completion occurs after the browser leaves.

### 3. Low: explicitly order save replay before freshness and revision checks

[FE-04 line 32](04-story-save-business-information.md) already promises the correct original save outcome on re-delivery. [The shared save rules at line 126](08-shared-contracts.md) also require an unexpired lookup and matching revision without specifying the order relative to replay resolution. This is an implementation-order clarification and missing boundary fixture, not a demonstrated contradiction of the promised outcome.

Example: a save commits just before lookup expiry, its response is lost, and the merchant retries just after expiry. Revalidating the lookup or old expected revision first could incorrectly return LOOKUP_EXPIRED or DRAFT_CONFLICT for a save that already succeeded.

**Recommended change:** after authentication/ownership and idempotency-payload comparison, replay an existing committed outcome before validating freshness, catalogue state, or revision for a new mutation. Define what happens while the original operation is still pending. Add the expiry-crossing and changed-revision replay fixtures. The intended behaviour is already present; this makes its ordering unambiguous.

## API and product boundaries to clarify

### 4. Medium: the ASIC-number endpoint is broader than ACNs

[The adapter rule at line 86](08-shared-contracts.md) confirms that the returned ASIC number matches the input, but matching nine digits does not by itself establish that the identifier is an ACN. The documented method can also search ARBN, ARSN, and ARFN identifiers. This is an API boundary, not a demonstrated defect in the illustrated ACN happy path. [Official method guide](https://abr.business.gov.au/Documentation/WebServiceMethods), [identifier definitions](https://abr.business.gov.au/FAQ/Names).

The existing supported-type policy can resolve this, but that policy is still undefined. [Epic A10](00-epic-business-information.md) uses “recognised” type, while [the contract at line 87](08-shared-contracts.md) additionally requires a “supported” type. Those are different tests.

**Recommended change:** make A10 use the same supported-type rule and label the allowlist as an integration decision. Add a fixture where the ASIC method returns a non-ACN registration and specify the intended fallback, rather than labelling every successful nine-digit response as an ACN. Do not treat the entire ABR entity-type catalogue as Zeller's eligibility policy.

## Interview presentation refinement

### 5. Low technical severity, high interview value: reduce the material on the critical reading path

[The epic requires agents to read the shared contract first](00-epic-business-information.md), and [the walkthrough introduces seven separate documents](README.md). This is thorough, but the preparation limit and frontend focus call for stronger visible prioritisation. Detailed billing reconciliation, concurrency ownership, HTTP mappings, catalogue versions, and telemetry schemas can crowd out the merchant journey if all receive equal airtime.

**Recommended change:** keep the requested comprehensive files, but provide a short interview view: one screen of goal/assumptions, four frontend tickets with the key acceptance criteria, a compact unhappy-path list, and one paragraph on backend/data dependencies. Link deeper details as appendices. Clearly identify what is needed to build the illustrated flow versus production hardening or a proposed integration default. This is a presentation improvement, not a reason to remove the user's requested backend and measurement work.

## What to preserve

- Both frontend and backend validate format and checksums before provider access; identifiers remain strings and preserve leading zeros. The ABN algorithm matches [ABR's specification](https://abr.business.gov.au/Help/AbnFormat), and the ACN algorithm matches [ASIC Appendix D](https://download.asic.gov.au/media/nvnh1kjr/msg_spec.pdf).
- Trading Name is optional free text first, with nonbinding current registered-business-name suggestions. Legacy trading names are correctly excluded; ABR advises against relying on them. [Names guidance](https://abr.business.gov.au/FAQ/Names).
- The normalised adapter is explicitly proposed, rather than misrepresented as ABR's native JSON API. The selected current methods and history setting are appropriate. [Method guide](https://abr.business.gov.au/Documentation/WebServiceMethods).
- Missing name/type and suppressed records are treated deliberately. ABR makes almost all entity attributes optional, so the proposed safe-stop behaviour is a product assumption rather than a claim the API always supplies these fields. [Response guide](https://abr.business.gov.au/Documentation/WebServiceResponse), [data dictionary](https://abr.business.gov.au/Documentation/DataDictionary).
- HTTP failures and application exceptions are separated; technical failures do not become “not found”. [Exception guide](https://abr.business.gov.au/Documentation/Exceptions).
- The 24-hour TTL has an exact expiry boundary, does not slide on reads, and preserves retrieval provenance. Not caching NOT_FOUND is an explicit trade-off for newly registered businesses. The implementation still needs the acknowledged provider/cache integration decisions; there is no inherent contradiction between sharing public snapshots and issuing private per-user lookup references.
- The $0.60 exercise assumption remains intact while free direct ABR access is distinguished accurately. Fixed 24–48-business-hour publication claims are avoided. [Registration guide](https://abr.business.gov.au/Documentation/WebServiceRegistration), [ABR missing-record guidance](https://www.abr.gov.au/general-information/scheduled-site-maintenance).
- Later document/AI checks, rate limiting, bot challenges, and payment readiness are correctly kept outside this step. A successful lookup is not represented as AML clearance.

## Recommended revision order

1. Resolve completion ownership and define the server analytics context.
2. Specify save replay ordering and the two boundary fixtures.
3. Align recognised/supported type language and add the non-ACN ASIC-result case.
4. Prepare the short interview view, retaining detailed appendices for questions.

## Review method and limits

Reviewed all original 12 Markdown files; extracted and visually inspected both pages of the supplied brief; rechecked official methods, response, exceptions, dictionary, registration, name/coverage guidance, and checksum specifications. No authenticated provider lookup or charged request was made. Operation-specific test-form URLs were unavailable through the browser tool, so no claim is made that the proposed normalised contract has been validated against live SOAP responses. No existing backlog file was changed during this critique.

Exercise source: [Product Manager task](../inputs/Zeller%20Product%20Manager%20Task.pdf).
