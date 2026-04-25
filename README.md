# Skills

A collection of reusable skills for AI coding agents (Claude Code, Codex, and others).

Each skill is a self-contained directory with a `SKILL.md` that defines when and how the agent should use it.

## Skills

| Skill | Description |
|---|---|
| [codebase-audit](codebase-audit/) | Deep codebase audit: understand the project first, then systematically find hardcoded constants and unfinished code |
| [cli-for-agent](cli-for-agent/) | Design and review CLIs for reliable agent automation: non-interactive flags, structured output, fast errors, idempotency |
| [consult-chatgpt-pro](consult-chatgpt-pro/) | Consult ChatGPT Pro through a desktop browser with sanitized review packets, action-time approval, and local validation |
| [consult-chatgpt-pro-browser](consult-chatgpt-pro-browser/) | Consult ChatGPT Pro through the Codex in-app browser with sanitized review packets, action-time approval, send verification, and local validation |
| [init](init/) | Generate a minimal AGENTS.md context file for a repository using the WHAT/WHY/HOW framework |
| [mcp-for-agents](mcp-for-agents/) | Design and review MCP servers for AI agents: outcome-oriented tools, flat parameters, actionable errors, token efficiency |
| [systematic-fix](systematic-fix/) | Fix-verify-lockdown workflow: catalog issues, fix one at a time with verification, then write e2e regression tests covering each fix and its failure family |
| [simplify-code](simplify-code/) | Review git diffs for reuse, quality, efficiency, and clarity issues with parallel sub-agent reviews, then optionally apply safe behavior-preserving fixes |
| [blueprint](blueprint/) | Generate visual HTML blueprint pages and structured markdown plan specs with architecture diagrams, phased task breakdowns, file change maps, dependency graphs, risk matrices, and verification checklists |
| [visual-explainer](visual-explainer/) | Generate self-contained HTML visualizations for architecture diagrams, flowcharts, data tables, timelines, and dashboards |

## Installation

### skills.sh (recommended)

Install any skill from this collection using the [`skills` CLI](https://skills.sh/docs):

```bash
npx skills add ReinaMacCredy/Skills
```

### Claude Code

```bash
# Install a single skill
claude skill add /path/to/Skills/systematic-fix

# Or symlink into your skills directory
ln -s /path/to/Skills/systematic-fix ~/.claude/skills/systematic-fix
```

### Manual

Copy or symlink the skill directory into your agent's skill search path.

## Structure

Each skill follows this layout:

```
skill-name/
  SKILL.md          # Required. Frontmatter (name, description) + instructions.
  reference/        # Optional. Docs loaded into context as needed.
  templates/        # Optional. File templates used in output.
  prompts/          # Optional. Sub-agent prompt templates.
```

## License

See individual skill directories for licensing. Skills without explicit licenses are provided as-is.
