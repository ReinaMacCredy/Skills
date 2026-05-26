# Orchestrator Pattern

`codex-goal-loop` uses an orchestrator pattern for implementation and gate-fix work. The main thread owns planning, dispatch, synthesis, verification, and commits. Subagents perform scoped edits when available.

## When To Apply

| Step | Action |
|---|---|
| Implement | Wave-based implementation tasks |
| Review-and-simplify fixes | Blocking finding fixes |
| Review-swarm fixes | Blocking finding fixes |

## Main Thread Responsibilities

- Read context and closest instructions.
- Maintain the Codex task plan.
- Verify `writes` path exclusivity before dispatch.
- Dispatch subagents when available.
- Synthesize subagent reports.
- Run verification and review gates.
- Stage and commit only scoped files when safe.

## Subagent Responsibilities

Each subagent gets exactly one task with:

- Objective.
- Owned `writes` paths.
- Acceptance criteria.
- Verification command expectation.
- `IMPLEMENTATION_NOTES.md` append requirement.
- Model setting: 5.5 with high reasoning.

Subagents must not edit peer-owned paths in the same wave.

## Fallback When Subagents Are Unavailable

1. Keep the same task decomposition.
2. Execute wave tasks serially in the main thread.
3. Preserve `writes` exclusivity.
4. Append a note that the phase used serial main-thread fallback.
5. Do not represent the run as parallelized.

## Dispatch Checklist

Every implementation or fix task must include:

- One-sentence objective.
- Exact file path scope.
- Output/report format.
- Acceptance criteria.
- Required tests/checks.
- Notes logging rule.
- Model setting: 5.5 with high reasoning.
- Any user quote that materially defines the task.
