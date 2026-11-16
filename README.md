# CallingManager

A private, offline tool for visualizing and planning ward callings — a working
replacement for callingmanager.com, which broke in April 2026 when the Church removed
the LCR JSON endpoint it depended on.

## What it does

- **Drag-and-drop planning board** of every calling in your ward, colour-coded by
  organisation.
- **One-click import from LCR** — a bookmark reads the Callings By Organization page
  and the members-without-callings report, then opens the tool with the data loaded.
  No copy and paste when the tool is opened from a web address.
- **Members Without Callings** — a filterable list (gender, age range) to drag into
  vacancies. Collapsed by default.
- **Names to Consider** pool — drag people in or add them manually, then drag them onto
  callings to propose changes. Can be pinned as a narrow left side panel or a sticky
  top bar, so it stays in view while you scroll.
- **Hide** bin for setting people aside.
- **Time in calling** (clock menu) — colours every member by how long they've served
  (on by default), or shows only callings held 2+, 3+ or 5+ years.
- **Changes view** — a visual board of just the proposed changes, printable.
- **Re-import keeps your plan** — a fresh LCR pull is merged under your planned moves.
  A report lists anything that no longer fits (e.g. someone already released), with an
  option to discard the plan instead.
- **Data check** — warns before loading if the LCR data looks wrong (for example after
  LCR changes its page layout).
- **Search** — fades callings and groups that don't match and collapses empty ones;
  Esc or × clears.
- **Layout** menu — 3, 4 or automatic columns, and Normal / Large / Extra large text for
  a TV or projector. Saved per browser.
- **Undo / redo**, **Print Changes**, **Print All Callings**.
- **Save / Open project** as a single `.json` file.
- **No network calls with your data.** Everything happens locally in your browser.

## How to use it

1. **Open the tool** — either the hosted copy or `callings-tool.html` double-clicked
   locally.
2. Click **Data ▸ Get data** and drag the **Grab Ward Callings** button onto your
   bookmarks bar. (One-time setup. Get data tells you if your bookmark is current and
   skips this step; drag it again only when it says the bookmark is out of date.)
3. Sign in to `lcr.churchofjesuschrist.org` and open **Callings → Callings By
   Organization**. If you click the bookmark too early it waits for the page; on the
   wrong page it offers to take you there.
4. Click your **Grab Ward Callings** bookmark.
   - Hosted copy: when **Ward data ready** appears, click **Open in Callings
     Manager**; the tool opens with the data loaded. If the browser asks, allow
     pop-ups for the LCR site.
   - Local file: a box appears; click **Copy**, return to the tool, choose
     **Data ▸ Paste**, paste, and click **Load data**.
5. Drag members between callings to plan changes. Use **Changes** to review them and
   **Print Changes** for a bishopric meeting handout.
6. Pull again any time; your plan is kept on top of the fresh data.

## Files in this folder

- `callings-tool.html` — the tool. Self-contained; this one file is everything you
  need to run it.
- `callingmanager-fix-notes.md` — technical write-up of what broke at callingmanager.com
  and how to fix it. Useful if you want to help the original developer ship a patch.
- `callingmanager-fix.zip` — the tool + the dev notes, zipped up for sharing.
- `CLAUDE.md` — context for Claude Code if you continue development in that tool.
- `HISTORY.md` — chronological log of how this project was built.

## Privacy

The original callingmanager.com's policy applies here verbatim: **church policy is
that membership data must not be sent to third-party servers.** This tool sends nothing.
The page is static. The bookmark hands data to the tool tab directly with
`postMessage`, and the tool accepts it only from the LCR site. Data never goes in a web
address, so it can't end up in browser history or history sync; old bookmarks that did
this are refused. Your board is kept in your browser's local storage (on that device
only, not synced) and in any project file you save. *Data ▸ Start over* erases it. The
only network request is the Google Fonts stylesheet, which carries no ward data. If you want to verify, disconnect from the internet after the page has
loaded — the tool keeps working.
