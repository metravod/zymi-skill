---
name: zymi-skill
description: "Use this skill when the user is building, scaffolding, debugging, or auditing an agent with zymi-core (the event-sourced agent engine distributed via `uv tool install zymi-core` or `pip install zymi-core`), or exposing zymi pipelines as MCP tools to another agent. Activates on mentions of `zymi`, `zymi run`, `zymi serve`, `zymi init`, `zymi fetch`, `zymi observe`, `zymi mcp serve`; on imports of `zymi` / `from zymi import tool`; on edits to `project.yml`, `pyproject.toml`, `pipelines/*.yml`, `agents/*.yml`, `tools/*.yml`, `tools/*.py`; and on questions about zymi pipelines, tools, MCP wiring (either direction), approvals, context window, fork/resume, events store, or `zymi` CLI. Covers the declarative YAML surface, the four tool kinds, MCP client and server (`expose.mcp:`, SEP-1686 tasks, elicitation approvals, `zymi.runs.*` introspection), HTTP connectors, event-sourced approvals, context-window tuning, observability, and the `uv tool install` + `zymi fetch` + per-project `.venv` install model (ADR-0032)."
metadata:
  version: "0.7.0"
  scope: "zymi-core-pip-user"
  file_policy: "markdown-only"
---

# zymi skill

Use this skill when the user is building or operating an agent **with zymi-core**. Do not use it for generic agent-architecture questions — for that there is the separate `agents-best-practices` skill. This skill is the opinionated, zymi-specific layer.

## Core stance

zymi is an event-sourced engine. The model proposes; the YAML defines the plan; the runtime appends events. Four load-bearing consequences:

1. **The pipeline YAML _is_ the plan.** Do not generate procedural Python loops to orchestrate agents. Add steps to `pipelines/<name>.yml`, connect them with `depends_on`, branch with `when:`, terminate with `output:`.
2. **Every tool call is an event.** Observability is free — `zymi observe` (TUI) and `zymi events` (CLI) read the same event log the runtime writes. Do not reinvent logging.
3. **Approvals are event-sourced.** Never gate side effects with `input()` or ad-hoc HTTP. Set `requires_approval: true` on the tool and configure a channel in `project.yml::approvals:`. The runtime publishes `ApprovalRequested` and waits.
4. **Pipelines are MCP tools (≥0.7.0).** A pipeline that declares `expose.mcp:` becomes a callable tool for any MCP host (Claude Code, Cursor, OpenHands, …) via `zymi mcp serve` — audit trail, approval gates, and introspection included. When another agent needs to trigger zymi work, expose a pipeline; don't write glue code.

## When to activate

Activate on any of:

- user explicitly mentions zymi / zymi-core / a `zymi` subcommand;
- user imports `zymi` or `from zymi import tool` in Python;
- user edits `project.yml`, `pipelines/*.yml`, `agents/*.yml`, `tools/*.yml`, `tools/*.py`;
- working directory contains `project.yml` and/or `.zymi/`;
- user asks to "build an agent with zymi", "add a tool", "wire MCP", "scaffold a pipeline", "add approvals", "tune the context window", "resume from event N";
- user wants another agent (Claude Code, Cursor, OpenHands, …) to call a zymi pipeline — "expose this pipeline", "zymi mcp serve", "pipeline as a tool".

Do **not** activate for generic single-turn Q&A unrelated to zymi.

## First action: orient

If you're not sure what the project looks like yet, run a fast inventory before suggesting anything:

```bash
ls project.yml pyproject.toml pipelines/ agents/ tools/ .venv/ .zymi/  2>/dev/null
zymi pipelines    # if zymi is installed; lists pipelines + their inputs
```

This tells you whether the project exists, which pipelines are defined, whether a `.venv` has been built (`zymi fetch` was run), and where to attach new work. Avoid suggesting `zymi init` in an already-initialized directory.

## Install model (ADR-0032)

The shipped UX is a thin global CLI plus per-project venvs:

```bash
uv tool install zymi-core    # one-time per machine; puts `zymi` on PATH globally
zymi init                    # scaffolds project.yml + pyproject.toml + .python-version
zymi fetch                   # `uv sync` under the hood — builds ./.venv from pyproject.toml
zymi serve <pipeline>        # transparently re-execs inside ./.venv/bin/zymi if present
```

