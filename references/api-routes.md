# API routes

Two audiences, two auth models:

| Surface | Caller | Credential |
|---|---|---|
| `/api/display/*` | Unattended TV | Per-display bearer token, minted at pairing |
| `/api/admin/*` | Staff browser | Existing staff session, min role `admin` |

Next.js 16 App Router: **`params` is a Promise** — always `const { id } = await params`.

## Environment

```bash
SIGNAGE_TOKEN_PEPPER=<32+ random bytes, base64>   # server-only, rotates all tokens
```

That is the only signage secret. There is deliberately **no** shared
`DISPLAY_ACCESS_TOKEN`-style variable: one token for every screen cannot be
revoked for a single screen, and anyone who reads it from one TV owns them all.

## Token minting and verification

```ts
// modules/signage/tokens.server.ts
import 'server-only';
import { createHash, randomBytes } from 'node:crypto';

/** Returned to the device exactly once, at pairing. Never stored in plaintext. */
export function mintDisplayToken(): string {
  return randomBytes(32).toString('base64url');
}

export function hashDisplayToken(token: string): string {
  const pepper = process.env.SIGNAGE_TOKEN_PEPPER;
  if (!pepper) throw new Error('SIGNAGE_TOKEN_PEPPER is not set');
  return createHash('sha256').update(`${pepper}:${token}`).digest('hex');
}
```

Lookup is by `tokenHash`, which is indexed and unique — so verification is one
indexed read, and revoking a screen is `tokenHash = null` on one row.

## Pairing — `POST /api/display/pair`

Two steps through one endpoint, mirroring the two screens of the pairing UI.

- `{ pin }` → the location's screens. **No token is issued.**
- `{ pin, displayId }` → mints and returns the token for that one screen.

```ts
// app/api/display/pair/route.ts
import { NextResponse, type NextRequest } from 'next/server';
import { z } from 'zod';
import { getLocationByPin } from '@/modules/locations/locations.server';
import { listDisplays, claimDisplay } from '@/modules/signage/displays.server';
import { mintDisplayToken, hashDisplayToken } from '@/modules/signage/tokens.server';
import { rateLimit } from '@/lib/rate-limit';

const Body = z.object({
  pin: z.string().regex(/^\d{4,8}$/),
  displayId: z.string().min(1).optional(),
});

export async function POST(req: NextRequest) {
  // A 4-8 digit PIN is brute-forceable in minutes without this.
  const ip = req.headers.get('x-forwarded-for')?.split(',')[0]?.trim() ?? 'unknown';
  if (!(await rateLimit(`pair:${ip}`, { max: 10, windowMs: 60_000 }))) {
    return NextResponse.json({ error: 'Too many attempts' }, { status: 429 });
  }

  const parsed = Body.safeParse(await req.json().catch(() => null));
  if (!parsed.success) {
    return NextResponse.json({ error: 'Invalid request' }, { status: 400 });
  }
  const { pin, displayId } = parsed.data;

  const location = await getLocationByPin(pin);
  // Same response for a bad PIN and a PIN with no screens: do not leak which.
  if (!location) return NextResponse.json({ error: 'Invalid PIN' }, { status: 401 });

  const displays = await listDisplays(location.id, { activeOnly: true });

  if (!displayId) {
    return NextResponse.json({
      locationId: location.id,
      locationName: location.name,
      displays: displays.map((d) => ({
        id: d.id,
        name: d.name,
        adCount: d.adIds.length,
        paired: d.tokenHash != null,
      })),
    });
  }

  const target = displays.find((d) => d.id === displayId);
  if (!target) return NextResponse.json({ error: 'Unknown display' }, { status: 404 });

  // Re-pairing mints a fresh token and invalidates the old one, which is how a
  // lost or stolen TV stick is dealt with.
  const token = mintDisplayToken();
  await claimDisplay(target.id, hashDisplayToken(token));

  return NextResponse.json({
    token,                       // the only time this value ever leaves the server
    displayId: target.id,
    displayName: target.name,
    locationId: location.id,
  });
}
```

The device never learns or sends a `locationId` for playback — the server
derives it from the screen the token resolves to.

PIN storage and comparison belong to the host's `getLocationByPin` — a seam,
per [adaptation.md](adaptation.md). Store PINs hashed, or at minimum compare
them in constant time.

### Rate limiting

```ts
// lib/rate-limit.ts — single-instance only; see the caveat below.
const buckets = new Map<string, { count: number; resetAt: number }>();

export async function rateLimit(
  key: string,
  { max, windowMs }: { max: number; windowMs: number },
): Promise<boolean> {
  const now = Date.now();
  const b = buckets.get(key);
  if (!b || now > b.resetAt) {
    buckets.set(key, { count: 1, resetAt: now + windowMs });
    return true;
  }
  if (b.count >= max) return false;
  b.count += 1;
  return true;
}
```

**This is per-instance.** On serverless it resets on cold start and does not
share state between concurrent instances, so it raises the cost of brute force
without ending it. For real protection use a shared store (Upstash Redis,
`@vercel/kv`) or a platform firewall rate-limit rule on the pairing path.

