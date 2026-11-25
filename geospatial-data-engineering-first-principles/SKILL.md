---
name: geospatial-data-engineering-first-principles
description: First-principles geospatial data engineering playbook from 23 talks (Paul Ramsey, Matt Forrest, Dunnington & Pebesma, Kyle Barron, DuckDB's Gabrielsson, Overture). Use for any lat/lon, distance, nearest-X, spatial join, CRS, PostGIS/DuckDB/GeoParquet, OSM, Overture or GTFS work.
---

# Geospatial data engineering from first principles

This skill turns 23 talks and lectures into working judgment for pipelines that touch
location: measuring distance, matching points to the nearest thing, joining points to
polygons, choosing an engine, index and file format, and keeping geodata correct.
This file holds the procedures, the principles the teachers keep returning to, and
checklists. `references/` (when present) holds detailed notes for every video, each
credited to its teacher.

Use it whenever coordinates appear, even if nobody says "GIS": a lat/lon column in a CSV,
"find the nearest airport/station/hotel", "which country/city is this point in",
deduplicating places, GTFS feeds, OSM or Overture extracts, or a slow join on millions of rows.

**How to use it**
1. Follow the procedure in section 1 that matches the job (nearest-X, spatial join, or a new
   pipeline).
2. Run the QA and edge-case checklists (section 5) before trusting any output.
3. If a `references/` folder sits next to this file, use it for depth. `references/INDEX.md`
   lists every video with its line number, or run `grep -n -i "keyword" references/*.md`. Each section ends with a "Relevance to Sam" or "For Sam" note: Sam builds
   Trameter's master location dataset (GeoNames ~6M rows, ~8M destinations), a global
   transit-terminal dataset from ~1,650 GTFS feeds, place-to-ferry-terminal and
   destination-to-airport matching, and supplier matching (Viator, RateHawk), all on a 16 GB
   laptop. Use those parts only in that context. Lines marked "(note: …)" and those Sam notes
   are helper additions, not the speaker's words. If `references/` is missing, the full notes
   live in the user's private GitHub repo `Biodun-ml/engineering-skills` (folder
   `geospatial-data-engineering-first-principles/references/`); clone it when you need depth.
4. Explain the *why*, and credit the teacher when you use a specific pattern.
5. Library versions decide capability *(Ramsey)*. DuckDB spatial, GeoParquet and Overture
   change fast; check current docs before quoting a function or schema.

---

## 1. Working procedures

### 1a. Any new geodata pipeline
1. **Pin the CRS of every input on ingest.** A bare lat/lon pair without a datum is
   ambiguous; assume WGS84 (EPSG:4326) for GPS, GeoNames and OSM, but verify national or
   survey data. Assert `a.crs == b.crs` before any spatial comparison. *(Sutton; Soma)*
2. **Check axis order.** `ST_Point`/`ST_MakePoint` take x = lon, y = lat. EPSG:4326
   transforms can flip to lat/lon; use an always-xy option. *(Gabrielsson; Forrest)*
3. **Validate geometry** with `ST_IsValid` and repair with `ST_MakeValid` before overlays
   and joins; invalid input gives undefined results. *(Gabrielsson)*
4. **Decide planar or spherical once, by extent.** Regional data: project to a local metric
   CRS and use geometry. Global data, or anything near the poles or antimeridian: use a
   spherical engine (S2, PostGIS geography, BigQuery) or haversine. *(Pebesma & Dunnington;
   Ramsey)*
5. **Store working data as GeoParquet (ZSTD, spatially sorted) or in one DuckDB file,**
   never CSV/GeoJSON/Shapefile for intermediate steps. *(GeoDev; Barron; Gabrielsson)*
6. **Keep a stable ID on every row and a source key per record** (OSM type+id like `w50`/
   `r123`, GTFS feed_id+stop_id, GeoNames id). Add a fresh key after any dump/explode.
   *(Silk; Anderson; swroclawski)*
