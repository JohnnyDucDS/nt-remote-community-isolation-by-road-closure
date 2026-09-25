# NT Remote Community Road-Isolation Validation

A methodology for assessing whether current road closures could disconnect remote Northern Territory communities from the modeled external vehicle-road network.

This repository includes the **getting data through API request, analysis logic, validation process, QGIS outputs, CSV results, and final interpretation**. The development Python scripts are not included.

The main idea is simple:

```text
Community locations
        +
OSM vehicle-road network
        +
Current NT road closures
        ↓
Build normal road network
        ↓
Identify genuine external community exits
        ↓
Apply current road closures to an event network
        ↓
Compare connectivity before vs after closures
        ↓
ACCESS_REMAINS / POTENTIAL_ROAD_ISOLATION / BASELINE_REVIEW
```

---

## Data sources

- **BushTel** — remote community information such as community ID, name, population, coordinates, region, services, connectivity, road accessibility and aerodrome availability.
- **OpenStreetMap / Geofabrik** — Northern Territory road-network data.
- **Northern Territory Road Report** — current road closures, restrictions, obstruction causes and closure start/end coordinates.
  (This project works on the Road closures dataset at 19/09/2026)

Community service fields use the project convention (BushTel):

- `1` = available / yes
- `0` = unavailable / no
- `0.5` = not available / unknown in the source data

---

## AI Use Declaration

This project was developed through ChatGPT collaborative workflow.

The project idea, analysis logic, assumptions, workflow design, validation decisions, testing, interpretation of results, and final methodological choices were directed by the user.

ChatGPT was used primarily to:
- generate and refine Python code based on the user's instructions and logic;
- explain NetworkX, GeoPandas, QGIS, CRS, graph connectivity, and road-closure modelling concepts;
- help debug errors.

All generated code and analytical outputs were reviewed, tested, and validated by the user using QGIS, CSV outputs, and visual inspection before being accepted into the final workflow.

The Python source code is upon request. This repository focuses on the methodology, validation logic, intermediate/final outputs, and results.

## Tools

- **QGIS** — mapping, reprojection, inspection and manual validation.
- **NetworkX** — graph creation, connected components, path finding and network-connectivity testing.
- **GeoPandas** — geospatial processing.
- **Pandas** — CSV processing and result tables.
- **Shapely** — buffering, snapping and geometry operations.
- **Python** — used to run the analysis workflow during development.

---

# Workflow

## Step 0 — Prepare communities and the vehicle-road network

### Step 0A — Load communities

The source community CSV contains longitude and latitude coordinates, so it is first loaded in QGIS using:

```text
X = Longitude
Y = Latitude
CRS = EPSG:4326 (WGS 84)
```

EPSG:4326 is used here because the original coordinates are geographic longitude/latitude values.

### Step 0B — Reproject communities to EPSG:9473

The community layer is exported to **EPSG:9473 — GDA2020 / Australian Albers**.

The reason is that the analysis later uses distances and buffers. EPSG:9473 uses metres, so:

```text
buffer(1000) = 1 km
road length   = metres
point-to-road distance = metres
```

### Step 0C — Download OSM road data

Northern Territory OpenStreetMap data was downloaded through Geofabrik:

https://download.geofabrik.de/australia-oceania/australia.html

### Step 0D — Keep the OSM road layer

The OSM GeoPackage contains many layers. Only:

```text
gis_osm_roads_free
```

is required for this road-isolation analysis.

The road layer is also reprojected to EPSG:9473 so all geometry calculations use the same metre-based CRS.

### Step 0E — Filter to vehicle usable roads
```text
run 
filter_vehicle_road_network.py
```

The analysis keeps road classes that a vehicle could plausibly use.

**Kept**

```text
trunk
trunk_link
primary
primary_link
secondary
secondary_link
tertiary
tertiary_link
unclassified
residential
living_street
service
track
track_grade1
track_grade2
track_grade3
track_grade4
track_grade5
```

**Removed**

```text
footway
path
cycleway
steps
pedestrian
bridleway
busway
unknown
```

