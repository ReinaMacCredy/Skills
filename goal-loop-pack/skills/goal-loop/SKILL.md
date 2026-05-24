---
name: goal-loop
description: Autonomous phase-by-phase plan execution loop. Plan-subagent decomposes each phase into a dependency-tagged wave graph, parallel Opus sub-agents implement via the inlined orchestrator pattern (main thread plans/synthesizes, sub-agents do all edits), advisor + /code-review --effort extra-high + the inlined review-swarm protocol gate every phase, IMPLEMENTATION_NOTES.md captures off-spec decisions. Self-contained — orchestrator and review-swarm protocols ship inline as references, no third-party skill dependencies. Use when the user invokes /goal-loop <plan-path>, when /goal hook fires with a goal-loop directive, or when the user wants autonomous phase-by-phase execution of a markdown plan file. Pairs with goal-distill (which produces compatible plan prompts).
user-invocable: true
---

# goal-loop

Implement the plan at `<PLAN_PATH>`. Loop until every phase is shipped.

## Invocation

- Slash: `/goal-loop <path-to-plan.md>`
- Skill: `Skill(skill="goal-loop", args="<path-to-plan.md>")`
- /goal hook: `/goal /goal-loop <path-to-plan.md>`

## Inputs

- `<PLAN_PATH>` = first argument. If empty, STOP and print usage: `/goal-loop <path-to-plan.md>`.
- `<NOTES>` = `IMPLEMENTATION_NOTES.md` at repo root. **Local-only working log. Never staged. Never committed.** See [references/notes-protocol.md](references/notes-protocol.md).

## Dependencies

Self-contained — no third-party skills required. The orchestrator pattern and review-swarm protocol are inlined as references. See [references/dependencies.md](references/dependencies.md) for the full list and preflight check.

| Dependency | Type | Used in |
|---|---|---|
| `Plan` subagent (Opus) | Built-in agent type | Step 1 |
| `advisor()` | Built-in tool | Step 2, EXIT, STUCK CLAUSE |
| `TaskCreate` | Built-in tool | Step 3 |
| `Agent` (sub-agent spawning) | Built-in tool | Steps 4, 6, gate-fixes |
| `/code-review` | Skill (Claude Code built-in) | Step 5 |

## BOOTSTRAP (once)

- Read `<PLAN_PATH>` fully; extract phase order + done/next state.
- If file disagrees with git history, trust git and surface the diff.
- Create `<NOTES>` if absent (`# Run: <date>` header). Ensure gitignored. Append `## Phase N — <name>` at the start of each phase.

## LOOP — each remaining phase, in order

1. **PLAN** — Spawn `Plan` subagent (`model: "opus"`). Reads `<PLAN_PATH>` fully, finds next phase, returns a file-level plan for THIS phase only. See [references/plan-contract.md](references/plan-contract.md) for required task + wave shape.

2. **ADVISOR — PLAN SANITY** (mandatory, once per phase) — `advisor()` on Plan output. See [references/advisor-protocol.md](references/advisor-protocol.md) for blocking vs. non-blocking criteria. Counts toward 3-strike.

3. **INGEST** (main session) — TaskCreate one task per step, preserving `depends_on` + `writes`. Verify 1:1 with Plan AND no path appears in two same-wave tasks. On conflict, reject and re-Plan.

4. **IMPLEMENT** (orchestrator pattern, wave-parallel) — Per wave: 1 task → 1 Opus sub-agent; N>1 → N sub-agents IN PARALLEL (single message, multiple Agent calls). Each owns one task and only its `writes` paths. Each sub-agent MUST append to `<NOTES>` per [references/notes-protocol.md](references/notes-protocol.md). Main thread waits for the full wave before starting the next. Task failure → isolate, do not roll back peers, re-dispatch only the failed task (3-strike). No scope creep, no adjacent edits. **Full protocol**: [references/orchestrator-pattern.md](references/orchestrator-pattern.md).

5. **CODE-REVIEW GATE** — `/code-review --effort extra-high` on changed surface. BLOCKING.

6. **REVIEW-SWARM GATE** — Apply the inlined review-swarm protocol on the changed surface. Four read-only reviewer sub-agents (Intent & Regression, Security & Privacy, Performance & Reliability, Contracts & Coverage) in parallel; main thread filters and synthesizes. High/medium findings BLOCKING; low non-blocking nits deferrable per commit note. **Full protocol**: [references/review-swarm-pattern.md](references/review-swarm-pattern.md).

   Shared fix rule (5+6): group findings by file. Disjoint → parallel fix sub-agents (orchestrator pattern); overlapping paths serialize. Fix sub-agents append to `<NOTES>` if a fix forces a design change. Re-run gate until zero (blocking) findings. **STUCK CLAUSE**: see [references/gate-stuck-clause.md](references/gate-stuck-clause.md).

7. **COMMIT** — One Conventional Commit per logical change. **Do NOT stage `<NOTES>`.** Never `--no-verify`. Gate-fixes may fold in or ship as follow-ups, but must land before step 8.

8. **RE-SYNC** — Re-read `<PLAN_PATH>`. Phase remains (including mid-flight additions) → continue loop. Else proceed to EXIT.

## EXIT — all must hold

- Every phase in `<PLAN_PATH>` has ≥1 commit on the current branch.
- No phase remains "next"/"todo" in `<PLAN_PATH>`.
- Latest `/code-review --effort extra-high` and review-swarm protocol on HEAD: zero blocking findings.
- Final `advisor()` (post last commit) reviews `<PLAN_PATH>` + `<NOTES>`. Confirms goal materially met AND no `<NOTES>` open-hazard. Concrete unmet criterion or open hazard → new phase, re-enter loop. Vague feedback does not block exit.

## SAFETY RAILS

- **3-strike counter** is per-phase-per-step across Plan, Advisor, Code-Review, Review-Swarm, AND per individual parallel task. On 3rd failure: STOP and surface (a) which step/task, (b) last 3 outputs, (c) what was tried.
- **Wave exclusivity is absolute**: same-wave tasks MUST NOT share any `writes` path. Detected at dispatch → collapse into a serial sub-wave.
- **`<NOTES>` rules**: local-only, append-only within a phase, never staged, never committed. Empty-but-explicit ("no off-spec decisions") allowed; silent omission is a 3-strike violation.
- **Advisor blocks** only on specific citations, contradictions, parallelism hazards, patch-pattern flags, or `<NOTES>` open-hazard flags. Polish never gates.
- Never skip `/code-review` or the review-swarm protocol. Never amend or force-push committed phases. Never mark a phase done with open findings.
- All code edits (implement + gate-fix) follow the orchestrator pattern: main thread plans, dispatches, and synthesizes; sub-agents do all file edits. Main thread MUST NOT edit files in steps 4, 5-fixes, or 6-fixes.
