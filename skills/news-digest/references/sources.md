# News Digest Sources

Plain-text RSS feeds for collecting headlines. Prefer Google News RSS (most permissive,
no auth, broad coverage). Publisher feeds are a backup and some are CloudFront/WAF-blocked
to raw curl — try them only if Google News is unavailable.

## Google News RSS (PRIMARY)
Base URL:
`https://news.google.com/rss/search?q=<QUERY>&hl=<lang>&gl=<country>&ceid=<country>:<lang>`

Region presets:
- Mexico: `hl=es-419&gl=MX&ceid=MX:es-419`
- International/EN: `hl=en-US&gl=US&ceid=US:en`
- US national: `hl=en-US&gl=US&ceid=US:en` (query = "United States")

Query syntax:
- Plain topic: `q=Mexico`, `q=artificial+intelligence`, `q=world+news`.
- Time window: append ` when:7d` (a week), `when:1w`, `when:24h`. **`when:7d`/`when:1w`
  is the valid operator.** `when:1w` alone in some contexts returns wrong results; test
  the date range. Do NOT use `when:1w` as the only filter for "this week" — combine with
  a pubDate cutoff check.
- Multiple queries: run several and merge (dedupe by title).

Parsing: items are `<item>` blocks. `<title>` = headline (strip ` - <SOURCE>` suffix to
get the clean title). `<source>` = publisher. `<link>` = a `news.google.com/rss/articles/...`
redirect (NOT the original article). `<pubDate>` = publish time (parse with
`email.utils.parsedate_to_datetime`).

Fetch with a browser-like User-Agent:
`Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 Chrome/120.0 Safari/537.36`

## Publisher feeds (BACKUP — some 403 on raw curl)
- BBC world: `https://feeds.bbci.co.uk/news/world/rss.xml`
- BBC technology: `https://feeds.bbci.co.uk/news/technology/rss.xml`
- Nature: `https://www.nature.com/nature.rss` (RDF format; parse `<item rdf:about=...>`)
- Reuters/AP/El País: usually CloudFront-403 on raw curl — skip in favor of Google News.

## Exa (primary search, when available)
`mcp__exa__web_search_exa(query=..., numResults=...)` — natural-language query. If it
returns HTTP 402 (credits exhausted) or the MCP server is unreachable, switch to Google
News RSS above. Do not retry the dead Exa path in a loop; fall back immediately.
