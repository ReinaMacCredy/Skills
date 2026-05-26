# Phase Plan Output Contract

The phase planning step must return a structure with the shape below. Use GPT-5.4 with high reasoning for planner work. Refuse to dispatch on any invariant violation.

## Required Fields Per Task

| Field | Type | Description |
|---|---|---|
| `id` | string | Unique within this phase, for example `t-001` |
| `action` | string | One discrete unit: file create/edit, test, or command |
| `writes` | string[] | Every file path the task creates or modifies |
| `depends_on` | string[] | Task ids that must complete first, or empty if independent |

## Required Wave Structure

| Field | Type | Description |
|---|---|---|
| `waves` | string[][] | Topological grouping. Each inner array is one wave of mutually independent tasks |

## Invariants Verified Before Dispatch

1. Same-wave tasks share no `writes` path.
2. Same-wave tasks have no `depends_on` pointing inside the same wave.
3. Every task id in `waves` appears in the task list exactly once.
4. Every task id in the task list appears in `waves` exactly once.

Any violation means reject and re-plan. This counts toward 3-strike on the Plan step.

## Example

```json
{
  "phase": "Phase 2 - Add session storage",
  "tasks": [
    {"id": "t-001", "action": "Add SessionStore interface", "writes": ["src/session/store.ts"], "depends_on": []},
    {"id": "t-002", "action": "Add InMemorySessionStore", "writes": ["src/session/in-memory.ts"], "depends_on": ["t-001"]},
    {"id": "t-003", "action": "Add SessionStore unit tests", "writes": ["tests/session/store.test.ts"], "depends_on": ["t-001"]},
    {"id": "t-004", "action": "Wire SessionStore into App", "writes": ["src/app.ts"], "depends_on": ["t-002"]}
  ],
  "waves": [["t-001"], ["t-002", "t-003"], ["t-004"]]
}
```

The dependency graph is the source of truth for parallelism. Without `writes`, the main thread cannot mechanically detect file-write races. Without `depends_on`, wave grouping is only a hint, not a contract.
