---
name: news-digest
description: Construir un digest de noticias y enviarlo por correo como cuerpo HTML con caratulas.
version: 2.0.0
metadata:
  hermes:
    tags: [News, Email, HTML, Digest, Weekly, Cron]
---

# News Digest (Stack Pro)

Construir un digest de noticias curado, deduplicado y enviarlo por correo como
**cuerpo HTML** (no como adjunto ni Markdown), con caratulas embebidas. Stack completo
que fusiona: (1) recoleccion, (2) curacion/dedupe, (3) formato 3-renglones, (4) envio
himalaya, (5) verificacion. Es un trabajo recurrente, suele ser un cron job.

## Flujo completo

1. **Fijar la ventana temporal**: confirmar la semana contra el reloj del sistema; usar una fecha de corte dura (ej. 2026-08-24 para una semana Lun-Dom) y filtrar por pubDate mayor o igual que el corte.
2. **Colectar titulares**: buscar en Exa (fuente primaria) y correr varias queries por categoria en paralelo.
3. **Filtrar y dedupar**: descartar items antes del corte; dedupar por titulo; dedupar por cluster tematico (la misma historia aparece en varios medios); filtrar basura.
4. **Curar calidad sobre cantidad**: preferir pocos titulares solidos sobre un N inflado.
5. **Construir y entregar**: renderizar el digest HTML (formato 3-renglones) y enviar; verificar en la carpeta sent.

## 1. Coleccion (Exa primero, Google News RSS como fallback)

Regla de BarraMan: para toda busqueda web usar Exa primero. Para los digests,
Exa es la fuente primaria y Google News RSS es SOLO un fallback (cuando Exa cae).

- **Exa (primaria)**: mcp__exa__web_search_exa(query, type:news, startPublishedDate,
  endPublishedDate, numResults). Devuelve titulo, URL real, fecha, autor y el cuerpo
  (Highlights) — suficiente para un item 3-renglones y para obtener og:image.
  Alternativa: REST directo https://api.exa.ai/search con X-Api-Key (bypass del MCP).
- **402 = credito agotado** (no es un fallo de conexion): NO fabricar titulares.
  O (a) esperar y reintentar Exa, o (b) fallback a Google News RSS + feeds de
  publishers (BBC, Reuters, Nature) via curl, y **revelar el fallback en el footer**.
- **Google News RSS (fallback)**: news.google.com/rss/search?q=QUERY&hl=&gl=&ceid=.
  Operador de tiempo VALIDO = when:7d (o when:1w). El link es un redirect
  news.google.com/rss/articles/... (NO el articulo original). Fetch con User-Agent
  tipo navegador. Algunos outlets (Reuters, AP, El Pais) dan CloudFront-403 a curl.

## 2. Curacion y dedupe (el bug de calidad #1)

Un solo tema mayor aparece 3-5 veces en distintos medios con titulos distintos.

**Step 1 - descartar pre-corte**: filtrar pubDate mayor o igual que week_start
(email.utils.parsedate_to_datetime).

**Step 2 - dedupe por cluster tematico (NO solo titulo exacto)**: mapear cada titulo
a un cluster por keyword y conservar uno por cluster (fuente mas fuerte / mas reciente):
- telescope: roman, telescope | quantum: quantum | ai: ai, artificial intelligence, openai
- robot: robot | chip: chip, semiconductor | crew: crew-13 | ev: electric, ev
- climate: climate | apple: apple | cyber: cyber | event: summit, forum

**Step 3 - junk filter (descartar)**:
- Corporativo/financiero/PR: half-year results, receives a buy, decline of, announced its, stock, shares, buy from.
- Eventos/digests: top headlines in, forum on, summit, industry news, daily releases, optimising.
- Opinion/letters: letters, as a relatively new, watch author, new ceo.
- Deportes: football, soccer, tds in, team news, fightin, leatherwood.
- Entretenimiento: miss universo, corona, videoanalisis, agenda en.
- Clima menor: sismo de 4.

**Step 4 - rank**: dentro de categoria, por puntaje de tema serio (invest, economia,
fiscal, seguridad, desaparicion, migracion, narcotrafico, reforma, salud, educativa) desc,
uego pubDate desc. Cap en N.