When the user adds a Python `@tool` that imports a third-party library, the dep belongs in **their project's `pyproject.toml`**, not the global tool env. After editing, they rerun `zymi fetch`. Re-exec can be bypassed for contributor workflows with `--no-venv`. `pip install zymi-core` into a traditional venv remains supported but is no longer the default user path.

## How to use this skill

Load references on demand, not all up front:

- **[references/quickstart.md](references/quickstart.md)** — `uv tool install zymi-core` → `zymi init` → `zymi fetch` → first run end-to-end. Load when the user is starting fresh.
- **[references/pipelines.md](references/pipelines.md)** — pipeline YAML schema, agent steps vs deterministic tool steps, `when:` branching, `any_of` outputs, fresh context, fork/resume. Load when the user is designing or editing a pipeline.
- **[references/tools.md](references/tools.md)** — the four tool kinds (declarative, Python `@tool`, MCP, builtin), when to pick which, schema introspection, `requires_approval`, `no_resume`. Load when adding a new capability.
- **[references/mcp-and-connectors.md](references/mcp-and-connectors.md)** — MCP servers (zymi as client), HTTP inbound/outbound, cron, file/stdin connectors. Load when wiring external systems *into* zymi.
- **[references/zymi-as-mcp-server.md](references/zymi-as-mcp-server.md)** — `zymi mcp serve` (zymi as MCP server): `expose.mcp:`, host wiring, SEP-1686 tasks for long pipelines, elicitation approval forms, `zymi.runs.*` introspection tools. Load when another agent should call zymi pipelines.
- **[references/observability.md](references/observability.md)** — `zymi observe`, `zymi events`, hash chain, `runs`, `verify`, fork-resume, event store backends. Load when debugging, auditing, or replaying.
- **[references/troubleshooting.md](references/troubleshooting.md)** — known papercuts and recoveries. Load when something looks wrong.

## Decision tree — "I need to do X"

| Need | Use |
|---|---|
| Run a Python function from an agent | `@tool` in `tools/*.py` |
| Hit one HTTP endpoint per pipeline | Declarative `kind: http` tool, _or_ inline tool step with `args:` |
| Use a service that exposes N tools (Pinecone, filesystem, GitHub, …) | MCP server in `project.yml::mcp_servers:` — preferred over hand-rolling N tools |
| React to incoming messages / files / cron ticks | `connectors:` in `project.yml` (`http_inbound`, `http_poll`, `cron`, `file_read`, `stdin`) |
| Send results out automatically | `outputs:` in `project.yml` (`http_post`, `file_append`, `stdout`) on `ResponseReady` |
| Agent picks between branches | Router agent calls a tool returning a label, downstream steps gate with `when:`, terminal `output: any_of:` |
| Deterministic step with no LLM in the middle | Tool step (`tool:` + `args:` instead of `agent:` + `task:`) — fails the pipeline loud on tool error |
| Skip a side-effect call when resuming a fork | `@tool(no_resume=True)` for Python; `no_resume: [name1, name2]` per-MCP-server in `project.yml` |
| Wipe context before a step (long-running goal loops, ETL fans) | `context: { mode: fresh }` on the step (ADR-0031) |
| Long conversational session that blows out the prompt | Tune `runtime.context:` — see references/pipelines.md "chat profile" |
| Human approves an action before it fires | `requires_approval: true` on the tool + `approvals:` channel in `project.yml` |
| Let another agent (Claude Code, Cursor, …) call a pipeline as a tool | `expose.mcp:` in the pipeline YAML + `zymi mcp serve` — see references/zymi-as-mcp-server.md |
| Pipeline runs minutes/hours when called over MCP | `mode: async` hint + SEP-1686 task-augmented calls; or raise the host's per-call timeout |
| Connected agent debugs its own zymi runs from inside the host | `zymi mcp serve --expose-observability` → `zymi.runs.list/get/events/step_io` |
| Watch the agent live | `zymi observe` (TUI run picker) or `zymi observe -r <stream-id>` |
| Reconstruct what happened after the fact | `zymi events --stream <id>` (add `--raw` for JSON-per-line) |
| Re-run from step N with edits | `zymi resume <stream-id> --from-step <id>` |

## Non-negotiables

