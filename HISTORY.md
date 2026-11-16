# Session History

A chronological record of the session that produced this project (May 24, 2026).
Written for future-Ken to pick up the thread.

## 1. The starting problem

Ken (the user) pointed at a Church Tech Forum thread
([`viewtopic.php?t=21315&start=20`](https://tech.churchofjesuschrist.org/forum/viewtopic.php?t=21315&start=20))
about callingmanager.com being broken, and asked: can we see how to get it working
again?

The original callingmanager.com is a client-side tool by Justin Sirois (forum handle
**jds580s**) that visualised ward callings. The user pasted in a JSON URL from LCR and
the site rendered the callings as a drag-and-drop board.

## 2. Diagnosis

Web fetch on the forum was blocked, but a web search surfaced enough to begin. Inside
the user's already-logged-in Chrome session, the two URLs the original site asks the
user to copy from:

- `/api/orgs/sub-orgs-with-callings?ip=true&lang=eng`
- `/api/orgs/out-of-unit-callings?includeTypes=LIVING_INSIDE&lang=eng`

**both returned HTTP 404** — the endpoints were removed, not reformatted. callingmanager.com's
own notice only mentioned the second one breaking, but the whole import path was dead.

The replacement page is `lcr.churchofjesuschrist.org/mlt/orgs`, a Next.js App Router
app that renders the calling data server-side and streams it as RSC flight data into
the page (`self.__next_f`). There is no longer any plain-JSON URL a signed-in user can
open in a tab. The backend that the Next.js server calls
(`mltp-api.churchofjesuschrist.org`) is referenced as a bare host in page config but
requires a bearer token, not just session cookies — which matches what jds580s already
suspected on the forum.

So the entire copy-a-URL-and-paste-it model the original tool used has nothing to
point at anymore. Any fix had to either parse the RSC flight data or scrape the
rendered DOM. **DOM scraping won** — semantic and stable; RSC format is internal to
Next.js and changes between framework versions.

Verified on a real ward: **10 main organizations,
58 sub-org tables, 229 callings.**

## 3. First deliverables

1. **A viewer + bookmarklet** — a single self-contained HTML file with a draggable
   bookmarklet that scraped LCR's DOM and opened a clean printable report.
2. **A developer write-up** (`callingmanager-fix-notes.md`) — diagnosis plus a
   verified scraper function and the new JSON schema, written for jds580s.

The bookmarklet is built at page load by concatenating `bmScrape.toString()` +
`bmDeliver.toString()` into a single `javascript:` URL with newlines collapsed.

## 4. Iteration: "behave more like the original"

The user asked for the tool to behave more like the original — change tracking, time-in-calling,
planning notes, polished view. The next pass added: a baseline-vs-current diff,
time-in-calling math, plan proposals with notes, names-to-consider, a tabbed UI
(Callings / Changes / Names / Setup), and Print All / Print Changes. 24 jsdom tests
all green.

## 5. Iteration: "make it look like the original"

The tabbed UI was not what the user wanted — the original is a colourful multi-column
masonry board with each organisation a coloured panel. The next pass dropped the tabs
and built a board-style UI with a dark header, masonry columns, colour-coded org
panels, calling cards.

## 6. Source code, and the faithful rebuild

The user then pasted the full source of `callingmanager.com/index.html`. With the
source plus the actual stylesheet (`callingmanager.com/style.min.css`, fetched
separately), a faithful rebuild became possible:

- **Single-file HTML** with `Fjalla One` header + `Lato` body (Google Fonts, with
  graceful fallback).
- **Drag-and-drop members between callings** — native HTML5 DnD.
- **Time-in-calling heatmap** — 12 buckets at 6-month intervals, toggled by a clock
  button.
- **"New / Names To Consider"** pool above the board, **"Hide"** bin below.
- **Add a member** (header `+` icon) and **add a calling** (per-sub `+`).
- **Print Changes** / **Print All Callings**.
- The data model:
  - `MODEL.callings[]` — fixed list of calling slots, each with `home` (original
    occupant id or null).
  - `MODEL.members{}` — keyed by id; each has `loc` (a calling id, `"pool"`, or
    `"hide"`).
  - A calling is "changed" iff its current occupants differ from `{home}`.

The user asked for **better colours than the original's loud pastels**. The palette in
use is a curated set of muted tones at similar lightness — soft tan (Bishopric), dusty
blue (Elders Quorum), sage (Sunday School), muted gold (Primary), rose (Relief Society),
lavender (Young Women), teal (Aaronic Priesthood), warm sand (Temple and Family History),
sky (Ward Missionaries), stone (Other Callings). Background: a calm light gradient,
not the original's blurred photo.

## 7. Verification

All checks run with jsdom in `/tmp` (puppeteer/playwright browser binary downloads are
blocked by the sandbox network, and browser automation can't open `file://` URLs).
The final run: **24/24 passed** — load without errors, bookmarklet href, demo ingest
(24 callings / 20 members), tenure math, board renders with all expected pieces,
member cards are `draggable="true"`, no changes initially, release creates a changed
calling, filling a vacancy creates another, addMember adds to pool, both print
builders produce content, model serialises and round-trips with changes preserved.

## 8. Tidy-up

The outputs folder was littered with junk from earlier experiments (`.DS_Store`, a
stray `ziCttwQ4` zip-temp file, a duplicate extracted folder). Cleared via the
file-delete permission flow. The folder was then copied here to
`~/Apps/CallingManager/`.

## 9. This handoff

Set up as a project: `CLAUDE.md` at root with architecture and invariants,
`README.md` for humans, this `HISTORY.md`, and a git repo with an initial commit so
future development is tracked from a clean baseline.

## 10. Continued development (May–November 2026)

Work after the handoff, driven by real use in bishopric planning. Roughly in order:

- **Undo / redo** (50 steps) with a FLIP animation so cards slide back to where they
  came from. Column controls and collapse-all.
- **Overlay scraper.** The bookmarklet shows its Copy box inside the LCR page instead of
  a popup window, which iPad Safari blocked.
- **Card layout.** Calling slots became a CSS grid; expanded member cards span two
  slots when there is room, and detail rows wrap instead of overflowing.
- **One-click import.** First version: when the tool is served over http(s), the
  bookmarklet navigated to `TOOL#import=<json>`; the tool read the fragment, cleared it
  from the address bar, and accepted scrapes only. A BroadcastChannel let an
  already-open tool tab adopt the import. `file:` still uses Copy/Paste. (Replaced in
  November; see below.)
- **Members Without Callings.** The bookmarklet loads
  `/mlt/report/member-callings?tab=without-callings` in a hidden off-screen iframe (LCR
  allows same-origin framing) and reads name, gender and age. Cells contain responsive
  labels, so the scraper takes each cell's last text node. Birth dates and phone numbers
  are deliberately not collected. The list is filterable, uses columns, and starts
  collapsed.
- **Get data instructions** rewritten for non-technical users, with a small animation of
  dragging the bookmark to the bar.
- **Changes view.** A visual board of just the changed callings (from → to), printable
  with colours.
- **Toolbar sections:** Data (menu: Get data, Paste, Open, Save), Plan (undo, redo, add
  member, Changes with an eye icon), View (Layout, collapse, time, print). Menus open
  toward whichever side has room.
- **Merge on re-import.** A fresh LCR pull no longer wipes the plan. `mergePlan` rebuilds
  the model from LCR and re-applies each planned move, matching people by name plus home
  calling, and carries over user-added callings. It reports moves already done in LCR,
  people released or newly called since the plan was made, and callings that no longer
  exist. The report offers *Discard plan, use LCR as-is* (undoable).
- **Scrape health check.** `validateScrape` warns before loading if calling names are
  blank, member names look like dates, sustained dates are missing or unreadable, the
  without-callings rows lack gender or age, or the calling count drops sharply. It runs
  in the tool, so it needed no bookmark change.
- **Time menu.** The clock opens a menu: *Color by time in calling* (now on by default)
  and *Show callings held* Any / 2+ / 3+ / 5+ years.
- **Search** hides groups with no matches, shows a "No callings match" note, and has a
  clear button (Esc also clears).
- **Layout menu.** 3 / 4 / Auto columns and Normal / Large / Extra large text (CSS
  `zoom` on the board) for TV or AirPlay use. Saved per browser in its own localStorage
  key, outside the project. This also fixed a bug where every redraw dropped the column
  choice and the board fell back to auto (4 columns on wide screens).
- **Pin Names to Consider.** From a pin icon on the section (or Layout), keep the pool in
  view as a narrow left side panel sized to the longest name (150–220px), or as a sticky
  top bar. Narrow screens fall back to the top bar.

- **Privacy fix: no data in URLs.** Review found that the `#import=` address, though
  never sent to the server, was recorded in browser history and could be uploaded by
  Chrome or Safari history sync. The bookmarklet now collects everything, shows a
  *Ward data ready* box, and on the *Open in Callings Manager* click opens the tool tab
  and sends the data with `postMessage` once the tool replies `callings-ready`. (A first
  attempt opened the tab immediately; that pushed LCR into the background, where Chrome
  throttled the hidden report page and members without callings never loaded.)
  The tool accepts only messages from `https://lcr.churchofjesuschrist.org` sent by its
  opener, and only scrapes, never full projects. If the handoff fails (blocked pop-up,
  or LCR severing the opener link), the bookmark falls back to the Copy box after 10
  seconds. Old `#import=` bookmarks are refused with instructions to redrag and clear
  history. After another tool tab adopts the import, the import tab closes itself.

The tool is also hosted as a static page on the user's own web server, which is what
makes one-click import possible; it still sends no data anywhere.

## Open threads

- "Out of Unit Callings" — the *Other Units and Leaders* toggle on `/mlt/orgs`. Same
  scraping approach would work but isn't implemented.
- Between-pulls tracking is now partly covered by merge on re-import (the plan survives
  a fresh pull and conflicts are reported). A separate "what changed in LCR since last
  pull" view is still not built.
- Ideas raised but not built: per-change status (proposed → sustained → set apart),
  private notes on a change, a sustaining/release script, flags for members with
  several callings, and an encrypted project file for sharing with counselors.
- The diagnosis in `callingmanager-fix-notes.md` should reach jds580s (forum post #28
  on `viewtopic.php?t=21315`) — he explicitly asked on April 28, 2026 for someone with
  web expertise to look at the new page and report what they see.
