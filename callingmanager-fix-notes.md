# CallingManager.com — What Broke and How to Fix It

*Investigation notes prepared for the developer (jds580s) — May 2026*

---

## TL;DR

- **Both** legacy LCR JSON endpoints now return **HTTP 404**. They were removed, not
  reformatted — so this is not the usual "small adjustment" fix.
- The LCR **Organizations** page was rebuilt as a **Next.js / React Server Components**
  app at `lcr.churchofjesuschrist.org/mlt/orgs`.
- There is **no longer any plain-JSON URL** a signed-in member can open in a tab and copy.
  The "open this link, Ctrl-A, Ctrl-C, paste" model has nothing to point at anymore.
- **Recommended fix:** replace the copy-a-URL step with a **bookmarklet that scrapes the
  rendered page**. It keeps CallingManager 100% client-side, so the privacy promise is
  unchanged. Working, verified scraper code is below.

---

## What changed

### Old model (pre-2026)

CallingManager relied on two endpoints that returned plain JSON and were reachable in a
browser tab using only the logged-in session cookie:

| Purpose | Old endpoint |
|---|---|
| Callings by Organization | `/api/orgs/sub-orgs-with-callings?ip=true&lang=eng` |
| Out of Unit Callings | `/api/orgs/out-of-unit-callings?includeTypes=LIVING_INSIDE&lang=eng` |

### Current state (verified May 2026)

- Both endpoints above now return `HTTP Status 404 – Not Found`. The whole `/api/orgs/*`
  family appears to be retired.
- The replacement page is `lcr.churchofjesuschrist.org/mlt/orgs`. It is a **Next.js App
  Router** application (React Server Components).
- Calling data is fetched **server-side** and streamed into the page as RSC "flight"
  payload (`self.__next_f.push(...)`). On load, the only same-origin POST is the RSC
  request back to `/mlt/orgs` itself — there is no client-side `fetch` to a JSON API.
- Searching the client JS bundles for `mltp`, `unit-data`, `/api/`, `sub-orgs`, etc.
  returns nothing — confirming the data layer moved entirely server-side.
- `mltp-api.churchofjesuschrist.org` **does** exist (it is the backend the Next.js server
  calls), but it appears in the page only as a bare base host in config. The server calls
  it with a **bearer token**, not the plain session cookie. So a member cannot open an
  `mltp-api …/unit-data` URL and copy JSON the way they used to — which matches the
  "authorization steps beyond just being logged in" you already suspected.

### Why the old approach can't simply be re-pointed

The copy/paste flow depended on a **cookie-authenticated endpoint that renders JSON in a
browser tab**. That category of endpoint no longer exists for callings. The data now lives
in the browser in only two places:

1. The **rendered DOM tables** on `/mlt/orgs`.
2. The **RSC flight payload** embedded in the page HTML.

Any fix has to read from one of those.

---

## Recommended fix: a scraper bookmarklet

A bookmarklet runs JavaScript on the LCR page the member already has open, reads the data,
and hands it to CallingManager. This is the natural evolution of the existing model:

- Still **100% client-side** — no membership data touches any server. The privacy FAQ on
  the site stays true verbatim.
- The user-facing change is small: instead of *"open this link and copy the text,"* it
  becomes *"click this bookmark."*

**Why scrape the DOM and not the RSC `__next_f` data:** the rendered tables are stable and
semantic (clear `Calling / Member / Sustained / Set Apart` columns). The RSC flight format
is an internal Next.js serialization that changes between framework versions and is
unpleasant to parse. The DOM is the more durable target.

### How to get the data into CallingManager

Any of these work; pick whichever fits the current code best:

1. **Clipboard** — bookmarklet builds the JSON and calls `navigator.clipboard.writeText()`;
   keep the existing paste box, just point the parser at the new schema. Smallest UX change.
2. **URL hash** — bookmarklet opens `callingmanager.com/#data=<encoded JSON>`; the page
   reads `location.hash` on load. Stays client-side (the hash is never sent to the server).
3. **postMessage** — if the bookmarklet opens CallingManager in a new tab, it can
   `postMessage` the payload to it.

### The scraper (verified against a live ward, May 2026 — 10 orgs, 58 tables, 229 callings)

