# WockingTocking — Trip App

A mobile-first web app for the WockingTocking vlog road trips. Open it on a
phone in Safari and "Add to Home Screen" to use it like a native app.

Live: <https://vlog-planner.vercel.app>

## What's inside

Four tabs:

- **Trip** — overview of the active trip: the three-day arc, kit, pre-trip
  checklist, and three Act cards. Tap an Act for the full chapter story.
- **Route** — tap-to-navigate Google Maps and Waze links; walking loops inside
  each town.
- **Plan** — the timed agenda for each day, with film cues.
- **Talk** — what to say on camera at each stop: a short story plus quick facts.

Tap the trip title in the header to switch between trips.

## Adding a new trip

1. Copy `trips/ep01.json` to `trips/<id>.json` and edit the content.
2. Append an entry to `trips/index.json`:
   ```json
   { "id": "ep02", "title": "…", "subtitle": "…", "dates": "…", "episode": "EP.02" }
   ```
3. In `sw.js`, add the new JSON path to the `SHELL` array and bump `CACHE` to a
   new version (e.g. `wt-v3`) so the service worker re-precaches.
4. Push to `main`. Vercel rebuilds automatically.

## Files

| File | Purpose |
|------|---------|
| `index.html` | App shell, styles, render code |
| `trips/index.json` | List of trips for the header selector |
| `trips/<id>.json` | All content for one trip |
| `sw.js` | Offline cache (precaches shell + trip JSON) |
| `manifest.json` | Web app manifest |
| Icons | App icons for Add-to-Home-Screen |

No build step, no dependencies.

## Local preview

```
python3 -m http.server 8080
```

Then visit `http://localhost:8080`. Opening `index.html` directly via `file://`
will not work because the app uses `fetch` to load JSON.

## Spec & plan

Design and implementation notes live in `docs/superpowers/`.
