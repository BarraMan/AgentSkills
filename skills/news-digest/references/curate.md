# Curation Rules

After collecting headlines, dedupe and curate. The #1 quality bug is a single major story
appearing 3–5× across outlets under different titles.

## Step 1 — drop pre-cutoff
Filter `pubDate >= week_start` (parse with `email.utils.parsedate_to_datetime`).

## Step 2 — dedupe by topic-cluster (NOT just exact title)
Map each title to a cluster by keyword, then keep one item per cluster
(strongest source / most recent first):

| Cluster | Keywords |
|---------|----------|
| telescope | roman, telescope |
| quantum | quantum |
| ai | ai, artificial intelligence, openai |
| robot | robot |
| chip | chip, semiconductor |
| crew | crew-13, crew 13 |
| ev | electric, ev |
| climate | climate |
| apple | apple |
| cyber | cyber |
| (event) | summit, forum |

A non-clustered item (unique topic) is kept as its own singleton.

## Step 3 — junk filter (drop these)
- Corporate/financial/PR: `half-year results`, `receives a buy`, `decline of`,
  `announced its`, `stock`, `shares`, `buy from`.
- Events/digests: `top headlines in`, `forum on`, `summit`, `industry news`,
  `daily releases`, `optimising`.
- Opinion/letters: `letters`, `as a relatively new`, `watch author`, `new ceo`.
- Sports: `football`, `soccer`, `tds in`, `team news`, `fightin`, `leatherwood`.
- Entertainment: `miss universo`, `corona`, `videoanálisis`, `agenda en`.
- Weather minor: `sismo de 4`.

## Step 4 — rank
Within a category, rank by a "serious topic" keyword score (invest, economía, fiscal,
seguridad, desaparición, migración, narcotráfico, reforma, salud, educativa, etc.) and
sort by that score desc, then by `pubDate` desc. Cap at N.

## Step 5 — quality over quantity (BarraMan preference)
If fewer than N clean items remain, **deliver the fewer clean items** and state the count
and reason. Do NOT pad with weak/irrelevant content to hit a number.

## Delivery note
Always disclose in the digest footer:
- Links are Google News aggregators, not original articles.
- Whether the search-provider fallback (Google News RSS) was used instead of Exa.
