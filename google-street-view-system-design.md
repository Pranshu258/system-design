# Google Street View System Design

A Google Street View-like system turns geographically positioned imagery into an interactive experience: choose a point on a map, look around a panorama, move along connected capture locations, and optionally explore older imagery. This design covers capture ingestion, image processing, spatial discovery, panorama navigation, progressive rendering, and privacy-safe publication. General map search, road routing, and live traffic belong to the companion [Google Maps design](./google-maps-system-design.md).

This is an illustrative architecture, not a description of Google's private infrastructure. Public Street View documentation establishes product concepts such as panorama metadata, adjacent panorama links, tiled imagery, and privacy reporting; the storage layout, release protocol, workload estimates, and SLOs below are design proposals. Assume the operator owns or has licensed the imagery and processing rights. Building an application with Google's APIs does **not** grant permission to reproduce this ingestion, caching, analysis, or offline-storage architecture using Google imagery.

## 1. Requirements
### 1.1 Functional requirements
- Ingest calibrated multi-camera capture sequences with GPS, inertial measurements, timestamps, and optional depth, plus already-stitched contributor panoramas.
- Process captures into aligned, privacy-reviewed panoramas with multiple resolutions, thumbnails, and navigable connections.
- Resolve a geographic point to suitable imagery, expose coverage on a map, and support filters for capture date and imagery type.
- Render a full spherical view with heading, pitch, field of view, and progressive detail on web and mobile.
- Move between connected panoramas while maintaining viewing direction and avoiding implausible jumps between roads, floors, or capture dates.
- Offer historical imagery where available, with explicit capture dates and gaps rather than implying continuous coverage.
- Accept abuse, privacy-blur, and removal reports; suppress affected imagery and rebuild safe derivatives.

### 1.2 Non-functional requirements
- **Interactive latency.** Illustrative targets: metadata p95 below `150ms` in-region and first useful view p95 below `1s` on a representative `20Mbps`, `80ms` RTT connection. Measure cold-start and warm navigation separately.
- **Serving availability.** Target `99.99%` monthly availability for eligible published imagery. Privacy denial is a correct outcome, not a reason to serve unsafe stale content.
- **Bandwidth efficiency.** Fetch low-resolution context and visible high-resolution tiles, not an entire maximum-resolution panorama.
- **Publication integrity.** A published panorama must reference complete, checksummed, policy-approved artifacts and a compatible navigation graph.
- **Bounded revocation.** Stop new server delivery of withdrawn content within an explicit propagation bound, including old URLs and cache hits.
- **Traceability.** Preserve source rights, capture provenance, processing versions, privacy decisions, and deletion lineage under restricted access.

### 1.3 The binding constraint

The binding constraint is that imagery should be immutable and aggressively cached for interactive global delivery, but privacy decisions can invalidate any previously published derivative. A face missed in one frame may appear in a thumbnail, several tile resolutions, an older revision, and neighboring captures. Changing the discovery index or publishing a new URL does not stop a CDN from serving an old one.

The architecture therefore needs two different kinds of state: immutable, versioned imagery and graph artifacts for performance and consistency; and a small, authoritative eligibility overlay that can deny access independently of release rollback. The difficult part is not merely storing panoramas. It is maintaining a coherent, navigable, privacy-safe view of a large derived corpus while sessions and caches retain older references.

## 2. Back-of-the-envelope estimation

These assumptions are illustrative, not Google usage or storage figures. Use decimal units unless stated otherwise.

| Quantity | Assumption and calculation | Design implication |
|---|---|---|
| Viewing sessions | `20M` daily users, `2` sessions each = `40M/day` | Discovery and session bootstrap are much smaller than tile traffic. |
| Panorama visits | `15` per session = `600M/day`, about `6.9k/s` average and `35k/s` at `5x` peak | Metadata needs regional replication and caching. |
| Image requests | `30` tile requests per visit = `18B/day`, about `208k/s` average and `1.04M/s` peak | Most bytes must terminate at the edge. |
| Image bandwidth | `40KB` average compressed tile = `720TB/day`, about `67Gbps` average and `333Gbps` peak | Tile size, visible-area selection, and prefetch waste dominate cost. |
| Tile origin load | `98%` request hit ratio gives about `21k/s` peak origin requests | Protect cold regions and releases with coalescing and admission control. Byte hit ratio must be measured separately. |
| Published imagery | `5B` panorama revisions at `8MB` per compressed highest-resolution image = `40PB` | Object storage, lifecycle policies, and regional placement matter. |
| Resolution pyramid | Idealized downsampled levels add `1/4 + 1/16 + ...`, giving about `4/3` of highest-resolution bytes | Approximately `53PB` before tile-container overhead, thumbnails, replication, and encoding differences. |
| Panorama metadata | `2KB` per revision = `10TB` before indexes and replication | Keep image bytes outside the metadata database. |
| New captures | `10M/day` at `50MB` raw = `500TB/day`, or `15PB` for `30` days | Raw retention and failed-job cleanup are first-order costs. |

