# Testing

What is verified, what is not, and the suites to port.

## Status

| Piece | Verified how | Result |
|---|---|---|
| All TypeScript modules | `tsc --strict`, again with `--noUncheckedIndexedAccess`, against zod 3.25 **and** zod 4.4 | clean |
| Behaviour | vitest 4, 76 cases in 9 files; DOM cases under happy-dom | pass |
| Schema + RLS | executed on PostgreSQL 18, 24 assertions, run as a non-owner API role and as the table owner | pass |
| `chrome.*` wiring snippets | **not compiled**, because no Chrome typings were available | unverified |
| `manifest.json` | **not loaded** into a browser | unverified |
| Supabase seam functions, Firestore notes | **not executed** | unverified |
| Capture on real sites, store review, accuracy and volume figures | **never measured**, here or in the earlier implementation | unknown |

The other suites sit beside their modules: privacy in [pii-scrubber.md](pii-scrubber.md), ingest in
[ingest.md](ingest.md), connect flow in [extension-auth.md](extension-auth.md), ranking in
[console-surfaces.md](console-surfaces.md).

## Running

Install from the package registry first. The registry is not an external service in the sense of an
unattended note that says none is reachable, and a suite converted to another runner no longer tests what
this file verified. Run the nine files as written, including `screenshot.test.ts` when screenshots are off:
76 cases. Tests of the host's own go in files beside them and are reported separately.

```bash
npm i zod && npm i -D vitest happy-dom @testing-library/react
npx vitest run                    # happy-dom is needed for label.test.ts
createdb pm_verify && psql -d pm_verify -v ON_ERROR_STOP=1 \
  -f db/process-mining/schema.sql -f db/process-mining/verify.sql
```

Run `verify.sql` against a **disposable** database. It creates a role and fixtures.

## Extension control logic

