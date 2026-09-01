# 3-row "formato de la casa" — the canonical news-digest output shape

BarraMan corrected this THREE times this session. A news-summary digest ("envíame por
correo el resumen") is **NOT a plain list** — each item is a **table of three rows**, in
this exact order. This is the "formato de la casa" published in
`BarraMan/AgentSkills/skills/news-digest`.

## The three rows (per item)
1. **Row 1 — Titular:** a category + number badge (e.g. "Nacional 01", "Ciencia 07")
   plus the headline.
2. **Row 2 — Carátula:** a **photograph, hyperlinked to the original article**
   (`<a href="URL"><img src="cid:newsN" ...></a>`). The carátula MUST link to the
   article. If the source has no `og:image`, **generate** a cover (solid category color
   + title via PIL) rather than leaving a blank row — the user expects a carátula per item.
3. **Row 3 — Resumen:** a short summary, then a **`Leer la noticia completa →`**
   hyperlink to the article, then `Fuente: <publisher> · <date>`.

## Non-negotiable rules (from news-digest)
- **HTML body, never Markdown** — Markdown renders as a plain text wall.
- **Accents are non-negotiable** — `ñ`, `á é í ó ú` must be correct (UTF-8); never
  pre-strip to ASCII. Verify by counting accent chars in the decoded HTML.
- **Summary (and title) in Mexican Spanish — mandatory translation:** the summary
  (row 3) and the title (row 1) are always written in Mexican Spanish (spelling and
  tone of MX), regardless of the source's original language. If the title/summary
  are in another language (e.g. English), translate them to Mexican Spanish — never
  leave them in the original language. The cover (row 2) still links to the original
  article (in its own language). Verify that no summary or title appears in
  English/another language in the decoded HTML.
- **Each item must carry a link to the full article** — BarraMan's real correction: a
  10-headline digest once omitted the article URL. Expect **2 links per item** (carátula
  image + "Leer la noticia completa →"). Verify by decoding the `text/html` part
  (it is base64 CTE'd — grep the RAW file and you'll see 0; decode first) and counting
  `href` — expect N items × 2.
- **Embed images as CID attachments, NOT data-URIs and NOT hotlinks.** Gmail does NOT
  render `data:image/...;base64,` inline (only a placeholder icon) and may block hotlinks.
  Attach each carátula as a MIME part with `Content-ID` and reference it as
  `src="cid:newsN"`. (The body must be `multipart/mixed`, image parts alongside
  `alternative`.)
- **One HTML body** unless a PDF attachment was requested; never attach a Markdown
  fallback file.

## Build & deliver
- Build the HTML with the `news-digest` skeleton (Helvetica font stack,
  table + inline CSS).
- Send: `himalaya message send --save "INBOX.Sent Items" < file.msg`.
- Verify: `himalaya envelope list -m "INBOX.Sent Items" --page-size 3` — confirm the
  subject + size. `message send` success alone is not sufficient.
- Disclose in the footer whether the search-provider fallback (Google News RSS) was used
  instead of Exa.
