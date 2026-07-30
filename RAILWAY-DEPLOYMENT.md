# Railway deployment notes (this fork)

Deployment notes for running this fork on Railway. Read this first if you are an agent asked to debug, fix, or extend this setup.

This file is committed to a PUBLIC repository, so it deliberately contains no account addresses, service URLs, or project ids. Those live in `OPERATOR-NOTES.local.md`, which is gitignored and present only in the local clone. If that file is missing, ask the operator for the specifics rather than guessing.

## What this fork is

A fork of `taylorwilsdon/google_workspace_mcp` with six Gmail tools added that upstream does not have. Remote `upstream` points at the original; `git fetch upstream && git merge upstream/main` merged cleanly with zero conflicts when last checked on 2026-07-30, at which point this fork was 3 commits ahead and 0 behind.

Prerequisites for any work here: the Railway CLI installed and signed in (`railway login`), and this repo cloned locally, since deployments are built from the local working tree by `railway up` rather than from a GitHub connection. The GitHub CLI (`gh`) is only needed to sync the fork with upstream.

Tools added in this fork, all in `gmail/gmail_tools.py` and registered in `core/tool_tiers.yaml`: `list_gmail_drafts`, `get_gmail_draft`, `update_gmail_draft`, `delete_gmail_draft`, `modify_gmail_thread_labels`, `search_gmail_threads`. They exist so this deployment matches Anthropic's official Gmail connector feature for feature (the official one has draft update/list and thread labels; upstream had only draft create) while keeping upstream's advantages: sending, batch reads, attachments, and filters.

These six tools have no committed unit tests. They were verified with mocked checks that were not kept. If you refactor shared helpers, verify them by hand.

## Live services

One Railway project holds several services, one per mailbox, all running this fork's `main` via `railway up`. Service names, URLs, and which mailbox each serves are in `OPERATOR-NOTES.local.md`; run `railway service list` to see the live set.

The authoritative mailbox mapping is the connector list in claude.ai, since the mailbox is decided by whichever Google account completed that connector's consent flow, not by anything in this repo.

Each service has its own Railway volume at `/data` and Serverless (sleep after 10 idle minutes) enabled. Measured: 265 MB memory, wake about 2.8 seconds, warm about 55ms, roughly $0.02 per service per month.

Verified working against a real mailbox: `list_gmail_labels`, `list_gmail_drafts`, and `search_gmail_threads` all return live data, confirming the volume-backed credential store, the OAuth binding, and the added tools function end to end.

## The two URLs, do not mix them up

Each service exposes two paths that look similar and are not interchangeable. Confusing them produces `Couldn't connect to the server. Check that the URL points to a valid MCP server.` in claude.ai, which reads like a broken deployment but is not.

| Path | Belongs in | Purpose |
|---|---|---|
| `<service-url>/mcp` | claude.ai, Add custom connector | the MCP endpoint Claude talks to |
| `<service-url>/oauth2callback` | Google Cloud, Authorized redirect URIs | where Google returns the user after consent |

In Claude, each service is added separately under Settings, Connectors, Add custom connector, at `<service-url>/mcp`.

## One mailbox per service, and why

With `MCP_ENABLE_OAUTH21=true` the server strips `user_google_email` from every tool schema and forces the mailbox to the OAuth-authenticated caller. One connector therefore reaches exactly one mailbox. Upstream's "multi-user" support means several different people each reaching their own mailbox, not one person reaching several of their own. claude.ai also holds only one identity per connector and rejects a second connector at the same URL. So: N mailboxes means N services at N URLs.

Do not "fix" this by disabling OAuth 2.1 to get a per-call mailbox argument. Upstream issue 162 documents cross-user mailbox leakage in that mode. It is unsafe for a public endpoint.

## Adding another mailbox

1. `railway link --project <project-name>`, then `railway add --service gmail-mcp-N`.
2. `railway service link gmail-mcp-N`, then `railway volume add --mount-path /data`. The `volume add` command has no working `--service` flag; it acts on the linked service.
3. `railway domain --service gmail-mcp-N` and note the URL.
4. Set env vars (see below), with `WORKSPACE_EXTERNAL_URL` set to this service's own URL.
5. In Google Cloud Console, signed in as the account that owns the project (see `OPERATOR-NOTES.local.md`), open the project, then APIs and Services, Credentials, and click the existing OAuth client. Under Authorized redirect URIs, click Add URI, paste `https://<new-url>/oauth2callback`, and Save. One shared client serves every service, and each service's own callback must be listed or its login fails.
6. From this repo: `railway up --service gmail-mcp-N`.
7. Enable Serverless so the service sleeps when idle. The Railway dashboard has a toggle under the service's Settings, Deploy. From the CLI there is no flag for it, so use the GraphQL API. Get the ids first, then send the mutation:
   ```bash
   railway api 'query { project(id: "<project-id>") { services { edges { node { id name } } } environments { edges { node { id name } } } } }'
   railway api 'mutation { serviceInstanceUpdate(serviceId: "<service-id>", environmentId: "<environment-id>", input: { sleepApplication: true }) }'
   ```
   The project and environment ids are in `OPERATOR-NOTES.local.md`. A successful mutation returns `"serviceInstanceUpdate": true`.
