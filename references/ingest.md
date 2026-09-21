# Ingest **[A]** (contract from the earlier design **[D]**, and the route was never built there)

One endpoint. The extension posts batches; the server decides, event by event, what is allowed to exist.

## Contract

`POST /api/v1/process-mining/ingest`, with `Authorization: Bearer <api key with process_mining:emit>`.

Body: `{ "events": ActionEvent[] }`, 1 to 200 events, at most 256 KB.

| Status | Body | The extension must |
|---|---|---|
| `200` | `{ accepted, retry: [], dropped, redactions }` | delete the batch |
| `207` | same, `retry: [3, 7]` | delete the batch, **re-queue only those indexes** |
| `400` | `invalid_batch` | dead-letter; the same bytes cannot succeed |
| `401` | `unauthorized` | drop to "Not connected" |
| `403` | `consent_required` | **purge the queue and stop capturing** |
| `403` | `missing_scope` | treat as `401` |
| `413` | `batch_too_large` | dead-letter; split smaller next time |
| `5xx` or a network error | none | back off 1 s, 4 s, 16 s, 64 s, 256 s, then dead-letter |

`dropped` counts events that are final and gone: `invalid`, `stale`, `excluded`, `credential_url`. They are
not errors and are never retried.

## Pipeline

```
authenticate key -> scope check -> size cap -> parse
  -> consent (re-read, for this tenant) -- none or revoked --> 403
  -> per event: schema -> age window -> exclusions -> scrub -> idempotent write
```

