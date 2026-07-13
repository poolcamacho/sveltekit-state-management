# Example: SSR-safe request state

**Problem.** A request needs derived, per-request values available to server `load`
functions and, in part, to the client: a request id for tracing, the resolved
locale, a feature-flag set for this user. It must be isolated per request and must
not touch browser APIs on the server.

**Category.** Request-scoped server state (category 6), with the client-visible
subset delivered through `load` (category 3). Owner: the request. This example
shows the correct plumbing that keeps state isolated and SSR-safe end to end.

## Shape

```
src/
├── hooks.server.ts        # populate event.locals per request
├── app.d.ts               # types for App.Locals
├── routes/
│   ├── +layout.server.ts  # expose the safe subset to the client
│   └── +layout.svelte     # read locale/flags for rendering
```

## Populate request state in the hook

```ts
// hooks.server.ts
import { randomUUID } from 'node:crypto';

export async function handle({ event, resolve }) {
  event.locals.requestId = randomUUID();                 // per request, not module scope

  const cookieLocale = event.cookies.get('locale');
  const headerLocale = event.request.headers.get('accept-language')?.split(',')[0];
  event.locals.locale = cookieLocale ?? headerLocale ?? 'en';

  const token = event.cookies.get('session');
  event.locals.user = token ? await resolveUser(token) : null;
  event.locals.flags = await resolveFlags(event.locals.user);

  // Use the locale during SSR (for example for the html lang attribute).
  return resolve(event, {
    transformPageChunk: ({ html }) => html.replace('%lang%', event.locals.locale)
  });
}
```

```ts
// app.d.ts
declare global {
  namespace App {
    interface Locals {
      requestId: string;
      locale: string;
      user: { id: string } | null;
      flags: Record<string, boolean>;
    }
  }
}
export {};
```

## Expose only the client-safe subset

```ts
// routes/+layout.server.ts
export function load({ locals }) {
  // requestId stays server-side (tracing/logs). Locale and flags go to the client.
  return { locale: locals.locale, flags: locals.flags };
}
```

```svelte
<!-- routes/+layout.svelte -->
<script>
  import { page } from '$app/state';
  let locale = $derived(page.data.locale);
  let flags = $derived(page.data.flags);
</script>

{#if flags.newNav}<NewNav />{:else}<LegacyNav />{/if}
```

## Why this shape

- Every per-request value lives in `event.locals`, created fresh for each request,
  so nothing leaks between users. There is no module-level `let requestId` or
  `let locale`.
- The locale is resolved on the server and used during SSR (the `lang` attribute),
  which a client-only value read from `localStorage` could not do without a
  hydration mismatch. Because it comes from a cookie or header, the server render
  is correct on the first paint.
- Only the safe subset crosses to the client via `load`. The request id stays on
  the server for logging; it is not leaked into page data.
- The client reads locale and flags from `page.data`, which is serializable and
  refreshes on navigation. No browser API is touched during SSR.

## Contrast with the unsafe version

```ts
// BROKEN: browser API on the server, and module scope
const locale = localStorage.getItem('locale');   // throws during SSR
let requestId = null;                              // shared across requests
```

Reading `localStorage` at module load runs on the server and throws, and a
module-level `requestId` is shared across every request. The request-scoped pattern
above avoids both. See `../references/ssr-safety.md`.