Tracks are retained because a 4WD or unsealed track may genuinely be the only vehicle connection to a remote community. Removing these roads could incorrectly make a community appear disconnected.

Service roads are also retained because some may provide vehicle access. Local service roads are dealt with later by the 1 km community access-zone and external-exit classification.

From this point onward, the analysis uses the filtered **vehicle-road layer**.
<img width="666" height="964" alt="image" src="https://github.com/user-attachments/assets/3136bf36-8011-4f6f-8bd2-eecd7ada0557" />

(195 selected remote communities and vehicle road network)


---

## Step 1 — Convert the OSM road network into a NetworkX graph
```text
run
build_road_graph_pkl.py
```

The filtered vehicle-road layer is converted into a graph.

In the graph:

```text
road coordinate / vertex = node
road segment between two nodes = edge
```

A road can therefore look like:

```text
A ─ B ─ C ─ D
```

where `A`, `B`, `C` and `D` are nodes and each connection is an edge.

A **connected component** is a group of nodes that can reach one another through connected edges. The largest connected component represents the largest interconnected modeled road network in the dataset; it is not literally one road or highway.

A diagnostic baseline was also produced to show which modeled road component each community was associated with.
<img width="486" height="244" alt="image" src="https://github.com/user-attachments/assets/e21aaf02-9334-43c7-bc14-b4d1faf0c518" />

## Analysis Scope

The source community dataset contains 195 communities.

For this road-isolation model, only communities with:

`Accessible by road = 1`

are included in the NetworkX vehicle-road analysis.

Communities recorded as `Accessible by road = 0` or `0.5` were reviewed separately in QGIS and, in this dataset, corresponded to island communities. These communities were excluded from the vehicle-road isolation model because their accessibility depends on other transport modes such as air or sea rather than the mainland road network.

This leaves 168 road accessible communities for the analysis.

<img width="766" height="519" alt="image" src="https://github.com/user-attachments/assets/5e319a11-bcd9-4da4-a2ba-9a7e275dff5d" />

---

## Step 2 — Create a 1 km community access zone

A **1 km analysis buffer** is created around each road-accessible community.

This is not an official community boundary. It is an analysis zone used to reduce the influence of small internal streets because the goal is to identify roads that lead **outside** the settlement.

Every vehicle road crossing the 1 km boundary becomes a **raw exit candidate**.

```text
run
build_communities_access_zones.py
```

Main outputs:

```text
community_access_zones_1km.gpkg
community_exit_candidates_raw.gpkg
community_exit_candidates_raw.csv
```

<img width="697" height="704" alt="image" src="https://github.com/user-attachments/assets/1acefd83-74fb-4681-a5e9-0a58fcecd0de" />


---

## Step 3 — Insert community boundary crossing points into NetworkX

The raw road-boundary crossing points are inserted into the road graph as graph nodes.

If a crossing falls in the middle of an existing edge:

```text
Before:
A ───────── B

After:
A ─── X ─── B
      ↑
 community boundary crossing
```

The original edge `A-B` is replaced by `A-X` and `X-B`.

If the crossing coordinate already exists as a graph node, that existing node is reused rather than creating another node at the same location.

```text
run
insert_communities_access_node_buffer.py
```

Main outputs:

```text
community_exit_nodes.csv
road_graph_with_exits.pkl
```

<img width="425" height="413" alt="image" src="https://github.com/user-attachments/assets/fea17c5f-3b3c-4bfa-891d-7b1a2cbc1035" />

<img width="844" height="259" alt="image" src="https://github.com/user-attachments/assets/d696fe2f-0f44-4f18-9acf-3e27b9f4f11b" />


---

## Step 4 — Classify genuine external exits

A road crossing the 1 km boundary is not automatically a useful external route. Some roads cross the boundary and then end locally.

The exit nodes were already inserted into NetworkX in Step 3. Step 4 adds a classification so only genuine external exits connecting the main road are used in the final isolation test.

For each community, road nodes inside the 1 km analysis zone are temporarily removed. The boundary crossing nodes remain.

