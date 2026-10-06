# El Niño Vulnerability Index (ENVI) – Cordillera Administrative Region (CAR)

A local web app that turns three input datasets into a normalized vulnerability index for the
77 municipalities of CAR. It also produces a choropleth map in the DA-RFO-CAR print layout and a
presentation-ready infographic.

The sample data covers the outlook **October 2026 – March 2027**, issued **September 2026**. The reporting
period is a setting, so later months only need new inputs and new dates. A companion
**upload portal for Vercel** lives in [`web/`](web/README.md). Field offices use it to download the input formats
and submit their files, reporting period and agricultural recommendations.

---

## Quick start (Windows)

1. Install **Python 3.11 or newer** from python.org. Tick *"Add python.exe to PATH"*.
2. *(Recommended)* Install **QGIS 3.x** (for example, the LTR from qgis.org) so maps are rendered with
   PyQGIS using the GIS template. Without QGIS the app falls back to matplotlib automatically.
3. Double-click **`run.bat`**.
   The first run creates `.venv` and installs `requirements.txt`, which takes a few minutes.
   The app then opens at <http://127.0.0.1:8000>.

Manual alternative:

```bat
py -3 -m venv .venv
.venv\Scripts\python -m pip install -r requirements.txt
.venv\Scripts\python app.py
```

Headless run (no browser), using the files in `data/input/`:

```bat
.venv\Scripts\python -m envi
.venv\Scripts\python -m envi --weights 0.2,0.4,0.4 --classes jenks --layout qpt
.venv\Scripts\python -m envi --reporting-month 2026-10 --period 2026-11:2027-04 --crops-as-of 2026-10-15 ^
    --drought "new\drought forecast.csv" --recommendations my_recommendations.txt
```

Tests:

```bat
.venv\Scripts\python -m pytest
```

If QGIS is installed somewhere unusual, point the app at its PyQGIS launcher before starting:
`set QGIS_PYTHON=D:\QGIS\bin\python-qgis-ltr.bat`

---

## The wizard

| Step | What happens |
|---|---|
| **1 Upload** | Set the **reporting period**: reporting month (as of), data period from → to, and the standing-crops as-of date. **Import portal submission.json** fills these and the recommendations from a portal submission. The sample files in `data/input/` load by default. You can upload replacements (CSV/XLSX) that use the same templates; **Download input templates** gives blank templates pre-filled with every GIS area name. Each file is validated (required columns, categories, numbers) and every row is matched to the CAR boundary layer. Unmatched, fuzzy and cross-province matches are listed with a drop-down so you can assign or confirm them. GIS areas missing from a file are listed too. |
| **2 Review & Normalize** | Edit the weights (they must sum to 1), the classification method (equal intervals, natural breaks or quantiles), the drought scaling and the missing-data policy. A sortable, filterable table shows raw values, normalized values, ENVI, class and rank, with **Export CSV / Excel**. |
| **3 Map** | An interactive Leaflet preview shows area, ENVI, class and indicator values on hover. **Generate** renders the print map (PNG at 300 dpi plus PDF) and the presentation images, with per-step progress. |
| **4 Presentation** | **Agricultural recommendations**: drafted automatically (**Generate automatically**) from the forecast timing, the High / Very High areas and past damage, and fully editable. **Apply to presentation** redraws the slide, poster, Excel and ZIP without re-rendering the map. Below that are a 1920×1080 slide and an A4 poster (300 dpi), styled after `canva_template.pdf`. |
| **5 Download** | Every file plus a ZIP of everything. Previous runs are listed. |

Each run is saved to **`outputs/<YYYY-MM-DD_HHMM>/`**:

```
inputs/                         copies of the 3 input files used
settings.json                   weights, period, recommendations, classification, breaks, methodology
meta.json                       computed summary (used to redraw the presentation after edits)
label_report.json               map label check: every name placed, overlaps (always 0), leader lines
envi_results.xlsx               ENVI Results · Summary · Methodology · Name Matching · Recommendations
envi_results.csv
envi_results.geojson / .gpkg    boundaries joined with ENVI
match_report.csv                how every input row was matched
envi_map.png (300 dpi) / envi_map.pdf
envi_map_only.png               transparent map used in the infographic
presentation_1920x1080.png
presentation_A4_poster.png
ENVI_outputs_<run>.zip          everything above
```

---

## Input data (what was found)

