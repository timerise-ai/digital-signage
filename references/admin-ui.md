# Admin UI

Three surfaces: **media library** (ads), **screens** (displays + playlist), and
**operations** (health, preview, remote control — see
[operations.md](operations.md)).

Layering, thin at each level:

```
page.tsx        auth guard + shell only, no data fetching
  └── List      owns fetch/sort/filter state, opens dialogs
        └── Form   controlled inputs, hands data up via onSave
```

Tailwind is fine here — the TV-browser constraint applies only to the device.

## Upload, then save

The one flow to get exactly right. Media goes to storage first, the row second.

```tsx
async function handleSave(form: AdFormData, metadata: AdFileMetadata, file?: File) {
  let { url, storagePath } = editing ?? {};

  if (file) {
    // Outside the try/catch below: an upload failure must reject out of here so
    // the form's own `finally` clears its saving state and shows the real
    // message ("File is 14.2 MB; the limit is 10 MB"), not a generic save error.
    ({ url, storagePath } = await uploadAdFile(file, setUploadPct));
  }

  try {
    const payload = { ...form, url, storagePath, ...metadata };
    if (editing) {
      // When the file was swapped, the server sees the new storagePath differs
      // from the row's old one and deletes the orphaned object itself — the
      // client never names a path to delete. See api-routes.md.
      await apiFetch(`/api/admin/ads/${editing.id}`, {
        method: 'PUT',
        body: JSON.stringify(payload),
      });
    } else {
      await apiFetch('/api/admin/ads', { method: 'POST', body: JSON.stringify(payload) });
    }
    toast.success(editing ? 'Ad updated' : 'Ad added');
    closeForm();
    await refresh();
  } catch (err) {
    toast.error(err instanceof Error ? err.message : 'Could not save ad');
  }
}
```

Ordering rationale: a stored file with no row is a harmless orphan you can sweep
up. A row pointing at a file that failed to upload is a broken screen.

## Client-side metadata extraction

Read dimensions and video length in the browser — the server never touches the
bytes, and these values drive the duration default and the fit warnings.

```ts
export interface AdFileMetadata {
  width?: number; height?: number; videoDuration?: number;
  fileSize?: number; mimeType?: string;
}

export function detectAdType(mime: string): AdType {
  if (mime === 'image/gif') return 'gif';
  if (mime.startsWith('video/')) return 'video';
  return 'image';
}

export function extractMetadata(file: File): Promise<AdFileMetadata> {
  const base: AdFileMetadata = { fileSize: file.size, mimeType: file.type };
  const objectUrl = URL.createObjectURL(file);

  return new Promise((resolve) => {
    // Always resolve, never reject: unreadable metadata is not a reason to
    // block an upload that would otherwise work.
    const done = (extra: AdFileMetadata) => {
      URL.revokeObjectURL(objectUrl);
      resolve({ ...base, ...extra });
    };

    if (file.type.startsWith('video/')) {
      const video = document.createElement('video');
      video.preload = 'metadata';
      // Guards matter: audio-only files report 0×0, and some webm/stream
      // containers report duration: Infinity — which JSON-serialises to null
      // and breaks loop-time math. Store undefined instead.
      video.onloadedmetadata = () => done({
        width: video.videoWidth || undefined,
        height: video.videoHeight || undefined,
        videoDuration: Number.isFinite(video.duration) ? video.duration : undefined,
      });
      video.onerror = () => done({});
      video.src = objectUrl;
    } else {
      const img = new Image();
      img.onload = () => done({
        width: img.naturalWidth || undefined,
        height: img.naturalHeight || undefined,
      });
      img.onerror = () => done({});
      img.src = objectUrl;
    }
  });
}
```

Type is **detected, never chosen** — a type picker is a field users get wrong.

## Ad form

```ts
interface AdFormData {
  name: string;
  type: AdType;
  duration: number;
  active: boolean;
}
const EMPTY: AdFormData = { name: '', type: 'image', duration: 10, active: true };
```

Rules the form enforces:

- **A file is required on create, optional on edit.** `if (!editing && !file) return;`
- **Hide the duration field when `type === 'video'`.** Video plays to its natural
  end; a duration input there is a control that does nothing.
- **Show upload progress.** A 10 MB video on venue Wi-Fi is not instant, and a
  frozen dialog reads as a broken app.
