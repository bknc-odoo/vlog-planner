# WockingTocking — Trips v2 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:subagent-driven-development` (recommended) or `superpowers:executing-plans` to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Turn the single-trip Episode 01 app into a multi-trip platform with a header trip selector, full-story Act drill-down overlays, and richer Talk cards with story + facts per stop.

**Architecture:** Refactor `index.html` from content-baked-in to a thin shell + render code. Trip content lives in `trips/<id>.json`, fetched at runtime. Service worker precaches all trip JSON for offline. No framework, no build step.

**Tech Stack:** Vanilla HTML / CSS / JS, `fetch`, Service Worker, `localStorage`. Deployed as static on Vercel.

**Spec:** `docs/superpowers/specs/2026-05-22-wockingtocking-trips-v2-design.md`.

**Security note (innerHTML usage):** The render code uses `element.innerHTML = ...` to compose tab content. All dynamic text fields from the JSON go through an `esc()` helper that HTML-escapes `& < > " '`. The JSON files live in this repo (no remote content, no user input), so this is defense-in-depth. If the app ever accepts external content, swap `innerHTML` for explicit DOM construction or a sanitizer like DOMPurify.

---

## Testing approach

This is a static HTML app with no test framework. Each task's "verification" step is a **manual smoke test** in a real browser via the local server:

```bash
cd wockingtocking-app
python3 -m http.server 8080
# Open http://localhost:8080 in a browser
```

For DevTools-driven checks (service worker, offline, network), use Chrome DevTools → Application / Network panels.

Each task ends with a commit. Keep commits small.

---

## File structure after this plan

```
wockingtocking-app/
  index.html              # shell, styles, render code (no per-trip content)
  sw.js                   # caches trips/*.json
  manifest.json
  icons / apple-touch-icon
  README.md
  trips/
    index.json            # [{id,title,subtitle,dates,episode}], default
    ep01.json             # full Episode 01 content
```

---

## Task 1: Scaffold `trips/` with `index.json` and `ep01.json` (verbatim port)

Move existing Ep 01 content out of `index.html` into JSON. No code change yet — the JSON files just sit alongside.

**Files:**
- Create: `trips/index.json`
- Create: `trips/ep01.json`

### Step 1: Create `trips/index.json`

- [ ] **Create `trips/index.json`** with this content:

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

### Step 2: Create `trips/ep01.json` (stories empty; filled in Task 3)

- [ ] **Create `trips/ep01.json`** organised per the spec data model. Top-level keys: `id`, `title`, `subtitle`, `dates`, `episode`, `heroKicker`, `intro`, `dayGlance[]`, `acts[]` (3), `kit[]`, `preTrip[]`, `goldenRule`, `days[]` (3), `stops{}` (lavenham, stamford, lincoln, pudsey, york, bakewell, castleton, home), `coldOpen{}`, `signOff{}`, `talkTipbox`, `goldenRules[]`, `outro`.

The full content for this file is in the spec's "Data model" section and in the existing `index.html` (current Ep 01 content lifted verbatim). Each act has `narrative: ""` and each stop has `story: ""` — Task 3 fills those in. `pudsey` and `home` get `hideOnTalk: true` so they only appear on Route, not Talk.

Use the exact field shapes shown in the spec. Important details to preserve from the existing `index.html`:

- Each stop's `routeCard` carries `sub`, `mapsHref`, `wazeHref`, optional `note`, `walkLeadNote`, `trailingNote`, `walkPoints[]` (n / name / pinHref), `walkLoopHref`.
- Each stop's `planCard` carries `time`, `leaveBy` (nullable), `see[]`, `film[]`. `null` if not in the plan.
- Each `day` has `id` (d1/d2/d3), `label`, `date`, `color`, `routeHeader`, `fullRoute` (label/sub/href), `planHeader`, `planNote` (nullable: kicker/color/text), `routeStops[]`, `planStops[]`, `planArrival` (title/note).
- `coldOpen` and `signOff` are `{title, color, where, facts[]}`.

### Step 3: Verify JSON parses

- [ ] **Validate both JSON files parse**

```bash
python3 -c "import json; json.load(open('trips/index.json')); json.load(open('trips/ep01.json')); print('OK')"
```

Expected: `OK`

### Step 4: Commit

- [ ] **Commit**

```bash
git add trips/
git commit -m "Add trips/ JSON: index + ep01 content port (no code change yet)"
```

---

## Task 2: Refactor `index.html` to render from JSON (visual no-op)

Strip hard-coded content from `index.html`. Add render functions that produce equivalent HTML from the JSON. After this task the app must look identical.

**Files:**
- Modify: `index.html`

### Step 1: Empty the four tab section bodies

- [ ] In `index.html`, replace the body of each of the four sections so they each look like:

```html
    <section class="tab active" id="trip"></section>
    <section class="tab" id="route"></section>
    <section class="tab" id="plan"></section>
    <section class="tab" id="talk"></section>
```

(Only the `class` differs — `trip` keeps `active`. Everything between the tags is now empty; render code fills them at runtime.)

