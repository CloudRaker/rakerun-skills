# CloudRaker Run skills

Agent skills for [cloudraker.run](https://docs.cloudraker.com/run) and its `rakerun` CLI, in the [Agent Skills](https://agentskills.io) format. Available as plugins for Claude Code and OpenAI Codex.

This repository holds two skill packs:

| Pack | Plugin | Folder | For |
| --- | --- | --- | --- |
| Developer | `rakerun` | [`skills/`](./skills) | Coding agents: git, `agent.yml`, YAML and the `rakerun` CLI |
| Business | `rakerun-business` | [`business/`](./business) | Business builders: the agent does the steps, the person tries each change in the console |

## Install the CLI

```bash
npm install -g rakerun      # or: npx rakerun <command>
rakerun start               # sign up, create the org, a project from templates, deploy
rakerun login --org acme    # an existing org
```

`rakerun login` opens the browser. You approve the connection on the console page (it shows the same code as the terminal) and the token comes back to the terminal on its own. Use `--paste` on a remote shell, or `--token -` to read a token from stdin in CI. An agent in a sandbox asks you for the line the `/cli` page shows and passes it as `--token '<org> crr_...'`.

## Install the skills

```bash
rakerun skills add developer   # or: rakerun skills add business
```

Without a pack name, `rakerun skills add` installs the developer pack in a git repository and the business pack outside one. `--global` installs for all your projects. `rakerun start --skills developer|business|none` does the same at sign-up.

Or with the `skills` CLI, one folder for each pack:

```bash
npx skills add https://github.com/CloudRaker/rakerun-skills/tree/main/skills --skill '*'     # developer
npx skills add https://github.com/CloudRaker/rakerun-skills/tree/main/business --skill '*'   # business
```

Or as a Claude Code plugin:

```
/plugin marketplace add CloudRaker/rakerun-skills
/plugin install rakerun@rakerun-skills              # developer
/plugin install rakerun-business@rakerun-skills     # business
```

## Developer pack

| Skill | What it covers |
| --- | --- |
| [`rakerun`](./skills/rakerun) | Umbrella: install, login, the repo model, the deploy loop, where each task goes |
| [`rakerun-projects`](./skills/rakerun-projects) | `new`, `template`, `lint`, `deploy`, `repo`, environments (`env`, `push`, the GitHub Action), previews, rollbacks |
| [`rakerun-agents`](./skills/rakerun-agents) | Writing an agent: `AGENTS.md`, `agent.yml` keys, modes, triggers, tools from the repo, inheritance |
| [`rakerun-runs`](./skills/rakerun-runs) | Start, stream, turns, history, trace, messages, cancel, the event types, runs against conversations |
| [`rakerun-doors`](./skills/rakerun-doors) | Front doors (`doors/{name}.yml`): web chat, phone, email, Slack and Teams, guards, router, handoff, limits, bindings, embed keys, conversations |
| [`rakerun-integrations`](./skills/rakerun-integrations) | Secrets and `network:`, MCP connectors, tokens, webhooks |
| [`rakerun-knowledge`](./skills/rakerun-knowledge) | Structured Memory (`_schemas/`, `data:`, records) and `knowledge/` folders |
| [`rakerun-records`](./skills/rakerun-records) | Who can open records: the `access` block, relation fields (`takes`, `gives`, `access.links`, one link and no further), people teams, the three roles, filing queues with `data.file`, `rakerun lint --explain` |
| [`rakerun-onboarding`](./skills/rakerun-onboarding) | Set up an organization from sign-in to a first tested agent, with the console checklist |

### The house rules of the developer pack

1. **A folder with `AGENTS.md` is an agent.** The folder path is the agent name. `agent.yml` is optional and every key has a default.
2. **Ship with `rakerun deploy`, or `rakerun push <env>` for an environment.** `deploy` lints, commits everything and pushes. `push` lints the last commit and pushes it to one project (one project each for development, staging and production). The push is the deploy, all agents or none.
3. **Lint before you deploy.** `rakerun lint` runs the same rules as the gateway, in a second, with the file, the line and the rule id.
4. **Secrets are references.** Write `${secret:NAME}`, declare it under `secrets:`, set the value with `rakerun secret set NAME`. A push that holds a value fails.
5. **Templates first.** `rakerun template list` has 85 agents. Import one and edit it before you write from scratch.
6. **Stream, do not poll.** `rakerun run start <agent> --stream` prints the events as they arrive and ends on the terminal event.

## Business pack

| Skill | What it covers |
| --- | --- |
| [`rakerun-business`](./business/skills/rakerun-business) | Sign-in, the words to use, the steps of a change (check, save, try, make it live), assistants for chat, errors in plain words |
| [`rakerun-business-records`](./business/skills/rakerun-business-records) | Record types, "Linked to" fields and the direction in which they open records, who can open what, teams, roles, sharing, a queue for incoming documents |
| [`rakerun-business-onboarding`](./business/skills/rakerun-business-onboarding) | A first setup: the profile, "What are you here to build?", the structure, the access sentences, the first try |

The agent does every step. The person answers in plain words, tries each change in the console, and a publisher selects "Make it live". The agent never says git words to the person. See [business/README.md](./business/README.md).
