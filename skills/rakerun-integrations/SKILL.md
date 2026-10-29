---
name: rakerun-integrations
description: |
  Connect cloudraker.run agents to the outside with the rakerun CLI: secrets and the network allow list, MCP connectors (OAuth, header from a secret or Run identity OIDC, grants), signed OIDC identity on outbound requests, API tokens and scopes, webhook deliveries. Use when an agent needs a credential, a remote MCP server, a token for a caller, or when a webhook delivery failed.
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

`network:` rules: exact host or one leading `*.`, HTTPS 443 only, private targets refused, a wildcard host cannot carry headers or `oidc`, the model cannot set `authorization`, injected headers and tokens are dropped on a redirect to another host. Hosts of `_apis`, `mcp.json` servers, connectors and `llm.gateway` join the allow list on their own.

## Signed identity instead of a secret (OIDC)

When an internal API can verify a JWT, give its host an `oidc:` block instead of a secret. The egress layer sends an RS256 token (at most 300 s) that names the agent (`sub: agent:{repo}/{agent}`) or the person of the turn (`sub: user:{workos_user_id}`, `email`, `act.sub`). Nothing to store, nothing to rotate on your side.

```yaml
network:
  api.internal.acme.com:
    oidc: { identity: user } # audience https://api.internal.acme.com; fixed for user
  vault.acme.com:
    oidc: { identity: agent, audience: vault://run, header: X-Run-Token }
```

- Issuer: `https://{org}.cloudraker.run`. Discovery: `/.well-known/openid-configuration`. JWKS: `/.well-known/jwks.json` (cache one hour). The console **Settings** screen shows both URLs.
- Tell the API owner to pin `iss`, `aud` and `sub`, and for a user token to require an allowed agent in `act.sub` (else any agent of the org can act for a member). Check `exp` but do not require a fixed lifetime, and refuse a `branch` claim when previews must not call. One key set signs for every org, so `iss` is the only org check. `jti` is not one-time: a token is reused while more than 60 s remain.
- `identity: user` needs a verified org member in the turn (console, team apps, mapped Slack or Teams sender). A mention run in a console conversation with no sender names the conversation owner. Schedules, webhooks, phone, email, web chat visitors and API tokens have none. Then a fetch, `_apis` or `_code` request fails with `identity_unavailable`. An MCP server or connector cannot list its tools, so its tools are missing from the turn.
- Audit rows show `oidc:agent` or `oidc:user` as the secret name. Never the token.

Full guide and a `jose` verification example: https://docs.cloudraker.com/run/guides/oidc-identity.

## MCP connectors

A connector is a remote MCP server the org admin sets up once, with an OAuth login, a header from a secret or a Run identity (OIDC), then grants to the org, a repo or an agent. Agents list it under `connectors: [name]` and get `mcp_{name}_{tool}`.

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
rakerun connector tools linear              # the server tools, and which ones the connector allows
rakerun connector rm linear
```

A Run identity (OIDC) connector sends a signed token per request: no login, no stored token, status always `connected`. The audience defaults to `https://{host}` of the URL. Create it in the console (**Add connector**, auth **Run identity (OIDC)**, optional audience); `rakerun connector add` does not offer it yet. The agent picks the identity: `- name: crm` then `identity: user` in `connectors:` (default `agent`). Grant it like any connector.

Statuses: `connected`, `expired` (an admin logs in again with `connect`), `error`, `not_connected`. A run fails before the first model step with `connector_unavailable` when a name in `connectors:` does not exist, is not granted to the agent, or has no valid login. Check `rakerun connector list` first when a run fails that way.

Each tool definition goes to the model on every step, so a large server (Linear, GitHub) costs many tokens. Keep only the tools the agent needs: `--tools` on the connector for the whole org, or a `tools:` list in agent.yml for one agent (`- name: linear` then `tools: [list_issues]`). An agent only narrows the connector list. A name the server lacks, or that the connector excludes, fails the run with `connector_unavailable`. `*` matches any characters. `mcp.json` servers take the same `tools` key.

Writes, logins and grants need the admin role (a token with the `admin` scope). The OAuth login needs a browser: ask the user to run `rakerun connector connect <name>`.

## Tokens

```bash
rakerun token list                                            # id, name, hint, scopes, agents, expiry, last used
rakerun token create ci --scopes run,read                     # value printed once
rakerun token create triage-only --scopes run,read --agents linear-triage --expires 2026-12-31T00:00:00Z
rakerun token revoke tok_01…
```

Scopes: `run` (start and turns), `read` (runs and events), `cancel`, `git` (push and clone), `secrets` (the `/_secrets` API), `admin` (the console API as an admin: secrets, connectors, tokens, data writes), `preview` (preview hosts), `act_as` (`x-run-as`). The CLI itself needs `admin` for writes, `read` for reads. A caller of an agent needs `run` and `read`. Never paste a token into a repo file or a chat message.

## Webhooks

Declared per agent under `webhooks:` (URL, events, `signing_secret`). Deliveries retry with backoff; dead ones stay listed.

```bash
rakerun webhook list --status failed --limit 50     # run, event type, url, attempt, status, http status, next try
rakerun webhook retry run_01… --event evt_01… --url https://ops.example.com/hooks
```

The receiver verifies the `v1` signature with the signing secret; two secrets are allowed during a rotation.
