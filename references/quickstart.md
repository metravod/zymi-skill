# Quickstart — pip install to first run

Use this reference when the user is starting fresh: no `project.yml`, no `.zymi/`, no scaffold.

## Install

```bash
pip install zymi-core
zymi --version
```

The Python package is `zymi-core`; the import is `zymi` (`from zymi import tool`); the CLI is `zymi`. There is also a Rust crate of the same name for non-Python consumers — pip is the primary product.

## Scaffold a project

```bash
zymi init my-agent
cd my-agent
```

`zymi init` lays down (paths relative to project root):

- `project.yml` — top-level config (LLM provider, policy, contracts, optional `connectors`/`outputs`/`approvals`/`mcp_servers`)
- `agents/default.yml` — one starter agent with a small tool list
- `pipelines/main.yml` — one starter pipeline calling the default agent
- `AGENTS.md` — conventions doc for any agent (human or AI) working inside this project
- `.env.example` — environment variables the scaffold expects

Two scaffolds ship. The minimal one (one agent, one pipeline, no LLM provider configured) is the default — `zymi init my-agent` lays it down. For a full Telegram chat bot with an approval-gated `broadcast` tool, a commented MCP filesystem example, and Telegram inbound + outbound + approval channel, pass `--example telegram`:

```bash
zymi init my-agent --example telegram
```

`zymi init` accepts `-n/--name` to override the project name and `--example <name>` to pick a non-default scaffold; `telegram` is currently the only named example.

## Set the LLM provider

Edit `project.yml`:

```yaml
llm:
  provider: openai          # or "anthropic"
  model: gpt-4o-mini        # or "claude-opus-4-7", "claude-sonnet-4-6", etc.
  api_key: ${env.OPENAI_API_KEY}
```

Both providers are first-class — pick by the user's API key. zymi reads `${env.NAME}` from `.env` (loaded automatically) or the process environment.

## First run

```bash
zymi run main -i task="Summarise what zymi is in 3 bullets"
```

`main` is the pipeline name (filename stem of `pipelines/main.yml`). `-i task=...` sets `${inputs.task}`. The CLI prints the final output and the stream-id of the run.

To watch live:

```bash
zymi observe                          # TUI run picker
zymi observe -r <stream-id>           # pre-select a specific run
```

To re-read what happened afterwards:

```bash
zymi events --stream <stream-id>
```

## Minimal viable shape

The smallest useful zymi project is three files:

```yaml
# project.yml
name: my-agent
llm:
  provider: openai
  model: gpt-4o-mini
  api_key: ${env.OPENAI_API_KEY}
```

```yaml
# agents/writer.yml
name: writer
description: "Writes short answers."
tools: []
max_iterations: 4
```

```yaml
# pipelines/answer.yml
name: answer
inputs:
  - name: question
    required: true
steps:
  - id: respond
    agent: writer
    task: "${inputs.question}"
output:
  step: respond
```

```bash
zymi run answer -i question="What is event sourcing?"
```

No tools, no connectors, no approvals — just an agent and a pipeline. Build up from here.

## What `zymi init` decides for you (and how to override)

The scaffold sets defaults that fit a coding-style agent. For a chat bot, override two things in `project.yml`:

1. `runtime.context:` — see references/pipelines.md "chat profile". Defaults (10 turns / 400k chars / 600k chars) are sized for long coding runs and never compact a normal conversation.
2. `defaults.max_iterations:` — drop to 2–4 for chat (a chat turn that loops 10 times is almost always a bug).

## Common first-task patterns

- "I want one tool the agent can call" → references/tools.md, prefer `@tool` in `tools/my_tool.py`.
- "I want the agent to hit an API" → references/tools.md (declarative HTTP) or references/mcp-and-connectors.md if the API has a published MCP server.
- "I want to react to incoming messages" → references/mcp-and-connectors.md, `connectors:` section.
- "I want a human to approve before X happens" → references/tools.md "Approvals" + references/pipelines.md "approval_channel".
