# Data model

Neutral TypeScript shapes first, then Postgres with row-level security, then what changes on a document store.

## Neutral types **[A]** (shapes follow the earlier tables **[P]** and event contract **[D]**)

```ts
// file: lib/process-mining/types.ts
// SDK-free shapes. ISO 8601 strings at every boundary; null, never undefined.

export type Iso = string;

/** How much the label resolver trusts its own read. Drives AI stage-1 routing. */
export type DomConfidence = 'high' | 'medium' | 'low';

export type CaptureAction =
  | 'input'
  | 'click'
  | 'change'
  | 'select'
  | 'network'
  | 'screenshot'
  | 'heartbeat';

export interface BoundingBox {
  x: number;
  y: number;
  w: number;
  h: number;
}

export interface Viewport {
  w: number;
  h: number;
  device_pixel_ratio: number;
}

/** One captured step. Wire format and stored payload are the same object. */
export interface ActionEvent {
  schema_version: 1;
  t: Iso;
  action: CaptureAction;
  session_id: string;
  step_index: number;
  tab?: { url: string; title?: string };
  /** null = top frame. Set for events raised inside a cross-origin iframe. */
  frame_origin?: string | null;
  element?: { selector?: string; label?: string | null; type?: string };
  dom_confidence?: DomConfidence;
  value?: string | null;
  api_context?: { endpoint: string; method: string; status: number; duration_ms?: number };
  screenshot_ref?: string | null;
  /** 0 = taken for this event; >0 = an earlier screenshot reused, this many ms stale. */
  screenshot_offset_ms?: number;
  screenshot_phash?: string;
  bounding_box?: BoundingBox;
  viewport?: Viewport;
  /** heartbeat only: events dropped by the capture budget since the last heartbeat. */
  dropped_count?: number;
}

export interface ConsentRecord {
  user_id: string;
  scope_id: string;
  consented_at: Iso;
  revoked_at: Iso | null;
  excluded_domains: string[];
  host_allowlist: string[];
}

export function isConsentActive(row: Pick<ConsentRecord, 'revoked_at'> | null): boolean {
  return row !== null && row.revoked_at === null;
}

export type RoutineFrequency =
  | 'hourly'
  | 'daily'
  | 'weekly'
  | 'monthly'
  | 'one_off'
  | 'on_demand';

export type RoutineEntryKind = 'task' | 'blocker' | 'decision' | 'tool_gap';

export interface RoutineEntry {
  id: string;
  user_id: string;
  scope_id: string;
  category_id: string;
  title: string;
  notes: string | null;
  frequency: RoutineFrequency | null;
  duration_minutes: number | null;
  entry_kind: RoutineEntryKind;
  tool_or_system: string | null;
  outcome: string | null;
  created_at: Iso;
  updated_at: Iso;
}
```

Why these shapes:

- **`ActionEvent` is both the wire format and the stored payload.** One schema to version. `schema_version`
  exists so events queued by an old extension are migrated or dropped, never misread.
- **Everything but identity and time is optional.** The AI stages tolerate sparse rows; a network event has no
  `element`, a heartbeat has almost nothing.
- **No request or response bodies.** The earlier design allowed request payloads **[D]**; this skill leaves
  them out **[A]**. In a shop, a request body *is* a customer record, and nothing downstream needs it: the
  status code and the endpoint carry the exception-handling signal.
- **`screenshot_offset_ms`** makes a reused screenshot honest: the model is told the image is N ms stale
  rather than led to believe it shows this step.

## Postgres **[A]**, executed against PostgreSQL 18, with all 24 checks in [testing.md](testing.md) passing

Three of these tables ran in the earlier implementation **[P]**: categories, entries and consent. What
differs, and why, is in [provenance.md](provenance.md).

