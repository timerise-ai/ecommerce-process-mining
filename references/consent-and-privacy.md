# Consent and privacy

Capture is a privacy system that happens to produce data. This file is the contract;
the other references implement it.

## Two groups of people

| | The employee | The customer on the screen |
|---|---|---|
| Relationship | Uses the extension | Appears in the order, ticket or return being handled |
| Consented? | Yes — per person, per tenant, revocable | **No, and cannot be asked** |
| Protected by | Layers 1–4 below | Value gating, the scrubber, screenshot mode |

The source design was written for payroll and HR tools, where the data on screen is
mostly the employee's own employer's **[D]**. In a shop almost every captured screen
shows a buyer's name, address, email and order. **[A]** That changes three defaults:

1. Turn on `CUSTOMER_PII_LABELS` in the scrubber ([pii-scrubber.md](pii-scrubber.md)).
2. Start with screenshot mode `metadata_only` ([screenshots.md](screenshots.md)).
3. Capture **structure, not content**: which field was touched and in what order matters to an SOP; the buyer's street does not.

## The four layers **[D]**, applied in order to every event

| # | Layer | Enforced where | Fails how |
|---|---|---|---|
| 1 | **Consent record** — row exists, `revoked_at` is null | Server, re-read on every batch | `403 consent_required`; extension purges its queue and stops |
| 2 | **Excluded domains** — per-employee denylist | Extension *before any listener reads the tab*; server again | Event never created / dropped server-side |
| 3 | **Scrubbing** | Extension best-effort; server authoritative, before persistence | Value masked, or whole event dropped |
| 4 | **Tab allowlist** — default deny; employee opts hosts in | Extension; host permission requested at runtime per host | Tab not captured |

Layer 2 beats layer 4: an excluded domain inside an allowed tab is not captured.

Pause sits in front of all four and is checked first:

```
event → paused? → consent? → allowlisted? → excluded? → budget? → capture
```

## Pause, revoke, disconnect **[D]**

| | Pause | Revoke consent | Disconnect |
|---|---|---|---|
| For | "I'm about to take a sensitive call" | "I no longer take part" | "Remove this browser" |
| Where | Extension | Employee page | Extension popup |
| Server state | none | `revoked_at` set; ledger row | API key deleted |
| Reversible by | one click | re-granting | reconnecting |
| Pending events | kept; user may discard | **purged** on the next `403` **[A]** | kept until a key exists again, max 7 days |

Pause must be one action, visible without opening anything (a badge), and must survive
the service worker being killed — see [extension.md](extension.md).

## Excluded domains

One hostname per line. `bank.com` is exact; `*.bank.com` is every subdomain **and the
apex**.

```ts
// file: exclusions.ts
// Excluded-domain list: parsing for the settings form, matching for the
// extension (pre-capture) and the ingest route (defence in depth). One module,
// imported by both sides, so the two filters cannot drift apart.

export interface ExclusionLineError {
  line: number;
  value: string;
  reason: 'not_a_hostname' | 'wildcard_position' | 'too_many';
}

export interface ExclusionParse {
  entries: string[];
  errors: ExclusionLineError[];
}

export const MAX_EXCLUDED_DOMAINS = 500;

const LABEL = '[a-z0-9]([a-z0-9-]{0,61}[a-z0-9])?';
const HOSTNAME = new RegExp(`^${LABEL}(\\.${LABEL})*$`);

/** Lowercase, strip scheme/port/path/trailing dot, punycode an IDN. null = unusable. */
export function normalizeHost(input: string): string | null {
  let s = input.trim().toLowerCase();
  if (!s) return null;
  s = s.replace(/^[a-z][a-z0-9+.-]*:\/\//, '');
  s = s.split(/[/?#]/)[0] ?? '';
  s = s.replace(/^[^@]*@/, '').replace(/:\d+$/, '').replace(/\.$/, '');
  if (!s || s.includes('*')) return null;
  try {
    // URL canonicalises IDNs to punycode, which is what location.hostname reports.
    s = new URL(`http://${s}`).hostname;
  } catch {
    return null;
  }
  return HOSTNAME.test(s) && s.length <= 253 ? s : null;
}

export function parseExcludedDomains(raw: string): ExclusionParse {
  const entries: string[] = [];
  const errors: ExclusionLineError[] = [];
  const seen = new Set<string>();
  raw.split(/\r?\n/).forEach((rawLine, i) => {
    const value = rawLine.trim();
    if (!value) return;
    const wildcard = value.startsWith('*.');
    const body = wildcard ? value.slice(2) : value;
    if (body.includes('*')) {
      errors.push({ line: i + 1, value, reason: 'wildcard_position' });
      return;
    }
    const host = normalizeHost(body);
    if (!host) {
      errors.push({ line: i + 1, value, reason: 'not_a_hostname' });
      return;
    }
    const entry = wildcard ? `*.${host}` : host;
    if (seen.has(entry)) return;
    if (entries.length >= MAX_EXCLUDED_DOMAINS) {
      errors.push({ line: i + 1, value, reason: 'too_many' });
      return;
    }
    seen.add(entry);
    entries.push(entry);
  });
  return { entries, errors };
}