```ts
// file: lib/process-mining/ingest.ts
// Ingest pipeline: validate -> consent -> exclusions -> scrub -> emit.
// Everything the route handler needs from the host arrives through IngestDeps,
// so this file has no framework, no SDK and no I/O of its own.

import { z } from 'zod';
import { hostOf, isHostExcluded } from './exclusions';
import { scrubEvent, type RedactionCounts, type ScrubOptions } from './scrub';
import type { ActionEvent, CaptureAction, ConsentRecord } from './types';
import { isConsentActive } from './types';

export const MAX_BATCH_EVENTS = 200;
export const MAX_EVENT_AGE_MS = 7 * 24 * 60 * 60 * 1000;
export const MAX_CLOCK_SKEW_MS = 5 * 60 * 1000;

const box = z.object({ x: z.number(), y: z.number(), w: z.number(), h: z.number() });

export const actionEventSchema = z.object({
  schema_version: z.literal(1),
  t: z.string().datetime(),
  action: z.enum(['input', 'click', 'change', 'select', 'network', 'screenshot', 'heartbeat']),
  session_id: z.string().min(8).max(64),
  step_index: z.number().int().min(0),
  tab: z.object({ url: z.string().max(2048), title: z.string().max(512).optional() }).optional(),
  frame_origin: z.string().max(512).nullable().optional(),
  element: z
    .object({
      selector: z.string().max(512).optional(),
      label: z.string().max(256).nullable().optional(),
      type: z.string().max(32).optional(),
    })
    .optional(),
  dom_confidence: z.enum(['high', 'medium', 'low']).optional(),
  value: z.string().max(4096).nullable().optional(),
  api_context: z
    .object({
      endpoint: z.string().max(2048),
      method: z.string().max(10),
      status: z.number().int().min(0).max(599),
      duration_ms: z.number().min(0).optional(),
    })
    .optional(),
  screenshot_ref: z.string().max(512).nullable().optional(),
  screenshot_offset_ms: z.number().min(0).optional(),
  screenshot_phash: z.string().regex(/^[0-9a-f]{16}$/).optional(),
  bounding_box: box.optional(),
  viewport: z.object({ w: z.number(), h: z.number(), device_pixel_ratio: z.number() }).optional(),
  dropped_count: z.number().int().min(0).optional(),
});

export const ingestBatchSchema = z.object({
  events: z.array(z.unknown()).min(1).max(MAX_BATCH_EVENTS),
});

export const EVENT_TYPE: Record<CaptureAction, string> = {
  input: 'process_mining.input.changed',
  change: 'process_mining.input.changed',
  select: 'process_mining.input.changed',
  click: 'process_mining.click.captured',
  network: 'process_mining.network.responded',
  screenshot: 'process_mining.screenshot.captured',
  heartbeat: 'process_mining.capture.rate_limited',
};

export interface EmitInput {
  type: string;
  /** `${session_id}:${step_index}`, so a retried batch cannot double-insert. */
  idempotencyKey: string;
  userId: string;
  scopeId: string;
  occurredAt: string;
  payload: ActionEvent;
}

export interface IngestDeps {
  /** The caller's consent row for THIS scope, or null. Never another scope's. */
  loadConsent(userId: string, scopeId: string): Promise<ConsentRecord | null>;
  /** Must resolve false (not throw, not silently succeed) when the write failed. */
  emit(input: EmitInput): Promise<boolean>;
  now(): number;
  scrub?: ScrubOptions;
}

export type IngestResult =
  | { ok: false; status: 400; error: 'invalid_batch' }
  | { ok: false; status: 403; error: 'consent_required' }
  | {
      ok: true;
      status: 200 | 207;
      accepted: number;
      /** Indexes the client must KEEP and retry. Everything else is final. */
      retry: number[];
      dropped: { invalid: number; stale: number; excluded: number; credential_url: number };
      redactions: RedactionCounts;
    };

export async function processIngestBatch(
  body: unknown,
  caller: { userId: string; scopeId: string },
  deps: IngestDeps,
): Promise<IngestResult> {
  const batch = ingestBatchSchema.safeParse(body);
  if (!batch.success) return { ok: false, status: 400, error: 'invalid_batch' };

  // Consent is re-read on EVERY batch. Caching it would keep accepting events
  // after a revoke for as long as the cache lives.
  const consent = await deps.loadConsent(caller.userId, caller.scopeId);
  if (!consent || !isConsentActive(consent)) {
    return { ok: false, status: 403, error: 'consent_required' };
  }

  const now = deps.now();
  const dropped = { invalid: 0, stale: 0, excluded: 0, credential_url: 0 };
  const redactions: RedactionCounts = {};
  const retry: number[] = [];
  let accepted = 0;

  for (const [i, raw] of batch.data.events.entries()) {
    // Validate per event: one malformed row must not poison the other 199.
    const parsed = actionEventSchema.safeParse(raw);
    if (!parsed.success) {
      dropped.invalid++;
      continue;
    }
    const ev = parsed.data as ActionEvent;

    const t = Date.parse(ev.t);
    if (t < now - MAX_EVENT_AGE_MS || t > now + MAX_CLOCK_SKEW_MS) {
      dropped.stale++;
      continue;
    }

    // Defence in depth: a stale extension may not have the latest list yet.
    const hosts = [ev.tab?.url, ev.frame_origin ?? undefined, ev.api_context?.endpoint]
      .filter((u): u is string => typeof u === 'string' && /^https?:\/\//i.test(u))
      .map((u) => hostOf(u) ?? '');
    if (hosts.some((h) => isHostExcluded(h, consent.excluded_domains))) {
      dropped.excluded++;
      continue;
    }

    const scrubbed = scrubEvent(ev, deps.scrub);
    if (!scrubbed.event) {
      dropped.credential_url++;
      continue;
    }
    for (const [k, n] of Object.entries(scrubbed.redactions)) {
      const key = k as keyof RedactionCounts;
      redactions[key] = (redactions[key] ?? 0) + (n ?? 0);
    }

    const stored = await deps.emit({
      type: EVENT_TYPE[ev.action],
      idempotencyKey: `${ev.session_id}:${ev.step_index}`,
      userId: caller.userId,
      scopeId: caller.scopeId,
      occurredAt: ev.t,
      payload: scrubbed.event,
    });
    if (stored) accepted++;
    else retry.push(i);
  }

  return { ok: true, status: retry.length ? 207 : 200, accepted, retry, dropped, redactions };
}
```

