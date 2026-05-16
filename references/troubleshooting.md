# Troubleshooting

Known papercuts and recoveries. Add to this file as new ones surface; keep entries terse — one line of symptom, one of cause, one of fix.

## `zymi events` shows nothing but I just ran a pipeline

**Symptom.** A run completed, `zymi observe` showed events live, but `zymi events --stream <id>` against the same SQLite store returns an empty or short list.

**Cause.** SQLite WAL mode. The writer hasn't checkpointed; a fresh reader doesn't see WAL pages.

**Fix.** Force a checkpoint and re-read:

```bash
sqlite3 .zymi/events.db 'PRAGMA wal_checkpoint(FULL);'
zymi events --stream <id>
```

Then move on. If this becomes routine, the user is on the wrong tool — switch to `zymi observe` or to a Postgres backend.

## MCP server's tools disappeared from the catalog

**Symptom.** Pipeline that used `mcp__<server>__<tool>` worked yesterday; today the planner errors with `unknown tool`. No code changed.

**Cause.** MCP server spawn missed `init_timeout_secs` (slow `npx -y` re-resolution on flaky DNS, server's own startup probe failed, …). zymi drops the server's tools from the catalog and **does not retry** until the process restarts.

**Fix.** Two parts:

1. Restart `zymi serve` (or whichever process owns the runtime) — that re-spawns the MCP server.
2. Switch the `command:` from `["npx", "-y", "@scope/server"]` to a globally installed binary (`npm i -g @scope/server`, then `command: ["/usr/local/bin/server-bin"]`). The `-y` form re-resolves on every spawn and is what makes initialisation flaky.

While debugging, raise `init_timeout_secs` on the server entry — defaults can be too tight for first-spawn cold caches.

## Pipeline silently hangs after an agent step

**Symptom.** `zymi run …` doesn't return. `zymi observe` shows the agent step in progress with no tool call event for a long time.

**Likely cause.** The agent called a `requires_approval: true` tool, the runtime emitted `ApprovalRequested`, no approval channel is configured (or the configured channel isn't reachable).

**Fix.** Check the event log:

```bash
zymi events --stream <id> | grep Approval
```

If you see `ApprovalRequested` and no resolution, configure or repair the approval channel in `project.yml::approvals:` and set `default_approval_channel:` (or `approval_channel:` on the pipeline). For interactive testing, the `terminal` channel is the simplest — it prompts on stdin.

## `zymi serve` rejects my pipeline with `unknown tool: my_python_tool`

**Symptom.** `zymi pipelines` lists the pipeline as valid, `zymi run` works, but `zymi serve` fails to start.

**Cause (historical, fixed in 0.6.2).** Python `@tool` discovery used to run only inside `Runtime::start`, so the load-time validator didn't know about Python tools. `zymi serve` validates before starting; the validator rejected names it couldn't resolve.

**Status.** Fixed in 0.6.2 (ADR-0025). If you still hit this on a recent version, confirm:

- The file is under `tools/` (or a directory the project scans).
- The function is decorated with `@tool` from `zymi`, not a local re-export.
- `python -c "import tools.my_python_tool"` succeeds outside zymi.

If all three pass and `zymi serve` still rejects the name, file a repro against zymi-core.

## Resume sent the email twice

**Symptom.** Forked a run from a step that contains `send_email`; the customer received the email a second time.

**Cause.** The tool was not registered `no_resume=True`. zymi resume re-executes every tool call in the re-executed subgraph by default — that's the contract.

**Fix.** Add the flag to the tool:

```python
@tool(no_resume=True)
def send_email(to: str, body: str) -> str: ...
```

For MCP-provided tools, declare per-server in `project.yml`:

```yaml
mcp_servers:
  - name: sendgrid
    command: [...]
    no_resume: [send_email, send_template]
```

Before resuming again, check that the `zymi resume` command output includes a `shadowed on resume (N):` section listing the protected tools. If it doesn't, the flag isn't taking effect.

## Context overflows on a long chat session

**Symptom.** After ~20 turns the agent's responses degenerate, latency spikes, prompt tokens climb past 50k.

**Cause.** Default `runtime.context:` is sized for long coding runs (10-turn observation window, 400k soft cap). For chat, tool_results dominate context size and never get masked before they pile up.

**Fix.** Override in `project.yml`:

```yaml
runtime:
  context:
    observation_window: 2
    soft_cap_chars: 40000
    hard_cap_chars: 80000
    min_tail_turns: 2
    forget_after_turns: 12
```

After the change, look for `ContextCompacted` events in the run — those confirm summarization is firing.

## `output step 'X' was skipped`

**Symptom.** Pipeline fails with `output step '<id>' was skipped` and exits non-zero.

**Cause.** The terminal step declared in `output: step:` had a `when:` that evaluated false, or was cascade-skipped because an ancestor was skipped. zymi refuses to silently substitute another step — the trace would be misleading.

**Fix.** Switch to `output: any_of: [a, b, …]` listing every legitimate terminal arm in priority order. The first non-skipped survivor becomes the answer. If _every_ candidate was skipped, the run still fails loud — that's the correct behaviour.

## "Tool returned not configured"

**Symptom.** `web_search` or `web_scrape` returns `"not configured"` literal string and the agent often hallucinates around it.

**Cause.** These ship as **stub placeholders** in scaffold templates. They're echo-only until the user wires a real provider.

**Fix.** Either:

- Replace with a real implementation: a `@tool` calling Brave / Serper / a custom search API, or attach a search MCP server.
- Or set the agent's system prompt to handle the stub gracefully: "If a tool returns 'not configured', answer from your own knowledge."

The scaffold's stubs are intentional — they're load-bearing for the `zymi init` happy path (no API keys required to see a pipeline run end-to-end).