/**
 * `*.bank.com` also matches the apex `bank.com`. A person who excludes every
 * subdomain of their bank does not mean "but do record the bare domain" — for a
 * privacy control, the wider reading is the safe one.
 */
export function isHostExcluded(host: string, entries: readonly string[]): boolean {
  const h = normalizeHost(host);
  // Unparseable host: we cannot prove it is allowed, so it is not captured.
  if (!h) return true;
  return entries.some((e) =>
    e.startsWith('*.') ? h === e.slice(2) || h.endsWith(e.slice(1)) : h === e,
  );
}

export function hostOf(url: string): string | null {
  try {
    return new URL(url).hostname || null;
  } catch {
    return null;
  }
}

export interface ExclusionDiff {
  added: string[];
  removed: string[];
}

/** Payload for the audit event emitted on every save. */
export function diffExclusions(before: readonly string[], after: readonly string[]): ExclusionDiff {
  const b = new Set(before);
  const a = new Set(after);
  return {
    added: after.filter((e) => !b.has(e)),
    removed: before.filter((e) => !a.has(e)),
  };
}
```

Behaviour worth knowing:

| Input | Result | Why |
|---|---|---|
| `https://MyBank.com/login?x=1` | `mybank.com` | People paste URLs. Rejecting a paste teaches them to give up on the privacy control **[A]** (the source specified reject **[D]**) |
| `*.bank.com` vs host `bank.com` | excluded | Nobody means "every subdomain of my bank, but do record the bare domain" **[A]** |
| `bücher.example` | stored as `xn--bcher-kva.example` | `location.hostname` reports punycode; an unconverted entry never matches |
| `bank.com.evil.test` vs `*.bank.com` | not excluded | Suffix match is on `.bank.com`, anchored at the end |
| A host that will not parse | **excluded** | Cannot prove it is allowed |
| Line 501 | `too_many` error on that line | A cap that says so |

On save: parse → if `errors` is non-empty, show them against their line numbers and
save nothing → otherwise write `entries` and let the ledger trigger record the diff.
Store the typed array, never the raw textarea.

The extension fetches the list when the popup opens and every five minutes. A change
therefore takes up to five minutes to reach a browser — say so next to the field; the
server-side re-check covers the gap.

## What the employee must be told **[D]**

Before the toggle can be switched on, in their language, in plain words:

- **Captured:** clicks, field changes with the field's label, page addresses, request status codes, optional screenshots of the visible tab — only on sites they allowed.
- **Never captured:** excluded domains; tabs not allowed; other tabs or windows; passwords, card fields and one-time codes; anything while paused; response bodies.
- **Who sees it:** their own events; managers see team *entries* and generated SOPs; nobody sees who opted in.
- **How long:** state both retention periods — hot store and archive.
- **How to stop:** pause, revoke, disconnect — and that none of them needs a reason.
- **That saying no has no consequence.** If that sentence is not true in the organisation, do not deploy.

## The legal layer — not optional, not technical

A toggle records a choice; it does not create a lawful basis. **[D]**

| Jurisdiction | What applies | Usual consequence |
|---|---|---|
| EU generally | GDPR Art. 6, Art. 88; DPIA under Art. 35 is very likely required | Consent from an employee is weak (imbalance of power) — counsel often prefers legitimate interest plus a genuinely voluntary opt-in |
| Germany | BDSG §26; works council co-determination (BetrVG §87(1) no. 6) | Works agreement *before* go-live |
| France | Code du travail L1222-4, L2312-38 | Inform staff individually; consult the CSE |
| Netherlands | WOR art. 27 | Works-council consent |
| Poland | Kodeks pracy art. 22²–22³ | Purpose and scope in work regulations; notice two weeks before start |
| Customers' data | GDPR Art. 5(1)(c) minimisation; Art. 28 if a vendor processes it | Favour `metadata_only` and label redaction; list AI providers as sub-processors |

**[A]** This table is orientation, not advice, and law moves. Every deployment gets its
own review by someone qualified, and the DPIA names the AI providers that will receive
event text and screenshots.

## Managers and the boundary **[P]** principle, **[A]** enforcement

The source stated "aggregate only — never the per-employee record" and then granted
managers row-level read on the consent table. The principle is right; enforce it in
the database ([data-model.md](data-model.md)), because a page that only *renders* a
count does not stop a browser client from selecting the rows.

Suppress the aggregate for small groups and when everyone has opted in — both reveal
individuals.

## Checklist

- [ ] Legal review done for this deployment; DPIA names AI sub-processors
- [ ] Consent copy written, translated by a person, and shown before first opt-in
- [ ] Customer-PII labels on, screenshot mode decided, for any tool showing buyer data
- [ ] Exclusions enforced in the extension *and* at ingest, from one module
- [ ] Revoke purges the extension's pending queue
- [ ] No manager path to individual consent; aggregates suppressed below five
