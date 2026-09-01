# Exa Search (primary provider for news digests)

BarraMan's standing rule: **for all web searches, use Exa first.**
"Para búsquedas siempre debes usar Exa web search." Google News RSS / publisher
feeds are a FALLBACK only (when Exa is down) — never the default.

## Call shape
- `mcp__exa__web_search_exa(query, type:"news", startPublishedDate, endPublishedDate,
  numResults)` — returns title, URL, `published`, author, and full `Highlights` body.
- `mcp__exa__web_fetch_exa(url)` — read a page's full content as clean markdown.
- Run several category queries in parallel with `type:"news"` and a hard date window
  (e.g. `startPublishedDate=2026-08-24`, `endPublishedDate=2026-08-31`).

## 402 = credits exhausted — do NOT fabricate
- Exa returns HTTP **402** when credits are exhausted; the MCP server may then show the
  tool as "unavailable" / auto-retry after ~1 min. This is a plan/credit issue, not a
  connection fault — say so to the user.
- Do NOT fabricate headlines. Either (a) wait and retry Exa (credits may have propagated
  or be topped up), or (b) fall back to **Google News RSS** + publisher feeds (BBC,
  Reuters, Nature) via curl, and **disclose the fallback in the digest footer**.
- A 402 that persists means the plan has no credits — report it honestly; do not invent
  titles/dates to fill a number.

## Why Exa, not Google News RSS, by default
- Exa returns the real article URL, the true publish date, author, and the article body
  (Highlights) — enough to write an accurate 3-row item (titular / carátula / resumen)
  and to fetch `og:image` for the carátula. Google News RSS gives only a redirect link
  (`news.google.com/rss/articles/...`) and no body/image.

## Fallback chain (only if Exa fails)
1. Google News RSS (`references/sources.md`) — permissive, plain text.
2. Publisher feeds (BBC, Reuters, Nature) via curl — some outlets CloudFront-403.
3. Still nothing usable → deliver the fewer clean items, state the count, never pad.
