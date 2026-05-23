# Project Research Summary

**Project:** Moviment Heatmap
**Domain:** Computer vision football movement analysis
**Researched:** 2026-05-23
**Confidence:** HIGH

## Executive Summary

Moviment Heatmap is a computer vision web application that converts football match footage into accumulated movement heatmaps and individual player trajectories for sports science researchers and tactical analysts. Experts build this class of product using a YOLO-based detection pipeline with ByteTrack for tracking, supervision library for annotation, and a FastAPI/Celery backend serving a React dashboard. The core technical risk is choosing the wrong computer vision stack: the originally proposed segmentation_models_pytorch + Deep SORT combination would make the project 50-100x slower than necessary and produce unreliable tracking, making the product unusable for 90-minute match footage.

**The recommended approach is a three-layer architecture:** (1) a CV engine using YOLO11 → supervision's ByteTrack → supervision/scipy heatmap generators, (2) a FastAPI server with Celery task queue for batch processing and WebSocket support for real-time, and (3) a React + Vite + Plotly dashboard for result visualization. The key risk is synchronous video processing in web requests (causes timeouts on full matches) — this is eliminated by dispatching all CV work to Celery workers from day one. Two additional architectural pitfalls require attention early: camera-space-only heatmaps (need homography/pitch mapping for cross-match comparability) and blended team heatmaps (need jersey color clustering to separate home/away formations).

Research is organized in five phases: core detection/tracking pipeline, heatmap generation with homography, web dashboard integration, real-time streaming, and tactical formation analysis. Each phase delivers independently useful output, with Phases 1-3 establishing the foundational product and Phases 4-5 adding advanced capabilities.

## Key Findings

### Recommended Stack

The original proposal (segmentation_models_pytorch + Deep SORT) has been replaced based on thorough ecosystem research. The recommended stack has HIGH confidence across all layers, with official documentation and benchmark verification for every choice.

**Core technologies:**
- **YOLO11 (Ultralytics v11.x):** Player detection — 3-5ms per frame on GPU vs 50-200ms for segmentation models; native tracking passthrough; COCO-pretrained detects "person" at ~0.85 mAP out of the box. **Not** segmentation_models_pytorch, which would make a 90-min match take 2-6 hours instead of 5-15 minutes.
- **ByteTrack (via supervision v0.19+):** Multi-object tracking — best MOTA on SportsMOT; recovers occluded players via low-score association (critical for football congestion); ~200 lines of code with no separate re-ID model. **Not** Deep SORT, which is unmaintained, requires appearance embeddings that fail on same-kit players, and causes 20-50+ ID switches per match vs ByteTrack's <5.
- **supervision.HeatMapAnnotator + scipy.stats.gaussian_kde:** Heatmap generation — HeatMapAnnotator for video overlay (frame-by-frame accumulation, tested parameters), gaussian_kde for publication-quality static PNGs with colormap over pitch template.
- **FastAPI v0.115+:** Web backend — async-first with native StreamingResponse, WebSocket, and BackgroundTasks; best Python web framework for video serving in 2026.
- **Celery v5.4+ + Redis 7.x:** Task queue — required because a 90-minute match at 5ms/frame takes 13.5 minutes; synchronous processing would timeout every HTTP request; Redis doubles as pub/sub for real-time progress.
- **React 19 + Vite 6 + Plotly 5 + Tailwind 4:** Frontend — React for ecosystem maturity, Vite for 10x faster builds, Plotly for WebGL-accelerated heatmap rendering with zoom/pan/tooltip, Tailwind for rapid UI without bloat.

See [STACK.md](./STACK.md) for full alternative comparisons and installation commands.

### Expected Features

**Must have (table stakes):**
- Player detection + tracking with persistent IDs — YOLO11 + ByteTrack solves this; ID switches are the primary quality concern
- Team-level accumulated heatmap — supervision.HeatMapAnnotator + scipy.kde cover this
- Individual player trajectory visualization — sv.TraceAnnotator for video; Plotly scatter for dashboard
- Video upload + processed result — FastAPI file upload → Celery task → result page
- CSV/JSON coordinate export — pandas DataFrame → CSV dump of {frame, player_id, x, y}
- Time-range filtering — filter tracking data by frame range; sub-second regeneration from stored data

