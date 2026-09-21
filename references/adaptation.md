# Adaptation

Everywhere this module touches its host. Fill the right-hand column before writing code. Nothing crosses a
seam implicitly.

## The seam contract

| Seam | The skill ships | The host supplies | A worked example |
|---|---|---|---|
| **Domain entities** | Canonical names (below) | Its vocabulary | "routine", "my routine", "process mining" |
| **Tenant scope** | `scope_id`, always server-derived | org / workspace / company / legal entity | `organization_id` from a JWT claim |
| **Session auth** | `pm_current_user()` in SQL; "who is signed in" in pages | Supabase, Clerk, NextAuth, custom | Supabase SSR + a request proxy |
| **Capability check** | Three capability strings | Its RBAC, or a role check | Data-driven RBAC table + `requireCapability()` |
| **API keys** | `IngestHost.authenticate`, `ExchangeDeps.mintKey` | Its key table: hashed secret, scopes, owner, tenant, revoke | An `api_keys` table with tiered prefixes |
| **Event store** | `IngestDeps.emit` + a reference `pm_events` table | Its event bus / audit log, if it has one | A unified `events` table with a `process_mining` category |
| **Data access** | Postgres schema; Firestore notes | Its ORM or SDK | Supabase client under RLS |
| **Object storage** | A key format and an upload/delete contract | Supabase Storage, Blob, S3 | Supabase Storage (planned) |
| **AI provider** | Stage contracts and routing rule | Its gateway and model ids | Vercel AI Gateway (planned) |
| **Background work** | What must run and how often | Its cron / queue | Vercel Cron |
| **UI primitives** | Structure, states, disabled logic | Buttons, toggles, toasts, tables | Carbon |
| **Strings** | Keys | Its i18n files, every locale | next-intl, which the module itself never used: see [provenance.md](provenance.md) |
| **Validation** | zod schemas (compile under zod 3 and 4) | zod, or a port to valibot/yup | zod |
| **Tests** | 76 vitest cases, 24 SQL checks | Its runner | none |

## Capabilities

| Capability | Held by | Gates |
|---|---|---|
| `process_mining:emit` | **API keys only, never a human role** | `POST /ingest` |
| `process_mining:read` | managers, analysts | team entries, aggregate stats, generated SOPs |
| `process_mining:configure` | admins | category CRUD, redaction patterns, screenshot mode |

`emit` staying off every human role is load-bearing: a session cookie must never be able to post capture
events, so a malicious page in the employee's browser cannot forge their activity. **[P]** The earlier
implementation enforced this in its role seed.

No capability gates the employee's own page. Being signed in is enough to log your own work and manage your
own consent. **[P]**

## Canonical names and the rename table

Decide the rename once, before generating, and apply it to types, tables, routes, components, comments and
strings together.

| Canonical | Meaning | Rename to |
|---|---|---|
| `scope_id` | the tenant boundary | `organization_id`, `workspace_id`, and so on |
| `pm_consent` | current consent row | |
| `pm_consent_ledger` | append-only consent history | |
| `pm_routine_entries` / "routine entry" | one unit of unseen work | "task note", "offline task", and so on |
| `pm_routine_categories` | buckets for entries | |
| `pm_extension_grants` | one-time connect codes | |
| `pm_events` | reference event sink | drop if the host has an event log |
| `ActionEvent` | one captured step | |
| "My routine" | the employee page | the one name the user must confirm |

Do **not** rename `dom_confidence`, `session_id`, `step_index`, `bounding_box`, `viewport`, `code_challenge`,
`code_verifier`, `redirect_uri`, `state`. They are wire-format or protocol terms; the extension and the AI
stages both depend on them.

## Background work

| Job | Cadence | Why |
|---|---|---|
| Delete expired `pm_extension_grants` | daily | Codes are dead after 60 s; this is housekeeping, not security |
| Archive then prune raw events | per retention policy | The hot tier is short, a day in the earlier implementation. **Never prune what has not been archived**: gate deletes on the export cursor |
| Narrate new events (AI stage 1) | every few minutes or nightly | Batch is far cheaper than per-event |
| Re-mine branches, regenerate SOPs | weekly, or on demand | |
| Delete screenshots whose events are gone | daily | Orphaned objects are billed forever |

## Placement

All TypeScript modules sit in **one folder** and import each other relatively (`./types`, `./exclusions`). The
server imports that folder; the extension's bundler resolves the same files. Do not fork them into two copies:
exclusion matching and scrubbing must be byte-identical on both sides, and a shared folder is the only thing
that keeps them so.

```
lib/process-mining/        types, exclusions, scrub, ingest, ingest-route, pkce, routine,
                           use-confirmed-toggle, and the tests beside each module
                           capture-control, label, screenshot   (extension-only, harmless on the server)
db/process-mining/         schema.sql, verify.sql
extension/                 own package.json and bundler; imports ../lib/process-mining/*
```

If the host forbids the extension importing from the app tree, publish the folder as a tiny internal package
instead. Copying is the one option that is not acceptable.

## Host probe additions

Beyond the usual dependency and convention probe, find out:

- **Does an API-key system exist?** Need: secret shown once, stored hashed, per-key scopes, bound to a user
  *and* a tenant, revocable by id. If not, that is a prerequisite project, not part of this one.
- **Does the host have an event log with retention and export?** If yes, `emit` writes there and `pm_events`
  is dropped.
- **Can one user belong to several tenants?** If yes, the composite consent key is mandatory, and the key's
  tenant, not the user's "primary" one, scopes every ingest.
- **Which locales?** Consent copy is legal copy. It is translated before launch, by a person.
- **Who is the DPO / works-council contact?** Find out in week one, not at go-live.

## Checklist

- [ ] Seam table filled; rename confirmed with the user
- [ ] `emit` capability granted to no human role
- [ ] Shared module folder imported by both app and extension, with no copies
- [ ] Background jobs named with their owner
- [ ] Multi-tenant membership answered; consent key shaped accordingly
- [ ] Maturity tags ([P]/[D]/[A]) preserved in anything written from this skill
