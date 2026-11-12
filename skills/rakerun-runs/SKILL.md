---
name: rakerun-runs
description: |
  Run cloudraker.run agents from the rakerun CLI: start a run with text or JSON input, stream the events, send turns and answers, list and filter the history, read a run, its trace and its messages, cancel. Explains runs against conversations, and how to run, answer and debug a machine agent (mode: machine). Use when the user wants to test, call or debug an agent.
allowed-tools:
  - Bash(rakerun *)
  - Bash(npx rakerun *)
---

# Runs

## Start and stream

```bash
rakerun run start linear-triage --input "Triage EGR-123" --stream
echo '{"ticket": "EGR-123"}' | rakerun run start linear-triage --stream     # JSON on stdin
rakerun run start linear-triage --data '{"ticket": "EGR-123"}'             # 202, prints the run id
rakerun run start triage --repo agents --branch try-it --stream            # a repo-qualified agent on a preview
```

`--stream` prints the events as they arrive: `message` text as is, `· progress` on stderr, `✓ completed` with the `done` result as JSON, `✗ failed: <code>: <message>`, a `?` question with the turn command to answer it. A lead agent (one with `agents:`) also prints its run tree on stderr: `→ delegated to <agent> (<run>)`, `← <agent>: <status> <summary>`, `✉ to <agent>: <text>` and `✉ from <agent>: <text>`. It ends on the terminal event. Without `--stream`, follow later with `rakerun run stream <id>` (replays from the first event).

`--json` prints one JSON event per line, the raw envelopes (`id`, `seq`, `type`, `ts`, `run_id`, `agent`, `turn`, `data`).

Event types: `run.started`, `turn.started`, `message`, `question`, `file`, `run.progress`, `data.changed`, `event.{name}` (custom), `child.started`, `child.reported`, `agent_message.sent`, `agent_message.received`, `turn.completed`, `run.completed`, `run.failed`, `run.cancelled`, `run.expired`.

A lead that delegated ends its turn with `turn.completed` reason `children` and waits. It resumes with one new turn when every child of that turn reports. A child that ends because its parent ended has `run.cancelled` with `by: parent`.

## Turns

A `mode: turns` agent ends each turn in `waiting`. A `mode: machine` agent waits in the same way for an answer or an input. Send the next one:

```bash
rakerun run turn run_01… --input "Yes, close it" --stream
rakerun run turn run_01… --question q_7 --input "blue"      # answer an ask
```

The stream of a turn ends at `turn.completed`. A `waiting` run expires after `idle_ttl` (24h by default).

## History

```bash
rakerun run list --limit 20
rakerun run list --agent linear-triage --status failed --from 2026-10-01T00:00:00Z
rakerun run list --cursor <next cursor>     # printed at the end of a page
rakerun run list --root-only                 # no child runs
rakerun run list --parent run_01…            # the child runs of one run (the PARENT column)
rakerun run children run_01…                 # the same, shorter
rakerun run show run_01…                     # status, turn, deployment sha, usage, result, error
rakerun run cancel run_01…
```

Statuses: `queued`, `running`, `waiting`, `completed`, `failed`, `cancelled`, `expired`. Failure codes: `model_error`, `timeout`, `step_limit`, `secret_missing`, `egress_denied`, `connector_unavailable`, `code_error`, `output_schema`, `concurrency_limited`, `credits_exhausted`, `insufficient_ai_credits`, and for machine agents `machine_held`, `machine_error`, `machine_tools_changed`.

## Debug

```bash
rakerun run trace run_01…      # spans: run, turn, model, tool; id, parent, status, start, duration
rakerun run messages run_01…   # the transcript: system, user, assistant, tool calls and results per turn
rakerun run show run_01… --json | jq .usage   # tokens, model steps, code calls, egress, credits
```

Read the trace first on a failed run: the failing span carries the error. `secret_missing` means `rakerun secret set NAME`; `egress_denied` means the host is not under `network:`; `connector_unavailable` means `rakerun connector list` shows the connector missing, not granted, or not `connected`.

## Machine agents