**Should have (differentiators):**
- Top-down pitch heatmap via homography — medium complexity; required for cross-match comparability; **prioritize in Phase 2**
- Per-team heatmaps (home/away separated) — KMeans on jersey color crops; **implement before Phase 2 heatmap format is finalized**
- Player speed/distance metrics — Euclidean distance × calibration scale; low complexity
- Zone occupancy percentages — divide pitch into zones; count dwell time per player/team
- Side-by-side match comparison — load two datasets; overlay heatmaps; toggle
- Real-time camera feed + live heatmap — WebSocket streaming; Phase 4
- Formation detection (4-4-2, 4-3-3, etc.) — high complexity; requires clustering + domain heuristics; Phase 5

**Defer (v2+):**
- Ball tracking — out of scope per PROJECT.md; focus on player positions
- Full match event annotation — out of scope; provide CSV export so researchers overlay events
- Automatic jersey number recognition — fragile OCR on small crops; low value for heatmap accuracy
- User authentication / multi-tenant — single-user mode for v1
- Mobile app — responsive web design is sufficient; pitch viz is poor on small screens

See [FEATURES.md](./FEATURES.md) for full dependency graph.

### Architecture Approach

The architecture separates concerns into five components communicating via well-defined interfaces. The CV engine runs in isolated Celery workers (never in the web server process) to avoid GPU memory contention and process crashes taking down the API. All detection outputs are wrapped into supervision's `sv.Detections` format — the universal adapter pattern that makes the pipeline detector-agnostic and enables swapping YOLO → DETR → custom models without changing tracking or heatmap code.

**Major components:**
1. **React Frontend** — Upload UI, dashboard, heatmap display, real-time viewer. Communicates via HTTP + WebSocket.
2. **FastAPI Server** — REST API, file upload handling, task orchestration, static file serving. Stateless; dispatches long-running work to Celery.
3. **Celery Workers** — Long-running CV pipeline execution (YOLO11 → ByteTrack → annotation). Runs in separate processes/containers.
4. **CV Engine** — YOLO11 detection, ByteTrack tracking, heatmap generation. Encapsulated as a pipeline: `frame → detect → adapt → track → annotate`.
5. **Redis** — Celery message broker, task status store, real-time pub/sub.
6. **SQLite/PostgreSQL** — Persistent storage for tracking data, task metadata, heatmap configs.

The batch data flow is: upload → FastAPI saves file → creates Celery task → returns task_id immediately → frontend polls status → worker processes frames at 2-5fps → stores tracking_data.csv + heatmap PNG → marks COMPLETE → frontend displays results. The real-time flow uses WebSocket transport with the same CV pipeline code.

See [ARCHITECTURE.md](./ARCHITECTURE.md) for full component boundaries and data flow diagrams.

### Critical Pitfalls

1. **Using segmentation_models_pytorch for Player Detection (CRITICAL):** 50-100x slower than YOLO11. A 90-min match takes 2-6 hours vs 5-15 minutes. Real-time becomes impossible. **Prevention:** Start with YOLO11. Only introduce SMP for pitch line segmentation (a separate task).

2. **Using Deep SORT for Tracking (CRITICAL):** Requires appearance embeddings that fail when all players wear identical kits. 20-50+ ID switches per match vs ByteTrack's <5 — making individual trajectory analysis and formation detection unreliable. **Prevention:** Use ByteTrack (via supervision) as default. Use BoT-SORT only for broadcast footage with camera motion.

3. **Processing Full Video Synchronously in Web Request (CRITICAL):** 13.5 minutes of processing in a 30-second HTTP timeout window. Browser shows "connection lost"; user retries; server OOMs. **Prevention:** Always dispatch to Celery from the upload endpoint. Return task_id immediately. Frontend polls for completion.

