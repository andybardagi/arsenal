---
name: babysit-pr
description: >-
  Get a GitHub PR ready to merge: fix CI failures caused by the PR, and triage
  Greptile/CodeRabbit review comments, fixing what matters and replying to each
  with "Fixed in <commit>" or "Not fixing now because <reason>". Resolve the
  thread after a fix. Use when the user invokes /babysit-pr or asks to handle
  failing checks or bot review comments on a PR.
---

# babysit-pr

Get the PR green and answer every bot review comment. Fix real problems with minimal changes; skip noise, and say why.

## 1. Set up

Use the PR the user named, or the current branch's PR (`gh pr view --json number,url,headRefName,baseRefName`). Check out its head branch and `git pull`. If the working tree has uncommitted changes, stop and ask.

## 2. Fix failing CI checks

```bash
gh pr checks                        # find failing checks
gh run view <run-id> --log-failed   # read why they failed
```

For each failure, decide whether this PR caused it:
- **Caused by the PR**: the failure is in code or tests the PR touches, or follows from its changes. Fix it with the smallest change that makes it correct. Fix the code if the code is wrong; update the test only if the PR intentionally changed that behavior.
- **Not caused by the PR**: it also fails on the base branch, it's flaky (passes on rerun, timing-dependent), or it's infrastructure (timeouts, network, runner errors). Don't fix it. Mention it in the final report.

Commit each fix as `fix: <what> (CI)`.

## 3. Triage bot review comments

Only handle comments from Greptile and CodeRabbit (login contains `greptile` or `coderabbit`). Ignore humans completely.

```bash
gh api --paginate repos/{owner}/{repo}/pulls/{number}/comments   # inline comments
gh api --paginate repos/{owner}/{repo}/pulls/{number}/reviews    # CodeRabbit nitpicks live in review bodies
```

Skip summaries/walkthroughs, resolved threads, threads a human replied to, and comments that already have a `Fixed in` / `Not fixing now` reply (so reruns only handle new comments).

For each remaining comment, read the surrounding code before deciding. Bots often miss context.

**Fix** real problems in realistic scenarios: bugs, crashes or unhandled errors on paths that can actually happen, security issues, data loss, leaks, broken contracts.

**Don't fix**: unlikely edge cases, overengineering (new abstractions, options, layers "for later"), style the linter doesn't enforce, false positives, or out-of-scope work. When in doubt, don't fix it, and explain why.

Fixes are minimal and follow the file's existing patterns. Don't blindly apply the bot's suggested code. One commit per fix (`fix: <what> (review feedback)`), shared when several comments flag the same issue.

## 4. Verify and push

Run the project's tests, lint, and typecheck locally. Fix anything your changes broke, then `git push`. Push before replying, so the commit links work.

## 5. Reply to each comment

Use exactly one of:
- `Fixed in <short-hash>`
- `Not fixing now because <specific one-sentence reason>`

A reason names the actual cause: "userId is validated by the route schema before this runs", never "not needed".

```bash
gh api repos/{owner}/{repo}/pulls/{number}/comments/{comment_id}/replies -f body="Fixed in a1b2c3d"
```

After a `Fixed in` reply, resolve that review thread. Leave `Not fixing now` threads open.

The thread id is a GraphQL node id. Find the `reviewThreads` node whose comments include the database id you replied to (follow `pageInfo.endCursor` while `hasNextPage`), then resolve it:

```bash
gh api graphql -f query='query($o:String!,$r:String!,$n:Int!,$after:String){repository(owner:$o,name:$r){pullRequest(number:$n){reviewThreads(first:100,after:$after){pageInfo{hasNextPage endCursor}nodes{id isResolved comments(first:100){nodes{databaseId}}}}}}}' -f o=OWNER -f r=REPO -F n=NUMBER

gh api graphql -f query='mutation($id:ID!){resolveReviewThread(input:{threadId:$id}){thread{isResolved}}}' -f id="$THREAD_NODE_ID"
```

Review-body items have no thread, so answer them in one PR comment with a line per item (`path:line — Fixed in …`) and there is nothing to resolve.

Never include secrets, keys, or customer data in replies.

## 6. Report

Tell the user, briefly:
- CI: what failed, what you fixed, and what you left alone (flaky, infra, or broken on base) and why.
- Comments: how many were fixed vs. not fixed, with one line each (`path:line — fixed a1b2c3d, thread resolved` / `not fixing: reason`).
- Anything worth a follow-up issue.
- Whether local checks passed and everything was pushed. CI re-runs on the push; run /babysit-pr again if new failures or comments appear.