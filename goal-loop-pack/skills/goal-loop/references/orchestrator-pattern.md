# Orchestrator Pattern

`goal-loop`'s implement step and gate-fix passes follow the orchestrator pattern. The main thread plans, dispatches, and synthesizes; sub-agents do the file edits. This reference documents the rules.

## When goal-loop applies this pattern

| Step | Why |
|---|---|
| Step 4 — IMPLEMENT | Per-wave parallel sub-agents do the edits, main thread tracks completion and waits for the wave |
| Step 5 fix dispatch | Code-review fixes go to sub-agents, main thread synthesizes summary and re-runs the gate |
| Step 6 fix dispatch | Review-swarm fixes go to sub-agents, main thread synthesizes summary and re-runs the gate |

The main thread NEVER does file edits in these steps. That is non-negotiable for the loop's correctness — the main thread is the planner/verifier, sub-agents are the implementers.

## The rules

### 1. Decompose before dispatching

Break the phase into discrete sub-agent tasks. Each task must have:

- A clear objective ("implement function X with signature Y")
- A scope (which file paths the sub-agent owns — exactly the `writes:` set from the Plan output)
- An expected output format
- Acceptance criteria (tests pass, lint clean, etc.)

Vague briefs like "go look at the auth module" are not tasks. Specific briefs like "in `src/auth/verify.ts`, add a `verifyToken(jwt: string)` function that validates against `JWKS_URL`, returns `Claims | null`, no edits outside this file" are.

### 2. Dispatch in parallel where the dependency graph permits

Within a wave, all tasks are mutually independent (see [plan-contract.md](plan-contract.md) for the wave structure invariants). Dispatch them in a single turn — one `Agent` call per task, all in the same message. Sequential dependencies move to the next wave.

### 3. Synthesize reports in the main context

When sub-agents return, the main thread reads each report and integrates findings before proceeding. Do not forward raw sub-agent output to the user as if it were main-thread analysis. The main thread is the synthesis layer.

### 4. No silent mode-exit

The main thread MUST stay in this pattern for every code-touching step. If the main thread decides "I'll just make this tiny fix myself," that's a violation — the loop's accounting (3-strike, wave tracking, `<NOTES>` rules) all assume sub-agent ownership of edits.

## What the main thread MAY do directly

- Read files for context (no Bash edits)
- Run verification commands (`bun test`, `git status`, lint checks)
- Update `<NOTES>` for synthesis summaries (sub-agents append their own decisions; main thread appends synthesis observations)
- TaskCreate / TaskUpdate to track per-task state
- Invoke `advisor()` and `/code-review` (the review-swarm protocol is applied via Agent dispatch within this very pattern — see [review-swarm-pattern.md](review-swarm-pattern.md))

## What the main thread MUST NOT do

- Edit files (Edit, Write, NotebookEdit, sed/awk Bash)
- Stage or commit changes
- Apply gate-fix patches
- Skip the sub-agent dispatch step "just for this tiny task"

## Sub-agent brief checklist

Every Agent call in IMPLEMENT or gate-fix steps must include:

- [ ] Objective (one sentence)
- [ ] Scope (exact `writes:` paths only — sub-agent must not touch peer paths in the same wave)
- [ ] Output format (what the report back should contain)
- [ ] Acceptance criteria (tests, lint, type-check, etc.)
- [ ] `<NOTES>` append requirement (per [notes-protocol.md](notes-protocol.md))
- [ ] If the user gave a specific quote that motivates this task, include it verbatim — do not paraphrase
