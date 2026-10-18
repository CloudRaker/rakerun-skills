---
name: rakerun-knowledge
description: |
  Give cloudraker.run agents memory and reference material: Structured Memory (typed records from _schemas/ files, the data: grant, the data.* tools, records from the rakerun CLI) and knowledge folders (Markdown under knowledge/ with knowledge: grants and the search and read tools). Use when an agent must store records, read records another agent wrote, or answer from documents in the repo.
allowed-tools:
  - Bash(rakerun *)
  - Bash(npx rakerun *)
---

# Structured Memory and knowledge

## Structured Memory: typed records of the org

A schema is one `_schemas/{name}.yml` file. `_schemas/` inherits like `_apis/`. Records belong to the org, not the agent: two agents on the same schema see the same records.

```yaml
# _schemas/lead.yml
name: lead            # [a-z][a-z0-9_]{0,62}, immutable after the first deploy
label: Lead
label_field: email
fields:
  email: { type: email, required: true, unique: true }
  company: { type: text }
  score: { type: number }
  stage: { type: select, options: [new, qualified, lost] }
  notes: { type: long_text }
```

Field types: `text`, `long_text`, `number`, `currency`, `boolean`, `date`, `datetime`, `email`, `phone`, `url`, `address`, `select`, `multi_select`. A push can add a field. A push that removes or retypes one fails with `schema_breaking`: add a new schema instead. `rakerun lint --repo <name>` checks that against the live tree before you deploy.

Grant access in `agent.yml`:

```yaml
data:
  read: [lead, '*']       # '*' = every Run schema of the org
  write: [lead]
  create_schemas: false   # true adds data.define_schema: a draft schema and a pull request, auto-merged in 7 days
```

The agent gets `data.schemas`, `data.query`, `data.get`, `data.create`, `data.update`, `data.delete`, `data.upsert` (create or update on a unique field; the usual tool for enrichment). Say in `AGENTS.md` when to write: "Upsert one `lead` record per qualified email". A request with `metadata.end_user` stamps records with that user; later queries of that user see only their records and the org-wide ones.

From the CLI:

```bash
rakerun data schemas agents                       # name, status (draft | stable | archived), records, pull request
rakerun data records agents lead --limit 50       # a table, one column per field
rakerun data records agents lead --json | jq '.records[] | select(.data.stage == "qualified")'
rakerun data add agents lead --data '{"email": "a@b.co", "score": 3}'
rakerun data update agents lead <id> --data '{"stage": "qualified"}'
rakerun data rm agents lead <id>
rakerun data merge agents lead                    # merge a draft schema now
rakerun data archive agents lead                  # close the draft; records archived
rakerun data drop agents lead                     # delete an archived schema and its records, for good
```

Writes need the admin scope. Prefer letting the agent write through its tools; use `data add` for seed and test data.

## Knowledge folders: Markdown the agents search

Put pricing sheets, policies and playbooks under `knowledge/` in the repo. Every live deploy indexes the `.md` and `.markdown` files (256 KB each, 2,000 files, 20,000 chunks per repo). A preview deploy does not index; it reads the live index.

```text
knowledge/
  sales/pricing.md
  sales/pre-sales/discovery.md
  legal/nda.md
```

Grant folders in `agent.yml`. The list replaces a parent list; a missing key gives no knowledge and no tools:

```yaml
knowledge: [sales/pre-sales, legal]   # that folder and everything under it
# knowledge: ['*']                      # all of knowledge/, the only way to get files at its root
```

The agent gets `knowledge_search` (`query`, `top_k` 1 to 10, optional `folder` inside a grant; results carry `path`, `title`, `heading`, `text`, `score`, and `stale: true` when the index lags) and `knowledge_read` (`path`; the file at the deployment SHA of the run, 100,000 characters at most). Tell the agent when to use them: "Use knowledge_search for any pricing question and quote the heading".

Write knowledge for search: one topic per file, `#` to `###` headings (the indexer chunks on them, about 1,500 characters per chunk), a front matter `title`. Lint warns about non-Markdown files, oversized files and grants that match no file (`knowledge_folder_missing`). `rakerun agent show <agent>` prints the knowledge grants; the console agent page shows the index status (`indexed`, `pending`, `failed`, `credits_exhausted`).

Costs: 1 credit per indexed chunk, 2 credits per search. Unchanged files are not indexed again.
