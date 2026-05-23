<!-- GSD:project-start source:PROJECT.md -->
## Project

**Moviment Heatmap**

A computer vision web application that processes football match footage to generate accumulated movement heatmaps and individual player trajectories. Built for sports science researchers and tactical analysts who need visual analytics of player positioning and movement patterns.

**Core Value:** Transform raw football match footage into actionable visual analytics — accumulated heatmaps and per-player trajectories — that reveal tactical patterns invisible to the naked eye.

### Constraints

- **Tech Stack**: Python-based for CV pipeline; web frontend for dashboard
- **Performance**: Must handle full 90-minute match footage within reasonable time
- **Data**: Needs both capture pipeline (camera input) and processing pipeline (video files)
<!-- GSD:project-end -->

<!-- GSD:stack-start source:research/STACK.md -->
## Technology Stack

## Recommended Stack
### Tier 0: Core Pipeline (Detection → Tracking → Heatmap)
### Detection & Tracking
| Technology | Version | Purpose | Why |
|---|---|---|---|
| **Ultralytics YOLO11** | 11.x (latest) | Player + referee detection | Best speed/accuracy tradeoff; 3-5ms per frame on GPU; natively supports tracking passthrough |
| **supervision** | 0.19+ | Detection → Detections adapter, annotators, ByteTrack integration | 39k+ GitHub stars; model-agnostic `Detections` format; purpose-built for this pipeline |
| **ByteTrack** (via supervision) | built-in | Multi-object tracking with ID persistence | Best MOTA on SportsMOT; recovers occluded players via low-score association critical for football congestion |
| **BoT-SORT** (via supervision) | optional | Tracking with camera motion compensation | Use for broadcast/panned footage; adds CMC via optical flow; higher accuracy on SoccerNet-tracking |
| **OpenCV** | 4.x (opencv-python-headless) | Video I/O, frame reading/writing, perspective transform | Ubiquitous CV library; no viable alternative for raw frame manipulation |
### Heatmap Generation
| Technology | Purpose | Why |
|---|---|---|
| **supervision.HeatMapAnnotator** | Per-frame heatmap overlay on video | Purpose-built; configurable opacity/radius; works directly with YOLO + ByteTrack; outputs video with density overlay |
| **NumPy + SciPy** | Accumulated static heatmap (team-level) | `scipy.stats.gaussian_kde` for density estimation from tracked positions; grid-based aggregation for non-video heatmap images |
| **Matplotlib + Pillow** | Static heatmap rendering to PNG | For dashboard display without re-processing video; colormap compositing over pitch template |
### Web Backend
| Technology | Version | Purpose | Why |
|---|---|---|---|
| **FastAPI** | 0.115+ | REST API + WebSocket + background tasks | Async-first; native `StreamingResponse` for video; `BackgroundTasks` for batch processing; 1st-class WebSocket for real-time streams; best Python web framework for this use case in 2026 |
| **Celery** | 5.4+ | Async task queue for 90-min match processing | Required; FastAPI alone cannot handle synchronous 90-min video processing without blocking; Celery + Redis broker for durable queue |
| **Redis** | 7.x | Celery broker + cache + WebSocket pub/sub | Dual-purpose: Celery message broker + real-time progress updates via pub/sub |
| **SQLite** (dev) / **PostgreSQL** (prod) | — | Tracking data persistence (positions, trajectories) | SQLite for zero-config dev; PostgreSQL for concurrent access + PostGIS spatial queries in production |
### Web Frontend
| Technology | Version | Purpose | Why |
|---|---|---|---|
| **React** | 19.x | Dashboard UI | Dominant ecosystem; best library support for charting/video; vast talent pool |
| **Vite** | 6.x | Build tool | 10x faster than CRA; native ESM; HMR for rapid iteration |
| **Plotly** (Dash) | 5.x | Interactive heatmap visualization | WebGL-accelerated; zoom/pan/tooltip; heatmap + scatter on pitch overlay; Python/JS dual support |
| **Tailwind CSS** | 4.x | Styling | Utility-first; rapid UI building; no CSS bloat |
### Supporting Libraries
| Library | Purpose | Condition |
|---|---|---|
| `pandas` | Tracking data manipulation, CSV/JSON export | Always |
| `shapely` | Pitch geometry (converting tracked positions to pitch coordinates via homography) | When doing perspective-to-top-down mapping |
| `httpx` | Async HTTP client for frontend↔backend data fetch | Always |
| `websockets` (Python stdlib) | Real-time video frame streaming | When using WebSocket mode |
| `docker` / `docker-compose` | Containerized deployment (CV pipeline + web server + Redis) | Production deployment |
## Deep Dive: Point-by-Point Rationale
### 1. YOLO11 — Why NOT segmentation_models_pytorch
| Criterion | YOLO11 (n/s/m/l/x) | segmentation_models_pytorch |
|---|---|---|
| **Inference speed** | 1-5ms per frame (nano: ~1ms, small: ~2ms) | 50-200ms per frame (Unet + ResNet34) |
| **Task focus** | Detection (bbox) + segmentation (mask) | Segmentation only |
| **Tracking integration** | Native: `model.track(persist=True)` | No built-in; must pipe to separate tracker |
| **Pre-trained weights** | COCO + ImageNet; football-specific fine-tunes available | ImageNet only |
| **Ecosystem** | Ultralytics ecosystem (training, export, deployment) | Standalone model library |
| **Player detection approach** | Bounding box (sufficient for tracking + heatmap) | Pixel mask (overkill; dramatically slower for minimal heatmap accuracy gain) |
| **Sports benchmark** | SportsMOT HOTA: 0.632 (YOLOX + ByteTrack) | Not benchmarked on sports tracking |
### 2. ByteTrack — Why NOT Deep SORT
| Criterion | ByteTrack | Deep SORT | BoT-SORT |
|---|---|---|---|
| **MOTA on SportsMOT** | 0.632 (SOTA tier) | Not benchmarked | 0.641 (best) |
| **Occlusion handling** | Two-stage matching (high + low confidence detections) | Appearance embedding + Kalman filter | ByteTrack + Camera Motion Compensation |
| **Complexity** | ~200 lines, no separate re-ID model | Requires separate appearance embedding model (e.g., ResNet-50) | ByteTrack + optical flow CMC |
| **Football suitability** | Excellent — recovers players when occluded in crowds | Moderate — appearance embeddings fail when all players wear similar kits | Best for broadcast footage with camera motion |
| **Maintenance** | Active (2024-2026) | Stale (last major update 2021) | Active (part of supervision trackers library) |
| **Integration with YOLO** | `model.track(source, tracker="bytetrack.yaml")` — one line | Requires custom pipeline | `tracker="botsort.yaml"` — one line |
- Clean `sv.Detections` adapter pattern (detector-agnostic)
- Seamless integration with supervision annotators (HeatMap, Trace, etc.)
- Access to all 4 trackers (SORT, ByteTrack, OC-SORT, BoT-SORT) through the same API
- Documented tuned parameters for SoccerNet/ SportsMOT datasets
### 3. Heatmap — supervision.HeatMapAnnotator vs Manual
| Approach | Best for | Why |
|---|---|---|
| **sv.HeatMapAnnotator** | Video overlay (real-time and recorded) | Built for this; processes frame-by-frame; outputs annotated video |
| **scipy.gaussian_kde + matplotlib** | Static team-level heatmap image | For the dashboard display where you need a pitch-sized PNG with colormap |
| **NumPy 2D histogram** | Accumulated occupancy grid | Simplest approach; just bin all tracked positions into a grid; fast enough for interactive exploration |
### 4. Web Dashboard — Stack Reasoning
| Layer | Choice | Rejected Alternatives |
|---|---|---|
| **Frontend framework** | React + Vite | **Svelte**: Smaller ecosystem for charting. **Vue**: Less JS chart library support. **Flask + Jinja2**: Cannot deliver interactive dashboard experience. |
| **Charting** | Plotly (Dash) | **D3.js**: Too low-level for rapid development. **Chart.js**: Lacks heatmap. **Observable Plot**: Great but smaller community. |
| **Styling** | Tailwind CSS | **Material UI**: Too opinionated for custom pitch layouts. **Chakra**: Less mature v3. |
| **State management** | React Context + useReducer (or Zustand for scale) | **Redux**: Overkill for this scope. **Recoil**: Deprecated patterns. |
### 5. Real-Time vs Batch Processing
| Dimension | Batch Processing | Real-Time Processing |
|---|---|---|
| **Use case** | Upload recorded match → process in background | Live camera feed → streaming analysis |
| **Framework** | FastAPI + Celery (task queue) | FastAPI + WebSocket |
| **Video source** | Uploaded file (MP4, MOV) | Webcam / RTSP stream |
| **Processing** | Full 90 min at once; progress via Celery task status | Frame-by-frame; ~2-5fps target (YOLO11n on mid-range GPU) |
| **Storage** | tracking_data.json saved to disk; heatmap PNG generated at end | No persistent storage; display-only |
| **Architecture** | `POST /upload` → returns task_id → `GET /task/{id}/status` polls progress → `GET /task/{id}/result` downloads | `WebSocket /ws/stream` → frames in, annotations out |
## Alternatives Considered
| Category | Recommended | Alternative | Why Not |
|---|---|---|---|
| Detection model | YOLO11 | segmentation_models_pytorch | 50-100x slower; segmentation is overkill for player position; no tracking integration |
| Detection model | YOLO11 | Detectron2 | Heavier, Facebook-maintained (lower community activity in 2025+); no native tracking |
| Detection model | YOLO11 | YOLOv8 | YOLO11 is the current version; ~15% faster with same accuracy; available since Sep 2024 |
| Tracker | ByteTrack (via supervision) | Deep SORT | Unmaintained since 2021; requires separate re-ID model; worse occlusion handling |
| Tracker | ByteTrack | BoT-SORT | More complex; CMC overhead not needed for static-camera footage; use only for broadcast |
| Backend framework | FastAPI | Flask | Flask has no native async, no WebSocket, no StreamingResponse; requires extensions for each |
| Task queue | Celery | Dramatiq | Celery is more battle-tested with FastAPI; larger ecosystem |
| Frontend framework | React + Vite | Next.js | SSR not needed for dashboard; Vite is simpler; avoid framework lock-in |
| Heatmap video | supervision | OpenCV custom | Why reinvent what supervision already provides with tested parameters |
| Heatmap static | scipy.kde + matplotlib | seaborn | Seaborn kdeplot wraps scipy; no benefit over direct scipy for custom grid |
## Installation
### Core CV Pipeline
### Web Backend
### Web Frontend
# Create React + Vite project
### Development
## Version Confidence
| Stack Component | Version Confidence | Source |
|---|---|---|
| YOLO11 (ultralytics) | HIGH — verified in Context7 (v11.x available, latest release v8.3.0) | Context7: `/degirum/ultralytics_yolov8` |
| supervision 0.19+ | HIGH — verified in Context7 (39k stars, active development) | Context7: `/roboflow/supervision` |
| ByteTrack / BoT-SORT | HIGH — verified in Context7 + roboflow trackers lib | Context7: `/roboflow/trackers`, `/foundationvision/bytetrack` |
| FastAPI | HIGH — verified in Context7 (v0.115+ available) | Context7: `/fastapi/fastapi` |
| Deep SORT rejection | HIGH — confirmed unmaintained; verified in Context7 | Context7: `/nwojke/deep_sort` |
| SMP rejection | MEDIUM — documentation confirms segmentation-only; speed comparison based on standard benchmarks | Context7: `/qubvel-org/segmentation_models.pytorch` |
| HeatMapAnnotator | HIGH — verified in supervision docs | supervision docs + heatmap example |
| React + Vite | HIGH — standard web dev practice | General web ecosystem |
## Sources
- [Ultralytics YOLO documentation (Context7)](https://context7.com/degirum/ultralytics_yolov8/llms.txt)
- [Supervision documentation (Context7)](https://context7.com/roboflow/supervision/llms.txt)
- [Roboflow Trackers library (Context7)](https://github.com/roboflow/trackers/) — ByteTrack, BoT-SORT, OC-SORT, SORT
- [ByteTrack original paper & repo (Context7)](https://context7.com/foundationvision/bytetrack/llms.txt)
- [Segmentation Models PyTorch (Context7)](https://context7.com/qubvel-org/segmentation_models.pytorch/llms.txt)
- [FastAPI (Context7)](https://context7.com/fastapi/fastapi/llms.txt)
- [Supervision HeatMap example](https://github.com/roboflow/supervision/blob/develop/examples/heatmap_and_track/README.md)
- [Supervision Annotators doc](https://github.com/roboflow/supervision/blob/develop/docs/detection/annotators.md)
- [Tracker comparison (SoccerNet/SportsMOT tuned configs)](https://github.com/roboflow/trackers/blob/develop/docs/trackers/botsort.md)
<!-- GSD:stack-end -->

<!-- GSD:conventions-start source:CONVENTIONS.md -->
## Conventions

Conventions not yet established. Will populate as patterns emerge during development.
<!-- GSD:conventions-end -->

<!-- GSD:architecture-start source:ARCHITECTURE.md -->
## Architecture

Architecture not yet mapped. Follow existing patterns found in the codebase.
<!-- GSD:architecture-end -->

<!-- GSD:skills-start source:skills/ -->
## Project Skills

No project skills found. Add skills to any of: `.claude/skills/`, `.agents/skills/`, `.cursor/skills/`, `.github/skills/`, or `.codex/skills/` with a `SKILL.md` index file.
<!-- GSD:skills-end -->

<!-- GSD:workflow-start source:GSD defaults -->
## GSD Workflow Enforcement

Before using Edit, Write, or other file-changing tools, start work through a GSD command so planning artifacts and execution context stay in sync.

Use these entry points:
- `/gsd-quick` for small fixes, doc updates, and ad-hoc tasks
- `/gsd-debug` for investigation and bug fixing
- `/gsd-execute-phase` for planned phase work

Do not make direct repo edits outside a GSD workflow unless the user explicitly asks to bypass it.
<!-- GSD:workflow-end -->



<!-- GSD:profile-start -->
## Developer Profile

> Profile not yet configured. Run `/gsd-profile-user` to generate your developer profile.
> This section is managed by `generate-claude-profile` -- do not edit manually.
<!-- GSD:profile-end -->
