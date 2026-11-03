---
name: rakerun-business-records
description: |
  Design the records of a CloudRaker Run project with a person who is not a developer: record types and their fields, "Linked to" fields and the direction in which they open records, who can open, change and delete each record type, teams of people, sharing, and a queue where incoming documents wait until they are filed. You write the files, read back the sentences of `npx rakerun lint --explain` and change the files until the person agrees. Use when the person wants to track something new, link records, decide who sees what, add a team, or sort incoming documents.
allowed-tools:
  - Bash(npx rakerun *)
  - Bash(rakerun *)
---

# Records, links and who can open what

The person decides in plain words. You write the files. Save each change as the [rakerun-business](../rakerun-business/SKILL.md) skill says: a change, a check, a try, then a publisher makes it live.

Never say space, reach, stamp, hop, relation, takes, gives or rows to the person. Say the sentences of `npx rakerun lint --explain` instead.

## Record types

A record type is one file, `_schemas/<name>.yml`, at the root of the project. The Records page of the console shows it with its `label`.

```yaml
# _schemas/intake.yml
name: intake          # lowercase letters, digits and _; never change it after the first save
label: Intake         # the name that people see
label_field: number   # the field that names one record, for example INT-0042
fields:
  number: { type: text, required: true, unique: true }
  nurse: { type: email }
  status: { type: select, options: [new, review, accepted, refused] }
  patient: { type: relation, to: patient, access: gives }
access:
  read: [nurses, triage]
  write: [nurses, triage, admins]
  rows:
    owner: nurse
    all: [triage, admins]
```

Field types: `text`, `long_text`, `number`, `currency`, `boolean`, `date`, `datetime`, `email`, `phone`, `url`, `address`, `select`, `multi_select` and `relation`. A saved change can add a field. It cannot remove a field or change its type: add a new field instead.

Ask the person, one question at a time:

1. "What do you keep track of?" Each thing with its own number, owner or status is a record type.
2. "What do you need to know about one intake?" The answers are the fields.
3. "What names one intake?" The answer is `label_field`.

## "Linked to" fields

A field with `type: relation` links a record to records of another record type of the same project. The console shows it as "Linked to".

| Key | Values | Meaning |
| --- | --- | --- |
| `to` | A record type of the project | The linked record type |
| `many` | `false` (default) or `true` | One record, or a list of at most 50 |
| `access` | `none` (default), `takes` or `gives` | The direction in which the link opens records |
| `filing` | `{ auto_at: 0.9 }` | Only on the field of a document queue (see below) |

The field is on one record and points to another record. `access` says which way the link opens records:

- `takes`: people who can open the linked record can also open this record. Put it on the part of a case: a lab result takes from its intake, and an invoice line takes from its invoice.
- `gives`: people who can open this record can also open the linked record. Put it on a case that points to a shared record: an intake gives its patient, and a claim gives its vendor.
- `none`: the link opens nothing. Use it when the link is only a reference.

Ask the person, one question at a time:

1. "Should the people who work on an intake also see its lab results?" A yes gives `access: takes` on the `intake` field of the lab result.
2. "Should the people who work on an intake also see its patient?" A yes gives `access: gives` on the `patient` field of the intake.
3. "Through the intake, can they change the lab results, or only read them?" The answer goes in `access.links` of the lab result: `write` (default), `read` or `none`.

### One link, never further

Access through a link stops there. It never continues through a second link. A nurse is the nurse of intake INT-1, and INT-1 is linked to patient P-1:

| Record | Can she open it? | Why |
| --- | --- | --- |
| Intake INT-1 | Yes, and change it | She is its nurse |
| Patient P-1 | Yes, and change it | Through INT-1 (the intake gives its patient) |
| Lab result L-1 of INT-1 | Yes, and change it | Through INT-1 (the lab result takes from its intake) |
| Appointment A-1 of INT-1 | Yes, but only read it | Through INT-1, and appointments have `links: read` |
| Appointment A-2 of P-1 and of intake INT-7 | No | Her access to P-1 comes through a link, and it stops there |
| Intake INT-7 of the same patient | No | INT-7 opens P-1. P-1 does not open INT-7 |

