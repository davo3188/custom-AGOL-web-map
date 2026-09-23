# Geoportale · AxpoSolar Italia

A single-file web GIS workbench that sits alongside the corporate **ArcGIS Online** portal
(`sig-urbasolar.maps.arcgis.com`) and adds the one thing AGOL cannot do out of the box:
**search an Italian cadastral parcel by _Comune + Foglio + Particella_ and zoom to it.**

Everything lives in one file — [`geoportale_axpo.html`](geoportale_axpo.html) (~340 KB, ~4,500 lines, SDK 4.34,
no build step, no bundler, no `node_modules`). Open it over HTTPS, log in with your org account, and it
works.

---

## 1. Why this exists

The prospecting team was using **FoMaps**, an external third-party site, to look up parcels — then
manually re-finding the same location in the AGOL maps. Two problems: an outside dependency in the
middle of the workflow, and no way to carry the result back into the corporate data.

The obvious in-portal fix does not work. The **Map Viewer Search widget can only query a geocoder or a
feature layer already in the map** — never an arbitrary REST API. Three options were evaluated:

| Option | Verdict |
|---|---|
| **A.** Publish a hosted layer of parcel centroids and use native Search | Rejected — national cadastre is ~22 M parcels / ~22 GB; extraction alone is days of credits |
| **B.** A standalone companion web page next to the portal | **Chosen** |
| **C.** A custom Experience Builder widget | Rejected — requires the self-hosted Developer Edition, which the org does not have |

Option B also turned out to be worth more than the search itself: because the page holds a real ArcGIS
session, it can read the corporate web maps, draw against them, and **write results back** into the
production layers.

## 2. Finding a usable cadastral source

The Agenzia delle Entrate (AdE) publishes the cadastre, but not in a form a browser can query:

- **AdE INSPIRE WFS** (`wfs.cartografia.agenziaentrate.gov.it/inspire/wfs/owfs01.php`) — no auth,
  EPSG:6706, GML tags prefixed `CP:`. But it accepts **BBOX queries only**: `FILTER` by attribute and
  `GetFeatureById` are both rejected, and there is **no CORS**. Unusable for "find parcel X".
- **AdE official viewer** — has exactly the search we want, and it is protected by a **CAPTCHA by
  design**. Closed, not worked around.
- **AdE bulk download** (`GetDataset.php?dataset=<REGION>.zip`, + `ITALIA.zip`, 13.9 GB) — no auth,
  **CC-BY 4.0**, refreshed twice a year. This is the sovereign source and the guaranteed fallback, but
  it is a pipeline, not a lookup.
