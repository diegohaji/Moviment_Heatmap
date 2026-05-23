# Moviment Heatmap

## What This Is

A computer vision web application that processes football match footage to generate accumulated movement heatmaps and individual player trajectories. Built for sports science researchers and tactical analysts who need visual analytics of player positioning and movement patterns.

## Core Value

Transform raw football match footage into actionable visual analytics — accumulated heatmaps and per-player trajectories — that reveal tactical patterns invisible to the naked eye.

## Requirements

### Validated

(None yet — ship to validate)

### Active

- [ ] Player detection and tracking from video footage
- [ ] Accumulated heatmap generation (team-level occupancy)
- [ ] Individual player trajectory drawing
- [ ] Web dashboard with video upload and result visualization
- [ ] Support for both recorded video upload and real-time camera processing
- [ ] Detection and visualization of tactical formations (4-4-2, 4-3-3, etc.)
- [ ] Export of movement data (coordinates, CSV/JSON)

### Out of Scope

- Full match event annotation (goals, fouls, cards) — focus is movement, not events
- Ball tracking — player movement is the primary object of study

## Context

- Academic research project focused on sports tactics analysis
- Targets both university researchers and professional club analysts
- Football (soccer) as the primary sport domain
- Player-level individual tracking required (not just team-level heat)
- Stakeholders need both visual output and raw data export for their own analyses
- Proposed stack: segmentation_models_pytorch, OpenCV, Deep SORT — but open to research-informed choices

## Constraints

- **Tech Stack**: Python-based for CV pipeline; web frontend for dashboard
- **Performance**: Must handle full 90-minute match footage within reasonable time
- **Data**: Needs both capture pipeline (camera input) and processing pipeline (video files)

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Web app format | Enables non-technical researchers/analysts to use the tool | — Pending |
| Football-first domain | Focused scope for v1, expandable later | — Pending |
| Individual tracking | Required for tactical formation analysis | — Pending |
| Stack open to research | Suggestions made but needs domain-informed validation | — Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd-transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd-complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-05-23 after initialization*
