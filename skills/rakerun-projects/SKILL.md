---
name: rakerun-projects
description: |
  Create and ship a cloudraker.run project with the rakerun CLI: new project, template catalog and import, local lint, deploy (commit and push with the deploy log), environments (one project each for development, staging and production, env add, env list, push and the GitHub Action), repos, base paths, branch previews and rollbacks. Use when the user wants to start a project, add a template agent, deploy changes, or push one tree to several environments.
allowed-tools:
  - Bash(rakerun *)
  - Bash(npx rakerun *)
  - Bash(git *)
---

# Projects: new, template, lint, deploy, environments

## New project

```bash
rakerun new my-agents                          # starter agent in hello/
rakerun new my-agents --template linear-triage # one template agent instead
rakerun new my-agents ./agents-dir             # another folder name
```

`new` creates the repo on the platform (`POST /_console/repos`), the folder, the files, `git init -b main`, the `origin` remote at `https://{org}.cloudraker.run/_git/{name}` and registers `rakerun git-credential` as the git credential helper of the host. Nothing is deployed until `rakerun deploy`. A repo name matches `[A-Za-z0-9][A-Za-z0-9._-]{0,99}`.

An existing repo: `rakerun repo clone <name> [dir]`. List: `rakerun repo list` (name, base path, agent count, URL).

## Templates

```bash
rakerun template list                      # 85 templates: id, name, category, mode, triggers, connectors (* = required)
rakerun template list --category finance
rakerun template show invoice-extractor    # files, what it does, connectors it needs
rakerun template import invoice-extractor  # writes invoice-extractor/ into the current project
rakerun template import invoice-extractor --folder invoices --dir ./my-agents
```

A template is one agent folder. Import writes it under `<folder>/` (the template id by default) and refuses an existing folder. A template with a phone, email or Slack channel also carries a front door: import writes it to `doors/<id>.yml` at the repo root and refuses an existing door file. The door names the agent by the template id: with `--folder`, change `entry` in the door file. Then bind the channel (see rakerun-doors). After an import: read the `AGENTS.md`, set the secrets and connectors the template names (`rakerun template show` lists them), then `rakerun deploy`.

Prefer a template over a blank agent: edit the prompt and the `agent.yml` of a working agent.

## Lint

```bash
rakerun lint                 # the current folder
rakerun lint ./my-agents
rakerun lint --repo my-agents   # also checks _schemas changes against the live tree (schema_breaking)
```

Output: one line per issue, `error|warning  path:line  rule  message`, then a summary. Exit 1 on an error: the push would be refused. The rules are the gateway rules (the lint code is bundled in the CLI): `agent_yml_schema`, `root_leaf_key`, `secret_undeclared`, `secret_value`, `nested_agent`, `code_tool_collision`, `knowledge_agent_folder`, the `door_*` rules of `doors/*.yml` and the rest. The summary counts agents and doors: `2 agent(s), 1 door(s), 0 error(s), 0 warning(s)`. Fix the named file and line.

## Deploy

```bash
rakerun deploy                       # lint, git add -A, commit "Deploy with rakerun", push
rakerun deploy -m "refund reviewer"  # your commit message
```

What it prints comes from the gateway: lint warnings, then `deployed N agent(s) at <sha>`, or `preview of N agent(s) at https://<branch>--<org>.cloudraker.run` on a branch other than `main`. A refused push prints the rule, the file and the line. Nothing to commit and nothing new on the remote prints `main is up to date.`

The push is the deploy: every agent of the repo or none. A run pins the deployment SHA it started on. Rollback: `rakerun repo redeploy <name> --sha <40 hex>` deploys an older commit and leaves the branch head alone (same as **Restore version** in the console); or revert in git and deploy.

Previews: `git checkout -b try-it`, edit, `rakerun deploy`. The branch gets its own host, `https://try-it--acme.cloudraker.run`. A token needs the `preview` scope to call it. Previews never run schedules, email or `_schemas` sync. Deleting the branch drops the preview.

`deploy` needs an `origin` at `https://{org}.cloudraker.run/_git/{name}` (from `rakerun new` or `rakerun repo clone`). When `origin` is GitHub or another git host, use environments and `rakerun push`.

## Environments

A project is one environment. It has its own records, knowledge index, secret values, connector grants, live commit, previews and base path. For development, staging and production, create three projects and push the same tree to each one. Run copies no records, secrets, grants or commits from one project to another. Run refuses an invalid push, but it is not a backup. The user's own origin (GitHub or another git host) stays the source of truth.

