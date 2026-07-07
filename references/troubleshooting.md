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

## MCP host times out calling my pipeline tool

**Symptom.** A pipeline exposed via `zymi mcp serve` works under `zymi run`, but the host (Claude Code, Cursor) reports a tool timeout mid-run — usually during a long deterministic step (test suite, build).

**Cause.** A plain `tools/call` blocks until the pipeline terminates; the deadline that fires is the **host's** per-tool-call timeout, not anything in zymi.

**Fix.** Raise the host's timeout (Claude Code: `MCP_TOOL_TIMEOUT`, in ms, in the host's environment), or — if the host supports SEP-1686 tasks — have the caller task-augment the call so it returns a task handle immediately. `mode: async` on the exposure is the hint for that. See references/zymi-as-mcp-server.md.

## Approval denies immediately under `zymi mcp serve`

**Symptom.** A `requires_approval: true` tool worked with a telegram/terminal channel, but over MCP the action is denied instantly — no form, no prompt.

**Cause.** The connected MCP client doesn't advertise `capabilities.elicitation`, and the elicitation approval bridge **fails closed**: `ApprovalDenied { reason: "client_no_elicitation" }`.

**Fix.** Use an elicitation-capable host (Claude Code ≥ 2.1.76 renders the approve/deny form natively), or declare a non-elicitation channel (`telegram`, `http`) in `project.yml::approvals:` + `default_approval_channel:` — project-declared channels run as configured under serve.

## `requires_approval: true` never prompts

**Symptom.** A tool is marked `requires_approval: true`, the channel is configured, but the tool executes silently with no `ApprovalRequested` in the event log.

**Cause (historical, fixed in 0.7.0).** The per-tool flag was dead before zymi-core 0.7.0: it parsed but never gated HTTP/Python/MCP tools, and shell tools gated only via the policy allowlist.

**Fix.** Upgrade to ≥ 0.7.0 (`uv tool upgrade zymi-core`, then rerun `zymi fetch` in the project so the venv picks it up). On ≥ 0.7.0 the explicit flag forces a human approval round-trip for every tool kind.

## Telegram approval channel refuses to start on a non-loopback bind

**Symptom (0.8.0+, breaking).** A `telegram` approval channel that bound a public / non-loopback address and previously worked now hard-errors at startup, refusing to start.

**Cause (ADR-0037).** The approval callback endpoint approves any pending action for whoever reaches it. Before 0.8.0 `secret_token` defaulted to `None` and nothing enforced it, so an off-host bind was an unauthenticated approve-anything hole. Now it **fails closed**: a non-loopback bind without a `secret_token` is a startup error; a loopback bind without one only logs a warning (fine for local dev / behind a tunnel).

**Fix.** Set `secret_token:` on the telegram approval channel in `project.yml::approvals:`, or keep the recommended topology — bind loopback (`127.0.0.1`) and put a tunnel in front. No config **schema** changed; only this validation was added. (Relatedly in 0.8.0, a callback for an already-decided approval is now ignored — first decision wins — so the audit trail can't record two contradictory decisions for one request.)

## Provider 401, or a config value arrives as the literal `${env.KEY}`

**Symptom.** An LLM call fails with a cryptic provider 401 (bad API key), or a tool/connector receives the string `${env.SOMETHING}` verbatim instead of the resolved value.

**Cause (behavior changed in 0.7.1, issue #9).** Before 0.7.1, if **any** `${env.*}` reference in `project.yml` was unset — even in an unused connector or output block — `load_project` swallowed the resolution error and fell back to the **raw YAML**. That sent literal `${env.KEY}` strings downstream for the whole file, including `llm.api_key`, which surfaced as a 401 with no hint about the real cause.

**Fix.** Upgrade to ≥ 0.7.1. The first config pass is now env-only and **fails loudly**: a missing `${env.*}` is a hard error naming the variable and the file path. Set the variable (a `.env` file is picked up) or remove the dead reference. Note this pass only resolves `${env.*}` — `${var}`, `${project.*}`, `${inputs.*}` and other namespaces are still resolved in later passes and are unaffected.

## "Tool returned not configured"

**Symptom.** `web_search` or `web_scrape` returns `"not configured"` literal string and the agent often hallucinates around it.

**Cause.** These ship as **stub placeholders** in scaffold templates. They're echo-only until the user wires a real provider.

**Fix.** Either:

- Replace with a real implementation: a `@tool` calling Brave / Serper / a custom search API, or attach a search MCP server.
- Or set the agent's system prompt to handle the stub gracefully: "If a tool returns 'not configured', answer from your own knowledge."

The scaffold's stubs are intentional — they're load-bearing for the `zymi init` happy path (no API keys required to see a pipeline run end-to-end).
