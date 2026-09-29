# Event-triggered screenshots: **[A]** logic, **[D]** model, not run in a browser

A screenshot is attached to a *significant event*, not taken on a timer. The aim is to give the AI pixels for
the minority of events whose DOM read is unreliable, at a few percent of the cost of recording.

**Decide the mode before anything else.**

| Mode | Uploads | Use when |
|---|---|---|
| `event_triggered` | PNG/WebP + bounding box | internal tools showing no third-party personal data |
| `metadata_only` | bounding box, viewport, hashes, and **no image** | buyer data on screen; **the e-commerce default [A]** |
| `disabled` | nothing | legal review says so |

The mode is a setting read by the gate, not a build option. With screenshots off, `screenshot.ts` and its
suite still ship unchanged, so the module and its cases stay verified if the mode is ever changed. **[A]**

**The scrubber never sees pixels.** A screenshot of an order page contains the buyer's name and address in a
form no regex touches, and stage 1 sends the crop to a third-party vision model. `metadata_only` trades
accuracy on low-confidence fields for the guarantee that no image leaves the browser. **[D]**

## What earns a screenshot **[D]**

| Trigger | Rule |
|---|---|
| Click on a button, link or submit | always (throttled) |
| Field commit (`change` / blur) | once per field per session |
| Response status of 400 or more | always: error states are the spine of exception handling |
| Route change | always (throttled) |
| Mutation burst (modal, inline expansion) | once per burst, debounced |
| Enter/Tab inside a form | once per form per minute |
| Typing | **never** |

Throttle: one capture per 600 ms per tab, inside Chrome's cap of roughly two a second. A throttled event
points at the previous screenshot and records how stale it is (`screenshot_offset_ms`), so the model is never
led to believe an old image shows this step.

Every privacy gate runs first. Paused, no consent, host not allowed or host excluded all mean no capture, and
no reuse of an earlier one either.

## The module

```ts
// file: lib/process-mining/screenshot.ts
// Event-triggered screenshots: which events earn one, how often, and when a new
// capture is a duplicate of the last upload. Pixel access and the capture call
// itself stay in the service worker; this file only decides.

import type { BoundingBox, DomConfidence, Viewport } from './types';

export type ScreenshotTrigger =
  | 'click'
  | 'field-commit' // change/blur, once per field per session
  | 'http-error' // response status >= 400
  | 'navigation'
  | 'mutation-burst'
  | 'form-submit-key'
  | 'typing'; // raw input events: never

export const SCREENSHOT_MIN_GAP_MS = 600; // captureVisibleTab is capped near 2/s
export const DUPLICATE_MAX_BITS = 5;

export function triggerWantsScreenshot(t: ScreenshotTrigger): boolean {
  return t !== 'typing';
}

export type CaptureDecision = { capture: true } | { capture: false; reuseOffsetMs: number | null };

/** Throttled events point at the previous screenshot and say how stale it is. */
export function decideCapture(now: number, lastCaptureAt: number | null): CaptureDecision {
  if (lastCaptureAt === null || now - lastCaptureAt >= SCREENSHOT_MIN_GAP_MS) {
    return { capture: true };
  }
  return { capture: false, reuseOffsetMs: now - lastCaptureAt };
}

/** Box-filter an RGBA bitmap region down to 8x8 luminance samples. */
export function toGray8x8(
  rgba: Uint8ClampedArray,
  width: number,
  height: number,
  region?: BoundingBox,
): number[] {
  const rx = Math.max(0, Math.floor(region?.x ?? 0));
  const ry = Math.max(0, Math.floor(region?.y ?? 0));
  const rw = Math.max(1, Math.min(width - rx, Math.floor(region?.w ?? width)));
  const rh = Math.max(1, Math.min(height - ry, Math.floor(region?.h ?? height)));
  const out: number[] = [];
  for (let cy = 0; cy < 8; cy++) {
    for (let cx = 0; cx < 8; cx++) {
      const x0 = rx + Math.floor((cx * rw) / 8);
      const x1 = Math.max(x0 + 1, rx + Math.floor(((cx + 1) * rw) / 8));
      const y0 = ry + Math.floor((cy * rh) / 8);
      const y1 = Math.max(y0 + 1, ry + Math.floor(((cy + 1) * rh) / 8));
      let sum = 0;
      let n = 0;
      for (let y = y0; y < y1 && y < height; y++) {
        for (let x = x0; x < x1 && x < width; x++) {
          const i = (y * width + x) * 4;
          sum += 0.299 * (rgba[i] ?? 0) + 0.587 * (rgba[i + 1] ?? 0) + 0.114 * (rgba[i + 2] ?? 0);
          n++;
        }
      }
      out.push(n ? sum / n : 0);
    }
  }
  return out;
}

/** 64-bit average hash as 16 hex chars. */
export function averageHash(gray64: readonly number[]): string {
  const mean = gray64.reduce((a, b) => a + b, 0) / 64;
  let hex = '';
  for (let nibble = 0; nibble < 16; nibble++) {
    let v = 0;
    for (let bit = 0; bit < 4; bit++) {
      v = (v << 1) | ((gray64[nibble * 4 + bit] ?? 0) > mean ? 1 : 0);
    }
    hex += v.toString(16);
  }
  return hex;
}

export function hammingHex(a: string, b: string): number {
  if (a.length !== b.length) return Number.POSITIVE_INFINITY;
  let bits = 0;
  for (let i = 0; i < a.length; i++) {
    let x = parseInt(a.charAt(i), 16) ^ parseInt(b.charAt(i), 16);
    while (x) {
      bits += x & 1;
      x >>= 1;
    }
  }
  return bits;
}

export interface ShotFingerprint {
  viewportHash: string;
  /** Hash of just the active element's rectangle; null when there is no element. */
  cropHash: string | null;
}

/**
 * A whole-viewport 8x8 hash cannot see one text field change on a 1440x900
 * page: every cell averages some 20,000 pixels. Deduplicating on it alone reuses
 * the OLD screenshot for exactly the events that need the new pixels: the
 * low-confidence ones, where the model reads the value off the image.
 * So: the crop must match too, and low-confidence events are never deduplicated.
 */
export function isDuplicateShot(
  next: ShotFingerprint,
  lastUploaded: ShotFingerprint | null,
  confidence: DomConfidence | undefined,
): boolean {
  if (!lastUploaded || confidence === 'low') return false;
  if (hammingHex(next.viewportHash, lastUploaded.viewportHash) > DUPLICATE_MAX_BITS) return false;
  if (next.cropHash === null || lastUploaded.cropHash === null) {
    return next.cropHash === lastUploaded.cropHash;
  }
  return hammingHex(next.cropHash, lastUploaded.cropHash) <= DUPLICATE_MAX_BITS;
}

/** CSS-pixel rect -> bitmap-pixel rect, clamped. Screenshots are taken at device resolution. */
export function cropRectInBitmap(box: BoundingBox, viewport: Viewport, pad = 8): BoundingBox {
  const k = viewport.device_pixel_ratio;
  const maxW = viewport.w * k;
  const maxH = viewport.h * k;
  const x = Math.max(0, Math.floor((box.x - pad) * k));
  const y = Math.max(0, Math.floor((box.y - pad) * k));
  return {
    x,
    y,
    w: Math.max(1, Math.min(maxW - x, Math.ceil((box.w + 2 * pad) * k))),
    h: Math.max(1, Math.min(maxH - y, Math.ceil((box.h + 2 * pad) * k))),
  };
}
```

