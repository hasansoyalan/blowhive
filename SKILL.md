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
   - `{"status": "approved", "api_key": "bh_live_…", "mcp_url": ..., "api_base": ..., "workspace": {name, website_url, prelaunch, knowledge_pct}}` → save the key for this session and proceed. The key is delivered exactly once — store it immediately, never echo it in full. Then follow "After connecting" below — the `workspace` object tells you which branch applies.
   - `"denied"` → the user declined; stop.
   - `"expired"` → start over from step 2.

If the user already has a `bh_live_` key, accept it directly and skip the flow.

With the key, prefer MCP when the client supports it:

```bash
claude mcp add --transport http blowhive <BASE_URL>/api/mcp \
  --header "Authorization: Bearer bh_live_…"
```

Otherwise call the REST API directly with `Authorization: Bearer bh_live_…`.

## After connecting — what to do next

Branch on the `workspace` object from the poll response (or `read_workspace`):

- **`prelaunch: true`** → the user told us their site isn't published. **Accept that and move forward — ask NO questions about the business.** Say you're connected and offer the thing that helps them right now: **wire their project's blog to blowhive** (next section) so published articles flow in automatically once content exists. Do not run onboarding.
- **User announces the site is live / gives a domain** (now or in any later session) → run `onboard_workspace` / `POST /v1/onboard` with `{website_url}`. This crawls the site, builds the Business Brain and a scored title backlog (3–5 min; poll `check_job` and narrate). They can also do this themselves in the blowhive app under Settings.
- **`knowledge_pct` ≥ 60** → the brain exists; go straight to the content workflow below.

## Blog integration (do this for pre-launch projects)

The user is building their site right now — add a blowhive-powered blog to their codebase so it's ready before launch. Adapt to their framework; the pattern is: **server-side fetch of published articles, key kept in env, revalidate periodically**.

1. Add env vars (never commit the key, never expose it client-side):

```bash
BLOWHIVE_API_KEY=bh_live_…
BLOWHIVE_API_BASE=<api_base from the connect response>
```

2. A tiny server-side client:

```ts
// lib/blowhive.ts
const BASE = process.env.BLOWHIVE_API_BASE!;
async function bh<T>(path: string): Promise<T> {
  const res = await fetch(`${BASE}${path}`, {
    headers: { authorization: `Bearer ${process.env.BLOWHIVE_API_KEY}` },
    next: { revalidate: 3600 }, // rebuild content hourly (Next.js; adapt per framework)
  });
  if (!res.ok) throw new Error(`blowhive ${res.status}`);
  return res.json();
}
export const listPublished = () =>
  bh<{ data: Array<{ id: string; slug: string; title: string; published_at: string }> }>(
    "/articles?status=published");
export const getArticle = (id: string) =>
  bh<{ title: string; html: string; meta_title: string; meta_description: string; schema_markup: unknown }>(
    `/articles/${id}?format=html`);
// Building a custom layout (own FAQ accordion, keyword tags, per-section
// components)? Use ?format=structured instead: { seo: {meta_keywords…},
// intro, sections: [{heading, html}], faq: [{question, answer_html}], cta }.
```

3. Blog pages: an index over `listPublished()` and a detail page that maps slug → id from the list, renders `html`, and injects `meta_title` / `meta_description` / `schema_markup` (JSON-LD) into the head. Static generation with revalidation is ideal — content syncs at build/revalidate time with no client-side key exposure.

4. Empty state: the list is empty until the user launches + onboards — render a tasteful "no posts yet". The moment content publishes in blowhive, their blog fills on the next revalidation.

## The content workflow (once the brain exists)

1. **Ground yourself**: `read_workspace` — identity, voice, topic areas. Every suggestion must fit this brain.
2. **Pick or add a title**: `list_titles` shows the scored backlog (prefer high `business_value` + `seo_score`); `add_title` for new ideas — specific and honest, no clickbait. Backlog dry? `discover_titles` researches the live web for 8–12 fresh scored titles.
3. **Start the article**: `create_article` with `title_id` (or a new `title`). Async: outline → live web source research → cited draft → quality checks (3–6 min).
4. **Poll** `check_job` every 20–30s and narrate progress ("researching sources", "writing in your brand voice"…).
5. **Report the draft**: `read_article` — title, word count, SEO/readability scores, cited source count. It waits in the user's review queue.
6. **Approval & publishing**: `approve_article` runs the 10-check suite. `publish_article` (optionally `{at}` to schedule) — connect keys carry the scope, but **always get the user's explicit go-ahead in conversation first**; show the draft summary, then ask. Published articles appear on their integrated blog automatically.

## Hard rules

- **Never fabricate content, statistics or sources.** blowhive's writer cites stored sources; your role is orchestration and reporting.
- **Never publish without the user's explicit go-ahead in this conversation**, even when the key allows it. Show the draft summary, then ask.
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
| `GET /v1/articles/{id}?format=` | Article in `md`, `html`, `structured` (SEO meta + keywords, intro, sections, separated FAQ, CTA) or `full` (default: md+html) |
| `POST /v1/articles/{id}/approve` | Run checks + approve |
| `POST /v1/articles/{id}/publish` | Publish now or `{at: ISO}` (publish scope) |
| `GET /v1/jobs/{id}` | Async job status + progress steps |
| `GET /v1/usage` | Plan, article quota, rate limit |
