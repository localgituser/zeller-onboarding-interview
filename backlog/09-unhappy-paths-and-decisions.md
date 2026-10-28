---
id: ZBI-RISKS
type: risk-and-decision-register
status: interview-draft
updated: 2026-10-28
---

# Unhappy paths and decisions to discuss

This is the list requested by the interview brief, extended with the user's operational considerations. “Core” means handled in the happy-path frontend work; “supporting” means explicitly specified outside that narrow handover; “deferred” means no implementation is claimed. The common [error catalogue](05-story-errors-and-help.md#error-catalogue) owns customer copy.

| Scenario | Deterministic behaviour / user recovery | Scope / owner |
| --- | --- | --- |
| Empty, wrong length, unsupported character, all zeros | Stop before lookup; focus input with format/required feedback | Core FE-01; server defence BE-01 |
| Correct length, wrong checksum | Stop before lookup; identify ABN/ACN as incorrect | Core FE-01; server defence BE-01 |
| Raw typing, labelled/formatted paste, or leading-zero ACN | Auto-space display; normalise supported labels/whitespace/hyphens identically client/server; canonical digits/type everywhere downstream | Core FE-01 / BE-01 |
| Conflicting prefix, extra words, multiple labelled numbers, overflow, or unsupported characters | Retain input for correction; reject before lookup; no arbitrary extraction/truncation | Core FE-01 / BE-01 |
| Valid identifier, no register match | No match feedback; primary manual entry with identifier retained; optional help/edit/retry | Supporting FE-05 |
| New registration / future activation date / recent change | Explain potential delay and check activation; preserve identifier; no automatic retries | Supporting help; A13 timing unconfirmed |
| Valid ACN but no listed ABN | Explain register coverage; try an ABN if available or contact support | Supporting FE-05 |
| ASIC method returns non-ACN or unconfirmed registration kind | UNSUPPORTED_IDENTIFIER_TYPE; confirmed non-ACN requires ABN/support, unclassified kind offers manual entry with unverified provenance; no automatic second lookup | Supporting BE-01 / FE-05; mapping release decision |
| Multiple registered business names | Blank optional text field with secondary suggestions; merchant may type any valid name | Core FE-03 |
| Missing name suggestion or name registered in last 24 hours | Accept custom name or blank; no matching requirement or extra lookup | Core FE-03 |
| Only legacy trading names returned | Exclude from current suggestions; manual optional field still works | Core FE-03 / BE-01 |
| Long, Unicode, punctuation, markup-like names | Respect explicit limit; render literal text; do not silently truncate | Core FE-03 / FE-04 |
| Suppressed record / missing official name or type | Offer manual name/type entry with unverified provenance; do not invent register fields or infer from other users | Supporting FE-05 / BE-01 |
| Inactive/cancelled, unknown status, unsupported type | Stop with mapped feedback; Product/Risk confirms eligibility policy | Supporting; release decision |
| Replaced/reissued identifier or multiple historical records | Use current semantics; do not silently follow replacement or select arbitrary historical identity | Supporting BE-01 / FE-05 |
| Mismatched identifier or conflicting current response | Stop and offer correction/support; no accidental association | Supporting BE-01 / FE-05 |
| Offline before submission | Feedback and manual entry without dispatch; save after connection returns | Supporting FE-05 |
| Connection lost after dispatch, slow provider, timeout | Stop lookup at 3 seconds (or immediately on failure); primary manual entry, at most one explicit retry after the initial lookup; prior call may still cost money and must be logged | Core loading + supporting recovery/ledger |
| HTTP 200 with application error | Parse payload; map explicit outcome; do not present success | Supporting BE-01 |
| Invalid credentials, HTTP failure, malformed/changed schema | Manual-entry fallback and internal alert; never “not found” | Supporting BE-01 / FE-05 |
| Double click, Enter twice, refresh, transport re-delivery | Pending suppression and server idempotency; do not double-charge same operation | Core FE-02 / supporting BE-01 |
| Browser Back/Forward, refresh, or direct details link | Entry/details have distinct URLs; one Back from details to editable identifier; restore matching owned state or replace with entry, never dispatch lookup/save automatically; no budget reset | Core FE-01–FE-05 / shared navigation |
| Browser Back during save; save later commits | Leave normally, invalidate response handler, no late redirect; read/reconcile saved revision on return, explicit same-key retry if unresolved | Core FE-04 / BE-01 |
| Help or sign-in detour | Overlay/full-page Back returns to origin; sign-in restores allowed route and owned saved state or falls back to identifier entry; no automatic lookup/save | Supporting FE-05 |
| Back during request; late A after newer B | Invalidate A for UI; preserve call accounting; B cannot be overwritten | Core FE-02 |
| Concurrent same-key lookups by several users | Shared call and cache snapshot; separate user-bound results and request records | Supporting BE-01 |
| Fresh cache, exact expiry, register changes within TTL | Fresh hit free; re-fetch at expiry; accept documented staleness trade-off | Supporting BE-01 |
| NOT_FOUND cache hides newly registered business | Do not cache negative results; explain unique retry spend risk | Supporting BE-01; abuse controls deferred |
| Cache/concurrency service failure | Fail safely if duplicate prevention cannot be guaranteed | Supporting BE-01 |
| Ledger failure or crash around dispatch | Do not call without durable intent; reconcile explicitly uncertain dispatch | Supporting BE-01 |
| Missing revenue/category, stale catalogue | Inline feedback, reload choices, retain other fields | Core FE-04 |
| Save rejected, response lost, version conflict | Retain form; idempotent retry or reload newer draft; no silent overwrite | Core FE-04 / supporting FE-05 |
| Committed-save replay after lookup expiry or newer revision | Replay original outcome before new-mutation checks; read latest draft; one backend completion | Core FE-04 / BE-01 / DATA-01 |
| Save still pending on retry | SAVE_IN_PROGRESS; explicit same-key retry; no duplicate mutation/polling | Supporting FE-05 / BE-01 |
| Saved lookup expired before changed save | Offer reviewed manual entry or explicit recheck; never silently initiate another paid call | Supporting FE-05 / BE-01 |
| Session expires or wrong draft/lookup reference | Existing sign-in recovery or generic access feedback; no unauthorised lookup/save | Supporting FE-05 / BE-01 |
| Analytics outage, blocked telemetry, abrupt browser exit | Keep onboarding usable; flag missing timing/coverage; ledger remains authoritative | Supporting DATA-01 |
| Missing visit/app context or retry from a new visit | Preserve initiating context, otherwise explicit null/status; server derives owned attempt; telemetry never changes save identity | Supporting DATA-01 / shared contract |
| Bot or scripted unique-identifier lookups by logged-in account | Authentication/caching alone do not cap spend; assess quotas, limits, bot checks and budget alerts before rollout | Deferred by requested scope; Security/Product |
| Second recoverable failure / third lookup request | Remove lookup retry/recheck; offer manual entry and help. Backend rejects a third new operation without cache/provider access; refresh/Back/other tabs cannot reset the per-identifier onboarding-attempt budget | FE-02 / FE-05 / BE-01 |
| Late lookup result after manual entry | Preserve typed details/source; ledger/cache may complete independently; no silent replacement | Core FE-02 / FE-03 |
| Manual identity save or restore | No lookup reference or provider call; validate name/type/identifier, persist user-entered/unverified provenance; known adverse register results keep specific recovery | Core FE-03 / FE-04 / BE-01 |
| Later certificate differs from captured register/custom name | Preserve provenance now; later AML workflow determines evidence and review | Deferred AML story; Risk/Product |

