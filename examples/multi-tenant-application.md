# Example: multi-tenant application

**Problem.** One deployment serves many tenants (organizations). Every request must
be scoped to the correct tenant, and no tenant may ever see another's data. Tenant
context must reach every query without being spoofable from the client.

**Category.** Request-scoped server state (category 6) as the backbone, with
server-loaded data (category 3) scoped to the tenant. Owner: the request. The
tenant id is derived server-side and never taken from client state. This is the
highest-stakes isolation case; the module-state leak here is a cross-tenant breach.

## Shape

```
src/
├── hooks.server.ts            # derive tenant from host/header/session -> locals
├── app.d.ts                   # types App.Locals.tenant
├── lib/server/
│   └── db.ts                  # queries that require a tenant id argument
├── routes/
│   └── (app)/
│       ├── +layout.server.ts  # loads tenant-scoped shell data
│       └── projects/+page.server.ts
```

## Derive the tenant server-side, per request

```ts
// hooks.server.ts
export async function handle({ event, resolve }) {
  const host = event.url.hostname;               // e.g. acme.app.com
  const subdomain = host.split('.')[0];
  const tenant = await lookupTenant(subdomain);  // server-side, authoritative
  if (!tenant) return new Response('Unknown tenant', { status: 404 });

  event.locals.tenant = tenant;
  const token = event.cookies.get('session');
  event.locals.user = token ? await resolveUser(token, tenant.id) : null;
  return resolve(event);
}
```

```ts
// app.d.ts
declare global {
  namespace App {
    interface Locals {
      tenant: { id: string; name: string };
      user: { id: string; role: string } | null;
    }
  }
}
export {};
```

## Every query takes the tenant id explicitly

```ts
// lib/server/db.ts
export function projectsForTenant(tenantId, userId) {
  // The tenant id is a required argument, never a module-scope default.
  return db.query('select * from projects where tenant_id = $1', [tenantId]);
}
```

```ts
// routes/(app)/projects/+page.server.ts
export function load({ locals }) {
  return { projects: projectsForTenant(locals.tenant.id, locals.user.id) };
}
```

## Why this shape

- The tenant is resolved from the host (or a header, or the session) on the
  server, so a client cannot claim a different tenant by editing a value. The id
  lives in `event.locals`, isolated per request.
- Every data-access function requires the tenant id as an argument. There is no
  ambient "current tenant" in module scope, which is what would leak across
  requests under load and breach isolation.
- Sessions are validated against the tenant, so a valid token for tenant A cannot
  read tenant B.
- Client code receives only tenant-scoped data through `load`. There is no global
  tenant store that could show the wrong tenant after a navigation.

## The failure to avoid

```ts
// NEVER: a module-scope current tenant
let currentTenant = null;                     // shared across all requests
export function setTenant(t) { currentTenant = t; }
```

Under concurrency, request B reads the tenant request A set, and one organization
sees another's data. In a multi-tenant system this is the most severe possible
bug. Keep tenant state request-scoped, pass the id explicitly, and test isolation
with two concurrent tenants (see `../references/testing-state.md` and
`../references/ssr-safety.md`).
