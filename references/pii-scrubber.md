# PII scrubber **[A]** (rules specified in the source **[D]**; never built there)

Runs inside the ingest route before anything is stored, so neither the event table nor
its audit trail ever holds a raw value. The extension runs the same module first as a
courtesy; the server pass is the one that counts.

**It reads JSON. It does not read screenshots.** See [screenshots.md](screenshots.md).

## What it does

| Pattern | Output | Rule |
|---|---|---|
| JWT | `[JWT]` | three base64url segments starting `eyJ` |
| Email | `[EMAIL @domain]` | local part masked, domain kept |
| Card | `[CARD]` | 14–19 digits, Luhn-valid **and** a real issuer prefix for that length |
| IBAN | `[IBAN]` | mod-97 valid |
| US SSN | `XXX-XX-XXXX` | `ddd-dd-dddd` |
| Phone | `[PHONE]` | `+` international, or a separated national format |
| Value of a sensitive-label field | `[REDACTED]` | label matches a pattern, whatever the value looks like |
| `type="password"` | `[REDACTED]` | always |
| URL carrying a credential | **whole event dropped** | see below |

## The module

```ts
// file: scrub.ts
// Server-side PII scrubber. Runs inside the ingest route BEFORE anything is
// persisted, so neither the event store nor its audit trail ever holds a raw
// value. The extension runs the same module as a best-effort first pass; the
// server pass is the authoritative one.

import type { ActionEvent } from './types';

export type RedactionKind =
  | 'jwt'
  | 'email'
  | 'card'
  | 'iban'
  | 'ssn'
  | 'phone'
  | 'label'
  | 'secret_field';

export type RedactionCounts = Partial<Record<RedactionKind, number>>;

export interface ScrubOptions {
  /** Field labels whose value is always redacted. Merged with DEFAULT_SENSITIVE_LABELS. */
  labelPatterns?: readonly RegExp[];
}

export const DEFAULT_SENSITIVE_LABELS: readonly RegExp[] = [
  /pass(word|code|phrase)|has[łl]o/i,
  /\b(cvv|cvc|cv2|security code)\b/i,
  /card ?(number|no)|numer karty/i,
  /\b(iban|swift|bic|account (number|no)|numer (konta|rachunku))\b/i,
  /\b(ssn|social security|pesel|national id|passport|tax id|nip|vat id)\b/i,
  /date of birth|\bdob\b|data urodzenia/i,
  /\b(secret|token|api[ _-]?key|otp|2fa|one[- ]time)\b/i,
];

/**
 * Opt-in: the person in these fields is a CUSTOMER, who never consented to
 * anything. Pass as labelPatterns when the captured apps show buyer records.
 */
export const CUSTOMER_PII_LABELS: readonly RegExp[] = [
  /\b(first|last|full|customer|recipient|buyer|contact) ?name\b|imi[eę]|nazwisko/i,
  /\b(street|address|addr|city|zip|post(al)? ?code)\b|ulica|adres|miasto|kod pocztowy/i,
  /e-?mail/i,
  /\b(phone|mobile|tel)\b|telefon/i,
  /\b(company|billing|shipping) (name|address)\b/i,
];

const JWT = /eyJ[A-Za-z0-9_-]{5,}\.eyJ[A-Za-z0-9_-]{5,}\.[A-Za-z0-9_-]{5,}/g;
const EMAIL = /[A-Za-z0-9._%+-]+@([A-Za-z0-9-]+(?:\.[A-Za-z0-9-]+)+)/g;
const DIGIT_RUN = /\b\d(?:[ -]?\d){12,18}\b/g;
const IBAN = /\b[A-Z]{2}\d{2}(?: ?[A-Z0-9]{4}){2,7}(?: ?[A-Z0-9]{1,4})?\b/g;
const SSN = /\b\d{3}-\d{2}-\d{4}\b/g;
const PHONE_E164 = /\+\d(?:[ .-]?\d){7,14}\b/g;
const PHONE_NATIONAL = /(?<![\d-])\(?\d{3}\)?[ .-]\d{3}[ .-]\d{3,4}(?![\d-])/g;

export function luhn(digits: string): boolean {
  let sum = 0;
  let double = false;
  for (let i = digits.length - 1; i >= 0; i--) {
    let d = digits.charCodeAt(i) - 48;
    if (double) {
      d *= 2;
      if (d > 9) d -= 9;
    }
    sum += d;
    double = !double;
  }
  return digits.length > 0 && sum % 10 === 0;
}

/**
 * Luhn alone is not enough in commerce data: about one in ten EAN-13 barcodes,
 * GTIN-14s and carrier tracking numbers passes it by chance. A digit run is a
 * card only if its length AND issuer prefix are ones a card network issues.
 */
export function looksLikeCard(digits: string): boolean {
  const n = digits.length;
  if (n < 14 || n > 19 || !luhn(digits)) return false; // 13-digit Visa is extinct; 13 digits is an EAN
  const p2 = Number(digits.slice(0, 2));
  const p3 = Number(digits.slice(0, 3));
  const p4 = Number(digits.slice(0, 4));
  if (n === 14) return p2 === 36 || p2 === 38 || (p3 >= 300 && p3 <= 305);
  if (n === 15) return p2 === 34 || p2 === 37;
  return (
    digits.startsWith('4') ||
    (p2 >= 51 && p2 <= 55) ||
    (p4 >= 2221 && p4 <= 2720) ||
    p4 === 6011 ||
    p2 === 65 ||
    p2 === 62 ||
    (p4 >= 3528 && p4 <= 3589)
  );
}

export function isValidIban(candidate: string): boolean {
  const s = candidate.replace(/ /g, '');
  if (s.length < 15 || s.length > 34) return false;
  const rearranged = s.slice(4) + s.slice(0, 4);
  let rem = 0;
  for (const ch of rearranged) {
    const code = ch.charCodeAt(0);
    const v = code >= 65 ? String(code - 55) : ch;
    for (const d of v) rem = (rem * 10 + (d.charCodeAt(0) - 48)) % 97;
  }
  return rem === 1;
}

function bump(counts: RedactionCounts, kind: RedactionKind): void {
  counts[kind] = (counts[kind] ?? 0) + 1;
}

/** Masks in place-holders that keep the SOP readable: "[EMAIL @acme.com]", "[CARD]". */
export function scrubString(input: string, counts: RedactionCounts = {}): string {
  let s = input;
  // Order matters: JWTs contain dots and digits the later patterns would chew on.
  s = s.replace(JWT, () => (bump(counts, 'jwt'), '[JWT]'));
  // Keep the DOMAIN, mask the local part: the domain tells the SOP "emailed the
  // supplier at acme.com"; the local part is what identifies a person.
  s = s.replace(EMAIL, (_m, domain: string) => (bump(counts, 'email'), `[EMAIL @${domain}]`));
  s = s.replace(IBAN, (m) => (isValidIban(m) ? (bump(counts, 'iban'), '[IBAN]') : m));
  s = s.replace(DIGIT_RUN, (m) =>
    looksLikeCard(m.replace(/[ -]/g, '')) ? (bump(counts, 'card'), '[CARD]') : m,
  );
  s = s.replace(SSN, () => (bump(counts, 'ssn'), 'XXX-XX-XXXX'));
  s = s.replace(PHONE_E164, () => (bump(counts, 'phone'), '[PHONE]'));
  s = s.replace(PHONE_NATIONAL, () => (bump(counts, 'phone'), '[PHONE]'));
  return s;
}

const CREDENTIAL_PARAMS = new Set([
  'access_token',
  'id_token',
  'refresh_token',
  'client_secret',
  'token',
  'password',
  'passwd',
  'samlresponse',
  'assertion',
]);
const AUTH_PATH = /\/(oauth2?|authorize|callback|sso|saml|login|signin|auth)(\/|$)/i;

/**
 * True when the URL itself is session-grade material. Such events are DROPPED,
 * not masked: an SSO redirect in the address bar can authenticate elsewhere.
 * `code=` alone is NOT enough — shops use it for discount and product codes —
 * so it only counts in an OAuth-shaped URL (`state=` alongside, or an auth path).
 */
export function urlCarriesCredential(url: string): boolean {
  if (new RegExp(JWT.source).test(url)) return true;
  let u: URL;
  try {
    u = new URL(url, 'http://relative.invalid');
  } catch {
    return false;
  }
  const params = new URLSearchParams(u.search);
  // Implicit-flow tokens travel in the fragment.
  new URLSearchParams(u.hash.replace(/^#/, '')).forEach((v, k) => params.append(k, v));
  let hasCode = false;
  let hasState = false;
  for (const key of params.keys()) {
    const k = key.toLowerCase();
    if (CREDENTIAL_PARAMS.has(k)) return true;
    if (k === 'code') hasCode = true;
    if (k === 'state') hasState = true;
  }
  return hasCode && (hasState || AUTH_PATH.test(u.pathname));
}

export interface ScrubbedEvent {
  /** null = the whole event must be discarded. */
  event: ActionEvent | null;
  dropReason: 'credential_url' | null;
  redactions: RedactionCounts;
}

export function scrubEvent(ev: ActionEvent, opts: ScrubOptions = {}): ScrubbedEvent {
  const redactions: RedactionCounts = {};
  const urls = [ev.tab?.url, ev.api_context?.endpoint, ev.frame_origin ?? undefined];
  if (urls.some((u) => u !== undefined && urlCarriesCredential(u))) {
    return { event: null, dropReason: 'credential_url', redactions };
  }

  const out: ActionEvent = structuredClone(ev);
  const clean = (s: string): string => scrubString(s, redactions);

  if (out.tab) {
    out.tab.url = clean(out.tab.url);
    if (out.tab.title !== undefined) out.tab.title = clean(out.tab.title);
  }
  if (out.api_context) out.api_context.endpoint = clean(out.api_context.endpoint);
  if (out.element?.label) out.element.label = clean(out.element.label);

  if (typeof out.value === 'string') {
    const label = out.element?.label ?? '';
    const patterns = [...DEFAULT_SENSITIVE_LABELS, ...(opts.labelPatterns ?? [])];
    if (out.element?.type === 'password') {
      out.value = '[REDACTED]';
      bump(redactions, 'secret_field');
    } else if (label && patterns.some((re) => re.test(label))) {
      out.value = '[REDACTED]';
      bump(redactions, 'label');
    } else {
      out.value = clean(out.value);
    }
  }
  return { event: out, dropReason: null, redactions };
}
```