**Step 5 - calidad sobre cantidad (preferencia de BarraMan)**: si quedan menos de N
items limpios, **entregar los pocos limpios** y declarar el conteo y el por que. No
rellar con contenido debil para llegar a un numero.

## 3. Formato de la casa (3 renglones por noticia) - CORREGIDO 3 VECES

Un digest de resumen NO es una lista simple. CADA item es una TABLA de 3 renglones
en este orden exacto (publicado en BarraMan/AgentSkills):

1. **Renglon 1 - Titular**: badge de categoria+numero (ej. Nacional 01, Ciencia 07)
   + el titular.
2. **Renglon 2 - Caratula**: una FOTOGRAFIA hipervinculada al articulo original
   (<a href=URL><img src=cid:newsN ...></a>). La caratula DEBE enlazar al articulo.
   Si la fuente no tiene og:image, **generar** una portada (color de categoria +
   texto con PIL) — nunca renglon en blanco.
3. **Renglon 3 - Resumen**: un resumen corto, luego un hyperlink
   **Leer la noticia completa ->** al articulo, luego Fuente: publisher - date.

**Reglas no negociables (corregidas por el usuario):**
- **Cuerpo HTML, NUNCA Markdown** (Markdown se ve como un muro de texto plano).
- **Acentos no negociables**: n, tildes (a e i o u) correctos (UTF-8). Verificar
  contando acentos en el HTML decodificado; nunca pre-estrillar a ASCII.
- **Resumen y titular en español de México (traducción obligatoria)**: el resumen
  (renglon 3) y el titular (renglon 1) se redactan SIEMPRE en español de Mexico
  (ortografia y tono de MX), sin importar el idioma original de la fuente. Si el
  titular/resumen vienen en otro idioma (ingles, etc.), traducirlos al español de
  Mexico — nunca dejarlos en el idioma original. La caratula del renglon 2 enlaza
  igualmente al articulo original (en su idioma). Verificar que ningun resumen ni
  titular aparezca en ingles u otro idioma en el HTML decodificado.
- **Cada item lleva link al articulo completo** (correccion real de BarraMan): se
  esperaba un digest de 10 titulares sin URL. Esperar 2 links por item (caratula +
  Leer completa). Verificar decodificando la parte text/html (va base64 CTE: grep
  RAW = 0; decodificar primero) y contar href; esperar N items x 2.
- **Imagines como CID attachment, NO data-URI y NO hotlinks**. Gmail NO renderiza
  data:image/...;base64 inline (solo un icono placeholder) y puede bloquear hotlinks.
  Adjuntar cada caratula como MIME part con Content-ID y referir src=cid:newsN.
  El cuerpo DEBE ser multipart/mixed (image parts al lado de alternative).
- **Un solo cuerpo HTML** salvo que pidan adjunto PDF; nunca adjuntar un Markdown
  como fallback.

## 4. Construccion y envio (HTML + CID + himalaya)