```ts
// file: lib/process-mining/capture-control.test.ts
import { describe, expect, it } from 'vitest';
import {
  CaptureBudget, badgeText, classifyIngestResponse, enforceQueueBounds, isCapturing, nextBackoffMs, nextStep,
  onSessionSignal, parsePauseState, reducePause, startSession, type PauseState, type QueuedBatch,
} from './capture-control';

describe('pause', () => {
  const active: PauseState = { kind: 'active' };
  it('toggle is its own inverse and is pure', () => {
    const frozen = Object.freeze({ ...active });
    const paused = reducePause(frozen, { type: 'toggle' });
    expect(paused).toEqual({ kind: 'paused-sticky' });
    expect(reducePause(paused, { type: 'toggle' })).toEqual(active);
    expect(reducePause(frozen, { type: 'toggle' })).toEqual(paused); // same in, same out
  });
  it('toggling out of a timed pause resumes', () => {
    expect(reducePause({ kind: 'paused-until', untilMs: 9e15 }, { type: 'toggle' })).toEqual(active);
  });
  it('a timed pause ends by the clock even if no alarm ever fires', () => {
    const s = reducePause(active, { type: 'pause-for', ms: 15 * 60_000, now: 1_000 });
    expect(isCapturing(s, 1_000)).toBe(false);
    expect(isCapturing(s, 1_000 + 15 * 60_000 - 1)).toBe(false);
    expect(isCapturing(s, 1_000 + 15 * 60_000)).toBe(true);
  });
  it('sticky pause never lapses', () => {
    expect(isCapturing({ kind: 'paused-sticky' }, Number.MAX_SAFE_INTEGER)).toBe(false);
  });
  it('badge: blank, bars, or minutes left rounded up', () => {
    expect(badgeText(active, 0)).toBe('');
    expect(badgeText({ kind: 'paused-sticky' }, 0)).toBe('||');
    expect(badgeText({ kind: 'paused-until', untilMs: 61_000 }, 0)).toBe('2');
    expect(badgeText({ kind: 'paused-until', untilMs: 61_000 }, 61_000)).toBe('');
  });
  it('corrupt stored state reads as PAUSED; absent state reads as active', () => {
    expect(parsePauseState(undefined)).toEqual(active);
    expect(parsePauseState({ kind: 'paused-until', untilMs: 5 })).toEqual({ kind: 'paused-until', untilMs: 5 });
    for (const junk of ['x', 7, {}, { kind: 'paused-until' }, { kind: 'nope' }]) {
      expect(parsePauseState(junk)).toEqual({ kind: 'paused-sticky' });
    }
  });
});

describe('CaptureBudget', () => {
  it('allows the burst, drops the excess, refills with time', () => {
    const b = new CaptureBudget(50, 0);
    let ok = 0;
    for (let i = 0; i < 80; i++) if (b.take(0)) ok++;
    expect(ok).toBe(50);
    expect(b.take(100)).toBe(true); // 100 ms -> 5 tokens back
  });
  it('heartbeats at most once a minute and only when it dropped something', () => {
    const b = new CaptureBudget(1, 0);
    expect(b.heartbeat(120_000)).toBeNull();
    b.take(0); b.take(0); b.take(0);
    expect(b.heartbeat(30_000)).toBeNull();
    expect(b.heartbeat(60_000)).toEqual({ dropped_count: 2 });
    expect(b.heartbeat(200_000)).toBeNull(); // counter was reset
  });
});

describe('offline queue', () => {
  const q = (id: string, createdAt: number, bytes: number): QueuedBatch => ({ id, createdAt, bytes, attempts: 0 });
  it('evicts by age, then oldest-first by size, and keeps chronological order', () => {
    const day = 86_400_000;
    const input = [q('old', 0, 10), q('a', 8 * day, 60), q('b', 9 * day, 60), q('c', 10 * day, 30)];
    const frozen = structuredClone(input);
    const { kept, evicted } = enforceQueueBounds(input, 10 * day, 100);
    expect(kept.map((b) => b.id)).toEqual(['b', 'c']);
    expect(evicted.map((b) => b.id).sort()).toEqual(['a', 'old']);
    expect(input).toEqual(frozen);
  });
  it('backs off 1s/4s/16s/64s/256s then gives up', () => {
    expect([0, 1, 2, 3, 4, 5].map(nextBackoffMs)).toEqual([1000, 4000, 16000, 64000, 256000, null]);
  });
  it('maps server answers to what the extension must do', () => {
    expect(classifyIngestResponse(200, 0)).toBe('delivered');
    expect(classifyIngestResponse(207, 0)).toBe('delivered');
    expect(classifyIngestResponse(401, 0)).toBe('disconnect');
    expect(classifyIngestResponse(403, 0)).toBe('purge_and_stop');
    expect(classifyIngestResponse(400, 0)).toBe('dead_letter');
    expect(classifyIngestResponse(503, 2)).toBe('retry');
    expect(classifyIngestResponse(503, 5)).toBe('dead_letter');
    expect(classifyIngestResponse(0, 0)).toBe('retry'); // network error
  });
});

describe('session rotation', () => {
  let n = 0;
  const newId = () => `s${++n}`;
  it('rotates on a new path, not on a query change to the same path', () => {
    const s0 = startSession(newId, '/orders');
    expect(onSessionSignal(s0, { type: 'navigate', path: '/orders' }, newId)).toBe(s0);
    const s1 = onSessionSignal(s0, { type: 'navigate', path: '/orders/1001' }, newId);
    expect(s1.sessionId).not.toBe(s0.sessionId);
    expect(s1.stepIndex).toBe(0);
  });
  it('rotates after more than five minutes away, not before', () => {
    const s0 = startSession(newId, '/x');
    const blurred = onSessionSignal(s0, { type: 'blur', now: 0 }, newId);
    expect(onSessionSignal(blurred, { type: 'focus', now: 300_000 }, newId).sessionId).toBe(s0.sessionId);
    expect(onSessionSignal(blurred, { type: 'focus', now: 300_001 }, newId).sessionId).not.toBe(s0.sessionId);
  });
  it('rotates on resume from pause, and step indexes are gapless', () => {
    const s0 = startSession(newId, '/x');
    const a = nextStep(s0); const b = nextStep(a.state);
    expect([a.stepIndex, b.stepIndex]).toEqual([0, 1]);
    expect(onSessionSignal(b.state, { type: 'resumed-from-pause' }, newId).stepIndex).toBe(0);
  });
});
```