## Decisions that look wrong and are not

**Emails keep their domain.** The source specified masking right of the `@` **[D]**.
That keeps `jan.kowalski` and hides `acme.com` — backwards. The local part identifies a
person; the domain tells the SOP "emailed the supplier". **[A]**

**Thirteen digits is never a card.** Thirteen-digit Visa numbers are long extinct;
thirteen digits in a shop is an EAN. Roughly one barcode in ten passes Luhn by chance,
as do GTIN-14s and carrier tracking numbers. Without length-plus-prefix matching the
scrubber turns a tenth of the product catalogue into `[CARD]` and the SOP reads "user
entered [CARD] into the Barcode field". **[A]**

**`code=` alone does not drop the event.** The source listed `code=` as a credential
parameter **[D]**. In commerce it is a discount code or product code a hundred times
for every OAuth code, and dropping those events deletes the whole promotions workflow.
It counts only beside `state=` or on an auth-shaped path. **[A]**

**Credential URLs are dropped, not masked.** An SSO redirect through an identity
provider puts session-grade material in the address bar. A masked URL still proves a
login happened at that second; the event has no SOP value. **[D]**

**Phones need a `+` or separators.** A bare ten-digit run is an order number.

**Label redaction ignores the value.** A "Card number" field holding `hello` is still
redacted. The label is the author's statement of what belongs there.