## Decisions required before integrated production delivery

| Decision | Proposed position for this exercise | Accountable partner |
| --- | --- | --- |
| Actual lookup provider and commercial model | ABR-compatible adapter; $0.60 per attempted outbound call until billing confirmed | Engineering / Finance |
| Eligible types and ACN classification | Active ABN plus identity and approved supported-type allowlist; separate verified company-registration mapping for ACN; manual unverified entry for unresolved kind; ABN/support for confirmed non-ACN | Product / Risk / Engineering |
| Revenue/category option catalogue and navigation routes | Reuse existing service IDs; distinct identifier/details URLs with shared browser-history contract; next-step/help/sign-in paths follow existing shell conventions | Product / Engineering |
| Trading-name UI | Optional free text first; nonbinding current registered-name suggestions | Product / Design; incorporates user clarification |
| Propagation advice | No fixed promise; 24–48-business-hour estimate needs provider evidence | Content / Product |
| Cache sharing, 24-hour staleness and negative results | Share only normalised public data, retain per-user references; no negative cache | Engineering / Privacy / Product |
| Timeout/retry/concurrency defaults | 2.5-second total backend / 3-second UI deadline; manual entry as primary recovery; initial plus one retry per identifier/type per onboarding attempt, durable atomic enforcement, same-key replay and single-flight | Engineering |
| Ledger and telemetry retention / access | Existing approved policies; minimum fields, restricted access | Engineering / Data / Privacy |
| No-ABN branch in existing shell | Do not implement or change its behaviour in these tickets; test only selected ABN/ACN branch | Product / Frontend |
| Support/help destinations and ownership | Existing support entry plus versioned in-app lookup-help content | Support / Content |
| Abuse controls for paid traffic | General traffic quotas/bot controls excluded; the two-attempt per-identifier lookup budget is included. Make a rollout risk decision before external release | Security / Product |
| Later registration evidence and AI matching | Separate workflow with entity-specific evidence and appropriate review | Risk / Product |

## Interview trade-offs to explain

- Free local checksum validation and deliberate submit prevent avoidable calls; server validation protects against client bypass.
- Paste flexibility and live spacing improve entry while one canonical value prevents inconsistent cache keys/later fields. Unknown text/conflicting labels fail safely instead of silently changing identity.
- A 24-hour positive cache reduces repeated spend, but can display recently changed details. An explicit cache age/provenance record supports later comparison.
- No negative cache avoids locking new registrations out for another day. The initial lookup and one retry may still be chargeable; the per-identifier attempt budget bounds these within an onboarding attempt, while broader abuse controls remain separate.
- A 3-second cap keeps lookup a convenience. Manual entry prevents a provider outage from blocking information capture; provenance stays user-entered/unverified for later checks.
- Custom optional names reduce friction; they remain merchant assertions, separate from official identity and later AML checks.
- Frontend tickets start against fixtures. The backend and analytics stories expose delivery dependencies without expanding the core frontend handover into an entire onboarding redesign.
