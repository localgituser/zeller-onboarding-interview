---
id: ZBI-SOURCES
type: evidence-notes
reviewed: 2026-10-28
status: interview-draft
---

# Source notes and evidence boundaries

## Supplied material

- [Product Manager task PDF](../inputs/Zeller%20Product%20Manager%20Task.pdf), pages 1–2: two-minute overall goal; exercise cost of $0.60 per lookup; frontend happy-path tickets; ignore the no-ABN branch; list unhappy paths. Screen 3 shows business name/type, optional trading name, revenue, and category. Final navigation is not fully visible. Both pages were extracted and visually inspected.
- [Role description](../inputs/2026-10-18-zeller-product-manager.txt): delivery, requirements, stakeholder communication, risk and data-informed decisions inform the handover style. Salary research in that file is irrelevant to this exercise and is not included in the backlog.
- User's instructions: human/AI-readable structure; authenticated mid-onboarding journey; user/draft identifiers; 24-hour cache; per-call cost logging; funnel timing; optional/custom trading name; help for newly registered businesses; later document checks; bot/rate controls outside happy-path scope.
- User's clarification: deterministic format/checksum checks must precede API calls; trading name is mainly typed, with suggestions secondary. This is incorporated in FE-01, FE-03, BE-01, and the shared contract.
- User's subsequent request: apply the critique; create one shorter interview walkthrough with detailed appendices; auto-space typed digits; accept labelled/spaced/hyphenated pastes; normalise client/server before validation/use; show useful identifier placeholders.

## Primary sources reviewed

| Source | Evidence used / implication |
| --- | --- |
| [ABN Lookup method guide](https://abr.business.gov.au/Documentation/WebServiceMethods) | Separate ABN/ASIC identifier methods; current documented versions `SearchByABNv202001` / `SearchByASICv201408`; history flag and registered business-name availability. |
| [Service operation listing](https://abr.business.gov.au/abrxmlsearch/abrxmlsearch.asmx) | Confirms the identifier operations are exposed. No authenticated live lookup was made. |
| [Response guide](https://abr.business.gov.au/Documentation/WebServiceResponse) | Individual/non-individual name differences; most attributes can be absent; suppressed records and multiple/historical values need deliberate handling. |
| [Exception guide](https://abr.business.gov.au/Documentation/Exceptions) | Transport and application failures differ; exception codes/descriptions must be classified rather than treating all failures as no match. |
| [Data dictionary](https://abr.business.gov.au/Documentation/DataDictionary) | Identity/name fields, dates, replacement identifiers, and registered business-name structure inform the normalised contract. |
| [ABN format](https://abr.business.gov.au/Help/AbnFormat) | Modulus-89 checksum and official sample used for deterministic preflight fixtures. |
| [ASIC EDGE message specification, Appendix D](https://download.asic.gov.au/media/nvnh1kjr/msg_spec.pdf) | ACN weighted modulus-10 validation and leading-zero sample. |
| [ABN Lookup registration](https://abr.business.gov.au/Documentation/WebServiceRegistration) | Direct web-service access is free and uses a registered authentication GUID. Exercise cost remains a separate assumed provider expense. |
| [Business/trading names FAQ](https://abr.business.gov.au/FAQ/Names) | Legacy trading names stopped being updated in 2012 and are unsuitable identity evidence. Use registered business names only as optional suggestions. |
| [Lookup FAQ](https://abr.business.gov.au/FAQ/ABNLookup) | Coverage depends on ABN registration; hourly feed updates do not guarantee complete/current details. ACN registration alone need not produce a match here. |
| [ABR missing-ABN guidance](https://www.abr.gov.au/general-information/scheduled-site-maintenance) | An ABN should appear on its activation day; a missing active record can require contacting ABN Lookup. This does not substantiate a universal 24–48-business-hour publication SLA. |
| [ABN basics](https://abr.business.gov.au/FAQ/ABNBasics) | Distinguishes an issued ABN from an application still being processed and supplies application-help links. |

## What is deliberately proposed, not sourced fact

Endpoint paths, JSON field names, HTTP mappings, stable error codes, eligible-type policy, cache/idempotency architecture, timeouts, help-content ID, name length, catalogue contract, event schema, retention integration, and abandonment definition are design proposals for this exercise. They are not statements about Zeller's existing systems or ABR guarantees.

The live formatter, label/separator grammar, caret behaviour and placeholders are product rules, not ABR API features. The ASIC method covers non-ACN numbers too; `ASICNumber` is not an explicit ACN-kind discriminator. The separate company-type classification mapping and supported-type allowlist need Engineering/Risk confirmation, with ABN/support fallback for unconfirmed kind.

## AI collaboration record for discussion

The brief explicitly permits AI and asks to discuss how it was used. Keep this chat available alongside the tickets; these notes are a summary of the actual collaboration, not a fabricated transcript or a claim that the candidate has already reviewed every choice.

- **Requested:** review supplied inputs and API guides; draft an epic and detailed, deterministic stories with guardrails, scope, feedback, caching, cost records, and timing.
- **Evidence checked:** both brief pages, depicted states/fields, and official provider guides.
- **Changed through research:** distinguished exercise pricing from free direct ABR access; replaced legacy trading-name reliance with current registered-name suggestions; marked propagation timing unconfirmed.
- **Changed through user steering:** made manual trading-name entry the primary interaction, with secondary suggestions; strengthened explicit client/server preflight validation.
- **Changed after critique:** completion solely from backend; nullable client-context transport and trusted attempt resolution; committed-save replay before new-mutation checks; broad ASIC-number handling; a short walkthrough with detailed appendices.
- **Changed through later UX steering:** live spacing, labelled paste/separator normalisation, canonical downstream identity, caret/overflow behaviour, shared client/server fixtures, and accessible placeholder/helper guidance.
- **For candidate review:** the additional architecture defaults and eligibility boundaries are proposals. Be ready to explain which you keep, simplify, or challenge; do not present them as established Zeller policies.
