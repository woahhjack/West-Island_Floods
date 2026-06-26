Access the map HERE -> https://woahhjack.github.io/West-Island_Floods/

# West Island Flood Map

A community-driven flood tracking map for the West Island of Montreal. Reports come in through a Google Form, get approved by an admin, and appear on a public Leaflet.js map hosted on GitHub Pages.

## How it works (the big picture)

```
Google Form  →  Google Sheet  →  Apps Script (Code.gs)  →  index.html (GitHub Pages)
 (reports)       (database)      (geocoding + JSON feed)    (the public map)
```

1. Someone fills out the Google Form to report flooding.
2. Their response lands as a new row in the Google Sheet.
3. The admin reviews it and types **"Approved"** in the Approved column.
4. The moment that happens, an Apps Script function (`onEdit`) automatically looks up the address and fills in Latitude/Longitude.
5. When anyone visits the map, the page asks the Apps Script Web App for the list of approved, geocoded reports (as JSON) and draws them.
6. The map also tries to highlight the actual street (not just a dot) using live OpenStreetMap road data, fetched directly from the visitor's own browser.

## Files

| File | Lives where | Job |
|---|---|---|
| `Code.gs` | Google Apps Script (attached to the Sheet) | Geocodes approved addresses, serves the JSON data feed (including the report's city) |
| `index.html` | GitHub Pages repo | The actual map page everyone visits |

## Setting things up from scratch

1. **Google Sheet** needs these exact column headers in row 1 (spelling/spacing matters — the code matches on exact text):
   - `Street, Address, or Intersection / Rue, adresse ou intersection (...)`
   - `Where is/was the flood`
   - `How severe was the flooding?`
   - `Any other details?`
   - `Approved?`
   - `Latitude`
   - `Longitude`

2. **Apps Script**: Extensions → Apps Script → paste in `Code.gs` → Save.

3. **Add the trigger** (this is the part that's easy to forget and breaks everything if skipped):
   - Click the clock icon (Triggers) in the Apps Script sidebar
   - **+ Add Trigger**
   - Function: `onEdit` / Event source: `From spreadsheet` / Event type: `On edit`
   - Save, authorize when prompted

4. **Deploy as a Web App**: Deploy → Manage deployments → set Access to "Anyone," Execute as "Me." Copy the Web App URL.

5. **Paste that URL** into `index.html`'s `DATA_FEED_URL` constant.

6. **Backfill old rows**: if you have rows that were already "Approved" before any of this existed, run `geocodeAllApprovedRows` once manually from the Apps Script function dropdown.

7. **Push `index.html` to GitHub Pages.**

## Making a code change later

- **Edited `Code.gs`?** You must create a **new deployment version** (Deploy → Manage deployments → ✏️ → New version → Deploy) or the live URL keeps serving the old code. The Web App URL itself doesn't change.
- **Edited `index.html`?** Just commit and push to GitHub — Pages updates automatically within a minute or two.

## How the street-highlighting feature works

Originally every report was just a colored circle. We later added a feature to highlight the actual street, and it's gone through a few iterations to get the rendering right. Every report is classified into one of six categories based on the **structure** of the address text:

| Category | Example | What gets drawn |
|---|---|---|
| `intersection` | "Blvd Pierrefonds / Blvd St Jean" | Both streets highlighted ±200m from the intersection point, forming a "+" shape |
| `address` | "500 Rue Westminster" | That one street highlighted ±80m around the geocoded point |
| `bare_street` | "Blvd Pierrefonds" (no house number) | The **entire run of that street within the reported city's boundary** — see below |
| `poi` | "Pierrefonds Comprehensive High School", "Maxi", "Parc Du Portage" | A 200m circle — no street to highlight |
| `postal_fsa` | "H9B" (3 characters) | A 200m circle — too vague an area for a line |
| `postal_full` | "H9B 1E6" / "H9B1E6" / "H9B-1E6" (6 characters) | A small 40m circle around the geocoded point — there's no street name attached to a postal code, so no line is possible |

**If anything fails along the way** (no road data found, name doesn't match, network error, timeout, unrecognized city) → the report **silently falls back to a circle**. Nothing ever breaks the map; worst case, you just don't get the bonus line.

### Why a "bare street" gets a totally different treatment

A report like "Blvd Pierrefonds" with no house number and no cross street doesn't tell us *where* along the street the flooding is — it could be anywhere along several kilometers. Rather than guess a point, the map looks up the **city** from the sheet's "Where is/was the flood" column, asks OpenStreetMap for every piece of road named "Blvd Pierrefonds" inside that municipality's actual administrative boundary, and highlights all of it. If the city value is missing, blank, or doesn't map to a known municipality, it safely falls back to the 200m circle instead.

The municipality name mapping lives in `CITY_TO_OSM_AREA_NAME` near the top of the Overpass section in `index.html`. It currently covers the six cities in the form's dropdown (Beaconsfield, Dollard-des-Ormeaux, Kirkland, Pierrefonds-Roxboro, Pointe-Claire, Dorval). Quebec municipal boundaries are tagged less consistently in OpenStreetMap than in some other provinces, so if a city's bare-street reports keep falling back to circles instead of getting the full highlight, this mapping is the first place to check — the OSM boundary name might not be exactly what's listed there.

### Why long streets need "stitching" before highlighting

OpenStreetMap routinely splits a single real-world street into many separate underlying pieces — this is standard, recommended OSM practice, not a data error (it commonly happens at every intersection, or wherever a tag changes). Early versions of this feature only highlighted whichever single piece happened to be closest to the report's point, which caused two visible bugs: streets only getting highlighted in one direction, and intersections only highlighting one of the two cross streets.

The fix: before trimming or drawing anything, the code finds every OSM piece with a matching name, and **stitches together any that share an endpoint** into one continuous line (handling pieces given in any order, and even pieces digitized in reverse direction, since OSM doesn't guarantee consistent direction). Only after stitching does it trim to the target distance (for `address`/`intersection`) or draw the whole thing (for `bare_street`).

### Why this runs in the browser, not in Apps Script

We originally tried doing the road lookup (querying OpenStreetMap's Overpass API) from inside `Code.gs`. This didn't work because Overpass's main server actively blocks requests coming from cloud-provider IP ranges (Google's servers fall into this category) as an anti-abuse measure — we consistently got HTTP 406 errors. Public Overpass mirrors we tried as a fallback were also frequently too slow or overloaded to respond in time.

Moving the lookup to the **visitor's own browser** sidesteps this, since each visitor's home/mobile IP isn't part of the blocked range. The tradeoff: each visitor's browser now makes its own small number of Overpass requests on page load. Fine at our current scale (~5 reports); could need rethinking if the map ever has dozens of simultaneous approved reports.

### Why there's a CORS proxy in the code

Even from the browser, some Overpass mirrors don't reliably send the right CORS headers back for arbitrary websites (this is inconsistent across mirrors — the main server supports it properly, but it's also the one blocking us for the IP reason above). To work around this, Overpass requests are routed through `corsproxy.io`, a free proxy service that explicitly supports GitHub Pages (`.github.io`) origins on its free tier.

**This is a real dependency on a third-party free service with no uptime guarantee.** If corsproxy.io ever goes down, changes its rules, or hits rate limits, the line-highlighting feature will stop working — but it will fail safely, falling back to circles, exactly like every other failure mode described above.

### The matching logic, briefly

- **Classification** is structural, not a keyword list: an address counts as a street reference if it starts with a house number, contains an intersection separator (`/`, `&`, "and"), or has a recognizable street-type word (Rue, Blvd, Boulevard, Avenue, Chemin, Place, etc.) as its **first or last word**. Anything else (school names, store names, parks, landmarks, in either language) is treated as a standalone place. This was a deliberate move away from an earlier keyword-list approach (`school|park|church|...`), which kept missing real submissions — French names ("Parc," "École"), specific store names ("Maxi," "Pharmaprix"), and anything else not predicted in advance.
  - The first-or-last-word restriction matters in practice: Quebec/West Island street names legitimately put the street-type word at either edge ("Rue Huron," "Place Bellerive," "Parc Lake Road"), but a street-type word stranded in the *middle* of a longer string is much more likely to be an incidental mention next to a business name than an actual street reference — e.g. "Esso Provi-soir boul gouin" or "Jean Coutu St Jean H9H" both contain a street-type word, but it's buried mid-string with other identifying words on both sides, so they correctly classify as `poi` rather than `bare_street`.
- Street names are normalized (lowercase, accents stripped, hyphens and slashes converted to spaces, "Boulevard" ↔ "Blvd," "Saint" ↔ "St," etc.) before comparing the form's address text against OpenStreetMap's road names, so "Blvd Pierrefonds" correctly matches "Boulevard Pierrefonds," and "Anselme-Lavigne" correctly matches as two separate words rather than one fused word.
- Small typos in submitted street names (e.g. "Lavigned" instead of "Lavigne") are tolerated using a word-by-word edit-distance check, without being loose enough to match genuinely different streets ("Thornhill" never matches "Thornton," "Westminster" never matches "Sommerset").
- Distance trimming uses the Haversine formula to walk along the real (stitched) road shape in both directions from the matched point until hitting the target distance (200m or 80m), interpolating the exact endpoint.
- If the closest road of any name is implausibly far away (>120m) for a single-address report, we don't use it — better to show a circle than a wrong line.

## Known limitations / things to keep an eye on

- **Geocoding** (turning an address into lat/lon) uses Google's Maps Geocoder from Apps Script — this part is reliable and unaffected by anything below.
- **Street-line highlighting** depends on two free, third-party services with no uptime guarantees (OpenStreetMap Overpass mirrors, and the corsproxy.io CORS proxy). Expect occasional periods where lines don't render and everything reverts to circles. This is by design, not a bug — the map keeps working either way.
- **City-boundary street highlighting** additionally depends on OpenStreetMap having a clean administrative boundary for the relevant municipality, and on `CITY_TO_OSM_AREA_NAME` mapping the form's dropdown value to the right OSM name. If a city is ever added to the form's dropdown, it needs a corresponding entry added to that mapping or its bare-street reports will just fall back to circles.
- If Overpass/corsproxy.io issues persist long-term, a fallback-only (circles everywhere) version of `index.html` would be simpler and have zero external dependencies beyond the Apps Script feed itself — worth keeping a copy of an older version if you ever want a "boring but bulletproof" fallback.
- Postal-code-only reports (3 or 6 character) never produce a line, by design — a postal code has no street name attached to match against.

## Quick troubleshooting

| Symptom | Likely cause |
|---|---|
| Map loads but shows no markers at all | Check `DATA_FEED_URL` is correct and the Apps Script is deployed with Access: Anyone |
| New approved rows don't get coordinates | Check the `onEdit` trigger exists (Triggers icon in Apps Script) and hasn't been deleted |
| Code changes don't show up on the live map | For `Code.gs`: did you create a **new deployment version**? For `index.html`: did GitHub Pages finish rebuilding (~1-2 min)? |
| Everything is circles, no lines | Normal under current conditions — check the browser console (F12) for Overpass/corsproxy errors. The map still fully works. |
| A business/landmark shows as a street line instead of a circle | Check `looksLikeStreetReference` in `index.html` — it only treats a street-type word as meaningful when it's the first or last word. If a business name happens to genuinely start or end with one ("... Place" or "... Court" as part of the business's actual name), it can still misclassify; worth a special-case check if it comes up. |
| A real street isn't getting a highlight at all | Less likely now, but check whether the street-type word in the submitted address falls in the *middle* of the string rather than at the start or end — that's treated as `poi` by design. If a legitimate street is ever phrased that way, it may need a small tweak to `looksLikeStreetReference`. |
| A bare street name isn't getting the full city highlight | Check the report's "Where is/was the flood" value matches one of the entries in `CITY_TO_OSM_AREA_NAME`, and that OSM actually has a clean administrative boundary for that city under that name |
| A long street only highlights part of its length | Should be fixed by the road-stitching logic, but if it recurs, the OSM pieces for that street may not share exact endpoint coordinates (rare, but possible with imperfect OSM data) |
