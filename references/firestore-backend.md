# Firestore backend

Collections `ads` and `displays`, per [data-model.md](data-model.md). Everything
here is server-side except the upload helper, which is deliberately client-side.

## Security posture

**Neither collection gets a rules block.** Default deny for all client SDK
access; every read and write goes through an API route using the Admin SDK,
which bypasses rules. The rules file is for collections the browser touches
directly — signage is not one of them.

The one exception is media: TVs load ad files straight from Storage, so the
`ads/` prefix is public-read.

## Lazy admin client

Never export an initialized instance — that runs credential parsing at import
time and breaks builds where the env is absent.

```ts
// lib/firebase-admin.ts
import 'server-only';
import { cert, getApps, initializeApp, type App } from 'firebase-admin/app';
import { getFirestore, type Firestore } from 'firebase-admin/firestore';

let _app: App | undefined;
let _db: Firestore | undefined;

function getAdminApp(): App {
  if (_app) return _app;
  const existing = getApps()[0];
  if (existing) return (_app = existing);

  const projectId = process.env.NEXT_PUBLIC_FIREBASE_PROJECT_ID;
  const clientEmail = process.env.FIREBASE_CLIENT_EMAIL;
  const privateKey = process.env.FIREBASE_PRIVATE_KEY?.replace(/\\n/g, '\n');
  const missing = [
    !projectId && 'NEXT_PUBLIC_FIREBASE_PROJECT_ID',
    !clientEmail && 'FIREBASE_CLIENT_EMAIL',
    !privateKey && 'FIREBASE_PRIVATE_KEY',
  ].filter(Boolean);
  if (missing.length) throw new Error(`Firebase admin env missing: ${missing.join(', ')}`);

  return (_app = initializeApp({
    credential: cert({ projectId, clientEmail, privateKey } as never),
  }));
}

export function getAdminDb(): Firestore {
  return (_db ??= getFirestore(getAdminApp()));
}
```

## Service module

`modules/signage/ads.server.ts`. Take `db` as a parameter on anything with real
logic — that is what makes it testable without an emulator.

