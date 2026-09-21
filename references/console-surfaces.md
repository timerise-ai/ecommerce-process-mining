# Employee and manager pages

The only slice the earlier implementation ran in production **[P]**, and the part worth shipping first,
because it delivers value with no extension at all.

## Employee page: "My routine"

Signed-in only; no capability. Two groups, privacy on top.

```
+- Privacy -----------------------------------------------+
| what is captured / never captured / who sees it  (copy)  |
| [ toggle ] Capture consent      state: server-confirmed  |
| Excluded domains  [ textarea ]  errors shown per line    |
| link: install and connect help                           |
+- Log work the browser cannot see ------------------------+
| Category [v]  Title ________  Notes (optional) ________  |
| [>] Add detail: kind, frequency, minutes, tool, outcome  |
| [ Add entry ]                                            |
+- Your entries -------------------------------------------+
| category, date, title, notes           [edit] [delete]   |
| [ Load more ]      keyset pagination, not a silent cap   |
+----------------------------------------------------------+
```

States to build: no categories (tell them who can add some), empty list, saving, per-field validation errors,
save failed.

### The consent toggle must not lie **[A]**

This fixes a live defect. The earlier implementation flipped the switch optimistically, then called the
server. On failure it showed an error toast and **left the switch where the user put it** **[P]**. An employee
turns capture off, the save fails, the switch says Off, the server still says On. For this one control, the
wrong state is worse than a spinner.

```ts
// file: lib/process-mining/use-confirmed-toggle.ts
// A toggle for settings where showing the wrong state is worse than showing a
// spinner. The switch moves only to what the server confirmed.

import { useCallback, useRef, useState } from 'react';

export interface ConfirmedToggle {
  /** What the server last confirmed. Render THIS, never the requested value. */
  value: boolean;
  pending: boolean;
  error: string | null;
  request(next: boolean): Promise<void>;
}

export function useConfirmedToggle(
  initial: boolean,
  save: (next: boolean) => Promise<{ ok: true } | { ok: false; error: string }>,
): ConfirmedToggle {
  const [value, setValue] = useState(initial);
  const [pending, setPending] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const inFlight = useRef(false);

  const request = useCallback(
    async (next: boolean): Promise<void> => {
      if (inFlight.current) return; // a second click mid-save would race the first
      inFlight.current = true;
      setPending(true);
      setError(null);
      try {
        const res = await save(next);
        if (res.ok) setValue(next);
        else setError(res.error);
      } catch (e) {
        setError(e instanceof Error ? e.message : 'save failed');
      } finally {
        inFlight.current = false;
        setPending(false);
      }
    },
    [save],
  );

  return { value, pending, error, request };
}
```

Render `value`; disable the control while `pending`; show `error` inline beside it, not only in a toast that
disappears.

### Server actions

| Action | Rules |
|---|---|
| `setConsent(enabled)` | Tenant = **the session's tenant**, the same source the RLS check uses. Grant = upsert on `(user_id, scope_id)` clearing `revoked_at`. Revoke = update setting `revoked_at`; **if no row was updated, say so**, and do not record a revocation that did not happen |
| `saveExclusions(raw)` | `parseExcludedDomains`; on any error return all the line errors and write nothing |
| `addEntry(input)` | `routineEntryInput.parse`; tenant from session; the DB trigger rejects foreign or retired categories |
| `updateEntry(id, input)` | Owner only, under RLS. The earlier implementation had no edit, so delete-and-retype meant hasty entries stayed hasty **[P]** |
| `deleteEntry(id)` | Check the affected-row count; "deleted" for zero rows is a lie **[P]** |

Two defects in the earlier `setConsent` to avoid **[P]**. It re-derived the tenant by joining user to store to
organisation while the page and the RLS policy used the session claim: two sources for one fact, which
disagree for anyone whose active tenant is not their home one. And its revoke path updated zero rows silently
while still writing a "consent revoked" audit event.

### Entry fields

| Field | Required | Why it exists |
|---|---|---|
| category, title | yes **[P]** | the five-second entry; never add a third required field |
| notes | no **[P]** | |
| `entry_kind`, one of task, blocker, decision, tool_gap | default task **[D]** | blockers cap what automation can reach; they sort first |
| `frequency`, `duration_minutes` | no **[D]** | without them nothing can be costed or ranked |
| `tool_or_system` | no **[D]** | autocomplete from existing values; see `normalizeTool` |
| `outcome` | no **[D]** | the verb-and-result phrasing the narrator needs: "confirms the invoice match, then books it in the ERP" |
| `linked_event_ids` | no **[D]** | loose link to a captured session; offered only when the person has recent events |

Keep the detail fields behind "Add detail". The default form stays at two inputs.