The `8MB` and `50MB` assumptions describe different encodings and stages, not a promise of a fixed compression ratio. Historical revisions are included in the `5B` count; retaining more history grows storage accordingly. Rights, privacy, and regional rules can shorten raw retention below the illustrative `30` days.

Processing has its own capacity model. If the measured pipeline consumes `2` accelerator-seconds per new panorama, `10M/day` requires about `232` continuously busy accelerator-equivalents. At `60%` effective utilization, that is roughly `386`, before retries and reprocessing. This is a planning placeholder: benchmark representative captures, resolutions, model versions, and stitching modes before purchasing capacity. A privacy-model upgrade over billions of panoramas can cost much more than daily ingestion, so prioritize likely violations and high-exposure imagery rather than blindly rebuilding everything at once.

> **Key idea.** Image delivery is a bandwidth-and-cache problem, metadata is a spatial-and-graph problem, and ingestion is a throughput-and-policy problem. Scaling them as one service wastes resources and couples unrelated failure modes.

## 3. API design

**Question.** Why is `GET /panorama?lat=...&lng=...` returning one giant JPEG not enough?
<details><summary>Answer</summary>

It conflates discovery with delivery, downloads pixels outside the viewport, lacks graph and capture-date context, and cannot identify which processing revision was approved. Resolve a point to a release-pinned panorama revision, return a manifest with orientation and adjacent links, and let the renderer request only the needed image levels. Every delivery still checks current eligibility.

</details>

The following are proposed first-party APIs, not Google's endpoint contracts:

```http
GET /v1/coverage/{release_id}/{z}/{x}/{y}.mvt

GET /v1/panoramas:resolve?lat=37.42&lng=-122.08&radius_m=50&imagery_type=outdoor&release=current

GET /v1/panoramas/{panorama_id}/manifest?revision=pr_17&release=gr_42

GET /v1/panoramas/{panorama_id}/history?cursor=opaque&limit=20

GET /v1/imagery/{panorama_id}/{revision}/{encoding}/{level}/{x}/{y}
Authorization: Bearer short-lived-view-capability

POST /v1/capture-uploads
Idempotency-Key: capture-upload-uuid
{
  "capture_run_id": "run_123",
  "expected_parts": 24,
  "declared_bytes": 50000000,
  "rights_record_id": "rights_7",
  "calibration_version": "rig_12"
}

POST /v1/capture-uploads/{upload_id}:complete
{
  "parts_manifest": "restricted-object-reference",
  "manifest_checksum": "sha256:..."
}

POST /v1/privacy-reports
Idempotency-Key: report-uuid
{
  "panorama_id": "p_123",
  "revision": "pr_17",
  "reason": "MISSED_FACE_BLUR",
  "view": {"heading_deg": 120, "pitch_deg": 4, "horizontal_fov_deg": 50}
}
```

Discovery returns `panorama_id`, `revision`, `graph_release_id`, distance, capture-date precision, imagery type, and selection reason. Its `current` alias resolves once to a concrete release; following links keeps that release until an explicit migration. `history` is a ranked set of nearby comparable captures, not necessarily repeated photographs from one exact coordinate. Pin its cursor to a history-index snapshot and filter current policy on every page.

A manifest contains image dimensions, tile size, projection, available levels and codecs, camera orientation, attribution, a report link, and graph links with target revisions and bearings. It also identifies the coordinate convention: headings clockwise from true north, pitch positive upward, and a defined camera-to-world rotation. Do not treat arbitrary sensor yaw as compass heading.

Image requests are safe reads. Authorization is checked **before** a shared cache can return bytes; bearer tokens are excluded from the object cache key only after validating their scope. Cache keys include panorama, revision, projection/encoding version, level, and tile coordinates. Credentials never belong in logs or durable image URLs.

Validation rejects invalid coordinates, excessive search radii, unsupported levels, and oversized uploads with explicit errors. No coverage returns a typed `NO_COVERAGE` result rather than a misleading nearby indoor scene. An explicitly withdrawn asset returns `410 Gone` or a policy-equivalent response without sensitive report details; a service outage returns `503`, not "no coverage." Repeated upload completion with the same checksum returns the same processing job; conflicting completion returns `409`.