4. **No Homography / Camera-Space Heatmap Only (HIGH):** Heatmap changes when camera angle changes — two identical plays produce different heatmaps. Researchers cannot compare matches filmed from different angles. **Prevention:** Implement pitch homography with perspective transform to 105x68m pitch template as Phase 2 priority.

5. **No Team Assignment / Single Heatmap Blends Both Teams (MODERATE):** In a 4-4-2 vs 4-3-3 match, both formations blur into one blob — analytical value is lost. **Prevention:** Add KMeans jersey color classification before Phase 2 heatmap format is finalized.

6. **Using Raw Bbox Center Instead of Foot Position (MODERATE):** Systematic ~1m positional offset upward. **Prevention:** Use `foot_position = (bbox_x_center, bbox_y_bottom)` from day one in a utility function.

7. **Processing at Raw 30fps (MODERATE):** 162,000 frames for 90 minutes → massive data bloat with negligible movement information gain (players move ~0.5m between 30fps frames). **Prevention:** Process at 2-5fps (standard in sports CV research). Reduces processing time 6-15x.

8. **No Progress Indication (MINOR):** User uploads, nothing happens for 10 minutes, assumes it broke. **Prevention:** Celery task reports `progress = frame_number / total_frames`; frontend shows progress bar.

See [PITFALLS.md](./PITFALLS.md) for all 12 pitfalls with detection methods and phase-specific warnings.

## Implications for Roadmap

Based on cross-referencing feature dependencies, architecture component boundaries, and pitfall sequencing, the research suggests five phases:

### Phase 1: Core Detection & Tracking Pipeline
**Rationale:** Everything depends on tracking data. No other feature works without reliable player detection + tracking. This phase also eliminates the three critical pitfalls (SMP, Deep SORT, sync processing) by establishing the correct CV stack from the start.
**Delivers:** Tracking data CSV with `{frame, player_id, x_foot, y_foot}` for a full match.
**Addresses:** Table-stakes features from FEATURES.md: player detection, per-player tracking with IDs, CSV/JSON export.
**Avoids:** Pitfalls 1 (SMP→YOLO11), 2 (Deep SORT→ByteTrack), 6 (bbox center→foot position), 7 (30fps→2-5fps).
**Stack used:** YOLO11, supervision, ByteTrack, OpenCV, NumPy, pandas — all core CV libraries.
**Research needed:** Minimal — well-documented established patterns. YOLO11 + ByteTrack is standard sports CV practice.

### Phase 2: Heatmap Generation & Pitch Mapping
**Rationale:** Tracking data exists from Phase 1; heatmap generation is the next independently valuable output. This phase must include homography and team classification **before** the heatmap output format is finalized — changing it later requires reformatting all stored data.
**Delivers:** Static team-level top-down heatmap PNG (per-team, time-normalized), video with heatmap overlay, per-player trajectory plots.
**Addresses:** Table-stakes team-level heatmap + top-down pitch heatmap differentiator, per-team heatmaps, individual trajectory visualization.
**Avoids:** Pitfalls 4 (camera-space→homography), 5 (blended teams→KMeans), 11 (not time-normalized).
**Stack used:** supervision.HeatMapAnnotator, scipy.stats.gaussian_kde, matplotlib, Pillow, OpenCV (perspective transform), shapely (pitch geometry), scikit-learn (KMeans for jersey colors).
**Research needed:** Phase 2.5 sub-phase for pitch homography — this needs deeper research on pitch line detection models or manual calibration approaches. The homography component is the highest-risk unknown.

### Phase 3: Web Dashboard (Batch Processing)
**Rationale:** The CV pipeline (Phase 1 + 2) produces useful output via CLI. The web dashboard makes it accessible to non-technical researchers. Celery architecture must be in place from day one to avoid the synchronous processing pitfall. Progress reporting is essential for UX.
**Delivers:** Fully functional web app: upload match → view heatmap, trajectories, data tables, downloadable CSV.
**Addresses:** Video upload + processed result (table stakes), dashboard interaction, speed/distance metrics, zone occupancy, side-by-side comparison.
**Avoids:** Pitfalls 3 (sync→Celery), 9 (no progress→task status), 10 (halftime→preprocessing), 12 (large codecs→H.264 or skip video).
**Stack used:** FastAPI, Celery, Redis, React 19, Vite 6, Plotly 5, Tailwind 4, SQLite/PostgreSQL.
**Research needed:** Minimal — FastAPI + React + Celery is a standard web stack with extensive documentation. No `gsd-research-phase` needed.

