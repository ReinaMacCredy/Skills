# IMPLEMENTATION_NOTES.md Protocol

`IMPLEMENTATION_NOTES.md` is a local-only running log of decisions, tradeoffs, and assumptions the agent makes that are not in the plan.

## File Rules

- Path: `IMPLEMENTATION_NOTES.md` at repo root.
- Lifecycle: local-only working log. Never staged. Never committed.
- Gitignore: ensure the file is ignored before first append. If `.gitignore` does not cover it, add an entry.
- Header: on first creation, write `# Run: <YYYY-MM-DD>`.
- Per-phase section: at the start of each phase, append `## Phase N - <phase name>`.
- Append-only within a phase: never rewrite prior phase sections.

## What Gets Written

Each implementation or gate-fix subagent must append at least one of:

| Trigger | What to write |
|---|---|
| Off-spec decision | "Decided to use X instead of Y because <reason from reality, not spec>" |
| Forced change | "Plan said edit file A; file A does not exist, edited file B instead" |
| Tradeoff | "Chose perf over readability here because <constraint>" |
| Assumption | "Assumed input is utf-8; revisit if non-ASCII paths show up" |
| Follow-up | "Left TODO at src/foo.ts:42 - needs cache invalidation" |
| Open hazard | "Race condition possible between X and Y; not addressed in this phase" |

## No Off-Spec Decisions Rule

If a subagent finished its task and made zero off-spec decisions, it must explicitly write:

```text
[task-id] no off-spec decisions
```

Silent omission is a 3-strike violation.

## Exit Consumption

At Exit, the final main-thread check reads `<PLAN_PATH>` and `IMPLEMENTATION_NOTES.md`. The loop blocks Exit if notes flag an open hazard that was not addressed in a later phase.
