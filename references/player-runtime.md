# Player runtime

What runs on the TV once it has a credential. Route group
`app/[locale]/(display)/display/`. Acquiring that credential — the bootstrap
page and the pairing hub — is in [pairing.md](pairing.md).

**Style rule: inline styles only, no Tailwind, no CSS modules, on every
device-side component.** Signage runs on Tizen, webOS, Android TV and £30 sticks
with browsers years behind. Inline styles are the one thing that always works,
and it removes any chance a global stylesheet or purge step breaks a screen
nobody is looking at. The admin UI has no such constraint.

## Layout — kiosk chrome

```tsx
// app/[locale]/(display)/display/layout.tsx
export default function DisplayLayout({ children }: { children: React.ReactNode }) {
  return (
    <div
      style={{ width: '100vw', height: '100vh', background: '#000', overflow: 'hidden' }}
      onContextMenu={(e) => e.preventDefault()}
    >
      {children}
    </div>
  );
}
```

Hide the cursor globally for this route group (`* { cursor: none }`). The
browser is expected to be launched in kiosk mode by the device — the page cannot
put itself fullscreen without a user gesture.

## The state machine

Extracted, pure, and therefore testable without a DOM. This is the file to get
right — the two most damaging bugs in a signage player both live here.

```ts
// components/signage/player-machine.ts
import type { AdItem } from '@/types/signage';

export interface PlayerState {
  ads: AdItem[];
  index: number;
  /** Current media has decoded / started. Gates the duration timer. */
  ready: boolean;
  /** Bumped on every slide change; used as a React key to force a remount. */
  epoch: number;
}

export type PlayerEvent =
  | { type: 'playlist'; ads: AdItem[] }
  | { type: 'ready' }
  | { type: 'advance' };

export const initialPlayerState: PlayerState = {
  ads: [], index: 0, ready: false, epoch: 0,
};

export function playerReducer(state: PlayerState, event: PlayerEvent): PlayerState {
  switch (event.type) {
    case 'playlist': {
      const { ads } = event;
      if (ads.length === 0) return { ...state, ads, index: 0, ready: false };

      // Keep the current slide on screen if it survived the update — a refresh
      // must never restart the loop or cut a slide short.
      const currentId = state.ads[state.index]?.id;
      const carriedOver = currentId ? ads.findIndex((a) => a.id === currentId) : -1;
      if (carriedOver >= 0) return { ...state, ads, index: carriedOver };

      // Current slide is gone, or this is the first load: start from the top.
      return { ...state, ads, index: 0, ready: false, epoch: state.epoch + 1 };
    }

    case 'ready':
      return state.ready ? state : { ...state, ready: true };

    case 'advance': {
      if (state.ads.length === 0) return { ...state, index: 0, ready: false };
      return {
        ...state,
        index: (state.index + 1) % state.ads.length,
        ready: false,
        epoch: state.epoch + 1,
      };
    }

    default:
      return state;
  }
}
```

**Why a reducer and not `setState`.** The tempting shortcut is to reconcile the
playlist inside a `setAds` updater and call `setIndex` from within it. React may
invoke an updater more than once for the same base state; a `setState` call
inside one is a side effect that then fires twice. A reducer is a pure function
of `(state, event)`, so re-invocation is free. The reconciliation logic in
`case 'playlist'` is exactly the code that must not be a side effect.

Test it directly:

```ts
it('keeps the current slide when the playlist changes around it', () => {
  const s0 = playerReducer(initialPlayerState,
    { type: 'playlist', ads: [ad('a'), ad('b'), ad('c')] });
  const s1 = playerReducer(s0, { type: 'advance' });        // showing 'b'
  const s2 = playerReducer(s1, { type: 'playlist', ads: [ad('x'), ad('b')] });
  expect(s2.index).toBe(1);
  expect(s2.ads[s2.index].id).toBe('b');
  expect(s2.epoch).toBe(s1.epoch);                          // no remount
});

it('clamps when the playlist shrinks under the current index', () => {
  const stale: PlayerState = { ads: [ad('a'), ad('b'), ad('c')], index: 2, ready: true, epoch: 5 };
  const next = playerReducer(stale, { type: 'playlist', ads: [ad('a')] });
  expect(next.index).toBeLessThan(next.ads.length);
});

it('is pure under double invocation', () => {
  const base: PlayerState = { ads: [ad('a'), ad('b')], index: 0, ready: true, epoch: 3 };
  expect(playerReducer(base, { type: 'advance' }))
    .toEqual(playerReducer(base, { type: 'advance' }));
  expect(base.index).toBe(0);                               // input never mutated
});
```