## Label resolution and value gating

```ts
// file: lib/process-mining/label.test.ts
// @vitest-environment happy-dom
import { describe, expect, it } from 'vitest';
import { cssPath, looksGenerated, mayReadValue, resolveLabel } from './label';

const mount = (html: string, sel: string): Element => {
  document.body.innerHTML = html;
  const el = document.querySelector(sel);
  if (!el) throw new Error(`no ${sel}`);
  return el;
};

describe('resolveLabel', () => {
  it('high: label[for], wrapping label, aria-label, aria-labelledby, button text', () => {
    expect(resolveLabel(mount('<label for="a">Tracking number</label><input id="a">', '#a'))).toEqual({ label: 'Tracking number', confidence: 'high' });
    expect(resolveLabel(mount('<label>Refund  amount <input id="a"></label>', '#a'))).toEqual({ label: 'Refund amount', confidence: 'high' });
    expect(resolveLabel(mount('<input id="a" aria-label="SKU">', '#a')).confidence).toBe('high');
    expect(resolveLabel(mount('<span id="l1">Ship</span><span id="l2">to</span><input id="a" aria-labelledby="l1 l2">', '#a')).label).toBe('Ship to');
    expect(resolveLabel(mount('<button id="a"> Mark as shipped </button>', '#a'))).toEqual({ label: 'Mark as shipped', confidence: 'high' });
  });
  it('medium: placeholder, legend, a short preceding sibling, a human name attribute', () => {
    expect(resolveLabel(mount('<input id="a" placeholder="Search orders">', '#a'))).toEqual({ label: 'Search orders', confidence: 'medium' });
    expect(resolveLabel(mount('<fieldset><legend>Carrier</legend><div><input id="a"></div></fieldset>', '#a')).confidence).toBe('medium');
    expect(resolveLabel(mount('<div><span>Weight (kg)</span><input id="a"></div>', '#a'))).toEqual({ label: 'Weight (kg)', confidence: 'medium' });
    expect(resolveLabel(mount('<input id="a" name="billing_address[zip]">', '#a'))).toEqual({ label: 'billing address zip', confidence: 'medium' });
  });
  it('low: nothing semantic, a generated name, a canvas', () => {
    expect(resolveLabel(mount('<div><input id="a"></div>', '#a'))).toEqual({ label: null, confidence: 'low' });
    expect(resolveLabel(mount('<input id="a" name="field_8839201">', '#a')).confidence).toBe('low');
    expect(resolveLabel(mount('<canvas id="a" aria-label="Report"></canvas>', '#a')).confidence).toBe('low');
  });
});

describe('selectors and value gating', () => {
  it('spots generated ids and routes around them', () => {
    expect(['ember1042', ':r1f:', 'tx_9f3a2c11', 'input-20394']).toSatisfy((ids: string[]) => ids.every(looksGenerated));
    expect(looksGenerated('order-note')).toBe(false);
    expect(cssPath(mount('<form id="refund"><p><input></p><p><input id="ember1042" class="t"></p></form>', '.t'))).toBe('form#refund > p:nth-of-type(2) > input');
  });
  it('never reads passwords, card fields, OTPs, hidden inputs or opted-out regions', () => {
    const no = ['<input id="a" type="password">', '<input id="a" type="hidden">', '<input id="a" autocomplete="cc-number">',
      '<input id="a" autocomplete="section-pay one-time-code">', '<div data-pm-ignore><p><input id="a"></p></div>'];
    for (const html of no) expect(mayReadValue(mount(html, '#a'))).toBe(false);
    expect(mayReadValue(mount('<input id="a" autocomplete="street-address">', '#a'))).toBe(true);
  });
});
```

## Screenshots

