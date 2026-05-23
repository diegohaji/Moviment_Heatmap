# Domain Pitfalls

**Domain:** Computer vision football movement analysis
**Researched:** 2026-05-23

---

## Critical Pitfalls

Mistakes that cause rewrites or major issues.

### Pitfall 1: Using segmentation_models_pytorch for Player Detection
**Severity:** CRITICAL — architectural mistake
**What goes wrong:** 50-100x slower inference than YOLO11. A 90-minute match at 30fps (162,000 frames) takes 2-6 hours instead of 5-15 minutes. The heatmap position accuracy gain from segmentation masks over bounding boxes is negligible (~1-2 pixel at pitch scale). The project will feel "too slow to use" and cannot support real-time.
**Why it happens:** segmentation_models_pytorch is well-known in academic CV circles. The original proposal used it. It appears to be the "correct" choice for a movement heatmap because it provides pixel-level masks.
**Consequences:** Unusable processing times; cannot iterate on parameters; users give up waiting. Real-time mode becomes impossible.
**Detection:** Time a single frame through SMP vs YOLO11. If >50ms per frame, you have this problem.
**Prevention:** Start with YOLO11. Only introduce SMP if you specifically need pitch line segmentation (a separate task).

### Pitfall 2: Using Deep SORT for Tracking
**Severity:** CRITICAL — technical debt
**What goes wrong:** Deep SORT requires a separate appearance embedding model (typically ResNet-50) to re-identify players. In football where all players on the same team wear near-identical kits, appearance embeddings provide no discriminative power. Worse, Deep SORT's architecture struggles with occlusion recovery in crowded scenes (penalty boxes, midfield clusters).
**Why it happens:** Deep SORT (2017) was the canonical "modern" tracker for years. Many 2020-2021 tutorials recommend it. The ecosystem moved to ByteTrack (2022) and BoT-SORT (2023).
**Consequences:** Frequent ID switches in crowded scenes; trajectories for individual players are unreliable; tactical formation analysis is impossible because player 7 becomes player 12 mid-play.
**Detection:** Count ID switches per minute in a match segment. ByteTrack achieves <5 switches/match on SportsMOT; Deep SORT typically 20-50+.
**Prevention:** Use ByteTrack (via supervision) as default. Use BoT-SORT only for broadcast footage with camera motion.

### Pitfall 3: Processing Full Video Synchronously in Web Request
**Severity:** CRITICAL — UX failure
**What goes wrong:** 90-minute match = 162,000 frames. At 5ms/frame (YOLO11n on GPU) = 13.5 minutes. The HTTP request times out at 30-60 seconds. The browser shows "connection lost." User retries. Server spawns another process. Eventually OOM.
**Why it happens:** Natural to write `def upload(file): result = process(file); return result` — works for small files. Fails catastrophically at match scale.
**Consequences:** Feature appears broken. Non-technical users cannot diagnose. Project abandoned as "doesn't work."
**Prevention:** Always dispatch to Celery from the upload endpoint. Return task_id immediately. Frontend polls for completion.

### Pitfall 4: No Homography / Camera-Space Heatmap Only
**Severity:** HIGH — diminished value
**What goes wrong:** The heatmap is drawn in camera perspective. At the start of the match (wide shot), players in the center of the pitch appear in the middle of the frame. After a zoom, the same pitch area covers a different pixel region. The raw heatmap is a visualization of "where the camera pointed" not "where the players were."
**Why it happens:** Top-down homography requires pitch line detection + perspective transform matrix calculation. This is a separate computer vision task.
**Consequences:** The heatmap changes when camera angle changes (even for the same players on the same pitch). Researchers cannot compare heatmaps across matches filmed from different angles.
**Detection:** Two identical plays from different camera angles produce different heatmaps.
**Prevention:** Phase 2 priority — implement pitch homography as soon as the basic pipeline works. Use a separate model (or OpenCV manual calibration) to compute the perspective transform to a standard 105x68m pitch template.

---

## Moderate Pitfalls

### Pitfall 5: No Team Assignment / Single Heatmap Blends Both Teams
**What goes wrong:** The accumulated heatmap shows both teams combined. In a 4-4-2 vs 4-3-3 match, both formations blur into one "blob." The entire analytical value of the heatmap (team shape and spacing) is lost.
**Prevention:** Add team classification: crop each detection, run KMeans on HSV histogram, assign to home/away/referee. Implement before Phase 2 heatmap generation is finalized (changing output format later is costly).

### Pitfall 6: Using Raw Bbox Center Instead of Foot Position
**What goes wrong:** Player bounding boxes' geometric center maps to approximately chest height. For pitch mapping, the foot position (center-bottom of bbox) is the correct contact point with the ground. Using bbox center shifts all positions upward by ~1m, causing systematic inaccuracy in heatmap positioning.
**Prevention:** Small code fix: `foot_position = (bbox_x_center, bbox_y_bottom)` instead of `center = (bbox_x_center, bbox_y_center)`. Wrap this in a utility function from day one.

