---
name: digital-signage
description: >
  Build a digital-signage module in a Next.js App Router app on Firestore or
  Supabase — ad media library, per-screen playlists, display registry, device
  pairing, and a fullscreen TV player. Use when: (1) building or extending
  in-venue screens, digital menu boards, lobby/waiting-room displays, or an
  advertising-slot system, (2) implementing playlist scheduling, media upload
  with duration and orientation handling, device pairing by PIN or provisioning
  URL, screen health monitoring, or remote screen control, (3) the user mentions:
  digital signage, display screens, TV player, ad playlist, signage module,
  kiosk display, screen management, proof of play. Carries player internals
  (video stall recovery, portrait rotation, TV-browser quirks) and a data/auth
  model, both hardened against the defects the audit found. Framework-specific to
  Next.js App Router; backend-agnostic between Firestore and Postgres/Supabase.
---

# Digital Signage — Displays + Ads

A screen in a venue is not a web page. It runs unattended for months on a cheap
TV stick, nobody is watching the console, and the failure mode is a frozen frame
that no one notices for a week. Every decision below exists to survive that.

This skill carries a complete module: admin CRUD for media and playlists, a
display registry, PIN and provisioning-URL pairing, a fullscreen player, and the
operational layer (health, preview, remote control) that makes it manageable.

## When to use

Building or extending screens that loop media in a physical space, on Next.js
App Router with Firestore or Supabase/Postgres.

## When NOT to use

- **Web ad serving** (impressions, bidding, tracking pixels, third-party tags) —
  a different problem with different infrastructure.
- **Interactive kiosks** where the user taps to transact. Signage is one-way.
- **A single embedded video** on a marketing page. Use a `<video>` tag.
- Generic Next.js, CMS, or auth setup — assumed to exist.

## Architecture

```
 ADMIN (browser)                     SERVER                      DEVICE (TV)
 ─────────────────                   ──────                      ───────────
 media upload ──────────────► object storage ◄──── public/signed URL ─┐
 ad CRUD ─────────┐                                                   │
 playlist edit ───┼──► /api/admin/*  ──► ads + displays ──┐           │
 issue command ───┘      (staff auth)      (scoped)       │           │
                                                          ▼           │
 pair screen ◄──── PIN ────────────► /api/display/pair    │           │
                                     mints per-display    │           │
                                     token                ▼           │
                                    /api/display/playlist ─── poll ───┤
                                     ▲ resolves adIds → ads           │
                                     └── carries telemetry up,        │
                                         commands down          player loop
```

One request type sustains the whole runtime: the device polls
`/api/display/playlist`, sending telemetry up and receiving content and commands
down. Everything operational rides that channel.

## Critical facts — read before designing anything

1. **The playlist is an ordered array on the display, not a separate entity.**
   `display.adIds: string[]` *is* the playlist — order is free, no joins, one
   read. Introduce a standalone playlist entity only when the same content must
   run on several screens; see [data-model.md](references/data-model.md) for the
   trade-off and [extensions.md](references/extensions.md) for the migration.
2. **Poll; do not stream.** A 30–60 s poll is cheaper, survives sleeping network
   stacks, and reconnects for free. Realtime is an optional upgrade, not the
   baseline. Cache the response with an ETag or you will pay for a full read per
   screen per poll.
3. **Media uploads go browser → storage directly**, never through an API route.
   Route handlers have body-size limits and burn compute proxying bytes.
4. **The device is untrusted and unattended.** It holds a long-lived credential
   in `localStorage` on hardware anyone can walk up to. That credential must be
   per-screen and revocable.
5. **The player must never be able to stop.** Every media element gets a timeout,
   an error handler, and a way to skip. A broken asset advances; it does not wedge.

## Hard rules

> **Never derive the poll interval from render state.** If the fetch callback
> closes over the current slide index and the interval effect depends on it, the
> timer is destroyed and recreated on every slide — so a 5-minute refresh behind
> 10-second slides **never fires** and screens silently stop updating. Poll on a
> stable interval; read slide state from a ref.

