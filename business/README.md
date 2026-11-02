# CloudRaker Run skills for business builders

Agent skills for [cloudraker.run](https://run.cloudraker.com), for a person who does not use git or a terminal. In the [Agent Skills](https://agentskills.io) format, as a plugin for Claude Code and OpenAI Codex.

Your AI agent does every step with the `rakerun` CLI. You answer questions in plain words, try each change in the console, and a publisher selects "Make it live". The agent never asks you to use git, YAML or a command.

For coding agents that work with git and YAML, use the developer pack in the [parent folder](../README.md).

In this version, only an admin can connect an AI agent to Run: the sign-in creates a CLI token, and only an admin can create one. A person with "Can build" who is not an admin makes changes in the Builder of the console.

## Install

```bash
npx rakerun skills add business            # in this folder
npx rakerun skills add business --global   # for all your projects
```

Or with the `skills` CLI:

```bash
npx skills add https://github.com/CloudRaker/rakerun-skills/tree/main/business --skill '*'
```

Or as a Claude Code plugin:

```
/plugin marketplace add CloudRaker/rakerun-skills
/plugin install rakerun-business@rakerun-skills
```

## Catalog

| Skill | What it covers |
| --- | --- |
| [`rakerun-business`](./skills/rakerun-business) | Sign-in, the words to use, the steps of a change (check, save, try, make it live), assistants for chat, errors in plain words |
| [`rakerun-business-records`](./skills/rakerun-business-records) | Record types, "Linked to" fields and the direction in which they open records, who can open what, teams, roles, sharing, a queue for incoming documents |
| [`rakerun-business-onboarding`](./skills/rakerun-business-onboarding) | A first setup: the profile, "What are you here to build?", the structure, the access sentences, the first try |

## The rules these skills teach

1. **The agent does the steps.** The person never sees git, YAML or a terminal. The agent never says branch, commit, push, repo, YAML, deploy, space, reach, stamp or hop.
2. **Every change is a Draft first.** The agent saves the change, and the person tries it in the console. Nothing is live until a publisher selects "Make it live". The only exception is the starter agent of a new project: the first save of a project goes live at once, so the agent saves only the starter there.
3. **Read the access back.** Before the agent saves a change to record types or access, it reads the sentences of `rakerun lint --explain` to the person, word for word, and waits for a yes.
4. **One link, never further.** A link opens records in the direction that its field declares, and access through a link stops there.
5. **Secrets stay out of the chat.** An admin adds secret values in the console. The agent never prints the sign-in line.
