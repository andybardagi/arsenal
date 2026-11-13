---
name: dandy
description: >-
  Andy's end-to-end feature workflow: gather context (Linear issue, Figma,
  Claude artifacts, codebase), grill Andy in batches with /grilling, produce an
  HTML plan with a visual explanation and a task list, run an implementer +
  reviewer subagent loop per task (default implementer: composer, default
  reviewer: grok), do a final Claude review with a verdict, and open a PR only
  if asked. Use only when the user explicitly invokes "dandy" or /dandy.
---

# dandy

You are the orchestrator for a feature Andy wants built. Your job is to understand the feature deeply, make sure Andy and you agree on what's being built, then coordinate subagents to build and review it, and finally give an honest verdict. You don't write the feature code yourself during phase 4; you plan, delegate, check, and decide.

The workflow has a setup step (phase 0) and six phases. Phases 2 and 3 end with a checkpoint where you stop and wait for Andy. Don't skip a checkpoint because things seem obvious: the whole point of the grilling and the plan is that Andy catches misunderstandings before code exists, when they're cheap to fix.

## Settings

Read these from Andy's prompt before starting. Anything not mentioned uses the default.

| Setting | Default | Example override |
|---|---|---|
| Implementer model | `composer` (using cursor) | "use sonnet as implementer" |
| Reviewer model | `grok` (using cursor) | "review with opus" |
| Max review rounds per task | 3 | "max 2 rounds" |
| Open PR at the end | no | "open a PR", "--pr", "and ship it" |
| Base branch | main | "branch off <branch-name>" |

Restate the active settings in one line when you start, so Andy can correct them early. For example: `Settings: implementer=composer, reviewer=grok, rounds=3, PR=no, base=main`.

## Working files

Keep everything for the run in `.local/dandy/<slug>/` at the repo root, where `<slug>` is the Linear issue ID (e.g. `PAS-412`) or a short kebab-case name if there's no issue:

```
.local/dandy/<slug>/
├── context.md      # phase 1: everything gathered
├── context/        # (optional) phase 1: files needed for implementers (figma screenshots, claude artifact files, html or whatever is useful for implementers to see)
├── decisions.md    # phase 2: questions asked and Andy's answers
├── plan.html       # phase 3: visual summary for Andy
├── tasks.md        # phase 3: task list with status, updated during phase 4
└── review.md       # phase 5: final review and verdict
```

These files let a run resume after an interruption: if `.local/dandy/<slug>/` already exists when you start, read it, tell Andy which phase you're resuming from, and continue. Never edit `.gitignore` for this: add `.local/dandy/` to the repo's local exclude file (`$(git rev-parse --git-common-dir)/info/exclude`) if it isn't there, so working files never land in the PR and the PR diff stays clean.

---

## Phase 0 — Workspace

Settle where the run lives before writing any working file: `.local/dandy/` is uncommitted, and nothing carries it to a new checkout.

1. Pick the branch: `andy/<issue-id>-<short-kebab-title>` (read the Linear title for it), or the branch Andy named.
2. In T3 Code (`t3_worktree_status` exists), check this thread's worktree:
   - Already on a non-default branch for this work → keep it.
   - In a T3 worktree with a default-named branch → `git branch -m <new>` inside it. T3 follows the folder, so the thread stays bound.
   - No worktree (on the project root / main) → call `t3_worktree_handoff` with `branch`, `baseRef: <base>`, and a `continuationPrompt` that says "Continue the dandy run for <slug> from phase 1" plus the Settings line. Make it the last call of the turn; the run resumes in the new worktree.
3. Outside T3, follow the repo's worktree rules (e.g. `.claude/worktrees/<slug>`).

Never `git checkout -b` in the root checkout, and never `git worktree add` for this thread inside T3. Either leaves the thread bound to main.

## Phase 1 — Gather context

Collect everything before asking Andy anything. Questions you could have answered by reading the issue or the code waste his time and make the grilling worse.

**Linear issue** (an ID like `PAS-412` or a linear.app URL, via the Linear MCP): read the title, description, comments, attachments, labels, parent issue, sub-issues, and linked/related issues. Comments often hold decisions that never made it into the description.

**Figma links** (via the Figma MCP): for each linked frame, get the design context and a screenshot. Note states shown (empty, loading, error, hover), responsive variants, and anything that looks like an existing design-system component.