## Route handler

```ts
// file: lib/process-mining/ingest-route.ts
// Framework-neutral handler: Request in, Response out. In Next.js App Router,
// `export const POST = (req: Request) => handleIngest(req, host)`.

import { processIngestBatch, type IngestDeps } from './ingest';

export const MAX_BODY_BYTES = 256 * 1024;
export const EMIT_SCOPE = 'process_mining:emit';

export interface IngestHost {
  /** Verify the Bearer key. scopeId comes from the KEY ROW, never from the request. */
  authenticate(req: Request): Promise<{ userId: string; scopeId: string; scopes: string[] } | null>;
  deps: IngestDeps;
}

const json = (status: number, body: unknown): Response =>
  new Response(JSON.stringify(body), {
    status,
    headers: { 'content-type': 'application/json', 'cache-control': 'no-store' },
  });

export async function handleIngest(req: Request, host: IngestHost): Promise<Response> {
  const caller = await host.authenticate(req);
  if (!caller) return json(401, { error: 'unauthorized' });
  if (!caller.scopes.includes(EMIT_SCOPE)) return json(403, { error: 'missing_scope' });

  // Content-Length can lie or be absent; measure what actually arrived.
  const raw = await req.text();
  if (new TextEncoder().encode(raw).byteLength > MAX_BODY_BYTES) {
    return json(413, { error: 'batch_too_large' });
  }
  let body: unknown;
  try {
    body = JSON.parse(raw);
  } catch {
    return json(400, { error: 'invalid_batch' });
  }

  const result = await processIngestBatch(body, caller, host.deps);
  if (!result.ok) return json(result.status, { error: result.error });
  const { status, ok: _ok, ...rest } = result;
  return json(status, rest);
}
```

## Why it is shaped this way

**Consent is read per batch, uncached.** A cached "yes" keeps accepting events after a revoke for as long as
the cache lives. One indexed primary-key read per batch is the price. **[D]**

**The tenant comes from the key row.** Not from the body, not from a header, not from the user's "primary"
tenant. A person in two tenants has two keys and two consents. **[A]**

**Validation is per event.** One malformed row from a buggy content script must not discard the other 199.
**[A]**

**`emit` returns `false` on failure, and the route says so.** The earlier event helper swallowed write errors
by design, which is correct for audit bookkeeping and fatal here: a `200` makes the extension delete its only
copy. Wrap the host's emitter so failure is visible. **[A]**

```ts
// file: lib/process-mining/ingest-route.ts (the emit seam)
// Host seam. `hostEmit` is whatever writes to the host's event log.
emit: async (e) => {
  try {
    return (await hostEmit(e)) !== null;
  } catch {
    return false; // never throw out of emit: one bad row would 500 the whole batch
  }
},
```

**Idempotency key is `session_id:step_index`.** A batch that timed out after the write is retried; without a
key every step lands twice and the branch miner finds loops that never happened. **[A]**

**Age window: 7 days back, 5 minutes forward.** Matches the extension's queue bound. Anything older was
captured under a consent state nobody can vouch for. **[A]**

**Exclusions check three hosts:** the tab, the iframe origin and the API endpoint. A bank widget embedded in
an allowed page is still the bank. **[A]**

## Event types **[D]**

| `action` | Stored `event_type` |
|---|---|
| `input`, `change`, `select` | `process_mining.input.changed` |
| `click` | `process_mining.click.captured` |
| `network` | `process_mining.network.responded` |
| `screenshot` | `process_mining.screenshot.captured` |
| `heartbeat` | `process_mining.capture.rate_limited` |

Consent changes are **not** process-mining events. They belong to the authorization audit trail:
`consent_granted`, `consent_revoked`, `exclusions_updated`.

Stored attributes: actor = the API key, visibility = the tenant only. Captured behaviour never crosses
tenants, even inside one corporate group. Aggregating one person's work across two employers is a line this
design does not cross. **[D]**

