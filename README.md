# Geoportale · AxpoSolar Italia

A single-file web GIS workbench that sits alongside the corporate **ArcGIS Online** portal
(`sig-urbasolar.maps.arcgis.com`) and adds the one thing AGOL cannot do out of the box:
**search an Italian cadastral parcel by _Comune + Foglio + Particella_ and zoom to it.**

The app is deployed as one file — [`geoportale_axpo.html`](geoportale_axpo.html) (~1 110 KB, SDK 4.34, no bundler, no
`node_modules`), written by `scripts/build.py` from the sources in `src/` (5 html, 14 css and 29 js files, in the order
of `src/manifest.txt`; see §3). Open it over HTTPS, log in with your org account, and it
works. **The map of the app for people who do not read the code** — which file does what, how the data travel, where it
is fragile, how it is tested — is [`docs/architecture.md`](docs/architecture.md).

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

### Build, tokens, icons and tests (since 2026-10-21, phases 0 and 1 of the ribbon refactor)

The app is still deployed as the one file `geoportale_axpo.html`, but it is **developed as sources** in `src/`
(`html/`, `css/`, `js/`, cut along the sections the file already had; `src/manifest.txt` is the order).
Edit the sources, never the built file:

```
py -3 scripts/build.py            writes geoportale_axpo.html from src/ (LF, byte for byte reproducible)
py -3 scripts/build.py --check    exit 1 if the built file is not up to date
py -3 scripts/make_tokens.py      writes src/css/10-tokens.css from design/tokens.json (the family's tokens); --check
py -3 scripts/make_icons.py       writes src/html/05-icons.html, the sprite of the family icons the sources use; --check, --list
py -3 scripts/make_kit.py         writes src/css/12-kit.css and 13-geo-kit.css from design/kit.css and geo-kit.css (verbatim); --check
py -3 tests/run.py                headless checks, Python + Chrome, 26 harnesses, about eight minutes (see tests/README.md)
```

`design/` holds the design system's inputs, copied verbatim from the family (`tokens.json`, `icons/*.svg`, `kit.css`,
`geo-kit.css`): change them there, then run the three generators. Before every pull request: `tests/run.py` green,
`build.py --check`, `make_tokens.py --check`, `make_icons.py --check` and `make_kit.py --check` ok. The plan of the refactor and its
decisions are in `docs/plan-ribbon-refactor.md`.

## 4. What it does

The page is a GIS workbench in Italian, laid out like the other apps of the family (PV Predesign): the **top bar**; six
**tabs**, one per step of the work, with the **ribbon** of the active tab under them; the workspace, with the
**Raccolta** on the left, the map, and the **right panel** (Proprietà · Layer · Legenda); the **status bar**. The work
goes: find the parcels → collect them in the Raccolta → draw the project areas and the constraints → analyse (terrain,
geoprocesses, context) → export, write back to the portal, or save it all as a `.axpo` project that PV Predesign opens.

Signed out, the parcel search, terrain, drawing, geoprocessing, import, export and the project file all work. Signing in
to ArcGIS Online adds the Check Vincoli maps and the portal layers, the context analysis and the road route, the AREAS
site and the writes to AREAS COLLECTION and IT - Site Features. This section follows the order of the screen and of the
*Guida rapida*; how the parts work inside, and their traps, are in §5.

**Names** (Italian as on screen; code names in the last column)

| Name | What it is | Code |
|---|---|---|
| **Barra superiore** | Brand, project name, the **Mappa** selector in the middle; folder, save, Accedi, theme, help on the right | `#topbar` |
| **Progetto** | The `.axpo` name in the top bar; a click saves under another name | `#projNameBtn`, `projSetName`, `projCur` |
| **Mappa** | The portal web map in use: the General Map or a regional Check Vincoli | `#wmBtn`, `#wmMenu`, `wmSelect` |
| **Schede** | Ricerca particelle · Terreno · Disegna · Geoprocessi e analisi · Importa · Salva ed esporta | `#tabs .tab[data-tab=ricerca\|terreno\|disegno\|geoproc\|import\|export]`, `showTab(name)` |
| **Ribbon** | The tools of the active tab, in groups, with a context figure at the right end | `#ribbon`, `SHELL[tab].groups`, `ribbonRender`, `ribbonSync`, `ribbonCtx` |
| **Gruppo** (ribbon) | A set of tools, closed by its name and an ⓘ | `.rgroup[aria-label]`, `.rinfo`, `RIBBON_INFO`, `#rinfoPop` |
| **Strumento** | A ribbon button, large, medium or small; some open a menu or a popover of settings | `.tool`, `toolBtn`, `toolClick`, `#rmenuPop`, `#rformPop` |
| **Raccolta** | The left panel: everything found, drawn, computed or imported, in groups | `#leftPanel`, `#drawerElenco`, `openPanel('elenco')` |
| **Gruppo** (Raccolta) | A container of objects: five standard ones, the user's own, the Immagini block | `.ggrp`, `grpBlock`, `GRP_STD`, `grpOf` |
| **Sottogruppo** | A set of parcels inside Particelle | `.psub`, `pgrpBlock`, `parcelGroups` |
| **Riga** | One object of the Raccolta, with its ⋯ menu | `.grow`, `rowMore`, `rowMenu` |
| **Pannello destro** | Three tabs: Proprietà · Layer · Legenda | `#side`, `#sideTabs`, `spanelSet(name)` |
| **Proprietà** | The module of the active tab; in Disegna also the card of the selected object | `#sideContent`, `.tabpane[data-tab]`, `#selCard` |
| **Modulo** | One block of a tab in Proprietà, one at a time, its title in the panel head | `SHELL[tab].mods`, `modShow(tab, id)`, `details.imod`, `.toolpane[data-tool]` |
| **Striscia** | What a closed panel leaves: its icon, its name, a count | `.strip`, `#openLeft`, `#openRight` |
| **Controlli mappa** | Zoom, compass, GPS, Zoom sulla Raccolta, Luoghi; search lens and measures; basemap chip | `view.ui`, `#ctlExtra`, `#trStack`, `#msStack`, `#bmBtn` |
| **Barra di stato** | 24 px under the workspace: coordinates, scale, cadastre, mode, snapping, autosave | `#statusbar`, `31-measure-coords.js` |
| **Finestra «Di più»** | The long explanation of a group, often with a drawn example | `#infoModal`, `impInfo(key)`, `RC_INFO`, `DRW_INFO`, `GP_INFO`, `IMP_INFO` |
| **Finestra** | A modal: Esporta, Migra, Salva in Site Features, CAD, Le mie categorie, Guida rapida | `#exportModal`, `#promoteModal`, `#sfModal`, `#cadModal`, `#catModal`, `#helpModal` |
| **Barra del telefono** | Up to 768 px: five entries at the foot of the screen | `#mbar`, `MBAR`, `#moreMenu`, `MORE` |
| **Scheda del telefono** | A panel as a sheet over the map on a phone | `.sheet`, `.handle`, `#sheetTools` |
| **Fine** | The phone's Esc, next to the mode banner | `#modeDoneBtn` |

### Top bar

- The Axpo logo (`29-logo.js`) and «Geoportale · AxpoSolar Italia»; then **Progetto**, the `.axpo` name («Senza nome»
  until saved; a click saves under another name), also the browser tab's title; hidden under 1,180 px.
- **Mappa**: a menu with the General Map and the 16 Check Vincoli maps (`CV_MAPS`; Liguria, Molise, Trentino-Alto Adige
  and Valle d'Aosta have none), a search by name, the current map ticked; disabled while signed out. Sign-in loads the
  General Map; Check Vincoli layers arrive switched off. A change of map keeps parcels, drawings, results, DWGs, images,
  services and added portal layers (`loadWebMap` moves them) and asks first only when layers switched on or attribute
  filters would be lost (`wmHasStateToLose`).
- On the right: **Apri progetto** (Ctrl+O), **Salva progetto** (Ctrl+S; Shift+click or Ctrl+Shift+S saves as),
  **Accedi / Esci** (one button, grey or green dot, the user's name in the tooltip; OAuth with the organisation
  account), the light or dark theme (`axpo_theme`), **?** for help.

### Tabs and ribbon

- Six tabs by name only, in the order of the work (arrow keys move between them). The ribbon shows the active tab's
  groups: 28 in all, each closed by its name and an ⓘ (see *Help*).
- A tool presses a panel control, flips a checkbox, shows a module in Proprietà, opens a menu (category, type, source,
  «Per categoria») or a popover of settings that mirrors hidden panel fields (*Supporto al disegno*, *Aree automatiche*,
  *Analizza le pendenze*); some carry an inline number or select (Buffer, Tolleranza, Estensione, Passo). A tool with a
  value shows it in its name («Buffer · 25 m», «Fonte · sopralluogo», «Sito C0000»); a disabled one says why in its
  tooltip. The panel controls the ribbon presses are hidden in the panel (class `rib`).
- At the right end the context figure (`ribbonCtx`): parcels and hectares, hectares over the slope limit, the automatic
  net area or the drawings, the last result, the imports, the project name — one per tab.
- Under 1,500 px the groups narrow; under 1,280 px small tools become icons and the context figure goes; when the groups
  still overflow the ribbon turns `tight` and only tools with a value keep a short text.

### Ricerca particelle

Groups **Catasto · Sulla mappa · Da un disegno · Raccolta**; the cadastral source is the Zornade API (§2).
- **Catasto** › *Cerca particelle*: in Proprietà, region → province → municipality, the sheet (*foglio*) and one or
  more parcels (comma or space), *Aggiungi alla Raccolta*. A chosen region offers its Check Vincoli map as a one-click
  link (`wmSuggest`); it never swaps the map by itself.
- **Sulla mappa**: *Puntuale*, *Rettangolo*, *Poligono*, *Linea*; the mode stays on until its button again or Esc.
  With no mode on, a click on the map opens the layers' popups (since 2026-10-30 the web maps' vector tiles' too).
- **Da un disegno**: a drawing, buffer, result or import chosen in Proprietà, then *Cerca*: the parcels it touches (a
  point finds the parcel under it). The module title counts the usable objects.
- **Raccolta**: *Crea area lorda*, *Etichette sulla mappa* (parcel numbers, hidden below 1:150,000, `axpo_labels`),
  *Zoom sulle particelle*.
- **Area selection is sampled**: Zornade has no spatial query, so a shape becomes points (a 14 × 14 grid over an area,
  at least 12 m apart; every 25 m along a line; at most 180), each sent to `/parcels/locate`, 28 at a time, stopping at
  120 parcels (`SEL_*`, `13-tools-select.js`). Slow on large areas; a parcel smaller than the step can be missed. Esc
  once stops the scan and adds what was found, twice stops that too. Every Zornade call gives up after 20 s; failed
  calls are counted in the final message.
- The map then frames only the parcels of that action. «Aggiungi alla Raccolta» always frames the parcels just added (at least
  about 400 m around them, so a small parcel keeps its context); a selection on the map frames them only if they are not all
  in view; a click never moves it.
  A parcel's area is the geodesic area of its outline; Zornade's own `area_m2` (Web Mercator) is never shown.
- **Crea area lorda** unions the ticked parcels, or all when none is ticked, into a drawing *Area lorda* named
  `Area lorda · <comune> · n particelle`. If one exists, a confirm asks to replace it in place (a saved one keeps its
  GlobalID, so the next save updates it) or to add another. With nothing ticked and parcel subgroups present: one gross
  area per subgroup plus one for the parcels without, each replacing its own earlier one (`sf_pg`).

### Terreno

Groups **Estensione · Modello del terreno · Analisi terreno · Profilo · Terreno**; modules *Pendenze da DTM*
(`17-slope.js`) and *Profilo altimetrico*. The terrain method is PV Predesign's, so both give the same numbers.
- **Estensione**: the extent drawn here (*Rettangolo*, *Poligono*; kept in the work and the project as
  `terrainExtent`), a drawing *Area lorda*, the AREAS site, or the Raccolta's parcels merged. *Salva come area lorda*
  makes it a drawing «Area lorda · estensione»; *Togli l'estensione* removes it.
- **Modello del terreno** › *Analizza le pendenze*: a popover with the slope limit — *in ogni direzione* (10 excludes
  everything steeper than 10 %) or *nord–sud ed est–ovest* (a cell is out when either component exceeds its value) —
  the switch *Esclusione nella Raccolta* and *Analizza* (then *Analizza di nuovo*). Cell (5 m, 2–50) and minimum patch
  (250 m²) are under *Avanzate* in Proprietà.
- Elevations of **Esri World Elevation** (about 10 m in Italy: scouting, not survey) over the extent **plus 25 m**,
  slopes in percent in a local metric frame (§5). No sign-in, no credits.
- **Analisi terreno**: one image at a time — *Classi di pendenza* (0–5–10–15–25 %), *Quote*, *Esposizione* (downhill
  north red, south green, east or west yellow, flat under 2 % grey), *Ombreggiatura*; *Curve di livello* over any of
  them at a *Passo* of 0.5–10 m. Cells over the limit are hatched in red; a new limit redraws at once, with no download.
- With the switch on, one drawing *Esclusione · Pendenza elevata* (`exclusion` / `steep-slope`) named «Pendenza > 10 %»
  follows the limit, clipped to the extent, with `sf_slope` for PV Predesign. The limits are the user's own: the
  design standards stay in PV Predesign, out of this public repository.
- The DTM image sits in the Immagini block (tag *DTM*); **Terreno** › *Rimuovi terreno* removes it after a confirm,
  the exclusion stays. **Profilo** › *Profilo altimetrico*: the Esri ElevationProfile widget along a drawn line.

### Disegna

Groups **Cosa disegni · Con quale strumento · Seleziona e modifica · Supporto al disegno · Aree automatiche · Collega a
sito AREAS**. Proprietà shows the card of the selected object (titled «Proprietà»), the AREAS site row, the current
category with *Cambia…* and *Le mie categorie…*, its attributes and note, the exact-shape box and *Ultimo disegno*.
- **Cosa disegni**: the category as a coloured card with its description; a click opens the gallery, grouped as the
  Raccolta. Beside it only what the category uses (*Tipo*, *Buffer*, *Larghezza*, *Altezza*, *Tensione*), then *Fonte*
  (a tavolino · sopralluogo · ufficiale) and *Esclusione da layer*. The values apply to the next drawings, the category
  is remembered (`axpo_sf_cat`); only the tools of its geometries are enabled; a drawing takes the category's colour.
- **The model** (`SF_CATS`, shared with the AGOL layer and PV Predesign via `../schemas/site_features_model.json`): the
  category is the role — *Area lorda* `gross`, *Area netta* `net`, *Percorso di connessione* `connection`, *Esclusione*
  `exclusion`, *Infrastruttura lineare* `linear`, *Ostacolo* `obstacle`, *Accesso e punto di connessione* `access`,
  *Mitigazione* `mitigation`, *Zona agricola* `agri`, *Nota e riferimento* `reference`; the type is what it is (power
  line, tree…). Old Sites Notes categories are mapped when loaded (`SF_FROM_SN`).
- **Non classificato** (`unclassified`) and **Le mie categorie** (`#catModal`, codes `u…`: name, colour, geometries,
  Raccolta group, «esclude dall'area netta», up to 12 fields of text, number, choice, date or yes/no, values in
  `sf_ext`) are app-only: in the project and the exports, not on the portal unless the save window gives them a model
  category. *Non classificato* never cuts the net area; deleting a custom category turns its objects into it.
- **Con quale strumento**: *Punto · Linea · Area · Rettangolo · Cerchio · A mano libera*; double click ends lines and
  areas, the tool stays armed, Esc stops. With no tool on, a click on a drawing takes it in hand (drag, rotate, scale;
  a second click edits vertices; Delete removes it).
- **Seleziona e modifica**: the selection **is** the Raccolta's checkbox, shared with «Spuntati» of the geoprocesses
  and the export window; blue outline; one selected object shows its card in Proprietà. *Seleziona* (Shift+click adds),
  *Riquadro*, *Rimodella*, *Sposta e ruota*, *Dividi* (a drawn line cuts the selected areas into «nome · 1», «nome · 2»…,
  new objects for the portal), *Unisci* (attributes of the first), *Copia*, *Elimina* (asks; for saved objects also
  about the portal), *Annulla · Ripeti* (Ctrl+Z / Ctrl+Y): one action is one Annulla, strips and net area included;
  Annulla also gives back parcels, groups, subgroups, own categories, the net-area settings, the terrain extent and the
  site; its tooltip and message say what («Annulla: Unisci 2 aree»); what is on the portal stays known (an object
  brought back becomes «da aggiornare», one taken away is said to stay on the portal); 40 commands, restarting when a
  project opens or the work is emptied. Editable: drawings, results, imports — not parcels, automatic strips and net area, or owners' parcels.
- **Esclusione da layer**: a click on a feature of a visible vector layer makes a drawing of it (area → *Esclusione*,
  line → *Infrastruttura lineare*, point → *Ostacolo*; source *ufficiale*; never twice); with *Riquadro*, every feature
  the rectangle touches in the visible feature layers (at most 200, a confirm above 20).
- **Supporto al disegno** (a popover; its name says what is on): *aggancio* to vertices and edges of drawings and
  parcels and, optionally, of DWGs and visible feature, GeoJSON, WFS and CSV layers (Ctrl breaks free); *misure e
  angoli* (Tab while drawing opens *Deflezione* and *Distanza*, Enter locks, a click places; angles relative to the
  previous side or as azimuth, `axpo_drw_dir`); *misure esatte* of rectangle (width × height, azimuth, *Da un lato*) and
  circle (radius), placed by their centre, exact to the centimetre; *griglia* in metres (10 m, `axpo_grid_step`, snap,
  *Allinea a un lato*, *Raddrizza*; not saved in the project).
- **Aree automatiche** (a popover; its name says «Area netta automatica / congelata · rientro n m»), PV Predesign's
  rules (`areas.js`): *exclusion*, *linear*, *obstacle*, *mitigation*, *agri* (`SF_CUTS`) and user categories so marked
  **cut**, by their buffer, or half the width for hedges and agricultural lines.
  - **Fasce**: an object with a cut distance gets a polygon that follows it — «Fascia 20 m · nome» (*Esclusione · Fascia
    di rispetto*, *Fascia DPA* for power lines) or «Fascia larga 6 m · nome»; it goes with its object or a distance of 0.
  - **Area netta**: with a gross drawing and the switch on, «Area netta · x ha» = the gross union shrunk by the **rientro
    dal confine** (metres), minus every cutting polygon (hidden ones too); its note says what was subtracted. Off =
    frozen; deleting its row switches it off. Strips and net area carry an «auto» chip and follow their sources.
- **Collega a sito AREAS** (optional, needs sign-in): on save each object without a site takes the AREAS COLLECTION
  area it overlaps most; outside every area, the chosen site if any. A chosen site is forced on new drawings. *Dalla
  mappa* (a click inside the area) or a code such as C0000; *Carica elementi* brings the site's saved features into the
  Raccolta; *Togli il sito*. The tool reads «Collega a sito AREAS» or «Sito C0000»; the site is saved (`site`).

### Geoprocessi e analisi

Groups **Oggetti di partenza · Geoprocessi · Contesto · Coordinate**; Proprietà shows the three steps (A, operation,
parameters and *Esegui*) or the module of a tool. Distances and tolerances are true metres (local metric frame, §5).
- **Oggetti di partenza** fills slot **A**: *Dalla mappa* (click again on a spot to step through overlapping objects),
  *Spuntati*, *Per categoria*. A click on a feature of a map vector layer (feature, REST, WFS, GeoJSON, map-service
  sublayers; not WMS) copies it once into the Raccolta with its full geometry and attributes (at most 80). Slot **B**,
  in Proprietà, serves *Parte comune* and *Sottrai B da A*.
- **Geoprocessi**: *Buffer* (metres, up to 100 km; one per object or one merged), *Unisci*, *Parte comune*, *Sottrai B
  da A*, *Copia*, *Contorno*, *Semplifica* (*Tolleranza*, 2 m by default, at least 0.5), *Ritaglia* (draw the area to
  keep after *Esegui*). A card states its condition and stays disabled, with the reason, until A fits. The status line
  ends with the measure and *Vedi nella Raccolta ›*, or the error in plain words. Results go to the group *Analisi*.
  Above 5,000 vertices the buffer simplifies first (said in the message and the name); the others warn above 20,000.
- **Copia**: pick an object (drawing, result, import, parcel or layer feature), click to place; each click another copy,
  Esc ends. *Parallela* copies at a distance (a line → a parallel line; an area → inset if you click inside, outset
  outside; mitred corners); *Solo il lato cliccato* copies one side. A copy keeps category, type, attributes, note and
  site but is a new object for the portal; a parcel or layer feature becomes a drawing of the current category.
- **Contesto** (needs sign-in): *Analisi di contesto* — click a point (or right-click › *Analisi di contesto qui*): the
  nearest substations within 80 km (380, 220, 150, 132 kV and new primary substations), the nearest HV line within
  60 km, PV projects within 15 km with capacity and status, from the portal's *elemens* layers; no credits.
  *Percorso su strada*: start, Shift+click via points, end; the organisation's route service (fallback Esri
  `Route_World`), a simple route of **0.005 credits**; *Strade e sentieri* (Walking Distance) or *Solo strade carrabili*
  (Rural Driving Distance, `axpo_rte_mode`); the result is a drawing *Percorso di connessione · Stimato*, «Su strada
  3,4 km».
- **Coordinate** › *Coordinate sconosciute*: paste X Y pairs (decimal comma, separators, DMS, a leading label);
  *Trova il sistema* keeps the readings among `SRS_IT` (23 systems, eight company ones) that land in Italy, X/Y swapped
  included; *Dove dovrebbe stare* plus a click ranks them by error in metres (ED50 vs WGS 84 in one zone: ~200 m). The
  chosen reading goes to the Raccolta as points or one area. Cassini-Soldner coordinates cannot be resolved.

### Importa

Groups **File · CAD · Immagini · Servizi web · Portale**; the four modules stay open in Proprietà, each with its «i».
- **File**: GeoJSON, KML, KMZ, Shapefile (.zip), into the group *Altro*. A file without a CRS is converted by itself
  only when Web Mercator is its only reading on Italian soil; otherwise *Coordinate sconosciute* opens with the
  candidates and the whole file is reprojected with the system chosen.
- **The owners file** of the separate *Estrazione intestatari* tool (recognised by `sister_stato`): parcels named
  `Fg 12 · 101 · <comune>`, coloured by set of co-owners, at the end of Particelle under *Per intestatario* (one group
  per set, largest area first, *Senza intestatari* last; *Nomi sulla mappa*; the bin on that line removes the whole
  file, Annulla brings it back); a row's «i» or a click on the map shows
  owners with tax code and share, status, land use, area, notes. **Personal data**: kept in the page and the `.axpo`,
  left out of the browser autosave, never sent to the portal, never written to the console.
- **CAD** › *Disegno DWG*: read in the browser (LibreDWG in WebAssembly, ~9.5 MB on the first DWG); the file never
  leaves the PC. `.dwg` only, up to 150 MB, cut at 250,000 elements; layers and colours kept. The dialog (`#cadModal`)
  proposes the CRS estimated among `SRS_IT` and shows where it lands, the unit (m, cm, mm), and drops elements more
  than 50 km from the median. A drawing in local coordinates is placed with *Allinea il disegno* (two point pairs,
  scale 1:1 by default), remembered per file (`cad_align`). Texts show below 1:5,000, at most 1,500. The list in the
  module switches and recolours layers, zooms, re-aligns, removes. Not in the autosave; the `.axpo` keeps the file.
- **Immagini**: a GeoTIFF, or an image with its world file and `.prj` selected together, places itself (no `.prj`: the
  CRS is estimated and the message says so). PNG, JPG and PDF (the chosen page rendered at 4,096 px) are placed with
  *Posiziona*, a draggable card (`#imgToolbar`, `axpo_img_card`): *Sposta / scala / ruota*, *Angoli*, *Punti* (2 pairs
  similarity, 3 affine, 4+ least squares or projective; residuals in metres), *Valori* (rotation, m/px or «1 : N at
  dpi», centre), *Ripristina*, *World file* (a zip with world file and `.prj` in Web Mercator). Limits: 30 MB per image,
  100 MB per GeoTIFF, 4,096 px on the long side. Into the Immagini block; not in the autosave, kept in the `.axpo`.