### Rules of links

- Delete never passes through a link.
- To add a link that opens records, a person must be able to change both records. To remove a link, a person must be able to change the record that holds the field.
- A person who can change an intake directly (a table rule, a row rule or a share, not another link) can add a lab result that takes from it, when `access.links` of lab results is `write`.
- A lab result that a person adds this way cannot give them direct access. This matters when the person can only read lab results and a row rule decides which ones they see. Then its owner field cannot name them, and its team field cannot name their team. With `rows.owner: $creator`, they cannot add a lab result this way, because Run makes them its creator.
- Never put `gives` on a shared record that points back to its cases. One patient would then open every intake of that patient.
- A `gives` field on a record type that every member can open opens the linked records to every member. `npx rakerun lint` warns about it.
- A link into a record type with `links: none` opens nothing.

## Who can open what

The `access` block of a record type decides who can open, change and delete its records. With no block, every member can open and change the records, and admins can delete them.

| Key | Meaning | Default |
| --- | --- | --- |
| `read` | Who can open the records | `[members]` |
| `write` | Who can change the records | `[members]` |
| `delete` | Who can delete the records. Each entry must also be in `write` | `[admins]` |
| `rows.owner` | An email field (or `$creator`): each person sees the records assigned to them (or that they added) | |
| `rows.team` | A select or text field with team slugs as values: each person sees the records of their teams | |
| `rows.all` | Who sees all records when `rows` is set | `[admins]` |
| `shares` | `none`, `read` or `write`: what a share of one record gives | `read` |
| `links` | `none`, `read` or `write`: the most that a link gives on this record type | `write` |

An entry is `members`, `admins` or a team slug. Never put an email in these lists. To open one record to one person, the person shares it.

When `write` does not have `admins` and the block has no `delete`, the check fails, because admins can delete by default. Add `admins` to `write`, or set `delete: []`.

Ask the person, one question at a time:

1. "Who works on intakes?" The answer gives `read` and `write`.
2. "Who can delete them?" The answer gives `delete`.
3. "Does each person see all intakes, or only their own?" The answer gives `rows.owner` or `rows.team`.
4. "Who sees all of them?" The answer gives `rows.all`.
5. "Can people share one intake with a colleague?" The answer gives `shares`.

## Read it back

After each change to record types, links or access, run:

```bash
npx rakerun lint --explain
```

```text
Intake (_schemas/intake.yml)
  Nurses, Triage and Admins can open intakes.
  Nurses, Triage and Admins can change intakes.
  Admins can delete intakes.
  Each person sees the intakes assigned to them.
  Triage and Admins see all intakes.
  People who can open an intake can also open its patient.
  People who can open an intake can also open its lab results.
  Access through a link stops there. It does not continue to other links.
```

Read each sentence to the person, word for word. Leave out the file names in parentheses. When you are signed in, the sentences use the names of the teams. Ask: "Is this right?" Never save a change to access before the person says yes.

When a publisher makes the change live, the dialog lists each record type that more people can open, for example "People who can open intakes can now open patients (412 records)." Ask the person to read that list before they confirm.

## Teams

Teams are groups of people (not Microsoft Teams). An access rule names a team by its slug.

```bash
npx rakerun teams ls                                # slug, name, source, people
npx rakerun teams add "Reception" --slug reception  # admins only
```

- When `teams add` answers `forbidden`, ask the person to ask an admin to add the team in Settings > Teams.
- Admins add people to teams in Settings > Teams. Teams from the company directory show the source `directory`.
- A team that does not exist yet is a warning on a change and an error when the change goes live. Add the team first.

## Roles

