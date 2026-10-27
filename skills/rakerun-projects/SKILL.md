---
name: rakerun-projects
description: |
  Create and ship a cloudraker.run project with the rakerun CLI: new project, template catalog and import, local lint, deploy (commit and push with the deploy log), repos, base paths, branch previews and rollbacks. Use when the user wants to start a project, add a template agent, or deploy changes.
allowed-tools:
  - Bash(rakerun *)
  - Bash(npx rakerun *)
  - Bash(git *)
---

# Projects: new, template, lint, deploy

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

## Base paths

Agent URLs are `https://{org}.cloudraker.run/{base}/{agent}`. The base is the repo name by default. `rakerun repo create <name> --base-path <path>` sets another one at creation; the console repo settings change it later. One repo of the org can be the root (`base_path: ''`), so its agents answer at `/{agent}`.

## See also

- [rakerun-agents](../rakerun-agents/SKILL.md) for what goes into the files you deploy