## The dedup trap **[A]**

The earlier design deduplicated on one 8 by 8 average hash of the whole viewport, skipping the upload within 5
bits **[D]**. On a 1440 by 900 page each of the 64 cells averages about 20,000 pixels; a text field filling in
moves no bit. The test in [testing.md](testing.md) shows it: the viewport distance stays at 5 or below while
the field's content changes completely.

The events that *need* fresh pixels are the low-confidence ones, where the model reads the value off the
image. Viewport-only dedup hands them a screenshot taken before the value existed, and the SOP confidently
records the wrong thing.

Shipped: fingerprint = viewport hash **and** a hash of the active element's rectangle; both must match;
low-confidence events are never deduplicated. Expect a lower dedup ratio than the earlier estimate of 50 to 70
per cent. That estimate was unmeasured too.

## Coordinates

`getBoundingClientRect()` is CSS pixels relative to the viewport; the bitmap is device pixels.
`cropRectInBitmap` multiplies by `device_pixel_ratio`, pads, and clamps. Read the rectangle **at capture
time**: after a scroll it is a different rectangle.

`captureVisibleTab` sees the visible viewport only. A field scrolled out of view has a bounding box outside
the image; clamp, and treat a degenerate crop as "no crop".

## Storage seam

| Rule | Why |
|---|---|
| Key: `process-mining/<scope>/<user>/<yyyy-mm-dd>/<session>_<step>.webp` | deletable per person, per day, per tenant |
| Store the **key** in the event, never a URL | URLs expire; keys delete |
| Private bucket; short-lived signed URLs at render time | |
| Upload from the extension straight to storage, through a signed upload URL issued by a key-authenticated route | route handlers should not proxy image bytes |
| Upload is **drop-on-fail**, not retry-forever | the event still stands; a missing image degrades that step to low confidence |
| Delete images when their events are deleted, and on consent revoke if policy says so | orphans are billed forever and outlive the consent they were taken under |
| Re-encode to WebP at quality 85 before upload | 3 to 5 times smaller, with no loss that matters for reading a field **[D]**, unmeasured |

## Checklist

- [ ] Mode chosen per deployment and recorded in the DPIA
- [ ] `captureVisibleTab` permission question settled (see [extension.md](extension.md))
- [ ] Privacy gates run before the throttle and before any reuse
- [ ] Dedup uses viewport **and** crop; never for low confidence
- [ ] Rect read at capture time; crop clamped
- [ ] Storage keys stored; images deleted with their events
