---
name: n8n
description: >
  Build, edit, debug, and reason about n8n workflows (self-hosted automation).
  Use when the user mentions n8n, workflow automation, a workflow .json export,
  nodes (HTTP Request, Webhook, Schedule Trigger, Code, IF, Switch, Set, Merge),
  n8n expressions ({{ }}, $json, $node, $items), credentials, or publishing
  content to social platforms (TikTok, Meta/Instagram/Facebook, Pinterest) via
  an n8n pipeline. Also use when importing/exporting workflow JSON into a repo.
---

# n8n

n8n is a source-available, node-based workflow automation tool. Workflows are
directed graphs of **nodes** connected by **connections**; data flows between
nodes as an array of **items**, each item being `{ "json": {...}, "binary": {...} }`.

This project (`visio-optical-n8n`) uses a **self-hosted** n8n instance as the
automation backend for "Visio Optical Auto": it publishes the operator's own
marketing videos to the operator's own social accounts (TikTok, Meta, Pinterest)
and tracks status in a Google Sheet. The GitHub repo itself only hosts the
public legal/verification pages (privacy/terms/TikTok verification). Actual
workflows live on the self-hosted server — when working with them, get the
exported JSON from the user or the server.

## Workflow JSON anatomy

An exported workflow is a single JSON object:

```json
{
  "name": "Publish video to TikTok",
  "nodes": [
    {
      "parameters": { ... },
      "id": "uuid",
      "name": "Schedule Trigger",
      "type": "n8n-nodes-base.scheduleTrigger",
      "typeVersion": 1.2,
      "position": [240, 300],
      "credentials": { ... }
    }
  ],
  "connections": {
    "Schedule Trigger": {
      "main": [[ { "node": "HTTP Request", "type": "main", "index": 0 } ]]
    }
  },
  "settings": { "executionOrder": "v1" },
  "pinData": {}
}
```

Key rules when hand-editing JSON:
- **`connections` are keyed by the source node's `name`**, not its id. Renaming a
  node means updating every connection key/target that references the old name.
- `main` is an array of output slots; each slot is an array of target links. IF /
  Switch nodes use multiple output slots (slot 0 = true/branch 0, etc.).
- `position` is `[x, y]` canvas coords — cosmetic, but keep them sane so the
  imported graph is readable.
- Keep `typeVersion` consistent with the n8n version on the server; importing a
  newer typeVersion into an older instance can break the node.
- `credentials` blocks reference credentials *by id/name*; ids are instance-
  specific. Never paste secret values into workflow JSON — credentials are stored
  separately and encrypted by n8n. Scrub any leaked tokens before committing.

## Core nodes

- **Triggers**: `scheduleTrigger` (cron/interval), `webhook` (HTTP in),
  `manualTrigger`, `executeWorkflowTrigger` (sub-workflows), `errorTrigger`
  (runs on workflow failure — wire this to alerting).
- **HTTP Request** (`n8n-nodes-base.httpRequest`): the workhorse for any REST
  API. Supports auth via "Predefined Credential Type" or generic header/OAuth2.
  Set `Response → Include Response Headers and Status` when you need to read
  status codes; enable pagination for cursor APIs.
- **Code** (`n8n-nodes-base.code`): run JS (or Python) over items. "Run Once for
  All Items" returns an array of `{json}`; "Run Once for Each Item" returns one.
- **Set / Edit Fields**: shape output without code.
- **IF / Switch**: branch on conditions. **Filter**: drop non-matching items.
- **Merge**: combine branches (append, by key, by position, SQL-like).
- **Split In Batches / Loop Over Items**: throttle/iterate (e.g. rate-limited APIs).
- **NoOp**: placeholder / join point.
- App nodes: Google Sheets, Gmail, Slack, etc. Prefer a dedicated app node over
  raw HTTP Request when one exists (handles auth + pagination for you).

## Expressions

Wrap expressions in `={{ ... }}`. Common references:
- `$json.fieldName` — current item's json on the *input* of this node.
- `$node["Node Name"].json.field` / `$('Node Name').item.json.field` — pull from
  another node's output.
- `$items("Node Name")` — all items from a node.
- `$now`, `$today` (Luxon DateTime), `$jmespath(...)`, `$if(cond, a, b)`.
- `$binary` for binary data; `$env` for env vars (if `N8N_BLOCK_ENV_ACCESS_IN_NODE`
  is not set).
