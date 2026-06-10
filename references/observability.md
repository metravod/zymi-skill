# Observability

zymi is event-sourced. Every meaningful runtime moment is an append to a hash-chained event log. There is no "logs" surface separate from the trace — the event log _is_ the trace.

## Two main consumer commands

```bash
zymi observe                       # TUI run picker
zymi observe -r <stream-id>        # pre-select a specific run
zymi events --stream <stream-id>   # full event log, CLI-friendly
```

`observe` is the live DAG view with side panels — use it while a run is in progress or to scrub a recent one (Shift+R inside the TUI triggers fork-resume). `events` is the dump — use it for forensics, grep, JSON piping (`--raw`).

## Event kinds you'll see most

| Event | Meaning |
|---|---|
| `PipelineRequested` | A connector or a CLI run kicked off a pipeline |
| `WorkflowStarted` / `WorkflowNodeStarted` / `WorkflowNodeCompleted` / `WorkflowCompleted` | The DAG and its steps started / finished (`success` flag on completions) |
| `AgentProcessingStarted` / `AgentProcessingCompleted` | A ReAct iteration inside an agent step |
| `LlmCallStarted` / `LlmCallCompleted` | One model call (with token usage on completion) |
| `ToolCallRequested` / `ToolCallCompleted` | The model called a tool; the tool returned (`is_error` flag included) |
| `ApprovalRequested` / `ApprovalGranted` / `ApprovalDenied` | Human-in-the-loop on a `requires_approval` tool |
| `StepSkipped` | A `when:` predicate was false, or an ancestor was skipped |
| `ContextCompacted` | Soft cap hit, summarization fired |
| `OutputResolved` | The pipeline's `output:` picked the answer step |
| `ResponseReady` | Final answer is ready (outputs subscribe to this) |
| `PipelineCompleted` | Run is over, with `success` flag and final output |
| `OutboundDispatched` / `OutboundFailed` | An `outputs:` entry succeeded / exhausted retries |
| `ResumeForked` | A `zymi resume` produced a new sub-stream forking off the parent |
| `McpServerConnected` / `McpServerDisconnected` | MCP subprocess handshake / shutdown |

## "What just happened?" recipe

```bash
zymi runs --limit 5                                       # list recent runs
zymi events --stream <id> | grep -E 'ToolCall|Approval'   # filter
zymi events --stream <id> --raw | jq .                    # one JSON event per line, pipe-friendly
zymi events --stream <id> --kind ToolCallCompleted        # filter by event kind
zymi events --stream <id> --verbose                       # extended per-event detail
```

If you want to know whether a tool actually executed: look for `ToolCallCompleted` with the right name. The agent may write "I'm searching the web…" into the assistant text without ever emitting a tool call — the event log is ground truth.

## Hash chain and verification

```bash
zymi verify --stream <stream-id>
```

Every event carries a hash of the previous event in the stream. `verify` walks the chain and confirms integrity. This is the audit story — if `verify` passes, no event was inserted, removed, or modified after the fact.

## Fork-resume

```bash
zymi resume <stream-id> --from-step <step_id>
```

Freezes events strictly upstream of the fork point and re-executes from there. The re-execution writes to a new sub-stream (linked to the parent), so you don't lose the original trace.

Read **references/pipelines.md "Fork-resume"** for the side-effect caveat: tools that aren't registered `no_resume=True` will re-fire. The resume command prints a `shadowed on resume (N):` section if it suppressed any side-effecting tools — eyeball that summary before resuming a production run.

## Store backends

`store:` is a **single string** at project root, parsed as a URL. Omit it and you get embedded SQLite at `<root>/.zymi/events.db` — the zero-config path.

```yaml
# project.yml
# (omit `store:` entirely for the SQLite default at .zymi/events.db)

store: sqlite                              # explicit default
store: "sqlite:/var/lib/myapp/events.db"   # SQLite at a custom path
store: "postgres://user:pass@host/zymi"    # Postgres backend
store: ${env.DATABASE_URL}                 # templated — keep secrets in .env
```

SQLite is the right answer for single-process work (one `zymi serve`, one `zymi run`). Postgres is for multi-process: several `zymi serve` workers consuming the same connectors against one event log, or sharing approval state across processes. Postgres requires the `postgres` Cargo feature — if `pip install zymi-core` wasn't built with it, you'll see a "not compiled in" error at startup; install a wheel with that feature or fall back to SQLite.

### SQLite + WAL gotcha

`zymi events` reads the DB directly. If a writer is mid-run and has un-checkpointed WAL pages, those events are invisible to a fresh reader. `zymi observe` is unaffected (it shares the writer process via the event bus).

Recovery:

```bash
sqlite3 .zymi/events.db 'PRAGMA wal_checkpoint(FULL);'
```

Then re-run `zymi events`. See references/troubleshooting.md.

## Introspection over MCP (`zymi.runs.*`)

The same trace is reachable by a *connected agent* — no terminal needed. `zymi mcp serve --expose-observability` adds four read-only tools (`zymi.runs.list` / `get` / `events` / `step_io`) so the agent that invoked a pipeline can investigate its own run from inside the host conversation. Scope defaults to runs the serve process started (`--observability-scope all` opens the whole store). Details and the narrow-before-deepen calling discipline: references/zymi-as-mcp-server.md.

## What's _not_ in the event log

- Hidden reasoning / chain of thought. zymi traces operational events, not the model's internal monologue. If a provider returns a thinking block, it's stored on the assistant message, not as a separate event.
- Raw network frames (HTTP request bodies between zymi and the LLM provider). The model output and the tool input/output are stored; provider-side wire format is not.
- Secrets in tool args. The runtime can be configured to redact specific arg keys; check the user's `project.yml` for `redact:` patterns before quoting tool args back to them.

## When to use which

| Goal | Command |
|---|---|
| Watch an in-flight run | `zymi observe` (TUI run picker) or `zymi observe -r <stream-id>` |
| See "did the tool actually run?" | `zymi events --stream <id> --kind ToolCallCompleted` |
| Confirm a human approval went through | `zymi events --stream <id> --kind ApprovalGranted` |
| Audit a finished run end to end | `zymi verify --stream <id>` then `zymi events --stream <id> --raw \| jq .` |
| List recent runs | `zymi runs --limit 20` (optionally `-p <pipeline>`) |
| Re-run with edits | `zymi resume <id> --from-step <step>` |