```sql
-- file: db/process-mining/schema.sql
-- Process mining: consent, exclusions, gap-capture entries, extension grants,
-- and a reference event sink. Plain Postgres 14+; no extension required.

-- ===== Host seams: replace the three bodies, keep the signatures. =====
-- Supabase:  select auth.uid()
create or replace function pm_current_user() returns uuid
  language sql stable as $$ select nullif(current_setting('app.user_id', true), '')::uuid $$;
-- The tenant the REQUEST is acting in (JWT claim / session), never a value the client sends.
create or replace function pm_current_scope() returns uuid
  language sql stable as $$ select nullif(current_setting('app.scope_id', true), '')::uuid $$;
create or replace function pm_has_capability(cap text) returns boolean
  language sql stable as $$ select cap = any(string_to_array(coalesce(current_setting('app.caps', true), ''), ',')) $$;

-- ===== Consent: one row per person PER SCOPE =====
create table pm_consent (
  user_id          uuid not null,
  scope_id         uuid not null,
  consented_at     timestamptz not null default now(),
  revoked_at       timestamptz,
  excluded_domains text[] not null default '{}',
  host_allowlist   text[] not null default '{}',
  updated_at       timestamptz not null default now(),
  primary key (user_id, scope_id)
);
create index pm_consent_scope_active_idx on pm_consent (scope_id) where revoked_at is null;

-- Append-only history. The current row answers "may we ingest now?"; the
-- ledger answers "were we allowed to on the 14th?", which is the question a
-- regulator or a works council actually asks.
create table pm_consent_ledger (
  id          bigint generated always as identity primary key,
  user_id     uuid not null,
  scope_id    uuid not null,
  action      text not null check (action in ('granted', 'revoked', 'exclusions_updated')),
  detail      jsonb not null default '{}',
  actor_id    uuid,
  occurred_at timestamptz not null default now()
);
create index pm_consent_ledger_user_idx on pm_consent_ledger (user_id, scope_id, occurred_at desc);

-- Append-only means nobody: not the API role, not a migration, not the owner.
-- RLS alone does not give this: without a policy a DELETE is a silent no-op
-- for API roles and fully permitted for the table owner.
create or replace function pm_refuse_mutation() returns trigger
  language plpgsql set search_path = '' as $$
begin raise exception '% is append-only', tg_table_name using errcode = '42501'; end $$;
create trigger pm_consent_ledger_immutable before update or delete on pm_consent_ledger
  for each row execute function pm_refuse_mutation();
create trigger pm_consent_ledger_no_truncate before truncate on pm_consent_ledger
  for each statement execute function pm_refuse_mutation();

create or replace function pm_consent_audit() returns trigger
  language plpgsql security definer set search_path = '' as $$
begin
  new.updated_at := now();
  if tg_op = 'INSERT' or (old.revoked_at is not null and new.revoked_at is null) then
    insert into public.pm_consent_ledger (user_id, scope_id, action, actor_id)
      values (new.user_id, new.scope_id, 'granted', public.pm_current_user());
  elsif old.revoked_at is null and new.revoked_at is not null then
    insert into public.pm_consent_ledger (user_id, scope_id, action, actor_id)
      values (new.user_id, new.scope_id, 'revoked', public.pm_current_user());
  end if;
  if tg_op = 'UPDATE' and new.excluded_domains is distinct from old.excluded_domains then
    insert into public.pm_consent_ledger (user_id, scope_id, action, detail, actor_id)
      values (new.user_id, new.scope_id, 'exclusions_updated', jsonb_build_object(
        'added',   (select coalesce(jsonb_agg(d), '[]') from unnest(new.excluded_domains) d where d <> all (old.excluded_domains)),
        'removed', (select coalesce(jsonb_agg(d), '[]') from unnest(old.excluded_domains) d where d <> all (new.excluded_domains))
      ), public.pm_current_user());
  end if;
  return new;
end $$;
create trigger pm_consent_audit_trg before insert or update on pm_consent
  for each row execute function pm_consent_audit();

alter table pm_consent enable row level security;
alter table pm_consent_ledger enable row level security;

-- Self only. There is deliberately NO manager/team policy on this table:
-- who opted in is not a manager's business. They get pm_enrollment_stats().
create policy pm_consent_self on pm_consent for all
  using (user_id = pm_current_user())
  with check (user_id = pm_current_user() and scope_id = pm_current_scope());
create policy pm_consent_ledger_self_read on pm_consent_ledger for select
  using (user_id = pm_current_user());

-- Aggregate-only enrollment. Below min_group the count IS the individuals, so
-- it is withheld. Pass headcount in: the employee directory is the host's.
create or replace function pm_enrollment_stats(headcount integer, min_group integer default 5)
  returns table (consented integer, suppressed boolean)
  language plpgsql stable security definer set search_path = '' as $$
declare n integer;
begin
  if not public.pm_has_capability('process_mining:read') then
    raise exception 'forbidden' using errcode = '42501';
  end if;
  select count(*) into n from public.pm_consent c
    where c.scope_id = public.pm_current_scope() and c.revoked_at is null;
  if headcount < min_group or n < min_group or headcount - n < 1 then
    return query select null::integer, true;   -- also hides "everyone opted in"
  else
    return query select n, false;
  end if;
end $$;

-- ===== Gap-capture form =====
create table pm_routine_categories (
  id         uuid primary key default gen_random_uuid(),
  scope_id   uuid not null,
  slug       text not null,
  name       text not null,
  sort_order integer not null default 0,
  deleted_at timestamptz,
  unique (scope_id, slug)
);
create index pm_routine_categories_list_idx on pm_routine_categories (scope_id, sort_order, slug) where deleted_at is null;

create table pm_routine_entries (
  id               uuid primary key default gen_random_uuid(),
  user_id          uuid not null,
  scope_id         uuid not null,
  category_id      uuid not null references pm_routine_categories (id) on delete restrict,
  title            text not null check (length(title) between 1 and 280),
  notes            text check (notes is null or length(notes) <= 2000),
  frequency        text check (frequency in ('hourly','daily','weekly','monthly','one_off','on_demand')),
  duration_minutes integer check (duration_minutes between 1 and 1440),
  entry_kind       text not null default 'task' check (entry_kind in ('task','blocker','decision','tool_gap')),
  tool_or_system   text,
  outcome          text,
  linked_event_ids uuid[],
  created_at       timestamptz not null default now(),
  updated_at       timestamptz not null default now()
);
create index pm_routine_entries_mine_idx on pm_routine_entries (user_id, scope_id, created_at desc, id desc);
create index pm_routine_entries_team_idx on pm_routine_entries (scope_id, created_at desc, id desc);
create index pm_routine_entries_tool_idx on pm_routine_entries (scope_id, lower(tool_or_system)) where tool_or_system is not null;

-- A category from another tenant, or a retired one, is not a valid parent. The
-- FK alone allows both.
create or replace function pm_routine_entry_guard() returns trigger
  language plpgsql set search_path = '' as $$
begin
  new.updated_at := now();
  if tg_op = 'INSERT' or new.category_id is distinct from old.category_id then
    perform 1 from public.pm_routine_categories c
      where c.id = new.category_id and c.scope_id = new.scope_id and c.deleted_at is null;
    if not found then raise exception 'category not available' using errcode = '23514'; end if;
  end if;
  return new;
end $$;
create trigger pm_routine_entry_guard_trg before insert or update on pm_routine_entries
  for each row execute function pm_routine_entry_guard();

alter table pm_routine_categories enable row level security;
alter table pm_routine_entries enable row level security;

create policy pm_categories_read on pm_routine_categories for select
  using (scope_id = pm_current_scope());
create policy pm_categories_write on pm_routine_categories for all
  using (scope_id = pm_current_scope() and pm_has_capability('process_mining:configure'))
  with check (scope_id = pm_current_scope() and pm_has_capability('process_mining:configure'));
create policy pm_entries_self on pm_routine_entries for all
  using (user_id = pm_current_user())
  with check (user_id = pm_current_user() and scope_id = pm_current_scope());
create policy pm_entries_team_read on pm_routine_entries for select
  using (scope_id = pm_current_scope() and pm_has_capability('process_mining:read'));

-- ===== One-time codes for the extension connect flow =====
create table pm_extension_grants (
  code           text primary key,
  user_id        uuid not null,
  scope_id       uuid not null,
  code_challenge text not null check (length(code_challenge) = 43),
  extension_id   text not null check (extension_id ~ '^[a-p]{32}$'),
  expires_at     timestamptz not null default now() + interval '60 seconds',
  created_at     timestamptz not null default now()
);
create index pm_extension_grants_expiry_idx on pm_extension_grants (expires_at);
alter table pm_extension_grants enable row level security;
-- RLS on and NO policy = invisible to every non-owner role. That is the intent:
-- only the privileged server path (which bypasses RLS) ever touches this table.

-- Redeem = delete. One statement, so two racing requests cannot both win.
create or replace function pm_claim_extension_grant(p_code text, p_extension_id text)
  returns table (user_id uuid, scope_id uuid, code_challenge text)
  language sql security definer set search_path = '' as $$
  delete from public.pm_extension_grants g
   where g.code = p_code and g.extension_id = p_extension_id and g.expires_at > now()
  returning g.user_id, g.scope_id, g.code_challenge
$$;
revoke all on function pm_claim_extension_grant(text, text) from public;

-- ===== Reference event sink (skip if the host already has an event log) =====
create table pm_events (
  id              uuid primary key default gen_random_uuid(),
  user_id         uuid not null,
  scope_id        uuid not null,
  event_type      text not null,
  idempotency_key text not null,
  occurred_at     timestamptz not null,
  received_at     timestamptz not null default now(),
  payload         jsonb not null,
  unique (user_id, idempotency_key)
);
create index pm_events_stream_idx on pm_events (scope_id, user_id, occurred_at);
alter table pm_events enable row level security;
create policy pm_events_self_read on pm_events for select using (user_id = pm_current_user());
create policy pm_events_team_read on pm_events for select
  using (scope_id = pm_current_scope() and pm_has_capability('process_mining:read'));
-- No insert policy: writes come only from the ingest route's privileged client,
-- after the key, consent, exclusion and scrub checks have all passed.
```

