# myReelsDownloader

An Antigravity agent skill that downloads all saved/favourited media from an **Instagram** or **TikTok** account into an organised local archive, and sets up a recurring **Windows Task Scheduler** job to keep it in sync automatically.

---

## Overview

| Feature | Detail |
|---|---|
| **Platforms** | Instagram (Saved Posts + Collections), TikTok (Favourites + Collections) |
| **Media types** | Videos (`.mp4`), single images, photo carousels |
| **Output root** | User-defined at startup (created automatically if it doesn't exist) |
| **Scheduling** | Windows Task Scheduler — daily or weekly, user-defined window |
| **Safety** | Skip-if-exists — re-runs are always safe |

---

## How it Works

### 1. Collect Info (asked before anything starts)

The skill will prompt you for three things:

1. **Account URL(s)** — one or more Instagram profile URLs and/or TikTok profile URLs.
2. **Schedule** — when the daily/weekly sync should run (e.g. `every Friday 1:00 PM to 10:00 PM` or `daily 09:00`).
3. **Project Root Path** — the local folder where all media will be saved (e.g. `D:\MyReels`). The folder is created automatically if it doesn't exist; each account gets its own subfolder inside it.

### 2. Enumerate Saved Media

**Instagram**
- Opens `instagram.com/<user>/saved/` in the in-app browser (you must be logged in).
- Fetches saved posts and named collections via the internal Instagram API.
- Builds `manifest.json` (all saved) and `manifest_unsorted.json` (uncategorised posts).

**TikTok**
- Opens `tiktok.com/@<user>?tab=favorites` in the in-app browser (you must be logged in).
- Fetches favourites and collections via TikTok's internal API, paginating until complete.
- Builds `tiktok_manifest.json`.

### 3. Download

**Instagram** — runs `node download.mjs`:
- Videos saved as `<creator>_<id>.mp4` inside the collection-name folder.
- Photo carousels saved into a folder named after the post caption (`01.jpg`, `02.jpg`, …).
- Runs `node fix-extensions.mjs` afterwards to correct any mislabelled file extensions.

**TikTok** — runs `node download_tiktok.mjs`:
- Videos downloaded via `yt-dlp` preferring H.264 (up to 720p with HEVC).
- Images fetched directly from the TikTok CDN.
- "Sensitive content" videos retried via in-browser fetch (uses session cookies).
- Runs `node verify_tiktok.mjs` for integrity checks.

### 4. Schedule the Sync

After the first download, the skill creates (or updates) a **Windows Task Scheduler** job:

- **Task name:** `MyReels <account> Sync`
- **Action:** runs `daily_download.bat` for the account.
- Schedule is derived from your answer to question 2.

---

## Output Structure

```
<project-root>\
└── <account>\
    ├── <Collection Name>\
    │   ├── <creator>_<id>.mp4
    │   └── <caption> (<id>)\
    │       ├── 01.jpg
    │       └── 02.jpg
    ├── Unsorted\
    │   └── ...
    ├── manifest.json
    └── manifest_unsorted.json
```

---

## End-of-Run Report

The skill prints a summary table after every run:

```
Source      | Posts | Videos | Images | Size  | Failures
------------|-------|--------|--------|-------|----------
Instagram   |  120  |   95   |   87   | 4.2GB |    2
TikTok      |   80  |   74   |   12   | 2.1GB |    0
```

Plus a per-collection folder breakdown and any unmatched collection items.

---

## Rules & Notes

- **Never overwrites** existing files — incremental re-runs are safe.
- Duplicate captions get ` (<id>)` appended to the folder name (base truncated to 55 chars).
- TikTok collection route `/@user/collection/…` is broken in the in-app browser — the skill works around this by matching via cover-image stem.
- `browser.evaluate` responses are capped at ~512 KB — large data is automatically chunked.

---

## Skill File

The agent skill definition lives in [`reelsDownloader.md`](./reelsDownloader.md).