- `$runIndex`, `$itemIndex` inside loops.

## Credentials & secrets

- Credentials are created in the UI / API and stored encrypted with
  `N8N_ENCRYPTION_KEY`. Back up that key — losing it makes all credentials
  unrecoverable.
- In workflow JSON, credentials appear only as references (`{ "id": "...",
  "name": "..." }`). **Never commit real tokens.** If a token appears inline in a
  HTTP node header, move it to a credential or an env var before committing.
- OAuth2 platforms (Meta, TikTok, Pinterest) store the access/refresh tokens in
  the credential; n8n refreshes them. Long-lived tokens (e.g. Meta page tokens)
  may need periodic manual rotation.

## Social publishing patterns (this project)

These are the API shapes the platforms require; in n8n each step is typically an
HTTP Request node, chained, with a Google Sheets node logging status.

- **Meta / Instagram (Graph API)** — two-step for video/Reels:
  1. `POST /{ig-user-id}/media` with `media_type=REELS`, `video_url`, `caption`
     → returns a **container id**.
  2. Poll `GET /{container-id}?fields=status_code` until `FINISHED`.
  3. `POST /{ig-user-id}/media_publish` with `creation_id=container-id`.
  Use a **Wait** or **Loop + IF** to poll; don't publish before `FINISHED`.
- **TikTok Content Posting API** — init upload, then either `PULL_FROM_URL`
  (TikTok fetches your hosted video) or chunked `FILE_UPLOAD`; then poll the
  publish status endpoint. Requires the verification file (the `tiktok…txt` in
  this repo) and approved scopes.
- **Pinterest API v5** — `POST /pins` with a `media_source` (video upload requires
  registering a media upload, then referencing the `media_id`).

Robustness tips: add an **Error Trigger** workflow for alerting, use
**Split In Batches** to respect rate limits, write idempotency keys (e.g. a
content id) to the Google Sheet so re-runs don't double-post, and capture each
API response's status into the sheet for bookkeeping (matches the privacy policy's
"post metadata" description).

## Import / export & version control

- Export: workflow menu → **Download** (single workflow JSON), or via the API:
  `GET /api/v1/workflows/{id}` (needs `X-N8N-API-KEY`).
- Import: workflow menu → **Import from File/URL**, or `POST /api/v1/workflows`.
- For git: store one JSON file per workflow under a `workflows/` dir, pretty-
  printed (`jq .`), with credentials scrubbed. Consider n8n's native
  `--output` CLI export: `n8n export:workflow --all --separate --output=workflows/`.
- `n8n import:workflow --separate --input=workflows/` re-imports them.

## Debugging

- Use **pinned data** on a trigger to develop downstream nodes without re-hitting
  live APIs.
- "Execute Node" runs a single node with current input; check the node's
  Input/Output tabs to see item shape.
- Common failures: connection keyed by stale node name after a rename; expression
  referencing a node that didn't run on that branch; HTTP node not surfacing a
  non-2xx body (enable "Never Error" + inspect, or "Full Response"); typeVersion
  mismatch after import.
- Check `Executions` list for error stack traces; set
  `Settings → Save failed executions` so failures are inspectable.

## Self-hosting notes

- Run via Docker (`n8nio/n8n`) or npm. Key env vars: `N8N_ENCRYPTION_KEY`,
  `N8N_HOST`/`WEBHOOK_URL` (must be the public URL for webhooks/OAuth callbacks),
  `DB_TYPE=postgresdb` for production (SQLite is default but not ideal at scale),
  `N8N_PROTOCOL=https`, `GENERIC_TIMEZONE`.
- OAuth callbacks from Meta/TikTok/Pinterest must point at the public
  `WEBHOOK_URL`; mismatches are the #1 cause of "redirect_uri" auth failures.

## When asked to build or modify a workflow

1. Ask for (or read) the current exported JSON if editing an existing one.
2. Make the change in JSON, preserving ids/typeVersions and fixing all
   `connections` references when renaming nodes.
3. Scrub secrets; keep credential references as id/name only.
4. Explain how to re-import (UI import or `import:workflow`).
5. Pretty-print the JSON when committing to the repo.
