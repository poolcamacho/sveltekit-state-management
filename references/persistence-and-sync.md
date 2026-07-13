# Persistence, synchronization, and real-time state

Browser persistence, cross-tab sync, real-time channels, optimistic updates, and
when to add an external cache library. These sit on top of the seven categories:
persistence and real-time are usually a client-state or server-state piece with an
extra transport attached.

## Table of contents
- [Cookies vs local storage vs session storage](#storage-choice)
- [Versioned, SSR-safe persistence](#versioned-persistence)
- [Cross-tab synchronization](#cross-tab)
- [Real-time updates: WebSocket and SSE](#realtime)
- [Optimistic vs server-confirmed state](#optimistic)
- [Cache state and external libraries](#cache-libraries)

## Cookies vs local storage vs session storage {#storage-choice}

| Need | Choose | Why |
| --- | --- | --- |
| Server must read it during SSR (session, locale) | Cookie | Cookies travel to the server on every request |
| Trusted session token | Cookie, `httpOnly` + `secure` + `sameSite` | Not readable by client JS, resists theft and CSRF |
| Client-only preference (theme, layout) | Local storage | Server never needs it; survives reload |
| Per-tab scratch state | Session storage | Cleared when the tab closes; not shared across tabs |
| Large or structured data the server owns | Neither | That is server-loaded data (category 3) |

Key rules: never store a trusted auth decision in local storage (a client value
can be forged; the server must verify). Never put secrets or personal data in a
non-`httpOnly` cookie or in the URL. Keep cookie payloads small (a token or id,
not a profile).

Reading cookies in SvelteKit: use the `cookies` API in server `load`, actions, and
hooks (`cookies.get`, `cookies.set` with options). On the client, prefer letting
the server manage session cookies; read client-only cookies with care.

## Versioned, SSR-safe persistence {#versioned-persistence}

Persisted state outlives your code, so a deploy that changes the shape must not
break returning users. Wrap access in a versioned, browser-guarded helper.

```ts
// src/lib/persist.svelte.ts
import { browser } from '$app/environment';

export function persisted(key, version, initial) {
  let value = $state(initial);

  if (browser) {
    try {
      const raw = localStorage.getItem(key);
      if (raw) {
        const parsed = JSON.parse(raw);
        if (parsed.version === version) value = parsed.data;
        // version mismatch: keep `initial`, or migrate here
      }
    } catch {
      // corrupt value: fall back to initial
    }
  }

  $effect(() => {
    if (browser) {
      localStorage.setItem(key, JSON.stringify({ version, data: value }));
    }
  });

  return {
    get value() { return value; },
    set value(v) { value = v; }
  };
}
```

The `version` tag lets new code detect and discard or migrate an old shape instead
of crashing. The `browser` guard keeps SSR from touching `localStorage`. Reading
inside an effect (or after a guard) avoids a hydration mismatch, because the server
renders `initial` and the browser applies the stored value after mount.

## Cross-tab synchronization {#cross-tab}

Two tabs of the same app do not share `$state`; each has its own JavaScript
context. To keep them in sync (for example, log out everywhere, or reflect a
preference change), use the `storage` event or a `BroadcastChannel`.

```ts
// sync theme across tabs
$effect(() => {
  if (!browser) return;
  const channel = new BroadcastChannel('prefs');
  channel.onmessage = (e) => { theme = e.data.theme; };
  return () => channel.close();
});

function setTheme(next) {
  theme = next;
  new BroadcastChannel('prefs').postMessage({ theme: next });
}
```

The `storage` event fires in other tabs when `localStorage` changes, which pairs
naturally with the versioned persistence above. Use cross-tab sync deliberately;
it is easy to create loops (tab A writes, tab B receives and writes back). Guard by
only broadcasting on genuine user changes, not on every received update.

For "log out everywhere", broadcasting a signal and then reloading or navigating is
usually simpler and safer than trying to mutate shared client state across tabs.

## Real-time updates: WebSocket and SSE {#realtime}

Real-time state is a client-state piece fed by a live transport. The transport
setup and teardown is a side effect, so it belongs in `$effect` (or a store's
start/stop callback), never at module top level (which would open a connection
during SSR).

```svelte
<script>
  import { browser } from '$app/environment';
  let messages = $state([]);

  $effect(() => {
    if (!browser) return;
    const ws = new WebSocket('wss://example.com/feed');
    ws.onmessage = (e) => { messages = [...messages, JSON.parse(e.data)]; };
    return () => ws.close(); // teardown when the component unmounts
  });
</script>
```

Guidance:

- Seed initial data from `load` (server-rendered, SSR-safe), then apply live
  updates on top. Do not rely on the socket for the first paint.
- Choose SSE (`EventSource`) for one-way server-to-client streams (notifications,
  a live counter); it is simpler and reconnects automatically. Choose WebSocket
  for bidirectional needs.
- Always return a teardown function from the effect so connections close on
  unmount. A leaked socket per navigation is a common bug.
- Reconcile live updates with server truth: on reconnect, refetch or
  `invalidate` rather than assuming the stream never dropped a message.
- Note that SvelteKit's default adapters do not host a long-lived WebSocket
  server inside the app; the socket endpoint is typically a separate service or a
  platform feature. Confirm what the target adapter supports.

## Optimistic vs server-confirmed state {#optimistic}

Optimistic updates apply the expected result immediately, before the server
confirms, then reconcile.

Use optimistic updates for high-frequency, low-stakes actions where latency hurts
UX and a rare rollback is acceptable: liking, reordering a list, toggling a flag.
Requirements:

- Keep the previous value so you can roll back on failure.
- Reconcile with the server's response (it may differ from your guess).
- Show a clear error and restored state on failure.

```svelte
<script>
  let liked = $state(false);
  async function toggle() {
    const previous = liked;
    liked = !liked;                       // optimistic
    try {
      const res = await fetch('/api/like', { method: 'POST' });
      if (!res.ok) throw new Error();
    } catch {
      liked = previous;                   // rollback
    }
  }
</script>
```

Prefer server-confirmed state (await the response, then update, or use a form
action and let `load` refresh) for money, permissions, inventory, and anything a
user must trust. Form actions with `use:enhance` give a clean server-confirmed
flow and can be made optimistic selectively.

## Cache state and external libraries {#cache-libraries}

SvelteKit already caches: `load` runs on navigation, dedupes, and re-runs on
`invalidate(dependency)` or `invalidateAll()`. For most apps this is the whole
server-state cache story, and adding a library duplicates it (see the redundant
cache anti-pattern).

Add an external cache library (for example TanStack Query) when you need features
`load` does not provide:

- Background refetching and window-focus or interval revalidation.
- Fine-grained cache keys shared across many components independent of routes.
- Client-side mutation caches with built-in optimistic helpers and retry.
- Infinite or paginated caches with per-page retention.

When you do adopt one, let it own those cases clearly rather than shadowing
`load`. A common clean split: `load` for the initial, SSR-rendered data and
route-driven fetches; the cache library for client-driven, cross-component, or
frequently-revalidated data. Avoid having both fetch and cache the same resource
with different policies. If you cannot name a feature of the library you are using,
you probably do not need it yet.