### Step 2: Replace the existing inline `<script>` block

- [ ] At the bottom of `index.html`, find the existing IIFE script and replace it with the new shell + renderers. The new script:

1. Declares `TRIP`, `TRIPS`, `ACTIVE` module-level vars.
2. Defines helpers: `esc(s)` (HTML-escape), `attr(s)` (same), `colorVar(c)` returns `var(--rust|ochre|forest)`, `stop(id)` returns `TRIP.stops[id]`.
3. Defines `showTab(name)` (existing behavior, persists `wt_tab`).
4. Defines `renderTrip()`, `renderRoute()`, `renderPlan()`, `renderTalk()` — each builds a string of HTML and assigns it to its section's `innerHTML`. Every interpolated text value passes through `esc()`. Links and structural HTML are static in the template.
5. Defines `renderAll()` calling all four.
6. Defines `loadTrip(id)` → `fetch('trips/<id>.json')` → assign to `TRIP` and `ACTIVE`.
7. Defines `boot()` → fetch `trips/index.json`, pick active from `localStorage.wt_trip` (validated against the list) or `index.default`, call `loadTrip` then `renderAll`, then `wireTabs`, then restore `wt_tab`.
8. On any fetch failure, replace `<main>` content with an inline error block: `<div style="…">Trips failed to load. Refresh when you have a connection.</div>` — must use `esc(err.message)` for the detail.
9. Existing service worker registration stays.

The render functions reproduce the markup currently in `index.html` byte-for-byte where text is concerned:

- **`renderTrip`** produces: `.hero` block, `.stripe`, intro `<p class="p0">`, "The three days" `h2` + lede + three `.glance` rows from `t.dayGlance`, "The story to film" `h2` + lede + three `.card.act-card[data-act=<id>]` from `t.acts` (use `colorVar(a.color)` for the title color), "Your kit" `h2` + `<ul>` from `t.kit`, "Before you go" `h2` + `.ck` rows from `t.preTrip`, and the golden-rule tipbox.
- **`renderRoute`** loops `t.days`, producing the `.day.d<n>` block (set `style="--c:var(--rust|ochre|forest)"` from `day.color`), the `.dh` header, the `<a class="full">` full-route link, then a `routeCardHTML(stop(sid))` for each id in `day.routeStops`.
- **`routeCardHTML(stopObj)`** builds the `.card.ac` with `.cap`, `.sub`, optional `.note`, `.btnrow` (Google Maps + Waze), and the `.walk` block — including `walkLeadNote`, `walkPoints` rows, `walkLoopHref` button, `trailingNote`. Omit the `.walk` block if `walkPoints` is empty.
- **`renderPlan`** loops `t.days`, producing `.day.d<n>` header + optional `.tipbox` from `day.planNote` + `planCardHTML(stop(sid))` for each id in `day.planStops` + arrival `.card.ac` from `day.planArrival`.
- **`planCardHTML(stopObj)`** builds `.card` with `.cap` (name), time `.chip.t`, optional `LEAVE BY` `.chip.lv`, "See" `mlabel` + `<ul>`, "Film" `mlabel` + `<ul>` (each `<li class="f">`).
- **`renderTalk`** writes the heading, lede, `On camera` tipbox using `t.talkTipbox`, then the cold-open bookend card, then for each act in `t.acts` and each `sid` in `act.stopIds`, a `talkCardHTML(stop(sid))`, then the sign-off bookend, then the "Golden rules" heading + list from `t.goldenRules`, then `t.outro` as a `.note` paragraph.
- **`talkCardHTML(stopObj)`** (Task 2 minimal version — Task 4 enriches it): `.card` with `.cap` colored by the act's color, `.sub` ("Angle: …"), then `<ul>` of `stopObj.facts`. Skip the card if `stopObj.hideOnTalk` is true.
- **`bookendCardHTML(b)`** (`b` = `t.coldOpen` or `t.signOff`): `.card` with `.cap` colored by `b.color`, `.sub` showing `b.where`, `<ul>` of `b.facts`.

The template strings should use string concatenation or `[].join('')` (no tagged templates needed). Each `+ '<div…>'` segment that contains a JSON value wraps it in `esc(…)`.

### Step 3: Verify visual parity

- [ ] **Start the dev server:**

```bash
cd wockingtocking-app
python3 -m http.server 8080
```

- [ ] **Open `http://localhost:8080`** in a browser.

Expected: the page looks **identical** to before this task on all four tabs. If empty or wrong, check the DevTools Console for parse errors and verify each JSON field name matches the render code.

### Step 4: Commit

- [ ] **Commit**

```bash
git add index.html
git commit -m "Refactor index.html: render Trip/Route/Plan/Talk from trips/<id>.json"
```

---

## Task 3: Add Act narratives and Stop stories to `ep01.json`

Pure content edit. Fill in the empty `narrative` on each act and `story` on each stop (except `pudsey` and `home`, which are `hideOnTalk: true`).