**`structuredClone` first.** The caller's event is never mutated — the ingest loop
reuses the parsed object for its own checks.

## Configuring per deployment

```ts
import { CUSTOMER_PII_LABELS, type ScrubOptions } from './scrub';

// Host seam: load extra patterns from settings, compile once per process.
export function scrubOptionsFor(extra: readonly string[]): ScrubOptions {
  const compiled = extra.flatMap((src) => {
    try {
      return [new RegExp(src, 'i')];
    } catch {
      return []; // a bad pattern in settings must not take ingest down
    }
  });
  return { labelPatterns: [...CUSTOMER_PII_LABELS, ...compiled] };
}
```

Admin-supplied regexes are a denial-of-service surface: a catastrophic pattern runs on
every event. Cap their length, test them against a long string with a time budget when
saved, and gate the setting behind `process_mining:configure`.

`DEFAULT_SENSITIVE_LABELS` carries English and Polish because the source deployment did.
**Add the labels of every language the captured tools are used in** — a German shop
admin says "Kartennummer", and an unmatched label is an unredacted value.

## Known limits — state these to whoever signs off

- Names and street addresses in **free text** are not detected. Only label redaction catches them, and only in labelled fields.
- National ID formats beyond US SSN are matched by *label*, not by value.
- Page titles are scrubbed by pattern only. A helpdesk title "Ticket from Jan Kowalski" passes through. If a captured tool puts names in titles, drop `tab.title` for that host.
- The scrubber is regex-based. It lowers risk; it does not make captured data anonymous, and the DPIA must not say it does.

