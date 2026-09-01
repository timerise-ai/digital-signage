# Operations

The layer that decides whether the module is usable at 40 screens or only at
two. A signage back-office that answers "what is configured" but never "what is
happening" fails the same way every time: a screen dies, nobody notices, and the
first report comes from a customer weeks later.

Four capabilities, all riding the poll channel already built in
[api-routes.md](api-routes.md).

## 1. Health

`recordHeartbeat` runs on every poll, including polls answered with a 304.

```ts
// modules/signage/displays.server.ts
export async function recordHeartbeat(
  displayId: string,
  { currentAdId, agentVersion }: { currentAdId: string | null; agentVersion?: string },
): Promise<void> {
  await getAdminDb().collection('displays').doc(displayId).update({
    lastSeenAt: FieldValue.serverTimestamp(),
    currentAdId,
    ...(agentVersion ? { agentVersion } : {}),
    // Deliberately NOT touching updatedAt: telemetry is not an edit, and
    // conflating them makes "last modified" useless in the admin list.
  });
}
```

Derive status rather than storing it — a stored status needs a cron to go stale,
and a cron that fails makes every screen look healthy.

```ts
// lib/signage-health.ts  (shared client + server)
export type DisplayHealth =
  | 'online' | 'stale' | 'offline' | 'never-seen' | 'never-paired';

/** Poll is 60s. Two missed polls = stale; ten = offline. */
export function displayHealth(
  display: { tokenHash: string | null; lastSeenAt: string | null },
  now: number = Date.now(),
): DisplayHealth {
  if (!display.tokenHash) return 'never-paired';
  // Paired but no poll yet — usually a provisioning URL minted and never
  // opened on the device. Distinct from never-paired: something is mid-setup.
  if (!display.lastSeenAt) return 'never-seen';
  const age = now - Date.parse(display.lastSeenAt);
  if (age < 3 * 60_000) return 'online';
  if (age < 10 * 60_000) return 'stale';
  return 'offline';
}
```

Pure, so it is trivially testable and usable in both the admin list and any
alerting job.

In the UI: a coloured pill per row, plus a **count of non-healthy screens at the
top of the page**. The count is what gets noticed; a pill in row 14 of a table
does not.

```tsx
const offline = displays.filter((d) => displayHealth(d) === 'offline');
{offline.length > 0 && (
  <div className="mb-4 rounded border border-red-500/40 bg-red-500/10 p-3 text-sm">
    {offline.length} screen{offline.length === 1 ? '' : 's'} offline:{' '}
    {offline.map((d) => d.name).join(', ')}
  </div>
)}
```

`currentAdId` is worth surfacing next to the pill — "online, showing *Summer
Promo*" answers the real question in one glance, and catches the case where a
screen is polling happily but stuck on one slide.

**Alerting.** A daily cron that lists screens offline for more than an hour and
emails whoever runs the venue is roughly twenty lines and is the difference
between finding out today and finding out next month.

## 2. Preview

Two variants, both cheap because the player is already a component:

**Playlist preview** — before publishing, in a dialog:

```tsx
<div style={{
  aspectRatio: orientation === 'portrait' ? '9 / 16' : '16 / 9',
  width: orientation === 'portrait' ? 240 : 480,
}}>
  <PlaylistPreview ads={resolvedAds} orientation={orientation} />
</div>
```

`PlaylistPreview` is the player with the network layer removed: pass `ads` in
directly and reuse `playerReducer` verbatim. That reuse is a direct payoff of
extracting the machine ([player-runtime.md](player-runtime.md)) — the preview is
not an approximation, it is the same loop.

**What is on screen now** — from telemetry, no device round trip: look up
`display.currentAdId` in the media library and render that ad's thumbnail beside
the screen's name.

Between them these remove almost every reason to physically walk to a screen to
check whether a change worked.

## 3. Remote control

Everything rides down on the poll response. There are two channels, and the
split is load-bearing:

- **`mode`** — durable state: `play`, `blank`, or `takeover`. Held on the
  display row and delivered on **every** poll, so it survives the device's
  daily reload and any power cycle. No acknowledgement needed.
- **`command`** — one-shot: `reload`. Acknowledgement-cleared, so a command
  issued while a screen is unplugged still runs when it comes back.

Put blank or takeover in the one-shot channel and the player's nightly reload
silently un-blanks a screen an operator meant to keep dark for a week.

No inbound connection to the device on either channel, which is what makes
this work through NAT, captive portals, and venue firewalls.

| Action | Channel | Use |
|---|---|---|
| `reload` | command | Screen is wedged, or you shipped a new player build |
| `blank` | mode | Maintenance, a private event, a screen that must go dark now |
| `takeover` | mode | Full-screen message pre-empting the playlist; optional `until` |
| `resume` | mode | Back to the playlist — ends a blank or takeover |

