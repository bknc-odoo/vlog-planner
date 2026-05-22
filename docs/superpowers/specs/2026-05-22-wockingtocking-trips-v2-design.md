# WockingTocking — Trips v2 Design

**Date:** 2026-05-22
**Author:** Ivan + Claude (brainstorming)
**Status:** Approved for implementation

## Goal

Turn the single-trip Episode 01 app into a small **trip platform**: support multiple trips with a trip selector, give each Act on the Trip tab a full-story drill-down, and enrich the Talk tab with a short narrative story plus quick facts per stop.

The app stays a static, zero-build, mobile-first PWA deployed on Vercel.

## Non-goals

- No backend, auth, or sync — content lives as static JSON in the repo.
- No framework (no React, Vue, etc.) — vanilla HTML/CSS/JS.
- No build step — Vercel serves the repo as-is.
- No editing UI — adding/editing trips is editing JSON in the repo.

## Architecture overview

Today: `index.html` contains structure, styling, and *all Ep 01 content* inline.

After: `index.html` is a thin shell + render code; **trip content lives in `trips/<id>.json`**. The app fetches the active trip on load and re-renders all four tabs from it.

```
wockingtocking-app/
  index.html            # shell, styles, render code, no per-trip content
  sw.js                 # precache shell + every trips/*.json
  manifest.json
  apple-touch-icon.png  icon-192.png  icon-512.png
  trips/
    index.json          # list of trips + default
    ep01.json           # full content for Episode 01
    ep02.json           # (later)
```

### Why this shape

- **One trip = one file.** Adding Ep 02 means dropping a JSON in `trips/` and appending an entry to `trips/index.json`. No code change.
- **Production target is Vercel + iOS Add-to-Home-Screen.** Both serve over HTTPS, so `fetch()` of relative paths works cleanly. Local preview already uses `python3 -m http.server`; the README change is one line.
- **Service worker can precache JSON,** so the app stays fully offline once visited.

## Data model

### `trips/index.json`

```json
{
  "default": "ep01",
  "trips": [
    {
      "id": "ep01",
      "title": "Essex to Yorkshire",
      "subtitle": "Little Yeldham → Pudsey",
      "dates": "Sat 23 – Mon 25 May 2026",
      "episode": "EP.01"
    }
  ]
}
```

### `trips/<id>.json`

One file containing every tab's content. Keys:

- `id`, `title`, `subtitle`, `dates`, `episode` — header + Trip tab hero.
- `intro` — the lede paragraph on the Trip tab.
- `acts` — array of 3 acts:
  ```json
  {
    "id": "act1",
    "label": "Act 1 · The journey north",
    "color": "rust",            // one of: rust | ochre | forest
    "tagline": "The old towns. Each stop a little more wow than the last.",
    "narrative": "Long-form story for the overlay — 2–3 paragraphs.",
    "stopIds": ["lavenham", "stamford", "lincoln"]
  }
  ```
- `days` — array of 3 days; each has `id`, `label`, `date`, `actId`, `route[]`, `plan[]`.
  - `route[]` items are the existing card structure: `{ name, sub, mapsLink, wazeLink, walkLinks? }`.
  - `plan[]` items are time-block cards: `{ time, label, leaveBy?, notes? }`.
- `stops` — object keyed by stop id:
  ```json
  "lavenham": {
    "name": "Lavenham",
    "actId": "act1",
    "angle": "A perfect medieval town",
    "story": "1–2 paragraph narrative in 'telling a mate' tone.",
    "facts": ["the timber was used unseasoned…", "…", "…"],
    "mapsLink": "https://www.google.com/maps/…",
    "wazeLink": "https://waze.com/ul?…"
  }
  ```
- `coldOpen`, `signOff` — `{ where, facts[] }` — the existing top/bottom Talk cards.
- `kit`, `preTrip`, `goldenRules` — string arrays for the Trip tab and Talk tab lists.

Field types are stable; render code treats unknown extra keys as ignored.

## UI changes

### Header

- Watermark `WOCKINGTOCKING` stays.
- New: tappable trip title in the header center — `Essex → Yorkshire ▾`.
- Episode chip moves to the right side — `EP.01`.
- Tap → **bottom sheet** with the trip list from `trips/index.json`. Tap a trip → app re-renders.

### Trip tab — Acts become drill-downs

- The three Act cards (rust / ochre / forest) gain a `›` chevron and are fully tappable.
- Tap → **full-screen overlay** slides up:
  - Sticky top: `‹ Back` + Act label in its theme color.
  - Hero: large Act title + tagline.
  - Body: the `narrative` paragraphs.
  - "Stops in this act" section — each stop is a row with `Talk →` and `Route →` quick-links that close the overlay and jump to that stop on the relevant tab.
- Overlay scrolls independently. Back button (in-overlay) and browser back both close it via a `#act/<id>` hash.

### Talk tab — Story + Facts per stop

Per-stop card layout:

```
┌──────────────────────────────────────┐
│ Lavenham                             │   cap (Fraunces 16px, act color)
│ ANGLE · A PERFECT MEDIEVAL TOWN      │   sub
│                                      │
│ THE STORY                            │   mlabel
│ Six hundred years ago Lavenham was   │
│ one of the richest towns in England… │   serif paragraph, 14.5px
│                                      │
│ QUICK FACTS                          │   mlabel
│ • the timber was used unseasoned…    │
│ • De Vere House = Godric's Hollow    │
│ • …                                  │
└──────────────────────────────────────┘
```

