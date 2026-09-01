# Pairing and bootstrap

How an unattended screen acquires a credential and keeps it. The player itself
is in [player-runtime.md](player-runtime.md); the endpoint it talks to is in
[api-routes.md](api-routes.md).

**Inline styles only** on every device-side component — see
[player-runtime.md](player-runtime.md) for why.

## Bootstrap and pairing

Two ways in, both ending at the same stored credential:

1. **Provisioning URL** — an admin opens `…/display?token=…` on the screen once.
2. **PIN pairing** — open `…/display` bare, type the venue PIN, pick the screen.

```tsx
// app/[locale]/(display)/display/page.tsx
'use client';
import { Suspense, useCallback, useEffect, useState } from 'react';
import { useSearchParams } from 'next/navigation';
import DisplayHub from '@/components/signage/DisplayHub';
import DisplayPlayer from '@/components/signage/DisplayPlayer';

const STORAGE_KEY = 'signage_credential';
const BLACK = <div style={{ width: '100vw', height: '100vh', background: '#000' }} />;

function DisplayContent() {
  const searchParams = useSearchParams();
  const [token, setToken] = useState<string | null>(null);
  const [ready, setReady] = useState(false);

  useEffect(() => {
    const fromUrl = searchParams.get('token');
    if (fromUrl) {
      // Private mode: setItem throws; the token then lives for this session only.
      try { localStorage.setItem(STORAGE_KEY, fromUrl); } catch { /* ignore */ }
      setToken(fromUrl);
      // Get the credential out of the address bar so it is not left on screen,
      // in history, or in a screenshot of the venue's TV.
      window.history.replaceState({}, '', window.location.pathname);
    } else {
      try {
        setToken(localStorage.getItem(STORAGE_KEY));
      } catch { /* private mode: fall through to pairing */ }
    }
    setReady(true);
  }, [searchParams]);

  // A revoked or deleted screen lands back on the pairing hub by itself.
  const handleUnauthorized = useCallback(() => {
    try { localStorage.removeItem(STORAGE_KEY); } catch { /* ignore */ }
    setToken(null);
  }, []);

  if (!ready) return BLACK;
  if (!token) {
    return <DisplayHub onPaired={(t) => {
      try { localStorage.setItem(STORAGE_KEY, t); } catch { /* ignore */ }
      setToken(t);
    }} />;
  }
  return <DisplayPlayer token={token} onUnauthorized={handleUnauthorized} />;
}

export default function DisplayPage() {
  return <Suspense fallback={BLACK}>{<DisplayContent />}</Suspense>;
}
```

`useSearchParams` requires the `Suspense` boundary.

## The hub

Two screens driven by one piece of state. Large hit targets, `type="tel"` for the
on-screen numeric keypad, and `Enter` handlers throughout — many screens are
driven by an IR remote, not a mouse.

```tsx
// components/signage/DisplayHub.tsx
'use client';
import { useState } from 'react';

interface DisplayOption { id: string; name: string; adCount: number; paired: boolean }
interface PinResult { locationId: string; locationName: string; displays: DisplayOption[] }

export default function DisplayHub({ onPaired }: { onPaired: (token: string) => void }) {
  const [pin, setPin] = useState('');
  const [result, setResult] = useState<PinResult | null>(null);
  const [error, setError] = useState<string | null>(null);
  const [busy, setBusy] = useState(false);

  async function submitPin() {
    setBusy(true); setError(null);
    try {
      const res = await fetch('/api/display/pair', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ pin }),
      });
      if (!res.ok) { setError(res.status === 429 ? 'Too many attempts' : 'Invalid PIN'); return; }
      setResult(await res.json());
    } catch {
      setError('Connection problem');
    } finally { setBusy(false); }
  }

  async function claim(displayId: string) {
    setBusy(true);
    try {
      const res = await fetch('/api/display/pair', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ pin, displayId }),
      });
      if (!res.ok) { setError('Pairing failed'); return; }
      const { token } = (await res.json()) as { token: string };
      onPaired(token);
    } catch {
      setError('Connection problem');
    } finally { setBusy(false); }
  }

  const shell: React.CSSProperties = {
    width: '100vw', height: '100vh', background: '#000', color: '#fff',
    display: 'flex', flexDirection: 'column', alignItems: 'center',
    justifyContent: 'center', gap: 24, fontFamily: 'system-ui, sans-serif',
  };

  if (!result) {
    return (
      <div style={shell}>
        <h1 style={{ fontSize: 40 }}>Enter venue PIN</h1>
        <input
          type="tel" inputMode="numeric" autoFocus value={pin} maxLength={8}
          onChange={(e) => setPin(e.target.value.replace(/\D/g, '').slice(0, 8))}
          onKeyDown={(e) => { if (e.key === 'Enter' && !busy) void submitPin(); }}
          style={{
            fontSize: 56, letterSpacing: 16, textAlign: 'center', width: 360,
            padding: 16, background: '#111', color: '#fff',
            border: '2px solid #333', borderRadius: 12,
          }}
        />
        {error && <p style={{ color: '#f87171', fontSize: 22 }}>{error}</p>}
        <button onClick={() => void submitPin()} disabled={busy || pin.length < 4}
          style={{ fontSize: 26, padding: '14px 42px', borderRadius: 10, border: 0 }}>
          Continue
        </button>
      </div>
    );
  }

  return (
    <div style={shell}>
      <h1 style={{ fontSize: 36 }}>{result.locationName}</h1>
      {result.displays.length === 0 ? (
        <p style={{ fontSize: 24, opacity: 0.7 }}>No screens configured</p>
      ) : (
        <div style={{
          display: 'grid', gap: 20, width: '80vw',
          gridTemplateColumns: 'repeat(auto-fill, minmax(280px, 1fr))',
        }}>
          {result.displays.map((d) => (
            <div key={d.id} role="button" tabIndex={0}
              onClick={() => void claim(d.id)}
              onKeyDown={(e) => { if (e.key === 'Enter') void claim(d.id); }}
              style={{
                padding: 24, borderRadius: 14, cursor: 'pointer',
                border: '2px solid #333', background: '#111',
              }}>
              <div style={{ fontSize: 26 }}>{d.name}</div>
              <div style={{ fontSize: 17, opacity: 0.6, marginTop: 8 }}>
                {d.adCount} item{d.adCount === 1 ? '' : 's'}
                {/* Warn before stealing a screen from another device. */}
                {d.paired && ' · already paired'}
              </div>
            </div>
          ))}
        </div>
      )}
      <button onClick={() => { setResult(null); setPin(''); }}
        style={{ fontSize: 20, padding: '10px 28px', borderRadius: 10, border: 0 }}>
        Change venue
      </button>
    </div>
  );
}
```

## Provisioning URL vs PIN

| | Provisioning URL | PIN pairing |
|---|---|---|
| Setup effort | Paste one URL, done | Type a PIN, pick a screen |
| Needs a keyboard on the TV | No | Yes, or a remote |
| Credential exposure | Sits in a URL an admin may keep | Never leaves the device |
| Best for | Bulk provisioning by a technician | Venue staff replacing a stick |

Support both. The URL path is faster when someone is imaging ten devices; the
PIN path is what venue staff can actually do unaided at 8 a.m. when a screen
has been swapped.

After either path the device holds the same token in `localStorage`, and a 401
on any poll clears it and returns the screen to the hub.
