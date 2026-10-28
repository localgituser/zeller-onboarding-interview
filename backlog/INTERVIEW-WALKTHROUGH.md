---
id: ZBI-INTERVIEW
type: interview-walkthrough
status: interview-draft
updated: 2026-10-28
detail_index: README.md
---

# Business information: interview walkthrough

Use this document for the 15-minute discussion. The linked appendices contain the full engineer/AI-readable tickets, contracts, evidence, and test cases.

## 1. Goal and assumptions — 3 minutes

Help a logged-in merchant retrieve their business from an ABN or ACN, confirm the identity, and complete the remaining fields quickly. This contributes to Zeller's **two-minute sign-up-to-payment-ready goal**; saving this step does not itself make the merchant payment-ready.

- Reuse existing design elements - authentication, navigation, option catalogues, and accessible components; ignore the no-ABN branch.
- Assume an Australian Business Registry (ABR)-compatible backend adapter and **AUD $0.60 per outbound attempt**, as the exercise states. Direct ABR access is free; confirm the actual provider's billing. [Source](https://abr.business.gov.au/Documentation/WebServiceRegistration).
- Normalise client/server identically before validation/use. Auto-space digits; accept labelled/hyphenated pastes. Check length and checksum locally before spending money.
- Cache valid entity responses for **24 hours**; do not cache not-found/errors, protecting recent registrations.
- Trading Name is optional text with secondary registered-business-name suggestions. Legacy ABR trading names are historical. [Source](https://abr.business.gov.au/FAQ/Names).
- Retrieved name/type are read-only; after recoverable lookup failure, users can enter name/type manually with unverified provenance. An active ABN and supported-type allowlist are proposed policy, requiring confirmation. Revenue/category are assumed required; options and final navigation remain integration decisions.
- Correlate events with user, temporary business, and onboarding-attempt IDs. Later document/AI checks and bot/rate controls sit outside this happy path.
- The best API for this lookup has already been determined.
- A rollout strategy is out-of-scope for this exercise.

## 2. Four frontend tickets — 7 minutes

### FE-01 — Enter, format, and validate an identifier

**As a merchant, I want to type or paste my ABN/ACN easily, so that I can find my business without avoidable errors.**

- `12345678901` displays `12 345 678 901`; pasted `acn: 004-085-616` displays `004 085 616`. Preserve zeros/caret; formatting never establishes validity or triggers lookup.
- Keep the label “ABN or ACN”, placeholder “e.g. 12 345 678 901”, and helper covering both lengths and accepted pastes. Store canonical digits; later fields format them only for display.
- On Next, normalise, reject malformed/conflicting labels, then check length/checksum. Invalid input shows inline feedback and makes zero requests. Valid input pushes the details URL once and starts one lookup. Invalid input stays at the identifier URL.

[Detailed FE-01](01-story-enter-identifier.md)

### FE-02 — Show lookup progress and prevent duplicates

**As a merchant, I want clear lookup feedback, so that I can wait confidently without submitting again.**

- Show “Looking up your business…”, disable editing/Next, and keep Back available. Repeated clicks create no extra request; re-delivery retains its operation key.
- Loading/results/errors/manual entry share the details history entry. Browser or form Back returns to the identifier URL with the number editable in one action; Forward/refresh never automatically triggers lookup. Display only the active attempt's result. Back or a newer attempt prevents late responses replacing the current screen.
- Cap lookup at **3 seconds**, then show “We couldn’t look up your business. Enter your business details manually to continue.” Offer **Enter details manually** as primary and **Try again** as secondary while budget remains; earlier failures expose fallback immediately. Allow the initial lookup plus **one retry per identifier per onboarding attempt**. After both fail, offer manual entry/help only; server enforcement survives refresh and Back. Technical failure never means “not found”; a cache hit needs no artificial wait.

[Detailed FE-02](02-story-lookup-and-loading.md)

### FE-03 — Review identity and optionally type a trading name

**As a merchant, I want to confirm my business and enter the name I trade under, so that my details reflect how I operate.**

- On success, show official name/type and formatted identifier read-only. A confirmed ACN lookup also shows its ABN; a broader ASIC result is not automatically an ACN.
- Trading Name starts blank. Accept custom text, no name, or a nonbinding current-name suggestion. Never require a match or call the API for name entry.
- After timeout, offline/provider failure, no match, or incomplete details, retain the validated ABN/ACN and open editable Business Name/Business Type with optional Trading Name and revenue/category. No extra lookup; save as user-entered/unverified. Late results cannot overwrite typing. Known inactive/unsupported/ambiguous identities retain correction/support recovery.
- A different business identifier clears earlier business-specific answers. Browser Forward restores only matching same-session details; direct links/reload restore owned saved state or return to identifier entry.

[Detailed FE-03](03-story-review-business-and-trading-name.md)

### FE-04 — Complete remaining fields and save

**As a merchant, I want to supply revenue/category and save, so that I can continue onboarding.**

- Require approved revenue/category choices; blank trading name remains valid. Show field-specific errors and retain answers after failure.
- Save either an owned lookup reference or valid manual identity without a lookup reference or provider call. Submit once, show saving feedback, and advance after server acknowledgement. Backend alone emits step completion. Saving adds no history entry; active success pushes the existing next-step URL once. Browser Back during save can leave; a late response cannot redirect, and return reconciles saved state without automatic resubmission.
- Retry replays a committed result even after lookup expiry/newer revision. Pending replay cannot start another save. A genuinely new register-sourced save with expired lookup offers manual entry or explicit recheck.

[Detailed FE-04](04-story-save-business-information.md)

## 3. Unhappy paths — 3 minutes

| Problem | Feedback / next action |
| --- | --- |
| Format/checksum failure | Type-specific correction; zero calls |
| Label/length conflict | “An ABN has 11 digits. Check the label or number.”; analogous ACN copy |
| No match / new registration | Check the number or enter details manually; help on delays, activation and register checks |
| Timeout/offline/provider failure | Explain the problem; primary manual entry, one optional retry after the initial lookup, then manual entry/help only; offline saves wait for reconnection |
| Suppressed/incomplete details | Manual entry with unverified provenance; preserve identifier |
| Inactive/unsupported identity | Preserve identifier; correction/support without invented details |
| Non-ACN/unconfirmed ASIC registration | Unconfirmed kind offers manual entry; confirmed non-ACN asks for ABN/support; no automatic extra lookup |
| Browser Back/Forward or refresh | Dedicated entry/details URLs; retain same-session state, guard missing/mismatched state, never dispatch lookup/save automatically |
| Save failure/conflict/expiry | Retain answers; safe retry, reload newer data, or manual entry/explicit recheck for expired lookup |

Appendices cover duplicates, navigation races, stale choices, telemetry gaps, and concurrency. Help does not promise a universal 24–48-business-hour publication window.

## 4. Dependencies, measures, and trade-offs — 2 minutes

Backend owns register mapping, validation, 24-hour caching, duplicate-call suppression, the durable two-attempt per-identifier lookup budget, a per-call cost ledger linked to the initiating user, and safe saves. Analytics measures stage visits and active/elapsed duration; requests carry optional visit/app context and server derives trusted user/draft/attempt identity. Missing telemetry never blocks onboarding. Measure completion rate by retrieved/manual path, manual fallback uptake, stage time, cache hit ratio, and estimated lookup cost per completion: outbound attempts × assumed $0.60 / completed attempts. Agree targets after a baseline.

The trade-off is 24-hour staleness for lower latency/cost. Not caching misses helps new registrations but leaves retries chargeable; abuse controls need a separate rollout decision. A register/custom-name match is not AML clearance. Engineers can start against fixtures while provider, eligibility/classification, catalogue, support, and navigation decisions are confirmed.

[Detailed appendices](README.md#detailed-appendices) · [Sources and AI collaboration](10-source-notes.md)
