# CloudRaker Run skills

This repo ships agent skills for `rakerun`, the CLI of cloudraker.run. The skills live in `skills/<name>/SKILL.md` (agentskills.io format). Start with `skills/rakerun/SKILL.md`, the umbrella.

If you work in this repo: the skills are hand-written docs, not generated. Keep the frontmatter to the spec keys (`name`, `description`, `allowed-tools`). Keep the house rules the same in every skill: a folder with `AGENTS.md` is an agent, `rakerun deploy` is the only way to ship, `rakerun lint` before every deploy, secrets are `${secret:NAME}` references and never values. Verify every command against `rakerun --help` before you document it. Verify every `agent.yml` key against https://docs.cloudraker.com/run/guides/agent-yml.