Google's public Map Tiles API documents panorama IDs as unsuitable for reliance beyond a user session. This design's durable internal IDs are an architectural choice, not a claim about those public IDs. An adapter using Google's API must rediscover its IDs and obey its session, attribution, and caching rules.

## 4. Data model

Start with `Panorama(id, lat, lng, image_url)`. This can show a picture, but it cannot explain whether the coordinates are accurate, which way the camera faces, or whether the image is safe to publish.

Add capture provenance, orientation, accuracy, rights, and processing revision. Now historical dates and reblurring are representable, but a spatial nearest-neighbor query still cannot determine where a viewer may move. Add typed graph edges with confidence, capture-run continuity, and road/floor context.

Finally, immutable revisions do not represent current removals. Add a separate monotonic policy state and derivative lineage. Ordinary rollback may restore an older graph, but must never restore an obsolete privacy approval.

```text
CaptureRun
- capture_run_id, contributor_or_fleet_id, region, rights_record_id
- calibration_version, sensor_manifest_ref, capture_time_range
- upload_state, retention_deadline

Capture
- capture_id, capture_run_id, sequence_number, restricted_raw_ref
- captured_at, gps_pose, pose_covariance, imu_ref, optional_depth_ref
- checksum, source_rights, processing_state

Panorama
- panorama_id, capture_id, corrected_pose, pose_confidence
- spatial_cell_id, elevation_m, floor_or_level, imagery_type
- capture_date, capture_date_precision

PanoramaRevision
- panorama_id, revision, projection, width, height, orientation
- artifact_manifest_ref, artifact_checksums, lineage_id
- stitch_version, privacy_model_version, review_decision_id
- policy_generation, state

NavigationEdge
- graph_release_id, source_panorama_revision, target_panorama_revision
- bearing_deg, distance_m, edge_type, confidence
- capture_run_continuity, road_or_floor_context

RegionalRelease
- graph_release_id, region, spatial_index_ref, coverage_tiles_ref
- graph_shards_ref, panorama_manifest_set_ref, boundary_compatibility_ref
- checksums, source_cutoff, state

PolicyState
- scope_type, scope_id, generation, status, effective_at
- reason_class, restricted_case_ref

DerivativeLineage
- lineage_id, capture_id, panorama_id, revision
- artifact_refs, thumbnail_refs, export_refs, processing_attempt_id
```

Capture manifests and raw files live in restricted object storage with resumable uploads. Approved image artifacts live in a separate private origin behind the serving gate. Metadata and job state use transactional records; spatial indexes, coverage tiles, and graph shards are derived release artifacts. A separate strongly consistent policy store issues revocation generations and regional replicas feed edge enforcement.

The spatial index uses hierarchical cells such as S2 to retrieve candidates, followed by exact spherical distance and semantic filtering. A radius query covers all intersecting cells, including neighbors across cell boundaries, the antimeridian, and polar regions. A cell ID is not a distance ordering or proof that the same road is reachable. Dense urban cells are subdivided or split internally so one landmark does not become a hot metadata partition.

Keep graph versions separate from image revisions so a graph fix need not rewrite image bytes. A release can reuse existing approved artifacts, but it must reference exact image revisions. Cross-region traversal uses declared compatible boundary shards; if those are missing, stop or explicitly re-resolve in the next region instead of silently mixing unrelated releases.

> **Key idea.** Identity, capture time, image processing revision, navigation release, and current serving eligibility are different dimensions. Collapsing them into one mutable row makes either browsing or removal unreliable.

## 5. High-level design

The naive system accepts an image, inserts its coordinates into a spatial database, and serves that image from the same application that handles navigation.

```mermaid
flowchart LR
    Capture["Capture upload"] --> API["Single application"]
    API --> DB[("Coordinates and image rows")]
    Viewer["Viewer"] --> API
```

This fails under large uploads, slow stitching, global bandwidth, ambiguous proximity, and privacy removals. Apply one fix per failure rather than starting with an unexplained fleet of services.

### Fix 1: Decouple upload from processing

Create resumable upload sessions with checksums, declared sizes, source rights, and bounded lifetimes. Upload directly into quarantine storage. The control plane marks an upload complete only after verifying its complete parts manifest, then transactionally records a job/outbox event. A reconciler repairs missed dispatch; abandoned multipart uploads expire.

Workers process immutable inputs through validation, pose refinement, stitching, privacy filtering, quality checks, and tiling. At-least-once delivery is acceptable because stage outputs are keyed by input checksum, configuration/model version, and attempt identity. A compare-and-set publication step selects one successful output; a worker crash cannot expose partial results.

