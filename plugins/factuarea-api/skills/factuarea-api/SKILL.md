---
name: factuarea-api
description: Build an integration with the Factuarea public API — invoicing, quotes, clients, suppliers, products, VeriFactu, FacturaE and webhooks for Spanish businesses. Use this when the user wants to call api.factuarea.com, generate a Factuarea API client, automate invoicing/billing, react to Factuarea webhooks, or work with VeriFactu/AEAT compliance from code.
---

# Factuarea public API

Factuarea is a multi-tenant invoicing SaaS for Spanish businesses. Its public
REST API lets you automate invoices, quotes, pro-formas, delivery notes,
recurring invoices, purchase invoices, clients, suppliers, products, document
series, taxes, VeriFactu (AEAT), FacturaE and webhooks.

This skill is the context Claude needs to write correct calls. It is a summary —
the **source of truth is the OpenAPI spec** (link below). When you need an exact
field, schema, or endpoint, read the spec; do not invent fields or endpoints.

## Start here (recommended workflow)

1. **Always start in test mode.** Use a `fact_test_` key first so you never touch
   production data, send real emails, or submit to AEAT while building.
2. **Generate a typed client from the OpenAPI spec** instead of hand-rolling
   HTTP calls. Download `https://docs.factuarea.com/api/openapi` and run your
   generator of choice (e.g. `openapi-typescript`, `openapi-generator`,
   `openapi-python-client`).
3. **Load full context when stuck.** Fetch `https://docs.factuarea.com/llms-full.txt`
   for the entire documentation as one Markdown file, or `/llms.txt` for a curated
   index. Every rendered page also has a "Copy Markdown" action.

## Base URL

```
https://api.factuarea.com/v1
```

All endpoints are under `/v1`. Responses are JSON.

## Authentication

Every request needs an API key, bound to one company. Send it one of two ways
(use only one; if both are present `Authorization: Bearer` wins):

```
Authorization: Bearer fact_live_xxxxxxxxxxxxxxxxxxxxxxxx
```

or

```
X-API-Key: fact_live_xxxxxxxxxxxxxxxxxxxxxxxx
```

Key prefixes determine the **environment** (the prefix is the source of truth —
no request parameter changes it):

- `fact_live_*` → **production**. Real fiscal numbering, VeriFactu → AEAT, emails
  to clients, outbound webhooks all fire.
- `fact_test_*` → **test / sandbox**. An isolated sandbox company; external
  effects (VeriFactu/AEAT, email, webhooks) are switched off. The API surface is
  identical to live.

Keys carry **fine-grained scopes** (e.g. `invoices:read`, `clients:write`).
A call outside a key's scopes returns `403` with `code: insufficient_scope`.

**Getting a key:** the dashboard at
`https://app.factuarea.com/settings/developers/api-keys` (Settings → Developers →
API Keys). Choose the Test environment for a `fact_test_` key. The API is in
**private beta** — companies must be allowlisted first; request access at
`info@factuarea.com` (see `https://docs.factuarea.com/beta-access`).

## Identity (IDs)

Every resource is identified by an opaque `id` that is a **UUID v7 string**
(not an integer). Foreign keys are exposed as `*_id` (e.g. `client_id`,
`series_id`) and also carry UUID v7 values. Do not parse or generate these IDs —
receive them and send them back as-is. On create (`POST`), do **not** send an
`id`; the server generates it and returns it.

## Idempotency

Send an `Idempotency-Key` header on **all** mutating requests (POST and other
writes) so retries never create duplicates:

- Value: any opaque string, **1–64 characters**. Recommended: a UUID v7
  (generate one per logical operation).
- TTL: the server remembers the key for **24 hours** and replays the original
  response for repeats.
- Reusing the same key with a **different request body** returns `409` with
  `code: idempotency_key_reused`.
- Generate the key **once per operation and reuse it across retries** — never
  regenerate inside a retry loop.

## Pagination (cursor-based)

