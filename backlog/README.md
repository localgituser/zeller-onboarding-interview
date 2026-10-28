# Business-information onboarding interview pack

**Present [the short interview walkthrough](INTERVIEW-WALKTHROUGH.md).** It contains the goal, assumptions, four concise frontend tickets, key unhappy paths, and dependencies in one document, with a suggested 15-minute allocation. Open the appendices when discussing implementation details.

## Detailed appendices

Existing filenames are retained so story links and handover references continue to work.

| Appendix | Content |
| --- | --- |
| [Epic](00-epic-business-information.md) | Full goal, assumptions, measures, readiness and non-goals |
| [FE-01](01-story-enter-identifier.md) | Live spacing, pasted labels, normalisation, validation, placeholders and accessibility |
| [FE-02](02-story-lookup-and-loading.md) | Request/loading, 3-second cap, one-retry limit, manual fallback and stale-response protection |
| [FE-03](03-story-review-business-and-trading-name.md) | Retrieved or manually entered identity and optional trading-name text with suggestions |
| [FE-04](04-story-save-business-information.md) | Revenue/category, register/manual saves, acknowledgement, replay and restoration |
| [FE-05](05-story-errors-and-help.md) | Error copy, explicit recovery and help content |
| [BE-01](06-story-lookup-cache-and-cost-ledger.md) | Provider mapping, cache, canonical keys, cost ledger and save guarantees |
| [DATA-01](07-story-onboarding-analytics.md) | Events, timing, client/server context and completion ownership |
| [Shared contracts](08-shared-contracts.md) | Shared URLs/browser history, deterministic rules, proposed requests/responses and fixtures |
| [Unhappy paths and decisions](09-unhappy-paths-and-decisions.md) | Risk register and remaining integration decisions |
| [Sources and AI collaboration](10-source-notes.md) | Official evidence, proposed defaults and changes through review |
| [Independent critique](11-independent-critique.md) | Original findings and a fix record; not a fresh independent re-review |

Detailed stories retain metadata, context, story statements, deterministic acceptance criteria, architectural guardrails, non-goals, and verification notes. Their provisional IDs are not existing Jira keys. The walkthrough summarises them for presentation; appendices govern engineering handover.

No application code or Jira tickets have been created, and no paid provider lookup has been made. Integration still requires provider, eligibility/classification, catalogue, navigation, and support decisions; frontend work can start against the proposed fixtures.
