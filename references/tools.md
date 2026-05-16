# Tools

zymi has four tool kinds in one catalog. Pick the lowest-friction one for the job; agents and tool steps both dispatch through the same registry.

| Kind | Where | Use when |
|---|---|---|
| **Python `@tool`** | `tools/*.py` | You're writing the logic in Python, sync or async |
| **Declarative HTTP / shell** | `tools/*.yml` | The job is one HTTP call or one shell command; no Python needed |
| **MCP server** | `project.yml::mcp_servers:` | A third party exposes a fleet of tools (Pinecone, GitHub, filesystem, …) |
| **Builtin** | nothing — always present | `read_file`, `write_file`, `shell`, `write_memory`, plus stubs for `web_search` / `web_scrape` |

## Python `@tool`

```python
# tools/get_weather.py
from zymi import tool

@tool
def get_weather(city: str) -> str:
    """Return the current weather for a city."""
    return f"It's mild and partly cloudy in {city} today."
```

That's it. Auto-discovered at startup. The function name becomes the tool name, the docstring becomes the LLM-facing description, parameter type hints become the JSON schema.

### Async tools

```python
@tool
async def translate(text: str, target_lang: str) -> str:
    """Translate text into target_lang."""
    async with httpx.AsyncClient() as client:
        ...
```

zymi awaits async tools natively; do not wrap them in `asyncio.run`.

### Approvals

```python
@tool(requires_approval=True)
def deploy(service: str, version: str) -> str:
    """Deploy a service to production."""
    ...
```

On invocation the runtime publishes `ApprovalRequested` and pauses. A configured approval channel (`project.yml::approvals:`) collects the human decision and emits `ApprovalGranted` / `ApprovalDenied`. The tool body only runs after a grant.

### `no_resume` — protect side effects on fork-resume

```python
@tool(no_resume=True)
def send_email(to: str, body: str) -> str:
    ...
```

When a fork-resume re-executes a step that contains this tool, the runtime emits a synthetic placeholder result (`{"ok": true, "replayed": true, "note": "..."}`) instead of calling the function. Without this flag, resume will send the email again.

### `intention` — for policy / metrics

```python
@tool(intention="web_search")
def my_custom_search(q: str) -> str: ...
```

Tags the tool with a semantic category, surfaced in events and consumable by policy.

## Declarative HTTP / shell tools

When the work is a single HTTP call, skip Python entirely.

```yaml
# tools/weather.yml
name: weather
description: "Current weather for a city via wttr.in."
parameters:
  type: object
  properties:
    city: { type: string }
  required: [city]
implementation:
  kind: http
  method: GET
  url: "https://wttr.in/${args.city}?format=3"
```

Shell variant:

```yaml
# tools/broadcast.yml
name: broadcast
description: "Send announcement to ops channel."
parameters:
  type: object
  properties:
    message: { type: string }
  required: [message]
requires_approval: true     # default for kind: shell is true; spell it out anyway
implementation:
  kind: shell
  command_template: "echo 'BROADCAST: ${args.message}'"
```

`${args.X}` in `command_template` is **plain string replacement** — no MiniJinja, no filters, no `shellquote`. If you need to quote-safely embed an argument, restructure the tool: pass the variable parts through a Python `@tool` that uses `shlex.quote`, or write a dedicated shell script the tool invokes by path.

`policy.allow` / `policy.deny` in `project.yml` gates shell commands. By default `kind: shell` is approval-required — keep it that way unless you have an explicit reason.

## MCP servers — N tools from one block

Prefer MCP when a third party already publishes a server. One block, every tool the server exposes lands in the catalog prefixed with `mcp__<server>__*`.

```yaml
# project.yml
mcp_servers:
  - name: fs
    command:
      - /usr/local/bin/mcp-server-filesystem    # globally installed; not `npx -y`
      - ./sandbox
    allow: [read_text_file, write_file, list_directory]
    init_timeout_secs: 15
    call_timeout_secs: 30
    cwd: ./services/fs                          # optional; relative resolves to project root
    no_resume: [write_file]                     # tools that must not re-fire on resume
```

Wire the tools into an agent by their prefixed name:

```yaml
# agents/builder.yml
name: builder
tools:
  - mcp__fs__read_text_file
  - mcp__fs__write_file
  - mcp__fs__list_directory
```

Why not `npx -y …`: `-y` re-resolves the package on every spawn. On a flaky network or a missed `init_timeout_secs`, the server fails to initialise and its tools silently disappear from the catalog until the runtime restarts. Pin via `npm i -g`.

## Builtin tools

Always available, no declaration needed:

- `read_file(path)` — gated by `contracts.file_write.allowed_dirs` (read-side reuses the same path roots).
- `write_file(path, content)` — gated by `contracts.file_write`.
- `execute_shell_command(command)` — gated by `policy.allow` / `policy.deny`. Runs through the persistent shell session pool (ADR-0015), so cwd/env persist across calls in the same step.
- `write_memory(name, content)` — convenience for `./memory/<name>`.
- `spawn_sub_agent(agent, task)` — invoke another configured agent from inside a turn. Advanced; in most cases a multi-step pipeline is the better shape.

Note: `web_search` and `web_scrape` are **not** builtins. The scaffold ships them as declarative `kind: shell` stubs in `tools/` that echo `"not configured"`. The file contains commented blocks for Brave / Serper / Google CSE / Tavily — uncomment one and add the API key, or delete the file if you don't need search.

## When the agent doesn't pick the right tool

Three levers, in order of preference:

1. **Tool name and docstring.** The LLM sees the name and description. `query_pinecone_for_doc_chunks` is better than `search`.
2. **Agent system prompt.** Add one sentence per tool: "Use `weather` for city names. Use `forecast_api` for postal codes."
3. **Tool list.** Narrowing `agents/<name>.yml::tools:` to a smaller set forces selection. The agent can't call what's not in its list.

Do not engineer prompts that pretend to constrain tool choice ("you must call X first") — the model often ignores it. Narrow the catalog instead.
