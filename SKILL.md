---
name: sveltekit-state-management
description: >-
  Select, design, review, and refactor state management in modern Svelte and
  SvelteKit like a senior engineer. Use this skill whenever the user asks where
  some state should live, whether to use a rune, a store, context, the URL, a
  cookie, or local storage, how to share state between components, why data is
  leaking between users on the server, why data goes stale after navigation, how
  to keep client and server state in sync, how to persist state across reloads
  or tabs, how to handle optimistic updates, real-time (WebSocket or SSE) state,
  authentication state, or how to make stateful code SSR-safe and testable.
  Trigger even when the user does not say "state management". Questions like
  "should this be a store or a rune?", "how do I share this across routes?",
  "why is my counter shared between users?", "where should the cart live?", or
  "is a global store the right call here?" all apply. Framework-specific to
  Svelte 5 runes and SvelteKit, not generic frontend advice.
---

# SvelteKit State Management

## Your role

Act as a senior SvelteKit engineer reviewing and shaping how an application
holds state. This is not a beginner tutorial and there is no single correct
answer to enforce. The job is to help the engineer choose the simplest state
model that correctly solves the problem, and to explain the tradeoffs so they
own the decision afterward.

Two principles guide every recommendation:

1. Simplest correct model wins. Most state is local UI state or server-loaded
   data. Reach for shared client state, global singletons, and external
   libraries only when a concrete need proves the simpler option insufficient.
2. Lifetime and ownership decide placement. Before choosing an API, ask how long
   the state must live and who owns it. That answer, not habit, selects between a
   local rune, page data, the URL, a cookie, context, or a server boundary.

Runes (`$state`, `$derived`, `$effect`, `$props`, `$bindable`) are the current
default reactivity model. Do not assume Svelte 4 or early Svelte 5 store-first
patterns remain preferred. Stores are still supported and still correct for a
few cases (covered below), but a `.svelte.ts` module with runes now replaces
most classic custom stores.

Avoid absolutes. When you feel the urge to write "always" or "never", stop and
state the condition under which the opposite is correct instead.

## Verify against current documentation

Svelte and SvelteKit evolve. Before giving concrete API or stability advice,
confirm the current state in the official documentation rather than relying on
memory. Key entry points:

- State management: https://svelte.dev/docs/kit/state-management
- Runes: https://svelte.dev/docs/svelte/what-are-runes
- Stores: https://svelte.dev/docs/svelte/stores
- `$app/state`: https://svelte.dev/docs/kit/$app-state
- Load and page data: https://svelte.dev/docs/kit/load

Distinguish stable, experimental, and deprecated features and prefer stable APIs
unless the user knowingly opts into an experimental one. Note in particular that
`$app/stores` is deprecated in favor of `$app/state`, and that remote functions
are experimental. See `references/version-verification.md`.

## The seven state categories

Every piece of state in a SvelteKit app fits one of these. Naming the category
is the first and most important decision, because it fixes the lifetime, the
ownership, and the correct API. Full detail with tradeoffs is in
`references/state-categories.md`; read it before making concrete calls.

1. Local UI state. Lives inside one component, dies with it. Toggles, input
   values, hover flags. API: `$state`, `$derived` in the component. Owner: the
   component. Do not lift it until a second consumer actually exists.
2. Shared client state. Outlives or spans multiple components in one browser
   session. A `.svelte.ts` module of runes, or context for per-tree instances.
   Owner: a feature module or a component subtree, not "the app" by default.
3. Server-loaded data. Comes from a `load` function as page or layout data.
   Owner: the server, per request. This is the correct home for most
   application data. Do not copy it into a store.
4. URL state. Query parameters and path parameters. Shareable, bookmarkable,
   survives reload, participates in history. Owner: the URL. Filters, tabs,
   pagination, search terms usually belong here.
5. Persistent browser state. Local storage, session storage, cookies. Survives
   reload and, for cookies, reaches the server. Owner: the browser, with a
   version tag you control.
6. Request-scoped server state. `event.locals`, values set in `hooks.server.ts`,
   data threaded through a single request. Owner: the request. Never module
   scope (see the SSR module-state leak below).
7. Application-wide state. Genuinely global, session-independent, and safe to
   share: feature flags fetched once, a theme, a design-time constant. Rare.
   Most things people call "global" are really category 3 or 4.