```ts
// file: lib/process-mining/routine.ts
// Gap-capture form: validation for the work the extension cannot see, and the
// ranking the manager view sorts by.

import { z } from 'zod';
import type { RoutineEntry, RoutineFrequency } from './types';

export const routineEntryInput = z.object({
  category_id: z.string().uuid(),
  title: z.string().trim().min(1).max(280),
  notes: z.string().trim().max(2000).nullable().default(null),
  frequency: z.enum(['hourly', 'daily', 'weekly', 'monthly', 'one_off', 'on_demand']).nullable().default(null),
  duration_minutes: z.number().int().min(1).max(1440).nullable().default(null),
  entry_kind: z.enum(['task', 'blocker', 'decision', 'tool_gap']).default('task'),
  tool_or_system: z.string().trim().min(1).max(80).nullable().default(null),
  outcome: z.string().trim().max(280).nullable().default(null),
});
export type RoutineEntryInput = z.infer<typeof routineEntryInput>;

/** "excel", " Excel ", "EXCEL" -> one chip. Without this the tool cluster splinters. */
export function normalizeTool(raw: string, known: readonly string[]): string {
  const t = raw.trim().replace(/\s+/g, ' ');
  return known.find((k) => k.toLowerCase() === t.toLowerCase()) ?? t;
}

// Working-year occurrences. 230 working days, 8 h, 46 weeks, 11 months.
const OCCURRENCES_PER_YEAR: Record<RoutineFrequency, number | null> = {
  hourly: 230 * 8,
  daily: 230,
  weekly: 46,
  monthly: 11,
  one_off: 1,
  on_demand: null, // unknowable from the form; never guessed
};

/** Hours a year this entry costs one person. null = not enough detail to say. */
export function annualHours(e: Pick<RoutineEntry, 'frequency' | 'duration_minutes'>): number | null {
  if (e.frequency === null || e.duration_minutes === null) return null;
  const n = OCCURRENCES_PER_YEAR[e.frequency];
  return n === null ? null : Math.round(((n * e.duration_minutes) / 60) * 10) / 10;
}

export interface RankedEntry<T> {
  entry: T;
  annual_hours: number | null;
}

/**
 * Blockers first (they cap what automation can reach), then by cost, with
 * un-costed rows last rather than dropped, because an entry nobody sized is a prompt
 * to ask, not something to hide.
 */
export function rankForAutomation<
  T extends Pick<RoutineEntry, 'frequency' | 'duration_minutes' | 'entry_kind'>,
>(entries: readonly T[]): RankedEntry<T>[] {
  return entries
    .map((entry) => ({ entry, annual_hours: annualHours(entry) }))
    .sort((a, b) => {
      const ab = a.entry.entry_kind === 'blocker' ? 0 : 1;
      const bb = b.entry.entry_kind === 'blocker' ? 0 : 1;
      if (ab !== bb) return ab - bb;
      return (b.annual_hours ?? -1) - (a.annual_hours ?? -1);
    });
}
```

## Manager page **[P]** stub, **[A]** corrections

Gate: `process_mining:read`. Sections: enrollment, automation candidates, recent entries, pipeline health.

| Section | Source | Rule |
|---|---|---|
| Enrollment | `pm_enrollment_stats(headcount)` | show "n of N", or "not shown for small teams" when `suppressed` |
| Automation candidates | team entries through `rankForAutomation` | show annual hours; un-costed rows last with a "needs sizing" tag |
| By tool | group on `lower(tool_or_system)` | where integration effort pays back |
| Recent entries | team entries, newest first, paginated | format dates in the **viewer's** locale and zone |
| Pipeline health | ingest counters ([ingest.md](ingest.md)) | last event received, redactions, dead letters |

What the earlier implementation got wrong here **[P]**: it counted consent by fetching rows and taking
`.length`, which is capped by the API's default page size, so the number stops growing at the cap with no
sign; it left that query unscoped, so an all-tenant admin saw more consents than employees; and it formatted
dates with the server's locale.

Never build: a per-person activity timeline for managers, a leaderboard, an "employees who have not opted in"
list. Each converts a documentation tool into monitoring, and each is one query away, which is why the
database, not the page, withholds the rows.

## Help page **[D]**: no sidebar entry, linked from the employee page

In this order: install, connect, turn on consent, set excluded domains (suggest the bank, health services and
personal email), allow sites, pause, what is and is not captured, where the data goes and for how long, pause
against revoke against disconnect, and troubleshooting.

The troubleshooting tree has four branches. *Not connected*: connect again. *Authorization came back with an
error*: the wrong console domain, or the extension is not on the allowlist. *A tab is silent*: excluded
domains first, then the pause badge, then the allowlist. *No events showing*: consent, then the connection.
*No screenshots*: the deployment's screenshot mode.

Do not promise "one click on the toolbar icon pauses" unless the extension has no popup. See
[extension.md](extension.md).

## Strings

Every string on these pages is a key, in every locale. The earlier implementation hard-coded English
throughout a fully internationalised app **[P]**. Consent copy is the last text that should be English only.