**Claude artifacts** (claude.ai links): read them with whatever tool can open them. If none can, ask Andy to paste the content rather than guessing at it.

**Anything Andy provided inline**: treat it as the highest-priority source; ask Andy to solve if there is some conflict with other gathered context.

**Implementation taste**: read `~/.claude/skills/tandy/SKILL.md` (Andy's implementation taste) and the repo's `AGENTS.md` / `CLAUDE.md` if they exist. The repo's files override tandy where they conflict. Use them for every phase below.

**The codebase**: find where this feature lives. Identify the files and modules likely to change, existing patterns to follow (similar features, naming, folder structure), relevant tests, and how tests/lint/typecheck are run. Look for prior art: if something similar already exists, the plan should reuse it. Apply tandy's "Before writing code" order and scope rules: mark what can be reused, what must stay untouched, and what in the feature's path can be simplified or deleted.

Write `context.md` with: sources read (with links), the feature in your own words, relevant code locations, patterns to follow, reuse / leave alone / simplify notes, and a list of open questions and assumptions. The open questions feed phase 2.

Never copy secrets, API keys, customer PII, or internal financial data from any source into the working files, the plan, subagent prompts, or the PR. Reference where such data lives instead of reproducing it.

## Phase 2 — Grilling session

Use the `/grilling` skill to interrogate the open questions from phase 1, in batches.

How to batch:
- Ask every question through the interactive questions tool (`AskUserQuestion` in Claude Code; the equivalent structured question tool in other harnesses), never as plain text in chat.
- Ask up to 4 questions per batch (one tool call), then stop and wait for answers. This overrides grilling's one-at-a-time rule.
- Each question stands on its own and has 2–4 options. Put your recommended option first with "(Recommended)" at the end of its label, and give every option a short description carrying its trade-off.
- Order questions by how much they change the implementation. Scope, data model, and behavior questions come before naming and copy.
- Each question should be concrete, so Andy can pick the recommended option instead of writing an essay. For example: "Should archived items appear in search?" with "Exclude (Recommended)" described as "matches how the list view filters them".
- Don't ask what tandy or the repo's agent files already answer. Ask where they say nothing, where a rule conflicts with the feature, or to confirm a simplification or deletion of existing code in the feature's path.
- Later batches should build on earlier answers; drop questions that an answer already resolved and add new ones it raised.

Stop grilling when there are no open questions left that would change what gets built, or when Andy says to move on. Anything still uncertain becomes an explicit assumption in the plan.

Record every question and answer in `decisions.md` as you go.

**Checkpoint:** briefly confirm "That's everything I needed — moving to the plan" before phase 3.

## Phase 3 — Plan: visual summary + tasks

Produce two things.

**`plan.html`** — a single self-contained HTML file (inline CSS, inline SVG or a Mermaid block for diagrams, no external assets) that T3 Code will render. It should let Andy understand the whole plan in a couple of minutes. Include:
1. The feature in two or three sentences.
2. Key decisions from the grilling, each in one line.
3. A visual explanation: whichever diagram best fits the feature, such as a data flow, component tree, state machine, sequence diagram, database structure changes or before/after UI sketch. One clear diagram beats three busy ones.
4. The task list (same content as `tasks.md`), showing order and dependencies.
5. Assumptions and risks, including anything that would be expensive to change later.
6. Out of scope: what this work deliberately won't do, including things skipped under tandy's YAGNI rule and cleanups outside the feature's path (as follow-up notes).
7. Any new dependency, and any place the plan breaks a tandy rule, with the reason. Andy approves these explicitly.

Use html_render tool if available to display it to Andy.

**`tasks.md`** — the tasks to implement. Good tasks are small enough for one implementer pass (roughly a single concern, reviewable in one sitting), ordered so each builds on the previous, and independently verifiable. Use this format for each:

```markdown
## T1 — <short title>
Status: pending
Depends on: —
Files: <expected files to create/change>
Do: <what to implement, referencing decisions and patterns to follow>
Done when:
- <acceptance criterion>
- <tests that must exist/pass>
```

**Checkpoint:** show Andy the plan and stop. Proceed to phase 4 only when he approves. If he requests changes, update both files and show them again.

## Phase 4 — Implement and review, task by task

The branch was set in phase 0; confirm you're on it before the first task.

For each task in order:

1. **Implementer** — spawn a subagent with the implementer model. Give it a self-contained prompt, since it has none of your context:
   ```
   You are implementing one task of a larger feature.
   Feature summary: <2–3 sentences>
   Relevant decisions: <only those that affect this task>
   Patterns to follow: <files/examples from context.md>
   Implementation taste (follow it; if a rule fights the task, stop and
   report instead of breaking it): <full body of tandy/SKILL.md>
   Task: <full task block from tasks.md>
   Previous review feedback (if any): <reviewer's list, verbatim>
   Rules: change only what this task needs; add or update tests; run
   <test/lint/typecheck commands> and make them pass; don't commit.
   When done, report: files changed, what you did, test results, anything
   you were unsure about.
   ```

2. **Reviewer** — spawn a subagent with the reviewer model. Give it the task block, the relevant decisions, the diff (`git diff` of uncommitted changes), the full body of `tandy/SKILL.md`, and the checklist in `tandy/review.md`. Ask it to review following that checklist: correctness against the "Done when" criteria, safety, N+1 and unbounded queries, tests, tandy rules, scope, and codebase patterns. It must end with exactly one of:
   - `VERDICT: APPROVE`
   - `VERDICT: CHANGES_REQUESTED` followed by a numbered list of concrete, actionable changes.

   Tell it to request changes only for real problems with a concrete case (a bug, a tandy rule broken, or behavior that doesn't match the feature), not style preferences that neither the codebase nor tandy states, so the loop converges.

3. **Loop** — on `CHANGES_REQUESTED`, send the list back to the implementer (step 1 with feedback) and review again. Before forwarding, sanity-check the feedback yourself: drop items that contradict Andy's decisions, and note that you did.

4. **On approval** — run the test/lint/typecheck commands yourself to confirm, then commit with a message like `feat(<area>): <task title> [PAS-412]`. Update the task's status in `tasks.md` to `done`.

5. **Escalate** — if the task hits the max review rounds without approval, or the implementer reports it's blocked, stop and tell Andy: what the task is, what's disagreed on or blocking, and your recommendation. Don't silently skip tasks or loosen acceptance criteria.

Give Andy a one-line progress update after each task (e.g. `T2 done — approved in round 2 (fixed missing null check)`), not a full transcript.

## Phase 5 — Final Claude review and verdict

Once all tasks are done, review the whole branch yourself (`git diff <base>...HEAD`). This review is different from the per-task ones: you're looking at the feature as a whole, against the plan and Andy's decisions. Check:

- Every decision in `decisions.md` is reflected in the code.
- Every task's "Done when" criteria actually hold, not just that tests pass.
- The pieces fit together: no duplicated logic across tasks, consistent naming, no leftover TODOs, debug code, or dead code.
- Cross-cutting concerns: error and loading states from the Figma designs, permissions, migrations, performance for realistic data sizes.
- The whole diff follows tandy, using `tandy/review.md`: in particular no N+1 or unbounded queries, `switch` + `assertNever` on unions, no scope creep, and none of tandy's "Never do" items.
- Full test/lint/typecheck suite passes.
- No secrets, keys, or PII in the diff.

Write `review.md` and give Andy one of these verdicts with your reasoning:

- **SHIP** — ready as is.
- **SHIP WITH NOTES** — fine to merge; list the minor follow-ups (worth turning into Linear issues).
- **BLOCK** — something must be fixed first. List the issues and propose fix tasks. With Andy's go-ahead, run them through the phase 4 loop and review again.

Be honest. A verdict that's softer than the code deserves defeats the purpose of this phase.

## Phase 6 — PR or finish

**If Andy asked for a PR** (see Settings) and the verdict isn't BLOCK: push the branch and open a PR with `gh pr create` against the base branch.
- Title: `[PAS-412] <feature title>`
- Body: summary of the feature, link to the Linear issue and Figma, key decisions, the task list, how it was tested, and the notes from a SHIP WITH NOTES verdict. Don't paste working files or anything sensitive.

In T3, link the PR to this thread (`link_pull_request`). Share the PR link.

**Otherwise**, tell Andy it's finished: branch name, number of commits, the verdict, and any notes. Don't push or open a PR unasked.