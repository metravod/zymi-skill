# zymi as an MCP server — pipelines as tools for any agent

Load this reference when the user wants an external agent (Claude Code, Claude Desktop, Cursor, Windsurf, OpenHands, LangGraph, …) to **invoke zymi pipelines as tools**, or asks about `zymi mcp serve`, `expose:`, approval forms popping in the host, reasoning delegation (`ask:` steps / `zymi/reasoning/resume`), or `zymi.runs.*` introspection.

Requires **zymi-core ≥ 0.7.0**.

## Direction matters

zymi speaks MCP in both directions. Don't confuse the two surfaces:

| Surface | Role | Direction |
|---|---|---|
| `project.yml::mcp_servers:` | zymi is the **client** | zymi's agents consume third-party MCP tools (ADR-0023) |
| `zymi mcp serve` | zymi is the **server** | external agents invoke zymi pipelines as MCP tools (ADR-0033) |

The server direction is the positioning move: instead of handing an agent a raw shell, you hand it a *procedure* — a deterministic, audited, approval-gated pipeline. One stdio server reaches every major MCP host; there are no per-runtime wrappers (LangChain/CrewAI adapters are deliberately not a thing — those runtimes consume MCP natively).

## Expose a pipeline (opt-in, per pipeline)

Pipelines are **not exposed by default**. Each one opts in via `expose.mcp:` in its own YAML:

```yaml
# pipelines/release.yml
name: release
description: "Cut a release: bump, check, commit, tag, push."
expose:
  mcp:
    name: release            # optional; defaults to the pipeline name
    description: "..."       # optional; defaults to the pipeline description:
    mode: sync               # sync (default) | async — see "Long pipelines" below
inputs:
  - name: version
    type: string             # string|integer|number|boolean|object|array
    required: true
    description: "Semver to release, e.g. 0.7.1"
steps:
  ...
```

Opt-in is deliberate: every exposed pipeline adds a tool descriptor to every connected agent's system prompt. Internal cron jobs and batch workflows should never appear in agent tool catalogs.

**The tool's input schema is auto-generated from `inputs:`.** Connected agents choose tools by reading names, descriptions, and schemas — an untyped, undescribed `inputs:` block produces a tool agents misuse. Before exposing a pipeline, tighten every input: explicit `type:`, a real `description:`, correct `required:`. Same goes for the pipeline-level `description:`.

## Serve

```bash
zymi mcp serve --dir /path/to/project        # stdio transport; run from anywhere
```

Flags:

- `--include <GLOB>` / `--exclude <GLOB>` — boot-time filters over the exposed set (`*`, `?`; exclude wins). Default with no flags: everything that declares `expose.mcp:`.
- `--expose-observability` — adds the four read-only `zymi.runs.*` tools (below). Off by default to keep the catalog lean.
- `--observability-scope session|all` — `session` (default) lets the agent see only runs this serve process started; `all` opens the whole event store (single-user dev setups only).

Wire into hosts:

```bash
# Claude Code
claude mcp add --transport stdio zymi -- zymi mcp serve --dir /path/to/project
```

```json
// Claude Desktop / generic mcpServers JSON, incl. project-level .mcp.json
{ "mcpServers": { "zymi": { "command": "zymi", "args": ["mcp", "serve", "--dir", "/path/to/project"] } } }
```

No LLM API key is needed by the *server* unless the exposed pipelines contain agent steps — a pipeline made of deterministic tool steps runs with a stub `llm:` block (the loader requires the key to be present, not valid).

## Long pipelines: sync vs async (SEP-1686 Tasks)

A plain `tools/call` **blocks until the pipeline terminates** and returns the output (pipeline failure surfaces as a tool result with `isError: true`, not a protocol error). That's the right shape for pipelines that finish within the host's per-tool-call timeout.

For longer runs, the server implements **SEP-1686 Tasks** (MCP spec `2025-11-25`): a client that supports tasks augments `tools/call` with `_meta["modelcontextprotocol.io/task"]`, gets a task handle back immediately, then polls `tasks/get` / fetches `tasks/result` / may `tasks/cancel`. Two facts that surprise people:

- **The client decides, not the YAML.** Any exposed pipeline can be task-augmented. `mode: async` doesn't change the tool descriptor — it's a hint that callers *should* task-augment this one.
- **Host adoption is the constraint.** If the connected host doesn't speak tasks, a long pipeline must fit the host's per-call timeout. In Claude Code, raise `MCP_TOOL_TIMEOUT` (env var, ms) when a sync pipeline step legitimately runs for minutes — a full test-suite step is the classic case.

There are no streaming/progress notifications in v1 — the result arrives when the pipeline terminates.

## Approvals over MCP — the elicitation bridge

This is the flagship behavior: a pipeline step that hits a `requires_approval: true` tool **pauses, and an approve/deny form pops in the host UI** (Claude Code ≥ 2.1.76 renders it natively). Flow: `ApprovalRequested` on the event bus → server sends `elicitation/create` to the client → human approves or denies in the host → `ApprovalGranted`/`ApprovalDenied` → pipeline resumes or halts. The whole round-trip is in the event log like any other approval.

Routing rules under `zymi mcp serve`:

- The elicitation channel is the **default route only when the project declares no `default_approval_channel:`**. Project-declared channels (telegram, http, …) start alongside and keep working as configured.
- **Fail-closed.** A connected client that doesn't advertise `capabilities.elicitation` gets `ApprovalDenied` with reason `client_no_elicitation` — the gated action never fires silently. If approvals "mysteriously deny" under serve, check the host's elicitation support first.