- **Don't suggest a Python orchestration loop.** If the user is reaching for `while True: agent.run(...)`, the answer is "put it in a pipeline."
- **Don't gate side effects in user code.** Use `requires_approval`; the engine owns the approval state.
- **Don't catch tool errors silently in a tool step.** Tool steps (ADR-0024/0027) fail-fast on purpose — that's the contract. Wrap in an agent step if you need LLM-mediated recovery.
- **Don't store conversation memory in `runtime.context` knobs.** Those bound context length; persistent memory belongs in the event store (the source of truth) or a future memory layer.
- **Don't generate examples that import internals** (`zymi._zymi_core`, `zymi.runtime.*`). Public surface is `from zymi import tool`, the CLI, and `Runtime` / `EventStore` / `ToolRegistry` from `docs/python-api.md`.
- **Don't recommend `npx -y` for MCP servers in production.** Resolve to a globally installed binary; the `-y` form re-resolves on every spawn and breaks on flaky DNS. See references/troubleshooting.md.
- **Don't claim a tool is "called" unless you see `ToolCallCompleted`.** Models occasionally write tool intent into the assistant message text without emitting a tool call. The event log is ground truth.

## Default output structure when advising

When the user asks "how do I add X to my zymi project", respond with the smallest concrete diff, in this order:

1. **What to add and where** — file path + minimal YAML/Python block, copy-pasteable.
2. **How to test it** — exact `zymi` command (`zymi run …`, `zymi serve …`, `zymi observe …`).
3. **What to watch for** — the event(s) that prove it worked (e.g. `ToolCallCompleted`, `ApprovalGranted`, `OutputResolved`).
4. **One gotcha if relevant** — link to references/troubleshooting.md only if there's a real, known pitfall.

Avoid theoretical multi-option answers when the user has a concrete need. Pick the lowest-friction option, name the alternatives in one line, move on.

## Tool registry — what ships in the catalog out of the box

Builtin tools available in any project (no declaration needed):

- `read_file`, `write_file` (gated by `contracts.file_write.allowed_dirs`)
- `execute_shell_command` (gated by `policy.allow` / `policy.deny`)
- `write_memory` (writes to `./memory/`, gated by `contracts.file_write`)
- `spawn_sub_agent` (advanced — spawn another configured agent from inside a turn)

Note: `web_search` and `web_scrape` are **not builtins**. The scaffold (under `tools/`) ships them as declarative `kind: shell` stubs that echo `"not configured"`; the user uncomments a real provider block in the YAML or deletes the file. Don't assume real search is wired.

Anything else (Pinecone, GitHub, filesystem with full access, weather, translate, …) is something the user attaches via MCP, declarative tool, or `@tool`.

## Gotchas

- `zymi events` against a SQLite store can miss recent events stuck in the WAL — checkpoint with `sqlite3 .zymi/events.db 'PRAGMA wal_checkpoint(FULL);'`. `zymi observe` is unaffected (reads via the same writer). See references/troubleshooting.md.
- MCP servers spawned with `npx -y …` re-resolve packages on every startup; a missed `init_timeout_secs` silently drops the server's tools from the catalog until the process restarts. Pin via global install.
- A `kind: agent` step that calls a `requires_approval: true` tool will pause the whole pipeline (correctly) — make sure the approval channel is configured before testing, otherwise the run hangs without explanation.
- `context.mode: fresh` (ADR-0031) wipes Layer A (observation history) for the step that declares it, not for downstream steps. It is not a global reset.
- Resume re-fires every tool call in the re-executed subgraph unless the tool was registered `no_resume=True`. For real send-email / send-money tools, set the flag; the runtime then emits a synthetic placeholder result instead of calling out.
- Under `zymi mcp serve`, a sync pipeline with a long deterministic step (test suite, build) hits the **host's** per-tool-call timeout, not zymi's. Raise it (Claude Code: `MCP_TOOL_TIMEOUT`) or have the caller task-augment the call (SEP-1686).
- Under `zymi mcp serve`, approval gates **fail closed** when the connected client doesn't support elicitation: `ApprovalDenied { reason: "client_no_elicitation" }`, not a hang. A "mysterious deny" usually means the host lacks elicitation support, not a broken channel.
- Per-tool `requires_approval: true` was a dead flag before zymi-core 0.7.0 — it parsed but never gated HTTP/Python/MCP tools. If approvals don't fire at all, check the version first.

## Source links

- zymi-core repo and docs: https://github.com/metravod/zymi-core
- Top-level map for agents: [llms.txt](https://github.com/metravod/zymi-core/blob/main/llms.txt)
- ADR index: https://github.com/metravod/zymi-core/tree/main/adr
- Generic agent architecture (provider-neutral, complementary): https://github.com/DenisSergeevitch/agents-best-practices