7. **Record provenance and a run manifest** (source URL, snapshot date, bbox, version,
   parent run), not dates in file names. *(Hanson; Gabrielsson)*

### 1b. "Match points to the nearest X" (airport, terminal, station, POI)
1. **Define "near" as a number in metres per use case.** City-level destination match may
   tolerate kilometres; airport-to-terminal matching needs tens of metres. *(EagleView
   podcast, For-Sam note)*
2. **Generate candidates with an index, never an N×M distance matrix.** 1M × 1M is 10¹²
   distances *(Parry)*. Options, in order of preference:
   - **PostGIS KNN:** `ORDER BY geom <-> pt LIMIT k` walks the index in distance order;
     ~10 ms for 10 neighbours on 2M GeoNames points, barely changing at 20M, because cost
     scales with k, not table size. For "nearest X for every row of A" use a **LATERAL**
     subquery with `<-> … LIMIT 1`. *(Ramsey)*
   - **Within-distance:** `ST_DWithin` / `st_is_within_distance`, index-backed and exact.
     Never buffer-then-intersect. *(Pebesma & Dunnington; Silk)*
   - **Cell keys on a laptop:** H3/S2/geohash cell of each point; join on the cell **plus
     its neighbours** (3×3 for geohash, k-ring for H3), then post-filter by exact distance.
     If nothing is found, expand the ring, as Uber's dispatch does. *(Hello Interview; Parry;
     O'Flaherty)*
   - **GeoPandas:** `sjoin_nearest(..., max_distance=...)` in a projected CRS; in EPSG:4326
     it measures in degrees and warns. For global data use a 3-D KD-tree or BallTree
     (section 8). *(Soma, note)*
