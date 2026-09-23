---
name: taskfolk-safe-writes-and-restore
description: >-
  Delete, undo, and retry safely in Taskfolk. What is recoverable, for how long, and what is not
  recoverable at all — the check to run before an agent takes a destructive action on a board it
  does not own.
api: openapi/taskfolk-product-api-openapi.yml
method: generated
generated: '2026-08-20'
source: >-
  openapi/taskfolk-product-api-openapi.yml (verbatim method+path),
  conventions/taskfolk-conventions.yml, and the 30-day window stated verbatim in
  https://taskfolk.ai/llms-full.txt
operations:
  - 'DELETE /v1/workspaces/{slug}/projects/{key}/issues/{issueKey}'
  - 'POST /v1/workspaces/{slug}/projects/{key}/issues/{issueKey}/restore'
  - 'DELETE /v1/workspaces/{slug}/projects/{key}'
  - 'POST /v1/workspaces/{slug}/projects/{key}/restore'
  - 'GET /v1/workspaces/{slug}/docs/{id}/versions'
  - 'POST /v1/workspaces/{slug}/docs/{id}/versions/{versionId}/restore'
  - 'DELETE /v1/workspaces/{slug}/projects/{key}/statuses/{id}'
  - 'POST /v1/workspaces/{slug}/webhooks/{id}/rotate-secret'
  - 'DELETE /v1/workspaces/{slug}/api-keys/{id}'
scopes: [issues:write, projects:admin, docs:write, webhooks:write, api_keys:write]
---

# Safe writes and restores in Taskfolk

## The one number to know: 30 days

Deleting an issue or a project is a **soft delete**. The record is archived and stays restorable
for **30 days**, after which a storage-reclaim sweep frees its attachments and the recovery door
closes. Taskfolk states this in its own confirm dialog: *"Issues in this column will be
soft-deleted. They are recoverable for 30 days."*

That is why a delete here is a decision an agent may reasonably take, and why a key revocation is
not.

## Recoverable

| Action | Undo | Window |
|---|---|---|
| `DELETE …/issues/{issueKey}` | `POST …/issues/{issueKey}/restore` | 30 days |
| `DELETE …/projects/{key}` | `POST …/projects/{key}/restore` | 30 days |
| `PATCH …/docs/{id}` | `POST …/docs/{id}/versions/{versionId}/restore` | version snapshots on all plans; no stated retention limit |
| `DELETE …/sprints/{id}`, `…/milestones/{id}`, `…/releases/{id}` | issues are **detached, not deleted** | n/a — the issues survive |

To undo a doc edit, list the snapshots first:

```
GET /v1/workspaces/{slug}/docs/{id}/versions
POST /v1/workspaces/{slug}/docs/{id}/versions/{versionId}/restore
```

## Recoverable, but only the hard way

`DELETE /v1/workspaces/{slug}/projects/{key}/statuses/{id}` soft-deletes **every issue sitting in
that column**. There is no single "undo column delete". Recovery is one restore call per issue,
and you need to know which issues they were. **Move the issues out first, then delete the empty
column.** A guard refuses to delete the last remaining status ("A board needs at least one
status.").

## Not recoverable — do not take these on someone's behalf

- `DELETE /v1/workspaces/{slug}/api-keys/{id}` — revocation is final.
- `POST /v1/workspaces/{slug}/webhooks/{id}/rotate-secret` — the previous secret is gone. Every
  existing consumer breaks until it is re-keyed.
- `DELETE /v1/workspaces/{slug}/oauth-apps/{clientId}` — revokes the client for everyone using it.
- `POST /api/mpp/v1/credits/*` (agent commerce) — a completed purchase returns a signed
  `Payment-Receipt`. **No refund, void or cancel operation is published.** Treat money as final.

## There is no dry run

Taskfolk publishes no preview/dry-run parameter on any write. The controls that exist are
refusals at write time, not rehearsals:

- **Per-agent field-write allowlist** — which issue fields this agent may write, enforced at the
  single API write path so it holds over REST *and* MCP.
- **Transition rules** — an illegal status move is rejected everywhere.
- **Scope filtering** — a read-only key does not even see the write tools in MCP `tools/list`.

If you need to rehearse, read first (`GET`) and describe the plan in a comment before acting.

## Retrying without doubling up

Put `Idempotency-Key: <uuid>` on every POST/PUT/PATCH/DELETE. Derive it from the *logical*
operation — this CI run, this import row — so a retry of the same logical action reuses it. On a
429, sleep `Retry-After` seconds and retry with the key still attached. A key reused with a
different body returns `idempotency_violation`; that is the system stopping you, not a fault.

Two writes are naturally idempotent and safe to repeat without a key:

- `POST …/issues/{issueKey}/watchers` — self-subscribe. 201 first time, 200 thereafter.
- `POST …/projects/{key}/members/{userId}` — 201 for a new grant, 200 for a role update in place.
