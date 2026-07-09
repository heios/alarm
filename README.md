# Cue Sheet

A single-file desktop reminder that watches a time-stamped schedule and gives you a heads-up — a ding plus an on-screen list — a few minutes before each cue. Built for the moment where you need to glance at another app *right when something is about to happen*.

**Live:** https://heios.github.io/alarm/

Paste a schedule as one `time: message` per line, press **Play** to arm the watch, and Cue Sheet tracks the current time across a timeline. When you're within your chosen lead time of the next batch of cues, it sounds an alert and shows those messages under the **NOW** marker.

## Features

- **Paste-and-go schedule** — `14:30: Call with client`, one per line; 24-hour or `am`/`pm` both parse. Prefix a date to cross days — `10 Jul 09:00: …` or `Jul 10, 09:00: …` (month name or abbreviation, comma optional). Cues at the same time group into one batch.
- **Timeline at a glance** — colored dots (matching the list), a live **NOW** marker that turns red while watching and gray when stopped, and a background band showing where the watch was running (green), stopped (gray), or not yet started (striped). Hover a dot to see every cue at that time.
- **Fired / left counter** — under the timeline, a running count of how many cues have passed and how many are still ahead (each cue counted individually).
- **The alert** — while armed, Cue Sheet sounds a ding and shows the next batch's messages under the NOW marker when you're within your lead time. Turn on **desktop alerts** to also get a native OS notification.
- **Configurable alert lead time** — set how many minutes ahead you want the heads-up (defaults to 7). Click anywhere in the box to edit it.
- **Focused list** — shows a five-cue window around now (a couple just done, the next one, a couple coming up); expand for the full schedule.
- **Light / Dark / Auto** theme (Auto follows your OS), keyboard-navigable.
- **Everything persists** — your schedule, settings, and the watch's running state are saved in the browser's local storage, so a reload picks up where you left off. Nothing leaves your machine.

## Run it locally

It's one static file with no build step and no dependencies. Either:

```sh
# open directly
open index.html            # macOS  (or: xdg-open index.html on Linux)

# …or serve it (needed only if your browser restricts local file features)
python3 -m http.server 8000   # then visit http://localhost:8000
```

## Deploy to GitHub Pages

Published as a **project site** at `https://heios.github.io/alarm/` — so the GitHub repo is named **`alarm`**.

**Option A — GitHub Actions (included).** Push to `main`. In the repo's **Settings → Pages**, set **Source: GitHub Actions**. The bundled workflow at [`.github/workflows/deploy-pages.yml`](.github/workflows/deploy-pages.yml) publishes the repo root on every push.

**Option B — deploy from a branch.** In **Settings → Pages**, set **Source: Deploy from a branch**, branch `main`, folder `/ (root)`. The included [`.nojekyll`](.nojekyll) tells Pages to serve the files as-is.

Either way the entry point is `index.html` at the repo root, so it works unchanged from the `/alarm/` subpath — all assets are inlined and every path is relative.

## Privacy

Cue Sheet runs entirely in your browser. The schedule and settings live in `localStorage` on your device; there are no network requests, analytics, or servers.