## The player

```tsx
// components/signage/DisplayPlayer.tsx
'use client';
import { useEffect, useReducer, useRef, useState } from 'react';
import type {
  AdItem, DisplayCommand, DisplayInfo, DisplayMode, PlaylistResponse,
} from '@/types/signage';
import { initialPlayerState, playerReducer } from './player-machine';

const POLL_INTERVAL_MS = 60_000;
/** A hung fetch on a flaky venue network must not outlive the poll interval. */
const FETCH_TIMEOUT_MS = 30_000;
const CROSSFADE_MS = 600;
/** No slide may ever exceed this, whatever the media does. */
const MAX_VIDEO_MS = 5 * 60_000;
/** A video whose playback clock stops advancing for this long is skipped. */
const VIDEO_STALL_MS = 15_000;
/** Full reload once a day — reclaims leaked memory on long-lived TV browsers. */
const RELOAD_AFTER_MS = 24 * 60 * 60_000;
/** Spread the daily reloads so a venue's screens don't all blink at once. */
const RELOAD_JITTER_MS = 60 * 60_000;
/** Set at build time so telemetry can spot a stale device. */
const AGENT_VERSION = process.env.NEXT_PUBLIC_SIGNAGE_AGENT_VERSION ?? 'dev';
/** Persisted command ack — must survive the reload the command causes. */
const ACK_KEY = 'signage_last_command_ack';

interface Props { token: string; onUnauthorized: () => void }
/** A crossfade slot remembers the epoch it was filled with; see the render. */
type Slot = { ad: AdItem; epoch: number } | null;

export default function DisplayPlayer({ token, onUnauthorized }: Props) {
  const [state, dispatch] = useReducer(playerReducer, initialPlayerState);
  const [display, setDisplay] = useState<DisplayInfo | null>(null);
  const [logoUrl, setLogoUrl] = useState<string | null>(null);
  const [mode, setMode] = useState<DisplayMode>({ kind: 'play' });

  // Refs keep the poll callback stable: it must NOT depend on slide state.
  const stateRef = useRef(state);
  stateRef.current = state;
  const etagRef = useRef<string | null>(null);
  const ackRef = useRef<string | null>(null);
  const unauthorizedRef = useRef(onUnauthorized);
  unauthorizedRef.current = onUnauthorized;
  /** The two crossfade slots; see the render section. */
  const slotsRef = useRef<[Slot, Slot]>([null, null]);
  /** Last time the current video reported playback progress. */
  const videoProgressAtRef = useRef(0);

  // ---- Poll: telemetry up, content and commands down -----------------------
  // Dependencies are [token] only. Deriving this effect from `state` would tear
  // down and recreate the interval on every slide change, so a 60s poll behind
  // 10s slides would never fire and the screen would never see new content.
  useEffect(() => {
    let cancelled = false;
    let inFlight = false;      // a hung fetch must not stack overlapping polls

    async function poll() {
      if (inFlight) return;
      inFlight = true;
      const controller = new AbortController();
      const timeoutId = setTimeout(() => controller.abort(), FETCH_TIMEOUT_MS);
      try {
        const headers: Record<string, string> = { 'Content-Type': 'application/json' };
        if (etagRef.current) headers['If-None-Match'] = etagRef.current;

        const snapshot = stateRef.current;
        const res = await fetch('/api/display/playlist', {
          method: 'POST',
          headers,
          signal: controller.signal,
          body: JSON.stringify({
            token,
            currentAdId: snapshot.ads[snapshot.index]?.id ?? null,
            agentVersion: AGENT_VERSION,
            ...(ackRef.current ? { ackCommandAt: ackRef.current } : {}),
          }),
        });

        if (cancelled) return;
        if (res.status === 304) { ackRef.current = null; return; }
        if (res.status === 401) { unauthorizedRef.current(); return; }
        if (!res.ok) return;                    // keep playing what we have

        ackRef.current = null;
        etagRef.current = res.headers.get('ETag');
        const data = (await res.json()) as PlaylistResponse;
        if (cancelled) return;

        setDisplay(data.display);
        setLogoUrl(data.fallbackLogoUrl);
        setMode(data.mode);                     // durable state, no ack needed
        dispatch({ type: 'playlist', ads: data.ads });
        if (data.command) applyCommand(data.command);
      } catch {
        // Offline or timed out. Keep looping the playlist already in memory —
        // media is served from the HTTP cache. Never blank on a failed poll.
      } finally {
        clearTimeout(timeoutId);
        inFlight = false;
      }
    }

    function applyCommand(command: DisplayCommand) {
      // `reload` destroys the in-memory ack before the next poll can deliver
      // it, so the server would re-send the command forever — an infinite
      // reload loop. Persist the ack locally and skip anything already done.
      let lastAck: string | null = null;
      try { lastAck = localStorage.getItem(ACK_KEY); } catch { /* ignore */ }
      ackRef.current = command.issuedAt;        // acked on the next poll
      if (lastAck && command.issuedAt <= lastAck) return;   // ran pre-reload
      try { localStorage.setItem(ACK_KEY, command.issuedAt); } catch { /* ignore */ }
      if (command.kind === 'reload') window.location.reload();
    }

    void poll();
    const id = setInterval(() => void poll(), POLL_INTERVAL_MS);
    return () => { cancelled = true; clearInterval(id); };
  }, [token]);

  // ---- Duration timer: images and GIFs only, and only once decoded ---------
  useEffect(() => {
    const ad = state.ads[state.index];
    if (!ad || ad.type === 'video' || !state.ready) return;
    const id = setTimeout(() => dispatch({ type: 'advance' }), ad.duration * 1000);
    return () => clearTimeout(id);
  }, [state.ads, state.index, state.ready]);

  // ---- Watchdog: the guarantee that the loop can never stop ----------------
  // Covers a video that never fires `ended`, an image whose `load` never
  // arrives, and any stall in between. One timer, every failure mode.
  useEffect(() => {
    const ad = state.ads[state.index];
    if (!ad) return;
    const cap = ad.type === 'video'
      ? MAX_VIDEO_MS
      : Math.max(ad.duration * 1000 * 3, 30_000);
    const id = setTimeout(() => dispatch({ type: 'advance' }), cap);
    return () => clearTimeout(id);
  }, [state.ads, state.index, state.epoch]);

  // ---- Video stall watchdog -------------------------------------------------
  // The 5-minute cap above is the last resort; this is the fast path. A video
  // whose playback clock stops advancing (buffer starvation, decoder wedge) is
  // skipped in seconds instead of freezing on one frame for minutes. Fed by
  // `onTimeUpdate` on the <video> element.
  useEffect(() => {
    const ad = state.ads[state.index];
    if (!ad || ad.type !== 'video' || !state.ready) return;
    videoProgressAtRef.current = Date.now();
    const id = setInterval(() => {
      if (Date.now() - videoProgressAtRef.current > VIDEO_STALL_MS) {
        dispatch({ type: 'advance' });
      }
    }, 5_000);
    return () => clearInterval(id);
  }, [state.ads, state.index, state.ready, state.epoch]);

  // ---- Long-run hygiene ----------------------------------------------------
  useEffect(() => {
    // Jittered so a venue's screens, powered on together, do not all reload
    // at the same moment.
    const id = setTimeout(() => window.location.reload(),
      RELOAD_AFTER_MS + Math.random() * RELOAD_JITTER_MS);
    return () => clearTimeout(id);
  }, []);

  useEffect(() => {
    // Best-effort: unsupported on most TV browsers, harmless where it is.
    let lock: WakeLockSentinel | null = null;
    const acquire = async () => {
      try { lock = await navigator.wakeLock?.request('screen'); } catch { /* ignore */ }
    };
    void acquire();
    const onVisible = () => { if (document.visibilityState === 'visible') void acquire(); };
    document.addEventListener('visibilitychange', onVisible);
    return () => {
      document.removeEventListener('visibilitychange', onVisible);
      void lock?.release();
    };
  }, []);

  // ---- Rotation ------------------------------------------------------------
  const [viewportPortrait, setViewportPortrait] = useState(false);
  useEffect(() => {
    const update = () => setViewportPortrait(window.innerHeight > window.innerWidth);
    update();
    window.addEventListener('resize', update);
    window.addEventListener('orientationchange', update);
    return () => {
      window.removeEventListener('resize', update);
      window.removeEventListener('orientationchange', update);
    };
  }, []);

  // A portrait-mounted panel that reports a landscape viewport is physically
  // rotated; rotate the content back so it renders upright.
  const rotate = display?.orientation === 'portrait' && !viewportPortrait;
  const container: React.CSSProperties = rotate
    ? {
        position: 'absolute', top: '50%', left: '50%',
        width: '100vh', height: '100vw',
        transform: 'translate(-50%, -50%) rotate(-90deg)',
        transformOrigin: 'center center',
        background: '#000', overflow: 'hidden',
      }
    : { position: 'relative', width: '100%', height: '100%',
        background: '#000', overflow: 'hidden' };

  // ---- Render --------------------------------------------------------------
  // `mode` is durable server state: it survives reloads and power cycles, and
  // an expired takeover is resolved back to `play` server-side.
  if (mode.kind === 'blank') return <div style={container} />;

  if (mode.kind === 'takeover') {
    return (
      <div style={{ ...container, display: 'flex', alignItems: 'center',
        justifyContent: 'center', padding: '6vw' }}>
        <p style={{ color: '#fff', fontSize: '6vmin', textAlign: 'center',
          fontFamily: 'system-ui, sans-serif', lineHeight: 1.3 }}>
          {mode.message}
        </p>
      </div>
    );
  }

  if (state.ads.length === 0) {
    return (
      <div style={container}>
        {logoUrl && (
          // eslint-disable-next-line @next/next/no-img-element
          <img src={logoUrl} alt="" style={{
            position: 'absolute', top: '50%', left: '50%',
            transform: 'translate(-50%, -50%)',
            maxWidth: '60%', maxHeight: '60%', objectFit: 'contain',
          }} />
        )}
      </div>
    );
  }

  const current = state.ads[state.index];
  const next = state.ads.length > 1
    ? state.ads[(state.index + 1) % state.ads.length]
    : null;

  // Two slots alternating by parity: the outgoing slide stays mounted in the
  // other slot and fades out, giving a real crossfade instead of a fade up
  // from black. Each slot keeps the epoch it was filled with — keying both
  // slots on the *current* epoch would remount the outgoing slide at opacity
  // 0 and cut the crossfade to a fade-from-black. Writing to the ref during
  // render is intentional and safe — it is derived state, read in the same
  // render.
  const active = state.index % 2;
  slotsRef.current[active] = current ? { ad: current, epoch: state.epoch } : null;
  const slots = slotsRef.current;

  return (
    <div style={container}>
      {[0, 1].map((slot) => {
        const entry = slots[slot];
        if (!entry) return null;
        const { ad } = entry;
        const isActive = slot === active;
        // Never keep an offscreen <video> mounted: it goes on decoding. The
        // cost is that transitions out of video hard-cut rather than fade —
        // deliberate.
        if (!isActive && ad.type === 'video') return null;

        const style: React.CSSProperties = {
          position: 'absolute', inset: 0, width: '100%', height: '100%',
          objectFit: 'contain', display: 'block', background: '#000',
          opacity: isActive && state.ready ? 1 : 0,
          transition: `opacity ${CROSSFADE_MS}ms ease-in-out`,
        };

        return ad.type === 'video' ? (
          <video
            key={`${slot}-${ad.id}-${entry.epoch}`}
            src={ad.url}
            autoPlay muted playsInline
            // Older iOS/WebKit-derived TV browsers need the legacy attribute.
            webkit-playsinline=""
            onPlaying={() => dispatch({ type: 'ready' })}
            onTimeUpdate={() => { videoProgressAtRef.current = Date.now(); }}
            onEnded={() => dispatch({ type: 'advance' })}
            onError={() => dispatch({ type: 'advance' })}
            style={style}
          />
        ) : (
          // eslint-disable-next-line @next/next/no-img-element
          <img
            key={`${slot}-${ad.id}-${entry.epoch}`}
            src={ad.url}
            alt=""
            onLoad={() => dispatch({ type: 'ready' })}
            onError={() => dispatch({ type: 'advance' })}
            style={style}
          />
        );
      })}

      {/* Preload the next image as a real DOM node — more reliable on TV
          browsers than `new Image()`, which some of them never schedule. */}
      {next && next.type !== 'video' && (
        // eslint-disable-next-line @next/next/no-img-element
        <img key={`preload-${next.id}`} src={next.url} alt="" aria-hidden="true"
          style={{ position: 'absolute', width: 1, height: 1, opacity: 0,
            pointerEvents: 'none' }} />
      )}
    </div>
  );
}
```