| File | Structure | Level |
|---|---|---|
| `drought forecast.csv` | `ID, PROVINCE, October … March`. The values are categorical: *not affected / dry condition / dry spell / drought*. `ID` = OBJECTID of the provincial boundary layer (65–70). | Province (6) |
| `historical damage.xlsx` (Sheet1) | `MUNICIPALITY_CLEAN, RICE_DMG_2019, RICE_DMG_2024, RICE_DMG_2026` holds rice damage area in ha. Province rows (ABRA, APAYAO, …, CAR) are subtotals and are skipped. | Municipality (77) |
| `standing crops ao sept 15 2026.xlsx` (SEP15) | Column A is a PSGC code (`PH140101000`). Column B (unnamed) is the area name. Then `Corn Total`, `Rice Total` in ha. CAR and province rows are subtotals and are skipped. | Municipality (77) |

The GIS template `gis template/drought_forecast.qgz` uses
`Regional_Map_Municipal_Boundary` (fields `Pro_Name`, `Mun_Name` such as "Bangued (Abra)", and `MunName2`) and
`Regional_Map_Provincial_Boundary` (`OBJECTID`, `Pro_Name`). The original project joined the CSV on
`OBJECTID = ID`. The shapefiles were referenced at `../../CAR Shapefiles/`, outside this folder, so they
were copied from `Documents\GIS\CAR Shapefiles` into **`data/gis/`**. The layout logos (`G:\LOGOS\…`) were
copied into **`assets/logos/`**. The PAGASA logo was not found on this machine, so it is hidden.

**Join key.** The layers have no PSGC field, so all three datasets are matched to the GIS layer **by
name**: trim, uppercase, strip accents, drop "(Province)" suffixes and "City of"/"City", and expand
"Mt."→"Mountain", "Sto."→"Santo", "Sta."→"Santa". The key is `PROVINCE|MUNICIPALITY`. The province
comes from the PSGC code, the province block in the file, or the province column. Remaining
differences go through fuzzy matching (≥ 85% similarity, flagged for confirmation). Anything left
over is shown as unmatched.

Data issues handled automatically (and reported in step 1):

* **Paracelis** is listed under the Kalinga block in `historical damage.xlsx`, but it is in Mountain
  Province. It is matched by its unique name and flagged *exact (other province)*.
* **PEÑARRUBIA** appears as `PE�ARRUBIA` (the Ñ was corrupted by a code page). It is handled.
* **Baguio** has a blank `Rice Total`. Blank crop cells are treated as 0 ha (warning shown).
* Baguio City / City of Baguio / Baguio and Tabuk City / City of Tabuk / Tabuk all resolve to the same area.

---

## Methodology

**Indicators.** All three are oriented so that higher means more vulnerable.

| Indicator | Raw value |
|---|---|
| Drought forecast (hazard) | Each month is converted to an ordinal score: **not affected = 0, dry condition = 1, dry spell = 2, drought = 3**. The score is summed over the 6 outlook months (range 0–18). The forecast is provincial, so every municipality gets its province's score. If the forecast is numeric instead, it is treated as **rainfall % of normal**: the monthly mean is taken and *inverted* so that lower rainfall gives a higher value. A "higher = worse" option exists for dry-day counts. |
| Historical damage (sensitivity) | Mean annual damaged area (ha) over the event years 2019, 2024 and 2026. Crop columns of the same year are summed first. |
| Standing crops (exposure) | Rice + corn standing area (ha) as of Sept 15, 2026. |

**Normalization.** Each indicator uses min–max scaling to 0–1: `x_norm = (x − min) / (max − min)`. If
max = min, the value is 0.

**Index.** `ENVI = w_hazard·hazard_norm + w_damage·damage_norm + w_crops·crops_norm`. The default
weights are ⅓ each; they can be edited in the UI and must sum to 1.

**Classes.** Very Low `#1a9850` · Low `#91cf60` · Moderate `#fee08b` · High `#fc8d59` · Very High `#d73027`.
The default is equal intervals 0–0.2–0.4–0.6–0.8–1.0, lower-inclusive: 0.2 is "Low". Natural breaks
(Fisher-Jenks) and quantiles are optional; their classes are upper-inclusive.

**No data.** A GIS area that is missing from any input gets no ENVI. It is drawn grey and hatched
as **No data**, unless you choose "treat missing indicator as 0".

> **Note on the sample drought forecast.** Five provinces have the identical sequence (score 14) and
> only Benguet differs (score 11). With min–max scaling the hazard indicator therefore becomes
> binary: Benguet = 0, everyone else = 1. That puts all 14 Benguet LGUs in *Very Low* (ENVI ≤ 0.03),
> regardless of their crops. To score against the theoretical maximum instead (Benguet 0.61,
> others 0.78), choose **Drought forecast scaling → Fixed scale** in step 2.