## Playback — `POST /api/display/playlist`

The single endpoint that sustains the runtime: telemetry up, content and
commands down.

```ts
// app/api/display/playlist/route.ts
import { NextResponse, type NextRequest } from 'next/server';
import { createHash } from 'node:crypto';
import { z } from 'zod';
import { hashDisplayToken } from '@/modules/signage/tokens.server';
import {
  getDisplayByTokenHash, recordHeartbeat, clearCommand,
} from '@/modules/signage/displays.server';
import { getAdsByIds } from '@/modules/signage/ads.server';
import { getLocation } from '@/modules/locations/locations.server';
import type { AdItem, DisplayMode, PlaylistResponse } from '@/types/signage';

const Body = z.object({
  token: z.string().min(1),
  /** Telemetry — what the screen is showing right now. */
  currentAdId: z.string().nullable().optional(),
  agentVersion: z.string().max(32).optional(),
  /** Set when the device has carried out the command it was given. */
  ackCommandAt: z.string().datetime().optional(),
});

export async function POST(req: NextRequest) {
  const parsed = Body.safeParse(await req.json().catch(() => null));
  if (!parsed.success) {
    return NextResponse.json({ error: 'Invalid request' }, { status: 400 });
  }
  const { token, currentAdId, agentVersion, ackCommandAt } = parsed.data;

  const display = await getDisplayByTokenHash(hashDisplayToken(token));
  if (!display || !display.active || display.deletedAt) {
    // Covers revoked, deactivated and deleted screens alike. The player treats
    // 401 as "stop and show the pairing screen".
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  await recordHeartbeat(display.id, { currentAdId: currentAdId ?? null, agentVersion });
  if (ackCommandAt) await clearCommand(display.id, ackCommandAt);

  const [ads, location] = await Promise.all([
    getAdsByIds(display.adIds),
    getLocation(display.locationId),
  ]);

  // A takeover past its `until` resolves to `play` here, server-side. The
  // device never does its own date math — its clock is whatever the installer
  // left it on — so it only ever renders the mode it is handed.
  const mode: DisplayMode =
    display.mode.kind === 'takeover' && display.mode.until &&
    display.mode.until <= new Date().toISOString()
      ? { kind: 'play' }
      : display.mode;

  const payload: PlaylistResponse = {
    display: {
      id: display.id,
      name: display.name,
      locationId: display.locationId,
      orientation: display.orientation,
      nativeWidth: display.nativeWidth ?? null,
      nativeHeight: display.nativeHeight ?? null,
    },
    ads: ads.map(({ id, name, type, url, duration }): AdItem =>
      ({ id, name, type, url, duration })),
    fallbackLogoUrl: location?.logoUrl ?? null,
    mode,
    command: ackCommandAt ? null : display.command,
  };

  // ETag over the *content*, so an unchanged playlist costs the device nothing
  // to re-poll. Telemetry above still ran — a 304 is not a skipped heartbeat.
  const etag = `"${createHash('sha1').update(JSON.stringify(payload)).digest('base64url')}"`;
  if (req.headers.get('if-none-match') === etag) {
    return new NextResponse(null, { status: 304, headers: { ETag: etag } });
  }

  return NextResponse.json(payload, {
    headers: { ETag: etag, 'Cache-Control': 'private, no-cache' },
  });
}
```

`no-cache` means "revalidate every time", not "do not store" — exactly the
behaviour an ETag needs.

**Cost note.** Every poll writes `lastSeenAt`. Twenty screens on a 60-second
poll is ~29k writes/day. If that matters, gate the write inside `recordHeartbeat`
to at most once per N minutes — at the cost of coarser offline detection.

## Admin routes

Standard staff-authenticated CRUD. The parts worth copying exactly:

```ts
// app/api/admin/ads/route.ts
import { NextResponse, type NextRequest } from 'next/server';
import { z } from 'zod';
import { requireStaffAuthWithLocation, StaffAuthError } from '@/lib/staff-auth';
import { listAds, createAd } from '@/modules/signage/ads.server';

const AdBody = z.object({
  name: z.string().min(1).max(120),
  type: z.enum(['image', 'gif', 'video']),
  url: z.string().url(),
  storagePath: z.string().min(1),
  duration: z.number().int().min(1).max(600),
  active: z.boolean().default(true),
  locationIds: z.array(z.string()).default([]),
  width: z.number().int().positive().optional(),
  height: z.number().int().positive().optional(),
  videoDuration: z.number().positive().optional(),
  fileSize: z.number().int().positive().optional(),
  mimeType: z.string().optional(),
});

export async function GET(req: NextRequest) {
  try {
    const { activeLocationId } = await requireStaffAuthWithLocation(req, 'admin');
    const includeInactive = new URL(req.url).searchParams.get('includeInactive') === 'true';
    return NextResponse.json(await listAds(activeLocationId, { includeInactive }));
  } catch (err) {
    return toErrorResponse(err, 'admin/ads GET');
  }
}

export async function POST(req: NextRequest) {
  try {
    const { activeLocationId } = await requireStaffAuthWithLocation(req, 'admin');
    const parsed = AdBody.safeParse(await req.json().catch(() => null));
    if (!parsed.success) {
      return NextResponse.json(
        { error: 'Invalid request', issues: parsed.error.flatten().fieldErrors },
        { status: 400 },
      );
    }
    // The active location is always included, whatever the client sent.
    const locationIds = [...new Set([...parsed.data.locationIds, activeLocationId])];
    return NextResponse.json(await createAd({ ...parsed.data, locationIds }), { status: 201 });
  } catch (err) {
    return toErrorResponse(err, 'admin/ads POST');
  }
}

function toErrorResponse(err: unknown, tag: string) {
  if (err instanceof StaffAuthError) return err.toResponse();
  console.error(`[${tag}]`, err);                  // log detail server-side
  return NextResponse.json({ error: 'Internal error' }, { status: 500 });
}
```