```js
function scrapeWardCallings() {
  // Unit name, e.g. "Rexburg 18th Ward (140341)"
  var um = (document.body.innerText || "")
             .match(/[^\n]*\b(?:Ward|Branch)\s*\(\s*\d+\s*\)/);
  var unit = um ? um[0].trim() : "My Unit";

  // The page lays out one <table> per sub-organization, each preceded by an
  // <h2> (main org) and optionally an <h3> (sub-org). Walk them in document order.
  var nodes = document.querySelectorAll("h1,h2,h3,h4,h5,table");
  var main = "", sub = "", orgs = [], map = {};

  for (var i = 0; i < nodes.length; i++) {
    var n = nodes[i], t = n.tagName;

    if (t === "TABLE") {
      var rows = n.querySelectorAll("tbody tr");
      if (!rows.length) continue;
      var callings = [];
      for (var j = 0; j < rows.length; j++) {
        var c = rows[j].children;                       // cells, in order:
        var name = ((c[1] && c[1].innerText) || "").trim();
        var vacant = /vacant/i.test(name) || name === "";
        callings.push({
          calling:  ((c[0] && c[0].innerText) || "").trim(),  // 0: calling name
          member:   vacant ? "" : name,                       // 1: member name
          vacant:   vacant,
          sustained:((c[2] && c[2].innerText) || "").trim(),   // 2: sustained date
          setApart: !!(c[3] && c[3].querySelector("svg"))      // 3: checkmark SVG = set apart
        });
      }
      var key = main || "(Organization)";
      if (!(key in map)) { map[key] = orgs.length; orgs.push({ name: key, subOrgs: [] }); }
      orgs[map[key]].subOrgs.push({ name: sub, callings: callings });

    } else {
      var txt = (n.innerText || "").trim();
      if (t === "H1" || t === "H2") { main = txt; sub = ""; }
      else if (t === "H3")          { sub = txt; }
      else if (!sub)                { sub = txt; }
    }
  }

  return {
    unit: unit,
    generated: new Date().toISOString(),
    source: "lcr.churchofjesuschrist.org/mlt/orgs",
    organizations: orgs
  };
}
```

### Output schema

```json
{
  "unit": "Rexburg 18th Ward (140341)",
  "generated": "2026-05-24T17:30:00.000Z",
  "source": "lcr.churchofjesuschrist.org/mlt/orgs",
  "organizations": [
    {
      "name": "Elders Quorum",
      "subOrgs": [
        {
          "name": "Elders Quorum Presidency",
          "callings": [
            {
              "calling":  "Elders Quorum President",
              "member":   "Last, First",
              "vacant":   false,
              "sustained":"4 Jan 2026",
              "setApart": true
            }
          ]
        }
      ]
    }
  ]
}
```

For top-level organizations with no sub-org heading (e.g. **Bishopric**), the single
`subOrgs` entry has an empty `name` — render its callings flush under the org.

---

## Field notes for updating the parser

- CallingManager's current parser expects the old `sub-orgs-with-callings` shape. Either
  map this new schema onto that internal shape (smallest diff) or retarget the parser.
- `setApart` is **derived** — the LCR page shows a checkmark SVG in the "Set Apart" column
  when true and an empty cell when false. There is no boolean field in the DOM.
- `vacant` is derived from the member cell reading "Calling Vacant" (or being empty).
- `sustained` is a **locale-formatted display string** (here `dd MMM yyyy`, e.g.
  `"4 Jan 2026"`), not an ISO date. Parse accordingly if you sort by date.
- The page renders **all** org tables into the DOM at once (no lazy/virtualized rows), so
  a single pass after the page finishes loading captures the whole ward.
- Verified counts on one ward: 10 main organizations, 58 sub-org tables, 229 callings.

## Out-of-unit callings

The old `out-of-unit-callings` endpoint is also 404. In the new UI this content is behind
the **"Other Units and Leaders"** radio toggle on the same `/mlt/orgs` page. The same
DOM-scraping approach applies — treat it as a second pass after toggling that view, if you
want to restore that optional feature.

## Caveats

- DOM scraping is inherently coupled to LCR's markup. If the Church changes the table
  structure, the column indices (`c[0]..c[3]`) may need adjusting — but this is a far
  smaller and clearer surface than parsing RSC internals.
- If the Church later publishes a documented, cookie- or token-accessible API, prefer that
  over scraping.

---

*Prepared from a live inspection of `lcr.churchofjesuschrist.org/mlt/orgs` and the
Tech Forum thread "Tool for managing callings" (topic 21315).*
