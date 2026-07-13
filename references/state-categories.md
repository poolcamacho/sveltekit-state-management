# State categories: lifetime, ownership, and the right API

Every piece of state fits one category. Naming it first fixes the lifetime, the
owner, and the correct API, and prevents most state-management mistakes. Work top
to bottom: prefer the earliest category that correctly holds the state, because
earlier categories are simpler and cheaper.

## Table of contents
- [1. Local UI state](#local-ui)
- [2. Shared client state](#shared-client)
- [3. Server-loaded data](#server-loaded)
- [4. URL state](#url-state)
- [5. Persistent browser state](#persistent-browser)
- [6. Request-scoped server state](#request-scoped)
- [7. Application-wide state](#application-wide)
- [Choosing between categories](#choosing)

## 1. Local UI state {#local-ui}

**Lifetime.** One render lifecycle. It is created when the component mounts and
discarded when it unmounts.

**Owner.** The component. No one else can see it.

**API.** `$state` for the value, `$derived` for anything computed from it.

```svelte
<script>
  let open = $state(false);
  let query = $state('');
  let hasQuery = $derived(query.trim().length > 0);
</script>
```

**When to use.** Toggles, input values, hover and focus flags, the currently
selected tab within one component, transient form field state. This is the
default and the largest category. Most state should stay here.

**When not to.** The moment a sibling or a parent needs the same value, or it
must survive navigation, it is no longer local. Do not lift it before that need
is real; premature lifting couples components.

**Tradeoff.** Local state is the simplest and safest option and has no SSR or
multi-user hazards. Its only limit is reach: it cannot be shared. Resist the
habit of reaching for a store "in case" it is needed later.

## 2. Shared client state {#shared-client}

**Lifetime.** As long as the browser session keeps the owning module or subtree
alive. It survives component unmounts but not a full reload (unless paired with
category 5).

**Owner.** A feature module or a component subtree. Not "the app" by default.

**API, option A: a `.svelte.ts` module of runes.** For state shared across a
feature without needing per-instance copies.

```ts
// src/lib/features/editor/editor-state.svelte.ts
export const editor = $state({ zoom: 1, tool: 'select' });
export function zoomIn() { editor.zoom = Math.min(editor.zoom * 1.2, 8); }
```

Any component imports `editor` and reads or mutates it reactively. This replaces
most classic custom stores. Note: a module-level `$state` singleton is a single
shared instance for the whole client. That is exactly what you want on the
client, but the same pattern on the server is the module-state leak (see
`references/ssr-safety.md`). Keep client-only shared singletons out of code that
runs during SSR, or scope them through context.

**API, option B: context for per-subtree instances.** When each subtree needs its
own copy, or when SSR safety requires per-request instances, create the state and
put it in context at the top of the subtree.

```ts
// wizard-state.svelte.ts
import { setContext, getContext } from 'svelte';
const KEY = Symbol('wizard');
export function createWizard() {
  const state = $state({ step: 0, data: {} });
  return {
    get step() { return state.step; },
    next() { state.step += 1; },
    setField(k, v) { state.data[k] = v; }
  };
}
export function provideWizard() { const w = createWizard(); setContext(KEY, w); return w; }
export function useWizard() { return getContext(KEY); }
```

**When to use.** A multi-step wizard, an editor shared by a toolbar and a canvas,
a filter panel shared by a list and a summary, all within one browser session.

**When not to.** When the data actually comes from the server (category 3) or
should be shareable by link (category 4). A shared client store holding a copy of
server data is a classic drift bug.

**Tradeoff.** Module singleton is simplest but is one instance for the whole
client and is unsafe if it runs on the server with per-user data. Context is
SSR-safe and supports multiple instances but adds a provide/consume step and
makes the data flow less obvious. Choose context when isolation matters, module
singleton when it does not and the data is client-only.

## 3. Server-loaded data {#server-loaded}

**Lifetime.** One request, refreshed on navigation and on explicit invalidation.

**Owner.** The server, per request. The client receives a snapshot.

**API.** A `load` function in `+page.ts`, `+page.server.ts`, `+layout.ts`, or
`+layout.server.ts`. Read it through the `data` prop or `page.data` from
`$app/state`.

```ts
// +page.server.ts
export async function load({ locals }) {
  const orders = await locals.db.orders.forUser(locals.user.id);
  return { orders };
}
```

```svelte
<script>
  let { data } = $props();
</script>
{#each data.orders as order}...{/each}
```

**When to use.** Almost all application data: the user's orders, a product, a
dashboard's numbers. This is the correct home for server-owned data and it is
already reactive to navigation.

**When not to.** For ephemeral UI state, or for data that should be shareable by
link (put the identifying part in the URL and load from it).

**Tradeoff.** `load` data is automatically fresh on navigation and on
`invalidate`, needs no manual store, and is SSR-safe. Its cost is that updates
are request-driven; for live updates you layer real-time on top (category
crossing, see `references/persistence-and-sync.md`). Do not copy `load` data into
a store to "make it global": that copy goes stale the instant the user navigates,
and you now maintain two sources of truth. If several components deep in the tree
need it, read `page.data` or use context seeded from `load`, not a global store.

## 4. URL state {#url-state}

**Lifetime.** As long as the URL, which means across reloads, in bookmarks, in
shared links, and in browser history.

**Owner.** The URL.

**API.** `page.url.searchParams` from `$app/state` for query parameters, route
`params` for path parameters, and `goto` to update the query. A `load` can read
`url` and return derived data.

```ts
// +page.ts
export function load({ url }) {
  const sort = url.searchParams.get('sort') ?? 'recent';
  const pageNum = Number(url.searchParams.get('page') ?? '1');
  return { sort, pageNum };
}
```

```svelte
<script>
  import { page } from '$app/state';
  import { goto } from '$app/navigation';
  function setSort(sort) {
    const params = new URLSearchParams(page.url.searchParams);
    params.set('sort', sort);
    goto(`?${params}`, { keepFocus: true });
  }
</script>
```

**When to use.** Filters, search terms, sort order, pagination, the active tab of
a shareable view, a selected entity id. Anything a user might reload into,
bookmark, or send to a colleague.

**When not to.** For high-frequency ephemeral changes (a slider being dragged) or
secrets. Never place personal or sensitive data in query strings.

**Tradeoff.** URL state is shareable, survives reload, and integrates with `load`
and history for free, which no client store gives you. The cost is that every
change is a navigation (mitigated with `keepFocus`, `noScroll`, and
`replaceState`), and values are strings you must parse and validate.

## 5. Persistent browser state {#persistent-browser}

**Lifetime.** Local storage: until cleared. Session storage: until the tab
closes. Cookies: until expiry, and they travel to the server on every request.

**Owner.** The browser, with a version tag you control.

**API.** `localStorage` / `sessionStorage` (client only, guard against SSR), or
cookies (`document.cookie` on the client, `cookies` in server `load`, actions,
and hooks). Wrap access so SSR never touches the browser API. See
`references/persistence-and-sync.md` for a versioned, SSR-safe wrapper.

**When to use.** Local storage for client-only preferences (theme, sidebar
collapsed, last-used values). Cookies when the server must read the value during
SSR (locale, session token). Session storage for per-tab scratch state.

**When not to.** For a trusted auth decision (a client value can be forged; the
server must verify). For large blobs (storage is small and synchronous). For
server-owned data (that is category 3).

**Tradeoff.** Persistence survives reload, which runes cannot. The costs are
schema drift (an old stored shape breaks new code unless you version it), SSR
hazards (the API does not exist on the server), and, for cookies, size limits and
the need to set `httpOnly`, `secure`, and `sameSite` correctly.

## 6. Request-scoped server state {#request-scoped}

**Lifetime.** One server request.

**Owner.** The request.

**API.** `event.locals`, populated in `hooks.server.ts`, read in server `load`
and actions. This is how per-user data reaches the server code that needs it
without ever touching module scope.

```ts
// hooks.server.ts
export async function handle({ event, resolve }) {
  const token = event.cookies.get('session');
  event.locals.user = token ? await resolveUser(token) : null;
  return resolve(event);
}
```

```ts
// +page.server.ts
export function load({ locals }) {
  if (!locals.user) throw redirect(303, '/login');
  return { user: locals.user };
}
```

**When to use.** The authenticated user, a tenant id, a request id, a per-request
database transaction. Anything specific to one request.

**When not to.** Never store per-user or per-request data in a module-level
variable on the server. Server modules are shared across all requests, so that
data leaks between users. This is the highest-severity anti-pattern; see
`references/ssr-safety.md`.

**Tradeoff.** `event.locals` is the SSR-safe channel for per-user server state
and integrates with hooks and `load`. It costs a small amount of plumbing
(populate in hooks, read in load), which is exactly the plumbing that keeps users
isolated.

## 7. Application-wide state {#application-wide}

**Lifetime.** The lifetime of the client application, independent of any user or
request.

**Owner.** The application.

**API.** A plain module singleton (a `.svelte.ts` rune object) is acceptable here
because the data is genuinely shared and not per-user.

```ts
// src/lib/theme.svelte.ts
export const theme = $state({ mode: 'system' });
```

**When to use.** A theme preference, a set of feature flags fetched once at
startup, a design-time constant table. The test: is it truly the same for every
user and safe to share? If yes, this category fits.

**When not to.** For anything that varies by user or request. Most state people
call "global" is really server data (category 3) or URL state (category 4) in
disguise. Application-wide state is rare and each instance should be a deliberate,
documented choice.

**Tradeoff.** A global singleton is the simplest possible sharing mechanism, but
it is also the easiest to misuse. On the server it is unsafe for per-user data,
and even on the client it makes data flow implicit. Reserve it for the few things
that are truly app-wide.

## Choosing between categories {#choosing}

Ask these questions in order and stop at the first that fits:

1. Does only one component need it, ephemerally? Category 1, local rune.
2. Should it survive reload or be shareable by link? Category 4 (URL) if
   shareable, category 5 (storage/cookies) if client-persistent.
3. Does it come from the server or belong to the server? Category 3 (`load`) for
   client-visible data, category 6 (`event.locals`) for per-request server data.
4. Do several components in one session share it, and it is not server data?
   Category 2, shared client state (module singleton or context).
5. Is it truly the same for every user forever? Category 7, application-wide.

Most mistakes are a piece of state placed one category too high: server data
copied into a global store, or a filter kept in a rune instead of the URL. When
in doubt, choose the lower-numbered category that still works.
