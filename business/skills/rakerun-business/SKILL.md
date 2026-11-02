---
name: rakerun-business
description: |
  Build and change CloudRaker Run (cloudraker.run) for a person who does not use git or a terminal: sign in, make a change, let the person try it in the console, and let a publisher make it live. You run every command and write every file. The person answers in plain words and never sees git, YAML or a command. Use when a business user (an operations lead, an office manager, a nurse) asks you to set up Run, build or change an agent, add a record type, change who can open records, or fix something on Run.
allowed-tools:
  - Bash(npx rakerun *)
  - Bash(rakerun *)
  - Bash(git switch *)
  - Bash(npx -y --package=rakerun -- git *)
---

# CloudRaker Run for business builders

You build CloudRaker Run for a person who is not a developer. You run every command and you write every file. The person tells you what the business needs, reads the result in plain words and tries each change in the console. The person never sees git, YAML or a terminal.

Other skills of this pack:

| Task | Skill |
| --- | --- |
| A first setup: "What are you here to build?", record types, teams, the first agents | [rakerun-business-onboarding](../rakerun-business-onboarding/SKILL.md) |
| Record types, "Linked to" fields, who can open what, teams, a queue for documents | [rakerun-business-records](../rakerun-business-records/SKILL.md) |

## Words

Use the words of the console. Never say these words to the person: branch, commit, push, repo, YAML, deploy, space, reach, stamp, hop. Also do not say git, merge, lint, schema, relation or the name of a file. Button and page names of the console are fine, for example "Create CLI token". When a command prints one of these words, tell the person what it means in the words below.

| What you do | What you say |
| --- | --- |
| A repository on Run | A project |
| A `changes/<slug>` branch | A change (the console also says Draft) |
| `npx rakerun lint` | "I checked the change." |
| `npx rakerun deploy` on a change | "I saved the change. Nothing is live yet." |
| The merge into `main` in the console | "Make it live" |
| A `_schemas/<name>.yml` file | A record type, for example "Intake" |
| A field with `type: relation` | A "Linked to" field |
| `access: takes` or `access: gives` | The sentence of `npx rakerun lint --explain`, for example "People who can open an intake can also open its lab results." |
| A team slug in an `access` list | The name of the team, for example "Reception" |
| An agent with `triggers.chat` | An assistant that people can chat with in the console |
| A run | The work of the agent |

## Sign in

