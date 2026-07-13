# Example: shared wizard state

**Problem.** A multi-step wizard (onboarding, checkout details) spans several
components: a step indicator, the current step's form, a summary, and navigation
buttons. They all read and write the same in-progress data within one session.

**Category.** Shared client state (category 2), provided via context so each
mounted wizard gets its own instance and it stays SSR-safe. Owner: the wizard
subtree. Lifetime: the session, until submit or abandon.

## Shape

```
src/lib/features/wizard/
├── wizard-state.svelte.ts    # factory + context provide/consume
├── Wizard.svelte             # provides the context, renders the current step
├── StepIndicator.svelte      # reads step
├── StepNav.svelte            # calls next/prev
└── steps/
    ├── AccountStep.svelte
    └── PlanStep.svelte
```

## State factory with context

```ts
// wizard-state.svelte.ts
import { setContext, getContext } from 'svelte';

const KEY = Symbol('wizard');

export function createWizard(totalSteps) {
  let step = $state(0);
  let data = $state({});

  return {
    get step() { return step; },
    get data() { return data; },
    get isLast() { return step === totalSteps - 1; },
    next() { if (step < totalSteps - 1) step += 1; },
    prev() { if (step > 0) step -= 1; },
    setField(key, value) { data[key] = value; }
  };
}

export function provideWizard(totalSteps) {
  const wizard = createWizard(totalSteps);
  setContext(KEY, wizard);
  return wizard;
}

export function useWizard() {
  return getContext(KEY);
}
```

```svelte
<!-- Wizard.svelte -->
<script>
  import { provideWizard } from './wizard-state.svelte';
  provideWizard(2);
</script>
<StepIndicator />
<slot />
<StepNav />
```

```svelte
<!-- StepNav.svelte -->
<script>
  import { useWizard } from './wizard-state.svelte';
  const wizard = useWizard();
</script>
<button onclick={wizard.prev} disabled={wizard.step === 0}>Back</button>
<button onclick={wizard.next}>{wizard.isLast ? 'Review' : 'Next'}</button>
```

## Why this shape

- Context, not a module singleton, so two wizards on the same client (or two
  concurrent SSR requests) do not share state. This is the SSR-safe way to share.
- Reads are exposed as getters over `$state`, so consumers are reactive but cannot
  bypass the methods.
- `isLast` is `$derived`-style logic expressed as a getter over `step`; no effect
  mirrors it.
- The wizard holds only in-progress client data. Final submission goes through a
  form action or endpoint, where the server validates and persists.

## When not to use context here

If the wizard state must survive a reload (the user might refresh mid-flow), add
persistence (see `../references/persistence-and-sync.md`) or move the step to the
URL so it is bookmarkable. If only one component used this data, it would be local
state, not a shared context.