**Files:**
- Modify: `trips/ep01.json`

### Step 1: Act narratives

Each is one paragraph, written in the "telling a mate" voice already used on the Talk page.

- [ ] **`acts[0].narrative`** (Act 1):

> You start in your own driveway, in a sleepy Essex village no one has heard of, and within an hour you're standing in Lavenham — a town so wonky and so old that it feels staged. From there it just keeps building. Stamford turns the dial up: a whole town carved from one honey-coloured stone, every house in conversation with the next. Then Lincoln, the headliner — a cathedral that was, for two and a half centuries, the tallest thing humans had ever built. The pattern of the day is the pattern of the trip: each stop should feel a little bigger, a little more 'wait, really?' than the one before. By the time you roll into Pudsey in the early evening, you've travelled five hundred years of England in a single day, and the camera has the proof.

- [ ] **`acts[1].narrative`** (Act 2):

> Day two is the change of register. After a Saturday spent reacting to one wow after another with your mate's voice off-camera, York is just you and a walled city. That solitude is the story — not a sad one, just a calmer one. You walk the most complete medieval walls in England with no one to perform for. You wander into the Shambles, the narrow lane that whispered the idea of Diagon Alley into J.K. Rowling's head. Somewhere quiet — Museum Gardens by the river works — you turn the camera on yourself and say, honestly, how the trip is going. That short piece is the heart of the episode. Then you point the car south, because the Peak District is waiting and you want to be there before the light goes.

- [ ] **`acts[2].narrative`** (Act 3):

> The third day rewinds the clock by a few hundred million years. You've spent two days in places where humans piled stone on stone; now you stand on stone that used to be a tropical sea. Winnats Pass is a limestone gorge a car can drive through, formed underwater long before England existed. Above it sits Mam Tor, the Shivering Mountain — the only A-road in Britain to ever be defeated by a hill, abandoned in 1979 because the slope kept slipping. From here you wind down to Bakewell, taste a pudding that exists because a cook got a recipe wrong in 1820, and then you point the car south for the long quiet drive home. Somewhere on a viewpoint or back in the village, you say the last line of Episode 01 on camera, and that is your wrap.

### Step 2: Stop stories

- [ ] **`stops.lavenham.story`:**

> Six hundred years ago Lavenham was one of the richest towns in England, and all of it was built on sheep. The wool from East Anglian farms went out through this market and the merchants who handled it built houses that screamed money — three storeys of oak, carved gables, the lot. The catch: they were in such a hurry to flaunt it, they used the timber before it was properly seasoned, and as the wood dried over the centuries it warped. That's why every wall here leans. The whole village is a wonky monument to medieval cash flow. The other thing to know — De Vere House, the timber-framed one on Water Street, was Godric's Hollow in Harry Potter, so you're literally walking through Harry's childhood front garden.

- [ ] **`stops.stamford.story`:**

> After Lavenham's leaning timbers, Stamford is the same idea in stone. Every wall, every roof, every garden gate is cut from the same honey-coloured Lincolnshire limestone — quarried just up the road — and the result is a town that looks like one continuous, harmonious composition. That harmony is so striking that in 1967 Stamford became the first ever Conservation Area in England — the place that invented the idea that towns themselves are worth protecting. It feels like a film set, and it is one: this is where they shot most of the 2005 Pride & Prejudice. You can walk through Mr Bingley's Meryton in twenty minutes, eat lunch, and be back in the car.

- [ ] **`stops.lincoln.story`:**

> Lincoln is the headliner of day one, and it earns it twice — once for the climb and once for the building at the top. Steep Hill is the spine of the old city; you grind up it past medieval houses and timbered shopfronts, and when you crest the top the cathedral fills the sky. Here's the line on camera: from the year 1311 until 1548, Lincoln Cathedral's spire was the tallest building in the world. For two hundred and thirty-seven years humans had built nothing taller. Then in 1548 the spire collapsed in a storm and the world had to wait three hundred more years for anything to overtake it. Next door, the castle holds one of only four surviving original copies of the 1215 Magna Carta — the actual sheet of vellum that ended the absolute power of English kings. Bring a wide lens and a moment of awe.

- [ ] **`stops.york.story`:**

> York is the city of day two and the quiet part of the episode. The walls are the move — almost three miles of medieval defences, the most complete city walls left in England, and the path runs straight along the top of them. You go up at Bootham Bar, walk a stretch overlooking the Minster, and the city lays itself out below you. The Shambles is the other one to film: a narrow cobbled lane of crooked timber-framed shops that has been a street of butchers since before America was a country, and is widely said to have inspired Diagon Alley — J.K. Rowling studied near here. The thing to remember on camera is that this is the solo day. Drop the tour-guide voice; somewhere quiet by the river, look into the lens and say honestly how the trip is going. That single clip will land harder than any wide shot.

- [ ] **`stops.bakewell.story`:**

