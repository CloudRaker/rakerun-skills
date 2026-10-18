---
name: rakerun-agents
description: |
  Write or change a cloudraker.run agent: the folder model, AGENTS.md prompts, every agent.yml key (mode, model, triggers, limits, network, secrets, connectors, data, knowledge, events, webhooks), inheritance from root to leaf, tools from the repo (skills, mcp.json, _apis, _code) and the lint rules a push enforces. Use when the user wants to author or edit an agent for CloudRaker Run.
allowed-tools:
  - Bash(rakerun *)
  - Bash(npx rakerun *)
---

# Writing an agent

Reference: https://docs.cloudraker.com/run/guides/agent-yml and https://docs.cloudraker.com/run/guides/repo-layout. Check `rakerun lint` after every edit.

## The folder model

```text
my-agents/
  AGENTS.md            # shared instructions, inherited by every agent (root to leaf, leaf last)
  agent.yml            # shared defaults (model, network, secrets), deep-merged, leaf wins
  skills/<name>/SKILL.md
  mcp.json             # remote MCP servers
  _schemas/lead.yml    # Structured Memory schemas, inherited
  knowledge/           # Markdown the agents search, never an agent
  linear-triage/       # the agent "linear-triage"
    AGENTS.md
    agent.yml
    _apis/linear.openapi.json
    _code/label.ts
```

Rules: a folder with `AGENTS.md` is an agent (R1); names match `[a-z0-9][a-z0-9-]{0,62}`, names with `_` are reserved (R2); an agent folder cannot hold another agent (R8, `nested_agent`); `knowledge/` never holds an agent (R10). Lists in `agent.yml` (`connectors`, `data.read`, `knowledge`, `model.fallback`) replace the parent list, maps deep-merge, scalars take the leaf. A root `agent.yml` cannot set a leaf-only key (`root_leaf_key`): `name`, `mode`, `idle_ttl`, `triggers`, `email`, `phone`, `output_schema`, `events`, `webhooks`, `protection`.

## AGENTS.md

The prompt. Plain Markdown, concatenated root to leaf. Say what the agent does, which tools to call and how to end:

```markdown
# Linear triage

You triage new Linear issues for the EGR team.
1. Read the issue with `mcp_linear_get_issue`.
2. Pick one label and one priority. Ask for a repro with `message` when it is missing.
3. End with `done` and `{ "label": ..., "priority": ... }`.
```

Built-in tools every agent has: `message` (text to the caller), `done` (the result, validated by `output_schema`), `ask` (`turns` mode only: a question the caller answers on the next turn), `list_files`, `read_file`, file tools (`file_read`, `file_parse`, `file_extract`, `file_redact`, `file_markdown_to_pdf`, `file_write`), `execute_code`, `save_code`. Name the tool in the prompt when the order matters.

## agent.yml, the keys that matter

```yaml
name: Linear triage          # display name, leaf-only
mode: zeroshot               # zeroshot (one request, one run) | turns (caller sends turns; ask tool; idle_ttl)
idle_ttl: 24h                # turns only, 1m to 30d
retention: 30d               # runs, events, files, 1d to 90d
timeout: 10m                 # one turn, 10s to 60m
model:
  id: anthropic/claude-sonnet-5-5   # provider/model
  temperature: 0.2
  max_output_tokens: 8000
  reasoning: medium          # low | medium | high
  allow_override: false      # true lets a run request pick model.id
triggers:                    # leaf-only; without the key, http is on
  http: true
  email: true                # {org}.{base}.{agent}@cloudraker.run, attachments become run files
  slack: true                # routed by slack.yml of the org routing repo
  schedule: '0 * * * *'      # cron, UTC, 1 minute minimum, skips while the previous tick runs
  inbound:
    linear: { events: [issue.created], secret: '${secret:LINEAR_WEBHOOK_SECRET}' }
input_schema: { type: object }   # 422 input_schema on a mismatch
output_schema: { type: object }  # leaf-only; the done payload
events:                          # leaf-only; each becomes a tool emit_{name}, the caller sees event.{name}
  label_suggested:
    description: Suggest one label.
    schema: { type: object, required: [label], properties: { label: { type: string } } }
webhooks:                        # leaf-only
  - url: https://ops.example.com/hooks
    events: [run.completed, run.failed, event.label_suggested]
    signing_secret: '${secret:HOOK_SECRET}'
network:                         # default deny; every host the agent reaches
  api.linear.app:
    headers: { Authorization: '${secret:LINEAR_TOKEN}' }   # secrets are filled here, and only here
  '*.slack.com': {}                                        # a wildcard host cannot carry headers
connectors: [linear]             # named MCP connectors of the org (see rakerun-integrations)
secrets:                         # declare every name you reference, else secret_undeclared
  LINEAR_TOKEN: { overridable: true }   # a run request may send a value for one turn
  HOOK_SECRET: {}
  LINEAR_WEBHOOK_SECRET: {}
limits: { max_steps: 60, max_code_calls: 40, code_cpu_ms: 30000, max_egress_bytes: 50MB }
data: { read: [lead], write: [lead], create_schemas: false }   # Structured Memory access
knowledge: [sales/pre-sales, legal]                             # folders under knowledge/, or ['*']
files: { read_raw: false }
protection: token                # token | public | access; public refuses secrets and model override
```

Formats: durations `10s 10m 24h 30d`, sizes `512KB 50MB`, secret references `${secret:NAME}` with `NAME` in `[A-Z][A-Z0-9_]*` (a bare `secret: NAME` fails with `secret_bare_name`), model ids `provider/model`.

## Tools from the repo

| Source | Tool the model sees |
| --- | --- |
| `skills/{name}/SKILL.md` | The skill, loaded on demand |
| `mcp.json` (remote servers) | `mcp_{server}_{tool}`; the host joins the network allow list |
| `_apis/*.json` or `*.yaml` (OpenAPI 3, first `servers[].url`, 5 MB) | `call_api` per `operationId`; `search_api` over 200 operations |
| `_code/{name}.ts` or `.js` | `{name}`, one fresh isolate per call |
| `events:` | `emit_{name}` |
| `knowledge:` | `knowledge_search`, `knowledge_read` |
| `data:` | `data.schemas`, `data.query`, `data.get`, `data.create`, `data.update`, `data.delete`, `data.upsert` |

A `_code` script exports `run(input)`. Its leading comment is the description; an `@input` line holds the input JSON Schema:

```ts
// Label a Linear issue.
// @input { type: object, required: [title], properties: { title: { type: string } } }
export function run(input) {
  return { label: input.title.includes('crash') ? 'bug' : 'triage' }
}
```

A script name cannot collide with a built-in tool or the prefixes `data_`, `file_`, `emit_`, `mcp_`, `knowledge_` (`code_tool_collision`).

## Checklist before `rakerun deploy`

1. `rakerun lint` is clean.
2. Every `${secret:NAME}` is declared under `secrets:` and set: `rakerun secret list`, `rakerun secret set NAME`.
3. Every host the agent calls is under `network:` (or comes from `_apis`, `mcp.json`, a connector).
4. Every name in `connectors:` exists, is granted and is `connected`: `rakerun connector list`.
5. `mode: turns` when the agent must ask questions; `zeroshot` otherwise.
6. After the deploy: `rakerun agent show <agent>` prints the resolved config, the tool list, the missing secrets and the deployments.

## See also

- [rakerun-runs](../rakerun-runs/SKILL.md) to test the agent
- [rakerun-integrations](../rakerun-integrations/SKILL.md) for secrets and connectors
- [rakerun-knowledge](../rakerun-knowledge/SKILL.md) for `_schemas/` and `knowledge/`
