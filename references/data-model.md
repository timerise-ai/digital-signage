# Data model

Backend-neutral. Firestore and Postgres shapes derive from this; see
[firestore-backend.md](firestore-backend.md) and
[supabase-backend.md](supabase-backend.md).

Put this file at `types/signage.ts`. Every other reference imports from it.

## Entities

Three, plus an optional fourth:

| Entity | Role |
|---|---|
| `Ad` | A media asset: one image, GIF, or video, with playback duration and targeting. |
| `Display` | A physical screen: its identity, playlist, pairing credential, and health. |
| `SignageLog` | Audit trail of who changed what. See [operations.md](operations.md). |
| `Playlist` *(optional)* | A named, reusable ordered list. Only when screens share content — see [extensions.md](extensions.md). |

## Types

```ts
// types/signage.ts
export type AdType = 'image' | 'gif' | 'video';
export type DisplayOrientation = 'landscape' | 'portrait';

/** A media asset in the library. */
export interface Ad {
  id: string;
  name: string;
  type: AdType;
  /** Public or signed URL the device loads. */
  url: string;
  /** Storage object key — required to delete or replace the file. */
  storagePath: string;
  /** Seconds on screen. Ignored for video, which plays to its natural end. */
  duration: number;
  /** Operator intent: is this ad in rotation? Toggled from the admin UI. */
  active: boolean;
  /** Lifecycle: set once, never unset. Distinct from `active` — see below. */
  deletedAt: string | null;
  /** Tenant scope. An ad may run at several locations. */
  locationIds: string[];
  // Captured at upload; used for warnings and admin display.
  width?: number;
  height?: number;
  /** Natural length of a video, seconds. */
  videoDuration?: number;
  fileSize?: number;
  mimeType?: string;
  createdAt: string;  // ISO 8601
  updatedAt: string;
}

/** A registered screen. */
export interface Display {
  id: string;
  name: string;
  /** Tenant scope. A screen hangs in exactly one place. */
  locationId: string;
  /** The playlist: ordered ad ids. Order is playback order. */
  adIds: string[];
  active: boolean;
  deletedAt: string | null;
  orientation: DisplayOrientation;
  nativeWidth?: number;
  nativeHeight?: number;

  // --- Pairing. Never store the token itself. ---
  /** SHA-256 of the pairing token. null = unpaired. */
  tokenHash: string | null;
  pairedAt: string | null;

  // --- Telemetry, written by the device's poll. ---
  lastSeenAt: string | null;
  currentAdId: string | null;
  /** Player build/version, for spotting a stale device. */
  agentVersion: string | null;

  // --- Remote control, read by the device's poll. ---
  /** Durable operator state: survives device reloads and power cycles. */
  mode: DisplayMode;
  /** One-shot command, cleared when the device acknowledges it. */
  command: DisplayCommand | null;

  createdAt: string;
  updatedAt: string;
}

/**
 * Durable operator state, delivered on every poll. Blank and takeover live
 * here, not in `command`: a one-shot command evaporates on the device's daily
 * reload or a power cycle, silently un-blanking a screen an operator meant to
 * keep dark. See operations.md.
 */
export type DisplayMode =
  | { kind: 'play' }
  | { kind: 'blank' }
  | { kind: 'takeover'; message: string; until: string | null };

/** One-shot: issued by an admin, acknowledged and cleared by the device. */
export type DisplayCommand = { kind: 'reload'; issuedAt: string };

export type AdInput = Omit<Ad, 'id' | 'createdAt' | 'updatedAt' | 'deletedAt'>;
export type DisplayInput = Omit<
  Display,
  | 'id' | 'createdAt' | 'updatedAt' | 'deletedAt'
  | 'tokenHash' | 'pairedAt' | 'lastSeenAt' | 'currentAdId' | 'agentVersion'
  | 'mode' | 'command'
>;
```

Fields excluded from `DisplayInput` are server-owned: an admin form can never
write a token hash or forge telemetry.