One endpoint accepts all four:

```ts
// app/api/admin/displays/[id]/command/route.ts
const CommandBody = z.discriminatedUnion('kind', [
  z.object({ kind: z.literal('reload') }),
  z.object({ kind: z.literal('blank') }),
  z.object({ kind: z.literal('resume') }),
  z.object({
    kind: z.literal('takeover'),
    message: z.string().min(1).max(280),
    until: z.string().datetime().nullable().default(null),
  }),
]);

export async function POST(req: NextRequest, { params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  const { activeLocationId, staff } = await requireStaffAuthWithLocation(req, 'admin');
  requireLocationScope(activeLocationId, await getDisplay(id));

  const parsed = CommandBody.safeParse(await req.json().catch(() => null));
  if (!parsed.success) return NextResponse.json({ error: 'Invalid request' }, { status: 400 });
  const body = parsed.data;

  if (body.kind === 'reload') {
    await updateDisplay(id, { command: { kind: 'reload', issuedAt: new Date().toISOString() } });
  } else {
    const mode: DisplayMode =
      body.kind === 'blank' ? { kind: 'blank' }
      : body.kind === 'resume' ? { kind: 'play' }
      : { kind: 'takeover', message: body.message, until: body.until };
    await updateDisplay(id, { mode });
  }

  await writeSignageLog({
    entity: 'display', entityId: id, action: 'COMMAND',
    detail: body.kind, staff, locationId: activeLocationId,
  });

  return NextResponse.json({ ok: true });
}
```

`until` is enforced **server-side**, at playlist resolution
([api-routes.md](api-routes.md)): an expired takeover resolves to `play` and
the screen reverts within one poll. The device never does its own date math —
its clock is whatever the installer left it on.

Reload acknowledgement is compare-and-clear, so an ack in flight cannot wipe a
newer command issued a second ago:

```ts
export async function clearCommand(displayId: string, ackIssuedAt: string): Promise<void> {
  const ref = getAdminDb().collection('displays').doc(displayId);
  await getAdminDb().runTransaction(async (tx) => {
    const snap = await tx.get(ref);
    const current = snap.data()?.command as DisplayCommand | null | undefined;
    if (current && current.issuedAt === ackIssuedAt) tx.update(ref, { command: null });
  });
}
```

The device persists the last acknowledged `issuedAt` locally **before**
reloading — otherwise the reload destroys the in-memory ack, the server
re-delivers the command on the next poll, and the screen reloads forever. See
[player-runtime.md](player-runtime.md).

`takeover` is the one worth building even if the others wait. Every venue
eventually needs to put "Range closed — safety briefing in progress" or
"Evacuate via the north exit" on every screen at once, and the alternative is
someone walking around with a USB stick. Add a **location-wide** variant that
writes the mode to every active screen in one batch.

## 4. Audit log

One append-only collection/table. Signage is where "who changed this?" gets
asked most, because the change is visible to customers.

```ts
export interface SignageLog {
  id: string;
  entity: 'ad' | 'display';
  entityId: string;
  entityName: string;
  action: 'CREATE' | 'UPDATE' | 'DELETE' | 'PURGE' | 'PLAYLIST' | 'COMMAND' | 'PAIR' | 'UNPAIR';
  /** Human-readable summary: "added Summer Promo at position 3". */
  detail: string;
  locationId: string;
  performedBy: string;
  performedByName: string;
  createdAt: string;
}
```

Write it in the route handler, where the authenticated staff identity is
available — not in the service layer, which does not know who is calling.

Index on `(locationId, createdAt desc)` and surface the last 50 entries on the
screens page. Playlist edits are the highest-value rows: they are frequent,
consequential, and otherwise completely untraceable.

## Unpairing

Revocation is a single field write, which is the entire point of per-display
tokens:

```ts
// app/api/admin/displays/[id]/unpair/route.ts
await updateDisplay(id, { tokenHash: null, pairedAt: null });
```

The screen's next poll gets a 401 and returns itself to the pairing hub. Compare
with a deployment-wide shared token, where the only revocation is rotating an
environment variable and physically re-pairing every screen.

Offer it whenever a device is lost, replaced, or moved, and show `pairedAt` in
the UI so an admin can tell a paired screen from a configured-but-never-deployed
one.

## Ownership checklist

Before a signage module is genuinely operable:

- [ ] Health pill per screen, plus an offline count at the top of the page
- [ ] `currentAdId` shown next to the pill
- [ ] Playlist preview in the editor
- [ ] Reload command; blank, takeover, and resume modes
- [ ] Location-wide emergency takeover
- [ ] Unpair, with `pairedAt` visible
- [ ] Audit log on ads, displays, playlists, and commands
- [ ] Loop duration in the playlist editor
- [ ] "Used by N screens" before any destructive action
- [ ] An alert when a screen has been offline for more than an hour
