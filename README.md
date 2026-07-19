# blowhive skill

Teach your AI assistant to drive [blowhive](https://blowhive.com) — the AI
content strategist that learns your business before writing a word.

## Install

```bash
npx skills add hasansoyalan/blowhive
```

Then get an API key from your blowhive workspace (**API & MCP** page) and ask
your assistant to "onboard my blowhive workspace" or "write this week's post".

## What it does

- Reads your Business Brain (identity, voice, topic areas)
- Picks high-value titles from your researched backlog
- Starts cited, brand-voice article generation and reports progress
- Requests your approval — publishing requires an explicit `publish`-scope key

MCP server: `https://blowhive.com/api/mcp` · REST: `https://blowhive.com/api/v1`
