---
name: taskfolk-webhook-consumer
description: >-
  Register a Taskfolk webhook, verify its signature correctly (the key derivation is NOT the
  usual one), and consume deliveries idempotently. Includes the event-name discrepancy between
  the provider's own two documents.
api: openapi/taskfolk-product-api-openapi.yml
method: generated
generated: '2026-08-20'
source: >-
  openapi/taskfolk-product-api-openapi.yml (verbatim method+path), asyncapi/taskfolk-webhooks.yml,
  https://taskfolk.ai/llms.txt and https://taskfolk.ai/llms-full.txt
operations:
  - 'GET /v1/workspaces/{slug}/webhooks'
  - 'POST /v1/workspaces/{slug}/webhooks'
  - 'PATCH /v1/workspaces/{slug}/webhooks/{id}'
  - 'DELETE /v1/workspaces/{slug}/webhooks/{id}'
  - 'POST /v1/workspaces/{slug}/webhooks/{id}/test'
  - 'GET /v1/workspaces/{slug}/webhooks/{id}/deliveries'
  - 'POST /v1/workspaces/{slug}/webhooks/{id}/rotate-secret'
scopes: [webhooks:read, webhooks:write]
---

# Consume Taskfolk webhooks

## 1. Register

```
POST /v1/workspaces/{slug}/webhooks
```

Subscribe to specific events or use the `*` wildcard. Registration is an owner-level action in
the Developer dashboard; over the API it needs `webhooks:write`.

Then fire a test delivery before you trust it:

```
POST /v1/workspaces/{slug}/webhooks/{id}/test
```

## 2. Subscribe to the right names

The v1 publisher emits **14** events:

`issue.created`, `issue.updated`, `issue.archived`, `issue.restored`,
`comment.created`, `comment.updated`, `comment.deleted`,
`doc.created`, `doc.updated`, `doc.archived`,
`member.added`, `member.removed`, `member.role_changed`, `workspace.updated`

⚠️ **`issue.deleted` is not one of them.** Taskfolk's own `llms.txt` lists a shorter six-event set
that includes `issue.deleted`; the longer list above comes from `llms-full.txt` and matches the
soft-delete model (a delete emits `issue.archived`, and a restore emits `issue.restored`). If you
subscribed to `issue.deleted` you will receive nothing. Subscribe to `issue.archived`.

## 3. Verify the signature — read this twice

Header: `X-Taskfolk-Signature: t=<unix>, v1=<hex>`

```
v1 = HMAC-SHA256( key = SHA256(your-signing-secret), message = "{t}.{raw_body}" )
```

**The HMAC key is the SHA-256 hash of your secret, not the raw secret.** Taskfolk stores only the
hash, so it signs with the hash. A verifier copied from a Stripe/GitHub-style implementation uses
the raw secret and will fail every delivery. This is the single most likely integration bug on
this surface.

Then: constant-time compare, and reject if `t` is stale.

## 4. Dedupe on the delivery id

The `id` in the envelope equals the `X-Taskfolk-Delivery` header. Deliveries are at-least-once —
key your consumer on that UUID so reprocessing is harmless.

## 5. Do not re-fetch the resource

`data` is the **full REST resource** for the affected entity — the same object a `GET` on it
would return. When `WEB-142` moves to Done, `data` is the complete issue. No second call.

## 6. When something goes missing

```
GET /v1/workspaces/{slug}/webhooks/{id}/deliveries
```

The delivery log is the first place to look, before you suspect your own endpoint.

## 7. Rotation is one-way

```
POST /v1/workspaces/{slug}/webhooks/{id}/rotate-secret
```

The old secret is gone and every consumer breaks until re-keyed. There is no un-rotate. Deploy
the new secret first if your consumer can accept two.