### Fix 2: Separate immutable pixels from spatial and graph metadata

Build a resolution pyramid once and put approved tiles behind regional origins and CDNs. Metadata services answer small discovery and graph requests; they do not proxy every pixel through the spatial database. A renderer fetches low-resolution context before visible high-resolution tiles.

### Fix 3: Publish compatible regional snapshots

Build and validate spatial indexes, coverage overlays, and navigation edges together. Check artifact completeness, graph targets, orientation, rights, and privacy approval before advancing the regional release pointer. Retain old releases for bounded session lifetimes and rollback, subject to **current** revocations.

### Fix 4: Put policy enforcement on every serving path

Process privacy decisions in a separate control plane. Publish a deny rule before relying on deletion or cache purge. Enforce it on metadata, tiles, thumbnails, historical revisions, coverage discovery, and exports. Raw storage is never publicly addressable.

```mermaid
flowchart TB
    Capture["Fleet and contributor uploads"] --> Upload["Upload control plane"]
    Upload --> Raw[("Restricted quarantine storage")]
    Upload --> Jobs["Durable job queue"]
    Jobs --> Process["Pose, stitch, blur, quality, tile"]
    Raw --> Process
    Process --> Approved[("Private approved artifacts")]
    Process --> Builder["Spatial and graph release builder"]
    Builder --> Releases[("Versioned manifests and indexes")]
    Client["Viewer"] --> Metadata["Discovery and metadata service"]
    Metadata --> Releases
    Client --> Edge["Policy gate before CDN cache"]
    Edge --> Approved
    Reports["Privacy reports and review"] --> Policy[("Policy authority and deny generations")]
    Policy --> Edge
    Policy --> Metadata
    Policy --> Process
```

The edge box denotes logical ordering: authorization and revocation run on hits as well as misses. If the chosen CDN cannot enforce that ordering and freshness contract, use a programmable gateway in front of it or choose a different delivery arrangement.

## 6. Deep dives
### 6.1 Turning noisy captures into publishable panoramas

**Question.** A capture vehicle uploads a run with GPS drift, one miscalibrated camera, and a privacy-model timeout. Which outputs may be published?
<details><summary>Answer</summary>

Only outputs that independently pass required geometry, rights, quality, and privacy checks. Retries may recover a transient timeout; an unresolved privacy stage blocks publication. GPS correction cannot compensate for an unknown camera calibration, and partial success must not turn into a success-shaped public release.

</details>

The pipeline begins by validating file formats, sensor timestamps, calibration versions, sequence gaps, and rights. Run untrusted image decoders in resource-limited workers; decompression bombs and malformed contributor uploads must not exhaust the processing pool. Already-stitched contributions skip multi-camera stitching but not orientation, quality, rights, or privacy checks.

Pose refinement combines GPS and inertial estimates with visual correspondences and, where available, depth or surveyed control points. Optimize a local capture graph or bundle-adjustment problem rather than snapping every frame independently to the closest road. The road map is evidence, not ground truth: a new road or pedestrian path may legitimately be absent from it. Retain pose confidence and quarantine implausible jumps.

Stitching projects calibrated views into a common sphere, compensates for exposure differences, and selects seams. Moving vehicles and pedestrians create ghosting; parallax near the rig prevents perfect stitching from a single center. Detect severe artifacts and choose a better source frame, mask a region, or reject the panorama. Depth improves alignment but does not justify inventing missing pixels or implying a photorealistic transition where geometry is unreliable.

Privacy filtering uses both source views and the stitched output. Source detections can be projected into panorama coordinates, while a post-stitch pass catches seam-crossing or distorted faces and plates. Expand masks conservatively around uncertainty and across the wraparound seam. Apply filtering before generating every public resolution and thumbnail so lower levels cannot retain an unfiltered copy. Sample and adversarial evaluation should include distant faces, reflections, occlusion, low light, and unusual plate formats.

```mermaid
stateDiagram-v2
    [*] --> Uploaded
    Uploaded --> Validated: "checksums, rights, calibration"
    Validated --> GeometryReady: "pose and stitch pass"
    GeometryReady --> PrivacyApproved: "required filters and review pass"
    PrivacyApproved --> ArtifactsReady: "all derivatives checksummed"
    ArtifactsReady --> Published: "release and policy generation validated"
    Published --> Withdrawn: "privacy or rights decision"
    Withdrawn --> Reprocessing: "authorized repair"
    Reprocessing --> GeometryReady: "new processing attempt"
```

