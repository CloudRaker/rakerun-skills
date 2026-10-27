---
name: rakerun-doors
description: |
  Put cloudraker.run agents in front of people: front doors (doors/{name}.yml with entry, handoff agents, languages, variables, input and output guards, router, session, limits, alerts and channels), channel bindings (phone numbers, email aliases, Slack workspaces and channels, Teams tenants), web chat embed keys and the hosted page, the kill switch, and conversations (start, send, stream, end). Use when the user wants a web chat, a phone line, an email address, a Slack or Teams bot, or a test chat for their agents on CloudRaker Run.
allowed-tools:
  - Bash(rakerun *)
  - Bash(npx rakerun *)
---

# Front doors and conversations

Reference: https://docs.cloudraker.com/run/guides/front-doors and https://docs.cloudraker.com/run/guides/doors-yml. Check `rakerun lint` after every edit.

A front door is a file `doors/{name}.yml` at the root of a Run repo. It says how people reach the agents of the repo. Its id is `{repo}/{name}`. A person who comes in through a door starts a **conversation** (`cv_...`). The conversation starts runs for the agents of the door.

```text
Bindings (org state)   phone number, email alias, Slack workspace or channel, Teams tenant or channel, web embed key
  -> Front door (git)  doors/{name}.yml
     -> Conversation   cv_...: the person, the transcript, the current agent and run
        -> Runs        run A (entry) -> run B (handoff) -> run B' (continuation)
```

Web chat, phone, email, Slack, Teams and the console are channels of the same model. `agent.yml` has no channel keys: `email`, `slack`, `teams` and `phone` under `triggers`, and the `email:` and `phone:` blocks, fail lint. `triggers.http`, `triggers.schedule` and `triggers.inbound` stay in `agent.yml`. Their runs are not conversations.

## A door

```yaml
# doors/support.yml
name: Acme support                 # display name
entry: support/desk                # or a map: { en: support/desk, fr: support/accueil }
agents:                            # the handoff set; the entry agents are in it
  - agent: billing
    when: Invoices, payments, refunds and plan changes.    # the router and the agents read it
  - agent: support/tech
    when: Errors, outages and questions about the API.
languages: [en, fr]                # 2 or more: the agents get switch_language
variables:                         # typed session values that the client sets; no string type
  simple: { type: boolean, default: false }
guards:
  input:                           # each message of the person, before any model call
    - kind: pii                    # fixed patterns, then a model; types default to all nine
      message: Please do not send personal details in this chat.
    - kind: decision
      name: medical_advice
      statement: The message asks for a diagnosis or a treatment.
      when_true: The person asks what illness they have, or what medicine to take.
      when_false: The person asks about opening hours, prices or an appointment.
  files: refuse                    # with input checks, files need allow
  output:                          # each reply, before the person sees it
    hold: reply                    # reply | sentence
    strikes: 3
    checks:
      - kind: regex
        name: card_number
        pattern: '\b(?:\d[ -]?){13,16}\b'
        rewrite: Never write a card number.
router:
  rules:
    - keywords: [invoice, refund, facture]
      agent: billing
session: { idle_ttl: 30m, retention: 7d, max_turns: 200 }
limits:
  daily_conversations: 500
  per_participant: { conversations_per_day: 5, turns_per_minute: 10 }
alerts:
  webhook: ${secret:INCIDENT_WEBHOOK_URL}   # an org secret; error-rate alert, incident.io body
channels:
  web:
    origins: [https://www.acme.com]         # https://host[:port] or https://*.domain, no path
    greeting: Hello. How can we help?
  phone:
    greeting: 'Hello, you reached Acme support. How can I help?'
    transfers:
      sales: { number: '+15145550123', label: 'the sales team' }
  email:
    allow_from: ['@acme.com']
  slack: { follow_ups: true }
  teams: {}
  console: {}                               # test chat in the console and the CLI
```

Rules to keep in mind:

- Every agent path names an agent of the same repo. A router rule agent must be in the handoff set.
- An agent of a door with `channels.phone` must be `mode: turns`. Put the phone settings under `channels.phone`, never in `agent.yml`.
- `lang` is reserved. Variables are `boolean`, `number` (`min`, `max`) or `enum` (`values`).
- A wildcard origin on a shared host suffix (`*.vercel.app`, `*.github.io`, `*.pages.dev`, `*.workers.dev`, `*.com`) fails lint.
- `guards.output` with `channels.phone` is a warning: the voice speaks before the check.
- On every channel but `console`, no person can approve a tool call. Approvals that need a person fail with `approval_unavailable` (warning `door_agent_approvals`).
- `doors/` never holds an agent. A door name matches `[a-z0-9][a-z0-9-]{0,62}`.

Lint rules: `door_invalid`, `door_name_invalid`, `door_unknown_agent`, `door_router_agent_not_in_set`, `door_entry_lang_unknown`, `door_entry_lang_missing` (W), `door_variable_reserved`, `door_variable_default`, `door_variable_type`, `door_regex_invalid`, `door_check_name_duplicate`, `door_origin_invalid`, `door_origin_shared_suffix`, `door_theme_invalid`, `door_phone_output_guard` (W), `door_phone_needs_turns`, `phone_number_format`, `phone_text_length`, `phone_voice`, `phone_durations`, `door_classify_no_when` (W), `door_no_channel` (W), `door_alert_secret`, `door_agent_approvals` (W). A root routing file for Slack or Teams fails with `legacy_routing_file`.

