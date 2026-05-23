# Feature Landscape

**Domain:** Computer vision football movement analysis
**Researched:** 2026-05-23

## Table Stakes

Features users expect. Missing any = product feels incomplete.

| Feature | Why Expected | Complexity | Notes |
|---|---|---|---|
| Player detection from video | Core function; replaces manual annotation | Medium | YOLO11 solves this; needs football-specific fine-tuning for best results |
| Per-player tracking with persistent IDs | Required for individual trajectory analysis | Medium | ByteTrack solves this; ID switches are the main quality concern |
| Team-level accumulated heatmap | Most basic sports analytics deliverable | Low | supervision.HeatMapAnnotator + scipy.kde cover this |
| Video upload + processed result | UX baseline; no CLI-only tools in 2026 | Medium | FastAPI file upload → Celery task → result page |
| CSV/JSON coordinate export | Researchers need raw data for their own analysis | Low | pandas DataFrame → CSV dump of {frame, player_id, x, y} |
| Time-range filtering (select match period) | Full match + specific half/scenario analysis | Low | Filter tracking data by frame range; regenerate heatmap subset |
| Individual player trajectory visualization | Per-player movement paths; required for tactical analysis | Medium | sv.TraceAnnotator for video; Plotly scatter for dashboard |

## Differentiators

Features that set product apart. Not expected, but valued.

| Feature | Value Proposition | Complexity | Notes |
|---|---|---|---|
| Formation detection (4-4-2, 4-3-3, etc.) | Distinctive academic value; no off-the-shelf solution | High | Requires clustering + domain heuristics; Phase 5 |
| Top-down pitch heatmap (homography) | Researchers see pitch-space occupancy, not camera-space | Medium | Requires pitch line detection + perspective transform matrix |
| Per-team heatmaps (home/away separated) | Tactical analysis of team shape vs opponent | Medium | Requires team classification from jersey colors (KMeans on bbox crop) |
| Side-by-side comparison (two matches) | Compare tactical setups across matches | Medium | Load two tracking datasets; overlay heatmaps; toggle | 
| Real-time camera feed + live heatmap | Live match analysis during training | High | WebSocket streaming + on-the-fly heatmap updates; Phase 4 |
| Player speed/distance metrics | Performance analytics beyond position | Low | Euclidean distance between frames × calibration scale |
| Zone occupancy percentages | Quantified pitch control statistics | Low | Divide pitch into zones; count dwell time per player/team |

## Anti-Features

Features to explicitly NOT build.

| Anti-Feature | Why Avoid | What to Do Instead |
|---|---|---|
| Ball tracking | In-scope per PROJECT.md; adds complexity for minimal movement-analysis value | Focus on player positions; ball is secondary |
| Full match event annotation (goals, cards, fouls) | Out of scope per PROJECT.md; requires different model (action recognition) | Provide CSV export so researchers can overlay events themselves |
| Real-time at 30fps | Unnecessary for heatmap (aggregates over time); 2-5fps is sufficient for movement analysis | Target 2-5fps for real-time; downstream aggregation smooths frames |
| Automatic player jersey number recognition | Requires OCR on small crops; fragile and low-value for heatmap accuracy | Manual assignment in dashboard if needed |
| Mobile app | Complex; pitch visualization is poor on small screens | Responsive web design; defer native apps |
| User authentication / multi-tenant | Not in requirements; adds auth complexity | Single-user mode for v1; add if needed |

## Feature Dependencies

```
Detection (YOLO11) → Tracking (ByteTrack) → Base tracking data (CSV)
    ↓
Tracking data → Heatmap generation (no other dependencies)
Tracking data → Trajectory visualization
Tracking data → CSV/JSON export
Tracking data → Speed/distance metrics
    ↓
Tracking data → Formation detection (requires aggregation over match periods)
Tracking data → Zone occupancy (requires pitch divisions)
    ↓
Web Dashboard (Batch) → All above + FastAPI + Celery
Real-time (WebSocket) → Detection + Tracking (same code, different transport)
    ↓
Pitch homography → Requires pitch line detection model (separate from player detection)
```

## MVP Recommendation

Prioritize:

1. **Batch CV pipeline** (Phase 1): YOLO11 detection + ByteTrack tracking → CSV tracking data
2. **Static heatmap generation** (Phase 2): scipy.gaussian_kde → pitch overlay PNG
3. **Video heatmap overlay** (Phase 2): supervision.HeatMapAnnotator → annotated match video
4. **Web dashboard** (Phase 3): FastAPI upload → Celery processing → react-plotly display

Defer:
- **Real-time streaming** (Phase 4): Requires WebSocket + frame transport complexity
- **Formation detection** (Phase 5): Requires multiple matches of training data to validate

**Why this order:** Each phase delivers independently valuable output. After Phase 1+2, a researcher can upload a match and get a heatmap. Phase 3 adds the UI. Phases 4-5 are additive.
