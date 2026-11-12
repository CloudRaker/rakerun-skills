---
name: rakerun-agents
description: |
  Write or change a cloudraker.run agent: the folder model, AGENTS.md prompts, every agent.yml key (mode, machine, model, triggers, limits, network with oidc identity, secrets, connectors, data, knowledge, tools, events, webhooks, agents, workspace_act_as), inheritance from root to leaf, tools from the repo (skills, mcp.json, _apis, _code), team apps (apps/*.html and .yml, apps.yml, actions), the flag tool, runs on one record, machine agents (mode: machine, machine.json procedures, machine.tools.json pins, effects) and the lint rules a push enforces. Use when the user wants to author or edit an agent for CloudRaker Run. For web chat, phone, email, Slack or Teams, use rakerun-doors.
allowed-tools:
  - Bash(rakerun *)
  - Bash(npx rakerun *)
---

# Writing an agent

Reference: https://cloudraker-run.docs.buildwithfern.com/guides/agent-yml and https://cloudraker-run.docs.buildwithfern.com/guides/repo-layout. Check `rakerun lint` after every edit.

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

Rules: a folder with `AGENTS.md` is an agent (R1); names match `[a-z0-9][a-z0-9-]{0,62}`, names with `_` are reserved (R2); an agent folder cannot hold another agent (R8, `nested_agent`); `knowledge/` never holds an agent (R10); `doors/` holds the front doors and never an agent (R11, see [rakerun-doors](../rakerun-doors/SKILL.md)). Lists in `agent.yml` (`connectors`, `data.read`, `knowledge`, `model.fallback`) replace the parent list, maps deep-merge, scalars take the leaf. A root `agent.yml` cannot set a leaf-only key (`root_leaf_key`): `name`, `mode`, `machine`, `idle_ttl`, `triggers`, `output_schema`, `events`, `webhooks`, `protection`, `workspace_act_as`, `agents`.

## AGENTS.md

The prompt. Plain Markdown, concatenated root to leaf. Say what the agent does, which tools to call and how to end:

```markdown
# Linear triage

You triage new Linear issues for the EGR team.

1. Read the issue with `mcp_linear_get_issue`.
2. Pick one label and one priority. Ask for a repro with `message` when it is missing.
3. End with `done` and `{ "label": ..., "priority": ... }`.
```

Built-in tools every agent has: `message` (text to the caller), `done` (the result, validated by `output_schema`), `ask` (`turns` mode only: a question the caller answers on the next turn), `list_files`, `read_file`, file tools (`file_read`, `file_write`, `file_replace`, `file`), and the Paperwork tools when the run holds a file grant (`file_parse`, `file_extract`, `file_redact`, `file_markdown_to_pdf`, `file_form_fields`, `file_fill`, `file_classify`, `file_split`, `file_redline_read`, `file_redline`, `file_convert_pdf`, `file_merge`, `file_render_page`, `file_sign_send`, `file_sign_status`, `file_sign_void`), `execute_code`, `save_code`, `flag` (an Inbox item for a person or a team, see [below](#flag-a-person-or-a-team)). An agent with an `agents:` key also gets `delegate` and `message_agent` (see below). Name the tool in the prompt when the order matters.

## agent.yml, the keys that matter

```yaml
name: Linear triage # display name, leaf-only
mode: zeroshot # zeroshot (one request, one run) | turns (caller sends turns; ask tool; idle_ttl) | machine (see below)
idle_ttl: 24h # turns and machine only, 1m to 30d
retention: 30d # runs, events, files, 1m to 90d (1m: zero data retention)
timeout: 10m # one turn, 10s to 60m
model:
  id: anthropic/claude-sonnet-5-5 # provider/model
  temperature: 0.2
  max_output_tokens: 8000
  reasoning: medium # none | low | medium | high
  allow_override: false # true lets a run request pick model.id
triggers: # leaf-only; without the key, http is on. Not conversations.
  http: true
  schedule: '0 * * * *' # cron, UTC, 1 minute minimum, skips while the previous tick runs
  inbound:
    linear: { events: [issue.created], secret: '${secret:LINEAR_WEBHOOK_SECRET}' }
  paperwork: [sign.ended] # a new run when an envelope of an ended run ends; input { event, envelope_id, status, signers, signed_file_id?, from_run_id }
input_schema: { type: object } # 422 input_schema on a mismatch
output_schema: { type: object } # leaf-only; the done payload
events: # leaf-only; each becomes a tool emit_{name}, the caller sees event.{name}
  label_suggested:
    description: Suggest one label.
    schema: { type: object, required: [label], properties: { label: { type: string } } }
webhooks: # leaf-only
  - url: https://ops.example.com/hooks
    events: [run.completed, run.failed, event.label_suggested]
network: # default deny; every host the agent reaches
  api.linear.app:
    headers: { Authorization: '${secret:LINEAR_TOKEN}' } # secrets are filled here, and only here
  api.internal.acme.com:
    oidc: { identity: user } # a signed Run OIDC token instead of a secret (see below)
  '*.slack.com': {} # a wildcard host cannot carry headers or oidc
connectors: [linear] # named MCP connectors of the org (see rakerun-integrations)
# connectors:                    # or narrow the tools of one connector (fewer tokens per step)
#   - name: linear
#     tools: [list_issues, get_*]  # server tool names; * matches any characters
#   - name: crm
#     identity: user               # only for a Run identity (OIDC) connector; default agent
secrets: # declare every name you reference, else secret_undeclared
  LINEAR_TOKEN: { overridable: true } # a run request may send a value for one turn
  LINEAR_WEBHOOK_SECRET: {}
limits: { max_steps: 60, max_code_calls: 40, code_cpu_ms: 30000, max_egress_bytes: 50MB }
data: { read: [lead], write: [lead], create_schemas: false } # Structured Memory access
knowledge: [sales/pre-sales, legal] # folders under knowledge/, or ['*']
tools: { flag: off } # built-in tool switches: flag is on by default; inheritable, the leaf wins
files: { read_raw: false }
protection: token # token | public | access; public refuses secrets and model override
```

People reach agents by web chat, phone, email, Slack or Teams through a front door, `doors/{name}.yml`, never through `agent.yml`. The old keys (`email`, `slack`, `teams` or `phone` under `triggers`, and the `email:` and `phone:` blocks) fail lint with `agent_yml_schema`, and the message names the door key. There is no alias. See [rakerun-doors](../rakerun-doors/SKILL.md).

Formats: durations `10s 10m 24h 30d`, sizes `512KB 50MB`, secret references `${secret:NAME}` with `NAME` in `[A-Z][A-Z0-9_]*` (a bare `secret: NAME` fails with `secret_bare_name`), model ids `provider/model`.

## Signed identity on outbound requests (`network.{host}.oidc`)

An exact host can take an `oidc:` block. The egress layer then sends a short RS256 token (at most 300 s) to that host on each request: `Authorization: Bearer {token}` by default. The agent, the model and the code never see it. The receiving API verifies it against the org issuer `https://{org}.cloudraker.run` (JWKS at `/.well-known/jwks.json`). No secret is stored.

```yaml
network:
  api.internal.acme.com:
    oidc:
      identity: user # required: agent | user
      # audience: https://api.internal.acme.com   default; the only value for user
      # header: Authorization                      default; another header gets the bare token
  vault.acme.com:
    oidc: { identity: agent, audience: vault://run }
```

- `agent`: `sub` is `agent:{repo}/{agent}`. `user`: `sub` is `user:{workos_user_id}`, plus `email` and `act.sub` (the agent).
- The person of the turn is a verified org member only: console, team apps, and Slack or Teams senders mapped to a member. A child run uses the lead person. A mention run in a console conversation with no sender names the conversation owner. Schedules, webhooks, events, phone, email, web chat visitors and API tokens have no person.
- No person, or no token: a fetch, `_apis` or `_code` request fails with `identity_unavailable`, and the run goes on. An MCP server or connector cannot list its tools, so its tools are missing from the turn. A user rule never falls back to the agent identity.
- A preview run gets tokens with a `branch` claim.
- `oidc` deep-merges like the rest of `network:`. Lint runs on the resolved config of each agent.

| Lint                      | Level   | Rule                                                                                                              |
| ------------------------- | ------- | ----------------------------------------------------------------------------------------------------------------- |
| `oidc_wildcard`           | error   | `oidc` on a `*.` host                                                                                             |
| `oidc_user_audience`      | error   | `identity: user` with an audience other than `https://{host}`                                                     |
| `oidc_header_conflict`    | error   | `oidc` and `headers` set the same header on one host                                                              |
| `user_identity_source`    | error   | `identity: user` on a host or connector of a `protection: public` agent                                           |
| `user_identity_no_person` | warning | Every trigger of the agent has no person (schedule, inbound, phone, email), and a connector has `identity: user`. |
| `api_security_header`     | warning | An `_apis` file needs a header that `network:` lacks. An `oidc` header counts as supplied                         |

Reference: https://cloudraker-run.docs.buildwithfern.com/guides/oidc-identity.

## Agents that work together (`agents`)

Leaf-only. A lead agent gives a task to another agent of the same repo (`delegate`). The task runs as a **child run** of the lead run. An agent can also send a message to a live `turns` run of the same run tree (`message`). A run tree is the first run and every run that it started, at any depth. The caller lists its targets. The target agent needs no opt-in.

```yaml
# lead/agent.yml
mode: turns
agents:
  delegate:
    - research/web # a full agent path of the repo
    - agent: finance/analyst
      when: Numbers, forecasts and spreadsheets # a routing hint for the lead, max 200 characters
  message:
    - review/editor # must be a mode: turns agent
  limits: # optional
    max_depth: 2 # 1 to 3. Read from the root run only.
    children_per_turn: 5 # 1 to 10
    children_per_run: 10 # 1 to 25
    messages_per_tree: 50 # 1 to 200. Read from the root run only.
    tree_credits: 500 # optional. Read from the root run only.
```

An entry is a full agent path, `{agent, when}`, or `'*'` (every other agent of the repo, alone in its list). Each list takes at most 20 entries, and an agent appears once.

### Example: a lead and a researcher

```text
my-agents/
  lead/
    AGENTS.md
    agent.yml        # mode: turns, delegates to researcher
  researcher/
    AGENTS.md
    agent.yml        # mode: zeroshot, output_schema
```

```yaml
# lead/agent.yml
name: Research lead
mode: turns
agents:
  delegate:
    - agent: researcher
      when: One factual question that needs web sources
```

```yaml
# researcher/agent.yml
name: Researcher
mode: zeroshot
output_schema:
  type: object
  required: [answer, sources]
  properties:
    answer: { type: string }
    sources: { type: array, items: { type: string } }
network:
  '*.wikipedia.org': {}
```

```markdown
# Research lead

1. Split the question of the user into at most 3 factual questions.
2. For each one, call `delegate` with agent `researcher`. Put every fact the
   researcher needs in `context`: it never sees this conversation.
3. When the reports come back, write one answer and list the sources.
```

### How it works

- `delegate({agent, task, expected_output?, context?, data?})` returns `{child_run_id}` at once. `data` is the input of a target with `input_schema`. The child starts on the same deployment as the lead.
- **Join.** After the step, the lead ends its turn and waits (`turn.completed` with reason `children`). A waiting lead holds no concurrency slot. It resumes with ONE new turn when every child of that turn reports. That turn holds one report per child: the agent, the status and the result (the `done` result, else the last message, cut at 8,000 characters).
- A `zeroshot` child reports when it ends. A `turns` child reports when its first turn ends (status `answered`) and stays alive for messages. A child that ends a turn with `ask` reports `question`: the lead answers with `message_agent` to that child run.
- `message_agent({to, text, expect_reply?})`: `to` is a run id of the tree or an agent path (the newest live run of that agent in the tree). With `expect_reply: true`, the sender waits for the reply in the same join.
- **Messages need `turns` agents.** A message reaches a live `mode: turns` run of the same tree only. It never starts a run. A message to a `zeroshot` run fails with `not_turn_based` (a child to its parent is the exception). A lead can always message its own children, and a child its own parent. A message to any other run of the tree needs the target agent in `agents.message`.
- Agent text is data. A report or a message is never a person turn and cannot answer an approval. The approvals of a child go to a person: the root conversation and the Inbox.
- A child acts for the same person as the lead turn. The `workspace_act_as` identity of the lead does not flow to a child. When a lead run ends, every child that is still live ends too (`run.cancelled` with `by: parent`).
- `rakerun run children <id>` lists the child runs. `rakerun run list --root-only` hides them.

**Limits and billing.** By default a tree is 2 levels deep, a run starts at most 5 children per turn and 10 per run, and a tree sends at most 50 agent messages. `agents.limits` raises them to 3, 10, 25 and 200. One run sends at most 20 messages to one other run, and the same text twice in a row is dropped (`duplicate_message`). A child is a normal run: it bills its own minutes and AI usage, and counts against the daily run cap and concurrency. The lead shows its own credits and the tree total. `tree_credits` refuses new children when the tree total passes it.

Tool errors: `delegate` answers `agent_not_allowed`, `delegate_depth`, `delegate_limit`, `delegate_cycle`, `tree_credits`, `agent_unavailable` or `start_refused`. `message_agent` answers `agent_not_allowed`, `run_not_in_tree`, `no_live_run`, `not_turn_based`, `run_ended`, `message_limit` or `duplicate_message`.

| Rule                      | Level   | Check                                                                                                     |
| ------------------------- | ------- | --------------------------------------------------------------------------------------------------------- |
| `agents_target_unknown`   | error   | A `delegate` or `message` entry names no agent of the repo                                                |
| `agents_target_self`      | error   | An entry names the agent itself                                                                           |
| `agents_delegate_cycle`   | warning | The `delegate` entries form a cycle. A run refuses it with `delegate_cycle`                               |
| `agents_message_zeroshot` | warning | A `message` target is a `zeroshot` agent. The message always fails                                        |
| `agents_root_limits`      | warning | A `delegate` target sets `max_depth`, `messages_per_tree` or `tree_credits`. Only the root run reads them |
| `machine_target`          | error   | A `delegate` or `message` entry (also through `'*'`) names a `mode: machine` agent                        |

## Act as a person (`workspace_act_as`)

Leaf-only. The agent works in the Google Workspace or Microsoft 365 mailbox and calendar of one person. The key was `delegation:`. There is no alias: `delegation:` fails the schema.

```yaml
workspace_act_as:
  tools:
    mail: [read, draft] # read, draft, send, attachments
    calendar: [read, free_busy] # read, free_busy, write
    directory: [search]
  as: { from: header, allow: [acme.com] } # or a fixed person: as: lea@acme.com
```

`from: header` takes the person from the `x-run-as` header. The caller needs a token with the `act_as` scope, and the agent needs `protection: token`. `from: sender` takes the Slack, Teams or verified email sender of a conversation. Lint `act_as_identity_source` refuses `protection: public`, `from: sender` when no door with `channels.slack`, `channels.teams` or `channels.email` lists the agent, and `from: sender` in a door with `channels.email` and no `channels.email.allow_from`. Run errors start with `act_as_`, for example `act_as_identity_required` and `act_as_scope_required`.

## Flag a person or a team

The built-in `flag` tool puts an item in the Inbox of one person or one team, and the run continues. It never waits for an approval. Use it when a person must look at something, for example three documents that the agent cannot file.

```json
{
  "to": "reception",
  "text": "Three referrals match no intake.",
  "records": [{ "schema": "inbound_document", "id": "doc_8f2k" }],
  "urgency": "high"
}
```

- `to`: the email of one member of the organization, or one team slug. Another value answers `unknown_recipient`.
- `text`: 1 to 500 characters. The item title is the text, cut at 140 characters.
- `records`: at most 20 `{ schema, id }` refs. Each one must be a record that this run read or wrote, else `record_not_read`.
- `urgency`: `normal` (default) or `high`. A `high` item shows first in its Inbox group.
- A run sends at most 10 flags. The eleventh answers `flag_limit`.
- The item shows in **Needs you** for the named person, or for each member of the named team. It counts in the badge and ranks with approvals. It stays open after the run ends.
- A record ref shows its label only to a person who can open the record. Other people see "A record you cannot open".

The tool is on in every mode, also `zeroshot`. `tools.flag: off` removes it. The key is inheritable and the leaf wins, so a leaf `flag: on` turns the tool on again under a root `flag: off`. Name the tool in `AGENTS.md`: "When no intake matches a document, call `flag` for the `reception` team."

## Runs on one record

A run can work on one record, its focus. It then reads and changes only the focus and the records that the focus passes access to (see [rakerun-records](../rakerun-records/SKILL.md#linked-records)). The rules of the person and the `data` grant of the agent still apply.

- **Ask an agent** on a record page starts a conversation on that record. It lists the agents with `triggers.chat`, the person in their audience and a `data` grant on the record type. The conversation shows "Working on Intake INT-0042".
- An API caller with a `crk_` key sends the focus in the body of `POST /{base}/{agent}`: `{"input": "...", "focus": {"schema": "intake", "id": "INT-0042"}}`. The `data` read grant of the agent on the focus record type decides. An anonymous or Cloudflare Access caller that sends `focus` gets `400 invalid_body`.
- A caller who cannot open the focus gets `404 record_not_found`, and no run starts.
- A run on a lab result does not open its intake. Access passes from the intake to the lab result, not back.
- A new record must belong to the focus. When its record type has a `takes` field to the focus record type, Run fills it. When the focus record type has a `gives` field to its record type, Run adds the new id to the focus. Another create answers `focus_create`.
- A call outside the focus answers `focus_scope`: "This conversation works on Intake INT-0042. Start a new conversation for Intake INT-7."
- A continuation, a mention, a handoff and a child run keep the focus. They cannot clear it.
- No `agent.yml` key turns this on. Write `AGENTS.md` so that the agent works on the record of the conversation, for example: "Work on the intake of this conversation. Add each lab result to it."

## Tools from the repo

| Source                                                              | Tool the model sees                                                                                  |
| ------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| `skills/{name}/SKILL.md`                                            | The skill, loaded on demand                                                                          |
| `mcp.json` (remote servers)                                         | `mcp_{server}_{tool}`; the host joins the network allow list                                         |
| `_apis/*.json` or `*.yaml` (OpenAPI 3, first `servers[].url`, 5 MB) | `call_api` per `operationId`; `search_api` over 200 operations                                       |
| `_code/{name}.ts` or `.js`                                          | `{name}`, one fresh isolate per call                                                                 |
| `events:`                                                           | `emit_{name}`                                                                                        |
| `knowledge:`                                                        | `knowledge_search`, `knowledge_read`                                                                 |
| `data:`                                                             | `data.schemas`, `data.query`, `data.get`, `data.create`, `data.update`, `data.delete`, `data.upsert` |
| `data:` write on a filing queue and read on its case                | `data.file` (see [rakerun-records](../rakerun-records/SKILL.md#filing-queues))                       |
| A front door with more agents or languages                          | `handoff`, `switch_language` (see rakerun-doors)                                                     |
| A front door with `channels.phone`                                  | `end_call`, `transfer_call`, `call_phone` (with `outbound: true`)                                    |

A `_code` script exports `run(input)`. Its leading comment is the description; an `@input` line holds the input JSON Schema:

```ts
// Label a Linear issue.
// @input { type: object, required: [title], properties: { title: { type: string } } }
export function run(input) {
  return { label: input.title.includes('crash') ? 'bug' : 'triage' }
}
```

A script name cannot collide with a built-in tool (also `handoff`, `switch_language`) or the prefixes `data_`, `file_`, `emit_`, `mcp_`, `knowledge_` (`code_tool_collision`).

## Apps

An app is a small HTML page next to an agent. The team finds it in **Apps** in the console. It reads and changes Memory rows and starts agent work from buttons (actions). Git is the truth: developers write the files, and business users get the same files from **Share with team**.

```text
sales-assistant/
  apps/
    quote-tracker.html   # the app body, one self-contained HTML file, at most 200 KB
    quote-tracker.yml    # the app details, optional
apps.yml                 # root only, optional: categories and the Featured row
agent.yml                # root: access.publishers
```

An `.html` with no `.yml` is an app with defaults. The file name is the app name: `[a-z0-9][a-z0-9-]{0,59}`. An `apps/` folder never holds an agent. Start an app with `rakerun app new <agent> <slug>`.

### Project apps (`apps/<slug>.*` at the root)

The new default is a project app: `apps/<slug>.html` and `apps/<slug>.yml` at the project root. No agent owns it.

```text
apps/
  claims-board.html
  claims-board.yml
_schemas/
  claim.yml
claims/triage/
  agent.yml
```

- Every member sees and opens a project app. `audience` (`members`, `admins`, emails or team slugs) only narrows who sees it.
- `tables` names tables of the project (`_schemas`). No agent grant caps them. A table in `write` is also read. Lint `app_table_unknown` refuses a table that the project does not have.
- Each action names its `agent:`. An action with no agent is a lint error.
- The app shows only the records the viewer can open. The record rules of `_schemas/<name>.yml` `access` decide (see [rakerun-records](../rakerun-records/SKILL.md)).
- The Builder changes a project app. A publisher makes the change live.

### `<agent>/apps/<slug>.yml`

```yaml
id: app_01JBF3Q8T6ZK2M4N7R9V1X3Y5A # stable id for links, pins and state; keep it on a move
title: Quote tracker # 1..60 characters; default: the file name
description: See open quotes and ask for a follow-up. # at most 200 characters
icon: file-text # a lucide name from the closed list; default layout-grid
category: Sales # from apps.yml categories or the default list; default Other
audience: members # members | admins | [emails]; default: the agent audience
listed: true # false: hidden from Apps, still opens by link and pin
featured: false # shown in the Featured row
tables: # default: read = data.read + data.write, write = data.write
  read: [quote, lead] # within the agent data grants
  write: [quote] # within agent data.write
  delete: [] # within tables.write; default none
actions: # at most 20
  - id: follow_up # [a-z][a-z0-9_]{0,39}, one per app
    label: Ask for a follow-up # the button text, 1..40 characters
    description: Writes a follow-up email. It asks before it sends.
    agent: sales-assistant # default: the owner agent; an agent of this project
    prompt: | # the first message; {{field}} comes from the input
      Write a {{tone}} follow-up for quote {{quote_id}}.
    input_schema: # type object; string, number, integer, boolean fields
      type: object
      required: [quote_id]
      properties:
        quote_id: { type: string, title: Quote, x-run-row: quote } # a row picker
        tone: { type: string, enum: [short, warm], default: short }
    confirm: true # default true: the console asks before it starts
source: { ... } # written by Share with team; do not edit
```

App details only narrow access. For an app of an agent, the app audience is the agent audience AND `audience`. The app tables are the agent `data` grants AND `tables`. An action needs use of the app and use of the target agent. A string field takes `enum`, `format` (`date`, `date-time`, `email`) and `maxLength`. No `$ref` and no nested objects.

Root `apps.yml`:

```yaml
categories: [Claims, Underwriting, Renewals] # replaces Sales, Operations, Finance, Support, People
featured: [app_01JBF3Q8T6ZK2M4N7R9V1X3Y5A] # order of the Featured row, at most 12
```

Root `agent.yml`: `access: { publishers: [admins, lea@acme.com] }`. A publisher makes apps live (default `[admins]`). Other people who can use the agent can only ask to share.

### The bridge (`window.run`)

| Call                                                                                 | Does                                                                                                                                                                                                                                                         |
| ------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `await run.data.query(table, {filter, limit})`                                       | Rows of a table that the app reads                                                                                                                                                                                                                           |
| `await run.records.query(table, {filter, limit, id})`                                | `{rows, partial, truncated}`. `partial` is true when the viewer cannot open every record of the table. Show "Showing the records you can open." `truncated` is true when there are more records than `limit` (at most 200). `id` reads one record by its id. |
| `run.context.record`                                                                 | `{table, id}` of the record page that opened the app, or `null`                                                                                                                                                                                              |
| `await run.data.create(table, data)`, `update(table, id, data)`, `delete(table, id)` | Change rows. `delete` works only in a team app, for a table in `tables.delete`                                                                                                                                                                               |
| `await run.state.get()`, `run.state.set(json)`                                       | Small shared state for layout and settings, 256 KB. A stale write fails with "Someone changed this. Try again."                                                                                                                                              |
| `await run.actions.list()`, `run.actions.start(id, input)`                           | Team apps only. The person confirms, then a conversation of the target agent starts                                                                                                                                                                          |
| `run.user()`                                                                         | `{name, admin}`. No email                                                                                                                                                                                                                                    |
| `run.ask(text)`                                                                      | A message to the agent                                                                                                                                                                                                                                       |
| `run.on('data' or 'action' or 'theme' or 'size', cb)`                                | `data` comes after own writes, after an action ends, on tab focus and every 30 s. Query again in the callback                                                                                                                                                |

Rows go to Memory. State is not a database. Tool approvals inside an action conversation follow the agent `approvals` and land in the Inbox.

### App lint rules

| Rule                        | Level   | Check                                                                                                                        |
| --------------------------- | ------- | ---------------------------------------------------------------------------------------------------------------------------- |
| `app_details_invalid`       | error   | The `.yml` is not valid (the message names the key)                                                                          |
| `app_html_missing`          | error   | A `.yml` with no `.html` of the same name                                                                                    |
| `app_too_large`             | error   | The `.html` is over 200 KB                                                                                                   |
| `app_id_missing`            | warning | No `id:`. The message gives the id to paste. `rakerun lint` reads the repo from the origin remote; else pass `--repo <repo>` |
| `app_id_duplicate`          | error   | Two apps with one id                                                                                                         |
| `app_audience_wider`        | error   | `audience` is wider than the agent audience (apps of an agent)                                                               |
| `app_table_not_granted`     | error   | `tables.read` or `tables.write` is outside the agent grants (apps of an agent)                                               |
| `app_table_unknown`         | error   | A project app `tables` entry is not a table of the project                                                                   |
| `app_delete_not_write`      | error   | `tables.delete` is outside `tables.write`                                                                                    |
| `app_action_duplicate`      | error   | Two actions with one `id`                                                                                                    |
| `app_action_agent_unknown`  | error   | `actions[].agent` is not an agent of the project                                                                             |
| `app_action_agent_not_chat` | error   | The target agent has no `triggers.chat` and no `input_schema`                                                                |
| `app_action_input_invalid`  | error   | `input_schema` is outside the subset, or `x-run-row` names a table the app does not read                                     |
| `app_action_prompt_field`   | warning | `{{field}}` in `prompt` is not in `input_schema`                                                                             |
| `app_action_audience`       | warning | Fewer people can use the target agent than the app                                                                           |
| `app_category_unknown`      | warning | The category is not in `apps.yml` or the default list                                                                        |
| `app_slug_invalid`          | warning | The file name is not a valid app name                                                                                        |
| `apps_yml_invalid`          | error   | `apps.yml` is not valid YAML, fails the schema or is not at the root                                                         |
| `app_folder_not_agent`      | warning | An `apps/` file is not in an agent folder. Apps ignores it                                                                   |
| `access_not_root`           | error   | `access` in an `agent.yml` that is not the root                                                                              |

### App commands

```bash
rakerun app new sales-assistant quote-tracker --title "Quote tracker"   # .html starter + .yml with a fresh id
rakerun apps ls --agent sales-assistant                                 # the team apps you can use
rakerun app open app_01JBF3Q8T6ZK2M4N7R9V1X3Y5A                         # print and open the console URL
rakerun app run app_01JBF3Q8T6ZK2M4N7R9V1X3Y5A follow_up --input '{"quote_id":"q_1"}'   # token scope run
```

To share an app as a developer, commit and push (`rakerun deploy`). The CLI never asks to share.

## Machine agents (`mode: machine`)

Beta. A machine agent runs a compiled procedure (a Machine) step by step, with no model loop. Each tool call goes through the Run tools, grants, approvals and trace of the agent. Use it for a process that must run the same way each time (invoice intake, vendor onboarding, a claim check). Reference: https://cloudraker-run.docs.buildwithfern.com/guides/machine-agents. Language: https://docs.cloudraker.com/paperwork/v2/machines-language.

```bash
rakerun machine new ap/invoice-intake   # AGENTS.md, agent.yml, machine.json, machine.tools.json
```

```yaml
# ap/invoice-intake/agent.yml
mode: machine
triggers: { http: true }
machine: # leaf-only
  procedure: machine.json # default; machine.yaml is accepted
  tools: machine.tools.json # default; the pinned MCP and _code tool contracts
  exception_model: default # default | off (off: no model rung)
connectors: [erp]
approvals: { require: [send, delete, call] } # an irreversible step counts as call
# machine defaults: timeout 60m, idle_ttl 7d
```

- Not used in mode machine: `model`, `llm`, `limits.max_steps` and the `AGENTS.md` prompt (it describes the Machine for people). `output_schema` is refused: the result is always `{outcome, report, machine: {run_id, ir_hash}}`.
- Inputs: a string `input` is `user_message`; an object maps each key to a procedure input; the files of the first turn are `f1`, `f2`, ... (`{file_id, name, content_type, bytes}`).

### `machine.json` (`bm-procedure/2`)

```json
{
  "language": "bm-procedure/2",
  "procedure": "invoice-intake",
  "inputs": {
    "user_message": { "type": "string" },
    "f1": { "type": "file", "description": "the invoice PDF" }
  },
  "steps": [
    {
      "id": "extract",
      "tool": "file_extract",
      "args": {
        "file_id": "@input.f1.file_id",
        "schema": {
          "type": "object",
          "properties": { "vendor": { "type": "string" }, "total": { "type": "number" } },
          "required": ["vendor", "total"]
        }
      }
    },
    { "id": "totals", "check": "extract.value.total > 0", "message": "the total is positive" },
    {
      "id": "vendor",
      "tool": "mcp_erp_find_vendor",
      "args": { "name": "@extract.value.vendor" },
      "on_error": {
        "cases": [{ "code": "not_found", "then": { "hold": "The vendor is not in the ERP." } }]
      }
    },
    {
      "id": "bill",
      "tool": "mcp_erp_create_bill",
      "args": { "vendor_id": "@vendor.id", "amount": "@extract.value.total" },
      "limit": 1
    }
  ],
  "finish": { "status": "done", "report": "Bill {{@bill.bill_id}} is in the ERP." }
}
```

Step kinds: `tool`, `ask` (`rung: ["decide", "model", "person"]`), `code`, `check`, `when` (needs `else`), `foreach` (set `max_items`), `await`, `hold`, `finish` (`done`, `refused`, `aborted`). References (`@step.key`) are checked against the output schema of the tool. A write or an irreversible step that uses an extracted value needs a check before it. An irreversible step needs `limit: 1`. End each `rung` list with `person`: the `model` and `decide` rungs are not available in every environment.

Tool names in steps: `file_*`, `data_*` (from the `data:` grants), `knowledge_search` and `knowledge_read`, the `operationId` of an `_apis` operation (args `{params, body}`), a `_code` stem, `mcp_{server}_{tool}`. A Machine cannot call `message`, `done`, `ask`, `list_files`, `read_file`, `delegate`, `message_agent`, door or phone tools.

### `machine.tools.json`

Pin every MCP tool that a step calls, copied from the server listing with the same `inputSchema` and `outputSchema`. A `_code` pin is optional: it adds an output schema. Do not pin `file_*`, `data_*`, `knowledge_*` or `_apis` tools: Run gives their contracts.

```json
{
  "tools": [
    {
      "name": "mcp_erp_find_vendor",
      "inputSchema": {
        "type": "object",
        "properties": { "name": { "type": "string" } },
        "required": ["name"]
      },
      "outputSchema": {
        "type": "object",
        "properties": { "id": { "type": "string" }, "name": { "type": "string" } },
        "required": ["id", "name"]
      },
      "annotations": { "readOnlyHint": true },
      "x-errors": {
        "not_found": { "type": "object", "properties": { "code": { "const": "not_found" } } }
      }
    },
    {
      "name": "mcp_erp_create_bill",
      "inputSchema": {
        "type": "object",
        "properties": { "vendor_id": { "type": "string" }, "amount": { "type": "number" } },
        "required": ["vendor_id", "amount"]
      },
      "outputSchema": {
        "type": "object",
        "properties": { "bill_id": { "type": "string" } },
        "required": ["bill_id"]
      },
      "x-errors": {
        "duplicate": { "type": "object", "properties": { "code": { "const": "duplicate" } } }
      },
      "x-effect": "irreversible"
    }
  ]
}
```

`annotations.readOnlyHint: true` gives `read`, `destructiveHint: true` gives `irreversible`, else `write`. `x-effect` raises the effect, never lowers it. At the first turn, Run compares each pin with the live listing: a changed schema or a higher live effect fails the run with `machine_tools_changed`. There is no `rakerun machine tools pull` yet: edit the file by hand.

### Effects and approvals

| Run effect       | Machine effect |
| ---------------- | -------------- |
| `read`           | `read`         |
| `internal`       | `write`        |
| `external_write` | `write`        |
| `delete`         | `irreversible` |
| `send`           | `irreversible` |
| `call`           | `irreversible` |

The approval gate uses the stricter of the Run effect and the Machine effect: an irreversible step counts as `call`. With `require: [send, delete, call]` a person approves each irreversible step in the console or the Inbox. A denied call gives the step the error `approval_denied`, an edited approval gives `approval_edited` (the call does not run), and a conversation on web chat, email, Slack or Teams gives `approval_unavailable` (no person can approve there). A door with these channels can name a machine agent: each run of its conversations runs the Machine. The procedure handles these codes with `on_error`, or holds.

A write that may have run is never sent twice: after a crash a read runs again, and a write holds with `unknown_outcome` unless the step is `idempotent`.

### Machine lint

`rakerun lint` does not compile a procedure (the CLI has no Machines compiler). It prints a note. The push lint compiles each procedure, and the deploy compiles it again: a procedure that does not compile fails the deploy of the whole tree.

| Rule                        | Level            | Check                                                                     |
| --------------------------- | ---------------- | ------------------------------------------------------------------------- |
| `machine_procedure_missing` | error            | The procedure file is absent, or not JSON or YAML                         |
| `machine_compile`           | error or warning | One finding per compiler problem: `<code> <path>: <message>`, with line   |
| `machine_tool_unknown`      | error            | A step or a pin names a tool that the agent cannot have                   |
| `machine_effect_lowered`    | error            | A pin or a step lowers the effect of a tool                               |
| `machine_model_ignored`     | warning          | The leaf `agent.yml` sets `model`                                         |
| `machine_output_schema`     | error            | A machine agent sets `output_schema`                                      |
| `machine_mcp_output_schema` | error            | An MCP pin has no `outputSchema`                                          |
| `machine_target`            | error            | A phone door, `agents.message` or `agents.delegate` names a machine agent |

A machine agent is not a console chat target in this beta. Start it from the API, a trigger or `rakerun run start` (see [rakerun-runs](../rakerun-runs/SKILL.md#machine-agents)).

## Checklist before `rakerun deploy`

1. `rakerun lint` is clean.
2. Every `${secret:NAME}` is declared under `secrets:` and set: `rakerun secret list`, `rakerun secret set NAME`.
3. Every host the agent calls is under `network:` (or comes from `_apis`, `mcp.json`, a connector). A host with `oidc: { identity: user }` works only in turns of a verified person; test it from the console, not with an API token.
4. Every name in `connectors:` exists, is granted and is `connected`: `rakerun connector list`. Every name in a `tools:` list is a server tool that the connector allows: `rakerun connector tools <name>`. A wrong name fails the run with `connector_unavailable`.
5. `mode: turns` when the agent must ask questions, receive messages from other agents, or take calls in a door with `channels.phone`; `mode: machine` when a fixed procedure must do the work; `zeroshot` otherwise.
6. After the deploy: `rakerun agent show <agent>` prints the resolved config, the tool list, the missing secrets and the deployments.

## See also

- [rakerun-runs](../rakerun-runs/SKILL.md) to test the agent
- [rakerun-integrations](../rakerun-integrations/SKILL.md) for secrets and connectors
- [rakerun-knowledge](../rakerun-knowledge/SKILL.md) for `_schemas/` and `knowledge/`
- [rakerun-records](../rakerun-records/SKILL.md) for who can open records, linked records, teams, roles and filing queues
- [rakerun-doors](../rakerun-doors/SKILL.md) for front doors, channels and conversations
