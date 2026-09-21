# The extension: **[A]** logic, **[D]** architecture, and nothing here has run in a browser

Manifest V3. It reads the page's structure, meaning which control, what label and what happened next, instead
of recording pixels.

| | Video + OCR | DOM events |
|---|---|---|
| Cost | GPU inference per frame | text processing |
| Scope | the whole screen | tabs the employee allowed |
| Searchable | after visual indexing | natively |
| Hidden failures | invisible | request status codes |

**[D]** The earlier design claimed 95 to 98 per cent field-read accuracy for DOM capture with a screenshot
fallback, against 70 to 90 per cent for DOM alone on enterprise apps. **Those figures were never measured.**
Treat accuracy as an open question your pilot answers.

## Layout **[D]**

```
extension/
  manifest.json
  src/background.ts   service worker: pause state, badge, exclusions cache, queue, screenshots
  src/content.ts      listeners, label resolution, value gate, budget, session rotation
  src/popup/          connect, pause, pending count + discard, per-host allowlist
  src/auth.ts         connect flow; see extension-auth.md
  _locales/<lang>/messages.json
```

Own `package.json`, own bundler (esbuild is enough), not part of the web app's build. It imports the shared
module folder; see [adaptation.md](adaptation.md).

## Manifest **[D]**, with corrections **[A]**, not loaded into Chrome

```jsonc
// file: extension/manifest.json
{
  "manifest_version": 3,
  "name": "__MSG_name__",
  "default_locale": "en",
  "key": "<public key, which pins the extension id>",
  "permissions": ["storage", "identity", "alarms", "webRequest", "scripting"],
  "optional_host_permissions": ["https://*/*"],
  "host_permissions": [],
  "background": { "service_worker": "background.js", "type": "module" },
  "action": { "default_popup": "popup.html" },
  "commands": { "toggle-pause": { "description": "Pause or resume capture" } }
}
```

- **`host_permissions` empty at install.** The prompt never says "read all your data on all websites". Each
  host is requested with `chrome.permissions.request` from a click in the popup; a person who allows nothing
  grants nothing.
- **No static `content_scripts`.** They would need install-time host access. Register per granted host with
  `chrome.scripting.registerContentScripts` (`allFrames: true`). **[A]**
- **`default_popup` and `action.onClicked` are mutually exclusive.** The earlier design promised "one click on
  the toolbar icon pauses" *and* a popup **[D]**, which Manifest V3 does not allow. Shipped: the popup opens
  with Pause as its first and largest control, and one-action pause is the `toggle-pause` command. Help text
  must describe that, not the single-click icon. **[A]**
- **`chrome.debugger`** (response bodies) is left out. It shows a persistent "is debugging this browser"
  banner, detaches when DevTools opens, and lengthens store review. Status codes carry the exception-handling
  signal.

## Control logic: no `chrome.*` inside, and fully tested

