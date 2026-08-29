---
name: email-report-delivery
description: "Send a report by email as an HTML body, not a file."
version: 1.0.0
metadata:
  hermes:
    tags: [Email, HTML, Reports, Delivery]
---

# Email Report Delivery

Delivering a report, memo, or summary **as the email body in HTML** (not as an
attachment, not as Markdown). This is the class of task for "send me the report
by email" / "envíame por correo el resumen" — distinct from CLI mechanics (see
`himalaya`) and from inbox triage.

## When to use
- User says send a report/summary/invoice "by email" **without** saying "document".
- A formatted report must reach an inbox and look like a web page, not a text blob.

## Non-negotiable rules (these HAVE been corrected by the user)
1. **HTML body, never Markdown.** Markdown in an email renders as a plain text
   wall. Compose HTML. If a Markdown template is the source, convert it to HTML
   first.
2. **Accents are non-negotiable.** `ñ` and tildes (`á é í ó ú`) must be correct —
   missing tildes change meaning. The charset is UTF-8; accent loss is an
   **authoring** error, not a charset one. Write the accented text directly;
   never pre-strip to ASCII.
3. **News-summary ("formato de la casa") emails MUST include a link to the FULL
   news per item.** This was a real correction: a 10-headline digest sent the
   title + summary + source but omitted the article URL. Each item needs at
   least one "Leer la noticia completa →" hyperlink (an `<a href=...>`), and for
   redundancy the cover image (row 2) should ALSO link to the same URL
   (`<a href=URL><img src="cid:newsN"></a>`). Verify by decoding the HTML MIME
   part and counting `href` occurrences — expect N per item (2 with the
   image+body redundancy). Note the HTML body is base64 CTE'd, so grep the RAW
   file for the URL and you'll see 0 — decode the `text/html` part first
   (`part.get_payload(decode=True)`).
4. **Embed images as CID attachments, NOT data-URIs and NOT hotlinks.** Gmail and
   most webmail clients do **NOT** render `data:image/...;base64,` inline — it shows
   only a placeholder "image icon" (this exact symptom was a real correction). Remote
   hotlinks may be blocked by CSP/anti-hotlink too. The robust, self-contained method
   that Gmail **does** render: attach each image as a MIME part with a `Content-ID`,
   and reference it as `src="cid:newsN"` (see step 3). The image then travels inside
   the message — no host dependency — and renders.
4. **Keep it a single HTML body** unless the user asked for a PDF attachment.
   Don't attach a Markdown file as a "fallback."

## Standard workflow
1. **Build the HTML body** (below). Table + inline CSS layout; Helvetica font stack
   (`Helvetica, Carlito, "Segoe UI", Roboto, Arial, sans-serif`).
2. **Embed images as CID attachments** (if any) — see step 3.
3. **Wrap in a MIME message** with the image parts attached — see step 4.
4. **Send** via `himalaya message send < message.txt` (see `references/himalaya-send.md`)
   or any SMTP path. Verify it lands in Sent.

### HTML body skeleton (email-safe)
```html
<!doctype html><html lang="es"><head><meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1"></head>
<body style="margin:0;background:#eef1f5;font-family:Helvetica,Carlito,
'Segoe UI',Roboto,Arial,sans-serif;font-size:14px;color:#222;line-height:1.5;">
<table role="presentation" width="100%" cellpadding="0" cellspacing="0"
 style="background:#eef1f5;"><tr><td align="center" style="padding:24px 12px;">
 <table role="presentation" width="640" cellpadding="0" cellspacing="0"
  style="width:640px;max-width:640px;background:#fff;border-radius:6px;">
  <!-- header, sections as <tr><td> rows with inline style -->
 </table></td></tr></table></body></html>
```

### Embed an image as a CID attachment (Gmail does NOT render data-URIs) — Python
```python
import urllib.request, re, ssl, io
from PIL import Image
ctx=ssl.create_default_context(); ctx.check_hostname=False; ctx.verify_mode=ssl.CERT_NONE
def cover_bytes(page_url):
    h=urllib.request.urlopen(urllib.request.Request(page_url,headers={"User-Agent":"Mozilla/5.0"}),
                             timeout=25,context=ctx).read().decode("utf-8","ignore")
    m=re.search(r'<meta[^>]+property=["\']og:image["\'][^>]*content=["\']([^"\']+)["\']',h)
    u=m.group(1).replace("&amp;","&") if m else None
    if not u: return None                      # generate a cover if none — see below
    im=Image.open(io.BytesIO(urllib.request.urlopen(u,timeout=30,context=ctx).read())).convert("RGB")
    if im.width>600: im=im.resize((600,int(im.height*600/im.width)),Image.LANCZOS)
    b=io.BytesIO(); im.save(b,"JPEG",quality=80); return b.getvalue()  # raw bytes -> CID attach
```
If a news item has no `og:image`, **generate** a cover (solid color + title text via
PIL) rather than leaving a blank row — the user expects a carátula per item.

### MIME message — use MIMEMultipart, NOT EmailMessage.add_alternative
Python 3.12's `EmailMessage.add_alternative(html, maintype=...)` raises
`TypeError: set_text_content() got an unexpected keyword argument 'maintype'`.
Build the message with `MIMEMultipart("alternative")` instead. **If the body has
images, use `multipart/mixed`** so the image parts sit alongside the text/HTML:

```python
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText
from email.mime.image import MIMEImage

outer = MIMEMultipart("mixed")          # outer holds: [ alternative, img1, img2, ... ]
alt = MIMEMultipart("alternative")
alt.attach(MIMEText("plain fallback text", "plain", "utf-8"))
alt.attach(MIMEText(html_body, "html", "utf-8"))   # html_body uses src="cid:newsN"
outer.attach(alt)

# Attach each cover with a matching Content-ID; reference it in the HTML as
# <img src="cid:news1">. MIMEImage handles the base64 CTE automatically.
for i, raw in enumerate(cover_raw_list, start=1):
    part = MIMEImage(raw)
    part.add_header("Content-ID", "<news%d>" % i)
    outer.attach(part)

outer["From"] = "BarraBOT <from@host>"; outer["To"] = "to@host"
outer["Subject"] = "Subject"
open("message.txt","w").write(str(outer))
# send:  himalaya message send --save "INBOX.Sent Items" < message.txt
```

**Why CID and not data-URI:** Gmail and most webmail clients do **not** render
`data:image/...;base64,` inline — the recipient sees only a placeholder "image
icon" (this was a real correction). CID attachments render in Gmail/Outlook and
travel inside the message, so they don't depend on a host that might block
hotlinks.

## Pitfalls
- **`%` in a `%`-format string** double-renders. Use plain string concatenation or
  `str.format` for HTML with `width="100%"`; don't use `%`-formatting.
- **Verify after send** — list the Sent folder and confirm the subject + size.
  `himalaya message send` success is necessary but not sufficient (SMTP may reject
  on AUTH — see `references/himalaya-send.md`).
- **Accents** — grep the generated HTML for `ñ`, `á`, `é`, `ó`, `ú` counts to
  confirm they survived before sending.
- **Don't ship a Markdown file** as a fallback to an HTML-only request.

## See also
- `references/himalaya-send.md` — send via himalaya v2.1.0 + config/AUTH gotchas.
- `himalaya` skill — full CLI reference (bundled; may be outdated for v2.1.0).