8. In Claude, add a custom connector at `https://<new-url>/mcp` and authorize the target Google account. Each service sleeps and wakes independently, so a query against one mailbox leaves the others asleep.

## Required env vars per service

```
MCP_ENABLE_OAUTH21=true
WORKSPACE_EXTERNAL_URL=https://<this service's own url>
WORKSPACE_MCP_ALLOWED_CLIENT_REDIRECT_URIS=https://claude.ai/api/mcp/auth_callback,https://claude.com/api/mcp/auth_callback
WORKSPACE_MCP_CREDENTIAL_STORE_BACKEND=local_directory
WORKSPACE_MCP_CREDENTIALS_DIR=/data/credentials
WORKSPACE_MCP_OAUTH_PROXY_STORAGE_BACKEND=disk
WORKSPACE_MCP_OAUTH_PROXY_DISK_DIRECTORY=/data/oauth-proxy
TOOLS=gmail
TOOL_TIER=complete
RAILWAY_RUN_UID=0
GOOGLE_OAUTH_CLIENT_ID=<shared, from the one OAuth client>
GOOGLE_OAUTH_CLIENT_SECRET=<shared, set with railway variable set --stdin>
FASTMCP_SERVER_AUTH_GOOGLE_JWT_SIGNING_KEY=<openssl rand -hex 32, per service, never change it>
```

Do not set `PORT`: Railway injects it and the app honors it. Do not set a custom start command: the image's `CMD` is env-var driven and an override replaces it.

`RAILWAY_RUN_UID=0` is mandatory. The container runs as a non-root user, Railway mounts volumes root-owned, and without this the app crash-loops with `Permission denied: '/data/credentials'`.

`TOOL_TIER` defaults to `core`, which is only 4 Gmail tools and excludes drafting. `complete` is required.

Changing `FASTMCP_SERVER_AUTH_GOOGLE_JWT_SIGNING_KEY` makes the persisted OAuth store on that service's volume undecryptable, forcing re-authorization.

## Google Cloud setup (done once, shared)

One Google Cloud project (named in `OPERATOR-NOTES.local.md`, owned by a specific Google account that you must be signed in as): Gmail API enabled; OAuth consent screen External and published **In production**; one OAuth client of type **Web application**. Note the project-owning account is not necessarily one of the connected mailboxes. Production status matters: Testing status force-expires refresh tokens every 7 days for Gmail scopes. The app is intentionally unverified, which is fine under Google's documented personal-use exception under 100 users; the "Google hasn't verified this app" screen is expected, click Advanced then continue.

## Debugging

Startup lines that must appear in `railway logs --service <name>`:

- `Credentials directory verified: /data/credentials`. Absent means the volume permission problem, check `RAILWAY_RUN_UID=0`.
- `Using FileTreeStore for FastMCP OAuth proxy client_storage (directory=/data/oauth-proxy)`. Absent means OAuth state will not survive a sleep, check the two proxy storage vars.
- `Tool tier 'complete' loaded`. If it says `core`, `TOOL_TIER` is unset.

Health and discovery checks, both should be 200 and the issuer must match that service's own URL:

- `<url>/health`
- `<url>/.well-known/oauth-authorization-server`
- `POST <url>/mcp` with no auth should return 401 with a `WWW-Authenticate` header

Being re-prompted for Google consent after an idle period means credentials did not persist: check both `/data` paths and that the signing key has not changed.

A 502 on the first request after idle is Railway's documented Serverless wake behavior. Retry once. Persistent 502 across retries is a real failure.

## Known limits

- The six added tools have no committed automated tests, so the repo's own `pytest` run will not catch a regression in them. Verify them by hand after any refactor of shared helpers such as the MIME builder.
- One mailbox per connector, as explained above.
- Railway Hobby blocks outbound SMTP, so any IMAP/SMTP-based mail server cannot send on this plan. This only matters if a non-Google mailbox is added later; Gmail and Microsoft Graph send over HTTPS and are unaffected.
- Out of scope by decision: Microsoft 365 work accounts (organizational policy risk) and Zoho (free plans have no IMAP).