> The story of Bakewell is the story of a cook who got it wrong. The legend goes that in 1820 the landlady of the White Horse Inn asked her cook to make a strawberry tart, and the cook misread the recipe — instead of stirring the egg-and-almond mixture into the pastry, she poured it on top of the jam. The customers loved the mistake and a thousand bakeries got into a slow-motion fight about what to call it. The original is the Bakewell **pudding** — soft, slightly wobbly, a hot mess. The neater, almond-iced disc you find in supermarkets is the Bakewell **tart** — a Victorian remix. Both exist; the pudding is the one you came for. Buy one, bite it on camera, react honestly. The mistake is the point.

- [ ] **`stops.castleton.story`:**

> Castleton flips the timescale. You've spent two days admiring things humans built; now you're standing on rock that built itself, underwater, three hundred and fifty million years ago. The Peak's limestone is fossilised tropical sea floor — the corals, the shells, all still in there. Winnats Pass is the gorge cut through it, a single road squeezed between two limestone walls, dramatic enough that drone shots feel illegal. Above the pass, Mam Tor has its own story: nicknamed the Shivering Mountain because its softer shale layers keep slipping, it defeated an A-road. The old A625 across its eastern face had to be repaired seven times in twenty years and was finally abandoned in 1979 — you can still walk the broken tarmac, fenced off, weirdly cinematic. And one last thing only Castleton has: Blue John, a violet-banded stone mined here and literally nowhere else on Earth. Buy a tiny piece; it's a souvenir with a postcode.

### Step 3: Validate JSON

- [ ] **Validate**

```bash
python3 -c "import json; json.load(open('trips/ep01.json')); print('OK')"
```

Expected: `OK`

### Step 4: Commit

- [ ] **Commit**

```bash
git add trips/ep01.json
git commit -m "Add Act narratives and Stop stories to Episode 01"
```

---

## Task 4: Talk tab — Story + Facts layout

Update the Talk card to render the story paragraphs above the facts list.

**Files:**
- Modify: `index.html`

### Step 1: CSS for the story block

- [ ] In `index.html`'s `<style>` block, add near the end (before `</style>`):

```css
  .story{font-family:'Fraunces',serif;font-weight:400;font-size:14.5px;line-height:1.55;
         margin-top:8px;color:var(--ink)}
  .story p{margin-bottom:8px}
  .story p:last-child{margin-bottom:0}
```

### Step 2: Rewrite `talkCardHTML` to include story + scrollable id

The new version:

- Skips if `hideOnTalk`.
- Computes the act color via `colorVar`.
- Splits `stopObj.story` on blank lines into `<p>` paragraphs (each escaped via `esc`).
- Adds a small `THE STORY` `.mlabel` (using the act color) and `.story` block when a story exists.
- Adds a `QUICK FACTS` `.mlabel` (same color, margin-top:14px) before the existing facts `<ul>`.
- The outer `.card` gets `id="talk-<actId>-<name-slug>"` (lowercased name, non-letters replaced with `-`) so Task 7's `#talk/<stopId>` routing can scroll it into view.

### Step 3: Verify

- [ ] Reload `http://localhost:8080`, switch to Talk tab.

Expected:
- Lavenham, Stamford, Lincoln, York, Bakewell, Castleton each show a Fraunces serif story paragraph under a `THE STORY` label, then a `QUICK FACTS` label and the existing bullet list (rust / ochre / forest by act).
- Cold open and Sign-off look unchanged.
- Pudsey and Home still hidden on Talk.

### Step 4: Commit

- [ ] **Commit**

```bash
git add index.html
git commit -m "Talk tab: render story + facts per stop"
```

---

## Task 5: Trip tab — Act overlay (full-screen drill-down)

Make Act cards tappable. Tapping opens a full-screen overlay with the act narrative and clickable stop links. Routes through `location.hash = '#act/<id>'` so browser back / iOS swipe-back close it.

**Files:**
- Modify: `index.html`

### Step 1: Add the overlay markup

- [ ] In `index.html`, right after the `<nav class="tabs">` block (before the closing `</div>` of `.app`), add:

```html
  <div class="overlay" id="actOverlay" aria-hidden="true">
    <div class="overlay-bar">
      <button class="overlay-back" id="overlayBack" aria-label="Back">‹ Back</button>
      <span class="overlay-act" id="overlayActLabel"></span>
    </div>
    <div class="overlay-body" id="overlayBody"></div>
  </div>
```

### Step 2: CSS for tappable act cards + overlay

- [ ] Add near the end of the `<style>` block:

```css
  .card.act-card{cursor:pointer;position:relative;padding-right:32px}
  .card.act-card::after{content:"›";position:absolute;right:13px;top:50%;
        transform:translateY(-50%);font-family:'Fraunces',serif;font-size:24px;
        line-height:1;color:var(--soft)}
  .card.act-card:active{transform:translateY(1px)}

  .overlay{position:fixed;inset:0;z-index:40;background:var(--paper);
           transform:translateY(100%);transition:transform .28s ease;
           overflow-y:auto;-webkit-overflow-scrolling:touch;
           padding-bottom:env(safe-area-inset-bottom)}
  .overlay.on{transform:translateY(0)}
  .overlay-bar{position:sticky;top:0;background:var(--ink);color:var(--paper);
        padding:calc(12px + env(safe-area-inset-top)) 18px 12px;
        display:flex;align-items:center;gap:12px;
        border-bottom:3px solid var(--c,var(--rust));z-index:1}
  .overlay-back{background:none;border:none;color:var(--paper);cursor:pointer;
        font-family:'Space Mono',monospace;font-size:12px;font-weight:700;
        letter-spacing:.06em;padding:6px 4px}
  .overlay-act{font-family:'Space Mono',monospace;font-size:10.5px;letter-spacing:.18em;
        text-transform:uppercase;color:var(--c,var(--rust))}
  .overlay-body{padding:18px 16px 28px;max-width:600px;margin:0 auto}
  .overlay-body .act-hero h1{font-family:'Fraunces',serif;font-weight:900;font-size:30px;
        line-height:1.02;color:var(--c,var(--rust))}
  .overlay-body .act-hero p{font-family:'Fraunces',serif;font-style:italic;font-size:15px;
        color:var(--soft);margin-top:6px}
  .overlay-body .narrative{font-family:'Fraunces',serif;font-weight:400;font-size:15px;
        line-height:1.6;margin-top:14px}
  .overlay-body .narrative p{margin-bottom:11px}
  .stop-link{display:block;border:1.5px solid var(--ink);background:var(--card);
        padding:11px 13px;margin-bottom:8px;text-decoration:none;color:var(--ink)}
  .stop-link .sn{font-family:'Fraunces',serif;font-weight:900;font-size:16px;color:var(--c,var(--rust))}
  .stop-link .sa{font-family:'Space Mono',monospace;font-size:9.5px;letter-spacing:.09em;
        text-transform:uppercase;color:var(--soft);margin-top:2px}
  .stop-link .row{margin-top:8px;display:flex;gap:7px}
  .stop-link .row a{flex:1;text-align:center;text-decoration:none;
        font-family:'Space Mono',monospace;font-size:11px;font-weight:700;
        letter-spacing:.04em;padding:7px;border:1.5px solid var(--c,var(--rust));
        color:var(--c,var(--rust));background:transparent}
  body.locked{overflow:hidden}
```

### Step 3: Wire `openAct` / `closeAct` / `wireOverlay` and the temporary hash router

In the `<script>` block, add (after `wireTabs`):

- **`openAct(actId)`**: finds `act` in `TRIP.acts`. Sets the overlay bar's `--c` CSS var to `colorVar(act.color)`. Sets the label's text. Splits `act.narrative` on blank lines into `<p>` paragraphs (each escaped). For each `sid` in `act.stopIds`, renders a `.stop-link` with `.sn` (name), `.sa` (angle), and a `.row` with two anchor children: `Talk →` linking to `#talk/<sid>` and `Route →` linking to `#route`. Assigns the composed HTML to `overlayBody.innerHTML`. Adds `.on` class to overlay, `aria-hidden="false"`, `body.classList.add('locked')`.
- **`closeAct()`**: removes `.on`, sets `aria-hidden="true"`, removes `locked`. If `location.hash` starts with `#act/`, `history.replaceState(null, '', location.pathname + location.search)` to strip it.
- **`wireOverlay()`**: hooks `#overlayBack` click → sets `location.hash = ''` (which triggers `applyHash` to close). Adds a delegated click handler on `#trip`: if the click target's `.closest('.act-card')` is non-null, read `dataset.act` and set `location.hash = '#act/<id>'` — routing through the hash so back-button works.

Add a minimal hash router for this task:

- **`applyHash()`**: if hash starts with `#act/`, call `openAct(hash.slice(5))`. Otherwise, if the overlay is on, hide it (same as the body of `closeAct` minus the history call).
- **`wireHash()`**: registers `window.addEventListener('hashchange', applyHash)` and calls `applyHash()` once at boot.

In `boot()`, after `wireTabs();` add `wireOverlay();` and `wireHash();`.

### Step 4: Verify

- [ ] Reload, open Trip tab, tap each Act card.

Expected:
- Card shows a `›` chevron at the right.
- Tapping slides up a full-screen panel: colored top bar, act label, large title + tagline (italic), narrative paragraphs (Fraunces serif), "Stops in this act" with one row per stop.
- `‹ Back` closes the overlay. Browser back also closes.
- Tapping `Talk →` on a stop link closes the overlay and navigates to `#talk/<sid>` — for now this just changes the hash; scrolling-to-stop comes in Task 7.

### Step 5: Commit

- [ ] **Commit**

```bash
git add index.html
git commit -m "Trip tab: full-screen Act drill-down overlay"
```

---

## Task 6: Header trip selector (bottom sheet)

Replace the static `Episode 01` chip with a tappable trip title + episode chip. Tap the title → bottom sheet listing all trips.

**Files:**
- Modify: `index.html`

### Step 1: Restructure the `<header>` markup

- [ ] Replace the existing `<header>` with:

```html
  <header>
    <span class="wm">WockingTocking</span>
    <button class="trip-pick" id="tripPick" aria-haspopup="true">
      <span class="trip-pick-title" id="tripPickTitle">…</span>
      <span class="trip-pick-caret">▾</span>
    </button>
    <span class="ep" id="tripPickEp">EP.01</span>
  </header>

  <div class="sheet-backdrop" id="sheetBackdrop" hidden></div>
  <div class="sheet" id="tripSheet" aria-hidden="true">
    <div class="sheet-handle"></div>
    <h3 class="sheet-title">Choose a trip</h3>
    <div class="sheet-list" id="sheetList"></div>
  </div>
```

### Step 2: CSS for the picker + sheet

- [ ] Add to the `<style>` block:

```css
  header{justify-content:flex-start;gap:10px}
  .trip-pick{flex:1;min-width:0;background:none;border:none;color:var(--paper);
        cursor:pointer;padding:0;display:flex;align-items:baseline;gap:6px;
        font-family:'Fraunces',serif;font-weight:900;font-size:15px;
        letter-spacing:-.01em;overflow:hidden;text-align:left}
  .trip-pick-title{overflow:hidden;text-overflow:ellipsis;white-space:nowrap}
  .trip-pick-caret{font-size:12px;color:#c8a86f;flex:none}
  header .ep{flex:none}

  .sheet-backdrop{position:fixed;inset:0;z-index:45;background:rgba(43,39,34,.55);
        opacity:0;transition:opacity .22s ease;pointer-events:none}
  .sheet-backdrop.on{opacity:1;pointer-events:auto}
  .sheet{position:fixed;left:0;right:0;bottom:0;z-index:46;
        background:var(--paper);border-top-left-radius:14px;border-top-right-radius:14px;
        transform:translateY(100%);transition:transform .25s ease;
        padding:8px 16px calc(20px + env(safe-area-inset-bottom));
        max-width:600px;margin:0 auto;max-height:70vh;overflow-y:auto}
  .sheet.on{transform:translateY(0)}
  .sheet-handle{width:38px;height:4px;background:var(--line);border-radius:2px;
        margin:6px auto 12px}
  .sheet-title{font-family:'Space Mono',monospace;font-size:10.5px;letter-spacing:.18em;
        text-transform:uppercase;color:var(--soft);margin-bottom:8px}
  .sheet-trip{display:block;width:100%;text-align:left;background:var(--card);
        border:1.5px solid var(--ink);padding:12px 14px;margin-bottom:8px;
        cursor:pointer;font:inherit;color:inherit}
  .sheet-trip.active{box-shadow:3px 3px 0 var(--rust)}
  .sheet-trip-title{font-family:'Fraunces',serif;font-weight:900;font-size:17px}
  .sheet-trip-sub{font-family:'Space Mono',monospace;font-size:10px;letter-spacing:.08em;
        text-transform:uppercase;color:var(--soft);margin-top:4px}
  .sheet-trip-dates{font-size:12.5px;color:var(--soft);margin-top:3px}
```

### Step 3: Wire the trip picker

Add to the `<script>` block:

- **`renderHeader()`**: looks up `ACTIVE` in `TRIPS.trips`, sets `#tripPickTitle.textContent = meta.title` and `#tripPickEp.textContent = meta.episode || ''`.
- **`openSheet()`**: composes `#sheetList` `innerHTML` from `TRIPS.trips`, each as a `<button class="sheet-trip[ active]" data-trip="<id>">` with `.sheet-trip-title` (escaped title), `.sheet-trip-sub` (episode + " · " + subtitle), `.sheet-trip-dates`. Reveals the backdrop and sheet (`hidden=false`, then add `.on` after a frame). Sets `aria-hidden="false"` on the sheet.
- **`closeSheet()`**: removes `.on` from both, sets `aria-hidden="true"`, hides backdrop after the 250ms transition.
- **`selectTrip(id)`**: if `id === ACTIVE`, just `closeSheet()`. Otherwise `closeSheet()`, then `loadTrip(id).then(...)` → persist `wt_trip` in `localStorage`, `renderAll()`, `renderHeader()`, scroll to top.
- **`wireHeader()`**: `#tripPick` click → `openSheet()`. `#sheetBackdrop` click → `closeSheet()`. `#sheetList` click → if `e.target.closest('.sheet-trip')`, `selectTrip(btn.dataset.trip)`.

In `boot()`, after `renderAll();` add `renderHeader();` and after `wireOverlay();` add `wireHeader();`.

### Step 4: Verify

- [ ] Reload.

Expected: header reads `WockingTocking | Essex to Yorkshire ▾ | EP.01`. Tap title → bottom sheet with one trip card marked active. Tap backdrop → sheet closes.

### Step 5: Commit

- [ ] **Commit**

```bash
git add index.html
git commit -m "Header: trip picker + bottom-sheet selector"
```

---

## Task 7: Full hash routing + checklist persistence

