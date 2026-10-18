# CloudRaker Run skills

Agent skills for [cloudraker.run](https://docs.cloudraker.com/run) and its `rakerun` CLI, in the [Agent Skills](https://agentskills.io) format. Available as a plugin for Claude Code and OpenAI Codex.

## Install the CLI

```bash
npm install -g rakerun      # or: npx rakerun <command>
rakerun start               # sign up, create the org, a project from templates, deploy
rakerun login --org acme    # an existing org
```

`rakerun login` opens the browser. You approve the connection on the console page (it shows the same code as the terminal) and the token comes back to the terminal on its own. Use `--paste` on a remote shell, or `--token -` to read a token from stdin in CI. An agent in a sandbox asks you for the line the `/cli` page shows and passes it as `--token '<org> crr_...'`.

## Install the skills

```bash
npx skills add cloudraker/rakerun-skills
```

Or as a Claude Code plugin:

```
/plugin marketplace add cloudraker/rakerun-skills
/plugin install rakerun
```

## Catalog

| Skill | What it covers |
| --- | --- |
| [`rakerun`](./skills/rakerun) | Umbrella: install, login, the repo model, the deploy loop, where each task goes |
| [`rakerun-projects`](./skills/rakerun-projects) | `new`, `template`, `lint`, `deploy`, `repo`, previews, rollbacks |
| [`rakerun-agents`](./skills/rakerun-agents) | Writing an agent: `AGENTS.md`, `agent.yml` keys, modes, triggers, tools from the repo, inheritance |
| [`rakerun-runs`](./skills/rakerun-runs) | Start, stream, turns, history, trace, messages, cancel, the event types |
| [`rakerun-integrations`](./skills/rakerun-integrations) | Secrets and `network:`, MCP connectors, tokens, webhooks |
| [`rakerun-knowledge`](./skills/rakerun-knowledge) | Structured Memory (`_schemas/`, `data:`, records) and `knowledge/` folders |
| [`rakerun-onboarding`](./skills/rakerun-onboarding) | Set up an organization from sign-in to a first tested agent, with the console checklist |

## The house rules these skills teach

1. **A folder with `AGENTS.md` is an agent.** The folder path is the agent name. `agent.yml` is optional and every key has a default.
2. **`rakerun deploy` is the only way to ship.** It lints, commits everything and pushes. The push is the deploy, all agents or none.
3. **Lint before you deploy.** `rakerun lint` runs the same rules as the gateway, in a second, with the file, the line and the rule id.
4. **Secrets are references.** Write `${secret:NAME}`, declare it under `secrets:`, set the value with `rakerun secret set NAME`. A push that holds a value fails.
5. **Templates first.** `rakerun template list` has 85 agents. Import one and edit it before you write from scratch.
6. **Stream, do not poll.** `rakerun run start <agent> --stream` prints the events as they arrive and ends on the terminal event.
