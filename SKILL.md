---
name: wechat-data-extraction
description: "Extract files from WeChat (Windows 4.x): locate the xwechat_files data dir, find cached videos (channels/chat/SNS), and understand what the web preview API can and cannot give you. Use when the user asks to save/download a 微信视频号 video, recover a chat video, or pull files out of WeChat's local cache."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [windows]
metadata:
  hermes:
    tags: [WeChat, Video-Download, Cache-Forensics, Playwright]
    related_skills: [playwright, image-content-analysis]
---

# WeChat Data Extraction (Windows 4.x)

## Locate the data directory

WeChat 4.x stores everything under a per-account `xwechat_files` folder. **Default is `Documents\xwechat_files` but users commonly relocate it** (this machine: `E:\个人文件\xwechat_files\`). Find it:

```
find /c/Users/<user> /d /e -maxdepth 3 -type d -name "xwechat_files" 2>/dev/null
```

Structure: `xwechat_files\<wxid>\` with `cache\<YYYY-MM>\` (HttpResource hash-named blobs), `msg\video\<YYYY-MM>\` (chat videos, `*.mp4` + `*_thumb.jpg`), `msg\attach\` (files), `db_storage\message\*.db` (SQLite message DBs).

## Saving a 微信视频号 (WeChat Channels) video

**TL;DR: the web preview page does NOT expose the video stream. The reliable path is: play it in the WeChat client once, then pull it from the client's cache.**

1. **Web preview page** (`https://channels.weixin.qq.com/finder-preview/pages/sph?id=<ID>`): shows description/author/counters and a cover image, but its API returns **no videoUrl** — video playback is gated behind WeChat client (扫码). Don't burn time reverse-engineering this for the stream.
   - Feed API (for metadata/cover verification): `POST /finder-preview/api/feed/get_feed_info` with body `{"baseReq":{"generalToken":""},"shortUri":"<ID>"}` (or `exportId` from `sceneInfo.dynamicExportId`). Response has `feedInfo.coverUrl` (finder.video.qq.com stodownload, image/jpg — usable to confirm WHICH video a cache file belongs to) but no video.
2. **Client-cache path**: ask the user to open the link in WeChat (paste into a chat, click it, play a few seconds — buffering downloads it), then scan the cache for new video blobs:
   - `msg\video\<month>\*.mp4` (chat-forwarded videos land here immediately)
   - `cache\<month>\HttpResource\*` — hash-named; detect MP4s by magic bytes (`ftyp` at offset 4)
   - Match the right one by size recency + comparing thumbnail/cover (cover URL from the API → decode `encfilekey` params, or just compare first-frame).
3. **Verify before deleting anything**: WeChat may be running and writing — check file size stability and don't delete the source until the copy is confirmed playable.

## Extracting API data from SPA pages (general technique)

When a JS SPA loads data via XHR/fetch and you need the response (WeChat preview page, games, dashboards):

- **Playwright network capture** is the reliable way (install from tsinghua mirror; use `channel="chrome"` with the system Chrome — no browser download needed):
  ```python
  from playwright.sync_api import sync_playwright
  with sync_playwright() as p:
      browser = p.chromium.launch(channel="chrome", headless=True)
      page = browser.new_page()
      page.on("response", lambda r: print(r.url, r.status))
      page.goto(url, wait_until="networkidle")
  ```
- Fetch-hook monkey-patching via console (`window.fetch = ...`) does NOT survive `browser_navigate`/reload — use it only for SPA internal navigation, or skip it and use Playwright.
- Compressed JS bundles: search the source map-less bundle for the API name (e.g. `get_feed_info`) to learn request shape; the caller pattern usually sits within ±1500 chars.

## Pitfalls

- **Cover vs video**: `finder.video.qq.com/.../stodownload?...picformat=200` returns a **JPEG cover**, not the video — don't mistake it for the stream.
- **Don't delete cache while WeChat runs** without checking; 4.x re-writes month dirs.
- **wxid hash**: the folder is `<wxid>_<hash>` — glob `xwechat_files/*/` if unsure.
- SNS (朋友圈) videos live in `cache\<month>\Sns\Video\<xx>\*.mp4` (may be `.tmp` extension but MP4 magic).