## Tests

```ts
// file: lib/process-mining/routine.test.ts
import { describe, expect, it } from 'vitest';
import { annualHours, normalizeTool, rankForAutomation, routineEntryInput } from './routine';

describe('routine entries', () => {
  it('minimum form is category + title; everything else defaults', () => {
    const r = routineEntryInput.parse({ category_id: '6f1c7e3a-6a54-4d0e-9d3e-2a1b0c9d8e7f', title: '  Call carrier about lost parcel ' });
    expect(r).toMatchObject({ title: 'Call carrier about lost parcel', notes: null, frequency: null, entry_kind: 'task' });
  });
  it('rejects an empty title and a 25-hour task', () => {
    const id = '6f1c7e3a-6a54-4d0e-9d3e-2a1b0c9d8e7f';
    expect(routineEntryInput.safeParse({ category_id: id, title: '   ' }).success).toBe(false);
    expect(routineEntryInput.safeParse({ category_id: id, title: 'x', duration_minutes: 1500 }).success).toBe(false);
  });
  it('folds tool spellings onto the known chip', () => {
    expect(normalizeTool('  excel ', ['Excel', 'SAP'])).toBe('Excel');
    expect(normalizeTool('Carrier  portal', ['Excel'])).toBe('Carrier portal');
  });
  it('costs an entry per working year, and refuses to guess', () => {
    expect(annualHours({ frequency: 'daily', duration_minutes: 15 })).toBe(57.5);
    expect(annualHours({ frequency: 'weekly', duration_minutes: 120 })).toBe(92);
    expect(annualHours({ frequency: 'on_demand', duration_minutes: 30 })).toBeNull();
    expect(annualHours({ frequency: 'daily', duration_minutes: null })).toBeNull();
  });
  it('ranks blockers first, then by cost, un-costed last, without mutating', () => {
    const rows = [
      { id: 'cheap', frequency: 'monthly', duration_minutes: 10, entry_kind: 'task' },
      { id: 'unknown', frequency: null, duration_minutes: null, entry_kind: 'task' },
      { id: 'dear', frequency: 'daily', duration_minutes: 45, entry_kind: 'task' },
      { id: 'block', frequency: 'weekly', duration_minutes: 5, entry_kind: 'blocker' },
    ] as const;
    const before = JSON.stringify(rows);
    expect(rankForAutomation(rows).map((r) => r.entry.id)).toEqual(['block', 'dear', 'cheap', 'unknown']);
    expect(JSON.stringify(rows)).toBe(before);
  });
});
```

```ts
// file: lib/process-mining/use-confirmed-toggle.test.ts
// @vitest-environment happy-dom
import { act, renderHook } from '@testing-library/react';
import { describe, expect, it } from 'vitest';
import { useConfirmedToggle } from './use-confirmed-toggle';

describe('useConfirmedToggle', () => {
  it('moves only after the server confirms', async () => {
    let release: (v: { ok: true }) => void = () => {};
    const save = () => new Promise<{ ok: true }>((r) => (release = r));
    const { result } = renderHook(() => useConfirmedToggle(true, save));
    let done: Promise<void> = Promise.resolve();
    act(() => { done = result.current.request(false); });
    expect(result.current).toMatchObject({ value: true, pending: true });
    await act(async () => { release({ ok: true }); await done; });
    expect(result.current).toMatchObject({ value: false, pending: false, error: null });
  });
  it('a failed revoke leaves the switch ON and says why', async () => {
    const { result } = renderHook(() => useConfirmedToggle(true, async () => ({ ok: false as const, error: 'offline' })));
    await act(() => result.current.request(false));
    expect(result.current).toMatchObject({ value: true, pending: false, error: 'offline' });
  });
  it('a thrown save is a failure, not a success', async () => {
    const { result } = renderHook(() => useConfirmedToggle(false, async () => { throw new Error('boom'); }));
    await act(() => result.current.request(true));
    expect(result.current).toMatchObject({ value: false, error: 'boom' });
  });
  it('ignores a second click while saving', async () => {
    let calls = 0; let release: (v: { ok: true }) => void = () => {};
    const save = () => (calls++, new Promise<{ ok: true }>((r) => (release = r)));
    const { result } = renderHook(() => useConfirmedToggle(false, save));
    let first: Promise<void> = Promise.resolve();
    act(() => { first = result.current.request(true); void result.current.request(false); });
    await act(async () => { release({ ok: true }); await first; });
    expect(calls).toBe(1);
    expect(result.current.value).toBe(true);
  });
});
```

## Checklist

- [ ] Toggle renders server-confirmed state; failure is visible and leaves it unchanged
- [ ] One source for the tenant across page, action and RLS
- [ ] Zero-row revoke and zero-row delete reported as such
- [ ] Entries paginate; nothing is silently capped
- [ ] Manager page has no route to individual consent
- [ ] All strings keyed and translated
