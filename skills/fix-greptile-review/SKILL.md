---
name: fix-greptile-review
description: Fetch Greptile's review comments on the current PR — both inline comments and the "Comments Outside Diff" findings in its main comment — judge each against the code, fix the valid ones, then commit and push. Use when the user says "grr", "greptile", "fix-greptile-review", "address greptile comments", "fix the PR review comments", or "check PR feedback" — even when they don't name the tool.
---

# fix-greptile-review (grr) — Greptile Review, Resolved

Pull Greptile's inline review comments on a PR, separate the real findings from the noise, fix the real ones, and push. The point is a trustworthy filter: Greptile is a useful reviewer but often wrong, so every comment earns its verdict from the actual code — never a blind apply, never a blanket dismiss.

## Arguments

Optional: a PR number or URL (e.g. `142` or `https://github.com/org/repo/pull/142`). With no argument, target the current branch's PR.

## Steps

1. **Find the PR.**
   - Argument given → extract the PR number from it.
   - Otherwise: `gh pr view --json number,url,headRefName,baseRefName`
   - No PR found → tell the user and stop. Never open one.

2. **Fetch Greptile's inline comments** — these are the actionable per-line findings:
   ```bash
   gh api repos/{owner}/{repo}/pulls/{number}/comments --paginate \
     --jq '.[] | select(.user.login == "greptile-apps[bot]") | {id, path, line, body}'
   ```
   The author login is `greptile-apps[bot]` (verify with `.user.login | contains("greptile")` if that ever changes). Each body opens with a priority badge — **P1** (likely bug), **P2** (worth fixing), **P3** (nit).

2b. **Also fetch Greptile's "Comments Outside Diff."** Greptile can only post inline on lines inside the PR diff; findings on untouched lines get parked in a collapsed section of its main comment instead, so step 2 misses them entirely. Read the main comment:
   ```bash
   gh api repos/{owner}/{repo}/issues/{number}/comments --paginate \
     --jq '.[] | select(.user.login == "greptile-apps[bot]") | .body'
   ```
   Inside it, find the block headed `Comments Outside Diff (N)` — a `<details>…</details>` section ending at the `<!-- /greptile_failed_comments -->` marker. Each entry is numbered and gives `` `path` ``, a line range, a priority badge, a bold title, and the finding (sometimes a ` ```suggestion ` block). Treat every one as a finding to judge in step 3, exactly like an inline comment — they are often the highest-priority ones (P1s about code the diff didn't touch). Ignore the rest of the main comment (PR summary, walkthrough, "Fix in …" badge links).

   If there are no inline comments **and** no Comments-Outside-Diff entries, say so and stop — there is nothing to fix or push.

3. **Judge each comment** (inline and outside-diff) against the code it points at. Read the file and enough surrounding context to reach a verdict — *valid* or *not* — with a one-line reason. Every comment gets a verdict; do not skim the batch and apply in bulk. Weight by priority: a P1 deserves real scrutiny before you dismiss it, a P3 style nit that fights project convention is usually a skip.
   - **Valid** (real bug, correctness risk, or a fix that fits the codebase) → apply the smallest fix that resolves it, following the project's own conventions (its `AGENTS.md` / `CLAUDE.md` / contributing guide, plus the surrounding code's formatting, naming, and error-handling patterns).
   - **Not valid** (false positive, already handled, or a nit that contradicts project convention) → skip it and keep the reason.
   - **Large refactor** → do not apply blindly; flag it for the user and move on.

4. **Commit and push** — only if step 3 actually changed files.
   - Stage the edits, commit with a message summarizing the fixes, and `git push` to the PR's head branch.
   - No edits → skip commit/push entirely; there is nothing to push.
   - Push to the existing PR branch only — never to `baseRef`/`main`, and never open a new PR. (Pushing is the user's standing instruction for this skill, so don't ask again — but stop and surface anything that would force-push or rewrite history.)

5. **Report back.** Give the user a compact summary: each comment as **fixed** (what changed) or **skipped** (why, with its priority), the commit pushed (or "nothing to push"), and any refactors left for their call.

## Notes

- Greptile also leaves a top-level PR summary (an issue comment) and a review entry; the actionable items are the inline comments step 2 fetches. Mention the summary only if it raises something the inline comments don't.
- To resolve a fixed comment's thread, use the GraphQL `resolveReviewThread` mutation; skip it silently if unavailable — resolving is a courtesy, not the job.
