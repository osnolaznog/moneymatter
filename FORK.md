# MoneyMatter fork (osnolaznog)

This fork of [letehaha/moneymatter](https://github.com/letehaha/moneymatter) adds a small change to the MCP server. Everything else matches upstream.

## What changed

The MCP tools `create_transaction` and `update_transaction` accept two fields that the REST API and the transactions service already support:

| Field | Type | `create_transaction` | `update_transaction` |
|---|---|---|---|
| `externalReference` | string, max 255 | Stores an external ID, e.g. a MODO or bank operation ID. | Same. `null` clears it. |
| `externalUrl` | http(s) URL, max 2048 | Stores a related URL, e.g. a receipt or order page. | Same. `null` clears it. |

Tools that return transactions through the shared MCP serializer (for example `search_transactions`) include `payeeId`, `externalReference` and `externalUrl` in each transaction, so clients can read back what they wrote.

`payeeId` on `create_transaction` / `update_transaction` comes from upstream, not from this fork.

Files touched:

- `packages/backend/src/services/mcp/tools/create-transaction.ts`: new schema fields, passed to `deserializeCreateTransaction`.
- `packages/backend/src/services/mcp/tools/update-transaction.ts`: new schema fields, passed to `updateTransaction`.
- `packages/backend/src/services/mcp/serializers.ts` (+ its unit test): the three fields added to `slimTransactionsForMcp`.
- `packages/frontend/public/.well-known/mcp/server-card.json`: public descriptions of both tools mention the external fields.
- `packages/backend/src/services/mcp/serializers.unit.ts`: the goal "reached" status test uses a date relative to today instead of a fixed date. Upstream bug: with a fixed date the test fails once the 30-day hold window passes.

No database, migration, service or business-logic changes.

## Why

MCP clients such as Claude had to put provider references (like a MODO operation ID) in the note field because the tools did not expose `externalReference` / `externalUrl`.

## Build and deployment

`.github/workflows/fork-build-backend.yml` runs daily on `dev` (06:17 Buenos Aires). Once the official backend image for upstream `main` is published, it cherry-picks the commits on `feat/mcp-transaction-payee-and-external-fields` (everything after upstream `dev`) onto upstream `main`, runs the backend typecheck and the `src/services/mcp` unit tests, and publishes `ghcr.io/osnolaznog/moneymatter-be` tagged `latest` and `sha-<upstream main short SHA>`. If the commits no longer apply, the run fails and the server keeps its current image.

### Deploying to the server

The server (Hostinger VPS, `/opt/moneymatter`, compose project `budget-tracker-prod`) runs this backend image with the official frontend image. Both are pinned to one tag through `IMAGE_TAG` in `/opt/moneymatter/.env`. To deploy a new build:

```bash
cd /opt/moneymatter
ls -lt backups | head -3                      # today's 04:15 backup must exist
grep '^IMAGE_TAG' .env                        # note the current tag for rollback
sed -i 's/^IMAGE_TAG=.*/IMAGE_TAG=sha-<new>/' .env
docker compose pull && docker compose up -d
docker compose ps                             # wait for healthy
```

Rollback: put the previous tag back in `.env` and repeat `pull` + `up -d`. The backend runs pending migrations on start, so roll back across migrations only together with a database restore from `backups/`.

After a backend restart, existing MCP sessions are invalid ("Invalid or expired session ID"); reconnect the MCP client.

### Server notes

- Traefik rate limit for MoneyMatter lives in `/opt/moneymatter/docker-compose.override.yml` (labels on the frontend service): `average=30`, `burst=300`. The first page load after an update makes ~250 requests, so a lower burst returns 429.
- All containers (MoneyMatter, n8n, Traefik, Postgres) use `restart: unless-stopped` or `always`, so they come back on their own after a reboot. After a reboot n8n takes about a minute before `/healthz` answers.
- Held apt packages (`apt-mark showhold`):
  - `containerd.io`, `docker-ce`, `docker-ce-cli`, `docker-ce-rootless-extras`, `docker-buildx-plugin`, `docker-compose-plugin`: major upgrades pending (containerd 1.7 to 2.x, Docker 28 to 29, Compose 2 to 5). Upgrade them on their own, with a fresh backup, then check every container comes back healthy. Release with `apt-mark unhold <pkg>`.
  - `libpq5`: the `apt.postgresql.org` repo on the host fails signature verification, so apt keeps a stale index pointing to a removed package. Nothing on the host depends on it (databases run in containers).
- Manual backups before risky maintenance: `/root/backups/n8n-db-<date>.sql.gz`, `/root/backups/n8n-data-<date>.tgz` (includes the n8n encryption key), `/opt/moneymatter/backups/mm-db-<date>.sql.gz`. MoneyMatter also has a daily dump at 04:15 in `/opt/moneymatter/backups/`.

## Keeping in sync with upstream

1. Sync `dev` with upstream (GitHub's "Sync fork" button, or `git fetch upstream && git rebase upstream/dev`).
2. Rebase the feature branch `feat/mcp-transaction-payee-and-external-fields` onto upstream `dev`.
3. If the rebase conflicts, upstream changed one of the files above. Keep upstream's code and re-add only what this document lists.
4. If upstream ships one of these fields itself, drop it from the fork and update this document.

## Sync log

- 2026-10-31: the daily build had failed since 2026-10-19, when upstream changed both MCP transaction tools (native `payeeId`, optional `paymentType`, `paymentLegs`). The feature branch was rebuilt on upstream `dev` (`cc5d8901`) with upstream's `payeeId` and only the fork's external fields on top. Verified on upstream `main` v0.33.7 (`cf716812`): commits apply cleanly, `npm run typecheck` passes, `src/services/mcp` unit tests pass (62/62). Previous branch kept as `backup/feat/mcp-transaction-payee-and-external-fields-2026-10-08`. Server moved from `sha-640811b` to `sha-cf71681`; all containers healthy. Same day: Traefik burst raised to 300, 72 Ubuntu packages upgraded (Docker and `libpq5` held), VPS rebooted; MoneyMatter and n8n back up and healthy.

## Upstream

No upstream PR is planned. The fork is maintained as long as the server needs the external fields.
