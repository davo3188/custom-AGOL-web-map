# Geoportale · AxpoSolar Italia

A single-file web GIS workbench that sits alongside the corporate **ArcGIS Online** portal
(`sig-urbasolar.maps.arcgis.com`) and adds the one thing AGOL cannot do out of the box:
**search an Italian cadastral parcel by _Comune + Foglio + Particella_ and zoom to it.**

Everything lives in one file — [`geoportale_axpo.html`](geoportale_axpo.html) (~580 KB, ~6,950 lines, SDK 4.34,
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

**Names** (agreed with the user on 2026-09-24; Italian as in the UI, code names in brackets)

| Name | What it is | Code |
|---|---|---|
| **Barra superiore** | Three zones (since 2026-10-01): brand on the left, the **🗺 Mappa** selector in the centre, on the right the project icons 📂 💾, one sign-in button (red/green dot + *Accedi*/*Esci*, user name in the tooltip), theme, help | `#topbar`, `.wmsel`, `#signin` |
| **Pannello sinistro** | Rail + tabs; this is what undocks | `#side` |
| **Rail** | The column of icons that picks the tab | `#sideRail` |
| **Scheda** | Ricerca particelle, Disegna, Geoprocessi, Importa dati (four since 2026-10-01: *Contesto sito* moved to Tools) | `.tabpane[data-tab=ricerca\|disegno\|geoproc\|import]` |
| **Modulo** | A collapsible block of a tab, with title, one line and an **i** (the **i** shows on hover or focus, always on touch) | `details.imod`, id `<tab prefix>Sec<Name>` (`rc`, `drw`, `gp`, `imp`) |
| **Finestra «i»** | The *how it works* window of a module | `RC_INFO`, `DRW_INFO`, `GP_INFO`, `IMP_INFO` |
| **Pannello destro** | Action bar + drawers | `#toolsPanel` |
| **Action bar** | The icons on the right, in groups: Raccolta (with the object-count badge) \| Layer · Mappa di base · Segnalibri \| Modifica AREAS · Stampa | `#actionIcons`, `#rcBadge` |
| **Cassetto** | Raccolta (ex *Elenco & azioni*, renamed 2026-10-01), Layer, Basemap, Segnalibri, Modifica AREAS, Stampa | `.drawer[data-panel=…]` |
| **Sezione** (of the Raccolta) | Particelle, Disegni, Geoprocessi, Importati | `.gsec` |
| **Tools** | The speed-dial menu bottom-left of the map: Misura, Profilo altimetrico, Pendenze da DTM (since 2026-10-04), Analisi di contesto, Coordinate sconosciute, Percorso su strada (since 2026-10-01) | `#toolsFab`, `TOOLS`, `tool*`, `slp*` |
| **Scheda Tools** | The floating card of the open tool (a bottom sheet when the map is narrower than 520 px) | `#toolCard` |
| **Finestra** (modal) | Salva sul portale, Esporta, Promuovi, CAD… | `#…Modal` |

Modules of *Disegna* (reorganised on 2026-10-01, see *Drawing & geoprocessing*): a compact *Sito di lavoro* row
at the top (`#sfRow`, opens only when needed), **①** *Cosa disegni* (category, type, attributes, source, note),
**②** *Con quale strumento* (five tool cards, `.drwcards`, with the exact measurements under them and ↶ ↷ in the
header), the *Ultimo disegno* box, then, pinned at the bottom under *Supporto al disegno* (`#drwDock`), Misure
esatte e aggancio (`drwSecAid`) and Griglia (`drwSecGrid`). *Copia* is now a card of *Geoprocessi*.
Of *Ricerca particelle*: Ricerca particelle (`rcSecFind`), Sulla mappa con selezione manuale particelle
(`rcSecMap`), Seleziona da un disegno esistente (`rcSecDraw`). Of *Geoprocessi*: three steps — **①** Oggetti di
partenza (A), **②** Cosa fare (eight cards: Buffer, Unione, Contorno, Semplificazione, Ritaglio, Parte comune,
Sottrai B da A, Copia), **③** Parametri ed Esegui. Of *Importa dati*: four cards always open — File di geometrie,
Disegni CAD, Immagini sulla mappa, Servizi web (`details.imod.always`); *Coordinate senza sistema* became the
Tools entry *Coordinate sconosciute*. An open module is tinted with the accent colour — border, background and
icon — so it stands out from the closed ones without hovering (since 2026-09-24; light theme: a pale blue veil on
white, dark: a lighter background).

**Undockable left panel** (since 2026-09-24). **⧉** in the header of the pannello sinistro moves it — rail and
tabs — into a window of its own, to put on a second screen; the map takes the freed space and a thin strip keeps
⧉ (bring the window to front) and ⇤ (dock it back). ⇤ in the window, or closing it, docks it back. In that window
the tabs adapt to the width (a container query on `#sideContent`): from 640 px the modules of *Ricerca* and
*Importa dati* go into columns of at least 360 px, and *Geoprocessi* and *Disegna* become a real grid — ① and ③
on the left, ② (the cards, on as many columns of 150 px as fit) on the right (`.paneBody.gpflow` /
`.paneBody.drwflow`, since 2026-10-01; before, CSS columns of about 300 px that the content did not fill). Size
and position are remembered (`axpo_pannello_finestra`); with
more screens, Chrome reopens the window on the other screen only with the *window management* permission, which
the window offers to ask once. The right panel stays docked for now (its ArcGIS widgets are not made to live in
another window).

**Resizable left panel** (since 2026-09-24, like the right one). Drag its right edge (`#sideResizer`) between
300 and 900 px (never leaving less than 360 px to the map); double click goes back to 352 px, ←/→ on the focused
edge move it by 20 px. The width is kept in `--side-w` and remembered in `axpo_side_w`; the edge is hidden when
the panel is collapsed or undocked. The container query also works docked, so a panel wider than 640 px shows
the modules in columns as the undocked window does.

**Parcel search**
- By *Comune / Foglio / Particella*, with an administrative cascade (region → province → municipality). Since
  2026-10-01 the region **no longer swaps the portal web map** as a side effect: under the select a shortcut
  *Vincoli di <regione>: Carica la mappa Check Vincoli ›* appears (`wmSuggest`), and the map is chosen in the
  **🗺 Mappa** selector of the top bar (see *Map & reporting*).
- By clicking the map, or by drawing a point, rectangle, polygon or line over an area.
- From a geometry you already drew — a drawing, a buffer, a geoprocessing result or an import; a point
  finds the parcel under it (since 2026-09-23; before, a point gave a sampling error).
- The tab has the same layout as *Importa dati* and *Geoprocessi* (since 2026-09-23), one module per way of
  searching: *Ricerca particelle* (the cascade, foglio and particelle together; the closed title shows the
  chosen comune), *Sulla mappa con selezione manuale particelle*, *Seleziona da un disegno esistente* (its title
  counts the usable objects). On 2026-09-24 the former *Dove cerchi* and *Foglio e particella* were merged into
  the first one and the other two renamed. The **i** windows draw each mode on a small parcel grid, found
  parcels in the map's amber (`RC_INFO`/`RDIA`).

Because Zornade exposes **no spatial query**, area selection works by sampling a grid of points inside
the geometry and calling `/parcels/locate` on each. It is bounded by explicit caps and is slow on large
areas — parcels smaller than the grid step can be missed. **Esc** interrupts it: the first press stops the
scan and still adds the parcels already found, a second one stops that too. Every Zornade call gives up
after 20 s with a message, instead of leaving the UI waiting forever.

**Drawing & geoprocessing**
- Native `SketchViewModel`: snapping, rectangle, circle, live distance/angle readouts and typed input.
  Verified on 4.34: with the pointer over the map, **Tab** opens the *Deflezione* and *Distanza* fields
  (deflection is relative to the previous side: 90 = right angle to the right, −90 to the left); **Enter**
  locks the values and a **click** anywhere places the vertex there. A second Enter completes the drawing
  *without* that vertex. The circle is drawn from its centre.
- The *Disegna* tab (reorganised on 2026-09-24 as *Disegno*, and again on 2026-10-01 from the user tests —
  the tab is a verb, the drawer on the right a noun, *Raccolta*, so the two stop being read as twin tool bars):
  a compact *Sito di lavoro* row at the top (`#sfRow`, shows the chosen site, opens only to change it), **①
  Cosa disegni** (category, type, attributes, source, note), **② Con quale strumento** (five tool cards,
  `aria-pressed` on the armed one, lit only for the geometries the category allows; the exact measurements box
  under them; ↶ ↷ in the header), the **Ultimo disegno** box (category chip that opens the object's form in
  the Raccolta, name, measure, zoom, ×, *Tutti i disegni nella Raccolta ›*: the answer to "where did it go" is
  given where the drawing was made, `drwLastRender`), and at the bottom *Supporto al disegno* — *Misure esatte e
  aggancio* and *Griglia*, sticky (`position:sticky; bottom:-10px` inside `#sideContent`, the −10 px being its
  bottom padding), always in view while the tab scrolls, closed to one line each with their state in the title
  (*aggancio · misure*, *10 m* / *spenta*, `drwDockSum`). The **i** windows draw the gesture of each tool and a
  typed side (`DRW_INFO`/`DDIA`). *Cancella tutti i disegni* went away: the trash of the *Disegni* section in
  the *Raccolta* does it (`sectionDelete('draw')`). One lexicon everywhere: "nella Raccolta", "Vedi nella
  Raccolta ›", in the tour, the help and the tooltips.
- **Exact rectangles and circles** (since 2026-09-24, under the tool cards): with *Rettangolo* or
  *Cerchio* armed, a box *Con misure esatte* appears (`#shpBox`). Unticked, the SDK draws freehand; ticked with
  valid values, the sketch is cancelled and the map mode `shape` previews the shape under the pointer, a click
  places it (`shpSync` switches between the two as the fields change). The armed tool stays armed across the
  switch: `drwModeEnd(keepDraw)` and `cadModesOff(keepDraw)`; Esc ends the shape mode and the tool together.
- **Copia** (since 2026-09-24; since 2026-10-01 a card of *Geoprocessi*, always enabled because it needs no A;
  its options — *Parallela*, distance, *Solo il lato cliccato* — and the button that arms the click are in step
  ③): plain copy picks any object under the click — drawing (points too), result,
  import, parcel, or a layer feature through `drwEdgeAt` — and moves it with the clicked point (`geomShift`, a
  translation in Web Mercator: 1 km north changes the scale by ~0.02%, negligible); every further click places
  another copy, Esc ends. A copy of our own object keeps category, type, attributes, note, site and style but
  not `sf_gid`/`sf_saved` (a new object on the portal); a copy of a parcel or layer is a new drawing with the
  category of *Disegna*. *Parallela* is the former parallel copy.
- **Connection route on roads** (since 2026-09-24; since 2026-10-01 the Tools entry **Percorso su strada**, a
  generic tool instead of a box inside the *Percorso di connessione* category — the result always gets category
  `connection` and type `estimated`, the only data change of the UX round): click the start, Shift+click via points,
  click the end (`drwMode` `rte`, stops as temporary markers). `rteSolve` calls the organisation's route service
  (`portal.helperServices.route.url`, fallback `route.arcgis.com/…/Route_World`) with `esri/rest/route`, stops in
  the given order (a *simple route*, **0.005 credits** whatever the number of stops — Esri's credit table),
  travel mode *Walking Distance* (shortest in km, no motorways, one-ways ignored) or *Rural Driving Distance* for
  *Solo strade carrabili* (by name, else by type `WALK`/`AUTOMOBILE` with a distance impedance). The service
  snaps to the nearest road: straight connectors from the clicked start and to the clicked end are added, z/m
  dropped. The result is a normal drawing (`Su strada 3,4 km`), the list shows the length of every drawn line.
  Needs sign-in; without it the button refuses (a token-less request would start a sign-in redirect).
- **Site Features model** (since 2026-09-24, replaces the Sites Notes categories). The **link to the site is
  automatic** (since 2026-09-24, `sfAutoSite`): on *Salva sul portale* every object without a site takes the
  AREAS COLLECTION area it touches — one query on the extent of all of them, then locally the area with the
  largest overlap (area for polygons, length for lines, any for points); outside every area it takes the working
  site if there is one, otherwise it stays unlinked and the message says so. `sfRefreshCodes` then re-reads
  `Project_Code` by GlobalID for links without a code (a freshly promoted area gets its code from the CRM later;
  `GlobalID IN ('{…}')` with braces and upper case, checked on the real layer). *Sito di lavoro* is optional
  (a compact row at the top of *Disegna*): it forces a site on new drawings, covers objects outside every area
  and loads a site's objects. *Cosa disegni* starts from the **category** (the role in the project: gross area, net area,
  connection route, exclusion, linear infrastructure, obstacle, access & connection point, mitigation,
  agricultural zone, note & reference), then the **type** (what it is: overhead power line, tree, landscape constraint…) and only the
  attributes that apply (buffer, height, width for lines, voltage for power lines), the source (survey /
  desk / official) and a note. Only the tools of the category's geometries are enabled, and drawings take
  the category colour. In the *Raccolta*, the coloured tag of every drawing, result or import opens a small
  form to change category, type, attributes, note, source and status; ☁ marks what is on the portal
  (orange: changed since). Old drawings with a Sites Notes category get the new one when loaded
  (`SF_FROM_SN`, e.g. *DPA* → exclusion · DPA corridor). The model and its codes are shared with the AGOL
  layer and with PV Predesign: `../schemas/site_features_model.json`.
- A drawing, buffer, result or import clicked on the map (no tool active) can be moved, rotated and
  scaled; a second click edits its vertices; Delete, on the first click, removes it. Since 2026-09-23 the
  end of an edit updates the stored area, the list and the auto-save (see §7).
- **CAD-style tools** (since 2026-09-23), in the *Disegna* tab (the parallel copy in *Geoprocessi › Copia*):
  - **Undo / redo** (buttons, Ctrl+Z / Ctrl+Y). The SDK's own undo only works inside the active sketch
    session (it removes the last vertex, and handles Ctrl+Z itself while the map has focus). For finished
    work the app keeps its own history (`drwHist`, 40 steps): "snapshots" of the tool layer holding references
    to graphics and geometries, recorded in `refreshDl()`. The SDK replaces geometry objects on edit, but at
    the start of an edit every snapshot still pointing at the live geometry gets a clone (`histFreeze`), in case
    it is ever mutated in place. History restarts after start-up restore, *Svuota il lavoro* and project open.
  - **Snapping to DWGs and map layers**: feature sources are rebuilt (`drwSnapSync`) when a drawing or an edit
    starts — the tool layer, parcels and, with the option on, every visible `feature`/`geojson`/`wfs`/`csv`
    layer (DWG sublayers included). Map-image and WMS layers cannot be snapped to.
  - **Angles as azimuth**: `valueOptions.directionMode` `relative` (deflection, default) or `absolute`
    (azimuth from north, clockwise — verified: east = 90°). Remembered in `axpo_drw_dir`.
  - **Shapes with measurements** (the *Con misure esatte* box of *Disegna sulla mappa*): a rectangle (width ×
    height, orientation of the width as azimuth) or a circle (radius), previewed under the pointer and placed
    with a click on the centre. *Da un lato* takes the orientation from the nearest side of a drawing, parcel,
    DWG or layer feature (full geometry re-queried), then goes back to placing.
  - **Parallel copy** (the *Copia* card with *Parallela* ticked): pick an object, then click the side. A line gives a parallel line, an area an inset
    (click inside) or an outset (outside), with mitered corners (`geometryEngine.offset`); *Solo il lato
    cliccato* copies just that side.
  - **Metric grid**: the SDK's `GridControlsViewModel` (a "measured" grid, `view.grid`, spacing in real
    metres even in Web Mercator — verified by snapping: vertices at whole 10 m cells from the grid centre).
    Spacing must be set *after* `trySetDisplayEnabled(true)` or it reverts to 1. *Allinea a un lato* uses
    `interactivePlacementState='interactive'` (two clicks: origin, then direction). `rotation` is in degrees,
    counter-clockwise from east. Not saved in the project.
- **Web Mercator is not conformal on the ellipsoid.** It projects with the sphere, but the coordinates are
  WGS84: at 45° one ground metre is ~0.17% more in x and ~0.16% less in y than the plain 1/cos(lat) — a
  100×50 m rectangle built with 1/cos(lat) measured 100.17×49.92 m. Shapes, single-side offsets and edge
  azimuths therefore use two scales (`mercK2`: kx=√(1−e²sin²φ)/cosφ, ky=(1−e²sin²φ)^1.5/((1−e²)cosφ)) and come
  out exact to the millimetre; whole-object offsets go through `geometryEngine.offset` with one scale, within
  0.15% (1.5 cm on 10 m).
- Client-side geoprocessing via turf on **individual objects**, not whole categories. Two slots, **A** and
  **B**, are filled by clicking objects on the map (parcels, drawings, imports, earlier results — click
  again on the same spot to step down through overlapping objects), from the rows ticked in the list,
  or with a whole category. On A: buffer (one per object, or merged), union, convex hull, simplify, clip
  against a polygon drawn on the spot; between A and B: intersect, difference. Every result lands in
  *Geoprocessi* with its area and can go back into A or B.
- The tab is a **three-step flow** (since 2026-10-01, from the user tests; before, one collapsible section
  per operation with the "fill A first" prerequisite invisible and every message in a grey line at the bottom):
  - **① Oggetti di partenza** — slot A, from the map, the ticked rows of the Raccolta or a whole category; the
    title counts "n oggetti · n aree";
  - **② Cosa fare** — eight cards in a radio group (`GP_OPS`, arrows, Home/End, Space/Enter): Buffer, Unione,
    Contorno, Semplificazione, Ritaglio, Parte comune, Sottrai B da A, Copia. Each card states its own
    condition ("solo aree", "serve B") and is disabled, with the reason and the way out, until A fits
    (`gpFlowSync`); Copia needs no A and is always on;
  - **③ Parametri ed Esegui** — the distance for Buffer, slot B for *Parte comune* and *Sottrai*, the Copia
    options, then **Esegui** and a status line with a coloured dot: ready · in corso · **Eseguito** with the
    measure and *Vedi nella Raccolta ›* · an error in plain words with what to do (`gpErrText`; the raw turf
    message goes to the console). The logic is unchanged (`gpRun`, `makeBuffer`, `gpDoClip`, `gpExec`).

  Operation and result names are in plain Italian (Unione, Contorno, Semplificazione, Ritaglio, Parte comune,
  Differenza); the GIS term stays in the tooltip. Each card's **i** opens an **example**: a small "before →
  after" SVG drawing in the map colours (A orange, B purple, result blue), a real scouting case, then how to
  use it and what to know (`GP_INFO`/`DIA` in the UI glue). *Buffer qui* in the right-click menu preselects the
  Buffer card (`gpOpenOp`).
- **Features of the map's own layers as sources** (since 2026-09-23). In *Dalla mappa*, a click on a
  vector layer adds its feature to the slot: web map feature layers, REST services, WFS, GeoJSON, and
  map-service sublayers (the last through identify). The feature is copied into *Importati* with its
  **full geometry** and its original attributes, which go into exports, so it works with every
  operation, the export and the promotion. The same feature is copied only once. WMS layers are images
  and have no geometry to use.
- **Raccolta** (the right drawer, ex *Elenco & azioni*, since 2026-10-01): its rail icon carries the number of
  objects (`rcBadge`), the right rail is grouped (Raccolta | Layer · Mappa di base · Segnalibri | Modifica AREAS
  · Stampa, `aria-expanded` on the icons), and a dock pinned at its bottom holds **Esporta** (full width) and,
  under it, **Migra particelle in AREAS Collection** | **Salva disegni in Site Features** (see *Export* and
  *Write-back*).
- **Colour picker** (since 2026-10-01, `makeColorSwatch`/`colPopOpen`): a button that opens swatches — *Recenti*
  (the last 8, `axpo_colori_recenti`), the model's *Categorie*, *Mappa* (the Okabe–Ito palette, safe for colour
  blindness) and *Tenui* — plus *Altro colore…* for the native picker; Esc closes, `aria-pressed` on the chosen one.
- **GPS locate button** (since 2026-10-06, user request): the SDK `Locate` widget under the compass
  (`view.ui`, top-left, index 2), `popupEnabled: false`, zooms to 1:2500; a refused or failed geolocation gives
  a toast (`locate-error`). Needs https, like the login — fine on Azure and on the local HTTPS server.
- **Legenda** (since 2026-10-06, user request: "the legend of the layers that are on, in the rail between the
  layer list and the basemaps"). A rail panel `data-panel="legend"` (`esri-icon-legend`) with two parts. Above,
  the SDK **Legend widget** (`esri/widgets/Legend`, mounted on the first opening by `legendMount`), with
  `respectLayerVisibility` (only layers that are on) and `hideLayersNotInCurrentView: false` — a layer that is on
  but out of scale stays in the legend (the first version hid it and the user read it as a defect: "some layers
  are missing"); a status line says how many are out of scale. It covers portal and web-map layers, services
  added by URL, GeoJSON, ArcGIS tile services, and follows a web-map swap because it reads `view.map`. Its own
  "Nessuna legenda" message is hidden (CSS): next to WMS images it was misleading; `#legendMsg` says instead
  "Nessun layer acceso" / "Nessun layer acceso ha una legenda da mostrare". It does not cover WMS — for those
  `legendWmsRender` adds the GetLegendGraphic image of each visible sublayer that declares a `legendUrl` (no
  constructed URLs, so no broken images; the cadastre WMS is `listMode: 'hide'` and stays out), honouring the
  groups that contain the layer (`legendAncestorsOn`) and its scale range (`legendScaleOk`), as the widget does
  for its layers (the first version ignored both: WMS images looked "always on") — and layers that are on but
  have nothing to show (vector tiles, WMTS, images, graphics, or "hide in legend" set in the web map,
  `legendEnabled: false`) are named in a line, *Accesi ma senza legenda* (`legendMissingRender`, comparing the
  operational layers that are on with the widget's `activeLayerInfos`). It does not cover
  the Raccolta's graphics layers either: below, the **Raccolta block** (`legendLocalRender`) lists what is on with
  its real colour and a count: Particelle, each Site Features category present among the drawings (polygon,
  line or point swatch), drawings without category, geoprocess results, imports, owner groups of the owners
  file, images. The block redraws 150 ms after every list redraw (`legendSoon` from `renderGeoms`) and the whole
  panel after a layer visibility change (a `reactiveUtils.watch` on `view.map.allLayers` visibilities), only
  while the panel is open.
- **Crea area lorda / Crea area netta** (since 2026-10-06, user request: the two steps that always follow the
  choice of parcels, without going through Geoprocessi). Under the *Particelle* list, **Crea area lorda dalle
  particelle** unions the ticked parcels — or all of them when none is ticked; the label says which — into a
  drawing of category *gross* named `Area lorda · <comune> · n particelle`, with the area in the note; disabled
  without parcels. One gross drawing: when one exists, a confirm asks to replace it — the geometry changes in
  place, so a drawing already on the portal keeps its GlobalID and the next save updates it — or to add another
  (`sfReplaceOrAdd` → `drwAddGraphic`, `sf_src: 'geoprocess'`).
- **Automatic strips and automatic net area** (same evening, user: "many categories have a buffer but choosing
  it creates no drawing; could the net area be automatic, with no selection?"). The rules are those of PV
  Predesign (`areas.js`, `cutDistance`): the categories that **cut** are *exclusion*, *linear*, *obstacle*,
  *mitigation* and *agri* (`SF_CUTS`); the distance is the buffer, or half the width for mitigation/agri lines
  (`sfCutDist`); polygons of those categories cut as they are, buffered when they have a buffer; lines and points
  cut only through a distance.
  - **Fasce.** Every object with a cut distance gets a companion polygon drawing that follows it: geometry, name,
    buffer/width, type. Exclusions, infrastructures and obstacles give an *Esclusione · Fascia di rispetto*
    (*Fascia DPA* for power lines) named `Fascia 20 m · <nome>`; a hedge or an agricultural strip gives a polygon
    of its own category, `Fascia larga 6 m · <nome>`. The link is `sf_of` → `sf_uid` (a persistent id given to
    the source when first needed); `sf_sig` (extent, vertex count, distance, name, type) says when to recompute.
    The strip disappears with its object or when the distance goes to 0; its × in the list refuses with a message
    (remove the buffer instead). It is an ordinary drawing otherwise: portal, export and projects carry it.
  - **Area netta automatica.** With at least one gross drawing and the switch under *Disegni* on (default), one
    drawing `Area netta · x ha` exists and is recomputed on every change: gross union minus the union of all
    cutting polygons (strips included, hidden ones included — the eye is display only). Its note says what was
    subtracted and how many lines/points without a distance were ignored. The switch off freezes it (it stays as
    it is, e.g. to save a precise version on the portal); on, it follows again. Deleting its row turns the
    switch off, so it does not come back; no gross drawing → no net drawing; cuts covering everything → the
    drawing is removed with a message. The flag travels with the work (`netAuto` in auto-save and project).
  - **Rientro dal confine** (same night, user request): a metres field next to the switch. The gross union is
    shrunk by that distance (`geodesicBuffer` with a negative distance) before the cuts — the setback from
    cadastral boundaries, PV Predesign's `boundarySetback`. The note says "− rientro dal confine 10 m"; a
    setback that eats the whole gross area removes the net drawing with a message. Saved with the work as
    `netSetback` (auto-save and project).
  - `sfAutoSync` runs 200 ms after every `renderGeoms` (`sfAutoSoon`), works in WGS 84 (`to4326`: sketch
    geometries are Web Mercator, parcels and imports degrees) with the Esri `geometryEngine` (`geodesicBuffer`,
    `union`, `difference`), and calls `refreshDl` only when it changed something; the signatures stop the loop.
- Per-geometry **colour and visibility**, plus per-section controls, all persisted. Since 2026-09-24 every
  section header of the *Raccolta* — *Importati* included, which had none — has: a tick box for all its rows
  (with the *some* state; `sectionSelCount`/`setSectionSel`), show/hide all, a **paint bucket** for the colour
  (the filled square looked like the selection tick box; the rows use the bucket too, `swatchPaint`) and a
  **trash** for the whole section (`sectionDelete`: asks first; for objects saved on the portal asks again
  whether to delete them there too, one `applyEdits` per layer, `sfDeleteRemoteMany`; ↶ Annulla restores what
  was removed from the list). *Disegni* has no section colour: there the colour is the category. *Svuota
  particelle* went away (the trash of *Particelle* does it). In *Importati* show/hide and the trash also act on
  the images, which now have an eye too.

**Site context** — since 2026-10-01 the Tools entry **Analisi di contesto** (a floating card; the rail tab is
gone). Nearby substations (380/220/150/132 kV), HV lines and PV projects, pulled live from the portal layers.
Needs sign-in; signed out, the card says why and keeps the button disabled. *Analisi di contesto qui* in the
right-click menu opens it on the clicked point.

**Import** (the *Importa dati* tab)
- Since 2026-09-23 the tab is one module per way of importing: files, CAD drawings, images, web services. Since
  2026-10-01 the four are **cards always open** (`details.imod.always`: no arrow, the saved state cannot close
  them; side by side in a wide panel) and *Coordinate senza sistema* moved to Tools as **Coordinate
  sconosciute** (a file without `.prj` opens that tool by itself). The complex modules have an **i** button that
  opens a "how it works" window (`IMP_INFO` in the UI glue): steps, then things to know. A title shows a
  counter ("2 in mappa") for DWGs, images and services. The code opens the right module when it sends you
  there: a DWG picked through the generic file button (`window.impOpen`). The tour steps do the same.
- Vectors: GeoJSON, KML, KMZ, SHP.
- **Owners file** (since 2026-10-05). The GeoJSON written by the separate *Estrazione intestatari* tool — the
  parcels exported from here, with the cadastral owners that tool reads from SISTER — is recognised by the
  `sister_stato` property on its features; every other import behaves as before. Each parcel is named by sheet and
  parcel (`Fg 12 · 101 · Comune`), coloured **by owner** (same set of owners = same colour, grey without owners)
  and keeps all the file's properties in `src_attrs`, so an export gives them back. In *Raccolta › Importati* the
  parcels come after the plain imports under a bar *Per intestatario* (count, and a switch *Nomi sulla mappa*) in
  **one collapsible group per owner**, largest total area first, *Senza intestatari* last. A group is a set of
  co-owners: ten parcels held by the same two people are one group and one colour, and the bar counts groups
  ("1 gruppo · 10 part."). The parcels **export** reimported by mistake — same file name as the owners file,
  without the `Intestatari_` prefix — has no owners: it is imported as before, and a message says which file to
  import instead (`impAdd.hint`, for any plain import whose features carry `foglio`, `particella` and
  `belfiore` or `ncr`). A group header has a
  tick box, eye and colour for the whole group, the owners' names and "n part. · ha"; each row has an **i**
  that opens the detail (one line per owner with tax code and share, then status, land use, cadastral area,
  notes). On the map every parcel carries the short name (`etichetta`, e.g. `ROSSI MARIO +1`) and **a click
  opens a popup** with the same detail instead of starting the shape edit. Files of the tool's version 1.1
  have no `nominativi_brevi`/`etichetta`: names are then cut from `nominativi` with the tool's own rule (what
  SISTER appends — "nato a … il …", "con sede in …" — is dropped).
  **Personal data.** Names and tax codes stay in the page and in the `.axpo` project (the save message says
  so); they are **left out of the browser auto-save** — after a reload the file must be imported again or the
  project reopened — and **never go to the portal** (*Salva sul portale* skips these parcels and says how many).
  Nothing of them is written to the console.
- Georeferenced **images and GeoTIFF** (rewritten on 2026-10-06, "immagini v2"). An image sits on the map
  by its four corners (`ControlPointsGeoreference`, projective); that is all the project stores, whatever put it
  there. Four ways to put it there, all in the browser:
  - **Metadata.** GeoTIFF: geokeys, `ModelTransformation` (rotated rasters) or a `.prj` picked with the file
    when the geokeys give no EPSG. **World file** (`.jgw`, `.pgw`, `.tfw`, `.wld`…) with its `.prj`: select
    the image, the world file and the `.prj` together (same base name; the file input accepts several
    files). Without a `.prj` the system is **estimated from the coordinates** with `srEstimate`, like a DWG;
    degrees are taken as WGS 84; the message says which system was used and warns when it is uncertain.
    Corner edges are at ±0.5 pixel of the original size, as the world file convention wants.
  - **Sposta / scala / ruota.** A dashed frame polygon follows the corners; the SketchViewModel `transform`
    tool gives drag, scale handles (*Proporzioni* locks the aspect ratio) and the rotation handle, with
    snapping to parcels and drawings. Arrow keys nudge by one screen pixel (Shift: ten). *Ripristina* returns
    to the placement at the start of the edit.
  - **Angoli.** The `reshape` tool on the same frame, vertices only: perspective, as the old handles did.
  - **Punti.** Pairs "point on the image → where it really is" (the target snaps to vertices of parcels,
    drawings and DWGs; Esc drops a pending point). 2 pairs: similarity — translate, scale, rotate, no
    distortion (the DWG alignment, for images); 3: affine; 4 or more: affine by least squares, or projective
    on request (*Modello*). Each pair shows its residual in metres (Web Mercator corrected by cos φ) and the
    text the RMS; pairs can be removed one by one and are kept with the image and in the `.axpo` (`cps`),
    so a placement can be refined after reopening. Source points are stored as image pixels, found through
    the inverse of the current placement.
  - Everything degenerate is refused silently: a folded frame is not applied, collinear points give a
    message, a transform that folds the image is not applied.
  - **The card** (same afternoon, user feedback "fixed and bulky in the middle, does not match the UI"): the
    toolbar became a floating card styled like the Tools card (`#imgToolbar`, header with name, **i** and ×),
    docked top-left next to the zoom control by default, **draggable by its header** and remembered in
    `localStorage` (`axpo_img_card`, clamped into the map on every open).
  - **Valori** (E, collapsible): rotation in degrees (anticlockwise, 0 = upright), metres per pixel of the
    original image — or "1 : N at dpi", which computes it; a PDF prefills its rendering dpi — and the centre in
    WGS 84 degrees. *Applica i valori* rebuilds the four corners as a rectangle (`imgGeoApply`): perspective
    is dropped, control points are cleared. The fields follow every placement change (`imgValsRefresh`).
  - **PDF** (F): rendered in the browser with pdf.js (ESM build loaded with `import()`, worker in a blob
    module so the cross-origin script can run off the main thread), the chosen page at 4096 px on the long
    side; the project keeps the PDF and the page number and renders it again on open (`page` in the record).
  - **World file export** (H, *World file* button): a zip with the image, its world file and a `.prj` in Web
    Mercator. Affine placement (frame still a parallelogram): the original file untouched plus the world file
    (full resolution); a PDF or an image without its file: the rendered PNG plus the world file; projective
    placement: the image is **resampled north-up** (`imgWarp`, nearest neighbour, long side ≤ 4096 px) as
    `<name>_nord.png` with its world file. World-file values are for the **original** pixel grid
    (`Ho = H · scale(pxW/ow, pxH/oh)`), C and F at the centre of the top-left pixel.
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
- **Coordinates whose CRS is unknown** (Tools › *Coordinate sconosciute*) — a decree, a survey, an email, a
  shapefile with no `.prj`.
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
- **Zoom after a selection** (since 2026-10-09, user request: "the zoom goes to all the extracted parcels,
  I want it only on the parcels of that last selection, or no zoom-out"). One rule: after a search, a click
  or an area selection the map frames **only the parcels of that action** (`render(true, fids)`, including
  the ones that were already in the Raccolta), and **only if they are not all inside the current view**
  (`parcelsInView`). So a click or an area selection never moves the map (the parcels are under the cursor or
  inside the shape you drew), a search by Foglio/Particella flies to what it found, and nothing ever zooms
  out to the whole Raccolta any more. The per-row Zoom button is unchanged; the saved viewpoint at start-up
  still wins (`bootVP`).
- **Cadastre source and bridge** (since 2026-10-08). The "Catasto AdE" layer on every map is the Agenzia delle
  Entrate WMS, but that service cannot be used from a browser (no CORS, no Web Mercator), so it needs a bridge.
  `CATASTO_SOURCES` lists them in order of preference: the app's own bridge at `api/catasto` next to the page
  (`scripts/serve_https_catasto.py` locally; for the site there is an Azure Function ready in
  `scripts/ponte_catasto_azure/`, not installed by the owner's choice of 2026-10-09), then the public GeoServer of
  Regione Sardegna, which cascades the AdE WMS. `catastoProbe` sends each one a 32-px GetMap 1.5 s after start
  and every 10 minutes: the first answering an image is used (the layer is rebuilt in place by `catastoSwap`,
  keeping visibility and opacity; the project key `app:catasto` does not change); if none does, a red toast
  repeats the service's own exception text and the layer is titled "Catasto AdE ⚠ non disponibile" in the Layer
  panel, with a green toast when it comes back. `?catasto=<url>` puts a bridge of your choice first (kept in
  `localStorage`; `?catasto=off` removes it).
- **Web services by URL** (since 2026-09-23): WMS, WFS and ArcGIS REST (MapServer, FeatureServer, a
  single layer, ImageServer…) go straight onto the map, without creating an item on the portal. The type
  is guessed from the URL, the service is read, and you tick the layers you want. The list has a text
  filter for services with hundreds of layers. The added layers are kept by the browser auto-save and by
  projects, and are removed from the Layer panel.
- The **top search box** takes addresses and coordinates: `lat, lon` (decimal comma and DMS included)
  or projected `X Y`, whose system it guesses and flies to. A coordinate pair wins over the geocoder;
  anything with letters in it (an address with house number and postcode) goes to the geocoder. Since
  2026-10-01 it is collapsed to a lens button on the map (`#searchToggle`, `window.searchCollapse`); open, it is
  80% opaque and closes with Esc, a click outside or a result.

**Export**
The **Esporta** button of the Raccolta dock opens one window (since 2026-10-01, `openExport`/`exRender`): the
format at the top (GeoJSON, KML, KMZ, SHP, XLSX), then one section per kind with *tutti* and one row per object;
the ticks are the Raccolta's own, both ways; with nothing ticked everything is exported; XLSX disables the
non-parcel sections (`buildFC`/`graphicsFeatures` take the chosen set). *Dissolve* moved to the AREAS migration
window. Names you assign in the UI end up in the file. The shapefile comes as one zip with a layer per geometry
type (`_punti`, `_linee`, `_aree`).
**File name** (since 2026-10-05, `exBaseName`): municipality, date and number of parcels —
`Milano_2026-10-05_6part.geojson`. With several municipalities, the one with most parcels and the count of the
others (`Milano_e_altri_2_…`); with no parcel in the export, `export_2026-10-05`. A parcel is any exported
feature with `foglio` and `particella`: the rows of *Particelle* and the imported owners-file parcels; drawings
and results do not count. Letters, digits, `_` and `-` only (accents and apostrophes go), and inside the
shapefile zip the hyphens of the date become `_`. A message names the file once it is downloaded. Until then
every export was `export.<ext>`: the owners tool names its results after its input, so two different exports
ended up in the same result file.

**Write-back to the portal**
- Parcels → **AREAS COLLECTION** (layer 426) with **Migra particelle in AREAS Collection** in the Raccolta dock
  (the *dissolvi* option is in its window). REGIONE and PROVINCIA are the parcel's own (Zornade names,
  the same spelling already in the layer), COMUNE and PRO_COM_T come from ISTAT, the area is geodesic —
  the same number the list shows.
- Drawings, buffers, geoprocessing output and imports → **IT - Site Features** with **Salva disegni in Site
  Features** in the dock (since 2026-09-24 as *⤴ Salva sul portale*, replacing *Promuovi → Sites Notes*; renamed
  2026-10-01 — "salva", not "migra", because the second time it updates). Category, type, attributes, name, note, source,
  status and the site (`AREA_GUID` = GlobalID of the AREAS COLLECTION area, `PROJECT_CODE`) are written. Each
  object gets a GlobalID made in the browser (`sf_gid`) and is saved with `applyEdits(…, {globalIdUsed:true})`:
  the first save adds it, the next ones update it — no duplicates; an update that fails (object deleted on the
  portal meanwhile) is retried as an add. Objects without a site take the site of work. *Solo gli spuntati*
  saves only the ticked rows; objects without a category get the one chosen in the dialog (default: note).
  *⤵ Carica i suoi elementi* brings the site's features back into the list (skipping those already there).
  Removing a saved object with × asks whether to delete it on the portal too.

**Map & reporting**
- Loads the 17 regional *Check Vincoli* web maps (16 of 20 regions covered) plus the General Map, chosen in the
  **🗺 Mappa** selector at the centre of the top bar (since 2026-10-01, `wm*` functions): a menu with the General
  Map and the 16 Check Vincoli, a search by name, the current one ticked, entries disabled when signed out,
  arrows, Esc, closes on a click outside; the name is repeated at the top of the Layer drawer. On sign-in the
  General Map always loads. A confirmation is asked only when something would be lost — layers switched on in
  a Check Vincoli map or attribute filters (`wmHasStateToLose`); parcels, drawings, results, DWGs, images and
  services survive the swap (`loadWebMap` moves them). The menu gets `z-index:1500` only while open: the centred
  container has a CSS transform, hence a stacking context, and without it the menu was painted under the map.
- LayerList with per-layer legend, transparency, popup toggle and drag-and-drop reordering, a search box
  by layer name, and an **attribute filter**, like QGIS *Filter…*: field, condition and value (values
  suggested from the data, domain values as a menu), `+ E` / `+ O` to combine, editable SQL, live count.
  The panel icon turns into a funnel while a filter is on. Layers you added (portal items, web services,
  DWGs, images) get a **…** menu with *Zoom al layer* and *Rimuovi dalla mappa*.
- Bookmarks that also restore **layer visibility** (Esri's own bookmarks do not).
- Right-click context menu acting on the clicked point: add parcel here, *Analisi di contesto qui*, *Buffer qui*
  (preselects the Buffer card), copy coordinates, open in Google Maps, centre here.
- Parcel labels on the map, hidden below 1:150,000; the on/off choice is remembered.
- **Tools** (since 2026-10-01): one speed-dial button bottom-left (`#toolsFab`, `aria-haspopup="menu"`,
  `aria-expanded`), a vertical menu with icon, name and one line per entry (44 px rows, `role=menu`/`menuitem`,
  arrows, Home/End, Esc with focus back, closes on a click outside or when a drawer opens): **Misura**, **Profilo
  altimetrico**, **Analisi di contesto**, **Coordinate sconosciute**, **Percorso su strada**. One tool at a time,
  in a floating card (`#toolCard`, 340 px, 560 for the profile; a bottom sheet when the map is narrower than 520
  px; the button shows the open tool, `aria-current` in the menu). A tool that acts on the map joins the single
  mode switch, the banner and the Esc chain (`toolModeOff`, `measureStop`, `elevStop`); opening one while a sketch
  is in progress **completes** the sketch if it has enough vertices (2 for a line, 3 for an area) instead of
  dropping it (`sketchFinishIfPossible`, also from `modeOffDraw`), and a toast says so. Signed out, context and
  route show the reason with the button disabled.
- **Pendenze da DTM** (Tools, since 2026-10-04, `slp*`): where the gross area is too steep. Pick the **gross
  area** (a drawing of category *Area lorda*, the site of work, or the parcels of the list merged), choose the
  **slope limit** — *in ogni direzione* (one value: 10 = everything steeper than 10 % is excluded, whatever its
  aspect) or *nord–sud ed est–ovest, separati* (two values: a cell is out when either component exceeds its
  own limit) — and press *Estrai*: the terrain is
  sampled over the gross area **plus 25 m** and shown on the map as a georeferenced image (hill-shaded
  elevations, or the slope classes 0–5–10–15–25 %), transparent outside the buffer, with the cells over the
  limit hatched in red. Changing the limit (or the minimum patch) redraws image and figures at once,
  with no new download. The limits are the user's own, nothing else: the comparison with the company's design
  standards, structure by structure, is PV Predesign's job when it opens the project (the user's decision of
  2026-10-04: those values stay out of this tool, whose repository is public). *Aggiungi l'esclusione alla Raccolta* makes **one drawing** *Esclusione · Pendenza
  elevata* (`exclusion` / `steep-slope`), clipped to the gross area, named *Pendenza > 10 %* (or *> 10 %
  N-S/E-O*); pressing it again after a change updates the same drawing (and marks it to be updated on the
  portal) — also after a page reload, when the grid is gone but the exclusion is not: a new extraction of the
  same area takes the identifier back from the drawing. No sign-in, no credits:
  a 10,560-cell grid took 1.9 s. The method is **PV Predesign's own** (its terrain module, ported function by
  function so both tools give the same numbers on the same grid): a grid of
  5 m cells (2–50, grown by 1.25 until under 400,000 cells) in a local Transverse Mercator frame centred on the
  site with scale 1 (true ground metres; the same definition as Predesign's local frame); elevations of **Esri
  World Elevation** at the cell centres (`ElevationLayer.queryElevation`, `finest-contiguous`, 5,000 points per
  request; about 10 m of native resolution in Italy: fine for scouting, not a survey); slope in percent with
  Horn's 3 × 3 weights; the limit is Predesign's own object — `{maxAny}` for the steepest slope in every
  direction, `{maxNS, maxEW}` for the two components — and `slpMask` is its limit mask; patches under the
  minimum area (250 m², as in Predesign) dropped; cells traced into rings. The image sits among the *Importati*
  of the Raccolta (eye, opacity, ×; tagged *DTM*, no corner handles) and in the layer list.
  **What PV Predesign receives** through the `.axpo` project: the exclusion drawing with its codes and
  `sf_slope` = `{limit: {maxAny} | {maxNS, maxEW}, cell, min_patch, buffer, source:'esri', dtm, grid, area,
  date}`; and `terrain`, a
  list with one record per DTM still on the map — `{id, lon0, lat0, x0, y0, cell, nx, ny, zmin, zmax, missing,
  loadedAt, source, name}` exactly as Predesign describes its own grid, plus `buffer`, `area_name`, `area_geom`
  (GeoJSON 4326), `limit`, `min_patch`, `view` and `file` — with the elevations in `terreno/<id>.f32`
  (Float32, little-endian, row by row from the south-west cell, NaN where unknown: the layout of Predesign's
  own grid file). On the portal only category, type and the note go to *IT - Site Features*: the limit is
  written in the note, and the user decided on 2026-10-04 **not** to add a field for it. The DTM is kept in projects, not in the browser
  auto-save (after a reload the exclusion stays, the image does not).
- Scale bar bottom-centre at 70% opacity (`#scaleDock`), coordinates widget glued to the bottom-right corner,
  Esri attribution at 18 px and 0.75 opacity (full on hover), mode banner small at the top edge of the map
  (11 px, ellipsis, full text in its tooltip) — the user's rule of 2026-10-01: lighten the map visually.
- Contrast (2026-10-01, measured on the tokens): filled buttons and active states use `--accent-strong` with
  white in the light theme (7.2:1) and `--accent` with dark text in the dark theme (8.9:1) — white on
  `--accent` gave 3.2:1 / 2.1:1; accent-coloured texts (links, counters, section titles, banner) use `--link`;
  ok/error outcomes are a coloured dot plus normal text, since green and yellow as text on white fail AA.
- Printing through the Esri Print widget. The custom A3 PDF site report (jsPDF) was **removed on
  2026-09-22** — it never worked reliably and will be redesigned; the old code is in
  `geoportale_axpo_pre-bugfix-export-pdf-geoproc_2026-09-22.html`, inside `backups/archive_2026-09-22.zip`.
- Auto-resume from `localStorage`, guided spotlight tour, per-icon tooltips.
- **Browser storage** panel (in *Impostazioni avanzate*): shows what is saved, *Svuota il lavoro*
  (parcels, drawings, results, imports, DWGs, added layers — preferences kept) and *Ripristina tutto*
  (every key of the app, then reload). DWGs and images are never saved there; `localStorage` is capped
  at ~5 MB per site, and hitting the cap now shows a warning instead of failing silently.

**Projects** (top bar, since 2026-09-23; since 2026-10-01 the two icons 📂 💾 on the right, names in the tooltips)
- **💾 Salva progetto** writes the whole working state to one `.axpo` file, like a QGIS `.qgz`:
  view (with rotation), basemap, work area (region/province/comune), the portal web map and every
  layer's visibility, transparency, popup toggle, attribute filter and order, the portal layers added by
  search, the web services added by URL, parcels,
  drawings, geoprocessing output, imports, label toggle — and the **original DWG and image files** with
  their placement (reference system, scale, two-point alignment, layer visibility and colours; image
  corners and transparency), plus, since 2026-10-04, the **elevation grids of *Pendenze da DTM*** with their
  slope limit (the DTM image is rebuilt from the grid on opening). PV Predesign opens the same file.
- **📂 Apri progetto** replaces the current work (after a confirmation) and rebuilds all of it; the DWGs
  are re-read from the file without the confirmation dialog. `Ctrl+S` saves again to the same file,
  `Ctrl+Shift+S` or Shift+click is *save as*, `Ctrl+O` opens. The project name shows in the top bar and
  in the tab title.
- The portal part needs a session: opened while signed out, the project restores everything local,
  says so, and keeps the web map pending — it is applied if the sign-in completes without a reload.
- Bookmarks, theme and the Zornade key are personal preferences and stay out of the file; no token is
  ever written.

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

**The undocked panel is moved, not copied.** `sideUndock()` opens an empty window (`window.open`, from the click),
copies the page's stylesheets into it and moves the `#side` node there with `adoptNode`: handlers, state and the
script stay those of the page, so there is nothing to synchronise. What that requires:
- every id lookup goes through `byId` (`$`, `$g`), which looks in the page and then in the panel window; the few
  `querySelector` calls made while working start from an element found that way;
- global listeners (clicks that switch modes, Esc, Ctrl+Z/Y, Ctrl+S/O) are registered with `docOn`, which also
  attaches them to the panel window;
- `confirm`/`alert`/`prompt`, the **i** windows, the CAD dialog, the messages and the project file pickers open in
  the window where the user is working (`sideActiveDoc`, `sideModalHere`); dialogs moved there come back on docking;
- the panel window's `pagehide` (closing it, or reloading it by hand) docks the panel back before the document
  goes; the page's `pagehide` closes the panel window; the guided tour docks it first (it points at page elements).
Tested on 2026-09-24 in the built-in browser with an iframe standing in for the window (the built-in browser
blocks popups): tab switching, drawing from the panel onto the map, **i**, Esc, theme, columns, both ways of
docking. Not yet tested as a real window on two screens.

**IT - Site Features replaced Sites Notes on 2026-09-24.** Sites Notes (`719dd7038c5547eb93896e2c2a11c0bf`) had
only `CATEGORIA` and `NOTE` (256 characters, name and note glued together), no editor tracking and no link to
the site, and its categories mixed what an object is with what it means for the design. The new service
(`aec2f4fb317f4aa4918b42d947ab8ad3`, private until shared) has three layers with the same fields — CATEGORY,
TYPE, NAME, NOTE, BUFFER_M, HEIGHT_M, WIDTH_M, VOLTAGE_KV, PROJECT_CODE, AREA_GUID, SOURCE, STATUS, LEGACY —
coded domains with English codes and Italian labels, editor tracking and attachments. It was created by
`../scripts/archive/crea_site_features.py` from the model, and the 1 840 Sites Notes features (with their 30
attachments) were copied by `../scripts/archive/migra_site_notes.py`, which also assigned the AREAS COLLECTION area by
intersection (or the nearest within 100 m: 1 369 of 1 840). Sites Notes itself was not changed; a snapshot is in
`../backups/`. In the app the model lives in `SF_CATS` (codes, labels, colours, geometries, attributes, types).
GlobalIDs are compared normalised (`sfGuid`: upper case, braces); the service returns them lower case without
braces and accepts every form in a `where`.
On 2026-09-24 the model went to 1.1.0 (user's request): new category **`connection` «Percorso di connessione»**
(lines only, types `estimated`/`confirmed`, violet), placed after the net area; `access` is now labelled
«Accesso e punto di connessione» (the point: primary substation, HV station; the cable route has its own
category). On the layer: two coded values on the lines' CATEGORY and TYPE domains, the access label on points
and lines, a violet renderer class and an editing template — additions only, done by
`../scripts/archive/aggiungi_percorso_connessione.py` after checking the owner; definitions before the change in
`../backups/site_features_layer{0,1,2}_def_pre-connection_2026-09-24.json`. Feature counts unchanged (114 / 556 /
1 170). The same change is in `../schemas/site_features_model.json` and in PV Predesign (`categories.js`, where
route words — *percorso di connessione*, *tracé de raccordement*, *Kabeltrasse*… — are tested before the power
line words; a plain *cavidotto* stays an existing underground line).

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

**One cadastre only: the app's WMS** (since 2026-10-01, the user's choice). Several web maps carry their own
national cadastre. The loader finds it **by item ID** (`382d9c13…`, `05ccc51f…`; title fallback only with
«AdE» and «catast»: there are ~20 regional and provincial cadastres with similar names, which stay) and
**removes it from the map loaded in the browser** — the web map item on the portal is untouched — then puts
the app's WMS on top, under parcels and tools. Until then the web map's layer was used instead of the app's.
The app's WMS starts at 50% transparency with the popup off (the GeoServer does not answer GetFeatureInfo from
the browser); a project keeps its own saved values. The ids removed go into `wmCatGone`, so an older project
that mentions that layer does not report it as missing.

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
  DXF's `& 1`). The DWG version is read from the first six bytes of the file (`AC1032` = 2018+), because
  the converted header leaves `ACADVER` empty. **Flags come out as numbers, not booleans** (`isCCW: 0`,
  not `false`): a hatch arc edge with `isCCW === false` never matched, so clockwise arcs were drawn on the
  wrong side of their chord (road fillets and turning areas became half-discs; fixed 2026-10-02, §7). For a
  clockwise edge the stored angles are mirrored: the real arc is (−end, −start) counter-clockwise, then
  reversed, as ezdxf does.
- **OCS (tilted planes).** Entities with an extrusion normal (LWPOLYLINE, POLYLINE2D, ARC, CIRCLE, SOLID,
  HATCH, TEXT, INSERT) are mapped to the world with AutoCAD's *arbitrary axis algorithm*, **including the OCS
  z** (elevation, centre z, insertion z). This is not academic: PVcase places every tracker block in a plane
  tilted by fractions of a degree to follow the terrain, with an insertion z of thousands of metres that
  shifts x and y by metres once projected; and above 1/64 of tilt the algorithm turns the OCS X axis by 90°
  (Z×N instead of Y×N), which PVcase compensates with a 270° block rotation. Until 2026-10-01 the reader
  only mirrored X for a −Z normal and dropped z: modules shifted by metres, rotated ones thrown to (y, −x).
  The whole entity pipeline therefore runs on 3D affine matrices (`c3Mul/c3T/c3R/c3S/c3Ap/c3Ocs`,
  column-major 3×4, `c3Ap` returns the plan x,y); the 2D ones (`cadMul`, `cadT`…) remain only for the
  alignment matrix `ch.M`, which is saved in projects. Ground truth for any DWG: arcpy reads it natively
  with INSERTs already exploded (`scripts/dwg_confronto_arcpy.py` in the AGOL Axpo repository; read only
  `\Polyline` and `\Point`, a cursor on `\Polygon` or `\Annotation` crashes the process on PVcase files).
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

**Project files (`.axpo`).**
- **Format.** A zip: `progetto.json` plus `file/dwg-<n>-<name>` and `file/img-<n>-<name>`, the
  originals as they were imported. The JSON is the auto-resume state (`captureWork`: parcels, `toolGeoms`,
  basemap, viewpoint, added items) plus `webmap`, `area`, `labels`, `layers`, `order`, `images`
  (corners in the view SR) and `dwg` (`wkid`, `sc`, the affine matrix `M`, `drop`, `manual`, per-layer
  visibility and user colour). `formato: "geoportale-axpo-progetto"`, `versione: 1`: bump the version
  when the meaning of a field changes, and keep reading the old ones.
- **Keeping the originals.** The source `File` is kept on each DWG (`f.file`) and image (`im.file`) at
  import. A browser `File` points at the disk, so this costs no memory until saving.
- **Layer keys.** Web map layers are matched by their web map id, which is stable. The layers the app
  creates get a new random id on every page load, so they are keyed by role: `app:catasto`, `dwg:<n>`,
  `img:<n>`, `item:<portal item id>`. On opening, the DWG/image keys map to the layers actually rebuilt
  (`keyMap`): if one file fails, the others keep their own state.
- **Order.** Order is restored by permuting the found layers among the slots they already occupy, per
  container (the map and each group). Internal layers (`listMode: 'hide'`) never move.
- **Sublayers.** Sublayer visibility (WMS, map services) is applied on `layer.when()` without forcing a
  load: the Check Vincoli maps must read their regional WMS only when switched on.
- **Save and open dialogs.** Saving uses the File System Access API (`showSaveFilePicker`,
  `showOpenFilePicker`) in Chrome/Edge, with a download as fallback. The picker and the write permission
  are requested *before* the zip is built: after a few seconds of work the click no longer counts as a
  user gesture.
- **Measured** on two real project DWGs (3.9 MB + 0.14 MB, 78,100 + 47,495 elements): the file is
  2.9 MB, saving takes 0.3 s, reopening 8.6 s. The vertices come back identical to 1e-7°. Memory peaks
  around 800 MB right after opening and falls back to ~360 MB within seconds. It is stable over
  repeated opens.
- **Web map swap.** `loadWebMap` now also moves the images, the added portal layers, the web services
  and the image handles into the new map. Before, they vanished from the map on a region change or at
  sign-in while staying in the list; the DWGs already moved (`cadReattach`).

**Web services (WMS / WFS / ArcGIS REST).**
- **The browser's two rules, not AGOL's.** An https page cannot load http resources (mixed content:
  blocked, no way around it from the page). Capabilities and data are read only if the server allows
  cross-origin reads (**CORS**). An `http://` URL is tried as `https://` automatically; if that fails, the
  message says why. Tested on 2026-09-23:
  - the Sardinia GeoServer (the catasto one) works (CORS `*`) — but see *The cadastre bridge* below: on
    2026-10-08 it stopped answering GetMap;
  - the **national PCN WMS** (`wms.pcn.minambiente.it`) has **no CORS** and cannot be read.

  Serving such services needs a proxy on the server side, e.g. an Azure Function on the future host,
  which would also fix http-only services. The future CSP must then allow `connect-src`/`img-src` to the
  services in use.
- **Recipes, not layers.** Each added layer carries a recipe (`_projSvc`: type, url, chosen layers)
  kept in `addedServices`. The recipe goes into the auto-save (`services`) and the project, and
  `svcCreate` rebuilds the layer from it. A service that no longer answers at start-up is dropped from
  the auto-save with a message.
- **WMS.** The layer is created with only the chosen sublayers. As for the catasto, when the
  capabilities list EPSG:3857 it is forced, because GeoServer rejects 102100. For zooming, the extent is
  that of the chosen sublayers: the service's own extent is often all of Italy.
- **The cadastre bridge** (`scripts/catasto_proxy.py`; the same in Node in `scripts/ponte_catasto_azure/`, 2026-10-08). The AdE WMS
  (`wms.cartografia.agenziaentrate.gov.it/inspire/wms/ows01.php`) serves EPSG:6706/4258 and the ETRS89 UTM
  zones, never 3857, and sends no CORS header (the WFS the same, GML only): useless from a browser. The bridge
  takes the SDK's GetMap in EPSG:3857 (or 102100/900913), converts the two BBOX corners to lon/lat and asks the
  AdE in WMS 1.1.1 with `SRS=EPSG:4258`, same WIDTH/HEIGHT, LAYERS, STYLES, FORMAT, TRANSPARENT, and returns the
  PNG untouched with `Access-Control-Allow-Origin: *` (images cached 10 min). No warping: inside one image the
  difference between the equirectangular and the Mercator grid is below a pixel at 1:25,000 and 2048 px (the
  vertical scale factor varies by about Δφ·tan φ, 1e-3 at most). GetCapabilities is forwarded with the GetMap
  `OnlineResource` rewritten to the bridge and `EPSG:3857`/`102100` added after `EPSG:4258`, so ArcGIS Online
  and QGIS can use the bridge as a normal WMS; the app also pins `mapUrl` to the bridge after load
  (`fixMapUrl`), because the SDK takes the GetMap URL from the capabilities. Only the AdE host is reached: it
  is not an open proxy. WIDTH/HEIGHT above 2048 → 400; other requests (GetLegendGraphic…) pass through. The
  Sardinia GeoServer (`webgis.regione.sardegna.it/geoserver/dbu/wms`, layers `AdE_*`) is a cascade of the same
  service; on 2026-10-08 its Java could not validate the AdE TLS chain ("Internal error PKIX path building
  failed") and every GetMap came back as an HTTP 200 `text/xml` exception, which the SDK swallows (no event, no
  console line) — hence the probe. **The Azure Function is ready but not installed**: it lives in
  `scripts/ponte_catasto_azure/` (with its own README) and the owner decided on 2026-10-09 not to touch the
  organisation's GitHub Actions for now (no control over them, unfamiliar: "a big expense without understanding
  exactly what I am doing"); do not propose it again unless asked. On the site the probe therefore skips the
  404 and the toast names only the Sardinia source, until Regione Sardegna repairs its trust store; locally
  `serve_https_catasto.py` already serves the bridge. The portal item `382d9c13…` ("IT - AdE - Cartografia
  Catastale (Web Mercator)", in 18 web maps) points at the Sardinia GeoServer too, and is broken the same way.
- **WFS.** ArcGIS `WFSLayer` wants WFS 2.0 with GeoJSON output, and before loading it also asks for a
  GML sample. Some GeoServers refuse that GML request: GeoBretagne serves GeoJSON fine and fails the
  GML one. When `WFSLayer` fails, the fallback runs a `GetFeature` in GeoJSON/WGS84 (2.0, then 1.1) of
  up to 20,000 features and shows them as a local `GeoJSONLayer`. The axis order is checked against
  the WGS84 box from the capabilities. The result is a snapshot: it does not refresh when panning, but
  it filters and queries like any layer. The choice is kept in the recipe (`mode: 'geojson'`).
- **ArcGIS REST.** `Layer.fromArcGISServerUrl` decides the kind:
  - map service: one `MapImageLayer`, with only the chosen sublayers switched on (and their groups);
  - feature service: one `FeatureLayer` per chosen layer, with names read from the service JSON;
  - anything else: the layer as it is.

**Map-layer features as geoprocessing sources** (`gpPickLayerFeature`).
- Our own objects (parcels, drawings, results) keep priority. Only when the click hits none of them
  does it look at the map's layers:
  - `hitTest`, then a query by OBJECTID, for feature/WFS/GeoJSON/CSV layers. Only what is drawn gets
    hit, so filters apply;
  - `identify` on visible, in-scale sublayers for map services.
- The geometry of the click is **not** used: it is generalized for drawing (Finistère: 5,598 vertices
  against 192,532). It is re-read in full, in WGS84.
- `_srcKey` (layer URL + OBJECTID; the title for local layers, whose blob URL changes) prevents
  duplicates. `src_attrs` keeps the original simple-valued attributes (at most 80) for the export.
  Both go into the auto-save and projects.
- **Heavy geometries.** The ArcGIS geometry engine is synchronous and costs about 0.3–1 ms per vertex.
  Measured on Finistère (192,532 vertices), every one of these was far too slow:

  | Method | Time |
  |---|---|
  | geodesic buffer | ~3 min |
  | 4.34 `geodesicBufferOperator` on 65,819 vertices | 28 s |
  | planar buffer in UTM | 54 s, plus 8 s of projection |
  | ArcGIS `generalize` alone | 17 s |

  So above `GP_HEAVY` (5,000) vertices, `makeBuffer` first simplifies with its own Douglas-Peucker in
  metres, which takes a few ms. turf's simplify rejects polygons whose islets collapse. The tolerance is
  capped at 1/10 of the buffer distance; the simplification is stated in the message and in the result
  name, and the message shows *before* the synchronous work.
  - 500 m: 50 m, 6,070 vertices, 4.6 s;
  - 100 m: 10 m, 28,965 vertices, about 25 s, area within 0.02% of the exact buffer.

  The other operations only warn beforehand above 20,000 vertices.

**Attribute filter.**
- Available on layers with `definitionExpression` and `queryFeatures`: feature layers, WFS,
  GeoJSON/CSV, and map-service sublayers. WMS has no filter in the protocol.
- The expression is tried with a count before being applied, so a wrong SQL leaves the previous
  filter in place with a message instead of an empty, broken layer.
- `contiene` / `inizia con` use `UPPER(field) LIKE`, so they ignore case. A date typed as dd/mm/yyyy
  covers the whole day.
- Filters go into projects (`def`, and `sdef` for sublayers), not into the auto-save. After a reload
  the layers come back unfiltered, like their visibility.

- **UX refactor traps (2026-10-01).** Top-level function declarations are properties of `window`: assigning
  `window.gpSelectOp = …` overwrote `gpSelectOp` itself and the function called itself forever (stack overflow)
  — use another name (`gpOpenOp`). The **i** buttons are wired by the selector `.im-info`, not `.imod .im-info`:
  the ones outside modules (geoprocessing steps, Tools card) were mute. A sketch interrupted by a tool is
  completed in `modeOffDraw` as well as in `cadModesOff`: clicking an armed tool button cancels the sketch there
  first, before `cadModesOff` ever runs. The centred `.wmsel` container has a CSS `transform`, which creates a
  stacking context without a `z-index`: the open menu was painted under `#mapWrap`, so `.wmsel.open` gets
  `z-index:1500` (and only while open, not to cover the modals). The wide-panel card grid needs
  `.paneBody .gpcards` (specificity), the later base rule overrode `.gpcards`. The Raccolta's export ticks and
  the export window share one state (`exSel`): keep them in sync both ways. The search box focus on open needs
  the fallback `sb.querySelector('input').focus()` after 60 ms.

- **Owners file (2026-10-05).** The graphics are ordinary imports (`_kind:'import'`) with one more attribute,
  `own = {k, t, l, n, st}`: class key (upper-cased short names, sorted — two parcels with the same owners in a
  different order are one class), group title, map label, number of owners, SISTER status. `ownIs(g)` is the
  test used everywhere.
  - **Three places keep the personal data in:** `serializeToolGeoms(true)` (auto-save) skips these graphics,
    `sfCandidates()` (portal) skips and counts them, and `applyWork` does not print the record when a restore
    fails. A new way of saving or publishing imports must add the same test.
  - The row name is the parcel, not the person, on purpose: geoprocessing results take their names from the
    inputs and are saved everywhere.
  - Text from the file goes to the DOM with `textContent` only (list, card, popup): it is a file from disk.
  - `GraphicsLayer` has no labels: the names are text graphics on the centroids in a second layer
    (`ownLblGL`, hidden from the layer list, same scale limit as the parcel labels), rebuilt 80 ms after any
    list redraw or visibility change (`ownLblSoon`). It is added again after a web map swap, like `toolsGL`.
  - A click on a graphic of `toolsGL` starts a `SketchViewModel` update, and while that is active the SDK turns
    `view.popupEnabled` off — a `popupTemplate` alone never shows. For these parcels the update is cancelled at
    its `start` event and the popup is opened by hand (`ownShow`, at the point of the last `immediate-click`).
    Nothing in the app calls `sketchVM.update()` itself, so only clicks are intercepted; a shift-click on two
    objects still edits.
  - Colours: twelve fixed ones, then hues at the golden angle; a new owner never takes a colour already used
    by another group, and an owner already on the map keeps the colour it has (also after a recolour).

- **Images v2 (2026-10-06).** Transforms are 3×3 homogeneous matrices (`h3Ap`, `h3Mul`, `h3Inv`, `h3Fit`)
  from image pixels (origin top-left, y down) to view coordinates. `h3Fit(pairs, model)` normalises both sides
  (centroid to 0, mean distance √2) before the normal equations: pixels are thousands, map units millions, and
  the projective rows carry their products. The similarity model is `X = a·px + b·py + c`, `Y = b·px − a·py + d`:
  rotation and scale **with the reflection** that turns the pixels' y-down into the map's y-up — the
  orientation-preserving form fits nothing (first attempt). `imgH(im)` is the current placement, an exact
  projective fit of the four corners; its inverse turns a map click into a pixel.
  - The SketchViewModel turns `view.popupEnabled` **off while an update is active** and restores it on
    cancel/complete; capture the value before the first `update()` (`imgPop0`) and restore it yourself in
    `imgDone`/`imgPtsEnd`, or the popups stay off after a points session.
  - `cancel()` on the frame may leave the graphic where the user dragged it: the image georeference is the
    source of truth, the frame is rebuilt from it (`imgFrameSync`) before every `update()`.
  - Transform events arrive on every pointer move with the live geometry; setting a new
    `ControlPointsGeoreference` each time is fine (Esri's own sample does it). A folded ring is skipped.
  - While an image is edited the main sketch's `updateOnGraphicClick` is paused: a click on the frame over a
    drawing would otherwise start two updates.
  - Arrow keys: the SketchViewModel does not move the selection itself (checked in the browser), so the
    one-pixel nudge is ours; it cancels and restarts the update, which is cheap.
  - World files: the six numbers are A, D, B, E, C, F in that order (rotation terms in the middle); C and F
    are the **centre** of the top-left pixel.
  - **pdf.js and the AMD loader.** The UMD build of pdf.js sees the ArcGIS `define` and registers itself as an
    AMD module instead of setting `window.pdfjsLib`: use the ESM build through a dynamic `import()` (the same
    trick as the DWG reader). A cross-origin worker script cannot be given to `new Worker(url)`; a blob module
    that `import`s it can, and `GlobalWorkerOptions.workerPort` takes it. Only loaded on the first PDF.
  - The card gets the image's `ow/oh/page/dpi` through `createGeorefImage(..., extra)` **before** `editImage`
    opens it: setting them after the call came too late for the dpi field (the first attempt).
  - **A file picked from the undocked panel is from another window** (another realm). pdf.js checks the data
    with `instanceof ArrayBuffer`, which is realm-bound, so `getDocument({data: await file.arrayBuffer()})`
    throws "Invalid PDF binary data: either TypedArray, string, or array-like object is expected" (the user's
    report of 2026-10-07: "many PDFs give an error"). Wrapping the foreign buffer in a `Uint8Array` passes the
    check but the worker never answers. The fix is a copy into this window: `new Uint8Array(buffer).slice()`.
    `geotiff.js` and `URL.createObjectURL` do not care. Reproduced and verified with a `File` built in an iframe.

## 6. Security & distribution

- The Zornade key is a read-only token **embedded in the source** (`DEFAULT_KEY`). This is a deliberate,
  accepted choice for an internal, access-controlled host — and it is why the page must not go on GitHub
  Pages or any public URL. **⚠ As of 2026-09-22 the GitHub repository `davo3188/custom-AGOL-web-map` is
  public** and contains the very key still in use: make the repository private and regenerate the key
  from the Zornade dashboard. In the longer run the key should leave the source (an Azure Function in
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
| Functions, variables, CSS ids never used | 0 (dead code removed on 2026-09-23; the second, hidden `msg()` and `#out` removed on 2026-09-24) |
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
- 2026-09-23: **Save / open project** (`.axpo`, §4 and §5), and images and added portal layers no longer
  lost on a web map swap. Backup before the change: `geoportale_axpo_pre-salva-apri-progetto_2026-09-23.html`.
  Tested signed out with a public Esri web map and a public item standing in for the portal. The real
  sign-in branch still has to be exercised.
- 2026-09-23: **DWG layer list** no longer jumps back to the first layer on every on/off click. The
  list is rebuilt on each change, and its own scroll box was recreated at the top; the scroll position
  is now kept. Also added:
  - web services by URL (WMS/WFS/REST);
  - the attribute filter;
  - the layer search box;
  - *Rimuovi dalla mappa* for added layers, which could not be removed at all before.

  Backup: `geoportale_axpo_pre-layer-servizi-filtro_2026-09-23.html`.
- 2026-09-23: **map-layer features as geoprocessing sources**, plus the guard for heavy geometries in
  the buffer. Tested with real clicks:
  - a REST polygon (9 vertices, buffer contains the source);
  - Texas through identify (847 vertices);
  - Finistère from the GeoJSON WFS (192,532 vertices, see §5);
  - the WMS-only message.

  Backup: `geoportale_axpo_pre-fonti-da-layer_2026-09-23.html`. That backup was first copied after the
  edit by mistake, then rebuilt by reverting it; it matches the pre-change file in lines and bytes
  (5,148 / 396,623).
- 2026-09-23: **Import tab reorganised** into collapsible sections with plain text and an *i* window per
  complex module (see §4). Also fixed a race: *Svuota il lavoro* during the start-up restore of web
  services got the services back afterwards. `svcGen` now drops what arrives after a clear. Backup:
  `geoportale_axpo_pre-import-sezioni_2026-09-23.html`.
- 2026-09-23: **Geoprocessi tab reorganised** the same way, with drawn examples in each *i* (§4). The
  open/closed memory of both tabs now uses the key `axpo_sezioni_aperte` (the old `axpo_import_aperte`
  is still read). All operations were re-run after the change: Parte comune 24.24 ha + Differenza
  127.27 ha = Unione 151.52 ha. Backup: `geoportale_axpo_pre-geoprocessi-sezioni_2026-09-23.html`.
- 2026-09-23: **Ricerca and Disegno tabs reorganised** the same way (§4), so all four working tabs now
  share the layout. The old `.step` CSS went with the old markup. Four defects found and fixed on the
  way, each reproduced first:
  - *Pulisci disegni* also deleted every **geoprocessing result** (they are `_kind:'drawing'` with
    `_sub:'geoproc'`), with no confirmation. Now *Cancella tutti i disegni* asks, and removes drawings only.
  - A drawing (or buffer, result, import) **moved or reshaped on the map was never saved**: nothing
    listened to the sketch `update` event, so a reload brought the old shape back, and the stored area of
    buffers and results stayed stale. Delete during an edit removed the object from the map but not from
    the list or the save. Handlers on `update` (complete) and `delete` now recompute `area_ha` and call
    `refreshDl()`.
  - *Da un disegno* with a **point** failed ("Errore nel campionamento": a point has no extent).
  - "1 particelle aggiunte": plurals of both search messages.

  Tested signed out: the three search modes, a foglio/particella search, the comune in the
  closed title, a search from a point drawing, a move, a reshape and a delete with the save checked,
  a buffer's area after a reshape (4.27 → 6.39 ha), the clear with a union and a buffer left intact, typed
  deflection/distance (90° / 120 m placed exactly), both tours, light and dark theme. Backup:
  `geoportale_axpo_pre-disegno-ricerca-sezioni_2026-09-23.html`.
- 2026-09-23: **CAD-style drawing** (§4): grid, snapping to DWGs and map layers, azimuth, undo/redo, shapes
  with measurements, parallel copy; three new *Disegno* sections with drawn examples in their **i**. Tested
  signed out:
  - rectangles 100×50 and 120×40 at 37°, circle r 50: sides, radii and areas exact to the millimetre;
  - *Da un lato* on a parcel side (15.67°), rectangle parallel within 0.004°;
  - 5 m inset of the parcel: 5 m ± 7 mm from the boundary, 7.1 m at the re-entrant corner as expected;
  - parallel line at 10 m (9.985 m), single side at 10.000 m on the clicked side, outset containing the source;
  - snapping to a real DWG: a click 0.52 m from a vertex lands on it with the option on, stays 0.52 m away
    with it off;
  - grid on at 10 m, aligned with two clicks (45°, as the clicked segment), straightened;
  - undo/redo with the buttons and Ctrl+Z/Ctrl+Y: drawing a line (last vertex), a finished drawing, a move
    with the mouse, a delete, *Cancella tutti i disegni*; *Svuota il lavoro* resets the history;
  - Esc, the mode banner, one map mode at a time, light and dark theme.

  **Not tested**: snapping to portal feature layers (needs sign-in; same mechanism as the DWG layers),
  `edgeOperation: "offset"` (not used). Backup: `geoportale_axpo_pre-disegno-cad_2026-09-23.html`.

**Checked and not a problem** (do not reopen): `projection.project` applies the default datum
transformation by itself, so Monte Mario GeoTIFFs are placed right; the AREAS `edits` handler gets the
layer's own SR from `queryFeatures`, not Web Mercator.

**Second audit (2026-09-24, by another model, read-only; checked here line by line)** — fixed:
- The three messages about deleting from the portal passed the text through `esc()`, but the toast writes plain
  text (`textContent`): an apostrophe showed as `&#39;`.
- `onSignedIn` set `signedIn=true` *before* `portal.load()` and swallowed the error: a portal that failed to
  load still showed «Connesso» and was retried at every `credential-create` (~20 with the General Map). Now
  «connected» only after the load; on failure one message, no automatic retries, «Accedi» retries (`force`);
  concurrent events share one load (`signInBusy`). Tested with a fake `Portal`: failing, slow ×3, then OK.
- `Login annullato: undefined` when the rejection had no `message`.
- `msg()` had two definitions: the first wrote into a hidden `#out`, the UI glue replaced it with the toast —
  messages sent before the replacement were invisible. Now one definition, the toast.
- **Mode logic written three times** (`cadModesOff`, `offSel/offDraw/offSub` in the UI glue, `offModes` of the
  coordinate pin): now `modeOffSel/Draw/Sub` + `cadModesOff` in the main script, and the other two use them
  (`DRW_BTN` once). While there, a bug of the same day: with an exact shape active, pressing *Rettangolo*
  again re-armed it instead of switching it off — the family handler ended the shape mode together with the
  tool before the button's own toggle (`drwModeEnd(fam==='draw')` now). The mode banner shows the exact-shape
  hint before the armed tool's.
- The pin waited for `window.view` polling every 400 ms: now the event `axpo-view`, fired where the view is born.
- The `drawType` comment and the size in §0 of this README.

Not taken: the 350 ms poll of the mode banner stays (cheap, rewrites only on change; events would mean touching
every mode start and end); a `swallow()` wrapper around every catch (see item 6); duplicated listeners when the
panel is undocked (they go on the new window's document and die with it); PKCE instead of the implicit flow — the
right direction, but it needs an OAuth app without a client secret on the portal and a real login test (the
comment at `registerOAuthInfos` explains why *implicit* was chosen).

**Third audit (2026-09-24, another model, static reading plus mocks; every point checked here first)** — all
confirmed; fixed and re-tested in the browser with fakes (no real AGOL or Zornade writes):
- **Dissolve no longer produces a partial area.** A failed `turf.union` was swallowed: the result lacked a parcel
  but declared all of them, in the export *and* in *Promuovi → AREAS*. Now `buildFC` stops with the parcels that
  failed, or those without an outline (point only); export and promotion show it and write nothing. Promotion
  also refuses parcels without an outline (AREAS is a polygon layer).
- **The web map title from a `.axpo`** went into `cvNote.innerHTML` unescaped when the map failed to load: now
  `esc(label)`.
- **No Web Mercator area.** Without a valid geometry the parcel took Zornade's `area_m2`, which is Web Mercator
  (~2× at 45°). Now the area stays empty (`area_nd`), the row says «area non disponibile» and the total counts them.
- **XLSX** holds only parcels: the export window now says so and switches the other categories off; with no
  parcels it stops with a message instead of writing an empty sheet.
- **Area selection tells Zornade errors from «no parcel»:** failed `locate` calls and unreadable details are
  counted; none answering → an error, some → a warning that the area may be incomplete.
- **Save results**: every object needs its own result (a missing one is an error and the object stays to save);
  after a failed update it is re-added only if a `GlobalID IN (…)` query says it is gone — permission or domain
  errors stay errors.
- **Fast region change**: `loadWebMap` numbers its requests; a map that arrives after a newer request is dropped
  (before, the older one could win and take parcels, tools, DWGs and images with it).
- **AREAS editor** is torn down also when the new map has no AREAS (a Check Vincoli map), with a note in the drawer.
- **Automatic site**: if the query on the common extent is cut (`exceededTransferLimit`), one query per object;
  still cut → a warning in the save message.
- Minor: the unused `--axpo-sky/sun/earth` variables became a comment (brand values kept), `h1,h2,h3` weight set
  once (700, as it was in effect), an unused `var w`.

**UI/UX review (2026-09-24, another model with design skills; every claim checked here; applied in this order)**
1. *Area selection is indicative, and says so.* A fixed note in *Sulla mappa…* and *Seleziona da un disegno
   esistente*; the result reads «N individuate campionando l'area: una particella molto piccola può sfuggire»
   instead of «N nell'area», which sounded like a complete inventory.
2. *Errors and partial results stay on screen.* `msg()` keeps an `err` toast until its × or the next message
   (role `alert`); others go after 6 s, 10 s with an action. `msg(t, cls, {action:{label, fn}})` adds a button:
   one rule for where results go — no drawer forced open (it shrinks the map while you work), but «Vedi
   nell'Elenco ›» (`vediElenco(sec)`, `showInElenco`) opens the Elenco on the right section. Used by the parcel
   search, map clicks, area selection and every import (which used to open the Elenco on their own).
3. *The map stays usable.* `ensureMapRoom(prefer)` in the UI glue: under 360 px of map it makes room one step at a
   time, measured after the 0.2 s transition, honouring what was opened last — a drawer just opened narrows, then
   collapses, the left panel; the left panel reopened or widened, or the window narrowed, closes the drawer first.
   A message says what it closed. Runs at start-up too (at ~500 px the map used to get zero width and never load).
4. *Dialogs*: Esporta, Promuovi, Salva sul portale, CAD, Guida, Aiuto get `role="dialog"`, `aria-modal`,
   `aria-labelledby`; opening moves the focus to the first control, Tab/Shift+Tab stay inside, closing gives the
   focus back to the button that opened it (a `MutationObserver` on the `style` of each modal).
5. *Promotion labels in Italian* — «Classe dell'area», «Tipo di area», «Fonte del dato», «Tipo di progetto» — with the
   field name beside them (`· Area class`…) for whoever knows the layer. To confirm with the people who fill AREAS.
6. *Rail*: it declared `role="tablist"` without tabs; now `role="tab"`, `aria-selected`, `aria-controls` to the
   `role="tabpanel"` panes, roving tabindex, arrows/Home/End.

Not done: the 10–11 px texts (to try with browser zoom first) and the usability test the review proposes
(5–8 prospection managers, tasks on staging data; the «site of work» task should become «after saving, which
site did the object go to?», and it is worth asking whether *Promuovi → AREAS* and *Salva sul portale* are told apart).

**Example data (2026-09-24).** The user's rule, shared with PV Predesign: codes, GUIDs and parcels used as
examples are always invented. The site code example was a real AREAS project code (placeholder and title of
`sfSiteCode`, the `sfFindByCode` message, the **i** of the site): now **C0000**, checked read-only on layer 426
(C0000: 0 features; the old one: 1). The foglio/particella placeholders are fictitious too (`es. 1`,
`es. 1, 2, 3`; their origin was unknown). The test notes in this README no longer name the project site or a
parcel. On the user's request the public repo history was rewritten the same day to take the real code,
site and parcel out of every commit (`git filter-branch`, only those strings changed; the original history is
kept locally as a git bundle in `backups/`). Backups: `geoportale_axpo_pre-codici-finti_2026-09-24.html`,
`README_tools_pre-codici-finti_2026-09-24.md`, `geoportale_axpo_pre-esempi-particelle_2026-09-24.html`.

**Small changes (2026-10-01, the user's list).**
- **Compass** always visible under the zoom buttons (SDK `Compass`, loaded on demand): the map can be rotated
  (right-drag, two fingers), one click puts north back up.
- **Cadastre**: the app's WMS is the only one, at 50% transparency with the popup off (§5).
- **«Mostra etichette»** in each layer's panel in *Layer*, next to «Popup al click», only when the layer or
  sublayer has labels configured (`llLabelable`: `labelsVisible` plus a non-empty `labelingInfo`; a layer still
  loading gets it when ready). Saved in `.axpo` projects as `lab`.
- **Aggiungi layer**: after a successful add the result list is cleared and the search text kept; if the add
  fails the list stays, with one error line on top.

  Tested signed out: compass after a 45° rotation (back to 0°); a public web map with a fake «IT - AdE -
  Cartografia Catastale» layer injected (removed, the app's WMS on top at 0.5 with the popup off, a project state
  for it not counted as missing); a REST layer with labels (checkbox, on/off, saved and restored) and one without
  (no checkbox, nor on the WMS); a public item added (list cleared) and a non-existent id (list kept, one error
  line). Backup: `geoportale_axpo_pre-bussola-catasto-etichette_2026-10-01.html`.

**DWG fix (2026-10-01, evening; ported from the UX prototype, its commit `772e554`).** The user's PVcase
layout came in with the modules "scattered": 305 of 334 trackers shifted by metres and 29 thrown to (y, −x),
7,000 km away, then dropped as "far elements". Cause and fix in §5, *OCS (tilted planes)*: the reader treated
the tilted Object Coordinate System of each tracker block as the world plane and ignored the insertion z; now
the entity pipeline runs on 3D matrices with the arbitrary axis algorithm (`c3Ocs`, `cadOcsM`), the INSERT
carries x, y, z of the insertion point, `zScale` and the base point z, LWPOLYLINE/POLYLINE2D their elevation,
ARC/CIRCLE/ELLIPSE the centre z, LINE/POLYLINE3D/LEADER/3DFACE/POINT the point z. Tested against arcpy as
ground truth: on two PVcase layouts (334 and 325 trackers) the extent of the module layer matches arcpy to the
millimetre, where before it was off by metres on both; a third DWG without tilted planes (163,344 items) gives
a byte-identical result (same coordinate checksum); through the real import flow the layout loads with 2,608
items in 18 layers, 334 modules inside the fence. The copy in `tools/` gives exactly the prototype's output on
the three files. Not tested: blocks drawn in elevation views (vertical OCS), which degenerate to segments in
plan as in AutoCAD. Backup: `geoportale_axpo_pre-dwg-ocs_2026-10-01.html`.

**Hatch fix (2026-10-02).** After the module fix the user noticed the road hatches of another layout: a
half-disc of 16 m radius over the site entrance and a turning area twice its size. Cause (§5, *Data model*):
the library returns the counter-clockwise flag of hatch arc and ellipse edges as the number 0/1, the reader
tested `=== false`, so clockwise edges were never mirrored and the arc went around the other side of its
chord. One-line fix in `cadHatchRings` (`isCCW === 0` counts as clockwise). Checked on that layout: of 298
hatches the 6 with clockwise arcs changed, and each now has exactly the extent of its boundary polyline
(bulges included); on the three DWGs of 2026-10-01 only the layers with such hatches changed (heavy
traffic, circulation, legend), and their extents now match arcpy's; the 163,344-item DWG keeps the same
item count. Backup: `geoportale_axpo_pre-hatch-cw_2026-10-02.html`.

**Slopes from the DTM (2026-10-04, the user's request; §4, *Pendenze da DTM*).** New Tools entry: threshold in
percent chosen by the user, DTM shown over the gross area plus 25 m, exclusion *Pendenza elevata* into the
Raccolta, grid and threshold passed to PV Predesign through the `.axpo` (Predesign computes the same cut today
with a limit fixed by the structure; its import is to be adapted — request in `../COORDINAMENTO.md`, prompt in
`../reports/prompt_pv_predesign_pendenze_2026-10-04.md`). Tested in headless Chrome and in the built-in browser
on an invented 15.64 ha area in hilly ground: the ported functions give **the same arrays as Predesign's
originals** loaded side by side (gradients, mask, small-patch removal, rings, grid definition) and 11.1803 % on
a plane of 10 % × 5 %; the grid juts 30 m beyond the area (25 m plus one cell) and the buffer mask measures
19.82 ha against 19.84 of the true 25 m buffer; a cell elevation equals the same point queried alone (309.31 m);
at 10 % the exclusion is 13.66 ha with 0.00005 ha outside the gross area, at 20 % the same drawing becomes 7.26
ha; the project keeps the grid bit for bit, and reopening it restores image, threshold and view; removing the
image drops the grid; sources also checked with a fake site of work and three fake parcels (merged into two
parts). Real time in the browser: 1.9 s for 10,560 cells. Not tested: saving the exclusion to the portal
(sign-in), very large areas at the 400,000-cell cap, the undocked panel. Backups:
`geoportale_axpo_pre-pendenze-dtm_2026-10-04.html`, `README_tools_pre-pendenze-dtm_2026-10-04.md`.
The same day the user decided that **the two tools must compute alike in everything**: so the limit can also
be two values, north–south and east–west, as in PV Predesign, which will get the same free choice of limits
(request in `../COORDINAMENTO.md`); that no field for the limit goes on the layer; and that the comparison with
the company's design standards **stays in PV Predesign only** — a first version that also listed those values
here was withdrawn before reaching the public repository. Retested on a clean browser profile: `slpMask` equals
Predesign's limit mask for `{maxNS:10, maxEW:10}`, `{maxAny:15}`, `{maxNS:8, maxEW:12}` and `{maxNS:10}`; on
the same area 10 % north–south and east–west excludes 12.85 ha (13.66 with 10 % in every direction), 10 % and
12 % give 12.62 ha, 15 % in every direction 10.91 ha; the limit survives the project round trip and comes back
in the fields; after removing the DTM (as after a reload) a new extraction updates the one existing exclusion.
The bottom sheet got 72 px of bottom padding: the Tools button covered the last command of a tall card.
Backups: `geoportale_axpo_pre-pendenze-direzioni_2026-10-04.html`,
`geoportale_axpo_pre-limiti-a-mano_2026-10-04.html`.

**Guide, tours and info windows realigned (2026-10-04, the user's request: "are the .axpo save, the tour and
the tutorial aligned with the changes?").** Audit of the three against the interface after the UX refactor and
the slope tool. *Project file*: aligned — drawings with their codes and `sf_slope`, DTM grids, site of work,
services, DWGs and images are all saved and restored (the round trip is in the slope test above); one gap closed:
reopening a project now also puts the grid cell size back in its field. Still not saved, as before: the drawing
grid (step, origin, rotation), an open proposal. *Quick guide* (`#helpModal`): it had no entry for Geoprocessi,
Importa dati or Tools, and described neither the two steps of Disegna nor the DTM in projects; it now has eleven
entries in the order of the interface (left panel, Ricerca, Disegna, Geoprocessi, Importa dati, right rail, Mappa,
Layer, Esporta / Migra / Salva, Tools, Progetto), with *Pendenze da DTM* and the right-click menu. *Tours*: every
selector of the two tours exists; texts updated (what sign-in is needed for, *Ultimo disegno*, the right rail in
three groups, the project with the DTM and PV Predesign), the Tools step now lists *Pendenze da DTM*, and a new
step opens that tool (the next step closes it): "Tutti gli strumenti" has 14 steps, "Inizia qui" 7. *Info
windows*: the *Copia* text named a button that no longer exists and the *Disegna* text a counter in a title that
is gone; both corrected, the *Disegna* window is now titled *Disegna*. Tested in headless Chrome: both tours
walked step by step to the end with a highlight box on a different, existing element at every step and the
popup inside the screen; the guide lists the eleven entries and none of the old names (Elenco, Contesto sito,
Salva sul portale, Promuovi); nine info windows open with their titles; no script error. Backups:
`geoportale_axpo_pre-guida-tour_2026-10-04.html`, `README_tools_pre-guida-tour_2026-10-04.md`.

**UX refactor (2026-10-01/02, from the user tests; prototyped in `../prototipo_ux/`, approved and integrated on
2026-10-02).** Four problems from the user tests, each a commit in the prototype's local git (its
`REPORT_UX.md` has the diagnosis, the options, the decisions, the verifications, before/after screenshots and
the backlog; `FASE2_diagnosi_e_proposte.md` the wireframes): **P1** *Disegno* and *Elenco & azioni* read as
twin tool bars → *Disegna* (verb) / *Raccolta* (noun, count badge, grouped right rail, *Ultimo disegno* box, one
lexicon); **P2** geoprocessing with an invisible prerequisite → three steps, operation cards with their own
conditions, *Esegui → Eseguito*, errors in plain words; **P3** the web map swapped as a side effect of the
search region → the **🗺 Mappa** selector in the top bar, region and map independent with a shortcut; **P4**
four tools in four places → the **Tools** speed-dial (Misura, Profilo altimetrico, Analisi di contesto,
Coordinate sconosciute, Percorso su strada), one floating card at a time, the elevation profile inside the
single mode switch, a sketch in progress completed rather than dropped. Second round (the user's notes after
trying it, signed in too): top bar in three zones, Raccolta dock (Esporta / Migra particelle in AREAS
Collection / Salva disegni in Site Features), one export window with per-object ticks shared with the
Raccolta, *dissolve* only in the AREAS migration, colour picker with swatches and recents, Misura as the first
tool, scale bar bottom-centre at 70%, coordinates in the corner, search as a lens. Third round: the map menu
painted under the map (transform → stacking context: `z-index` only while open), *Disegna* in two steps with the
compact site row, *Copia* as a Geoprocessi card, *Importa dati* as four open cards, a real grid for the wide or
undocked panel, and the "lighten the map" rule (OAuth notice in the sign-in tooltip, **i** on hover/focus,
discreet attribution, small banner). Palette kept: contrast failures fixed with the existing tokens
(`--accent-strong`, `--link`). No service, geoprocess, query or data model changed — the one data change is
the road route always writing category `connection` / type `estimated`. Verified by the assistant without
sign-in (keyboard order and Esc chain, ARIA roles and states, `prefers-reduced-motion`, 1366/1024/768/375
layouts, contrast on the tokens, a real-click sketch completed from Tools, every former function still
reachable) and by the user signed in (context, route, Check Vincoli swap, AREAS migration). Integration: the
prototype file equals `tools/` plus its 15 UX commits replayed one by one, byte for byte (the two DWG fixes were
already in both), so it was copied over; a headless smoke test of the copy loads the map, opens the Tools
menu and the map menu, counts the cards (8 operations, 5 tools) and reports no script error. Still untested
by the assistant: the undocked panel with the Tools menu, the profile widget with real clicks after a web map
swap. Backups: `geoportale_axpo_pre-prototipo-ux_2026-10-02.html`, `README_tools_pre-prototipo-ux_2026-10-02.md`.
Backlog (from the report, not done): geoprocessing in a Web Worker with a real cancel, Raccolta on the left, a
phone layout, zoom and compass as a chip, *Impostazioni avanzate* in one place, Stampa and Modifica AREAS out
of the rail, a user test with 5–8 project managers.

**Owners file classified by owner (2026-10-05, user request).** Importing the GeoJSON of the *Estrazione
intestatari* tool gave rows without names in *Importati*: `impAdd` kept the geometry and dropped every
property, and the file's `nome` is empty. Now the file is recognised and classified by owner (§4 *Owners file*,
§5 for the traps). Checked in headless Chrome with a file of **invented** names (8 parcels: one company with
two, two people in both orders, one alone, one "ente urbano", one not found, one never searched): 4 owner
groups plus *Senza intestatari*, colours per group, 6 labels that follow the eyes and the switch, detail card
and popup as text (a name containing `<b>` stays text), group tick, auto-save without the parcels and without
any name in `localStorage`, project capture with all 8 and their properties, portal candidates without them,
GeoJSON export with the original properties, reopen from the captured work (groups, colours, labels, popups
back), a second file in the 1.1 format (same owners → same colours, a new one → an unused colour), undo, no
name in the console. On screen (built-in browser): colours, labels, the grouped list in a 285 px drawer, and a
real click opening the popup. The user's own file was imported once in the test page and only its structure
read (6 parcels, one group). The quick guide, the *Importa dati* tour step and the hint under *File di
geometrie* mention it; both tours still run to the end. Not tested: with the portal login (the save window's
count line), files with hundreds of parcels, the undocked panel. Export check asked by the tool's author:
GeoJSON and XLSX already carry `belfiore`/`Cod_Belfiore`, `foglio`, `particella`, `provincia`, `ncr`, and the
export window already limits to the ticked rows — nothing changed there. Backups:
`geoportale_axpo_pre-intestatari_2026-10-05.html`, `README_tools_pre-intestatari_2026-10-05.md`.

**Export file name (2026-10-05, user request: "comune, data e numero di particelle").** §4 *Export*. Checked in
headless Chrome with the downloads intercepted: one municipality in the five formats
(`Borgo_Inventato_2026-10-05_3part.*`, the shapefile zip holding `…2026_10_05_3part_aree.shp`), two
municipalities plus a drawing (`Sant_Angelo_Lodigiano_e_altri_1_…_7part`), two ticked rows (`…_2part`, GeoJSON
and XLSX), an accented name (`Forli_…`), a drawing alone (`export_2026-10-05`), owners-file parcels (counted),
an empty Raccolta (no file, the usual message). Not tested: a real download in the browser's folder. Backups:
`geoportale_axpo_pre-nome-export_2026-10-05.html`, `README_tools_pre-nome-export_2026-10-05.md`.

**Wrong file imported (2026-10-05, evening, from the user's first real try).** "No colour per owner": the file
imported was the parcels export, not the tool's result — since the same day the two differ only by the
`Intestatari_` prefix. Nothing to fix in the classification (the result file, read for structure only, gives
10 parcels in one group of two co-owners, one colour, 10 labels); added the message that names the right file
(§4 *Owners file*) and changed the wording of the count from "intestatari" to "gruppi", which was wrong for
co-owners. Checked in headless Chrome with the user's two files (counts only) and with the invented-names
suite again.

**Images v2 (2026-10-06, user request: "spostamento, scala, ruota, o altri metodi di georiferimento", approved
D + A + B + C of the review).** The image module was the weakest tool: four point handles moved one at a time,
nothing else. Rewritten (§4 *images and GeoTIFF*, §5 *Images v2*): world file + `.prj` (or estimated system),
GeoTIFF rotation and `.prj`, a frame with move/scale/rotate and aspect lock, corner reshape, control points
with similarity/affine/projective fits and residuals, keyboard nudges, *Ripristina*, pairs saved in the project.
Checked in headless Chrome: fits recover known transforms to 1e-9 (noisy affine: RMS as expected), inverse,
too-few and collinear points refused, folded ring refused; world file + `.prj` lands on the projected corners
exactly, without `.prj` the estimated system is 2·10⁻⁵ m away, a world file in degrees with rotation terms is
placed and rotated, an orphan world file is reported; a photo without metadata opens the frame in transform
mode with the main sketch paused; a simulated transform event moves the corners, a bow-tie is ignored, an arrow
key moves one pixel, reshape mode sets the reshape tool, *Ripristina* restores; points: a click outside the
image is refused, 2 pairs give a pure translation with zero residual, a third inconsistent pair gives an exact
affine, a fourth gives least squares with four residuals around 0.9 m, *Proiettiva* makes it exact, removing a
pair recomputes, Esc drops the pending point then leaves the mode with popups, cursor and sketch restored; the
project carries `cps` and restores them without a frame; *Fatto* clears everything. On screen (built-in
browser): the frame with its handles, a real drag, an arrow key (one pixel, no double move), the corner
handles of *Angoli*, two picked pairs that moved and rotated the image with the markers and the panel. Not
tested: a real GeoTIFF with `ModelTransformation`, a `.prj` in a system the projection engine does not know
(it reports an error and centres the image), the undocked panel. Backups:
`geoportale_axpo_pre-immagini-v2_2026-10-06.html`, `README_tools_pre-immagini-v2_2026-10-06.md`.

**Images v2, second round (2026-10-06 afternoon, user: "the window is fixed and bulky in the middle, make it
draggable and open at the side, align it with the rest of the UI; add E, F, H").** The toolbar became a
draggable card docked top-left (§4 *The card*), and the three remaining items of the review were added: *Valori*
(rotation, metres per pixel or scale + dpi, centre), PDF pages through pdf.js, and the world-file export (§4,
§5). Checked in headless Chrome: default position 64/12 px, a simulated header drag moves and saves the
position and the next open keeps it; values round-trip (30°, 0.25 m/px, centre 9.1/45.1 → read back exactly,
100 × 75 m, convex); scale 1:2000 at 300 dpi → 0.169333 m/px; export of an affine placement gives the original
PNG + `.pgw` + `.prj` with the six numbers equal to the placement matrix and C/F at the first pixel centre;
a trapezoid placement gives `_nord.png` + `_nord.pgw` with zero rotation terms and the bbox origin; a two-page
PDF written by hand asks the page (stub answers 2), renders 4096 × 2894 at 350 dpi, prefills the dpi field,
exports PNG + world file, and the project saves `page` and restores the same rendering without asking. On
screen: the card next to the zoom control, a real drag to the lower right. Not tested: a real multi-page PDF
from a CAD printer, very large PDFs (rendering time), Safari.

**Gross and net area buttons (2026-10-06, user request with two placement candidates: the Raccolta's
Particelle module or the search panel).** Placed in the Raccolta — under *Particelle* for the gross area, under
*Disegni* for the net one — because the tick boxes that choose the parcels and the drawings that make the
exclusions live there, and the result lands in the same drawer (§4 *Crea area lorda / Crea area netta*). The
search panel knows nothing of the selection. Checked in headless Chrome with invented parcels: buttons disabled
when empty; three parcels of two municipalities → one gross drawing of 2.62 ha named `Area lorda · Borgo
Inventato +1 · 3 particelle`, orange, `sf_src geoprocess`, listed under Disegni with a toast; two ticked →
the label says "dalle 2 spuntate" and the confirm (OK) replaces the same graphic (1.75 ha); confirm (Annulla)
adds a second; net with no exclusion → message, no drawing; a half-parcel polygon, a line with a 20 m buffer
and a point without buffer → net 0.93 ha out of 1.75, note "2 esclusioni (1 senza buffer, ignorata)"; a second
click replaces in place; an exclusion covering everything → message, the previous net stays; the GeoJSON export
carries `category: gross/net`; with the parcels removed the gross button goes off, the net one stays on. Guide
and tour mention the buttons; both tours still run. Not tested: with the portal (a replaced gross area already
saved, `sf_dirty`), multipolygon unions of parcels far apart (turf union returns a MultiPolygon, converted as
such). Backups: `geoportale_axpo_pre-area-lorda-netta_2026-10-06.html`,
`README_tools_pre-area-lorda-netta_2026-10-06.md`.

**Automatic strips and net area (2026-10-06, late, user: "geoprocess results and imports may be exclusions too;
the net area should be automatic, a layer that appears with the gross one and follows the exclusions; and a
buffer should create its own drawing").** The manual net button became the automation above (§4 *Automatic
strips and automatic net area*), with the cut rules copied from PV Predesign's `areas.js` (the user pointed at
it; `cutDistance`: buffer, or half the width of mitigation/agri lines; categories with `cuts: true`). Checked in
headless Chrome with invented data: the net drawing is born with the gross one and equals it; a power line with a
20 m buffer gives a *Fascia DPA* of 0.44 ha and a smaller net; buffer 40 → the same strip grows and the net
shrinks; a moved line moves its strip; an obstacle point with buffer 15 gives a *setback* strip, a point without
buffer is counted as ignored; a hedge line 6 m wide gives a *mitigation* strip of 0.05 ha (inside other cuts in
the test, so the net did not change) and width 0 removes it; × on a strip refuses with a message; buffer 0
removes the strip; a hidden exclusion still counts; switch off freezes the net through a new exclusion, on
resumes; × on the net turns the switch off and it stays gone, on again recreates it; auto-save keeps `netAuto`,
`sf_uid`/`sf_of`/`sf_sig` and a restore gives one strip and one net without duplicates; removing the gross
drawing removes the net. Guide, tour and the Disegna info window updated; both tours run. Not tested: speed with
a slope exclusion of thousands of vertices (synchronous `geometryEngine`, 200 ms debounce, no simplification),
the portal (a strip or net already saved gets `sf_dirty` on each recompute), PV Predesign reading a project that
now holds strips as exclusion polygons (harmless duplicate of the buffered line, same area). Backups:
`geoportale_axpo_pre-automatismi_2026-10-06.html`, `README_tools_pre-automatismi_2026-10-06.md`.

**Boundary setback (2026-10-06, night, user: "next to the switch add the setback from the gross area, for the
distances from cadastral boundaries").** §4 *Rientro dal confine*. Checked in headless Chrome: 10 m → the net
goes from 0.381 to 0.132 ha and the note says it; 500 m → no net drawing and the message "il rientro di 500 m
consuma tutta l'area lorda"; back to 0 → the previous value; `netSetback` saved and restored with the field.
Backup: `geoportale_axpo_pre-rientro_2026-10-06.html`.

**Legend panel (2026-10-06, night, user: "add the legend of the layers that are on, in the rail between the
layer list and the basemaps; use the ArcGIS SDK, or show me something better").** §4 *Legenda*: the SDK Legend
widget plus a block for the Raccolta, which the widget cannot see. Checked in headless Chrome: the rail order
(Raccolta, Layer, Legenda, Mappa di base, Segnalibri, Modifica, Stampa), icon and tooltip; the drawer opens and
the widget mounts; the Raccolta block lists Particelle, a connection line (line swatch), an exclusion, a
geoprocess result and an import with their colours and counts; an exclusion with the eye off and hidden parcels
leave the block; closed, the panel is not redrawn, reopened it is current; no WMS rows and no images for the
cadastre. On screen (built-in browser): the panel with "Nessuna legenda" above and the Raccolta block below; a
GeoJSON layer with a unique-value renderer added to the map is listed by the widget once the view reaches it,
and leaves when switched off. Not tested: a real web-map swap with the panel open, WMS services that declare a
`legendUrl`. Backups: `geoportale_axpo_pre-legenda_2026-10-06.html`, `README_tools_pre-legenda_2026-10-06.md`.

**Legend, second round (same night, user with a screenshot of the logged-in map: "why are some layers always on
and some missing from the legend?").** Two causes, both fixed (§4 *Legenda*): the WMS images ignored the group
that contains the layer and its scale range, so switched-off groups still showed their WMS legends; and
`hideLayersNotInCurrentView` dropped layers that are on but out of scale. Now every layer that is on is listed,
a status line counts the ones out of scale, the widget's own "Nessuna legenda" is replaced by a line that
accounts for WMS too, and layers that are on but have no legend are named. Checked in headless Chrome: a tile
service with "hide in legend" is named in *Accesi ma senza legenda* and leaves when switched off; inside a
switched-off group it does not count; with a tiny `minScale` the status line says one layer is out of scale.

**PDF from the undocked panel (2026-10-07, user: "many PDFs give 'Invalid PDF binary data…', is it easy to
fix?").** Yes: the PDF bytes are now copied into the main window before pdf.js sees them (§5 *Images v2*, last
trap). Checked in headless Chrome with a File created in an iframe: the old call reproduces the user's message,
the new one renders both pages and the whole import goes through; the image suite still passes.

**GPS locate button (2026-10-06, night, user request).** §4 *GPS locate button*. Checked in headless Chrome: the
`Locate` widget sits in the top-left UI right after the compass; the guide mentions it; both tours run. Not
tested: a real geolocation (headless has none). Backup: `geoportale_axpo_pre-gps_2026-10-06.html`.

**Cadastre gone from every map (2026-10-08, user: "il catasto che aggiungi ad ogni mappa ora non si vede e non
segna nessun errore").** Not the app: the Sardinia GeoServer that bridges the AdE WMS answers every GetMap with a
`ServiceException` ("Internal error PKIX path building failed…": its Java cannot validate the AdE server's TLS
chain; the AdE certificate is a Let's Encrypt one issued 2026-07-22, nothing new in October), HTTP 200
`text/xml`, which the SDK swallows. The AdE WMS cannot be used directly from the browser (no CORS, no
EPSG:3857; the WFS the same, GML only); no other public cascade found (Veneto, Lazio, Basilicata GeoServers
checked). Done: (1) `CATASTO_SOURCES` + `catastoProbe` (§4 *Cadastre source and bridge*); (2) a bridge of our
own, `scripts/catasto_proxy.py` (wired into `serve_https_catasto.py` at `/api/catasto`) and the same in Node for
the Azure site, now in `scripts/ponte_catasto_azure/` (§5 *The cadastre bridge*). Checked:
the bridge with curl (capabilities rewritten, GetMap 3857 → PNG with parcels, 400 on oversize, pass-through);
headless Chrome A) without a bridge → `catastoBad`, marked title, error toast; B) `?catasto=<local bridge>` →
source swapped, `mapUrl` on the bridge, `fetchImage` at Pavia 1:2000 returns 39,851 opaque pixels, no errors.
Not tested: the Azure Function itself (no Node here). **2026-10-09, user:** no GitHub Actions work on the
company repo for now (no control, unfamiliar), so the function stays in `scripts/`, out of the published files. Backup:
`geoportale_axpo_pre-catasto-ponte_2026-10-08.html` (+ README, `serve_https_catasto.py`).

**Zoom only on the last selection (2026-10-09, user request).** §4 *Zoom after a selection*. `render(zoom, fids)`
frames the graphics of the given fids only when `parcelsInView` says they are not all in the view (extents
unioned in WGS84, points checked one by one; a point parcel without geometry never moves the map). The three
callers pass their fids: the area selection its hits, the text search the found fids plus the fids of parcels
already present, the click its hit. Checked in headless Chrome with invented parcels: out of view → flies to
those only (same scale when the second one is elsewhere, no zoom-out to both); in view → no movement; `render()`
without zoom → no movement; a re-selected duplicate out of view → framed; `bootVP` set → no movement;
`render(true)` without fids keeps the old "all visible" behaviour (no caller uses it). Backup:
`geoportale_axpo_pre-zoom-selezione_2026-10-09.html` (+ README).

**Open, known, non-blocking**

1. **The public GitHub repository with the Zornade key** (§6) — needs the owner: make it private,
   regenerate the key.
2. Third-party scripts from three CDNs (five on unpkg) **without Subresource Integrity**, on a page that
   holds a portal OAuth token. Best fixed by hosting the libraries on the future Azure site.
3. **SDK 5.x.** The page is on 4.34, the last AMD release. 5.x on the CDN is ES modules only — no
   `require` — so moving to 5 means rewriting module loading and replacing the widgets with components.
4. The **Legend** widgets created inside LayerList panels are never destroyed on a web map swap.
5. `deleteEnabled: true` on the AREAS editor: the Esri Editor asks its own confirmation before deleting;
   check it when logged in before adding one of ours.
6. **About 240 empty `catch` blocks** (260 with other variable names; counted again on 2026-09-24, the earlier
   ~160 was low). Errors vanish silently — e.g. a Union that fails on one piece skips it without saying so. Many
   are deliberate (localStorage in private browsing, optional bits of the UI); the ones on the critical paths —
   promotion (ISTAT lookup, overlap check), restoring the saved work, re-reading project codes — now write a
   `console.warn`. Rewriting all of them mechanically was ruled out: case by case, on the paths that matter.
7. Not tested logged in: Check Vincoli maps, the real writes to AREAS and to IT - Site Features, the AREAS
   editor, site context, ISTAT zoom; DWG alignment with a real mouse; Print (CORS error from localhost). On 4.34
   in particular the logged-in branch has not been exercised at all. The Site Features flow (site of work by
   code, first save = adds with GlobalID, second save = one update, delete by GlobalID, load skipping objects
   already listed, old categories converted, .axpo with site and codes) was tested on 2026-09-24 with a fake
   `FeatureLayer` recording the calls; the REST forms it relies on (adds and GUID filters) were checked on the
   real service with the API key while copying Sites Notes. Also only with fakes (2026-09-24): the automatic
   link to the site (fake AREAS layer: largest overlap, lines by length, points, outside, code re-read by GUID),
   the section trash deleting from the portal, and **routing on roads** — the API key has no routing privilege
   (403 on `Route_World`), so no real route was ever solved: the travel mode names, the service answer and the
   users' *network analysis* privilege need a first try logged in.

## 8. Repository contents

```
geoportale_axpo.html       the whole application
README.md                  this file
```

There is no build, no test suite and no CI. Verification is done in the browser against the live
portal, which is the only place the corporate layers and OAuth actually exist.