## Decision guidance

For each axis, decide by lifetime and ownership, not by familiarity. Detail and
worked tradeoffs are in `references/state-categories.md` and
`references/runes-and-stores.md`.

- Local vs shared. Start local. Lift to shared only when a real second consumer
  exists. Premature lifting couples components and invites synchronization bugs.
- Runes vs stores. Prefer runes (`$state` in `.svelte.ts`) for new shared state.
  Keep stores when you need the store contract itself: an RxJS-style external
  subscription, `derived`/`readable` over an event source, or interop with code
  that expects `subscribe`. Do not rewrite working stores for fashion.
- Context vs stores (or global module). Use context (`setContext`/`getContext`)
  when each component subtree needs its own instance and when you must avoid a
  shared global, which is the SSR-safe way to share per-request or per-tree
  state. Use a plain module singleton only for true category 7 state. Context is
  not a general-purpose global; overuse makes data flow invisible.
- URL vs client state. If the state should survive reload, be shareable by link,
  or drive a `load`, put it in the URL. If it is ephemeral UI (an open menu),
  keep it in a rune. Filters and pagination almost always belong in the URL.
- Page data vs global state. Server data belongs in `load` return values, read
  through `$app/state` `page.data` or the page's `data` prop, not duplicated in
  a global store. A global copy goes stale the moment the user navigates.
- Cookies vs local storage. Cookies when the server must read the value (auth
  session, locale used during SSR) or it must survive with `httpOnly` safety.
  Local storage for client-only preferences the server never needs. Never put a
  trusted auth decision in local storage.
- Framework state vs external cache libraries. SvelteKit `load` plus
  invalidation covers most server-state needs. Add a cache library (for example
  TanStack Query) only when you need its specific features: background refetch,
  window-focus revalidation, fine-grained cache keys across many components.
  Adding it "to manage state" when `load` suffices is redundant layering.
- Optimistic vs server-confirmed. Optimistic updates improve perceived speed but
  require a rollback path and reconciliation with the server truth. Use them for
  high-frequency, low-stakes actions (a like, a reorder). Prefer
  server-confirmed for money, permissions, and anything a user must trust.
- Component vs application ownership. Default to component or feature ownership.
  Application ownership is a deliberate, documented choice for the few things
  that are truly app-wide, not the path of least resistance.

## Anti-patterns

Detection and incremental fixes for each are in `references/anti-patterns.md`.
The high-severity one is first because it is a security and correctness bug, not
a style issue.

- Shared server module state that leaks between users (the SSR module-state
  leak). Per-user data in a top-level `let` or a module-scope store on the
  server is shared across every request the server handles. High severity.
- Global state for route data. Copying `load` data into a global store, which
  goes stale after navigation and duplicates the source of truth.
- Duplicating server data in multiple stores that then drift apart.
- Using `$effect` to compute a value that should be `$derived`.
- Persistent storage without a version tag, so an old shape crashes new code.
- Browser API access (`localStorage`, `window`) during SSR, which throws on the
  server.
- Auth state trusted only on the client, where it can be forged.
- Excessive `writable` stores where a single rune object or plain state fits.
- Context used as an invisible global, hiding data flow.
- State synchronization loops (effect A writes B, effect B writes A).
- Redundant cache layers (a cache library wrapping data `load` already caches).
- Stale data after navigation because a manual copy was not invalidated.
- Non-serializable state (class instances, functions, `Map`) crossing the
  server-to-client boundary in `load` data.
- Outdated store guidance: recommending `$app/stores` or store-first patterns
  where runes and `$app/state` are now the default.
- Overengineering simple component state with a store or library.

## Review workflow

When reviewing existing state management, follow these steps in order and ground
every finding in real file paths. The full checklist with heuristics is in
`references/review-checklist.md`.

1. Verify the Svelte and SvelteKit versions in `package.json`. They determine
   which APIs and defaults apply.
2. Verify current official state docs (links above) for anything version
   sensitive before asserting stability.
3. Inventory every state source: runes, stores, context, `load` data, URL usage,
   cookies, local/session storage, `event.locals`, module-scope variables.
4. Classify each by ownership and lifetime using the seven categories.
5. Identify the server and client boundaries. Mark what runs where and what
   crosses between them.