This prevents NetworkX from doing something misleading such as:

```text
dead-end exit
     ↓
travel back through the community
     ↓
use another exit
     ↓
reach the main road network
```

After removing the internal community road nodes, NetworkX recalculates the outside connected components.

A boundary point is classified as:

- **EXTERNAL_EXIT** — the point remains connected to the largest outside road component.
- **LOCAL_OR_DEAD_END** — the point is connected only to a smaller local/dead-end component.

Example logic:

```text
Exit 1 → small component → LOCAL_OR_DEAD_END
Exit 2 → large external component → EXTERNAL_EXIT
Exit 3 → small component → LOCAL_OR_DEAD_END
```

Only `EXTERNAL_EXIT` nodes are used in the final isolation test.

```text
run
classify_external_exits_all.py
```

Main outputs:

```text
community_external_exits.gpkg
community_external_exits.csv
community_external_exit_summary.csv
```

<img width="940" height="189" alt="image" src="https://github.com/user-attachments/assets/e93c863e-e0e3-4aae-8136-968fc9c0ea09" />

<img width="940" height="102" alt="image" src="https://github.com/user-attachments/assets/0b86407c-cf01-4cf5-8525-f9e17440e0f0" />

<img width="940" height="781" alt="image" src="https://github.com/user-attachments/assets/7f505a39-51ff-4405-b177-026fe431b4db" />
(Red dot is local dead-end while green dot is external exit)


---

## Step 5 — Collect NT Road Report closures and match endpoints to OSM roads

Current `Road Closed` and `Impassable` records are collected from the NT Road Report.

Each closure can contain a reported start coordinate and end coordinate. These coordinates do not always lie exactly on the OSM road geometry.

Therefore each reported endpoint is matched to a nearby OSM vehicle road. The source-to-road distance is retained as a validation measure.

Conceptually:

```text
NT Road Report point ●
                     |
                     | match distance
                     ↓
OSM road ─────────── X ───────────
```

A larger match distance does not automatically mean the record is wrong, but it should receive more attention during validation. Intersections, parallel roads and approximate source coordinates can cause the nearest-road match to select the wrong OSM feature.

```text
run
get_road_repots.py
prepare_road_closure_endpoints.py
```

Main outputs:

```text
road_closure_start_matches.gpkg
road_closure_end_matches.gpkg
```
<img width="905" height="459" alt="image" src="https://github.com/user-attachments/assets/0ad74cea-7e18-4cf5-afce-05ce10c2c807" />

<img width="940" height="246" alt="image" src="https://github.com/user-attachments/assets/27d62eea-b4d4-4ba5-8fc5-784b87efebe5" />

<img width="940" height="71" alt="image" src="https://github.com/user-attachments/assets/384a3f45-c60e-4e55-a528-6049aca93b3b" />

The match distance is the distance between NTG road report coordinate with the OSM road. If the distance is not genuinely large (1000m) we then reference that coordinates onto the OSM road. 


---

## Step 6 — Insert closure start/end nodes into the road graph

The matched start and end points are inserted into the same working NetworkX graph used for the community exits.

The logic is the same as Step 3.

If the endpoint falls in the middle of an edge:

```text
A ───────── B
```

becomes:

```text
A ─── X ─── B
      ↑
 closure endpoint
```

An **EXISTING NODE** means the endpoint landed on a coordinate that was already represented by a NetworkX node.

A **NEW NODE** means the original edge had to be split at the endpoint location.

When one edge is split:

```text
+1 new node
+2 new edges
-1 old edge
```

so the net graph change is:

```text
+1 node
+1 edge
```

```text
run
insert_road_closure.py
```

Main outputs:

```text
road_closure_graph_nodes.csv
road_graph_with_exits_and_closures.pkl
```
<img width="552" height="566" alt="image" src="https://github.com/user-attachments/assets/3a0567cd-8ad2-4e7d-9f4c-c4890eae6db9" />

<img width="988" height="170" alt="image" src="https://github.com/user-attachments/assets/e73521d4-d870-4179-abb5-d0e886c87fe6" />


