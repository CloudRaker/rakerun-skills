---
name: rakerun-integrations
description: |
  Connect cloudraker.run agents to the outside with the rakerun CLI: secrets and the network allow list, MCP connectors (OAuth or header from a secret, grants), API tokens and scopes, webhook deliveries. Use when an agent needs a credential, a remote MCP server, a token for a caller, or when a webhook delivery failed.
allowed-tools:
  - Bash(rakerun *)
  - Bash(npx rakerun *)
---

# Secrets, connectors, tokens, webhooks

## Secrets

An agent never sees a value. It writes `${secret:NAME}` in a `network:` header template, declares `NAME` under `secrets:`, and the egress layer fills the header for that host only.

```bash
rakerun secret list                                     # name, scope, repo, agent, last used; never a value
rakerun secret set LINEAR_TOKEN                         # value prompted, hidden
printf '%s' "$VALUE" | rakerun secret set LINEAR_TOKEN  # value on stdin
rakerun secret set LINEAR_TOKEN --repo agents           # repo scope
rakerun secret set LINEAR_TOKEN --repo agents --agent linear-triage   # agent scope
rakerun secret rm LINEAR_TOKEN --repo agents
```

Scopes, highest first at run time: the run request `secrets` (only for `overridable: true` names), the agent value, the repo value, the org value. A missing value deploys with a warning and fails the first run that needs it with `secret_missing`. Values are 8 bytes or longer, encrypted, never readable after write.

`network:` rules: exact host or one leading `*.`, HTTPS 443 only, private targets refused, a wildcard host cannot carry headers, the model cannot set `authorization`. Hosts of `_apis`, `mcp.json` servers, connectors and `llm.gateway` join the allow list on their own.

## MCP connectors

A connector is a remote MCP server the org admin sets up once, with an OAuth login or a header from a secret, then grants to the org, a repo or an agent. Agents list it under `connectors: [name]` and get `mcp_{name}_{tool}`.

```bash
rakerun connector list                      # name, url, auth, status, identity, grants
rakerun connector add linear --url https://mcp.linear.app/mcp --auth oauth
rakerun connector add linear --url https://mcp.linear.app/mcp --auth oauth --scopes read,write --client-id <id> --client-secret CLIENT_SECRET_NAME
rakerun connector add internal --url https://mcp.example.com/mcp --auth secret --secret MCP_TOKEN --header Authorization --template 'Bearer {secret}'
rakerun connector add notion --url https://mcp.notion.com/mcp --auth oauth --tools search,fetch   # allowed tools only
rakerun connector connect linear            # opens the OAuth login in the browser; a human finishes it
rakerun connector grant linear              # org-wide
rakerun connector grant linear --repo agents --agent linear-triage
rakerun connector revoke linear --repo agents
rakerun connector disconnect linear
rakerun connector rm linear
```

Statuses: `connected`, `expired` (an admin logs in again with `connect`), `error`, `not_connected`. A run fails before the first model step with `connector_unavailable` when a name in `connectors:` does not exist, is not granted to the agent, or has no valid login. Check `rakerun connector list` first when a run fails that way.

Writes, logins and grants need the admin role (a token with the `admin` scope). The OAuth login needs a browser: ask the user to run `rakerun connector connect <name>`.

## Tokens

```bash
rakerun token list                                            # id, name, hint, scopes, agents, expiry, last used
rakerun token create ci --scopes run,read                     # value printed once
rakerun token create triage-only --scopes run,read --agents linear-triage --expires 2026-12-31T00:00:00Z
rakerun token revoke tok_01…
```

Scopes: `run` (start and turns), `read` (runs and events), `cancel`, `git` (push and clone), `secrets` (the `/_secrets` API), `admin` (the console API as an admin: secrets, connectors, tokens, data writes), `preview` (preview hosts), `delegate` (`x-run-as`). The CLI itself needs `admin` for writes, `read` for reads. A caller of an agent needs `run` and `read`. Never paste a token into a repo file or a chat message.

## Webhooks

Declared per agent under `webhooks:` (URL, events, `signing_secret`). Deliveries retry with backoff; dead ones stay listed.

```bash
rakerun webhook list --status failed --limit 50     # run, event type, url, attempt, status, http status, next try
rakerun webhook retry run_01… --event evt_01… --url https://ops.example.com/hooks
```

The receiver verifies the `v1` signature with the signing secret; two secrets are allowed during a rotation.
