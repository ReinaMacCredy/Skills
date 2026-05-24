# Dependencies

`goal-loop` is **self-contained** — no third-party skills required. The orchestrator and review-swarm protocols are inlined as references in this directory.

## Dependency list

All dependencies ship with Claude Code itself:

| Dependency | Type | Used in |
|---|---|---|
| `Plan` subagent (Opus) | Built-in agent type | Step 1 |
| `advisor()` | Built-in tool | Step 2, EXIT, STUCK CLAUSE |
| `TaskCreate` | Built-in tool | Step 3 |
| `Agent` (sub-agent spawning) | Built-in tool | Step 4 + parallel fixes |
| `/code-review` | Skill (Claude Code built-in) | Step 5 |

The protocols that goal-loop applies — the orchestrator pattern and review-swarm — live as references in this directory:

| Protocol | Reference file |
|---|---|
| Orchestrator pattern (main thread plans, sub-agents edit) | [orchestrator-pattern.md](orchestrator-pattern.md) |
| Review-swarm (four parallel reviewer sub-agents) | [review-swarm-pattern.md](review-swarm-pattern.md) |
| Plan-subagent task + wave shape | [plan-contract.md](plan-contract.md) |
| Advisor placement and blocking criteria | [advisor-protocol.md](advisor-protocol.md) |
| Gate stuck clause | [gate-stuck-clause.md](gate-stuck-clause.md) |
| Notes protocol (off-spec decision log) | [notes-protocol.md](notes-protocol.md) |

## Preflight check

At BOOTSTRAP, the main thread should sanity-check the built-in surface:

1. Confirm an `advisor()` tool entry exists.
2. Confirm `TaskCreate` and `Agent` are callable.
3. Confirm `/code-review` is in the skills list (system-reminder block).

If any built-in is missing the harness is misconfigured — STOP before INGEST and surface what's missing.

## Why inlined instead of bundled skills

A previous iteration of this pack shipped `orchestrator-only` and `review-swarm` as sibling skills. That created two problems:

1. **Drift surface**: each protocol lived in two places (canonical skill + bundled copy), so edits in one didn't reach the other.
2. **Install fragility**: missing/renamed sibling skills broke goal-loop entirely.

Inlining the protocols as references makes goal-loop a single artifact with one source of truth for its own behavior. The standalone `orchestrator-only` and `review-swarm` skills still exist independently in `~/Code/Skills/` for ad-hoc invocation outside the loop — they are no longer a dependency of this bundle.
