# MCP and connectors

zymi has four external-system surfaces. They are not interchangeable — pick by direction and shape.

| Surface | Direction | What it carries |
|---|---|---|
| `mcp_servers:` | Pulled by an agent during a turn | Tools (Pinecone, GitHub, filesystem, …) |
| `connectors:` | Pushed _into_ a pipeline | Inbound events (Telegram messages, cron ticks, files, stdin) |
| `outputs:` | Pushed _out_ of a pipeline | Outbound side effects (HTTP POST, file append, stdout) |
| `zymi mcp serve` | Pulled by an **external** agent | This project's pipelines, exposed as MCP tools — see references/zymi-as-mcp-server.md |

This file covers the first three (zymi consuming the outside world). The server direction — another agent calling zymi pipelines — lives in **references/zymi-as-mcp-server.md**.

## MCP servers

See references/tools.md "MCP servers — N tools from one block" for the canonical example. Key points repeated:

- One block in `project.yml::mcp_servers:` registers N tools, prefixed `mcp__<server>__<tool>`.
- `allow:` whitelists tool names to import. `deny:` is the inverse. The two are mutually exclusive; omitting **both** loads zero tools from the server with a startup warning (opt-in UX — intentional).
- `env:` sets explicit environment variables for the subprocess. The parent process env is **not** inherited by default (security posture, ADR-0023).
- `init_timeout_secs` (default 10s) and `call_timeout_secs` (default 60s) — a server that fails to initialise drops its tools silently until restart. See references/troubleshooting.md.
- `cwd:` (per server) sets the working directory the server is spawned in. Relative paths resolve to project root, absolute paths are used as-is. Use this when the server expects to be launched from a sandbox / data directory.
- `requires_approval:` (per server, bool) forces approval on every tool imported from this server — useful for a "do not autopilot anything from this MCP" stance.
- `no_resume:` (per server) is a list of tool short-names (no prefix) that should be shadowed on fork-resume — see references/pipelines.md "Fork-resume".
- `restart:` (per server) — `{ max_restarts, backoff_secs }`. Reactive: a transport failure on the first call after a crash triggers re-spawn + re-handshake. Omitted = no auto-restart.

## Inbound connectors — fire a pipeline when something arrives

```yaml
# project.yml
connectors:
  - type: http_inbound          # listens on a port; common for webhooks
    name: github_hook
    host: "127.0.0.1"           # default; bind address
    port: 8090                  # required
    path: /github               # default "/"
    auth:                       # optional. Variants: none (default), bearer, hmac
      type: hmac
      header: X-Hub-Signature-256
      secret: ${env.GITHUB_WEBHOOK_SECRET}
      prefix: "sha256="
    extract:
      stream_id: "$.repository.full_name"
      content:   "$.action"
    pipeline: triage
    pipeline_input: payload

  - type: http_poll             # polls a URL on an interval
    name: telegram
    url: "https://api.telegram.org/bot${env.TELEGRAM_BOT_TOKEN}/getUpdates"
    interval_secs: 2
    extract:
      items:     "$.result[*]"
      stream_id: "$.message.chat.id"
      content:   "$.message.text"
    cursor:                     # mandatory for polling APIs that hand out cursors
      param: offset
      from_item: "$.update_id"
      plus_one: true
      persist: true
    pipeline: chat
    pipeline_input: message

  - type: cron                  # runs on a schedule
    name: daily_digest
    cron: "0 9 * * *"           # 5-field or 6-field croner syntax
    inputs:                     # fans out — one tick fires one PipelineRequested
      audience: ops
    pipeline: digest

  - type: file_read             # tails a file, one line per event
    name: ingest_log
    path: /var/log/intake.jsonl
    pipeline: process

  - type: stdin                 # one line per event from stdin
    name: cli_chat
    pipeline: chat
    pipeline_input: message
```

Each accepted event becomes a `PipelineRequested`. Run `zymi serve <pipeline-name>` (or `zymi serve --all`) to consume them.

### Auth, filters, cursors

`http_inbound` supports `auth:` with three variants: `none` (default), `bearer { token }`, `hmac { header, secret, prefix? }`. Validation runs before the body is parsed — a failing signature returns 401/403 and never publishes a `PipelineRequested`.

`http_poll` supports `filter:` (a map of `<json_path>: { one_of: [...] }` or `{ equals: ... }`) — use it to drop messages from unknown users _before_ they touch the agent. There is no `filter:` on `http_inbound`; for inbound webhooks, gate via `auth:` and inspect the payload inside the agent if needed.

### Multi-pipeline projects

`zymi serve foo bar` consumes connector events for pipelines `foo` and `bar` in one process. `zymi serve --all` consumes every pipeline in the project. This is the right shape for a project with both a chat pipeline and a cron-driven sync pipeline.

## Outbound outputs — send results out automatically

```yaml
# project.yml
outputs:
  - type: http_post
    name: telegram_reply
    on: [ResponseReady]
    url: "https://api.telegram.org/bot${env.TELEGRAM_BOT_TOKEN}/sendMessage"
    method: POST
    headers:
      Content-Type: "application/json"
    body_template: |
      {
        "chat_id": "{{ event.stream_id }}",
        "text": {{ event.content | tojson }}
      }
    retry:
      attempts: 3
      backoff_secs: [1, 5, 30]

  - type: file_append
    name: audit_log
    on: [PipelineCompleted]
    path: ./audit/runs.jsonl
    template: |
      {"ts": "{{ event.ts }}", "stream": "{{ event.stream_id }}", "ok": {{ event.success }}}

  - type: stdout
    name: console
    on: [ResponseReady]
    template: "{{ event.content }}"
```

`on:` filters by event kind — most chat-style bots subscribe to `ResponseReady`. The MiniJinja template has access to the event fields plus `tojson`, `shellquote`, and other safe filters.

Outputs are fire-and-forget from the pipeline's perspective; failures retry per `retry:` and emit `OutputDispatchFailed` after attempts are exhausted. Surface these in `zymi observe` / `zymi events`.

## Picking the right surface

- **The agent should _decide_ when to use it** → MCP or tool.
- **It happens _because something arrived_** → connector.
- **It happens _because the pipeline finished_** → output.

A common confusion: "I want the agent to message the user." That's almost always an **output** on `ResponseReady`, not a tool the agent calls. The pipeline produces a response; the output ships it. The agent doesn't need to know about transport.

Exception: a tool that posts to a _different_ channel than the user's (e.g. "broadcast to the team channel" while the user chats in DMs) — that's a tool, because the agent is choosing a side effect distinct from the normal reply path.
