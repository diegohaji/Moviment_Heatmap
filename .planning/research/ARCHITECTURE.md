# Architecture Patterns

**Domain:** Computer vision football movement analysis
**Researched:** 2026-05-23

## Recommended Architecture

```
┌─────────────────────────────────────────────────────────┐
│                     CLIENT (Browser)                     │
│  ┌──────────────────────────────────────────────────┐   │
│  │              React + Vite + Plotly                │   │
│  │  ┌─────────┐ ┌──────────┐ ┌───────────────────┐  │   │
│  │  │ Upload  │ │ Dashboard│ │ Real-time Stream  │  │   │
│  │  │  Page   │ │ (Results)│ │ (WebSocket view)  │  │   │
│  │  └────┬────┘ └────┬─────┘ └────────┬──────────┘  │   │
│  └───────┼───────────┼────────────────┼──────────────┘   │
└──────────┼───────────┼────────────────┼──────────────────┘
           │           │                │
     POST /upload │   GET /tasks/{id}  │ WebSocket /ws
           │           │                │
┌──────────▼───────────▼────────────────▼──────────────────┐
│                    FASTAPI SERVER                         │
│                                                            │
│  ┌────────────────────────────────────────────────────┐   │
│  │              API Layer (REST + WebSocket)           │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────────────┐  │   │
│  │  │ Upload   │ │ Task     │ │ WebSocket        │  │   │
│  │  │ Endpoint │ │ Endpoints│ │ Handler          │  │   │
│  │  └────┬─────┘ └────┬─────┘ └────────┬─────────┘  │   │
│  └───────┼────────────┼────────────────┼──────────────┘   │
│          │            │                │                   │
│  ┌───────▼────────────▼────────────────▼──────────────┐   │
│  │              Service Layer                          │   │
│  │  ┌────────────┐ ┌─────────────┐ ┌──────────────┐  │   │
│  │  │ Video      │ │ Tracking    │ │ Heatmap      │  │   │
│  │  │ Processing │ │ Data Store  │ │ Generator    │  │   │
│  │  └────────────┘ └─────────────┘ └──────────────┘  │   │
│  └────────────────────────────────────────────────────┘   │
│                                                            │
│  ┌────────────────────────────────────────────────────┐   │
│  │           Task Queue (Celery + Redis)               │   │
│  │  ┌──────────────────────────────────────────────┐  │   │
│  │  │  Batches: YOLO11 → ByteTrack → CSV/Heatmap   │  │   │
│  │  └──────────────────────────────────────────────┘  │   │
│  └────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────┘
                               │
┌──────────────────────────────▼────────────────────────────┐
│                      CV ENGINE                            │
│                                                            │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐  │
│  │   YOLO11     │──▶│   ByteTrack  │──▶│   supervision │  │
│  │  (detection) │   │  (tracking)  │   │  (HeatMap/    │  │
│  │              │   │              │   │   Trace)      │  │
│  └──────────────┘   └──────┬───────┘   └──────────────┘  │
│                            │                              │
│                     ┌──────▼───────┐                     │
│                     │  tracking_   │                     │
│                     │  data.json   │                     │
│                     └──────────────┘                     │
└────────────────────────────────────────────────────────────┘
```

### Component Boundaries

| Component | Responsibility | Communicates With |
|---|---|---|
| **React Frontend** | Upload UI, dashboard, heatmap display, real-time viewer | FastAPI (HTTP + WebSocket) |
| **FastAPI Server** | REST API, file upload handling, task orchestration, static file serving | Frontend, Celery, Redis |
| **Celery Workers** | Long-running CV pipeline execution | Redis (broker), FastAPI (status) |
| **CV Engine** | YOLO11 detection, ByteTrack tracking, heatmap generation | OpenCV (video I/O), NumPy (data) |
| **Redis** | Celery message broker, task status, real-time pub/sub | FastAPI, Celery |
| **SQLite/PostgreSQL** | Persistent storage: tracking data, task metadata, heatmap configs | FastAPI |

### Data Flow: Batch Processing

```
1. User uploads match video via React UI
2. FastAPI receives file, saves to disk, creates Celery task
3. Returns {task_id} immediately — frontend polls GET /tasks/{task_id}
4. Celery worker:
   a. Opens video with OpenCV
   b. For each frame:
      - YOLO11 → detection boxes
      - ByteTrack → assign/update tracking IDs
      - supervision.HeatMapAnnotator → accumulate heatmap
      - Store {frame, player_id, bbox_center} in list
   c. After all frames:
      - Save tracking_data.csv
      - Generate static heatmap PNG (scipy.gaussian_kde)
      - Generate annotated video (if requested)
      - Mark task as COMPLETE
5. Frontend polls detects COMPLETE → fetches results
6. Dashboard displays: heatmap PNG, trajectory plot, data table
```

