# Roadmap: Moviment Heatmap

## Overview

Moviment Heatmap transforms football match footage into actionable visual analytics. Phase 1 establishes the core computer vision pipeline — detecting players with YOLO11, tracking them with ByteTrack, and exporting structured tracking data for analysis. Phase 2 adds visual output generation: accumulated team-level heatmaps and individual player trajectory paths. Phase 3 wraps everything in a web dashboard with video upload, result visualization, and data download — delivering a complete end-to-end tool usable by sports science researchers and tactical analysts.

## Phases

- [ ] **Phase 1: Core Detection & Tracking Pipeline** - Detect and track players from match video, outputting structured tracking data with persistent IDs
- [ ] **Phase 2: Heatmap & Trajectory Generation** - Generate team-level heatmaps and individual player trajectory visualizations from tracking data
- [ ] **Phase 3: Web Dashboard** - Upload video through a web UI, trigger processing, view heatmaps/trajectories, and download results

## Phase Details

### Phase 1: Core Detection & Tracking Pipeline
**Goal**: System detects and tracks all players in match footage, outputting structured tracking data
**Mode**: mvp
**Depends on**: Nothing (first phase)
**Requirements**: CV-01, CV-02, CV-05
**Success Criteria** (what must be TRUE):
  1. User can run match footage through the detection pipeline and receive per-frame player bounding boxes
  2. Each detected player maintains a persistent ID that follows them across the full match duration with minimal ID switches
  3. User can export the complete tracking data as a CSV file with columns: frame, player_id, x, y
**Plans**: TBD

### Phase 2: Heatmap & Trajectory Generation
**Goal**: System generates accumulated team-level heatmaps and individual player trajectory visualizations from tracking data
**Mode**: mvp
**Depends on**: Phase 1
**Requirements**: CV-03, CV-04
**Success Criteria** (what must be TRUE):
  1. User can generate a color-coded heatmap image showing team-level spatial occupancy from any processed tracking dataset
  2. User can render individual player trajectory paths as an overlay on the video or pitch template
  3. Both heatmap and trajectory outputs are saved as downloadable image/video files ready for use in reports or presentations
**Plans**: TBD

### Phase 3: Web Dashboard
**Goal**: User can upload match video through a web browser, trigger CV processing, view heatmaps/trajectories, and download all results
**Mode**: mvp
**Depends on**: Phase 2
**Requirements**: WEB-01, WEB-02, WEB-03
**Success Criteria** (what must be TRUE):
  1. User can upload a video file through the browser and trigger CV processing with visible progress feedback
  2. Dashboard displays the accumulated team-level heatmap and individual trajectory paths after processing completes
  3. User can download heatmap image, trajectory overlay, and raw tracking data CSV from the dashboard
**Plans**: TBD
**UI hint**: yes

## Progress

**Execution Order:**
Phases execute in numeric order: 1 → 2 → 3

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Core Detection & Tracking Pipeline | 0/0 | Not started | - |
| 2. Heatmap & Trajectory Generation | 0/0 | Not started | - |
| 3. Web Dashboard | 0/0 | Not started | - |