```ts
// file: lib/process-mining/capture-control.ts
// Extension-side control logic with no chrome.* calls in it: pause state,
// capture budget, offline queue bounds, session rotation. The service worker
// and content script feed these with timestamps and persist what they return.

// ---------- pause ----------

export type PauseState =
  | { kind: 'active' }
  | { kind: 'paused-sticky' }
  | { kind: 'paused-until'; untilMs: number };

export type PauseCommand =
  | { type: 'toggle' }
  | { type: 'pause-for'; ms: number; now: number }
  | { type: 'pause' }
  | { type: 'resume' };

export function reducePause(state: PauseState, cmd: PauseCommand): PauseState {
  switch (cmd.type) {
    case 'toggle':
      return state.kind === 'active' ? { kind: 'paused-sticky' } : { kind: 'active' };
    case 'pause':
      return { kind: 'paused-sticky' };
    case 'pause-for':
      return { kind: 'paused-until', untilMs: cmd.now + Math.max(0, cmd.ms) };
    case 'resume':
      return { kind: 'active' };
  }
}

/**
 * Derived from the clock, not from "did the wake-up alarm fire". A timed pause
 * ends when its time is up even if the alarm was lost with an evicted worker.
 */
export function isCapturing(state: PauseState, now: number): boolean {
  if (state.kind === 'active') return true;
  if (state.kind === 'paused-sticky') return false;
  return now >= state.untilMs;
}

export function badgeText(state: PauseState, now: number): string {
  if (isCapturing(state, now)) return '';
  if (state.kind === 'paused-until') {
    return String(Math.max(1, Math.ceil((state.untilMs - now) / 60_000)));
  }
  return '||';
}

/** Anything unreadable in storage means PAUSED. Never fail open on a privacy switch. */
export function parsePauseState(stored: unknown): PauseState {
  if (stored === undefined || stored === null) return { kind: 'active' };
  if (typeof stored === 'object') {
    const s = stored as { kind?: unknown; untilMs?: unknown };
    if (s.kind === 'active') return { kind: 'active' };
    if (s.kind === 'paused-sticky') return { kind: 'paused-sticky' };
    if (s.kind === 'paused-until' && typeof s.untilMs === 'number') {
      return { kind: 'paused-until', untilMs: s.untilMs };
    }
  }
  return { kind: 'paused-sticky' };
}

// ---------- capture budget (per tab) ----------

export class CaptureBudget {
  private tokens: number;
  private last: number;
  private dropped = 0;
  private lastHeartbeat: number;

  constructor(
    private readonly perSecond: number,
    now: number,
  ) {
    this.tokens = perSecond;
    this.last = now;
    this.lastHeartbeat = now;
  }

  /** true = capture this event; false = over budget, drop it. */
  take(now: number): boolean {
    const elapsed = Math.max(0, now - this.last);
    this.tokens = Math.min(this.perSecond, this.tokens + (elapsed / 1000) * this.perSecond);
    this.last = now;
    if (this.tokens >= 1) {
      this.tokens -= 1;
      return true;
    }
    this.dropped++;
    return false;
  }

  /**
   * At most one heartbeat a minute, and only when something was dropped. It is
   * what tells the AI pipeline the stream was SAMPLED. Without it, a gap reads
   * as "the user did nothing here".
   */
  heartbeat(now: number): { dropped_count: number } | null {
    if (this.dropped === 0 || now - this.lastHeartbeat < 60_000) return null;
    const out = { dropped_count: this.dropped };
    this.dropped = 0;
    this.lastHeartbeat = now;
    return out;
  }
}

// ---------- offline queue ----------

export interface QueuedBatch {
  id: string;
  createdAt: number;
  bytes: number;
  attempts: number;
}

export const QUEUE_MAX_BYTES = 8 * 1024 * 1024; // headroom under storage.local's 10 MB quota
export const QUEUE_MAX_AGE_MS = 7 * 24 * 60 * 60 * 1000;
const BACKOFF_MS = [1_000, 4_000, 16_000, 64_000, 256_000] as const;

/** null = give up on this batch (dead letter). */
export function nextBackoffMs(attempts: number): number | null {
  return BACKOFF_MS[attempts] ?? null;
}

export function enforceQueueBounds<T extends QueuedBatch>(
  queue: readonly T[],
  now: number,
  maxBytes = QUEUE_MAX_BYTES,
  maxAgeMs = QUEUE_MAX_AGE_MS,
): { kept: T[]; evicted: T[] } {
  const fresh = queue.filter((b) => now - b.createdAt <= maxAgeMs);
  const evicted = queue.filter((b) => now - b.createdAt > maxAgeMs);
  const newestFirst = [...fresh].sort((a, b) => b.createdAt - a.createdAt);
  const kept: T[] = [];
  let total = 0;
  for (const b of newestFirst) {
    if (total + b.bytes <= maxBytes) {
      kept.push(b);
      total += b.bytes;
    } else {
      evicted.push(b); // oldest lose: recent activity is what the user can still vouch for
    }
  }
  kept.reverse();
  return { kept, evicted };
}

/** What the extension does with a server response. */
export type IngestOutcome = 'delivered' | 'retry' | 'purge_and_stop' | 'disconnect' | 'dead_letter';

export function classifyIngestResponse(status: number, attempts: number): IngestOutcome {
  if (status === 200 || status === 207) return 'delivered'; // 207: requeue only body.retry indexes
  if (status === 401) return 'disconnect'; // key revoked: drop to "Not connected"
  // Consent is gone. Events captured under it must not be held for a later resend.
  if (status === 403) return 'purge_and_stop';
  if (status === 400 || status === 413) return 'dead_letter'; // resending the same bytes cannot succeed
  return nextBackoffMs(attempts) === null ? 'dead_letter' : 'retry';
}

// ---------- session rotation ----------

export interface SessionState {
  sessionId: string;
  stepIndex: number;
  lastPath: string;
  blurredAt: number | null;
}

export type SessionSignal =
  | { type: 'navigate'; path: string }
  | { type: 'blur'; now: number }
  | { type: 'focus'; now: number }
  | { type: 'resumed-from-pause' };

export const FOCUS_LOSS_ROTATE_MS = 5 * 60 * 1000;

export function startSession(newId: () => string, path: string): SessionState {
  return { sessionId: newId(), stepIndex: 0, lastPath: path, blurredAt: null };
}

export function onSessionSignal(
  s: SessionState,
  sig: SessionSignal,
  newId: () => string,
): SessionState {
  switch (sig.type) {
    case 'navigate':
      // Same path = a query/hash tweak (filters, tabs), not a new screen.
      return sig.path === s.lastPath ? s : startSession(newId, sig.path);
    case 'blur':
      return { ...s, blurredAt: sig.now };
    case 'focus': {
      const away = s.blurredAt === null ? 0 : sig.now - s.blurredAt;
      return away > FOCUS_LOSS_ROTATE_MS
        ? startSession(newId, s.lastPath)
        : { ...s, blurredAt: null };
    }
    case 'resumed-from-pause':
      return startSession(newId, s.lastPath);
  }
}

/** Returns the index for this event and the state to keep. */
export function nextStep(s: SessionState): { stepIndex: number; state: SessionState } {
  return { stepIndex: s.stepIndex, state: { ...s, stepIndex: s.stepIndex + 1 } };
}
```

