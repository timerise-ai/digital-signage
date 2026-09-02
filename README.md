# digital-signage

An [Agent Skill](https://agentskills.io) that teaches an agent to build a digital-signage module in a
**Next.js App Router** app: ad media library, per-screen playlists, display registry, device pairing by PIN or
provisioning URL, and a fullscreen TV player with health monitoring and remote control. Backend-agnostic
between **Firestore** and **Supabase/Postgres**.

A screen in a venue is not a web page. It runs unattended for months on a cheap TV stick, nobody is watching
the console, and the failure mode is a frozen frame no one notices for a week. This skill was written by the
engineer who has shipped this signage module; the earlier implementation it was audited against ran the
screens of a multi-venue booking system. The templates hold the properties that keep a screen alive
unattended: a player that skips a stalled video or a broken asset on its own and keeps looping through an
outage, a poll that fires on a stable interval whatever the slide state, a per-device token stored hashed
and revoked with one row update, and every playlist and route scoped to its venue server-side. The player's
behaviour contract and the route tests state each one; [`references/provenance.md`](references/provenance.md)
has the record.

## Install

One command, via the [skills.sh](https://www.skills.sh) CLI, which installs the skill into every
skills-compatible agent it detects, including Claude Code, Codex CLI and Gemini CLI:

```bash
npx skills add timerise-ai/digital-signage
```

Name the agents instead with `-a`, for example `npx skills add timerise-ai/digital-signage -a claude-code -a codex`.

Or clone it yourself. Nothing here is Claude-specific: the skill is a plain [Agent
Skills](https://agentskills.io) folder, `SKILL.md` plus markdown references with no file that calls a model,
so cloning it into an agent's skills directory is all an install is. For Claude Code:

```bash
git clone https://github.com/timerise-ai/digital-signage.git ~/.claude/skills/digital-signage
```

To scope it to a single project instead, clone it into that project's `.claude/skills/` directory. For another
agent, clone into that agent's skills directory, or symlink the Claude Code copy so one `git pull` updates
every agent:

```bash
mkdir -p ~/.agents/skills
ln -s ~/.claude/skills/digital-signage ~/.agents/skills/digital-signage
```

Update the skill with `git pull` in its directory. The current release is **0.1.6**. See
[`CHANGELOG.md`](CHANGELOG.md). The [skills index](https://github.com/timerise-ai/skills) lists the other
Timerise Skills and how to install them all at once.

## Activation

The skill activates automatically when a task matches its description: building in-venue screens, digital
menu boards or lobby displays, playlist scheduling, media upload with duration and orientation handling,
pairing a device by PIN or provisioning URL, screen health monitoring, remote screen control. Invoke it
explicitly with `/digital-signage` in Claude Code, `$digital-signage` in Codex CLI, or from `/skills` in
Gemini CLI.

Each host matches a task against the description its own way, so invoke the skill explicitly on a first run
rather than assuming it fired. Only `SKILL.md` is read up front; the `references/` files load on demand, so
the skill stays cheap in context until a topic is actually needed.

## What's inside

| File | Contents |
|---|---|
| `SKILL.md` | Entry point: architecture diagram, critical facts, hard rules, quick start, and the reference directory |
| `references/adaptation.md` | The seam contract with the host app:  the domain rename, the non-negotiables, what the host must already provide, integration points, order of work |
| `references/data-model.md` | Entities and types, the wire payload, array-vs-junction trade-off, `active` vs `deletedAt`, playback semantics, tenant scoping |
| `references/firestore-backend.md` | Security posture, lazy admin client, service module, client upload and server-side deletion, rules and composite indexes, Firestore traps |
| `references/supabase-backend.md` | Schema and migration, playlist resolution and reordering, RLS, storage, device-route services, optional Realtime |
| `references/api-routes.md` | Token minting and verification, pairing and playlist endpoints, admin routes, ETag caching, tests |
| `references/pairing.md` | Bootstrap and pairing, the hub screen, provisioning URL vs PIN |
| `references/player-runtime.md` | Kiosk chrome, the state machine, the player loop, crossfade, stall recovery, rotation, wake lock, error boundary, behaviour contract, offline |
| `references/admin-ui.md` | Upload-then-save, client-side metadata extraction, ad form and fit warning, media library, playlist editor, screen list and provisioning |
| `references/operations.md` | Heartbeat and health, preview, remote control (reload, blank, emergency takeover), audit log, unpairing |
| `references/extensions.md` | Dayparting, reusable playlists, proof of play, offline media precaching, multi-zone layouts |
| `references/provenance.md` | The engineering ledger: what the audit of the earlier implementation changed and how the templates verify it, what was kept on purpose, and what is new in the skill |

The backend seam runs through the reference set: `api-routes.md` and `operations.md` are written against
Firestore as the canonical backend, and `supabase-backend.md` defines every substitution: the junction table
that replaces `display.adIds`, and SQL equivalents of the service functions. The host app's auth, tenancy,
styling and i18n stay the host app's; `adaptation.md` is where you wire them in.

## The three non-negotiables

These travel with the module and are never optional (see `references/adaptation.md`):

1. **Inline styles on device components.** Signage runs on TV browsers years behind current, so the player
   cannot depend on the host app's CSS pipeline. Not a style preference; a compatibility requirement.
2. **Poll interval independent of render state.** The poll runs on a stable timer and reads slide state from
   a ref, so a refresh longer than a slide still fires on time. The behaviour contract in
   `references/player-runtime.md` pins the poll's timing and failure handling.
3. **Per-display tokens stored hashed.** Each screen holds its own token, the server stores only the hash,
   and revocation is one row update; the device is untrusted hardware anyone can walk up to. The route tests
   in `references/api-routes.md` cover a revoked token.

Everything else is the host app's: domain names, backend, auth, tenancy, styling, i18n.

## Not this

| Not this | Use instead |
|---|---|
| Web ad serving: impressions, bidding, tracking pixels, third-party tags | Ad-tech infrastructure; a different problem |
| Interactive kiosks where the user taps to transact | The sibling [`booking-kiosk`](https://github.com/timerise-ai/booking-kiosk) skill; signage is one-way |
| A single embedded video on a marketing page | A `<video>` tag |
| Generic Next.js, CMS or auth setup | The host app; the skill assumes these exist |

## Contributing

Issues and pull requests are welcome here. Pure markdown, with no build, lint or test step. Claims in this
skill are meant to be verifiable: if you change a factual claim, say how you verified it, whether against the
library, the docs, or a reproduction.

Adding, removing or renaming a file in `references/` means updating the quick start and the reference
directory table in `SKILL.md`, the file table above, and any relative cross-links. Every odd-looking part of
the templates is there for a reason, and `references/provenance.md` is the ledger that must stay truthful:
read it before simplifying anything, and add an entry for anything you change. Commits follow Conventional
Commits and releases follow [STANDARD.md](https://github.com/timerise-ai/skills/blob/main/STANDARD.md) in the
index; `CLAUDE.md` carries the full editing conventions.
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