> **Never overload one boolean as both "paused" and "deleted".** Use `active` for
> operator intent and a separate `deletedAt` for lifecycle, or a deleted item
> reappears the moment someone toggles it back on.

> **Never trust a client-supplied tenant/location scope.** Derive it server-side
> from the authenticated staff session or from the display row the token
> resolves to, and enforce it on every `[id]` route — returning **404, not 403**,
> so ids cannot be probed.

> **Never issue one shared token to every screen.** Mint a per-display token at
> pairing, store only its hash, and make revocation a single row update.

> **Never start an image's duration timer before `onLoad`.** On a slow TV the
> slide will otherwise expire before it is visible.

> **Never carry durable screen state in a one-shot command.** Blank and
> takeover must survive the device's daily reload and a power cycle — deliver
> them as state on every poll (`mode`), and keep the ack-cleared command
> channel for genuine one-shots like reload. And any command whose effect is a
> reload must persist its ack **before** reloading, or the server re-delivers
> it forever.

## Quick start

0. Fill in the seam contract for your app and confirm the domain rename —
   [adaptation.md](references/adaptation.md).
1. Model the entities and pick array-vs-junction —
   [data-model.md](references/data-model.md).
2. Create tables/collections, indexes, security rules and the media bucket —
   [firestore-backend.md](references/firestore-backend.md) or
   [supabase-backend.md](references/supabase-backend.md).
3. Build the device and admin endpoints, including pairing and token
   verification — [api-routes.md](references/api-routes.md).
4. Get a screen paired — [pairing.md](references/pairing.md) — then drop in the
   player loop — [player-runtime.md](references/player-runtime.md).
5. Build the back-office: media library, display list, playlist editor —
   [admin-ui.md](references/admin-ui.md).
6. Add health, preview and remote control before going live —
   [operations.md](references/operations.md).
7. Verify against the behaviour contract table in
   [player-runtime.md](references/player-runtime.md) — pull the network cable,
   delete the current ad mid-loop, issue a reload — and ship the
   player-machine tests as regression cover.

## Reference directory

Load the reference matching the trigger keywords. For greenfield design, read
`data-model.md` first.

Code in `api-routes.md` and `operations.md` is written against Firestore as
the canonical backend; `supabase-backend.md` defines every substitution — the
junction table replacing `adIds`, and SQL equivalents of the service
functions.

| Scenario | Trigger keywords | Reference |
|---|---|---|
| Fitting this into an existing app | adapt, rename, seam, integrate, tenant, host app | [adaptation.md](references/adaptation.md) |
| Entities, fields, playlist shape | schema, model, Ad, Display, playlist, junction, soft delete | [data-model.md](references/data-model.md) |
| Firestore/Firebase backend | Firestore, firebase-admin, security rules, composite index, Firebase Storage | [firestore-backend.md](references/firestore-backend.md) |
| Supabase/Postgres backend | Supabase, Postgres, RLS, migration, storage bucket, Realtime | [supabase-backend.md](references/supabase-backend.md) |
| Endpoints, pairing, tokens, caching | API route, pairing, PIN, token, ETag, scope check, zod | [api-routes.md](references/api-routes.md) |
| Getting a screen paired | pairing, PIN, provisioning URL, hub, credential, unpair | [pairing.md](references/pairing.md) |
| The TV player loop | player, loop, crossfade, video stall, rotation, portrait, wake lock, offline | [player-runtime.md](references/player-runtime.md) |
| Back-office UI | admin, upload, media library, playlist editor, provisioning URL | [admin-ui.md](references/admin-ui.md) |
| Health, preview, remote control | heartbeat, last seen, offline, preview, reload, blank, emergency takeover, audit log | [operations.md](references/operations.md) |
| Scheduling, reuse, reporting | dayparting, start date, campaign, shared playlist, proof of play, precache, multi-zone | [extensions.md](references/extensions.md) |
| Why the templates differ from a naive port | provenance, defect, hardening, rationale | [provenance.md](references/provenance.md) |