### Wiring it to Chrome **[A]**, not compiled

```ts
// file: extension/src/background.ts
// The single owner of pause state.
const KEY = 'pause';
async function loadPause(): Promise<PauseState> {
  return parsePauseState((await chrome.storage.local.get(KEY))[KEY]);
}
async function applyPause(cmd: PauseCommand): Promise<void> {
  const next = reducePause(await loadPause(), cmd);
  await chrome.storage.local.set({ [KEY]: next });          // content scripts hear this
  if (next.kind === 'paused-until') chrome.alarms.create('resume', { when: next.untilMs });
  else await chrome.alarms.clear('resume');
  await chrome.action.setBadgeText({ text: badgeText(next, Date.now()) });
}
chrome.commands.onCommand.addListener((c) => { if (c === 'toggle-pause') void applyPause({ type: 'toggle' }); });
chrome.alarms.onAlarm.addListener((a) => { if (a.name === 'resume') void applyPause({ type: 'resume' }); });
```

The content script never asks for that state; it keeps a copy.

```ts
// file: extension/src/content.ts
// A local copy of the pause state, kept current by storage events.
let pause: PauseState = { kind: 'paused-sticky' };          // paused until proven otherwise
void chrome.storage.local.get('pause').then((r) => { pause = parsePauseState(r.pause); });
chrome.storage.onChanged.addListener((c, area) => {
  if (area === 'local' && c.pause) pause = parsePauseState(c.pause.newValue);
});
const capturing = (): boolean => isCapturing(pause, Date.now());
```

## Why it is built this way

**Pause state is persisted and clock-derived.** MV3 workers are evicted after ~30 s idle. A sticky pause held
in a variable lapses on the next wake, and capture silently resumes during the sensitive call the person
paused for. A timed pause driven only by `setTimeout` never ends. So: state in `storage.local`,
`chrome.alarms` for the wake-up, and `isCapturing` decided from the clock so a lost alarm cannot strand it.

**The content script does not ask the worker on each event.** The earlier design described a per-event
`sendMessage` round-trip that "returns synchronously in microseconds" **[D]**. `sendMessage` is asynchronous,
costs milliseconds, and wakes an evicted worker and at 50 events a second that is the extension's whole CPU
budget. Keep a local copy updated by `storage.onChanged`, and **start paused** until the first read lands.
**[A]**

