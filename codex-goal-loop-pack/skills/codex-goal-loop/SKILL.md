---
name: codex-goal-loop
description: Isolated Codex goal-loop skill that combines spec-to-plan, goal-loop-native phase planning, and gated phase-by-phase execution. Use when the user asks to turn a spec into a goal-loop plan, run a goal-loop plan, or package the spec-to-goal-plan, goal-loop-planner, and goal-loop flow as one Codex skill. Planner work uses GPT-5.4 high, implementation and gate-fix subagents use model 5.5 high, and review gates use model 5.4.
---

# codex-goal-loop

One isolated Codex skill for the full flow:

```text
SPEC.md -> plan markdown -> phase planning -> implementation -> review gates -> verify -> commit -> next phase
```

This skill intentionally folds the old three-skill chain into one skill:

- `spec-to-goal-plan`
- `goal-loop-planner`
- `goal-loop`

It does not depend on those standalone skills being installed.

## Invocations

- Create a plan from a spec: `Use $codex-goal-loop spec <SPEC_PATH> [PLAN_PATH]`
- Run a plan: `Use $codex-goal-loop run <PLAN_PATH>`
- Plain language: "Turn this spec into a goal-loop plan", "Run goal-loop on this plan"

If the mode is missing, infer it:

- A spec, PRD, design doc, issue bundle, or rough brief means Spec Mode.
- An existing markdown plan means Run Mode.
- An existing plan plus a current phase means Phase Planner Mode.

## Model Policy

- Planner work: GPT-5.4 with high reasoning.
- Implementation and gate-fix subagents: model 5.5 with high reasoning.
- Review gates: model 5.4.

## External Dependencies

This pack is isolated from the standalone planner/spec/loop skills, but still uses these Codex review skills when available:

- `$review-and-simplify-changes`
- `$review-swarm`

If either review skill is unavailable, stop and report the missing skill. If multi-agent tooling is unavailable, run the same task waves serially in the main thread and record that fallback in `IMPLEMENTATION_NOTES.md`.

See [references/dependencies.md](references/dependencies.md) for the dependency and fallback policy.

## Spec Mode

Use when the user provides a spec and wants a phase-ready plan.

### Inputs

- `<SPEC_PATH>`: implementation spec, PRD, design doc, issue list, architecture note, or rough brief.
- `<PLAN_PATH>`: optional output path. If omitted, create `PLAN-<spec-stem>.md` next to the spec.

### Workflow

1. Read the closest applicable `AGENTS.md` instructions and the spec.
2. For large specs, first build a heading index, then read sections defining scope, non-goals, data contracts, architecture, build order, tests, and acceptance criteria.
3. Inspect the repo enough to ground paths, command surfaces, test surfaces, entry points, and existing patterns.
4. Extract goal, non-goals, constraints, artifacts, dependencies, verification commands, migration order, and risks.
5. Write exactly one plan file.
6. Validate the plan shape before reporting completion.

### Required Plan Shape

```markdown
# PLAN: <name>

Source spec: `<SPEC_PATH>`
Goal-loop invocation: `Use $codex-goal-loop run <PLAN_PATH>`

## Goal
<one paragraph>

## Non-Goals
- <explicit exclusions>

## Global Constraints
- <repo, style, security, compatibility, and verification constraints>

## Phase Status
| Phase | Name | Status | Exit Gate |
| --- | --- | --- | --- |
| 1 | <name> | ready | <gate> |

## Phase 1: <name>

Goal: <phase-level goal>

Demo/Validation:
- <commands or manual checks>

### Task 1.1: <name>
- Location: `<path>`
- Description: <specific work>
- Dependencies: none
- Writes: `<path-or-directory>`
- Acceptance Criteria:
  - <observable criterion>
- Validation:
  - `<command>` or <manual check>

Phase Gate:
- <what must pass before the next phase>

## Final Acceptance
- <end-to-end criteria>

## Open Risks
- <risk or `None identified during planning`>

## Run Prompt
`Use $codex-goal-loop run <PLAN_PATH>`
```

### Spec Mode Self-Check

- Every task has `Location`, `Description`, `Dependencies`, `Writes`, `Acceptance Criteria`, and `Validation`.
- Every phase has a runnable or inspectable gate.
- Non-goals and constraints from the spec are represented.
- Phase order follows real dependencies.
- Same-wave work never shares a `Writes` path.
- The plan ends with the exact run prompt.

## Phase Planner Mode

Use internally during Run Mode for the current phase only.

### Workflow

1. Read the current phase, global plan constraints, repo instructions, git status, and relevant files.
2. Reconcile the phase with current code. Trust git for completed work and surface mismatches.
3. Split work into waves based on dependencies and `Writes`.
4. Return only the current phase decomposition.
5. If the phase cannot be made wave-safe, collapse conflicting work into serial waves.