### Data Flow: Real-Time Processing

```
1. User connects via WebSocket from browser
2. FastAPI accepts WebSocket, spawns async handler
3. Browser captures webcam frames (navigator.mediaDevices)
4. Sends each frame as base64 JPEG over WebSocket
5. FastAPI receives frame, passes to CV Engine (same code as batch)
6. CV Engine returns:
   - Annotated frame (base64 JPEG with bounding boxes + trails)
   - Current heatmap overlay
7. FastAPI sends result back over WebSocket
8. Browser renders frame in <video> element + Plotly overlay
```

---

## Patterns to Follow

### Pattern 1: Adapter Pattern for Detections
**What:** Wrap all detection model outputs into supervision's `sv.Detections` format.
**When:** Always — `sv.Detections` is the universal interface for all downstream consumers (trackers, annotators, analyzers).
**Why:** Detector-agnostic architecture. Swap YOLO → DETR → custom model without changing tracking/heatmap code.
**Example:**
```python
import supervision as sv
from ultralytics import YOLO

model = YOLO("yolo11n.pt")
result = model(frame)[0]
detections = sv.Detections.from_ultralytics(result)
# Now detections flows to ByteTrack, HeatMapAnnotator, etc. seamlessly
```

### Pattern 2: Pipeline Pattern for CV Engine
**What:** Frame → Detection → Tracking → Annotation as linear pipeline.
**When:** Every video processing task (batch or real-time).
**Example:**
```python
def process_frame(frame: np.ndarray, tracker: sv.ByteTrack, heatmap: sv.HeatMapAnnotator):
    result = model(frame)[0]                    # Step 1: Detect
    detections = sv.Detections.from_ultralytics(result)  # Step 2: Adapt
    detections = tracker.update_with_detections(detections)  # Step 3: Track
    annotated = heatmap.annotate(frame.copy(), detections)  # Step 4: Annotate
    return annotated, detections
```

### Pattern 3: Background Task Status Pattern
**What:** Return immediately from upload endpoint; poll for completion.
**When:** All batch processing (matches can take 10-60 min).
**Example:**
```python
# FastAPI
@app.post("/upload")
async def upload_video(file: UploadFile):
    task = process_video_task.delay(file_path)
    return {"task_id": task.id}

@app.get("/tasks/{task_id}")
async def get_task_status(task_id: str):
    task = AsyncResult(task_id, app=celery_app)
    return {"status": task.status, "result": task.result}
```

---

## Anti-Patterns to Avoid

### Anti-Pattern 1: Synchronous Video Processing
**What:** Processing a 90-minute match in the HTTP request handler.
**Why bad:** Request times out (30s default); server blocks other requests; browser hangs.
**Instead:** Always dispatch to Celery; return task_id immediately.

### Anti-Pattern 2: Processing Full Video for Every Request
**What:** Re-running detection + tracking every time the user changes a filter (e.g., time range).
**Why bad:** 10-60 min per run; users will not wait.
**Instead:** Process once, store per-frame tracking data. Filters (time range, player selection) operate on stored data only — sub-second.

### Anti-Pattern 3: Mixing CV and Web Server in Same Process
**What:** Running YOLO inference in the same Python process as FastAPI.
**Why bad:** Model loading uses 2-4GB GPU memory; GPU contention; Celery worker crash takes down API.
**Instead:** Separate Celery worker processes (or containers) for CV tasks. FastAPI is stateless.

### Anti-Pattern 4: Segmentation over Detection for Position
**What:** Using pixel-perfect segmentation masks to determine player position.
**Why bad:** 50-100x slower; heatmap spatial resolution gain is negligible; far harder to fine-tune.
**Instead:** Use bbox center-bottom (standard foot position convention) for pitch mapping.

---

## Scalability Considerations

| Concern | At 1 user / 1 match | At 10 users / 10 matches | At 100 users / concurrent |
|---|---|---|---|
| **Video storage** | Local filesystem | Object storage (S3/MinIO) | S3 + CDN for results |
| **Task queue** | Celery + Redis on same machine | Celery + Redis (separate) | Celery worker auto-scaling (Kubernetes) |
| **Model serving** | Single GPU on worker | 1 GPU per 3-4 concurrent workers | GPU cluster with Triton Inference Server |
| **Database** | SQLite | PostgreSQL | PostgreSQL read replicas + connection pooling |
| **Heatmap regeneration** | On-demand from CSV | On-demand from CSV (same) | Pre-cached; periodic refresh |
| **Real-time streaming** | Single WebSocket | Mutex per camera stream | Dedicated stream processing nodes |

**Key insight for this project:** At academic scale (1-10 researchers), a single GPU machine with Celery workers on the same host is sufficient. The architecture should support horizontal scaling but should NOT over-engineer for it in v1.
