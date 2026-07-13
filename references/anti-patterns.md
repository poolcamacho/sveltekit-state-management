# Anti-patterns: detection and remediation

State problems that recur in SvelteKit codebases. Each entry covers what it looks
like, how to detect it, why it hurts, and how to fix it incrementally. Frame these
to the user as tendencies to correct, not failures. Most result from a codebase
growing faster than its state model, or from carrying Svelte 4 habits into runes.

The first entry is a security and correctness bug and takes priority over every
style issue below it.

## Table of contents
- [Shared server module state that leaks between users](#module-leak)
- [Global state for route data](#global-route-data)
- [Duplicating server data in multiple stores](#duplicate-server-data)
- [Effects for derived values](#effect-derived)
- [Persistent storage without versioning](#no-versioning)
- [Browser API access during SSR](#ssr-browser-api)
- [Auth trusted only on the client](#client-auth)
- [Excessive writable stores](#excessive-stores)
- [Context as an invisible global](#context-global)
- [State synchronization loops](#sync-loops)
- [Redundant cache layers](#redundant-cache)
- [Stale data after navigation](#stale-after-nav)
- [Non-serializable state crossing the boundary](#non-serializable)
- [Outdated store guidance](#outdated-stores)
- [Overengineering simple component state](#overengineering)

## Shared server module state that leaks between users {#module-leak}
**Severity: high (security and correctness).**
**Looks like.** A top-level `let user`, `let cart`, or `export const store = writable(...)`
in a module that runs on the server, holding data that belongs to one user.
**Detect.** Mutable module-scope variables assigned per request in a
`.server.ts`, hook, or shared module; symptoms described as "works locally but
users see each other's data in production" (local single-user testing hides it).
**Why it hurts.** A server process is shared across all requests and users, so
module scope is shared too. One user's data becomes visible to another. This is a
data-leak class of bug, not a styling nit.
**Fix.** Move per-user state to `event.locals` (set in `hooks.server.ts`), return
it from `load`, and pass it down via context or `load` data. Reserve module scope
for true constants. See `references/ssr-safety.md`. Docs:
https://svelte.dev/docs/kit/state-management

## Global state for route data {#global-route-data}
**Looks like.** A `load` returns data, and a component copies it into a global
store so "other parts of the app can read it".
**Detect.** A store whose value is set from a component's `data` prop or in a
`$effect` mirroring `page.data`; the same field readable from both `load` data and
a store.
**Why it hurts.** The store is a second source of truth that does not update when
the user navigates, so it goes stale, and now two things must be kept in sync.
**Fix.** Read server data where it is needed via the `data` prop or `page.data`
from `$app/state`. If deep components need it, seed context from `load` rather
than a global store. Delete the copy.

## Duplicating server data in multiple stores {#duplicate-server-data}
**Looks like.** The current user, or a list of projects, held in two or three
stores populated at different times.
**Detect.** Several stores initialized from the same endpoint or `load`; update
code that writes the same fact to more than one place.
**Why it hurts.** The copies drift; a bug fixed in one is still wrong in another;
readers disagree about the truth.
**Fix.** Pick one owner (usually `load` data for server truth) and derive
everything else from it. If a computed view is needed, use `$derived`, not another
stored copy.

## Effects for derived values {#effect-derived}
**Looks like.** `$effect(() => { total = items.reduce(...) })` or an effect that
assigns one state variable from others.
**Detect.** Assignments to `$state` inside `$effect` that only mirror other
reactive values; values that are briefly wrong on first render.
**Why it hurts.** It is slower (extra reactivity pass), runs after render (so it is
missing during SSR and flashes on load), and is a common source of loops.
**Fix.** Replace with `$derived` or `$derived.by`. Reserve `$effect` for real side
effects (DOM, storage, subscriptions). See `references/runes-and-stores.md`.

## Persistent storage without versioning {#no-versioning}
**Looks like.** `JSON.parse(localStorage.getItem('prefs'))` read directly into
typed state, with no shape check.
**Detect.** Reads from storage or a cookie with no version field and no fallback;
crashes or wrong behavior after a deploy that changed the shape.
**Why it hurts.** A user with an old stored value hits new code that expects a new
shape, and the app breaks for exactly the returning users you least want to break.
**Fix.** Store a `{ version, data }` envelope. On read, if the version does not
match, migrate or discard and fall back to defaults. See
`references/persistence-and-sync.md`.

## Browser API access during SSR {#ssr-browser-api}
**Looks like.** `const saved = localStorage.getItem(...)` or `window.matchMedia(...)`
at module top level or in component init.
**Detect.** `ReferenceError: localStorage is not defined` (or `window`,
`document`) on the server; errors only on first server render, not in the browser.
**Why it hurts.** These APIs do not exist during SSR, so the render throws.
**Fix.** Access browser APIs inside `$effect` (browser-only) or guard with
`if (typeof window !== 'undefined')`, or use SvelteKit's `browser` flag from
`$app/environment`. Initialize state to a safe default and hydrate the stored
value in an effect. See `references/ssr-safety.md`.

## Auth trusted only on the client {#client-auth}
**Looks like.** A client store `isAdmin` gating a protected action, with no server
check; or a route protected only by hiding a link.
**Detect.** Access decisions made from client state alone; server endpoints that
do not re-verify the user; `load` that trusts a value the client can set.
**Why it hurts.** Client state can be forged. Anyone can call the endpoint
directly. Hidden UI is not access control.
**Fix.** Verify auth on the server in `hooks.server.ts` and in every protected
`load` and action via `event.locals`. Client auth state is for UI convenience
only (showing or hiding controls), never the enforcement point.

## Excessive writable stores {#excessive-stores}
**Looks like.** A dozen `writable` stores for related fields of one concept, or a
store for what is really local component state.
**Detect.** Many small stores updated together; a store read by exactly one
component; store ceremony around a single boolean.
**Why it hurts.** Fragmented state is hard to update atomically and to reason
about, and the ceremony obscures simple logic.
**Fix.** Collapse related fields into one `$state` object (a `.svelte.ts` module
or local state). Demote single-consumer stores to local `$state`. Keep a store
only where the store contract earns its place (see `references/runes-and-stores.md`).

## Context as an invisible global {#context-global}
**Looks like.** Deep components pulling half their data from `getContext` with no
visible prop trail, context used to avoid passing props everywhere.
**Detect.** Components that cannot be understood without hunting for the matching
`setContext`; context holding app-wide constants.
**Why it hurts.** Data flow becomes implicit and components become hard to test in
isolation and to reuse.
**Fix.** Use props for shallow, clear data flow. Reserve context for genuine
per-subtree instances or SSR-safe per-request state. Put true constants in a plain
module, not context.

## State synchronization loops {#sync-loops}
**Looks like.** Effect A writes state B; an effect reading B writes A. Or two
stores kept in sync by mutual subscriptions.
**Detect.** Effects that both read and write overlapping state; "maximum update
depth" style warnings; values oscillating.
**Why it hurts.** Loops waste cycles, can hang, and make behavior unpredictable.
**Fix.** Establish one direction of truth. Derive B from A with `$derived` instead
of syncing both ways. If two-way is truly required, guard writes so a value is not
re-written when unchanged, and prefer a single owner that others read.

## Redundant cache layers {#redundant-cache}
**Looks like.** A cache library (for example TanStack Query) wrapping data that
`load` already fetches and invalidation already refreshes, with no use of the
library's distinct features.
**Detect.** A query client fetching the same endpoint a `load` fetches; two cache
policies for one resource; confusion about which layer is authoritative.
**Why it hurts.** Two caches disagree, invalidation must be done twice, and the
extra layer adds weight for no gain.
**Fix.** Use `load` plus `invalidate`/`invalidateAll` as the base. Add a cache
library only for features `load` lacks (background refetch, window-focus
revalidation, cross-component fine-grained keys), and let it own those cases
clearly rather than shadowing `load`. See `references/persistence-and-sync.md`.

## Stale data after navigation {#stale-after-nav}
**Looks like.** The UI shows old data after moving between pages because a manual
copy was not refreshed.
**Detect.** Data read from a store set once, not from `load`; missing
`invalidate` after a mutation; a form action that changes data but the list does
not update.
**Why it hurts.** Users act on stale information.
**Fix.** Prefer `load` data, which re-runs on navigation. After a mutation, call
`invalidate(dependency)` or `invalidateAll()`, or return updated data from the
action. Avoid manual copies that you must remember to refresh.

## Non-serializable state crossing the boundary {#non-serializable}
**Looks like.** A `load` returns a class instance, a function, a `Map`, or a
`Date`-wrapped custom object that must survive server-to-client transfer.
**Detect.** Errors about non-serializable values from `load`; data that is present
on the server render but wrong or missing after hydration.
**Why it hurts.** `load` data is serialized to send from server to client.
SvelteKit's serialization (via devalue) handles more than JSON (including `Date`,
`Map`, `Set`, `BigInt`), but not class instances with methods or functions, so
those break the boundary.
**Fix.** Return plain, serializable data from `load` and reconstruct rich objects
on the client if needed. Keep behavior (methods) in modules, not in transferred
data. See `references/ssr-safety.md`.

## Outdated store guidance {#outdated-stores}
**Looks like.** New code reaching for `writable` by default, or reading `$page`
via the deprecated `$app/stores`.
**Detect.** `import { page } from '$app/stores'`; store-first shared state where a
`.svelte.ts` rune module would be simpler; tutorials-era patterns in a runes
project.
**Why it hurts.** It ignores the current default, adds subscription ceremony, and
uses a deprecated module slated for removal.
**Fix.** Use `$app/state` for `page`/`navigating`/`updated`, and `.svelte.ts` rune
modules for shared state. Keep stores only where the contract is needed. See
`references/version-verification.md`.

## Overengineering simple component state {#overengineering}
**Looks like.** A store, a context, or a state library introduced for a single
component's toggle or form.
**Detect.** Sharing machinery around state only one component reads; a dependency
added for what `$state` handles natively.
**Why it hurts.** It adds indirection and cognitive load with no benefit, and it
sets a misleading precedent for the codebase.
**Fix.** Use local `$state`/`$derived`. Introduce sharing only when a real second
consumer appears. The simplest correct model wins.