## Error boundary

A render crash on an unattended screen is a frozen frame forever. Catch it and
reload.

```tsx
// components/signage/PlayerBoundary.tsx
'use client';
import { Component, type ReactNode } from 'react';

export default class PlayerBoundary extends Component<{ children: ReactNode }> {
  state = { crashed: false };
  static getDerivedStateFromError() { return { crashed: true }; }

  componentDidCatch(error: Error) {
    console.error('[signage] player crashed', error);
    setTimeout(() => window.location.reload(), 10_000);
  }

  render() {
    if (this.state.crashed) {
      return <div style={{ width: '100vw', height: '100vh', background: '#000' }} />;
    }
    return this.props.children;
  }
}
```

Wrap `<DisplayPlayer>` in it. Black for ten seconds then a reload beats a stack
trace on a wall in a customer-facing space.

## Behaviour contract

| Situation | Behaviour |
|---|---|
| Poll fails (offline, 5xx, timeout) | Keep looping the playlist in memory; media comes from HTTP cache |
| Poll hangs | Aborted at 30 s; overlapping polls never stack |
| Poll returns 304 | Nothing changes, no re-render, heartbeat still recorded server-side |
| Poll returns 401 | Credential cleared, screen returns to the pairing hub |
| Playlist updated, current ad survives | Slide keeps playing, no restart |
| Playlist updated, current ad removed | Restart from the top |
| Playlist becomes empty | Venue logo, or black |
| Image fails to load | Skip immediately |
| Video errors | Skip immediately |
| Video stalls (no progress for 15 s) | Skip |
| Video never ends | Skip at the 5-minute cap |
| Slide never becomes ready | Watchdog advances at 3× duration, min 30 s |
| `blank` / `takeover` mode set | Applied on every poll; survives reload and power cycle; ends on `resume` |
| `takeover` past its `until` | Server resolves it to `play`; screen reverts within one poll |
| `reload` command | Runs once — the ack is persisted in `localStorage` first, so it cannot loop |
| Screen deleted or unpaired in admin | Next poll 401s → back to pairing |

## Offline

Deliberately minimal: the playlist stays in memory and media is served from the
browser HTTP cache (the `immutable`, one-year `Cache-Control` set at upload is
what makes this work). No IndexedDB, no service worker, no retry backoff.

That is sufficient for a screen with occasional connectivity. For screens that
are offline for hours at a time, add a service worker that precaches the
playlist's media — see [extensions.md](extensions.md).
