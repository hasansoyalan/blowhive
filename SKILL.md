---
name: blowhive
description: Drive blowhive — the AI content strategist that learns a business before writing a word. Use when the user asks to write blog posts/articles with blowhive, manage their content backlog, check content pipeline status, or publish approved articles. Requires a blowhive API key.
---

# blowhive — AI content strategist

blowhive is not a writing tool: each workspace has a **Business Brain** (researched identity, competitors, voice, topic areas) and every article is written from real web sources with inline citations. Your job is to drive the pipeline — never to write article prose yourself.

## Setup (once)

1. Ask the user for their **blowhive API key** (`bh_live_…`) — created in the app under **API & MCP**. Never echo the full key back.
2. Ask which base URL to use if unknown: production `https://blowhive.com`, or a self-hosted/dev URL like `http://localhost:3000`.

Prefer the MCP server when the client supports it:

```bash
claude mcp add --transport http blowhive <BASE_URL>/api/mcp \
  --header "Authorization: Bearer bh_live_…"
```

Otherwise use the REST API directly (same key):

```bash
curl -H "Authorization: Bearer bh_live_…" <BASE_URL>/api/v1/workspace
```

## The workflow (follow this order)

1. **Ground yourself first**: `read_workspace` (MCP) or `GET /v1/workspace` — the business identity, voice and topic areas. Every suggestion you make must fit this brain.
2. **Pick or add a title**: `list_titles` / `GET /v1/titles` shows the scored idea backlog. Prefer high `business_value` + `seo_score`. Add new ideas with `add_title` / `POST /v1/titles` — titles must be specific and honest, no clickbait.
3. **Start the article**: `create_article` / `POST /v1/articles` with `title_id` (or a new `title`). This runs asynchronously: outline → live web source research → cited draft → quality checks. Typical duration 3–6 minutes.
4. **Poll**: `check_job` / `GET /v1/jobs/{id}` every 20–30 seconds. Report progress steps to the user ("researching sources", "writing in your brand voice"…). Stop polling on `succeeded` or `failed`.
5. **Report the draft**: fetch it with `read_article` / `GET /v1/articles/{id}` — share the title, word count, SEO/readability scores and how many cited sources it used. The draft waits in the user's review queue.
6. **Approval & publishing**:
   - `approve_article` runs the 10-check quality suite and approves only if blocking checks pass.
   - `publish_article` (optionally with `at` for scheduling) **only works if the key has the `publish` scope**. If it fails with `insufficient_scope`, tell the user to review and publish in the blowhive app — that is by design, not an error.

## Hard rules

- **Never fabricate content, statistics or sources.** blowhive's writer cites stored sources; your role is orchestration and reporting.
- **Nothing ships without the human** unless their key explicitly carries the `publish` scope. Default to requesting their review.
- **Respect quota errors** (`quota_exceeded`): tell the user their plan's monthly article limit is reached — don't retry.
- Rate limits are per plan (`X-RateLimit-*` headers). On 429, wait and retry once.

## Quick REST reference

| Method & path | Purpose |
|---|---|
| `GET /v1/workspace` | Business Brain: identity, voice, topic areas |
| `GET /v1/titles` · `POST /v1/titles` | Idea backlog · add `{text, pillar_id?}` |
| `GET /v1/articles?status=` | Pipeline list (`in_review`, `published`…) |
| `POST /v1/articles` | Start writing `{title_id}` or `{title}` → 202 `{job_id}` |
| `GET /v1/articles/{id}` | Full article: markdown, html, meta, sources |
| `POST /v1/articles/{id}/approve` | Run checks + approve |
| `POST /v1/articles/{id}/publish` | Publish now or `{at: ISO}` (publish scope) |
| `GET /v1/jobs/{id}` | Async job status + progress steps |
| `GET /v1/usage` | Plan, article quota, rate limit |
