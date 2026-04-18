# `nav.py` README

This document explains what `nav.py` does, how obstacle processing works, and what the depth values mean.

## What `nav.py` does

`nav.py` runs Depth Anything V2 on images and can optionally overlay a simple navigation policy.

- Input: RGB image(s)
- Intermediate: predicted depth map
- Output (with `--nav`): one action per frame (`LEFT`, `RIGHT`, `FORWARD`, `STOP`) plus visualization panels

Without `--nav`, it behaves like a standard depth-visualization script.

## Obstacle processing pipeline

When `--nav` is enabled, the logic is:

1. **Depth normalization**
   - Raw model output `depth` is min-max normalized per image:
   - `depth_norm = (depth - depth.min()) / (depth.max() - depth.min())`
   - So each frame has values in `[0, 1]`.

2. **Binary obstacle mask**
   - Threshold normalized depth with `CLOSE_THRESHOLD` (default `0.4`):
   - `obstacle_mask = (depth_norm > CLOSE_THRESHOLD)`
   - Mask value `1` means "obstacle/close", `0` means "free".

3. **Cost map from mixed (hard + continuous) depth**
   - Start with `cost = depth_norm` (continuous values in `[0,1]`).
   - Hard-mask close pixels to max cost:
   - `cost[depth_norm > CLOSE_THRESHOLD] = 1.0`
   - Smooth this map with a `31x31` elliptical kernel via `cv2.filter2D`.
   - Result: close obstacles are strongly penalized, while non-obstacle regions remain continuous (not binary).

4. **Three-region scoring**
   - The frame is split into three vertical regions:
     - left third
     - center/forward third
     - right third
   - Region cost is the mean cost-map value inside each third.

5. **State-machine action**
   - If all three region costs exceed `NOGO_THRESHOLD` (default `0.6`), action is `STOP`.
   - Otherwise choose the region with the minimum cost:
     - min in left -> `LEFT`
     - min in center -> `FORWARD`
     - min in right -> `RIGHT`

## Does depth have physical units?

Short answer: **not in `nav.py`**.

- `nav.py` uses the base Depth Anything V2 model (`depth_anything_v2/dpt.py`) for **relative depth**.
- Relative depth is not guaranteed to be in meters (or any absolute physical unit).
- Then `nav.py` applies per-image normalization to `[0,1]`, which further removes any absolute scale.

So thresholds like `0.4` and `0.6` are **unitless normalized thresholds**, not metric distances.

## Practical implication

Because depth is relative + normalized per image:

- Threshold direction depends on your model output convention. In this script, larger normalized values are treated as "closer" (`depth_norm > CLOSE_THRESHOLD`).
- Thresholds are scene-dependent and camera-dependent.
- You should tune `--close-thresh` and `--nogo-thresh` empirically for your setup.
- If you need true metric distance (meters), use the metric-depth pipeline in `metric_depth/` rather than `nav.py`.

## Example usage

```bash
python3 nav.py \
  --encoder vitl \
  --img-path assets/examples \
  --outdir vis_depth_nav \
  --nav \
  --close-thresh 0.4 \
  --nogo-thresh 0.6
```

