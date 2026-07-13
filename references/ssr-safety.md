# SSR safety, hydration, and multi-user isolation

Server-side rendering makes SvelteKit state management different from a pure SPA.
Code runs on the server first, then hydrates in the browser. Two hazards dominate:
per-user data placed in shared server scope, and browser-only APIs touched during
server render. Confirm details against
https://svelte.dev/docs/kit/state-management.

## Table of contents
- [The module-state leak](#module-leak)
- [The request-scoped pattern](#request-scoped)
- [Browser APIs during SSR](#browser-apis)
- [Hydration mismatches](#hydration)
- [Serialization across the boundary](#serialization)
- [Multi-user isolation checklist](#isolation-checklist)

## The module-state leak {#module-leak}

The single most dangerous state bug in SvelteKit. A module's top-level scope is
evaluated once per server process and shared by every request that process
handles. Putting per-user data there leaks it between users.

```ts
// BROKEN: src/lib/server/session.ts
let currentUser = null;               // shared across ALL requests
export function setUser(u) { currentUser = u; }
export function getUser() { return currentUser; }
```

Under concurrent load, request B can read the `currentUser` that request A just
set. It appears to work in local single-user testing, which is why it reaches
production. The same applies to a module-scope `writable` store holding per-user
data on the server.

The rule: on the server, module scope is for constants and stateless helpers
only. Anything that varies by user or request must be request-scoped.

## The request-scoped pattern {#request-scoped}

Thread per-user state through the request, not through module scope.

```ts
// hooks.server.ts
export async function handle({ event, resolve }) {
  const token = event.cookies.get('session');
  event.locals.user = token ? await resolveUser(token) : null;
  event.locals.tenantId = event.request.headers.get('x-tenant') ?? null;
  return resolve(event);
}
```

```ts
// app.d.ts
declare global {
  namespace App {
    interface Locals {
      user: { id: string; role: string } | null;
      tenantId: string | null;
    }
  }
}
export {};
```

```ts
// +page.server.ts
export function load({ locals }) {
  if (!locals.user) throw redirect(303, '/login');
  return { user: locals.user };
}
```

`event.locals` exists for exactly one request, so there is nothing to leak. To
make per-user state reactive on the client, return it from `load` and, if deep
components need it, seed a context at a layout so each render gets its own
instance. Never bridge it through a server module singleton.

If you need a per-request database transaction or client, create it in the hook,
attach it to `event.locals`, and clean it up after `resolve`. Do not cache a
per-user connection in module scope.

## Browser APIs during SSR {#browser-apis}

`window`, `document`, `localStorage`, `sessionStorage`, `navigator`, and
`matchMedia` do not exist on the server. Touching them during server render
throws.

Safe patterns:

```svelte
<script>
  import { browser } from '$app/environment';

  // Start with a safe default that renders on the server.
  let theme = $state('system');

  // Hydrate the stored value in an effect, which runs only in the browser.
  $effect(() => {
    const saved = localStorage.getItem('theme');
    if (saved) theme = saved;
  });

  // Persist on change, also browser-only.
  $effect(() => {
    localStorage.setItem('theme', theme);
  });
</script>
```

Use `browser` from `$app/environment` when you need a one-off guard outside an
effect. Prefer an effect for read-then-write storage sync, because effects never
run on the server. Do not read `localStorage` at module top level in code that is
imported into a server render path.

## Hydration mismatches {#hydration}

Hydration expects the browser's first render to match the server's HTML. If state
differs between the two (for example, a value read from `localStorage` used
directly in the initial markup, or `Math.random()`/`Date.now()` in render), the
DOM and the server HTML disagree and you get a hydration warning or a visible
flash.

Avoid by rendering a deterministic default on the server and applying the
client-only value after mount (in an effect), accepting a brief, intentional
update. For content that is truly client-only, gate it on `browser` so the server
renders a placeholder.

## Serialization across the boundary {#serialization}

Data returned from `load` is serialized on the server and revived on the client.
SvelteKit uses devalue, which supports more than JSON: `Date`, `Map`, `Set`,
`BigInt`, `RegExp`, and repeated references. It does not support class instances
with methods, functions, or anything whose behavior cannot be serialized.

Guidance:

- Return plain data (objects, arrays, and the devalue-supported types) from
  `load`.
- Keep behavior in modules. If the client needs a rich object, reconstruct it
  from the plain data on the client.
- Do not put functions, class instances, or DOM nodes in `load` return values.
- The same applies to values you persist: a `Map` can go into devalue-backed
  transfer, but `JSON.stringify` for `localStorage` drops it, so choose the
  serializer that matches the sink.

## Multi-user isolation checklist {#isolation-checklist}

Run this on any server-side state:

1. Is any per-user or per-request value stored in a module-level variable or a
   module-scope store on the server? If yes, move it to `event.locals`. High
   severity.
2. Does every protected `load` and action re-check `event.locals.user`, rather
   than trusting a client value?
3. Are database clients or transactions per-request, not cached per-user in
   module scope?
4. In a multi-tenant app, is the tenant id derived server-side (from host,
   header, or session) and applied to every query, never taken from client state?
5. Is client-visible per-user state delivered through `load` data or
   context-seeded-from-load, not a global store?
6. Do browser-only reads happen in effects or under a `browser` guard, so SSR
   never touches them?

A clean answer to all six means the app isolates users correctly and renders
safely on the server.