The diagram shows the success path; any stage can enter retry, quarantine, or terminal rejection with an explicit reason. Privacy approval is not a permanent property of an image. Publication uses a policy-generation fence: serialize activation with the authority's current eligibility check, or require its conditional approval token. A concurrent withdrawal must prevent the old attempt from becoming eligible. Even an artifact completed after withdrawal remains inaccessible.

Use per-stage queues, bounded retries, dead-letter handling, and weighted scheduling. Isolate urgent privacy reprocessing from routine historical backfills. Monitor stage latency, retry amplification, quarantine counts, calibration failures, seam quality, and privacy false negatives estimated on independently labeled samples. Raw captures and review access need strict retention and audit controls; blurring does not make the original location traces harmless.

> **Strong-answer criteria.** A senior answer treats processing as a resumable, versioned dependency graph with fail-closed privacy checks, not a single image-conversion job.

| Level | Approach | Why |
|---|---|---|
| Bad | Stitch and publish immediately after upload. | Exposes malformed, mislocated, or unfiltered imagery. |
| Good | Add asynchronous stitching, blur detection, and retries. | Handles throughput, but retries and policy races can still publish stale approvals. |
| Great | Use immutable stage outputs, measured quality gates, complete lineage, and policy-fenced publication. | Makes failures recoverable without treating partial or obsolete processing as safe. |

### 6.2 Geographic discovery is not the navigation graph

**Question.** Two panoramas are five meters apart in latitude and longitude. Should the UI always offer a move between them?
<details><summary>Answer</summary>

No. They may lie on a bridge and the road below it, across a barrier, on different building floors, or in incompatible capture runs. Proximity retrieves candidates; pose, sequence continuity, road/floor context, visual overlap, and confidence determine navigable edges.

</details>

Discovery first covers a bounded search radius with spatial cells, retrieves eligible candidate revisions from a pinned release, and ranks them using distance, image quality, pose confidence, requested imagery type, heading, and capture date. Do not maximize recency blindly: a slightly older panorama on the requested road may be better than a newer one across a divided highway.

Navigation edges are built offline from consecutive captures and verified cross-run matches. Sequence adjacency is strong evidence but not sufficient: dropped frames may create a large gap or the vehicle may have been transported. Use distance, orientation, visual overlap, and compatible context to validate each edge. Public outdoor travel, indoor floor changes, and explicit historical transitions are different edge types.

A panorama graph is not a route planner. Its edges describe supported viewing transitions, not legal driving maneuvers. A capture sequence might traverse an area no longer open to cars. Keep road routing in the map system and avoid converting visual adjacency into road access permission.

At a junction, expose several directional links rather than automatically jumping to the spatially closest point. The response pins target revisions to the same graph release. Store each camera's world orientation so the client can maintain world-facing heading after a transition; copying texture yaw between differently oriented panoramas makes the user spin unexpectedly.

History is another selection problem. Cluster nearby captures using position, road/floor identity, orientation, and overlap. Label the actual date precision; do not invent a day when only month-level metadata exists. Switching dates is an explicit selection that may move the camera slightly. By default, navigation prefers a coherent capture run or date range and clearly labels any fallback to a different period. A historic panorama is not evidence that the real-world scene still looks that way.

Graph releases have reference-integrity checks, bounded-edge-length checks, component analysis, and sampled visual traversals. Compare coverage and connectivity by region to catch a builder that accidentally disconnects an entire city. Revoked nodes are filtered at request time even when the release remains pinned. If a target disappears, return available safe links or ask the client to re-resolve; do not silently bridge across an unvalidated gap.

Monitor resolution distance, no-coverage rate, dead links, graph component changes, unexpected floor/date changes, and transition abandonment. Spatially close test fixtures should include overpasses, tunnels, parallel roads, indoor/outdoor boundaries, and points on cell edges.

> **Strong-answer criteria.** A senior answer separates spatial candidate retrieval from semantically validated graph traversal and makes date, floor, orientation, and release compatibility explicit.

| Level | Approach | Why |
|---|---|---|
| Bad | Move to the nearest coordinate in the requested direction. | Jumps across barriers, floors, and capture dates. |
| Good | Connect consecutive captures and use a spatial index. | Works along clean runs but misses bad poses and cross-run ambiguity. |
| Great | Build versioned, confidence-scored typed edges and enforce current eligibility during traversal. | Supports coherent browsing without confusing proximity with navigability. |

### 6.3 Rendering a panorama without downloading all of it

