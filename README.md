# zymi-skill

> An opinionated Agent Skill for building, scaffolding, and debugging agents with [zymi-core](https://github.com/metravod/zymi-core) — the event-sourced agent engine distributed as `pip install zymi-core`.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Agent Skill](https://img.shields.io/badge/Agent-Skill-7c3aed)](SKILL.md)
[![Claude Code](https://img.shields.io/badge/Claude_Code-compatible-8b5cf6)](SKILL.md)

This is the **zymi-specific** layer. For provider-neutral agent architecture (loops, permissions, planning, context), see the complementary [`agents-best-practices`](https://github.com/DenisSergeevitch/agents-best-practices) skill.

## What this gives you

Drop this skill into your agent and asking it to "build an agent with zymi", "add a tool", "wire MCP", "scaffold a pipeline", "set up approvals", or "tune the context window" produces zymi-native answers — not generic LLM advice that you then have to re-translate into zymi YAML.

The skill activates on:

- mentions of `zymi`, `zymi run`, `zymi serve`, `zymi init`, `zymi observe`;
- imports of `zymi` or `from zymi import tool` in Python;
- edits to `project.yml`, `pipelines/*.yml`, `agents/*.yml`, `tools/*.yml`, `tools/*.py`;
- working directories that contain `project.yml` and/or `.zymi/`.

## Install

### Claude Code, user-level (every project)

```bash
mkdir -p "$HOME/.claude/skills"
git clone https://github.com/metravod/zymi-skill.git \
  "$HOME/.claude/skills/zymi-skill"
```

### Claude Code, project-level (one project)

```bash
mkdir -p .claude/skills
git clone https://github.com/metravod/zymi-skill.git \
  .claude/skills/zymi-skill
```

### Codex

```bash
mkdir -p "${CODEX_HOME:-$HOME/.codex}/skills"
git clone https://github.com/metravod/zymi-skill.git \
  "${CODEX_HOME:-$HOME/.codex}/skills/zymi-skill"
```

### Or paste this to your AI agent

```text
Install the zymi-skill skill for me:

1. Clone https://github.com/metravod/zymi-skill into the skill directory my
   agent reads on this machine (e.g. ~/.claude/skills/ for Claude Code, or
   ~/.codex/skills/ for Codex).
2. Verify that SKILL.md and the references/ directory are present.
3. Confirm the install path when done.
```

## Layout

```text
SKILL.md                          # activation rules + decision tree + non-negotiables
references/
  quickstart.md                   # pip install → zymi init → first run
  pipelines.md                    # pipeline YAML schema, branching, fresh context
  tools.md                        # the 4 tool kinds, approvals, no_resume
  mcp-and-connectors.md           # MCP servers, inbound connectors, outbound outputs
  observability.md                # zymi observe, zymi events, fork-resume, store backends
  troubleshooting.md              # known papercuts and recoveries
```

Progressive disclosure: `SKILL.md` is loaded by the agent runtime when the skill activates; references are loaded only when relevant to the user's task.

## What this is _not_

- Not a tutorial. The skill assumes the user is already an engineer; it teaches the agent how to advise, not how to teach humans.
- Not a generic agent-architecture skill. For that there's [`agents-best-practices`](https://github.com/DenisSergeevitch/agents-best-practices), and the two compose cleanly.
- Not auto-generated. Every example here is grounded in the current zymi-core scaffold and ADR set; updated as the engine ships.

## Versioning

The skill version (in `SKILL.md` frontmatter) tracks the zymi-core minor version it was last verified against. Patch-level zymi releases don't bump the skill.

## Contributing

Issues and PRs welcome. Two ground rules:

1. Examples must run against current `pip install zymi-core` — no aspirational APIs.
2. Don't grow the skill body for completeness — keep `SKILL.md` tight and push detail into `references/`.

## License

MIT — see [LICENSE](LICENSE).