## Retention **[D]**

| Tier | Holds | For |
|---|---|---|
| Operational DB | raw events | about a day: volume is high, and nothing reads them here |
| Warehouse | everything, append-only | the AI stages read from here |
| Object storage | screenshots | as long as the events that reference them |

The prune job deletes only rows at or behind the warehouse export cursor. A prune that runs ahead of a stalled
export is unrecoverable data loss.

## Operating it **[A]**

| Signal | Healthy | Means trouble when |
|---|---|---|
| `redactions` per batch | non-zero most of the day | a flat zero means the scrubber is not running, or the labels do not match the UI language |
| `dropped.excluded` | rare | a high value means extensions are running a stale list; check the refresh |
| `dropped.credential_url` | a few per login | zero across an SSO shop means the detector is not matching |
| `207` rate | near zero | sustained means the event store is failing writes |
| heartbeat `dropped_count` | occasional | constant on one host means raising that host's budget, or not capturing it |

Log counts, never payloads. An ingest log line containing an event body is a second, unscrubbed copy.

## Tests

```ts
// file: lib/process-mining/ingest.test.ts
import { describe, expect, it } from 'vitest';
import { processIngestBatch, type EmitInput, type IngestDeps } from './ingest';
import type { ConsentRecord } from './types';

const NOW = Date.parse('2026-05-02T12:00:00Z');
const consent: ConsentRecord = {
  user_id: 'u1', scope_id: 'org1', consented_at: '2026-05-01T00:00:00Z', revoked_at: null,
  excluded_domains: ['*.mybank.com'], host_allowlist: [],
};
const ev = (over: Record<string, unknown> = {}) => ({
  schema_version: 1, t: '2026-05-02T11:59:00Z', action: 'click', session_id: 'sess-0001', step_index: 0,
  tab: { url: 'https://admin.shop.example/orders' }, ...over,
});
function deps(over: Partial<IngestDeps> = {}) {
  const emitted: EmitInput[] = [];
  const d: IngestDeps = {
    loadConsent: async () => consent,
    emit: async (e) => (emitted.push(e), true),
    now: () => NOW, ...over,
  };
  return { d, emitted };
}
const caller = { userId: 'u1', scopeId: 'org1' };

describe('processIngestBatch', () => {
  it('403s without consent, after revoke, and emits nothing', async () => {
    for (const row of [null, { ...consent, revoked_at: '2026-05-02T00:00:00Z' }]) {
      const { d, emitted } = deps({ loadConsent: async () => row });
      expect(await processIngestBatch({ events: [ev()] }, caller, d)).toMatchObject({ status: 403 });
      expect(emitted).toEqual([]);
    }
  });
  it('asks for consent in the CALLER scope', async () => {
    const seen: string[][] = [];
    const { d } = deps({ loadConsent: async (u, s) => (seen.push([u, s]), consent) });
    await processIngestBatch({ events: [ev()] }, caller, d);
    expect(seen).toEqual([['u1', 'org1']]);
  });
  it('rejects an empty or oversized batch outright', async () => {
    const { d } = deps();
    expect(await processIngestBatch({ events: [] }, caller, d)).toMatchObject({ status: 400 });
    expect(await processIngestBatch({ events: Array(201).fill(ev()) }, caller, d)).toMatchObject({ status: 400 });
    expect(await processIngestBatch('nope', caller, d)).toMatchObject({ status: 400 });
  });
  it('one bad event does not sink the batch', async () => {
    const { d, emitted } = deps();
    const r = await processIngestBatch({ events: [ev(), { junk: true }, ev({ step_index: 1 })] }, caller, d);
    expect(r).toMatchObject({ status: 200, accepted: 2, dropped: { invalid: 1 } });
    expect(emitted.map((e) => e.idempotencyKey)).toEqual(['sess-0001:0', 'sess-0001:1']);
  });
  it('drops excluded hosts server-side: tab, iframe origin and API endpoint', async () => {
    const { d, emitted } = deps();
    const r = await processIngestBatch({ events: [
      ev({ tab: { url: 'https://login.mybank.com/x' } }),
      ev({ step_index: 1, frame_origin: 'https://mybank.com' }),
      ev({ step_index: 2, action: 'network', api_context: { endpoint: 'https://api.mybank.com/v1', method: 'GET', status: 200 } }),
      ev({ step_index: 3 }),
    ] }, caller, d);
    expect(r).toMatchObject({ accepted: 1, dropped: { excluded: 3 } });
    expect(emitted).toHaveLength(1);
  });
  it('drops stale and future-dated events', async () => {
    const { d } = deps();
    const r = await processIngestBatch({ events: [
      ev({ t: '2026-04-20T00:00:00Z' }), ev({ t: '2026-05-02T13:00:00Z', step_index: 1 }),
    ] }, caller, d);
    expect(r).toMatchObject({ accepted: 0, dropped: { stale: 2 } });
  });
  it('stores the SCRUBBED payload, never the raw one', async () => {
    const { d, emitted } = deps();
    await processIngestBatch({ events: [ev({ action: 'input', value: 'card 4111111111111111', element: { label: 'Note' } })] }, caller, d);
    expect(emitted[0]?.payload.value).toBe('card [CARD]');
    expect(emitted[0]?.type).toBe('process_mining.input.changed');
  });
  it('reports failed writes as retryable instead of claiming success', async () => {
    let n = 0;
    const { d } = deps({ emit: async () => n++ !== 1 });
    const r = await processIngestBatch({ events: [ev(), ev({ step_index: 1 }), ev({ step_index: 2 })] }, caller, d);
    expect(r).toMatchObject({ status: 207, accepted: 2, retry: [1] });
  });
});
```

