---
name: taskfolk-triage-and-plan
description: >-
  File, triage and schedule work in Taskfolk over the REST API or MCP — find the duplicate before
  you create one, create the issue, label and route it, put it in a sprint or a release, and read
  the report that says whether the sprint is going to land.
api: openapi/taskfolk-product-api-openapi.yml
method: generated
generated: '2026-08-20'
source: >-
  openapi/taskfolk-product-api-openapi.yml (verbatim method+path — the contract declares no
  operationIds) and https://taskfolk.ai/api/v1/reference
operations:
  - 'GET /v1/workspaces'
  - 'GET /v1/workspaces/{slug}/projects'
  - 'GET /v1/workspaces/{slug}/search'
  - 'GET /v1/workspaces/{slug}/projects/{key}/issues'
  - 'POST /v1/workspaces/{slug}/projects/{key}/issues'
  - 'POST /v1/workspaces/{slug}/projects/{key}/issues/bulk'
  - 'PATCH /v1/workspaces/{slug}/projects/{key}/issues/{issueKey}'
  - 'GET /v1/workspaces/{slug}/projects/{key}/labels'
  - 'GET /v1/workspaces/{slug}/projects/{key}/statuses'
  - 'POST /v1/workspaces/{slug}/projects/{key}/sprints'
  - 'PATCH /v1/workspaces/{slug}/projects/{key}/sprints/{id}'
  - 'GET /v1/workspaces/{slug}/projects/{key}/reports/{report}'
scopes: [workspaces:read, projects:read, search:read, issues:read, issues:write, labels:read]
---

# Triage and plan work in Taskfolk

Base URL `https://taskfolk.ai/api`. Auth `Authorization: Bearer $TASKFOLK_API_KEY`.

## 1. Find your ground

```
GET /v1/workspaces
GET /v1/workspaces/{slug}/projects
```

A project is addressed by its uppercase `key` (e.g. `WEB`), not a UUID. Issues are
`PROJECT-NUMBER` (e.g. `WEB-12`) and that key is stable — quote it in commits and chat.

## 2. Search before you create

```
GET /v1/workspaces/{slug}/search?q=rate%20limit&types=issue,doc
```

This is the step agents skip and it is the one that causes duplicate tickets. Search covers
issues *and* knowledge docs, so an existing RFC surfaces here too.

## 3. Read the board's vocabulary, do not invent it

```
GET /v1/workspaces/{slug}/projects/{key}/statuses
GET /v1/workspaces/{slug}/projects/{key}/labels
```

Statuses are **per project**, named and coloured by that team, each mapped to a category
(to-do / in progress / done / cancelled / failed). Labels must already exist — writing an unknown
label returns `400 validation` with a message like
`Label "foo" does not exist on this project.`

## 4. Create

```
POST /v1/workspaces/{slug}/projects/{key}/issues
Idempotency-Key: import-row-4821
```

Derive the key from the logical thing you are importing so a retry cannot double-create. For a
batch, use `POST …/issues/bulk` rather than a loop — one request, one rate-limit slot.

## 5. Route it

```
PATCH /v1/workspaces/{slug}/projects/{key}/issues/{issueKey}
```

`status_id`, `assignee`, `milestone_id`, `sprint_id`, `release_id` are all set here. An agent id
goes in `assignee` exactly like a person's — that is the whole point of the model.

To move status through the enforced path instead, use
`POST …/issues/{issueKey}/transition`.

## 6. Schedule

```
POST /v1/workspaces/{slug}/projects/{key}/sprints
PATCH /v1/workspaces/{slug}/projects/{key}/sprints/{id}
```

Attach issues by PATCHing the issue's `sprint_id`, not by posting to the sprint. Setting a
sprint's `status=active` **demotes any other active sprint** unless the project runs parallel
sprints — check before you flip it.

Deleting a sprint, milestone or release is a soft delete that **detaches** its issues rather than
deleting them.

## 7. Read whether it will land

```
GET /v1/workspaces/{slug}/projects/{key}/reports/{report}
```

`{report}` is one of velocity, burndown, burnup, cycle-time, throughput, cumulative-flow,
status-breakdown, workload. Cumulative flow and burnup include the `failed` bucket as of June
2026, so historical comparisons across that date are not like-for-like.

## Paging

Everything list-shaped is keyset-paged:

```
GET …?limit=50
→ { "data": [...], "pagination": { "next_cursor": "eyJ…" } }
GET …?limit=50&cursor=eyJ…
```

`next_cursor: null` is the last page. There is no page-number jumping, and open-issue counters
read `1000+` past a thousand rather than an exact number.

## Errors you will actually hit

| Code | Status | What to do |
|---|---|---|
| `validation` | 400 | Read `details` — it is field-level. |
| `forbidden` | 403 | Key lacks the scope, or a field-write allowlist blocked the field. |
| `not_found` | 404 | Also returned for another workspace's resources. Not transient — do not retry. |
| `rate_limited` | 429 | Sleep `Retry-After`, retry with the same `Idempotency-Key`. |
| `idempotency_violation` | 409 | Same key, different body. Use a new key or replay the identical request. |