- **[Zornade API v2](https://api.zornade.com/api/v2)** — the one source callable straight from a
  browser (**`CORS: *`**), 10,000 req/h, `x-api-key` header. Same AdE cartography underneath:
  spot-checked against the WFS on four live prospecting areas, **4/4 exact matches**.

Zornade is what the tool calls. The AdE bulk route stays documented as plan B.

## 3. Quick start

The page needs HTTPS **even locally**: the ArcGIS OAuth redirect is `https://`, and Chrome blocks an
`https → http://localhost` redirect with `ERR_UNSAFE_REDIRECT`. Opening the file via `file://` does not
work either.

1. Serve the folder over HTTPS on port **8443** (a self-signed local cert is fine):
   `python scripts/serve_https_catasto.py`, then open `https://localhost:8443/geoportale_axpo.html`.
2. Register `https://localhost:8443/geoportale_axpo.html` as a **Redirect URI** on the portal OAuth app
   (`CLIENT_ID` is already wired in the source — a client ID is public by design; the client *secret*
   is never used here).
3. Open the page and sign in. It prints the exact redirect URI it is using, right under the login
   button, so a mismatch is self-diagnosing.

**Login is mandatory, not optional.** The Check Vincoli maps and AREAS COLLECTION are shared *to
groups*, not publicly — you need a session just to **see** them, well before any write.

`?noauto` in the URL skips auto-login (useful when testing). Note that a plain static file server plus
browser cache will happily serve a stale copy after an edit — add a cache-buster when verifying
changes.

## 4. What it does

**Parcel search**
- By *Comune / Foglio / Particella*, with an administrative cascade (region → province → municipality).
- By clicking the map, or by drawing a point, rectangle, polygon or line over an area.
- From a geometry you already drew.

Because Zornade exposes **no spatial query**, area selection works by sampling a grid of points inside
the geometry and calling `/parcels/locate` on each. It is bounded by explicit caps and is slow on large
areas — parcels smaller than the grid step can be missed. **Esc** interrupts it: the first press stops the
scan and still adds the parcels already found, a second one stops that too. Every Zornade call gives up
after 20 s with a message, instead of leaving the UI waiting forever.

**Drawing & geoprocessing**
- Native `SketchViewModel`: snapping, rectangle, circle, live distance/angle readouts and typed input
  (90° = perpendicular).
- Client-side geoprocessing via turf on **individual objects**, not whole categories. Two slots, **A** and
  **B**, are filled by clicking objects on the map (parcels, drawings, imports, earlier results — click
  again on the same spot to step down through overlapping objects), from the rows ticked in the list,
  or with a whole category. On A: buffer (one per object, or merged), union, convex hull, simplify, clip
  against a polygon drawn on the spot; between A and B: intersect, difference. Every result lands in
  *Geoprocessi* with its area and can go back into A or B.
- Per-geometry **colour and visibility**, plus per-section colour/visibility, all persisted.

**Site context**
Nearby substations (380/220/150/132 kV), HV lines and PV projects, pulled live from the portal layers.

**Import**
- Vectors: GeoJSON, KML, KMZ, SHP.
- Georeferenced **images and GeoTIFF** — images get four draggable corner handles, GeoTIFFs are
  auto-placed from their bounding box and geokeys, reprojected into the view SR.
- **CAD drawings (DWG)**, read in the browser — the file never leaves the PC. Layers and colours are
  kept (ByLayer/ByBlock resolved, blocks and arrays exploded, ACI 7 flips black/white with the basemap).
  The spatial reference is **estimated from the coordinates** among the systems used in Italy (the eight
  company ones — RDN2008 and WGS 84 UTM 32/33/34, Monte Mario Italy 1/2 — plus ETRS89, IGM95, ED50, Web
  Mercator and the geographic ones) and confirmed by the user in a dialog, which
  shows where the drawing lands (comune and region when logged in). Drawings in local coordinates — and
  many layouts arrive that way — are placed by hand with a **two-point alignment** (like AutoCAD's
  ALIGN): a point on the drawing, where it really is, twice; vertices snap to the drawing, to loaded
  parcels, drawings and other DWGs. The placement is remembered per file and offered again next time.
  The drawings themselves are session-only.
- **Coordinates whose CRS is unknown** — a decree, a survey, an email, a shapefile with no `.prj`.
  Paste the pairs (decimal comma, thousands separators, DMS and a leading label are all understood)
  and the tool projects them through the Italian systems, keeping the ones that land in Italy and
  showing *where* each one falls, X/Y swapped included. With **📍 Dove dovrebbe stare** you click the
  real location on the map (snapping to parcel and drawing vertices) and the candidates are ranked by
  error in metres — precise enough to tell ED50 from WGS 84 inside the same UTM zone (≈200 m apart).
  The result goes into *Importati* as points or one closed area.
- **Files without a declared CRS** (a shapefile with no `.prj`, a GeoJSON in projected coordinates) go
  through the same engine. The file is converted automatically only when Web Mercator is the *only*
  reading that lands on Italian soil; otherwise the import stops, the candidates appear in the
  coordinates module, and the whole file is reprojected with the system the user picks.
- The **top search box** takes addresses and coordinates: `lat, lon` (decimal comma and DMS included)
  or projected `X Y`, whose system it guesses and flies to. A coordinate pair wins over the geocoder;
  anything with letters in it (an address with house number and postcode) goes to the geocoder.

**Export**
GeoJSON, KML, KMZ, SHP, XLSX, with optional dissolve. Names you assign in the UI end up in the file.
The shapefile comes as one zip with a layer per geometry type (`_punti`, `_linee`, `_aree`).

**Write-back to the portal**
- Parcels → **AREAS COLLECTION** (layer 426). REGIONE and PROVINCIA are the parcel's own (Zornade names,
  the same spelling already in the layer), COMUNE and PRO_COM_T come from ISTAT, the area is geodesic —
  the same number the list shows.