**Question.** A panorama is `16,384 x 8,192` pixels. How can a phone show it quickly without decoding the whole image?
<details><summary>Answer</summary>

Deliver a small complete preview, then a multiresolution tile pyramid whose manifest describes projection and dimensions. Render the sphere locally, select levels from viewport pixel density, and fetch only intersecting tiles plus a small margin. Cancel obsolete work as the viewer turns.

</details>

For this proposed representation, use an equirectangular panorama with width twice its height. Longitude maps horizontally, latitude vertically, and the left/right boundary wraps. These are panorama image tiles, not Web Mercator map tiles; the coverage-map overlay is a separate vector-tile product.

At `512 x 512` pixels per tile, the highest level contains `32 x 16 = 512` tiles. A `90`-degree horizontal view covers roughly a quarter of the horizontal extent, but actual requests depend on pitch, vertical field of view, selected level, and distortion. The `30` requests per visit in the estimate is a workload assumption, not a consequence of the full-resolution grid.

Select level of detail from projected texel density and device resolution, with hysteresis to prevent repeated level switching. Near the poles, equirectangular sampling wastes pixels; a cubemap representation is a reasonable alternative if measurements justify it. Its six faces and edge filtering require a different manifest and cache namespace. Do not mix projection conventions or assume Google's published zoom dimensions match this proposed power-of-two pyramid.

Fetch a low-resolution whole-sphere preview for immediate orientation, then prioritize the viewport center, visible detail, and a narrow surrounding margin. A bounded GPU texture cache holds visible tiles and nearby levels. Border padding or projection-aware sampling prevents seams across tile boundaries and the panorama wrap. Lower-resolution tiles remain visible until replacements are decoded and uploaded.

Prefetching can improve movement but easily doubles traffic. Fetch only the likely next panorama's preview and a small set of forward-facing tiles when motion and bandwidth support it. Disable speculative fetches on metered connections or under memory pressure. Track useful-prefetch ratio, cap concurrent requests and decoded bytes, and cancel obsolete requests where possible; cancellation cannot recover bytes already transferred.

```mermaid
sequenceDiagram
    participant C as Viewer
    participant M as Metadata
    participant E as Edge policy gate and cache
    C->>M: Resolve point and pin release
    M-->>C: Panorama revision and manifest
    C->>E: Preview request with view capability
    E->>E: Check eligibility before cached delivery
    E-->>C: Small blurred preview
    C->>E: Visible tile requests at selected level
    E-->>C: Approved cached tiles
    C->>C: Decode, render, bound texture memory
    C->>M: Follow release-pinned adjacent link
    M-->>C: Next eligible panorama manifest
```

Popular tourist sites create hot objects, while rural exploration produces a long tail. Coalesce concurrent origin misses, replicate hot approved artifacts regionally, and prewarm only measured demand. Do not render or re-encode arbitrary tiles synchronously on every miss. Codec negotiation chooses from prebuilt supported encodings and is represented explicitly in the cache key.

Measure first useful view, viewport-complete latency, interaction frame time, decoded memory, bytes per visit, cache byte hit ratio, origin errors, and prefetch waste. Test a deterministic trace of rapid turns, zooms, and transitions on slow networks and low-memory devices. A high CDN hit ratio alone does not prove a smooth viewer.

> **Strong-answer criteria.** A senior answer ties tile projection and level selection to device pixels, bandwidth, texture memory, cancellation, and measurable visual latency.

| Level | Approach | Why |
|---|---|---|
| Bad | Download and decode the maximum-resolution JPEG for every step. | Wastes bandwidth and memory before the first useful frame. |
| Good | Use a CDN and a multiresolution pyramid. | Reduces origin load, but indiscriminate fetching still wastes bytes. |
| Great | Use preview-first rendering, viewport-aware tiles, bounded caches, and measured speculative fetches. | Improves user-visible latency while controlling bandwidth and device cost. |

### 6.4 Privacy withdrawal when immutable URLs already exist

**Question.** A missed face is reported after a panorama has been cached worldwide. Is publishing a blurred replacement and deleting the original object sufficient?
<details><summary>Answer</summary>

No. Old graph releases, thumbnails, historical views, processing retries, CDN hits, and exports can retain references. First revoke the affected serving scopes through a gate that runs before cache delivery, then purge and remove derivatives, repair the imagery, and publish a separately approved revision. No server-side protocol can retract bytes a user already downloaded.

</details>

Separate a submitted report from an adjudicated removal. Reports need rate limits, evidence, triage, and abuse protection so an attacker cannot automatically erase a neighborhood. High-confidence urgent cases can trigger precautionary suppression under policy. Store reporter identity and detailed evidence in a restricted case system, not public manifests or general telemetry.

