# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.11] - 2026-10-08

Security fix release, from scoring the prompt-1 agent eval runs against 0.1.10. Each defect was found by an
agent's edit and reproduced against the templates before the fix was adopted.

### Security

- One venue's admin could delete another venue's media. Upload paths were flat and chosen by the client, the
  purge deletes the stored path with server credentials, and the Supabase bucket's select policy let anyone
  list every file. Uploads now land in `ads/<locationId>/`; both ad routes reject a `storagePath` outside the
  active venue's folder (`ownsStoragePath`); the Supabase insert policy and the Firebase Storage rules match
  the folder; the select policy is removed and the Storage rule allows `get`, not `read`, so neither bucket
  can be listed. Apps built from earlier versions should copy all four, and move existing objects or accept
  that ads saved before the change keep their flat paths.
- A playlist could play another venue's ad. Neither the `display_ads` write policy nor resolution checked the
  ad's venue. `getAdsByIds(ids, locationId)` drops foreign ads, and the Supabase resolution query carries the
  same `exists` on `ad_locations`. Apps built from earlier versions should add the venue check to their
  resolution.

### Fixed

- The crossfade slot followed index parity, so a playlist edit that moved the current slide remounted it,
  restarting a playing video, and the wrap of an odd-length loop faded from black. `activeSlot` in
  `player-machine.ts` follows the epoch.
- `uploadAdFile` takes `(file, locationId, onProgress?)` on both backends; the Supabase version took only the
  file while the admin UI passed a progress callback.

### Added

- Tests that fail on the old code: the player-machine slot test, a Firestore `getAdsByIds` test with a stub
  `db`, a playlist-route test that resolution receives the screen's venue, and a PUT test that refuses a
  file outside the venue's folder.

### Changed

- `SKILL.md`: every code block is copied to the path its first line names, changing only the documented
  renames, and a template that looks wrong is reported in the closing summary, not rewritten; the admin
  routes are one file per row of the route table, never a catch-all. The hard rules are rewrapped to 110
  columns to stay within the line budget.

## [0.1.10] - 2026-10-07

Fix release, from scoring the prompt-1 agent eval runs against 0.1.9.

### Fixed

- Editing an ad reset the fields the request left out. The PUT in `references/api-routes.md` parsed
  `AdBody.partial()`, and zod 4 applies a `.default()` even under `.partial()`, so renaming a paused ad sent
  `active: true` and `locationIds: []`: the ad went back on air and dropped out of its venues. `AdFields` now
  carries no defaults and the PUT parses `AdFields.partial()`; `AdBody` adds the defaults for create only. A
  new route test fails on the old schema. Apps built from earlier versions should make the same split in
  `app/api/admin/ads/route.ts` and `app/api/admin/ads/[id]/route.ts`.

### Changed

- The quick start in `SKILL.md` says the package registry is not an external service, so what the templates
  import (`zod`, the backend's SDK, `vitest`) is installed rather than replaced by a hand-written client; that
  each row of the route table is its own route file; that the tests ship unmodified on vitest; and that the
  closing summary carries the `SIGNAGE_TOKEN_PEPPER` rule.
- `references/api-routes.md` names the Supabase variables `.env.example` lists and the vitest setup the tests
  run on, and forbids converting them to another runner.
- `references/provenance.md` records the PUT fix under *Added*.

## [0.1.9] - 2026-10-07

Wording release, from scoring the prompt-1 agent eval runs against 0.1.8. Templates are unchanged.

### Changed

- `references/adaptation.md` says what to build when the host app has no staff auth, venues or PIN: staff
  auth on the backend's own auth with a `staff_locations` membership, a hashed per-venue PIN set from the
  back-office, and a seed script for the first venue and admin. A location taken from a header, cookie or
  query parameter, a default venue, a PIN in code, a placeholder secret and fallback data for an unreachable
  database are named as never acceptable.
- Hard rule 4 in `SKILL.md`, the README and `adaptation.md` names the header, cookie and query parameter as
  client-supplied scope.
- The quick start in `SKILL.md` repeats the empty-app clause, says to keep the 10 MB upload limit, the 60 s
  poll, the 15 s video stall and the 3x-duration watchdog as the templates set them, to ship the tests
  unmodified, and to hand over a tracked `.env.example` and the rule that rotating `SIGNAGE_TOKEN_PEPPER`
  unpairs every screen. `references/api-routes.md` and the integration checklist say the same.
- `references/provenance.md` records the empty-app clause under *Added*.

## [0.1.8] - 2026-09-29

Documentation release. Templates and technical content are unchanged from 0.1.7; the skill now follows the
index's skill standard throughout.

### Changed

- The hard rules in `SKILL.md` and the non-negotiables in `README.md` are one list of seven, in the same
  order, restated in `references/adaptation.md` and `CLAUDE.md`: inline styles on device components, the poll
  interval independent of render state, per-display tokens stored hashed, server-derived tenant scope
  answering 404, `active` separate from `deletedAt`, the image timer started at `onLoad`, and durable screen
  state as `mode` with the reload ack persisted first. Each was already a hard rule or a non-negotiable, so no
  template changes behaviour.
- Plain punctuation across every markdown file, code comments included. Glyphs the device or admin UI renders
  are kept through escapes, so what a screen shows is unchanged.
- `SKILL.md`, `README.md` and `CLAUDE.md` follow the standard's section shapes: the frontmatter description
  in its fixed order with more trigger vocabulary, the README intro, manual install, full file table and
  contributing paragraphs, and `CLAUDE.md` in three sections.
- `references/provenance.md` marks the operations layer as designed in the skill and never run in production.

## [0.1.7] - 2026-09-21

Wording release. The skill content is unchanged from 0.1.6.

### Added

- `SKILL.md` closes with a line linking the
  [Timerise Skills](https://github.com/timerise-ai/skills) index, so an agent that
  has the skill loaded can find the sibling skills for neighbouring modules without
  leaving the entry point.

### Changed

- `CLAUDE.md` records the closing line in the `SKILL.md` layout.

## [0.1.6] - 2026-09-02

Wording release. Templates and technical content are unchanged from 0.1.5.

### Changed
- The front door (`README.md`, `SKILL.md`, `CLAUDE.md`) describes the module by the
  properties the behaviour contract and the route tests verify: a player that skips a
  stalled video on its own, a poll on a stable interval, hashed per-device tokens,
  venue-scoped playlists. The audit record stays in `references/provenance.md`.

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
  Codex CLI, Gemini CLI and other skills-compatible agents: `~/.agents/skills`,
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
