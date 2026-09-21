---
name: ecommerce-process-mining
description: >
  Build employee process mining for an e-commerce back office: a consented Manifest V3
  browser extension that captures DOM events (not pixels) from the admin tools staff
  already use — shop admin, marketplace seller panels, carrier portals, helpdesk, WMS,
  ERP — plus a gap-capture form for the work a browser cannot see, feeding an AI
  pipeline that writes per-role SOPs and ranks automation candidates. Use when: (1)
  documenting how orders, returns, listings, supplier and support work is really done,
  for onboarding, audit or automation scoping, (2) building or auditing an employee
  activity-capture extension, its consent model, PII scrubbing, or its OAuth-style
  PKCE connect flow, (3) the user mentions: process mining, task mining, SOP generation,
  "document what employees do", routine capture, browser extension capture, DOM event
  capture, MutationObserver, captureVisibleTab, chrome.identity launchWebAuthFlow,
  consent toggle, excluded domains, PII scrubber, dom_confidence, perceptual hash,
  works council, GDPR Art. 88. Carries a tested scrubber that tells a card from an
  EAN, exclusion matching shared by both sides, an atomic one-time-code exchange, a
  consent ledger, and aggregate-only enrollment stats. Next.js App Router + Postgres
  oriented; backend is a seam. Not product analytics, not session replay, not
  screen recording, not event-log mining from ERP tables.
---

# E-commerce Process Mining

Capture what back-office staff actually do — as structured events read from the page's
DOM, not as video — and turn it into SOPs and an automation shortlist. The shaping
insight: **in e-commerce the screens are full of other people's data.** The employee
consents; the customer whose order is on screen never did. Every design choice below
follows from treating capture as a privacy system first and a data pipeline second.

## Maturity — read this before trusting anything

Every section in every reference carries one of these tags.

| Tag | Meaning |
|---|---|
| **[P] Proven** | Ran in the source application. |
| **[D] Designed** | Specified in the source's architecture documents; never built there. |
| **[A] Added** | Introduced by this skill. Verified only by the tests shipped with it. |

The source app shipped the consent record, the gap-capture form and a manager stub
**[P]**. The extension, ingest route, connect flow, scrubber and AI pipeline were a
detailed design **[D]**. All code here is new **[A]**: written to that design, hardened
where the design had defects, and tested (76 unit tests, 24 SQL checks). **No part of
the capture path has met a production browser.** Plan a spike — see
[extension.md](references/extension.md) "Unproven assumptions".

## When to use

- Mapping real order, return, listing, purchasing or support workflows across several web tools
- Building the capture extension, its consent/exclusion/pause controls, or the ingest API
- Generating per-role SOPs, or ranking manual work by cost for automation

## When NOT to use

- **Customer behaviour on the storefront** — that is product analytics; use your analytics stack.
- **"Who opened this link"** — server-side visit logging; use `visit-logger`.
- **Classic process mining over ERP/OMS event logs** (case id + activity + timestamp tables) — a data-warehouse job, no capture needed.
- **Productivity scoring or surveillance.** This design cannot do it and must not be bent to: consent is per-person and revocable, managers see aggregates.
- **Desktop apps.** A browser extension sees browser tabs only.

## Architecture

```
 EXTENSION (MV3)                    SERVER                         AI (batch)
 content script                     POST /ingest                   1 narrate each event
  label + confidence  ─┐             Bearer key, emit scope           (cheap text model;
  value gate           │             consent re-read per batch         low-confidence →
 service worker        ├─ batch ──▶  exclusions (again)                vision on the crop)
  pause · budget       │             scrub  ◀── authoritative      2 mine branches across
  exclusions (first)   │             idempotent insert                sessions
  screenshot gate     ─┘                  │                        3 write the SOP
  offline queue                           ▼
                                    events ◀── gap-capture form (phone, floor, paper)
 connect: PKCE ──▶ consent page ──▶ one-time code ──▶ exchange ──▶ scoped API key
```

## Critical facts

