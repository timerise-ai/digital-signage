# Supabase / Postgres backend

The same model as [data-model.md](data-model.md), normalized. Postgres earns its
keep here: the junction table gives referential integrity for free, so the
dangling-playlist-reference problem that needs manual cleanup in Firestore
cannot occur.

> Load the `supabase-postgres-best-practices` skill before writing migrations if
> it is available — it governs schema, RLS, and index conventions.

## Schema

`supabase/migrations/0001_signage.sql`:

```sql
create type ad_type as enum ('image', 'gif', 'video');
create type display_orientation as enum ('landscape', 'portrait');

create table public.ads (
  id             uuid primary key default gen_random_uuid(),
  name           text not null check (length(trim(name)) > 0),
  type           ad_type not null,
  url            text not null,
  storage_path   text not null,
  -- Seconds on screen. Ignored for video, which plays to its natural end.
  duration       integer not null default 10 check (duration between 1 and 600),
  -- Operator intent ("paused"), distinct from lifecycle ("gone").
  active         boolean not null default true,
  deleted_at     timestamptz,
  width          integer,
  height         integer,
  video_duration numeric(8,2),
  file_size      bigint,
  mime_type      text,
  created_at     timestamptz not null default now(),
  updated_at     timestamptz not null default now()
);

-- An ad may run at several locations.
create table public.ad_locations (
  ad_id       uuid not null references public.ads(id) on delete cascade,
  location_id uuid not null references public.locations(id) on delete cascade,
  primary key (ad_id, location_id)
);
create index ad_locations_location_idx on public.ad_locations(location_id);

create table public.displays (
  id             uuid primary key default gen_random_uuid(),
  -- A screen hangs in exactly one place. restrict: never orphan a screen.
  location_id    uuid not null references public.locations(id) on delete restrict,
  name           text not null check (length(trim(name)) > 0),
  active         boolean not null default true,
  deleted_at     timestamptz,
  orientation    display_orientation not null default 'landscape',
  native_width   integer,
  native_height  integer,

  -- Pairing: only the hash is ever stored.
  token_hash     text unique,
  paired_at      timestamptz,

  -- Telemetry, written by the device poll.
  last_seen_at   timestamptz,
  current_ad_id  uuid references public.ads(id) on delete set null,
  agent_version  text,

  -- Durable operator state (play / blank / takeover), read by the device poll.
  mode           jsonb not null default '{"kind": "play"}'::jsonb,
  -- One-shot command (reload), cleared on device acknowledgement.
  command        jsonb,

  created_at     timestamptz not null default now(),
  updated_at     timestamptz not null default now()
);
create index displays_location_idx on public.displays(location_id)
  where deleted_at is null;

-- The playlist. Composite PK on (display_id, position) permits the same ad
-- more than once in a loop, which is a legitimate schedule.
create table public.display_ads (
  display_id uuid not null references public.displays(id) on delete cascade,
  ad_id      uuid not null references public.ads(id)      on delete cascade,
  position   integer not null check (position >= 0),
  primary key (display_id, position)
);
create index display_ads_ad_idx on public.display_ads(ad_id);
```

`on delete cascade` on `display_ads.ad_id` is the whole argument for the
junction: purging an ad cannot leave a broken playlist behind.

`display_ads_ad_idx` powers the admin's "used by N displays" reverse lookup
before a destructive action.

### `updated_at` trigger

```sql
create or replace function public.touch_updated_at() returns trigger
language plpgsql as $$
begin new.updated_at = now(); return new; end;
$$;

create trigger ads_touch before update on public.ads
  for each row execute function public.touch_updated_at();

-- Telemetry is not an edit: heartbeats and command acks must not bump
-- updated_at, or "last modified" in the admin list becomes meaningless.
-- The trigger fires only when none of the device-written columns changed.
create trigger displays_touch before update on public.displays
  for each row
  when (
    old.last_seen_at  is not distinct from new.last_seen_at
    and old.current_ad_id is not distinct from new.current_ad_id
    and old.agent_version is not distinct from new.agent_version
    and old.command       is not distinct from new.command
  )
  execute function public.touch_updated_at();
```

## Playlist resolution

One query, ordered, integrity-checked. This is the Postgres payoff:

```sql
select a.id, a.name, a.type, a.url, a.duration
from public.display_ads da
join public.ads a on a.id = da.ad_id
where da.display_id = $1
  and a.active
  and a.deleted_at is null
order by da.position;
```

