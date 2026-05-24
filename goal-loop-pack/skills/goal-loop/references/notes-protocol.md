# `IMPLEMENTATION_NOTES.md` Protocol

A local-only running log of decisions, tradeoffs, and assumptions the agent makes that aren't in the plan. Designed to surface silent direction-drift that wouldn't show up in commits or diffs.

## File rules

- **Path**: `IMPLEMENTATION_NOTES.md` at repo root.
- **Lifecycle**: Local-only working log. **Never staged. Never committed.**
- **Gitignore**: Ensure the file is gitignored before first append. If `.gitignore` doesn't cover it, add an entry.
- **Header**: On first creation, write `# Run: <YYYY-MM-DD>` at the top.
- **Per-phase section**: At the start of each phase, append `## Phase N — <phase name>`.
- **Append-only within a phase**: Never rewrite prior phase sections.

## What gets written

Each sub-agent (implement step + gate-fix step) MUST append to the current phase section with at least one of:

| Trigger | What to write |
|---|---|
| Off-spec decision | "Decided to use X instead of Y because <reason from reality, not spec>" |
| Forced change | "Plan said edit file A; file A doesn't exist, edited file B instead" |
| Tradeoff | "Chose perf over readability here because <constraint>" |
| Assumption | "Assumed input is utf-8; need to revisit if non-ASCII paths show up" |
| Follow-up | "Left TODO at src/foo.ts:42 — needs cache invalidation" |
| Open hazard | "Race condition possible between X and Y; not addressed in this phase" |

## "No off-spec decisions" rule

If a sub-agent finished its task and made **zero** off-spec decisions, it MUST explicitly write:

```
[task-id] no off-spec decisions
```

**Silent omission is a 3-strike violation.** The empty signal must be explicit, not inferred, because silence is indistinguishable from "I forgot to log a decision."

## Advisor consumption

At EXIT, the final `advisor()` call reads:

- `<PLAN_PATH>` — the goal
- `<NOTES>` — the run log

Advisor blocks EXIT if `<NOTES>` flags an **open hazard** that wasn't addressed in a later phase. This is the primary safety net against "passed all gates but didn't actually finish the goal."

## Example log

```markdown
# Run: 2026-05-24

## Phase 1 — Add SessionStore interface

- [t-001] no off-spec decisions
- [t-002] Decided to make `SessionStore.get` return `Promise<Session | undefined>` instead of throwing on miss. Reason: simplifies the cache layer in phase 3.
- [t-003] Open hazard: tests use a global `Date.now()` mock. If phase 4 introduces concurrent sessions, the mock may need scoping.

## Phase 2 — Wire SessionStore into App

- [t-004] Plan said edit `src/app.ts`; the actual entry is `src/app/index.ts` after a 2025-12 refactor. Edited the real path.
- [t-005] no off-spec decisions
```

## Why this exists

Sub-agents make hundreds of small decisions during a multi-phase run. The diff captures *what* changed. The commit message captures *why this commit*. Neither captures *the tradeoffs and assumptions made along the way*. That gap is where silent direction-drift hides — and where the goal-vs-built mismatch shows up at EXIT if you don't catch it earlier.
