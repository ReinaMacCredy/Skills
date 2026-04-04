---
name: blueprint
description: Generate beautiful, self-contained HTML blueprint pages that visually plan implementations with architecture diagrams, phased task breakdowns, file change maps, dependency graphs, risk matrices, and verification checklists. Use this skill whenever the user asks for a blueprint, implementation plan, feature spec, architecture plan, project plan, or says things like "plan this out visually", "blueprint this feature", "give me a full plan", "create a spec for X", or "how should I build X". Also use proactively when the user describes a non-trivial feature or change that would benefit from structured planning before implementation -- especially multi-file changes, architectural decisions, or cross-cutting concerns. Even if the user doesn't say "blueprint", if the work is complex enough to warrant a plan, offer to generate one.
---

# Blueprint

Generate two artifacts for every blueprint:

1. **Plan spec** (`.md`) -- the executable plan in structured markdown, saved to `~/.claude/blueprint/`
2. **Visual blueprint** (`.html`) -- the interactive HTML presentation, saved to `~/.agent/diagrams/`

The `.md` file is the working document you hand to an agent or follow yourself. The `.html` is the visual version of the same content. Both files share a matching name (e.g., `ws-events-blueprint.md` + `ws-events-blueprint.html`).

## Workflow

### 1. Explore (Subagents)

Before generating anything, understand the codebase. Launch parallel explore subagents to investigate the areas relevant to the user's request.

**When to use subagents:**
- 1 agent: isolated change, user gave specific file paths
- 2-3 agents: multiple areas involved, need to understand existing patterns
- Each agent gets a focused objective: "find existing auth middleware and trace its usage", not "explore the codebase"

**What to look for:**
- Existing implementations that can be reused
- Patterns and conventions the codebase follows
- Files that will need modification
- Type definitions, interfaces, and data models involved
- Test patterns used in the project
- Dependencies and integration points

### 2. Synthesize and Decide

After subagents report back, synthesize their findings. This is where you determine:

**Depth level** -- auto-detect based on the request, override if the user says "full blueprint" or "quick blueprint":

| Level | When | Sections included |
|---|---|---|
| **Light** | 1-2 files, obvious change, single concern | Summary, File Changes, Verification |
| **Standard** | Feature spanning several files, clear scope | Summary, Architecture, Phases, File Changes, Testing, Verification |
| **Full** | System redesign, cross-cutting, multi-sprint, architectural | All 7 sections at max depth |

**Structural forks** -- if you found something during exploration that creates a genuine fork in the blueprint (e.g., "there are 3 existing queue implementations -- consolidate or add new?"), ask the user NOW. One or two targeted questions max. If the path is clear, skip straight to generation.

Questions should reference what you actually found: file paths, code patterns, existing implementations. Never ask generic questions you could answer by reading code.

### 3. Generate the Blueprint

**Read the reference template** at `./templates/blueprint-full.html` before generating. Don't memorize it -- read it each time to absorb the patterns and structure.

**For CSS/layout patterns**, read `./references/css-patterns.md`. This has blueprint-specific components: phase cards, timeline, risk matrix, file change indicators, KPI summary cards, and dependency graph styling.

**For Mermaid diagrams and external libraries**, read `./references/libraries.md`.

**For pages with 4+ sections** (standard and full blueprints), read `./references/responsive-nav.md` for sticky sidebar TOC on desktop and horizontal scrollable bar on mobile.

**For detailed guidance on each section's content**, read `./references/sections-guide.md`.

### 4. Structure the HTML

Every blueprint is a single self-contained `.html` file. The sections adapt based on depth level.

**Full blueprint sections (7):**

1. **Executive Summary** -- KPI cards (scope, effort, risk level, timeline), one-paragraph problem statement
2. **Architecture Diagrams** -- Mermaid: system topology, data flow, sequence diagrams as needed
3. **Phased Implementation** -- Timeline view with phases, milestones, and duration estimates
4. **Per-Phase Details** -- Collapsible sections: file changes, task cards, API contracts, test plans per phase
5. **Dependency Graph** -- Mermaid DAG showing task ordering, critical path highlighted
6. **Risk Matrix** -- Impact vs likelihood table with mitigation strategies
7. **Verification Checklist** -- Commands to run, expected outcomes, acceptance criteria

**Standard blueprint sections (5):**
Sections 1, 2, 3, 4, 7 -- skip dependency graph and risk matrix.

**Light blueprint sections (3):**
Sections 1 (compact), 4 (single phase, no collapsibles), 7.

### 5. Style

Apply visual-explainer quality aesthetics to every blueprint:

**Typography.** Pick a distinctive font pairing from Google Fonts. A display/heading font with character, plus a mono font for technical labels. Never use Inter, Roboto, Arial, or system-ui as the primary font. Load via `<link>` in `<head>`. Include a system font fallback.

**Color tells a story.** Use CSS custom properties for the full palette. Define at minimum: `--bg`, `--surface`, `--border`, `--text`, `--text-dim`, and accent colors. Use semantic naming where possible (`--phase-active` not `--blue-3`). Support both light and dark themes via `prefers-color-scheme`.

**Surfaces whisper.** Build depth through subtle lightness shifts (2-4% between levels), not dramatic color changes. Borders should be low-opacity rgba.