Inactive and deleted ads drop out; ordering is explicit; a purged ad is already
gone. No in-memory reconciliation, no chunked `in` queries, no cap that silently
truncates the library.

## Reordering a playlist

Replace the whole list atomically. Renumbering rows one at a time collides with
the composite primary key halfway through.

```sql
create or replace function public.set_display_playlist(
  p_display_id uuid,
  p_ad_ids uuid[]
) returns void
language plpgsql
security invoker           -- run as the caller so RLS still applies
set search_path = ''
as $$
begin
  delete from public.display_ads where display_id = p_display_id;
  insert into public.display_ads (display_id, ad_id, position)
  select p_display_id, ad_id, ordinality - 1
  from unnest(p_ad_ids) with ordinality as t(ad_id, ordinality);
end;
$$;
```

`with ordinality` preserves the array's order, including duplicates.
`security invoker` is deliberate — a `security definer` function here would let
any caller rewrite any screen's playlist.

## RLS

Enable on every table. Two audiences: staff (admin CRUD) and devices (read-only,
served exclusively through a service-role route).

```sql
alter table public.ads         enable row level security;
alter table public.ad_locations enable row level security;
alter table public.displays    enable row level security;
alter table public.display_ads enable row level security;

-- Which locations may the signed-in staff member touch?
create or replace function public.has_location_access(p_location_id uuid)
returns boolean
language sql stable security definer set search_path = ''
as $$
  select exists (
    select 1 from public.staff_locations sl
    where sl.staff_id = (select auth.uid())
      and sl.location_id = p_location_id
  );
$$;

create policy "staff read displays" on public.displays
  for select to authenticated
  using (public.has_location_access(location_id));

create policy "staff write displays" on public.displays
  for all to authenticated
  using (public.has_location_access(location_id))
  with check (public.has_location_access(location_id));

create policy "staff read playlist" on public.display_ads
  for select to authenticated
  using (exists (
    select 1 from public.displays d
    where d.id = display_ads.display_id
      and public.has_location_access(d.location_id)
  ));

create policy "staff write playlist" on public.display_ads
  for all to authenticated
  using (exists (
    select 1 from public.displays d
    where d.id = display_ads.display_id
      and public.has_location_access(d.location_id)
  ))
  with check (exists (
    select 1 from public.displays d
    where d.id = display_ads.display_id
      and public.has_location_access(d.location_id)
  ));

create policy "staff read ads" on public.ads
  for select to authenticated
  using (exists (
    select 1 from public.ad_locations al
    where al.ad_id = ads.id and public.has_location_access(al.location_id)
  ));
```

Mirror the write policy for `ads` and `ad_locations`.

**There is no `anon` policy, by design.** The device does not hold a Supabase
JWT; it holds a pairing token this schema knows nothing about. Device reads are
served by a Next.js route handler using the **service-role key**, which bypasses
RLS *after* the route has verified the token and resolved the screen. Never ship
the service-role key to a client. See [api-routes.md](api-routes.md).

The `with check` clauses matter as much as `using`: without them a staff member
could move a row to a location they do not have access to.

## Clients

```ts
// lib/supabase/server.ts
import 'server-only';
import { createServerClient } from '@supabase/ssr';
import { createClient } from '@supabase/supabase-js';
import { cookies } from 'next/headers';

/** Staff-scoped: RLS applies, uses the caller's session. */
export async function getSupabaseServer() {
  const store = await cookies();
  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll: () => store.getAll(),
        setAll: (list) => list.forEach(({ name, value, options }) =>
          store.set(name, value, options)),
      },
    },
  );
}

/**
 * Device-scoped: bypasses RLS. Only ever call after verifying a pairing token
 * and scoping the query to the display that token resolved to.
 */
export function getSupabaseServiceRole() {
  return createClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.SUPABASE_SERVICE_ROLE_KEY!,
    { auth: { persistSession: false, autoRefreshToken: false } },
  );
}
```

## Storage

