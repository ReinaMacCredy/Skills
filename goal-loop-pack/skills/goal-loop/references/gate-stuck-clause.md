# Gate Stuck Clause

When a code-review or review-swarm gate iterates more than once on the same phase, you risk **patching symptoms instead of fixing root cause**. This clause inserts an `advisor()` consultation before the 3rd fix attempt.

## Trigger condition

Both must hold:

1. The same gate (`/code-review --effort extra-high` OR the review-swarm protocol) has run at least 2 times on this phase.
2. The 2nd run's findings include at least one finding whose **changed surface area overlaps** the 1st run's findings.

## Surface overlap detection

A 2nd-pass finding overlaps a 1st-pass finding if ANY of:

- It cites the **same file** AND the line ranges intersect.
- It describes the **same symptom class** — e.g., "null pointer in X" after a 1st-pass "null check missing in X".
- The 1st-pass fix touched the same function/symbol the 2nd-pass finding criticizes.

If a 2nd-pass finding is in a completely different file with no thematic overlap, the clause does NOT trigger — that's normal iteration.

## Action when triggered

Before the 3rd fix pass:

1. Call `advisor()` with **both fix-attempt diffs in context** (1st-pass fix diff + 2nd-pass fix diff).
2. Frame the question explicitly: *"Are these fixes addressing root cause, or are we suppressing symptoms?"*
3. Advisor blocks only on a specific patch-pattern citation, e.g.:
   - "The 2nd patch wraps the bug in a try/catch instead of fixing the invariant."
   - "Both patches added optional chains; the actual issue is that the upstream value can be undefined."
   - "The fix changed the test assertion instead of the production code."
4. If advisor blocks: surface the patch-pattern critique to the main thread, abandon the symptom-fix approach, and re-plan the fix **at the design level** (counts toward 3-strike).
5. If advisor doesn't block: proceed with the 3rd pass.

## Why this is the right time

| Pass | Status | Action |
|---|---|---|
| 1st | Initial gate output | Fix normally |
| 2nd | Re-run after fix | Normal hygiene — gate may have found follow-on issues |
| 3rd | Pattern: gate keeps finding things in same surface | Time for stronger reviewer |
| 4th+ | 3-strike counter fires; STOP and surface | — |

Beyond the 3rd pass, the per-phase-per-step 3-strike counter takes over.

## Cost note

Advisor calls are expensive (transcript re-read, cache miss). The STUCK CLAUSE is intentionally self-limiting: it fires at most **once per gate per phase** in a clean-ish run, and not at all on phases that pass the first or second pass cleanly.
