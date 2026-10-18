---
name: rakerun
description: |
  Any task on cloudraker.run through the rakerun CLI: create a project of agents from templates, write or change an agent (AGENTS.md and agent.yml), deploy it as an HTTPS endpoint, run it and stream the answer, manage secrets, MCP connectors, tokens, webhooks, Structured Memory records and knowledge folders. Use whenever the user wants to build, deploy, run or operate agents on CloudRaker Run.
allowed-tools:
  - Bash(rakerun *)
  - Bash(npx rakerun *)
---

# rakerun

`rakerun` is the CLI of cloudraker.run: headless agents from a git repo. Each folder with an `AGENTS.md` is an agent. A push deploys every agent of the repo at `POST https://{org}.cloudraker.run/{repo}/{agent}`. The console is at https://run.cloudraker.com.

## Install and log in

```bash
npm install -g rakerun          # or prefix every command with: npx rakerun
rakerun --version
rakerun login --org acme        # existing org: browser sign-in, approve on the page, done
rakerun start                   # no org yet: sign-up, org, token, first project, deploy
rakerun whoami
```

`login` and `start` open the browser and wait on a loopback port. The console page shows the org, the port and a code; the terminal prints the same code. The user approves on the page. When you run on the user's computer, run the command yourself: the tab opens on their screen, you read the code from your output and tell them "click Connect when the page shows <code>". `rakerun start --name <project> --templates <id,id>` answers the prompts, so no terminal input is needed. When you run elsewhere (a cloud agent), add `--paste`: the user opens the link and pastes back the line the page shows. On a remote shell, `--paste` shows one line on the page to paste. In CI, `rakerun login --base-url https://acme.cloudraker.run --token -` reads a `crr_` token from stdin; create one with `rakerun token create ci --scopes run,read` or in the console **Tokens** screen.

Profiles: one per org host in `~/.config/rakerun/config.json`. `rakerun profile list`, `rakerun profile use <name>`, `--profile <name>` on any command. `--base-url https://acme.cloudraker.run` targets another host.

Verify auth: `rakerun repo list` (any table means the token works). A `unauthorized` error means: run `rakerun login` again.

## Early access and the waitlist

CloudRaker Run is in early access. An organization without access is waitlisted. A waitlisted org can sign up, do the onboarding, create projects, import templates and deploy agents. It cannot run them: every run, turn and Builder session answers `403 run_access_required`. The CLI then prints the waitlist message, not the raw error.

```bash
rakerun access                         # granted or waitlisted, for the org of the profile
rakerun access redeem RUN-ABCD-EFGH    # redeem an access code; runs open at once
```

- `rakerun start` checks the access after the first deploy. On a waitlisted org it prints the waitlist message and, on a terminal, asks once for an access code (Enter skips).
- Do not retry a run that answers `run_access_required`. Tell the user that the agents are saved and deployed, and that runs start when the org has access.
- Ask the user for an access code. Never guess one. A code looks like `RUN-XXXX-XXXX`; the CLI trims and uppercases it.
- Redeem errors: the code is not valid, has expired, or has no uses left; too many attempts (wait one minute); or access could not be turned on (the code stays unused, try again later).
- For faster access, the user writes to support@cloudraker.com or opens the chat in the console.

## The loop

```bash
rakerun new my-agents --template linear-triage   # repo on the platform, folder here, git, remote
cd my-agents
# edit linear-triage/AGENTS.md and linear-triage/agent.yml
rakerun lint                                     # the push lint, locally: rule, file, line
rakerun deploy -m "triage: ask for a repro"      # commit everything, push, deploy log
rakerun run start linear-triage --input "Triage EGR-123" --stream
```

`deploy` prints what the gateway says: lint warnings, `deployed N agent(s) at <sha>`, or a preview host on a branch other than `main`. One lint error rejects the whole push. Fix the file it names and deploy again.

## Where each task goes

| Task | Skill |
| --- | --- |
| New project, templates, lint, deploy, repo, previews, rollback | [rakerun-projects](../rakerun-projects/SKILL.md) |
| Write or change an agent: prompt, `agent.yml`, triggers, tools | [rakerun-agents](../rakerun-agents/SKILL.md) |
| Start, stream, turns, history, trace, cancel | [rakerun-runs](../rakerun-runs/SKILL.md) |
| Secrets, `network:`, MCP connectors, tokens, webhooks | [rakerun-integrations](../rakerun-integrations/SKILL.md) |
| Structured Memory records and schemas, `knowledge/` folders | [rakerun-knowledge](../rakerun-knowledge/SKILL.md) |
| Set up a new organization, the onboarding checklist | [rakerun-onboarding](../rakerun-onboarding/SKILL.md) |

## Rules that hold everywhere

- **Never write a secret value into the repo.** Write `${secret:NAME}` and `rakerun secret set NAME` (value prompted or on stdin). A push with a value fails with `secret_value`.
- **Never edit a token.** `rakerun token create` shows the value once. Store it where the user asks, never in the repo.
- **`--json` on every command** gives the raw API answer. Use it when you need ids.
- **`run_access_required` (403)** means the org is on the waitlist. See [Early access and the waitlist](#early-access-and-the-waitlist).
- **Exit codes:** 0 ok, 1 API or runtime error, 2 usage. The message names the API error code (`not_found`, `forbidden`, `invalid_body` with details).
- The CLI never edits agent files. You do, in the repo, then `rakerun deploy`.
