# WockingTocking — Episode 01 Trip App

A mobile-first web app for the first WockingTocking vlog episode: a road trip from
Little Yeldham, Essex to Pudsey (Leeds) and back, 23–25 May 2026.

Open it on a phone in Safari and "Add to Home Screen" to use it like a native app.

## What's inside

The app has four tabs:

- **Trip** — the overview, the three-day arc, kit list and pre-trip checklist.
- **Route** — tap-to-navigate Google Maps and Waze links: driving links between towns
  and walking loops within each one.
- **Plan** — the timed agenda for each day: windows, "leave by" times, what to film.
- **Talk** — what to say on camera at every location.

## Files

| File | Purpose |
|------|---------|
| `index.html` | The whole app — HTML, CSS and JS in one file |
| `manifest.json` | Web app manifest (Add to Home Screen) |
| `sw.js` | Service worker — caches the app for offline use |
| `icon-192.png`, `icon-512.png`, `apple-touch-icon.png` | App icons |

It is a fully static site — no build step and no dependencies.

## Deploying to Vercel

This repo is ready to deploy as-is. Two ways:

1. **From this repo** — in the Vercel dashboard, "Add New Project", import
   `vlog-planner`, and deploy. No framework, no build command, output is the root.
2. **Direct upload** — drag this folder onto vercel.com.

The service worker needs HTTPS, which Vercel provides automatically.

## Local preview

Open `index.html` directly in a browser, or run a static server:

```
python3 -m http.server 8080
```

then visit `http://localhost:8080`.
