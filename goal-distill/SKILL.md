---
name: goal-distill
description: Distill the current session — conversation, decisions, corrections, repo state, and recent history — into a single paste-ready /goal prompt that drives an autonomous phase-by-phase loop (decompose → Plan-subagent on Opus → main-thread execute → re-measure → repeat) until the success criterion is met or a stop condition fires. Universal — works in any repo, writes no files. Copies the prompt to the clipboard on macOS. Use this whenever the user says "distill goal", "distill this session", "goal-distill", "reconstruct goal", "rebuild the spec", "rebuild the masterplan", "give me a /goal", "phase-by-phase to the goal", "loop until done", "iterate toward the goal", or "turn this into a /goal prompt I can paste later" — and prefer it even when the user doesn't name the skill but clearly wants this session compressed into a runnable prompt for a future agent.
---

# Goal Distill

One deliverable: a self-contained `/goal` prompt the user can paste into a fresh session to drive an autonomous phase loop. **No spec file, no `.goals/` directory, no on-disk artifact.** The output is the prompt and a clipboard copy.

## Why this exists

`goal-distill` runs *inside* a working session with rich context — conversation, decisions, corrections, repo state. That context evaporates when the session ends. The skill compresses it into a paste-ready `/goal` prompt the user (or a future agent) can run later without needing the original conversation. The prompt itself carries the loop instructions, so the future agent can execute end-to-end with no further prompting.

## Workflow

1. **Read evidence (silent)** — in this order:
   - Conversation history: corrections and repeated preferences are authoritative; distinguish exploratory ideas from finalized decisions; capture *why*, not just *what*.
   - Repository: `git status`, key configs, recent file diffs.
   - Recent history: `git log -20`, surfacing TODO/FIXME comments touched lately.
   - Docs: README, AGENTS/CLAUDE files, ADRs, ROADMAP, CHANGELOG — only those that exist.

2. **Reconstruct intent (mental model, not dumped output)**:
   - The problem actually being solved (not the README framing).
   - Key decisions and the *why* behind them.
   - Architectural and product philosophy.
   - Constraints, invariants, trust boundaries.
   - Current gaps between intent and code.
   - A single verifiable success criterion.

3. **Emit the `/goal` prompt** — see the structure below. Self-contained, ≤4000 chars, paste-ready. The future runtime agent reads only the prompt; do not reference files that may not be present.

   **Shape switch:** if the session has a real plan, spec, or brainstorm to draw from, emit the full prompt with the **Phase loop** section (multi-phase, autonomous). If the session is a one-shot task (no plan, no spec, no brainstorm — just "do this thing"), omit the Phase loop section entirely and produce a single-action `/goal`: identity, mission, exit check, hard invariants, pointers. Don't fabricate phases that don't exist.

4. **Copy to clipboard (best effort)** — pipe the prompt through `pbcopy` on macOS. If unavailable, say so and move on. Never block.

5. **Output to the user (terse)**:
   - One short paragraph of the reconstructed intent so the user can sanity-check.
   - The `/goal` prompt in a fenced ```text block.
   - One line: `(copied to clipboard)` or `(pbcopy unavailable — copy manually)`.

   Do not print the full 8-section analysis. The intent paragraph is the only narrative output.

## /goal prompt structure (≤4000 chars, paste-ready)

The prompt must let the runtime agent take a concrete first action with no further input. It is self-contained: do not reference external files unless they are part of the repo and load-on-demand. Sections in this order:

1. **Identity** — one line: repo name, branch, stack.
2. **Mission** — one verifiable sentence that names the change.
3. **Success criterion** — what "done" objectively looks like.
4. **Measurement** — pick one:
   - *Numeric goal:* baseline command (with literal command string), the current numbers, and a verdict rule (which numbers must drop, which must not regress, by how much).
   - *Build/ship goal:* a short exit checklist (3–7 items max).
5. **Phase loop** *(include only if the session carries a real plan/spec/brainstorm; for one-shot tasks, skip this section entirely and let the /goal be a single-action directive)* — embed verbatim:

   ```
   Loop until success criterion OR stop condition:
   a. Decompose → next smallest vertical slice {name, scope, exit_check}.
   b. Plan     → Agent(subagent_type="Plan", model="opus"), pass {scope, exit_check,
                 criterion, rule|checklist, stops, pointers}. No human approval.
   c. Execute  → in-thread Read/Edit/Bash; verify per step; small commits.
   d. Measure  → numeric: delta + verdict (better|same|worse).
                 checklist: each item pass|fail + 1-line evidence.
   e. Gate     → success → done.  worse | 3× same/worse | tests red | user halt → stop.
   f. Commit   → 1 logical change; numbers/evidence in body.  Goto (a).
   ```

6. **Hard invariants** — project-specific rules, derived from session: re-read before destructive moves. Examples: "don't bypass --no-verify", "match existing project style", "deletions require itemized evidence".
7. **Stop conditions** — explicit bullets (regression, repeated no-improvement, test failures, user halt).
8. **Pointers** — `path:line` references the runtime agent should load on demand only (not eagerly).

### Self-test before printing

Read the draft as if you were the runtime agent seeing it cold. Can you take a concrete first action? If the natural response would be "let me orient myself and ask what next", the prompt is broken — rewrite step (a) with a more concrete entry point and re-test.

## Clipboard copy

After printing the fenced block, attempt:

```bash
printf '%s' "<prompt-contents>" | pbcopy 2>/dev/null \
  && echo "(copied to clipboard)" \
  || echo "(pbcopy unavailable — copy manually)"
```

Do not block on failure. Linux/Windows users: skip the copy step silently and tell them to copy by hand.

## Anti-patterns


- **Asking the user for per-phase approval inside the `/goal` prompt.** The loop is autonomous; only stop conditions interrupt it.
- **Calling the Plan subagent on a model other than Opus.** The user asked for the highest model — honor it.
- **Emitting an orientation-style `/goal`** ("first, explore the codebase..."). It must drive a concrete first action.
- **Bloating past 4000 chars.** If it doesn't fit, compress the prose and pointer list. Do not extend the cap.
- **Dumping the full 8-section analysis to the user.** One paragraph of reconstructed intent is enough; the `/goal` prompt itself carries everything the runtime agent needs.