1. **Two consents, not one.** Connecting the extension authorises it to *talk*; the consent toggle authorises the server to *accept*. Ingest re-reads consent on every batch — never cache it.
2. **Exclusions beat the allowlist, and run twice.** Extension first (no local state is ever created), server again (a stale client still leaks nothing). Both import one matcher.
3. **Consent is per person *per tenant*.** Staff shared across two legal entities consent to each separately; one row per user silently moves consent between them.
4. **Managers never see who opted in.** Aggregate only, and suppressed below a group of five — with three employees the count *is* the names.
5. **A card number and a barcode look alike.** About 1 in 10 EAN/GTIN/tracking numbers passes Luhn. Match on length *and* issuer prefix or the scrubber eats your product data.
6. **`code=` is a discount code far more often than an OAuth code.** Drop credential URLs by shape (`state=` alongside, or an auth path), not by parameter name.
7. **A whole-viewport perceptual hash cannot see one field change.** Dedup on it alone and low-confidence events reuse a screenshot taken *before* the value existed. Hash the element crop too; never dedup low-confidence events.
8. **Service workers die mid-handshake.** The PKCE verifier lives in `chrome.storage.session`; timed pauses end by the clock, not by an alarm that may never fire.
9. **Screenshots bypass the scrubber.** It reads JSON, not pixels. Default to `metadata_only` where buyer data is on screen.
10. **The consent toggle is necessary, not sufficient.** EU deployments need a legal basis and usually works-council consultation before go-live.

## Hard rules

> **Never fail open on a privacy switch.** Unreadable pause state means paused; an unparseable host means excluded; a failed consent save must leave the toggle showing what the server believes.

> **Never read a value you would have to scrub.** Password, card, OTP and hidden fields are skipped at the source. The scrubber is a net, not a licence.

> **Never let the redeem be two statements.** One-time codes are claimed with a single `DELETE … RETURNING`.

> **Never accept an authorization request from an unknown extension ID.** A well-formed ID proves nothing; keep an allowlist.

> **Never report success to the extension for an event you did not store.** It will delete its only copy.

> **Never present [D] or [A] material as production-proven.** Keep the tags when you adapt this.

## Quick start

1. Read [adaptation.md](references/adaptation.md); fill the seam table against the host.
2. Settle the legal posture and screenshot mode — [consent-and-privacy.md](references/consent-and-privacy.md).
3. Land the schema — [data-model.md](references/data-model.md).
4. Build the employee and manager pages — [console-surfaces.md](references/console-surfaces.md). This alone is shippable and is the only **[P]** slice.
5. Add ingest and the scrubber — [ingest.md](references/ingest.md), [pii-scrubber.md](references/pii-scrubber.md).
6. Add the connect flow — [extension-auth.md](references/extension-auth.md).
7. Build the extension — [extension.md](references/extension.md), [screenshots.md](references/screenshots.md); port the tests from [testing.md](references/testing.md).
8. Choose target apps and categories — [ecommerce-playbook.md](references/ecommerce-playbook.md); then the pipeline — [ai-pipeline.md](references/ai-pipeline.md).

## Reference directory

| Scenario | Trigger keywords | Reference |
|---|---|---|
| Fitting this to a host app | seam, rename, tenant, auth guard, api key, event bus, i18n | [adaptation.md](references/adaptation.md) |
| Tables, RLS, neutral types | schema, migration, consent ledger, RLS, Firestore, enrollment stats, k-anonymity | [data-model.md](references/data-model.md) |
| What may be captured, and from whom | consent, excluded domains, allowlist, pause vs revoke, GDPR, BDSG, works council, customer data | [consent-and-privacy.md](references/consent-and-privacy.md) |
| Masking before storage | scrub, PII, Luhn, EAN, IBAN, JWT, email mask, discount code, label redaction | [pii-scrubber.md](references/pii-scrubber.md) |
| The ingest API | ingest route, batch, 207, retry, idempotency, 403 consent_required, event types, retention | [ingest.md](references/ingest.md) |
| Connecting the extension | PKCE, launchWebAuthFlow, chromiumapp.org, one-time code, invalid_grant, revoke, storage.sync | [extension-auth.md](references/extension-auth.md) |
| The extension itself | Manifest V3, service worker, MutationObserver, label resolution, dom_confidence, pause, budget, queue, session_id, Shadow DOM | [extension.md](references/extension.md) |
| Screenshots | captureVisibleTab, throttle, perceptual hash, aHash, bounding box, crop, metadata_only | [screenshots.md](references/screenshots.md) |
| Employee + manager pages | my routine, gap capture, consent toggle, optimistic update, automation ranking, help page | [console-surfaces.md](references/console-surfaces.md) |
| Events to SOPs | narrative, branching, SOP, model routing, pgvector, VLM, cost | [ai-pipeline.md](references/ai-pipeline.md) |
| What to capture in a shop | Shopify, marketplace, returns, RMA, carrier portal, WMS, categories, rollout | [ecommerce-playbook.md](references/ecommerce-playbook.md) |
| Proving it | vitest, happy-dom, SQL checks, psql | [testing.md](references/testing.md) |
| What was fixed, kept, added | provenance, defects, deviations, fix order | [provenance.md](references/provenance.md) |
