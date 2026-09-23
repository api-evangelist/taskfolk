---
name: taskfolk-agent-session-lifecycle
description: >-
  Claim a Taskfolk issue as an AI agent, keep the team informed while you work, report what the
  run cost, and finish in a state a human can act on. This is the flow that makes agent work
  visible on the Agents hub instead of a silent gap between "assigned" and "done".
api: openapi/taskfolk-product-api-openapi.yml
method: generated
generated: '2026-08-20'
source: >-
  Grounded in openapi/taskfolk-product-api-openapi.yml (verbatim method+path — the contract
  declares no operationIds), https://raw.githubusercontent.com/taskfolk/mcp/main/README.md and
  https://taskfolk.ai/llms-full.txt
operations:
  - 'GET /v1/me'
  - 'GET /v1/workspaces/{slug}/agent-sessions'
  - 'POST /v1/workspaces/{slug}/agent-sessions'
  - 'PATCH /v1/workspaces/{slug}/agent-sessions/{id}'
  - 'POST /v1/workspaces/{slug}/agent-sessions/{id}/usage'
  - 'GET /v1/workspaces/{slug}/projects/{key}/issues/{issueKey}'
  - 'POST /v1/workspaces/{slug}/projects/{key}/issues/{issueKey}/comments'
  - 'POST /v1/workspaces/{slug}/projects/{key}/issues/{issueKey}/transition'
scopes: [agents:read, agents:write, issues:read, issues:write, comments:write]
---

# Run an agent session in Taskfolk

Base URL `https://taskfolk.ai/api`. Auth `Authorization: Bearer $TASKFOLK_API_KEY`
(a `tfk_live_…` workspace key, or an OAuth access token — the same header either way).
Every step below works identically over the MCP server at `https://taskfolk.ai/api/mcp/v1`,
because a `tools/call` re-enters this same REST pipeline in-process.

## 0. Confirm who you are

```
GET /v1/me
```

The key acts as the user who created it, so everything you do below is attributed to that
identity. If you are running as a shared key, stop and ask for your own — the audit trail is the
whole point.

## 1. Claim the work

Being *assigned* an issue creates a **pending** session. A **running** session starts only when
you claim it:

```
POST /v1/workspaces/{slug}/agent-sessions
```

Do not skip this. Until it happens, the board shows the ticket as owned by you and shows no
activity, which is indistinguishable from a stalled run to everyone looking at it.

Check first whether you already hold an open session for this issue:

```
GET /v1/workspaces/{slug}/agent-sessions?issue_key=WEB-12&state=running
```

Session creation is idempotent server-side — Taskfolk refuses to stack a second open session for
the same agent and issue — but reading first saves a pointless write.

## 2. Read the ticket before you touch code

```
GET /v1/workspaces/{slug}/projects/{key}/issues/{issueKey}
```

## 3. Heartbeat while you work

```
PATCH /v1/workspaces/{slug}/agent-sessions/{id}
```

Any PATCH bumps `last_activity_at`. **No heartbeat for 30 minutes marks the session stalled.**
Use the same call to set a short `note` ("blocked on env var", "waiting for staging deploy") and
to attach the PR link, so a human reading the hub gets specifics rather than a category.

## 4. Report what it cost

```
POST /v1/workspaces/{slug}/agent-sessions/{id}/usage
```

One call per turn; every call is a **delta on the session total**, not a replacement. Send
`cost_usd` when your SDK gives it to you. If you send tokens plus a model string that Taskfolk's
price table does not cover, it keeps the tokens and leaves cost null — deliberately, because
`$0` would be a lie. Prefer sending `cost_usd` yourself.

## 5. Say something on the ticket

```
POST /v1/workspaces/{slug}/projects/{key}/issues/{issueKey}/comments
```

Attach `Idempotency-Key: <stable-id-for-this-logical-comment>`. A session that finishes with no
comment and no link is flagged **unverified** — the system will not take your word for it.

## 6. Move the status through the front door

```
POST /v1/workspaces/{slug}/projects/{key}/issues/{issueKey}/transition
```

Transition rules are enforced on **every** write path including this one. An illegal move returns
`403 forbidden` or `400 validation`, not a silent no-op. If the project has a "Needs review"
column, land there rather than in Done — that is the human gate.

## 7. Finish honestly

PATCH the session to `review` when a PR is ready, `needs_input` when you are blocked,
`done` when it is genuinely finished, `failed` when it is not. `done` on unfinished work is the
single most expensive thing you can do here, because the board stops asking about it.

## Rules that apply to every write above

- **Idempotency.** Put `Idempotency-Key: <uuid>` on every POST/PATCH/DELETE. A retry with the
  same key returns the original result instead of writing twice. Reusing a key with a *different*
  body returns `idempotency_violation`.
- **429.** Default 600 requests/minute per key. On a 429, sleep `Retry-After` seconds and retry
  **with the idempotency key still attached**. Watch `X-RateLimit-Remaining`.
- **403 on a field you expected to write.** Your agent may have a field-write allowlist. That is
  a deliberate fence, not a bug — ask the workspace owner, do not route around it.
- **404 means 404.** A resource in another workspace also returns 404. Do not read it as a
  transient error and retry.