---

## Step 7 — Classify road closures as POINT or SECTION

Each current road closure is classified using the start/end relationship.

- **POINT** — start and end overlap or represent the same graph location.
- **SECTION** — start and end are separated and represent a road section.

For a POINT closure:

```text
START = END = X
```

For a SECTION closure:

```text
START S ======================== E END
```

```text
run
classify_road_closure.py
```

Main output:

```text
road_closure_plan.csv
```
<img width="869" height="638" alt="image" src="https://github.com/user-attachments/assets/8311196d-6765-4364-9e04-e4b004f5cbf1" />

<img width="730" height="1094" alt="image" src="https://github.com/user-attachments/assets/a4dafb4f-a6a8-4f4a-a256-145c901923a8" />



---

## Step 8 — Identify the likely actual road path for SECTION closures

For a SECTION closure, inserting the start and end nodes does not automatically create one single "middle edge".

The road between `S` and `E` is normally a sequence of many graph edges:

```text
S ─ 1 ─ 2 ─ 3 ─ 4 ─ 5 ─ E
        |           |
     branches     branches
```

Before the closure can be applied, the analysis must identify which sequence of OSM/NetworkX edges most plausibly represents the reported closed section.

NetworkX is used to find a candidate path between the inserted start and end nodes, while preferring road names that match the NT Road Report road name.

### Automatic validation checks

The candidate path is evaluated using three checks:

1. **Preferred-road percentage**

```text
preferred_road_pct >= 80%
```

This is the percentage of the path whose OSM road name is the same as, or sufficiently similar to, the NT Road Report road name.

2. **Other-road percentage**

```text
other_road_pct <= 10%
```

This measures how much of the path clearly belongs to differently named roads.

3. **Path ratio**

```text
path_ratio = network_path_length / straight_endpoint_distance
path_ratio <= 2.0
```

The straight endpoint distance is the direct line between the reported start and end coordinates. It is **not** treated as the real road length. It is only a sanity-check baseline.

For example:

```text
straight distance = 100 km
network path      = 115 km
```

is plausible, while:

```text
straight distance = 100 km
network path      = 450 km
```

suggests that NetworkX may have taken an unrealistic detour.

A candidate path is marked:

- **LOOKS GOOD** — all automatic thresholds pass.
- **REVIEW** — one or more thresholds fail.

`LOOKS GOOD` means the path passes the automatic checks; it does not guarantee that the route is correct.

Very long closures can still be manually inspected even when they pass the rules.

```text
run
find_section_closure_path.py
```

Main outputs:

```text
section_closure_candidate_paths.gpkg
section_closure_path_summary.csv
```

<img width="1003" height="223" alt="image" src="https://github.com/user-attachments/assets/8a582dec-4034-44da-a3f7-1e809b6da0c5" />

---

## Step 9 — Manually validate SECTION paths in QGIS before snaping on the even graph

SECTION paths marked `REVIEW` are manually inspected in QGIS before they are allowed to modify the event graph.

For easier checking, `section_closure_path_summary.csv` can be joined to `section_closure_candidate_paths.gpkg` using `obstruction_id`, then filtered by `review_status`.

The manual review checks whether the candidate path:

- starts and ends at the expected closure locations,
- follows the expected road corridor,
- avoids unrealistic detours,
- is consistent with the reported road name,
- and remains geographically plausible.

A long path is not automatically wrong. For example, a long road can still receive `LOOKS GOOD` when it follows the expected road consistently and its network-path length remains reasonable compared with the direct endpoint distance.

<img width="797" height="1025" alt="image" src="https://github.com/user-attachments/assets/fa4c3393-5349-4b72-9c25-3c9ce0c5e8f1" />
(explanation generated by ChatGPT)


<img width="940" height="388" alt="image" src="https://github.com/user-attachments/assets/86686184-ca33-4293-944f-787b9710b61b" />
(Sandover Highway)
(Looks good, the NetworkX ignores all the branches and pathways, focusing on the main road. Even though in reality, those branches might on the closing stage as well, but we no need to care about it since we will remove the whole edges (red line))