- **Servizi web**: paste a WMS, WFS or ArcGIS REST address (type guessed), *Leggi il servizio*, tick layers (filter,
  *Tutti*, *Nessuno*), *Aggiungi alla mappa*; nothing is created on the portal. Only https services that allow
  cross-origin reads work (http is tried as https); the national PCN WMS does not. A WFS the SDK refuses comes as a
  GeoJSON snapshot of up to 20,000 features. Kept by the autosave and the project.
- **Portale** › *Layer dal portale*: the Layer tab's search of portal items by title (needs sign-in).

### Salva ed esporta

Groups **Esporta · Portale AGOL · Progetto · Stampa**; the default module is *Progetto* (name, last save or opening,
what the work holds, *Salva progetto*, *Salva come…*, *Apri progetto…*).
- **Esporta** opens the export window, also on a format (*GeoJSON · KML · KMZ*, *Shapefile (.zip)*, *Tabella XLSX*). It
  lists Particelle with its subgroups, the owners' parcels, then the Raccolta's groups; its ticks are the Raccolta's;
  nothing ticked = everything; XLSX is the parcel table. Features carry `gruppo` and `sottogruppo`; the KML has a folder
  per group and subgroup; the shapefile zip a folder per group, one file per geometry type (`_punti`, `_linee`,
  `_aree`). **File name** (`exBaseName`): municipality, date, parcels — `Borgo_Inventato_YYYY-MM-DD_6part.geojson`;
  several municipalities add `_e_altri_n`; no parcels gives `export_YYYY-MM-DD`.
- **Portale AGOL** (needs sign-in; each write confirmed in a window):
  - *Migra particelle in AREAS*: the Raccolta's parcels become features of **AREAS COLLECTION** (layer 426), one each or
    one dissolved area (*Dissolvi*), with area class, area type, source, *Estrazione massiva*, project type. REGIONE and
    PROVINCIA are the parcel's own, COMUNE and PRO_COM_T from ISTAT, the area geodesic, `ID_univoco` automatic.
  - *Salva disegni in Site Features*: drawings, results and imports go to **IT - Site Features** with category, type,
    attributes, name, note, source, status and site (`AREA_GUID`, `PROJECT_CODE`); the first save adds, the next update
    (`sf_gid`, no duplicates). *Solo gli spuntati*; objects of an app-only category stay out unless the window gives
    their geometry a model category; owners' parcels are skipped. A cloud icon marks saved rows (orange: changed since).
  - *Modifica AREAS COLLECTION*: the Esri Editor in Proprietà, on the General Map only (computer only).
- **Stampa**: the Esri Print widget in Proprietà; the cadastre image does not print; computer only.

### The Raccolta

The left panel, *Raccolta*, with a count badge (parcels and objects); one scroll.
- **Toolbar**: *Filtra per nome* (accent-insensitive, «n trovati», Esc clears), *+ Gruppo*, *Comprimi / Espandi*.
- **Groups**: *Particelle*, *Aree di progetto*, *Aree di esclusione*, *Analisi*, *Altro*, then the *Immagini* block. An
  object sits in the group it was put in, or by default (`GRP_BY_CAT`): gross, net → Aree di progetto; exclusion,
  obstacle, linear, mitigation, agri → Aree di esclusione; buffers and results → Analisi; the rest → Altro; a custom
  category → its group. Standard groups show when populated, personal ones always. Groups are containers only: the
  category stays the role, and the portal never sees them.
- **Group header**: tick all, name, count, total area, eye, names on the map, ⋯ (*Zoom sul gruppo*, *Spunta tutti*, *Togli le spunte*,
  *Sposta qui i selezionati*, *Elimina tutti gli oggetti*; personal groups also *Rinomina*, *Sposta su / giù*, *Elimina
  il gruppo*). Open or collapsed is remembered (`axpo_grp_open`).
- **Particelle**: ⋯ with *Zoom sulle particelle* and *Nuovo sottogruppo*; «x,xx ha catastali»; the **subgroups** (one level, same
  tools, *Zoom sul sottogruppo* first in their ⋯); the
  parcels without one; the owners' parcels last.
- **Rows** of one line: checkbox (the selection), colour (picker: *Recenti*, `axpo_colori_recenti`; *Categorie*;
  *Mappa*, Okabe–Ito; *Tenui*; *Altro colore…*), name (details in the tooltip), «auto», the figure on the right
  (hectares with a comma, `fmtHa2`; metres; «punto»), portal state, eye, ⋯. The menu, also on right-click: *Rinomina*,
  *Zoom*, *Proprietà: categoria, tipo e attributi* (`sfEditInProps`), *Sposta in: ‹gruppo›*, *Elimina* (asks about the
  portal for saved objects; *Annulla* restores), only the items the kind of object allows (`RULES`, 7b-1): an
  automatic strip has *Zoom* and *Sposta in*, the automatic net area *Zoom*, *Sposta in* and *Elimina*; their colour
  is the automation's. Past 30 rows: *Mostra altre n* (`RC_LIMIT`).
- At the foot, while a drawing exists, the net-area switch and the setback; *Immagini* lists images and DTMs with eye,
  opacity, reposition and ×.

### The right panel

Tabs **Proprietà · Layer · Legenda** (`spanelSet`); the head carries the module's title, *Sgancia* and close. A tab
change or a ribbon tool with a module brings Proprietà back; at its foot, *Impostazioni avanzate* (see *Project*).
- **Layer**: *Aggiungi layer dal portale* (search by title, `ADDABLE` item types, sign-in); *Layer in mappa*, the Esri
  LayerList with a search by name, drag-and-drop order and, per layer, transparency, popup on/off and an **attribute
  filter** like QGIS *Filter…* (field, condition, suggested value, `+ E` / `+ O`, editable SQL, live count; the icon
  turns into a funnel while on). Added layers have ⋯ *Zoom al layer* / *Rimuovi dalla mappa*. The vector tile layers whose tiles carry attributes open a
  popup too (§5.7). A red card (`#svcStatus`)
  with *Riprova ora* shows while the cadastre does not answer.
- **Catasto AdE**: the Agenzia delle Entrate WMS drawn as one image over the map (EPSG:4258, `adeOv*`): sheets with
  their numbers from 1:25,000, parcels and buildings below 1:5,000, parcel numbers below 1:2,000; its Layer entry is the
  switch and the opacity (50 % at start). The web maps' own national cadastre (`382d9c13…`, `05ccc51f…`) is removed in
  the browser, the items untouched. If the Agenzia does not answer, `catastoProbe` tries the bridges (`api/catasto`;
  the Regione Sardegna GeoServer) at start and every 10 minutes; `?catasto=<url>` puts one first, `?catasto=off` drops it.
- **Legenda**: the Esri Legend of the layers on (out of scale included and counted), the WMS legend images, a line
  *Accesi ma senza legenda*, and the *Raccolta* block with the real colours and counts of what is on.
- **Both panels** close to a strip (the Raccolta's with the count), resize from their inner edge (`.pgrip`: 260–560 px,
  `axpo_w_left` / `axpo_w_right`; double click, Home or Enter back to the default; arrows 16 px, 48 with Shift; the map
  keeps at least 360 px, `MAP_MIN`) and **undock** into a window for a second screen (`rcUndock`, `sideUndock`): a strip
  in the page brings it forward or docks it back, closing it docks it, its position is remembered
  (`axpo_raccolta_finestra`, `axpo_pannello_finestra`); wider than 640 px it lays modules out in columns. From 769 to
  1,279 px only one panel is open at a time.

### On the map

- **Top right**: the search lens, the Esri Search widget (Esc, a click outside or a result closes it): addresses, or
  `lat, lon` (decimal comma, DMS) or a projected `X Y` whose system it guesses; a coordinate pair wins over the geocoder.
  Under it the **measures**: *Misura lineare*, *Misura area*, *Pulisci*, the Esri Measurement widget in a small card
  (`#msCard`); Esc and other map modes close it; measures are not saved.
- **Top left**: zoom, compass (the map rotates; a click puts north up), GPS (Esri Locate at 1:2,500; needs https),
  **Zoom sulla Raccolta**, **Luoghi** (Esri Bookmarks that also restore layer visibility, `cat_bookmarks`).
- **Bottom right** the basemap chip: *Topografica*, *Satellite*, *Ortofoto AGEA* 2024, 2023 and 2022, *Stradale*, with
  an opacity slider (`axpo_bm_opacity`). **Bottom centre** the scale bar at 70 %; the Esri attribution is a thin line.
- **Right-click**: *Aggiungi particella qui*, *Analisi di contesto qui*, *Buffer qui*, *Copia coordinate*, *Apri in Google
  Maps*, *Centra qui*.
- **Messages** sit bottom left on the map, with an action where useful («Vedi nella Raccolta ›»). A map mode shows a
  small banner at the top (`#modeBanner`); one mode at a time, and a tool opened mid-sketch completes the sketch when it
  has enough vertices.
- **Keys**: Esc leaves any mode, stops an area selection, cancels an edit, closes popovers; Ctrl+Z, Ctrl+Y (or
  Ctrl+Maiusc+Z), Ctrl+S, Ctrl+Maiusc+S, Ctrl+O, shown in the tooltips of their ribbon buttons; Ctrl+K the search; Tab
  while drawing for typed lengths; Delete on the drawing in hand.

### Status bar

The **pin** (freezes the coordinates at the next click); the pointer's coordinates in the system of its menu (`#sbCrs`,
`axpo_sb_crs`: WGS 84 decimal or DMS, UTM 32N/33N, Monte Mario 3003/3004, RDN2008 6707/6708); the **scale**, a menu of
the usual scales (1:500 to 1:250.000; the current one on top when it is not one of them; a choice moves the map there,
exactly: the view does not snap to the zoom levels of the tiles); with a terrain model (after *Analizza le pendenze*)
the **elevation** and the **slope** under the pointer, the slope as the limit reads it (one figure, or N–S and E–O),
red over the limit, «—» outside the model; the cadastre (checked at start and every 10 minutes, red when down); on the
right the mode, the *Aggancio* switch and «Salvato nel browser · 14:32», red «Non salvato: spazio del browser pieno»
while the browser refuses the work. Tablets hide the mode and the autosave note (not the red one); a phone hides the
bar.

### The `.axpo` project

- **Salva progetto** writes the whole work to one `.axpo` file, like a QGIS `.qgz`: a zip with `progetto.json`, the
  original files (`file/dwg-<n>-<name>`, `file/img-<n>-<name>`) and the terrain grids (`terreno/<id>.f32`). It holds the
  view, basemap, work area, the portal web map with every layer's visibility, transparency, popup, filter and order,
  the added portal layers and services, parcels and subgroups, drawings, results, imports, groups, custom categories,
  the AREAS site, the net-area settings, the terrain extent, labels, DWGs and images with their placement, the DTMs with
  their limit. Bookmarks, theme and the Zornade key stay out; no token is ever written.
- **Apri progetto** replaces the work after a confirm and rebuilds it all. Ctrl+S saves to the same file (written in
  place in Chrome and Edge, downloaded elsewhere). Opened signed out, the local part returns and the web map waits for
  the sign-in.
- **PV Predesign** opens the same file: areas, exclusions (the steep-slope one with `sf_slope` = `{limit: {maxAny} |
  {maxNS, maxEW}, cell, min_patch, buffer, source, dtm, grid, area, date}`), the site, and `terrain`, one record per DTM
  on the map in Predesign's own grid format (`{id, lon0, lat0, x0, y0, cell, nx, ny, zmin, zmax, missing, loadedAt,
  source, name}` plus `buffer`, `area_name`, `area_geom`, `limit`, `min_patch`, `view`, `file`; elevations Float32
  little-endian, row by row from the south-west cell, NaN where unknown). It ignores groups and custom categories.
- **The format is written** in `docs/axpo-format.md` (phase 9): every field, its default, who writes and who reads it,
  the rules of the version. **Salva una copia da condividere…** (the Progetto module) saves the same project without
  personal data: the owners' parcels stay, without names or the owners' file record, and the user's name is left out;
  the open project does not change.
- **The browser autosave** (`localStorage` `cat_state`) keeps the work across reloads except DWGs, images, DTMs and the
  owners' personal data; at about 5 MB the browser is full and a message says so. *Impostazioni avanzate*: a personal
  Zornade key, what is saved, *Svuota il lavoro* (preferences kept), *Ripristina tutto* (every key, then reload).

### Help

- **?** opens a menu: tour *Inizia qui* (8 steps: sign-in, Catasto, Raccolta, Disegna, right panel, Salva ed esporta,
  project, done), tour *Tutti gli strumenti* (15 steps: the tabs, one per tab with its ribbon, Raccolta, right panel,
  resize and undock, search and measures, zoom and Luoghi, status bar, Mappa, done) — `TOUR_START`, `TOUR_ALL` in
  `32-tour.js` — and the *Guida rapida* (`#helpModal`, 14 items in the order of this section). *Inizia qui* starts on the
  first visit from a computer (`axpo_tour_seen`).
- **Profilo: Base · Avanzato** (7b-2), at the foot of the «?» menu and of the guide: Base shows only the tools of the
  main flow, Avanzato (the default) all of them. Remembered in this browser (`axpo_profilo`); a link with
  `?profilo=base` opens in Base (to send to a project manager). In Base the «?» tooltip says «profilo Base», a tour
  skips a step on a hidden group, and its last step says where the other tools are. What Base hides: in Disegna
  *Supporto al disegno*, *Collega a sito AREAS* and Rimodella, Sposta e ruota, Dividi, Unisci, Copia; in Geoprocessi e
  analisi *Oggetti di partenza*, *Geoprocessi* and *Coordinate* (the tab opens on *Analisi di contesto*); in Importa
  *CAD*, *Immagini*, *Servizi web*; in Salva ed esporta *Migra particelle in AREAS* and *Modifica AREAS COLLECTION*;
  and the card *Collega a sito AREAS COLLECTION* of the Disegna panel (`PROF_ADV_PANELS`).
