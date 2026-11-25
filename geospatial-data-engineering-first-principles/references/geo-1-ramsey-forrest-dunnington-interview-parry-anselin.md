# geo-1 — cross-cutting principles

1. **A spatial DB is just types + indexes + functions.** PostGIS adds `geometry`/`geography`, GiST R-trees and `ST_*` to ordinary SQL, so spatial filters compose with ordinary WHERE/JOIN/GROUP BY. *(Ramsey; Forrest PostGIS)*
2. **Location is a universal join key.** The most common spatial pattern is the spatial join that enriches table A with B's attributes (point-in-polygon, within-distance). Second is K-nearest-neighbour. *(Ramsey; Anselin pipeline)*
3. **Use index-assisted KNN, not full distance scans.** `ORDER BY geom <-> pt LIMIT k` costs ~10 ms on 2 M GeoNames points and barely changes at 20 M, because cost scales with k, not table size. For "nearest X per row" use a LATERAL join. *(Ramsey)*
4. **Select by distance with DWithin, never buffer-then-intersect.** `ST_DWithin` / `st_is_within_distance` are index-backed and exact; buffers are slow and, on the sphere, approximated. *(Pebesma & Dunnington; Silk)*
5. **"Make it smaller."** Run `ST_Subdivide` on large polygons before joins: 30–50× (Ramsey) to ~400× (Silk: 2 min → 0.3 s) faster, because small pieces have tight bboxes and are cheap to read. Simplify first if boundary precision doesn't matter. *(Ramsey; Silk)*
6. **Index, ANALYZE, keep keys.** Every intermediate table needs a GiST index, planner statistics and a unique primary key; add `ROW_NUMBER()` after any dump/explode. Physically cluster rows in Hilbert/Morton order. If a query runs ~2 min, rethink it. *(Silk; Ramsey)*
7. **Two families of spatial index:**
   - Custom disk trees (R-tree/R\*-tree, BKD) for shapes and exact geometry, with costlier writes.
   - Encoded cell keys (geohash, S2, H3) in a plain B-tree/sorted set for high-write point workloads.
   - Choose by geometry type and write rate. *(Hello Interview)*
8. **Cell keys are candidate filters, not answers.** Neighbouring points across a cell edge get different prefixes. Always query the cell plus its neighbours (3×3 for geohash, k-ring for H3), then post-filter by exact distance. *(Hello Interview; Parry)*
9. **Geohash = Z-order curve.** Binary-partition lon over [−180,180) and lat over [−90,90), interleave lon first, then base-32 encode in 5-bit chunks (max 12 chars = 60 bits). Cells are lon/lat rectangles that become slivers toward the poles. S2 (cube + Hilbert, 64-bit) and H3 (hexagons, 16 resolutions, 6 equidistant neighbours) fix this for global work. *(Parry; Hello Interview)*
10. **Lon/lat is not a plane.**
    - Planar ops on degrees break at the antimeridian and poles, and distort with latitude (1° lon ≈ 111 km × cos lat).
    - For regional work, project to a local metric CRS. For global data, use spherical/geodesic engines (S2, PostGIS geography, BigQuery).
    - Expect "invalid spherical geometry" when moving planar-built data onto the sphere. *(Pebesma & Dunnington; Ramsey)*
11. **Geography is correct but expensive.** Haversine needs ~5 transcendentals vs one sqrt for planar. Only distance/length/area/DWithin are native on geography; everything else casts. If data fits a region, use geometry in a good projection. *(Ramsey)*
12. **Bounding boxes lie near the antimeridian.** A 4-number bbox for Alaska spans −180..180. Handle xmin > xmax boxes explicitly, or use S2 cell coverings or caps (same bytes, far better fit). *(Pebesma & Dunnington)*
13. **CRS = ellipsoid + datum (+ projection + central meridian/zone).**
    - Lat/lon only has meaning relative to a datum.
    - Reproject all layers to one CRS before overlay.
    - Measure only in a CRS suited to the region. Distortion grows away from the line of tangency, so pick the nearest zone/meridian (UTM/TM).
    - Never buffer or measure in degrees. *(Sutton; Q-tips; GeoDev demo)*
14. **Store working data as GeoParquet, not GeoJSON/Shapefile.** 3.3 M polygons: 420 MB / ~7 s read vs 2.7 GB / 53 s (SHP) vs 3.25 GB / 4+ min (GeoJSON). Shapefile also truncates field names to 10 characters. Pair it with DuckDB for larger-than-RAM work. *(GeoDev)*
15. **Pick the engine by workload.**
    - One-off or batch analytics: GeoPandas/DuckDB/Sedona on files.
    - Shared, edited, long-lived data serving apps: PostGIS (OLTP), with access control, backups and views.
    - Don't run heavy raster/ML in PostGIS. *(Forrest PostGIS)*
