# skills

Personal [agent skills](https://github.com/vercel-labs/skills) by [@martinbavio](https://github.com/martinbavio).

## Install

```bash
npx skills add martinbavio/skills
# or a single skill:
npx skills add martinbavio/skills --skill tenacious-review
```

## Skills

| Skill | Description |
|:------|:------------|
| `tenacious-review` | Consolidated local PR review — runs the thermo correctness/security + code-quality rubrics (plus React/studio lenses when the diff touches a web frontend) over the worktree diff, then synthesizes one prioritized, deduped report. Harness-agnostic (Claude Code, Cursor, Codex, …); parallel sub-agents where supported, sequential otherwise. |
| `thermo-nuclear-review` | Deep correctness & security branch audit (bugs, breaking changes, security, devex regressions, feature-flag leaks), scoped to the diff. |
| `thermo-nuclear-code-quality-review` | Strict maintainability audit (abstraction quality, giant files, spaghetti-condition growth, boundaries). |

`tenacious-review` orchestrates the two `thermo-nuclear-*` rubrics as its always-on lenses.

## Attribution

`thermo-nuclear-review` and `thermo-nuclear-code-quality-review` are adapted from the
[Thermos plugin](https://github.com/cursor/plugins/tree/main/thermos) (MIT) — ported to be
harness-agnostic. `tenacious-review` is original.

## License

MIT
