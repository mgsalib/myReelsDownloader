---
name: myreels-download
description: Download all saved/favourited media from an Instagram or TikTok account the user provides. Enumerates saved collections / favourites via the in-app browser, generates a manifest, downloads videos and photo carousels into tag-named folders, skips existing files, and reports totals. Use when the user gives a social media account (username or profile URL) and asks to download their saved reels/favourites.
---

# myreels-download

## When to use

Load this skill when the user provides a social media account (Instagram handle/URL or TikTok handle/URL) and asks to download their saved/favourited media. Works for Instagram (saved posts + collections) and TikTok (favourites + collections).

## Collect from user first (ask before doing anything)

Before starting, ask the user these three questions:

1. **What is the Social Media Account URL(s)?** — Instagram profile URL(s) and/or TikTok profile URL(s) to download saved/favourited media from. Accept one or many.
2. **What is the Schedule Start and End Day and time?** — when should the daily/weekly sync run (e.g. "every Friday 1:00 PM to 10:00 PM" or "daily 09:00"). Used to create/update the Windows Task Scheduler job.
3. **What is the Project Root Path?** — the local folder where all media will be saved (e.g. `D:\MyReels`). If the path does not exist, create it. New accounts get a subfolder created under it automatically.

Only proceed after all three answers are provided. Derive the username from the URL if not given separately.

## Inputs (derive, do not ask if missing)

- Account identifier: Instagram username/URL or TikTok username/URL.
- Project root: provided by the user (question 3). Create the path if it does not exist; new accounts get a subfolder created under it.
- Browser tab: use the in-app browser; user must be logged in to the account.

## Work loop (copy this order)

### Instagram

1. Open `https://www.instagram.com/<user>/saved/` (or saved collections). Log in if prompted.
2. Enumerate saved posts via internal API `/api/v1/feed/saved/posts/?count=12` and collections via `/api/v1/feed/collection/<id>/posts/?count=12`.
3. Build `manifest.json` (all saved) + `manifest_unsorted.json` (no collection) with rows `{tag, code, user, caption, mt, vid, img, car[]}`.
   - `mt`: 2 = video, 8 = photo carousel, 1 = single image.
   - `tag`: collection name or `Unsorted`.
4. Run `node download.mjs --manifest <manifest> --root <root> --csv <csv> --referer "https://www.instagram.com/"`.
   - Skips files that already exist; re-running is safe.
5. Run `node fix-extensions.mjs --root <root>` to fix mislabelled extensions.

### TikTok

1. Open `https://www.tiktok.com/@<user>?tab=favorites`. Log in if prompted. Switch to "Collections" sub-tab (focus + Enter; click refs don't work).
2. Enumerate favourites via `/api/user/collect/item_list/?...&secUid=<secUid>` (page-context fetch with cookies). Paginate with cursor until `hasMore` false.
3. Enumerate collections via `/api/user/collection_list/`. Match each collection to an item by cover-image stem (collection route is broken in this browser).
4. Build `tiktok_manifest.json` with rows `{tag, code, user, caption, mt, vid, img, car[]}`.
   - `mt`: 2 = video, 8 = carousel, 1 = single image.
   - `tag`: collection name or `Unsorted`.
5. Run `node download_tiktok.mjs`.
   - Videos: `yt-dlp` with `-S "codec:h264,res,br"` (H.264 caps at 540p; HEVC reaches 720p). URL form: `https://www.tiktok.com/@<user>/video/<id>` (percent-encode Arabic/space usernames).
   - Images: direct fetch from TikTok CDN (200 OK).
   - For "sensitive content" videos yt-dlp can't fetch: use in-browser fetch with credentials, split into 2 MB parts via blob download, then concatenate.
   - Set `PYTHONUTF8=1` / `PYTHONIOENCODING=utf-8` in spawn env.
6. Run `node verify_tiktok.mjs` for integrity.

## Scheduling

After the download completes, create (or update) a Windows Task Scheduler job using the user's answer to question 2:

- Parse "Start and End Day and time" into a schedule. Examples:
  - "every Friday 1:00 PM to 10:00 PM" → `/SC WEEKLY /D FRI /ST 13:00 /RI 60 /DU 09:00`
  - "daily 09:00" → `/SC DAILY /ST 09:00`
  - "every Monday and Thursday 2 PM to 8 PM" → two tasks or one with `/D MON,THU`.
- Action: run `daily_download.bat` (or a per-account equivalent).
- Name the task `MyReels <account> Sync`.
- Confirm the created task (next run time) to the user.

## Rules

- Never overwrite existing files; always skip-if-exists.
- Prefer H.264 for compatibility (TikTok caps it at 540p).
- Photo carousels → one folder per post named by caption's first line, images `01.jpg`, `02.jpg`, …
- Videos → `<creator>_<id>.mp4` in the tag folder.
- Report totals at the end: posts, videos, images, size (GB), failures.
- If a video fails with "sensitive content", retry via in-browser fetch (page context has the session cookies).
- For duplicate captions, append ` (<id>)` to the folder name (truncate base to 55 chars so the id isn't cut off).

## Pitfalls

- TikTok collection route (`/@user/collection/…`) is broken in this browser — match by cover-image stem.
- TikTok CDN rejects plain Node fetch for videos (403) — needs session cookies; use yt-dlp or in-browser fetch.
- `/api/collection/item_list/` returns 400 "missing required fields" — use `/api/user/collect/item_list/` instead.
- Browser `click` refs are unusable on TikTok; use JS `.click()` or focus + Enter.
- `browser.evaluate` caps responses at ~512 KB — chunk large data.
- Code Mode `execute` cannot call `write`/`edit` tools; use top-level tools for file writes.

## Report format

```
Source | Posts | Videos | Images | Size | Failures
```

plus per-tag folder breakdown and any unmatched collection items.