```ts
// file: lib/process-mining/ingest-route.test.ts
import { describe, expect, it } from 'vitest';
import { handleIngest, type IngestHost } from './ingest-route';

const consent = { user_id: 'u1', scope_id: 'org1', consented_at: '2026-05-01T00:00:00Z', revoked_at: null, excluded_domains: [], host_allowlist: [] };
const host = (scopes: string[] | null): IngestHost => ({
  authenticate: async () => (scopes ? { userId: 'u1', scopeId: 'org1', scopes } : null),
  deps: { loadConsent: async () => consent, emit: async () => true, now: () => Date.parse('2026-05-02T12:00:00Z') },
});
const post = (body: string) => new Request('https://x.test/ingest', { method: 'POST', body });
const event = { schema_version: 1, t: '2026-05-02T11:59:00Z', action: 'click', session_id: 'sess-0001', step_index: 0 };

describe('handleIngest', () => {
  it('401 without a key, 403 for a key lacking the emit scope', async () => {
    expect((await handleIngest(post('{}'), host(null))).status).toBe(401);
    expect((await handleIngest(post('{}'), host(['orders:read']))).status).toBe(403);
  });
  it('413 on an oversized body, 400 on junk', async () => {
    expect((await handleIngest(post('x'.repeat(300_000)), host(['process_mining:emit']))).status).toBe(413);
    expect((await handleIngest(post('{not json'), host(['process_mining:emit']))).status).toBe(400);
  });
  it('200 with counts, and is never cacheable', async () => {
    const res = await handleIngest(post(JSON.stringify({ events: [event] })), host(['process_mining:emit']));
    expect(res.status).toBe(200);
    expect(res.headers.get('cache-control')).toBe('no-store');
    expect(await res.json()).toMatchObject({ accepted: 1, retry: [] });
  });
});
```

## Checklist

- [ ] Tenant read from the key row
- [ ] Consent uncached
- [ ] `emit` wrapper returns `false` on failure, never throws
- [ ] Unique idempotency key in the event store
- [ ] Rate limit per key in front of the route
- [ ] Counts logged; payloads never
