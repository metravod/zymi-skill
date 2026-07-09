# Pipelines

The pipeline YAML is the plan. Read this when the user is designing or editing `pipelines/<name>.yml`.

## Schema (essential subset)

```yaml
name: my_pipeline           # must match filename stem
description: "..."          # optional
inputs:                     # optional; declares ${inputs.*} the pipeline accepts
  - name: topic
    required: true

steps:                      # one or more
  - id: research            # unique within the pipeline
    agent: researcher       # OR `tool:` + `args:` — never both
    task: "Investigate ${inputs.topic}"
    depends_on: []          # default empty (root)
    when: "..."             # optional conditional edge (ADR-0028)
    context:                # optional per-step context overrides (ADR-0031)
      mode: fresh

output:                     # which step is "the answer"
  step: research            # OR `any_of: [a, b, …]` for routed pipelines

approval_channel: ops_tg    # optional; overrides project's default_approval_channel
```

## Three step kinds

Each step is **exactly one** of an agent step, a deterministic tool step, or an ask step. Declaring more than one, or none, is a parse error.

### Agent step

Runs an LLM ReAct loop bounded by the agent's `max_iterations`.

```yaml
- id: respond
  agent: assistant
  task: "${inputs.message}"
  depends_on: [...]
```

Use when the work needs reasoning, tool selection, or natural-language synthesis. The step's `output` is the agent's final assistant message.

### Tool step (ADR-0024)

Dispatches a tool directly with templated args, no LLM in the middle. The tool must exist in the catalog — a Python `@tool`, a declarative `tools/*.yml`, an MCP tool (e.g. `mcp__fs__read_text_file`), or a builtin (`read_file`, `write_file`, `execute_shell_command`, `write_memory`, `spawn_sub_agent`). There is no implicit `http_get` builtin — define a declarative HTTP tool first or use a Python `@tool`.

```yaml
- id: fetch                # uses a declarative tool defined in tools/get_user.yml
  tool: get_user
  args:
    id: "${inputs.id}"
  depends_on: []
```

Use when the work is deterministic glue: known input shape, known tool, no choice to make. Tool steps fail-fast (ADR-0027) — if the tool errors, the pipeline halts with non-zero exit. That is the contract; don't try to swallow errors.

Templated args resolve `${inputs.*}`, `${steps.<id>.output}`, `${env.*}`, and project-level `${variables.*}` on string leaves, then convert to JSON for catalog dispatch.

### Ask step (ADR-0042)

Delegates a reasoning question to **whoever called the pipeline** instead of configuring a second, separately-billed model. The run parks, surfaces the (templated) prompt, and resumes with the answer as the step's `output` — the approval mechanism (ADR-0022) generalized from a *decision* to a *value*.

```yaml
- id: summarize
  ask: "Summarize this deploy diff in two sentences:\n${steps.diff.output}"
  channel: mcp_reasoning     # optional; defaults to the connected caller under
                             # serve, else the zero-config terminal channel
  depends_on: [diff]
```

Use when the caller "could just answer this" (*summarize this*, *does this look right?*) and it's already a capable model. Key properties:

- **No `llm:` needed.** An `ask:` step does not count toward ADR-0041's "needs a provider" check — a pure tool + ask pipeline builds model-free.
- **Who answers:** under `zymi mcp serve`, the connected agent (the task goes `input_required` with `{ prompt, resume_token }`; the caller reasons and calls `zymi/reasoning/resume` — see references/zymi-as-mcp-server.md). Under `zymi run` / `zymi resume`, a human at the terminal or a configured reasoning channel.
- **Fails closed**, never a silent fallback to an HTTP model. Prompt + answer are recorded (`ReasoningRequested` / `ReasoningAnswered`), so replay reads the answer back and never re-asks (byte-identical).
- **The answer is untrusted** — model-generated free text. It flows to downstream `${steps.<id>.output}` through the *same* path as any tool output, so the same sink guards apply. Don't splice it into a raw `execute_shell_command` any more than you would a tool output.

## Interpolation