16. **Enforce integrity in the database.** Typed geometry columns `geometry(Point,4326)`, CHECK constraints (e.g. no overlaps), and stored functions for business rules. Example: assign a building to a suburb with DWithin candidates → largest intersection area → else nearest → LIMIT 1, which handles features just outside every polygon. *(Silk)*
17. **Stable shared IDs beat repeated fuzzy matching.** One hotel can sit in 100+ systems with 100+ IDs. Overture's GERS plus bridge tables (internal ID ↔ GERS) means integrating once instead of N×N. Coverage gates adoption. *(Wiedersberg/Overture)*
18. **POI freshness is a first-class quality dimension.** About 1 in 4 businesses open, close or move each year, and closures go unreported. Errors are asymmetric (wrongly open vs wrongly closed), so there is no safe default. Fuse many weak signals into a "heartbeat" rather than trusting one source. *(Wiedersberg/Overture)*
19. **Spatial ≠ lat/lon as features.** Spatial data is not locationally invariant: identical histograms can hide totally different maps. Test for spatial autocorrelation (Moran's I/LISA), aggregate to deliberate units, and treat distance and interaction as part of the method (e.g. spatial CV). *(Anselin)*
20. **Pre-bake spatial relationships for LLM/agent consumers.** Containment and proximity become graph edges with type and confidence, regenerated every data release, so agents don't run spatial joins on the fly. *(Wiedersberg/Overture)*

---

## Paul Ramsey (Crunchy Data) — PostGIS Introduction, STL PostGIS Day 2019 (https://www.youtube.com/watch?v=g4DgAVCmiDE)
**A spatial database is just a normal database plus spatial types, spatial indexes and spatial functions; if you already run Postgres, `CREATE EXTENSION postgis` gives you a GIS that you drive with SQL.**

- **Mental model.** A database offers (1) primitive types, (2) indexes suited to them (B-tree, hash), (3) functions over them. A spatial DB adds the "spatial" version of each: `geometry`/`geography` columns, multi-dimensional indexes (R-tree, quad-tree, KD-tree), and `ST_*` functions (length, distance, intersects). The `ST_` prefix comes from the OGC Simple Features for SQL spec (~60 functions) and later ISO SQL/MM Part 3 (~300); PostGIS has 700+. PostGIS is now the de facto standard: other vendors advertise "PostGIS-compatible".
- **Why a DB over files.** Shapefile/geodatabase piles give no versioning and no single source of truth. Putting everything in one DB gives a shared truth, a standard query language, and lets unrelated tools (QGIS, ArcGIS, GeoServer, MapServer, every language driver) integrate implicitly by reading/writing the same tables. Ramsey's consultancy found it was "doing GIS without opening a GIS": reports became SQL, then repeated parameterised questions became web forms.
- **"Spatial is the universal key"** (Martin Davis, building on Tobler's first law: near things are more related than distant ones). Two tables with no foreign key can still be joined by location. The canonical example is enriching a customer/sales table with census-tract demographics via a 5–7 line spatial join. Ramsey calls the **spatial join the single most common pattern** in spatial SQL (enrich table A with attributes of table B); the **nearest-neighbour query is second**.
- **Typical queries are a handful of lines:** parcels within 1000 m of a fire (`ST_DWithin`); bus distance last week (filter by id + time, `SUM(ST_Length)`); trucks in depots (`ST_Contains`/`ST_Within` join); nearest truck to transformer.
- **Index-assisted KNN: the `<->` operator** ("TIE fighter"). `ORDER BY geom <-> point LIMIT k` walks the index in distance order. Benchmark: 2 M GeoNames points, 10 nearest neighbours ≈ 10 ms, and it barely changes at 20 M. Cost scales with *k* (how much you pull off the index), not table size. For "nearest X for every row of table A" use a **LATERAL join** (per-row subquery with `<-> ... LIMIT 1`). Directly relevant to Sam's GeoNames-to-airport/terminal matching.
- **Type system.** Columns are declared like any other with type modifiers, e.g. `geometry(Point, 26910)` (type + SRID, here UTM zone 10). Types: Point/LineString/Polygon, Multi*, GeometryCollection; ISO adds CircularString, CompoundCurve, curve polygons; draft-ISO adds PolyhedralSurface, Triangle, TIN. Coordinates can be XY, XYZ, XYM, XYZM. Indexes support N-D; 2-D is the default. A 3-D KNN needs an N-D index (`gist_geometry_ops_nd`). Plain functions are 2-D; `ST_3D*` variants give 3-D answers (3-D length matters in hilly terrain).
- **M (measure) dimension** exists for linear referencing: highways ("asset at mile 35 on Hwy 6") and fisheries keep one canonical network plus side tables of from/to measures instead of exploding into tiny overlapping segments. Today M is often used as time (epoch seconds) for trajectories, with functions like closest point of approach between two moving objects.
- **Geometry vs geography.** Geometry is Cartesian (R²/R³): straight lines between vertices, cheap math (one sqrt). Geography is on the sphere/spheroid (S²): great-circle edges. Even haversine needs ~5 transcendentals plus 2 sqrts and still assumes a sphere, so geography is much slower. **Who needs geography:** (a) newcomers with lat/lon who want metres out without learning projections; (b) truly global data that crosses the dateline or poles (satellite tracks, Russia/Alaska across ±180), where no planar projection works. **Advice:** if your data fits a region, pick a good local projection and use geometry. Only distance, length, area, DWithin and a few others are native on geography; everything else silently casts to geometry and back.
- **Libraries underneath:** GEOS (computational geometry: buffer, intersection, union, DE-9IM predicates; also used by Shapely, QGIS, MapServer), PROJ (reprojection and datum transforms; `ST_Transform(geom, srid)` is the only call a user needs; PROJ 6+ uses an internal EPSG DB and direct transforms, removing the 1–10 cm error of older pivot-through-WGS84 methods; time-dependent datums matter in fast-moving plates like Australia/NZ at ~7–10 cm/yr), SFCGAL (3-D/volumetric, straight skeleton, extrusion), GDAL (raster). Library versions decide capability, so check them.
- **DE-9IM** (dimensionally extended 9-intersection model) classifies a relationship by intersecting interior/boundary/exterior of A and B. You rarely use raw `ST_Relate` patterns except for **data QA rules** that booleans can't express (e.g. a valid dock touches the lake shoreline at exactly one end along a line). Day to day use boolean predicates: Intersects, Contains, Within, Touches (share boundary only), Overlaps (share interior), Crosses (mostly lines), Covers.
- **Constructive functions:** union/intersection/difference/symdifference/buffer, plus the aggregate `ST_Union(geom) GROUP BY state` (counties → states). **Generalisation trick:** positive buffer then negative buffer erodes fine detail, then Douglas-Peucker removes collinear vertices, giving clean overview shapes. **Partitioning trick (Peru):** `ST_GeneratePoints` → `ST_ClusterKMeans` → centroids → `ST_VoronoiPolygons` → intersect with outline = K similar sub-regions entirely in SQL. Also `ST_ClusterDBSCAN` (distance-tolerance clusters, e.g. 1000 m), MakeLine (points→track), BuildArea (lines→polygon), and `ST_Dump`/`DumpRings`/`DumpPoints` set-returning functions to explode collections (faster than `ST_GeometryN` loops).
- **Performance rules (the key takeaways):**
  1. **"Make it smaller."** A single 90,000-vertex merged polygon in a spatial join is slow. `ST_Subdivide` (default max 256 vertices per piece) typically gives **30–50× faster** joins: small pieces have tight bounding boxes (better index selectivity) and are cheap to de-TOAST from disk. Optionally simplify first (Douglas-Peucker for lines, Visvalingam-Whyatt for polygons) if boundary precision doesn't matter. Directly relevant to point-in-country/admin-polygon joins on GeoNames.
  2. **Physically order rows spatially.** `ORDER BY geom` now sorts by Morton (Z-order) and later Hilbert curve; `CLUSTER` the table on the spatial index so spatially-near rows share disk pages. Old x-coordinate sorting was poor.
  3. Send less to clients: simplify before returning; produce GeoJSON/SVG/MVT (`ST_AsMVT`) directly in the DB for a thin middleware.
  4. "Is PostGIS faster than Oracle/ArcGIS/SQL Server?" Depends on workload; the real win is expressiveness.
- **Raster:** don't store imagery in the DB for viewing (bigger DB, no speed gain over a good GeoTIFF). Do load continuous surfaces (elevation, temperature) as tiled chips to run raster–vector joins (mean slope per cutblock, buildings below +30 m sea level).
- **Ecosystem built on PostGIS geometry:** pgRouting (routing/TSP over any network), pgPointcloud (LiDAR), the TIGER geocoder (US), **h3-pg** (Uber H3 hex cells castable to geometry), `ogr_fdw` (expose shapefiles or remote SQL Server/Oracle tables as Postgres tables, editable). Topology extension for shared-edge data such as parcels.
- **Gotchas:** dateline-crossing data in plain lon/lat geometry breaks 2-D renderers; geography casts hide cost; huge polygons kill joins; 3-D vs 2-D distance must be chosen explicitly.

## Albi Wiedersberg (Overture Maps Foundation) with Matt Forrest — Grounding AI in the Real World (https://www.youtube.com/watch?v=EOqHCFitoC0)
**The bottleneck for location-aware products and AI is not the model but a complete, fresh, ID-stable places layer; Overture's bet is GERS, a permanent shared ID for every place, building, road and boundary.**

- **Problem framing.** A cited stat: AI assistants recommend only ~1.2% of local businesses; the long tail (the 20-year-old corner bakery) is invisible. Another member study (Voyager) found ~1 in 5 LLM answers about places hallucinated or out of date. Wiedersberg treats this as a **data problem / market failure, not a model problem**: no layer is complete and reliable enough for product teams to "bet the answer on". If a source only works some of the time, teams don't wire it in, and the product falls back to model memory. Shared infrastructure is the classic fix for market failures.
- **Places is the hardest theme because it decays fastest:** about 1 in 4 (or 5) businesses open, close or move each year. Static things (Eiffel Tower, neighbourhood names) are fine from memory; POIs aren't.
- **GERS (Global Entity Reference System).** Every entity in every Overture theme (places, buildings, transportation, divisions/boundaries, addresses, base) carries a stable GERS ID. Analogy: the **ISBN**. It didn't improve books, but it made everything built on top of books possible by ending per-shop reconciliation.
  - **Travel-specific:** one hotel can live in **100+ systems** (PMS, booking, loyalty, maps, ride-hail, tour operators), each with its own ID. Every link is a custom integration, and bookings fail when systems can't agree they mean the same property or sub-part of it. Overture's answer is **bridge tables** (your internal ID ↔ GERS): integrate once against the shared ID instead of N×N adapters. Wiedersberg's term is "a universal plug for travel". An LLM agent can also look up the ID to verify a hotel exists rather than hallucinating it. (Highly relevant to Trameter's supplier matching: Viator/RateHawk/Coras product IDs could be bridged via GERS rather than pairwise matching.)
  - You can't assign an ID to what you don't know exists, so **coverage gates adoption**. Below a threshold, builders keep their own stitched pipelines.
- **Data-quality example: operating status (open/closed).** It looks trivial in the schema but is very hard. Nobody has an incentive to report a closure; a business just stops. **Errors are asymmetric, with no safe default:** "wrongly open" sends a user to a shuttered door, "wrongly closed" hurts the merchant. No single open dataset solves it. In **May (2026) Overture reset every place's operating status to blank**, then from June rebuilt it from pooled **signals**. Example: Uber sends aggregated, anonymised "a ride ended at this place" counts (no traces or rider data). One signal is weak; combined with Meta, TomTom and TripAdvisor signals it forms a **"heartbeat"** for a place, especially for volatile categories (restaurants, bars, venues). Absence of rides alone doesn't prove closure, so multiple signals are needed. The June release refreshed status for ~8.5 M US places.
- **Coverage strategy:** region by region and category by category. US first (food & drink, shopping, landmarks, lodging), then Western Europe, with Asia and LatAm wanted. Releases are **monthly**. (Note for Sam: Nigeria/Africa coverage is likely weaker; validate before relying on it.)
- **Grounding layer = authoritative, citable entity layer** that fixes recall (no 98% missing) and controls hallucination about existence. Rich attributes can be fetched afterwards, and a good grounding layer helps choose which rich data to pull so the context window doesn't explode.
- **Spatial knowledge graph.** Overture themes today are flat lists ("what a thing is"); a graph adds "what it's part of / near". Real queries are mostly relational ("somewhere near the station to work for an hour before my train"). In a member prototype ("Orator", led by Daniel Smith), **geometry acts as the join key**: a place point inside a building polygon becomes a pre-baked edge, so an LLM doesn't have to run the spatial join on the fly. Edges carry metadata: relation type (inside / likely-same), confidence, derivation. Built by one automated pipeline for both the SF Bay Area and Manhattan (very different densities), with no manual curation, so it can be regenerated each monthly release. Without edge metadata and confidence you only have a list of assertions, not a graph.
- **Contrast noted by Forrest:** on-demand SQL (+ an MCP tool) can answer these relational questions, but pre-computing relationships scales better for products.
- **Rules of thumb:**
  - Key entities on stable shared IDs and keep bridge tables, rather than re-matching by name/coordinates each time.
  - Treat freshness as a first-class quality dimension and design for asymmetric error costs.
  - Fuse many weak signals rather than trusting one source.
  - Pre-compute spatial relationships (containment, proximity) as edges with confidence if downstream consumers are LLMs/agents.
  - The Overture Explorer and docs.overturemaps.org are free with no permission needed. Check sources and licences per theme there (note: places and some themes carry different licences, e.g. CDLA vs ODbL; verify before commercial use).

## Dewey Dunnington & Edzer Pebesma — FOSS4G 2021: Open source geometry on the sphere using S2 Geometry and R (https://www.youtube.com/watch?v=zqRhF2XM1CE)
**Lon/lat is not a plane: treating it as R² produces nonsense at the antimeridian, poles and high latitudes. S2 (Google's spherical geometry + cell index) does geometry on the sphere, and R's `sf` (≥1.0) now uses it by default for unprojected data.**

- **The problem.** Data often arrives as two lon/lat columns in a spreadsheet and gets plotted as x/y. In that plate-carrée view Antarctica becomes a long strip with an edge along −90 that doesn't exist, and Russia is cut in two at ±180. On a sphere Antarctica is one polygon containing the pole, and nothing crosses a "line" at 180.
  - A planar buffer on lon/lat (e.g. "within X of land" for ocean work) creates polygons beyond ±180 and south of −90, and its "distance" in degrees means different metres at different latitudes (a degree of longitude shrinks with cos(lat)). Meaningless for global analysis.
  - The Earth is an ellipsoid, not a sphere, but a sphere is a far better assumption than flat for global questions.
- **Bounding boxes break on the sphere.** A 4-number box for Alaska spanning −180..180 is huge and wrong (Alaska's Aleutians cross the antimeridian). And what the box's edges mean (constant-latitude lines vs geodesics) is undefined. Better approximations: a **union of a few S2 cells** (S2 "covering"). Each cell is a single 64-bit integer, so four cells cost the same bytes as a 4-double bbox but fit far better. Another option is a **cap**: centre + radius. (For Sam: bbox-based destination matching should handle boxes with xmin > xmax across the antimeridian, or use cell coverings.)
- **S2 library:** C++, object-oriented, from Google (original author Eric Veach). It is both a geometry engine and an indexing system (hierarchical cells on a cube projected onto the sphere), which is why many operations are fast. It powers BigQuery GIS, Google Earth Engine and Google Maps. Wiedersberg's "Google is 10 years ahead" point: they started from "how do we do the whole world" rather than GIS conventions. The R package `s2` wraps it in an R-friendly API; `sf` ≥1.0 sends ellipsoidal (unprojected) data to s2 instead of GEOS, and `sf_use_s2(FALSE)` toggles it off.
- **Mechanism/assumption:** **edges between two vertices are geodesics (great-circle arcs)**, not straight lines in lon/lat. A consequence: a large "rectangle" not on the equator no longer follows constant-latitude lines east–west, so large bboxes change shape. That breaks code that relied on planar behaviour.
- **Gotchas found via reverse-dependency checks (hundreds of packages):**
  - **"Invalid spherical geometry" errors.** Many datasets were built assuming planar lon/lat edges; they are valid in R² but invalid on S² (e.g. self-intersecting once edges become arcs, or rings that only close by wrapping). Repairing these was a big part of the work. Expect this when moving global data from planar tools into BigQuery/S2/geography types.
  - **Buffers on the sphere** are implemented as cell approximations (`s2_buffer_cells`): slightly larger, blocky; smooth needs many cells, which costs time and memory. There is no proper spherical buffer yet.
  - **Don't use buffer+intersect to select things within a distance.** It is inefficient even in GEOS. Use **`st_is_within_distance`** (s2 `dwithin`), which is index-backed, exact and fast. (General rule, matches PostGIS `ST_DWithin`.)
  - Slow cases (as of 2021): unioning many overlapping polygons; dwithin with very many points scaled poorly. Missing: segmentize.
  - Exporting clean spherical data back to planar tools (QGIS etc.) needs **antimeridian cutting and pole stretching**, which wasn't implemented yet and is non-trivial.
- **Standards critique:** OGC Simple Features doesn't let you declare whether an edge is planar or geodesic. The newer draft standard adds only spherical area/distance, not operations or representations. Pebesma's view: the rest of the open stack (PostGIS, QGIS, GRASS, Python/Shapely) should adopt spherical/ellipsoidal operations. Global datasets in Euclidean GIS make oceanographers and meteorologists "laugh". (Note: since 2021, Python has `spherely`/S2 bindings and DuckDB/BigQuery offer spherical options, but planar remains the default in GeoPandas/Shapely.)
- **Engineering lessons:**
  - Binding a large OO C++ library to a function-style standard is hard. Shipping S2 is harder than GEOS (no stable C API, not packaged on most systems, compiler churn).
  - Avoid feature creep by keeping the user-facing API close to core Simple Features. Pebesma planned to remove sf functions with no S2 equivalent.
  - Dunnington's pitch: interoperate via a **common columnar data structure (Apache Arrow / GeoArrow)** passing coordinates, rather than through more C wrappers.
- **Decision rule:**
  - Regional analysis: project to a suitable local CRS and use planar (GEOS).
  - Global analysis, or data near poles/antimeridian: use spherical (S2 / PostGIS geography / BigQuery), use dwithin rather than buffers, and validate geometries under spherical rules.

## Matt Forrest — PostGIS Explained: The Spatial Database Every GIS Pro Should Know (https://www.youtube.com/watch?v=FKzw9HJ6XYA)
**PostGIS is a transactional (OLTP) spatial backend: use it when data is shared, edited and long-lived. Use DuckDB, Sedona, GeoPandas or a warehouse for one-off or very large analytical batch work.**

- **Mental model.**
  - PostgreSQL is an open-source *transactional* database: inserts, updates and deletes are committed atomically. An edit made in QGIS on a PostGIS layer is saved immediately as a transaction, unlike a shapefile you must remember to save or export.
  - PostGIS adds geometry, geography and raster types plus the largest function set of any spatial DB (MySQL and others have fewer).
  - Spatial columns live in the same rows as attributes, so normal SQL filters (dates, categories) combine with spatial ops in one query.
- **Ecosystem:**
  - Desktop: QGIS, ArcGIS.
  - Python: GeoPandas `read_postgis`.
  - Extensions: pgRouting, H3 (h3-pg).
  - Analytical engines alongside: DuckDB (can attach to Postgres), Apache Sedona, warehouses.
  - Loading is usually done with GDAL/ogr2ogr; everything speaks SQL.
- **Scaling levers:** spatial (GiST) indexes, table partitioning. There is no hard row limit; it reaches "millions of features" comfortably, but at some point compute cost and batch size push you to a processing engine.
- **Views** store a query, not data. A view over a constantly-updated points table always returns fresh results to maps and apps. (Materialized views are the cached variant; not covered here.)
- **Decision matrix (Forrest's 0–5 need-for-PostGIS score):**
  - **One-off analysis (0):** files plus GeoPandas, DuckDB or Sedona; PostGIS setup is overkill even for a big join.
  - **Personal but recurring (~3):** worth it once you keep returning to the same data and want backups.
  - **Small team (4):** you need a DB for access control and a central truth instead of emailing files. Someone must act as part-time DBA.
  - **Enterprise (5):** Postgres is probably already deployed. Enabling PostGIS adds spatial to existing lat/lon columns, within the organisation's access controls.
- **Getting started:** run it once via **Docker** (the postgis image, some with pgRouting bundled). Managed clouds generally offer PostGIS; pgRouting may need extra work. Connect with QGIS/pgAdmin, run `CREATE EXTENSION postgis`, load with GDAL, then use `ST_` functions: `ST_MakePoint(lon, lat)` (note the x=lon order), `ST_Transform`, `ST_Buffer`, `ST_DWithin`, and point-in-polygon spatial joins with counts and averages. The main hurdle is networking: ports, auth and passwords.
- **Pitfalls:**
  - Data has to be loaded into tables. Querying remote GeoParquet in place is awkward, unlike DuckDB or Sedona.
  - Indexes speed queries but cost memory and storage, and may be overkill for tiny tables.
  - You must handle backups yourself unless the managed service does.
  - **Cloud cost:** an always-on DB bills compute plus storage even when idle, and grows with heavier operations.
- **When NOT to use PostGIS:**
  - Small, offline, simple work.
  - Heavy imagery or raster processing: use GDAL or raster tools (PostGIS is fine for reading and light raster work).
  - Large-scale batch processing or ML feature crunching: use Sedona or DuckDB.
  - Teams without DB skills to maintain, back up and update it.
- **OLTP vs OLAP framing:** warehouses (OLAP) are for big analytical retrieval; PostGIS (OLTP) is for collaborative, frequently-updated data and serving apps. A typical stack hands heavy prep to a processing engine and serves results from PostGIS (geospatial data engineering).
- **AI angle:** Snowflake acquired Crunchy Data and Databricks acquired Neon (2025) because agentic apps, like web apps, need a transactional DB per app to write state. PostGIS becomes the spatial backend for agents. LLM training/ingest volume still needs large-scale processing systems.
- **For Sam (16 GB laptop):** a GeoNames ~6M-row one-off transform fits DuckDB spatial/GeoParquet better. A shared, continuously-edited master locations table that apps query (nearest-airport, point-in-polygon) fits PostGIS with GiST indexes.

## Hello Interview — Proximity Search & Geospatial Indexes Explained (https://www.youtube.com/watch?v=dQXdSxn7d1g)
**A 1-D index can't keep 2-D neighbours together, so production systems either ship a disk-friendly spatial tree (R-tree, BKD) or encode lat/lon into one sortable integer (geohash, S2, H3) and reuse a plain B-tree.**

- **Why B-trees fail for location.** A B-tree range (age 20–30) is one seek plus a sequential read because sorted keys share pages. Two separate B-trees on lat and lon give a horizontal and a vertical strip; their intersection can still be millions of rows you must distance-check by hand. The core problem: **1-D sort order doesn't preserve 2-D closeness**. Two cabs five blocks apart in Midtown can be millions of rows apart when sorted by longitude alone.
- **Why the B-tree is still "the prize":** 50 years of tuning, self-balancing, page-packed sorted keys, one seek plus sweep, and every DB ships one. All approaches end with a balanced tree of disk pages you can range-scan.
- **Camp 1: custom spatial trees**
  - **Quadtree (Finkel & Bentley 1974).** Recursively split into 4 until each leaf holds few points; query by descending, then check neighbouring leaves for boundary cases.
    - Pro: adapts to density (Manhattan deep, an empty lake one cell).
    - Con: splits at the geometric midpoint, not at the data, so dense areas get very deep and **latency depends on where you query**; the hot regions are the slowest and worst case is hard to reason about.
    - Con: pointer-based, so off-RAM every hop risks a random disk read.
    - Still used for map tiles and game collision detection.
  - **KD-tree (Bentley 1975).** Binary; alternate the split axis per level (x, y, x…), split at the median so it stays balanced. Long the workhorse for nearest-neighbour search, but has the same pointer/disk problem. **BKD tree** (block KD): points packed into page-sized blocks, built once from a batch. Elasticsearch/Lucene geo fields use it. It is effectively write-once (immutable segments), so it suits slowly-changing data.
  - **R-tree (Guttman 1984).** The first spatial index designed for databases. It handles **shapes** (lines, polygons, zones), not just points, and is disk-friendly. Each object is wrapped in its **minimum bounding rectangle** (MBR, axis-aligned; a point is a zero-size rectangle); nearby MBRs are grouped into parents up to the root.
    - Balanced like a B-tree: all leaves at the same depth, so a predictable page count per query; nodes are page-sized; inserts split and merge.
    - **Trade-off:** sibling rectangles can **overlap**, so a query may descend several branches, and bad insertion causes lots of overlap and slow queries. **R\*-tree (1990)** improved the insertion heuristics to minimise overlap; most modern implementations derive from it.
    - Used by PostGIS (R-tree over GiST), SQLite, Oracle Spatial.
- **Camp 2: encoded keys (space-filling cells) in a plain B-tree**
  - **Geohash (Gustavo Niemeyer, 2008).**
    - Split the world into 32 cells (one base-32 char = 5 bits) and recurse.
    - Size by length: **5 chars ≈ 5 km**, **9 chars ≈ 5 m**.
    - A shared prefix usually means nearby. Because it is just bits, it can be stored as text (Postgres, `LIKE 'dr5ru%'` = B-tree prefix scan) or as an integer.
    - **Redis GEOADD** stores a 52-bit geohash as the score in a sorted set; GEOSEARCH is a ZRANGEBYSCORE range scan.
    - **Boundary problem:** points 1 m apart across a cell edge get different prefixes (dr5ru vs dr5rg), so a naive prefix scan misses the nearest driver. **Fix: the 3×3 trick.** Query the cell plus its 8 neighbours, then **post-filter by exact distance** (drop the corners).
    - **Globe problem:** geohash treats lon/lat as a flat rectangle. A degree of longitude is ~111 km at the equator and → 0 at the poles, so cells are squares at the equator and slivers near the poles.
  - **S2 (Google, ~2011).** Wraps the sphere in a cube, projects onto 6 faces and subdivides each with a Hilbert curve.
    - Cells are roughly equal-size everywhere. Each cell has a **64-bit ID**, hierarchical (truncate the ID to get the parent).
    - Powers MongoDB `2dsphere`; it correctly handles **polygons crossing the antimeridian**, where flat indexes think the polygon wraps the whole globe.
  - **H3 (Uber).** (note: the video says open-sourced 2008; it was actually 2018.) Hexagons, because a square has 4 edge neighbours plus 4 farther corner neighbours, while a **hexagon has 6 equidistant neighbours**. That gives clean math for k-ring ("within n cells") analytics and heatmaps.
    - ~7 children per parent (approximate, not exact nesting); **16 resolutions** from continent-size down to ~1 m²; 64-bit hierarchical IDs.
    - Hexes aren't sortable themselves; their IDs follow a deterministic walk so numerically close IDs are usually close in space.
    - Uber dispatch: drivers report positions, each mapped to a ~200 m cell. A rider's cell plus its 6 neighbours (k-ring) is searched, expanding rings until a driver is found.
- **Decision rule (the summary):**
  - **Shapes and exact geometry** (does highway X intersect country Y? which zone contains this address?): use an **R-tree** (PostGIS GiST) or BKD. Needs a spatial extension; writes are costlier (R-tree insertion heuristics; BKD effectively immutable), so it fits less well for millions of moving points.
  - **Points at scale with heavy writes:** use **encoded keys** (geohash/S2/H3) in any B-tree or sorted set. No extension, writes are one integer update (how Uber handles millions of pings/sec), and you can coarsen by truncating. Costs: mostly points only, and boundary artifacts mean you must query neighbour rings plus an exact-distance post-filter.
- **For Sam:**
  - Static GeoNames/airport/terminal matching with polygons: use R-tree (PostGIS, or DuckDB's R-tree index).
  - Fast bulk point-to-point candidate generation on a laptop: H3 or geohash cell keys with neighbour rings in a hash/merge join, then exact haversine filtering. At high latitudes prefer S2/H3 over geohash.

## Josiah Parry — Geohash: the algorithm inside and out, Part 1 (https://www.youtube.com/watch?v=vGKs-c1nQYU)
**A geohash is a Z-order (Morton) code: binary-partition lon and lat, interleave the bits, then base-32 encode in 5-bit groups. The result is a string whose shared prefix bounds how far apart two points can be.**

- **Context.** Twitter's open-sourced recommendation code tracks user location as geohashes (also used for country blocklists and media restrictions). Geohash sits in the same family as H3 and S2, which R `sf` users already rely on via s2.
- **Precision table (from the talk):** 1 char ≈ 5,000 km × 5,000 km; 12 chars ≈ 3.7 cm × 1.9 cm (he says "32 mm × 18.6 mm"). Max 12 characters, because 12 × 5 = 60 bits fits in 64.
  - Example distances: points differing only in the last of 12 chars were 0.13 m apart; sharing only 6 chars, 141 m apart in his example.
  - Precision 6 is roughly a 1.2 km × 0.6 km cell. He says sharing 6 chars means "at most ~0.75 mi apart". (note: the cell diagonal is ~1.35 km ≈ 0.84 mi, so treat it as approximate. The converse does not hold: nearby points across a cell edge may share few or no characters; see the Hello Interview notes on the 3×3 neighbour trick.)
- **Why it's useful for databases:** a DB is good at string/prefix matching and bad at an N×N pairwise distance matrix (1 M × 1 M = 10¹² distances). Use a prefix match as a cheap **candidate filter**, then rank or verify exactly.
- **Worked example (R):** at Park Street Station, Boston, query Google Places for cafés, geohash each result, count the shared leading characters with the station's 12-char geohash, and keep those sharing ≥6. This filters to nearby cafés without computing distances. (note: shared-prefix length is a coarse proxy; rank by true haversine distance after filtering.)
- **History:** G. M. Morton (IBM, 1966) described the same recursive gridding / Z-order "file sequencing" for a geodetic database. Gustavo Niemeyer reinvented it as geohash in 2008. Z-order curves are a kind of **space-filling curve** mapping 2-D/3-D to 1-D, also used in Spark, Cassandra and similar systems to speed up range search. (Ties to PostGIS's Morton/Hilbert `ORDER BY geom` clustering in the Ramsey notes.)
- **Mechanism, step by step:**
  1. **Bit interleaving = Z coordinate.** Write x and y in binary and alternate bits x₀y₀x₁y₁…; interleaving two n-bit values produces 2n bits (example: x=23, y=42 as 8-bit values gives a 16-bit Z).
  2. Floats are 64-bit, so interleaving two would give 128 bits; geohash therefore encodes each axis to **32 bits** so the interleaved code fits 64 bits.
  3. **Encoding each axis is binary search, not raw float bits.** Longitude starts on [−180, 180). Split at the midpoint: value in the left half [−180, 0) gives bit 0, the right half [0, 180) gives bit 1. Narrow to that half and repeat, 32 times. Latitude does the same on [−90, 90). Intervals are left-inclusive, right-exclusive. Example: lon 19 gives 1 (≥0), then 0 (<90), and so on.
  4. **Interleave, longitude first** (lon bit, lat bit, lon bit…). The order matters for compatibility.
  5. **Base-32:** chunk into 5-bit groups (values 0–31), then map each through the geohash alphabet `0123456789bcdefghjkmnpqrstuvwxyz` (digits plus letters minus a, i, l, o). Worked example: bit groups 26, 2, 20, 14, 9, 0 give "u2fn90" (0-indexed lookup).
- **Takeaways:**
  - A geohash is just a stack of bits. Store it as an integer (Redis 52-bit score) or as text; the sort order is the same.
  - Truncating a prefix gives the parent cell.
  - Cells are lon/lat rectangles, so their metric size depends on latitude (narrower east–west toward the poles).
  - Geohash is a candidate-generation index, not a distance function.

## Luc Anselin (GeoDa / U. Chicago) — Week 1a: What is spatial analysis? (Introduction to Spatial Data Science) (https://www.youtube.com/watch?v=JwHxJsesG2Y)
**Spatial analysis differs from ordinary analysis because data is not locationally invariant: shuffle values among locations and what you learn changes. Real spatial data science puts location, distance and interaction into the method itself, not just lat/lon as extra features.**

- **Definition.** Going beyond making a map: transforming, manipulating and applying analytical methods to geo-referenced data to add value. Classic example: John Snow's 1854 cholera map and the Broad Street pump (note: the transcript says "1984"). Modern example: newspaper homicide maps. "Spatial" can also mean non-geographic spaces such as networks. It is one stage of **knowledge discovery from databases (KDD)**: raw data → information → knowledge (→ "wisdom", tongue in cheek).
- **The four questions of spatial analytics:**
  1. **Where** do things happen (patterns, clusters, hot spots, disparities)?
  2. **Why** there (decisions, causal mechanisms)?
  3. **Interaction**: how does what happens in one place affect others, and how does context shape outcomes? This is the critical concept.
  4. **Where should** things go (normative/optimisation: clinic or fire-station siting). For Sam: hub or pickup placement.
- **Locational invariance test.**
  - Non-spatial analysis is invariant to reshuffling locations; spatial analysis is not.
  - Milwaukee example (the most segregated US city): a real map of % African-American by tract next to a map with the same values randomly reshuffled. The **histograms are identical**, because a histogram is aspatial. Only the map, and a **spatial autocorrelation (Moran) scatterplot**, distinguish them: flat slope = spatial randomness; steep positive slope = similar values cluster together.
  - NYC rents: a box plot flags 6 high-rent outliers but not where they are. A **box map** puts them all in Manhattan and **adjacent to each other**, which you couldn't know a priori.
- **Progression:** mapping/geovisualisation → **exploratory spatial data analysis (ESDA)** → modelling (explanation, prediction, simulation, optimisation, causation). This course covers mostly the first two.
- **Spatial data science context:**
  - Computational social science treats computation as a third paradigm (after experiment and observation); data-intensive science is the "fourth paradigm" (Microsoft Research book), driven by big industry data (Facebook, Twitter, Google).
  - Data science = statistical computing plus how to access, transform, manipulate, explore, visualise and reason about data.
  - The Venn diagram is hacking + maths/stats + **substantive expertise**; drop any one and you get traditional research, ML, or the "danger zone". Domain knowledge is often forgotten.
- **Key distinction:**
  - **Data science applied to spatial data** = lat/lon as just more variables, same methods.
  - **Spatial data science** = location, distance and spatial interaction change the methodology. It needs specialised storage, retrieval, exploration, analysis and learning (spatial statistics, spatial databases, geocomputation).
  - "Putting lat/lon into an ML model doesn't make it spatial analysis." (For Sam: in ML on places, use spatially-aware features such as neighbour counts, distance to nearest X and spatial lags, and use **spatial cross-validation**. Random splits leak because nearby points are correlated.)
- **Process (from R for Data Science):** import → tidy → loop of transform/visualise/model → communicate. Tasks include wrangling, **integration of multiple sources**, exploration and pattern recognition, visualisation and modelling. **Be eclectic with tools**: GeoDa, R and Python each fit different steps.
- **Worked pipeline (with Sarah Williams, MIT: "digital neighbourhoods" in NYC).** One week of full Twitter firehose plus Foursquare check-ins: ~6,000 JSON files, 5 M+ messages, 20+ GB.
  1. Python converted the JSON to CSV.
  2. Loaded into **PostGIS**.
  3. Aggregated points to census block groups (spatial join + group-by) in PostGIS.
  4. Computed **local Moran's I (LISA)** to find hot spots, cold spots and **spatial outliers** (places very different from their neighbours).
  - This is a reusable pattern: raw semi-structured → tabular → spatial DB → aggregate to areal units → local spatial statistics → map.
- **Rules of thumb:**
  - Always map, not just summarise; check for spatial autocorrelation before assuming independence.
  - Pick aggregation units deliberately, since the units affect results (relates to the MAUP, not named here).
  - Combine tools rather than forcing one to do everything.

## GeoDev — All you need to know about GeoParquet (https://www.youtube.com/watch?v=jDffNgXtKlQ)
**GeoParquet is Parquet (columnar, compressed) plus a standard for storing geometry as WKB with CRS metadata. Expect files several times smaller and reads roughly 10–30× faster than Shapefile/GeoJSON at the million-feature scale.**

- **What it is.** A cloud-native vector format: Apache Parquet with geometry columns encoded as **WKB** (well-known binary) plus file-level metadata (CRS, geometry types, bbox). Parquet gets its size and speed from columnar layout, compression and encodings. Supported by GDAL/OGR, GeoPandas (`read_parquet`/`to_parquet`, needs `pyarrow`) and DuckDB; backed by Microsoft, Planet, Carto and others. Spec version at recording was 1.0.0 / 1.1 (2024).
- **Claimed advantages:**
  - Small files.
  - Fast read-heavy analytics.
  - Scales to big-data environments.
  - Standard WKB encoding, easy to share between organisations and tools.
  - Multiple CRSs and multiple geometry columns per file.
  - Partitioning support, so data can be processed in chunks.
  - Pairs with cloud-native engines like DuckDB.
- **Benchmark (NZ building outlines, 3.3 M multipolygons, EPSG:2193 NZTM2000):**

  | Format | Size | Read (GeoPandas) | Write |
  |---|---|---|---|
  | GeoParquet | ~420 MB | ~7–8 s | ~14 s |
  | Shapefile | ~2.74 GB | ~53 s | ~1.5 min |
  | GeoJSON | ~3.25 GB | >4 min | >4 min |

  That is **~6.5–7.7× smaller** and **~7× faster than Shapefile, ~35× faster than GeoJSON** to read. (For Sam: on a 16 GB laptop, keep GeoNames/OSM extracts as GeoParquet, not GeoJSON/CSV-WKT.)
- **Workflow shown:** `gpd.read_parquet()` → attribute filter (`town_city == 'Marton'` gives 7,910 rows) → `.buffer(10)` in a projected CRS, so it means 10 metres (NZTM is metric; buffering in EPSG:4326 would be 10 *degrees*) → `.head(5000)` → `.plot()` → write with `to_parquet()` or `to_file(driver=...)`.
- **Gotchas:**
  - **Shapefile truncates field names to 10 characters** (the warnings in the demo). It also has a 2 GB per-component limit, though not mentioned here.
  - GeoJSON is huge and slow; it's an exchange format, not a working format. (Its RFC 7946 version also mandates WGS84.)
- **Limitations listed:**
  - Still evolving, with fewer tutorials.
  - Integration with legacy tools is harder than Shapefile/GeoJSON.
  - Overkill for small datasets.
  - (note: by 2025–26 GeoParquet is widely supported, including QGIS via GDAL ≥3.8, DuckDB spatial, Overture's distribution format and BigQuery/Snowflake loaders; Parquet itself added native GEOMETRY/GEOGRAPHY logical types in 2025, which GeoParquet 2.0 aligns with.)
- **Missing from the video, worth knowing:**
  - **Row-group bbox statistics / a "covering" bbox column (GeoParquet 1.1)** enable spatial predicate pushdown when rows are spatially sorted (Hilbert/geohash order). Combined with **column projection**, you read only the needed columns and row groups. This is why Overture can be queried remotely by bbox from S3.
  - **Partitioning** (e.g. by country or H3 parent) lets a laptop process chunks within RAM.
  - GeoPandas reads the whole file into memory. For >RAM data, use DuckDB, `pyogrio` with bbox/where filters, or `pyarrow` row-group iteration.

## Daniel Silk (LINZ, Land Information New Zealand) — 10 Things I've Learnt About PostGIS (https://www.youtube.com/watch?v=93bX1AO95nY)
**Practitioner lessons from a national mapping agency: spatial indexes do the heavy lifting, subdivide big polygons, keep keys and indexes through every step, and push business rules and integrity into the database.**

- **Context.** LINZ has used PostGIS since 2004 (geodetic team) and since ~2014 has used QGIS as the front end with PostGIS as the back end. Systems include the gazetteer (official place names), addressing, roads, building outlines and aerial survey management.
- **1. Spatial indexes are your greatest asset.** The index stores each geometry's bounding box. A complex building (Te Papa) becomes a simple rectangle, and boxes are grouped into progressively larger rectangles (R-tree over GiST). Predicates first compare the cheap boxes and only fetch and test the full geometry for survivors. Create the index (`CREATE INDEX ... USING GIST(geom)`) and run `ANALYZE` so the **query planner** has statistics and actually uses it.
- **2. It's fast (mostly because of #1).** Centroids of ~300,000 Wellington building outlines took ~23 s in QGIS 3 and ~3 s in PostGIS.
- **3. If it's slow, there's probably a better way.** His rule: if a query runs for ~2 minutes, kill it and rethink (missing index, wrong approach). Case: LiDAR QA, finding which of **1.1 M national grid squares** fall in the latest survey polygon. `ST_Intersects` of one huge polygon against many small squares took **~2 minutes**. After `ST_Subdivide` on the big polygon (pieces with bounded vertex counts, smaller where the edge is detailed), the same intersects took **~0.3 s**, roughly 400× faster. Subdivided pieces have tight bboxes, so both indexes become selective. (Same lesson as Ramsey's "make it smaller"; directly applicable to country/admin polygons vs GeoNames points.)
- **4. Consistency is key: keep a primary key and a spatial index at every step.** Set-returning ops break uniqueness; `ST_DumpPoints` on a polygon yields 5 rows sharing one ID. Fix: wrap the query and add `ROW_NUMBER() OVER () AS id` for a fresh key. Re-index intermediate tables, or a later step in the pipeline silently goes slow.
- **5. Encapsulate business logic in stored SQL functions** and reuse them. Example `suburb_intersecting_polygon(geom)` for assigning a building to one suburb:
  1. Get candidates with `ST_DWithin(geom, suburb, 1000)`. This is index-assisted, cutting thousands of suburbs to about 5. It also handles wharf buildings that sit **outside every suburb polygon** (recently extended wharf).
  2. If the building straddles a boundary, rank by **area of intersection** (largest wins).
  3. Otherwise order by distance ascending.
  4. `LIMIT 1`.
  - The function is then called inside bulk-import inserts. This is exactly the pattern for Sam's "assign each place/airport/terminal to a destination" job: dwithin candidates → overlap area or nearest distance → pick one, which handles points just outside polygons (coastal airports, ports, piers).
- **6. Constrain geometry types.** A bare `geometry` column accepts points, lines and polygons mixed, which most clients can't display. Declare `geometry(Point, 4326)` and the insert errors on the first non-point. Richer integrity: a **CHECK constraint** calling a function that tests `ST_Intersects AND NOT ST_Touches` against existing rows (≈ "no overlaps") rejects overlapping polygons at insert. If a business rule says "X must never happen", enforce it in the database.
- **7. Community functions beyond core:** e.g. `normalize_geometry` (Kaspar's): spike removal (out-and-back slivers) and vertex simplification with tunable parameters. Look before you write your own.
- **8. LINZ publishes open-source Postgres extensions and code** (e.g. the building-outlines system) on GitHub.
- **9. pgAdmin 4 geometry viewer.** Select a geometry column and click the eye icon to see geometries. There is an OSM basemap only when the data is in **EPSG:4326**: `ST_Transform(geom, 4326)` for display, otherwise you get a blank background. It can show mixed points, lines and polygons, which helps debug intersections that return unexpected geometry types (e.g. lines when you expected polygons; use `ST_CollectionExtract`).
- **10.** Solving spatial problems efficiently in SQL is satisfying, and LINZ keeps moving more projects onto it.
- **Checklist distilled:**
  - GiST index + ANALYZE.
  - Subdivide big polygons before joins.
  - DWithin for candidates, not buffer.
  - New primary key after dump/explode.
  - Typed geometry columns with SRID.
  - CHECK constraints for topology rules.
  - Reusable SQL functions for assignment logic.
  - Transform to 4326 only for viewing.

## Q-tips (South African GIS instructor) — GIS Coordinate Reference Systems (https://www.youtube.com/watch?v=WWp1k0SlMUU)
**A CRS is either geographic (angles on an ellipsoid positioned by a datum) or projected (that geographic CRS flattened by a projection with a chosen line of tangency). Distortion grows with distance from the line of contact, so choose a central meridian near your study area.**

- **Geographic ("spherical") CRS.**
  - Locations are angles measured from Earth's centre. Latitude is the angle up from the equatorial plane (e.g. 40° N); longitude is the angle east/west of the Greenwich prime meridian (e.g. 95° W).
  - The point sits where the two intersect.
- **Shape hierarchy:**
  - **Geoid:** Earth's true, irregular "potato" shape, set by gravity and rotation.
  - **Ellipsoid:** the mathematical equation approximating it. It can never fit the geoid perfectly everywhere.
  - **Datum:** positions the ellipsoid relative to Earth's centre to get the best fit for a region. Example: South Africa uses **Hartebeesthoek94**, which fits southern Africa well but would be wrong for Spain. (note: Hartebeesthoek94 uses the WGS84 ellipsoid and is effectively aligned with ITRF/WGS84; the "local fit" story is truer of older datums like Cape/Clarke 1880 or NAD27.)
- **Projected CRS:** coordinates for a flat surface (paper, screen).
  - **Light-bulb analogy:** a lamp at the globe's centre casts the wireframe onto a plane, cylinder or cone. (note: only a few projections, like gnomonic, are literally perspective; most are mathematical formulas, but the developable-surface idea holds.)
  - **Families:** planar/azimuthal, **cylindrical** (normal, **transverse**: cylinder on its side, **oblique**), and **conic**.
  - **Line of tangency/contact** is where the surface touches the globe; distortion is minimal there and **grows with distance from it**. The region projected stretches compared with its true size, which corrupts area, perimeter and length calculations.
  - So pick the central meridian closest to your study area. South Africa's Transverse Mercator "Lo" zones use odd meridians: Cape Town **Lo19**, Pretoria **Lo29**, East London **Lo27**.
- **Every projected CRS is built on a geographic CRS.** You need the round model before you can flatten it.
- **Full recipe (4 parts):** ellipsoid (**WGS84**) + datum (**Hartebeesthoek94**) + projection (**Transverse Mercator**) + line of tangency/central meridian (**19° E** for Cape Town).
  - (note: the video calls WGS84 "fairly recent"; it dates from 1984, with periodic realisations since. Generalise the recipe to UTM: TM with a 6°-wide zone and scale factor 0.9996.)
- **For Sam:**
  - For metre-accurate distances or areas in a region, project to a local TM/UTM zone.
  - For global point-to-point distance, use geodesic or haversine on lon/lat rather than any single projection.
  - Never measure in degrees.

## Dr Christopher Sutton (Sutton Geospatial) — Coordinate Reference Systems: Introducing CRS (https://www.youtube.com/watch?v=ZaP5hnr8EM4)
**A CRS (ellipsoid + datum, optionally + projection) is what gives coordinates meaning. Layers must share one CRS to align, and measurements are only valid in a CRS suited to the purpose and region.**

- **Components:**
  1. **Reference ellipsoid:** a smooth mathematical approximation of Earth's shape, the basis for geodetic calculations.
  2. **Datum:** fixes the ellipsoid's origin, orientation and dimensions relative to Earth. Latitude and longitude are only defined *with respect to a datum*, so the same lat/lon under different datums can be metres or even hundreds of metres apart.
  3. **Projection:** transforms 3-D positions onto a 2-D plane and always introduces distortion. Choose it by purpose and region.
  - **Geoid** (equipotential gravity surface ≈ mean sea level) is not part of most CRSs but informs datum design and vertical systems.
- **Why it matters:**
  - **Alignment:** layers in different CRSs misalign (his exaggerated Illinois/Iowa interstate example). Integrating imagery, survey and map data requires transforming to a **common CRS** first.
  - **Measurement:** distances, areas and directions are only correct in a suitable CRS. The wrong one gives significant errors.
  - **Analysis:** proximity, network analysis and interpolation depend on correct spatial relationships.
- **Four types of coordinate system:**
  - **Geographic (GCS):** global; lat/lon in **angular units (degrees)**; origin at the equator × prime meridian. The foundation for all others; used for global positioning (e.g. EPSG:4326 WGS84).
  - **Projected (PCS):** 2-D Cartesian derived from a GCS via a projection. Coordinates are **eastings/northings in linear units (metres/feet)**; used for mapping, analysis and measurement locally or regionally. US examples: **UTM** and **State Plane**.
  - **Local (LCS):** project or indoor systems with an origin at a building corner or survey marker; not tied to a global datum.
  - **Vertical:** heights and depths relative to mean sea level, the geoid or another reference surface.
- **Rules of thumb:**
  - Check every dataset's CRS on ingest. Reproject all layers to one CRS before overlay or joins.
  - Measure in a projected (metric) CRS appropriate to the area, or use ellipsoidal/geodesic calculations.
  - Remember that a bare lat/lon pair without a datum is ambiguous. In practice assume WGS84 for GPS, GeoNames and OSM, but verify for national or survey data.
- (Series intro; ellipsoids, geoid and datums are covered in later videos not in this group.)
