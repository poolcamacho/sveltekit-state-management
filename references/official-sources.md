# Official sources

The official documentation and repositories consulted for this skill. Prefer these
over memory when giving concrete API or stability advice, and re-check them when a
detail is version-sensitive. Each line notes what the source covers.

## Documentation

- https://svelte.dev/docs/kit/state-management
  The primary SvelteKit state-management guide: avoiding shared server state,
  using `load` and `event.locals`, context, snapshots, and stores.
- https://svelte.dev/docs/svelte/what-are-runes
  What runes are and the reactivity model overview.
- https://svelte.dev/docs/svelte/$state
  `$state`, deep reactivity, and `$state.raw`.
- https://svelte.dev/docs/svelte/$derived
  `$derived` and `$derived.by` for computed values.
- https://svelte.dev/docs/svelte/$effect
  `$effect`, `$effect.pre`, `$effect.root`, and why effects are not for derived
  values.
- https://svelte.dev/docs/svelte/$props
  `$props` for component inputs.
- https://svelte.dev/docs/svelte/$bindable
  `$bindable` for two-way prop binding.
- https://svelte.dev/docs/svelte/stores
  Svelte stores (`writable`, `readable`, `derived`) and when they still apply.
- https://svelte.dev/docs/svelte/context
  `setContext`/`getContext` and component-tree-scoped state.
- https://svelte.dev/docs/kit/$app-state
  `page`, `navigating`, and `updated` reactive objects (the successor to
  `$app/stores`).
- https://svelte.dev/docs/kit/load
  `load` functions, page and layout data, `url`, `params`, dependencies, and
  invalidation.
- https://svelte.dev/docs/kit/form-actions
  Form actions and progressive enhancement with `use:enhance` for
  server-confirmed state.
- https://svelte.dev/docs/kit/hooks
  `hooks.server.ts`, `handle`, and populating `event.locals`.
- https://svelte.dev/docs/kit/@sveltejs-kit#Cookies
  The `cookies` API for reading and setting cookies on the server.
- https://svelte.dev/docs/kit/$env-static-private
  Private environment variables and the server boundary.
- https://svelte.dev/docs/kit/testing
  The current testing guidance (Vitest, Vitest browser mode, Playwright).
- https://svelte.dev/docs/kit/migrating-to-sveltekit-2
  Migration notes, including deprecations relevant to state.

## Repositories and release notes

- https://github.com/sveltejs/svelte
  Svelte source, releases, and changelog. Confirm the current stable version and
  runes behavior.
- https://github.com/sveltejs/kit
  SvelteKit source, releases, and changelog. Confirm the current stable version,
  `$app/state`, and adapter behavior.
- https://github.com/sveltejs/svelte/blob/main/packages/svelte/CHANGELOG.md
  Svelte changelog for exact version-to-feature mapping.
- https://github.com/sveltejs/kit/blob/main/packages/kit/CHANGELOG.md
  SvelteKit changelog, including when features moved from experimental to stable.

## How to use these

When a user asks about a feature's stability or an API's current shape, open the
relevant page above and quote the current behavior rather than relying on training
memory. If documentation cannot be reached at run time, say so and avoid claiming
current-year verification (see `references/version-verification.md`).