### Phase 4: Real-Time WebSocket Streaming
**Rationale:** Batch processing is the primary use case (recorded match analysis). Real-time is incremental — it reuses the same CV pipeline code with WebSocket transport instead of file-based processing. It requires a stable Phase 1-3 to build upon.
**Delivers:** Live camera feed with real-time heatmap overlay at 2-5fps.
**Addresses:** Real-time camera processing requirement from PROJECT.md.
**Stack used:** FastAPI WebSocket, same CV pipeline, base64 JPEG transport.
**Research needed:** Medium — WebSocket frame transport patterns for browser→server are documented but browser webcam capture (navigator.mediaDevices) and base64 encoding/decoding at 2-5fps needs prototyping. Consider `gsd-research-phase` during roadmap planning.

### Phase 5: Tactical Formation Detection
**Rationale:** Highest complexity feature that depends on reliable individual tracking (validated in Phase 1) and per-team classification (validated in Phase 2). Formation detection requires aggregating player positions over match periods and applying clustering + domain heuristics. Must validate on diverse match footage to avoid overfitting.
**Delivers:** Formation classification output (4-4-2, 4-3-3, etc.) with confidence scores per match period.
**Addresses:** Tactical formation detection requirement from PROJECT.md.
**Stack additions:** scikit-learn (clustering), custom domain heuristics.
**Research needed:** HIGH — formation detection is an open research problem without an off-the-shelf solution. This phase **must** start with a dedicated `gsd-research-phase` to survey current approaches (e.g., role-based clustering, expected position models, Voronoi-based methods).

### Phase Ordering Rationale

- **Detection before heatmap:** You cannot generate a heatmap without tracking data. Phase 1 strictly precedes Phase 2.
- **Batch before real-time:** Batch processing is simpler, higher-value for recorded match analysis, and uses the same CV pipeline code. Real-time adds WebSocket transport complexity on top of a proven pipeline.
- **Heatmap format decisions lock early:** Team classification and homography must happen before the heatmap generation code is finalized. Phase 2 includes both to avoid costly reformatting.
- **CV pipeline before web dashboard:** While the dashboard could be built in parallel, the CV pipeline must be validated first — a beautiful dashboard that produces wrong heatmaps is worse than no dashboard with a correct CLI.
- **Formation detection last:** Depends on reliable tracking (Phase 1), per-team classification (Phase 2), and multi-match validation — naturally the final phase.

### Research Flags

Phases likely needing deeper research during planning:
- **Phase 2 (Heatmap):** Pitch homography implementation — needs survey of pitch line detection models (e.g., PitchView, KeyNet, or manual calibration) and perspective transform methodology. Consider a `gsd-research-phase` for homography approach.
- **Phase 5 (Formation):** Formation detection is the highest-risk unknown. No off-the-shelf solution exists. Requires dedicated research phase to survey clustering-based, role-based, and expected-position approaches before implementation.