- **Story** in Fraunces serif, slightly larger and with a touch more line-height than the existing bullets — reads like a magazine paragraph.
- **Facts** keep the existing tiny-rhombus bullet style — the "glance down while filming" affordance from today.

### Route / Plan tabs

No visual changes. Content is rendered from the JSON instead of being hard-coded in HTML.

### Tab bar

Unchanged. Trip / Route / Plan / Talk.

## Rendering & state

### Render pipeline (vanilla JS, no framework)

On load:
1. `fetch('trips/index.json')` → populate selector, pick active trip (`localStorage.wt_trip` if valid, else `default`).
2. `fetch('trips/<active>.json')` → store in module-level `TRIP`.
3. Call four pure render functions — `renderTrip(TRIP)`, `renderRoute(TRIP)`, `renderPlan(TRIP)`, `renderTalk(TRIP)` — each produces the inner HTML of its `<section class="tab">`.
4. Wire event handlers: tab buttons, trip-selector button, act cards, overlay back button, hash router.

On trip switch:
1. `fetch('trips/<newId>.json')` → reassign `TRIP`.
2. Re-run the four render functions.
3. Scroll to top, persist `localStorage.wt_trip`.

Templates are plain string templates (`.map().join('')` and tagged-template helpers if useful), kept readable. No bundler, no transpile.

### `localStorage` keys

- `wt_trip` — active trip id.
- `wt_tab` — active tab (already exists).
- `wt_check` — checklist tick state, keyed by trip id (e.g. `{ ep01: { 0: true, 2: true } }`) — fixes today's behavior where ticks reset on refresh.

### Hash routing

One `hashchange` listener, no router lib. Recognised hashes:

- `#trip/<tripId>` — switch to that trip on the Trip tab.
- `#tab/<tabName>` — open a tab (`trip` | `route` | `plan` | `talk`).
- `#talk/<stopId>` — open Talk tab and scroll the named stop into view.
- `#act/<actId>` — open the act overlay for that act on the current trip.

Hashes are progressive: shareable URLs, browser-back behaves naturally, swipe-back on iOS closes the overlay.

### Service worker

- Precache list expands to include `trips/index.json` and every `trips/<id>.json` that exists.
- Strategy: **cache-first for the shell**, **stale-while-revalidate for trip JSON** — so edits land on next visit without a hard refresh.
- Bump `CACHE_NAME` on every release. Existing `sw.js` already follows this pattern.

## Ep 01 content port

All Ep 01 content moves from `index.html` into `trips/ep01.json`. New content authored as part of this work:

- **3 Act narratives** — 2–3 paragraphs each, painting the arc of the day.
- **6 Stop stories** — 1–2 paragraphs each, in "telling a mate" tone, grounded in well-known facts:
  - **Lavenham** — wool wealth → unseasoned timber → De Vere / Godric's Hollow.
  - **Stamford** — honey limestone harmony → first Conservation Area (1967) → Pride & Prejudice (2005).
  - **Lincoln** — Steep Hill → cathedral spire as the tallest building on Earth for 238 years until 1548 → 1215 Magna Carta in the castle.
  - **York** — solo reflective tone → Shambles / Diagon Alley → most complete city walls in England.
  - **Bakewell** — pudding vs tart, the 1820 accident → on-camera taste reaction.
  - **Castleton & Winnats Pass** — limestone formed in a tropical sea → Mam Tor "Shivering Mountain", road abandoned 1979 → Blue John found nowhere else.

Existing 3-bullet `facts` lists are kept verbatim from today's Talk page so the on-camera moments don't change.

Cold open and Sign-off stay as fact lists (no story needed — they're already script-shaped).

## Error handling

- Missing `trips/index.json` → render an inline error block in the header area: "Trips list failed to load — check your connection and refresh."
- Missing `trips/<id>.json` for the requested id → fall back to `default` from index; if that also fails, show the same error block.
- Unknown hash → ignore, render default state.
- Malformed JSON → caught at parse, surfaced as the same inline error so it isn't a silent blank screen on iOS.

## Testing

This is a static app with no test framework today; testing is manual and pragmatic:

- **Smoke test on iPhone Safari:** open the deployed URL, switch between all four tabs, tap each act card, switch the trip selector (once Ep 02 stub exists), refresh while offline (PWA installed) to confirm cached load.
- **Smoke test on desktop:** Chrome DevTools, network throttled to Offline, confirm SW serves cached JSON.
- **Content sanity:** every `stopId` referenced in `acts[].stopIds` exists in `stops`; every `actId` referenced in `days[]` exists in `acts`. A small `validateTrip(trip)` function runs after fetch and `console.warn`s mismatches in dev.

## Deployment

No change — Vercel auto-deploys on push to `main`. Production URL stays `https://vlog-planner.vercel.app`.

## Out of scope (future)

- An admin UI for editing trips in the browser.
- Photos per stop.
- Weather widget per day.
- Cross-trip search.
