---
name: systematic-fix
description: >-
  Disciplined bug-fix-and-lock-down workflow: catalog every issue, fix each one
  with verification, then create comprehensive e2e integration tests that would
  have caught each fixed issue plus similar failure modes. Use this skill whenever
  the user has just identified multiple bugs or problems to fix, wants to fix
  everything systematically without losing track, or wants regression test coverage
  after fixes. Trigger on phrases like "fix all of that", "fix absolutely everything",
  "keep a TODO list of all items", "make sure this can't happen again", "write tests
  for what we just fixed", "add regression tests", "lock down these fixes", "create
  a test safety net", "write e2e tests for these fixes", "don't lose track of
  anything", or any request that combines thorough bug-fixing with regression
  prevention. Also trigger when the user asks to "fix and verify" multiple issues,
  wants granular tracking of fix progress, or says things like "I need you to fix
  each of those problems and then write tests so this never happens again".
---

# Fix, Verify, and Lock Down

This skill covers the full lifecycle: from a pile of known issues to a codebase
where every fix is proven and guarded by tests. The three phases build on each
other -- skipping straight to tests without disciplined fixing and verification
is how regressions slip through.

## Phase 1: Catalog Every Issue

Before touching any code, build a complete inventory. This is the tracking
backbone that prevents issues from falling through the cracks.

### How to catalog

Review the conversation, error logs, code review comments, or whatever source
identified the problems. For each issue, capture:

- **ID**: a short label (e.g., `FIX-1`, `FIX-2`) for tracking
- **What's broken**: the observable symptom (error message, wrong output, crash)
- **Where**: file(s) and line(s) involved
- **Severity**: does it block other fixes? Is it a data-loss risk?
- **Root cause** (if known yet): why it's broken, not just what's broken

Use a task list or structured tracking to manage these. Every item must be
individually trackable -- no "fix the validation stuff" umbrella items. Each
distinct problem gets its own entry.

### Ordering

Sort by dependencies first (if fix B depends on fix A, do A first), then by
severity. Data-loss and crash bugs before cosmetic issues. If two fixes touch
the same code, do them together to avoid merge pain.

## Phase 2: Fix and Verify (One at a Time)

This is where discipline matters most. The rule is simple: **one fix at a time,
verified before moving on.** Batching fixes and verifying later is how you end
up with "I fixed 8 things, 2 of them broke something else, and now I can't tell
which."

### For each issue:

1. **Understand the root cause.** Read the relevant code. If the root cause
   wasn't clear from Phase 1, dig until you find it. A fix aimed at the symptom
   instead of the cause will break again.

2. **Make the smallest fix that addresses the root cause.** Don't refactor
   neighboring code, don't "improve" things while you're in there. Scope creep
   in a fix session is how new bugs get introduced.

3. **Verify the fix works.**
   - Run the specific test or reproduction case that demonstrates the issue is
     resolved.
   - Run the broader test suite to confirm nothing else broke.
   - If there's no existing test for this case, manually verify the behavior
     and note that a test is needed (Phase 3 will handle it).

4. **Update your tracking.** Mark the item as done with a short note on what
   you changed and how you verified it. This record feeds directly into Phase 3.

5. **Move to the next item.** Don't go back and tweak previous fixes unless a
   later fix reveals a problem. If it does, that's a new tracking item.

### What "verified" means

"It looks right" is not verified. Verified means one of:

- A test passes that previously failed (or would have failed)
- You ran a command and showed the output proving correct behavior
- You demonstrated the error no longer occurs with the same input that triggered it

If you can't verify a fix right now (e.g., it requires a running server you don't
have), document exactly what verification is pending and flag it.

### When a fix reveals new issues

It happens. You fix a null check and discover the function has a separate
off-by-one error. Don't fix it inline -- add it to your tracking list as a new
item. This keeps your current fix atomic and your tracking complete.

## Phase 3: Regression Tests

Now every issue is fixed and verified. This phase creates the permanent safety
net: tests that will catch these bugs (and their relatives) if they ever try to
come back.