```sql
insert into storage.buckets (id, name, public, file_size_limit, allowed_mime_types)
values ('ad-media', 'ad-media', true, 10485760,
        array['image/png','image/jpeg','image/webp','image/gif',
              'video/mp4','video/webm'])
on conflict (id) do nothing;

create policy "public read ad media" on storage.objects
  for select to public using (bucket_id = 'ad-media');

-- Staff only: a bare `to authenticated` would let any signed-in customer
-- fill a public-read bucket.
create policy "staff upload ad media" on storage.objects
  for insert to authenticated
  with check (
    bucket_id = 'ad-media'
    and exists (
      select 1 from public.staff_locations sl
      where sl.staff_id = (select auth.uid())
    )
  );

-- No delete policy: purge runs server-side with the service-role key.
```

Unlike Firebase Storage rules, `file_size_limit` and `allowed_mime_types` are
enforced by the bucket itself. Still validate client-side — a rejection after a
10 MB upload is a bad experience either way.

```ts
// lib/signage-upload.ts  (client)
import { createBrowserClient } from '@supabase/ssr';

export const MAX_AD_BYTES = 10 * 1024 * 1024;
export const ALLOWED_AD_MIME = /^(image|video)\//;

export async function uploadAdFile(file: File): Promise<{ url: string; storagePath: string }> {
  if (file.size > MAX_AD_BYTES) {
    throw new Error(`File is ${(file.size / 1e6).toFixed(1)} MB; the limit is 10 MB.`);
  }
  if (!ALLOWED_AD_MIME.test(file.type)) {
    throw new Error(`Unsupported file type: ${file.type || 'unknown'}.`);
  }

  const supabase = createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
  );
  const safeName = file.name.replace(/[^a-zA-Z0-9._-]/g, '_');
  const storagePath = `${Date.now()}_${safeName}`;

  const { error } = await supabase.storage.from('ad-media').upload(storagePath, file, {
    contentType: file.type,
    cacheControl: '31536000',   // path is unique per upload; cache hard
    upsert: false,
  });
  if (error) throw error;

  const { data } = supabase.storage.from('ad-media').getPublicUrl(storagePath);
  return { url: data.publicUrl, storagePath };
}
```

Supabase's JS upload does not report progress. For a visible progress bar, use a
signed upload URL with `XMLHttpRequest` and its `upload.onprogress` event.

Purge, server-side, mirroring the Firestore version:

```ts
const svc = getSupabaseServiceRole();
const { data: ad } = await svc.from('ads').select('storage_path').eq('id', id).single();
await svc.from('ads').delete().eq('id', id);          // cascade clears display_ads
if (ad) await svc.storage.from('ad-media').remove([ad.storage_path]);
```

## Device-route services

The Firestore service functions used by [api-routes.md](api-routes.md) and
[operations.md](operations.md) map to single statements on the service-role
client:

```ts
const svc = getSupabaseServiceRole();

// getDisplayByTokenHash — token_hash is unique, so one indexed read.
const { data: display } = await svc.from('displays')
  .select('*').eq('token_hash', tokenHash).maybeSingle();

// recordHeartbeat — the touch trigger above skips these columns, so
// updated_at stays an edit timestamp.
await svc.from('displays').update({
  last_seen_at: new Date().toISOString(),
  current_ad_id: currentAdId,
  ...(agentVersion ? { agent_version: agentVersion } : {}),
}).eq('id', displayId);

// clearCommand — the compare-and-clear that needs a transaction in Firestore
// is one guarded UPDATE here: only the command that was acknowledged clears,
// so an ack in flight cannot wipe a newer command issued a second ago.
await svc.from('displays').update({ command: null })
  .eq('id', displayId).eq('command->>issuedAt', ackIssuedAt);
```

Playlist resolution replaces `getAdsByIds(display.adIds)` with the ordered
join query above — the junction table is the source of order, so there is no
`adIds` array on this backend.

## Optional: Realtime instead of polling

The device has no Supabase session, so it cannot subscribe directly. To use
Realtime, mint a short-lived Supabase JWT at pairing carrying a `display_id`
claim, and add a policy scoped to it:

```sql
create policy "device reads own playlist" on public.display_ads
  for select to authenticated
  using (display_id = ((select auth.jwt()) ->> 'display_id')::uuid);
```

Then subscribe to `postgres_changes` on `display_ads` filtered by `display_id`,
and keep the poll as a slow fallback heartbeat.

**Weigh this honestly.** It adds a JWT lifecycle to unattended hardware and a
websocket that must survive months of flaky venue Wi-Fi, to save a request per
minute. Polling with an ETag is the right default; reach for Realtime when
sub-second updates genuinely matter.