### Pitfall 7: Processing at Raw Video Framerate (30fps)
**What goes wrong:** 30fps → 162,000 frames for 90 minutes. Most frames are redundant — players move ~0.5m between frames at 30fps. The tracking output data is unnecessarily large (potentially millions of rows).
**Prevention:** Process at 2-5fps. This is the standard in sports CV research. At 2fps, you get 10,800 frames for 90 minutes — still captures all player movements (players don't teleport 5m in 0.5s). Reduces processing time by 6-15x and data size proportionally.

### Pitfall 8: Training a Custom Detection Model from Scratch
**What goes wrong:** YOLO11 pre-trained on COCO already detects "person" at ~0.85 mAP. A custom model trained on 1,000 football images will perform worse. Football-specific fine-tuning can improve from ~0.85 to ~0.92 mAP, but training from scratch with small datasets will regress.
**Prevention:** Use COCO-pretrained YOLO11 for v1. Only fine-tune on football data if you have >5,000 labeled images. Fine-tune from the pretrained checkpoint, never from scratch.

---

## Minor Pitfalls

### Pitfall 9: No Progress Indication During Processing
**What goes wrong:** User uploads a match. Nothing happens for 10 minutes. User assumes "it broke" and refreshes. Cancels processing. Retries. Repeat.
**Prevention:** Celery task reports progress: `self.update_state(state='PROCESSING', meta={'progress': frame_number / total_frames})`. Frontend shows progress bar from first poll.

### Pitfall 10: Forgetting to Handle Match Halftime / Black Frames
**What goes wrong:** Halftime segment contains no players (or ads/crowd). Black frames at match start/end. These introduce noise into the heatmap (people detected in stands, or false positives on dark frames).
**Prevention:** Add pre-processing: detect black frames (mean pixel brightness < threshold), skip them. Add a "crowd filter" (detections above a certain y-threshold are likely stands, not pitch).

### Pitfall 11: Not Normalizing Heatmap for Time
**What goes wrong:** A 5-minute segment with fast play produces a low heatmap value. A 5-minute segment with slow buildup play produces the same. Dwell-time heatmaps should normalize by total tracked player-seconds in each region.
**Prevention:** Divide raw heatmap counts by total observation time per cell. This gives "probability of a player being in this location" which is comparable across segments of different lengths.

### Pitfall 12: Using OpenCV's Default Video Codecs for Output
**What goes wrong:** OpenCV defaults to DIVX or MJPG codecs, producing large uncompressed output videos. A 90-minute annotated video can be 50-100GB.
**Prevention:** Use H.264 codec: `cv2.VideoWriter_fourcc(*'avc1')` (macOS) or `*'mp4v'` (cross-platform). Or skip annotated video entirely and only generate the static heatmap + tracking CSV.

---

## Phase-Specific Warnings

| Phase Topic | Likely Pitfall | Mitigation |
|---|---|---|
| Phase 1: Detection + Tracking | Using SMP → too slow | Use YOLO11 |
| Phase 1: Detection + Tracking | Using Deep SORT → ID switches | Use ByteTrack |
| Phase 1: Detection + Tracking | Using bbox center → foot position error | Use bbox bottom-center |
| Phase 1: Detection + Tracking | Processing at 30fps → slow + bloated | Downsample to 2-5fps |
| Phase 2: Heatmap | No homography → camera-space only | Add perspective transform in Phase 2.5 |
| Phase 2: Heatmap | Both teams blended → useless | Add team classification |
| Phase 2: Heatmap | Not time-normalized → misleading | Normalize by observation time |
| Phase 3: Web Dashboard | Sync processing → timeout | Celery from day one |
| Phase 3: Web Dashboard | No progress bar → bad UX | Celery progress reporting |
| Phase 3: Web Dashboard | Large output video → storage issues | Offer heatmap PNG + CSV only by default |
| Phase 4: Real-time | 30fps inference → impossible | Target 2-5fps; use YOLO11n (smallest) |
| Phase 5: Formation | Overfitting heuristics to one match | Validate on match diversity |

---

## Sources

- [Ultralytics YOLO11 tracking docs](https://context7.com/degirum/ultralytics_yolov8/llms.txt) — confirms built-in ByteTrack/BoT-SORT
- [Roboflow Trackers sports dataset tuned configs](https://github.com/roboflow/trackers/blob/develop/docs/trackers/botsort.md) — confirms SoccerNet/ SportsMOT parameter pain points
- [ByteTrack vs Deep SORT benchmarks](https://github.com/roboflow/trackers) — through library benchmarks
- [Supervision HeatMap example](https://github.com/roboflow/supervision/blob/develop/examples/heatmap_and_track/README.md) — reference implementation
- [Context7 SMP docs](https://context7.com/qubvel-org/segmentation_models.pytorch/llms.txt) — confirms segmentation-only, no tracking integration
- [Deep SORT Context7](https://context7.com/nwojke/deep_sort/llms.txt) — confirms unmaintained status
- Domain expertise: Sports CV research standard practices (2-5fps sampling, foot-position convention, time-normalized heatmaps)
