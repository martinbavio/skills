---
name: tenacious-review
description: "Consolidated local PR review — applies the thermo-nuclear correctness/security and code-quality rubrics (plus React/studio lenses when the diff touches a web frontend) over the worktree diff and synthesizes one prioritized, deduped report. User-invoked."
disable-model-invocation: true
---

# Tenacious Review

Run one consolidated review of the current worktree by applying several independent review lenses (fresh eyes each), then synthesizing a single prioritized, deduped report. This replaces juggling separate review sessions.

## 1. Establish scope

- **Base ref:** if the user named a base ref, use it; otherwise default to `origin/main`.
- `git fetch origin --quiet`, then compute the diff with the merge-base (three-dot):
  - `git diff <base>...HEAD --stat`
  - `git diff <base>...HEAD --name-only`
  - `git diff <base>...HEAD` (the full patch)
- Read the **current full contents** of the changed files (skip deletions, lockfiles, and generated output) so reviewers evaluate real code, not just the patch.
- If the diff is empty, stop and report "no changes vs `<base>`".
- If you are reviewing the result of a **structural recommendation** from a prior pass (an extraction or refactor you advised), re-run the affected lens against it — that new surface has been reviewed by no one.

## 2. Decide which lenses apply

- **Always:** the two thermo-nuclear lenses — `thermo-nuclear-review` (correctness/security) and `thermo-nuclear-code-quality-review` (maintainability).
- **Frontend lenses only if changed files include `web/` paths:**
  - any changed `.ts`/`.tsx`/`.jsx` under `web/` → add `react-doctor`, `vercel-react-best-practices`, `vercel-composition-patterns`
  - any change under `web/apps/studio/**` → add the `studio-review` lens **if that rubric is available in this environment** (it is repo-local; skip it silently when it doesn't exist).
- If the diff is backend-only (no `web/`), skip every frontend lens and say so in the report.

## 3. Run the lenses

Build the shared context block once and give it verbatim to every lens:

```
### Git / diff output
<git diff <base>...HEAD>

### Changed file contents
<path + full contents, per changed file>
```

Run each applicable lens as an **independent parallel sub-agent if your harness supports them; otherwise run the lenses sequentially.** Each sub-agent gets the same shared context block, applies exactly one rubric to **only** this diff, and **reports findings only — it must not modify any files or run fixes.** Do not spawn nested sub-agents; every lens is independent.

For each lens, the instruction is: *"Apply the **`<lens>`** rubric to ONLY this diff (load the `<lens>` skill if your harness has one; otherwise follow that rubric's criteria). Report findings only — do not modify any files.\n\n<shared context block>"*

Lenses:

1. `thermo-nuclear-review` — bugs, breaking changes, security, devex regressions, feature-flag leaks.
2. `thermo-nuclear-code-quality-review` — maintainability, structure, 1k-line rule, spaghetti, boundaries.
3. *(if studio + available)* `studio-review` — studio invariants.
4. *(if React)* `react-doctor`.
5. *(if React)* `vercel-react-best-practices`.
6. *(if React)* `vercel-composition-patterns`.

## 4. Synthesize

Once all lenses have finished, produce **one** report:

- **Verdict** line first (e.g. *Ship / Minor polish / Needs changes / Blocker*).
- **Findings grouped by severity** (Blocker → High → Medium → Low → Nit), **deduped across lenses**. When ≥2 lenses raise the same issue, merge it and mark it **corroborated** (weight it higher).
- Each finding: `file:line` + one-line evidence + which lens(es) raised it.
- Drop cosmetic nits when structural/correctness issues dominate.
- Don't restate each lens wholesale — surface the unified verdict, highest-signal findings, and any remaining uncertainty.
