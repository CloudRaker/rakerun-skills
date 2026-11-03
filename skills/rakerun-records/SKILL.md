---
name: rakerun-records
description: |
  Decide who can open, change and delete the records of a cloudraker.run project: the access block of _schemas files (table rules, row rules, shares), relation fields that link records and pass access in one direction (takes, gives, access.links, one link and no further), people teams, the three roles (admin, builder, member), filing queues with data.file, and rakerun lint --explain. Use when the user wants records that link to each other, wants to control who sees or changes records, or receives documents that an agent must file into cases.
allowed-tools:
  - Bash(rakerun *)
  - Bash(npx rakerun *)
---

# Records: who can open what

A record is the unit of access in Run. The `access` block of each `_schemas/<name>.yml` file says who can open, change and delete the records of that record type. A relation field links two records. A link can pass access, in the direction that the field declares.

The same rules apply on every path: the Records screen, record pages, apps, search, exports and the `data.*` tools of an agent. A rule change goes live with a push to the default branch, or with **Make it live** in the console. A Draft or another branch never changes live access.

Reference: https://docs.cloudraker.com/run/guides/records-access. Field types, `data:` grants and the `rakerun data` commands are in [rakerun-knowledge](../rakerun-knowledge/SKILL.md).

## The order of work

1. Ask the questions of [the access block](#the-access-block) and of [linked records](#linked-records), one at a time.
2. Write each `_schemas/<name>.yml`: the fields, the relation fields and the `access` block.
3. Check the teams that the rules name: `rakerun teams ls`. Add a missing team with `rakerun teams add` when the user confirms it.
4. Give each agent its `data:` grants. Add `triggers.chat` to the agents that people call (see [Roles](#roles)).
5. Run `rakerun lint --explain`. Tell the user the sentences of each record type, word for word. Change the files until the user agrees.
6. Deploy. Test the access with records of a development project (see [Test the access](#test-the-access)).

## The access block

```yaml
# _schemas/claim.yml
name: claim
label: Claim
label_field: claim_number
fields:
  claim_number: { type: text, required: true, unique: true }
  handler: { type: email } # the person who works the claim
  region_team: { type: select, options: [claims-east, claims-west] }
access:
  read: [claims-handlers, claims-leads] # who can open claims
  write: [claims-handlers, claims-leads] # who can change and add claims. Write includes read.
  delete: [claims-leads] # default [admins]. Each entry must also be in write.
  rows: # optional: which claims each person sees
    owner: handler # an email field, or $creator
    team: region_team # a select, multi_select or text field of team slugs
    all: [claims-leads] # see every claim. Default [admins].
  shares: read # none | read | write
  links: write # none | read | write
```

Ask the user, in plain words:

1. "Who works on claims?" The answer gives `read` and `write`.
2. "Who can delete them?" The answer gives `delete` (default: admins).
3. "Does each person see all of them, or only their own?" The answer gives `rows`.
4. "Who sees all of them?" The answer gives `rows.all` (default: admins).
5. "Can people share one claim with a colleague?" The answer gives `shares` (`none`, `read` or `write`, default `read`).

An entry is `members`, `admins` or a team slug. An email is never an entry (`schema_access_email`): to add one person, share one record. With no `access` block, every member can open and change the records, and admins can delete them.

How Run decides:

- **Open.** A table rule (`read` or `write`) names the person. When the table has `rows`, a row rule must also match: the person is in `rows.all`, an owner field names them, or a team field names one of their teams. A share of the record also opens it.
- **Change.** The same check, with `write`. A change can move a record out of the rows of the person. This is a hand-off.
- **Delete.** The person needs `delete` and must be able to change the record. A share or a link never gives delete.
- **Create.** The person needs `write`, and the new record must fall in their rows (or the person is in `rows.all`). A link gives one more way, but such a record cannot give the person direct access: [create through a link](#link-writes).
- `$creator` is the person who added the record. A background run or an API token adds records with no creator, so such a record matches `$creator` for nobody.

## Teams

A team is a WorkOS group of the organization (people teams, not Microsoft Teams). The rules name a team by its slug. A rename never changes the slug.

```bash
rakerun teams ls                                     # slug, name, source, people, deleted teams
rakerun teams add "Claims West" --slug claims-west   # token scope admin; without --slug, Run makes one
```

- A push to the default branch fails with `schema_access_team_unknown` when a rule names a team that does not exist. A push to another branch only warns. Add the teams before the live push.
- `teams add` adds no people. An admin adds people with **Add a person** in **Settings > Teams** of the console, or picks a directory group there.
- A person who joins or leaves a team gets the change within 60 seconds. No push is necessary.
- `teams add` answers "Only an admin can add a team." for a token without the `admin` scope. Then ask an admin to add the team in the console.

## Linked records

A relation field holds the id of one record of another record type of the project, or a list of ids.

| Key | Values | Meaning |
| --- | --- | --- |
| `fields.<f>.type` | `relation` | A link to records of `to` |
| `fields.<f>.to` | a `_schemas` name of the project | Required |
| `fields.<f>.many` | `false` (default) or `true` | `true`: a list of at most 50 ids |
| `fields.<f>.access` | `none` (default), `takes` or `gives` | Which way access passes through the link |
| `fields.<f>.filing.auto_at` | 0.5 to 1, default 0.9 | `takes` fields only. See [filing queues](#filing-queues) |
| `access.links` | `none`, `read` or `write` (default) | The most that a link gives on this record type |

- `takes`: people who can open the linked record can also open this record. Use it on a record that reports to a parent: a lab result takes from its intake.
- `gives`: people who can open this record can also open the linked record. Use it on a case that opens a shared record: an intake gives its patient.
- `none`: the link passes no access. To add it, a person needs only to change the record that holds the field.
- A relation field cannot be `unique` or the `label_field`. A change of `to` or `many` is breaking (`schema_breaking`): add a new field instead.
- A record cannot link to itself. A link to a missing or deleted record passes nothing, and Run removes the id of a deleted record. History keeps the old value.

```yaml
# _schemas/intake.yml
name: intake
label: Intake
label_field: number
fields:
  number: { type: text, required: true, unique: true }
  nurse: { type: email }
  status: { type: select, options: [new, review, accepted, refused] }
  patient: { type: relation, to: patient, access: gives } # the intake opens its patient
access:
  read: [nurses, triage]
  write: [nurses, triage]
  delete: [triage]
  rows:
    owner: nurse
    all: [triage]
```

```yaml
# _schemas/lab_result.yml
name: lab_result
label: Lab result
label_field: title
fields:
  title: { type: text, required: true }
  intake: { type: relation, to: intake, access: takes } # the intake opens this result
  patient: { type: relation, to: patient } # no access through this link
access:
  read: [clinicians]
  write: [clinicians]
  delete: [clinicians]
```

```yaml
# _schemas/appointment.yml
name: appointment
label: Appointment
fields:
  starts_at: { type: datetime, required: true }
  patient: { type: relation, to: patient, access: takes } # people who open the patient open it
  intake: { type: relation, to: intake, access: takes } # people who open the intake open it
access:
  read: [clinicians]
  write: [clinicians]
  delete: [clinicians]
  links: read # through a link, people can only read appointments
```

Ask the user, in plain words:

1. "Should the people of an intake also see its lab results?" A yes gives `access: takes` on the `intake` field of the lab result (the child record).
2. "Should the people of an intake also see its patient?" A yes gives `access: gives` on the `patient` field of the intake.
3. "Through a link, can people change appointments, or only read them?" The answer gives `access.links` of appointments.

### One link, never further

Only direct access passes through a link. Direct access comes from table rules, row rules and shares. Access that a person gets through a link stops there.

A nurse owns intake INT-1 (`nurse` holds her email), and INT-1 links to patient P-1:

| Record | Can she open it? | Why |
| --- | --- | --- |
| Intake INT-1 | yes, and change it | Direct: the owner rule |
| Patient P-1 | yes, and change it | Through INT-1 (`gives`) |
| Lab result L-1 with `intake: INT-1` | yes, and change it | Through INT-1 (`takes`) |
| Appointment A-1 with `intake: INT-1` | yes, read only | Through INT-1 (`takes`), capped by `links: read` |
| Appointment A-2 with `patient: P-1` and `intake: INT-7` | no | Her access to P-1 comes through a link |
| Intake INT-7 of the same patient | no | INT-7 gives P-1. That does not open INT-7 |

So a person on one case never sees the other cases of a shared record. Tell the user: "Access through a link stops there. It does not continue to other links."

### The level through a link

The level through a link is the level of the person on the source record. `access.links` of the record type that gets the access caps it: `write` keeps the level, `read` gives read only, `none` gives nothing. Delete never passes through a link.

A write through a link cannot give the person direct access to the record. For example, a person who changes an intake only through a link cannot set its `nurse` field to their own email. Run answers `403 record_forbidden`: "You cannot give yourself access to this record." A create through a link has the same limit (see [link writes](#link-writes)).

### Link writes

- To add a `takes` or `gives` link, the person must be able to change both records: the record that holds the field and the linked record. Direct access and access through a link both count.
- To remove a link, the person needs only to change the record that holds the field.
- **Create through a link.** A person who can change a record directly (table rule, row rule or share) can add a record that takes from it. The new record type must have `access.links: write`. The person needs no `write` rule on that record type.
- A record that a person adds through a link cannot give them direct access. This limit applies only when a `read` rule names the person, no `write` rule names them, and a row rule decides which records they open. Then:
  - An owner field of the new record cannot name them, and a team field cannot name one of their teams. Run answers `403 record_forbidden`: "You cannot give yourself access to a record that you add through a link." The "Add" sheet of the record page fills in only the link, not the owner or team fields.
  - A `$creator` row rule always names them, because Run writes the person as the creator. So they cannot add a record of that type through a link, and the record page shows no "Add" button. Do not use `$creator` on a record type when people who can only read it must add its records through a link.
- A refused link names the linked record only for a person who can open it: "You cannot link this lab result to Intake INT-7. You cannot change that intake."
- Run checks the records of a link write again after the write. When one of them changed in the meantime, Run removes the link that it added at once and decides again, at most 3 times. Then it answers `409 record_changed`: "This record changed while you saved. Try again." Read the records again before you try again.

### Shares and links

A share is direct access. So a share of a record also opens the records that this record passes access to, one link away. The share sheet says what else opens: "Also opens 3 linked records."

A person who can change a record only through a link can also share it, with two limits:

- The share cannot name the sharer, or a team of the sharer.
- No share raises the access of the person who made it. Run ignores the share for its sharer, also when the sharer joins the team of the share later.

Agents and API tokens never create shares.

### What agents see

- A turn that a person sends acts for that person. The agent opens a record only when its `data` grant covers the record type and the person can open the record, also through links.
- A background run (a schedule, an inbound webhook, an API token run) acts as the agent. The `data` grant decides alone.
- `data.schemas` shows each relation field with `to` and its access sentence.
- `data.get` and `data.query` give a relation value as `{ id, label }` for each linked record that the actor can open, and as `{ hidden: N }` for the others. An update that sends back `{ hidden: N }` keeps the links that the agent cannot see.
- `data.query` filters a relation field on ids: `{ "patient": "P-1" }`, or `{ "intake": ["INT-1", "INT-2"] }` for any of a list.
- A run can work on one record and the records that it passes access to (a focused run). See [rakerun-agents](../rakerun-agents/SKILL.md#runs-on-one-record).

## Filing queues

When documents come in (email attachments, uploads), make a queue record type that only the queue team can open. Give it a `takes` field to the case with `filing`, and a select field `status` with the options `waiting`, `filed` and `refused`.

```yaml
# _schemas/inbound_document.yml
name: inbound_document
label: Inbound document
label_field: subject
fields:
  subject: { type: text }
  sender: { type: email }
  file_id: { type: text } # the Run file id of the attachment
  status: { type: select, options: [waiting, filed, refused] }
  intake:
    type: relation
    to: intake
    access: takes
    filing: { auto_at: 0.9 }
access:
  read: [reception]
  write: [reception]
  delete: [reception]
```

- A filing field is a `takes` relation field with `filing`. A record type has at most one filing field, and it cannot be `required`.
- Run offers the tool `data.file` (the model sees `data_file`) to an agent with `data.write` on the queue record type and `data.read` on the record type of the case.
- Run sets `status` only when the record type has a select field `status` with that option.

```yaml
# mail-intake/agent.yml
name: Mail intake
data:
  read: [intake]
  write: [inbound_document]
```

```markdown
# Mail intake

1. Add one `inbound_document` record for each attachment.
2. Find the intake of the document with `data_query`.
3. Call `data_file` with the intake, your confidence from 0 to 1 and a short reason.
4. When no intake matches three documents, call `flag` for the `reception` team.
```

| `data.file` input | Rule |
| --- | --- |
| `schema` | The queue record type |
| `document` | The id of the document record |
| `field` | The filing field |
| `target` | The id of the case record |
| `confidence` | A number from 0 to 1 |
| `reason` | 1 to 500 characters. The queue team sees it |

1. At or above `filing.auto_at` (default 0.9), Run writes the link and sets `status: filed`. The tool answers `{ "filed": true }`. History says "Filed by Mail intake (96%)".
2. Below it, Run opens one Inbox item "Where does this go?" with the suggested case, the confidence and the reason. The document stays `waiting`. The tool answers `{ "suggested": true, "inbox_item": "..." }`. A second call while the item is open replaces the suggestion.
3. When a person decided first, the tool answers `already_decided`.
4. When a person sent the turn, that person must be able to change the document and the case, as for any link.

The decision in the Inbox:

- The item shows in **Needs you** for each person who can change the document. It counts in the badge and ranks with approvals.
- **File here** and **Choose another record** need write on the document and on the case. **Refuse** sets `status: refused` and keeps the note.
- The first decision wins. A later one answers `409 already_decided`, for example "Rita already filed this document." A filing item closes only with a decision: dismiss answers `409 use_filing`.
- **Unfile** on the record page of a document removes the link and sets `status: waiting`. The filing item opens again for the queue team, with no suggested case.
- The console record form shows the filing field read-only. Only the Inbox decision and Unfile set it in the console.

A confident wrong guess files a document into the wrong case, and the people of that case can open it. Tell the user to check **Filed automatically** on the Records screen of the queue: it lists the documents that an agent filed in the last 7 days. Raise `auto_at` for sensitive cases.

An email front door gives the filing agent its documents: see [rakerun-doors](../rakerun-doors/SKILL.md). The `flag` tool puts an item in the Inbox of a person or a team, and the run continues: see [rakerun-agents](../rakerun-agents/SKILL.md#flag-a-person-or-a-team).

## Roles

Run has three roles. An admin gives **Can build** (role `builder`) in **Settings > People**.

| Can | admin | builder | member |
| --- | --- | --- | --- |
| Use Home, Inbox, Conversations, Apps, Outputs and Records | yes | yes | yes |
| Open the build pages: projects, agent settings, the Builder, code, runs, schemas, front doors | yes | yes | no |
| Make changes live | yes | only in `access.publishers` | app changes only, in `access.publishers` |
| Open Settings: Teams, People, Billing | yes | no | no |
| Open records | the rules of this skill | the rules of this skill | the rules of this skill |
| Start an agent from the console | in its audience | in its audience | in its audience |

- **Admin.** Everything, but the record rules apply to admins too. An admin opens the records of a record type directly only when its `read` or `write` names `admins`, `members` or a team of the admin. A share or a link can also open a record. `rows.all` defaults to `[admins]`: on a record type with `rows`, an admin who passes the table rule sees every row. Set `all: []` to keep admins out of the rows.
- **Builder.** A builder reads every schema of every project and changes projects in Drafts. A builder opens records only through the rules, as a member does. In the console, a builder who is not in `access.publishers` works only on Drafts: a change of the live project answers `403 draft_required`.
- **Member.** A member uses only the work pages and the agents that they can call. An agent is callable when it has `triggers.chat` and its `audience` includes the person. Other pages answer 404.
- `access.publishers` in the root `agent.yml` lists `admins` and the emails of the people who make changes live.
- In WorkOS, `builder` is an environment role with no permissions. The role priority is admin, builder, member.
- A `crr_` token with the `admin` scope acts as an admin with no email and no teams. So it passes only the rules that name `admins` or `members`. A `read` or `run` token opens directly only the record types with `members` in `read` and no `rows`.

```yaml
# intake-assistant/agent.yml: an agent that nurses and triage call from the console
name: Intake assistant
mode: turns
triggers:
  http: true # a triggers block without http turns off POST /{base}/{agent}
  chat:
    title: Intake assistant
    description: Answers questions about one intake and adds lab results.
    audience: [nurses, triage] # members (default), admins, or emails and team slugs
data:
  read: [intake, patient, lab_result]
  write: [lab_result]
```

## Lint

| Rule | Level | Check |
| --- | --- | --- |
| `relation_target_unknown` | error | `to` names no `_schemas` record type of the project |
| `relation_access_invalid` | error | `access` is not `none`, `takes` or `gives` |
| `relation_filing_invalid` | error | `filing` on a field that is not `takes`, on a `required` field, with `auto_at` outside 0.5 to 1, or twice in one record type |
| `schema_links_invalid` | error | `access.links` is not `none`, `read` or `write` |
| `relation_gives_open_table` | warning | A `gives` field on a record type that every member can open, to a record type that not every member can open |
| `relation_link_widens` | warning, Draft only | A new link lets more people open a record type. The make-live dialog says, for example, "People who can open intakes can now open patients (412 records)." |
| `relation_links_none` | info | A `takes` or `gives` link into a record type with `access.links: none`. The link passes nothing |
| `schema_access_email` | error | An entry is an email |
| `schema_access_team_unknown` | error on a live push, warning on a branch | A team slug that the organization does not have |
| `schema_delete_not_write` | error | A `delete` entry, also the default `[admins]`, is not in `write`, and `write` does not have `members` |
| `schema_rows_owner_field`, `schema_rows_team_field` | error | `rows.owner` or `rows.team` names a missing field or a field of the wrong type |
| `schema_access_widened` | warning, Draft only | The change lets more people open or change a record type |

A value with more than 50 ids is a write error, not a lint rule: "Patients takes at most 50 records."

## Explain the access: `rakerun lint --explain`

```bash
rakerun lint --explain          # the issues, then who can open each record type
rakerun lint --explain --json   # adds "explain": [{ table, label, path, sentences }]
```

```text
0 agent(s), 0 door(s), 0 error(s), 0 warning(s)

Intake (_schemas/intake.yml)
  Nurses and Triage can open intakes.
  Nurses and Triage can change intakes.
  Triage can delete intakes.
  Each person sees the intakes assigned to them.
  Triage see all intakes.
  People who can open an intake can also open its patient.
  People who can open an intake can also open its lab results.
  Access through a link stops there. It does not continue to other links.
```

- The Builder, the console and the make-live dialog use the same sentences.
- Logged in, the sentences give the team names. Logged out, they give the slugs as words, and the CLI makes no call.
- Before each deploy that changes `_schemas/`, tell the user the sentences of the record types that changed, word for word. Ask whether they are right.

## Test the access

- The rules and the links of the live project decide. A link that only a branch or a Draft declares passes nothing in a preview, and a preview does not sync `_schemas`. So test the access on a development project: see [rakerun-projects](../rakerun-projects/SKILL.md#environments).
- `rakerun data add` acts as an admin with no email and no teams (see [Roles](#roles)). It adds a record only when `write` names `admins` or `members`, and `rows.all` too when the record type has `rows`. Most access rules name teams, so it answers 403, for example "You cannot add lab results." Do not add `admins` to the rules only for a test. Add test records in one of these ways:
  - A person of the team adds them in the console, on the Records screen of the development project. This also tests the rules of that person.
  - Start a run of an agent that has a `data.write` grant on the record type: `rakerun run start <agent> --repo <dev project> --input "Add a lab result 'Blood test' to intake INT-1" --stream`. A run that a token starts acts as the agent, so its `data` grant decides.
- A relation value is a record id, for example `{"title": "Blood test", "intake": "<intake id>"}`.
- After a change of links, a list can take a few seconds to show a record. A record page decides at once.
- The record page shows **Who can open this**: the direct rules, then one line for each record that gives access through a link.