```ts
import 'server-only';
import { FieldPath, FieldValue, type Firestore, Timestamp } from 'firebase-admin/firestore';
import { getAdminDb } from '@/lib/firebase-admin';
import type { Ad, AdInput } from '@/types/signage';

const ADS = 'ads';

/** Firestore Timestamps -> ISO strings, so the app never leaks SDK types. */
function toAd(id: string, data: FirebaseFirestore.DocumentData): Ad {
  const iso = (v: unknown) => (v instanceof Timestamp ? v.toDate().toISOString() : null);
  return {
    ...(data as Omit<Ad, 'id' | 'createdAt' | 'updatedAt' | 'deletedAt'>),
    id,
    deletedAt: iso(data.deletedAt),
    createdAt: iso(data.createdAt) ?? new Date(0).toISOString(),
    updatedAt: iso(data.updatedAt) ?? iso(data.createdAt) ?? new Date(0).toISOString(),
  };
}

export async function listAds(
  locationId: string,
  opts: { includeInactive?: boolean; limit?: number } = {},
): Promise<Ad[]> {
  let q = getAdminDb()
    .collection(ADS)
    .where('locationIds', 'array-contains', locationId)
    .where('deletedAt', '==', null)          // never surface deleted rows
    .orderBy('createdAt', 'desc');
  if (!opts.includeInactive) q = q.where('active', '==', true);
  const snap = await q.limit(opts.limit ?? 500).get();
  return snap.docs.map((d) => toAd(d.id, d.data()));
}

/**
 * Resolve exactly the ads a playlist references, preserving `adIds` order.
 * Chunked at 30 — the `in` operator's limit. Never "list newest N and filter":
 * that silently drops older ads from playlists once the library outgrows N.
 */
export async function getAdsByIds(
  ids: string[],
  db: Firestore = getAdminDb(),      // injectable, so it is testable without an emulator
): Promise<Ad[]> {
  if (ids.length === 0) return [];
  const unique = [...new Set(ids)];
  const chunks: string[][] = [];
  for (let i = 0; i < unique.length; i += 30) chunks.push(unique.slice(i, i + 30));

  const snaps = await Promise.all(
    chunks.map((chunk) =>
      db.collection(ADS).where(FieldPath.documentId(), 'in', chunk).get(),
    ),
  );
  const byId = new Map<string, Ad>();
  for (const snap of snaps) for (const d of snap.docs) byId.set(d.id, toAd(d.id, d.data()));

  // Order by adIds, keep duplicates, drop inactive/deleted/missing.
  return ids
    .map((id) => byId.get(id))
    .filter((ad): ad is Ad => ad != null && ad.active && ad.deletedAt == null);
}

export async function createAd(data: AdInput): Promise<Ad> {
  const ref = await getAdminDb().collection(ADS).add({
    ...data,
    active: data.active ?? true,
    deletedAt: null,                          // must exist for the list query's equality filter
    createdAt: FieldValue.serverTimestamp(),
    updatedAt: FieldValue.serverTimestamp(),
  });
  const doc = await ref.get();
  return toAd(doc.id, doc.data()!);
}

export async function updateAd(id: string, data: Partial<AdInput>): Promise<void> {
  await getAdminDb().collection(ADS).doc(id).update({
    ...data,
    updatedAt: FieldValue.serverTimestamp(),
  });
}

/** Soft delete — reversible only by an admin, invisible to lists. */
export async function softDeleteAd(id: string): Promise<void> {
  await getAdminDb().collection(ADS).doc(id).update({
    deletedAt: FieldValue.serverTimestamp(),
    updatedAt: FieldValue.serverTimestamp(),
  });
}

/**
 * Hard delete. All three steps or you leak storage and break playlists:
 * remove the doc, delete the file, strip the id from every playlist.
 */
export async function purgeAd(id: string): Promise<void> {
  const db = getAdminDb();
  const doc = await db.collection(ADS).doc(id).get();
  const storagePath = doc.data()?.storagePath as string | undefined;

  const referencing = await db.collection('displays')
    .where('adIds', 'array-contains', id).get();

  const batch = db.batch();
  for (const d of referencing.docs) {
    batch.update(d.ref, {
      adIds: FieldValue.arrayRemove(id),
      updatedAt: FieldValue.serverTimestamp(),
    });
  }
  batch.delete(doc.ref);
  await batch.commit();

  if (storagePath) await deleteStorageObject(storagePath);
}
```

`arrayRemove` strips **all** occurrences, which is correct — a purged ad should
vanish from every position in every loop.

Displays follow the same shape in `modules/signage/displays.server.ts`:
`listDisplays`, `getDisplay`, `createDisplay`, `updateDisplay`, `softDeleteDisplay`,
`purgeDisplay`, plus the pairing and telemetry writers in
[api-routes.md](api-routes.md) and [operations.md](operations.md).
`createDisplay` writes `mode: { kind: 'play' }`, `command: null`, and
`deletedAt: null` explicitly — the equality filter and the player both rely on
the fields existing.

**Always `orderBy` a list query.** Without it, row order is whatever Firestore
happens to return, and it can change between deploys — anything that leans on
it (a UI list, an export, "the first screen") shifts silently.

## Server-side storage deletion

The client SDK cannot be trusted to clean up; do it from the Admin SDK where the
purge happens.

```ts
// modules/signage/storage.server.ts
import 'server-only';
import { getStorage } from 'firebase-admin/storage';

export async function deleteStorageObject(storagePath: string): Promise<void> {
  try {
    await getStorage().bucket().file(storagePath).delete();
  } catch (err) {
    if ((err as { code?: number }).code === 404) return;  // already gone
    throw err;
  }
}
```

Call it on hard delete **and when a file is replaced during an edit** — the old
object is otherwise orphaned and billed forever.

## Client upload

Media goes browser → Storage directly. Use the **resumable** API so the UI can
show progress: a 10 MB video over venue Wi-Fi is not instant.