Never return the caught error's message to the client — log it and answer with a
constant.

### Scope enforcement on `[id]` routes

The step most often skipped, and the one that makes every other tenant's records
reachable by id.

```ts
// app/api/admin/ads/[id]/route.ts
export async function PUT(
  req: NextRequest,
  { params }: { params: Promise<{ id: string }> },   // Next 16: params is a Promise
) {
  try {
    const { id } = await params;
    const { activeLocationId } = await requireStaffAuthWithLocation(req, 'admin');

    const existing = await getAd(id);
    requireLocationScope(activeLocationId, existing);   // throws 404 if out of scope

    const parsed = AdBody.partial().safeParse(await req.json().catch(() => null));
    if (!parsed.success) return NextResponse.json({ error: 'Invalid request' }, { status: 400 });

    await updateAd(id, parsed.data);

    // The edit swapped the file: the row now points at a new object, so the
    // old one is orphaned — delete it here, where the trusted previous path is
    // known. Never delete a client-named path; that would let any admin body
    // aim the delete at an arbitrary storage object.
    if (parsed.data.storagePath && parsed.data.storagePath !== existing.storagePath) {
      await deleteStorageObject(existing.storagePath);
    }

    return NextResponse.json({ ok: true });
  } catch (err) {
    return toErrorResponse(err, 'admin/ads PUT');
  }
}
```

```ts
// lib/staff-auth.ts
/**
 * 404, not 403 — a 403 confirms the record exists, letting a caller enumerate
 * other tenants' ids. Handles both scoping shapes.
 */
export function requireLocationScope(
  activeLocationId: string,
  doc: { locationId?: string; locationIds?: string[] } | null,
): void {
  if (!doc) throw new StaffAuthError(404, 'Not found');
  const inScope = Array.isArray(doc.locationIds)
    ? doc.locationIds.includes(activeLocationId)
    : doc.locationId === activeLocationId;
  if (!inScope) throw new StaffAuthError(404, 'Not found');
}
```

### Delete semantics

One endpoint, two behaviours, matching [data-model.md](data-model.md):

```ts
export async function DELETE(
  req: NextRequest,
  { params }: { params: Promise<{ id: string }> },
) {
  const { id } = await params;
  const { activeLocationId } = await requireStaffAuthWithLocation(req, 'admin');
  requireLocationScope(activeLocationId, await getAd(id));

  const permanent = new URL(req.url).searchParams.get('permanent') === 'true';
  // purgeAd removes the row, deletes the storage object, and strips the id
  // from every playlist. Skipping any one of those leaks or breaks something.
  await (permanent ? purgeAd(id) : softDeleteAd(id));
  return NextResponse.json({ ok: true });
}
```

### Full route surface

| Route | Method | Purpose |
|---|---|---|
| `/api/display/pair` | POST | List screens for a PIN, then mint a token for one |
| `/api/display/playlist` | POST | Poll: telemetry up, content + command down |
| `/api/admin/ads` | GET, POST | Media library |
| `/api/admin/ads/[id]` | PUT, DELETE | Edit, soft delete, `?permanent=true` purge |
| `/api/admin/ads/[id]/usage` | GET | Which displays reference this ad |
| `/api/admin/displays` | GET, POST | Screen registry |
| `/api/admin/displays/[id]` | PUT, DELETE | Edit incl. `adIds` reorder, delete |
| `/api/admin/displays/[id]/command` | POST | Issue reload / blank / takeover / resume |
| `/api/admin/displays/[id]/unpair` | POST | `tokenHash = null` — instant revoke |

The last three are covered in [operations.md](operations.md).

## Testing

Route handlers are plain functions — import and call them with a stub request,
mocking the service module. No server, no emulator:

```ts
vi.mock('@/modules/signage/displays.server', () => ({
  getDisplayByTokenHash: vi.fn(),
  recordHeartbeat: vi.fn(),
  clearCommand: vi.fn(),
}));

import { POST } from '@/app/api/display/playlist/route';

it('rejects a revoked token', async () => {
  vi.mocked(getDisplayByTokenHash).mockResolvedValue(null);
  const req = { json: async () => ({ token: 'nope' }) } as never;
  expect((await POST(req)).status).toBe(401);
});
```

To exercise a `catch` → 500 branch, throw at the `req.json()` boundary rather
than from a service mock.
