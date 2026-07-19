---
name: blowhive
description: Drive blowhive — the AI content strategist that learns a business before writing a word. Use when the user asks to write blog posts/articles with blowhive, manage their content backlog, check content pipeline status, or publish approved articles. Requires a blowhive API key.
---

# blowhive — AI content strategist

blowhive is not a writing tool: each workspace has a **Business Brain** (researched identity, competitors, voice, topic areas) and every article is written from real web sources with inline citations. Your job is to drive the pipeline — never to write article prose yourself.

## Setup (once) — the connect flow

No manual key copying. Connect the user's account like this:

1. Determine the base URL: production `https://blowhive.com`, or the user's self-hosted/dev URL (e.g. `http://localhost:3000`). Ask if unknown.
2. Request a connect code (no auth needed):

```bash
curl -s -X POST <BASE_URL>/api/connect/start \
  -H "content-type: application/json" \
  -d '{"client_name": "Claude"}'
# → { user_code, verification_url, poll_token, poll_url, interval_seconds, expires_in_seconds }
```

3. **Send the user to `verification_url`** and tell them: "Open this link, sign in to blowhive, and click **Approve access** (code `<user_code>`)." They pick which website this is for (or create a new one — including pre-launch projects with no site yet).
4. **Poll** `poll_url` every `interval_seconds` with `{"poll_token": "..."}` until:
   - `{"status": "approved", "api_key": "bh_live_…", "mcp_url": ..., "api_base": ..., "workspace": {name, website_url, prelaunch, knowledge_pct}}` → save the key for this session and proceed. The key is delivered exactly once — store it immediately, never echo it in full. Use `workspace` to decide the next step: `prelaunch: true` → ask the user to describe the project and onboard with `{description}`; `website_url` present or user gives one → onboard with `{website_url}`; `knowledge_pct` ≥ 60 → brain already exists, go straight to content.
   - `"denied"` → the user declined; stop.
   - `"expired"` → start over from step 2.

If the user already has a `bh_live_` key, accept it directly and skip the flow.

With the key, prefer MCP when the client supports it:

```bash
claude mcp add --transport http blowhive <BASE_URL>/api/mcp \
  --header "Authorization: Bearer bh_live_…"
```

Otherwise call the REST API directly with `Authorization: Bearer bh_live_…`.

## The workflow (follow this order)

1. **Ground yourself first**: `read_workspace` (MCP) or `GET /v1/workspace` — the business identity, voice and topic areas. Every suggestion you make must fit this brain.
   - **New workspace with a live site?** Ask for the website and run `onboard_workspace` / `POST /v1/onboard` with `{website_url, socials?}`. Crawls, researches identity/competitors/voice/topic areas, configures everything and generates a scored title backlog (3–5 min; poll `check_job` and narrate). Everything remains editable in the blowhive app.
   - **Pre-launch, no website yet?** Ask the user to describe the project in their own words (what it is, who it's for, what makes it different) and call onboard with `{description, project_name?}`. In ~1 min this builds a **provisional brain** (`knowledge_pct` 60) plus one starter title and a starter draft in their review queue. **When their site launches**, call onboard again with `{website_url}` — provisional brains upgrade to full research without `force`.
   - Backlog running dry later? `discover_titles` / `POST /v1/titles/discover` researches the live web and adds 8–12 fresh scored titles.
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
| `POST /v1/onboard` | Train the brain from `{website_url, socials?}` → 202 job |
| `POST /v1/titles/discover` | Research fresh titles into the backlog → 202 job |
| `GET /v1/titles` · `POST /v1/titles` | Idea backlog · add `{text, pillar_id?}` |
| `GET /v1/articles?status=` | Pipeline list (`in_review`, `published`…) |
| `POST /v1/articles` | Start writing `{title_id}` or `{title}` → 202 `{job_id}` |
| `GET /v1/articles/{id}` | Full article: markdown, html, meta, sources |
| `POST /v1/articles/{id}/approve` | Run checks + approve |
| `POST /v1/articles/{id}/publish` | Publish now or `{at: ISO}` (publish scope) |
| `GET /v1/jobs/{id}` | Async job status + progress steps |
| `GET /v1/usage` | Plan, article quota, rate limit |