- **Validate size and MIME before uploading**, with the actual numbers in the
  message. Letting the storage rule reject it means a long wait followed by an
  opaque failure.
- **Warn on orientation mismatch** — see below.

## Fit warning

The metadata is already captured; use it. A portrait ad on a landscape screen
letterboxes into two black bars, and today nobody finds out until they walk past
the screen.

```tsx
function fitWarning(ad: { width?: number; height?: number },
                    display: { orientation: DisplayOrientation }): string | null {
  if (!ad.width || !ad.height) return null;
  const adOrientation = ad.width >= ad.height ? 'landscape' : 'portrait';
  if (adOrientation === display.orientation) return null;
  return `This ad is ${adOrientation} but the screen is ${display.orientation} — `
       + 'it will be letterboxed.';
}
```

## Media library list

Columns: thumbnail · name · type · dimensions · size · duration · status · actions.

- **Thumbnail inline** — `<img>` for images, `<video muted preload="metadata">`
  for video. Recognising content by sight is most of what this screen is for.
- **Duration column** shows `Auto (0:42)` for video, `10s` otherwise.
- **Status pill** distinguishes three states, not two: Active · Paused ·
  Deleted. Collapsing "paused" and "deleted" into one boolean is what lets a
  deleted ad get silently resurrected — see [data-model.md](data-model.md).
- **Search and type filter.** A library grows past what a flat list serves.
- **Paginate**, or the list query's cap silently truncates the library.

### Confirm destructive actions with their consequences

```tsx
async function confirmDelete(ad: Ad) {
  // Reverse lookup: never let someone break four screens without being told.
  const { displays } = await apiFetch(`/api/admin/ads/${ad.id}/usage`);
  setConfirm({
    title: `Delete "${ad.name}"?`,
    body: displays.length
      ? `In use on ${displays.length} screen(s): ${displays.map((d) => d.name).join(', ')}. `
        + 'It will be removed from those playlists.'
      : 'Not used on any screen.',
    confirmLabel: 'Delete permanently',
    onConfirm: () => apiFetch(`/api/admin/ads/${ad.id}?permanent=true`, { method: 'DELETE' }),
  });
}
```

Backed by `/api/admin/ads/[id]/usage`, which is one indexed query
(`displays where adIds array-contains :id`, or `display_ads where ad_id = :id`).

## One shared playlist editor

If the playlist can be edited from both the screen form and a dedicated
playlists page, it is **one component used twice**. Two copies drift, and the
divergence shows up as two screens behaving differently for no visible reason.

```tsx
// components/admin/PlaylistEditor.tsx
'use client';
import { useMemo } from 'react';
import type { Ad } from '@/types/signage';

interface Props {
  ads: Ad[];                                  // candidates, active only
  value: string[];                            // ordered ad ids
  onChange: (adIds: string[]) => void;
  orientation?: DisplayOrientation;           // enables the fit warning
}

export default function PlaylistEditor({ ads, value, onChange, orientation }: Props) {
  const byId = useMemo(() => new Map(ads.map((a) => [a.id, a])), [ads]);

  const add = (id: string) => onChange([...value, id]);
  const remove = (index: number) => onChange(value.filter((_, i) => i !== index));
  const move = (index: number, dir: -1 | 1) => {
    const target = index + dir;
    if (target < 0 || target >= value.length) return;
    const next = [...value];
    const from = next[index];
    const to = next[target];
    // Explicit reads rather than a destructured swap: the tuple form does not
    // type-check under `noUncheckedIndexedAccess`.
    if (from === undefined || to === undefined) return;
    next[index] = to;
    next[target] = from;
    onChange(next);
  };

  // Total loop time — the number an operator actually needs and never has.
  const loopSeconds = value.reduce((sum, id) => {
    const ad = byId.get(id);
    if (!ad) return sum;
    return sum + (ad.type === 'video' ? (ad.videoDuration ?? ad.duration) : ad.duration);
  }, 0);

  const available = ads.filter((a) => !value.includes(a.id));

  return (
    <div className="grid grid-cols-2 gap-4">
      <section>
        <h3 className="mb-2 text-sm font-medium">Available</h3>
        {available.length === 0
          ? <p className="text-sm opacity-60">All ads are in the playlist.</p>
          : available.map((ad) => (
              <button key={ad.id} type="button" onClick={() => add(ad.id)}
                className="flex w-full items-center gap-2 border-b px-2 py-1.5 text-left text-sm">
                <span className="flex-1">{ad.name}</span>
                <span className="opacity-60">{ad.type}</span>
              </button>
            ))}
      </section>

      <section>
        <h3 className="mb-2 flex justify-between text-sm font-medium">
          <span>Playlist ({value.length})</span>
          <span className="opacity-60">Loop {formatDuration(loopSeconds)}</span>
        </h3>
        {value.map((id, i) => {
          const ad = byId.get(id);
          const warning = ad && orientation ? fitWarning(ad, { orientation }) : null;
          return (
            // Index in the key: the same ad may legitimately appear twice.
            <div key={`${id}-${i}`} className="flex items-center gap-1 border-b px-2 py-1.5 text-sm">
              <span className="w-6 opacity-50">{i + 1}</span>
              <span className="flex-1">{ad?.name ?? <em className="opacity-50">missing</em>}</span>
              {warning && <span title={warning}>⚠️</span>}
              <button type="button" onClick={() => move(i, -1)} disabled={i === 0}>↑</button>
              <button type="button" onClick={() => move(i, 1)}
                disabled={i === value.length - 1}>↓</button>
              <button type="button" onClick={() => remove(i)}>×</button>
            </div>
          );
        })}
      </section>
    </div>
  );
}
```

