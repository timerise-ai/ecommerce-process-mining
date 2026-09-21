# E-commerce playbook **[A]** — none of this comes from the source

The source was designed around payroll, HR and ERP portals. This file refits it to the
back office of a company that sells online. It is judgement, not measurement: use it to
start the pilot, then replace it with what the pilot shows.

## Where the work actually happens

| Tool family | Examples | Capture fit | Watch for |
|---|---|---|---|
| Shop admin | Shopify, WooCommerce, Magento, Shopware, PrestaShop admin | good — server-rendered or labelled React forms | buyer PII on almost every screen |
| Marketplace seller panels | Amazon Seller Central, Allegro, eBay, Kaufland, Zalando partner | mixed — heavy SPAs, generated ids | bot protection; terms of use (below) |
| Carrier and label portals | DHL, DPD, InPost, UPS, GLS business portals; label aggregators | good for forms; label previews are canvas/PDF | tracking numbers vs the card scrubber |
| Helpdesk | Zendesk, Freshdesk, Gorgias, Front | fair — rich-text editors | message bodies are free-text PII; consider label-only capture |
| WMS / ERP / OMS | web clients of the warehouse and finance systems | good where web-based | handheld scanners and desktop clients are invisible → gap form |
| Payments and risk | PSP dashboards, fraud review, chargeback portals | **exclude by default** | card data, bank data, regulated screens |
| Supplier / B2B portals | wholesaler ordering sites, EDI web front ends | good | each supplier's site differs — long tail |
| Spreadsheets | Google Sheets, Excel online | poor — canvas grid | gap form, `tool_gap` kind |

**Pre-populate every employee's exclusions** with the PSP dashboards, banking, payroll,
webmail and the identity provider. They can add more; they should not have to think of
these.

**Read each marketplace's seller terms before capturing its panel.** Some restrict
automated access or third-party tools that read panel data. A read-only extension
operated by the seller's own staff is usually a different case from scraping — but that
is a call for whoever owns the marketplace relationship, made per marketplace, before
the pilot.

## Processes worth mining first

Rank by *volume × variation × cost of error*. High-variation work is where SOPs and
automation pay back; low-variation work is usually already scripted.

| Process | Why it is a good target | Typical hidden steps (gap form) |
|---|---|---|
| Returns and refunds (RMA) | many branches: reason, channel, payment method, item condition | inspecting the item; calling the buyer |
| Order exceptions | address fixes, splits, out-of-stock substitutions, fraud holds | asking the warehouse; checking with a supervisor |
| "Where is my order" | three tools per ticket: helpdesk, shop admin, carrier portal | carrier phone line |
| Listing and catalogue upkeep | the same product edited in shop admin and N marketplaces | photo handling; supplier spreadsheet |
| Purchasing and supplier follow-up | portal ordering, confirmations, delay chasing | email and phone |
| Marketplace case handling | A-to-Z claims, disputes, account-health notices | — |
| Payout and payment reconciliation | matching PSP and marketplace payouts to orders | spreadsheet work — mostly `tool_gap` |
| Shipping claims | damaged/lost parcels, carrier claim forms | photos, paper forms |

Start with **one** process, **one** team, **two or three** tools.

## Default gap-form categories

The source seeded phone, in person, paperwork, training, other **[P]**. For a shop:

| slug | Name | Covers |
|---|---|---|
| `phone` | Phone calls | buyers, carriers, suppliers |
| `warehouse_floor` | Warehouse floor | picking checks, inspections, stock counts, scanner work |
| `messaging` | Chat and email outside the helpdesk | supplier email, internal chat |
| `spreadsheet` | Spreadsheets | reconciliation, price lists, purchase planning |
| `paperwork` | Paper and PDFs | customs forms, CMR, invoices, claim forms |
| `in_person` | In person | supervisor approvals, handovers |
| `training` | Training | |
| `other` | Other | |

Categories are per tenant and editable by admins. Soft-delete, never hard-delete — old
entries still point at them.

## What "done" looks like

| Output | Used for |
|---|---|
| SOP per process, signed off by someone who does the job | onboarding seasonal staff before peak |
| Decision-point list per process | the rules an automation or a checklist must encode |
| Automation shortlist ranked by annual hours, blockers first | roadmap input — with the blocker named, not just the task |
| Tool-hop map: which screens are visited together for one case | integration candidates: "every WISMO ticket opens three tools" |
| Exception catalogue by observed HTTP error and recovery path | runbooks; vendor bug reports |

## Rollout

1. **Before any code:** legal review, works-council or staff consultation, DPIA, marketplace terms. Decide screenshot mode.
2. **Gap form only, two weeks.** No extension. It costs nothing, surfaces the offline work, and shows whether people will engage at all.
3. **Spike the extension** on the real target tools — the unproven assumptions in [extension.md](extension.md).
4. **Pilot: 5–10 volunteers, one process.** Volunteers, genuinely. Measure: confidence mix per tool, redaction counts, events per session, storage per person-day.
5. **Review a week of stored events with the pilots in the room.** Show them exactly what was kept. Fix labels and exclusions. This meeting decides whether the wider team opts in.
6. **First SOP**, reviewed by the people it describes.
7. **Widen by process, not by headcount.**

Avoid peak season for steps 3–5. Nobody volunteers in the second half of November.

## How this goes wrong

| Failure | Looks like | Prevent by |
|---|---|---|
| It becomes monitoring | a manager asks "who handled the fewest returns" | aggregates only; SOPs per role; say no in writing, early |
| Buyer data in the warehouse | names and addresses in event payloads | customer-PII labels on; `metadata_only`; sample stored events weekly |
| Scrubber eats product data | `[CARD]` in a barcode field | the prefix-and-length rule; your own identifier corpus as a test |
| Nobody opts in | 3 of 40 | it was announced, not discussed; start with the gap form and volunteers |
| SOPs nobody trusts | generated text contradicts practice | sign-off by practitioners; show divergence as open questions, do not average it away |
| Capture breaks a seller panel | panel degrades with the extension active | test each tool; list incompatible ones; exclude them |
| Volume surprise | storage or AI bill | budgets, heartbeats, per-tenant cost ceiling, retention from day one |

## Checklist

- [ ] One process, one team, two or three tools chosen
- [ ] Payments, banking, webmail, IdP pre-excluded for everyone
- [ ] Marketplace terms checked for each captured panel
- [ ] Categories seeded for this business
- [ ] Pilot metrics named before it starts
- [ ] Stored-events review scheduled with the pilots
