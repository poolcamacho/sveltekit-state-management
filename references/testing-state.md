# Testing stateful code

State is where subtle bugs live, so the transitions (not just the final values)
are what to test. Match the test type to the kind of state. Confirm tooling
against the current SvelteKit testing docs; the stack below is the current stable
default.

## Table of contents
- [What to test at each level](#levels)
- [Unit-testing rune logic](#unit-runes)
- [Testing stores](#stores)
- [Component tests for reactive UI](#component)
- [Testing load functions and server state](#load)
- [End to end for full flows](#e2e)
- [Testing the hazards](#hazards)

## What to test at each level {#levels}

- Unit (Vitest): pure state logic extracted into `.ts`/`.svelte.ts` modules.
  Fastest and most valuable. Test transitions: given state X and action A, expect
  state Y.
- Component (Vitest browser mode with vitest-browser-svelte): reactive UI,
  bindings, derived rendering, effects that touch the DOM.
- End to end (Playwright): flows that cross the server and client boundary,
  navigation, persistence, and auth.

Push logic down into modules so most of it is unit-testable without a browser. A
component that only renders derived state needs far less testing than one holding
business rules.

## Unit-testing rune logic {#unit-runes}

Runes work outside components in `.svelte.ts` files, so a factory that returns
state and methods is directly testable. Vitest runs `.svelte.ts` through the
Svelte plugin.

```ts
// counter.svelte.ts
export function createCounter(start = 0) {
  let count = $state(start);
  return {
    get count() { return count; },
    inc() { count += 1; },
    reset() { count = start; }
  };
}
```

```ts
// counter.svelte.test.ts
import { describe, it, expect } from 'vitest';
import { createCounter } from './counter.svelte';

describe('counter', () => {
  it('increments and resets', () => {
    const c = createCounter(2);
    c.inc();
    expect(c.count).toBe(3);
    c.reset();
    expect(c.count).toBe(2);
  });
});
```

To assert on `$derived` or `$effect` reactions inside a test, wrap the code in
`$effect.root(() => { ... })` so effects run outside a component, and call the
returned cleanup when done. Prefer testing derived values by reading the getter
after a mutation, which needs no effect.

## Testing stores {#stores}

Classic stores remain easy to test with `get(store)` from `svelte/store` and by
subscribing. If you still maintain stores, assert the value after each action and
that derived stores recompute. When migrating a store to a rune factory, keep the
old test as a behavior contract and point it at the new implementation.

## Component tests for reactive UI {#component}

Use Vitest browser mode with vitest-browser-svelte to render a component and
assert on what the user sees as state changes.

```ts
import { render } from 'vitest-browser-svelte';
import { expect, test } from 'vitest';
import Counter from './Counter.svelte';

test('button updates the count', async () => {
  const screen = render(Counter, { props: { start: 0 } });
  await screen.getByRole('button').click();
  await expect.element(screen.getByText('1')).toBeInTheDocument();
});
```

Test bindings by driving the input and asserting the bound value's effect, and
test derived rendering by changing inputs and asserting the output. Keep component
tests focused on reactivity and rendering; put pure logic in unit tests.

## Testing load functions and server state {#load}

`load` functions are plain functions; call them with a mock event.

```ts
import { describe, it, expect, vi } from 'vitest';
import { load } from './+page.server';

it('redirects anonymous users', async () => {
  const event = { locals: { user: null } };
  await expect(load(event)).rejects.toMatchObject({ status: 303 });
});

it('returns the user orders', async () => {
  const event = {
    locals: { user: { id: 'u1' }, db: { orders: { forUser: vi.fn().mockResolvedValue([{ id: 'o1' }]) } } }
  };
  const result = await load(event);
  expect(result.orders).toHaveLength(1);
});
```

For request-scoped isolation, add a test that two different `locals.user` values
produce independent results, which guards against reintroducing module-scope
leaks.

## End to end for full flows {#e2e}

Use Playwright for anything that spans the boundary or persists:

- Auth: log in, confirm a protected page loads, confirm logout blocks it.
- URL state: apply a filter, reload, confirm it persists via the query string.
- Persistence: set a preference, reload, confirm it survives; where relevant open
  a second tab and confirm cross-tab sync.
- Optimistic updates: perform the action, confirm the immediate UI change, and
  simulate a failure to confirm rollback.
- Navigation freshness: mutate data, navigate, confirm the list reflects the
  change (guards against stale-after-navigation).

## Testing the hazards {#hazards}

Write explicit tests for the high-risk patterns:

- Multi-user isolation: two concurrent simulated users must not see each other's
  data. A `load` unit test with two different `locals` values, plus an end-to-end
  check with two sessions, catches module-state leaks.
- SSR safety: a server render (or a `load` unit test) must not throw from a
  browser API. Running the component test suite in both server and browser
  environments surfaces top-level browser access.
- Versioned persistence: seed storage with an old version tag and assert the code
  falls back to defaults or migrates, rather than crashing.
- Derived vs effect: assert a derived value is correct on first read (no
  post-render flash), which fails if it was implemented as an effect.