## Tests

```ts
// file: privacy.test.ts
import { describe, expect, it } from 'vitest';
import { diffExclusions, isHostExcluded, parseExcludedDomains } from './exclusions';
import { isValidIban, looksLikeCard, scrubEvent, scrubString, urlCarriesCredential, CUSTOMER_PII_LABELS } from './scrub';
import type { ActionEvent } from './types';

describe('excluded domains', () => {
  it('normalises case, scheme, path, port, trailing dot and duplicates', () => {
    const r = parseExcludedDomains('MyBank.com\nhttps://mybank.com/login?x=1\nmybank.com.\n*.Health.example:8443\n\n');
    expect(r.errors).toEqual([]);
    expect(r.entries).toEqual(['mybank.com', '*.health.example']);
  });
  it('reports the line number of every bad entry and keeps the good ones', () => {
    const r = parseExcludedDomains('ok.com\nnot a host\nfoo.*.com\n-bad-.com');
    expect(r.entries).toEqual(['ok.com']);
    expect(r.errors.map((e) => [e.line, e.reason])).toEqual([
      [2, 'not_a_hostname'], [3, 'wildcard_position'], [4, 'not_a_hostname'],
    ]);
  });
  it('punycodes IDNs so they match location.hostname', () => {
    expect(parseExcludedDomains('bücher.example').entries).toEqual(['xn--bcher-kva.example']);
    expect(isHostExcluded('BÜCHER.example', ['xn--bcher-kva.example'])).toBe(true);
  });
  it('wildcard covers subdomains AND the apex, but not look-alikes', () => {
    const list = ['*.bank.com'];
    expect(isHostExcluded('login.bank.com', list)).toBe(true);
    expect(isHostExcluded('a.b.bank.com', list)).toBe(true);
    expect(isHostExcluded('bank.com', list)).toBe(true);
    expect(isHostExcluded('notbank.com', list)).toBe(false);
    expect(isHostExcluded('bank.com.evil.test', list)).toBe(false);
  });
  it('bare entry is an exact match only', () => {
    expect(isHostExcluded('bank.com', ['bank.com'])).toBe(true);
    expect(isHostExcluded('www.bank.com', ['bank.com'])).toBe(false);
  });
  it('fails closed on a host it cannot parse', () => {
    expect(isHostExcluded('', [])).toBe(true);
    expect(isHostExcluded('exa mple.com', [])).toBe(true);
  });
  it('caps the list and says so', () => {
    const raw = Array.from({ length: 502 }, (_, i) => `h${i}.example`).join('\n');
    const r = parseExcludedDomains(raw);
    expect(r.entries).toHaveLength(500);
    expect(r.errors).toHaveLength(2);
    expect(r.errors[0]?.reason).toBe('too_many');
  });
  it('diffs for the audit payload', () => {
    expect(diffExclusions(['a.com', 'b.com'], ['b.com', 'c.com'])).toEqual({ added: ['c.com'], removed: ['a.com'] });
  });
});

describe('scrubString', () => {
  it('masks the local part of an email and keeps the domain', () => {
    expect(scrubString('mail jan.kowalski@acme.com now')).toBe('mail [EMAIL @acme.com] now');
  });
  it('masks real cards, with and without separators', () => {
    expect(scrubString('4111 1111 1111 1111')).toBe('[CARD]');
    expect(scrubString('pay 5555555555554444 ok')).toBe('pay [CARD] ok');
    expect(scrubString('378282246310005')).toBe('[CARD]');
  });
  it('leaves commerce identifiers alone even when they pass Luhn', () => {
    expect(looksLikeCard('4006381333931')).toBe(false); // EAN-13, 13 digits
    const gtin14 = '10614141000415';
    expect(scrubString(`GTIN ${gtin14}`)).toBe(`GTIN ${gtin14}`);
    expect(scrubString('order 1000234567 tracking 00340434161094042557')).toBe('order 1000234567 tracking 00340434161094042557');
  });
  it('validates IBANs by checksum, not by shape', () => {
    expect(isValidIban('PL61 1090 1014 0000 0712 1981 2874')).toBe(true);
    expect(scrubString('IBAN PL61109010140000071219812874.')).toBe('IBAN [IBAN].');
    expect(scrubString('SKU PL12ABCD5678EFGH')).toBe('SKU PL12ABCD5678EFGH');
  });
  it('masks SSNs, phones and JWTs', () => {
    expect(scrubString('123-45-6789')).toBe('XXX-XX-XXXX');
    expect(scrubString('call +48 600 700 800 or (212) 555-0199')).toBe('call [PHONE] or [PHONE]');
    expect(scrubString('t=eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiIxMjM0In0.abcDEF123_-x')).toBe('t=[JWT]');
  });
  it('counts what it removed', () => {
    const counts = {};
    scrubString('a@b.co c@d.co 4111111111111111', counts);
    expect(counts).toEqual({ email: 2, card: 1 });
  });
});

describe('urlCarriesCredential', () => {
  it('drops token-bearing and OAuth-shaped URLs', () => {
    expect(urlCarriesCredential('https://idp.example/cb?access_token=abc')).toBe(true);
    expect(urlCarriesCredential('https://app.example/#id_token=abc')).toBe(true);
    expect(urlCarriesCredential('https://app.example/oauth/callback?code=xyz')).toBe(true);
    expect(urlCarriesCredential('https://app.example/x?code=xyz&state=123')).toBe(true);
    expect(urlCarriesCredential('https://a.example/p?SAMLResponse=PHNhbWw')).toBe(true);
  });
  it('keeps discount-code and product-code URLs', () => {
    expect(urlCarriesCredential('https://shop.example/admin/discounts?code=SUMMER10')).toBe(false);
    expect(urlCarriesCredential('/api/products?code=AB-1234')).toBe(false);
  });
});

const base: ActionEvent = {
  schema_version: 1, t: '2026-05-02T10:05:12.001Z', action: 'input', session_id: 'sess-0001', step_index: 3,
  tab: { url: 'https://admin.shop.example/orders/1001?email=jan@client.pl', title: 'Order #1001 — jan@client.pl' },
  element: { selector: 'input#note', label: 'Internal note', type: 'text' },
  value: 'refund to PL61109010140000071219812874',
};

describe('scrubEvent', () => {
  it('scrubs url, title and value; never mutates its input', () => {
    const frozen = structuredClone(base);
    const r = scrubEvent(base);
    expect(base).toEqual(frozen);
    expect(r.event?.tab?.url).toBe('https://admin.shop.example/orders/1001?email=[EMAIL @client.pl]');
    expect(r.event?.tab?.title).toBe('Order #1001 — [EMAIL @client.pl]');
    expect(r.event?.value).toBe('refund to [IBAN]');
    expect(r.redactions).toEqual({ email: 2, iban: 1 });
  });
  it('redacts by field label and by password type, whatever the value looks like', () => {
    expect(scrubEvent({ ...base, element: { label: 'Card number' }, value: 'hello' }).event?.value).toBe('[REDACTED]');
    expect(scrubEvent({ ...base, element: { label: 'x', type: 'password' }, value: 'hunter2' }).event?.value).toBe('[REDACTED]');
  });
  it('customer-PII labels are opt-in', () => {
    const ev = { ...base, element: { label: 'Recipient name' }, value: 'Jan Kowalski' };
    expect(scrubEvent(ev).event?.value).toBe('Jan Kowalski');
    expect(scrubEvent(ev, { labelPatterns: CUSTOMER_PII_LABELS }).event?.value).toBe('[REDACTED]');
  });
  it('drops the whole event when a URL is a credential', () => {
    const r = scrubEvent({ ...base, tab: { url: 'https://sso.example/callback?code=a&state=b' } });
    expect(r.event).toBeNull();
    expect(r.dropReason).toBe('credential_url');
  });
});
```

## Checklist

- [ ] Server pass runs before any write, including before any log line
- [ ] Label patterns cover every UI language in use
- [ ] `CUSTOMER_PII_LABELS` on wherever buyer records are on screen
- [ ] Redaction counts recorded per batch — a sudden zero means the scrubber stopped running
- [ ] Admin regexes length-capped and time-tested on save
