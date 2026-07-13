# State review checklist

The full checklist behind the review workflow in SKILL.md. Each item has a
detection heuristic and a remediation. Ground every finding in a real file path.
Run the items in workflow order; stop to flag anything high severity (a leak or a
client-only auth decision) immediately rather than saving it for the report.

## 1. Version and docs baseline
- **Detect.** Read `package.json` for `svelte` and `@sveltejs/kit` versions.
  Confirm whether the project uses runes (Svelte 5) or legacy stores (Svelte 4).
- **Why.** The right advice depends on the version. `$app/state` requires runes;
  `$app/stores` is deprecated.
- **Action.** Note the versions and verify anything version-sensitive against the
  official docs before asserting stability. Record the result in section 13.

## 2. State inventory
- **Detect.** Grep for `$state`, `$derived`, `$effect`, `writable`, `readable`,
  `derived(`, `setContext`, `getContext`, `localStorage`, `sessionStorage`,
  `cookies`, `event.locals`, and top-level `let`/`export const` in server modules.
  List every hit with its path.
- **Action.** Group each into one of the seven categories. A source you cannot
  categorize is usually misplaced.

## 3. Ownership and lifetime classification
- **Detect.** For each source, ask who owns it and how long it lives.
- **Why.** Mismatches are the root of most bugs: persistent data in a rune (lost on
  reload), server data in a global store (stale), a filter in a rune (not
  shareable).
- **Action.** Build the ownership map (section 4) and lifetime map (section 5).
  Flag every mismatch.

## 4. Server and client boundary
- **Detect.** Identify what runs on the server (`.server.ts`, hooks, `load` in
  server files) vs the client. Trace what crosses via `load` return values.
- **Action.** Confirm per-user data crosses only through `load`/`locals`, never a
  shared server module. Confirm no server-only module is imported into client
  code.

## 5. Duplication and conflicting state
- **Detect.** The same fact in two places: `load` data also copied into a store, a
  user object in several stores, a value mirrored by an effect.
- **Why.** Copies drift and disagree.
- **Action.** Pick one owner, derive the rest with `$derived`, delete the copies.

## 6. SSR and hydration safety
- **Detect.** Module-scope per-user state on the server (high severity); browser
  APIs (`window`, `localStorage`) at module top level or in component init;
  non-serializable values returned from `load`; values read from storage used in
  initial render (hydration mismatch).
- **Action.** Move per-user state to `event.locals`; move browser access into
  effects or `browser` guards; return plain serializable data from `load`. See
  `references/ssr-safety.md`.

## 7. Persistence review
- **Detect.** Reads from `localStorage`/cookies with no version tag and no
  fallback; large payloads; secrets in non-`httpOnly` cookies or the URL; trusted
  auth in local storage.
- **Action.** Add a `{ version, data }` envelope with migrate-or-discard; move
  secrets to `httpOnly` cookies; move trusted auth to server verification. See
  `references/persistence-and-sync.md`.

## 8. Synchronization review
- **Detect.** Effects that both read and write overlapping state (loop risk);
  cross-tab needs not handled; real-time transports opened at module top level or
  without teardown.
- **Action.** Establish one direction of truth with `$derived`; add
  `BroadcastChannel`/`storage` handling where cross-tab sync is needed; move
  socket setup into effects with cleanup.

## 9. Auth assumptions
- **Detect.** Access decisions made from client state alone; protected endpoints
  or `load` that do not re-check `event.locals.user`; routes "protected" only by
  hiding UI.
- **Why.** Client state can be forged. High severity.
- **Action.** Verify auth on the server in hooks and in every protected `load`/
  action. Client auth state is for UI convenience only.

## 10. Simplification opportunities
- **Detect.** Stores or context or libraries wrapping state that a local rune or
  `load` would handle; many small stores that could be one object; a cache library
  shadowing `load`.
- **Action.** Recommend moving state to the simplest category that fits. Note where
  the current approach is overengineered and where it is under-built.

## 11. Framework-currency
- **Detect.** `import ... from '$app/stores'`; store-first patterns in a runes
  project; `$effect` used to derive values; custom stores that could be `.svelte.ts`
  factories.
- **Action.** Recommend `$app/state`, `$derived`, and rune factories, without
  forcing rewrites of working store code that has a real reason to remain.

## 12. Test coverage of transitions
- **Detect.** Whether the important state transitions have tests: auth, URL
  persistence, optimistic rollback, multi-user isolation, versioned persistence.
- **Action.** Recommend the specific tests from `references/testing-state.md`,
  prioritizing isolation and auth.

## Scoring guidance

Map findings to the 1 to 10 rubric in SKILL.md. Weight correctness and security
(the module-state leak, client-only auth, drifting duplicates) far above style.
An app with clean local state and `load` data and no shared stores at all is not
under-built; it can score 8 or higher. Reserve high marks for state that is in its
simplest correct category, SSR-safe, and tested at its transitions.
