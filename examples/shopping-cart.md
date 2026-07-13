# Example: shopping cart

**Problem.** A cart holds line items across pages, should survive a reload, and
must be authoritative on the server at checkout (price and stock cannot be trusted
from the client).

**Category.** A split. The working cart is shared client state (category 2) with
browser persistence (category 5) for reload survival. The authoritative cart at
checkout is server-owned (category 3 and 6). Owner: the client for the draft, the
server for the transaction.

## Shape

```
src/lib/features/cart/
├── cart.svelte.ts            # client cart: shared state + versioned persistence
└── CartButton.svelte
src/routes/
├── checkout/
│   ├── +page.server.ts       # server recomputes totals, validates stock
│   └── +page.svelte
```

## Client cart: shared state with versioned, SSR-safe persistence

```ts
// cart.svelte.ts
import { browser } from '$app/environment';

const KEY = 'cart';
const VERSION = 2;

function load() {
  if (!browser) return [];
  try {
    const raw = localStorage.getItem(KEY);
    if (!raw) return [];
    const parsed = JSON.parse(raw);
    return parsed.version === VERSION ? parsed.data : []; // discard old shape
  } catch { return []; }
}

let items = $state(load());

$effect.root(() => {
  $effect(() => {
    if (browser) localStorage.setItem(KEY, JSON.stringify({ version: VERSION, data: items }));
  });
});

export const cart = {
  get items() { return items; },
  get count() { return items.reduce((n, i) => n + i.qty, 0); },
  get subtotal() { return items.reduce((n, i) => n + i.qty * i.price, 0); },
  add(product) {
    const line = items.find((i) => i.id === product.id);
    if (line) line.qty += 1; else items.push({ id: product.id, price: product.price, qty: 1 });
  },
  remove(id) { items = items.filter((i) => i.id !== id); }
};
```

## Server is authoritative at checkout

```ts
// checkout/+page.server.ts
export const actions = {
  default: async ({ request, locals }) => {
    const submitted = await request.formData();               // ids + quantities
    const ids = submitted.getAll('id');

    // Recompute prices and stock from the database, never trust client prices.
    const priced = await locals.db.products.priceAndCheck(ids);
    if (priced.some((p) => !p.inStock)) {
      return fail(409, { error: 'Some items are out of stock.' });
    }
    const order = await locals.db.orders.create(locals.user.id, priced);
    throw redirect(303, `/orders/${order.id}`);
  }
};
```

## Why this shape

- The draft cart is client state because it is per-session, edited constantly, and
  needs to feel instant. `subtotal` and `count` are derived getters, not stored
  copies.
- Persistence uses a `VERSION` tag so a schema change discards or migrates old
  carts instead of crashing returning shoppers, and all `localStorage` access is
  `browser`-guarded so SSR never touches it.
- The server recomputes prices and checks stock at checkout. The client cart is a
  convenience; the money decision is server-owned and cannot be tampered with from
  the browser.
- The client cart is a module singleton here, which is acceptable because it holds
  no per-user secret and never runs on the server with per-user data (the guard
  keeps it client-only). For a logged-in cart that must sync across devices,
  persist it server-side and load it via `load` instead.

## When to change categories

If carts must survive across devices or be visible to support staff, promote the
cart to server-owned storage, load it via `load`, and mutate it through actions or
endpoints. The client state then becomes an optimistic view over server truth (see
`real-time-dashboard.md` and `../references/persistence-and-sync.md`).
