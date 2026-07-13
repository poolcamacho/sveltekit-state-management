# Example: authenticated user state

**Problem.** The app needs to know who the current user is: to guard routes, to
show or hide UI, and to attribute actions. This must be correct on the server (for
enforcement) and available on the client (for display).

**Category.** Request-scoped server state (category 6) established in hooks, made
visible to the client through `load` data (category 3). Owner: the request on the
server, the page on the client. Never a shared server module. Client auth state is
for display only; the server enforces.

## Shape

```
src/
├── hooks.server.ts            # resolves the session into event.locals.user
├── app.d.ts                   # types App.Locals.user
├── routes/
│   ├── +layout.server.ts      # returns user to the client (shell-scoped)
│   ├── +layout.svelte         # reads user for nav display
│   └── (app)/
│       └── +layout.server.ts  # guards the authed area
```

## Server resolves the session per request

```ts
// hooks.server.ts
export async function handle({ event, resolve }) {
  const token = event.cookies.get('session');
  event.locals.user = token ? await resolveUser(token) : null;
  return resolve(event);
}
```

```ts
// app.d.ts
declare global {
  namespace App {
    interface Locals {
      user: { id: string; name: string; role: 'admin' | 'member' } | null;
    }
  }
}
export {};
```

## Guard on the server, expose to the client

```ts
// routes/(app)/+layout.server.ts
import { redirect } from '@sveltejs/kit';
export function load({ locals }) {
  if (!locals.user) throw redirect(303, '/login');
  return { user: locals.user };
}
```

```svelte
<!-- routes/+layout.svelte -->
<script>
  import { page } from '$app/state';
  let user = $derived(page.data.user ?? null);
</script>

{#if user}
  <span>{user.name}</span>
  {#if user.role === 'admin'}<a href="/admin">Admin</a>{/if}
{:else}
  <a href="/login">Sign in</a>
{/if}
```

## Why this shape

- The user is resolved once per request in the hook and lives in `event.locals`,
  which is isolated per request. There is no module-level `let user`, so there is
  no cross-user leak.
- Enforcement happens on the server: the `(app)` layout load redirects anonymous
  users, and every protected endpoint re-checks `locals.user`. The client cannot
  bypass this.
- The client reads the user from `page.data`, which refreshes on navigation.
  Showing or hiding the Admin link is a convenience, not a security control; the
  admin routes and endpoints verify the role server-side themselves.
- No global auth store. A store copy would go stale after navigation and tempt the
  team into trusting client state.

## Anti-patterns this avoids

- Client-only auth: hiding a link is not access control. Every protected server
  path re-checks `locals.user`.
- Module-state leak: the user is never stored in module scope on the server.
- Trusting local storage: the session lives in an `httpOnly` cookie the client JS
  cannot read or forge, verified on the server. See
  `../references/ssr-safety.md` and `../references/persistence-and-sync.md`.
