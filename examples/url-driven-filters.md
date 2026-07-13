# Example: URL-driven filters

**Problem.** A product list has a category filter, a sort order, a search term, and
pagination. Users want to reload into the same view, bookmark it, and share the
link. Results are fetched from the server based on these values.

**Category.** URL state (category 4), feeding server-loaded data (category 3).
Owner: the URL. Lifetime: as long as the link. The filters drive a `load` that
returns the matching data.

## Shape

```
src/routes/products/
├── +page.ts              # reads url.searchParams, returns filters + data
├── +page.svelte          # renders results, updates the URL on change
└── FilterBar.svelte      # controls that write to the query string
```

## Load reads the URL

```ts
// +page.ts
export async function load({ url, fetch }) {
  const category = url.searchParams.get('category') ?? 'all';
  const sort = url.searchParams.get('sort') ?? 'popular';
  const q = url.searchParams.get('q') ?? '';
  const page = Number(url.searchParams.get('page') ?? '1');

  const res = await fetch(
    `/api/products?category=${category}&sort=${sort}&q=${encodeURIComponent(q)}&page=${page}`
  );
  const { items, totalPages } = await res.json();

  return { category, sort, q, page, items, totalPages };
}
```

## Components update the URL, not a store

```svelte
<!-- FilterBar.svelte -->
<script>
  import { page } from '$app/state';
  import { goto } from '$app/navigation';

  function setParam(key, value) {
    const params = new URLSearchParams(page.url.searchParams);
    if (value) params.set(key, value); else params.delete(key);
    params.set('page', '1');                       // reset paging on filter change
    goto(`?${params}`, { keepFocus: true, noScroll: true });
  }
</script>

<select onchange={(e) => setParam('category', e.currentTarget.value)}>...</select>
<input oninput={(e) => setParam('q', e.currentTarget.value)} />
```

```svelte
<!-- +page.svelte -->
<script>
  let { data } = $props();
</script>
{#each data.items as item}...{/each}
```

## Why this shape

- The filters live in the URL, so reload, bookmark, and share all work with zero
  extra code, and the back button navigates filter history.
- `load` re-runs when the query string changes, so the results are always fresh
  for the current filters. There is no client store to keep in sync and no
  stale-after-navigation risk.
- `goto` with `keepFocus` and `noScroll` keeps the interaction smooth despite each
  change being a navigation. For a rapidly-typed search box, debounce the
  `setParam` call, or use `replaceState` to avoid flooding history.
- Values are parsed and defaulted in one place (`load`), so the rest of the app
  sees typed, validated filters.

## When not to use the URL

For a transient UI toggle (an open filter drawer) keep local `$state`; it should
not pollute the URL. For very high-frequency continuous input (a live-dragged
range slider) hold the value in a rune and commit to the URL on release. Never put
sensitive values in the query string.
