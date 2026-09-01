# Provenance

Written by the engineer who has shipped this module. The earlier implementation
it was audited against ran the screens of a multi-venue booking system; the
architecture is that implementation's, the templates are not a transcription of
it.

Ten defects were found while auditing the earlier implementation. The templates
ship the fixed version, and this file records each one, so the reasoning is
inspectable and so anyone upgrading an implementation like it recognises what
they are about to copy.

Read it when a template looks more complicated than it needs to be. It usually
encodes a failure someone has already had.

## Fixed in the templates

### 1. The refresh timer that never fires

The player fetched the playlist through a callback that closed over the current
slide index, and the interval effect listed that callback in its dependencies.
Every slide change tore down the interval and started a new one.

With a five-minute refresh behind ten-second slides, the timer never reached
zero. Screens only picked up new content when someone reloaded them — while the
code, the tests anyone would write, and the admin UI all looked correct. This is
the most consequential bug in the module and the least visible: it presents as
"the screens seem slow to update", months apart, with no error anywhere.

**Shipped:** the poll effect depends on `[token]` only and reads slide state
from a ref. See [player-runtime.md](player-runtime.md).

### 2. A side effect inside a state updater

Playlist reconciliation ran inside a `setAds(prev => …)` updater, and called
`setCurrentIndex` from within it. React may invoke an updater more than once for
the same base state; a nested state write then fires twice.

**Shipped:** the reconciliation is a pure `playerReducer` case. React can replay
it freely. It is also now testable without a DOM, which is what made the
regression test for defect 1 possible.

### 3. `active` meaning two different things

Soft delete set `active: false` — the same field the admin form's Active toggle
writes — and the list query filtered on neither. A deleted ad appeared in the
library as merely paused, and toggling it back on resurrected it.

**Shipped:** `active` is operator intent, `deletedAt` is lifecycle. Three states
in the UI: Active, Paused, Deleted. See [data-model.md](data-model.md).

### 4. A list cap that silently truncated playlists

Playlist resolution fetched "the 200 newest ads" and filtered them by
membership. Once a venue's library passed 200 items, older ads disappeared from
playback with no error, no warning, and nothing in the admin UI to suggest it.

**Shipped:** `getAdsByIds` resolves exactly the referenced ids, chunked to the
backend's limit. Scale-dependent behaviour with no failure signal is the hardest
class of bug to diagnose from a support ticket.

### 5. An unordered query behind a fallback

A device polling without a screen id got "the location's first display" from a
query with no `orderBy`. Which screen that was could change between deploys.

**Shipped:** every list query is ordered. In the templates the device always
resolves its screen from its own token, so the ambiguity is gone entirely.

### 6. Tenant scope unchecked on `[id]` routes

Collection routes derived the location from the staff session, but the `[id]`
routes did not re-check it — so any admin could read, edit, or permanently
delete another venue's ads and screens by id. A scope helper existed in the
codebase and was used on two other modules; signage simply missed it.

**Shipped:** `requireLocationScope` on every `[id]` route, returning **404 not
403** so ids cannot be probed. See [api-routes.md](api-routes.md).

### 7. Storage objects orphaned forever

A delete helper was written, exported, and never called. Every permanent delete
and every file replacement left its object in storage — up to 10 MB each, billed
indefinitely, with no way to tell which were still referenced.

**Shipped:** purge deletes the row, the object, and the playlist references
together; edits that replace a file delete the old one.

### 8. Dangling playlist references

Removing an ad left its id in every playlist that referenced it. The player
dropped unresolvable ids silently, so loops quietly got shorter.

**Shipped:** purge strips the id from every playlist in the same batch. On
Postgres, `on delete cascade` makes it structurally impossible — the main reason
[supabase-backend.md](supabase-backend.md) uses a junction table.

### 9. One token for every screen, handed out for a PIN

A single deployment-wide token lived in an environment variable. The pairing
endpoint returned it verbatim to anyone who guessed a 4–6 digit venue PIN, with
no rate limiting, and it granted access to every venue's playlist. Revoking one
screen meant rotating the variable and re-pairing all of them.

**Shipped:** a per-display token minted at pairing, stored only as a hash,
rate-limited issuance, and revocation as a single field write. The device sends
its token and nothing else — it cannot name a venue it should not see.

### 10. No validation

Request bodies were cast (`as AdInput`) with no runtime checking, and a helper
existed purely to strip `undefined` values the database would reject. A schema
validator was already a project dependency, used elsewhere.

**Shipped:** zod at every route boundary, which validates and drops absent
optionals in one step.

## Kept deliberately

Not everything unusual in the earlier implementation was wrong. These were considered and
retained:

- **Inline styles on device components.** Reads as a mistake, is not: signage
  runs on TV browsers years behind current, and inline styles remove any chance
  a stylesheet or purge step breaks a screen nobody is watching.
- **Polling rather than realtime.** A 30–60 second poll survives sleeping
  network stacks, captive portals and NAT, and reconnects for free. With an ETag
  it is cheap. Realtime is documented as an upgrade, not the default.
- **Direct browser-to-storage upload.** Route handlers have body-size limits and
  burn compute proxying bytes.
- **Soft delete as the default destructive action**, with permanent delete
  behind an explicit flag.
- **The playlist as an ordered array on the screen.** One read, free ordering,
  atomic reorder. The trade-off and the migration path are documented rather
  than pre-empted.
- **`duration` ignored for video.** Video plays to its natural end; the field
  stays on the type because images and GIFs need it.

## Added

Capabilities the earlier implementation did not have, built because their
absence is what makes a signage module unmanageable past a handful of screens:
health, preview, remote control, audit log. See [operations.md](operations.md).

Remote control was itself hardened after a second audit of the templates:

- **Durable state and one-shot commands are separate channels.** Blank and
  takeover are `mode`, held on the display row and delivered on every poll —
  a one-shot command evaporates on the device's daily reload, silently
  un-blanking a screen an operator meant to keep dark. Only `reload` stays in
  the ack-cleared command channel.
- **The reload ack is persisted before reloading.** Reloading destroys an
  in-memory ack, so the server would re-deliver the command on the next poll
  and the screen would reload forever.
- **`until` on a takeover is enforced server-side** at playlist resolution;
  the device's clock is never trusted with date math.

The earlier implementation also had no automated tests. The templates are
structured so the two pieces most worth testing, playlist resolution and the
player state machine, are pure functions callable without a database or a DOM.

## If you are upgrading an existing implementation

Fix in this order. The first item is the one users are already experiencing:

1. **Defect 1** — the poll interval. Screens are not updating.
2. **Defect 6** — cross-tenant access on `[id]` routes.
3. **Defect 9** — the shared token.
4. **Defects 3, 4, 8** — data integrity: soft-delete semantics, the list cap,
   dangling references.
5. **Defects 2, 5, 7, 10** — correctness and hygiene.
6. Then the operations layer, which is what makes the rest visible.
