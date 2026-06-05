# Factuarea — Claude Code plugins

Official [Claude Code](https://claude.com/claude-code) plugin marketplace for
**[Factuarea](https://factuarea.com)** — the multi-tenant invoicing SaaS for
Spanish businesses (invoicing, quotes, clients, suppliers, products, VeriFactu
and webhooks).

## What's inside

| Plugin | What it does |
| --- | --- |
| `factuarea-mcp` | Connects Claude Code to the **Factuarea MCP server** (`https://mcp.factuarea.com`) so Claude can call Factuarea tools directly — search/create/send invoices, manage clients, check VeriFactu, configure webhooks. Authenticate via OAuth (`/mcp` → Authenticate, with company + environment selection) or an API-key header. Ships a skill covering the tool domains, scopes, cursor pagination, the error envelope and test mode. |

## Install

In Claude Code:

```text
/plugin marketplace add factuarea/claude-plugins
/plugin install factuarea-mcp@factuarea
```

`/plugin marketplace add` registers this catalog; `/plugin install` installs the
plugin from it. To update later, run `/plugin marketplace update factuarea`.

## Connect the server

The plugin declares the server **without an auth header**, so the recommended
path is OAuth:

1. Run `/mcp`, pick **factuarea**, choose **Authenticate**.
2. Approve in the browser — Dynamic Client Registration + PKCE are automatic. On
   the consent screen you **select the company and the environment** (live or
   test) and the scopes to grant.

Prefer your own API key? Connect with a static header instead:

```bash
claude mcp add --transport http factuarea https://mcp.factuarea.com \
  --header "Authorization: Bearer fact_live_xxxxxxxxxxxxxxxxxxxxxxxx"
```

Use a `fact_test_` key for the isolated sandbox (external effects off). The
canonical endpoint is the root `https://mcp.factuarea.com`; the older
`https://mcp.factuarea.com/mcp` keeps working as a compatibility alias.

## What the skill knows

- **Connecting** via OAuth (consent with company + environment selection) or an
  API-key header.
- **Channel policy** — an API key reaches the full **223 tools**; OAuth uses a
  curated **215**, never granting `verifactu:write` or the GDPR signature-forget
  operation to third-party apps.
- **15 tool domains** and their scopes, plus how plan/module and feature flags
  further narrow what's listed.
- **Identity** — opaque `id` (UUID v7), foreign keys as `*_id`.
- **Cursor pagination** — `{ data, has_more, next_cursor }`, no page numbers.
- **Errors** — `insufficient_scope`, `addon_not_active`, `422` business-rule
  violations, `429` with `Retry-After`.
- **Test mode** — the `fact_test_` prefix / test OAuth grant points at an
  isolated sandbox with external effects switched off.

## Resources

- MCP docs: <https://docs.factuarea.com/mcp> — connect · authentication · tools ·
  scopes · errors · test-mode
- Docs home: <https://docs.factuarea.com>
- OpenAPI spec: <https://docs.factuarea.com/api/openapi>
- Dashboard / API keys: <https://app.factuarea.com/settings/developers/api-keys>
- Beta access: <https://docs.factuarea.com/beta-access> · `info@factuarea.com`

## License

[MIT](./LICENSE)
