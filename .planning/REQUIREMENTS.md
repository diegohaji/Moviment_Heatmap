# Requirements: Moviment Heatmap

**Defined:** 2026-05-23
**Core Value:** Transform raw football match footage into actionable visual analytics — accumulated heatmaps and per-player trajectories — that reveal tactical patterns invisible to the naked eye.

## v1 Requirements

Requirements for initial release. Each maps to roadmap phases.

### Core CV Pipeline

- [ ] **CV-01**: System detects players from match video footage using YOLO11
- [ ] **CV-02**: System tracks each player with persistent ID across frames using ByteTrack
- [ ] **CV-03**: System generates accumulated team-level heatmap from tracking data
- [ ] **CV-04**: System draws individual player trajectory paths on video overlay
- [ ] **CV-05**: System exports tracking data as CSV with columns: frame, player_id, x, y

### Web Dashboard

- [ ] **WEB-01**: User can upload match video and trigger CV processing pipeline
- [ ] **WEB-02**: Dashboard displays accumulated heatmap and player trajectories after processing
- [ ] **WEB-03**: User can download processed results (heatmap, trajectories, raw CSV data)

## v2 Requirements

Deferred to future release. Tracked but not in current roadmap.

### Advanced CV

- **CV-06**: Camera motion compensation for broadcast footage with pan/tilt/zoom
- **CV-07**: Top-down pitch homography (camera-space to real pitch coordinates)
- **CV-08**: Per-team heatmaps based on jersey color classification (KMeans)
- **CV-09**: Time-range filtering to analyze specific match periods

### Dashboard

- **WEB-04**: Side-by-side comparison of two matches' heatmaps
- **WEB-05**: Interactive timeline scrubber with live overlay updates
- **WEB-06**: Player speed and distance metrics display

### Real-Time

- **RT-01**: Live camera feed processing at 2-5fps
- **RT-02**: Live heatmap updates during streaming

### Tactical Analysis

- **TAC-01**: Formation detection (4-4-2, 4-3-3, etc.) from aggregated positions
- **TAC-02**: Zone occupancy percentages and pitch control statistics

## Out of Scope

| Feature | Reason |
|---------|--------|
| Ball tracking | Adds complexity for minimal movement-analysis value; focus is player positions |
| Full match event annotation (goals, cards) | Requires action recognition model; out of scope per project vision |
| Real-time at 30fps | Unnecessary for heatmap (aggregates over time); 2-5fps sufficient |
| Automatic jersey number OCR | Fragile on small crops; low value for heatmap accuracy |
| Mobile app | Pitch visualization poor on small screens; responsive web sufficient |
| User authentication / multi-tenant | Single-user mode for v1; add if needed |

## Traceability

| Requirement | Phase | Status |
|-------------|-------|--------|
| CV-01 | Phase 1: Core Detection & Tracking Pipeline | Pending |
| CV-02 | Phase 1: Core Detection & Tracking Pipeline | Pending |
| CV-03 | Phase 2: Heatmap & Trajectory Generation | Pending |
| CV-04 | Phase 2: Heatmap & Trajectory Generation | Pending |
| CV-05 | Phase 1: Core Detection & Tracking Pipeline | Pending |
| WEB-01 | Phase 3: Web Dashboard | Pending |
| WEB-02 | Phase 3: Web Dashboard | Pending |
| WEB-03 | Phase 3: Web Dashboard | Pending |

**Coverage:**
- v1 requirements: 8 total
- Mapped to phases: 8
- Unmapped: 0 ✓

---
*Requirements defined: 2026-05-23*
*Last updated: 2026-05-23 after initial definition*