**The budget exists because SPAs are loud.** Enterprise apps emit >1,000 mutations a second. 50 captured
events per second per tab, excess dropped, one heartbeat a minute saying how many, so a gap is read as
"sampled" and not as "idle". **[D]**

**The queue is bounded and drops oldest.** 8 MB (under `storage.local`'s 10 MB; do not request
`unlimitedStorage`, which adds an install warning) and 7 days. On `403 consent_required` it is purged: events
captured under a withdrawn consent are not held for later. **[A]**

**Sessions rotate on path change.** SPAs fire no `load` on route change; hook
`history.pushState`/`replaceState` and `popstate`, or a five-step wizard collapses into one step. Query-only
changes do not rotate, because filters would shred every list page. **[A]**

## Naming the field

```ts
// file: lib/process-mining/label.ts
// Content-script side: name the field the user touched, say how much that name
// can be trusted, and decide whether its value may be read at all.

import type { DomConfidence } from './types';

export interface ResolvedLabel {
  label: string | null;
  confidence: DomConfidence;
}

const text = (s: string | null | undefined): string => (s ?? '').replace(/\s+/g, ' ').trim();
const clip = (s: string): string => (s.length > 120 ? `${s.slice(0, 117)}...` : s);

export function resolveLabel(el: Element): ResolvedLabel {
  // A canvas has no DOM inside it to read. Whatever we find is a guess.
  if (el instanceof HTMLCanvasElement) return { label: null, confidence: 'low' };

  const root = el.getRootNode() as Document | ShadowRoot;
  const byId = (id: string): Element | null =>
    'getElementById' in root ? root.getElementById(id) : null;

  // --- high: an explicit, author-declared association ---
  const labelledBy = el.getAttribute('aria-labelledby');
  if (labelledBy) {
    const joined = text(
      labelledBy
        .split(/\s+/)
        .map((id) => byId(id)?.textContent ?? '')
        .join(' '),
    );
    if (joined) return { label: clip(joined), confidence: 'high' };
  }
  const aria = text(el.getAttribute('aria-label'));
  if (aria) return { label: clip(aria), confidence: 'high' };

  const labels = (el as HTMLInputElement).labels;
  const first = labels?.[0];
  if (first) {
    const t = text(first.textContent);
    if (t) return { label: clip(t), confidence: 'high' };
  }
  if (el instanceof HTMLButtonElement || el instanceof HTMLAnchorElement) {
    const t = text(el.textContent);
    if (t) return { label: clip(t), confidence: 'high' };
  }

  // --- medium: inferred from nearby text ---
  const hint = text(el.getAttribute('placeholder')) || text(el.getAttribute('title'));
  if (hint) return { label: clip(hint), confidence: 'medium' };

  const legend = el.closest('fieldset')?.querySelector('legend');
  if (legend && text(legend.textContent)) {
    return { label: clip(text(legend.textContent)), confidence: 'medium' };
  }
  let prev = el.previousElementSibling;
  for (let hops = 0; prev && hops < 2; hops++, prev = prev.previousElementSibling) {
    const t = text(prev.textContent);
    if (t && t.length <= 60) return { label: clip(t), confidence: 'medium' };
  }
  const name = text(el.getAttribute('name'));
  if (name && !looksGenerated(name)) {
    return { label: name.replace(/[_\-.[\]]+/g, ' ').trim(), confidence: 'medium' };
  }

  // --- low: nothing semantic to anchor on; the screenshot crop carries the load ---
  return { label: null, confidence: 'low' };
}

/** `input#tx_9f3a2c11`, `:r1f:`, `ember1042`: ids that change on the next render. */
export function looksGenerated(id: string): boolean {
  return (
    /\d{4,}/.test(id) ||
    /[0-9a-f]{8,}/i.test(id) ||
    /^(:r|ember|react-|mui-|radix-|headlessui-|__)/i.test(id)
  );
}

