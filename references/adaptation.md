# Adaptation contract

Everywhere this module touches its host. Fill in the right-hand column from the
target app **before** generating code, and confirm the rename with the user.

| Seam | This skill ships | Your app supplies |
|---|---|---|
| Domain entities | `Ad`, `Display` | Its own vocabulary, see below |
| Tenant scope | `locationId` / `locationIds[]` | org / site / store / venue id |
| Auth guard | `requireStaffAuth(req, 'admin')` | Clerk, NextAuth, Supabase, custom |
| PIN lookup | `getLocationByPin(pin)` | Location-by-PIN lookup; PIN storage, hashing, rotation |
| Scope check | `requireLocationScope(scope, doc)` | Its equivalent, or copy this one |
| Data access | Firestore and Supabase references | Its ORM or SDK |
| Object storage | `uploadAdFile` / `deleteStorageObject` | Firebase, Supabase, Blob, S3 |
| Transport | `apiFetch` with a bearer token | Its fetch wrapper |
| UI primitives | Admin structure and semantics | Its buttons, dialogs, tables, toasts |
| Styling | Tailwind layout intent | Its design tokens |
| Strings | English literals in the templates | Its i18n system and language |
| Validation | zod schemas | zod / valibot / yup |
| Secret | `SIGNAGE_TOKEN_PEPPER` | Its secret store |

## Renaming the domain

The canonical names are generic on purpose. Common targets:

| Canonical | Retail | Restaurant | Corporate | Healthcare |
|---|---|---|---|---|
| `Ad` | Asset, Creative | Menu item | Slide | Notice |
| `Display` | Screen | Board | Panel | Monitor |
| `Location` | Store | Site | Office | Ward |

Decide once, apply everywhere at once: types, columns, routes, components,
comments, strings. A half-done rename teaches the next reader that both names are
live.

**Do not rename** the platform terms: `orientation`, `mimeType`, `ETag`,
`duration`, `storagePath`. They belong to the web, not the domain.

## What is not negotiable

Seven rules travel with the module or it stops working. They are the hard rules
in `SKILL.md` and the non-negotiables in `README.md`, in the same order. Keep
them, and keep the comments explaining why, or the first reader "cleans them up".

1. **Inline styles on device components.** Signage runs on TV browsers years
   behind current, and inline styles remove any dependency on the host's CSS
   pipeline. Not a style preference; a compatibility requirement.
2. **The poll interval is independent of render state.** The refresh runs on a
   stable timer and reads slide state from a ref, so a refresh longer than a
   slide still fires. See [player-runtime.md](player-runtime.md).
3. **Per-display tokens, stored hashed.** A shared credential on unattended
   hardware cannot be revoked for one screen; a per-display hash makes
   revocation one row update.
4. **Tenant scope is derived server-side and enforced on every `[id]` route,
   answering 404, not 403.** A client-supplied scope is not a scope, and a 403
   tells a prober the id exists. See [api-routes.md](api-routes.md).
5. **`active` is operator intent, `deletedAt` is lifecycle.** One boolean for
   both lets a toggle bring a deleted item back. See [data-model.md](data-model.md).
6. **An image's duration timer starts at `onLoad`.** On a slow TV the slide
   would otherwise expire before it is visible.
7. **Durable screen state is `mode`, delivered on every poll; a reload persists
   its ack before reloading.** Blank and takeover must survive the daily reload
   and a power cycle, and an unpersisted ack is re-delivered forever. See
   [operations.md](operations.md).

Everything else (styling, naming, storage provider, auth, i18n) is yours.

## What the host must already have

- **A tenant/location concept.** If the app is single-tenant, collapse the scope
  fields to a constant rather than deleting them; multi-site arrives later.
- **Staff authentication with roles.** The admin surface assumes an admin role.
- **A per-location pairing PIN** (4 to 8 digits) with an admin surface to set and
  rotate it. `getLocationByPin` is a seam: the module never stores PINs. Store
  them hashed, or at minimum compare in constant time.
- **Object storage** with public-read or signed URLs. Devices are unauthenticated
  to the storage layer.
- **A venue logo or equivalent fallback image**, or accept a black screen when a
  playlist is empty.

## Integration points to wire up

Easy to forget, because nothing fails loudly without them:

- [ ] Register the admin pages in the app's navigation
- [ ] Add the device route to any auth middleware's public allowlist
- [ ] Add `SIGNAGE_TOKEN_PEPPER` to every environment, including preview
- [ ] Set `NEXT_PUBLIC_SIGNAGE_AGENT_VERSION` at build time, or telemetry
      cannot spot a stale device
- [ ] Deploy the indexes (Firestore) or run the migration (Postgres)
- [ ] Apply the storage bucket rules or policies
- [ ] Add signage strings to **every** locale file, not just the default
- [ ] Exclude the device route from analytics: screens generate constant traffic
- [ ] Decide whether the module sits behind a feature flag

## Order of work

Types first, then schema, data access, API routes, the admin UI, and the device
runtime last.

Build the device player last: it is the only part needing a physical screen to
validate, and everything it consumes must exist first. Type-check after each
layer: a rename fixed at step one is a single edit.
