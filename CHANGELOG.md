# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.5] - 2026-09-02

Wording release. The origin and audit statements across the skill follow section 2 of
the skill standard; templates and technical content are unchanged from 0.1.4. The
repository history starts at this release.

### Changed
- Origin and audit wording across `SKILL.md`, `README.md`, `CLAUDE.md` and
  `references/provenance.md` now follows the skill standard: the skill is written by
  the engineer who has shipped the module, the reference point for the audit is the
  earlier implementation, stated in the standard's own words. The frontmatter
  description says the internals are hardened against the defects the audit found.

## [0.1.4] - 2026-09-02

Documentation-only release. The skill itself, `SKILL.md` and `references/`, is
unchanged from 0.1.3.

### Changed
- README: the install leads with `npx skills add timerise-ai/digital-signage`, which installs the skill
  into every skills-compatible agent it detects, with the `-a` form for named agents; the
  Claude Code clone moves under a *Manual install* heading. Activation gets its own
  heading, and a *Not this* table points neighbouring problems to the right skill or tool.
- README: the skill's origin is reworded. It was written by the engineers who built the
  module it describes; the reference point for `provenance.md` is the earlier
  implementation rather than "the source"; the index is called Timerise Skills.
- README: every em-dash, arrow and en-dash in the prose is rewritten as a comma, colon,
  full stop or conjunction.

## [0.1.3] - 2026-09-01

### Added
- Install section covers the one-command `npx skills add timerise-ai/digital-signage`
  route through [skills.sh](https://www.skills.sh), and how the skill is used from
  Codex CLI, Gemini CLI and other skills-compatible agents — `~/.agents/skills`,
  symlinking rather than cloning twice, and the differing invocation syntax.

### Changed
- `README.md` reworked so every claim holds against `SKILL.md` and `references/`:
  each contents-table row names the sections its reference file actually has,
  the backend seam is stated explicitly (Firestore canonical in `api-routes.md`
  and `operations.md`, substitutions in `supabase-backend.md`), and the trigger
  phrases mirror the `description` frontmatter.
- The skill is described as an Agent Skill rather than a Claude Code skill, in
  the README and in the repository description and topics. Nothing in it is
  Claude-specific.

## [0.1.2] - 2026-08-30

### Changed
- `LICENSE` names the legal entity, Timerise Sp. z o.o., matching every other
  skill in the [index](https://github.com/timerise-ai/skills).

## [0.1.1] - 2026-08-30

### Added
- `README.md` describing the skill, its install command, the contents of each
  reference file, and the three non-negotiables; MIT `LICENSE`.
- Author credit and a link to the Timerise skills index in the README.

### Changed
- Documentation synced with the repository: the extensions row in both the
  README contents table and `SKILL.md`'s reference directory now lists the
  multi-zone layouts topic, so it is reachable through the trigger keywords.

## [0.1.0] - 2026-08-05

Initial release of the digital-signage skill.

### Added
- `SKILL.md` entry point with architecture overview, critical facts, hard rules,
  and the reference directory table mapping trigger keywords to reference files.
- `references/` topic files covering the seam contract (`adaptation.md`), data
  model, Firestore and Supabase/Postgres backends, API routes, device pairing,
  the TV player runtime, admin UI, operations, and extensions.
- `references/provenance.md` documenting the ten defects found in the original
  production module and how the templates fix them.

### Fixed
- Hardened templates after a second audit of the skill (tightened player
  internals and data/auth model against the documented defect classes).
