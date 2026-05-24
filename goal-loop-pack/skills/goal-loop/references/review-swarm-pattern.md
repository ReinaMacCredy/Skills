# Review-Swarm Pattern

`goal-loop`'s REVIEW-SWARM GATE (step 6) follows this protocol. Four read-only reviewer sub-agents review the changed surface in parallel; the main thread filters, orders, and synthesizes. This reference documents the rules.

## When goal-loop applies this pattern

Step 6 only. Once per phase after step 5's `/code-review --effort extra-high` returns zero findings.

## Step 6.1: Determine review scope

Use the changed surface area from the current phase — the union of all `writes:` paths from the Plan output. If the phase committed before this gate, scope is `git diff <last-commit>..HEAD`.

Before launching reviewers, read closest local instructions for context:

- `AGENTS.md` in repo root + relevant subdirs
- Architecture or contract docs for the touched modules
- Project-specific style guides

Build a short **intent packet** for all four reviewers:

1. What behavior is meant to change (from the phase definition in `<PLAN_PATH>`)
2. What behavior should remain unchanged
3. Stated constraints (compatibility, rollout, security, migration expectations)

If intent is unclear, infer from the diff and label the inference as possibly incomplete.

## Step 6.2: Launch four read-only reviewer sub-agents in parallel

Single message, four `Agent` calls. Same scope and same intent packet for each. Read-only — sub-agents MUST NOT edit files, stage, commit, or mutate workspace state.

Each sub-agent reports findings in this shape:

- File and line (or nearest symbol)
- Issue description
- Why it matters (concrete impact, not speculation)
- Recommended follow-up
- Confidence: high / medium / low

Tell each sub-agent: avoid nits, style preferences, and speculative concerns without concrete impact. Better to miss a nit than bury the main thread in low-value noise.

### Reviewer 1 — Intent & Regression

Does the diff match the intended behavior change without introducing drift?

- Unintended behavior changes outside the stated scope
- Broken edge cases or fallback paths
- Contract drift between callers and callees
- Missing updates to adjacent flows that should change together

Sub-agent type: `general-purpose`

### Reviewer 2 — Security & Privacy

- Missing or weakened authn/authz checks
- Unsafe input handling, injection risks, validation gaps
- Secret/token/sensitive data exposure
- Risky defaults, permission expansion, trust of unverified data

Sub-agent type: `general-purpose`

### Reviewer 3 — Performance & Reliability

- Duplicate work, redundant I/O, unnecessary recomputation
- Added work on startup, render, request, or other hot paths
- Leaks, missing cleanup, retry storms, subscription drift
- Ordering, race, or failure-handling problems

Sub-agent type: `general-purpose`

### Reviewer 4 — Contracts & Coverage

- API, schema, type, config, or feature-flag mismatches
- Migration or backward-compatibility fallout
- Missing or weak tests for the changed behavior
- Missing logs, metrics, assertions, or error paths that would mask future regressions

Sub-agent type: `general-purpose`

## Step 6.3: Aggregate and filter

The main thread owns synthesis. Treat sub-agent output as raw review input, not final output.

Merge findings across all four. Filter aggressively:

- Drop duplicates
- Drop weak or speculative claims
- Drop issues that conflict with the stated intent
- Drop minor style/readability comments unless they hide a real bug

Normalize surviving findings:

1. File and line (or nearest symbol)
2. Category: regression / security / reliability / contracts
3. Severity: high / medium / low
4. Why it matters
5. Recommended fix or follow-up
6. Confidence

If a reviewer may be correct but intent is unclear, turn it into an open question rather than a finding.

## Step 6.4: Order findings

1. High-severity, high-confidence first
2. Medium-severity issues likely worth fixing before this phase commits
3. Lower-severity follow-ups

## Step 6.5: BLOCKING semantics

For `goal-loop`'s purposes:

- **High and medium severity findings BLOCK** the loop. Goal-loop cannot proceed to step 7 (COMMIT) until they are resolved.
- **Low severity findings are non-blocking nits**. Goal-loop may defer them, but only if the phase commit note explicitly mentions them.
- "No material issues" is a valid result. Say so directly; do not manufacture findings.

## Step 6.6: Fix dispatch

Goal-loop fixes blocking findings via the orchestrator pattern (see [orchestrator-pattern.md](orchestrator-pattern.md)). The shared fix rule applies to both step 5 and step 6:

- Group findings by file
- Disjoint groups → parallel fix sub-agents
- Overlapping paths → serialize
- Fix sub-agents append to `<NOTES>` if a fix forces a design change
- Re-run the gate until zero blocking findings

The STUCK CLAUSE (see [gate-stuck-clause.md](gate-stuck-clause.md)) applies if a 2nd fix pass surfaces a finding overlapping the 1st pass's surface area.
