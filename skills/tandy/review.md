# tandy review checklist

Review a diff against tandy (`SKILL.md` in this folder) and the task it implements. Read the code the diff touches, not only the diff: callers of every changed function, the functions it calls, and the tests.

## Rules for findings

- **No concrete case, no finding.** Each finding names the input or situation and the wrong result it leads to, or the specific tandy rule it breaks and where.
- Re-read the lines before reporting: confirm the caller exists, the value can really be empty, the code is really unused.
- Propose the smallest fix. Prefer fixes that delete code.
- No style preferences the codebase and tandy don't state. No "consider…", no vague worries.

## Check, in this order

1. **Correct:** does it do what the task's "Done when" says? Edge cases (empty, zero, last item, time zones)? Callers broken by a changed signature or behavior?
2. **Safe:** validation at trust boundaries, swallowed errors, data loss, weakened auth or permission checks, secrets or PII in the diff.
3. **Data and performance (always check):**
   - Every new or changed loop that touches data: is there a query inside it (**N+1**)? This is always a must-fix.
   - Every new query or list: is it bounded (paginated or limited)?
   - Migrations: safe on existing data?
4. **Tested:** bug fixes have a test that fails without the fix. Non-trivial logic is tested. DB behavior is tested against a real DB, not mocks. No tests weakened or skipped.
5. **Types:** no `any`, no silencing casts, no `@ts-ignore`. Unions and enums branch with `switch` + `assertNever` and explicit no-op cases, not `if` chains.
6. **Scope:** no changes outside the task's path, no unrequested backwards compatibility, no new dependency that wasn't approved in the plan.
7. **Lean:** reuse instead of re-implementing (name the existing path), no single-use abstractions, no dead or commented-out code, no debug leftovers, no magic values.
8. **Conventions:** comments, file placement and naming follow tandy and the repo.

## Output

Number findings across groups so they can be referenced ("fix 2 and 4"):

- **Must fix:** bugs, security, data loss, N+1 or unbounded queries, broken "Done when" criteria.
- **Should fix:** missing tests, tandy rule violations, scope creep, duplication.
- **Nice to have:** small simplifications.

Each finding: `file:line`, one line on the problem with its concrete case, one line on the fix.

When reviewing inside dandy, end with exactly one of:

- `VERDICT: APPROVE` (when there are no Must fix or Should fix items)
- `VERDICT: CHANGES_REQUESTED`, followed by the numbered Must fix and Should fix items
