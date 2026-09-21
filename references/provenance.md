# Provenance

Extracted from a Next.js 16 / Supabase commerce platform's employee process-mining
module. **The honest shape of the source:** about 750 lines of running code — two
migrations, an employee page with its actions, a manager stub — beside roughly 1,700
lines of architecture documents describing an extension, ingest route, connect flow,
scrubber and AI pipeline that did not exist in the repository. The project's own
guidance file listed several of those pieces as shipped; none were present.

So this skill is not a transplant of proven code. It is: the proven slice, audited and
hardened; the design, implemented as tested modules with its defects corrected; and an
e-commerce refit the source never had. The **[P]/[D]/[A]** tags throughout say which is
which. Fidelity position: **hardened**, every deviation recorded below.

## Fixed — defects in the running code **[P]**

### 1. The consent switch could show Off while the server said On
The toggle updated local state before calling the server and never rolled back on
failure. A failed revoke left the switch Off, an error toast faded, and the server
still held an active consent. Latent only because no capture existed yet.
**Shipped:** a server-confirmed toggle — [console-surfaces.md](console-surfaces.md).

### 2. Managers could read who opted in
The principle was written down — aggregate only. The row-level policy granted
`process_mining:read` holders `SELECT` on the consent table, so any manager's browser
client could list individuals regardless of what the page rendered.
**Shipped:** no team policy; a suppressing aggregate function — [data-model.md](data-model.md).

### 3. Consent was one row per user in a multi-tenant system
Primary key `user_id`, with a tenant column. The platform supported staff shared across
tenants. Granting in a second tenant rewrote the row: the first tenant's consent
vanished with no revoke recorded, while captured events were strictly per-tenant.
**Shipped:** composite key `(user_id, scope_id)`.

### 4. Two sources of truth for the tenant
The consent action re-derived the tenant through user → store → organisation; the page
and the policy's `with check` used the session claim. When the two disagree the
policy refuses the write. (Read from the code; not reproduced at runtime.)
**Shipped:** the session's tenant everywhere.

### 5. No consent history
Re-granting overwrote `consented_at`. Revoking with no row updated nothing, silently,
and still emitted a "revoked" audit event. History lived only in an event log with
seven-day hot retention.
**Shipped:** an append-only ledger with an immutability trigger; zero-row revoke is reported.

### 6. Enrollment count capped and unscoped
Counted by fetching rows and reading `.length` — bounded by the API's maximum-rows setting
(1,000 by default on Supabase) — with no tenant filter, so an all-tenant admin saw more consents than employees.
**Shipped:** a server-side count in the stats function.

### 7. Smaller ones
Entries accepted under soft-deleted categories · delete reported success for zero rows ·
the entry list stopped at 200 with no signal · every string hard-coded in English
inside an internationalised app · dates formatted in the server's locale.

## Corrected — defects in the design **[D]**, caught before anyone built them

| # | The design | Problem | Shipped |
|---|---|---|---|
| 1 | Exchange: `SELECT` grant, verify, then `DELETE` | two concurrent redeems both mint a key | single `DELETE … RETURNING`; verify after |
| 2 | Any well-formed extension id accepted | a look-alike extension obtains a real key from a real consent click | id allowlist |
| 3 | Extension id `[a-z]{32}` | alphabet is a–p | `[a-p]{32}` |
| 4 | Challenge "44-char" in one section, 43 in another | unpadded S256 is 43 | 43 |
| 5 | Verifier "in memory only" vs `storage.session` | worker eviction mid-handshake loses it | `storage.session` |
| 6 | API secret in `chrome.storage.sync` | syncs the secret to the user's Google account and other machines | `storage.local` |
| 7 | Email: mask right of `@` | hides the company, keeps the person | mask the local part |
| 8 | Any 13–19-digit Luhn run is a card | ~10% of barcodes and tracking numbers | length + issuer prefix |
| 9 | Drop any URL with `code=` | that is every discount-code page | OAuth shape only |
| 10 | Dedup on a whole-viewport hash | blind to a field changing; stale image for exactly the events needing a fresh one | viewport + crop; never for low confidence |
| 11 | Per-event `sendMessage` pause check, "synchronous, microseconds" | asynchronous, milliseconds, wakes the worker | local copy via `storage.onChanged`; start paused |
| 12 | "One click on the toolbar icon pauses" plus a popup | MV3 allows one or the other | popup-first; command shortcut |
| 13 | `activeTab` relied on for event-triggered capture | `activeTab` comes from a gesture on the extension, not the page | flagged as an unproven assumption to spike |
| 14 | Ingest emitter that never surfaces failure | extension deletes events the server never stored | `emit` returns false → `207` with retry indexes |
| 15 | `*.host` means subdomains only; pasted URLs rejected | privacy control that under-matches and frustrates | apex included; URLs normalised |

Not a defect, reinforced anyway: the design's `redirect_uri` regex is correct as
written. It is reimplemented as a parsed-hostname comparison because a regex stays
correct only while its punctuation survives editing.

## Kept deliberately

- **Two consents** (connect ≠ consent). Looks redundant; it is what lets someone install first and decide later.
- **Consent re-read on every batch.** Looks wasteful; a cache is a window in which revoked consent is ignored.
- **`emit` capability on no human role.** Looks like an oversight in the role matrix; it stops a session cookie from posting forged activity.
- **Captured events never cross tenants**, even within one corporate group.
- **One-day hot retention with archive-gated pruning.**
- **Empty install-time host permissions.**
- **No capability on the employee's own page.**
- **Soft-deleted categories.**
- **Status codes, not response bodies.**

## Added **[A]**

The e-commerce refit (customer-as-non-consenting-data-subject, customer-PII labels,
`metadata_only` as default, playbook, categories, rollout) · IBAN detection · value
gating at source (`mayReadValue`) · per-event validation · idempotency key · age window ·
iframe/endpoint exclusion checks · queue purge on consent loss · small-group
suppression · automation ranking · role-level SOPs · entry edit · all tests.

Removed from the design: request-payload capture, `chrome.debugger` response bodies,
`user_email` on the exchange response.

## Unverified — do not mistake for proven

- Every `chrome.*` snippet and the manifest: not compiled, not loaded.
- Supabase seam functions and Firestore notes: not executed.
- Accuracy (95–98%), dedup ratio (50–70%), volume and storage figures: the source's estimates, never measured.
- Legal references: orientation only. One French citation in the source (Code du travail L1221-9, which concerns job applicants) was replaced with L1222-4 and L2312-38 from memory. **Have counsel confirm every citation before it appears in a DPIA.**

## If you are maintaining the original

Fix order, most damaging first:

1. Remove the manager `SELECT` policy on the consent table; add an aggregate function. *(live privacy hole)*
2. Make the consent toggle server-confirmed. *(before any capture ships)*
3. Re-key consent to `(user_id, organization_id)`; derive the tenant from the session in the action.
4. Add a consent ledger; report zero-row revokes.
5. Correct the project guidance and architecture docs: they describe the extension, the authorization page, the exchange and revoke routes, the grants table, the excluded-domains column and the help page as shipped. None exist. An agent or a new engineer reading them will build against ghosts.
6. Apply the design corrections table before implementing the deferred pieces.
7. Key the page strings; paginate entries; fix the unscoped count.
