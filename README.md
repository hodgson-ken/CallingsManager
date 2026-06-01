# CallingManager

A private, offline tool for visualizing and planning ward callings — a working
replacement for callingmanager.com, which broke in April 2026 when the Church removed
the LCR JSON endpoint it depended on.

## What it does

- **Drag-and-drop planning board** of every calling in your ward, colour-coded by
  organisation.
- **Time-in-calling heatmap** — click the clock to shade every member by how long
  they've served.
- **Names to Consider** pool — drag people in or add them manually; drag onto callings
  to propose changes.
- **Hide** bin for setting people aside.
- **Print Changes** and **Print All Callings**.
- **Save / Open project** as a single `.json` file.
- **No network calls with your data.** Everything happens locally in your browser. You
  can disconnect from the internet after the page loads and the tool still works.

## How to use it

1. **Open `callings-tool.html`** in any modern browser (just double-click it).
2. Click **Get data** and drag the **Grab Ward Callings** button onto your bookmarks
   bar. (One-time setup.)
3. Sign in to `lcr.churchofjesuschrist.org`, go to **Callings → Organizations**, and
   let the page finish loading.
4. Click your **Grab Ward Callings** bookmark, then click **Copy**.
5. Come back to the tool, click **Paste**, paste in the JSON, click **Load data**.
6. Drag members between callings to plan changes. Use **Print Changes** for a
   bishopric meeting handout.

## Files in this folder

- `callings-tool.html` — the tool. Self-contained; this one file is everything you
  need to run it.
- `callingmanager-fix-notes.md` — technical write-up of what broke at callingmanager.com
  and how to fix it. Useful if you want to help the original developer ship a patch.
- `callingmanager-fix.zip` — the tool + the dev notes, zipped up for sharing.
- `CLAUDE.md` — context for Claude Code if you continue development in that tool.
- `HISTORY.md` — chronological log of the Cowork session that produced this project.

## Privacy

The original callingmanager.com's policy applies here verbatim: **church policy is
that membership data must not be sent to third-party servers.** This tool sends nothing.
It loads in your browser, processes the data you paste in entirely on your computer,
and produces print output locally. If you want to verify, disconnect from the internet
after the page has loaded — the tool keeps working.