### Phase Planner Output

```markdown
# Phase <N>: <name> - Execution Plan

## Phase Goal
<goal>

## Preconditions
- <repo state, prior phase output, or none>

## Waves

### Wave 1

#### Task <N>.1: <name>
- Location: `<path>`
- Description: <specific implementation work>
- Dependencies: none
- Writes: `<path-or-directory>`
- Acceptance Criteria:
  - <observable criterion>
- Validation:
  - `<command>` or <manual check>

## Phase Gate
- <commands and checks required before commit/resync>

## Notes
- <assumptions, conflicts, or `No off-spec decisions`>
```

The lower-level machine-checkable task contract is in [references/plan-contract.md](references/plan-contract.md).

## Run Mode

Implement the plan at `<PLAN_PATH>`. Loop until every phase is shipped or the loop is explicitly blocked by the 3-strike rule.

### Inputs

- `<PLAN_PATH>`: first markdown plan path.
- `<NOTES>`: `IMPLEMENTATION_NOTES.md` at repo root. Local-only working log. Never staged. Never committed. See [references/notes-protocol.md](references/notes-protocol.md).

### Bootstrap

1. Read `<PLAN_PATH>` fully and extract phase order plus current done/next state.
2. If the plan disagrees with git history, trust git and surface the discrepancy.
3. Create `<NOTES>` if absent with `# Run: <date>`. Ensure it is ignored by git before appending.
4. Append `## Phase N - <name>` at the start of each phase.
5. Create or update the Codex task plan with one item per loop step for the current phase.

### Loop Per Phase

1. **Plan**: Use Phase Planner Mode on GPT-5.4 high. The output must satisfy [references/plan-contract.md](references/plan-contract.md) and cover this phase only.
2. **Plan Sanity Check**: Validate planner output against `<PLAN_PATH>`, repo state, dependency order, and write-scope conflicts. Any concrete mismatch counts toward 3-strike.
3. **Ingest**: Add or update Codex task plan items for each task, preserving `depends_on` and `writes`. Verify 1:1 with the phase plan and reject any same-wave shared `writes` path.
4. **Implement**: Follow [references/orchestrator-pattern.md](references/orchestrator-pattern.md). Dispatch independent file-edit tasks to subagents in parallel when multi-agent tools are available. If unavailable, implement serially while preserving wave boundaries and logging the fallback in `<NOTES>`.
5. **Review-And-Simplify Gate**: Use `$review-and-simplify-changes` on the changed surface before commit. Use model 5.4 for all review subagents. Use review-only unless the phase explicitly asks for safe simplification. Blocking findings must be fixed.
6. **Review-Swarm Gate**: Use `$review-swarm` on the same changed surface. Use model 5.4 for all review-swarm subagents. High and medium findings block.
7. **Fix Blocking Findings**: Group findings by file. Disjoint groups can be fixed in parallel by subagents; overlapping groups serialize. Re-run the relevant gate until zero blocking findings. Use [references/gate-stuck-clause.md](references/gate-stuck-clause.md) before a third fix pass on the same overlapping surface.
8. **Verify and Commit**: Run the smallest relevant tests plus any build, lint, or type checks for touched surfaces. Commit one scoped Conventional Commit per logical change unless the user said not to commit or a safe scoped commit is unclear. Do not stage `<NOTES>`.
9. **Resync**: Re-read `<PLAN_PATH>` and git state. If another phase remains, continue the loop. Otherwise proceed to Exit.

### Exit Criteria

All must hold:

- Every phase in `<PLAN_PATH>` has at least one commit on the current branch, unless the phase was explicitly documented as no-op.
- No phase remains "next" or "todo" in `<PLAN_PATH>`.
- Final review-and-simplify and review-swarm gates on HEAD have zero blocking findings.
- Final main-thread exit check reviews `<PLAN_PATH>` and `<NOTES>` and confirms the goal is materially met with no open hazards.

## Safety Rails

- 3-strike counter is per-phase-per-step across Plan, Plan Sanity Check, Review-And-Simplify, Review Swarm, and individual implementation tasks. On the 3rd failure, stop and surface: step or task, last 3 outputs, and what was tried.
- Wave exclusivity is absolute: same-wave tasks must not share any `writes` path. Collapse conflicts into serial sub-waves.
- `<NOTES>` is local-only and append-only within a phase. Empty-but-explicit `no off-spec decisions` is allowed; silent omission is not.
- Never skip review gates. Never amend or force-push committed phases unless the user explicitly asks.
- Respect the current repo's closest `AGENTS.md` instructions and dirty worktree state. Do not revert unrelated user changes.
