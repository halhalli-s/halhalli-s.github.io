---
layout: default
title: LiDAR Sensor Modeling & Point Cloud SLAM
---

# LiDAR Sensor Modeling & Point Cloud SLAM

**CS5335 Robotics Science and Systems · Northeastern University · Sep 2025 – Dec 2025**  
**Stack:** Python · NumPy · GTSAM · ICP · scikit-learn · Velodyne HDL-64E · Argo AI Dataset

---

## What This Is

Two connected projects: building a 2D LiDAR sensor model from scratch (including realistic noise), then applying it to real-world Velodyne point clouds from an autonomous vehicle — implementing voxel grid downsampling, ICP scan matching, and GTSAM pose graph optimization to estimate the vehicle trajectory.

Everything implemented from scratch. No Open3D or external point cloud libraries for the core algorithms.

---

## Part 1 — 2D LiDAR Sensor Model

### Ideal Ray Tracing Model

Implemented a 2D LiDAR simulator using ray tracing on a 2D occupancy grid map. For each beam, march along the ray at a fixed resolution until hitting an occupied cell or reaching max range. Parameters: beam count, angular limits, range limits, range resolution.

```python
class Lidar2D(SensorModel):
    # num_beams, range_resolution, range_limits, angle_limits
    # For each beam angle: march ray, return range at first occupied hit
```

Integrated with the `gym-neu-racing` mobile robot simulator and validated against ground truth state.

### Gaussian Noise Model

Extended to `Lidar2DGaussian` — adds zero-mean Gaussian noise to range readings with configurable variance. Models sensor measurement uncertainty.

### Realistic 4-Component Noise Model

Implemented `Lidar2DRealistic` using the beam model from Thrun's Probabilistic Robotics — a weighted mixture of four noise sources:

| Component | Models |
|---|---|
| **p_hit** (Gaussian) | Correct measurement with sensor noise |
| **p_short** (exponential decay) | Short readings from dynamic obstacles |
| **p_max** (uniform spike at max range) | Missed detections, glass, fog |
| **p_rand** (uniform) | Random electronic interference |

The final range measurement is sampled from this mixture PDF. Parameters w_hit, w_short, w_max, w_rand must sum to 1.0.

This produces qualitatively different behavior from simple Gaussian noise — the heavy tail from p_rand and p_max creates outliers that a downstream localization algorithm must handle robustly.

---

## Part 2 — Point Cloud SLAM on Velodyne Data

### Dataset

180 Velodyne HDL-64E point cloud scans (~88,000 points/scan) from Argo AI's autonomous driving dataset, captured in Miami over 18 seconds. The vehicle starts stationary at a T-intersection, then navigates through urban traffic.

### Task 0 — Voxel Grid Downsampling

88k points per scan is too dense for real-time ICP. Implemented voxel grid downsampling from scratch:

1. Divide 3D space into voxels of configurable leaf size
2. For each occupied voxel, replace all points with the centroid
3. Discard remaining points

```python
def voxel_grid_downsampling(cloud, leaf_size=0.1):
    voxel_indices = np.floor(cloud / leaf_size).astype(int)
    # group by voxel index, take centroid of each group
```

At leaf_size=0.5m, reduces 88k points to ~3-5k while preserving geometry. Tested across leaf sizes 0.1–2.0m and verified shape preservation visually.

### Task 1 — ICP from Scratch

Implemented all ICP components:

**transform_cloud** — Apply a gtsam.Pose3 transform to each point using homogeneous coordinates. Vectorized with NumPy (no loops):
```python
# Convert to homogeneous, apply 4x4 transform matrix, return 3D
cloud_h = np.vstack([cloud, np.ones(n)])
transformed = (pose.matrix() @ cloud_h)[:3]
```

**assign_closest_pairs** — KD-Tree nearest neighbor matching using sklearn.NearestNeighbors. Distance threshold filters outlier matches. Returns only pairs within threshold — critical for robustness since early ICP iterations have large misalignment.

**estimate_transform** — SVD-based rigid transform estimation. Given matched point pairs, compute the rotation R and translation t that minimize Σ‖R·aᵢ + t − bᵢ‖². Uses centered point clouds and SVD of the cross-covariance matrix.

### Task 2 — ICP Loop

Full ICP: initialize transform → transform cloud → assign closest pairs → estimate transform → iterate until convergence or max iterations. Returns final Pose3 and intermediate states for animation.

Validated on synthetic triangle point clouds before running on Velodyne data.

### Task 3 — GTSAM Pose Graph SLAM

ICP gives pairwise transforms between consecutive scans, but errors accumulate over 180 frames. Used GTSAM pose graph optimization to distribute error globally:

- **Nodes:** Robot pose at each scan (gtsam.Pose3)
- **Edges:** ICP-estimated relative transforms between consecutive scans (odometry factors)
- **Prior:** Fixed prior on first pose to anchor the graph
- **Solver:** Levenberg-Marquardt via GTSAM

The optimized trajectory shows the vehicle path through the Miami intersection, with loop closure reducing drift accumulated over 180 scans.

---

## What Was Hard

**Vectorizing nearest neighbor matching** — A naive double for-loop over 88k points is O(n²) and completely intractable. The KD-Tree drops this to O(n log n) and runs in under a second per scan.

**Distance threshold tuning** — Too tight: ICP rejects valid matches during early iterations when clouds are misaligned. Too loose: includes spurious matches that corrupt the transform estimate. Required empirical tuning per dataset.

**Coordinate frame consistency** — GTSAM's Pose3 uses a specific convention (right-hand, z-up). Getting the frame convention right across scan loading, ICP output, and GTSAM factor construction took careful debugging.

---

## Key Takeaways

A real sensor model has structure beyond Gaussian noise — the 4-component beam model captures physically meaningful failure modes (dynamic obstacles, max-range misses, interference) that a Gaussian can't represent. Designing a localization algorithm that's robust to these requires understanding the actual noise distribution.

ICP is sensitive to initialization. On well-aligned consecutive scans (~10 frames apart) it converges reliably. On poorly initialized scans, it gets stuck in local minima. Robust SLAM requires either good initial alignment or outlier rejection.

Pose graph optimization is the right abstraction for trajectory estimation: it separates local scan matching (frontend) from global consistency (backend), and lets you add loop closures cleanly as additional graph edges.