```ts
// file: lib/process-mining/screenshot.test.ts
import { describe, expect, it } from 'vitest';
import { averageHash, cropRectInBitmap, decideCapture, hammingHex, isDuplicateShot, toGray8x8, triggerWantsScreenshot } from './screenshot';

function bitmap(w: number, h: number, paint: (x: number, y: number) => number): Uint8ClampedArray {
  const px = new Uint8ClampedArray(w * h * 4);
  for (let y = 0; y < h; y++) for (let x = 0; x < w; x++) {
    const v = paint(x, y); const i = (y * w + x) * 4;
    px[i] = v; px[i + 1] = v; px[i + 2] = v; px[i + 3] = 255;
  }
  return px;
}
const page = (fieldValue: number) => bitmap(320, 200, (x, y) =>
  x >= 100 && x < 140 && y >= 90 && y < 100 ? fieldValue : (Math.floor(x / 40) + Math.floor(y / 25)) % 2 ? 230 : 40);
const field = { x: 100, y: 90, w: 40, h: 10 };

describe('capture decisions', () => {
  it('typing never earns a screenshot; a click does', () => {
    expect(triggerWantsScreenshot('typing')).toBe(false);
    expect(triggerWantsScreenshot('click')).toBe(true);
  });
  it('throttles to one per 600 ms and reports staleness', () => {
    expect(decideCapture(1000, null)).toEqual({ capture: true });
    expect(decideCapture(1599, 1000)).toEqual({ capture: false, reuseOffsetMs: 599 });
    expect(decideCapture(1600, 1000)).toEqual({ capture: true });
  });
});

describe('perceptual hash', () => {
  it('is 16 hex chars, stable, and distance is symmetric', () => {
    const h = averageHash(toGray8x8(page(40), 320, 200));
    expect(h).toMatch(/^[0-9a-f]{16}$/);
    expect(averageHash(toGray8x8(page(40), 320, 200))).toBe(h);
    expect(hammingHex('ffff000000000000', '0000000000000000')).toBe(16);
    expect(hammingHex('f0', '0f')).toBe(hammingHex('0f', 'f0'));
    expect(hammingHex('ab', 'abc')).toBe(Infinity);
  });
  it('the viewport hash is BLIND to one field changing - the crop hash is not', () => {
    const before = page(40); const after = page(230);
    const vp = (p: Uint8ClampedArray) => averageHash(toGray8x8(p, 320, 200));
    const crop = (p: Uint8ClampedArray) => averageHash(toGray8x8(p, 320, 200, field));
    expect(hammingHex(vp(before), vp(after))).toBeLessThanOrEqual(5);
    const a = { viewportHash: vp(before), cropHash: crop(before) };
    const halfTyped = bitmap(320, 200, (x, y) =>
      x >= 100 && x < 120 && y >= 90 && y < 100 ? 230 : x >= 120 && x < 140 && y >= 90 && y < 100 ? 40 : 128);
    const b = { viewportHash: a.viewportHash, cropHash: averageHash(toGray8x8(halfTyped, 320, 200, field)) };
    expect(hammingHex(a.cropHash, b.cropHash)).toBeGreaterThan(5);
    expect(isDuplicateShot(b, a, 'high')).toBe(false);
  });
  it('identical shots dedupe, except for low-confidence events', () => {
    const fp = { viewportHash: 'aaaaaaaaaaaaaaaa', cropHash: 'bbbbbbbbbbbbbbbb' };
    expect(isDuplicateShot(fp, fp, 'high')).toBe(true);
    expect(isDuplicateShot(fp, fp, 'medium')).toBe(true);
    expect(isDuplicateShot(fp, fp, 'low')).toBe(false);
    expect(isDuplicateShot(fp, null, 'high')).toBe(false);
    expect(isDuplicateShot({ ...fp, cropHash: null }, fp, 'high')).toBe(false);
  });
  it('maps CSS pixels to bitmap pixels and clamps to the image', () => {
    const vp = { w: 1440, h: 900, device_pixel_ratio: 2 };
    expect(cropRectInBitmap({ x: 412, y: 318, w: 220, h: 32 }, vp)).toEqual({ x: 808, y: 620, w: 472, h: 96 });
    const edge = cropRectInBitmap({ x: 1430, y: 895, w: 50, h: 50 }, vp);
    expect(edge.x + edge.w).toBeLessThanOrEqual(2880);
    expect(edge.y + edge.h).toBeLessThanOrEqual(1800);
  });
});
```

## Schema checks

