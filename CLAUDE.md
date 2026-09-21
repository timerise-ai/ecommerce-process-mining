# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

An [Agent Skill](https://agentskills.io) package: markdown only. There is no `package.json` here and nothing
in this repository executes. It teaches an agent how to build employee process mining for an e-commerce back
office in a **Next.js App Router** app: a consented Manifest V3 extension that reads DOM events rather than
pixels, a gap-capture form for the work a browser cannot see, an ingest route that scrubs before it stores,
and a batch AI pipeline that writes per-role SOPs and ranks automation candidates.

Keep the two straight: the commands and code in `references/` describe the app the agent will generate, not
this repository. The `lib/process-mining/*` modules, the `db/process-mining/*.sql` files, the `extension/`
tree, the `npx vitest run` and `psql` invocations in `testing.md` all belong to that generated app. What is
checked here is that the templates compile and their suites pass, and that check runs in a scratch project.

The skill was written by the engineer who has shipped this module. `references/provenance.md` is the ledger
of the audit: the maturity tags, what the audit changed and how the templates verify it, what was kept
deliberately with the reason it is safe, what was designed here and has never run, and what is unverified.
That file is the rationale layer: read it before "simplifying" anything.

## Structure

- `SKILL.md`: entry point, loaded whole on every activation, so it stays between 130 and 160 lines, the
  closing index line aside. The frontmatter `description` is the trigger surface; the body carries the
  architecture diagram, ten **critical facts**, six **hard rules**, the quick-start order, the **reference
  directory table** mapping trigger keywords to files, and a closing line linking the skills index.
- `README.md`: the human-facing front door, in the section order of the skill standard: install, activation,
  the file table, the six non-negotiables, requirements, verification, the *Not this* table, contributing.
- `references/*.md`: one topic per file, loaded on demand. `adaptation.md` (the seam contract) and
  `consent-and-privacy.md` (the privacy contract the rest implements) are the entry points; `data-model.md`,
  `console-surfaces.md`, `ingest.md`, `pii-scrubber.md`, `extension-auth.md`, `extension.md` and
  `screenshots.md` carry the templates; `ai-pipeline.md` and `ecommerce-playbook.md` carry the decisions
  around them; `testing.md` the suites; `provenance.md` the audit.

## Editing conventions

- **Maturity tags are load-bearing.** Every section carries **[P]** (ran in the earlier implementation),
  **[D]** (specified in its design notes, never built there) or **[A]** (designed here, held by the tests
  shipped with this skill). Never promote a tag without evidence, and never drop one. The distinction is the
  skill's credibility, and `provenance.md` defines it.
- **Code blocks name their destination on the first line** as a comment, for example
  `// file: lib/process-mining/scrub.ts`. That line is what makes a block extractable, so keep it and keep
  imports complete. Shared modules live in `lib/process-mining/`, SQL in `db/process-mining/`, and
  extension-only wiring in `extension/`.
- **Identifiers are shared across files.** `ActionEvent`, `DomConfidence`, `ConsentRecord`, `RoutineEntry`,
  `isConsentActive`, `normalizeHost`, `parseExcludedDomains`, `isHostExcluded`, `diffExclusions`,
  `scrubEvent`, `CUSTOMER_PII_LABELS`, `DEFAULT_SENSITIVE_LABELS`, `mayReadValue`, `resolveLabel`,
  `CaptureBudget`, `reducePause`, `isCapturing`, `classifyIngestResponse`, `enforceQueueBounds`,
  `challengeFor`, `validateAuthorizeParams`, `annualHours`, `rankForAutomation`, `useConfirmedToggle`, and
  the SQL names `pm_consent`, `pm_consent_ledger`, `pm_routine_entries`, `pm_routine_categories`,
  `pm_extension_grants`, `pm_events`, `pm_current_user`, `pm_current_scope`, `pm_has_capability`,
  `pm_enrollment_stats`, `pm_claim_extension_grant` appear in several references. Rename in all of them or
  none.
- **Keep the three tables in sync** with `references/`: the reference directory in `SKILL.md`, the
  quick-start list in `SKILL.md`, and the file table in `README.md`. Links are relative:
  `[x.md](references/x.md)` from `SKILL.md`, `[x.md](x.md)` between references.
- **Do not remove the odd-looking parts.** The composite consent key, the missing manager policy, the
  immutability trigger, the second exclusion check on the server, the length-plus-prefix card rule, the
  `code=` exception, the crop hash beside the viewport hash, the verifier in `chrome.storage.session`, the
  single-statement claim and the `emit` that returns false: each is an entry in `provenance.md`. Check it
  before touching one.
- **The numbers that remain are load-bearing.** 76 vitest cases, 24 SQL checks, the 60-second grant, the
  43-character challenge, the 5-minute exclusion refresh, the group of five, the 200-event batch, the 256 KB
  cap, the 600 ms screenshot throttle, the 7-day queue. They were verified against this repository or are
  design parameters the next implementation needs. Do not restate them loosely and do not add new ones.
  Figures describing the earlier implementation's deployment do not appear anywhere.
- **Mark additions as additions.** Anything designed here and never run belongs in the *Added* section of
  `provenance.md`, tagged **[A]** where it appears, or stated as a design in the reference that carries it.
- **Never present the non-negotiables as optional.** The privacy switch that never fails open, the value
  never read, the single-statement claim, the extension-id allowlist, the honest ingest response and the
  aggregate-only enrollment are hard rules in `SKILL.md` and non-negotiables in `README.md`; keep them that
  way everywhere.
- The canonical domain names (`scope_id`, `pm_*` tables, "routine entry", "My routine") are meant to be
  renamed by the host, and `adaptation.md` carries that procedure. The wire and protocol terms
  (`dom_confidence`, `session_id`, `step_index`, `bounding_box`, `viewport`, `code_challenge`,
  `code_verifier`, `redirect_uri`, `state`) are the authoring contract and are not renamed.