3. **Rank survivors by true distance** (haversine/geodesic for global data), never by
   shared geohash prefix length. *(Parry; O'Flaherty)*
4. **Cap the distance and keep the unmatched.** Use a left join so points with no X in range
   show as nulls instead of vanishing. *(Soma)*
5. **For point-to-polygon assignment use Silk's function pattern:** `ST_DWithin` candidates
   (e.g. 1000 m) → largest intersection area if it straddles a boundary → else nearest →
   `LIMIT 1`. This catches features just outside every polygon (piers, coastal airports,
   extended wharves). *(Silk)*
6. **Score and band if names are involved:** distance + normalised name + country/admin,
   stored as 0–100 with auto-accept / review / reject bands. *(GeoMindz)*

### 1c. "Spatial join at scale on 16 GB"
1. **Pick the smallest engine that copes.** GeoPandas/DuckDB first; PostGIS when the data is
   shared and long-lived; Sedona only for planet-scale or complex overlays, since Spark
   overhead makes it slower on medium data. *(Forrest — PostGIS and Sedona; Gabrielsson)*
2. **Read only what you need.** Column projection plus a bbox predicate on spatially sorted
   GeoParquet reads a fraction of the file; filter remote Overture by bbox and export only
   the region. *(Barron; Anderson)*
3. **Two-stage filter:** cheap bbox (index, row-group stats) first, exact predicate second.
   *(Anderson; Gabrielsson; Barron; Silk)*
4. **Broadcast the small side.** Keep the few thousand polygons/destinations in memory (with
   an STRtree/R-tree) and stream the millions of points in chunks. Never materialise all
   candidate pairs at once. *(Forrest — Sedona; Soma)* Recent DuckDB does this for you
   (section 8).
5. **"Make it smaller."** `ST_Subdivide` big polygons (default 256 vertices per piece)
   before joining: 30–50× faster *(Ramsey)*, 2 min → 0.3 s (~400×) on 1.1M grid squares
   *(Silk)*. Simplify first if boundary precision doesn't matter.
6. **Sort spatially.** Cluster rows by Hilbert/Morton order so neighbours share pages and
   row groups. *(Ramsey; Barron)*
7. **Index, ANALYZE, key every intermediate table.** If a query runs ~2 minutes, kill it and
   rethink. *(Silk)*
8. **Check row counts after the join.** Shared borders and overlapping polygons duplicate
   left rows; inner joins drop offshore points. Compare `len(result)` to `len(left)`.
   *(Soma)*

---

## 2. Sam's real bug: a fixed ±4° box to find airports near destinations

**What goes wrong.** Degrees are not distances. A degree of latitude is ~111 km everywhere,
but a degree of longitude is ~111 km × cos(lat) and shrinks to 0 at the poles
*(Hello Interview; Pebesma & Dunnington)*. So a ±4° box:
- is ~890 km wide at the equator but only ~445 km at 60° and ~155 km at 80° (arithmetic
  from 111 km × cos lat): destinations in Norway, Iceland, Alaska, northern Canada and
  Siberia lose airports that sit within the intended radius;
- is a rectangle, so corners reach ~1.4× farther than the edges: neither a radius nor fair;
- never wraps the antimeridian: Fiji (~178° E) or the Aleutians get `lon + 4 > 180` and
  silently miss airports at −179°. A 4-number bbox cannot express a region crossing ±180
  *(Pebesma & Dunnington)*;
- can exceed ±90° latitude near the poles.

**Fix (decision order):**
1. Choose a radius in km, not degrees, and justify it per use.
2. Best: let an index answer it. PostGIS geography with `ST_DWithin(geog, pt, metres)` or
   KNN `<->` in a LATERAL join *(Ramsey)*; or H3 cell + k-ring candidates then haversine
   *(Hello Interview)*.
3. If you keep a box prefilter, make it latitude-aware and wrap-aware, then filter exactly:

```python
import math
R_KM = 6371.0
def candidate_boxes(lat, lon, radius_km):
    """Boxes covering a spherical cap of radius_km; 1 box, or 2 if it crosses +/-180."""
    r = radius_km / R_KM                                   # angular radius, radians
    dlat = math.degrees(r)
    lat_lo, lat_hi = lat - dlat, lat + dlat
    if lat_hi >= 90 or lat_lo <= -90:                      # cap contains a pole
        return [(max(lat_lo, -90.0), min(lat_hi, 90.0), -180.0, 180.0)]
    dlon = math.degrees(math.asin(math.sin(r) / math.cos(math.radians(lat))))
    lo, hi = lon - dlon, lon + dlon
    if lo < -180: return [(lat_lo, lat_hi, lo + 360, 180.0), (lat_lo, lat_hi, -180.0, hi)]
    if hi > 180:  return [(lat_lo, lat_hi, lo, 180.0), (lat_lo, lat_hi, -180.0, hi - 360)]
    return [(lat_lo, lat_hi, lo, hi)]
# then keep candidates with haversine(dest, airport) <= radius_km, sort by distance, take k;
# if none, widen the radius (ring expansion) instead of returning nothing.
```

The `asin(sin r / cos lat)` longitude extent is the exact cap width, not `r / cos lat`
(helper addition, standard spherical trigonometry; not from the talks). Test with fixtures
at 0°, 60°, 78° N (Svalbard), 89.5° S, lon 179.5° and −179.5°. Simpler still on a laptop:
a 3-D unit-vector KD-tree or a haversine BallTree has no edges at all (section 8).

---

## 3. Principles that recur across the teachers

### Coordinates, datums and measuring
- **A CRS = ellipsoid + datum (+ projection + central meridian/zone).** Lat/lon only means
  something relative to a datum; the same pair under different datums can be metres to
  hundreds of metres apart. *(Sutton; Q-tips)*
- **Every projection distorts, and distortion grows away from the line of tangency,** so
  pick the zone/central meridian nearest your area (UTM: transverse Mercator, 6° zones,
  scale 0.9996). *(Q-tips, with note)*
- **Never buffer or measure in degrees.** `.buffer(10)` in EPSG:4326 is 10 *degrees*.
  *(GeoDev; Q-tips)*
- **Reproject all layers to one CRS before overlay.** NAD83 vs WGS84 is ~1–2 m in North
  America; projected-vs-geographic mismatches are silently catastrophic. *(Sutton; Soma)*
- **Geometry vs geography.** Geometry is Cartesian: cheap (one sqrt). Geography uses
  great-circle edges: even haversine needs ~5 transcendentals. Only distance, length, area
  and DWithin are native on geography; everything else casts. Use geography for global data
  crossing the dateline or poles; otherwise pick a good local projection. *(Ramsey)*
  (Review note: PostGIS geography `ST_Distance`/`ST_DWithin` default to the *spheroid*
  (`use_spheroid = true`); haversine was Ramsey's cost illustration. Pass `false` for the
  faster sphere when ~0.5% error is acceptable.)
- **Edges are geodesics on the sphere.** Large "rectangles" stop following constant-latitude
  lines, and data built for planar tools often fails as "invalid spherical geometry".
  *(Pebesma & Dunnington)*

### Indexes
- **1-D sort order doesn't preserve 2-D closeness,** so two B-trees on lat and lon still
  leave millions of rows to check. *(Hello Interview)*
- **Two camps:** disk trees (R-tree/R*-tree, BKD) for shapes and exact geometry, with costly
  writes; encoded cell keys (geohash, S2, H3) in any B-tree or sorted set for high-write
  point workloads. *(Hello Interview)*
- **The R-tree stores each object's bounding rectangle;** predicates test cheap boxes first,
  then fetch full geometry for survivors. Create the GiST index and run `ANALYZE` so the
  planner uses it. *(Silk; Hello Interview)*
- **Cell keys are candidate filters, not answers.** Points 1 m apart across a cell edge can
  share no prefix. Always query cell + neighbours, then post-filter. *(Hello Interview;
  Parry; O'Flaherty)*
- **Geohash is a Z-order (Morton) curve:** lon bit first, interleave, base-32 (alphabet
  omits a, i, l, o), max 12 chars = 60 bits. 5 chars ≈ 5 km, 6 ≈ 1.2 × 0.6 km, 9 ≈ 5 m.
  Cells narrow toward the poles. *(Parry; O'Flaherty; Hello Interview)*
- **S2:** cube + Hilbert curve, 64-bit hierarchical IDs, roughly equal cells, handles
  antimeridian-crossing polygons. **H3:** hexagons with 6 equidistant neighbours, 16
  resolutions, ~7 children per parent (approximate nesting), open-sourced 2018. *(Hello
  Interview, with note)*
- **Quadtree** adapts to density but latency depends on where you query; **KD-tree** is the
  classic in-memory nearest-neighbour structure; both are pointer-based and disk-unfriendly.
  *(Hello Interview)*

### Engines
- **A spatial DB is ordinary SQL + spatial types + spatial indexes + `ST_*` functions,** so
  spatial filters compose with WHERE/JOIN/GROUP BY. *(Ramsey)*
- **PostGIS is OLTP:** shared, edited, long-lived data serving apps, with access control,
  backups and views. Overkill for one-off analysis; wrong for heavy raster or ML batch.
  *(Forrest — PostGIS)*
- **DuckDB spatial fills the "too big for pandas, too small for a cluster" middle,** follows
  PostGIS names on GEOS/GDAL/PROJ, has typed `POINT_2D`, an R-tree index (~100× for
  selective filters against a constant geometry), and reads remote Parquet in one query.
  Keep pipeline state in one `.duckdb` file, never CSV. *(Gabrielsson, recorded Sept 2024;
  see section 8 for what has shipped since)*
- **Sedona is "PostGIS for Spark":** worth it only at volume or complexity walls (zonal
  stats, area-weighted interpolation); SedonaDB is the single-node option. *(Forrest — Sedona)*
- **Enforce integrity in the database:** typed columns `geometry(Point,4326)`, CHECK
  constraints, stored functions for business rules. *(Silk)*

### Formats and cloud-native
- **GeoParquet beats Shapefile/GeoJSON by a wide margin:** 3.3M polygons = ~420 MB / ~7 s
  read vs 2.74 GB / 53 s (Shapefile) vs 3.25 GB / 4+ min (GeoJSON). Shapefile truncates
  field names to 10 characters. *(GeoDev)*
- **Spatial pruning needs spatial sorting.** The bbox covering column (GeoParquet 1.1) gives
  per-row-group min/max, but with random order every group spans the globe. `ORDER BY
  ST_Hilbert(geom)` before writing; row groups ~100k–1M rows; ZSTD. *(Barron)* (Review
  note: Parquet itself now has native GEOMETRY/GEOGRAPHY logical types with geo statistics,
  and GeoParquet 2.0, at release-candidate stage, builds on them. Sorting still decides
  whether pruning works.)
- **Partitioning (coarse) is not indexing (fine).** GeoParquet scales to billions;
  FlatGeobuf/GeoPackage/DuckDB R-tree index every row for small lookups. *(Barron)*
- **Cloud-native = small addressable chunks + a table of contents** (COG tiles, Zarr chunks,
  Parquet row groups), read with range requests and in parallel. Avoid thousands of tiny
  files. *(Hanson 2022/2023)*
- **STAC is for asset files, not features.** If the geometry is the data (POIs, airports),
  use a feature format/API; if it is a footprint of separate files (scenes, tiles), use
  STAC. *(Hanson 2023)*
- **GeoParquet for compute, PMTiles for display.** *(Barron; Anderson)*

### Data models: OSM, Overture, GTFS
- **OSM has nodes, ways and relations.** Multipolygons, boundaries, many airports
  (`aeroway=aerodrome`) and transit routes are relations whose tags sit on the relation;
  assemble geometry from member ways with roles. Extract all three types; use
  point-on-surface for areas. *(swroclawski, with note)*
- **Overture is a conflation layer downstream of OSM and others,** with explicit precedence
  (OSM > community > ML gap-fill), per-attribute `sources`, a 0–1 existence confidence
  (0 = "no longer exists here", a useful signal), and monthly releases. *(Anderson —
  Sept 2023 alpha webinar: the files are now real GeoParquet and GERS IDs are stable
  UUID-style IDs, not the H3-prefixed design shown; treat schema details as historical.)*
- **Stable shared IDs beat repeated fuzzy matching.** One hotel lives in 100+ systems;
  bridge tables (internal ID ↔ GERS) mean integrating once instead of N×N. Coverage gates
  adoption. *(Wiedersberg)*
- **Licences travel per layer:** ODbL (OSM-derived roads, buildings) is share-alike for
  derivative databases; CDLA-Permissive (Overture places) is not. *(Anderson)*
- **GTFS: only `stops` (and optional `shapes`) are geographic.** Mode lives on `routes`
  (stop → stop_times → trips → routes). Collapse platforms to `parent_station`; entrances
  (2) and generic nodes (3) are not boarding points. Times can exceed 24:00; some feeds use
  only `calendar_dates`. IDs are unique only within a feed. *(ArcGIS GTFS)*

### Matching, accuracy and analysis
- **Geocoding = standardise → parse into components → compare with a reference → score →
  band.** Same normaliser on both sides; ~80+ accept, 60–79 review, <60 reject; locale
  matters. *(GeoMindz)*
- **Accuracy has axes: resolution, positional accuracy vs ground truth, freshness.** Cost
  rises steeply, so set tolerance by the decision. Sharp pixels can be in the wrong place.
  *(EagleView podcast)*
- **POIs decay:** ~1 in 4 businesses open, close or move each year; errors are asymmetric
  (wrongly open vs wrongly closed). Fuse weak signals into a "heartbeat". *(Wiedersberg)*
- **Spatial data is not locationally invariant:** identical histograms can hide different
  maps. Check spatial autocorrelation (Moran's I/LISA), choose aggregation units
  deliberately, and use spatial cross-validation for ML. *(Anselin)*
- **Pre-bake spatial relationships for LLM consumers** as edges with type and confidence,
  regenerated each release. *(Wiedersberg)*

---

## 4. Choosing a spatial index

| Situation | Choose | Why |
|---|---|---|
| Polygons/lines, exact predicates, data in a DB | R-tree (PostGIS GiST, DuckDB R-tree) | Handles shapes; balanced, disk-friendly *(Hello Interview; Silk)* |
| Millions of points, heavy writes, any store | Geohash/S2/H3 key in a B-tree or sorted set | One integer update; prefix/range scan *(Hello Interview)* |
| Global or high-latitude points | S2 or H3, not geohash | Near-equal cells; geohash slivers near poles *(Hello Interview; O'Flaherty)* |
| Rings, heatmaps, "within n cells" | H3 | 6 equidistant neighbours *(Hello Interview)* |
| Crosses the antimeridian | S2 covering or geography | Flat bboxes span −180..180 *(Pebesma & Dunnington)* |
| Static batch, slowly changing | BKD / sorted GeoParquet + bbox stats | Write-once, cheap pruning *(Hello Interview; Barron)* |
| Map tiles, collision | Quadtree | Density-adaptive *(Hello Interview)* |

---

## 5. Checklists

**Geodata QA (run on every new dataset and every pipeline output)**
- [ ] CRS present and identical across layers; axis order lon, lat confirmed.
- [ ] `ST_IsValid` count = 0 invalid (or repaired with `ST_MakeValid`, and logged).
- [ ] Geometry types constrained (no mixed points/polygons in one column).
- [ ] Lat in [−90, 90], lon in [−180, 180]; no (0,0) "null island" or swapped pairs.
- [ ] Row counts before and after every join; duplicates from shared borders handled.
- [ ] Unmatched rows inspected (left join), not silently dropped.
- [ ] Unique primary key after every dump/explode; stable source IDs kept.
- [ ] Coverage profiled per region (count per zoom-8 tile or H3 cell) before trusting a
      match rate. *(Anderson)*
- [ ] Sample validated against ground truth in *your* region (Nigeria/Africa coverage in
      Overture is likely weaker). *(Wiedersberg note; EagleView)*
- [ ] Source and capture/snapshot date per record; confidence kept, not just filtered.
- [ ] Licence of every layer recorded before merging.
- [ ] Output mapped and eyeballed (pgAdmin geometry viewer, lonboard, QGIS). *(Silk;
      Gabrielsson; Anselin)*

**Edge cases (test each with a fixture row)**
- [ ] Antimeridian: points at ±179.9°, a polygon spanning it (Fiji, Chukotka, Aleutians).
- [ ] Poles and high latitudes: 70–90° N/S; boxes clamped; geohash cells are slivers there.
- [ ] Equator/prime meridian: geohash prefixes split there, so neighbours share nothing.
- [ ] Points just outside every polygon (coast, piers, offshore terminals).
- [ ] Points on shared borders (match twice).
- [ ] Planar-valid but spherically invalid polygons when moving to geography/S2.
- [ ] Multipolygon relations with missing members at extract edges.
- [ ] GTFS: times > 24:00, blank interpolated times, expired calendars, IDs with leading
      zeros read as strings.
- [ ] Names: "St" (Saint vs Street), "Intl"/"International", accents, non-Latin scripts.

**Performance**
- [ ] Spatial index + ANALYZE on every table used in a join.
- [ ] Big polygons subdivided; data spatially sorted.
- [ ] DWithin/KNN, never buffer-then-intersect or a full distance matrix.
- [ ] Small side in memory, big side streamed in chunks.

---

## 6. Failure modes to hunt for in reviews

- Distances or buffers in degrees; fixed-degree boxes as "radius".
- Planar operations on global lon/lat; bboxes that don't wrap ±180.
- Ranking by geohash prefix length instead of real distance.
- A single cell lookup without neighbours.
- Nested Python loops or a materialised N×M pair table (Sam's 514M-pair OOM).
- Inner spatial joins hiding unmatched points; duplicates from border points.
- One huge unsubdivided country polygon in a join.
- Joining layers in different CRSs, or swapped lat/lon.
- CSV/GeoJSON as intermediate storage; `final_v3.csv` sprawl.
- Extracting OSM nodes and ways but not relations.
- Treating GTFS `stop_id` as globally unique; parsing 25:10:00 as a clock time.
- Trusting one POI source for open/closed status.
- Re-fuzzy-matching the same entities every run instead of keeping bridge tables.

---

## 7. Where the teachers disagree, and how to resolve it

- **Geometry or geography?** Ramsey: geography is slow, project regionally. Pebesma &
  Dunnington: the stack should go spherical. *Resolve by extent:* regional → projected
  geometry; global or polar/antimeridian → spherical, using DWithin not buffers.
- **PostGIS or DuckDB/Sedona?** Forrest scores PostGIS 0 for one-off work but 4–5 for teams;
  Gabrielsson pushes DuckDB for the ~100 GB middle; Forrest's Sedona talk pushes scale-out.
  *Resolve by workload:* batch transforms in DuckDB, serving and shared editing in PostGIS,
  Sedona only after DuckDB visibly fails.
- **Index every row or partition?** FlatGeobuf/R-tree vs GeoParquet row-group bboxes
  *(Barron)*. *Resolve by access pattern:* many small lookups → row index; analytical scans
  → sorted, partitioned GeoParquet.
- **Geohash or H3/S2?** Parry and O'Flaherty show geohash as simple and string-friendly;
  Hello Interview shows its polar distortion. *Resolve:* geohash for a quick regional
  blocking/sort key; H3/S2 for global or high-latitude work.

---

## 8. Not covered by the sources, but know this

> Reviewer additions (senior geospatial engineer, checked against docs as of late 2026).
> Not from the 23 talks; verify versions before quoting.
>
> - **Distance maths.** Haversine (sphere) is off by up to ~0.5% vs the ellipsoid; fine for
>   "nearest airport", not for survey work. Exact: Karney geodesics (`pyproj.Geod.inv`,
>   geographiclib). In DuckDB, `ST_Distance_Sphere` is haversine and `ST_Distance_Spheroid`
>   / `ST_DWithin_Spheroid` are ellipsoidal (metres).
> - **Projection choice.** Local metric work: UTM zone `floor((lon+180)/6)+1`, EPSG 326xx
>   north / 327xx south (Lagos: 32631). Distance from one point: azimuthal equidistant
>   centred on it. Area: an equal-area CRS (global EPSG:6933, Europe EPSG:3035). Never
>   measure in Web Mercator (EPSG:3857): its scale error is 1/cos(lat), ×2 at 60°.
> - **Geodesic buffering.** If you truly need a buffer polygon, project to an azimuthal
>   equidistant CRS centred on the feature, buffer in metres, transform back. Usually you
>   don't: use DWithin.
> - **Laptop nearest-neighbour with no edge cases:** convert lat/lon to 3-D unit vectors
>   and use `scipy.spatial.cKDTree` (chord length is monotonic in great-circle distance),
>   or `sklearn.neighbors.BallTree(metric="haversine")` on radians. Both handle the
>   antimeridian and poles natively, query in batches, and replace any degree box.
> - **H3 resolution choice.** Average edge length (H3 v4): res 5 ≈ 9.9 km, 6 ≈ 3.7 km,
>   7 ≈ 1.4 km, 8 ≈ 0.53 km, 9 ≈ 0.20 km. Pick the resolution whose edge is a fraction of
>   your search radius, use `grid_disk(cell, k)` with k ≈ radius / edge + 1, then filter
>   exactly. h3-py v4 renamed the API (`latlng_to_cell`, `grid_disk`; v3 was `geo_to_h3`,
>   `k_ring`); AI-written code often mixes them. Pentagons (12 per resolution) and cell
>   distortion mean k-rings are not exact circles.
> - **DuckDB spatial specifics.** 1.3 added a `SPATIAL_JOIN` operator: it builds an
>   in-memory R-tree on the smaller (right) side and streams the left, so joins on
>   `ST_Intersects`/`ST_DWithin` no longer need hand-rolled chunking, but the right side
>   must fit in RAM. 1.5 (Mar 2026) moved `GEOMETRY` into core with an optional,
>   still-experimental CRS parameter, and changed axis order for the spheroid functions:
>   set `geometry_always_xy = true` so they take x = lon, y = lat. There is no geography
>   type; `ST_Hilbert` gives the sort key for GeoParquet.
> - **Polars / GeoArrow.** Polars has no native geometry type; community plugins exist, but
>   check maturity. Move data between DuckDB, GeoPandas, pyogrio and lonboard as GeoArrow
>   (zero-copy) rather than via WKT strings.
> - **PMTiles.** A single-file vector/raster tile archive read by HTTP range requests:
>   build with `tippecanoe`, host on any static bucket, no tile server needed.
> - **OSM extracts.** Do not touch the planet file on 16 GB: download a Geofabrik country/
>   region `.osm.pbf`, cut with `osmium extract --strategy=complete_ways` or `smart` (keeps
>   multipolygon relations whole), filter with `osmium tags-filter`, then load with
>   QuackOSM/pyosmium/osm2pgsql.
> - **GeoNames caveat.** Coordinates are often centroid-ish with unknown error; feature
>   codes (`AIRP`, `PRT`) are not a substitute for an IATA/ICAO or GTFS source of truth.

---

## 9. Applying this to Sam's work

- **GeoNames ~6M / destinations ~8M:** a one-off transform fits DuckDB + GeoParquet; a
  shared master table that apps query fits PostGIS with GiST. *(Forrest — PostGIS note)*
- **Destination → airport / place → ferry terminal:** use section 1b and the section 2 fix;
  the ferry join's 5-hour loop and 3.83 GiB OOM (514M materialised pairs) are cured by an
  index-backed candidate step (a 3-D cKDTree `query_ball_point`/`query`, DuckDB
  `SPATIAL_JOIN` with `ST_DWithin_Spheroid`, PostGIS KNN, or H3 k-ring) streamed in chunks,
  not by more vectorised pairs.
- **Assign places to countries/admins:** subdivide polygons, left join, Silk's
  candidate → overlap → nearest function for coastal points.
- **GTFS terminals:** collapse to `parent_station`, namespace IDs per feed, derive modes via
  routes, drop stops with no active service. *(ArcGIS GTFS)*
- **Supplier matching (Viator, RateHawk):** standardise → component match → distance + name
  score → bands; keep bridge tables, consider GERS IDs. *(GeoMindz; Wiedersberg)*
- **OSM airports/stations:** include relations, keep `n/w/r` + id as the key. *(swroclawski)*

---

## 10. Where to look in `references/`

| File | Covers |
|---|---|
| `geo-1-ramsey-forrest-dunnington-interview-parry-anselin.md` | PostGIS intro and KNN (Ramsey), Overture/GERS and POI freshness (Wiedersberg), S2 and spherical geometry (Pebesma & Dunnington), PostGIS decision matrix (Forrest), spatial index survey (Hello Interview), geohash internals (Parry), spatial analysis (Anselin), GeoParquet benchmark (GeoDev), 10 PostGIS lessons (Silk), CRS basics (Q-tips; Sutton) — 11 videos |
| `geo-2-anderson-ouazza-dill-forrest-hanson-geomindzcom.md` | Working with Overture (Anderson), DuckDB spatial (Gabrielsson), data accuracy (EagleView), Apache Sedona (Forrest), cloud-native geo and STAC (Hanson ×2), address standardisation (GeoMindz), why GeoParquet (Barron), GeoPandas spatial joins (Soma), GTFS (ArcGIS), geohash intuition (O'Flaherty), OSM relations (swroclawski) — 12 videos |
| `INDEX.md` | Every video with line numbers |