### The decisions that are easy to undo by accident

| Decision | The tempting alternative | What it breaks |
|---|---|---|
| Consent keyed `(user_id, scope_id)` | `user_id` primary key **[P]** | Staff shared between two entities: granting in B rewrites the row's tenant, A's consent vanishes without a revoke |
| No manager policy on `pm_consent` | A `team_read` policy for managers **[P]** | Any manager lists who opted in, straight from the browser client, whatever the page chooses to render |
| Stats via a definer function with suppression | `select count(*)` under a team policy | Small teams: the number names the people |
| Ledger + immutability trigger | History only in a short-retention event log **[P]** | "Were we allowed to capture on the 14th?" has no answer after the log is pruned |
| Trigger checks category tenant + `deleted_at` | FK only **[P]** | Entries land under retired or foreign categories |
| Grants: RLS on, zero policies | A `using (false)` policy | Same effect; fewer moving parts. Either way, *test it* |
| `unique (user_id, idempotency_key)` on events | Plain inserts | A timed-out batch, retried, is every step twice, and the branch miner sees loops that never happened |

### Traps

- **RLS without a policy turns `DELETE` into a silent no-op, not an error**, and it does nothing at all
  against the table owner. "Append-only" needs the trigger. This was found by the verification run, not by
  reading.
