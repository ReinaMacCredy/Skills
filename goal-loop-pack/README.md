# goal-loop-pack

Autonomous phase-by-phase plan execution for Claude Code. A single self-contained skill that drives a plan markdown file from "next phase" to "all phases shipped" without leaving the loop.

## What's inside

One skill, fully self-contained:

| Skill | Role |
|---|---|
| `goal-loop` | The main loop. Decomposes each phase via `Plan` subagent, dispatches parallel implementation waves through the inlined orchestrator pattern, gates every phase with `advisor()` + `/code-review --effort extra-high` + the inlined review-swarm protocol, commits, repeats. |

The orchestrator pattern and review-swarm protocol that goal-loop applies are inlined as reference files under `skills/goal-loop/references/`. There are no third-party skill dependencies.

Built-in dependencies (ship with Claude Code, not bundled):
- `Plan` subagent (Opus)
- `advisor()` tool
- `TaskCreate` / `Agent` tools
- `/code-review` skill

## Install

Extract this directory into a stable location, then symlink the skill into `~/.claude/skills/`:

```bash
# 1. Extract or clone to a stable location
git clone <repo> ~/Code/Skills/goal-loop-pack
# or: unzip goal-loop-pack.zip -d ~/Code/Skills/

# 2. Symlink the skill into your active skills directory
ln -s ~/Code/Skills/goal-loop-pack/skills/goal-loop ~/.claude/skills/goal-loop
```

The symlink keeps `~/Code/Skills/goal-loop-pack/` as the single source of truth — edit there, the live skill updates immediately, and zipping the directory produces a fresh shareable artifact in one step.

## Usage

```bash
# Once a plan markdown file exists:
/goal-loop ~/.claude/plans/your-plan.md

# Combined with the /goal hook (which blocks session stop until the goal holds):
/goal /goal-loop ~/.claude/plans/your-plan.md
```

See `skills/goal-loop/SKILL.md` for the full loop specification and `skills/goal-loop/references/` for protocol details on the orchestrator pattern, review-swarm protocol, Plan-subagent contract, advisor placement, gate-stuck clause, notes protocol, and dependency contract.

## Plan file format

`goal-loop` expects a markdown plan file with explicit phases. Each phase becomes one cycle of: Plan → Advisor → Ingest → Implement → CodeReview → ReviewSwarm → Commit. Pair with the `goal-distill` skill (separate install) to generate compatible plans from session context.

## Relationship to standalone orchestrator-only and review-swarm skills

If you also use `orchestrator-only` and/or `review-swarm` as standalone skills outside goal-loop, install them independently (they live in `~/Code/Skills/orchestrator-only/` and `~/Code/Skills/review-swarm/` as separate canonical sources). goal-loop does not depend on them — the protocols are inlined here as references so this bundle is self-contained.

## Sharing

This directory IS the shareable artifact. To ship a new version:

```bash
cd ~/Code/Skills
zip -r goal-loop-pack.zip goal-loop-pack/
```

Send the zip. Recipient unzips, runs the symlink step above, done.

## License

MIT
