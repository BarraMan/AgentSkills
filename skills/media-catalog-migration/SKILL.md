---
name: media-catalog-migration
description: Process & migrate BarraMan's new media downloads.
---

# BarraMan Media Catalog Migration

Pipeline that classifies, renames, organizes, embeds (ID3+cover), and serves a
media collection for BarraMan.

## Locations (all real, verified)
- Catalog root:      `CATALOG/`
- New downloads in:  `$HOME/_incoming/audio/` and `$HOME/_incoming/video/`
- Legacy media in:   `$HOME/inventario/multimedia/audio|video/`
- Web server:        `serve_inventario.py` (port **9000**)
- Python (venv):     `python3`
- EXA API key:       `<EXA_KEY_FILE>` (chmod 600, NEVER print)
- Index:             `CATALOG/index.json`
- Backup manifest:   `CATALOG/backup_manifest_pre_mig.json`
- Covers cache:      `CATALOG/_covers/<sha1>.jpg`
- Restart helper:    `_restart_web.sh` (kill+relaunch server)

Key modules: `processor.py` (classify/rename/move/embed), `artist_kb.py`
(curated artist→genre KB), `genres.py` (taxonomy), `covers.py` (cover fetch).

## Standard workflow (DO IN ORDER)
1. **DRY-RUN first** (never mutate before verifying classification):
   `python3 -c "import processor; processor._ML_MODEL={}; ...; processor.process_all(files, dry_run=True)"`
   Inspect genre/artist/album per file. Fix issues, THEN run for real.
2. **Migrate for real**: `processor.process_all(files, dry_run=False, verbose=True)`.
3. **Verify**: integrity (mutagen), covers via HTTP, contamination, counts.
4. **Restart web server** after any new cover/index change (cover sibling cache
   is built once at startup — stale until restart).

## processor.py public API
- `classify_file(fpath) -> {artist, album, song, genre, confidence, source, cover}`
- `process_one(fpath, dry_run=False) -> {path, ...}` (move+rename+embed+index)
- `process_all(files, dry_run=False, verbose=True)` — seeds used-dests, processes all
- `move_to_target(fpath, result, dry_run=False)` — rename to dest, collision-safe
- `embed_metadata(fpath, artist, album, song, cover=None)` — ID3/APIC, try/except
- `identify_album_from_itunes(parsed)` / `identify_album(parsed)` — album via web-search
- `get_cover` in `covers.py` — 3-tier: YouTube (video id), iTunes (audio API), ffmpeg (frame)

## CRITICAL pitfalls (each caused real data/quality loss)
- **DRY-RUN before any real migration.** Renaming is destructive. Always preview
  classification and resolve issues before `dry_run=False`.
- **Exclude in-progress downloads.** If a download (yt-dlp/etc.) is still running,
  migrate ONLY complete/stable files (age > ~180s, no `.part`/`.tmp`). Migrating a
  half-written file = corrupt. Check `ps aux | grep yt-dlp`.
- **COLLISION avoidance is built into `move_to_target`** (numeric suffix, never
  overwrites). `_seed_used_dests()` registers pre-existing files first. Two files
  → same dest (e.g. "Vals y Corridita" twice) must not lose data.
- **FLAC extension bug**: an old bug forced `.mp3` for all audio, corrupting FLAC.
  `move_to_target` now preserves the original extension (`os.path.splitext`).
  If FLACs are mis-named `.mp3`, rename back + re-embed (mutagen reads flac via
  magic bytes `fLaC`).
- **MIX/compilation folders are NOT artists.** `_is_mix_folder(name)` detects
  mix/compilación/mashup/playa/hits/best of/"de los 2000"; such folders must NOT
  override the artist (use filename's artist), but DO feed the genre haystack
  (e.g. "pop"). `_parent_artist` overrides artist ONLY for a real single-artist folder.
- **iTunes album contamination**: matching `trackName == song` alone contaminates
  (e.g. "Big Girls Don't Cry" exists in a Frankie Valli compilation vs Fergie).
  `identify_album_from_itunes` MUST verify the track's `artistName` matches our
  artist; otherwise skip → EXA → "Varios".
- **Cover sibling cache is LAZY** (`_cover_sibling_index` built once at startup).
  After any new cover/index change, RESTART the web server or covers 404.
- **Use os.walk, NOT glob.glob** for covers — glob silently skips files with
  `...` in the name (e.g. "…Baby One More Time"); os.walk finds them.
- **Accent/encoding**: always unquote URLs (`urllib.parse.unquote`) and normalize
  accents; "Rócio/Rocío/Dúrcal/Durcal" variants unify via the folder name.
- **ML disabled** (`processor._ML_MODEL = {}`): GTZAN misclassifies regional-mexicano
  as hiphop. Genre = curated KB (0.95) > EXA web-search > keyword.

## Verification (definition of "al 100%")
- Integrity: every media opens in mutagen, `info` non-null, 0 problems.
- Covers via HTTP: for each item, fetch `/cover/<ident>` → 200 and >500 bytes.
  (Test the URL the API produces, not a hand-built ident.)
- Contamination: album must not contain another artist's tokens (e.g. "frankie
  valli" in an album for a different artist).
- Counts: `_incoming` and `multimedia` should be 0 after migration.
- Genre distribution printed from `index.json`.

## Web server + Nginx (Docker)

- Web server Python: `serve_inventario.py` en `:9000`.
  - Start: `python3 serve_inventario.py 9000` (background daemon; NO nohup/setsid).
  - Restart clean: `bash _restart_web.sh` (kills stragglers, frees port).
  - Endpoints: `/` (UI), `/api/list` (returns `{"files":[...]}` — key is "files",
    NOT "items"), `/cover/<ident>` (ident = basename without `.cover.jpg`).
  - LAN `<LAN_IP>:9000`. Port 8080 = OWUI (do NOT use).
- **Nginx en Docker** (BarraMan autorizó a BarraBot a administrar
  `barra-nginx-catalog`, nginx:stable):
  reverse proxy `:80` -> `host.docker.internal:9000`.
  - Config: `nginx/nginx.conf`.
  - Arranque: `nginx/start_nginx.sh` (sólo gestiona
    `barra-nginx-catalog`, NO toca los criticos).
  - **CRITICO**: usa `host.docker.internal` via
    `--add-host=host.docker.internal:host-gateway`. NO usar la IP LAN del host
    (<LAN_IP>): el contenedor NO la enruta -> "Connection refused".
  - **CRITICO**: `upstream` NO puede ir dentro de `server` (directive invalida);
    usar `proxy_pass http://host.docker.internal:9000;` directo.
  - Streaming: `proxy_buffering off` + `proxy_read/send_timeout 3600s`
    -> 206 (Range) para video/audio. Verificado: 206 con Range 0-9999 y
    1000000-1099999.
- **NO tocar** los docker criticos: qwen38-vllm (8000), gemma4-vllm (8002),
  open-webui (8080=OWUI), barrabrain (litellm), barrabrain-postgres (5432).
- **UI (serve_inventario.py)**: al reproducir video, ocultar la portada
  (.nowart) y `pause()` el otro reproductor — display:none NO detiene el audio
  en HTML5; si no pausas, video+cancion suenan juntos.
- El cache de covers del server se construye 1 vez (lazy) al ARRANQUE; tras
  mover/renombrar/añadir covers, **reiniciar el web server** (cache stale -> 404).
- `os.walk` (NO glob.glob) para enumerar covers: glob se salta nombres con `...`.
## Do NOT
- Do NOT print the EXA key or any credential.
- Migrate while a download is active without excluding in-progress files.
- Skip the dry-run.