<img width="609" height="689" alt="image" src="https://github.com/user-attachments/assets/f0e4f2f9-7b3e-4ac3-b328-c9c6b7831d16" />
(path-ratio > 2)


### Step 9B — Known problem cases

This version of the workflow recognises two main problem types if manual validation fails.

#### Case 1 — Endpoint matching issue

The NT Road Report start/end coordinates may not lie directly on the OSM road. The nearest-road matching stage can therefore select the wrong nearby road, especially:

- at intersections,
- where two roads are parallel,
- where roads are closely spaced,
- or where the source coordinate is approximate.

In this case, the inserted NetworkX node is technically correct for the matched OSM road, but the **matched OSM road itself is wrong**.

#### Case 2 — Path-selection issue

The start/end matches may be correct, but NetworkX may still choose an implausible alternative route between them.

This remains a manual-review case in the current workflow. The technical correction approach is still under development.

---

## Step 10 — Apply validated POINT closures

A POINT closure normally sits on a road edge that was split during closure-node insertion.

<img width="466" height="542" alt="image" src="https://github.com/user-attachments/assets/9f991fb5-d05a-4c22-8388-7caae8a498c0" />


Before blocking:

```text
A ─── X ─── B
      ↑
 point closure
```

The two road edges touching the closure node are removed:

```text
A     X     B
```

Removing both edges prevents travel through the closure point.

This does not mean two separate roads were closed. It means the original road edge was split into two graph edges around the inserted closure node.

```text
run
block_point_closures.py
```

Main outputs:

```text
point_closure_blocked_edges.gpkg
point_closure_blocked_edges.csv
road_graph_event_points_only.pkl
```
<img width="625" height="355" alt="image" src="https://github.com/user-attachments/assets/12620ecf-fd48-4746-a595-b08101d7fac7" />


---

## Step 11 — Apply validated SECTION closures

For each accepted SECTION closure, all graph edges along the validated start-to-end path are removed from the event graph.

Conceptually:

```text
Normal graph:
A ─ S ─ 1 ─ 2 ─ 3 ─ E ─ B

Event graph:
A ─ S               E ─ B
      [closed path]
```

Only the validated section path is removed; the entire connected component is not removed.

```text
run
block_section_closures.py
```

Main outputs:

```text
section_closure_blocked_edges.gpkg
section_closure_blocked_edges.csv
road_graph_event_all_closures.pkl
```
<img width="681" height="436" alt="image" src="https://github.com/user-attachments/assets/a898285b-98bd-45d0-8e53-17875ec84beb" />

<img width="499" height="577" alt="image" src="https://github.com/user-attachments/assets/375965b1-ffe4-4f43-8e92-818f8d5c1349" />

<img width="381" height="713" alt="image" src="https://github.com/user-attachments/assets/c2b168e4-f973-4f18-b44d-d1113c1213bd" />
(The red lines are removed)


---

## Step 12 — Compare the normal and event graphs

The final stage compares the normal graph with the event graph after the accepted closure edges have been removed.

The key idea is not to look for missing coordinates. Most graph nodes remain in both graphs. The important change is that some **edges are removed**, which can split the network into new connected components.

Example:

```text
Normal:
A ─ B ─ C ─ D ─ E

After closure:
A ─ B ─ C     D ─ E
          X
```

The nodes still exist, but connectivity has changed.

Only the genuine `EXTERNAL_EXIT` nodes identified in Step 4 are used for the final community test.

For each road-accessible community:

- **ACCESS_REMAINS** — at least one valid external exit still connects to the wider modeled road network.
- **POTENTIAL_ROAD_ISOLATION** — external access existed in the normal graph, but every valid external exit is disconnected in the event graph.
- **BASELINE_REVIEW** — no valid external connection existed in the modeled road network before the current closure event, so the event is not treated as the cause.

- ```text
run
test_community_isolation_final.py
```

<img width="466" height="683" alt="image" src="https://github.com/user-attachments/assets/2236a5e6-0440-4cfe-a834-5bea934d2b76" />


