# Review-Swarm Gate

Use `$review-swarm` for this gate.

## Model Policy

Run all review-swarm subagents on model 5.4.

## Scope

Review only the current phase's changed surface:

- Before commit: union of all phase `writes` paths plus any files changed by fixes.
- After commit: `git diff <phase-start>..HEAD` or the narrowest equivalent range.

Read closest `AGENTS.md` files and relevant architecture or contract docs before review.

## Intent Packet

Give `$review-swarm` the same short packet:

1. Intended behavior change.
2. Behavior that must remain unchanged.
3. Constraints from the plan, repo instructions, and user messages.
4. Touched paths and verification already run.

## Blocking Semantics

- High and medium findings block the loop.
- Low findings may be deferred only if the commit note or final summary mentions them.
- "No material issues" is valid. Do not manufacture findings.

## Fix Dispatch

Group blocking findings by path:

- Disjoint paths can be fixed in parallel by implementation subagents on model 5.5 high.
- Overlapping paths serialize.
- Re-run `$review-swarm` until zero blocking findings.
- Apply [gate-stuck-clause.md](gate-stuck-clause.md) before a third fix pass on the same overlapping surface.
