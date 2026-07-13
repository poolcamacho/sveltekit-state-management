# SvelteKit State Management Skill

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Install with skills](https://img.shields.io/badge/install-npx%20skills%20add-black)](https://skills.sh)
[![Built for Claude Code](https://img.shields.io/badge/built%20for-Claude%20Code-6f42c1)](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview)

A reusable [Agent Skill](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview)
that turns Claude into a senior SvelteKit engineer for state management. It helps
select, design, review, and refactor how a Svelte and SvelteKit application holds
state, and produces a concrete, tradeoff-aware report and migration plan.

It favors the simplest state model that correctly solves the problem, and it treats
runes (`$state`, `$derived`, `$effect`, `$props`, `$bindable`) as the current
default rather than assuming Svelte 4 or early Svelte 5 store-first patterns. It is
opinionated where correctness demands it (per-user server isolation, client-only
auth, SSR safety) and non-dogmatic everywhere else.

## What it does

- Classify state into seven categories: local UI state, shared client state,
  server-loaded data, URL state, persistent browser state, request-scoped server
  state, and application-wide state. Naming the category fixes the lifetime, the
  owner, and the correct API.
- Decide by lifetime and ownership: local vs shared, runes vs stores, context vs
  global, URL vs client, page data vs global store, cookies vs local storage,
  framework state vs cache library, optimistic vs server-confirmed.
- Detect anti-patterns with fixes, led by the SSR module-state leak (per-user
  data in shared server scope), plus global state for route data, effects used for
  derived values, unversioned persistence, browser APIs during SSR, client-only
  auth, redundant cache layers, and more.
- Review an existing app and emit a structured report: executive summary, an
  architecture score (1 to 10), a state inventory, ownership and lifetime maps, SSR
  and hydration risks, duplication and synchronization issues, immediate fixes, a
  recommended target model, a migration plan, a test strategy, references, and the
  verified versions and date.

## When it triggers

Asking where some state should live, whether to use a rune, a store, context, the
URL, a cookie, or local storage, how to share state across components or routes,
why data leaks between users on the server, why data goes stale after navigation,
how to persist across reloads or tabs, how to handle optimistic updates or
real-time (WebSocket or SSE) state, how to manage authentication state, or how to
make stateful code SSR-safe and testable. It triggers even when the phrase "state
management" is not used, on questions like "should this be a store or a rune?" or
"why is my counter shared between users?".

## Contents

```
sveltekit-state-management/
├── SKILL.md                          # entry point: role, categories, workflow, output format
├── README.md                         # this file
├── references/
│   ├── state-categories.md           # the seven categories: lifetime, ownership, API
│   ├── runes-and-stores.md           # runes, derived vs effect, props, bindings, stores, context
│   ├── anti-patterns.md              # detection and fixes, module-state leak first
│   ├── persistence-and-sync.md       # cookies, storage, cross-tab, real-time, optimistic, cache
│   ├── ssr-safety.md                 # SSR, hydration, serialization, multi-user isolation
│   ├── testing-state.md              # unit, component, and end-to-end testing of state
│   ├── review-checklist.md           # the full review checklist with heuristics
│   ├── official-sources.md           # official docs and repos consulted
│   └── version-verification.md       # verification date, method, versions, stability
└── examples/
    ├── local-component-form-state.md
    ├── shared-wizard-state.md
    ├── url-driven-filters.md
    ├── authenticated-user-state.md
    ├── shopping-cart.md
    ├── real-time-dashboard.md
    ├── multi-tenant-application.md
    └── ssr-safe-request-state.md
```

Claude loads `SKILL.md` when the skill triggers, then pulls in the specific
reference or example it needs, so the deep material stays out of context until it
is relevant.

## Design principles

1. Simplest correct model wins. Most state is local UI state or server-loaded
   data. Shared stores, global singletons, and external libraries appear only when
   a concrete need proves the simpler option insufficient.
2. Lifetime and ownership decide placement. The category, not habit, selects the
   API.
3. Correctness over style. Per-user server isolation, SSR safety, and server-side
   auth enforcement are treated as non-negotiable; formatting preferences are not.
4. Framework-current. Runes and `$app/state` are the default; `$app/stores` is
   flagged as deprecated and remote functions as experimental.
5. Evidence-based reviews. Findings cite real paths and are ordered by impact, and
   the score is anchored to a rubric.

## Official documentation

- State management: https://svelte.dev/docs/kit/state-management
- Runes: https://svelte.dev/docs/svelte/what-are-runes
- Stores: https://svelte.dev/docs/svelte/stores
- `$app/state`: https://svelte.dev/docs/kit/$app-state
- Load and page data: https://svelte.dev/docs/kit/load

## Installation

Install with the [skills](https://skills.sh) CLI, which works across Claude Code
and other agents:

```bash
npx skills add poolcamacho/sveltekit-state-management
```

The command clones this repo, detects the skill from `SKILL.md`, and lets you
choose which agents to install it into (Claude Code and the universal target are
selected by default).

You can also install it manually by copying the `sveltekit-state-management`
folder into your agent's skills directory (for Claude Code, `.claude/skills/`).

## Usage

Once installed, ask naturally, for example "should the cart be a store or a rune?",
"review how my SvelteKit app manages state", "why is my user data leaking between
requests?", or "where should these filters live?". The agent consults this skill
and responds as a senior engineer, using the report format for reviews.

## License

Released under the [MIT License](LICENSE). Free to use, adapt, and redistribute
with attribution.