Note: per-tool `requires_approval: true` was a dead flag before 0.7.0 (it parsed but never gated HTTP/Python/MCP tools). If approval gates don't fire at all, check the zymi version before debugging the channel.

## Reasoning delegation — `ask:` steps over MCP (ADR-0042)

Where the elicitation bridge borrows the caller's *human*, an [`ask:` step](pipelines.md#ask-step-adr-0042) borrows the caller's *model*. A pipeline exposed over `zymi mcp serve` can hand a reasoning question back to the connected agent instead of configuring a second `llm:` — *summarize this diff*, *does this output look right?* — and the agent answers from its own loop.

Crucially this is **not** MCP `sampling` (deprecated, SEP-2577) and **not** elicitation (that returns human form data). It rides the SEP-1686 task + resume surface — plain request/response tools:

1. The caller **task-augments** the `tools/call` (this is required — a sync call can't answer a reasoning question, there's no one to ask mid-blocking-call).
2. When the run hits the `ask:` step it parks; the task goes **`input_required`**, and `tasks/get` carries an `inputRequest` payload: `{ status: "needs_reasoning", prompt, resume_token }`.
3. The caller reads the prompt, **reasons in its own loop**, and calls the method **`zymi/reasoning/resume`** with `{ resume_token, answer }`.
4. The server publishes `ReasoningAnswered`, the parked run resumes with `answer` as the step output, and the task proceeds to `completed` (or the next `input_required`).

Properties to know:

- **`resume_token` is opaque + signed + expiring.** Don't inspect or mutate it — echo it back verbatim. A tampered or expired token is rejected (`resume rejected: …`). If the caller never resumes, the parked step fails closed on the timeout (it does not hang forever).
- **Sync calls can't do reasoning delegation.** An `ask:` step only works on a task-augmented call. On a plain sync `tools/call` there's no channel to answer under serve, so it fails closed. (This is the mirror of "async tasks don't pause for *approvals*" — approvals are sync-only via elicitation; reasoning is task-only via resume.)
- **The answer is untrusted model output** re-entering zymi as a step output — the ordinary sink guards apply downstream (see pipelines.md). Design the pipeline so an `ask:` answer feeding a shell/HTTP/file sink goes through a guarded tool, not a raw `execute_shell_command`.
- **A pure tool + ask pipeline needs no `llm:`** (ADR-0041), which is the point: the model lives on the other end of the MCP connection.

## Self-introspection: `zymi.runs.*` (opt-in)

With `--expose-observability`, the connected agent can investigate its own runs without anyone switching to a terminal:

| Tool | Returns | Cost |
|---|---|---|
| `zymi.runs.list(pipeline?, status?, limit)` | compact run summaries | cheap |
| `zymi.runs.get(run_id)` | status, current step / total steps | cheap |
| `zymi.runs.events(run_id, since_seq?, types?, limit)` | flat event projection, no nested payloads | moderate |
| `zymi.runs.step_io(run_id, step_id)` | **full** prompt-in / output-out for one step | expensive |

Drive the agent **narrow-before-deepen**: `list → get → events(filtered) → step_io(one step)`. `step_io` returns full prompts and outputs — calling it before narrowing down the failing step blows the agent's context for nothing. The reconstructed prompt is the *unmasked* exchange (the exact masked per-iteration context isn't stored verbatim; the result says so in a `note`).

This surface is what makes "the pipeline failed" debuggable inside the host conversation: the agent reads its own trace, diagnoses, and proposes a fix — combined with this skill's YAML knowledge, that's the whole loop.

## Gotchas

- **Host per-tool-call timeouts** are the #1 operational surprise for sync pipelines with long deterministic steps (test suites, builds). Raise the host's timeout (`MCP_TOOL_TIMEOUT` in Claude Code) or move the caller to task-augmented calls.
- **Startup latency is on you now.** `zymi mcp serve` is a long-lived child process of the host; slow boot is exactly what we complain about in third-party MCP servers. Keep project load lean; don't put network calls in module-level Python tool code.
- **Loose `inputs:` = bad agent behavior.** The agent only sees what `tools/list` shows. If the agent passes garbage, fix the input schema (types + descriptions) before blaming the agent.
- **Approval hangs vs denies.** Under `zymi run`/`zymi serve` a missing approval channel makes the run *hang* on `ApprovalRequested`; under `zymi mcp serve` a non-elicitation client makes it *deny* fail-closed. Different symptoms, same root area — read the event log.
- **`ask:` needs a task-augmented call.** If an exposed pipeline with an `ask:` step "does nothing / fails closed" under serve, check that the caller task-augmented the `tools/call` — a plain sync call has no way to answer a reasoning question and fails closed. The caller must poll `tasks/get`, read the `inputRequest.resume_token`, and call `zymi/reasoning/resume`.

## Real-world example

zymi-core releases itself through this surface: the `ops/` project in the [zymi-core repo](https://github.com/metravod/zymi-core/tree/main/ops) exposes a `release` pipeline as an MCP tool (clean-tree guard → version bump → clippy+tests → commit → approval-gated tag+push). The human's role in a release is reading the diff and clicking Approve in the elicitation form. Use it as the canonical wiring reference.
