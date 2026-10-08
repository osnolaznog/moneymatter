# MoneyMatter fork (osnolaznog)

This fork of [letehaha/moneymatter](https://github.com/letehaha/moneymatter) adds one small change to the MCP server. Everything else matches upstream.

## What changed

The MCP tools `create_transaction` and `update_transaction` accept three fields that the REST API and the transactions service already support:

| Field | Type | `create_transaction` | `update_transaction` |
|---|---|---|---|
| `payeeId` | record ID | Links the payee. Must belong to the account owner. Overrides automatic payee resolution. | Links the payee and locks it against automatic re-resolution. `null` unlinks it. |
| `externalReference` | string, max 255 | Stores an external ID, e.g. a MODO or bank operation ID. | Same. `null` clears it. |
| `externalUrl` | http(s) URL, max 2048 | Stores a related URL, e.g. a receipt or order page. | Same. `null` clears it. |

Files touched:

- `packages/backend/src/services/mcp/tools/create-transaction.ts`: new schema fields, passed to `deserializeCreateTransaction`.
- `packages/backend/src/services/mcp/tools/update-transaction.ts`: new schema fields, passed to `updateTransaction`.
- `packages/frontend/public/.well-known/mcp/server-card.json`: updated public descriptions of both tools.

No database, migration, service or business-logic changes. Payee ownership is still validated by the existing service code.

## Why

Before this change, MCP clients such as Claude could create a payee but could not link it to a transaction, and had to put provider references in the note field.

## Verification

Checked against upstream `dev` at the time of the change:

- `npm run typecheck` in `packages/backend`: no errors.
- Unit tests in `src/services/mcp` (26 tests, including the server-card drift test): passing.

## Keeping in sync with upstream

1. Sync `dev` with upstream (GitHub's "Sync fork" button, or `git fetch upstream && git rebase upstream/dev`).
2. Rebase the feature branch `feat/mcp-transaction-payee-and-external-fields` onto `dev`.
3. If the rebase conflicts, upstream changed one of the files above. Re-apply the change by hand using this document.

## Upstream PR

The goal is to upstream this change. Once it is merged, this fork can be archived and the deployment can go back to the official images.
