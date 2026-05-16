---
name: bug-hunt-swarm
description: "Parallel read-only multi-agent root-cause investigation for bugs, regressions, crashes, flaky behavior, or unexplained failures. Use when the user asks to investigate a bug, find the root cause, trace a regression, understand why something broke, or wants a ranked diagnosis with the fastest proof path without making code edits."
---

# Bug Hunt Swarm

Investigate a bug with four read-only sub-agents in parallel, then have the main agent rank the likely causes and recommend the fastest path to prove or fix the issue. This skill is diagnosis-first: do not edit files or implement fixes as part of this workflow.

## Step 1: Build the Bug Packet

Start by collecting the smallest useful investigation packet:

1. Symptom
2. Expected behavior
3. Actual behavior
4. Reproduction steps, if known
5. Scope of impact
6. Relevant evidence, such as logs, stack traces, failing tests, screenshots, recent diffs, or environment details

Prefer this source order:

1. Direct user description
2. Explicit files, stack traces, logs, tests, or screenshots provided by the user
3. Current git changes or recent repo history when the bug appears regression-like
4. The smallest relevant code path or subsystem surrounding the failure

If the bug report is underspecified, infer a minimal problem statement and say what is still unknown.

Before launching sub-agents, read the closest project instructions and relevant docs for the touched area, such as:

- `AGENTS.md`
- repo workflow docs
- architecture, state, routing, schema, or runtime docs for the affected subsystem

## Step 2: Bound the Investigation

Write a short investigation brief for the swarm:

1. What appears broken
2. What is not yet proven
3. What part of the system is most likely involved
4. What evidence already exists
5. What kind of proof would count as confirmation

Use read-only evidence gathering where useful:

- `rg`, `git diff`, `git log`, `git show`
- reading logs, crash traces, and config
- existing test runs or the smallest safe reproduction command

Do not edit files, inject new instrumentation, or implement fixes as part of this skill.

## Step 3: Launch Four Read-Only Investigators in Parallel

Launch four sub-agents when the problem is large or ambiguous enough that parallel investigation helps. For a tiny and obvious issue, it is acceptable to investigate locally instead.

For every sub-agent:

- give the same bug packet and investigation brief
- state that the sub-agent is read-only
- do not let the sub-agent edit files, run `apply_patch`, stage changes, commit, or perform any other state-mutating action
- ask for concise investigation output only
- ask for: hypothesis, supporting evidence, missing evidence, smallest proof step, and confidence
- tell the sub-agent to avoid generic code quality feedback, nits, or speculative guesses without evidence
- tell the sub-agent to send findings back to the main agent only

Use these four investigation roles.

### Sub-Agent 1: Reproduction and Scope Investigation

Clarify the exact failure shape and its boundaries.

Check for:

1. The narrowest reliable trigger
2. Conditions that make the bug appear or disappear
3. Expected versus actual behavior at the failure boundary
4. Whether the impact is local, cross-cutting, deterministic, or flaky

This sub-agent is read-only. It must not edit files, apply patches, or make any other workspace changes.

Recommended sub-agent type: `general-purpose`

### Sub-Agent 2: Code Path and Failure Seam Investigation

Trace the most likely execution path and identify the seam where behavior diverges.

Check for:

1. State transitions, lifecycle edges, or ordering problems
2. Mismatched assumptions between caller and callee
3. Data-flow or control-flow breaks
4. The smallest code region most likely responsible for the failure

This sub-agent is read-only. It must not edit files, apply patches, or make any other workspace changes.

Recommended sub-agent type: `Explore` for broad tracing, or `general-purpose` when a stronger local reasoning pass is more useful

### Sub-Agent 3: Recent Change and Regression Investigation

Look for likely regressors in nearby history or changed contracts.

Check for:

1. Recent diffs that correlate with the symptom
2. Config, flag, dependency, schema, or migration drift
3. Partial updates where several entry points should have changed together
4. Behavior changes that fit the timing of the bug report

This sub-agent is read-only. It must not edit files, apply patches, or make any other workspace changes.

Recommended sub-agent type: `general-purpose`

### Sub-Agent 4: Proof Plan and Observability Investigation

