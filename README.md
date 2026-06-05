# Factuarea — Claude Code plugins

Official [Claude Code](https://claude.com/claude-code) plugin marketplace for the
**[Factuarea](https://factuarea.com) public API** — the REST API to automate
invoicing, quotes, clients, suppliers, products, VeriFactu and webhooks for
Spanish businesses.

## What's inside

| Plugin | What it does |
| --- | --- |
| `factuarea-api` | Ships a skill that gives Claude the full context of the Factuarea public API — base URL, API-key auth (`fact_live_` / `fact_test_`), UUID v7 identity, idempotency, cursor pagination, the error envelope, webhook HMAC verification, VeriFactu/FacturaE compliance, and how to generate a typed client from the OpenAPI spec. Claude loads it automatically when you ask it to build a Factuarea integration, or you can trigger it with `/factuarea-api`. Best for **writing code** against the REST API. |
| `factuarea-mcp` | Connects Claude Code to the **Factuarea MCP server** (`https://mcp.factuarea.com/mcp`) so Claude can call Factuarea tools directly — search/create/send invoices, manage clients, check VeriFactu, configure webhooks. Authenticate via OAuth (`/mcp` → Authenticate, with company + environment selection) or an API-key header. Ships a skill covering scopes, cursor pagination, the error envelope and test mode. Best for **operating your account** through tools. |

## Install

In Claude Code, add the marketplace once, then install the plugin you want:

```text
/plugin marketplace add factuarea/claude-plugins
/plugin install factuarea-api@factuarea
/plugin install factuarea-mcp@factuarea
```

`/plugin marketplace add` registers this catalog; `/plugin install` installs the
plugin from it. To update later, run `/plugin marketplace update factuarea`.

### Use it

Once installed, just ask Claude to build a Factuarea integration and it will pull
in the skill automatically. To invoke it manually:

```text
/factuarea-api
```

## What the skill knows

- **Base URL** `https://api.factuarea.com/v1` and JSON responses.
- **Auth** via `Authorization: Bearer <key>` or `X-API-Key`, with `fact_live_*`
  (production) and `fact_test_*` (isolated sandbox, external effects off) keys and
  fine-grained scopes.
- **Identity** — opaque `id` (UUID v7), foreign keys as `*_id`.
- **Idempotency** — `Idempotency-Key` header (1–64 chars, UUID v7 recommended,
  24h TTL).
- **Pagination** — cursor-based (`limit` / `starting_after` / `ending_before`,
  `{ data, has_more, next_cursor }`).
- **Errors** — the `{ error: { type, code, subcode, message, param, doc_url,
  request_id } }` envelope and status-code catalog.
- **Webhooks** — `Factuarea-Signature` HMAC SHA-256 verification with dual-signing
  on rotation and exponential retries.
- **Compliance** — VeriFactu (AEAT, RD 1007/2023), FacturaE 3.2.x, Modelos
  303/347.
- **Workflow** — start in test mode, generate a typed client from the OpenAPI
  spec, and use `/llms-full.txt` for full context.

## Resources

- Docs: <https://docs.factuarea.com>
- OpenAPI spec: <https://docs.factuarea.com/api/openapi>
- LLM index: <https://docs.factuarea.com/llms.txt> · full dump:
  <https://docs.factuarea.com/llms-full.txt>
- Claude Code guide: <https://docs.factuarea.com/guides/claude-code>
- Beta access: <https://docs.factuarea.com/beta-access> · `info@factuarea.com`

## License

[MIT](./LICENSE)