- Drawings, buffers, geoprocessing output and imports → **Sites Notes**. Each drawing keeps the category
  it was drawn with; the dialog's menu only applies to objects without a valid one (buffers, results,
  imports).

**Map & reporting**
- Loads the 17 regional *Check Vincoli* web maps (16 of 20 regions covered) plus the General Map.
- LayerList with per-layer legend, transparency, popup toggle and drag-and-drop reordering.
- Bookmarks that also restore **layer visibility** (Esri's own bookmarks do not).
- Right-click context menu acting on the clicked point: add parcel here, site context here, buffer here,
  copy coordinates, open in Google Maps, centre here.
- Parcel labels on the map, hidden below 1:150,000; the on/off choice is remembered.
- Measurement, coordinates + pin, scale bar, elevation profile.
- Printing through the Esri Print widget. The custom A3 PDF site report (jsPDF) was **removed on
  2026-09-22** — it never worked reliably and will be redesigned; the old code is in
  `geoportale_axpo_pre-bugfix-export-pdf-geoproc_2026-09-22.html`, inside `backups/archive_2026-09-22.zip`.
- Auto-resume from `localStorage`, guided spotlight tour, per-icon tooltips.
- **Browser storage** panel (in *Impostazioni avanzate*): shows what is saved, *Svuota il lavoro*
  (parcels, drawings, results, imports, DWGs, added layers — preferences kept) and *Ripristina tutto*
  (every key of the app, then reload). DWGs are never saved; `localStorage` is capped at ~5 MB per
  site, and hitting the cap now shows a warning instead of failing silently.

## 5. Architecture notes

Single file, ArcGIS Maps SDK for JS **4.34** from CDN (moved from 4.30 on 2026-09-23), Axpo-branded fork
of an earlier Leaflet prototype. The parts that are non-obvious and worth knowing before editing:

**SDK 4.34, and the road to 5.** 4.34 is the last 4.x with the AMD loader (`require`); from 5.0 the CDN
serves ES modules only, and most widgets become web components. What the move to 4.34 required:
- **Saved bookmarks** were restored with `new Bookmark({viewpoint: <JSON>})`. 4.30 guessed the type of
  the plain `targetGeometry`; 4.34 does not, so the bookmark came back with no position (and an
  `Invalid property value` error at load). They now go through `Viewpoint.fromJSON`, like the saved view.
- `obj.watch()` (deprecated in 4.32) → `reactiveUtils.watch`; `Polygon.centroid` (deprecated in 4.34) →
  `centroidOperator` (`watchProp` / `centroidOf` in the source).
- Widget roots now carry their `esri-*` class on the container itself (`#scaleBarBox.esri-scale-bar`),
  so CSS written as `#box .esri-…` may need both forms.
- **Dark theme and Calcite.** Widgets built from Calcite components (LayerList, Bookmarks, Editor,
  ElevationProfile…) follow the `calcite-mode-dark` / `calcite-mode-light` class, not Esri's dark CSS:
  without it they stayed white in dark mode — already true on 4.30 — and our light text in the layer
  panel was unreadable. The theme switch now sets the class on `<html>`.

Still deprecated, and left for the 5.x migration because it means rewriting rather than renaming: the
widgets (BasemapGallery, LayerList, Legend, Bookmarks, Search, Print, ScaleBar, Measurement, Editor,
ElevationProfile → `<arcgis-…>` components) and the `projection` / `geometryEngine` modules
(→ `projectOperator`, `geographicTransformationUtils` and the other geometry operators). They only
print one-off warnings in the console.

**AMD/UMD collision.** The ArcGIS SDK ships the Dojo AMD loader, so `define` exists. Every UMD library
(`xlsx`, `jszip`, `shp-write`, `turf`, `togeojson`, `shpjs`, `geotiff`) must be loaded inside
the `window.define = undefined` block near the top, or it registers as an anonymous AMD module instead
of attaching to `window`.

**OAuth uses the implicit flow.** The registered app has a confidential client secret, and the
authorization-code token exchange fails silently in the browser. Implicit returns the token in the
hash — no exchange, no secret. Appropriate for an internal tool.

**Writing to AREAS COLLECTION is not a plain `applyEdits`.** The layer is in **wkid 6876** (RDN2008), so
geometry must be reprojected 4326 → 6876. GeoJSON rings are counter-clockwise and Esri wants clockwise:
without `geometryEngine.simplify` first, features land with `isSimple: false` and the popup's Arcade
`Intersects()` expressions silently fail. `PRO_COM_T` and `COMUNE` are filled from an ISTAT lookup on
the centroid because the Arcade seismic-zone expression depends on them. REGIONE and PROVINCIA used to
come from the region/province drop-downs, which are wrong for parcels clicked elsewhere and empty in a
dissolved promotion: they now come from the parcel (Zornade's names — the spelling already used in the
layer, e.g. `Forli'-Cesena`, `Massa Carrara`), with the drop-downs only as a last resort. The `area`
field is geodesic: planar area in 6876 (central meridian 12°) drifts by up to 0.7% at the edges (Lecce).
The recalculation after an edit in the Editor asks the server for WGS 84 geometry to do the same.

**`queryFeatures` with a plain query object and no `outSpatialReference` returns the layer's native SR**,
even when the layer is in a Web Mercator view (verified). Ask for the SR you need explicitly.

**Sites Notes is three layers, not one** — 0 Points / 1 Lines / 2 Areas, each with its **own** coded-value
domain on `CATEGORIA`. There is no name field, so the geometry name goes into `NOTE`. The point domain's
code for *Beni interesse culturale* has a **trailing space** (`'Beni interesse culturale '`): the app
matches categories ignoring spaces, accents and case and always writes the exact domain code. The
domains are read from the service when the dialog opens; the copy in the source is only a fallback.

**The Search widget puts its default sources first.** With "All", it takes the first result in source
order, and `defaultSources` always come before `sources` — so the geocoder won every time: `2789113
4471597` went to Portugal (read as postcode 4471-597), `41,9028 12,4964` to Greece. The coordinate source is
now put first through `includeDefaultSources` as a function, and it only answers when the text is *just*
a pair of numbers. A custom source's result must carry a real `Extent`: with a plain object plus a
`{target, zoom}` the widget drew the marker but never moved the map (hidden until then, because the
geocoder always won).

**Cadastral WMS spatial reference.** Esri tiled basemaps force wkid 102100, which the AdE GeoServer
refuses; the WMS layer needs an explicit `spatialReferences: [3857]`. The same trap applies to the
regional WMS layers inside the Check Vincoli maps.

**Do not add the internal cadastral WMS blindly.** Several web maps already carry their own cadastre.
The loader matches **by item ID** (national item `382d9c13…`), brings the existing layer to the top and
turns it on; only if none is found does it add the internal WMS. Matching by title would be wrong —
there are ~20 regional and provincial cadastres with similar names.

**Zornade's `area_m2` is computed in Web Mercator** and is inflated by roughly 1/cos²(lat) — ×1.85 at
Rome's latitude. Never surface it as a real area; the tool computes
`geometryEngine.geodesicArea(poly4326, 'square-meters')` instead.

**Calcite quirks (seen on SDK 4.30; the workarounds still hold on 4.34).** `ListItemPanel` needs `className: 'esri-icon-*'`; passing `icon:` renders
an invisible button. `bookmark-select` does **not** fire when you click a bookmark in the list — the fix
overrides `viewModel.goTo`. And a synthetic `el.click()` on a `calcite-list-item` selects nothing, so
Calcite UI can only be validated with a real mouse.

**GraphicsLayers cannot label themselves** (`labelingInfo` is a FeatureLayer feature), so parcel labels
are individual `TextSymbol` graphics on a dedicated layer.

**`map.allLayers` includes the basemap and the ground.** Both had to be filtered out of the bookmark
capture (and of the former PDF legend).

**Ring orientation, everywhere.** ArcGIS wants exterior rings **clockwise** and holes counter-clockwise;
GeoJSON (RFC 7946) wants the opposite; shapefiles follow ArcGIS. Until 2026-09-22 the GeoJSON → ArcGIS
converters copied rings as they were, so every imported polygon, parcel and turf result was *reversed*
in ArcGIS terms: a negative area, and a 50 m buffer of a 17.3 ha square came out at 19.1 ha instead of
26.4. `esriRings` now decides exterior vs hole by nesting (even/odd, from a majority of sample vertices,
so parts touching at one vertex are not mistaken for holes) and orients accordingly; the other way
round, `esriGeomToGeojson` nests with `cadGjPolys` and emits a real `MultiPolygon` (it used to put every
part into one `Polygon`, which turned parts 2..n into holes).

**Guessing a CRS from coordinates.** One table, `SRS_IT` (23 systems, `co` marking the eight company
ones), feeds the DWG estimate, the confirmation dialog's menu and the "Trova SR" module; `srEstimate`
projects the point through each of them, keeps what lands in Italy, and groups the candidates by
resulting position — RDN2008, ETRS89, WGS 84 and IGM95 in the same zone are the *same point* (< 1 m)
and no amount of arithmetic can separate them, so they are offered together and the user picks the
datum. Without a known location the groups are ranked by plausibility (comune from ISTAT when logged
in, otherwise the embedded Italy outline, then closeness to the current view and whether the longitude
sits in the zone's own band); with one, by geodesic error, per candidate, which does separate ED50 from
WGS 84. Everything runs in the ArcGIS projection engine (WebAssembly, offline): the two public tools
that do this either ship a proj4 table of their own (projection.dogeo.fr, 528 systems, almost nothing
for Italy) or send the coordinates to a cloud function (ihatecoordinatesystems.com, ~6,000 EPSG
systems via pyproj, up to 30 s). Measured in the browser: 2,000 wkids projected in 0.09 s, so a
brute-force pass over all ~6,400 projected CRSs the engine knows would take seconds — worth adding
only if the Italian table ever falls short. What it cannot resolve: **Cassini-Soldner cadastral
coordinates**, whose hundreds of local origins are not in the EPSG registry.

**Files without a CRS: never trust "it lands in Italy".** The old import rule converted from Web Mercator
whenever the *first* point, read as Web Mercator, fell in the box 6–19°E / 35–47.5°N. Plenty of UTM and
Gauss-Boaga Ovest coordinates do: a UTM 33 point in Lecce ended up in the sea between Sardinia and
Algeria, a Gauss-Boaga Ovest point in Sassari on dry land in Sicily. Now the median point goes through
`srEstimate`, and only a file whose sole plausible reading (on land, or in a comune when logged in) is
Web Mercator is converted without asking.

**`@mapbox/shp-write` 0.4.3.** `download()` is a no-op in this build: use `zip()` and save the blob. It
silently drops `MultiPolygon` and `MultiPoint` features, and gives every geometry type the same file
name unless `types` is passed — `shpPrep` flattens multi-geometries and orients rings clockwise first.

**Saved symbols are Esri JSON** (`esriSFS`…), which the `Graphic` constructor does not autocast: restored
graphics used to come back with default colours. They go through `symbols/support/jsonUtils.fromJSON`.

**`SketchViewModel.updateOnGraphicClick`** (on by default) grabs any click on a drawing to start editing
it. Map-picking modes (geoprocessing slots, DWG alignment) pause it and restore it on exit
(`sketchClickPause` / `sketchClickResume`), or the first click on a drawing would end the mode.

**DWG import.** The reader is `@mlightcad/libredwg-web` 0.7.14 — LibreDWG compiled to WebAssembly,
~9.5 MB, lazy-loaded from jsDelivr on the first DWG. It is an ES module, loaded with dynamic `import()`,
so it never touches the AMD loader. Worth knowing before editing:
- **One fresh WASM instance per file.** In 0.7.14 the *second* `convert()` on the same instance fails
  with `RuntimeError: null function`, with or without `dwg_free`. The compiled module stays cached, so
  a new instance costs ~50 ms — but each asks for 1 GB of initial memory (`INITIAL_MEMORY=1GB` in
  their build), which can fail on a memory-starved machine.
- **No DXF.** The DXF branch of their `dwg_read_data` is commented out upstream.
- **Data model.** Angles are radians; an entity's true colour (`color`) beats its ACI index
  (`colorIndex`: 0 = ByBlock, 256 = ByLayer); a closed LWPOLYLINE is `flag & 512` (DWG encoding, not
  DXF's `& 1`); OCS entities with extrusion Z < 0 are mirrored on X. The DWG version is read from the
  first six bytes of the file (`AC1032` = 2018+), because the converted header leaves `ACADVER` empty.
- **SR estimation.** A DWG almost never declares its CRS. The robust centre (median, so a title block
  at the origin does not move it) is projected into each company SR. The trap is that the same UTM X
  read in the *adjacent* zone still lands inside Italy's bounding box, so candidates are ranked by land
  vs sea — ISTAT comuni when logged in, otherwise an embedded coarse outline of Italy (~6 KB) — then by
  proximity to the current view and whether the longitude sits in the zone's own band. WGS 84 and
  RDN2008 UTM coincide within 1 m, so that choice is remembered, not estimated. For Monte Mario the code
  asks `getTransformation` for the drawing's area, but the engine (checked on 4.30) returns **1660 everywhere**,
  Sardinia and Sicily included (1661/1662 are listed but never chosen); and `projection.project` without
  a transformation already applies that same default, so GeoTIFFs in Monte Mario land where DWGs do.
  A drawing in mm or cm is caught by retrying the estimate at 1/1000 and 1/100.
- **Stray geometry.** Elements more than 50 km from the median are excluded by default (a checkbox in
  the dialog), and so is a cluster around (0,0) that is detached from the drawing — title blocks,
  source blocks, and the items of AutoCAD **associative arrays**: libredwg-web returns them in model
  space at their array-local positions, because the array's container insert is not read. On the first
  real layout tested, the 45 trees of one associative array came out at the origin this way.
- **What libredwg-web returns that needs care.** Block attributes appear twice — inside the INSERT and
  again as loose ATTRIB entities in model space: only the INSERT copy is used. PVcase tracker blocks
  hold the modules twice: as 3DSOLIDs (ACIS, not drawable) and as 3D polylines (drawn). MULTILEADER
  callouts are rebuilt from `textContent`, `textAnchor` and `leaderSections`.
- **Rendering: GeoJSONLayers, not GraphicsLayers.** Each CAD layer is a `GroupLayer` holding up to three
  `GeoJSONLayer`s (fills, lines, points) built in memory as blob URLs. The first version used one
  `GraphicsLayer` per CAD layer: on a real 78,000-element layout the JS heap went from 200 MB to 1.3 GB
  and the map dropped to 1 frame per second on an integrated GPU, even after the drawing was removed.
  With GeoJSONLayers the features live in the ArcGIS worker: two real layouts together sit at ~360 MB and
  60 fps. Colours live in a unique-value renderer keyed by style, so recolouring a layer or flipping
  ACI 7 on a basemap change rebuilds renderers, never features.
- **Two ring-orientation traps.** (1) Fills are projected as *polylines*: for ArcGIS a counter-clockwise
  ring is a hole, and a polygon made only of "holes" is the whole world minus itself — the projection
  engine returns it with vertices at (-180,-90). (2) GeoJSON wants exterior rings counter-clockwise and
  holes clockwise (RFC 7946), so rings are re-oriented and nested even/odd, like AutoCAD's hatch rule.
- **Texts are drawn on the fly** into one `GraphicsLayer`: only visible layers, only the current
  extent, at most 1,500, and at real size — the font comes from the text height on the ground, and a
  text that would be under 5 px is skipped, as in CAD. At a fixed size a dense layout at 1:4,500 was a
  wall of labels.
- **Local coordinates and the two-point alignment.** Layout drawings are often not georeferenced at all
  (the first real one spans X 2,742–3,360). "Position by hand" drops the drawing at the map centre in
  the RDN2008 UTM zone of the area, then the alignment computes rotation and translation (scale locked
  1:1 by default, or free) on the midpoints of the two pairs, in the file's metric working SR, and
  re-places everything. Verified on real data: a 2023 layout in local coordinates aligned onto the 2026
  georeferenced one with 0.03° of rotation and 2.7 cm of misfit. Each file carries its own affine
  matrix (`f.ch.M`), saved in `localStorage` under name + size, so a re-import offers it again.
- Each DWG group is re-attached on every web map swap (`cadReattach` in `loadWebMap`), with the text
  layer. A layer's colour in the list is the most frequent colour of its entities, not its table
  colour, because that is what the user actually sees.

## 6. Security & distribution

- The Zornade key is a read-only token **embedded in the source** (`DEFAULT_KEY`). This is a deliberate,
  accepted choice for an internal, access-controlled host — and it is why the page must not go on GitHub
  Pages or any public URL. In the longer run the key should leave the source (an Azure Function in
  front of Zornade on the future host would keep it server-side).
- PV project fields are deliberately limited to non-sensitive ones (`plant_name`, `capacity_mw`,
  `status`, `procedure_type`). Commercial licensing on the elemens data means **internal use only, no
  third parties** — never expose SPV names, VAT numbers, PPA or tariff fields.
- The DWG reader is LibreDWG, licensed **GPL-3.0**, downloaded from jsDelivr on first use. Use inside
  the organisation is generally not "distribution" under the GPL, but it belongs in the dependency list
  given to the Digital committee — and a Content-Security-Policy on the future host must allow
  `cdn.jsdelivr.net` and WebAssembly compilation.
- Intended hosting: **Azure Static Web App behind Entra ID** (the org already runs on Entra). Add that
  host's URL as a redirect URI on the OAuth app before sharing.
- For a quick hand-off to a colleague: zip the HTML with the local HTTPS server script — the
  `localhost:8443` redirect is already registered and works from any machine, and they sign in with
  their own org account.

## 7. Audit status — 2026-09-22

Full read of the file plus checks in the browser on a local copy, **not logged in**. The two promotions
were exercised against a simulated service (the real layers were not written to); everything else ran
against the real Zornade API and the real SDK. Backup of the version audited:
`geoportale_axpo_pre-fix-audit-settembre_2026-09-22.html`, inside `backups/archive_2026-09-22.zip`.
(Backups of closed days are zipped per day by `scripts/archivia_backup.py`; only the current day's stay
loose in `backups/`.)

**Clean**

| Check | Result |
|---|---|
| Console at load | 0 errors; on 4.34 only the one-off deprecation warnings listed in §5 |
| Duplicate element IDs | 0 of 249 |
| Functions, variables, CSS ids never used | 0 (dead code removed on 2026-09-23) |
| Unresolved `$('id')` DOM references | 0 |
| Insecure `http://` endpoints | none (only XML namespace URIs) |
| Third-party library versions | all pinned |

**Fixed in this round** (each one reproduced first, then re-tested)

- Files without a `.prj` silently misplaced by the Web Mercator rule (see §5). The 2026-07-29 fix had
  already been "worse than the bug" once; its "lands in Italy" rule was still wrong.
- AREAS promotion: REGIONE/PROVINCIA from the drop-downs → from the parcel; area planar 6876 → geodesic.
- Sites Notes promotion: per-drawing category lost, four domain values missing from the dialog, the
  trailing-space domain code never matched.
- Top search box: the geocoder hijacked coordinate searches, and coordinate results never moved the map.
- Changing the colour or visibility of the parcel section (or removing a parcel) moved the map.
- "Trova SR" read `41.902` as forty-one thousand (single separator now means decimal).
- Search markers piled up in `toolsGL`, invisible in the list and impossible to remove.
- From the 2026-08-06 list: Zornade timeout (20 s), cancellable area selection (Esc), AREAS Editor
  destroyed before being recreated, `esc()` escapes quotes, label toggle persisted, substations queried
  with their two fields only, OAuth comment and the HTTPS script pointing at the current file name.
- Zornade names (regions, provinces, comuni) escaped before going into the HTML; province/comune lists
  ignore a stale answer when the user has already picked something else.
- 2026-09-23: **SDK 4.30 → 4.34** (see §5: saved bookmarks, `watch`, `centroid`, Calcite dark mode) and
  **dead code removed** — the old drawn-buffer mode (`bufPoint`…, `offBuf`, the `'buffer'` sketch purpose
  and banner), the hidden `toolsToggle`/`toolsHead` and `#sideOpen`, `gpSlotGeoms`, `PARCEL_SYMBOL`,
  `AXPO_LOGO_RATIO`, `bufferGeom`, `layerListExpand`, the unused `esri/config` and `Expand` modules, CSS
  for elements that no longer exist (`#sideTabs`, `#listHead`, `.wordmark`, the old Expand search) and the
  bottom-centre elevation dock rules that the bottom-left ones overrode. Backup before the change:
  `geoportale_axpo_pre-434-codice-morto_2026-09-23.html` (in `backups/`, then in `archive_2026-09-23.zip`).

**Checked and not a problem** (do not reopen): `projection.project` applies the default datum
transformation by itself, so Monte Mario GeoTIFFs are placed right; the AREAS `edits` handler gets the
layer's own SR from `queryFeatures`, not Web Mercator.

**Open, known, non-blocking**

1. Third-party scripts from three CDNs (five on unpkg) **without Subresource Integrity**, on a page that
   holds a portal OAuth token. Best fixed by hosting the libraries on the future Azure site.
2. **SDK 5.x.** The page is on 4.34, the last AMD release. 5.x on the CDN is ES modules only — no
   `require` — so moving to 5 means rewriting module loading and replacing the widgets with components.
3. The **Legend** widgets created inside LayerList panels are never destroyed on a web map swap.
4. `deleteEnabled: true` on the AREAS editor: the Esri Editor asks its own confirmation before deleting;
   check it when logged in before adding one of ours.
5. **About 160 empty `catch` blocks.** Errors vanish silently — e.g. a Union that fails on one piece
   skips it without saying so.
6. Not tested logged in: Check Vincoli maps, the real writes to AREAS and Sites Notes, the AREAS editor,
   site context, ISTAT zoom; DWG alignment with a real mouse; Print (CORS error from localhost). On 4.34
   in particular the logged-in branch has not been exercised at all.

## 8. Repository contents

```
geoportale_axpo.html   the whole application
README.md              this file
```

There is no build, no test suite and no CI. Verification is done in the browser against the live
portal, which is the only place the corporate layers and OAuth actually exist.
