# Dependencies

This pack is isolated from the standalone `$spec-to-goal-plan`, `$goal-loop-planner`, and `$goal-loop` skills. Their behavior is folded into one `$codex-goal-loop` skill.

## Required External Skills

| Skill | Used for | Model policy |
|---|---|---|
| `$review-and-simplify-changes` | Code-review replacement gate | Use model 5.4 for all review subagents |
| `$review-swarm` | Parallel regression, security, reliability, and contracts review gate | Use model 5.4 for all review-swarm subagents |

## Built-In Codex Surfaces

| Surface | Used for |
|---|---|
| Codex `update_plan` | Main-thread task tracking |
| Multi-agent tools, when available | Parallel implementation and gate-fix waves |
| Git CLI | Scoped staging, commits, status, and history checks |

## Model Policy

- Planner work: GPT-5.4 with high reasoning.
- Implementation and gate-fix subagents: model 5.5 with high reasoning.
- Review gates: model 5.4.

## Fallback Policy

If a required review skill is missing, stop and report it. If multi-agent tools are unavailable, preserve the same task decomposition and run the work serially in the main thread. Record that fallback in `IMPLEMENTATION_NOTES.md` and do not claim parallel subagent execution happened.

## Preflight Check

At Run Mode bootstrap:

1. Confirm the repo-local instructions that apply to the plan and touched files.
2. Confirm `$review-and-simplify-changes` and `$review-swarm` are available in the skill list or by path.
3. Check whether multi-agent tools are available with `tool_search` when parallel work or review is valuable.
4. Confirm git status and identify unrelated dirty files before implementation.