**Squeleto HTML** (tabla + CSS inline, stack Helvetica, Carlito, Segoe UI, Roboto,
Arial, sans-serif; fondo #eef1f5, tabla 640px). Cada item 3 renglones.

**Caratula = og:image normalizado via PIL (OBLIGATORIO)**: por cada URL del titular,
GET el articulo, regex og:image (fallback twitter:image), **luego normalizar via PIL
antes de adjuntar**:
  im = Image.open(io.BytesIO(raw)); im = im.convert(RGB); im.save(dest, JPEG, quality=88)
Adjuntar bytes crudos es la causa #1 de un envio roto (InvalidContentType). Si no hay
og:image, generar una portada (color de categoria + texto) — nunca renglon en blanco.

**MIME con MIMEMultipart, NO EmailMessage.add_alternative** (da TypeError en py3.12).
Si el cuerpo tiene imagines usar multipart/mixed para que las image parts vayan al lado
de alternative. Referir src=cid:carN. MIMEImage maneja el base64 CTE solo.

**Pitfall verificado (2026-08-31)**: himalaya message send da rc=1 InvalidContentType
 cuando una caratula no es JPEG valido (un fetch crudo que devolvio HTML/0 bytes, o
una imagen que PIL verify acepta pero no es real). Solucion = la normalizacion PIL de
arriba. Tras reenviar, re-chear la carpeta sent (un intento fallido puede dejar una
copia duplicada en Sent).

**Enviar y verificar**:
  himalaya message send --save sent -- <file.eml>      # rc=0 = entregado + guardado
  himalaya envelope list --mailbox sent                # confirmar asunto + size, sin dup
message send success SOLO NO es prueba — siempre confirmar en sent y checar duplicados.

## 5. himalaya v2.1.0 (config y gotchas)

El bloque de config del skill himalaya empaquetado es el formato pre-v1.2.0 y es
INCORRECTO para v2.1.0+. Si da No backend matching auto is configured, es el viejo formato.

- **Instalar**: curl .../install.sh | PREFIX=~/.local sh ; binario ~/.local/bin/himalaya.
- **Password NO en .env** (secret-guardado): en ~/.local/creds/email (chmod 600),
  pasado via password.command = cat /ruta/creds/email.
- **config.toml v2.1.0 (URIs, sasl.login, mailbox.alias)**:
  [accounts.x] email=, display-name=, default=true
  imap.server = imaps://host:993   (imaps:// = TLS implicito)
  imap.sasl.login.username = user
  imap.sasl.login.password.command = cat /ruta/creds/email
  smtp.server = smtps://host:465  (smtps:// = TLS implicito 465)
  smtp.sasl.login.username = user
  smtp.sasl.login.password.command = cat /ruta/creds/email
  mailbox.alias.inbox = INBOX ; .sent = INBOX.Sent Items ; .drafts = ... ; .trash = ...
  Claves viejas (rechazadas): backend.type/host/port, backend.auth, folder.aliases.*.
- **Probar el mecanismo AUTH, no adivinar**: el server puede rechazar sasl.plain
  (504 PLAIN not supported). Sondear: printf EHLO... | openssl s_client -connect H:465
  -quiet 2>/dev/null | grep AUTH ; emparejar sasl.<mech> (ej. LOGIN -> sasl.login.*).
- **Comandos que cambiaron**: folder list -> mailbox list ; envelope list --limit N ->
  --page-size N ; --folder Sent -> -m INBOX.Sent Items (alias-resolved).
- **Enviar MIME pre-construido**: himalaya message send --save INBOX.Sent Items < msg.txt
  (lee RFC 5322 de stdin). Confirmar en Sent antes de reportar exito.

## 6. Modelo de cron (BarraMan)

El digest va a su correo Y a una copia. **Elaborar las titulares UNA vez y enviar un
mensaje unico** con To: primary y Cc: copy — NUNCA construir el digest dos veces ni
fijar dos cron jobs (BarraMan lo llamo sumamente ineficiente). La segunda persona va en
Cc del mismo mensaje.

Un job recurrente (semanal, Lun 07:00) elabora una vez y entrega a primary@example.com
con Cc copia. No mantener un segundo job copia. Si una corrida falla (ej. timeout del
modelo), **debe auto-reportar estado al canal del usuario** — no no-op en silencio.

## 7. Timeout en turnos largos no-streaming (2026-08-31)

Un turno pesado (muchas busquedas + fetch de imagines + dos emails grandes) puede
exceder el stale timeout y fallar con Non-streaming API call timed out after 180s.
El 180s es el piso del reasoning-model, NO el timeout del terminal. El override
que gana es providers.<id>.stale_timeout_seconds (y request_timeout_seconds) via
hermes config set --force providers.<id>.stale_timeout_seconds 1500 (NO editar
config.yaml a mano). El resolver lee config[providers] (un DICT), NO custom_providers
(list) — el override va bajo providers.<id> (ej. custom, match model.provider).

## Files
- references/sources.md - Google News / BBC / Reuters / Nature RSS + sintaxis.
- references/curate.md - reglas de dedupe por cluster + lista junk.
- references/exa-search.md - shape de busqueda Exa + 402 fallback.
- references/formato-casa.md - el shape 3-renglones canonico.
- references/build-send-script.md - como el digest se produce end-to-end.
- references/himalaya-send.md - envio via himalaya v2.1.0 + gotchas AUTH.
