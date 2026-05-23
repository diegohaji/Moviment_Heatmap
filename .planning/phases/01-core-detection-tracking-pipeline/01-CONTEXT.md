# Phase 1: Core Detection & Tracking Pipeline - Context

**Gathered:** 2026-05-23
**Status:** Ready for planning

<domain>
## Phase Boundary

Deliver a CLI-based computer vision pipeline that takes football match video input, detects players using YOLO11, tracks each player with persistent IDs using ByteTrack, and exports structured tracking data as CSV. This is the foundation phase — Phase 2 (heatmap/trajectory) and Phase 3 (web dashboard) build on this pipeline's output.

Requirements in scope: CV-01 (YOLO11 detection), CV-02 (ByteTrack tracking), CV-05 (CSV export). Ball tracking, real-time processing, camera motion compensation, and team classification are out of scope for this phase.
</domain>

<decisions>
## Implementation Decisions

### Detection Model
- **D-01:** Use YOLO11 medium (m) — best accuracy/speed balance for mid-range GPU
- **D-02:** Use football fine-tuned weights (SportsMOT/SoccerNet) rather than COCO pre-trained — improves detection accuracy for football-specific scenarios and reduces false positives from fans/coaches
- **D-03:** Detect players only (22 outfield players). Referees and ball excluded from detection scope.

### Hardware
- **D-04:** Target mid-range GPU (RTX 3060/4060-class). Pipeline must work within this constraint.

### the agent's Discretion
- **Frame processing rate:** Agent to decide best frame skipping strategy (research suggests every 3rd frame = 10fps effective is optimal for heatmap quality vs processing time). Benchmark on test footage and document the tradeoff.
- **ByteTrack parameters:** Agent to tune detection threshold, match threshold, and track buffer for football-specific occlusion patterns using research recommendations.
- **CSV output format:** Agent to define column schema (frame, player_id, x, y, confidence, bbox coords).

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Project Definition
- `.planning/ROADMAP.md` §Phase 1 — Phase goal, success criteria, mode
- `.planning/REQUIREMENTS.md` — CV-01, CV-02, CV-05 requirement definitions
- `.planning/PROJECT.md` — Core value, context, constraints

### Research Findings
- `.planning/research/STACK.md` — YOLO11 + ByteTrack stack recommendations, versions, installation
- `.planning/research/PITFALLS.md` — Pitfalls relevant to Phase 1: occlusion handling, frame processing bottlenecks, ID switches
- `.planning/research/ARCHITECTURE.md` — Pipeline architecture, component boundaries, data flow
- `.planning/research/SUMMARY.md` — Executive summary, phase ordering rationale

### No External Specs
Requirements fully captured in decisions above and in REQUIREMENTS.md.
</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
No existing assets — greenfield project. All Phase 1 code will be written from scratch.

### Integration Points
- Pipeline output (CSV) is the integration contract with Phase 2.
- CLI entry point should support both single video file and batch processing modes.

</code_context>

<specifics>
## Specific Ideas

- Output CSV should include sufficient metadata for Phase 2 to identify tracking quality (confidence scores, detection counts per frame).
- Consider using supervision library's `sv.Detections` as the intermediate data format throughout the pipeline — it's the standard bridge between YOLO, ByteTrack, and export.
</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.
</deferred>

---

*Phase: 1-Core Detection & Tracking Pipeline*
*Context gathered: 2026-05-23*