**Background atmosphere.** Subtle gradients, faint grid patterns, or gentle radial glows. The background should feel like a space, not a void.

**Visual weight signals importance.** Executive summary and KPI cards dominate the viewport on load. Per-phase details and risk matrix are compact, use `<details>/<summary>` for drill-down. Not every section deserves equal visual treatment.

**Animation earns its place.** Staggered fade-ins on load guide the eye. Mix animation types: `fadeUp` for section cards, `fadeScale` for KPIs, `drawIn` for Mermaid connectors. Always respect `prefers-reduced-motion`.

**Vary the aesthetic.** Don't default to "dark theme with blue accents" every time. Rotate palettes and font pairings. The swap test: if you replaced your styling with a generic dark theme and nobody noticed, you haven't designed anything.

### 6. Write the Plan Spec (.md)

Before generating the HTML, write the structured markdown plan spec. Save to `~/.claude/blueprint/{feature-name}-blueprint.md`.

**Plan spec structure:**

```markdown
# Blueprint: Feature Name

> One-line summary of the change and why it matters.

## Context

What exists today, what's broken/missing, what the world looks like after.

## Scope

- **Files:** N files changed (N new, N modified, N deleted)
- **Phases:** N phases
- **Estimated LOC:** ~N
- **Risk:** Low / Medium / High

## Architecture

Description of system changes. Include mermaid diagram in fenced code block:

\`\`\`mermaid
graph TD
  A --> B
\`\`\`

## Phases

### Phase 1: Phase Name (~duration)

**Delivers:** What this phase produces.

#### File Changes

| Op | File | Reason |
|----|------|--------|
| + | `src/new-file.ts` | Description |
| ~ | `src/existing.ts` | Description |
| - | `src/old-file.ts` | Description |

#### Tasks

1. **Task name** -- Description. _Acceptance: what proves this works._
2. **Task name** -- Description. _Acceptance: criteria._

#### Test Plan

- Unit: what to test
- Integration: what to test

### Phase 2: ...

## Dependencies

\`\`\`mermaid
graph TD
  T1 --> T3
  T2 --> T3
\`\`\`

Tasks that can run in parallel: T1 and T2.

## Risks

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| What could happen | High/Med/Low | High/Med/Low | What to do |

## Verification

- [ ] Build: `command` -- expected outcome
- [ ] Tests: `command` -- expected outcome
- [ ] Feature: manual step -- expected outcome
```

### 7. Generate the Visual Blueprint (.html)

Now generate the HTML version. Same content, visual presentation.

**Output location:** Write HTML to `~/.agent/diagrams/{feature-name}-blueprint.html`.

**Open in browser:**
- macOS: `open ~/.agent/diagrams/filename.html`
- Linux: `xdg-open ~/.agent/diagrams/filename.html`

**Tell the user** both file paths:
- Plan spec: `~/.claude/blueprint/{name}-blueprint.md`
- Visual: `~/.agent/diagrams/{name}-blueprint.html`

## File Structure

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Blueprint: Feature Name</title>
  <link href="https://fonts.googleapis.com/css2?family=...&display=swap" rel="stylesheet">
  <style>/* All CSS inline */</style>
</head>
<body>
  <!-- TOC nav (for standard/full) -->
  <!-- Section 1: Executive Summary with KPI cards -->
  <!-- Section 2: Architecture (Mermaid diagrams with zoom controls) -->
  <!-- Section 3: Phased Implementation (timeline) -->
  <!-- Section 4: Per-Phase Details (collapsible) -->
  <!-- Section 5: Dependency Graph (Mermaid DAG) -->
  <!-- Section 6: Risk Matrix (HTML table) -->
  <!-- Section 7: Verification Checklist -->
  <script type="module">/* Mermaid init, scroll spy, zoom controls */</script>
</body>
</html>
```

## Mermaid Usage

Blueprints typically need 2-3 Mermaid diagrams:

- **Architecture:** `graph TD` or `graph LR` showing system components and data flow
- **Dependency DAG:** `graph TD` showing task dependencies with critical path highlighted via `classDef`
- **Sequence (optional):** `sequenceDiagram` for complex interaction flows

Always use `theme: 'base'` with custom `themeVariables` matching the page palette. Always add zoom controls (+/-/reset) to every `.mermaid-wrap` container. See `./references/libraries.md` for full Mermaid theming guide.

## Quality Checks

Before delivering, verify:
- **The squint test**: Blur your eyes. Can you still perceive section hierarchy? Are KPIs prominent?
- **The swap test**: Would replacing fonts and colors with a generic dark theme make this indistinguishable? Push the aesthetic further.
- **Both themes**: Toggle OS light/dark. Both should look intentional.
- **Information completeness**: Does the blueprint cover the full scope of the request? Every file that needs changing should be listed. Every phase should have clear tasks.
- **No overflow**: Resize browser. No content clips or escapes containers. Every grid/flex child needs `min-width: 0`.
- **Mermaid zoom controls**: Every diagram needs +/-/reset buttons, Ctrl/scroll zoom, click-drag pan.
- **Collapsible sections work**: Per-phase details should expand/collapse cleanly.
- **File opens cleanly**: No console errors, no broken fonts, no layout shifts.
- **Actionability**: Someone should be able to read this blueprint and start coding immediately. File paths, function names, type definitions -- not vague descriptions.
