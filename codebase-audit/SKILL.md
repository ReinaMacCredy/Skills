---
name: codebase-audit
description: Deep codebase audit that first reads project docs (AGENTS.md, README.md) and investigates the architecture, then systematically finds hardcoded constants that should be dynamic and unfinished code (TODOs, "will"/"would" in comments). Use this skill when the user wants a thorough code review for hygiene issues, asks to find hardcoded values, wants to audit for TODOs or incomplete code, says things like "audit this codebase", "find hardcoded stuff", "what's unfinished", "find tech debt", "code hygiene check", or asks you to look across the whole project for issues. Also use when the user asks to understand a project before making changes, or wants a comprehensive health check of their codebase.
---

# Codebase Audit

A systematic audit that builds deep understanding of the project before identifying problems. The understanding phase isn't just a formality -- it's what lets you distinguish "this constant is fine" from "this constant will break in production."

## Phase 1: Understand the Project

Before looking for problems, you need to know what the project *is* and how it's built. Without this context, you'll flag things that are intentional and miss things that are actually broken.

1. **Read the docs thoroughly.** Start with AGENTS.md and README.md (or whatever top-level docs exist). Read them completely -- don't skim. These tell you what the project does, how it's structured, and what conventions it follows. If there's a CLAUDE.md, CONTRIBUTING.md, or architecture doc, read those too.

2. **Investigate the code structure.** Use code navigation to understand the technical architecture: what the main modules are, how they connect, what the entry points are, what external services it talks to. Build a mental model of the system before you start auditing it.

   The goal is to understand the project well enough that you could explain its architecture to someone in 2-3 paragraphs. If you can't do that yet, keep reading.

3. **Summarize your understanding.** Before moving to Phase 2, briefly state what the project does and how it's structured. This forces you to crystallize your understanding and gives the user a chance to correct any misunderstandings before you start the audit.

## Phase 2: Find Hardcoded Constants

Search the entire project for values that are hardcoded but should be dynamic -- meaning they'll be wrong in some environments, over time, or under different conditions.

**What to look for:**

- **Environment-specific values**: URLs, hostnames, ports, file paths, database names that assume a specific environment (dev, staging, prod)
- **Time-sensitive values**: dates, version numbers, year references (especially in copyright strings), expiration times that will become stale
- **Scale-dependent values**: buffer sizes, timeouts, retry counts, batch sizes, page limits that assume a specific load or data volume
- **User/account-specific values**: API keys, account IDs, email addresses, usernames embedded in code
- **Platform-specific values**: OS paths (forward slash vs backslash), architecture assumptions, platform-specific defaults
- **Magic numbers**: numeric constants without clear meaning that should be named constants or configuration

**What to leave alone:**

- Mathematical constants (pi, e, etc.)
- Enum-like values that are genuinely fixed (HTTP status codes, protocol versions)
- Default values that are clearly documented and appropriate
- Test fixtures and mock data (these are supposed to be hardcoded)
- Configuration defaults that are reasonable and overridable

For each finding, explain *why* it should be dynamic -- what scenario would make the current hardcoded value wrong.

## Phase 3: Find Unfinished Code

Search for signals that code is incomplete or placeholder:

- **TODO/FIXME/HACK/XXX comments**: The classic markers. Note what they say -- some are old and forgotten, others are active blockers.
- **"will" or "would" in comments**: Language like "this will be replaced" or "would need to handle" suggests the author intended to come back and finish something.
- **"should" in comments**: Similar to will/would -- "should validate input here" means someone knew validation was missing.
- **Placeholder implementations**: Functions that return hardcoded values, empty catch blocks, methods that just throw "not implemented"
- **Commented-out code**: Large blocks of commented code often indicate incomplete refactors or abandoned approaches

**For each finding, assess:**
- Is this actually unfinished, or is the comment outdated and the work was done elsewhere?
- How critical is it? A TODO about logging is different from a TODO about auth validation.
- Is there enough context in the comment to understand what needs to be done?

## Presenting Results

Organize findings into two clear sections:

**1. Hardcoded Constants** -- grouped by severity:
- [!] Critical: will break in production or other environments
- [~] Warning: may cause issues under certain conditions
- [?] Minor: could be improved but unlikely to cause problems

**2. Unfinished Code** -- grouped by type (TODO, placeholder, commented-out) with:
- File and line reference
- The comment or code in question
- Your assessment of severity and what needs to happen

Lead with the most important findings. If there are many findings, prioritize rather than dumping everything -- the user can ask for the full list if they want it.
