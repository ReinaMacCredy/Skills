# Advisor Protocol

`advisor()` is a stronger reviewer with full conversation transcript context. It catches **direction errors** that deterministic gates miss — misinterpretation, false-success, patch-pattern smell.

## Mandatory slots

| Slot | When | Purpose |
|---|---|---|
| **PLAN SANITY** | Step 2, once per phase, after Plan returns | Catch phase misread before any edit lands |
| **EXIT GOAL-MET** | After final commit, before EXIT | Confirm the goal in prose is materially met |

## Conditional slot

| Slot | Trigger | Purpose |
|---|---|---|
| **STUCK CLAUSE** | 2nd gate-fix pass surfaces a finding overlapping 1st pass's surface area | Judge root cause vs. paper-over before 3rd pass |

See [gate-stuck-clause.md](gate-stuck-clause.md) for the overlap-detection rules.

## Blocking criteria

Advisor findings **block** the loop only when they cite ONE of:

1. **A specific line of `<PLAN_PATH>`** that contradicts Plan output
2. **A specific prior-phase commit/decision** that contradicts current direction
3. **A parallelism hazard** — two same-wave tasks writing same path, missing dep edge, etc.
4. **A patch-pattern flag** — e.g., "this fix suppresses the symptom by adding a try/catch instead of fixing the invariant"
5. **An open-hazard flag in `<NOTES>`** — advisor surfacing that a sub-agent flagged something the main thread didn't address

## Non-blocking

The following are noted but do NOT gate the loop:

- Style preferences
- Refactor suggestions not asked for
- "Could be improved" without a citation
- General code quality observations
- Subjective polish

## 3-strike counter

A blocking advisor finding counts toward the per-phase-per-step counter, like Plan or gate failures. 3 consecutive blocks on the same step → STOP and surface to user with (a) which step, (b) last 3 outputs, (c) what was tried.

## Why advisor and not just gates

| Reviewer | Reads | Catches |
|---|---|---|
| `/code-review` | Diff (the lines) | Bugs, correctness, type errors |
| review-swarm protocol (inlined) | Diff + scope context | Regressions, perf, security, contract gaps |
| `advisor()` | Full transcript + prose | Direction drift, intent misread, patch-pattern smell |

Each one reads a different artifact, so they don't duplicate. Advisor is the only reviewer that can tell you "you understood the phase wrong" or "the code is correct but doesn't actually achieve the goal."