## What happens in a conversation

1. The first message: the router picks the agent (rules, then a classifier on the `when` texts, else the entry). The input checks run at the same time. A refused message never reaches a model and is not kept.
2. The agent run starts with the transcript. The text streams to the person (unless output checks hold it).
3. The next messages go to the same run while it waits. After the run ends or expires, the next message starts a new run of the same agent with the transcript.
4. `handoff(to, reason)` ends the run (`run.cancelled` by `handoff`) and starts the target agent with the transcript. At most 3 handoffs per message. The person stays on the same connection.
5. `switch_language(to)` sets `lang`. With an `entry` map, it hands off to the agent of that language.
6. The conversation ends on `end`, `session.idle_ttl`, `max_turns`, a cap, the kill switch or the output strikes. After `session.retention`, the content and the identity of the person are deleted. A stub with counts and cost stays 90 days.

The run input gets one line `Session: lang=fr; simple=true` with the session values. It is never in the transcript.

`metadata.end_user` comes from the person only: stable for a verified email, Slack, Teams or console user, else one value per conversation (`phone:cv:{cv}`). Two calls from the same number never share records.

## Bindings

Bindings are org state. One key binds one door. Unbind before you move it. You need the `admin` role.

```bash
rakerun channel list
rakerun channel bind phone +15145550100 --door support/support
rakerun channel bind email help --door support/support        # {org}.help@cloudraker.run
rakerun channel bind slack T0123ABCD --door support/support    # workspace default
rakerun channel bind slack T0123ABCD/C0456EFGH --door support/billing   # a channel wins
rakerun channel bind teams <tenant_id> --door support/support
rakerun channel unbind email help
```

- Buy phone numbers on the console **Phone** screen. The number list shows the door of each number.
- Install the Slack or Teams app once on the console **Slack** or **Teams** screen, then bind.
- A binding to a missing door, or to a door without that channel, answers `door_not_found`. The **Channels** page shows it in red.
- Only the live deployment of the default branch has doors. Previews have none.

## Web chat

```bash
rakerun embed create --door support/support --label Website            # prints the script tag and the React snippet
rakerun embed create --door support/support --hosted --label "Hosted"  # also https://{org}.cloudraker.run/_chat/{repo}/{name}
rakerun embed list --door support/support
rakerun embed revoke rek_...                                           # its open conversations end
```

```html
<script src="https://acme.cloudraker.run/_chat/assets/run-embed.js" defer></script>
<cloudraker-chat base-url="https://acme.cloudraker.run" embed-key="rek_..."></cloudraker-chat>
```

React: `import { RunChat } from '@cloudraker/run-embed/react'`. An embed key is public and works only from `channels.web.origins`. The chat runs a Turnstile check, then holds a `cvs_` session token for one conversation. The browser never gets a `crr_` token. Theme, gate (a notice to accept), greeting, files and custom events are under `channels.web`.

## Test and operate

```bash
rakerun door list                                              # doors, channels, switch, counts of today
rakerun door show support/support                              # bindings, embed keys, lint issues, YAML
rakerun conversation start --door support/support --input "My invoice is wrong" --stream   # needs channels.console
rakerun conversation send cv_... --input "It is invoice 1042" --stream
rakerun conversation send cv_... --question q_... --answer yes
rakerun conversation stream cv_... --after-seq 12               # WebSocket, SSE fallback
rakerun conversation list --scope agents --door support/support --channel web
rakerun conversation show cv_...
rakerun conversation end cv_...
rakerun door switch support/support --on --message "Back at 9 AM."   # kill switch: open conversations end
rakerun door switch support/support --off
```

`conversation start` uses the console channel: the door needs `channels.console: {}`. A chat with one agent in the console uses an implicit door (that agent, no guards, no limits).

Conversation errors: `door_not_found`, `door_unavailable` (kill switch), `at_capacity` (a door limit, with the door message), `conversation_ended`, `turn_in_flight` (one message at a time), `invalid_item` (an answer not in the choices, a variable out of range), `unsupported_item` (a file on a door that refuses files), `binding_taken`.

## Checklist

1. The agents of the door exist and `rakerun lint` is clean. Phone doors: every agent is `mode: turns`.
2. Each agent that can take over has a `when` text, and the entry agent knows the others exist.
3. A public door (web, phone, email) has input checks and limits. Personal data checks: `kind: pii`.
4. `rakerun deploy`, then bind each channel or create the embed key.
5. Test with `channels.console` and `rakerun conversation start`, then on the real channel.

## See also

- [rakerun-agents](../rakerun-agents/SKILL.md) to write the agents of the door
- [rakerun-runs](../rakerun-runs/SKILL.md) to read the runs of a conversation
- [rakerun-integrations](../rakerun-integrations/SKILL.md) for the alert webhook secret