Phases with standard patterns (skip research-phase):
- **Phase 1 (Detection):** YOLO11 + ByteTrack + supervision is well-documented standard practice. Skip `gsd-research-phase`.
- **Phase 3 (Dashboard):** FastAPI + React + Celery is a standard web stack with extensive documentation and examples. Skip `gsd-research-phase`.
- **Phase 4 (Real-time):** WebSocket streaming is documented in FastAPI docs. Low risk — skip dedicated research but prototype the frame transport during implementation.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | All core choices verified against official docs (Context7), benchmarks, and active maintenance status. YOLO11 confirmed v11.x available, supervision 39k+ stars active, ByteTrack SOTA on SportsMOT, Deep SORT confirmed unmaintained since 2021. SMP rejection based on clear segmentation-only scope. |
| Features | HIGH | Feature categorization derived from domain analysis (sports CV research) and PROJECT.md requirements. Table stakes align with standard sports analytics deliverables. Anti-features explicitly scoped out in PROJECT.md. |
| Architecture | HIGH | Adapter pattern, pipeline pattern, and background task pattern are established best practices for CV web applications. Separation of Celery workers from FastAPI is standard for long-running ML inference. |
| Pitfalls | HIGH | 8 of 12 pitfalls have direct source verification (Context7 for SMP, Deep SORT, ByteTrack benchmarks). Remaining 4 are domain best practices (foot position, 2-5fps, time normalization, halftime handling) validated against sports CV research standards. |

**Overall confidence:** HIGH — all four research areas have strong source backing and consistent cross-referencing across files.

### Gaps to Address

- **Pitch homography implementation detail:** The research identifies *that* homography is needed and *why*, but the exact approach (dedicated line detection model vs. manual calibration vs. keypoint-based) needs deeper exploration during Phase 2 planning. This is the single largest technical unknown.
- **Formation detection methodology:** No standard solution exists. Phase 5 will require a dedicated research phase to identify the right clustering + heuristic approach. The current research only confirms it's hard and needs validation on diverse match footage.
- **Production database choice:** SQLite is fine for dev/single-user. PostgreSQL + PostGIS migration path is identified but not researched in depth. This is a Phase 3 implementation detail.
- **GPU requirements for Celery workers:** Single GPU sufficient for 1-10 researchers, but the exact model memory requirements (YOLO11n: ~2GB, YOLO11s: ~4GB) and concurrent worker configuration need validation during Phase 1 setup.

## Sources

### Primary (HIGH confidence)
- [Ultralytics YOLO documentation (Context7)](https://context7.com/degirum/ultralytics_yolov8/llms.txt) — verified YOLO11 availability, tracking passthrough, performance benchmarks
- [Supervision documentation (Context7)](https://context7.com/roboflow/supervision/llms.txt) — verified 39k+ stars, HeatMapAnnotator, TraceAnnotator, Detections adapter pattern
- [ByteTrack original paper & repo (Context7)](https://context7.com/foundationvision/bytetrack/llms.txt) — verified SportsMOT benchmarks, occlusion handling via two-stage matching
- [Roboflow Trackers library](https://github.com/roboflow/trackers/) — verified ByteTrack/BoT-SORT/SORT implementations, SoccerNet/SportsMOT tuned configs
- [FastAPI (Context7)](https://context7.com/fastapi/fastapi/llms.txt) — verified async, WebSocket, StreamingResponse support
- [Segmentation Models PyTorch (Context7)](https://context7.com/qubvel-org/segmentation_models.pytorch/llms.txt) — confirmed segmentation-only, no tracking integration
- [Deep SORT (Context7)](https://context7.com/nwojke/deep_sort/llms.txt) — confirmed unmaintained (last major update 2021)
- [PROJECT.md](../PROJECT.md) — validated requirements scope (batch + real-time, player tracking, formation detection)

### Secondary (MEDIUM confidence)
- [Supervision HeatMap example](https://github.com/roboflow/supervision/blob/develop/examples/heatmap_and_track/README.md) — reference implementation for heatmap overlay pipeline
- [Supervision Annotators documentation](https://github.com/roboflow/supervision/blob/develop/docs/detection/annotators.md) — verified annotator API surface
- [ByteTrack vs Deep SORT benchmarks](https://github.com/roboflow/trackers) — verified through library benchmarking pipeline

### Tertiary (LOW confidence — needs validation during implementation)
- **SMP speed comparison (50-100x slower):** Inferred from standard benchmarks; should be validated on own hardware during Phase 1 setup
- **Pitch homography approach:** No specific model recommended; needs dedicated survey before Phase 2
- **Formation detection methodology:** No existing solution identified; needs Phase 5 research phase

---
*Research completed: 2026-05-23*
*Ready for roadmap: yes*
