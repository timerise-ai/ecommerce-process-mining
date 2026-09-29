# Provenance

The engineering ledger for this skill. Three things are kept apart here, and keeping them apart is the skill's
credibility: what the audit of the earlier implementation changed and how the templates verify it, what was
kept deliberately and why it is safe, and what was designed here and has never run in production. Read this
file before simplifying anything in `references/`: most of the odd-looking parts of the templates are an entry
below.

The earlier implementation was the employee process-mining module of a back-office platform. Part of it ran:
the consent record, the gap-capture form and a manager view. The extension, the ingest route, the connect
flow, the scrubber and the AI pipeline existed as design notes rather than as code, while the project's own
guidance file listed several of those pieces as shipped.

## Maturity tags

Every section in every reference carries one of these, and anything written from this skill keeps them.

| Tag | Meaning |
|---|---|
| **[P] Proven** | Ran in the earlier implementation. |
| **[D] Designed** | Specified in the earlier implementation's design notes; never built there. |
| **[A] Added** | Introduced by this skill. Verified by the build and the tests shipped with it, and by nothing else. |

The templates are all **[A]** code: written to the design, corrected where the audit found the design wrong,
and covered by 76 unit tests and 24 SQL checks. **No part of the capture path has met a production browser.**
The spike that settles that is in [extension.md](extension.md) under *Unproven assumptions*.

## Fixed: defects in the running code **[P]**

Six entries. Each names what the templates do instead and where that is verified.

### 1. The consent switch could show Off while the server said On
The toggle updated local state before calling the server and never rolled back on failure. A failed revoke
left the switch Off, an error toast faded, and the server still held an active consent. Latent only because no
capture existed yet. **Shipped:** a server-confirmed toggle, with its suite, in
[console-surfaces.md](console-surfaces.md).

### 2. Managers could read who opted in
The principle was written down: aggregate only. The row-level policy granted `process_mining:read` holders
`SELECT` on the consent table, so any manager's browser client could list individuals regardless of what the
page rendered. **Shipped:** no team policy, and a suppressing aggregate function, in
[data-model.md](data-model.md). The SQL checks assert that a manager role reads zero consent rows.

### 3. Consent was one row per user in a multi-tenant system
Primary key `user_id`, with a tenant column. The platform supported staff shared across tenants. Granting in a
second tenant rewrote the row: the first tenant's consent vanished with no revoke recorded, while captured
events were strictly per-tenant. **Shipped:** the composite key `(user_id, scope_id)`.

### 4. Two sources of truth for the tenant
The consent action re-derived the tenant by joining user to store to organisation; the page and the policy's
`with check` used the session claim. When the two disagree the policy refuses the write. Read from the code,
not reproduced at runtime. **Shipped:** the session's tenant everywhere, in the page, the action and the
policy.

### 5. No consent history
Re-granting overwrote `consented_at`. Revoking with no row updated nothing, silently, and still emitted a
"revoked" audit event. History lived only in an event log with a short hot retention. **Shipped:** an
append-only ledger with an immutability trigger; a zero-row revoke is reported to the caller.

### 6. Enrollment count capped and unscoped
Counted by fetching rows and reading `.length`, so it was bounded by the API's maximum-rows setting, 1,000 by
default on Supabase, and it carried no tenant filter, so an all-tenant admin saw more consents than employees.
**Shipped:** a server-side count inside the stats function, scoped to the acting tenant.

### Smaller ones, fixed in the same pass
Entries were accepted under soft-deleted categories. A delete reported success for zero rows. The entry list
stopped at a fixed cap with no signal. Every string was hard-coded in English inside an internationalised app.
Dates were formatted in the server's locale rather than the viewer's.

## Corrected: defects in the design **[D]**, caught before anyone built them

Fifteen entries. The middle column is what the defect costs, not what it broke.