A `mode: machine` agent (see [rakerun-agents](../rakerun-agents/SKILL.md#machine-agents-mode-machine)) runs like other agents. Start it from the CLI, the API or a trigger. The console does not list it as a chat target in this beta.

```bash
rakerun run start ap/invoice-intake --input "File this invoice" --stream             # input is user_message
rakerun run start ap/invoice-intake --data '{"vendor": "Acme", "po": "PO-7"}' --stream # each key is a procedure input
```

- Each tool call of the Machine is one `run.progress` work row (call id `mc_<seq>`) and one tool span. A model, decide or code step of the engine is one `machine` span in `rakerun run trace`.
- An `ask` step or a failed step that reaches a person sends a `question` with the id `mq_<seq>` and opens an Inbox item. Answer it with the question id: `rakerun run turn run_01… --question mq_12 --data '"billing"'`. A value that does not match the answer schema does not end the wait: the run sends a message and keeps waiting.
- An `await` step waits for a later input: `rakerun run turn run_01… --input "Here is the PO number: PO-7"`.
- A gated call waits for an approval. A person approves or denies it in the console or the Inbox. A denied call gives the step the error `approval_denied`.
- The run completes on `done`, `refused` and `aborted`. The report is the last `message`, and the result is `{outcome, report, machine: {run_id, ir_hash}}`. Check `result.outcome`: a refused claim is a completed run.
- A waiting machine run expires after `idle_ttl` (7d by default). `timeout` (60m by default) limits one turn.
- `rakerun run cancel` also stops the Machine. A done write is not undone: the trace lists the calls that ran.

Failure codes of machine runs: `machine_held` (a `hold` step, or an error that the procedure holds; the Inbox item has the reason), `machine_error` (the engine stopped, or the Machine service was not available) and `machine_tools_changed` (a pinned MCP tool changed since the deploy; copy the new contract into `machine.tools.json` and deploy again). A machine run is billed by the Run meter (minutes), as other runs.

A machine agent cannot take an agent message: `message_agent` to its run answers `not_turn_based`.

## Runs and conversations

A **run** is one agent, one execution: the unit of the trace and of billing. `rakerun run start`, an HTTP call, a schedule, an inbound webhook and a delegation start runs. They are not conversations.

A **conversation** (`cv_...`) is one exchange with one person through a front door: web chat, phone, email, Slack, Teams or the console. It starts runs for its agents: a `main` run, then a `handoff` run when an agent passes the conversation on, and a `continuation` run when the person writes after a run ended. Each run of a conversation has `metadata.conversation_id`, `door`, `channel` and `end_user`.

- Test a door with `rakerun conversation start --door <repo>/<name> --input ... --stream`, not with `run start`. See [rakerun-doors](../rakerun-doors/SKILL.md).
- `rakerun conversation show <cv>` lists the runs of the conversation. Debug one run with `rakerun run trace <run>` and `rakerun run messages <run>`.
- A run that a handoff ended has `run.cancelled` with `by: handoff`. A run of an ended conversation has `by: conversation`.
- The door `session.retention` replaces the agent `retention` for the runs of its conversations.

## From code

The CLI uses the console API with your profile token. An integration calls the public endpoint with a `run`-scoped `crk_` key: `POST https://{org}.cloudraker.run/{repo}/{agent}` with `Authorization: Bearer crk_…`, `Accept: text/event-stream` for the stream, body `{ "input": …, "metadata": {…}, "secrets": {…} }`. The answer carries `x-run-id` and a read token `rrt_…` for `GET /_run/{id}`.

Create the key of an integration on the API keys page of Dash (`https://dash.cloudraker.com/api-keys`). Select the Live environment, because Run refuses a test key. Select only the scopes `run` and `read`. Give the agent list as full paths, one on each line (`{repo}/{agent}`). An empty list allows every agent. You cannot change the list after you create the key. Dash shows the value of a live key once, so put it in the secrets manager of the integration.

`rakerun token create --name <name> --scopes run,read` is only for a CLI profile on the same computer. It saves the key as the profile `<name>` (use it with `--profile <name>`) and prints only a hint, not the value. It cannot set an agent list.
