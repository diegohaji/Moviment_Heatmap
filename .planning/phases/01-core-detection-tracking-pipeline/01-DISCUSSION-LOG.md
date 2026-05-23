# Phase 1: Core Detection & Tracking Pipeline - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-05-23
**Phase:** 1-Core Detection & Tracking Pipeline
**Areas discussed:** YOLO Model Size, Detection Scope, Frame Processing Rate

---

## YOLO Model Size

| Option | Description | Selected |
|--------|-------------|----------|
| Medium (Recommended) | Best accuracy/speed balance. ~2-3ms per frame. Sufficient for football detection. | ✓ |
| Small | Faster (~1-2ms), slightly less accuracy. Good if processing 90-min matches frequently. | |
| Large | Most accurate (~4-5ms). Overkill for this use case but catches edge cases better. | |
| You decide | Let the agent choose based on benchmarks | |

**User's choice:** Medium
**Notes:** User has mid-range GPU (RTX 3060/4060). Medium is the best fit.

---

## Detection Scope

| Option | Description | Selected |
|--------|-------------|----------|
| All players + referees | 22 players + 3-4 referees. Standard approach for full tactical analysis. | |
| Players only | 22 players. Skip referees — simplifies downstream analysis. | ✓ |
| Players + refs + ball | Most comprehensive but ball tracking is out of scope per requirements. | |

**User's choice:** Players only
**Notes:** Also chose football fine-tuned YOLO11 (SportsMOT/SoccerNet) rather than COCO pre-trained.

---

## Frame Processing Rate

| Option | Description | Selected |
|--------|-------------|----------|
| Every 3rd frame (10fps effective) | ~54K frames for 90min. Good balance — research shows 10fps sufficient for heatmap accuracy. | |
| Every 2nd frame (15fps) | ~81K frames. Safer for tracking but 50% more processing time. | |
| Every frame (30fps) | ~162K frames. Only if you need frame-perfect tracking for analysis. | |
| You decide | Let the agent benchmark the best tradeoff | ✓ |

**User's choice:** You decide
**Notes:** Agent to determine optimal frame skipping strategy during planning.

---

## the agent's Discretion

- **Frame processing rate:** User deferred to agent. Benchmark every-2nd, every-3rd, and every-6th frame skipping.
- **ByteTrack parameters:** Agent to tune for football occlusion patterns.
- **CSV output format:** Agent to define column schema.

## Deferred Ideas

None.