Determine the fastest way to confirm or reject the leading hypotheses.

Check for:

1. The smallest existing test or reproduction that should fail
2. The most useful current logs, traces, metrics, or assertions
3. A minimal non-mutating command that could raise confidence quickly
4. What evidence is missing and how to collect it without broad churn

This sub-agent is read-only. It must not edit files, apply patches, or make any other workspace changes.

Recommended sub-agent type: `general-purpose`

Report only hypotheses that materially improve the odds of finding the real cause. It is better to return two evidence-backed theories than six vague guesses.

## Step 4: Synthesize Ranked Hypotheses

The main agent owns synthesis. Treat sub-agent output as raw investigation input, not final output.

Merge and rank the hypotheses:

- combine duplicates
- discard weak speculation
- prefer evidence over elegance
- separate likely root causes from mere contributing factors
- keep alternate theories only when they remain plausible

Normalize the surviving hypotheses into this shape:

1. Hypothesis
2. Supporting evidence
3. Missing or conflicting evidence
4. Smallest proof step
5. Confidence: high, medium, or low

If the evidence is too weak for a real ranking, say so directly and present the leading open questions instead.

## Step 5: Main Agent Verifies the Leading Hypothesis Against the Real Code

Sub-agents can hallucinate: invented file paths, misremembered function names, wrong line numbers, plausible-sounding but non-existent symbols, fabricated call chains. Before declaring a diagnosis final, the main agent must independently confirm the leading hypothesis is real by inspecting the actual code and artifacts. Do not trust the sub-agent's account on its own.

This step does not run tests, reproductions, or any commands beyond read-only inspection. It is a citation check.

For the leading hypothesis, the main agent reads the source directly and confirms each load-bearing claim:

1. **Files exist at the cited paths.** Open them with `Read`. If a sub-agent cited a path that does not exist, the claim is hallucinated.
2. **Symbols exist where claimed.** Use `rg` or `Read` to confirm the named function, variable, type, config key, route, or migration is present in the cited file and roughly at the cited location.
3. **Quoted code matches the file.** If a sub-agent quoted or paraphrased code, confirm the real file says what the hypothesis depends on. Watch for invented signatures, wrong argument order, or behavior attributed to code that does not actually do that.
4. **The causal chain is mechanically possible.** Read the surrounding code at the suspected seam and confirm the failure mode the hypothesis describes can actually arise from what is written there. If the chain skips a step that the real code blocks, the hypothesis is wrong.
5. **Cited diffs, commits, or regressors are real.** If a sub-agent blamed a recent change, confirm with `git log` or `git show` that the commit and the cited lines exist.
6. **No surviving evidence from the packet contradicts the hypothesis** once the code is read directly.

Outcomes:

- **Verified.** Every load-bearing citation checks out against the real code. Mark the hypothesis as the verified root cause and proceed to Step 6.
- **Partially verified.** Some citations check out, others do not, or the causal chain has a gap once the real code is read. Do not declare final. Strip the unsupported parts, demote confidence, and either promote the next-best hypothesis for verification or return to Step 4 with the corrected evidence.
- **Hallucinated or contradicted.** Cited files, symbols, or behavior do not exist as described. Discard the hypothesis, record what was fabricated so it is not re-used, and verify the next candidate. If no candidate survives independent verification, say so directly and present the strongest remaining open questions instead of forcing a conclusion.

Never promote a sub-agent's conclusion to "final root cause" without the main agent reading the cited code itself.

## Step 6: Output a Clear Diagnosis Path

Present the result in this order:

1. Verified root cause (only if Step 5 confirmed every load-bearing citation against the real code) or leading unverified hypothesis with what is still missing
2. The citations the main agent personally checked (file paths, symbols, commits) and what each confirmed
3. Plausible alternate causes, if any
4. Recommended fix path
5. Open questions or blockers

When verification did not confirm the hypothesis end-to-end, label the diagnosis as unverified and recommend the next proving step instead of pretending the diagnosis is complete.

When helpful, group actions into:

- `prove now`
- `fix next`
- `follow up later`

Do not implement fixes as part of this skill. The output is a read-only diagnosis with a prioritized path forward.
