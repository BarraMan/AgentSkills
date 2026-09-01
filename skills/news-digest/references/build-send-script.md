# Building & sending the news digest by script (verified 2026-08-31)

How the recurring weekly digest is actually produced end-to-end. Use this when the
`news-digest` pipeline needs to run as a one-shot script (cron or on-demand) rather
than inline in the agent turn.

## Workflow — elaborate once, deliver double
BarraMan's standing requirement: the digest goes to his email AND a copy. **Elaborate
the headlines ONCE and send a single message** with `To: <primary>` and
`Cc: <copy>` — never build the digest twice and never fire two cron jobs (he called
double-elaboration "sumamente ineficiente"). Put the second recipient in the Cc header
of the same message.

## Collection via Exa direct HTTP (reliable, bypasses the MCP bridge)
The Exa MCP bridge can misfire ("tool_call requires a 'name' argument") while the
service is healthy. Call the REST endpoint directly with the same `EXA_API_KEY`
(from `config.yaml` `mcp_servers.exa.env.EXA_API_KEY`):

```python
import urllib.request, json
body = {"query": q, "numResults": 16, "contents": {"text": False, "highlights": True}}
req = urllib.request.Request("https://api.exa.ai/search",
    data=json.dumps(body).encode(),
    headers={"X-Api-Key": KEY, "Content-Type": "application/json", "Accept": "application/json"})
results = json.loads(urllib.request.urlopen(req, timeout=60).read().decode()).get("results", [])
# each: {title, url, publishedDate, author, highlights:[...]}
```
Filter by `publishedDate` within the week window; dedupe by title AND by topic-cluster
(see `references/curate.md`); drop junk. A fresh HTTP 402 is a credit problem, not the
bridge — report it honestly, never fabricate.

## Carátula = og:image, normalized through PIL (mandatory)
For each headline URL: GET the article, regex `og:image` (fallback `twitter:image`),
**then normalize through PIL before attaching**:
```python
im = Image.open(io.BytesIO(raw)); im = im.convert("RGB"); im.save(dest, "JPEG", quality=88)
```
Attaching raw fetched bytes is the #1 cause of a broken send (see below). If no
`og:image`, generate a cover (solid category color + title) — never a blank row.

## Build & send — the "InvalidContentType" failure and its fix
Build a `multipart/related` message: `multipart/alternative` (plain + html) plus each
carátula as an image part with a `Content-ID`, referenced from the HTML as
`src="cid:carN"`.

**Pitfall (verified):** `himalaya message send` returns
`rc=1 ... received corrupt message of type InvalidContentType` when a carátula part is
not a valid JPEG (a raw fetch that returned HTML/0 bytes, or an image PIL's `verify()`
accepts but isn't real). **Fix = the PIL normalization above.** After re-sending,
re-check the sent folder: a failed attempt may have left a duplicate copy in Sent.

## Send + verify
```bash
himalaya message send --save sent -- <file.eml>      # rc=0 = delivered + saved
himalaya envelope list --mailbox sent                # confirm subject + size, no dup
```
`message send` success alone is NOT proof — always confirm in the sent folder and check
for duplicates.

## Cron model (BarraMan 2026-08-31)
One recurring job (weekly, Mon 07:00) elaborates once and delivers to
`primary@example.com` with a Cc copy. Do not maintain a second "copy" job. If a run
fails (e.g. a model timeout), it must self-report status to the user's channel —
do not silently no-op.

## Timeout on long non-streaming turns (2026-08-31)
A heavy digest turn (many searches + image fetches + two big emails) can exceed the
non-streaming stale timeout and fail with "Non-streaming API call timed out after
180s". The 180s is the reasoning-model floor, NOT the terminal timeout.
The override that wins is `providers.<id>.stale_timeout_seconds` (and
`request_timeout_seconds`) — set via `hermes config set --force
providers.<id>.stale_timeout_seconds 1500` (not by hand-editing config.yaml). The
resolver reads `config["providers"]` (a DICT), NOT `custom_providers` (a list) — so the
override must live under `providers.<id>`, matching `model.provider` (e.g. `custom`).
Verify with the resolver that it now returns the new value for the model.