# CLAUDE.md

Project context for Claude Code working on **CallingManager** — a private, offline,
single-file replacement for callingmanager.com after the Church retired the LCR JSON
endpoint that the original depended on.

## What this project is

`callings-tool.html` is a **single self-contained HTML file** (HTML + CSS + JS, all
inline) that lets a bishopric visualize and plan ward callings on a colorful
drag-and-drop board, in the spirit of the original callingmanager.com. The file:

- Pulls calling data via a bookmarklet that scrapes the LCR `/mlt/orgs` page (the Church
  removed the old JSON endpoint in April 2026; see `callingmanager-fix-notes.md`).
- Runs 100% in the browser. **Nothing — names, callings, or any other data — is ever
  uploaded.** This privacy property is non-negotiable; any change that would require a
  network call with user data should be rejected.

## Files

| File | Purpose |
|---|---|
| `callings-tool.html` | The app itself. Single-file, self-contained. |
| `callingmanager-fix-notes.md` | Technical write-up for the original callingmanager.com developer (jds580s on the Church Tech Forum) — explains the diagnosis and the recommended scraper-based fix. |
| `callingmanager-fix.zip` | Bundle of the tool + the dev notes for sharing. Regenerated from the other files. |
| `README.md` | Human-facing description. |
| `HISTORY.md` | Chronological narrative of the Cowork session that produced this project: what was investigated, decided, and built. |
| `CLAUDE.md` | This file. |

## Architecture of `callings-tool.html`

The script is organised into clearly-labelled sections (search for `/* ==== ` to
navigate):

- **`bmScrape` / `bmDeliver`** — the scraper that runs on the LCR page (compiled into
  the bookmarklet at startup via `Function.prototype.toString`). Reads the rendered
  `<table>` rows from `lcr.churchofjesuschrist.org/mlt/orgs` and produces JSON.
- **helpers** — `esc`, `deepCopy`, `parseLcrDate`, `tenureMonths`, `tenureBucket`,
  `tenureText`, `fmtDate`.
- **colour palette** — `ORG_COLORS` (by org name) and `FALLBACK` palette. Refined,
  muted pastels — deliberately *not* the original's loud `rgba(255,174,67,.5)` etc.
- **state / model** — `MODEL = {unit, loadedAt, callings:[], members:{}, seq}`.
  - `callings`: array of `{id, org, sub, name, home}` where `home` is the id of the
    original occupant (or `null` for vacant).
  - `members`: keyed by id; each has `{id, name, sustained, setApart, home, loc}`.
    `loc` is a calling id, or `"pool"` (Names to Consider), or `"hide"`.
- **persistence** — `localStorage` autosave + explicit Save/Open `.json` project file.
- **ingest** — accepts either a fresh scrape (`obj.organizations`) or a saved project
  (`obj.callings && obj.members`).
- **rendering** — masonry board (CSS `column-width:340px`); pool above, columns of org
  panels, hide bin below.
- **drag-and-drop** — native HTML5; event delegation on `#board`.
- **print** — `buildPrintAll()` and `buildPrintChanges()` render into `#printArea`;
  `@media print` swaps to it.
- **bookmarklet build** — concatenates `bmScrape.toString()` + `bmDeliver.toString()`
  into a single `javascript:` URL with newlines collapsed.

### Key invariants

1. **Single file, no build step.** No npm, no bundler. If you split into modules you
   defeat the "open the .html locally" deployment model.
2. **No external runtime calls.** The Google Fonts stylesheet link is the only network
   request, and it degrades gracefully when offline.
3. **`bmScrape` and `bmDeliver` must remain self-contained** (no references to
   anything outside themselves). Their `.toString()` is what gets compiled into the
   bookmarklet that runs on the LCR page.
4. **Member identity is by id.** Calling ids are stable strings of the form
   `` `${org}␟${sub}␟${name}␟${occurrence}` ``. Member ids are
   `m0, m1, …` from a monotonic `MODEL.seq`.
5. **"Changed" callings** are computed, not stored: a calling is changed iff the set
   of members with `loc===calling.id` differs from `{calling.home}`.

## Verification

There is no test framework checked in (single-file app). The verified approach is
**jsdom in `/tmp`**:

```bash
cd /tmp && npm install jsdom --no-audit --no-fund
# write a test that loads the file, calls window.ingest(window.DEMO),
# exercises moveMember/changedCallings/buildPrintChanges, etc.
node test.js
```

The last full test pass exercised: load without errors, bookmarklet href starts with
`javascript:`, demo ingest (24 callings, 20 members), tenure math, board renders org
panels and pool and hide and member cards (draggable), no changes initially, releasing
creates a changed calling, filling a vacancy creates another, `addMember` adds to pool,
`buildPrintAll`/`buildPrintChanges` produce content, model serialises and re-opens
preserving changes. **24/24 passed.**

There is no headless-browser screenshot available from the Cowork sandbox (network
restrictions block puppeteer/playwright browser downloads, and the Chrome MCP rewrites
`file://` URLs). Visual verification has to be done by opening the file in a real
browser.

## How the bookmarklet flow works

1. User loads `callings-tool.html` (just opens the file in a browser).
2. User clicks **Get data** → drags the "Grab Ward Callings" bookmarklet to their
   bookmarks bar (one time).
3. User signs in to `lcr.churchofjesuschrist.org`, opens Callings → Organizations,
   waits for the page to load.
4. User clicks the bookmarklet → a small window opens with the JSON in a textarea +
   a Copy button. User clicks Copy.
5. User comes back to the tool, clicks **Paste**, pastes, **Load data**.
6. The tool renders the board. From here it's drag-and-drop planning.

## Constraints worth remembering

- **The Church's `mltp-api.churchofjesuschrist.org` backend requires a bearer token**,
  not just session cookies — confirmed in `callingmanager-fix-notes.md`. The
  cookie-paste model the original used is impossible to revive against the new API.
  Scraping the rendered page is the only path that keeps the tool fully client-side.
- DOM-scraping is coupled to LCR's current markup. If they change the
  `<th>` layout of the org tables, the column indices `c[0]..c[3]` in `bmScrape` need
  to follow.
- The Church publishes calling data in a locale-formatted date string (e.g.
  `"4 Jan 2026"`) — `parseLcrDate` handles that and the variant `"2026 Jan 4"`.
- Real-world ward verified against: Rexburg 18th Ward (140341) — 10 main organizations,
  58 sub-org tables, 229 callings.

## Suggested next work

If `HISTORY.md` is up to date, possible follow-ups:

- "Out of Unit Callings" pass: same scraping approach against the
  *Other Units and Leaders* toggle on `/mlt/orgs`.
- A second-pull change-tracking mode (snapshot a baseline `MODEL.current` and diff
  against a fresh pull) — an earlier prototype had this; it was removed when the UI
  was rebuilt around the drag-to-plan model. Re-adding it cleanly without confusing
  the "changed = my plan changed it" semantics is non-trivial.
- Hand the diagnosis + scraper code to jds580s on the Church Tech Forum
  (`viewtopic.php?t=21315`) so a patch can land in callingmanager.com proper.
