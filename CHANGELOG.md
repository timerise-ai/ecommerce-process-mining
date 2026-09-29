# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.4] - 2026-09-29

Wording release, from scoring the prompt-1 agent eval runs against 0.1.3. The
templates and the suite are unchanged.

### Changed

- `SKILL.md` quick start: every `// file:` block is written as shipped, with the
  renames in `references/adaptation.md` the only edits, and stricter privacy
  goes in the host's wiring rather than inside a template.
- `SKILL.md` quick start and `references/testing.md`: `zod`, `vitest`,
  `happy-dom` and `@testing-library/react` are installed from the package
  registry, which is not an external service, and the nine suites run as
  written under `npx vitest run`, 76 cases; a suite is never ported to another
  runner.
- `SKILL.md` quick start and `references/screenshots.md`: with screenshots off,
  `screenshot.ts` and its suite still ship, because the mode is a setting read
  by the gate, not a build option.
- `SKILL.md` quick start ends with the handover: pin the extension id and add it
  to the allowlist, plan the browser spike, and do not treat the consent toggle
  as a legal basis. Two quick-start steps merged to stay within the line budget.
- `references/provenance.md` records these under *Added*.

## [0.1.3] - 2026-09-28

Wording release. The skill content is unchanged from 0.1.2.

### Changed

- The six hard rules in `SKILL.md` are now the same list, in the same order, as
  the six non-negotiables in `README.md`: the sixth rule is that a manager never
  sees who opted in, and each rule states the reason it holds. Carrying the
  **[P]**, **[D]** and **[A]** tags remains an instruction in the framing
  paragraph.
- The file table in `README.md` and the structure in `CLAUDE.md` list `evals/`
  and the agent eval workflow, and `CLAUDE.md` records that evals are not skill
  content.

## [0.1.2] - 2026-09-21

Wording release. The skill content is unchanged from 0.1.1.

### Changed

- The `Not this` table in `README.md` and the `When NOT to use` table in
  `SKILL.md` name the sibling
  [`browser-extension-connector`](https://github.com/timerise-ai/browser-extension-connector)
  skill, so a request to pull data out of a service with no API lands there
  rather than here. That extension acts inside the user's session; this one
  only observes what the employee does.

## [0.1.1] - 2026-09-21

Wording release. The skill content is unchanged from 0.1.0.

### Added

- `SKILL.md` closes with a line linking the
  [Timerise Skills](https://github.com/timerise-ai/skills) index, so an agent that
  has the skill loaded can find the sibling skills for neighbouring modules without
  leaving the entry point.

### Changed

- `CLAUDE.md` records the closing line in the `SKILL.md` layout, and the line budget
  it states holds that line aside.

## [0.1.0] - 2026-09-21

Initial release of the `ecommerce-process-mining` skill: consented DOM-event capture
from the admin tools an e-commerce back office already uses, a gap-capture form for the
work a browser cannot see, and a batch pipeline that turns both into per-role SOPs and a
ranked automation shortlist.

### Added
- `SKILL.md` entry point: the architecture diagram, ten critical facts, six hard rules,
  the quick-start order, and the reference directory table mapping trigger keywords to
  `references/`.
- `references/adaptation.md`: the seam contract with the host app, the capability table,
  the canonical rename table, the background jobs, module placement shared by the app and
  the extension, and the host probe.
- `references/consent-and-privacy.md`: the employee and the customer on the screen, the
  four capture layers in order, pause against revoke against disconnect, the
  excluded-domain module shared by both sides, what the employee must be told, and the
  legal layer per jurisdiction.
- `references/data-model.md`: the neutral TypeScript shapes, the Postgres schema with
  row-level security, the append-only consent ledger and its immutability trigger, the
  suppressing enrollment function, the Supabase wiring and the Firestore differences.
- `references/console-surfaces.md`: the employee page and the manager page, the
  server-confirmed consent toggle, the gap-capture form and its validation, automation
  ranking, the help page, and their suites.
- `references/ingest.md`: the ingest contract, the consent re-read per batch, per-event
  validation, the age window, idempotency, `207` with retry indexes, counters and
  retention.
- `references/pii-scrubber.md`: the scrubber, the card-against-barcode rule, IBAN, JWT
  and email masking, sensitive labels, credential URLs, per-deployment configuration and
  the known limits.
- `references/extension-auth.md`: the PKCE handshake, the extension-id allowlist, the
  single-statement claim, minting the scoped key, and revoking it.
- `references/extension.md`: Manifest V3 with its corrections, the tested control logic
  for pause, budget, queue and session rotation, label resolution and `dom_confidence`,
  the Chrome wiring, and the assumptions to settle in a spike.
- `references/screenshots.md`: the screenshot modes, the capture gate and throttle,
  viewport and crop perceptual hashing, bounding boxes, upload and retention.
- `references/ai-pipeline.md`: the three batch stages, routing on `dom_confidence`,
  embeddings, SOP output and the guardrails.
- `references/ecommerce-playbook.md`: which tools to capture in a shop, which processes
  to mine first, the default gap-form categories, the rollout order and the failure
  modes.
- `references/testing.md`: what is verified and how, the vitest suites with 76 cases, the
  24 SQL checks in `verify.sql`, and what to add in the host.
- `references/provenance.md`: the engineering ledger, the maturity tags, what the audit
  changed and how the templates verify it, what was kept deliberately, what was designed
  here and has never run, the unverified list, and the fix order for anyone maintaining
  the earlier implementation.
- Templates for `lib/process-mining/*`, `db/process-mining/schema.sql` and `verify.sql`,
  and the `extension/` tree, written to compile under `strict` and
  `noUncheckedIndexedAccess` against zod 3 and zod 4.
- The properties the templates hold: consent per person and per tenant, re-read on every
  batch; one exclusion matcher imported by the extension and the ingest route; values
  that would need scrubbing never read; a scrubber that separates a card number from a
  barcode; a one-time code redeemable exactly once; a consent history nobody can rewrite;
  an ingest response that never claims an unstored event; and an enrollment count
  withheld when it would name the people.
- `README.md`: install, activation, the file table, the six non-negotiables,
  requirements, verification, the *Not this* table and the contributing conventions.
- `CLAUDE.md`: editing conventions for this repository.
- `LICENSE`: MIT.
