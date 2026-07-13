# Example: local component form state

**Problem.** A single form (a contact form, a settings panel) needs field values,
validation, and a submit state. No other component needs this data.

**Category.** Local UI state (category 1). Owner: the component. Lifetime: the
render. Do not lift it to a store or context; nothing else reads it.

## Shape

```
src/routes/contact/
├── +page.svelte          # the form, local state only
└── +page.server.ts       # form action: validates and persists on the server
```

## Component

```svelte
<script>
  let { form } = $props();          // action result from the server

  let name = $state('');
  let email = $state('');
  let submitting = $state(false);

  // Derived validation, not an effect.
  let emailValid = $derived(/^[^@]+@[^@]+\.[^@]+$/.test(email));
  let canSubmit = $derived(name.trim().length > 0 && emailValid && !submitting);
</script>

<form method="POST" use:enhance={() => {
  submitting = true;
  return async ({ update }) => { await update(); submitting = false; };
}}>
  <input name="name" bind:value={name} />
  <input name="email" bind:value={email} type="email" />
  {#if !emailValid && email}<p>Enter a valid email.</p>{/if}
  <button disabled={!canSubmit}>Send</button>
  {#if form?.error}<p>{form.error}</p>{/if}
</form>
```

## Why this shape

- `$state` for field values, `$derived` for validation. Validation is a pure
  function of the fields, so it is derived, never an effect.
- `bind:value` is the idiomatic way to read live input; the two-way binding is
  appropriate inside one form.
- The server action (not shown in full) re-validates. Client validation is UX;
  the server is the source of truth for what gets saved.
- No store, no context, no library. Adding any of those here is the
  overengineering anti-pattern.

## When to graduate

If a second component needs these values (a live preview beside the form, a
multi-step flow), lift to shared client state or context, not before. See
`shared-wizard-state.md`.