Extend `applyHash` for `#tab/<name>`, `#trip/<id>`, `#talk/<stopId>`. Persist pre-trip checklist ticks in `localStorage` keyed by trip id.

**Files:**
- Modify: `index.html`

### Step 1: Expand `applyHash`

Update `applyHash()` to handle, in this order:

1. `#act/<id>` → `openAct(id)` (existing).
2. (No matching prefix from above) close the overlay if it's open.
3. `#trip/<id>` → if `TRIPS.trips` contains the id, `selectTrip(id)`.
4. `#tab/<name>` → if `document.getElementById(name)` exists, `showTab(name)`.
5. `#talk/<sid>` → `showTab('talk')`, then in `requestAnimationFrame` look up the stop, compute the slug (`name.toLowerCase().replace(/[^a-z]+/g, '-')`), and `scrollIntoView({behavior:'smooth', block:'start'})` on `#talk-<actId>-<slug>`.

### Step 2: Persist pre-trip checklist

- [ ] In `renderTrip`, replace the line that builds the `pre` variable so it reads from `readChecks()` and adds `data-ck="<i>"` on the `.ck` row and an `.on` class if checked. The `<b>` inside no longer needs `data-ck`.
- [ ] Add helpers above `renderTrip`:
  - `readChecks()`: `JSON.parse(localStorage.wt_check || '{}')[ACTIVE] || {}`. Try/catch returns `{}` on error.
  - `writeChecks(map)`: read the same wrapper, write `[ACTIVE] = map`, store back.
- [ ] Replace the existing `.ck` / `.ck b` CSS with:

```css
  .ck{display:flex;gap:9px;align-items:flex-start;padding:6px 0;font-size:13px;
      border-bottom:1px solid var(--line);cursor:pointer}
  .ck b{flex:none;width:15px;height:15px;border:1.5px solid var(--rust);margin-top:2px;
        position:relative}
  .ck.on b{background:var(--rust)}
  .ck.on b::after{content:"";position:absolute;left:2px;top:-1px;width:5px;height:9px;
        border:solid var(--paper);border-width:0 2px 2px 0;transform:rotate(45deg)}
  .ck.on span{color:var(--soft);text-decoration:line-through}
```

- [ ] Add `wireChecks()`: delegated click handler on `#trip` — if `e.target.closest('.ck')` matches, toggle the index in `readChecks()`, persist via `writeChecks()`, and toggle the `.on` class on the row.
- [ ] In `boot()`, after `wireHeader();` add `wireChecks();`.

### Step 3: Verify

- [ ] Reload.

Expected:
- Trip tab: tap a checklist row, checkbox fills and text strikes through. Refresh → state persists.
- `http://localhost:8080/#talk/lincoln` → opens Talk tab and smooth-scrolls to Lincoln.
- From Trip tab → Act 1 overlay → tap `Talk →` on Lavenham → overlay closes, Talk tab scrolls to Lavenham.
- Browser back from the Talk position → overlay reopens.

### Step 4: Commit

- [ ] **Commit**

```bash
git add index.html
git commit -m "Hash routing + persistent pre-trip checklist"
```

---

## Task 8: Service worker, `validateTrip`, README

Cache JSON for offline; add a dev-time sanity check that warns when ids referenced in `acts[].stopIds` or `days[]` don't exist in `stops{}`; update the README.

**Files:**
- Modify: `sw.js`
- Modify: `index.html`
- Modify: `README.md`

### Step 1: Service worker — precache trip JSON, stale-while-revalidate

- [ ] Rewrite `sw.js`:

```javascript
/* WockingTocking — offline cache */
var CACHE = 'wt-v2';
var SHELL = [
  './',
  './index.html',
  './manifest.json',
  './icon-192.png',
  './icon-512.png',
  './apple-touch-icon.png',
  './trips/index.json',
  './trips/ep01.json'
];

self.addEventListener('install', function (e) {
  e.waitUntil(caches.open(CACHE).then(function (c) { return c.addAll(SHELL); }));
  self.skipWaiting();
});

self.addEventListener('activate', function (e) {
  e.waitUntil(
    caches.keys().then(function (keys) {
      return Promise.all(keys.filter(function (k) { return k !== CACHE; })
        .map(function (k) { return caches.delete(k); }));
    })
  );
  self.clients.claim();
});

self.addEventListener('fetch', function (e) {
  if (e.request.method !== 'GET') return;
  var url = new URL(e.request.url);

  if (url.pathname.indexOf('/trips/') !== -1 && url.pathname.endsWith('.json')) {
    e.respondWith(
      caches.open(CACHE).then(function (cache) {
        return cache.match(e.request).then(function (cached) {
          var network = fetch(e.request).then(function (res) {
            if (res && res.ok) cache.put(e.request, res.clone());
            return res;
          }).catch(function () { return cached; });
          return cached || network;
        });
      })
    );
    return;
  }

  e.respondWith(
    caches.match(e.request).then(function (hit) {
      return hit || fetch(e.request).then(function (res) { return res; })
        .catch(function () { return caches.match('./index.html'); });
    })
  );
});
```