When a withdrawal is accepted, increment the affected policy generation and deny the panorama scope and known derivative scopes. The authority provides regional replication and a bounded-freshness lease to serving gates. As an illustrative contract, require a new server-delivery cutoff within `60s` of the authoritative decision: push deny updates immediately, grant leases lasting at most `30s`, budget for clock skew and in-flight transfer termination, and test the remaining margin. Gates with expired leases fail closed rather than serving cached pixels.

That contract must define whether it covers response start or continued streaming. For a strict new-delivery bound, cap image response duration and terminate deliveries that cross the allowed lease window. Small tiles make that tractable. It is not a guarantee that previously delivered pixels disappear from a screen.

Signed capabilities limit session and asset access but cannot substitute for revocation: a still-valid token for a denied panorama must fail. Origin buckets deny direct public access; alternate thumbnail paths, resizing proxies, debugging endpoints, and regional failover must enforce the same gate. Internal caches may retain immutable bytes, but they cannot bypass eligibility checks.

Then enumerate lineage: every tile level, preview, thumbnail, old image revision, supported export, and downstream processor. Purge CDN entries and delete or quarantine objects according to policy and legal retention requirements. Backups need access isolation and a deletion/suppression journal applied before restored data becomes servable. Never assume a database tombstone physically deletes all backups immediately.

Neighboring captures can contain the same subject. Expand the investigation across relevant capture time and spatial overlap, suppress uncertain candidates when warranted, and reprocess with projected masks plus fresh detection. Track completeness of the derivative inventory; incomplete lineage is a release blocker for systems claiming bounded removal.

Repaired content receives a new revision and policy approval. If the old release still references a withdrawn revision, it stays denied; clients must resolve the repaired revision through a compatible manifest rather than transparently mixing it into old image metadata. Graph rollback cannot roll back the policy authority. Conservative panorama-wide denial may briefly hide safe historical revisions; finer scopes are allowed only when the evidence and lineage support them.

For first-party clients, keep local image caches session-bounded, require online eligibility before displaying a newly opened panorama, and stop displaying withdrawn content after notification or lease expiry. Offline imagery cannot satisfy immediate connected revocation. The default design therefore excludes persistent offline viewing; any licensed offline variant must declare its weaker removal bound. Screenshots and third-party copies are outside technical retraction guarantees.

Monitor decision-to-deny propagation by region, oldest valid gate lease, denied old-URL probes, purge/deletion completion, and accesses after withdrawal. Run synthetic removals against warm CDN caches, old releases, valid tokens, disconnected policy replicas, and restored backups. An outage in policy replication must be visible as a serving blocker, not silently interpreted as approval.

> **Strong-answer criteria.** A senior answer distinguishes blocking new access from deleting stored bytes and retracting downloaded content, with a tested bound and no stale-cache bypass.

| Level | Approach | Why |
|---|---|---|
| Bad | Replace the image URL in the database. | Old URLs and caches continue to serve the violation. |
| Good | Reblur, delete origin objects, and purge the CDN. | Necessary cleanup, but purge races and alternate derivatives remain. |
| Great | Deny before cache hits, fence re-publication, enumerate lineage, and verify bounded propagation. | Prevents old releases and retries from restoring access while cleanup proceeds. |

## 7. Variants

**10x viewing traffic.** Tile peak rises to about `10.4M/s` and the assumed peak image bandwidth to about `3.33Tbps`. CDN byte efficiency, hot-object replication, origin shielding, and useful-prefetch ratio matter more than increasing metadata workers. Policy enforcement still runs on hits, so distribute compact deny state and leases rather than synchronously querying one central database per tile.

**Contributor and indoor imagery.** Contributions have less consistent calibration, rights, pose quality, and capture continuity. Add contributor reputation and moderation, but never substitute reputation for required privacy checks. Indoor navigation needs floor identity and explicit stairs/elevator transitions; public outdoor discovery should not accidentally select a nearby restricted interior.

**Historical comparison.** A split-screen viewer pins two independent approved revisions and synchronizes world-facing orientation. Position and lighting changes can make comparisons misleading, so expose capture dates and displacement. Budget for roughly two visible tile working sets and do not claim exact change detection from unmatched imagery.

**Privacy-model upgrade.** Shadow-evaluate a new model on representative labeled data, prioritize high-risk published imagery, and reserve isolated reprocessing capacity. Deny confirmed unsafe content immediately instead of waiting for a global backfill. Model version alone is not an approval decision, and rollback never re-enables known violations.

