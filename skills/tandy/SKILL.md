---
name: tandy
description: >-
  Andy's implementation taste: how code should be scoped, structured, typed,
  tested and commented in Andy's repos. Use on any coding task in Andy's repos
  (writing, fixing, refactoring, reviewing, planning), when deciding what a
  change should and shouldn't touch, when dandy gathers context, grills, plans,
  implements or reviews, or when the user says "tandy" or /tandy. Pair with
  review.md in this folder when reviewing a diff.
---

# tandy

## A note from Andy

I like simple systems and code that feels obvious. Don't preserve complexity just because it already exists, and don't add machinery because it looks impressive. Understand the real constraint, then build the smallest thing that makes the correct behavior unsurprising. Measure twice, cut once, and YAGNI. Finish everything the task needs, and nothing it doesn't.

These are strong defaults. The repo's own `AGENTS.md` / `CLAUDE.md` and my explicit instructions override them. **If a rule here fights the task, say so clearly and wait for my sign-off before breaking it. Never break one silently.**

## Scope

- Simplify or delete existing code only when it is **in the path of the task**, and say so in the plan or PR.
- Anything worth fixing outside that path becomes a one-line follow-up note. Don't touch it.
- No "while I was here" changes: no unrelated formatting, renames or refactors.
- No backwards compatibility nobody asked for: no old code paths kept alive, no deprecation shims, no `v2` copies. Change the code.

## Before writing code

Take the first option that fully works:

1. **Does it need to exist?** If not, skip it and name what you skipped in one line.
2. **Already in this codebase?** Reuse the helper, component or pattern the way the surrounding code does.
3. **An installed dependency or the platform/stdlib?** Use it.
4. **Can you extend something existing** (a variant, an option, a case) instead of creating a parallel copy? Extend it.
5. **Otherwise, write new code:** the least that works, in the structure the codebase already has.

Never add a dependency for a few lines. Any new dependency must be called out in the plan for approval.

## Errors

- Validate at trust boundaries: user input, API requests and responses, webhooks, third-party data. Inside the app, trust your own types.
- Never swallow an error. Handle it in a way that has meaning (retry, fallback, a message the user sees) or let it propagate. A catch that only logs and continues is a bug.
- Errors the user sees are actionable ("Couldn't save, try again"), never raw stack traces or internal messages.
- No defensive code for impossible states. If it truly matters, assert and fail loudly.

## Types

- Let types be inferred. Annotate only at public boundaries: exported functions, API contracts, component props.
- No `any`. Use `unknown` and narrow it at the trust boundary.
- No casts (`as Foo`) to silence the compiler. Fix the type model, or explain in one line why the cast is safe.
- Make impossible states unrepresentable: discriminated unions instead of several optional fields or booleans.
- One source of truth for each type. Derive types from the schema, DB or validator instead of redefining a shape by hand.
- **Branch on a union or enum with `switch` + `assertNever`, never an `if` chain.** Every value gets its own `case`. A value that should do nothing gets an explicit case, so adding a new value fails the build until someone decides what it does:

  ```ts
  switch (status) {
    case "active":
      activate();
      break;
    case "paused":
      pause();
      break;
    case "archived":
      // no-op: archived items are read-only
      break;
    default:
      assertNever(status);
  }
  ```

  Use the repo's existing `assertNever`. If the repo has none, add one to the shared utils. For plain value lookups, use `Record<MyEnum, T>`, which gives the same build-time check.

## State (React)

- Derive, don't sync. If a value can be computed from props or state, compute it in render.
- `useEffect` is only for syncing with something outside React (subscriptions, the DOM, timers). Logic triggered by a user action goes in the event handler.
- Server data lives in the repo's data-fetching layer, not copied into `useState` or a global store.
- Keep state as local as possible. Lift it only when two components really share it. Global stores are for truly app-wide state only.
- Every async view handles loading, empty and error states explicitly.

## Data and performance

- **No N+1 queries, at all costs.** Never query inside a loop. Load related data in batches: joins, `IN (...)`, a dataloader, or the ORM's include/preload.
- No unbounded queries or lists. Always paginate or limit.
- Migrations must be safe on existing data. No destructive change without a plan for the rows already there.

## Tests

- Default to **integration tests against a real database**. Don't mock DB interactions.
- Use unit tests with mocked infrastructure only for edge cases that are hard to reproduce for real, like rare error paths.
- Test behavior through public interfaces, not internals. Never mock code from the same repo.
- Must have a test: every bug fix (a test that fails before the fix), and new non-trivial logic (branches, parsers, money, permissions, data writes). Trivial wiring and pure layout need none.
- One good test beats coverage padding. No snapshot tests by default, and no tests that only check that a mock was called.
- Follow the repo's existing test framework, style and location.

## Comments

- No comment by default. Write one only for what the code can't show: a constraint, a quirk of an external system, an invariant, a non-obvious why. One line where possible, four at most.
- Never restate the code, narrate steps, record history, or explain the fix you just made. That belongs in the PR.
- No TODOs unless they link to an issue in the tracker: `// TODO(<issue-id>): ...`.
- Docstrings only on exported APIs whose signature doesn't explain itself.

## Files and naming

- Follow the repo's conventions for folders, file names and casing. Copy the closest sibling feature. Never introduce a new convention in passing.
- Put code next to where it's used. Move a helper to shared code only when a second feature needs it. No catch-all `helpers.ts`.
- One concept per file. Split by responsibility, never by line count.
- Name things after what they mean in the domain (`invoiceDueDate`), not their type (`dateObj`, `dataList`). No abbreviations beyond well-known ones (`id`, `url`).
- No barrel files (`index.ts` re-exports) unless the repo already uses them.

## Never cut

Even under YAGNI and minimal diffs, always keep:

- Validation at trust boundaries.
- Error handling that prevents data loss or silent failure.
- Security and permission checks. Never weaken one to make something work.
- Loading, empty and error states for async UI.
- Accessibility basics: semantic elements, labels, keyboard access.
- Safe migrations.
- No N+1 queries, and no unbounded queries or lists.
- Anything Andy explicitly asked for. You may say it seems unnecessary, but build it.

## Never do

1. Change unrelated files: formatting, renames, "while I was here" refactors.
2. Over-abstract: a factory, interface or generic for a single use.
3. Add backwards compatibility nobody asked for: old paths, shims, `v2` copies.
4. Say it's done without running tests, lint and typecheck, or claim they pass when they don't.
5. Weaken a test, or add `skip`, `any`, `@ts-ignore` or `eslint-disable` to get to green.
6. Leave `console.log`, debug code or commented-out code behind.
7. Hardcode magic values that belong in a constant, config or the DB.
8. Write long final reports or PR descriptions. Say what changed, how it was verified, and what's left. Nothing more.

## Using tandy while planning

When gathering context, grilling or planning, use tandy to decide what to change and what to leave alone:

- Mark what can be reused, what must stay untouched, and what in the path of the task can be simplified or deleted.
- Don't ask questions tandy already answers. Ask only where tandy says nothing, or where a rule conflicts with the task.
- Put deliberate omissions and out-of-path cleanups in the plan's "Out of scope" section.
