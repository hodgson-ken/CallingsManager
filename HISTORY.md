# Session History

A chronological record of the Cowork session that produced this project (May 24, 2026).
Written for future-Ken and for Claude Code to pick up the thread.

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

Verified on the real ward (Rexburg 18th Ward, 140341): **10 main organizations,
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
blocked by the Cowork sandbox network, and the Chrome MCP can't open `file://` URLs).
The final run: **24/24 passed** — load without errors, bookmarklet href, demo ingest
(24 callings / 20 members), tenure math, board renders with all expected pieces,
member cards are `draggable="true"`, no changes initially, release creates a changed
calling, filling a vacancy creates another, addMember adds to pool, both print
builders produce content, model serialises and round-trips with changes preserved.

## 8. Tidy-up

The outputs folder was littered with junk from earlier experiments (`.DS_Store`, a
stray `ziCttwQ4` zip-temp file, a duplicate extracted folder). Cleared via the
`allow_cowork_file_delete` permission flow. The folder was then copied here to
`~/Apps/CallingManager/`.

## 9. This handoff

Set up as a Claude Code project: `CLAUDE.md` at root with architecture and invariants,
`README.md` for humans, this `HISTORY.md`, and a git repo with an initial commit so
future development is tracked from a clean baseline.

## Open threads

- "Out of Unit Callings" — the *Other Units and Leaders* toggle on `/mlt/orgs`. Same
  scraping approach would work but isn't implemented.
- An earlier prototype had a between-pulls change-tracking mode (baseline snapshot vs
  fresh pull). It was dropped when the UI moved to the drag-to-plan model to avoid
  two competing notions of "changed". Re-adding it cleanly is plausible future work.
- The diagnosis in `callingmanager-fix-notes.md` should reach jds580s (forum post #28
  on `viewtopic.php?t=21315`) — he explicitly asked on April 28, 2026 for someone with
  web expertise to look at the new page and report what they see.
