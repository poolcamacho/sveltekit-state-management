# Example: real-time dashboard

**Problem.** A dashboard shows metrics that update live (active users, queue depth,
recent events). The first paint must be server-rendered and fast; subsequent
updates arrive over a live channel.

**Category.** Server-loaded data (category 3) for the initial snapshot, plus shared
client state (category 2) fed by a real-time transport. Owner: the server for the
seed, the component for the live layer. Lifetime: the page, with the connection
torn down on unmount.

## Shape

```
src/routes/dashboard/
├── +page.server.ts       # initial metrics snapshot (SSR)
├── +page.svelte          # seeds state from load, subscribes to live updates
```

## Seed from load, then layer live updates

```ts
// +page.server.ts
export async function load({ locals }) {
  const metrics = await locals.db.metrics.current();
  return { metrics };                       // server-rendered first paint
}
```

```svelte
<!-- +page.svelte -->
<script>
  import { browser } from '$app/environment';
  import { invalidateAll } from '$app/navigation';

  let { data } = $props();

  // Seed live state from the server snapshot.
  let metrics = $state(data.metrics);

  // Keep in sync if load re-runs (navigation or invalidation).
  $effect(() => { metrics = data.metrics; });

  $effect(() => {
    if (!browser) return;                   // never open a socket during SSR
    const source = new EventSource('/api/metrics/stream');
    source.onmessage = (e) => {
      metrics = { ...metrics, ...JSON.parse(e.data) };
    };
    source.onerror = () => {
      source.close();
      invalidateAll();                      // on drop, refetch the truth
    };
    return () => source.close();            // teardown on unmount
  });
</script>

<MetricCard label="Active users" value={metrics.activeUsers} />
<MetricCard label="Queue depth" value={metrics.queueDepth} />
```

## Why this shape

- `load` provides an SSR-rendered first paint, so the dashboard is useful before
  any socket connects and is indexable and fast. The live layer is additive.
- The transport is opened inside `$effect`, which runs only in the browser, and
  the effect returns a teardown that closes it on unmount. A socket opened at
  module top level would try to connect during SSR and would leak across
  navigations.
- SSE (`EventSource`) fits a one-way server-to-client metrics stream and
  reconnects on its own; a WebSocket would be the choice if the client also sent
  messages.
- On error, the code refetches the authoritative snapshot via `invalidateAll`
  rather than trusting a possibly-gapped stream. Live updates are reconciled with
  server truth, not treated as the sole source.
- Metrics are held in one `$state` object updated by spreading, so the derived
  cards re-render without any store ceremony.

## Notes and alternatives

- SvelteKit's default adapters do not run a persistent WebSocket server inside the
  app process. The `/api/metrics/stream` endpoint here is an SSE response, which
  standard adapters can stream; a bidirectional WebSocket usually needs a separate
  service or a platform feature. Confirm what the target adapter supports.
- If many components across routes need the same live data, consider a cache
  library with revalidation (see `../references/persistence-and-sync.md`) rather
  than duplicating the subscription in each component.