List endpoints use **cursor** pagination, not page numbers. Do not use `?page=`.

- Request params: `limit`, `starting_after` (id of the last item from the
  previous page → next page), `ending_before` (id → previous page).
- Response shape:

```json
{ "data": [ /* ... */ ], "has_more": true, "next_cursor": "<id>" }
```

Loop while `has_more` is `true`, passing the previous page's last `id` (or
`next_cursor`) as `starting_after`.

## Errors

Failures return a consistent envelope:

```json
{
  "error": {
    "type": "authentication_error",
    "code": "missing_api_key",
    "subcode": null,
    "message": "Falta la clave de API.",
    "param": null,
    "doc_url": "https://docs.factuarea.com/guides/errors",
    "request_id": "req_..."
  }
}
```

- `type` — machine-readable category. One of: `authentication_error`,
  `authorization_error`, `permission_error`, `invalid_request_error`,
  `not_found_error`, `conflict_error`, `idempotency_error`, `rate_limit_error`,
  `api_error`, `service_unavailable_error`.
- `code` — stable error code (e.g. `missing_api_key`, `insufficient_scope`,
  `parameter_invalid`, `invalid_status_transition`). Branch on `code`, not on the
  human message.
- `subcode` — extra detail on some `409`s (e.g. `code=resource_already_exists`,
  `subcode=tax_id_already_exists`).
- `message` — human text **in Spanish** (it is the API's real response — surface
  it as-is or map by `code`).
- `param`, `doc_url`, `request_id` — offending field, docs link, and a tracing id
  to include in support requests.

Status codes: `401` (auth), `403` (scope/permission), `404` (not found),
`409` (conflict / duplicate / idempotency reuse), `422` (validation or business
rule violation), `429` (rate limited — back off), `5xx` (server / unavailable).

## Webhooks

Events are delivered as signed `POST` requests. Verify them:

- Header `Factuarea-Signature: t=<unix_ts>,v1=<hex>`. The signature is
  **HMAC SHA-256** of the string `{t}.{raw_body}` using your endpoint secret.
- **Dual signing on rotation:** during a secret-rotation grace window the header
  carries two `v1=` values (current and previous) — accept the event if **either**
  matches. Use a constant-time comparison.
- **Retries:** failed deliveries are retried with exponential back-off, **up to 8
  attempts**, then marked failed (and can be retried manually from the dashboard).

## Compliance

Factuarea is built for Spanish fiscal/legal requirements:

- **VeriFactu** (AEAT, RD 1007/2023) — certified, tamper-evident invoice records.
- **FacturaE 3.2.x** — official Spanish e-invoice XML.
- **Modelos 303 / 347** — VAT and annual operations tax reports.

Document **series** provide legal numbering per document type and are read-only
via the API to guarantee tax-number continuity.

## Resources

- Docs home: `https://docs.factuarea.com`
- OpenAPI spec (JSON): `https://docs.factuarea.com/api/openapi`
- LLM index: `https://docs.factuarea.com/llms.txt`
- Full docs (one Markdown file): `https://docs.factuarea.com/llms-full.txt`
- Beta access: `https://docs.factuarea.com/beta-access`
- Dashboard / API keys: `https://app.factuarea.com/settings/developers/api-keys`

## Minimal example

Create a client in the sandbox. Note the `fact_test_` key, Bearer auth, the
`Idempotency-Key`, and the minimal body (`name` is the only required field):

```bash
curl -X POST https://api.factuarea.com/v1/clients \
  -H "Authorization: Bearer fact_test_xxxxxxxxxxxxxxxxxxxxxxxx" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{
    "name": "Acme S.L.",
    "tax_id": "B12345678",
    "email": "facturacion@acme.es"
  }'
```

List invoices (cursor pagination), still in the sandbox:

```bash
curl "https://api.factuarea.com/v1/invoices?limit=20" \
  -H "Authorization: Bearer fact_test_xxxxxxxxxxxxxxxxxxxxxxxx"
```
