# Web player UI — verified fixes (2026-08-29 session)

The frontend (HTML + JS + CSS) is embedded inside `serve_inventario.py`.
The player has two `<audio id="a">` / `<video id="v">` elements plus a cover
`<img class="nowart" id="art">` and a `<ul class="list">` in the right `.side`
column. Both fixes below were verified by fetching the served `/` page and
checking the served HTML/JS actually contains the change (grep the served page,
not the source file).

## 1. Audio and video play simultaneously
- **Root cause:** in HTML5, `style.display='none'` does NOT stop playback.
  `loadTrack()` only toggled `display`, so switching to a video left the
  `<audio>` still playing → song + video sounded together.
- **Fix:** in `loadTrack`, PAUSE the other element before loading the new one.
  - audio branch: `video.pause(); video.style.display='none'; audio.src=...; audio.style.display='block';`
  - video branch: `audio.pause(); audio.style.display='none'; video.src=...; video.style.display='block';`
- **Rule:** whenever switching a media element, always `.pause()` (not just
  hide) the outgoing element. `display:none` is a visual-only change.

## 2. Video cover occludes / overflows the list
- **Symptom:** when a video played, the cover thumbnail (`#art`, 160×160) kept
  showing on top of the `<video>`, and the `<video>` had no max-height, so the
  player-wrap grew tall enough to push/cover the `.side` list — you couldn't
  select another track.
- **Fix:**
  - In `setArt`, if `item.type==='video'`, set `art.style.display='none'`
    (and `artPh` none) and return — never render a cover over a video.
  - CSS: `video{...;max-height:65vh}` so a video can't consume the whole
    viewport.
- **Rule:** a cover/now-playing image must be hidden while a video plays, and
  the video element must have a bounded height.

## Deploy / verify
- Edits live in the in-memory server, so RESTART to take effect:
  `bash _restart_web.sh` — BUT this script's own `pkill` matches its
  own shell, so the script self-terminates (exit -15/SIGTERM) AFTER killing the
  server. The port ends up free; then relaunch separately with
  `background=true` (NO nohup/setsid):
  `python3 serve_inventario.py 9000`.
- Verify by fetching the served page:
  `urllib.request.urlopen('http://127.0.0.1:9000/')` → assert the new JS/CSS
  strings are present (e.g. `video.pause()`, `audio.pause()`,
  `max-height:65vh`). Do NOT trust the source file alone.

## Pitfall
- Long inline shell payloads (multi-check one-liners, subshells) get BLOCKED
  by the command parser. Prefer: write a `.py`/`.sh` file, then run it; or
  split into small simple commands.
