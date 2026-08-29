# Catalog pipeline — verified bug-fix log (2026-08-29 session)

Concrete, tested fixes from a full run: 608 files migrated, 608/608 integrity
OK, 595/595 covers served over HTTP (0 fail), 0 album contamination, 0 leftover
in `_incoming`/`multimedia`. Every item below was verified by a script, not
asserted.

## 1. FLAC files saved as `.mp3` (corrupt playback)
- **Symptom:** `move_to_target` forced `ext = ".mp3"` for any audio, so 216 FLAC
  files became `.mp3`. `mutagen.File().info` returned None for them.
- **Fix:** `ext = os.path.splitext(fpath)[1] or ".mp3"` (preserve the original).
- **Recovery recipe:** detect by magic bytes (`open(f,'rb').read(4) == b'fLaC'`),
  rename `.mp3` → `.flac` (and the `.cover.jpg` sibling), then re-embed metadata
  (the original embed failed because mutagen couldn't parse a FLAC under a
  `.mp3` extension) and rewrite `index.json` `path`/`thumb`.
- **Integrity check that WORKS:** compare **total size** pre/post (embedding only
  adds bytes, never removes) + `mutagen.File(f).info is not None`. Do NOT use a
  first-1MB hash — ID3/APIC embedding rewrites the header, so that hash changes
  even though audio is intact (this caused a false "422 missing" alarm).

## 2. Collision → data loss
- **Symptom:** two distinct recordings (e.g. two "Vidalita", two "Ñau Ñau")
  resolved to the same destination; `os.rename` would overwrite → 144 files
  across 64 collisions at risk.
- **Fix:** `_USED_DESTS` set; on collision append `_2`, `_3` … before rename.
  `process_all` calls `_seed_used_dests()` first (walk the catalog, record every
  existing `.mp3/.mp4/.flac/.wav/.ogg`) so a move never clobbers a prior
  migration.

## 3. Mid-download migration
- **Symptom:** a live yt-dlp run left 4 files with `mtime < 120s`; migrating
  them moved a half-written file.
- **Fix:** exclude names ending `.part/.ytdl/.tmp/.download` AND files with
  `mtime < ~180s`. Migrate only complete+stable; re-migrate stragglers after the
  downloader finishes (0 procs, 0 partials, no files <180s). Ran in 2 batches
  (98 then 21) as the download advanced.

## 4. Mix/compilation folder used as the artist
- **Symptom:** folder `los 2000 💋： mix pop en ingles y español` was forced as
  the artist of every file (Britney, Shakira, Reik … all got artist="los 2000
  mix pop…"). The "folder = canonical artist" rule (added for Rocío Durcal
  case/accent unification) broke on a compilation folder.
- **Fix:** `_is_mix_folder(name)` — True if it contains mix/mash/compil/playlist/
  hits/greatest/best of/medley, or a decade token (2000/90s/80s) with "mix". When
  the parent folder is a mix: keep the artist parsed from the filename, but put
  the folder name into the GENRE keyword haystack so genre still resolves (pop).
  Result: 83/83 pop, 0 Sin_clasificar, artists correct.

## 5. iTunes album cross-contamination
- **Symptom:** `Fergie - Big Girls Don't Cry` → album "The Very Best of Frankie
  Valli and the Four Seasons". `identify_album_from_itunes` matched
  `trackName == song` only, ignoring that the matching track's `artistName` was a
  different artist.
- **Fix:** after a `trackName` match, require `artistName` (normalized) to
  contain the artist (or vice-versa); else `continue` to the next hit. No
  artist-token fallback (that assigns one album to many songs). Fergie → album
  "Varios", genre pop, correct path. Scan for residual contamination by checking
  whether an album mentions a known other-artist token not present in artist/song
  (0 remaining after fix).

## 6. Store-ID slugs mistaken for album names
- **Symptom:** `b01knha3ug`, `b000qzr1em` appeared as album names.
- **Fix:** in `_clean_slug`, reject a single token containing any digit as a
  store ID (Apple/Deezer/Spotify/Qobuz). A "no-vowels" test is insufficient —
  these IDs *can* contain a vowel (`b01knha3ug` has an `a`).

## 7. Web server covers 404 — the "thumbnails don't show" cluster
1. `import glob` was missing at module top; the lazy `_cover_sibling_index()`
   raised `NameError` → every cover 404. Add the import.
2. `_COVER_SIBLING_CACHE` is built once (lazy) at first request — it doesn't see
   covers added after startup. **Restart the server** to rebuild it.
3. `glob.glob("**/*.cover.jpg")` silently skips names containing `...`
   (e.g. `...Baby_One_More_Time...`); `os.walk` finds them. Build the sibling
   index with `os.walk`, not `glob`.
4. The `/cover/<id>` handler must serve BOTH the hex cache
   (`_covers/<hash>.jpg`, id = `[a-f0-9]{20,64}`) AND the new `.cover.jpg`
   siblings (id = URL-quoted file basename without `.cover.jpg`).
5. After a rename/repair, `index.json` `cover`/`thumb`/`has_cover` may point at
   a `.mp3` that no longer exists (real file is `.flac`/`.mp4`); realign those
   fields from the files actually on disk.
6. **Verify via the API URL the browser uses**, not a hand-built path:
   `GET /api/list` → item `cover` field → `GET` that URL → assert `200` + `>500
   bytes`. Final: 595/595. (A hand-built URL often mismatches the server's
   encoding and yields false failures.)

## Verification template (always run before reporting "done")
- media count + `mutagen` OK / N
- covers served over HTTP: N/total, 0 fail (via API URLs)
- contamination count (album vs artist mismatch)
- leftover in `_incoming`/`multimedia` (should be 0)
- genre distribution
- any count mismatch (e.g. 297 vs 477 covers) → diagnose source
  (sibling vs cache vs index) before trusting it.
