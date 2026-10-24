---
name: rakerun-runs
description: |
  Run cloudraker.run agents from the rakerun CLI: start a run with text or JSON input, stream the events, send turns and answers, list and filter the history, read a run, its trace and its messages, cancel. Use when the user wants to test, call or debug an agent.
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

A `mode: turns` agent ends each turn in `waiting`. Send the next one:

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

Statuses: `queued`, `running`, `waiting`, `completed`, `failed`, `cancelled`, `expired`. Failure codes: `model_error`, `timeout`, `step_limit`, `secret_missing`, `egress_denied`, `connector_unavailable`, `code_error`, `output_schema`, `concurrency_limited`, `credits_exhausted`, `insufficient_ai_credits`.

## Debug

```bash
rakerun run trace run_01…      # spans: run, turn, model, tool; id, parent, status, start, duration
rakerun run messages run_01…   # the transcript: system, user, assistant, tool calls and results per turn
rakerun run show run_01… --json | jq .usage   # tokens, model steps, code calls, egress, credits
```

Read the trace first on a failed run: the failing span carries the error. `secret_missing` means `rakerun secret set NAME`; `egress_denied` means the host is not under `network:`; `connector_unavailable` means `rakerun connector list` shows the connector missing, not granted, or not `connected`.

## From code

The CLI uses the console API with your profile token. An integration calls the public endpoint with a `run`-scoped token: `POST https://{org}.cloudraker.run/{repo}/{agent}` with `Authorization: Bearer crr_…`, `Accept: text/event-stream` for the stream, body `{ "input": …, "metadata": {…}, "secrets": {…} }`. The answer carries `x-run-id` and a read token `rrt_…` for `GET /_run/{id}`. Create the token with `rakerun token create <name> --scopes run,read --agents linear-triage`.