## The wire payload

Deliberately narrower than `Ad`. The device gets what it needs to render and
nothing else — no storage paths, no targeting, no file sizes.

```ts
export interface AdItem {
  id: string;
  name: string;
  type: AdType;
  url: string;
  duration: number;
}

export interface DisplayInfo {
  id: string;
  name: string;
  locationId: string;
  orientation: DisplayOrientation;
  nativeWidth: number | null;
  nativeHeight: number | null;
}

export interface PlaylistResponse {
  display: DisplayInfo | null;
  ads: AdItem[];
  /** Shown when the playlist is empty — a venue logo beats a black screen. */
  fallbackLogoUrl: string | null;
  /** Durable state, sent on every poll — no acknowledgement needed. */
  mode: DisplayMode;
  command: DisplayCommand | null;
}
```

## Design decision: array vs junction

`display.adIds: string[]` **is** the playlist.

| | Embedded array | Junction table |
|---|---|---|
| Ordering | Free — array index | `position` column, renumber on reorder |
| Reads to render | 1 display + N ads | 1 display + 1 join + N ads |
| Reorder write | One field update | Multi-row transaction |
| Same list on many screens | Copy per screen | One playlist, many screens |
| Referential integrity | Manual cleanup on ad delete | `ON DELETE CASCADE` |
| Duplicates in a loop | Allowed by shape | Allowed via `position` |

**Start with the array.** It is one read, ordering is free, and the reorder
write is atomic. In Firestore it is the only sane choice.

**In Postgres, use `display_ads(display_id, ad_id, position)` anyway** — you get
cascade delete for free (fixing dangling references structurally) and the
migration to shared playlists later is a schema change instead of a rewrite. The
API surface stays identical: the route reads the junction and returns the same
ordered `AdItem[]`.

**Move to a named `Playlist` entity when** the same content runs on more than a
couple of screens. That threshold arrives fast: with per-screen playlists,
changing a campaign on ten screens means ten edits. See
[extensions.md](extensions.md).

## Design decision: `active` and `deletedAt` are different things

They are routinely collapsed into one boolean. Do not do it.

- `active: false` — **operator intent.** "Pause this, I will bring it back."
  Reversible from the UI, and the operator expects it to be.
- `deletedAt: string` — **lifecycle.** "This is gone." Set once. Filtered out of
  every list query.

Collapse them and a deleted ad shows up in the library as merely inactive, where
toggling Active silently resurrects it. Two fields, two meanings, no ambiguity.

**Both delete paths:**

- `DELETE /api/admin/ads/:id` → soft: set `deletedAt`, leave the file in storage.
- `DELETE /api/admin/ads/:id?permanent=true` → hard: remove the row, **delete the
  storage object**, and **strip the id from every `display.adIds`**. All three, or
  you leak storage and break playlists.

## Playback semantics

Contract the player and the API both honour:

1. **Order is `adIds` order.** Duplicates play twice; that is a valid loop.
2. **Inactive, deleted, and missing ads are dropped silently** at resolution
   time. A playlist can legitimately resolve to zero items.
3. **Empty playlist is not an error** — the player shows `fallbackLogoUrl`, or
   black if there is none.
4. **`duration` applies to `image` and `gif` only.** Video plays to its natural
   end, capped by a safety timeout ([player-runtime.md](player-runtime.md)).
5. **Resolve exactly the referenced ads** — fetch by the ids in `adIds`, never
   "list the newest N ads and filter". A capped list query silently drops older
   ads from playlists once the library outgrows the cap.

## Tenant scoping

Note the deliberate asymmetry:

- **`Ad.locationIds: string[]`** — one asset can run in many venues.
- **`Display.locationId: string`** — a screen hangs in exactly one place.

Scope is **always** derived server-side: from the staff session for admin
routes, and from the display row the token resolves to for device routes. The
device never sends a location id — it sends a token, and the server looks up
where that screen lives. See [api-routes.md](api-routes.md).