- **Cerca oggetti e comandi** (7c): Ctrl K, or the item of the «?» menu, opens a box at the top centre of the map
  (the kit's `.search`, as PV Predesign). As one types it lists the objects of the Raccolta — drawings, results,
  imports, strips and net area, the parcels by comune, sheet and number, the owners' parcels by sheet and number (the
  owners' names are not searched) — with colour, category and group, the figure on the right and «nascosto» for a
  hidden one; then the commands of the ribbon that the profile shows («Comando · tab › group», the shortcut). Every
  word must appear, case and accents ignored; 8 results (up to 3 commands) and «e altri n»; arrows, Enter, Esc. An
  object: its tick (the selection), the zoom, its row shown and flashed in the Raccolta (block opened, «Mostra
  altre» and the filter undone); a parcel: the zoom and its row; a command: as its button, or the reason it is off.
  Not while a map mode is on (the mode banner), not on a phone.
- **ⓘ** of each ribbon group (`RIBBON_INFO`): what it is for, its tools by their button names, where the result goes.
  **«Di più ›»** opens the long window (`impInfo(key)`), the same as the «i» of a module: 19 windows, most with a drawn
  example — `rcFind`, `rcMap`, `rcDraw`; `drwDraw`, `drwSel`, `drwAid`, `drwCopy`, `drwSite`, `ctx`; `gpA`, `gpBuf`,
  `gpShape`, `gpClip`, `gpAB`; `slope`, `sr`, `cad`, `img`, `svc`. The `texts` harness checks every group has its ⓘ
  naming all its tools.

### The phone

- Up to 768 px the tabs, ribbon, strips and status bar hide; a bar at the foot has *Cerca · Terreno · Disegna ·
  Raccolta* (with the count) *· Altro* (Geoprocessi e analisi, Importa, Salva ed esporta, Layer, Legenda, Mappa di base,
  Luoghi, Guida rapida). Stampa, Modifica AREAS COLLECTION and the tours stay on the computer; **?** opens the guide,
  with the profile at its foot.
- The panels are sheets over the map, one at a time: 31 % of the screen at rest, 84 % with the handle, × to close. A
  tab's sheet shows its ribbon tools first, group by group, then the module. A map mode drops the sheet to rest and
  **Fine** works as Esc. The map keeps the zoom column, the search and the basemap chip; messages show at the top.
- Tablets (769–1,000 px) keep the coordinates and the cadastre in the status bar, with a tighter ribbon.

## 5. Architecture notes

How the app works inside, by subject, with the traps to know before editing. The reasons behind each choice are in
`docs/plan-ribbon-refactor.md` and §7; the old notes, one per phase, are kept word for word in
`docs/archive/readme-2026-10-26.md`.

### 1. Sources and build

- `scripts/build.py` writes the deployed `geoportale_axpo.html` from `src/` following `src/manifest.txt`: a path
  under `src/` is copied, `= text` emits that line, `=` an empty line; blank lines and `#` comments are ignored. The
  output is reproducible byte for byte, with LF endings, and goes through a `.tmp` file that then replaces the target.
- Order: `html/00-head.html`; all CSS in one `<style>` (tokens, base, kit, geo-kit, icon placement, then the app's
  rules by area); the body (`05-icons.html`, top bar, app body, modals); four classic `<script>` blocks: `09-icons.js`
  to `29-logo.js` in one, then `30-ui-glue.js`, `31-measure-coords.js` and `32-tour.js` in one each.
- `design/` holds the family's inputs verbatim (`tokens.json`, `icons/*.svg`, `kit.css`, `geo-kit.css`); change them
  there, then run the generators. `make_tokens.py` → `src/css/10-tokens.css` (light values on `:root`, dark values that
  differ on `[data-theme="dark"]`, `{ref}` → `var(--ref)`, old names kept as aliases: `--acc`, `--on-accent`,
  `--font-ui`, `--font-head`, `--line-strong`). `make_icons.py` → `src/html/05-icons.html`, a sprite of only the icons
  the sources name (it scans `src/html` and `src/js` for `data-icon="…"`, `href="#i-…"`, `icon('…')` and
  `icons-used: a, b` comments; `--list` says where each is used). `make_kit.py` → `src/css/12-kit.css` and `13-…`.
- Icons: markup says `data-icon="name"` (drawn once by `iconsRender()`) or carries `<use href="#i-name">`; scripts call
  `icon(name, cls)` (`sm` 14 px, `xs` 12 px). No emoji or glyphs in labels; CSS carets are a `mask` of the chevron
  (`--chevron-mask`). Colours by role: filled buttons and active states `--fill` with `--on-fill`, accent texts
  `--link`, hover and selection `--accent`, focus rings `--focus`; an outcome is a coloured dot plus normal text (green
  or yellow text on white fails AA). The text floor is 11 px. Before every pull request: `tests/run.py` green, and
  `build.py`, `make_tokens.py`, `make_icons.py`, `make_kit.py` all pass with `--check`.
- **Trap:** edit `src/`, never the built file: the next build overwrites it, and `build.py --check` fails meanwhile.
- **Trap:** an icon named through a variable (`TOOLS`, `SF_ICON`, the theme toggle, the phone bar) is invisible to the
  scan: list it in an `icons-used:` comment, or it renders blank while `--check` stays green. A name missing from
  `design/icons/` stops `make_icons.py` («icona sconosciuta»).
- **Trap:** the kit comes first and the app must not restyle a kit class (placement is the app's). Its global rules
  matter: `button` has no border, background or padding; `[hidden]` always hides; `body` 13/19 px, `small` 11 px,
  `.icon` 18 px. Kit names the app must not reuse: `.grp`, `.sheet`, `.menu` (a grid capped at 420 px), `.sec`.
- **Trap:** the blocks share one global scope; a top-level `function` is a property of `window`, so assigning
  `window.gpSelectOp = …` replaced the function, which then called itself forever: expose under another name
  (`gpOpenOp`). `let`/`const` are shared across blocks but are not on `window`.
- **Trap:** a syntax error kills its whole block (in the UI glue: no ribbon, one cryptic console line). A `//` comment
  in the middle of a compact line silences the rest of it (it happened twice in the ribbon configuration); the test
  runner parses every block with Node first.
- **Trap:** a Python patch sent through a shell heredoc loses a backslash (write an escaped JS quote with `chr(92)`),
  and a script that opens a file for writing and then fails leaves it empty: write to a temporary file and rename.

### 2. Runtime: SDK, loader, sign-in

- ArcGIS Maps SDK for JS 4.34 from the CDN, the last 4.x with the AMD loader (`require`); 5.0 serves ES modules only
  and turns most widgets into `<arcgis-…>` components. `11-map-model.js` loads the modules in one `require` and keeps
  the shared ones in `esriMods`; the event `axpo-view` announces the view.
- The UMD libraries (`xlsx`, `jszip`, `shp-write`, `togeojson`, `shpjs`, `geotiff`) load inside the
  `window.define = undefined` block of `00-head.html`, or they register as anonymous AMD modules instead of globals.
  pdf.js and the DWG reader are ES modules loaded with `import()` on first use.
- OAuth uses the **implicit** flow: the registered app has a confidential secret, and the authorization-code exchange
  fails silently in a browser. The page signs in by itself unless the URL has `?noauto`; signed out, `onSignedOut`
  prints the exact redirect URI to register (console; sign-in tooltip on localhost).
- Still deprecated, left for 5.x because it means rewriting: the widgets (BasemapGallery, LayerList, Legend,
  Bookmarks, Search, Print, ScaleBar, Measurement, Editor, ElevationProfile) and the `projection` and `geometryEngine`
  modules. New code uses `watchProp` (`reactiveUtils.watch`) and `centroidOf` (`centroidOperator`).
- **Trap:** `credential-create` fires once per authenticated service (about twenty for the General Map): `onSignedIn`
  is guarded (`signInBusy`, `signInFailed`) and says «connected» only once the portal has loaded.
- **Trap:** on 4.34 `view.viewpoint = <JSON>` and `new Bookmark({viewpoint: <JSON>})` do not autocast (no move, no
  position): use `Viewpoint.fromJSON` (`vpFromJSON`). Saved symbols are Esri JSON, which `Graphic` does not autocast
  either (they came back black): use `symbolJsonUtils.fromJSON`.
- **Trap:** Calcite widgets (LayerList, Bookmarks, Editor, ElevationProfile) follow the `calcite-mode-dark`/`-light`
  class on `<html>`, not Esri's dark CSS; `applyTheme` sets both (`axpo_theme`). Widget roots carry their `esri-*`
  class on the container itself, so `#box .esri-…` may need both forms.
- **Trap:** a `ListItemPanel` needs `className: 'esri-icon-*'` (`icon:` gives an invisible button); `bookmark-select`
  does not fire on a click in the list, so the app wraps `viewModel.goTo`; a synthetic `click()` on a
  `calcite-list-item` selects nothing (check Calcite with a real mouse).
- **Trap:** `queryFeatures` without `outSpatialReference` returns the layer's own SR, even in a Web Mercator view.
  `map.allLayers` includes the basemap and the ground: filter them with `baseGroundSet()`.
- **Trap:** a widget whose panel may sit in another window gets its container as an element (`byId`), not an id
  string: the SDK looks ids up in the page only, and the BasemapGallery does not render at all with one.
- **Trap:** a watch on `view.map.<property>` stays on the old `Map` after a web map swap: watch through the view
  (`reactiveUtils.watch(() => view.map.basemap, …)`).

### 3. Shell: tabs, ribbon, popovers

- Rows: top bar · `#tabs` (`ricerca`, `terreno`, `disegno`, `geoproc`, `import`, `export`) · `#ribbon` · `#appBody`
  (`#leftPanel`, `#mapWrap`, `#side`) · `#statusbar`. `showTab(name)` activates the tab and its `.tabpane` in `#side`,
  renders the ribbon (the sheet tools on a phone) and shows the tab's current module.
- `SHELL` (`30-ui-glue.js`) describes each tab: `mods` (id, title, line, icon, and `roots`, the selectors of the pane
  elements that belong to the module) and `groups` (`label`; `more`, the key of its «Di più» window; `tools`; optional
  `when()`, `end`, `cls`). `modShow(tab, id)` shows one module (the others get `.mod-off`), opens a `details`, titles
  the panel head and brings «Proprietà» back; `modShowFor(el)` opens the module that holds an element.
- A tool is `{id, t, i, size: large|medium|small}` plus an action — `press` (clicks a control and mirrors its `.active`,
  `aria-pressed`, `aria-checked`, `disabled`, `title`), `toggle`, `mod`, `fn`, `focus` (a setting, not a module
  switcher), `menu()` + `pick(v)`, `pop()` (a form), `num` (an inline number: `unit`, `min`, `step`), `sel` (a mirrored
  `select.rsel`; `stacked`), `card` (the category card: `desc`, `tip`) — and its state and layout: `off()` (why it is
  disabled; `offOf` borrows another control's), `on()`, `label()`, `icon()`, `color()` (`--c`), `show()`, `stack`
  (same key, same `.tool-stack`), `after`, `primary`, `wide`.
- `ribbonRender(tab)` draws large tools in a row, small ones in stacks of three, medium ones two per column, then each
  group's name and ⓘ. `ribbonSync()` re-reads every control; `ribbonSoon()` runs it on a 0 ms timeout (not a frame: a
  background tab must not delay the states), from a MutationObserver on `#side`, `#leftPanel` and `#signin`, from every
  `change` and from the app. When groups overflow, `#ribbon.tight` hides the context figure (`ribbonCtx`) and turns
  small tools without a value (no `val`) into icons; under 1280 px small and medium tools are icons.
- Panel controls the ribbon holds carry `rib`, hidden at every width. Two hidden docks keep controls the ribbon mirrors
  or presses: `#drwDock` (snapping, angles, exact shapes, grid) and `#rcDock` (Esporta, Migra, Salva).
- Popovers, one at a time, closed by Esc (capture listeners), a click outside or a resize: `#rinfoPop` (the ⓘ from
  `RIBBON_INFO`; «Di più ›» calls `impInfo(more)`), `#rmenuPop` (ribbon menus and the Raccolta's row menu; `{sec}` and
  `{h}` items make a `gallery`), `#rformPop`. A form (`aidForm`, `netForm`, `slpForm`) is `{title, html, sync, foot}`:
  `formCopy` fills each `data-mirror="#id"` field from the hidden control (value, options, checked, disabled),
  `formWrite` writes a change back and dispatches `input` and `change`, so the panel's handlers do the work; a
  `data-press` button presses a panel button and closes.
- Texts: `RIBBON_INFO` (per group: purpose, tools by button name, where the result goes), the «Come funziona» windows
  (`RC_INFO`, `DRW_INFO`, `GP_INFO`, `IMP_INFO` via `impInfo(key)`; `drwGrid` aliases `drwAid`), the *Guida rapida*
  (`#helpModal`), the two tours of `32-tour.js` (a step targets `#ribbon .rgroup[aria-label="…"]`; the first visit on a
  computer starts «Inizia qui», `axpo_tour_seen`). The `texts` harness forbids the words of the old layouts.
- Modules are `details.imod` with an «i» (`.im-info`), open state in `axpo_sezioni_aperte`; the *Importa* cards are
  `details.imod.always`. `#sideContent` is a size container: from 640 px modules go into columns and the steps of
  Geoprocessi and Disegna become a grid (`.paneBody.gpflow`, `.drwflow`), docked or undocked.
- **Trap:** the ribbon enables tools on a timeout: a `.click()` right after a state change finds them still disabled.
- **Trap:** the rule `label:not(.chk):not(.opt)` of `20-shell.css` makes every label a block in capitals: scope
  overrides to their container (`#rformPop label.rf-sw`, `#exList .exrow`). The global `select,input{width:100%}`
  stretches checkboxes.
- **Trap:** `#rformPop` is a kit `.menu` (a grid capped at 420 px), so it is `display:block` with a 620 px cap. The
  wide-panel card grid needs `.paneBody .gpcards`, or the later base rule wins.
- **Trap:** «i» buttons are wired by `.im-info`, not `.imod .im-info`: those outside modules would be mute.
- **Trap:** the hidden `#drwDock` keeps old module titles (the `texts` harness skips it). A control moved into a
  popover must stay reachable from it, like «Da un lato» (`#shpRotPick`) through a `data-press` button of `aidForm`.

### 4. Panels and the undocked windows

- `#leftPanel` holds only the Raccolta. `#side` has three tabs (`#sideTabs`): «Proprietà» (`#sideContent`, the module of
  the ribbon tab), «Layer» (`#sideLayers`), «Legenda» (`#sideLegend`), switched by `spanelSet(name)`
  (`#side[data-spanel]`). `openPanel(name)` is the router: `elenco` (or its window), `layers`, `legend`,
  `basemaps`/`bookmarks` (map popovers), `edit`/`print` (modules of «Salva ed esporta»), `null` (closes the left
  panel). Esc closes no panel.
- Widths 320 px left (300 under 1500 px), 300 px right (280 under 1400 px). Both resize by their inner edge (`.pgrip`, a
  `role="separator"`) within 260–560 px, kept in `axpo_w_left`/`axpo_w_right`; double click, Home or Enter reset; the
  arrows move 16 px (48 with Shift). `pwMax` keeps the map at least `MAP_MIN` (360 px) wide; `pwApply` re-applies on
  resize and drops the widths on a phone. Under 1280 px (`isNarrow`) one panel at a time; wider, `ensureMapRoom` closes
  the panel not just opened when the map gets narrower than 360 px. A closed panel leaves a strip.
- **Undock** (`sideUndock`/`sideDock`, `rcUndock`/`rcDock`): the node is **moved, not copied** — `window.open` from the
  click, a written skeleton, the page's stylesheets and icon sprite imported (a `<use>` looks the sprite up in its own
  document), then `adoptNode`. Handlers, state and script stay the page's: nothing to synchronise. A strip takes its
  place (`#sideDockStrip`, `#rcDockStrip`); closing or reloading the window docks back (`pagehide`, `beforeunload`,
  and a watchdog on `win.closed`, from the kept node `sideNode`/`rcNode`); closing the page closes both; the window's
  geometry is kept (`axpo_pannello_finestra`, `axpo_raccolta_finestra`); the theme follows by MutationObserver; with
  several screens the right panel's window offers the *window management* permission once. What it asks of the code:
  - id lookups go through `byId` (`$`, `$g`: the page, then both windows), queries in the Raccolta through `rcQ`/`rcQA`,
    and the ribbon's lookups of panel controls (`press`, `toggle`, `offOf`, `num`, `sel`, `data-mirror`, `data-press`,
    `focus`, the modules' `roots`) through `qAny(sel)`: the page, then both windows;
  - global listeners (Esc, mode-ending clicks, Ctrl+Z/Y, Ctrl+S/O) use `docOn`, which attaches them to open and future
    windows (`window.__docL`);
  - `confirm`/`alert`/`prompt` are wrapped to open where the focus is; the «i» windows, the CAD dialog, the project
    pickers and the toasts open there too (`sideActiveDoc`, `sideActiveWin`, `sideModalHere`), back on docking;
  - popovers open in their opener's window (`popHome`) and come back before a window closes (`undockPopsHome`).
- **Trap:** `document.querySelector` and `getElementById` do not see an undocked panel: a new lookup of a panel control
  goes through `byId`, `qAny` or `rcQ`, never the page alone (until the fix of 2026-10-26 the ribbon's presses, mirrors
  and module switches did nothing with a panel undocked). The `panelswap` harness presses the ribbon with both
  panels in their windows.
- **Trap:** a popup closed with its × may fire only `beforeunload` (no `pagehide`), and by then `win.closed` is true, so
  `sideDoc()`/`rcDoc()` give `null`: the dock functions take the window's document or the kept node, or the panel is
  lost until a reload.
- **Trap:** the tour docks the right panel first (it points at page elements); the Raccolta stays where it is.
- **Trap:** a file picked in an undocked window comes from another realm: `instanceof ArrayBuffer` fails (pdf.js refused
  such PDFs; a `Uint8Array` over the foreign buffer hung its worker). Copy it first:
  `new Uint8Array(await file.arrayBuffer()).slice()`.
- **Trap:** the desktop app's built-in browser blocks popups: try the undock in Chrome or with the `undock` harness.

### 5. Messages, map modes and Esc

- `msg(text, cls, opt)` (`28-export-window.js`) shows one toast in `#toasts`, bottom-left on the map (top-left on a
  phone). Errors stay until their × or the next message, others go after 6 s (10 s with an action); `opt.action` adds
  a button, such as «Vedi nella Raccolta ›» (`vediElenco`, only while the Raccolta is closed). Text via `textContent`.
  A failing service is not repeated in a toast: «Vedi ›» opens the Layer tab, where `#svcStatus` (`catastoStatus`)
  keeps the reason and «Riprova ora»; the status bar's `#sbSvc` turns red.
- **One map mode at a time**: parcel selection (`selMode`), drawing (`drawType`), context (`subActive`), the CAD modes
  of `drwMode` (shape, side, copy, parallel copy, route), grid placement, geoprocess pick (`gpPick`), Seleziona,
  Esclusione da layer, terrain extent, DWG alignment, «Dove dovrebbe stare», site pick, measure, profile, the pin.
  `cadModesOff(keepDraw)` (`27-align-sr.js`) is the one place that ends them all; the three button families
  (selection, drawing, context) end by pressing their own button again, so each resets its state; the UI glue catches
  clicks on those buttons (capture phase) to end the other families.
- Map-picking modes pause `updateOnGraphicClick` (`sketchClickPause`/`sketchClickResume`; otherwise the first click on
  a drawing starts editing it and ends the mode) and save and restore `view.popupEnabled` and the cursor. Another tool
  taking the map completes a sketch with enough vertices (2 for a line, 3 for an area: `sketchFinishIfPossible`).
- `#modeBanner` (`role="status"`) is rewritten only when its text changes, from a 350 ms poll (`modeInfo`). The Esc
  chain (end of the UI glue) closes the first of: a dialog; the site pick; an area selection in progress (the first Esc
  stops the scan and adds what was found, the second stops that too); the DWG alignment; the terrain extent; a selection
  or exclusion box; Seleziona and Esclusione; an active SDK update (cancelled: the SDK sees Esc only when the map has
  the focus); the geoprocess pick; `drwMode`; the grid placement; «Dove dovrebbe stare»; the measure; the profile; the
  tool module of «Geoprocessi e analisi» (only while that tab is in front); the three families. Menus, popovers and
  the search take Esc first (capture listeners).
- **Trap:** a new mode goes into `cadModesOff`, `modeInfo` and the Esc chain, and pauses `updateOnGraphicClick`.
- **Trap:** clicking an armed tool button cancels its sketch before `cadModesOff` runs, so `modeOffDraw` also completes
  an interrupted sketch.
- **Trap:** during an SDK update `view.popupEnabled` is off (restored on cancel or complete): a `popupTemplate` alone
  never shows on a graphic being edited, and a mode that starts then must capture the value before the first `update()`
  and restore it itself.

### 6. Phone and tablets

- Up to 768 px (`isPhone`): the bar `#mbar` (`MBAR`) and the menu «Altro» (`#moreMenu`, `MORE`; Stampa and Modifica
  AREAS COLLECTION stay on the computer and say so). `mbarApply` gives `#side` and `#leftPanel` the class `sheet` (31 %
  of the screen, 84 % with `full`, from the `.handle`): no node moves, so the Esri widgets stay. A tab's sheet starts
  with `#sheetTools` (its groups through `sheetToolsRender` and the ribbon's `toolBtn`, `toolClick`, `ribbonSync`).
- A map mode drops the open sheet to rest (an observer on the banner); «Fine» (`#modeDoneBtn`) dispatches an Escape
  keydown. Tabs, ribbon, strips, scale and status bars, undock, grips and theme toggle hide; the «?» opens the guide.
- 769–1279 px: one panel at a time; up to 1000 px the status bar keeps only coordinates and cadastre. A rotation across
  768 px re-renders the sheet tools and the panel head.

### 7. Map, layers and legend

- The page starts with a plain `Map` (basemap «Topografica», ground Esri World Elevation; layers: cadastre, `gl`
  parcels, `labelsGL`, `toolsGL` for the Raccolta's drawings, results and imports, `ownLblGL`); sign-in loads the
  General Map. The selector `#wmBtn` (`wm*`, `18-parcels.js`) lists it and the Check Vincoli maps of `CV_MAPS` (16
  regions; none for Liguria, Molise, Trentino-Alto Adige, Valle d'Aosta). A region chosen in the search only offers its
  map (`wmSuggest`). A confirmation is asked only when switched-on layers of a Check Vincoli map, or attribute filters,
  would be lost (`wmHasStateToLose`).
- `loadWebMap(id, hideLayers)` loads the item and, for a Check Vincoli map, switches every operational layer off (they
  load when switched on; no `loadAll`, which would read the regional WMS eagerly and hit CORS); removes the map's own
  national cadastre and puts the app's on top; moves from the old map the DWG groups (`cadReattach`), images, portal
  layers, web services and image handles; adds `gl`, `labelsGL`, `toolsGL`, `ownLblGL` last; sets `view.map`; calls
  `setupAreasEditor`. Requests are numbered (`wmReq`): a late answer to an older request is dropped.
- **One cadastre**: `findWebmapCatasto` finds a web map's national cadastre by item id (`CATASTO_ITEM_ID`, `…_ALT`; a
  title fallback only with both «AdE» and «catast»: about twenty regional cadastres have similar names and stay). It
  leaves the map in the browser only; its id goes into `wmCatGone`, so a project naming it reports nothing missing.
- **Layer list**: `LayerList` with `layerListItem`, drag-and-drop and a name filter; an item's panel (legend, opacity,
  popup, labels, attribute filter) is built on first opening; the icon becomes a funnel while filtered. Layers the app
  added (`llMine`) get «Zoom al layer» and «Rimuovi dalla mappa».
- **Vector tile popups** (2026-10-30, `vtp*` at the end of `16-layers-webmap.js`, wired by `vtpWire` in
  `11-map-model.js`): on a click with `view.popupEnabled` (no mode armed), no measure, no sketch and no object of the
  Raccolta under the pointer, `vtpFeatures` hit-tests the `vector-tile` layers of `view.map.allLayers` that are not the
  basemap's and whose layer view is not suspended (scale, visibility). Per result: the source layer
  (`getStyleLayer(origin.layerId)['source-layer']`), skipped when it ends in `/label` or the attributes have no key
  without «_»; one element per layer, source layer and visible attributes (fill and outline come back twice). The popup
  comes from `popup.json` among the item's resources (`vtpConfig`: a promise cached by item id, `esri/request` with the
  session's token; `{version, layers:{<source-layer>:{title, fields:[{name, label}]}}}`) or falls back to the source
  layer's name and every field. The native popup of the same click (its `location` within 4 px) is waited for
  (`waitingForResult`, 5 s at most) and keeps its features first. No highlight: the tiles carry no shape.
- **Trap:** tiles made in ArcGIS Pro carry no attributes, so they open nothing; `popup.json` is written by whoever
  publishes the tiles (for Campania `../scripts/campania_vincoli_04_vector_tile.py`). A `VectorTileLayer` has no
  `popupEnabled`, so the Layer panel offers no switch for it.
- **Attribute filter** (like QGIS *Filtro…*) for layers with `definitionExpression` and `queryFeatures` (not WMS):
  `llApply` tries the expression with a count first, so a wrong SQL keeps the previous filter. «contiene» and «inizia
  con» use `UPPER(field) LIKE`; a dd/mm/yyyy date covers the day. Filters go into projects (`def`, `sdef`) only.
- **Legenda** (`21-legend-auto.js`, only while its tab is open, 150 ms after a change: `legendSoon`): the SDK `Legend`
  (mounted on first opening; `respectLayerVisibility`; `hideLayersNotInCurrentView: false`, so a layer on but out of
  scale stays and `#legendMsg` counts them); WMS legends as GetLegendGraphic images of the visible sublayers with a
  `legendUrl` (`legendWmsRender`, honouring parent groups and scale: `legendAncestorsOn`, `legendScaleOk`); «Accesi ma
  senza legenda» (`legendMissingRender`); the Raccolta block (`legendLocalRender`, real colours and counts).
- **Luoghi** (`#bkPop`): Esri bookmarks plus the layers' visibility (`_layers`, captured when a bookmark is made,
  applied when it is chosen), in `cat_bookmarks` with a backup by name.
- **Controls**: zoom, compass, GPS (`Locate`, 1:2,500, needs https), then `#ctlExtra` at index 3 of `view.ui`; top
  right the lens `#trStack` (the kit's `.geosearch` around the Esri Search) and `#msBtn` (its menu drives the
  Measurement widget's `activeTool`; card `#msCard`; `measureUi` and a watch on `activeTool` keep them in step); bottom
  right the basemap chip (`#bmPop`: BasemapGallery, AGEA orthophotos as `ImageryTileLayer` basemaps `agea<year>` in
  `BM_BY_ID`, an opacity slider for every base layer, `axpo_bm_opacity`). The right-click menu (`wireContextMenu`,
  `#ctxMenu`) acts on the clicked point (add a parcel, context, «Buffer qui», copy coordinates, Google Maps, centre).
- **Status bar** (`31-measure-coords.js`): the pin (arm, then freeze the next click); the pointer in the system of
  `#sbCrs` (WGS 84 decimal or DMS, UTM 32/33, Monte Mario 3003/3004, RDN2008 6707/6708; `projection` loads on the
  first projected choice; `axpo_sb_crs`); scale; cadastre; mode; snapping (one state with `#drwSnap`); autosave.
  Phase 8: `ground(p)` reads `slpAt(p)` (`17-slope.js`: the cell of `slpCur` under the point, in the grid's local frame;
  null without a model or outside the extent plus 25 m) and shows `#sbZIt` / `#sbSlopeIt` only while a model is in
  memory; `#sbScaleSel` (`SCALES`, `scSync` on the view's scale, `scTxt` writes 1:5.000 with the dot, which the Italian
  locale leaves out under 10.000); the view has `constraints.snapToZoom:false` (`11-map-model.js`), as PV Predesign, or
  a chosen scale would land on the nearest tile level (1:5.000 became 1:4.514); `window.sbSaveSet(ok)`, called by
  `saveState`, writes the time or the red state; `window.sbStatusAt(p)` shows the bar for a point (the tests).
- **Trap:** the zoom is free between the tile levels: code that needs a whole level rounds `view.zoom` itself.
- **Search**: a custom source answers only when the text is *just* a pair of numbers (`lat, lon` with decimal comma
  or DMS, or a projected `X Y`, whose system `srSearch` estimates); anything with letters goes to the geocoder.
- **Trap:** with «All» the Search widget takes the first result in source order and puts `defaultSources` before
  `sources`, so the geocoder won every time: the coordinate source goes first through `includeDefaultSources` as a
  function. Its result must carry a real `Extent` (a plain object with `{target, zoom}` drew the marker, never moved).
- **Trap:** the selector's centred wrapper has a `transform`, hence a stacking context: `z-index:1500` only while open
  (`.wmwrap.open`), or the menu is painted under the map; always on, it would cover the dialogs.
- **Trap:** a `GraphicsLayer` cannot label itself: parcel labels, owners' names and the Raccolta's names are text
  graphics in layers of their own, hidden below 1:150,000.
- **Trap:** a layer the app adds to `view.map` is lost at the next web map swap unless `wmCarry` (called by `loadWebMap`)
  moves it, or its draw function re-adds it when missing (as `selRender` and `gpDrawSlots` do). `wmCarry` moves the
  images, portal layers, services, image handles, the Raccolta's layers, the contours (`slpCtGL`), the terrain extent
  (`slpExtGL`) and the names of the Raccolta (`geomLblGL`); a new on-demand layer goes into its list.

### 8. Cadastre and parcel search

- **Zornade** (`12-zornade.js`): `z(path, params)` calls the v2 API with `x-api-key` (the shared key, or the override
  `zrn_tok`) and gives up after 20 s. The cascade uses `/admin/*`; zooming to the chosen area reads the portal's ISTAT
  layer (signed in only), matching names client-side (`istatNorm`); for Liguria Zornade gives `-` for Genova
  (`ZB_PROV_FIX`).
- Parcels are plain objects in `parcels[]` (GeoJSON 4326 geometry, `comune`, `foglio`, `particella`, `regione`,
  `provincia`, `belfiore`, `fid`, `area_m2`, `sel`, `hidden`, `color`, `sub`), drawn by `render()` on `gl` (`gpParcelOf`
  maps a graphic back). Zornade has no spatial query, so `selectByGeometry` samples the shape — about 14 × 14 points in
  an area (step ≥ 12 m), one every ≥ 25 m on a line, at most 180 points, `/parcels/locate` 28 at a time, at most 120
  parcels — then reads each parcel. Parcels smaller than the step can be missed; failed calls are reported, never
  shown as «0 particelle».
- **Zoom rule**: an action frames only its own parcels. `render(true, fids)` (a selection on the map) moves only if they are
  not all in view (`parcelsInView`); `render('frame', fids)` (the search, «Aggiungi alla Raccolta») always frames them
  through `zoomGraphics`, which also serves «Zoom sul gruppo», «Zoom sul sottogruppo» and «Zoom sulle particelle»
  (`zoomParcelList`): the union of the extents with a margin of 1.4 and a floor of about 400 m (0.004° or 400 m of Web
  Mercator) when the objects share one system, a lone point at zoom 17, otherwise the SDK's `goTo` on the graphics. A
  viewpoint restored at start wins (`bootVP`).
- **Trap:** in headless Chrome an animated `goTo` may wait seconds for its target before it moves: the `zoom` harness checks
  the new zooms on the target they ask for, not on the motion.
- **Trap:** Zornade's `area_m2` is computed in Web Mercator (inflated by about 1/cos²φ, ×1.85 at Rome): `trueAreaM2`
  recomputes it geodesically. Never show Zornade's figure as an area.
- **The cadastre image** (`15-catasto.js`, `adeOv*`): the Agenzia delle Entrate WMS serves EPSG:4258/6706 and ETRS89
  UTM, never 3857, without CORS, so the SDK cannot draw it, but an `<img>` can. `CATASTO_SOURCES[0]` is
  `{kind:'overlay'}`: `catastoMake` gives a `GraphicsLayer` without graphics, the Layer panel entry and the switch
  (the image follows its `visible` and `opacity`; `minScale` 1:25,000). `adeOvBind` puts `#adeOverlay` before
  `.esri-ui` in the view container (over the canvas, under the widgets, `pointer-events:none`). `adeOvRefresh`, 250 ms
  after the view stops, asks a WMS 1.1.1 GetMap with `SRS=EPSG:4258` for the view's extent (× device pixel ratio up
  to 2, capped at the Agenzia's 2,048 px) with `CP.CadastralZoning` (sheets, served up to 1:200,000),
  `CP.CadastralParcel` and `fabbricati` (below 1:5,000), `codice_plla` (numbers, below 1:2,000): out-of-scale layers
  come back blank, so one list serves every scale. `adeOvPlace` sets the image from `view.toScreen` of three corners
  (translate, rotate, size), exact between refreshes; the old image stays until the new one loads; late answers are
  dropped (`adeOv.pending`). Within one image lon/lat and Web Mercator differ by less than a pixel.
- **Fallbacks** (`kind:'wms'`): the app's bridge `api/catasto`, then Regione Sardegna's GeoServer (a cascade of the same
  WMS); `?catasto=<url>` puts a bridge first (`axpo_catasto_url`; `?catasto=off` removes it). `catastoProbe` pings each
  source with a 32 px GetMap 1.5 s after start and every 10 minutes (an image = alive; XML or a network error = dead,
  in the service's words) and swaps in the first alive (`catastoSwap`, keeping visibility and opacity; the project key
  `app:catasto` stays). Three images in a row that do not decode call the probe.
- The bridge (in the AGOL Axpo folder, outside this repository: `scripts/catasto_proxy.py`, a Node version in `scripts/ponte_catasto_azure/`,
  served locally by `serve_https_catasto.py`) turns the SDK's EPSG:3857 GetMap into a WMS 1.1.1 `SRS=EPSG:4258` request
  and returns the PNG with `Access-Control-Allow-Origin: *` (cached 10 minutes); it rewrites the capabilities to itself
  with `EPSG:3857`/`102100`, reaches only the Agenzia and refuses sizes over 2,048 px. The Azure Function is ready but
  not installed, by the owner's choice: do not propose it again unless asked.
- The cadastre starts at 50 % opacity with the popup off: no source answers GetFeatureInfo from a browser.
- **Trap:** the overlay is not an Esri layer: it does not print, and it draws nothing above 1:25,000.
- **Trap:** `.esri-ui` is not a direct child of the view container: insert next to it (`ui.parentNode.insertBefore`).
- **Trap:** Esri tiled basemaps force wkid 102100, which GeoServer refuses: a WMS layer needs
  `spatialReferences: [3857]` (bridges, the Check Vincoli regional WMS, services by URL). Behind a bridge `fixMapUrl`
  pins `mapUrl` to it after load, since the SDK takes the GetMap URL from the capabilities.
- **Trap:** a failing GeoServer may answer HTTP 200 `text/xml`, which the SDK swallows (no event, no console line): only
  a probe notices. The Sardinia server has failed this way (a TLS chain its Java could not validate), and the portal
  item of the national cadastre points at it too.

### 9. The Raccolta and its groups

- `#drawerElenco` is one scroll: the toolbar `#rcTool` (filter, «+ Gruppo», «Comprimi»), the static **Particelle** block
  (`render()`: subgroups, parcels without one, then the owners' parcels in `#listOwn`), the groups (`#grpList`,
  `renderGeoms` → `grpBlock`), the net-area foot `#grpFoot` (`#netAuto`, `#netSetback`), the **Immagini** block
  (`data-sec="images"`). `#rcEmpty` shows when empty; the count `#rcBadge` repeats on the strip and the phone bar.
- **Trap:** a block shows only when populated (`secShow`), so its count must cover all it holds: Particelle counts the
  searched parcels and the owners' parcels (`secParcelsN`). Until 2026-10-28 it counted the first only, and an
  owners' file imported with no parcel searched was in the badge but nowhere in the list.
- **Groups** (`22a-groups.js`) are containers: the category stays the role, and the portal never sees them. Standard
  groups (`GRP_STD`: `parcels`, `project`, `exclusion`, `analysis`, `other`) show when populated; custom ones (`grpNew`,
  `grpRename`, `grpMove`, `grpAssign`; `grpDelete` sends its objects back to their default) always. An object's group
  is `grp` if set, else `grpDefault`: by category (`GRP_BY_CAT`, or a custom category's `group`), then by origin
  (buffers and results → analysis, everything else → other). Assigning the default group removes `grp`, so old and
  new files read alike. Open state in `axpo_grp_open`; names on the map per group (`LBL`, `axpo_labels_sec`,
  `renderGeomLabels` → `geomLblGL`); rows sorted by category, with a `.gcat` line when categories mix.
- **Parcel subgroups** (`22b-parcel-groups.js`): one level; a parcel has `sub`; the list is `parcelGroups` (`id`,
  `name`, `order`, `labels`); a subgroup's labels switch off on their own (`renderLabels` skips them); its menus are
  `pgrpMoveActs` and `pgrpHeadActs`.
- **Rows** (`geomRow`; parcels `rowOf`), one line: checkbox, colour (`colPopOpen`: *Recenti* in `axpo_colori_recenti`,
  *Categorie*, *Mappa* Okabe–Ito, *Tenui*, *Altro colore…*), name, «auto», the figure (`rcFig`, hectares with a comma
  via `fmtHa2`), the portal state (orange cloud = changed since), then eye and ⋯ on hover or focus (always on touch and
  on hidden objects). Details are the name's tooltip (`.gsub` hidden). Row menu (⋯ or right-click: `rowMore`,
  `rowMenu`): Rinomina in place (`rowRename`), Zoom, «Proprietà…», «Sposta in…», Elimina (presses the hidden `.gdelx`).
- Past `RC_LIMIT` (30) rows a block offers «Mostra altre n» (`rcListsPost`; `rcExpanded`, for the session). The filter
  ignores accents (`rcNorm`), hides blocks without a match and opens collapsed ones while it filters.
- **The selection is the tick**: `_sel` on graphics, `sel` on parcels, shared by «Seleziona e modifica», the rows,
  «Spuntati» of the geoprocesses, the export window and «Solo gli spuntati» of the portal save.
- **The object editor** lives in «Proprietà»: `sfEditInProps(g)` selects, opens Disegna and shows `#selCard`
  (`selCardRender` + `sfEditor`: category among those that accept the geometry, type, numbers, custom fields, note,
  source, status, site, portal state). The name follows the category label until renamed; the card is not rebuilt
  while the focus is inside it.
- Delete: a strip refuses; deleting the net area turns its automation off; a saved object asks, when signed in, whether
  to delete it on the portal too (`sfDeleteRemote`; a whole group `sfDeleteRemoteMany`, one `applyEdits` per layer).
  Annulla restores the list, not the portal.
- **Trap:** `.grp` is the kit's toolbar, so a group block is `.ggrp`; `.gsub` is the row subtitle, so a subgroup block
  is `.psub` (narrow: `.psubN` stacks the name above the figures).
- **Trap:** the automatic net area makes «Aree di progetto» a mixed group: switch it off before testing category lines.
- **Trap:** the section helpers (`sectionGraphics`, `setSectionSel`, `sectionDelete`, `wireSectionTools`…) take a group
  id and still accept the origin keys `draw`, `geo`, `import`, `images`. `wireSectionTools` runs after every redraw and
  also refreshes the names on the map and the terrain's list of gross areas.

### 10. Drawing, selection and the automatic areas

- **Model**: `SF_CATS` (`11-map-model.js`) is the Site Features model shared with the layer «IT - Site Features» and
  PV Predesign (`../schemas/site_features_model.json`, Predesign's `app/js/categories.js`; change all three). A category
  is a role (`gross`, `net`, `connection`, `exclusion`, `linear`, `obstacle`, `access`, `mitigation`, `agri`,
  `reference`) with label, colour, geometries `g`, numbers `a` (`buffer`, `height`, `width`, `voltage`; `SF_ATTR` maps
  each to attribute, field and label), types `t`, hint `h`. `sfAttrKeys` gives voltage only to power lines (`SF_VOLT`)
  and width only to lines.
- `unclassified` («Non classificato», `app:true`): every geometry, no type or number, never cuts, never goes to the
  portal. `sfOf` reads no code, an unknown code or an unknown old label as `unclassified` (the attribute untouched); old
  Sites Notes labels map through `SF_FROM_SN` (`sfUpgrade`); `sfNewAttrs` falls back to it when the category refuses the
  geometry, so no object is ever without a category.
- **Custom categories** (`19b-custom-cats.js`): `customCats` entries are shaped like `SF_CATS` (code `u…`) plus
  `app:true`, `custom:true`, `cuts`, `group`, up to 12 `fields` (`key`, `label`, `type` text/number/choice/date/bool,
  `options`, `unit`); values in `sf_ext`. `sfCat` looks in `SF_CATS`, then in `customCats`, so the `sf*` helpers work
  unchanged; `catAll()` feeds every list. Deleting one (`#catModal`) makes its objects «Non classificato» (keeping
  `sf_ext`); `catSetState` drops malformed entries with a console warning.
- **Drawing**: the SDK `SketchViewModel` (`sketchVM`) on `toolsGL`, with snapping, tooltips and typed values (Tab opens
  deflection and distance, Enter locks, a click places, a second Enter completes without that vertex). `sketchPurpose`
  tells the `create` handler what a sketch is for (`drawing`, `selection`, `geoproc` clip, `geoproc-split`,
  `terrain-extent`, `select-box`, `excl-box`); it routes the result and re-arms (`sketchArm`; freehand is a polygon in
  `mode:'freehand'`). A click on our object starts an SDK update; the `update` handler calls `histFreeze` at `start`
  and at `complete` recomputes `area_ha` geodesically and sets `sf_dirty` on saved objects. The «Ultimo disegno» box
  (`drwLastRender`) follows `drwLastG`, the last drawing made by any tool.
- **History: one command, one Annulla** (`23-draw-cad.js`, `29b-commands.js`). The SDK's undo works only inside the
  active sketch (it handles Ctrl+Z while the map has the focus). For finished work, 40 photos (`drwHist`) of the work
  the user edits: the Raccolta's objects by reference (`HIST_KINDS` = drawings, buffers, imports: graphic, geometry, a
  copy of the attributes, visibility, symbol), a copy of each parcel without its tick, and the groups, subgroups, own
  categories, net-area settings (`st`), the site AREAS and the terrain extent. A photo is taken **when the work
  settles**: `saveStateDebounced` calls `histSoon`, which records 500 ms after the last change (`HIST_SETTLE`), after the
  automations (200 ms), so that a drawing and its strip are one entry. Each entry carries the name of its command:
  `histName(label)` at the entry point, the first since the last photo wins; `29b-commands.js` wraps the entry points by
  name (`drwAddGraphic`, `selMerge`, `gpExec`, `grpNew`, `catUpsert`, `slpAdd`…), the inline ones call it themselves;
  a command without a name is «Modifica». `histSame` compares graphics by reference and by a signature of their
  attributes (`histSig`, without the tick, the portal identity and derived values: `HIST_SKIP`), parcels by reference
  and signature, the small state as JSON: an unchanged work refreshes the current photo instead of adding one.
  `histApply` puts a photo back while the automations are held (`histAuto`, `applying`) and keeps, for an object that is
  on the portal now, its identity (`HIST_PORTAL`: `sf_gid`, `sf_saved`, `sf_dirty`, `sf_site`), marking it `sf_dirty`
  when its shape or values go back; it counts the objects it takes away that are on the portal (`histSaid` names them).
  Right after Annulla or Ripeti, a photo without a new name replaces the current one, so the automations never drop
  Ripeti. `histFreeze` clones a geometry being edited; `drwHist.busy` still holds the recording inside a multi-step
  tool. Reset after the start-up restore, «Svuota il lavoro» and a project open. Ctrl+Z/Y skip text fields.
- **Trap:** a new action that changes the work needs a name: wrap its function in `29b-commands.js` (called by its name,
  not through a reference stored before the wrap) or call `histName` at its start; otherwise its Annulla reads
  «Modifica». A new part of the work state goes into `histSnap` and `histApply` too, or Annulla will not give it back.
- **The registry of the commands** (`29b-commands.js`, 7a-2): `cmdList()` builds, once, one entry per tool of the
  ribbon from its description `SHELL` (exposed as `window.SHELL` by `30-ui-glue.js`): id (the tool's id; `tab:id` if
  two tabs share one), tab, group, name, icon, tooltip, kind (action, menu, form, value, switch) and the shortcuts.
  The shortcuts are data (`CMD_KEYS`); one `docOn('keydown')` handler (`cmdForKey`) serves them all — Ctrl+Z and
  Ctrl+Y stay with a text field and with a drawing in progress on the map (the SDK's last vertex). `cmdRun(id)` runs a
  command as its ribbon button does (shows the tab, presses the button; `false` if missing or disabled); the commands
  with a key run their function (`CMD_RUN`). `toolBtn` and `ribbonSync` add the shortcut to the tooltip
  (`cmdKeyText`). 7b and 7c read `cmdList()`.
- **Trap:** a new shortcut goes into `CMD_KEYS` (and `CMD_RUN` if it is not a ribbon tool), not into a handler of its
  own: two handlers on the same key would both run.
- **What can be done to what** (`22c-rules.js`, 7b-1): `objKind(o)` names the kind of a parcel (or of its graphic on
  the map), of an owners' parcel and of every object of the Raccolta: `parcel`, `owner`, `drawing`, `result` (buffers
  and geoprocess results), `import`, `strip` (the automatic strip of a buffer or a width), `net` (the automatic net
  area). `RULES[kind][action]` is absent (allowed) or the sentence the user reads when it is not; `can(o, action)`
  returns `true` or that sentence, `canOk` a boolean. The actions read today: `rename`, `colour`, `props`, `edit`,
  `split`, `merge`, `move`, `delete`, `zoom`, `sfSave`, `promote`. Read by the row menu and its colour swatch and ×
  (`geomRow`), `sfEditInProps`, `selEditable` and `selClick` (a click on an object that cannot be selected says why in
  the Disegno message and keeps the selection), Dividi of the geoprocesses. The export, the save to IT - Site
  Features (`sfCandidates`) and the migration to AREAS COLLECTION keep their own filters, which the `rules` harness
  checks against the table. A copy (`drwCopyAdd`) is always an ordinary drawing: it drops the automatic link
  (`sf_auto`, `sf_of`), the portal identity (`sf_uid`, `sf_sig`, `sf_gid`), the owners' data (`own`, `src_attrs`),
  the layer key (`_srcKey`) and the subgroup gross area (`sf_pg`).
- **Trap:** a new action on the objects asks the table (`can(o, 'action')`) and shows its sentence, instead of a test of
  its own on `sf_auto` or `own`; a new kind of object gets a line in `objKind` and in `RULES`, and a case in the
  `rules` harness. A new field that ties an object to something (an automation, the portal, a person) goes into the
  list that `drwCopyAdd` drops.
- **The profiles Base / Avanzato** (`29b-commands.js`, 7b-2): every entry of the registry has a `level`
  (`cmdLevel(group, tool)`): «advanced» when its group is in `PROF_ADV_GROUPS` or the tool in `PROF_ADV_TOOLS`,
  otherwise «base». `profGet()`, `profSet(p)` (stores, fires the event `axpo-profile`, says it in a message),
  `profHides(group, tool)`, `profGroupHidden(label)`; `profInit(location.search)` at load: `?profilo=` wins and is
  remembered, then `axpo_profilo`, then Avanzato. `groupsOf` (`30-ui-glue.js`) leaves out the hidden tools and the
  groups left empty, so the ribbon and a phone's sheet follow; `modShow` without a module opens the first one with a
  visible tool (`modFirst`), asked by name it opens any (`modShowFor`: a DWG dropped on the map). On `axpo-profile`
  the ribbon is drawn again and a remembered module now hidden is forgotten. `32-tour.js`: the buttons
  `[data-prof]` of the «?» menu and of the guide, `stepShown` (a step whose `sel` is a hidden group is skipped),
  `profNote` on the last step. Hidden is not forbidden: the right-click menu of the map, a dropped file and the
  shortcuts still reach what Base hides; `cmdRun` of a hidden command returns false, as for a missing button. Parts of
  a panel hidden in Base too: `PROF_ADV_PANELS` (ids; the class `prof-hide`, set by `profPanels`), today the site
  card of Disegna (`#sfWrap`).
- **The search Ctrl K** (`29c-search.js`, 7c, from PV Predesign `ui/search.js`): `srchItems()` builds, at each
  opening, the list of what can be found — `{kind:'object'|'command', ok, ref, name, sub, hay, colour, icon, fig,
  hidden}`: the parcels, every graphic of `toolsGL` with an `objKind`, the entries of `cmdList()` the profile shows
  (not the `value` kind, not a tool whose `show()` is false); `srchMatch(q, list)` (every folded word in name +
  `hay`; objects first, commands up to `SRCH_CMD_MAX`); `srchMark` the bold. Ctrl K is a shortcut of the registry
  (`CMD_KEYS.search`, `CMD_RUN.search` → `srchOpen`). A pick: `srchGoTo` (`selSet`, `zoomGraphics` /
  `zoomParcelList`, `srchShowRow`, which finds the row by `row._obj`, set by `geomRow` and the parcel rows) or
  `srchRunCmd` (`cmdRun`, else the button's title in a message). `srchArmed()` is the mode banner.
- **Trap:** the owners' data never enter `hay` (only the name, sheet and number of their parcels). A new kind of
  object of the Raccolta needs its line in `srchItems` (and `SRCH_KIND`); a new row builder sets `row._obj`.
  `cmdRun` runs a `CMD_RUN` command also when it is not a ribbon tool (fixed in 7c: before, such a shortcut did
  nothing).
- **Trap:** a new tool or group of the ribbon is «base» unless it is listed: decide its level when adding it, and
  update the list of advanced commands in the `profiles` harness. The ⓘ, the guide and the tours keep one text for
  both profiles (plan §6t W8).
- **«Supporto al disegno»** (`aidForm` over `#drwDock`): snapping (`drwSnapSync` rebuilds the sources when a drawing or
  an edit starts: `toolsGL`, `gl` and, with the option, every visible `feature`/`geojson`/`wfs`/`csv` layer, DWGs
  included; map-image and WMS layers cannot be snapped to); angles (`directionMode`: `relative` = deflection, `absolute`
  = azimuth from north, clockwise; `axpo_drw_dir`); exact shapes; the grid (the SDK's `GridControlsViewModel`, true
  metres even in Web Mercator; «Allinea a un lato» = `interactivePlacementState = 'interactive'`; `rotation` in degrees
  counter-clockwise from east; not in the project).
- **Exact shapes**: with valid values the sketch is cancelled and the mode `shape` previews a rectangle (width ×
  height, the width's azimuth) or a circle, built in metres in the frame (`shpGeom` → `geoShape`); a click places it.
  `shpSync` switches freehand/exact as the fields change; `drwModeEnd(keepDraw)` and `cadModesOff(keepDraw)` keep the
  armed tool. «Da un lato» reads the nearest side of a drawing, parcel, DWG or layer feature (`drwEdgeAt`: our graphics,
  then the full geometry re-queried, then `identify`) and its ground azimuth (`geoSegment`).
- **Copy** (`drwMode` `cp1`/`cp2`): each click places a copy shifted with the clicked point (`geomShift`); our objects
  keep category, type, attributes, note, site and style, not `sf_gid`/`sf_saved`; a parcel or a layer feature becomes a
  drawing of the current category. **Parallel copy** (`off1`/`off2`): a parallel line, an inset (click inside) or an
  outset, mitred; «Solo il lato cliccato» moves one side.
- **Seleziona e modifica** (`23b-select-edit.js`): `selEditable` = drawings, results, imports (not parcels, automatic
  objects `sf_auto`, owners' parcels). `selSet(list, 'only'|'add'|'toggle')`, `selList`, `selCount`, `selClear`;
  `selRender` draws the outline (`selGL`), re-renders the rows unless `{rows:false}`, fills `#selCard`, refreshes the
  ribbon, and runs from `refreshDl`. Seleziona: a click (`hitTest`); Shift or touch toggles; nothing clears. Riquadro: a
  rectangle sketch (`selBoxDone`, tested after `geoProject`; `selShift` tracks Shift, which the create event lacks).
  Rimodella (`reshape`) and Sposta e ruota (`transform`, no scaling) are SDK updates after `selEditPrep` projects to the
  view's system. Dividi arms the cut (`gpPendingSplit`); Unisci replaces the areas with their `geoUnion` and the first
  one's attributes; Copia starts `cp2` from the centroid; Elimina asks, then asks again for the portal.
- **Esclusione da layer** (`exlStart`/`exlEnd`): a click reads a complete feature of a visible vector layer
  (`gpLayerFeatureAt`) and `exlAdd` draws it (area → *Esclusione*, line → *Infrastruttura lineare*, point →
  *Ostacolo*; source `official`, `src_attrs`, `_srcKey` so it is added once); its Riquadro queries every visible
  queryable layer (at most 200, a confirm over 20).
- **Automatic strips and net area** (`sfAutoSync`, `21-legend-auto.js`, PV Predesign's `areas.js` rules). What cuts:
  `SF_CUTS` (exclusion, linear, obstacle, mitigation, agri) and custom categories with `cuts`; the distance
  (`sfCutDist`) is the buffer, half the width of a mitigation or agri line, or a custom number field `buffer`, at most
  100 km. Each object with a distance gets a following polygon (`sf_auto:'buffer'`, `sf_of` → its `sf_uid`): an
  *Esclusione · Fascia di rispetto* (*Fascia DPA* for power lines), or for a hedge or agri line a strip of its own
  category. With a gross drawing and `netAuto` on, one `sf_auto:'net'` drawing = the union of the gross areas shrunk by
  `netSetback` (a negative `geoBuffer`), minus the union of all cutting polygons, strips and hidden objects included;
  lines and points cut only through a distance (the note counts the others). Off = frozen; deleting it turns it off.
- **«Crea area lorda»** (`sfGrossMake` → `sfGrossFrom`; the ribbon presses the hidden `#grossBtn`): the ticked parcels,
  or all, unioned (`geoUnion`) into a *gross* drawing; with subgroups and none ticked, one per subgroup plus one for the
  rest (`sf_pg`), each replacing its own; otherwise `sfReplaceOrAdd` replaces in place (same GlobalID, so the next save
  updates it) or adds another.
- **Trap:** `sfAutoSync` runs 200 ms after each `renderGeoms` (`sfAutoSoon`) and calls `refreshDl` only on a change; the
  signatures `sf_sig` (extent, vertex count, distance, name, type; for the net area every input and the setback) stop
  the loop render → sync → refresh → render. Whatever should trigger a recompute must be part of the signature.
- **Trap:** a strip cannot be deleted: it goes with its source, or when the distance goes to 0.

### 11. The local metric frame and the geoprocesses

- Constructions and geometric operations run in a local frame (`16b-geo-frame.js`, plan §6h): a Transverse Mercator of
  scale 1 centred on the inputs (`geoFrameWkt`, the WKT of PV Predesign's `geo/localframe.js`), true ground metres
  (scale error under 1e-7 within a few km, about 1e-5 at 50 km); one frame per operation, centred on the inputs' extent
  rounded to 0.01° and cached (`geoFrames`).
- Web Mercator never measures: it is not conformal on the ellipsoid (at 45° a ground metre is about 0.17 % more in x
  and 0.16 % less in y than 1/cos φ; a 100 × 50 m rectangle built that way measured 100.17 × 49.92 m). turf.js is gone:
  it worked in degrees, and its simplify used a fixed 0.0002° (about 22 × 16 m at 45°).
- The engine is `projectOperator`, loaded once (`geoFrameLoad`, 2.5 s after the view is ready); `geoProject` uses
  `webMercatorUtils` between WGS 84 and Web Mercator and the engine otherwise. Operations take any system and return
  the view's: `geoUnion`, `geoIntersect`, `geoDifference`, `geoHull`, `geoSimplify` (generalize, metres), `geoBuffer`
  (negative shrinks), `geoOffset`, `geoCut`, `geoMeasure`, `geoOverlap`, `geoShape`, `geoSegment` (azimuth 0–180°,
  length). `geoRun` drops non-finite inputs (`geoFinite`), projects and cleans (`geoClean` = `simplify`); `geoOut`
  cleans, drops empty results and checks the result is finite.
- In the frame: exact shapes, «Da un lato», parallel copy, geoprocesses, the parcel dissolve (export, AREAS), gross
  area, strips, setback, net area, Unisci, Dividi, the selection card. Outside it: DWG import and alignment,
  «Coordinate sconosciute», images, cadastre, the Measurement widget, Zornade's parcels, the terrain grid (own frame,
  same WKT), the automatic AREAS site link (overlaps compared in Web Mercator: a ranking only). The figures shown are
  geodesic from the SDK (`geodesicArea`, `geodesicLength`), except the selection card, which reads the frame.
- **Geoprocesses** (module `gp`, `#gpStep1`–`#gpStep3`). Slots **A** and **B** (`gpSlots`) hold objects: from the map
  (`gpPickStart`; a second click on the same spot steps down through overlaps), the ticked rows, or a category; dashed
  outlines in `gpSlotGL`. The `GP_OPS` cards (a radio group; `gpFlowSync` disables one, with the reason, until A fits):
  Buffer (`makeBuffer`, each or merged), Unisci, Contorno, Semplifica (`#gpSimpTol` in metres, 2 by default), Ritaglia
  (a polygon drawn now), Parte comune and Sottrai B da A (need B), Copia (no A), Dividi (`gpRun('split')`, from
  «Seleziona e modifica»). «Esegui» (`gpExec` → `gpRun`); results go to «Analisi» (`gpAddResult`; buffers
  `_kind:'buffer'`, the others `_kind:'drawing', _sub:'geoproc'`); errors in plain words (`gpErrText`), the technical
  text in the console; «Buffer qui» in the right-click menu preselects Buffer (`gpOpenOp`).
- **Map-layer features** (`gpPickLayerFeature` → `gpLayerFeatureAt`), when a click hits none of our objects: `hitTest`,
  then a query by OBJECTID on feature, WFS, GeoJSON and CSV layers (only what is drawn is hit, so filters apply), or
  `identify` on visible, in-scale map-service sublayers. The feature is copied as an import with its **full** geometry
  in WGS 84 (the hit one is generalised: 5,598 vertices against 192,532 on a real coastline) and its simple attributes
  (`src_attrs`, at most 80, exported); `_srcKey` (layer URL + OBJECTID; the title for blob-URL layers) keeps one copy.
- **Dividi** (`gpDoSplit`, `geoCut`): each area of A that is a graphic (not a parcel) and is crossed side to side
  becomes parts «nome · 1», «nome · 2»… with the same attributes and symbol but no portal id (new objects; the original
  stays on the portal until deleted there); the parts replace it in A; one Annulla restores it.
- **Heavy geometries**: the synchronous engine costs about 0.3–1 ms per vertex (on 192,532 vertices: a geodesic buffer
  about 3 minutes, a planar buffer in UTM 54 s plus 8 s of projection, `generalize` 17 s; the 4.34
  `geodesicBufferOperator` 28 s on 65,819). Above `GP_HEAVY` (5,000) `makeBuffer` first simplifies with `geoSimplify`
  at 0.5, 1, 2, 5, 10, 20, 50 m (at most 1/10 of the distance) until under 5,000 vertices, says so in the message and
  the name, and shows the message *before* the synchronous work (a 60 ms timeout); other operations warn over 20,000.
- **Guards** (the `crash` harness): `geoFinite`; `gpEmptyResult` (no rings, non-finite or zero area → «Nessun
  risultato»); the rejection of an interrupted `goTo` caught; `GP_DMAX` (100 km); `drwAddGraphic` refuses a non-finite
  shape; `gpRun` drops objects no longer on the map. Tolerated: shapes very far apart (one local frame: a poor result,
  nothing breaks). **Oracle**: `tests/harness/geo.py` against `tests/fixtures/geo_expected.json`, made by
  `scripts/make_geo_fixtures.py` with arcpy in the same frame (its 100 × 50 m rectangle measures 5,000.000 m²); arcpy
  is developer-only, the app never uses it and the runner only reads the JSON.
- **Trap:** load the engine before the first projection (`geoFrameFor` throws otherwise): entry points check
  `geoFrameReady()` and wait (`makeBuffer`, `gpRun`, `sfAutoSync`) or ask to retry.
- **Trap:** rings keep their source's orientation (GeoJSON counter-clockwise, SDK clockwise): every local geometry goes
  through `simplify` first, or areas come out negative and unions empty.
- **Trap:** results go back in Web Mercator or WGS 84, never in the frame: `to4326()` assumes one of those two.
- **Trap:** the sign of `geometryEngine.offset` is undocumented: the parallel copy computes both sides and picks by
  area (inset or outset) or by distance from the click.
- **Trap:** `esriMods` has no `Polyline`: build lines through a `Graphic`, which autocasts a plain object (`mkLine`,
  `geoGeom`). The `Polygon` constructor sanitises NaN vertices, so junk for a test must come as a plain object.
- **Trap:** a ribbon geoprocess tool only arms the operation; «Esegui» runs it.

### 12. Terrain

- «Terreno» holds the modules `slope` (`17-slope.js`) and `elev` (`.toolpane[data-tool=…]`); `toolOpen(k)` opens the
  tab, and `modShow` → `toolSync` turns off the previous tool's map mode (`toolModeOff`).
- **Extent** (`slpSources`): one drawn here (`slpExtentStart` → `terrain-extent` → `slpExtentSet`; `slpExt`, outlined
  in `slpExtGL`, saved as `terrainExtent`; `slpExtentSave` makes it a gross drawing), a gross drawing, the site AREAS,
  or the parcels unioned.
- **Model** (`slpExtract`, PV Predesign's terrain method ported function by function, so both give the same numbers on
  the same grid): a grid in a local TM of scale 1 centred on the area (`slpWkt` = `geoFrameWkt`), 5 m cells (2–50,
  grown by 1.25 until under 400,000 cells), over the area plus 25 m (`SLP_BUF`); Esri World Elevation at the cell
  centres (`queryElevation`, `finest-contiguous`, 5,000 points per request; about 10 m native in Italy: scouting, not a
  survey; no sign-in, no credits); slope in percent with Horn's 3 × 3 weights. Extracting the same area again reuses
  its grid id, also after a reload (from the exclusion's `sf_slope.grid`).
- **Limit** (`slpForm`): `{maxAny}` or `{maxNS, maxEW}`, as in PV Predesign; `slpMask`; patches under 250 m² (default)
  dropped; cells traced into rings (`slpRings`) and clipped to the area. A new limit, minimum or view redraws without a
  new download (`slpRefresh`). The limits are the user's: the design standards stay in PV Predesign (this repository is
  public).
- **Images** (`slpCanvas`, one at a time): slope classes 0–5–10–15–25 %, shaded elevations, aspect (`slpAspectColour`),
  hillshade (light from the north-west at 45°, relief × 3), cells over the limit hatched in red; the image is an
  Immagini entry (`im._slp`, no handles). Contours (`slpContourLines`/`slpJoinSegs`, marching squares) go in `slpCtGL`,
  every fifth thicker, memoised per grid and step.
- **Exclusion** (`slpAdd`/`slpCutRemove`): one drawing per grid, *Esclusione · Pendenza elevata*, with
  `sf_slope = {limit, cell, min_patch, buffer, source, dtm, grid, area, date}`; it follows the limit while the switch is
  on, and «Rimuovi terreno» (`slpClear`) keeps it. On the portal the limit is only in the note (no field, by decision).
- **For PV Predesign** the `.axpo` carries `terrain`: per DTM on the map
  `{id, lon0, lat0, x0, y0, cell, nx, ny, zmin, zmax, missing, loadedAt, source, name}` (Predesign's own grid
  description) plus `buffer`, `area_name`, `area_geom`,
  `limit`, `min_patch`, `view` (`s`, `z`, `a`, `h`) and `file`; the elevations in `terreno/<id>.f32` (Float32,
  little-endian, rows from the south-west cell, NaN = unknown). `slpProjRestore` rebuilds the image. Not in the browser
  save: after a reload the exclusion stays, the image does not. The profile is the SDK `ElevationProfile` (`elevWatch`).
- **Trap:** the terrain uses the older `projection` module and loads its own `Multipoint`, `SpatialReference` and
  `Polyline` (`slpMods`), not `projectOperator`; its frame is centred to 1e-6°, not 0.01°.
- **Trap:** the ribbon's terrain tools press or mirror hidden controls (`div.rib`) wired by `slpWire`; `slpFill` runs
  from `wireSectionTools` while the module is open, so the list of areas follows the Raccolta.

### 13. Import

- **Vector files** (GeoJSON, KML through `togeojson`, KMZ, zipped Shapefile through `shpjs`): `impAdd` makes imports
  (`_kind:'import'`, one colour per file), skipping and counting features with non-numeric coordinates (`gjFinite`). A
  file without a declared CRS goes through `srEstimate` on its median point and is converted alone only when Web
  Mercator is the *only* plausible reading; otherwise «Coordinate sconosciute» opens with the candidates
  (`srFromImport`; `srImportPending` reprojects with the system chosen). `impOpen(id)` opens the card the code needs.
- **Trap:** never trust «it lands in Italy»: many UTM and Gauss-Boaga Ovest coordinates read as Web Mercator fall in
  Italy's box (a UTM 33 point of Lecce landed in the sea off Algeria, a Gauss-Boaga point of Sassari in Sicily).
- **Owners file** (from the separate *Estrazione intestatari* tool, SISTER data), recognised by `sister_stato`
  (`ownFile`). Each parcel is an import with `own = {k, t, l, n, st}` (class key = the upper-cased short names, sorted,
  so co-owners in any order are one class; group title; map label; number of owners; SISTER status) and the file's
  properties in `src_attrs`; `ownIs(g)` is the test everywhere. Rows are named by parcel
  (`Fg 12 · 101 · Borgo Inventato`), never by person, since geoprocess results take their inputs' names; coloured by
  class (twelve fixed
  colours, then hues at the golden angle; a class on the map keeps its colour, a new one never reuses one; grey without
  owners); listed at the end of Particelle (`#listOwn`, `ownRenderList`); `ownRemoveAll` (the bin of *Per intestatario*)
  removes them all with one confirm and one Annulla. Names on the map in `ownLblGL` (`ownLblSoon`).
  A click opens a popup, not an edit: the SDK update is cancelled at `start` and `ownShow` opens the popup at the last
  `immediate-click` (a Shift-click on two still edits). Files of version 1.1 lack `nominativi_brevi`/`etichetta`: names
  are cut from `nominativi` (`ownShort`). A plain parcel export imported by mistake gets a hint naming the right file.
- **Personal data** stay in the page and the `.axpo` (the save message says so): `serializeToolGeoms(true)` (browser
  save) skips them, `sfCandidates()` (portal) skips and counts them, `applyWork` does not print a failed record; the
  file's text reaches the page only through `textContent`, never the console. **Trap:** a new way of saving or
  publishing imports must add the `ownIs` test.
- **CAD (DWG)** (`26-dwg.js`): read in the browser by `@mlightcad/libredwg-web` 0.7.14 (LibreDWG in WebAssembly, about
  9.5 MB, from jsDelivr via `import()` on the first DWG). Layers and colours are kept (ByLayer/ByBlock resolved, blocks
  and arrays exploded, ACI 7 flips with the basemap). A CAD layer is a `GroupLayer` of up to three `GeoJSONLayer`s from
  blob URLs, coloured by a unique-value renderer keyed by style (a recolour rebuilds renderers, not features). Texts are
  drawn on the fly in `cadTxtGL` (visible layers, current extent, at most 1,500, at real size; under 5 px skipped). A
  layer's colour in the list is its most frequent entity colour. `cadReattach` follows web map swaps. Drawings are
  never in the browser save; their placements are (`cad_align`, per name + size).
- **CRS of a DWG**: `srEstimate` on the median (a title block at the origin does not move it) among `SRS_IT` (23
  systems; `co` marks the eight company ones: RDN2008 and WGS 84 UTM 32/33/34, Monte Mario 1/2), confirmed in a dialog.
  Candidates within 1 m of each other (RDN2008, ETRS89, WGS 84, IGM95 in one zone) are offered together, the datum
  choice kept in `cad_utm_fam`; the ranking is land against sea (ISTAT when signed in, else an embedded outline),
  closeness to the view, longitude in the zone's band. mm and cm are caught by retrying at 1/1000 and 1/100. Elements
  over 50 km from the median, and a detached cluster at (0,0) (title blocks, associative-array items at their local
  positions), are dropped by default. **Two-point alignment** (`27-align-sr.js`, like AutoCAD's ALIGN): rotation and
  translation (scale 1:1 or free) on the two pairs' midpoints in the file's metric working system; `f.ch.M` per file.
- DWG traps:
  - **One fresh WASM instance per file**: a second `convert()` fails with `RuntimeError: null function`; an instance
    costs about 50 ms but asks for 1 GB of initial memory. No DXF (commented out upstream).
  - Angles in radians; true colour beats the ACI index (0 ByBlock, 256 ByLayer); a closed LWPOLYLINE is `flag & 512`;
    the version comes from the first six bytes (`AC1032` = 2018+). **Flags are numbers** (`isCCW: 0`): test
    `=== false || === 0`. A clockwise hatch arc is (−end, −start) counter-clockwise, reversed, as ezdxf does.
  - **OCS**: extruded entities go through AutoCAD's arbitrary axis algorithm **including the OCS z**: PVcase tilts
    tracker blocks by fractions of a degree with insertion z of thousands of metres (metres of shift once projected),
    and above 1/64 of tilt the OCS X axis turns by 90°. The pipeline uses 3D matrices (`c3*`), the 2D ones only for the
    saved alignment. Ground truth: arcpy (`scripts/dwg_confronto_arcpy.py` in the AGOL Axpo folder, outside this repository; read only
    `\Polyline` and `\Point`: a cursor on `\Polygon` or `\Annotation` crashes on PVcase files).
  - Block attributes come twice (only the INSERT copy is used), PVcase modules too (3DSOLIDs skipped, 3D polylines
    drawn); MULTILEADERs are rebuilt from `textContent`, `textAnchor` and `leaderSections`.
  - Fills are projected as polylines (a lone counter-clockwise ring is a hole, and the engine returns a polygon made of
    holes at (−180, −90)), then re-oriented and nested even/odd like AutoCAD's hatch rule.
  - Monte Mario: the engine returns transformation 1660 everywhere (1661 and 1662 listed, never chosen), and `project`
    without a transformation applies the same, so GeoTIFFs land where DWGs do.
  - One `GraphicsLayer` per CAD layer took a 78,000-element layout to 1.3 GB and 1 fps; GeoJSONLayers keep features in
    the worker (about 360 MB and 60 fps for two layouts). Cassini-Soldner coordinates cannot be resolved (not in EPSG).
- **«Coordinate sconosciute»** (module `sr`, `srFind`): pasted pairs (decimal comma, thousands separators, DMS, a
  label: `srParse`) are projected through `SRS_IT`; those landing in Italy are shown, X/Y swapped included, ranked by
  plausibility or, after «Dove dovrebbe stare» (`srRefStart`, a snapped click), by geodesic error, which separates
  ED50 from WGS 84 (about 200 m). All offline, in the SDK's engine (2,000 wkids in 0.09 s).
- **Images** (`25-images.js`): an image is placed by its four corners (`ControlPointsGeoreference`, projective, in a
  `MediaLayer`), which is all a project stores. Ways in: GeoTIFF metadata or a `.prj`; a world file with its `.prj`
  (without one, `srEstimate`; degrees = WGS 84); the SDK `transform` («Sposta / scala / ruota») or `reshape` («Angoli»)
  on a dashed frame polygon; «Punti» (2 pairs = similarity, 3 = affine, 4+ = least-squares affine or projective,
  residuals in metres, kept as `cps`); «Valori» (`imgGeoApply`); a PDF page through pdf.js. The card `#imgToolbar` is
  remembered (`axpo_img_card`). «World file» exports image, world file and a Web Mercator `.prj`: the original when the
  placement is affine, else resampled north-up (`imgWarp`, nearest neighbour, at most 4096 px) as `<name>_nord.png`.
- Image traps:
  - Transforms are 3 × 3 homogeneous matrices (`h3*`) from pixels (top-left, y down) to view coordinates; `h3Fit`
    normalises both sides first (centroid 0, mean distance √2). The similarity includes the **reflection** y-down →
    y-up (`X = a·px + b·py + c`, `Y = b·px − a·py + d`); the orientation-preserving form fits nothing.
  - World files list A, D, B, E, C, F (rotation terms in the middle); C and F are the **centre** of the top-left pixel,
    for the original pixel grid.
  - Capture `view.popupEnabled` before the first `update()` (`imgPop0`) and restore it yourself. The georeference is
    the truth: `cancel()` may leave the frame where it was dragged, so `imgFrameSync` rebuilds it before every
    `update()`. The main sketch's `updateOnGraphicClick` is paused during an edit; the arrow-key nudge is the app's.
  - `ow`, `oh`, `page` and `dpi` go through `createGeorefImage(…, extra)` *before* `editImage` opens the card.
    Degenerate placements (a folded frame, collinear points) are refused silently.
  - pdf.js's UMD build registers as an AMD module: use the ES build via `import()`; its cross-origin worker goes
    through a blob module (`GlobalWorkerOptions.workerPort`).
- **Web services** (WMS, WFS, ArcGIS REST) by URL, without a portal item. Each layer carries a recipe (`_projSvc`:
  type, url, layers, `mode`) in `addedServices`, saved in the browser and the project and rebuilt by `svcCreate`; a dead
  service is dropped at start with a message. WMS: the chosen sublayers only, EPSG:3857 forced when listed, zoom to the
  chosen sublayers' extent. WFS: `WFSLayer` wants WFS 2.0 in GeoJSON and first asks a GML sample, which some GeoServers
  refuse; the fallback is a GeoJSON, WGS 84 `GetFeature` (2.0, then 1.1, at most 20,000 features) as a local
  `GeoJSONLayer` (a snapshot; axis order checked against the capabilities' box; `mode:'geojson'`). REST:
  `Layer.fromArcGISServerUrl` (a map service → one `MapImageLayer` with the chosen sublayers on; a feature service →
  one `FeatureLayer` per chosen layer). Portal items by title: `addPortalLayer`; `addedPortalItems` re-added at sign-in.
- **Trap:** the limits are the browser's, not AGOL's: no http from an https page (an `http://` URL is retried as
  `https://`), and capabilities and data only with CORS (the national PCN WMS has none). Those need a server-side proxy,
  and a future CSP must allow them in `connect-src` and `img-src`.

### 14. Export and the portal

- **Export** (`28-export-window.js`; «Esporta» presses `#dl`): a section per group in the Raccolta's order
  (`exSections`), the ticks shared both ways with the Raccolta (none ticked = everything; XLSX = parcels only).
  `buildFC` (parcels) and `graphicsFeatures` (the rest) give each feature `gruppo`, `sottogruppo`, the model in clear
  (`category`, `type`, numbers, `project_code`), the custom fields (`catExtProps`) and the original attributes of layer
  and owners' imports. KML: a `<Folder>` per group, sub-folders per parcel subgroup, the document named after the file
  (`toKML`, `exGroupOrder`); KMZ the same, zipped; Shapefile: one `<base>_shp.zip`, a folder per group, a shapefile per
  group and geometry type (`shpZipByGroup`; `_punti`, `_linee`, `_aree`). File names (`exBaseName`):
  `Borgo_Inventato_<yyyy-mm-dd>_6part.geojson`; several municipalities → the largest plus `_e_altri_<n>`; no parcel →
  `export_<yyyy-mm-dd>`; a parcel is any feature with `foglio` and `particella`; letters, digits, `_`, `-` only.
- **Trap:** `@mapbox/shp-write` 0.4.3: `download()` does nothing (use `zip()`), `MultiPolygon` and `MultiPoint` vanish
  silently, and every type gets the same file name unless `types` is passed; `shpPrep` merges multi-polygons with
  clockwise exteriors and splits multipoints. DBF names take 10 characters: `shpDbfProps` (`sottogruppo` → `sottogr`,
  longer keys cut and made unique). Inside the zip the date's hyphens become `_`.
- **Trap:** ArcGIS wants exterior rings clockwise and holes counter-clockwise, GeoJSON the opposite, shapefiles follow
  ArcGIS. `esriRings` decides exterior or hole by nesting (even/odd, by a majority of sample vertices, so parts touching
  at a vertex are not holes); `esriGeomToGeojson` nests with `cadGjPolys` and writes a real `MultiPolygon`.
- **Migra particelle in AREAS** (`confirmPromote`, presses `#promote`): layer 426 (`AREAS_LAYER_URL`), wkid 6876
  (RDN2008). Parcels, or their dissolve (`buildFC(true)`, which stops and names any parcel without an outline or failing
  the union), are reprojected 4326 → 6876 and simplified (otherwise GeoJSON rings land as `isSimple: false` and the
  popup's Arcade `Intersects()` fails silently). `PRO_COM_T` and `COMUNE` come from ISTAT on the centroid (the
  seismic-zone Arcade needs them); REGIONE and PROVINCIA from the parcel (Zornade's spelling, as in the layer), the
  drop-downs last; `area` is geodesic (planar in 6876 drifts up to 0.7 % at Italy's edges). Overlaps are reported once.
- **Modifica AREAS COLLECTION** (computer only): the SDK `Editor` on layer 426 while it is in the map, destroyed and
  rebuilt at every web map swap. After an edit the area is recomputed geodesically (WGS 84 from the server) with the
  ISTAT lookup; `recomputingArea` stops the `edits` loop for 800 ms.
- **Salva disegni in Site Features** (`sfConfirmSave`, presses `#sfSave`): `SF_URL`, layers 0, 1, 2 for points, lines,
  polygons (`SF_LID`); fields CATEGORY, TYPE, NAME, NOTE, BUFFER_M, HEIGHT_M, WIDTH_M, VOLTAGE_KV, PROJECT_CODE,
  AREA_GUID, SOURCE, STATUS (coded domains: English codes, Italian labels). Each object gets a GlobalID made in the
  browser (`sf_gid`) and `applyEdits(…, {globalIdUsed: true, rollbackOnFailureEnabled: false})`: added the first time,
  updated after, no duplicates. Each needs its own result (none = an error, still to save); a failed update is re-added
  only when a `GlobalID IN (…)` query shows it gone. Geometry goes in Web Mercator, polygons simplified (`sfGeomOut`).
  To save (`sfNeedsSave`): never saved, `sf_dirty`, or without a site while a site AREAS is set. Kept out: owners'
  parcels, and app-only categories unless the dialog's select gives their geometry a model category (`sfEffCat`,
  `sfStaysOut`; `sfAttrsOut` never sends an app-only code).
- **Automatic site link** (`sfAutoSite`): an object without a site takes the AREAS COLLECTION area it touches — one
  query on the extent of all (one per object when the answer has `exceededTransferLimit`), then the largest overlap
  (area, length, any for points) — else the site AREAS, else none, and the message says so. `sfRefreshCodes` re-reads
  `Project_Code` by GlobalID (a new area gets its code from the CRM later). The site AREAS (`sfSite`: guid, code,
  comune, regione, ha, GeoJSON) is optional: it forces a site on new drawings, covers objects outside every area, and
  «Carica elementi» (`sfLoadSite`) brings its saved features in, skipping those already there.
- **Trap:** GlobalIDs are compared normalised (`sfGuid`: upper case with braces); the service returns lower case without
  braces and accepts every form in a `where`.
- **Trap:** an object changed after a save must get `sf_dirty` (the update handler, rename, the editor, the automations
  and `sfReplaceOrAdd` do); a copy, a split and a merge drop `sf_gid`/`sf_saved`, so the original stays on the portal.
- **Percorso su strada** (module `rte`): `rteSolve` uses the organisation's route service (`helperServices.route`, else
  `Route_World`) with `esri/rest/route`, stops in order (a simple route: 0.005 credits whatever the stops), *Walking
  Distance* (shortest, no motorways, one-way streets ignored) or *Rural Driving Distance* for «Solo strade carrabili»
  (by name, else by type `WALK`/`AUTOMOBILE` with a distance impedance). Straight connectors join the clicked ends to
  the road; z and m are dropped; the result is category `connection`, type `estimated`. **Trap:** signed out it refuses:
  a request without a token would start a sign-in redirect.
- **Analisi di contesto** (module `ctx`, signed in): `querySubstations` reads portal layers around the click —
  substations within 80 km (six nearest), the nearest HV line within 60 km, PV projects within 15 km (non-sensitive
  fields only, `PV_PROC_FIELDS`). **Stampa** (computer only) is the SDK `Print` widget.

### 15. The `.axpo` project and the browser storage

- **Work state** (`captureWork(noOwn)`, `14-state-project.js`), shared by the browser save and the project: `basemap`,
  `viewpoint`, `parcels`, `addedItems`, `services`, `site`, `netAuto`, `netSetback`, `terrainExtent`, `groups`,
  `parcelGroups`, `customCats`, `toolGeoms` (`serializeToolGeoms`: drawings, buffers and imports as
  `{g: GeoJSON 4326, a: whitelisted attributes, v: visible, s: symbol JSON}`). `applyWork` restores the flags, groups,
  subgroups and
  custom categories before any object, then the site, extent, parcels (junk dropped) and graphics (symbols through
  `fromJSON`, old categories upgraded).
- **Browser save**: `saveState` writes `cat_state` 800 ms after a change (`saveStateDebounced`) with `captureWork(true)`
  (no owners). `restoreState` sets `bootVP`, applies the work, view and basemap, rebuilds the services in order
  (`svcRestoreAll`; `svcGen` discards a restore overtaken by a clear or a project) and keeps the portal items for after
  sign-in; the history restarts. Not saved: DWGs, images, DTMs, layer visibility, opacity, order and filters, the web
  map, owners. Over the quota (about 5 MB per site) the state is saved without geometries, with one message. Other keys:
  `axpo_*`, `cad_*`, `cat_bookmarks`, `zrn_tok`. *Impostazioni avanzate* (`#tokBox`, at the foot of «Proprietà»):
  `storeInfo`, «Svuota il lavoro» (`clearWork`; theme, bookmarks, DWG placements stay), «Ripristina tutto» (every
  `APP_KEYS` key, then a reload).
- **`.axpo`** (`projBuild`): a zip with `progetto.json`, `file/dwg-<n>-<name>`, `file/img-<n>-<name>` and
  `terreno/<id>.f32`. The JSON is the work state plus `formato: "geoportale-axpo-progetto"`, `versione: 1`, `nome`,
  `salvato`, `utente`, `webmap {id, title}`, `area {reg, prov, com}`, `labels`, `layers` (key, title, visibility,
  opacity, popup, labels, `def`; sublayers `sub` and `sdef`), `order` (per container), `images` (`name`, `orig`,
  `file`, four `corners` with their wkid, `cps`, `page`, `slp`), `dwg` (`name`, `file`, `wkid`, `sc`, `M`, `drop`,
  `manual`, `open`, per-layer visibility and colour) and `terrain`. The reference is `docs/axpo-format.md` (phase 9):
  `versione` stays 1 while fields are only added, rises when a meaning changes or a field goes; a newer file opens with
  a warning. `projNormalize(st, warn)` gives every missing value its default before `applyWork` (arrays, objects,
  `labels`, `netAuto`, `netSetback`, `_kind`), and reads a category this version does not know as `unclassified`,
  keeping the code in `sf_cat_orig` (serialized again). `projBuild(name, {share:true})` is the copy to share: `own`
  and `src_attrs` removed from the owners' parcels, their `source` neutral, `utente` null, `copia_condivisa: true`;
  `projShare()` saves it (never over the open file). PV Predesign opens it and ignores `groups`, `parcelGroups`,
  `customCats`, `sf_ext`.
- **Trap:** a new field of the project goes into `docs/axpo-format.md` and gets its default in `projNormalize`; a new
  field holding personal data is removed by the copy to share (`projBuild` with `share`) and by the autosave
  (`captureWork(true)`); the `axpo` harness checks both.
  Bookmarks, theme and the Zornade key stay out; no token is written.
- **Layer keys**: web map layers by id (`id:<id>`, the title as fallback); the app's own layers get new ids at every
  load, so they go by role (`app:catasto`, `dwg:<n>`, `img:<n>`, `svc:<n>`, `item:<portal id>`), and on opening `keyMap`
  maps the rebuilt ones, so a failed file shifts nothing. The order is restored by permuting the found layers among
  their own slots, per container (`listMode:'hide'` layers never move). Sublayer visibility waits for `layer.when()`
  without forcing a load (a Check Vincoli map reads its regional WMS only when switched on).
- **Open** (`projOpenFile`): a confirm, `clearWork`, the portal part if signed in (else `projPortalPending`, applied
  when the sign-in completes without a reload), area, basemap, services, work, images (DTMs from their grid), DWGs
  (`projCadRestore`, no CRS dialog), layer states and order, labels, and the viewpoint last. The name goes to the top
  bar (`projSetName`, `#projNameBtn`), the tab title and «Progetto» (`#modProj`, `projModRender`).
- **Save** (`projSave`): Ctrl+S to the same file; Ctrl+Shift+S, Shift+click or the top-bar name to save as; Ctrl+O
  opens. With the File System Access API (pickers in the window where the user works) the file is written in place,
  else downloaded. Measured on two real layouts (78,100 + 47,495 elements): 2.9 MB, saved in 0.3 s, reopened in 8.6 s,
  vertices identical to 1e-7°, a peak of about 800 MB on opening that settles near 360 MB.
- **Trap:** a new attribute of a graphic must join the whitelist of `serializeToolGeoms`, or it is lost on reload and
  in the project; a new part of the work state goes into `captureWork`, `applyWork` and `clearWork`, restored before
  what depends on it.
- **Trap:** the picker and the write permission are asked *before* the zip is built: after a few seconds of work the
  click no longer counts as a user gesture.
- **Trap:** the source `File` kept on each DWG (`f.file`) and image (`im.file`) points at the disk and costs no memory
  until saving; a placement without its file cannot be saved, and the message names what is missing.

### 16. Tests

- `tests/run.py` (Python and headless Chrome) writes `tests/out/proj.html` (the build with a hook that collects
  `console.error`, uncaught errors and rejections in `window.__errs`) and, per harness, `tests/out/<name>.html` (the app
  plus a script that drives real functions with invented data and writes its JSON into `<pre id="out">`). It serves
  `tests/out/` on a free port and opens each page with `--dump-dom`, virtual time and `--disable-popup-blocking`, at the
  harness's `SIZES` (default 1366 × 768), with `?noauto&catasto=off`: no session, no portal. A harness is a Python file
  with `NAME`, `JS` (an async IIFE), optional `SIZES`/`QUERY` and `check(result, size)`, which returns the failures.
  `py -3 tests/run.py [names] [--build] [--keep]`.
- First, Node parses every `<script>` block (`new Function`); a syntax error stops the suite (skipped without Node).
  Network noise (a basemap, the terrain or an AGEA orthophoto not loaded, an SDK chunk or an icon not served,
  `TypeError: Failed to fetch`, an anonymous `Script error. @0` from a cross-origin script) is filtered (`NOISE`); a
  run spoilt by it, or a page with no result (Chrome left early), is repeated once. The page keeps a copy of its console
  errors next to the result (`<pre id="errs">`), which the runner reads when a harness died before collecting them
  (the SDK not served by the CDN: «SDK non ancora caricato»), so that run is repeated too.
- Every test page is a first visit (a fresh Chrome profile): the hook marks the first-visit tour as seen
  (`axpo_tour_seen`) before the app starts.
- The twenty-six harnesses:
  - `smoke`: the view is ready, the app's layers, the Agenzia cadastre, six tabs and the panels' tabs.
  - `zoom`: the zoom after a selection goes only to the action's parcels, only when they are out of view.
  - `overlay`: the cadastre image placed to the pixel through pan, rotation, scale, switch and opacity (needs the AdE).
  - `responsive`: at 390, 820 and 1366 px: phone bar and sheets, one panel at a time, messages, the status card.
  - `automations`: gross area, strips, net area, setback, freeze, browser save.
  - `legend`: the Legenda tab mounts, the Raccolta block follows the eyes, a ribbon tab brings «Proprietà» back.
  - `tour`: at 1214 and 1366 px every tour step is visible and on screen; the guide; the «i» windows.
  - `select`: selection, the box with and without Shift, Unisci and Elimina with Annulla, exclusion from a layer.
  - `groups`: groups and subgroups, gross area per subgroup, `gruppo`/`sottogruppo`, KML folders, shapefile per group.
  - `geo`: the local frame against arcpy, including the figures the user reads.
  - `cats`: «Non classificato» and custom categories through editor, legend, export, DBF, cut, state, save dialog.
  - `crash`: about fifty junk cases: no error logged, a message when due, nothing added for an empty result.
  - `layout`: the ribbon at 1214 and 1366 px, the popovers, the measures, the panels' tabs and widths.
  - `undock`: both panels in windows and back, alone and together.
  - `raccolta`: one-line rows, hectares, the filter, «Mostra altre», «Comprimi», the editor not rebuilt while typing.
  - `texts`: every ⓘ names its tools, the «Di più» windows, the guide, no old words; the «?» on a phone.
  - `panelswap`: the ribbon with both panels undocked, a window closed with its ×, the page closed, a web map swap
    (`wmCarry`) and the basemap chip after it.
  - `undo`: one command, one Annulla: drawings, a drawing with its strip, parcels, groups, a rename; Ripeti after the
    automations; the portal identity through Annulla; the names in the tooltips and the message.
  - `commands`: the registry against the ribbon of every tab, the shortcuts (and Ctrl+Z left to a text field), `cmdRun`.
  - `rules`: the kinds of object, the row menu by kind, a strip that cannot be deleted or recoloured, the net area out
    of the editor, the selection on the map that says why, Dividi on a strip, the copy that drops the internal fields,
    the portal and migration filters against the table.
  - `profiles` (1366 and 390 px): Avanzato shows every command; Base from the «?» (the guide on a phone) hides
    exactly the advanced commands of the registry and the groups left empty; the first module; a hidden command does
    not run; the tours; `?profilo=` and the memory; back to Avanzato; the site card hidden in Base.
  - `search`: Ctrl K; words, case and accents; every kind of object, the owners' names not searched, hidden;
    objects then commands, 8 and «e altri n»; the pick of a drawing, a parcel, a command, a disabled command; Esc;
    the «?» menu; a map mode; the profile Base.
  - `statusbar`: elevation and slope from a synthetic terrain grid (over the limit, split limit, outside, no model);
    the scale menu; the autosave time and the full browser.
  - `axpo`: a project saved and opened again; the copy to share; an old file; a file of a newer version.
  - `owners`: the owners' file with no parcel searched shows at the end of Particelle; its bin and Annulla; a plain
    GeoJSON of 120 features in «Altro».
  - `vtpopup`: the vector tile popups on a simulated hit test (popup.json and fallback, labels, fill and outline once,
    after the native popup, not with a tool armed or on an object of the Raccolta, never the basemap).
- **Trap:** the history photographs the work 0.5 s after it settles, and every change pushes the photo back: a harness
  that removes something and then tests Annulla waits for `drwHist.timer` to be 0 first, or the state before the
  removal was never photographed (`owners` failed once under the full suite on 2026-10-30).
- **Debugging**: «nessun risultato nella pagina» means the script never wrote its JSON (an error in the harness, such
  as a duplicate `const`, or in the app). Serve `tests/out/` with any static server and open `<name>.html` to read the
  console; `--keep` also keeps each raw result as `tests/out/<name>_<size>.json`.
- **Trap:** a harness waits for `view.ready`, closes the tour, calls `geoFrameLoad()`, stubs `window.confirm`, and
  waits a tick after a state change before it clicks a ribbon tool. A `type="number"` input refuses a decimal comma:
  harnesses write `12.5`.
- **Trap:** «close the tour if it is on screen» is not enough on a first visit: the tour starts by itself 1.8 s after
  load, and when the view is ready earlier it starts later, in the middle of the harness, docking the right panel
  (`startTour` → `sideDock`) and switching tabs. This made `undock` fail about one run in three on 2026-10-27; the hook
  now marks the tour as seen. Prefer waiting for the state a step checks (`until` in `undock`) to a fixed wait: under
  virtual time a wait costs nothing while nothing happens, and Chrome stops the clock while a request is pending.
- **Trap:** without a session the portal writes, the route and the context analysis are never tested: try them by
  hand, signed in, on test data — the checklist is `../reports/collaudo_login_vero_2026-10-27.md`.
- `scripts/make_geo_fixtures.py` rebuilds `tests/fixtures/geo_expected.json` with ArcGIS Pro's Python (arcpy; about six
  minutes, most of it the import).

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

**Where things stand on 2026-10-23.** `main` of `UBS-IT/custom-AGOL-web-map` = `ba40df2` (PR 23): the ribbon refactor
through review round 2 and phase 4b is merged (PRs 4–23, one per phase, in the order of the plan
`docs/plan-ribbon-refactor.md` §6g); next is phase 5 (guide, tours, ⓘ texts, §4 and §5 of this README rewritten once,
`docs/architecture.md` written last from the «Files touched per phase» list of §8), then 6 (the company repo), 7a–c,
8 and 9. The entries below are the dated record, one per phase, newest last. The handover for a new session (state,
procedure, open items) is `HANDOFF.md` in the project folder, outside this repository.

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
company repo for now (no control, unfamiliar), so the function stays in `scripts/`, out of the published files.
**2026-10-10, later:** the published page showed the cadastre once the browser cache was refreshed (Ctrl+F5); the
user then missed the parcel numbers and the sheets: the image now asks for `CP.CadastralZoning` and
`codice_plla` too and its limit is back to 1:25,000 (sheets from there, parcels below 1:5,000, numbers below
1:2,000). Looked at in the built-in browser: sheets with numbers at 1:18,000, parcels at 1:2,257, numbers at
1:1,128 (zoom 19). Backup: `geoportale_axpo_pre-catasto-fogli-numeri_2026-10-10.html`. Backup:
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

**Messages, service status, phone and tablet (2026-10-10, UX review points 1 and 2, user: "procedi con 1 e 2").**
§4 *Messages and service status* and *Phone and tablet layout*. Checked in headless Chrome with a test page that
drives the real handlers (`scratchpad/dwg/make_resp_test.py`): at phone width the bottom bar has nine buttons, the
page starts with the map, Cerca opens the left panel full screen and closes the drawer, Raccolta opens the drawer
full screen and collapses the left panel, × closes, Disegna + × and Layer toggle work, the status card shows and
hides, «Vedi ›» opens the Layer panel; at tablet width the two panels exclude each other; at desktop both stay open
and the toast is centred at the top; no console errors. Looked at in the built-in browser at 375×812 (map, Cerca,
Raccolta with invented parcels, Layer with the status card). Headless screenshots at 390 px are cropped (Chrome's
minimum window is ~500 px): use the pane's Mobile preset for phone visuals. Backup:
`geoportale_axpo_pre-telefono-messaggi_2026-10-10.html` (+ README).

**Cadastre back without a server (2026-10-10 evening, user: "trovare un modo senza creare strumenti complessi").** §4
*Cadastre as an image over the map*, §5 *The cadastre image*. Exploration first: Zornade draws parcels as vectors
from its own database (its `locate?bbox=` returns centroids only, 200 per call, no geometry); the ArcGIS Online
sharing proxy answers 403; the Agenzia refuses every Web Mercator code; OnData's `dati_catastali` gives one point
per parcel, no geometries. A throwaway `<img>` overlay in the scratchpad proved the approach, then it was built in.
Checked in headless Chrome (`scratchpad/dwg/make_ov2_test.py`): the layer is a GraphicsLayer at index 0, the box
sits before `.esri-ui`, the image loads at the view size, its top-left matches `view.toScreen` of the extent corner
to 0 px, a 100-px pan moves it by 100 px and triggers a reload, 30° of rotation gives 0.524 rad, above 1:6,000 it
hides and comes back below, the Layer entry's visibility and opacity drive it, three broken answers call the probe;
no console errors. Looked at in the built-in browser at Pavia 1:1,500 with two Zornade parcels on the AdE
outlines and the Layer panel showing «Catasto AdE». Backup: `geoportale_axpo_pre-catasto-immagine_2026-10-11.html`
(+ README; the file name carries the wrong date, as do the branches `update/2026-10-11`).

**Phase 0 of the ribbon refactor (2026-10-21, decisions D14 and D15 of `docs/plan-ribbon-refactor.md`).** The one
file is cut into `src/` (3 html, 10 css, 23 js files, along its own sections) and `scripts/build.py` writes it back
byte for byte (checked with `cmp` against the file before the cut, and with `--check` since). `tests/run.py` runs
seven harnesses in headless Chrome (`smoke`, `zoom`, `overlay`, `responsive` at three widths, `automations`,
`legend`, `tour`): the scratchpad harnesses of the last weeks, now in the repository with their checks; a run
spoilt by the Esri CDN (modules, basemaps, icons that do not load) is repeated once and the network errors are
listed as a note. Traps: the test script must be inserted at the **last** `</body>` (an earlier one sits inside the
JS string of the undocked window); the Esri Legend renders into its container, so `#legendBox` itself becomes
`.esri-legend`. No visual change; `geoportale_axpo.html` unchanged. Backup:
`geoportale_axpo_pre-fase0-split_2026-10-21.html`.

**Phase 1 of the ribbon refactor (2026-10-21, decisions E1–E9 of `docs/plan-ribbon-refactor.md` §6b).** The
family's design system enters the app: `design/tokens.json` and the 143 icons copied verbatim, `scripts/make_tokens.py`
→ `src/css/10-tokens.css` (the base rules and the dark-theme contrast fixes moved to `src/css/11-base-theme.css`),
`scripts/make_icons.py` → `src/html/05-icons.html` (79 icons used), `src/js/09-icons.js` (`icon()`, `iconsRender()`),
`src/css/15-icons.css`. Every emoji, glyph, Esri icon-font class and hand-drawn inline SVG in the UI became a
family icon (the mapping of E3; `«` → `chevron-left`, the copy button → `copy`, «legenda ↗» → `legend`; in the
`<select>` «＋ Categoria» is a plain `+`, the AdE layer title says «non disponibile» in words); the filled accent
is `--fill`/`--on-fill`, accent text `--link`, focus `--focus`, the light `--warn` is `#d99a00`, the `A` chip is
`--slot-a`; the 25 font sizes under 11 px are 11 px; the sign-in dot is grey when signed out with a `login` icon,
green with `user` when signed in. Checks: the seven harnesses green (one `overlay` run failed on Esri CDN noise and
passed on the rerun), the three `--check`s ok, no console error, 114 `<use>` all resolved, no visible text under
11 px, screenshots at 1366 / 820 / 390 in both themes. Known: at 390 px the `#mbar` labels truncate at 11 px (the
bar is replaced in phase 3). Backup: `geoportale_axpo_pre-fase1-token-icone_2026-10-21.html`.

**Phase 2A-1 of the ribbon refactor (2026-10-21, decision F2 of `docs/plan-ribbon-refactor.md` §6c).** The family's
kit enters the app: `design/kit.css` and `design/geo-kit.css` copied verbatim, `scripts/make_kit.py` → `src/css/12-kit.css`
and `13-geo-kit.css` in the manifest after the tokens. The collision audit found 36 class names on both sides; the ones
that met were the components the family drew from the app, which now use the kit's classes (see §5 *The family's kit*):
57 buttons became `.btn` / `.btn.primary`, the top bar, notices and message boxes, toasts, checkbox labels, hints, link
buttons, the seven modal boxes (`.dialog`), the Tools / context / help menus (`.menu`); `#toolsFab.sheet` → `.fabsheet`.
The fields keep element-level rules with the kit's values; the Raccolta rows wait for 2A-2. No layout change: rails,
drawers and the Tools speed-dial are still there. Checks: the seven harnesses green (one `overlay` run lost to Esri CDN
noise, green on the rerun), the four `--check`s ok, no console error, no button left without its own look, screenshots
at 1366 / 820 / 390 of the tabs, a dialog, toasts, the three menus, light and dark. Backup:
`geoportale_axpo_pre-fase2a1-kit_2026-10-21.html`.

**Phase 2A-2 of the ribbon refactor (2026-10-21, decisions F1 · F3 · F5 · F6 · F10 · F11 · F12 of §6c).** The family's
shell replaces the rails and drawers: a tab strip with the five steps by name, a ribbon with one tool per module (tab
5 with the Raccolta actions as proxies and the two portal widgets as modules), the module panel docked right with
one module at a time, the Raccolta · Layer · Legenda · Base · Luoghi panel docked left with the dock at its foot, strips
for the closed panels, the family's fixed widths (the two drag handles and `--side-w` / `--tools-w` are gone). The
markup of every module, form, list and widget container is unchanged (ids kept); the five former drawers moved as
they were. Narrow screens: one panel at a time up to 1279 px; the phone keeps `#mbar`, re-pointed. The tours point at
`#tabs` and the ptabs; the harnesses (smoke, responsive, legend) read the new shell. Checks: the seven harnesses
green (the legend harness now closes the panel from its button), the four `--check`s ok, no console error, screenshots
at 1366 / 820 / 390, light and dark: every tab, the module switch, tab 5, the left panel and its dock, the strips, the
phone. Known: Base and Luoghi are ptabs only until 2A-3; the tour and guide texts still say «rail» and «cassetto»
(phase 4). Backup: `geoportale_axpo_pre-fase2a2-shell_2026-10-21.html`.

**Phase 2A-3 of the ribbon refactor (2026-10-21, decisions F7 · F8 · F9 of §6c).** The last part of the shell: the map
controls of the family (the Esri column with «Zoom sulla Raccolta» and «Luoghi», the geosearch top right, the basemap
chip bottom right with the Esri BasemapGallery in a popover, the Esri Bookmarks in a popover; Base and Luoghi leave
the left panel), the status bar (pin and coordinates in eight systems, scale, cadastre, mode, snapping, autosave; the
coordinates box over the map is gone) and the project name in the top bar. Checks: the seven harnesses green, the four
`--check`s ok, no console error, the coordinates verified in UTM 32N, Monte Mario and DMS, the popovers, the phone
(bar hidden, chip under the search, Base and Luoghi from the bottom bar) and the dark theme. Backup:
`geoportale_axpo_pre-fase2a3-controlli_2026-10-21.html`.

**Phase 2B-1 of the ribbon refactor (2026-10-21, decisions G1–G9 of §6d).** The group renderer of the ribbon with the ⓘ
popover, the context figure and the compact CSS, and the groups of tabs 1 (Catasto · Sulla mappa · Da un disegno ·
Vincoli · Raccolta), 4 (File · CAD · Immagini · Servizi web · Portale) and 5 (Esporta · Portale AGOL · Progetto ·
Stampa): 28 tools, each a proxy of an existing control (see §5 *The ribbon groups*). The tools of the mockup without a
function of their own are left out (G1). The 25 ⓘ texts of §5 with the two corrections (G2). The panel's selection
buttons and the three file buttons are hidden where the ribbon is (G5). Checks: the seven harnesses green, the four
`--check`s ok, no console error, every tool's state verified against its control (pressed, disabled with the
reason, the dynamic Vincoli name), the popover and «Di più ›», the compact ribbon at 820 px, the phone (ribbon
hidden, panel buttons back). Tabs 2 and 3 keep the module switchers until 2B-2. Backup:
`geoportale_axpo_pre-fase2b1-ribbon_2026-10-21.html`.

**Phase 2B-2 of the ribbon refactor (2026-10-21, G3 · G4 · G5 of §6d).** The groups of tabs 2 (Sito di lavoro · Cosa
disegni · Con quale strumento · Supporto al disegno · Storico · Aree automatiche, 23 tools) and 3 (Oggetti di partenza ·
Geoprocessi · Terreno · Contesto · Tools, 19 tools). The Tools speed-dial, its menu and its floating card leave the map:
the six tools are modules of tab 3 in the right panel, one at a time (`toolSync` from `modShow`, Esc back to the
geoprocesses), still opened by `toolOpen(k)` from the import of a file without a CRS, the align-SR flow and the
context menu. The drawing settings are small tools with their value that focus the field (G3 A); the category tool
follows the chip (name, icon, colour); the five drawing cards, undo / redo, the site buttons, the operation cards,
Dalla mappa / Spuntati of A and the grid buttons are hidden where the ribbon is (G5). On a phone `#modStrip` switches
the modules until phase 3. The ribbon of tab 2 overflowed at 1366 px: the context hides when the groups do not fit
and the small tools' text is capped at 120 px. Tour steps 23, 27, 28 re-targeted; the guide, the «i» windows, the
import and the align-SR messages no longer say «menu Tools». Checks: the seven harnesses green (smoke now checks the
six panes and no `#toolsFab`), the four `--check`s ok, no console error at 1366 / 820 / 375 px in both themes, the
tools' states verified against their controls (pressed, disabled with the reason, the dynamic category), Esc; the
undock could not be tried in the embedded browser (pop-up blocked). Backup:
`geoportale_axpo_pre-fase2b2-ribbon_2026-10-21.html`.

**Phase 3 of the ribbon refactor (2026-10-21, decisions H1–H10 of §6e).** The phone: the family's bottom bar of five
entries and the two panels as sheets over the map (31 % at rest, 84 % with the handle, one at a time), the tab's tools
at the top of the sheet group by group (H2 B), the active module below, the Raccolta sheet with its dock, «Altro» as a
menu, a map mode drops the sheet to rest and «Fine» runs the Esc chain (H5), the sheet of Cerca open at start, the
duplicated panel controls hidden at every width, the mockup's chrome and its 769–1000 px rule (see §5 *The phone*).
Gone: the nine-proxy bar, the full-screen panels, `#modStrip`. Also fixed on the way: the Esc chain closed the tool
module of tab 3 even from another tab, so the first Esc did not stop a drawing. Checks: the seven harnesses green
(responsive at 390 px rewritten for the bar and the sheets), the four `--check`s ok, no console error at 375 / 820 /
1366 px in both themes, the bar, the sheets, the handle, «Altro», «Fine» and the export sheet tried in the embedded
browser. Backup: `geoportale_axpo_pre-fase3-telefono_2026-10-21.html`.

**Review round 1 (2026-10-22, the user's list after phases 1–3).** Messages bottom-left on the map; basemap opacity
and the three AGEA orthophotos in the gallery; the Raccolta with «names on the map» icons in every section header
(drawings, geoprocesses and imports get labels of their own), sections only when populated, the Esporta / Migra / Salva
dock gone (tab 5 keeps them), rename / zoom / category in a row menu (right-click or «…»); the ribbon of Disegna with
the category, the type and the source as menus, Buffer and Rientro as inline numbers, a freehand tool; Geoprocessi with
«Per categoria» as a menu and the context analysis armed from the ribbon (an «Esegui» tool was added and removed the same
day: the panel's Esegui is enough; see §5 *Review round 1*).
Checks: the seven harnesses green (responsive checks the messages against the map's rect), the four `--check`s ok, no
console error at 1366 and 375 px; two first-run failures were CDN noise. Backup:
`geoportale_axpo_pre-giro1-raccolta-ribbon_2026-10-22.html`.

**Review round 1b (2026-10-22): the terrain as PV Predesign.** The Terreno group of tab 3 opens, with the module
«Pendenze da DTM», a terrain ribbon modelled on the Terrain tab of PV Predesign: the gross area and «Analizza le
pendenze», the limit with its values, the exclusion as a switch, the slope classes, the elevations, contour lines with
their step, «Rimuovi terreno»; the panel shows the model, the limit and the slopes in three cards (see §5). Checks: the
seven harnesses green, the four `--check`s ok, a real extraction over an invented gross area near Rome in the embedded
browser (the grid, the exclusion following the limit in both rules, 84 contour lines at 1 m and 18 at 5 m, the views,
back and forth with the geoprocess groups, the removal), no console error. Backup:
`geoportale_axpo_pre-giro1b-terreno_2026-10-22.html`.

**Review round 1c (2026-10-22): «Dividi», and the ribbon «Esegui» removed.** The ninth geoprocess cuts the areas of A
with a line drawn on the map and replaces each one with its parts (see §5 *«Dividi»*); the parcels are never cut. The
ribbon «Esegui» of round 1 was one Esegui too many and went (the panel's stays). The user approved the next step: a
«Terreno» tab between Ricerca particelle and Disegna (plan §6f, T1–T6), to be built after this round is merged. Checks:
the seven harnesses green, the four `--check`s ok, the cut tried in the embedded browser on two areas (one in Web
Mercator, one in WGS 84) with a third left whole, Annulla and Ripeti.

**The Terreno tab (2026-10-22, plan §6f, T1–T6).** The slopes from the DTM, the contour lines and the profile leave tab
3 for a tab of their own, the second one, as in PV Predesign; the extent of the analysis can be drawn on the map and is
kept with the project; tab 4 is back to the geoprocesses with the context, the route, the measure and the unknown
coordinates; the phone bar gets Terreno and moves the geoprocesses under «Altro» (see §5 *The Terreno tab*). Checks: the
seven harnesses green (smoke counts six tabs, responsive eight «Altro» items), the four `--check`s ok, in the embedded
browser a drawn extent analysed for real, saved as a gross area, removed, restored after a reload, the profile, Esc
while drawing the extent, the phone bar and the Terreno sheet; the ribbon fits at 1366 px with the two drawing tools
small. Backup: `geoportale_axpo_pre-scheda-terreno_2026-10-22.html`.

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

**The local metric frame (2026-10-22, phase 4a, plan §6h).** Audit of the measures before the change: the figures
shown were already geodesic, the buffers geodesic, the terrain grid in a local TM, the CAD corrected with two Web
Mercator scale factors (within 0.1 %); the geoprocesses ran with turf in degrees and Semplifica with a fixed 0.0002°.
Done: one frame module shared by everything (§5 *The local metric frame*), turf removed, Semplifica in metres, an
oracle harness with arcpy fixtures (areas, lengths, buffer, union, intersection, difference, hull, cut, simplify,
offset, a shape, a segment, mixed systems, empty results: all within 0.05 %, arcs within 0.2 %), the §1 line about
«one file of ~580 KB» corrected. In the embedded browser: Unisci 13 250 m², Contorno 17 075, Semplifica, Differenza
4 250, Dividi 2 500 + 2 500, a 100 × 50 m rectangle turned by 30° placed with a click (5 000 m², 300 m), the parallel
copy inward (30 × 30 = 900 m²) and the side copy at 10 m, the automatic net area with a 10 m setback (80 × 30 =
2 400 m²), the Tolleranza number in the ribbon mirroring the panel and shown only with Semplifica; the eight
harnesses green; no console error. Backup: `geoportale_axpo_pre-cornice-metrica_2026-10-22.html`.

**G2-1, the small fixes of round 2 (2026-10-22, plan §6i).** Requests R2, T7 and D1 of round 2 (§6g): the Check
Vincoli tool out of tab 1, the captions of Estensione and Limite above their selects, the site group of Disegna
renamed «Collega a sito AREAS» and moved to the right end, the panel head renamed. Checks: the eight harnesses
green, the four `--check` ok, the icon sprite down to 105; in the embedded browser the three tabs at 1366 px. Backup:
`geoportale_axpo_pre-g2-1-ritocchi_2026-10-22.html`.

**G2-2, the Terreno tab (2026-10-22, plan §6j).** Requests T8 and T9 of round 2 (§6g): «Analizza le pendenze» opens a
form popover (a new tool kind of the ribbon engine, `pop()`), the images group «Analisi terreno» with the aspect and the
shaded relief ported from PV Predesign. Checks: the eight harnesses green, the four `--check` ok, the icon sprite at
107; in the embedded browser at 1366 px: the popover with its mirrored fields (the rule switches the rows, a threshold
typed in the form reaches the panel), Esc, a real analysis from the foot button on an invented extent in the
Apennines (132 columns), the four images with their legends and radio-like states, the contour lines, the switch that
makes the exclusion drawing, the stats that follow a threshold change (16,15 → 21,53 ha over the limit); no overflow
at 1366 px; no console error. Backup: `geoportale_axpo_pre-g2-2-terreno_2026-10-22.html`.

**G2-3, Seleziona e modifica (2026-10-23, plan §6k).** Requests D4 and G1 of round 2 (§6g): the selection group of
Disegna as PV Predesign's, on the Raccolta's own selection; the exclusion from the map layers. Checks: the nine
harnesses green (the new `select`: two selected, the box with and without Shift, Unisci 180 × 50 m and its Annulla,
Elimina and its Annulla, a feature of a fake layer twice → one exclusion, a line → linear), the four `--check` ok, the
sprite at 109; in the embedded browser at 1366 px with real clicks: Seleziona by click, Shift+click, a click on nothing
clears, Riquadro by drag, Sposta e ruota and Rimodella from the ribbon with Esc leaving the shapes as they were, Copia
placed by a click, the card and the outline, the banner texts, no overflow, no console error. Found on the way: Esc
from the ribbon did not end an edit (fixed in the Escape chain); an unescaped quote slipped into the build through a
heredoc (a Node syntax check of the script blocks now runs before the harness). Backup:
`geoportale_axpo_pre-g2-3-seleziona_2026-10-23.html`.

**G2-4a, the Raccolta in groups (2026-10-23, plan §6l).** Requests R4 (and the ground of R1) of round 2: the drawer
divided by groups as PV Predesign's Layers panel, five standard groups from the category and the origin, custom
groups saved in the work state and the `.axpo`, the row menu «Sposta in…». Checks: the ten harnesses green (the new
`groups`: the standard blocks and their order, the group of each object, a custom group made, filled, kept in the
state, labelled, renamed and deleted, the category lines of a mixed group, the menus), the four `--check` ok; in the
embedded browser at 1366 px: the blocks with their figures, the category lines, the custom group «Vincoli PRG» with
its object surviving a reload, the ⋯ menu of a group, the row menu with the moves; no console error. Found on the way:
the kit's `.grp` class (fixed with `ggrp`). Backup: `geoportale_axpo_pre-g2-4a-raccolta-gruppi_2026-10-23.html`.

**The six drawing tools in three columns of two (2026-10-23).** Asked by the user after G2-4a: medium icons in two rows
and three columns instead of small icons in two columns. Checks: smoke, responsive and select green; in the browser the
three columns at 1366 px with no overflow (the groups take 1 302 px), the same at 1600 and 1800 px. Backup:
`geoportale_axpo_pre-g2-4a-2-strumenti-medi_2026-10-23.html`.

**G2-4b, the parcel subgroups and the exports by group (2026-10-23, plan §6m).** Requests R1 and R4 of round 2 closed:
subgroups inside Particelle, a gross area per subgroup, the export window by group, KML folders, one shapefile per
group, the XLSX column. Checks: the ten harnesses green (`groups` grew: a subgroup made and filled, its block and
figures, the state, the row and header menus, two gross areas that replace themselves, the `gruppo`/`sottogruppo`
properties, the KML folders and placemarks, the shapefile entries per group, the export sections, the subgroup
deleted), the four `--check` ok; in the embedded browser the subgroup block in the drawer and the export window by
group. Backup: `geoportale_axpo_pre-g2-4b-sottogruppi_2026-10-23.html`.

**G2-5a, «Non classificato» (2026-10-23, plan §6n).** Decision D2 (a) of round 2 closed: the app-only category, the save
modal keeping it out, the net area untouched, the legend, the texts. Checks: the eleven harnesses green (`cats` is new:
the attributes of a new drawing, the net area, the group, the legend, the tag and the editor, the save modal and its
selects, the export label, the work state, an object without a code and one with an unknown code), the four `--check`
ok; in the embedded browser the ribbon menu, a drawing, the drawer tag and the save modal. Backup:
`geoportale_axpo_pre-g2-5a-non-classificato_2026-10-23.html`.

**G2-5b, the custom categories (2026-10-23, plan §6n).** Decision D2 (b) of round 2 closed: the categories made by the
user with fields, the values on the objects, the exports, the cut on the net area, the modal. Checks: the eleven
harnesses green (`cats` grew: a category with three fields, an object with values, the editor inputs, the subtitle, the
export properties and the DBF names, the cut with and without «buffer», the group, the state round trip, a malformed
definition dropped, the modal, the rename, the delete fallback), the four `--check` ok; in the embedded browser the
modal with the form, the editor card with the fields, the ribbon menu. Backup:
`geoportale_axpo_pre-g2-5b-categorie-personali_2026-10-23.html`.

**4b, tests with an oracle and «never crash» (2026-10-23, plan §6o).** The oracle of the figures the user reads (the
parcels, the subgroups, the net area, the strip, the road) and the harness `crash` with 51 junk cases; seven guards
added where the app had none. Checks: the twelve harnesses green (`responsive` reran once for the AGEA layers), the
four `--check` ok; in the embedded browser a bow-tie unioned and a 1e9 m buffer refused, with their messages. Backup
taken before the guards: `geoportale_axpo_pre-4b-test-oracolo_2026-10-23.html`.

**R3-1, the Disegna ribbon (2026-10-23, plan §6p).** Q1–Q3 and Q10 of review round 3: the tools centred, «Cosa disegni»
with the category card, its own fields and the gallery, «Supporto al disegno» and «Aree automatiche» as popovers,
Annulla/Ripeti in «Seleziona e modifica», the module «Progetto», three new icons, Dividi in one place, the Disegna panel
without the chips. Checks: the harnesses green (`layout` is new and runs at 1 214 and 1 366 px), the four `--check` ok;
in the embedded browser the ribbon at both widths, the gallery and the popovers. Backup:
`geoportale_axpo_pre-r3-1-ribbon_2026-10-23.html`.

**R3-2, the measures on the map (2026-10-23, plan §6p).** Q4 of review round 3: one button under the search lens with
a menu (Misura lineare, Misura area, Pulisci), the result in a card beside it; the group «Coordinate» in Geoprocessi.
Checks: the harnesses green (`layout` grew: the button under the lens, the menu and its three items, the card, Esc with
and without the menu, Pulisci, the ×, a click outside, the group; `smoke` counts five tool panes), the four `--check`
ok; in the embedded browser the menu and the card. Backup:
`geoportale_axpo_pre-r3-2-misure_2026-10-23.html`.

**R3-3, the panels (2026-10-23, plan §6p).** Q5, Q6, Q8 and Q7 for the right panel: the Raccolta alone on the left,
Proprietà, Layer and Legenda as tabs on the right, both panels resizable from the inner edge. Checks: the harnesses
green (`layout` grew: the tabs, the panes, the return to «Proprietà», the widths and their limits; `legend` and
`responsive` follow the new places; `smoke` counts one tab on the left and three on the right), the four `--check`
ok; in the embedded browser the two panels and the Layer tab. The undock could not be tried there (popups blocked).
Backup: `geoportale_axpo_pre-r3-3-pannelli_2026-10-23.html`.

**R3-3b, the Raccolta in a window of its own (2026-10-23, plan §6p).** The left half of Q7: the Raccolta undocks like the
right panel, both can be undocked at once, the popovers follow the window; the Legend in an undocked window fixed.
Checks: the harnesses green (`undock` is new: the right panel with its tabs, the Raccolta with a new row, its menu and
Esc, both windows at once, docking back), the four `--check` ok. The runner allows popups now. Backup:
`geoportale_axpo_pre-r3-3b-sgancio-raccolta_2026-10-23.html`.

**R3-4, the Raccolta in order (2026-10-24, plan §6p).** Q9 of review round 3, which closes the round: one scroll, rows
of one line, hectares with a comma, the cadastral area named, «auto», the filter, «Mostra altre», «Comprimi», the editor
in «Proprietà». Checks: the harnesses green (`raccolta` is new), the four `--check` ok; in the embedded browser the
Raccolta with invented parcels, subgroups and drawings, and an object's properties. Backup:
`geoportale_axpo_pre-r3-4-raccolta_2026-10-23.html`.

**5a, the texts of the app (2026-10-26, plan §6q).** The first PR of phase 5: the guide, the two tours, the 28 ⓘ, the
«Di più» windows (two new), «sito AREAS», the old section names, the «?» on a phone, the dead tooltips out, «Da un lato»
back in *Supporto al disegno*. Checks: the harnesses green (`texts` is new, `tour` at 1214 and 1366 px with 8 and 15
steps), the four `--check` ok; in the embedded browser the guide with its icons and «Da un lato» from the popover.
Backup: `geoportale_axpo_pre-5a-testi_2026-10-26.html` (+ README).

**5b, README §4 and §5 rewritten (2026-10-26, plan §6q F9–F11).** §4 describes the app as it is, in the order of the
screen; §5 explains how it works by subject (sixteen of them), each with its traps. No dates and no history in either:
the old text, with the notes of every phase, is word for word in `docs/archive/readme-2026-10-26.md`. Three texts of the
app corrected on the way: the message after «Crea area lorda» named a «Crea area netta» button that no longer exists;
the «Copia» window gave a 0,15 % error that the local metric frame removed; the «Ricerca particelle» window said the
region loads its Check Vincoli map, while it only offers it. Backup: `geoportale_axpo_pre-5b-readme_2026-10-26.html`
and `README_pre-5b-readme_2026-10-26.md`.

**Fix of 2026-10-26: the undocked panels and the web map swap.** Found by reading the code for README §5 and confirmed
by a new harness (`panelswap`) before the fix. (1) With a panel undocked the ribbon looked its controls up in the page
only, so «Crea area lorda», «Aree automatiche», «Esporta», «Migra», «Salva disegni», the geoprocess cards and the module
switches of the right panel did nothing: `qAny` looks in the page and in both windows. (2) A window closed with its ×
lost its panel until a reload (the browser fired only `beforeunload`, and the code read the window as closed): the dock
keeps the moved node and listens to `pagehide`, `beforeunload` and a watchdog. (3) Closing the page left the
Raccolta's window open. (4) A web map swap (the sign-in loading the General Map, or the Mappa selector) left the names
on the map, the terrain extent and the contours on the old map: `wmCarry` moves them. (5) The basemap chip and its
opacity watched the first map only: a reactive watch through the view. Backup:
`geoportale_axpo_pre-fix-sganci-mappa_2026-10-26.html`.

**Zoom (2026-10-26, the user's request).** «Aggiungi alla Raccolta» always frames the parcels just added, also when they
are already in view (before, the map moved only if they were out of view); a click on the map still never moves it. The ⋯
of a group of the Raccolta has «Zoom sul gruppo», that of a parcel subgroup «Zoom sul sottogruppo», that of Particelle
«Zoom sulle particelle». The frame keeps at least about 400 m around small objects. Checked in the embedded browser and by
the `zoom` harness (on the frame asked for).

**5c, `docs/architecture.md` (2026-10-26, plan §6q F12).** The last PR of phase 5: the map of the Geoportale for people
who do not read the code, on the model of PV Predesign's — the app in one picture, the screen and the files behind it
(the file names are historical: the table says where each part really lives), how the data travel from the parcel to
the portal and the `.axpo`, the generated files, «if you touch X, Y breaks», how every part is tested and what no test
covers, where to look for a question. Linked from the intro of this README and from §8.

**7a-1, one command, one Annulla (2026-10-27, plan §6s V1–V8).** Three defects of Annulla found with a temporary harness
and fixed: (U1) after a save to IT - Site Features, Annulla brought the saved objects back as «never saved», so the next
save would have duplicated them on the portal; (U2) a drawing with a buffer could not be undone (its strip came back
and was recorded again); (U3) parcels, groups, subgroups, own categories, net-area settings, the terrain extent and the
site were not in the history (Annulla after adding a parcel undid the last drawing). Now a photo of the whole work is
taken when it settles after a command, with the command's name, and the portal identity survives Annulla. New file
`src/js/29b-commands.js`; new harness `undo`. Backup: `geoportale_axpo_pre-7a1-annulla_2026-10-26.html`.

**7a-2, the registry of the commands (2026-10-27, plan §6s V6).** One entry per tool of the ribbon, built from the
ribbon's own description (one source: nothing to keep in step), with tab, group, name, icon, tooltip, kind and
shortcuts; `cmdRun` runs any of them as its button does. The keyboard shortcuts became data: one handler instead of
two, and the tooltips of Annulla, Ripeti, Salva progetto, Salva come… and Apri progetto… show their keys. Ctrl+Maiusc+Z
is Ripeti too. Phase 7a is complete. Backup: `geoportale_axpo_pre-7a2-registro_2026-10-27.html`.

**7b-1, what can be done to what (2026-10-27, plan §6t W1–W3).** The rules «which action on which kind of object»,
scattered in the code, became one table (`RULES` in `22c-rules.js`) that the menus and the tools read, each refusal
with its reason. No rule was relaxed; where two places of the code disagreed, the stricter one won:
(a) the ⋯ menu of a row lists only what its kind allows: no *Rinomina*, *Proprietà* or *Elimina* on an automatic strip
(*Elimina* used to be offered and then refused), no *Rinomina* or *Proprietà* on the automatic net area (both used to
work and were undone by the next recalculation); (b) the colour swatch of a strip and of the net area is disabled
(the automation repainted them); (c) the editor in *Proprietà* refuses the automatic objects; (d) a click with
*Seleziona* on a parcel, an owners' parcel, a strip or the net area says why it cannot be selected and keeps the
selection (it used to clear it); (e) *Dividi* in the geoprocesses refuses strips, the net area and owners' parcels
with the reason (it used to split them, and the automation rebuilt them at once); (f) *Copia* made a copy that kept
the automatic link, the portal identity and the owners' data — a copy of a strip became a second, false automatic
strip, two objects shared one portal id, and a copy of an owners' parcel carried the personal data into a drawing
that could be saved to the portal: the copy is now an ordinary drawing. Backup:
`geoportale_axpo_pre-7b1-regole_2026-10-27.html`.

**7b-2, the profiles Base / Avanzato (2026-10-27, plan §6t W4–W8).** Every command of the registry has a level; the
profile Base, for the project managers, hides the advanced ones (31 commands in 11 groups: 8 groups whole, and 7
commands of *Seleziona e modifica* and *Portale AGOL*), Avanzato shows everything and stays the default, so nothing
changes for today's users. Chosen at the foot of the «?» menu and of the guide (the «?» of a phone), remembered in the
browser; a link with `?profilo=base` opens in Base. Two details beyond the plan: a tab whose default module is hidden
opens on its first visible one (Geoprocessi e analisi on *Analisi di contesto*), and in Base the last step of a tour
and the «?» tooltip say where the other tools are. *Riquadro*, not named in W5, stays in Base with *Seleziona*. The
panels are unchanged: in Base the Disegna panel still shows *Collega a sito AREAS COLLECTION*. Backup:
`geoportale_axpo_pre-7b2-profili_2026-10-27.html`.

**7c, the search Ctrl K (2026-10-27, plan §6u X1–X8).** A box at the top centre of the map, opened by Ctrl K or from
the «?» menu, that finds as one types the objects of the Raccolta and the commands of the ribbon (those the profile
shows); a pick does what the object's row or the command's button does. As PV Predesign's, with the commands added.
Found while doing it: `cmdRun` ignored a `CMD_RUN` command that is not a ribbon tool, so the 7a-2 rule «a new
shortcut goes into `CMD_KEYS` and `CMD_RUN`» did not work for such a command; fixed. With it, the user's request of
the same day: in Base the card *Collega a sito AREAS COLLECTION* of the Disegna panel is hidden too
(`PROF_ADV_PANELS`). Backup: `geoportale_axpo_pre-7c-ricerca_2026-10-27.html`.

**8, the status bar (2026-10-27, plan §6v Y1–Y6).** From PV Predesign: with a terrain model, the elevation and the slope
under the pointer (red over the limit); the scale as a menu of the usual scales (no lock); «Salvato nel browser ·
14:32» instead of a fixed text, and a red state that stays while the browser is full (before, a message said it once).
To keep a chosen scale exact the view no longer snaps to the zoom levels of the tiles (`snapToZoom:false`, as PV
Predesign): the wheel and the pinch may stop between two levels. The phone is unchanged. Backup:
`geoportale_axpo_pre-8-barra-stato_2026-10-27.html`.

**9, the `.axpo` format written (2026-10-27, plan §6w P1–P6).** `docs/axpo-format.md` describes the project file field
by field — type, default, who writes and who reads it (the Geoportale, PV Predesign) — and the rules of the version
(`versione` stays 1: fields were only added). Opening a file goes through `projNormalize`, which gives every missing
value its default and reads an unknown category as «Non classificato» keeping its code. *Salva una copia da
condividere…* in the Progetto module writes the project without the owners' data and the user's name. A request to
the PV Predesign session is in `COORDINAMENTO.md`. Backup: `geoportale_axpo_pre-9-formato-axpo_2026-10-27.html`.

**Fix of 2026-10-27: the flaky `undock` harness (tests only).** `undock` failed about one run in three, on the new and
on the older builds alike: every test page is a first visit, so the «Inizia qui» tour started by itself 1.8 s after
load and, in the middle of the harness, docked the right panel. The runner marks the tour as seen before the app
starts, keeps the console errors next to the result (a harness that died early because the CDN did not serve the SDK
is now repeated as network noise) and treats the AGEA orthophotos not loading as noise (the user's decision);
`undock` waits for the state each step checks. Found and fixed by a separate session (PR 42). The app is unchanged.

**Fix of 2026-10-28: the owners' file invisible in the Raccolta (reported by the user).** An owners' file imported
with no parcel searched was counted in the badge («99+») while the Raccolta said it was empty: its rows live at the
end of the Particelle block, which since 22/10 shows only with parcels in it, and counted the searched ones only. The
block now counts both (`secParcelsN`, in `render` and `renderGeoms`); the line *Per intestatario* has a bin that
removes the whole file (`ownRemoveAll`, one confirm, one Annulla «Togli intestatari»), and the guide says where the
file goes. «Nomi sulla mappa» on that line was in capitals: the shell's rule for labels (`label:not(.chk)`) beat
`.gownbar label`; the label is now a `.chk`. A plain GeoJSON was not affected (it goes to «Altro»). New harness `owners`. Backup:
`geoportale_axpo_pre-fix-intestatari_2026-10-28.html`.

**2026-10-30: popups on the web maps' vector tiles (decisions V1–V6, asked by the session «Pubblicazione layer
campania», approved by the user).** The SDK opens no popup on a `VectorTileLayer`, but tiles built by ArcGIS Online with
«Configura attributi» carry the fields and `view.hitTest` returns them. On a click with popups on (V1), the map's
vector tile layers shown at that scale, never the basemap (V2), are hit-tested; their elements join the native popup
of the same click, after its features (V3), with title and fields from the item's `popup.json` or, without it, the
source layer's name and its fields (V4); one line in the guide, no new control (V5). No highlight: the tiles carry no
shape. First use: CV Campania, «Vincoli paesaggistici (PPR) › Vista d'insieme» (item `b71293c2`, 25 source layers
configured). Tried in the built-in browser with a public Esri vector tile service added as a layer (a real click on a
road: «Road», 1 of 2); the real map needs the sign-in: item 1.6 of the checklist
`../reports/collaudo_login_vero_2026-10-27.md` (V6). New harness `vtpopup`; `owners` now waits for the history's photo
before its bin (it failed once under the full suite: tests only). Backup:
`geoportale_axpo_pre-popup-tasselli_2026-10-30.html`.

## 8. Repository contents

```
geoportale_axpo.html       the whole application, built from src/ (deployed as is)
README.md                  this file
src/                       the sources: html/, css/, js/ and manifest.txt (the build order)
src/js/16b-geo-frame.js    the local metric frame (phase 4a): every geometric operation in true metres
src/js/22a-groups.js       the groups of the Raccolta (G2-4a): the model, the custom groups, the blocks of the drawer
src/js/22b-parcel-groups.js  the parcel subgroups (G2-4b): the model, the menus, the nested blocks of Particelle
src/js/22c-rules.js        what can be done to what (7b-1): the kinds of object, the table of rules, can()
src/js/19b-custom-cats.js  the custom categories (G2-5b): the model, the values sf_ext, the modal «Le mie categorie»
src/js/23b-select-edit.js  Seleziona e modifica (G2-3): the selection on the map, the edit tools, the exclusion from the layers
src/js/29b-commands.js     the commands (7a): the name each action gives to its Annulla; the registry and the shortcuts; the profiles (7b-2)
src/js/29c-search.js       the search Ctrl K (7c): the objects of the Raccolta and the commands of the ribbon
design/                    the family's design system as copied: tokens.json, icons/*.svg (143), kit.css, geo-kit.css
scripts/build.py           writes geoportale_axpo.html from src/; --check says whether it is up to date
scripts/make_tokens.py     writes src/css/10-tokens.css from design/tokens.json; --check
scripts/make_icons.py      writes src/html/05-icons.html with the icons the sources use; --check, --list
scripts/make_kit.py        writes src/css/12-kit.css and 13-geo-kit.css from design/; --check
scripts/make_geo_fixtures.py  writes tests/fixtures/geo_expected.json with arcpy (the ArcGIS Pro Python): the oracle of the geo harness
tests/                     run.py and harness/*.py: headless checks, Python + Chrome (tests/README.md); fixtures/ = the expected measures, made once by scripts/make_geo_fixtures.py with arcpy (developer-only: the app and the runner never need ArcGIS Pro)
docs/                      architecture.md: the map of the app for people who do not read the code; axpo-format.md: the
                           project file, field by field (phase 9);
                           plan-ribbon-refactor.md: the refactor plan and its decisions; archive/readme-2026-10-26.md:
                           §4 and §5 as they were before 5b, word for word (the dated notes of every feature and phase)
```

There is no CI yet: the suite runs on the developer's PC before each pull request. The portal (OAuth, the
corporate layers) is only exercised by hand in the browser.

**Files touched per phase** (one line per file created or changed, the rule since 2026-10-22; the earlier phases are
in §7):

- **4a · The local metric frame (2026-10-22).** New: `src/js/16b-geo-frame.js`, `scripts/make_geo_fixtures.py`,
  `tests/harness/geo.py`, `tests/fixtures/geo_expected.json`. Changed: `src/manifest.txt` (the new module before
  `17-slope.js`); `src/js/11-map-model.js` (`SpatialReference` and `projectOperator` in `esriMods`, the engine loaded
  after the view); `17-slope.js` (`slpWkt` delegates to `geoFrameWkt`); `20-export-geoproc.js` (`gpSlotPolys`,
  `gpAddResult` takes a geometry, `gpSimpTol`, every operation through the frame, `gpSameSR`, `dpPts` and
  `simplifyM` gone); `21-legend-auto.js` (gross area, strips, setback and net area through the frame);
  `22-raccolta-ui.js` (the dissolve of the parcels); `23-draw-cad.js` (`mercK`/`mercK2` gone, `shpGeom`, `drwEdgeAt`
  and `drwDoOffset` in the frame); `30-ui-glue.js` (the `simp-tol` number); `src/html/00-head.html` (turf out);
  `src/html/20-app-body.html` (the Tolleranza box of Semplifica); `tests/README.md`; this README (§1, §5, §7, §8).
- **G2-1 · The small fixes of round 2 (2026-10-22).** Changed: `src/js/30-ui-glue.js` (the Vincoli group and the `cv*`
  helpers gone, `stacked` selects, the site group); `src/css/20-shell.css` (`.rsel-tool.stacked`);
  `src/html/20-app-body.html` (the panel head of the site); `src/html/05-icons.html` (regenerated, `shield` gone);
  this README (§4, §5, §7, §8).
- **G2-2 · The Terreno tab (2026-10-22).** Changed: `src/js/30-ui-glue.js` (the form popover engine `openForm`/`formCopy`/
  `formWrite`/`formSync`, `slpForm`, the Terreno groups, the ⓘ texts, the icons list); `src/js/17-slope.js`
  (`slpAspectColour`, the views `a` and `h` in `slpCanvas`, the legend, the two buttons, the project field);
  `src/css/20-shell.css` (`#rformPop` and its fields); `src/html/20-app-body.html` (`#rformPop`, the two view buttons,
  the card texts); `src/html/05-icons.html` (regenerated: `aspect`, `sun`); this README (§5, §7, §8).
- **G2-3 · Seleziona e modifica (2026-10-23).** New: `src/js/23b-select-edit.js`, `tests/harness/select.py`. Changed:
  `src/manifest.txt`; `src/js/30-ui-glue.js` (the group «Seleziona e modifica», the six drawing tools small, the
  «Esclusione da layer» tool, `selNone`, the ⓘ texts, the mode banner, the Escape chain, the icons list);
  `src/js/11-map-model.js` (the `select-box` and `excl-box` purposes); `src/js/22-raccolta-ui.js` (the checkbox and
  `refreshDl` call `selRender`); `src/js/20-export-geoproc.js` (`gpSay`); `src/js/27-align-sr.js` (`cadModesOff` ends
  the modes); `src/html/20-app-body.html` (`#selCard`); `src/css/50-panels.css` (the card); `src/html/05-icons.html`
  (regenerated: `cursor`, `move`, `merge`, `copy`); `tests/README.md`; this README (§4, §5, §7, §8).
- **G2-4a · The Raccolta in groups (2026-10-23).** New: `src/js/22a-groups.js`, `tests/harness/groups.py`. Changed:
  `src/manifest.txt`; `src/html/20-app-body.html` (the three origin sections replaced by `#grpList`, `#grpFoot` and
  the Immagini block; `#listOwn` under Particelle); `src/js/22-raccolta-ui.js` (`renderGeoms` by group, the section
  helpers by group id, the labels by group, the row menu); `src/js/14-state-project.js` (`groups` in the state,
  `grp` in the serialised objects); `src/js/24-import-files.js` (the owners under Particelle);
  `src/js/20-export-geoproc.js`, `21-legend-auto.js`, `28-export-window.js` (the messages name the groups);
  `src/css/50-panels.css` (the blocks); `tests/run.py` (a page with no result is repeated once); `tests/README.md`;
  this README (§5, §7, §8).
- **Medium drawing tools (2026-10-23).** Changed: `src/js/30-ui-glue.js` (the `medium` size stacks, the six tools);
  `src/css/20-shell.css` (`.tool.medium` in the ribbon); this README (§5, §7, §8).
- **G2-4b · The parcel subgroups and the exports by group (2026-10-23).** New: `src/js/22b-parcel-groups.js`. Changed:
  `src/manifest.txt`; `src/js/18-parcels.js` (`rowOf`, the subgroup blocks, the menus, the labels); `src/html/20-app-body.html`
  (the ⋯ of Particelle); `src/js/21-legend-auto.js` (`sfGrossMake` → `sfGrossFrom`, one area per subgroup);
  `src/js/22-raccolta-ui.js` (`gruppo`/`sottogruppo` on the parcels, `toKML` with folders, `exGroupOrder`);
  `src/js/20-export-geoproc.js` (`gruppo` on the graphics' features); `src/js/28-export-window.js` (`exSections`,
  the XLSX column, `shpZipByGroup`, `exSlug`, the DBF field); `src/js/14-state-project.js` (`parcelGroups`, `sf_pg`);
  `src/css/80-proto-ux.css` (the export window rules keyed on `#exList`);
  `src/css/50-panels.css` (`.psub`); `tests/run.py` (the fetch noise); `tests/harness/groups.py`; `tests/README.md`;
  this README (§5, §7, §8).
- **G2-5a · «Non classificato» (2026-10-23).** New: `tests/harness/cats.py`. Changed: `src/js/11-map-model.js` (the
  category, a text); `src/js/19-undock-sitefeatures.js` (`sfOf`, `sfNewAttrs`, the editor, the save modal with
  `sfDefPrev` / `sfEffCat` / `sfStaysOut`, the save routine, `sfAttrsOut`); `src/js/21-legend-auto.js` (the legend row);
  `src/js/22-raccolta-ui.js`, `src/js/22a-groups.js`, `src/js/23b-select-edit.js`, `src/js/23-draw-cad.js` (the texts);
  `src/js/30-ui-glue.js` (the icon, the banner, the ⓘ); `tests/README.md`; this README (§5, §7, §8).
- **G2-5b · The custom categories (2026-10-23).** New: `src/js/19b-custom-cats.js`. Changed: `src/manifest.txt`;
  `src/js/19-undock-sitefeatures.js` (`sfCat`, `sfCatsRebuild`, the editor select and fields, the save selects, the
  subtitle); `src/js/21-legend-auto.js` (the legend list, `sfCutsCat`, `sfCutDist`); `src/js/22-raccolta-ui.js` (the
  colour popover); `src/js/22a-groups.js` (the order, `grpDefault`); `src/js/20-export-geoproc.js` (the properties);
  `src/js/28-export-window.js` (`shpDbfProps`); `src/js/14-state-project.js` (`sf_ext`, `customCats`, `catSetState`);
  `src/js/30-ui-glue.js` (the menu entries, the icon, the ⓘ); `src/html/30-modals.html` (the modal);
  `src/html/20-app-body.html` (the button); `src/css/40-map-chrome.css`, `src/css/80-proto-ux.css`;
  `tests/harness/cats.py`; `tests/README.md`; this README (§5, §7, §8).
- **4b · Tests with an oracle and «never crash» (2026-10-23).** New: `tests/harness/crash.py`. Changed:
  `scripts/make_geo_fixtures.py` (the figures), `tests/fixtures/geo_expected.json` (regenerated), `tests/harness/geo.py`
  (the figures); the guards: `src/js/16b-geo-frame.js` (`geoFinite`, `geoRun`, `geoOut`), `src/js/20-export-geoproc.js`
  (`gpEmptyResult`, `gpAddResult`, `GP_DMAX`, `makeBuffer`, the slots in `gpRun`), `src/js/21-legend-auto.js`
  (`sfCutDist`), `src/js/23-draw-cad.js` (`drwAddGraphic`), `src/js/14-state-project.js` (`applyWork`),
  `src/js/24-import-files.js` (`gjFinite`, `impAdd`); `tests/README.md`; this README (§5, §7, §8).
- **R3-1 · The Disegna ribbon of round 3 (2026-10-23).** New: `tests/harness/layout.py`, `design/icons/link.svg`,
  `height.svg`, `image-pin.svg`. Changed: `src/js/30-ui-glue.js` (the Disegna groups, `catHas`, `catMenu`, `aidLabel` /
  `aidForm`, `netLabel` / `netForm`, the card in `toolBtn` and `ribbonSync`, the gallery in `openMenu`, `data-press` in
  the form popover, `cls` on a group, the module «Progetto», the icons, the ⓘ texts); `src/js/19-undock-sitefeatures.js`
  (`#sfCatNow`); `src/js/14-state-project.js` (`projModRender`, `projSavedAt`); `src/html/20-app-body.html` (the
  category line of the panel, `#modProj`, the split card gone); `src/html/05-icons.html` (regenerated, 112 icons);
  `src/css/20-shell.css`; `tests/README.md`; this README (§5, §7, §8).
- **R3-2 · The measures on the map (2026-10-23).** New: `design/icons/measure-area.svg`. Changed:
  `src/html/20-app-body.html` (the toolpane «Misura» gone; `#msStack` with `#msBtn` and `#msMenu`, and `#msCard`, under
  the lens);
  `src/js/30-ui-glue.js` (the module and the two tools gone, the group «Coordinate», its ⓘ, the icon list);
  `src/js/16-layers-webmap.js` (`measureUi`, `msMenuSet`, `msMenuOpen`); `src/js/13-tools-select.js` (the button, the
  menu, Pulisci, the ×); `src/js/11-map-model.js` (the
  watch); `src/css/40-map-chrome.css`; `src/html/05-icons.html` (113 icons); `tests/harness/layout.py`,
  `tests/harness/smoke.py`, `tests/harness/automations.py` (waits for the projection engine); `tests/README.md`;
  this README (§5, §7, §8).
- **R3-3 · The panels (2026-10-23).** Changed: `src/html/20-app-body.html` (the left tabs, the two drawers moved into
  `#side`, `#sideTabs`, the two `.pgrip`); `src/js/30-ui-glue.js` (`openPanel`, `spanelSet`, the tabs' handlers, `modShow`,
  `mbarSync`, `pwMax` / `pwSet` / `pwApply` and the grips); `src/js/21-legend-auto.js` (`legendOpen`); `src/js/32-tour.js`
  (the step on the panels); `src/css/20-shell.css`, `src/css/85-status-phone.css`; `tests/harness/layout.py`,
  `tests/harness/legend.py`, `tests/harness/responsive.py`, `tests/harness/smoke.py`; `tests/README.md`; this README (§5, §7,
  §8).
- **R3-3b · The Raccolta in a window of its own (2026-10-23).** New: `tests/harness/undock.py`. Changed: `src/js/10-config.js`
  (`rcWin`, `byId`, `rcDocNow`, `rcQ`, `rcQA`); `src/js/19-undock-sitefeatures.js` (two windows; `rcUndock`, `rcDock`,
  `rcButtons`, `undockPopsHome`); `src/js/30-ui-glue.js` (`popHome`, `menuPlace`, the popovers, `openPanel`, `leftClose`, the
  button); `src/js/18-parcels.js`, `src/js/22-raccolta-ui.js`, `src/js/28-export-window.js` (the queries); `src/js/21-legend-auto.js`,
  `src/js/16-layers-webmap.js` (the containers); `src/html/20-app-body.html` (`#leftUndock`); `src/css/20-shell.css`,
  `src/css/85-status-phone.css`; `tests/run.py` (`--disable-popup-blocking`); `tests/harness/crash.py` (the network noise of a
  case: the AGEA layers, the Esri texts, an anonymous cross-origin error); `tests/README.md`; this README (§5, §7, §8).
- **R3-4 · The Raccolta in order (2026-10-24).** New: `tests/harness/raccolta.py`. Changed: `src/js/10-config.js`
  (`fmtHa2`); `src/js/22-raccolta-ui.js` (`rcFig`, `geomRow`, the figures, `rcListsPost`, `rcCollapseAll`, the toolbar);
  `src/js/18-parcels.js` (the parcel row, the header line); `src/js/19-undock-sitefeatures.js` (`sfEditInProps`);
  `src/js/23b-select-edit.js` (the card with the editor); `src/js/13-tools-select.js` (the «Ultimo disegno» tag);
  `src/js/24-import-files.js`, `src/js/21-legend-auto.js` (figures, the plural); `src/html/20-app-body.html` (the
  toolbar); `src/css/50-panels.css`; `tests/README.md`; this README (§5, §7, §8).
- **5a · The texts of the app (2026-10-26).** New: `tests/harness/texts.py`. Changed: `src/html/30-modals.html` (the
  guide, `#uiTip` out); `src/js/32-tour.js` (the tours, the «?» on a phone, the old tooltips out); `src/js/30-ui-glue.js`
  (`RIBBON_INFO`, the `more` links, the windows, «Da un lato»); `src/js/16-layers-webmap.js` (the windows of the context
  tools); `src/html/20-app-body.html` (module titles, messages, «Aggiungi alla Raccolta»); `src/js/27-align-sr.js`,
  `src/js/20-export-geoproc.js` (messages); «sito AREAS» in `src/js/11-map-model.js`, `13-tools-select.js`,
  `14-state-project.js`, `17-slope.js`, `19-undock-sitefeatures.js`; `src/css/60-tour.css` (`#uiTip` out);
  `tests/harness/tour.py`; `tests/README.md`; this README (§3, §5, §7, §8).
- **5b · README §4 and §5 (2026-10-26).** New: `docs/archive/readme-2026-10-26.md`. Changed: this README (intro,
  §4 and §5 rewritten, §7, §8); `src/js/21-legend-auto.js` (the message after «Crea area lorda»); `src/js/30-ui-glue.js`
  (the «Copia» and «Ricerca particelle» windows); `tests/README.md` (the `smoke` row).
- **Fix · Undocked panels and web map swap (2026-10-26).** New: `tests/harness/panelswap.py`. Changed:
  `src/js/10-config.js` (`qAny`); `src/js/30-ui-glue.js` (the ribbon's lookups, the basemap watch);
  `src/js/19-undock-sitefeatures.js` (docking on close, the page closing both windows); `src/js/16-layers-webmap.js`
  (`wmCarry`); `tests/README.md`; this README (§5, §7, §8).
- **Zoom · groups, subgroups and the search (2026-10-26).** Changed: `src/js/18-parcels.js` (`zoomGraphics`,
  `zoomParcelList`, `render('frame')`); `src/js/22a-groups.js`, `src/js/22b-parcel-groups.js` (the menu items);
  `src/html/30-modals.html` (the guide); `tests/harness/zoom.py`; `tests/run.py` (the anonymous `Script error. @0` of a
  cross-origin script is network noise); `tests/README.md`; this README (§4, §5, §7, §8).
- **5c · `docs/architecture.md` (2026-10-26).** New: `docs/architecture.md`. Changed: this README (intro, §5 wording on
  the AGOL Axpo folder, §7, §8).
- **7a-1 · One command, one Annulla (2026-10-27).** New: `src/js/29b-commands.js`, `tests/harness/undo.py`. Changed:
  `src/js/23-draw-cad.js` (the history engine); `src/js/14-state-project.js` (`saveStateDebounced` → `histSoon`);
  `src/js/22-raccolta-ui.js` (`refreshDl` no longer records); `src/js/18-parcels.js`, `src/js/13-tools-select.js`,
  `src/js/19-undock-sitefeatures.js`, `src/js/11-map-model.js` (names of inline commands); `src/manifest.txt`;
  `tests/README.md`; `docs/architecture.md`; this README (§3, §4, §5, §7, §8).
- **7a-2 · The registry of the commands (2026-10-27).** New: `tests/harness/commands.py`. Changed:
  `src/js/29b-commands.js` (the registry, the shortcuts); `src/js/30-ui-glue.js` (`window.SHELL`, the shortcut in the
  tooltips); `src/js/12-zornade.js`, `src/js/13-tools-select.js` (their keyboard handlers removed); `tests/README.md`;
  `docs/architecture.md`; this README (§3, §4, §5, §7, §8).
- **7b-1 · What can be done to what (2026-10-27).** New: `src/js/22c-rules.js`, `tests/harness/rules.py`. Changed:
  `src/js/22-raccolta-ui.js` (the row menu, the swatch and × by kind); `src/js/19-undock-sitefeatures.js`
  (`sfEditInProps`); `src/js/23b-select-edit.js` (`selEditable`, `selClick` says why); `src/js/20-export-geoproc.js`
  (Dividi); `src/js/23-draw-cad.js` (`drwCopyAdd` drops the internal fields); `src/manifest.txt`; `tests/README.md`;
  `docs/architecture.md`; this README (§3, §4, §5, §7, §8).
- **7b-2 · The profiles Base / Avanzato (2026-10-27).** New: `tests/harness/profiles.py`. Changed:
  `src/js/29b-commands.js` (the levels, the profile); `src/js/30-ui-glue.js` (`groupsOf`, `modFirst`, the event, the
  first focus of a window); `src/js/32-tour.js` (the profile buttons, the tours); `src/html/30-modals.html` (the
  profile in the «?» menu and the guide); `src/css/60-tour.css`; `tests/README.md`; `docs/architecture.md`; this
  README (§3, §4, §5, §7, §8).
- **7c · The search Ctrl K (2026-10-27).** New: `src/js/29c-search.js`, `tests/harness/search.py`. Changed:
  `src/js/29b-commands.js` (Ctrl K, `cmdRun`, `PROF_ADV_PANELS`); `src/js/22-raccolta-ui.js`, `src/js/18-parcels.js`
  (`row._obj`); `src/js/32-tour.js` (the item of the «?» menu); `src/html/20-app-body.html` (the box);
  `src/html/30-modals.html` (the menu item, the guide); `src/css/60-tour.css`; `src/manifest.txt`;
  `tests/harness/profiles.py`; `tests/README.md`; `docs/architecture.md`; this README (§3, §4, §5, §7, §8).
- **8 · The status bar (2026-10-27).** New: `tests/harness/statusbar.py`. Changed: `src/js/31-measure-coords.js`
  (terrain, scale menu, autosave); `src/js/17-slope.js` (`slpAt`); `src/js/14-state-project.js` (`saveState` →
  `sbSaveSet`); `src/js/11-map-model.js` (`snapToZoom:false`); `src/html/20-app-body.html` (the bar);
  `src/html/30-modals.html` (the guide); `src/js/32-tour.js` (the step on the bar); `src/css/40-map-chrome.css`;
  `tests/README.md`; `docs/architecture.md`; this README (§3, §4, §5, §7, §8).
- **9 · The `.axpo` format written (2026-10-27).** New: `docs/axpo-format.md`, `tests/harness/axpo.py`. Changed:
  `src/js/14-state-project.js` (`projNormalize`, `projBuild` with `share`, `projShare`, `sf_cat_orig`);
  `src/html/20-app-body.html` (the button in the Progetto module); `src/html/30-modals.html` (the guide);
  `tests/README.md`; `docs/architecture.md`; `../COORDINAMENTO.md` (the request to PV Predesign); this README (§3, §4,
  §5, §7, §8).
- **Fix · The flaky `undock` harness (2026-10-27, tests only).** Changed: `tests/run.py` (the tour marked as seen in the
  hook, the console errors kept next to the result, the AGEA orthophotos in `NOISE`); `tests/harness/undock.py`
  (`until` instead of fixed waits, the tour check); `tests/README.md` (PR 42); then `docs/architecture.md` and this
  README (§5, §7, §8).
- **Fix · The owners' file invisible in the Raccolta (2026-10-28).** New: `tests/harness/owners.py`. Changed:
  `src/js/22-raccolta-ui.js` (`secParcelsN`, the block shown from `renderGeoms`); `src/js/18-parcels.js` (`render`);
  `src/js/24-import-files.js` (the bin of *Per intestatario*, `ownRemoveAll`, the label a `.chk`); `src/css/60-tour.css`
  (`.gownbar label.chk`); `src/html/30-modals.html` (the guide);
  `tests/README.md`; this README (§3, §4, §5, §7, §8).
- **Vector tile popups (2026-10-30).** New: `tests/harness/vtpopup.py`. Changed: `src/js/16-layers-webmap.js` (`vtp*`);
  `src/js/11-map-model.js` (`vtpWire`); `src/html/30-modals.html` (the guide); `tests/harness/owners.py` (waits for the
  history's photo before the bin); `tests/README.md`;
  `docs/architecture.md`; `../reports/collaudo_login_vero_2026-10-27.md` (item 1.6); this README (§3, §4, §5, §7, §8).