```ts
// lib/signage-upload.ts  (client)
import { ref, uploadBytesResumable, getDownloadURL } from 'firebase/storage';
import { getFirebaseStorage } from './firebase';

export const MAX_AD_BYTES = 10 * 1024 * 1024;
export const ALLOWED_AD_MIME = /^(image|video)\//;

export interface UploadResult { url: string; storagePath: string }

export async function uploadAdFile(
  file: File,
  onProgress?: (pct: number) => void,
): Promise<UploadResult> {
  // Validate client-side too. Relying only on storage rules turns a 9 MB
  // overage into an opaque "save failed" toast after a long upload.
  if (file.size > MAX_AD_BYTES) {
    throw new Error(`File is ${(file.size / 1e6).toFixed(1)} MB; the limit is 10 MB.`);
  }
  if (!ALLOWED_AD_MIME.test(file.type)) {
    throw new Error(`Unsupported file type: ${file.type || 'unknown'}.`);
  }

  const safeName = file.name.replace(/[^a-zA-Z0-9._-]/g, '_');
  const storagePath = `ads/${Date.now()}_${safeName}`;
  const task = uploadBytesResumable(ref(getFirebaseStorage(), storagePath), file, {
    contentType: file.type,
    cacheControl: 'public, max-age=31536000, immutable',  // path is unique per upload
  });

  await new Promise<void>((resolve, reject) => {
    task.on('state_changed',
      (s) => onProgress?.(Math.round((s.bytesTransferred / s.totalBytes) * 100)),
      reject,
      resolve);
  });

  return { url: await getDownloadURL(task.snapshot.ref), storagePath };
}
```

The immutable `cacheControl` matters: every screen re-requests media on every
loop, and without it you pay egress for the same file forever.

## Rules and indexes

```js
// firestore.rules — no `ads` or `displays` block anywhere in this file.
// Absence is the policy: default deny for clients, Admin SDK for everything.
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /{document=**} { allow read, write: if false; }
  }
}
```

```js
// storage.rules
match /ads/{fileName} {
  allow read: if true;                                  // TVs are unauthenticated
  // `staff` is a custom claim the host sets at sign-in. A bare
  // `request.auth != null` lets any signed-in customer fill a public bucket.
  allow create, update: if request.auth != null
    && request.auth.token.staff == true
    && request.resource.size < 10 * 1024 * 1024
    && request.resource.contentType.matches('image/.*|video/.*');
  allow delete: if false;                               // server-side purge only
}
```

```json
// firestore.indexes.json
{
  "indexes": [
    { "collectionGroup": "ads", "queryScope": "COLLECTION", "fields": [
      { "fieldPath": "locationIds", "arrayConfig": "CONTAINS" },
      { "fieldPath": "deletedAt", "order": "ASCENDING" },
      { "fieldPath": "active", "order": "ASCENDING" },
      { "fieldPath": "createdAt", "order": "DESCENDING" }
    ]},
    { "collectionGroup": "displays", "queryScope": "COLLECTION", "fields": [
      { "fieldPath": "locationId", "order": "ASCENDING" },
      { "fieldPath": "deletedAt", "order": "ASCENDING" },
      { "fieldPath": "name", "order": "ASCENDING" }
    ]},
    { "collectionGroup": "displays", "queryScope": "COLLECTION", "fields": [
      { "fieldPath": "adIds", "arrayConfig": "CONTAINS" },
      { "fieldPath": "name", "order": "ASCENDING" }
    ]}
  ]
}
```

The third index powers both the purge cleanup and the admin's "used by N
displays" reverse lookup.

## Firestore-specific traps

- **`undefined` is rejected on write.** Strip optional keys before saving:
  `Object.fromEntries(Object.entries(obj).filter(([, v]) => v !== undefined))`.
  Better, parse the body with zod and let it drop absent optionals.
- **Equality filters need the field to exist.** `where('deletedAt', '==', null)`
  skips documents with no `deletedAt` field at all, so always write `null`
  explicitly on create.
- **`in` queries cap at 30 ids** — chunk, as `getAdsByIds` does.
- **`array-contains` allows one per query.** You cannot filter `locationIds` and
  `adIds` in the same query; do the second pass in memory.