Tables are the exception. The definition of each `_schemas` name belongs to the org, and every project of the org shares it: the fields, `required`, `unique`, options, `label`, `label_field`, and `to` and `many` of a relation. Only `access` and `filing` come from the files of each project. So a live push of a schema change to the dev project changes the table for staging and production at once:

- Removing `unique` makes their `data.upsert` on that field fail. A new `required` makes their `data.create` fail on a record without that field.
- A change of `to` or `many` fails the push with `type_change_blocked` while any project of the org has records in the table.
- Lint compares a push only with the live tree of the same project, so it does not catch this.

Keep `_schemas/` the same in every project of the org, and push a schema change to all environments together (a new optional field or option is the safest change). For a strict boundary, put production in another org. Tell the user before you push a `_schemas/` change to one environment only.

```bash
rakerun repo create acme-agents-dev           # once per environment (admin scope)
rakerun repo create acme-agents-staging
rakerun repo create acme-agents
rakerun env add dev acme-agents-dev           # git remote "dev" with the rakerun credential helper
rakerun env add staging acme-agents-staging
rakerun env add prod acme-agents
git commit -am "triage: ask for a repro"      # push sends the last commit only
rakerun push dev                              # lint HEAD, push it, print the deploy log
rakerun push prod --branch main               # deploy live from another local branch
rakerun env list                              # each environment with its live SHA (--json: full SHA)
```

| | `rakerun deploy` | `rakerun push <env>` |
| --- | --- | --- |
| Remote | `origin`, a project of the active host | any remote of `env add`, on any saved host |
| Commit | `git add -A` and a commit | none: commit first (it warns about changes that are not committed) |
| Lint | the working tree | the last commit (`HEAD`) |
| Branch | the current branch | the current branch, or `--branch <b>` |
| Force | never | only with `--force` |

- The push goes to the project branch with the name of the current branch. `main` deploys live. Another branch gets a preview of that project only.
- Each project has its own base path (the project name by default), so each environment has its own agent URLs: `https://acme.cloudraker.run/acme-agents-dev/triage`. Name the project in run commands: `rakerun run start triage --repo acme-agents-dev`.
- `env add` refuses a project that does not exist and a remote name in use (`git remote set-url` changes one). It needs `rakerun` on `PATH`: git calls `rakerun git-credential` for the token. With `npx rakerun` only, ask the user to run `npm install -g rakerun`.
- The remotes live in the git config of the clone, not in the tree. Run `env add` once in each clone.
- Another org (for example a production org): `rakerun login --org acme-prod --profile prod`, `rakerun profile use <previous>` (login makes the new profile current), then `rakerun env add prod acme-agents --profile prod`. `push` and `env list` find the token of each host in the saved profiles.
- Secrets per project: `rakerun secret set NAME --repo <project>`. The project value wins over the org value. Connectors belong to the org: `rakerun connector grant <name> --repo <project>`, and an org grant covers every project. For another connector account per environment, use a header-from-a-secret connector with a project secret, or another org. See [rakerun-integrations](../rakerun-integrations/SKILL.md).
- A preview of a project uses the records, secrets and grants of that project. Test on the dev project, never on a branch of production.
- Push to a production environment only when the user asks.
- A refused push with "has commits that your branch does not have" means that the project got commits from elsewhere, usually a Draft made live in the console. Run `git pull <env> <branch>`, then push again. Never add `--force` unless the user asks: it removes those commits from the project.
- Token scopes: `repo create` and `connector grant` need `admin`, `env add` and `env list` need `read`, `push` needs `git`, `secret set` needs `secrets`. The token of `rakerun login` has all of them. A token opens every project of its org: for a hard boundary, put production in another org.
- A shallow clone can fail to push: `git fetch --unshallow`.

CI: the GitHub Action `CloudRaker/run-deploy` pushes the checked-out commit. Inputs: `project`, `token` (a `crr_` token with the `git` scope, in a GitHub secret), `org` or `base-url`, `branch` (default `main`), `force` (default `false`). Use one job, one project and one token secret for each environment, with `fetch-depth: 0` on `actions/checkout`. Reference: https://docs.cloudraker.com/run/guides/environments.

## Base paths

Agent URLs are `https://{org}.cloudraker.run/{base}/{agent}`. The base is the repo name by default. `rakerun repo create <name> --base-path <path>` sets another one at creation; the console repo settings change it later. One repo of the org can be the root (`base_path: ''`), so its agents answer at `/{agent}`.

## See also

- [rakerun-agents](../rakerun-agents/SKILL.md) for what goes into the files you deploy
