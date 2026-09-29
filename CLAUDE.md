# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

An [Agent Skill](https://agentskills.io) package: markdown only. There is no `package.json` here and nothing
in this repository executes. It teaches an agent how to build a digital-signage module (media library,
playlists, display registry, device pairing, fullscreen TV player) in a **Next.js App Router** app, with the
data layer as a seam between Firestore and Supabase/Postgres.

Keep the two straight: the commands and code in `references/` describe the app the agent will generate, not
this repository. The migrations, security rules, route handlers and tests all run in that generated app.

The skill was written by the engineer who has shipped this module; `references/provenance.md` is the
engineering ledger, ten entries recording what the audit of the earlier implementation changed and how the
templates verify it, what was kept on purpose, and what was designed in the skill. That file is the rationale
layer for the whole skill.

## Structure

- `SKILL.md`: the entry point, loaded whole on every activation, so it stays at 130 to 160 lines. The
  frontmatter `description` is the trigger surface. The body carries the architecture diagram, the critical
  facts, the hard rules, the quick start, the reference directory table and a closing line linking the
  skills index.
- `references/*.md`: one concern per file, loaded on demand. `adaptation.md` (the seam contract and the
  non-negotiables restated) and `data-model.md` are the design entry points; the rest cover the two
  backends, API routes, pairing, the player runtime, admin UI, operations, extensions, and the ledger.
- `README.md`: the human front door, in the fixed section order of the index's STANDARD.md. `CHANGELOG.md`:
  Keep a Changelog, newest first.
- `evals/`: `prompts.md` holds what an operator types after installing, in their words; the first prompt is
  the agent eval run before every release. Every other file there is one eval run: measured frontmatter that
  is never edited, then the notes of the person who ran it. Add a prompt rather than rewording one that has
  results. The procedure is section 10 of the index's STANDARD.md.
- `.github/workflows/agent-eval.yml`: the caller of the index's reusable eval workflow, copied verbatim from
  STANDARD.md. Never edit it and never add a trigger.

## Editing conventions

- **Code blocks name their destination** on the first line as a comment, for example `// types/signage.ts`. A
  continuation block that extends a file already introduced omits it.
- **Identifiers are shared across files.** A function, type or field named in one reference has the same name
  in every other. Rename in all of them or none.
- **Keep three lists in sync with `references/`**: the reference directory table in `SKILL.md`, the quick
  start in `SKILL.md`, and the file table in `README.md`. Links are relative:
  `[data-model.md](references/data-model.md)` from `SKILL.md`, `[data-model.md](data-model.md)` between
  references.
- **The odd-looking parts stay.** Complexity in the templates usually holds a ledger entry: the poll interval
  decoupled from slide state, per-display hashed tokens, 404 not 403 on scope failures, inline styles on
  device components. Check `references/provenance.md` before removing anything; a change to a template adds
  an entry there.
- **Measured numbers are load-bearing.** The 30 to 60 s poll, the 3x-duration watchdog and the 10 MB
  upload limit are design parameters; do not restate them loosely and do not invent new ones.
- **Additions are marked as additions.** Anything designed in the skill and never run in production goes
  under *Added* in `provenance.md`, and `extensions.md` stays opt-in.
- **The seven non-negotiables are never optional.** They are the hard rules in `SKILL.md`, the list in
  `README.md` and the list in `adaptation.md`, in the same order: inline styles on device components, the poll
  interval independent of render state, per-display tokens stored hashed, server-derived tenant scope
  answering 404, `active` separate from `deletedAt`, the image timer started at `onLoad`, durable screen state
  as `mode` with the reload ack persisted first. Never present any of them as optional in a reference.
- **The host renames the domain; the platform terms stay.** `Ad`, `Display` and `Location` are generic on
  purpose and the rename table lives in `adaptation.md`. `orientation`, `mimeType`, `ETag`, `duration` and
  `storagePath` are the authoring contract and are never renamed.
- **The description is the trigger surface.** If the skill's scope changes, update its trigger keywords and
  the *When to use* and *When NOT to use* sections together.
- **Plain punctuation, prose wrapped at 110 columns.** No em-dashes, en-dashes, arrows, middle dots or smart
  quotes anywhere in the markdown, code comments included.
