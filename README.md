# ecommerce-process-mining

[![Agent Skills](https://img.shields.io/badge/Agent_Skills-open_format-059669)](https://agentskills.io)
[![skills.sh](https://img.shields.io/badge/skills.sh-npx_skills_add-059669)](https://www.skills.sh)
[![Claude Code](https://img.shields.io/badge/Claude_Code-compatible-059669)](https://docs.claude.com/en/docs/claude-code/skills)
[![Codex CLI](https://img.shields.io/badge/Codex_CLI-compatible-059669)](https://developers.openai.com/codex/skills)
[![Gemini CLI](https://img.shields.io/badge/Gemini_CLI-compatible-059669)](https://github.com/google-gemini/gemini-cli/blob/main/docs/cli/skills.md)

An [Agent Skill](https://agentskills.io) that teaches an agent to build employee process mining for an
e-commerce back office in a **Next.js App Router** app: a consented Manifest V3 browser extension that reads
DOM events rather than pixels from the admin tools staff already use, a gap-capture form for the work a
browser cannot see, an ingest route that scrubs before it stores, and a batch AI pipeline that turns the
result into per-role SOPs and a ranked automation shortlist.

**In e-commerce the screens are full of other people's data.** The employee consents; the customer whose
order, return or ticket is on screen never did, and never can be asked. That single fact decides the design:
consent is re-read on every batch rather than cached, exclusions run in the extension and again on the
server, values that would have to be scrubbed are never read at all, screenshots default to off because the
scrubber reads JSON and not pixels, and managers get suppressed aggregates instead of rows. Capture is a
privacy system that happens to produce data.

This skill was written by the engineer who has shipped this module. The earlier implementation it was audited
against was the employee process-mining module of a back-office platform, of which the consent record, the
gap-capture form and a manager view ran while the capture path existed as design notes. The templates here
hold the properties such a module has to hold: consent that is per person and per tenant and answered from
the database on every batch, one exclusion matcher imported by both sides, a scrubber that tells a card
number from a barcode, a one-time code that can be redeemed exactly once, a consent history nobody can
rewrite, and an enrollment count that is withheld when the number would name the people. Each is stated as a
rule below and held by the suites in [`references/testing.md`](references/testing.md);
[`references/provenance.md`](references/provenance.md) has the record, and the maturity tags say which parts
have run in production and which have not.

## Install

One command, via the [skills.sh](https://www.skills.sh) CLI, which installs the skill into every
skills-compatible agent it detects, including Claude Code, Codex CLI and Gemini CLI:

```bash
npx skills add timerise-ai/ecommerce-process-mining
```

Name the agents instead with `-a`, for example
`npx skills add timerise-ai/ecommerce-process-mining -a claude-code -a codex`.

### Manual install

Nothing here is Claude-specific: the skill is a plain [Agent Skills](https://agentskills.io) folder,
`SKILL.md` plus markdown references with no file that calls a model, so cloning it into an agent's skills
directory is all an install is. For Claude Code:

```bash
git clone https://github.com/timerise-ai/ecommerce-process-mining.git \
  ~/.claude/skills/ecommerce-process-mining
```

To scope it to a single project instead, clone it into that project's `.claude/skills/` directory. For
another agent, clone into that agent's skills directory, or symlink the Claude Code copy so one `git pull`
updates every agent:

```bash
mkdir -p ~/.agents/skills
ln -s ~/.claude/skills/ecommerce-process-mining ~/.agents/skills/ecommerce-process-mining
```

Update the skill with `git pull` in its directory. The current release is **0.1.4**. See
[CHANGELOG.md](CHANGELOG.md). The [skills index](https://github.com/timerise-ai/skills) lists the other
Timerise Skills and how to install them all at once.

## Activation

The skill activates automatically when a task matches its description: documenting how order, return,
listing, supplier or support work is really done; building or auditing an employee activity-capture
extension, its consent model, its PII scrubbing or its PKCE connect flow; generating SOPs from captured
activity; or ranking manual work for automation. Invoke it explicitly with `/ecommerce-process-mining` in
Claude Code, `$ecommerce-process-mining` in Codex CLI, or from `/skills` in Gemini CLI.

Each host matches a task against the description its own way, so invoke the skill explicitly on a first run
rather than assuming it fired. Only `SKILL.md` is read up front; the `references/` files load on demand, so
the skill stays cheap in context until a topic is actually needed.

## What's inside

| File | Contents |
|---|---|
| `SKILL.md` | Entry point: the architecture diagram, ten critical facts, six hard rules, the quick start, and the reference directory |
| `references/adaptation.md` | The seam contract with the host app: tenancy, auth, capabilities, API keys, the event store, the rename table, background jobs, placement |
| `references/consent-and-privacy.md` | The two groups of people, the four capture layers, pause against revoke against disconnect, the excluded-domain module, what the employee is told, the legal layer |
| `references/data-model.md` | Neutral TypeScript shapes, the Postgres schema with RLS and the consent ledger, the Supabase and Firestore variants, volume estimates |
| `references/console-surfaces.md` | The employee page and the manager page: the server-confirmed consent toggle, the gap-capture form, automation ranking, the help page, their suites |
| `references/ingest.md` | The ingest contract: authentication, the consent re-read, per-event validation, idempotency, `207` with retry indexes, counters and retention |
| `references/pii-scrubber.md` | The scrubber: card against barcode, IBAN, JWT, email masking, sensitive labels, credential URLs, per-deployment configuration |
| `references/extension-auth.md` | The connect handshake: PKCE, `launchWebAuthFlow`, the extension-id allowlist, the atomic claim, minting and revoking the scoped key |
| `references/extension.md` | Manifest V3, the tested control logic for pause, budget, queue and session, label resolution and `dom_confidence`, the Chrome wiring, the unproven assumptions |
| `references/screenshots.md` | Screenshot modes, the capture gate and throttle, viewport and crop perceptual hashing, bounding boxes, upload and retention |
| `references/ai-pipeline.md` | The three batch stages, routing on `dom_confidence`, embeddings, SOP output and the guardrails |
| `references/ecommerce-playbook.md` | Which tools to capture in a shop, which processes to mine first, default gap-form categories, the rollout order, how it goes wrong |
| `references/testing.md` | What is verified and how: the vitest suites, 76 cases, and `verify.sql`, 24 SQL checks, plus what to add in the host |
| `references/provenance.md` | The engineering ledger: the maturity tags, what the audit changed, what was kept deliberately, what was added here, what is unverified |
| `evals/` | The prompts an operator types after installing (`prompts.md`) and one file per agent eval: the skill installed into an empty Next.js app, one prompt, no help, then type-checked, built and tested |
| `.github/workflows/agent-eval.yml` | The caller of the index's eval workflow: runs prompt 1 in Claude Code, Codex CLI and Gemini CLI on every published release, and any prompt on a maintainer's dispatch |

The seam is the table at the top of `references/adaptation.md`, and it bounds everything the host app already
owns: tenancy, session auth, the capability check, the API-key system, the event store, object storage, the
AI gateway, background jobs, UI primitives, strings and validation. The data layer is Postgres in the
templates, with Firestore differences stated row by row, and the canonical vocabulary (`scope_id`,
`pm_consent`, `pm_routine_entries`, `ActionEvent`) is renamed once through the table in that file. The wire
and protocol terms (`dom_confidence`, `session_id`, `step_index`, `bounding_box`, `code_challenge`) are not
renamed, because the extension and the AI stages both depend on them.

## The six non-negotiables

These travel with the module and are never optional. Each is stated as a hard rule in `SKILL.md` and covered
by the suites in `references/testing.md`:

1. **A privacy switch never fails open.** Unreadable pause state reads as paused, an unparseable host reads
   as excluded, and a failed consent save leaves the toggle showing what the server believes. The pause
   reducer, the exclusion matcher and the toggle hook each assert their failure case.
2. **A value you would have to scrub is never read.** Password, card, one-time-code and hidden fields are
   skipped at the point of capture, and the scrubber is the net behind that, not a licence to read. The
   value gate and the scrubber have separate suites, and the scrubber runs again on the server, where it is
   authoritative.
3. **A one-time code is claimed in one statement.** `DELETE ... RETURNING`, with the challenge verified
   after the claim, so two concurrent redeems cannot both mint a key. The SQL checks run a first claim, a
   second claim, a wrong extension id and an expired code.
4. **An authorization request from an unknown extension id is refused.** A well-formed id proves nothing
   about who published the extension, so the console keeps an allowlist and re-validates it in the authorize
   action, where hidden inputs are attacker-controlled.
5. **Success is never reported for an event that was not stored.** The emitter returns false rather than
   swallowing a write error, and the route answers `207` with the indexes to retry, because a `200` makes
   the extension delete its only copy. An in-memory store standing in for a missing database is not storage.
6. **Who opted in is not a manager's business.** There is no team policy on the consent table, only an
   aggregate function that suppresses small groups and full enrollment alike. The SQL checks assert that a
   manager role reads zero consent rows, whatever the page renders.

Everything else is the host app's: tenancy, auth, styling, i18n, the key system, the event log and the AI
provider.

## Requirements

A Next.js App Router app with server-side auth and a tenant on the session, an API-key system that stores
hashed secrets with per-key scopes bound to a user and a tenant, and Postgres or Firestore. An API-key
system is a prerequisite rather than part of this work; `references/adaptation.md` states what it must
provide. Chrome or another Chromium browser is needed for the extension, which is built and bundled outside
the web app. The legal groundwork, meaning the lawful basis, the DPIA and any works-council consultation,
belongs before the first line of capture code.

## Verification

The TypeScript templates compile under `strict` and again under `noUncheckedIndexedAccess`, against zod 3
and zod 4. The behaviour suites are 76 vitest cases in 9 files, with the DOM cases under happy-dom. The
schema, its policies and its triggers were executed on PostgreSQL 18 with 24 assertions, run both as a
non-owner API role and as the table owner. The `chrome.*` wiring, the manifest, the Supabase seam functions
and the Firestore notes are not executed anywhere, and `references/testing.md` says so row by row.

## Not this

| Not this | Use instead |
|---|---|
| Customer behaviour on the storefront | Product analytics. This module captures staff at work, not shoppers |
| "Who opened this link" | Server-side visit logging; the `visit-logger` skill |
| Data out of, or actions into, a service with no API | The sibling [`browser-extension-connector`](https://github.com/timerise-ai/browser-extension-connector) skill, which replays the page's own session; this one only observes what the employee does |
| Process mining over ERP or OMS event logs | A data-warehouse job over case id, activity and timestamp. No capture is needed |
| Productivity scoring, leaderboards, who has not opted in | Nothing here. The database withholds the rows, and `references/consent-and-privacy.md` says why |
| Session replay or screen recording | A replay vendor. This reads the DOM and stores structure, not video |
| Work in desktop applications | The gap-capture form. A browser extension sees browser tabs only |

## Contributing

Issues and pull requests are welcome here. Pure markdown, with no build step, but the code blocks are
checked: every TypeScript block names its destination on the first line, and the module and test blocks are
written to compile as one project under `strict` and `noUncheckedIndexedAccess` and to run under vitest,
while the SQL is executed against a disposable PostgreSQL database. Claims in this skill are meant to be
verifiable: if you change a factual claim, say how you verified it, whether against the Chrome extension
documentation, the Postgres documentation, RFC 7636, the zod release notes or a reproduction.

Adding, removing or renaming a file in `references/` means updating the quick start and the reference
directory table in `SKILL.md`, the file table above, and any relative cross-links. Every odd-looking part of
the templates is there for a reason, and `references/provenance.md` is the ledger that must stay truthful:
read it before simplifying anything, and add an entry for anything you change, including anything designed
here that has never run. Commits follow Conventional Commits and releases follow
[STANDARD.md](https://github.com/timerise-ai/skills/blob/main/STANDARD.md) in the index; `CLAUDE.md` carries
the full editing conventions.

## Part of the Timerise Skills

This is one of the [Timerise Skills](https://github.com/timerise-ai/skills): modules for **Next.js App
Router** apps written by our own senior engineers from the modules they have shipped, not synthetic, each
published as its own repository and indexed there. They share one layout, so an agent that has read one knows
how to read the next: a `SKILL.md` entry point, `references/` loaded on demand, and a seam contract carrying
the module's non-negotiables.

## Author

Built and maintained by [Timerise](https://timerise.ai).

## License

MIT. See [LICENSE](LICENSE).