| Form | Resolved | Notes |
|---|---|---|
| `${inputs.<key>}` | Pipeline input | Set via `zymi run -i k=v` or connector `pipeline_input:` |
| `${steps.<id>.output}` | Upstream step's string output | The step **must** be in `depends_on`, else runtime error |
| `${env.<NAME>}` | Process / `.env` env var | |
| `${<var>}` | `project.yml::variables:` entry | Resolved at parse time, not runtime |

## Conditional branching (`when:`, ADR-0028)

A step with a falsy `when:` is skipped — its body never runs, a `StepSkipped` event is appended, the DAG topology is unchanged. **Cascade-skip**: descendants of a skipped step are also skipped (reason `"ancestor_skipped"`).

Grammar (intentionally tiny): `VALUE (==|!=) VALUE (&&|||) …`. `VALUE` is a bare token or single-quoted string. No parens, no regex, no `in`. If you find yourself needing more, route through an agent.

### Concierge / router pattern

The router agent picks a label by calling a tool; downstream steps gate on its output. `any_of:` picks the surviving branch as the pipeline answer.

```yaml
name: concierge
inputs:
  - { name: query, required: true }

steps:
  - id: router
    agent: router_agent       # prompt: "Call route('short' | 'rag')."
    task: "Pick a route for: ${inputs.query}"

  - id: short_answer
    agent: helper
    task: "Answer briefly: ${inputs.query}"
    depends_on: [router]
    when: "${steps.router.output} == 'short'"

  - id: rag_lookup
    tool: pinecone_query
    args: { query: "${inputs.query}" }
    depends_on: [router]
    when: "${steps.router.output} == 'rag'"

  - id: rag_answer
    agent: writer
    task: "Answer using: ${steps.rag_lookup.output}"
    depends_on: [rag_lookup]
    # No `when:` — cascade-skip handles it when rag_lookup was skipped.

output:
  any_of: [short_answer, rag_answer]    # first non-skipped wins
```

`any_of:` walks the list **in declared order**. If every candidate is skipped, the run fails loud (`all any_of outputs were skipped`) — silent fallback would defeat the trace.

## Per-step fresh context (ADR-0031)

`context: { mode: fresh }` on a step strips Layer A (observation history) from the prompt assembled for that step only. Use for:

- long-running goal loops where each iteration should not see the previous one;
- fan-out ETL steps where every shard processes independent data;
- summarizer steps that should look only at the current step's input, not the whole session.

It is **not** a global reset and not a memory-wipe — earlier steps' `${steps.X.output}` substitutions still flow through.

## Chat profile — `runtime.context:`

zymi's defaults (10-turn observation window, 400k soft cap, 600k hard cap) target long coding agents. For conversational use, override in `project.yml`:

```yaml
runtime:
  context:
    observation_window: 2          # mask tool_results older than 2 turns
    soft_cap_chars: 40000          # ~10k tokens — summarize before that
    hard_cap_chars: 80000          # fail loud if summarization can't fit
    min_tail_turns: 2              # always keep the last 2 turns full
    forget_after_turns: 12         # drop turns older than 12 entirely
```

Masking replaces old tool_results with `[masked: tool_name(arg) → ok, N chars]`. Summarization (`ContextCompacted` event) fires when the soft cap is hit. Hard cap is a guardrail — if even summarization can't fit, the step fails rather than silently truncating.

## Fork-resume

Every run is a stream of hash-chained events. To re-run with edits from a specific step:

```bash
zymi resume <stream-id> --from-step <step_id>
```

The runtime freezes events strictly upstream of the fork point, re-executes the fork step and its descendants, and writes a new sub-stream. **Side-effecting tools re-fire** in the re-executed subgraph unless the tool is registered `no_resume=True` — see references/tools.md.

The resume planner prints a `shadowed on resume (N):` summary if `no_resume` tools were skipped. If you see no shadow line and the tool had side effects (sent emails, charged cards), the side effects happened again.

## Validation expectations

zymi rejects at parse / load time:

- duplicate step ids;
- cycles in `depends_on`;
- `when:` syntax errors or references to non-ancestor steps;
- `output: step:` pointing at a non-declared step;
- `output: any_of:` with empty list, duplicates, or unknown steps;
- agent or tool name referenced but not in the catalog (Python `@tool` discovery counts here — `zymi serve` validates with discovery enabled).
