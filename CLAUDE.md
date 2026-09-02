# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

A **Claude Code skill package** — pure markdown, no build, lint, or test commands. It teaches an agent how to build a digital-signage module (media library, playlists, display registry, device pairing, fullscreen TV player) in a Next.js App Router app, backend-agnostic between Firestore and Supabase/Postgres.

The skill was written by the engineer who has shipped this module; `references/provenance.md` is the engineering ledger, ten entries recording what the audit of the earlier implementation changed and how the templates verify it. That file is the rationale layer for the whole skill.

## Structure

- `SKILL.md` — entry point. Frontmatter (`name`, `description`) drives skill discovery/triggering. The body carries the architecture diagram, the critical facts, the hard rules, and the **reference directory table** that maps trigger keywords to files in `references/`.
- `references/*.md` — one topic per file, loaded on demand by the consuming agent. `adaptation.md` (the seam contract with the host app) and `data-model.md` are the design entry points; the rest cover backends, API routes, pairing, the player runtime, admin UI, operations, and extensions.

## Editing conventions

- **Keep SKILL.md's reference directory table in sync** with `references/`. Adding, removing, or renaming a reference file means updating that table and any cross-links (all links are relative, e.g. `[data-model.md](references/data-model.md)` from SKILL.md, `[data-model.md](data-model.md)` between references).
- **Do not "simplify" template code that looks over-engineered.** Complexity in the templates usually holds one of the ten ledger entries (e.g. the poll interval decoupled from slide state, per-display hashed tokens, 404-not-403 on scope failures, inline styles on device components). Check `references/provenance.md` before removing anything; if a fix changes behavior the provenance describes, update provenance too.
- **Three non-negotiables travel with the module** (documented in `adaptation.md`): inline styles on device components, poll interval independent of render state, per-display tokens stored hashed. Never present these as optional in any reference.
- Code samples are TypeScript for Next.js App Router. Domain names (`Ad`, `Display`, `Location`) are deliberately generic — the rename procedure lives in `adaptation.md`. Platform terms (`orientation`, `mimeType`, `ETag`, `duration`, `storagePath`) are never renamed.
- SKILL.md's `description` frontmatter is the trigger surface — if the skill's scope changes, update its trigger keywords and the "When to use / When NOT to use" sections together.
