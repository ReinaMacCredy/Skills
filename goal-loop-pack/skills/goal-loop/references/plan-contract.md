# Plan Subagent Output Contract

The `Plan` subagent (`model: "opus"`) MUST return a structure with the shape below. The main thread refuses to dispatch on any invariant violation.

## Required fields per task

| Field | Type | Description |
|---|---|---|
| `id` | string | Unique within this phase (e.g. `t-001`, `t-002`) |
| `action` | string | One discrete unit: file create/edit, test, or command |
| `writes` | string[] | Every file path the task creates or modifies. Absolute or repo-relative. |
| `depends_on` | string[] | Task ids that must complete first. Empty if independent. |

## Required wave structure

| Field | Type | Description |
|---|---|---|
| `waves` | string[][] | Topological grouping. Each inner array is one wave of mutually-independent tasks. |

## Invariants verified before dispatch

1. Same-wave tasks share NO `writes` path.
2. Same-wave tasks have NO `depends_on` pointing inside the same wave.
3. Every task id in `waves` appears in the task list exactly once.
4. Every task id in the task list appears in `waves` exactly once.

Any violation → reject and re-Plan (counts toward 3-strike on the Plan step).

## Example

```json
{
  "phase": "Phase 2 — Add session storage",
  "tasks": [
    {"id": "t-001", "action": "Add SessionStore interface",  "writes": ["src/session/store.ts"],         "depends_on": []},
    {"id": "t-002", "action": "Add InMemorySessionStore",    "writes": ["src/session/in-memory.ts"],     "depends_on": ["t-001"]},
    {"id": "t-003", "action": "Add SessionStore unit tests", "writes": ["tests/session/store.test.ts"],  "depends_on": ["t-001"]},
    {"id": "t-004", "action": "Wire SessionStore into App",  "writes": ["src/app.ts"],                   "depends_on": ["t-002"]}
  ],
  "waves": [["t-001"], ["t-002", "t-003"], ["t-004"]]
}
```

In this example:
- Wave 1: `t-001` alone (interface must exist before anything else).
- Wave 2: `t-002` and `t-003` in parallel (different files, both depend only on `t-001`).
- Wave 3: `t-004` alone (depends on the impl).

## Why this shape

The dependency graph is the **source of truth for parallelism**. Without `writes:` declared per task, the main thread can't mechanically detect file-write races. Without `depends_on:`, the wave grouping is just a hint, not a contract. The Plan subagent owns both.
