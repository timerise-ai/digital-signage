# Extensions

Opt-in. None of these are in the core module; each earns its place only when the
matching need is real. Build them when the need arrives, not before.

## Scheduling and dayparting

Needed when campaigns have end dates, or content differs by time of day (a
breakfast menu, a happy-hour promo).

```ts
export interface AdSchedule {
  /** Inclusive campaign window. null = unbounded on that side. */
  startsAt: string | null;
  endsAt: string | null;
  /** 0 = Sunday … 6 = Saturday. Empty = every day. */
  daysOfWeek: number[];
  /** Local wall-clock "HH:mm". null = all day. */
  startTime: string | null;
  endTime: string | null;
}
```

Filter server-side, at playlist resolution, in the **venue's** timezone. The
device's clock is unreliable and its timezone is whatever the installer left it
on.

```ts
export function isScheduledNow(
  schedule: AdSchedule | null,
  now: Date,
  timeZone: string,
): boolean {
  if (!schedule) return true;

  const iso = now.toISOString();
  if (schedule.startsAt && iso < schedule.startsAt) return false;
  if (schedule.endsAt && iso > schedule.endsAt) return false;

  const parts = new Intl.DateTimeFormat('en-GB', {
    timeZone, weekday: 'short', hour: '2-digit', minute: '2-digit', hour12: false,
  }).formatToParts(now);
  const get = (t: string) => parts.find((p) => p.type === t)?.value ?? '';

  if (schedule.daysOfWeek.length > 0) {
    const dow = ['Sun', 'Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat'].indexOf(get('weekday'));
    if (!schedule.daysOfWeek.includes(dow)) return false;
  }

  const hhmm = `${get('hour')}:${get('minute')}`;
  if (schedule.startTime && schedule.endTime) {
    // A window that wraps midnight (22:00–02:00) is a union, not a range.
    return schedule.startTime <= schedule.endTime
      ? hhmm >= schedule.startTime && hhmm <= schedule.endTime
      : hhmm >= schedule.startTime || hhmm <= schedule.endTime;
  }
  return true;
}
```

Consequences to plan for:

- **The ETag changes with the clock**, so a scheduled playlist stops being
  cacheable across the whole poll interval. Round "now" down to the poll interval
  when computing it, or accept the extra reads.
- **A playlist can resolve to empty at 3 a.m.** and the screen falls back to the
  venue logo. Usually correct — but say so in the UI, or it reads as a fault.
- **Show the effective playlist for a chosen time** in the editor. Scheduling
  without a "what will be showing at 14:00 on Friday?" view is very hard to
  reason about.

## Reusable playlists

The array model puts one playlist on one screen. Editing a campaign across ten
screens is then ten edits — and the tenth gets forgotten.

Move when: the same content runs on more than two or three screens, or screens
fall into obvious groups (all lobby screens, all bar screens).

```sql
create table public.playlists (
  id          uuid primary key default gen_random_uuid(),
  location_id uuid not null references public.locations(id) on delete restrict,
  name        text not null,
  created_at  timestamptz not null default now(),
  updated_at  timestamptz not null default now()
);

create table public.playlist_items (
  playlist_id uuid not null references public.playlists(id) on delete cascade,
  ad_id       uuid not null references public.ads(id) on delete cascade,
  position    integer not null check (position >= 0),
  primary key (playlist_id, position)
);

alter table public.displays
  add column playlist_id uuid references public.playlists(id) on delete set null;
```

Migration, in order:

1. Add the tables and the nullable `displays.playlist_id`.
2. Backfill: one playlist per screen, named after it, items copied from
   `display_ads` / `adIds`.
3. Resolve from `playlist_id` when set, falling back to the per-screen list.
4. Deduplicate identical playlists in the UI, offering to merge them.
5. Drop the old column once nothing reads it.

Keep the per-screen override — "this one screen shows something different" is a
permanent requirement, not a transitional state.

The wire payload does not change: the device still receives an ordered
`AdItem[]`, so no player change is needed.

## Proof of play

Required the moment slots are sold to third parties: advertisers want evidence
their spot ran.

The device already reports `currentAdId` on every poll. That gives sampled
evidence for free — at a 60-second poll, a 10-second slide is usually missed. If
sampling is enough, aggregate the telemetry you already store and stop here.

For contractual reporting, batch actual playback events:

```ts
interface PlaybackEvent {
  adId: string;
  startedAt: string;
  durationMs: number;
  completed: boolean;      // false = skipped on error or cut short
}
```

Buffer them in the player, flush on the next poll (piggybacking the request you
are already making), and aggregate server-side into daily per-ad counts. Do not
write one row per impression: forty screens on a twenty-item loop generate
~10 million rows a year, and nobody queries them individually.

Report per ad per day: plays, completions, total seconds on screen, screens
reached. Buffer in memory only — persisting to `localStorage` risks a
double-count after a crash, and inflated play counts are worse than missing ones
when someone is being invoiced.

## Offline media precaching

The baseline offline story — playlist in memory, media from HTTP cache — covers
brief outages. Screens that go offline for hours, or run on genuinely bad
connections, need the media itself held locally.

```ts
// In the player, after each successful poll:
async function precache(ads: AdItem[]) {
  if (!('caches' in window)) return;
  const cache = await caches.open('signage-media-v1');

  await Promise.allSettled(ads.map(async (ad) => {
    if (await cache.match(ad.url)) return;      // already held
    const res = await fetch(ad.url, { mode: 'cors' });
    if (res.ok) await cache.put(ad.url, res);
  }));

  // Evict media no longer in the playlist, or the cache grows forever.
  const wanted = new Set(ads.map((a) => a.url));
  for (const req of await cache.keys()) {
    if (!wanted.has(req.url)) await cache.delete(req);
  }
}
```

Pair with a service worker serving cache-first for the media origin. Watch two
things: **storage quota** (a few hundred MB on TV browsers; evict before adding)
and **eviction under pressure** — treat the cache as a hint, never as a
guarantee, and keep the network path working.

## Multi-zone layouts

Splitting a screen into regions (main video, sidebar, a ticker) is the point at
which this stops being a playlist player and becomes a layout engine: a template
model, per-zone playlists, synchronisation between zones, and a visual editor.

That is a substantially larger product. Before building it, check whether two
physical screens, or pre-composed media produced in a design tool, solves the
actual need — they usually do, and they cost nothing to run.
