# Depth Navigation Project Plan

## Goal
- Reproduce Depth Anything V2 metric-depth performance for Small and Base models on NYU-Depth-v2 and KITTI.
- Implement and evaluate a depth-only navigation policy:
  - Input: depth map
  - Output: one of four states (`LEFT`, `RIGHT`, `FORWARD`, `STOP`)

## Timeline (Suggested)
- Week 1: Environment setup, checkpoints, dataset paths, smoke tests.
- Week 2: Reproduction runs (Small/Base on NYU + KITTI), collect baseline metrics.
- Week 3: Navigation state machine tuning and offline evaluation on recorded sequences.
- Week 4: Human-in-the-loop walkthroughs (25+ runs), aggregate metrics, final report.

## Reproduction Setup
- Install base repo dependencies:
  - `pip install -r requirements.txt`
- Install metric-depth dependencies:
  - `pip install -r metric_depth/requirements.txt`
- Place model checkpoints in `checkpoints/`:
  - Relative depth: `depth_anything_v2_vits.pth`, `depth_anything_v2_vitb.pth`
  - Metric depth (indoor/outdoor variants as needed):
    - `depth_anything_v2_metric_hypersim_vits.pth`
    - `depth_anything_v2_metric_hypersim_vitb.pth`
    - `depth_anything_v2_metric_vkitti_vits.pth`
    - `depth_anything_v2_metric_vkitti_vitb.pth`

## Reproduction Runs (Small + Base)
Use these as templates; replace dataset paths with your local NYU/KITTI inputs.

```bash
# Small encoder (vits) - indoor-like model
python metric_depth/run.py \
  --encoder vits \
  --load-from checkpoints/depth_anything_v2_metric_hypersim_vits.pth \
  --max-depth 20 \
  --img-path <NYU_OR_INDOOR_PATH> \
  --outdir outputs/nyu_vits \
  --save-numpy

# Base encoder (vitb) - indoor-like model
python metric_depth/run.py \
  --encoder vitb \
  --load-from checkpoints/depth_anything_v2_metric_hypersim_vitb.pth \
  --max-depth 20 \
  --img-path <NYU_OR_INDOOR_PATH> \
  --outdir outputs/nyu_vitb \
  --save-numpy

# Small encoder (vits) - outdoor-like model
python metric_depth/run.py \
  --encoder vits \
  --load-from checkpoints/depth_anything_v2_metric_vkitti_vits.pth \
  --max-depth 80 \
  --img-path <KITTI_OR_OUTDOOR_PATH> \
  --outdir outputs/kitti_vits \
  --save-numpy

# Base encoder (vitb) - outdoor-like model
python metric_depth/run.py \
  --encoder vitb \
  --load-from checkpoints/depth_anything_v2_metric_vkitti_vitb.pth \
  --max-depth 80 \
  --img-path <KITTI_OR_OUTDOOR_PATH> \
  --outdir outputs/kitti_vitb \
  --save-numpy
```

## Navigation Decision Script
Implemented in `nav.py`.

- Obstacle detection:
  - Threshold normalized depth at `d` into binary close-obstacle mask.
- Free-space estimate:
  - `free_space_ratio` over the drive-relevant lower frame region.
- Steering policy:
  - Split into L / C / R thirds.
  - Compute obstacle-density cost per region.
  - Choose minimum-cost direction.
  - `STOP` only if all three region costs exceed the no-go threshold.

Example run:

```bash
python nav.py \
  --encoder vitb \
  --img-path <IMAGE_OR_DIR> \
  --outdir vis_nav \
  --nav \
  --close-thresh 0.40 \
  --nogo-thresh 0.60 \
  --decision-csv outputs/nav_decisions.csv
```

## Human Walkthrough Test Protocol
- Operator wears chest-mounted camera and follows state-machine outputs.
- 25 walkthroughs, each 3 minutes.
- Environments:
  - Empty
  - Cluttered
- Record RGB + depth from sensor (e.g., iPad Pro LiDAR stream when available).
- Evaluate navigation with:
  - Sensor depth
  - DA-V2 predicted depth

## Metrics
- Steering accuracy:
  - Compare predicted action vs human judgement label.
- Collision-rate proxy:
  - Count action decisions that would lead to unsafe close-obstacle moves.
- 3-minute success rate:
  - Success = no stuck state for full sequence.
  - Stuck = decisions repeatedly force `STOP` / no progress.
- Research metrics:
  - Free-space IoU
  - Obstacle precision / recall

## Logging Template (Per Decision)
- `timestamp`
- `sequence_id`
- `frame_id`
- `depth_source` (`sensor` or `dav2`)
- `pred_action`
- `human_action`
- `left_cost`, `forward_cost`, `right_cost`
- `free_space_ratio`
- `collision_proxy` (0/1)

## Metrics Script
Use `evaluate_nav_metrics.py` to compute:
- Steering accuracy
- Collision-rate proxy
- 3-minute sequence success rate

Example:

```bash
python3 evaluate_nav_metrics.py \
  --decisions-csv outputs/nav_decisions.csv \
  --labels-csv outputs/human_labels.csv \
  --key-col filename \
  --human-action-col human_action \
  --collision-col collision_proxy \
  --sequence-col sequence_id \
  --json-out outputs/nav_metrics.json
```