### 3a. Categorize the fixes

Group your completed fixes by failure category. Common categories:

- **Input validation**: missing/wrong checks on user input, API payloads, config values
- **State management**: stale state, race conditions, incorrect initialization
- **Boundary conditions**: off-by-one, empty collections, max values, integer overflow
- **Data flow**: wrong data passed between components, incorrect transformations
- **API contracts**: mismatched expectations between caller and callee
- **Error handling**: swallowed exceptions, missing error paths, wrong error types
- **Concurrency**: race conditions, deadlocks, ordering assumptions

This categorization drives the "similar issues" coverage. Each category tells
you where else in the codebase the same class of bug could be hiding.

### 3b. Design tests for each fix

For every fixed issue, write at least three kinds of tests:

**Exact reproduction test.** A test that exercises the precise code path that
broke, with the precise input (or conditions) that triggered it. This test
would have failed before your fix and passes now. This is the direct regression
guard.

**Boundary variants.** Tests that probe the edges around the fix. If the bug
was an off-by-one on array index 0, test index -1, the last index, one past
the last, and an empty array. If it was a missing null check, test null,
undefined, empty string, and valid input. The goal is to stress the area where
the fix lives.

**Category siblings.** Look at other places in the codebase where the same
*category* of bug could occur. If you fixed a missing input validation on one
API endpoint, check the other endpoints. If you fixed a race condition in one
event handler, check the similar handlers. You're not just testing the bug you
found -- you're testing the bug *family*.

### 3c. Write the tests

Follow the project's existing test framework and conventions. Match the style,
directory structure, naming patterns, and assertion library of existing tests.

Key principles:

- **Test real code paths.** These are integration/e2e tests, not unit tests with
  mocks. The bug happened in real code, so test real code. If the project has
  both unit and integration tests, put regression tests at the integration level.

- **Descriptive names.** The test name should explain what regression it prevents.
  `it("rejects negative quantities instead of silently truncating to zero")` tells
  you exactly what went wrong and what the test guards. `it("test quantity fix")`
  tells you nothing.

- **Link to the fix.** Add a brief comment explaining what issue this test covers
  so future readers understand the "why." Something like:
  `// Regression: FIX-3 -- negative quantities were silently accepted`

- **Group by category.** Put related regression tests in a clearly labeled
  `describe` block (e.g., `describe("regression: input validation")`) so they're
  easy to find and extend.

- **No test bloat.** Don't add tests for things already well-covered by the
  existing suite. Focus on the gaps the bugs revealed. If an existing test
  *almost* covers the case, extend it rather than adding a near-duplicate.

### 3d. Verify the tests themselves

Tests that always pass are useless. For each new test:

- **Confirm it passes** with the fix in place.
- **Reason through (or demonstrate) that it would fail** without the fix. The
  gold standard is temporarily reverting the fix and watching the test fail,
  then re-applying. If that's impractical, at minimum explain *which line of
  the fix* makes this test pass and *what would happen* without it.
- **Run the full suite** to make sure new tests don't conflict with existing ones.

### 3e. Coverage check

After all regression tests are written, do a final review:

- Every fixed issue from Phase 1 has at least one direct regression test
- Each failure category has boundary and sibling coverage
- No tracking items were skipped or forgotten
- The test suite passes cleanly

Report the final state: how many issues were fixed, how many tests were added,
and what categories they cover.

## Principles

- **Track everything, lose nothing.** The tracking list is the source of truth.
  Every issue gets an entry, every entry gets resolved.
- **One at a time.** Fix, verify, move on. Don't batch.
- **Prove it works.** "Seems right" is not done. Show what you ran and what it
  produced.
- **Test the category, not just the instance.** Every bug is a data point about
  a class of problems. Use it to hunt the siblings.
- **Real code paths.** Integration tests that hit actual code. The bug didn't
  happen in a mock.
- **Depth over breadth.** A few thorough tests that genuinely stress the failure
  mode beat many shallow ones that only cover the happy path.