### Step 2: `validateTrip` warning

- [ ] Add to the `<script>` block, near the other helpers:

```javascript
  function validateTrip(t) {
    if (!t || !t.stops || !t.acts) return;
    var ids = Object.keys(t.stops);
    (t.acts || []).forEach(function (a) {
      (a.stopIds || []).forEach(function (sid) {
        if (ids.indexOf(sid) === -1) console.warn('[wt] act ' + a.id + ' references missing stop "' + sid + '"');
      });
    });
    (t.days || []).forEach(function (d) {
      (d.routeStops || []).concat(d.planStops || []).forEach(function (sid) {
        if (ids.indexOf(sid) === -1) console.warn('[wt] day ' + d.id + ' references missing stop "' + sid + '"');
      });
    });
    Object.keys(t.stops).forEach(function (sid) {
      var s = t.stops[sid];
      if (s.actId && !(t.acts || []).some(function (a) { return a.id === s.actId; })) {
        console.warn('[wt] stop ' + sid + ' has unknown actId "' + s.actId + '"');
      }
    });
  }
```

- [ ] In `loadTrip`, after `TRIP = t; ACTIVE = id;` call `validateTrip(t);`.

### Step 3: README update

- [ ] Replace `README.md` with:

````markdown
# WockingTocking — Trip App

A mobile-first web app for the WockingTocking vlog road trips. Open it on a phone
in Safari and "Add to Home Screen" to use it like a native app.

Live: <https://vlog-planner.vercel.app>

## What's inside

Four tabs:

- **Trip** — overview of the active trip: the three-day arc, kit, pre-trip checklist, and three Act cards. Tap an Act for the full chapter story.
- **Route** — tap-to-navigate Google Maps and Waze links; walking loops inside each town.
- **Plan** — the timed agenda for each day, with film cues.
- **Talk** — what to say on camera at each stop: a short story plus quick facts.

Tap the trip title in the header to switch between trips.

## Adding a new trip

1. Create `trips/<id>.json` (copy `trips/ep01.json` as a starting point).
2. Append an entry to `trips/index.json`:
   ```json
   { "id": "ep02", "title": "…", "subtitle": "…", "dates": "…", "episode": "EP.02" }
   ```
3. Add the new file path to the `SHELL` array in `sw.js`, and bump `CACHE` to a new version (e.g. `wt-v3`) so the service worker re-precaches.
4. Push. Vercel rebuilds automatically.

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

Then visit `http://localhost:8080`. (Opening `index.html` directly via `file://`
won't work because the app uses `fetch` to load JSON.)

## Spec & plan

Design and implementation notes live in `docs/superpowers/`.
````

### Step 4: Verify offline + sanity

- [ ] Reload `http://localhost:8080`. Console: no `[wt]` warnings.
- [ ] DevTools → Application → Service Workers: `wt-v2` active.
- [ ] DevTools → Network → check "Offline" → reload. App still loads, all tabs render, act overlay still works.
- [ ] Temporarily rename a `stopId` in `acts[0].stopIds` to something nonexistent → reload → console shows `[wt] act act1 references missing stop "..."`. Revert.

### Step 5: Commit and push

- [ ] **Commit**

```bash
git add sw.js index.html README.md
git commit -m "Cache trips JSON for offline, validateTrip warnings, update README"
```

- [ ] **Push**

```bash
git push origin main
```

Vercel auto-deploys. Confirm `https://vlog-planner.vercel.app` loads, the trip selector opens, and an Act overlay opens with the narrative.

---

## Self-review

- **Spec coverage:** trip data model ✅ (Tasks 1, 3), header selector ✅ (Task 6), Act drill-down ✅ (Task 5), Talk story+facts ✅ (Tasks 3, 4), render pipeline ✅ (Task 2), hash routing ✅ (Tasks 5, 7), service worker ✅ (Task 8), error handling ✅ (Task 2 catch + Task 8 validateTrip), README ✅ (Task 8).
- **Placeholders:** None — every step describes the change with the actual code or command. Story drafts in Task 3 are complete prose.
- **Type/name consistency:** `TRIP`, `TRIPS`, `ACTIVE`; helpers `esc`, `attr`, `colorVar`, `stop`; functions `renderTrip/Route/Plan/Talk`, `renderAll`, `renderHeader`, `loadTrip`, `boot`, `showTab`, `openAct`, `closeAct`, `wireOverlay`, `wireTabs`, `wireHeader`, `wireChecks`, `wireHash`, `applyHash`, `selectTrip`, `openSheet`, `closeSheet`, `validateTrip`, `readChecks`, `writeChecks`, `routeCardHTML`, `planCardHTML`, `talkCardHTML`, `bookendCardHTML`. DOM ids: `actOverlay`, `overlayBack`, `overlayActLabel`, `overlayBody`, `tripPick`, `tripPickTitle`, `tripPickEp`, `tripSheet`, `sheetBackdrop`, `sheetList`. All used as named.