| # | The design said | Problem | Shipped |
|---|---|---|---|
| 1 | Exchange: `SELECT` the grant, verify, then `DELETE` | two concurrent redeems both mint a key | a single `DELETE ... RETURNING`; verify after |
| 2 | Any well-formed extension id accepted | a look-alike extension obtains a real key from a real consent click | an id allowlist |
| 3 | Extension id `[a-z]{32}` | the alphabet is a to p | `[a-p]{32}` |
| 4 | Challenge "44-char" in one section, 43 in another | an unpadded S256 challenge is 43 | 43, checked in SQL and in code |
| 5 | Verifier "in memory only" against `storage.session` | worker eviction mid-handshake loses it | `chrome.storage.session` |
| 6 | API secret in `chrome.storage.sync` | syncs the secret to the user's account and to other machines | `chrome.storage.local` |
| 7 | Email: mask right of the `@` | hides the company, keeps the person | mask the local part, keep the domain |
| 8 | Any 13 to 19 digit Luhn run is a card | about one barcode or tracking number in ten passes Luhn | length plus issuer prefix |
| 9 | Drop any URL carrying `code=` | that is every discount-code page in a shop | OAuth shape only |
| 10 | Dedup on a whole-viewport hash | blind to one field changing, and it reuses a stale image for exactly the events that need a fresh one | viewport plus crop, and never for low confidence |
| 11 | Per-event `sendMessage` pause check, "synchronous, microseconds" | asynchronous, milliseconds, and it wakes the worker | a local copy kept current by `storage.onChanged`; start paused |
| 12 | "One click on the toolbar icon pauses" plus a popup | Manifest V3 allows one or the other | popup first, with a command shortcut for one-action pause |
| 13 | `activeTab` relied on for event-triggered capture | `activeTab` comes from a gesture on the extension, not from a click in the page | flagged as an unproven assumption to spike |
| 14 | An ingest emitter that never surfaces failure | the extension deletes events the server never stored | `emit` returns false, the route answers `207` with retry indexes |
| 15 | `*.host` means subdomains only, and pasted URLs are rejected | a privacy control that under-matches and frustrates | the apex is included; pasted URLs are normalised |

Not a defect, reinforced anyway: the design's `redirect_uri` regex is correct as written. It is reimplemented
as a parsed-hostname comparison because a regex stays correct only while its punctuation survives editing.

## Kept deliberately

Nine choices that look redundant and are not. Each is safe for the reason given.

- **Two consents** (connecting is not consenting). It is what lets someone install first and decide later.
- **Consent re-read on every batch.** A cache is a window in which revoked consent is still honoured.
- **The `emit` capability on no human role.** It stops a session cookie from posting forged activity.
- **Captured events never cross tenants**, even within one corporate group.
- **One-day hot retention with archive-gated pruning.** Volume is high and nothing reads raw events online.
- **Empty install-time host permissions.** Every host is granted at runtime, by the employee.
- **No capability on the employee's own page.** Being signed in is enough to manage your own consent.
- **Soft-deleted categories.** Old entries still point at them.
- **Status codes, not response bodies.** The status carries the exception signal; the body is a customer.

## Added **[A]**

Designed in this skill and run only against its own tests: the e-commerce refit (the customer as a
non-consenting data subject, customer-PII labels, `metadata_only` as the default, the playbook, the
categories, the rollout), IBAN detection, value gating at the point of capture through `mayReadValue`,
per-event validation, the idempotency key, the age window, the iframe and endpoint exclusion checks, the queue
purge on consent loss, small-group suppression, automation ranking, role-level SOPs, entry editing, and every
test.

Found by the agent evals and added to the instructions rather than the templates: every template is
written as shipped, with privacy tightened in the host's wiring; the suites are installed from the registry
and run as written; `screenshot.ts` and its suite ship even with screenshots off; and the handover names the
pinned extension id, the unproven capture path and the legal basis.

Removed from the design: request-payload capture, `chrome.debugger` response bodies, and `user_email` on the
exchange response.

## Unverified: do not mistake any of this for proven

- Every `chrome.*` snippet and the manifest: not compiled, not loaded.
- The Supabase seam functions and the Firestore notes: not executed.
- Field-read accuracy, dedup ratio, volume and storage figures: estimates from the earlier design, never
  measured, here or there.
- Legal references: orientation only. The earlier design cited Code du travail L1221-9, which concerns job
  applicants; L1222-4 and L2312-38 are cited here instead, and no citation in this skill has been checked
  against a current consolidated text. **Have counsel confirm every citation before it appears in a DPIA.**

## If you are maintaining the earlier implementation

Fix order, most damaging first.

1. Remove the manager `SELECT` policy on the consent table and add an aggregate function. This is a live
   privacy hole.
2. Make the consent toggle server-confirmed, before any capture ships.
3. Re-key consent to `(user_id, organization_id)` and derive the tenant from the session in the action.
4. Add a consent ledger and report zero-row revokes.
5. Correct the project guidance and the design notes. They describe the extension, the authorization page,
   the exchange and revoke routes, the grants table, the excluded-domains column and the help page as
   shipped. None exist, and an agent or a new engineer reading them builds against ghosts.
6. Apply the corrections table above before implementing the deferred pieces.
7. Key the page strings, paginate entries, and scope the enrollment count.