---

# Current result

For the road-closure snapshot used in this analysis:

```text
Road-accessible communities assessed: 168
ACCESS_REMAINS:                       167
POTENTIAL_ROAD_ISOLATION:               0
BASELINE_REVIEW:                         1
```

The `BASELINE_REVIEW` community was **Dhipirrinjura**. Its modeled vehicle-road connection was already outside the large external road network before the current closures were applied, so it was not classified as newly potentially isolated by the event.

These results apply only to the road and closure data used in this run.

<img width="688" height="1059" alt="image" src="https://github.com/user-attachments/assets/970839bf-18f0-42c2-805f-12f0c4c50f41" />

<img width="556" height="567" alt="image" src="https://github.com/user-attachments/assets/0e2be5d0-8f3c-4b7c-a99d-4f47440a691e" />



---

# Repository contents

This repository is intended to share the analysis results and methodology rather than the development code.

```text
.
├── README.md
├── data/
| ├── nt_remote_communities_connectivity_updated.csv
| ├── community_road_baseline.csv
| ├── community_exit_candidates_raw.csv
| ├── community_exit_nodes.csv
| ├── road_closure_end_matches.csv
| ├── road_closure_start_matches.csv
| ├── road_closure_graph_nodes.csv
| ├── road_closure_plan.csv
| ├── point_closure_blocked_edges.csv
| ├── section_closure_path_summary.csv
| ├── section_closure_blocked_edges.csv
| ├── community_external_exits.csv
| ├── community_road_isolation_final.csv
| └── community_external_exit_summary.csv
└── QGIS project and selected analysis layers
```

The Python development scripts are intentionally not included.

---

# Limitations

- The 1 km community buffer is an **analysis zone**, not an official settlement boundary.
- OpenStreetMap coverage and topology may be incomplete in remote areas.
- Reported road-closure coordinates may be approximate and may match the wrong nearby OSM road.
- SECTION closure paths require validation because a technically connected route may still be geographically implausible.
- The current endpoint/path correction process still includes manual review.
- The model tests **vehicle-road connectivity only**. It does not assess air, sea, seasonal, informal or emergency-only access.
- A community can remain connected while still experiencing a large detour or reduced road redundancy; this version focuses on complete modeled road disconnection.
- `POTENTIAL_ROAD_ISOLATION` is a model-derived analytical result, not an authoritative declaration that a community is isolated.
- Current Road Report closures can have different causes, including flooding, road damage, roadworks or park-related closures. A later hazard-specific model should distinguish the cause before attributing isolation to a particular hazard.

---

# AI use and authorship

The **project idea, analysis objective, data selection, assumptions, workflow logic, validation decisions, manual QGIS checking, interpretation of results, debugging direction, and final methodological decisions were directed by the project author**.

ChatGPT was used as a **coding and technical-support assistant** during development. Python code was generated and iteratively refined by ChatGPT in response to the author's instructions, requested logic, observed outputs, identified problems and manual validation feedback.

The development process was therefore collaborative in the following sense:

```text
Author defines the problem and analysis logic
                ↓
ChatGPT translates the requested logic into code / technical steps
                ↓
Author runs the workflow and checks the outputs
                ↓
Author identifies issues or changes the methodology
                ↓
ChatGPT helps revise the implementation
                ↓
Author validates and interprets the final result
```

The author remained responsible for deciding **what the model should do**, checking whether the outputs were reasonable, and accepting or rejecting methodological changes.

The Python implementation is not included in this repository. The repository documents the **methodology, validation process, QGIS analysis, CSV outputs and final results**.

---

# Possible extension

The road-network disruption component can be reused for hazard analysis once a hazard can be translated into credible affected road points or sections.

For example:

```text
Flood / bushfire / cyclone information
                ↓
identify potentially affected road edges
                ↓
apply road-network disruption logic
                ↓
recalculate connectivity
                ↓
assess potential community road isolation
```

A hazard polygon alone should not automatically be treated as proof that every road inside it is closed. Authoritative closure information or an explicit hazard-to-road impact rule is still required.