Notes worth keeping:

- **Filtering `available` by membership prevents duplicates.** That is a product
  decision, not a technical one — the data model allows the same ad twice, which
  is a reasonable way to weight a short spot in a long loop. If you want that,
  drop the filter; the rest already handles it.
- **Loop duration and per-item share** are the two numbers an operator needs to
  reason about a screen, especially if slots are sold.
- **Missing ids render as `missing`** rather than vanishing, so a stale
  reference is visible instead of silently shortening the loop.

## Screen list and provisioning

Columns: name · resolution · orientation · items · **health** · status · actions.
The health column is what turns this from a config table into an operations
dashboard — see [operations.md](operations.md).

Resolution presets save typing and produce consistent values:

```ts
const RESOLUTION_PRESETS = [
  { label: 'Full HD',     width: 1920, height: 1080, orientation: 'landscape' },
  { label: '4K UHD',      width: 3840, height: 2160, orientation: 'landscape' },
  { label: 'Portrait HD', width: 1080, height: 1920, orientation: 'portrait'  },
] as const;
```

Provisioning URL — the fast path for setting up a screen:

```tsx
function ProvisioningUrl({ token, locale }: { token: string; locale: string }) {
  const [copied, setCopied] = useState(false);
  const url = `${window.location.origin}/${locale}/display?token=${token}`;

  return (
    <button onClick={async () => {
      await navigator.clipboard.writeText(url);
      setCopied(true);
      setTimeout(() => setCopied(false), 2000);
    }}>
      {copied ? 'Copied' : 'Copy setup URL'}
    </button>
  );
}
```

**This URL contains the screen's credential.** Show it once, immediately after
pairing or re-pairing; never render it in a list where it sits on an admin's
screen indefinitely. Pair the button with "Unpair" so a lost device is one click
from revoked.

## Transport hook

```ts
// hooks/useAdminApi.ts
export function useAdminApi() {
  const { user } = useAuth();

  const apiFetch = useCallback(async <T,>(url: string, options?: RequestInit): Promise<T> => {
    if (!user) throw new Error('Not authenticated');
    const res = await fetch(url, {
      ...options,
      credentials: 'include',      // the active-location cookie rides on this
      headers: {
        'Content-Type': 'application/json',
        Authorization: `Bearer ${await user.getIdToken()}`,
        ...options?.headers,
      },
    });
    if (!res.ok) {
      const body = await res.json().catch(() => ({}));
      throw new Error(body.error || `HTTP ${res.status}`);
    }
    return res.json() as Promise<T>;
  }, [user]);

  return { apiFetch };
}
```

Surfacing `body.error` is what lets validation messages from the route reach the
toast instead of a bare status code.

## Localisation

Route every string through the app's dictionary from the start. Admin UIs get
written in the team's language "for now" and that decision survives for years —
by the time a second market appears, the strings are spread across a dozen
components. The device UI has only a handful of strings (PIN prompt, errors) and
should be localised too.