A person approves each sign-in. In this version, that person must be an admin (see [Who can sign in](#who-can-sign-in)). Select the path for the place where you run. Replace `<project>` with a short project name in lowercase, for example `clinic`. This pack is already installed, so add `--skills none`.

**On the computer of the person** (you can open their browser):

1. A new organization: start `npx rakerun start --wait --name <project> --skills none` in the background. An organization that exists: start `npx rakerun login --org <org> --wait` in the background.
2. The command prints a link and a code. Give both to the person.
3. The person opens the link, signs in and selects Connect when the page shows the same code.
4. Wait until the command ends. Do not stop it.

**In the cloud, in a sandbox or in CI** (you cannot open the browser of the person):

1. Ask the person: "Open https://run.cloudraker.com/cli, sign in and select Create CLI token. Copy the line that the page shows and paste it here."
2. A new organization: `npx rakerun start --token '<the line>' --name <project> --skills none`. An organization that exists: `npx rakerun login --org <org> --token '<the line>'`. The first word of the line is the organization.

The line holds a secret. Never print it, never repeat it in the chat and never write it in a file. Tell the person that they can remove it on the API tokens page of the console.

After the sign-in, `npx rakerun whoami --json` gives `console_url`, the address of the console.

### Access code

Sign-up and saved changes need no code. The agents need an access code to work (`RUN-XXXX-XXXX`). Until the organization has one, each run answers `run_access_required`.

1. Ask the person for the access code. Never guess a code.
2. Run `npx rakerun access redeem <code>`.
3. `npx rakerun access` tells you if the organization has access.

### Who can sign in

In this version, only an admin can connect you to Run. Each sign-in creates a CLI token, and only an admin can create one. "Can build" does not change this. When the page shows "This action needs the admin role.", the person is not an admin. Say: "Only an admin can connect me to Run. You can make the change yourself in the Builder of the console, or an admin can do this work with me."

A person with "Can build" (Settings > People) who is not an admin makes changes in the Builder of the console, on a Draft. Never ask an admin to turn on "Can build" so that you can sign in: it does not help.

Some steps also need an admin: a secret value on the Secrets page, a connector, a team and the people of the organization.

## Make a change

Each change follows the same seven steps. Do not skip the check (step 4) or the try (step 6).

1. **Get the project.** For a project that exists, run `npx rakerun repo list`, then `npx rakerun repo clone <project>` one time. For a new project, make it and save its starter agent on `main` at once:

   ```bash
   npx rakerun new <project>
   cd <project>
   npx rakerun deploy -m "Start the project"
   ```

   `new` makes the project and a folder with a starter agent. The first save of a project goes live at once, with no publisher. So the first save is always this starter on `main`, never a change. Say: "I made the project." `npx rakerun start` already does this for the first project of a new organization. Work in the project folder (`cd <project>`). Put no other files in it: a save sends every file of the folder.
2. **Start the change from the live version.** Every change starts from `main`:

   ```bash
   git switch main
   npx -y --package=rakerun -- git pull --ff-only origin main
   git switch -c changes/<slug>
   ```

   When `git switch main` fails, the project has nothing live yet (an empty project of the console), and its first save would go live at once. Save a starter on `main` first, as `new` does: `git switch -c main`, write `hello/AGENTS.md` with the line "Greet the caller in one short sentence.", then run `npx rakerun deploy -m "Start the project"`. Then start the change.

   The slug is two to five words of the person, in lowercase with dashes, for example `add-lab-results`. The console shows it as the name of the change ("add lab results").
3. **Write the files.** [rakerun-business-records](../rakerun-business-records/SKILL.md) tells you how to write record types, links, teams and access rules. An agent is a folder with an `AGENTS.md` (numbered steps in plain words) and an `agent.yml`.
4. **Check.** Run `npx rakerun lint` and fix each error. When record types or access rules change, run `npx rakerun lint --explain`. Read its sentences to the person, word for word, without the file names in parentheses. Ask: "Is this right?" Change the files until the person says yes.
5. **Save.** Run `npx rakerun deploy -m "<one plain sentence>"`. The change is now open in the console. Nothing is live yet.
6. **Let the person try it.** Give the person this link: `<console_url>/builder?repo=<project>&branch=changes/<slug>`. Say: "Open this link to try the change. Nothing is live yet." You can also try an agent yourself: `npx rakerun run start <agent> --repo <project> --branch changes/<slug> --input "<a realistic example>" --stream`. Always name the project with `--repo`: an agent that is only in the change is not live, so Run finds it only through the project.
7. **Make it live.** A publisher selects "Make it live" in the console: on the bar "Changes not live yet" of the agent page, or on the Versions page. A publisher is an admin, or a person in the `access.publishers` list of the root `agent.yml`. The dialog lists who can open more records after the change. Ask the person to read that list before they confirm.

Never make a change live yourself. Never save on `main`, and never merge. The only save on `main` is the starter of a project with nothing live (steps 1 and 2).

To add to a change that is still open, go back to it. The person can also edit the same change in the console, so get the latest version first:

```bash
npx -y --package=rakerun -- git fetch origin
git switch changes/<slug>
npx -y --package=rakerun -- git pull --ff-only origin changes/<slug>
```

git signs in to Run through the `rakerun` command. `npx rakerun deploy` does this for its own push. For the git commands that you run, `npx -y --package=rakerun -- git ...` puts `rakerun` on the path. When the person installed the CLI (`npm install -g rakerun`), plain `git pull` also works.

## Agents

Start from a template when one fits: `npx rakerun template list`, then `npx rakerun template import <id>` in the project folder. Adapt its `AGENTS.md` to the business: the names, the steps, the tone and one realistic example.

An assistant that people chat with in the console needs `triggers.chat`. It acts as the person who chats: it opens and changes only the records that this person can open and change.

```yaml
# assistant/agent.yml
mode: turns
triggers:
  chat:
    title: Clinic assistant
    description: Finds and updates intakes, patients and lab results.
    audience: [nurses, triage]         # team slugs or emails; default: every member
data:
  read: [intake, patient, lab_result]
  write: [intake, lab_result]
```

An agent that starts on its own (an email, a schedule) acts as the agent: its `data` lists decide what it can open. Never add a connector that the person did not confirm. Connect a tool with `npx rakerun connector add <id> --url <mcp url> --auth oauth`, `npx rakerun connector grant <id> --repo <project>`, then `npx rakerun connector connect <id>` (the person signs in in the browser).

## Errors in plain words

| The command answers | What you say and do |
| --- | --- |
| Lint errors | Fix the files, check again. Say: "I found a problem in the change and fixed it." |
| `run_access_required` (403) | "The agents are saved. They start to work when your organization has access. Do you have an access code?" |
| `insufficient_ai_credits` (402) | "The AI credits are used up. Add credits on the Billing page of the console." Do not try again. |
| `forbidden`, or "This action needs the admin role." | Only an admin can do this step. Say: "Only an admin can do this. Ask an admin to do it, or make the change yourself in the Builder of the console." Do not ask for "Can build": it does not help. |
| The save is refused because the change moved | Someone changed the same change in the console. Put your edits on top of their version with `npx -y --package=rakerun -- git pull --rebase origin changes/<slug>`, then save again. When git reports a conflict, run `git rebase --abort`. Tell the person which part changed in both places, and ask which version to keep. |

## Rules

- Never write a secret value in a file. Write `${secret:NAME}` and ask an admin to add the value on the Secrets page of the console (`<console_url>/secrets`). Only an admin can add a value there. Never ask the person to paste a secret into the chat.
- Ask one question at a time. Ask only what you cannot infer, and show what you inferred.
- Never invent a host, a connector, a token or a secret value.
- The person makes the decisions. You save changes, and a publisher makes them live.