| Role | Where | What the person can do |
| --- | --- | --- |
| Admin | Settings > People | Everything. Makes changes live. |
| Builder | "Can build" in Settings > People | Makes changes and reads every record type. Opens records only through the access rules, as a member does. |
| Member | The default | Home, Inbox, Conversations, Apps, Outputs, Records and the assistants that the person can use. No build pages. |

Admins make changes live. A builder or a member makes changes live only when the root `agent.yml` lists their email in `access.publishers`.

## Sharing

A person can share one record with a colleague or a team on the record page. The share also opens the records that the shared record opens through its links. The share sheet says so: "Also opens 3 linked records." A person who can change a record, also only through a link, can share it. Such a share never names the person who shares, or a team of that person. A share never gives more access to the person who made it.

## A queue for documents

When documents come in (by email or by upload) and someone must sort them, make a queue. A document waits in the queue until a confident agent or a person files it to its case. While it waits, only the queue team can open it.

```yaml
# _schemas/inbound_document.yml
name: inbound_document
label: Inbound document
label_field: subject
fields:
  subject: { type: text }
  sender: { type: email }
  file_id: { type: text }          # the file id of the attachment
  status: { type: select, options: [waiting, filed, refused] }
  intake:
    type: relation
    to: intake
    access: takes                  # after the filing, the people of the intake open it
    filing: { auto_at: 0.9 }       # an agent files alone at 0.9 or more
access:
  read: [reception, admins]
  write: [reception, admins]
```

- A record type has one filing field. It must be `access: takes`, and it cannot be required. `auto_at` is from 0.5 to 1 (default 0.9).
- The filing agent adds one document record for each attachment. Then it calls `data.file` with the document, the field, the intake, a confidence from 0 to 1 and a reason.
- At or above `auto_at`, Run writes the link and sets `status` to `filed`. Below it, the queue team gets "Where does this go?" in Needs you of the Inbox: file here, choose another record, or refuse. The person who files must be able to change the intake. The first decision wins.
- When the agent finds no intake, it calls `flag` to the queue team with the document. The item shows in Needs you, and the agent goes on.
- When someone removes the link of a filed document, the document goes back to the queue. The queue team gets "Where does this go?" again, with no suggestion.
- The reply of the filing agent goes to the sender by email. The sender may not be able to open the intakes. So the reply holds no record data: it never names an intake, a patient or another record, and it never says whether a record exists.
- A confident wrong guess can file a document to the wrong case. Tell the person to check the documents that the agent filed alone. A higher `auto_at` sends more documents to a person.

The filing agent works alone, so its `data` lists decide what it can open:

```yaml
# mail-intake/agent.yml
mode: zeroshot
data:
  read: [inbound_document, intake]
  write: [inbound_document]
```

```markdown
# mail-intake

You file the documents that come in by email.

1. For each attachment, add one inbound document with `data.create`: the subject, the sender, the file id and the status `waiting`.
2. Find its intake with `data.query` on intakes: the intake number, or the name of the patient.
3. Call `data.file` with the document, the field `intake`, the id of the intake, your confidence from 0 to 1 and a short reason.
4. When no intake fits, call `flag` to `reception` with the document. Do not guess.
5. Reply with one sentence: the number of documents that you received. Never name an intake, a patient or another record, and never say whether a record exists.
```

An email front door starts the filing agent for each message:

```yaml
# doors/inbound-mail.yml
name: Inbound documents
entry: mail-intake
channels:
  email: { allow_from: ['@clinic.example'] }   # the senders that may write
```

After the change is live, bind an email address to the door on the Channels page of the console, or with `npx rakerun channel bind email <alias> --door <project>/inbound-mail`.

## Test records

```bash
npx rakerun data schemas <project>                         # the record types and their counts
npx rakerun data records <project> intake --limit 20
npx rakerun data add <project> intake --data '{"number": "INT-0001", "status": "new"}'
```

`data add` needs an admin. Prefer to let the person add records in the console, or an agent through its tools.
