---
name: taskfolk-product
description: Connect to Taskfolk — the project-management tool your AI agent uses to plan and run real projects. You are not authenticated yet; this doc explains how to sign up, create a workspace, mint an API key, and connect Claude Code / Cursor / MCP. Once a key is set, re-fetch this skill to get the full workspace-customised guide.
allowed-tools: Bash(curl:*), Read, Write
---

# Taskfolk — get connected

**Taskfolk is your AI agent's project-management tool.** Workspaces hold
projects; projects hold issues (epics, stories, tasks, bugs, subtasks) on a
kanban board with a backlog, comments, time tracking, custom fields, and docs.
You can drive all of it over a REST API or MCP.

You reached this skill **without an API key or session**, so here is how to
get one. Once you have a key, re-fetch this same URL with
`Authorization: Bearer <your-key>` and you will get the full guide,
customised to your workspace (its projects, labels, members, and fields).

## 1. Sign up

Go to **https://taskfolk.ai** and sign in. Auth is magic-link only — no passwords.
Enter your email, click the link, and you are in.

## 2. Create a workspace

First-time users land in onboarding and create a workspace (the tenant
boundary — one organisation, one workspace). Give it a name and a slug, then
create your first project inside it (an uppercase `key` like `WEB` plus a
lowercase `slug`).

## 3. Mint an API key

In your workspace, open **Developer → Keys** and create a key. It is shown in
full **once** at creation — copy it then. Keys are prefixed `tfk_live_`.

Mint **one key per agent** for a clean audit trail: a key acts as the user who
created it, so everything that agent does is attributed to that identity.

## 4. Connect

Set the key in your environment:

```bash
export TASKFOLK_API_KEY=tfk_live_xxxxxxxxxxxxxxxxxxxxxxxx
```

**Claude Code / MCP (recommended)** — one line:

```bash
claude mcp add taskfolk-product https://taskfolk.ai/api/mcp/v1/ \
  --header "Authorization: Bearer $TASKFOLK_API_KEY"
```

**Cursor** — add to `~/.cursor/mcp.json`:

```json
{ "mcpServers": { "taskfolk-product": { "url": "https://taskfolk.ai/api/mcp/v1/", "headers": { "Authorization": "Bearer YOUR_API_KEY" } } } }
```

**Plain curl** — verify the key resolves:

```bash
curl -H "Authorization: Bearer $TASKFOLK_API_KEY" https://taskfolk.ai/api/v1/me
```

## 5. Get the full skill

Re-fetch this skill with your key and your workspace slug to get the
workspace-customised guide (projects, members, recipes):

```bash
curl -H "Authorization: Bearer $TASKFOLK_API_KEY" \
  "https://taskfolk.ai/api/skill/taskfolk-product.skill.md?workspace=YOUR_WORKSPACE_SLUG"
```

The machine-readable OpenAPI spec is public at `https://taskfolk.ai/api/v1/openapi.json`.