Two things this file taught, both now encoded in it:

- For an API role, RLS with no matching policy makes `DELETE`/`UPDATE` a **silent no-op**, not an error.
  Assert "the rows are still there", not "the statement failed".
- The same statements run by the table **owner** bypass RLS entirely. Only the immutability trigger stops
  those, so that case is asserted as the owner.

```sql
-- file: db/process-mining/verify.sql
\set ON_ERROR_STOP on
create role app_user nologin;
grant usage on schema public to app_user;
grant select, insert, update, delete on all tables in schema public to app_user;
grant execute on function pm_enrollment_stats(integer, integer) to app_user;

-- fixtures as owner
insert into pm_routine_categories (id, scope_id, slug, name) values
  ('00000000-0000-0000-0000-0000000000c1', '00000000-0000-0000-0000-00000000000a', 'phone', 'Phone'),
  ('00000000-0000-0000-0000-0000000000c2', '00000000-0000-0000-0000-00000000000b', 'phone', 'Phone (other tenant)');
insert into pm_routine_categories (id, scope_id, slug, name, deleted_at) values
  ('00000000-0000-0000-0000-0000000000c3', '00000000-0000-0000-0000-00000000000a', 'old', 'Retired', now());

create or replace function expect_fail(sql text, label text) returns void language plpgsql as $$
begin
  begin execute sql; exception when others then raise notice 'PASS (refused): % [%]', label, sqlstate; return; end;
  raise exception 'FAIL: % was allowed', label;
end $$;
create or replace function expect_rows(sql text, want bigint, label text) returns void language plpgsql as $$
declare got bigint;
begin
  execute format('select count(*) from (%s) q', sql) into got;
  if got <> want then raise exception 'FAIL: %, got %, want %', label, got, want; end if;
  raise notice 'PASS: % (% rows)', label, got;
end $$;

set role app_user;
select set_config('app.user_id', '00000000-0000-0000-0000-000000000001', false);
select set_config('app.scope_id', '00000000-0000-0000-0000-00000000000a', false);
select set_config('app.caps', '', false);

insert into pm_consent (user_id, scope_id) values (pm_current_user(), pm_current_scope());
select expect_fail($$insert into pm_consent (user_id, scope_id) values (pm_current_user(), '00000000-0000-0000-0000-00000000000b')$$, 'consent into a scope the request is not acting in');
select expect_fail($$insert into pm_consent (user_id, scope_id) values ('00000000-0000-0000-0000-000000000002', pm_current_scope())$$, 'consent on behalf of someone else');
update pm_consent set excluded_domains = '{mybank.com,*.health.example}' where user_id = pm_current_user();
update pm_consent set revoked_at = now() where user_id = pm_current_user();
update pm_consent set revoked_at = null, consented_at = now(), excluded_domains = '{mybank.com}' where user_id = pm_current_user();
select expect_rows($$select 1 from pm_consent_ledger$$, 5, 'ledger: granted, exclusions, revoked, granted, exclusions');
delete from pm_consent_ledger;  -- not an error for this role: RLS hides every row from DELETE
update pm_consent_ledger set action = 'granted';
select expect_rows($$select 1 from pm_consent_ledger where action = 'revoked'$$, 1, 'employee cannot delete or rewrite consent history');

insert into pm_routine_entries (user_id, scope_id, category_id, title) values (pm_current_user(), pm_current_scope(), '00000000-0000-0000-0000-0000000000c1', 'Call carrier');
select expect_fail($$insert into pm_routine_entries (user_id, scope_id, category_id, title) values (pm_current_user(), pm_current_scope(), '00000000-0000-0000-0000-0000000000c2', 'x')$$, 'entry under another tenant''s category');
select expect_fail($$insert into pm_routine_entries (user_id, scope_id, category_id, title) values (pm_current_user(), pm_current_scope(), '00000000-0000-0000-0000-0000000000c3', 'x')$$, 'entry under a retired category');

-- second user, same tenant, manager capability
select set_config('app.user_id', '00000000-0000-0000-0000-000000000009', false);
select set_config('app.caps', 'process_mining:read', false);
select expect_rows($$select 1 from pm_consent$$, 0, 'manager sees NO individual consent rows');
select expect_rows($$select 1 from pm_consent_ledger$$, 0, 'manager sees no consent history');
select expect_rows($$select 1 from pm_routine_entries$$, 1, 'manager sees team entries');
update pm_routine_entries set title = 'hijack';
select expect_rows($$select 1 from pm_routine_entries where title = 'Call carrier'$$, 1, 'manager cannot edit an employee entry');
select expect_rows($$select 1 from pm_enrollment_stats(40) where suppressed and consented is null$$, 1, 'stats suppressed when fewer than 5 opted in');
select set_config('app.caps', '', false);
select expect_fail($$select * from pm_enrollment_stats(40)$$, 'stats without process_mining:read');

-- other tenant's manager
select set_config('app.scope_id', '00000000-0000-0000-0000-00000000000b', false);
select set_config('app.caps', 'process_mining:read', false);
select expect_rows($$select 1 from pm_routine_entries$$, 0, 'other tenant manager sees nothing');

reset role;
-- stats un-suppress at scale
insert into pm_consent (user_id, scope_id) select gen_random_uuid(), '00000000-0000-0000-0000-00000000000a' from generate_series(1, 6);
set role app_user;
select set_config('app.scope_id', '00000000-0000-0000-0000-00000000000a', false);
select expect_rows($$select 1 from pm_enrollment_stats(40) where consented = 7 and not suppressed$$, 1, 'stats show 7 of 40');
select expect_rows($$select 1 from pm_enrollment_stats(7) where suppressed$$, 1, 'stats suppressed when EVERYONE opted in');
reset role;

-- grant: redeem once, expired never
insert into pm_extension_grants (code, user_id, scope_id, code_challenge, extension_id) values
  ('live', gen_random_uuid(), gen_random_uuid(), repeat('a', 43), repeat('a', 32));
insert into pm_extension_grants (code, user_id, scope_id, code_challenge, extension_id, expires_at) values
  ('dead', gen_random_uuid(), gen_random_uuid(), repeat('a', 43), repeat('a', 32), now() - interval '1 second');
set role app_user;
select expect_rows($$select 1 from pm_extension_grants$$, 0, 'one-time codes are invisible to the API role');
select expect_fail($$select * from pm_claim_extension_grant('live', repeat('a', 32))$$, 'API role calling the claim function');
reset role;
select expect_fail($$delete from pm_consent_ledger$$, 'OWNER deleting consent history');
select expect_rows($$select * from pm_claim_extension_grant('live', repeat('b', 32))$$, 0, 'claim with the wrong extension id');
select expect_rows($$select * from pm_claim_extension_grant('live', repeat('a', 32))$$, 1, 'first claim wins');
select expect_rows($$select * from pm_claim_extension_grant('live', repeat('a', 32))$$, 0, 'second claim gets nothing');
select expect_rows($$select * from pm_claim_extension_grant('dead', repeat('a', 32))$$, 0, 'expired code');

-- event sink idempotency
insert into pm_events (user_id, scope_id, event_type, idempotency_key, occurred_at, payload) values
  ('00000000-0000-0000-0000-000000000001', '00000000-0000-0000-0000-00000000000a', 'process_mining.click.captured', 's1:0', now(), '{}');
insert into pm_events (user_id, scope_id, event_type, idempotency_key, occurred_at, payload) values
  ('00000000-0000-0000-0000-000000000001', '00000000-0000-0000-0000-00000000000a', 'process_mining.click.captured', 's1:0', now(), '{}')
  on conflict (user_id, idempotency_key) do nothing;
select expect_rows($$select 1 from pm_events$$, 1, 'retried batch does not double-insert');
\echo ALL SQL CHECKS PASSED
```

## What to add in the host

- An end-to-end run of the connect flow against a real sideloaded build.
- A fixture page per target app (saved HTML of the order, return and listing screens) driven through
  `resolveLabel`, asserting the confidence mix. That number is your real accuracy baseline.
- A scrubber corpus from your own data: 200 real order numbers, SKUs, barcodes and tracking numbers, asserting
  that **none** are masked.
