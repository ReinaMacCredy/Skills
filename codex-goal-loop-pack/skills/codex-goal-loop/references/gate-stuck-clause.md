# Gate Stuck Clause

When a review-and-simplify or review-swarm gate iterates more than once on the same phase, insert a root-cause check before the 3rd fix attempt.

## Trigger Condition

Both must hold:

1. The same gate has run at least 2 times on this phase.
2. The 2nd run's findings include at least one finding whose changed surface overlaps the 1st run's findings.

## Surface Overlap Detection

A 2nd-pass finding overlaps a 1st-pass finding if any of these are true:

- It cites the same file and the line ranges intersect.
- It describes the same symptom class.
- The 1st-pass fix touched the same function or symbol the 2nd-pass finding criticizes.

If a 2nd-pass finding is in a different file with no thematic overlap, the clause does not trigger.

## Action When Triggered

Before the 3rd fix pass:

1. Run a main-thread root-cause check with both fix-attempt diffs in context.
2. Ask explicitly: "Are these fixes addressing root cause, or are we suppressing symptoms?"
3. Block only on a specific patch-pattern citation, for example a try/catch hiding an invariant bug, repeated optional chaining around the same upstream issue, or a test assertion changed instead of production behavior.
4. If the check blocks, surface the critique, abandon the symptom-fix approach, and re-plan the fix at the design level. This counts toward 3-strike.
5. If the check does not block, proceed with the 3rd pass.