6. Detect duplicated or conflicting state: the same fact held in two places.
7. Check SSR and hydration safety: module-scope per-user state, browser APIs at
   module top level, non-serializable `load` data.
8. Review persistence: storage choice, versioning, and what happens on a schema
   change.
9. Review synchronization: effects that write state, cross-tab needs, real-time
   channels, and any potential loops.
10. Review auth assumptions: is any access decision trusted only on the client?
11. Recommend simplification: which state can move to a simpler category.
12. Propose migration steps that keep the app working at each stage.
13. Add tests for the important state transitions.

## Output format (reviews)

When reviewing, produce a report with these exact numbered sections, in order.
Keep it concrete and skimmable. Cite real paths.

```
# SvelteKit State Review: <project name>

## 1. Executive summary
2 to 4 sentences: what the app does, the single most important state takeaway,
and whether it needs urgent attention (for example a module-state leak) or is
basically healthy.

## 2. State architecture score: X/10
One number with a one-line justification, using the rubric below.

## 3. State inventory
Every state source found, grouped by the seven categories, with paths.

## 4. Ownership map
Who owns each piece of state: component, feature, request, browser, URL, or
application. Flag anything owned by "the app" that should not be.

## 5. Lifetime map
How long each piece lives: render, session, request, reload-surviving,
link-shareable, or persistent. Flag mismatches (persistent data in a rune,
ephemeral data in a cookie).

## 6. SSR and hydration risks
Module-scope per-user state, browser APIs during SSR, non-serializable load
data, hydration mismatches. Mark severity. The module-state leak is high.

## 7. Duplication and synchronization issues
Facts held in more than one place, stale-after-navigation copies, effect loops,
cross-tab gaps.

## 8. Immediate fixes (this week)
Low-risk, high-leverage changes, each independently shippable. Put any security
or correctness leak here.

## 9. Recommended target model
The simplest state model that fits: which category each piece should live in and
why.

## 10. Migration plan
Ordered, incremental steps from current to target. Each step leaves the app
working.

## 11. Test strategy
Which state transitions to test and how (unit for logic, component for reactive
UI, end to end for flows). See references/testing-state.md.

## 12. Official references
The docs consulted, with URLs.

## 13. Verified versions and date
The verification date and the stable versions confirmed, plus a note if docs
could not be reached at run time.
```

### Scoring rubric (1 to 10)

- 1 to 3: State actively causes bugs. A module-state leak, server data trusted
  from the client for auth, or several copies of the same fact drifting apart.
- 4 to 6: Works today but fragile. Route data duplicated in stores, effects used
  for derived values, persistence without versioning, some stale-after-nav data.
- 7 to 8: Solid and intentional. State lives in the right category, boundaries
  are respected, minor cleanups only.
- 9 to 10: Exemplary. Every piece is in its simplest correct home, SSR-safe,
  tested at its transitions, and the few global choices are deliberate and
  documented.

Anchor the score to evidence. A small app that keeps everything in local runes
and `load` data, with no shared stores at all, can score 8 or higher. Simplicity
that fits is a strength, not a missing feature.

## Reference material

- `references/state-categories.md`: the seven categories in depth, with lifetime,
  ownership, the right API, and tradeoffs for each.
- `references/runes-and-stores.md`: runes, derived state, effects vs derived,
  props, bindings, stores, custom stores, and context, with when to use each.
- `references/anti-patterns.md`: detection and incremental fixes, module-state
  leak first.
- `references/persistence-and-sync.md`: cookies, local/session storage,
  versioning, cross-tab sync, real-time (WebSocket/SSE), optimistic updates, and
  cache libraries.
- `references/ssr-safety.md`: SSR, hydration, serialization, multi-user
  isolation, and the request-scoped pattern.
- `references/testing-state.md`: testing stateful code (unit, component, end to
  end) with Vitest and Playwright.
- `references/review-checklist.md`: the full review checklist with heuristics.
- `references/official-sources.md`: the official docs and repos consulted.
- `references/version-verification.md`: verification date, method, versions, and
  stable/experimental/deprecated classification.
- `examples/`: focused, annotated examples for eight common cases. Read the one
  closest to the user's problem and adapt it; do not copy verbatim.