**Regional residency and licensed offline use.** Keep raw captures and sensitive pose traces in their required jurisdictions; replicate only approved derivatives where rights allow. Offline packages are an opt-in contract with signed manifests, expiry, and explicit revocation limitations, not an automatic consequence of having tile files.

## 8. The transferable pattern

This is the pattern of an immutable derived corpus with mutable eligibility. Precompute expensive transformations, address outputs by exact version, and serve them near users. Keep the small state that determines whether those bytes may still be served outside ordinary release rollback.

The same tension appears in [YouTube](./youtube-system-design.md) for transcoded media and takedowns, [Dropbox-like storage](./dropbox-system-design.md) for shared-file access, and [Google Maps](./google-maps-system-design.md) for stable map releases plus urgent overlays. The reusable lesson is that cacheability and authorization are separate concerns: immutable bytes do not imply permanently valid access.

## Review

Street View is not simply a geospatial database of JPEGs. It is an asynchronous capture-processing pipeline, a spatial discovery system, a versioned viewing graph, a progressive image renderer, and a privacy control plane. The binding constraint forces immutable artifacts for scale alongside mutable, fail-closed eligibility for removals. A strong design explains how approved pixels become discoverable, how a viewer follows coherent links, how only useful detail crosses the network, and how every old delivery path becomes inaccessible after withdrawal.

### Check your understanding
1. Why should a panorama revision and its navigation graph have separate versions?
   <details><summary>Answer</summary>

   Image processing and graph connectivity evolve independently. A graph repair can reuse approved pixels, while reblurring creates new image artifacts. Exact revision references preserve compatibility, and current policy overrides both.

   </details>
2. Why is the nearest panorama not always the best result?
   <details><summary>Answer</summary>

   Geographic distance ignores barriers, road level, indoor/outdoor context, pose confidence, date, and viewing direction. Spatial retrieval supplies candidates; semantic ranking and graph validation determine suitable views and transitions.

   </details>
3. What makes an image-processing retry safe?
   <details><summary>Answer</summary>

   Immutable input and stage identities, checksummed outputs, idempotent completion, explicit failure states, and a policy-fenced publication step. At-least-once job execution does not itself provide exactly-once public visibility.

   </details>
4. Why can a CDN cache hit still require policy work?
   <details><summary>Answer</summary>

   Cached bytes may have become ineligible since they were stored. Authorization and a fresh-enough deny state must be checked before returning them, including requests made with old valid capabilities.

   </details>
5. What does a sixty-second withdrawal objective actually guarantee?
   <details><summary>Answer</summary>

   Under the stated lease, clock, and transfer assumptions, controlled servers stop delivering the withdrawn content within that bound. It does not erase screenshots, force deletion of third-party copies, or retract pixels already downloaded. Those limits must be explicit.

   </details>
6. Which measurements establish that tile prefetching is useful?
   <details><summary>Answer</summary>

   Compare transition latency and first useful view against additional bytes, useful-prefetch ratio, abandoned requests, decoded memory, and metered-network behavior using representative navigation traces. Cache hit ratio alone is not sufficient.

   </details>

## Sources and further reading
| Source | Article | Why it matters |
|---|---|---|
| Google Maps Platform | [Street View Tiles](https://developers.google.com/maps/documentation/tile/streetview) | Documents tiled panoramas, image dimensions, orientation, attribution, adjacent links, and the limited lifetime assumptions for public panorama IDs. |
| Google Maps Platform | [Street View Publish API](https://developers.google.com/streetview/publish) | Establishes contributor upload, position, orientation, and connectivity as distinct concerns. It does not specify the proposed internal processing pipeline. |
| Google Street View | [Street View privacy policy](https://www.google.com/streetview/policy/) | Describes automatic face/license-plate blurring and requests for additional blurring across published copies; motivates derivative-aware privacy handling. |
| Google Maps Platform | [Map Tiles API policies](https://developers.google.com/maps/documentation/tile/policies) | Explains attribution, permitted uses, and caching/storage restrictions. The first-party architecture here is not a license to store or analyze Google API imagery. |
| S2 Geometry | [S2 Cell Hierarchy](https://s2geometry.io/devguide/s2cell_hierarchy.html) | Provides spherical hierarchical cells for spatial candidate retrieval; does not establish navigation connectivity. |
| IETF | [RFC 9111: HTTP Caching](https://www.rfc-editor.org/rfc/rfc9111.html) | Grounds cache keys, freshness, revalidation, and storage directives. Application-level revocation still requires the explicit serving contract described above. |
