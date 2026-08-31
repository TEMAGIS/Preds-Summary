# PREDS + Districts — forked viewer with boundary layers

A separate app forked from the PREDS Mobile viewer (`app-notes.md`), built 2026-08-31.
Same `index.html` + `sw.js` pair, same OAuth/offline/mission-scoping architecture —
adds TN House, TN Senate, and US Congressional districts, plus Census Tracts, as
toggleable map boundary layers with damage-count popups. Not yet deployed/hosted;
delivered to Tema as files, not yet pushed to a repo.

## Data sources
- **TN House**: `https://services1.arcgis.com/kILp9lqGUeOhnDbI/arcgis/rest/services/HouseDistricts_ExportFeature/FeatureServer/0`
  — token-secured (same org as the PDA feature service). Fields: OBJECTID, DISTRICT
  (zero-padded string, e.g. "01"), NAME (incumbent's title + name, e.g. "Representative
  Diana Harshbarger"), POPULATION, LongName ("District 1"), District_Filter
  ("District 01"), First_Year, Title, Leadership_Position, Committees, Shape__Area/Length.
- **TN Senate**: `https://services1.arcgis.com/kILp9lqGUeOhnDbI/arcgis/rest/services/TN_Senate_District/FeatureServer/0`
  — token-secured, identical schema shape to House (33 districts).
- **US Congressional**: `https://services1.arcgis.com/kILp9lqGUeOhnDbI/arcgis/rest/services/TN_Congressional_Districts_Update/FeatureServer/0`
  — token-secured. Only OBJECTID, DISTRICT (unpadded, e.g. "1"), NAME, POPULATION were
  confirmed present (9 districts) — no LongName/District_Filter seen, so the header
  builds off DISTRICT directly rather than assuming a longer name field exists.
- **Census Tracts**: `https://services.arcgis.com/P3ePLMYs2RVChkJx/arcgis/rest/services/USA_Census_Tracts/FeatureServer/0`
  — Esri's public "USA Census Tracts" Living Atlas layer, no token needed. Fields used:
  STATE_ABBR, STATE_FIPS, COUNTY_FIPS, TRACT_FIPS, FIPS, POPULATION (2022), SQMI.
  minScale 3,000,000 on the source layer itself; TN alone has 1,700+ tracts statewide.
- None of the three TN legislative/congressional layers could be introspected directly
  from the build sandbox (403 without an AGOL token — same situation the original PDA
  schema hit). Field names came from the user pasting sample query results/a fields
  list pulled while signed into AGOL, not a live fetch from this session.

## Design decisions (confirmed with user before building)
- **Separate app**, not a change to PREDS Mobile itself — forked from its `index.html`/`sw.js`.
- **Filter-panel toggle**, not a basemap-style cycle button or a checkbox menu — a new
  "Map boundaries" pill row in the existing filter drawer, one option active at a time
  (None / TN House / TN Senate / US Congress / Census Tracts), matching the drawer's
  existing distance-buffer row pattern exactly (same drawer-closes-on-pick behavior).
- **One boundary layer visible at a time** — selecting a new one always removes
  whatever was showing; no multi-layer overlay.
- **Full-breakdown popup** on tapping a boundary polygon: damage-level counts for
  all 5 levels (shown even at 0, plus an "Unclassified" row if any), *and* a secondary
  breakdown by dwelling type (top 6, "+N more types" beyond that), plus a total count
  and the boundary's own Census population (for tracts) where available.
- **Counts scoped to the currently selected mission** — reuses `scopedAssets()`, the
  same mission-scoping the rest of the app (list/map/tile-cache) already uses, not a
  cross-mission total.

## Implementation notes
- `BOUNDARY_LAYERS` config object (near `DAMAGE_LEVELS`) holds the 4 layers' URLs,
  field mappings, per-layer outline color (House `#1565C0`, Senate `#6D4C41`, Congress
  `#00838F`, Tracts `#616161` — chosen to stay visually distinct from the 5 damage-level
  colors already in use for markers), and `mode: 'full' | 'viewport'`.
- **House/Senate/Congress (`mode: 'full'`)**: fetched once via `.../query?...&f=geojson`
  (token-secured — reuses the same `ensureFreshToken()`/`arcgisFetch()` helpers the PDA
  fetch already has, including its 498/499 silent-refresh-and-retry handling) and cached
  in memory (`boundaryFullCache`) for the rest of the session — switching back to an
  already-viewed layer is instant, no refetch.
- **Census Tracts (`mode: 'viewport'`)**: no full-layer fetch at all — queries just the
  current map bounds (`geometryType=esriGeometryEnvelope`) plus `STATE_ABBR='TN'`,
  refetching on `moveend` (debounced 600ms) while the layer stays active. Gated behind
  a `minZoom: 10` — below that, a small on-map hint badge reads "Zoom in to load census
  tracts" instead of pulling a huge/pointless statewide set.
- **Point-in-polygon spatial join is hand-written** (`rayCastRing`/`pointInPolygonRings`/
  `pointInGeoJsonGeometry`), not a turf.js dependency — deliberate, since the app is a
  single self-contained file and this is the only thing that would've needed a geometry
  library. Standard even-odd ray-casting, handles Polygon + MultiPolygon and holes
  (a ring after the first subtracts from the outer ring rather than adding to it).
  Unit-tested against known square/hole/multipolygon fixtures during the build (all
  passed) — not yet tested against the real TN district/tract geometries themselves,
  since the three token-gated layers couldn't be queried from the build sandbox at all.
- Counts are computed **lazily per click** (via `computeBoundaryCounts()`, called from
  a popup content *function* so it's fresh every open), not precomputed for every
  district/tract up front — avoids an expensive full join across ~1,700 tracts × every
  assessment just to show one clicked boundary's numbers. A cheap bounding-box
  pre-filter (via a throwaway `L.geoJSON(feature).getBounds()`) skips the per-vertex
  ray-cast for assessments nowhere near the clicked polygon.
- Gotcha caught during the build: boundary pills initially reused the `.buft` class
  (same as the distance-filter pills) for visual consistency, but `setBuffer()`/
  `clearBufferFilter()` select `.buft` globally — clicking a distance filter would've
  visually cleared whichever boundary pill was active (the layer itself would've stayed
  on the map; just the pill's highlight would've looked wrong). Fixed by giving
  boundary pills their own `.bndt` class with identical CSS, rather than touching the
  existing buffer-filter code.
- The active boundary layer's "which one is showing" indicator reuses the existing
  active-filter chip bar (a chip reading e.g. "TN House" with an × to clear) rather than
  a separate on-map badge — one less new UI element, and it's the same clear-affordance
  users already know from the damage-level/distance chips.

## Still needed before this can go live
- **Hosting + OAuth redirect URI**: `CONFIG.redirectUri` in the delivered file still
  points at `https://temagis.github.io/Mobile_Preds/` (copied from the PREDS Mobile
  source) — needs updating to wherever this new app actually gets hosted, and that URL
  needs adding to the AGOL app item's Redirect URIs list (either the existing "Preds
  Mobile" Native Application item, or a new AGOL app item if this should be tracked
  separately from PREDS Mobile itself — not yet decided).
- **App identity**: title/branding was left as "PREDS" (unchanged from the fork) since
  renaming wasn't part of what was confirmed — worth deciding on a distinct name before
  this and PREDS Mobile are both live, so users don't confuse the two.
- **Live verification of the 3 token-gated layers**: House/Senate/Congress queries
  (including the `f=geojson` output format specifically) were never actually run
  against the live services from the build sandbox — no AGOL token available there.
  Census Tracts' `/query` endpoint *was* confirmed live and returning real polygon data
  with the expected fields, but not specifically with `f=geojson` (WebFetch's response
  summarization wasn't reliable enough to confirm the exact output format byte-for-byte;
  `f=geojson` itself is a long-standing, well-documented ArcGIS REST parameter, so this
  is a low-risk assumption, not a known issue — but worth a quick real-device check
  after deploying, same as any new feature).
- Repo/GitHub Pages setup for this new app not yet done — see "Hosting / deploy notes"
  in `app-notes.md` for the GitHub Pages timeout workaround if the same org is used.
