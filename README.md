Access the map HERE - > https://woahhjack.github.io/West-Island_Floods/


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
| `Code.gs` | Google Apps Script (attached to the Sheet) | Geocodes approved addresses, serves the JSON data feed |
| `index.html` | GitHub Pages repo | The actual map page everyone visits |

## Setting things up from scratch

1. **Google Sheet** needs these exact column headers in row 1 (spelling/spacing matters — the code matches on exact text):
   - `Street, Address, or Intersection / Rue, adresse ou intersection (...)`
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

Originally every report was just a colored circle. We later added a feature to highlight the actual street:

- **Intersections** (e.g. "Blvd Pierrefonds / Blvd St Jean") → both streets get highlighted ±200m from the intersection, forming a "+" shape.
- **Single addresses** (e.g. "500 Rue Westminster") → the one street gets highlighted ±80m around that point.
- **POIs / landmarks** (school names, parks, etc. — anything matching `school|high|park|centre|church|hospital|library|mall|arena|stadium`) → always stays a circle. These don't sit on a single named street, so line-matching is skipped entirely for them.
- **If anything fails** (no road data found, name doesn't match, network error, timeout) → the report **silently falls back to a circle**. Nothing ever breaks the map; worst case, you just don't get the bonus line.

### Why this runs in the browser, not in Apps Script

We originally tried doing the road lookup (querying OpenStreetMap's Overpass API) from inside `Code.gs`. This didn't work because Overpass's main server actively blocks requests coming from cloud-provider IP ranges (Google's servers fall into this category) as an anti-abuse measure — we consistently got HTTP 406 errors. Public Overpass mirrors we tried as a fallback were also frequently too slow or overloaded to respond in time.

Moving the lookup to the **visitor's own browser** sidesteps this, since each visitor's home/mobile IP isn't part of the blocked range. The tradeoff: each visitor's browser now makes its own small number of Overpass requests on page load. Fine at our current scale (~5 reports); could need rethinking if the map ever has dozens of simultaneous approved reports.

### Why there's a CORS proxy in the code

Even from the browser, some Overpass mirrors don't reliably send the right CORS headers back for arbitrary websites (this is inconsistent across mirrors — the main server supports it properly, but it's also the one blocking us for the IP reason above). To work around this, Overpass requests are routed through `corsproxy.io`, a free proxy service that explicitly supports GitHub Pages (`.github.io`) origins on its free tier.

**This is a real dependency on a third-party free service with no uptime guarantee.** If corsproxy.io ever goes down, changes its rules, or hits rate limits, the line-highlighting feature will stop working — but it will fail safely, falling back to circles, exactly like every other failure mode described above.

### The matching logic, briefly

- Street names are normalized (lowercase, accents stripped, "Boulevard" ↔ "Blvd," "Saint" ↔ "St," etc.) before comparing the form's address text against OpenStreetMap's road names, so "Blvd Pierrefonds" correctly matches "Boulevard Pierrefonds."
- Distance trimming uses the Haversine formula to walk along the real road shape in both directions from the matched point until hitting the target distance (200m or 80m), interpolating the exact endpoint.
- If the closest road of any name is implausibly far away (>120m) for a single-address report, we don't use it — better to show a circle than a wrong line.

## Known limitations / things to keep an eye on

- **Geocoding** (turning an address into lat/lon) uses Google's Maps Geocoder from Apps Script — this part is reliable and unaffected by anything above.
- **Street-line highlighting** depends on two free, third-party services with no uptime guarantees (OpenStreetMap Overpass mirrors, and the corsproxy.io CORS proxy). Expect occasional periods where lines don't render and everything reverts to circles. This is by design, not a bug — the map keeps working either way.
- If you ever add a new POI-type location (e.g. a community center with a name that doesn't include any of the current trigger words), add its keyword to the `isLikelyPOI` regex in `index.html` so it doesn't accidentally get treated as a street.
- If Overpass/corsproxy.io issues persist long-term, the fallback-only (circles everywhere) version of `index.html` is simpler and has zero external dependencies beyond the Apps Script feed itself — worth keeping a copy if you want a "boring but bulletproof" version to revert to.

## Quick troubleshooting

| Symptom | Likely cause |
|---|---|
| Map loads but shows no markers at all | Check `DATA_FEED_URL` is correct and the Apps Script is deployed with Access: Anyone |
| New approved rows don't get coordinates | Check the `onEdit` trigger exists (Triggers icon in Apps Script) and hasn't been deleted |
| Code changes don't show up on the live map | For `Code.gs`: did you create a **new deployment version**? For `index.html`: did GitHub Pages finish rebuilding (~1-2 min)? |
| Everything is circles, no lines | Normal under current conditions — check the browser console (F12) for Overpass/corsproxy errors. The map still fully works. |
| A landmark/POI shows as a line instead of a circle | Add its distinguishing keyword to the `isLikelyPOI` regex in `index.html` |