- **`security definer` + `set search_path = ''`**, always, and schema-qualify every name inside.
- **`using` without `with check`** lets a row be updated out of the caller's tenant.
- **`revoke all ... from public`** on the claim function. Functions are executable by everyone by default.
- The three seam functions read `current_setting(..., true)`; the `true` makes a missing setting `NULL`
  instead of an error, and `NULL` fails every policy. Closed by default.

### Supabase wiring **[A]**, not executed

```sql
-- file: db/process-mining/schema.supabase.sql
create or replace function pm_current_user() returns uuid
  language sql stable as $$ select auth.uid() $$;
create or replace function pm_current_scope() returns uuid
  language sql stable as $$ select nullif(auth.jwt() -> 'app_metadata' ->> 'organization_id', '')::uuid $$;
```

Read tenant and capability claims from `app_metadata` only. `user_metadata` is writable by the user.

The host's own SQL (its `pm_has_capability`, an API-key table, helper functions for the routes) goes in a file
of its own, such as `db/process-mining/host.supabase.sql`, applied after this one. Appending to a template
file makes it differ from the block it was verified against. **[A]**

## Document store (Firestore) **[A]**, not executed

| Concern | Postgres | Firestore |
|---|---|---|
| Access posture | RLS is the control; routes are a convenience | **No client rules at all.** Every read and write goes through a route using the Admin SDK. Absence of rules *is* the policy: write that down |
| Consent | `pm_consent (user_id, scope_id)` | `pmConsent/{scopeId}_{userId}` |
| Ledger | table + immutability trigger | `pmConsentLedger/{autoId}`, written in the same `runTransaction` as the consent change; never updated |
| Claim a grant | `DELETE ... RETURNING` | `runTransaction`: `get`, check the expiry and the extension id, then `delete`. Both steps inside the transaction, or two redeemers win |
| Idempotent event | unique constraint | document id = `${userId}_${sessionId}_${stepIndex}`, written with `create()`, where an existing id throws, and that throw is the dedupe |
| Aggregate stats | definer function | `count()` aggregation query in a route; apply the same suppression in code |
| Category guard | trigger | checked in the route before the write |
| Absent values | `null` | write `null` explicitly: equality filters skip missing fields, and `undefined` is rejected |

Raw capture volume is high and append-mostly. On either backend, keep it out of the operational database past
a day or so: archive to a warehouse, mine from there.

## Volume **[D]**: estimates from the earlier design, unmeasured

| Item | Estimate |
|---|---|
| Events per session | ~80 |
| Sessions per person per day | ~5 |
| Unique screenshots per session after dedup | 25 to 35 |
| Screenshot size | 100 to 150 KB as PNG at 1080p; 30 to 50 KB as WebP q85 |
| Storage | 20 to 25 MB per person per day; 500 to 600 GB per 100 people per year |

Treat these as a sizing starting point. Measure in the pilot and replace them.

## Checklist

- [ ] Seam functions replaced with the host's auth; nothing reads a client-supplied tenant
- [ ] `verify.sql` from [testing.md](testing.md) run against the adapted schema
- [ ] Consent key is composite if one person can belong to several tenants
- [ ] No manager-readable path to individual consent rows, checked on the API role and not on the page
- [ ] Retention and archive job in place before capture is switched on
