# Geoportale · AxpoSolar Italia

A single-file web GIS workbench that sits alongside the corporate **ArcGIS Online** portal
(`sig-urbasolar.maps.arcgis.com`) and adds the one thing AGOL cannot do out of the box:
**search an Italian cadastral parcel by _Comune + Foglio + Particella_ and zoom to it.**

Everything lives in one file — [`geoportale_axpo.html`](geoportale_axpo.html) (~218 KB, ~3,200 lines,
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

1. Serve the folder over HTTPS on port **8443** (a self-signed local cert is fine).
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
areas — parcels smaller than the grid step can be missed.

**Drawing & geoprocessing**
- Native `SketchViewModel`: snapping, rectangle, circle, live distance/angle readouts and typed input
  (90° = perpendicular).
- Client-side geoprocessing via turf: buffer, union, clip, intersect, difference, convex hull, simplify.
- Per-geometry **colour and visibility**, plus per-section colour/visibility, all persisted.

**Site context**
Nearby substations (380/220/150/132 kV), HV lines and PV projects, pulled live from the portal layers.

**Import**
- Vectors: GeoJSON, KML, KMZ, SHP.
- Georeferenced **images and GeoTIFF** — images get four draggable corner handles, GeoTIFFs are
  auto-placed from their bounding box and geokeys, reprojected into the view SR.

**Export**
GeoJSON, KML, KMZ, SHP, XLSX, with optional dissolve. Names you assign in the UI end up in the file.

**Write-back to the portal**
- Parcels → **AREAS COLLECTION** (layer 426).
- Drawings, buffers, geoprocessing output and imports → **Sites Notes**.

**Map & reporting**
- Loads the 17 regional *Check Vincoli* web maps (16 of 20 regions covered) plus the General Map.
- LayerList with per-layer legend, transparency, popup toggle and drag-and-drop reordering.
- Bookmarks that also restore **layer visibility** (Esri's own bookmarks do not).
- Right-click context menu acting on the clicked point: add parcel here, site context here, buffer here,
  copy coordinates, open in Google Maps, centre here.
- Parcel labels on the map, hidden below 1:150,000.
- Measurement, coordinates + pin, scale bar, elevation profile.
- **A3 landscape PDF site report** (jsPDF): WYSIWYG map screenshot, legend built from visible layers
  only, north arrow, a scale bar computed from real ground distance, coordinates, and a parcel table.
- Auto-resume from `localStorage`, guided spotlight tour, per-icon tooltips.

## 5. Architecture notes

Single file, ArcGIS Maps SDK for JS **4.30** from CDN, Axpo-branded fork of an earlier Leaflet
prototype. The parts that are non-obvious and worth knowing before editing:

**AMD/UMD collision.** The ArcGIS SDK ships the Dojo AMD loader, so `define` exists. Every UMD library
(`xlsx`, `jszip`, `shp-write`, `turf`, `togeojson`, `shpjs`, `geotiff`, `jspdf`) must be loaded inside
the `window.define = undefined` block near the top, or it registers as an anonymous AMD module instead
of attaching to `window`.

**OAuth uses the implicit flow.** The registered app has a confidential client secret, and the
authorization-code token exchange fails silently in the browser. Implicit returns the token in the
hash — no exchange, no secret. Appropriate for an internal tool.

**Writing to AREAS COLLECTION is not a plain `applyEdits`.** The layer is in **wkid 6876** (RDN2008), so
geometry must be reprojected 4326 → 6876. GeoJSON rings are counter-clockwise and Esri wants clockwise:
without `geometryEngine.simplify` first, features land with `isSimple: false` and the popup's Arcade
`Intersects()` expressions silently fail. `PRO_COM_T` and `COMUNE` are filled from an ISTAT lookup on
the centroid because the Arcade seismic-zone expression depends on them.

**Sites Notes is three layers, not one** — 0 Points / 1 Lines / 2 Areas, each with its **own** coded-value
domain on `CATEGORIA`. There is no name field, so the geometry name goes into `NOTE`.

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

**Calcite quirks (SDK 4.30).** `ListItemPanel` needs `className: 'esri-icon-*'`; passing `icon:` renders
an invisible button. `bookmark-select` does **not** fire when you click a bookmark in the list — the fix
overrides `viewModel.goTo`. And a synthetic `el.click()` on a `calcite-list-item` selects nothing, so
Calcite UI can only be validated with a real mouse.

**GraphicsLayers cannot label themselves** (`labelingInfo` is a FeatureLayer feature), so parcel labels
are individual `TextSymbol` graphics on a dedicated layer.

**`map.allLayers` includes the basemap and the ground.** Both had to be filtered out of the PDF legend
and the bookmark capture.

## 6. Security & distribution

- The Zornade key is a read-only token **embedded in the source**. This is a deliberate, accepted
  choice: the page is meant for an internal, access-controlled host. It is why this repository is
  **private** and why the page must not go on GitHub Pages or any public URL.
- PV project fields are deliberately limited to non-sensitive ones (`plant_name`, `capacity_mw`,
  `status`, `procedure_type`). Commercial licensing on the elemens data means **internal use only, no
  third parties** — never expose SPV names, VAT numbers, PPA or tariff fields.
- Intended hosting: **Azure Static Web App behind Entra ID** (the org already runs on Entra). Add that
  host's URL as a redirect URI on the OAuth app before sharing.
- For a quick hand-off to a colleague: zip the HTML with the local HTTPS server script — the
  `localhost:8443` redirect is already registered and works from any machine, and they sign in with
  their own org account.

## 7. Audit status — 2026-08-06

Static audit of the committed file. Nothing here blocks use; the tool has been exercised repeatedly in
the browser and behaves correctly.

**Clean**

| Check | Result |
|---|---|
| JS syntax (V8 parser, all 5 inline blocks, 2,400+ lines) | 0 errors |
| Duplicate element IDs | 0 of 202 |
| Unresolved `$('id')` DOM references | 0 of 143 |
| Hardcoded credentials beyond the known Zornade key | none |
| Insecure `http://` endpoints | none (only XML namespace URIs) |
| Third-party library versions | all pinned |

The three bugs found in the 2026-07-29 audit are fixed and present in this commit: viewpoint restore via
`Viewpoint.fromJSON` (assigning raw JSON to `view.viewpoint` does not autocast, so the saved view had
in fact *never* been restored), basemap/ground excluded from the PDF legend, and projected-coordinate
validation on import.

That last one is worth remembering: the first attempt at the fix was **worse than the bug**. UTM
coordinates fall inside the valid Web Mercator range, so (500000, 4600000) was cheerfully "converted"
to (4.49, 38.14) — a plausible-looking point in the sea west of Sardinia, i.e. a silent wrong answer
instead of an obviously broken geometry. The shipped rule converts from Web Mercator **only if the
result actually lands in Italy** (lon 6–19, lat 35–47.5), and otherwise rejects the file with a message
that says what to do.

**Open, known, non-blocking**

1. `z()` wraps a bare `fetch` with **no timeout** — if the Zornade API hangs, the UI stays on
   "Individuo la particella…" indefinitely. An `AbortController` would fix it.
2. Area selection **cannot be cancelled** once started; it can be several hundred sequential requests.
3. The AREAS **Editor is recreated on every web map swap without `destroy()`**, leaving orphaned
   `edits` handlers. Same pattern for the **Legend** widgets created inside LayerList panels.
4. `esc()` escapes `<`, `>` and `&` but **not quotes**, and its output is interpolated into
   `data-title="…"`. A portal item whose title contains a double quote breaks the attribute.
5. `deleteEnabled: true` on the AREAS COLLECTION editor — one click deletes a feature from the portal's
   most important layer, with no confirmation step of our own.
6. The parcel-label toggle is **not persisted** across reloads.
7. If parcels alone exceed the `localStorage` quota, the geometry-less fallback save also fails and does
   so **silently**.
8. Third-party scripts are loaded from three CDNs **without Subresource Integrity**, on a page that
   holds a live portal OAuth token.
9. `querySubstations` uses `outFields: ["*"]` — over-fetching, though not on sensitive fields.
10. The OAuth comment block still cites the **old dev URL** (`http://localhost:8125/ricerca_catastale_gis.html`)
    rather than the current `https://localhost:8443/geoportale_axpo.html`.
11. PDF export with a regional WMS enabled has **not been tested while logged in** — `view.takeScreenshot`
    can throw on a tainted canvas if that WMS lacks CORS headers.

Items 1 and 3 are the natural next pair: small, contained, low risk.

## 8. Repository contents

```
geoportale_axpo.html   the whole application
README.md              this file
```

There is no build, no test suite and no CI. Verification is done in the browser against the live
portal, which is the only place the corporate layers and OAuth actually exist.
