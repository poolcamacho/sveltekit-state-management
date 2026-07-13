# Version verification

## Verification date and method

- Verification date: 2026-07-13 (current calendar year 2026).
- Method: official documentation and repositories checked via web (svelte.dev
  docs and the sveltejs/svelte and sveltejs/kit repositories and changelogs). See
  `references/official-sources.md` for the exact URLs.

If documentation access is unavailable at run time, the agent must re-verify and
must not claim current-year verification. In that case, state that versions could
not be confirmed and recommend the user check `package.json` and the official docs.

## Verified stable versions

- Svelte: 5.56.4
- SvelteKit (`@sveltejs/kit`): 2.69.2
- Build tooling: Vite 8 (SvelteKit requires `vite ^8.0.12`).

Do not hardcode these numbers inside prose recommendations. In prose, prefer
phrasing like "in the current stable SvelteKit". The concrete pins live here so the
skill stays accurate as versions advance.

## Stable, experimental, and deprecated (state-relevant)

### Stable
- Runes: `$state`, `$derived`, `$effect`, `$props`, `$bindable`, usable in
  `.svelte` and `.svelte.ts`/`.svelte.js` files. The default reactivity model.
- `$app/state`: `page`, `navigating`, `updated` as reactive objects (requires
  runes). The successor to `$app/stores`.
- `load` functions and page/layout data, form actions, hooks (server, client,
  universal), `event.locals`, the `cookies` API, `$env` modules, adapters.
- Svelte stores (`writable`, `readable`, `derived`): still supported, no longer
  the default for in-app shared state.
- Context: `setContext`/`getContext`.
- Testing: Vitest (unit and component), Vitest browser mode via
  vitest-browser-svelte for component tests, Playwright for end to end.

### Experimental (opt-in via config flag; available since SvelteKit 2.27)
- Remote functions (`query`, `form`, `command`, `prerender` in `.remote.ts`).
  Relevant to state because they change how client and server data flow. Never
  recommend as a production default; present as an experimental alternative and
  label it clearly.

### Deprecated
- `$app/stores` (the `$page`, `$navigating`, `$updated` stores). Superseded by
  `$app/state`, which requires runes, and subject to removal in a future major.
  Do not recommend `$app/stores` for new code; migrate reads to `$app/state`.

## Guidance for the agent

- Treat runes and `$app/state` as the default when advising on new state code.
- Do not present Svelte 4 or early Svelte 5 store-first patterns as current best
  practice.
- Keep stores where the store contract earns its place (external subscriptions,
  event-source wrappers, interop), not as the default shared-state tool.
- When a user is on an older version, tailor advice to what that version supports
  and note the upgrade path rather than assuming the latest APIs.