**Standing crop area at risk** = the total standing rice + corn area in municipalities classed
High or Very High.

---

## Map rendering

* **PyQGIS (preferred).** `envi/qgis_render.py` runs headless with QGIS's own Python
  (`python-qgis-ltr.bat`). It opens `drought_forecast.qgz` with the boundary datasources re-pointed
  to the results GeoPackage, keeps the template's label and boundary styling, and applies a
  **graduated 5-class renderer** on `envi` plus a hatched *No data* layer. It then fills a print layout:
  * **Full DA layout** (default): the `drougt_layout` stored inside the `.qgz` (300×300 mm, with logos,
    title box, legend, note, disclaimer and sources). Its texts are replaced with the ENVI title,
    period, method note and sources.
  * **Compact template**: `droug_forecast_layout_template.qpt`. This file only contains the map
    frame, north arrow and scale bar, so the app adds the title, legend and source note around it.
* **matplotlib fallback.** This is used when QGIS is not installed or fails. It reproduces the full
  layout: map, right-hand panel with logos, title, legend with ranges, note, disclaimer and sources,
  north arrow, scale bar and labels.

Exports: PNG at 300 dpi, PDF, and a transparent map-only PNG for the infographic.

### Labels

All 77 municipality names and 6 province names are always shown, and labels never overlap.

* A municipality name goes inside its own polygon when it fits (municipal borders are weighted obstacles).
  Otherwise it sits just outside, joined by a thin leader line. Higher-ENVI areas get priority.
* Province names are placed after the municipalities, inside their province. Long names wrap
  ("Mountain / Province") and shrink slightly if needed.
* If a name still has no room, municipality labels shrink in 0.5 pt steps (down to 6 pt). Any remaining
  name is pinned to the nearest free spot.
* The verified positions are then fixed for the final 300 dpi PNG/PDF and checked again. The result is
  written to `label_report.json`. The matplotlib fallback uses the same rules (`envi/labels.py`), and a
  test measures the real text boxes.

### Fonts

All generated files use **Montserrat**: map labels and layout text, presentation images, Excel, and both web UIs.
The font is bundled in `assets/fonts/` (SIL Open Font License) and registered at render time, so it does not
need to be installed. The PDF stores text as vector outlines and looks the same on any computer.

## Presentation

`envi/presentation.py` draws the infographic with Pillow, so it is fully reproducible. The recommended actions
appear in the left column of the slide (condensed to each action's lead-in) and in full on the A4 poster. The layout
follows `canva_template.pdf`: a DA header with logos and title, then a left column with overview,
methodology, weights, risk hotspots and legend. The map is in the centre. The right column has
counts per class, standing crop area at risk (ha), and the top 10 most vulnerable areas. A footnote
lists the indicators, normalization, weights, period and issue date, followed by the data sources.

---

## Project structure

```
app.py                  FastAPI server + JSON API (wizard back-end)
envi/
  config.py             paths, labels, colours, defaults, QGIS discovery
  io_loader.py          read/validate inputs, clean names, match to GIS, templates
  normalize.py          min–max, fixed scale, drought hazard normalization
  index.py              weights, ENVI, classification (equal/quantile/Jenks), summary
  mapping.py            join to boundaries, GeoJSON, PyQGIS job / matplotlib fallback
  qgis_render.py        headless PyQGIS renderer (runs under QGIS's Python)
  presentation.py       Pillow infographic (slide + A4 poster)
  recommendations.py    rule-based draft of agricultural recommendations
  labels.py             collision-free label placement for the matplotlib fallback
  pipeline.py           end-to-end run, Excel export, outputs/<timestamp>/, ZIP
  __main__.py           command-line run (python -m envi)
static/                 index.html, app.js, styles.css, vendored Leaflet
data/input/             default/sample inputs
data/gis/               CAR municipal + provincial boundaries (copied from the GIS template sources)
assets/logos/           logos used by the layout and infographic
assets/fonts/           Montserrat (TTF for rendering, WOFF2 copies in static/fonts and web/fonts)
web/                    Vercel upload portal (see web/README.md)
scripts/                build_web_assets.py – regenerates the portal's templates and CAR name list
gis template/           QGIS project + print layout template
tests/                  pytest: normalization, weights, classification, name matching, API flow
```

**Why FastAPI rather than Streamlit.** The wizard needs a Leaflet map with custom hover tooltips,
a client-side sortable table, editable name-matching rows, and a background job with step-by-step
progress for the roughly 10 s PyQGIS render. A small JSON API and a plain JavaScript page handle
this directly. Streamlit's rerun-the-script model makes multi-step state, Leaflet events and long
jobs awkward.
# ENVI
