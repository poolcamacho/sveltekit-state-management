# Runes, stores, and context

The reactive primitives, when to use each, and why runes are the current default
for new shared state. Confirm API details against
https://svelte.dev/docs/svelte/what-are-runes and
https://svelte.dev/docs/svelte/stores before asserting stability.

## Table of contents
- [$state](#state)
- [$derived vs $effect](#derived-vs-effect)
- [$props and $bindable](#props)
- [Bindings](#bindings)
- [Stores: when they still fit](#stores)
- [Custom stores vs .svelte.ts modules](#custom-stores)
- [Context](#context)
- [Migration notes](#migration)

## $state {#state}

`$state` declares reactive state. Reading it in markup or in a `$derived`/`$effect`
tracks it; assigning to it triggers updates. It works in `.svelte` components and,
crucially, in `.svelte.ts` and `.svelte.js` modules, which is what lets shared
state live outside a component.

```svelte
<script>
  let count = $state(0);
  let list = $state([]);            // arrays and objects are deeply reactive
</script>
<button onclick={() => count++}>{count}</button>
```

Deep reactivity means mutating `list.push(x)` or `obj.field = y` is tracked; you
do not need to reassign the whole value as you did with stores. For a large object
you never mutate internally, `$state.raw` avoids the proxy overhead.

## $derived vs $effect {#derived-vs-effect}

This is the most common runes mistake and worth stating plainly.

**`$derived` is for computing a value from other reactive values.** It is lazy,
cached, and has no side effects. Use it whenever a value is a pure function of
other state.

```svelte
<script>
  let items = $state([]);
  let total = $derived(items.reduce((s, i) => s + i.price, 0));
  let expensive = $derived(items.filter((i) => i.price > 100));
</script>
```

For multi-step computation use `$derived.by(() => { ... })`.

**`$effect` is for side effects that must run when state changes**: talking to a
non-reactive browser API, drawing to a canvas, setting up and tearing down a
subscription, logging. It is not for producing a value.

```svelte
<script>
  let theme = $state('dark');
  $effect(() => {
    document.documentElement.dataset.theme = theme; // side effect, correct
  });
</script>
```

**Why effects are not for derived values.** Writing state inside an effect to
"compute" something is slower (an extra reactivity cycle), runs after render
rather than during, and easily forms loops (effect A writes B, an effect reading B
writes A). It also breaks the mental model: `$derived` declares a relationship,
`$effect` schedules work. The rule: if you are assigning to a state variable
inside `$effect` to mirror another value, it should almost certainly be
`$derived`. Effects also run only in the browser after mount, so using one to
derive a value means that value is missing during SSR.

Legitimate `$effect` uses: syncing to `localStorage`, wiring a WebSocket,
integrating a third-party chart, focusing an element. All of these touch the world
outside Svelte's reactivity, which is exactly the point.

## $props and $bindable {#props}

`$props` reads component inputs. Props are one-way by default: the parent owns the
value, the child reads it. This is the correct default because it keeps ownership
clear.

```svelte
<script>
  let { label, count = 0, onchange } = $props();
</script>
```

`$bindable` opts a prop into two-way binding, so the child can write back to the
parent's state via `bind:`. Use it sparingly, for genuine input-like components
(a custom text field, a toggle) where two-way flow is the natural contract.

```svelte
<!-- child -->
<script>
  let { value = $bindable('') } = $props();
</script>
<input bind:value />
```

```svelte
<!-- parent -->
<CustomInput bind:value={name} />
```

**When not to use `$bindable`.** For passing data down that the child only reads
(plain prop), or for events the parent should handle (a callback prop like
`onchange` keeps the parent in control). Overusing `bind:` makes ownership
ambiguous: two components can write the same value and it becomes unclear who is
authoritative.

## Bindings {#bindings}

`bind:value`, `bind:checked`, `bind:group`, and element bindings (`bind:this`,
`bind:clientWidth`) connect DOM or child state to a `$state` variable. They are
convenient for forms and are the idiomatic way to read live input values.

Tradeoff: a binding is a two-way channel, so it blurs the one-way data flow that
makes state easy to reason about. For a form it is worth it. For cross-component
application state, prefer explicit reads and a single writer. Do not use bindings
to smuggle shared state between distant components; that is what context or a
shared module is for.

## Stores: when they still fit {#stores}

Svelte stores (`writable`, `readable`, `derived` from `svelte/store`) are still
supported and still correct in specific cases. They are not deprecated. What
changed is that they are no longer the default tool for in-app shared state; runes
in a `.svelte.ts` module cover that more simply.

Keep or reach for a store when:

- You need the store contract itself: something external calls `subscribe`, or you
  interoperate with a library that expects a store.
- You are wrapping an event source (`readable` with a start/stop function over a
  WebSocket, an interval, or `matchMedia`), where the store's lifecycle callback
  is a clean fit.
- You are composing derived values across store boundaries with `derived`.
- Existing, working store code does not need rewriting. Do not migrate stores to
  runes for fashion; migrate when you touch the code for another reason or when
  the store is fighting you.

The `$store` auto-subscription syntax still works in components. In a `.svelte.ts`
module you subscribe manually or convert to runes.

## Custom stores vs .svelte.ts modules {#custom-stores}

The classic "custom store" pattern (a function returning `{ subscribe, ...methods }`
around a `writable`) is largely replaced by a `.svelte.ts` module or a factory
returning an object with `$state` and getters.

Old custom store:

```ts
function createCounter() {
  const { subscribe, update, set } = writable(0);
  return { subscribe, inc: () => update((n) => n + 1), reset: () => set(0) };
}
```

Runes equivalent:

```ts
// counter.svelte.ts
export function createCounter() {
  let count = $state(0);
  return {
    get count() { return count; },
    inc() { count += 1; },
    reset() { count = 0; }
  };
}
```

The runes version reads as plain state and methods, is deeply reactive, and needs
no subscription. Expose reads via getters so callers cannot bypass your methods.
Prefer this for new shared logic. Keep the store version if something still relies
on `subscribe`.

## Context {#context}

`setContext(key, value)` in a parent makes `value` available to
`getContext(key)` in any descendant, scoped to that component subtree and to the
current render. Context is the SSR-safe way to share an instance without a global,
because each render (each request, in SSR) gets its own context.

Use context for:

- Per-subtree state that should not be a global singleton (a wizard, a form
  controller, a scoped theme).
- SSR-safe sharing of per-request state seeded from `load`, so no server module
  holds per-user data.
- Dependency injection of a service instance down a tree.

Do not use context as an invisible global. Its weaknesses are that data flow
becomes implicit (a descendant reads state with no visible prop), and that it only
works within the component tree during a render, so it is not a place for truly
app-wide constants (use a module for those) or for anything a non-component needs.
A good rule: if a prop would be clear and shallow, use a prop; reach for context
when prop-drilling would cross many uninterested layers or when you need SSR-safe
per-request isolation.

Pair context with runes by putting a `$state`-backed object into context, so
descendants get reactive reads:

```ts
const KEY = Symbol('theme');
export function provideTheme() {
  const theme = $state({ mode: 'system' });
  setContext(KEY, theme);
  return theme;
}
export function useTheme() { return getContext(KEY); }
```

## Migration notes {#migration}

- Prefer `$app/state` over the deprecated `$app/stores`. `page`, `navigating`,
  and `updated` are available as reactive objects from `$app/state` (which
  requires runes). Reading `page.data`, `page.url`, `page.params` replaces the
  `$page` store. See https://svelte.dev/docs/kit/$app-state.
- Replace `$: x = f(a, b)` reactive statements with `let x = $derived(f(a, b))`.
- Replace `$: { sideEffect() }` with `$effect(() => { sideEffect() })`, and only
  when it is truly a side effect.
- Replace most custom stores with `.svelte.ts` factories as above.
- Do not present store-first patterns as current best practice. Runes are the
  default; stores are a targeted tool.