export function cssPath(el: Element, maxDepth = 4): string {
  const parts: string[] = [];
  let node: Element | null = el;
  while (node && parts.length < maxDepth) {
    if (node.id && !looksGenerated(node.id)) {
      parts.unshift(`${node.tagName.toLowerCase()}#${CSS.escape(node.id)}`);
      break; // a stable id is already unique; stop climbing
    }
    const tag = node.tagName.toLowerCase();
    const parent: Element | null = node.parentElement;
    const sameTag = parent ? [...parent.children].filter((c) => c.tagName === node!.tagName) : [];
    parts.unshift(sameTag.length > 1 ? `${tag}:nth-of-type(${sameTag.indexOf(node) + 1})` : tag);
    node = parent;
  }
  return parts.join(' > ');
}

const NEVER_READ_AUTOCOMPLETE =
  /\b(cc-(number|csc|exp|exp-month|exp-year|name)|current-password|new-password|one-time-code)\b/;

/**
 * Whether the VALUE of this element may leave the page at all. The scrubber
 * downstream is a net, not a licence: what is never read cannot leak.
 */
export function mayReadValue(el: Element): boolean {
  if (el instanceof HTMLInputElement) {
    if (el.type === 'password' || el.type === 'hidden' || el.type === 'file') return false;
  }
  if (NEVER_READ_AUTOCOMPLETE.test(el.getAttribute('autocomplete') ?? '')) return false;
  // Host-page opt-out, for fields the merchant's own apps want kept dark.
  if (el.closest('[data-pm-ignore]')) return false;
  return true;
}
```

| Confidence | Means | Downstream |
|---|---|---|
| `high` | author-declared: `<label>`, `aria-label`, `aria-labelledby`, button text | cheapest text-only narration |
| `medium` | inferred: placeholder, legend, nearby text, a human-looking `name` | text-only by default |
| `low` | nothing semantic; canvas; generated id | needs the screenshot crop |

`mayReadValue` is the most important function in the extension. What it refuses is never in memory, never
queued, never sent, never scrubbed.

## Where DOM capture is weak **[D]**

| Pattern | Seen in | Effect |
|---|---|---|
| Closed Shadow DOM | Salesforce Lightning, web-component UIs | observers do not pierce closed roots, so `low` |
| Canvas-rendered UI | report viewers, label designers, some PDF viewers | no nodes, so `low` |
| Cross-origin iframes | payment widgets, embedded BI, e-signature | own content script, own `session_id`, `frame_origin` set |
| Bot protection | Akamai, Imperva, DataDome-fronted portals | the page may degrade with the extension active, so test each target and list the incompatible ones |
| Virtualised tables | order and product grids | rows unmount on scroll; capture the click, not the mutation storm |

## Unproven assumptions: settle these in a spike before committing **[A]**

1. **Event-triggered `captureVisibleTab` under runtime-granted hosts.** Documented as needing `<all_urls>` or
   `activeTab`; `activeTab` is granted by a user gesture *on the extension*, not by a click in the page. If a
   granted host permission does not suffice, screenshots need `<all_urls>`, which changes the install prompt
   and the privacy story. **Test first; it decides whether screenshots ship.**
2. **`webRequest` in MV3** is observe-only and needs host access. Verify status codes arrive for `fetch` calls
   on your target apps.
3. **Hash cost.** The earlier design estimated some 50 microseconds per perceptual hash **[D]**, which ignores
   decoding a data-URL PNG into pixels in a worker, through `createImageBitmap` and `OffscreenCanvas`, and
   that takes milliseconds. Measure.
4. **Store review.** `identity` + broad optional hosts + employee-monitoring purpose draws scrutiny: budget
   weeks. Enterprise force-install by policy avoids the public listing.
5. **Chromium only.** Edge runs the same build. Firefox and Safari differ materially.

## Checklist

- [ ] Spike items 1 to 3 answered on the real target apps
- [ ] Empty install-time host permissions; per-host runtime grants
- [ ] Pause persisted, alarm-backed, clock-derived; content script starts paused
- [ ] `mayReadValue` on every value read, no exceptions
- [ ] Exclusions checked before listeners attach
- [ ] Queue bounded; purged on `403 consent_required`
- [ ] History API hooked for session rotation
- [ ] Popup strings in `_locales`, every language
