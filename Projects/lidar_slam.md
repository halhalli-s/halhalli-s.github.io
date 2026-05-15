---
layout: default
title: LiDAR Sensor Modeling & Point Cloud SLAM
---

# LiDAR Sensor Modeling & Point Cloud SLAM

**CS5335 Robotics Science and Systems · Northeastern University · Sep 2025 – Dec 2025**  
**Stack:** Python · NumPy · GTSAM · ICP · scikit-learn · Velodyne HDL-64E · Argo AI Dataset

---

## Overview

This project covers two connected problems in mobile robotics: building a 2D LiDAR sensor model from the ground up, and using real Velodyne point cloud data to estimate a vehicle's trajectory through scan matching and pose graph optimization.

The sensor modeling work focused on understanding how real LiDAR sensors fail — not just adding Gaussian noise, but modeling the four physically distinct noise sources that affect range measurements in practice. The SLAM work used 180 real Velodyne scans from an Argo AI autonomous vehicle, implementing voxel grid downsampling, ICP scan alignment, and GTSAM-based trajectory estimation from scratch.

---

## Part 1 — 2D LiDAR Sensor Model

### Ideal Ray Tracing

The starting point was a clean 2D LiDAR simulator using ray tracing on an occupancy grid. For each beam angle, the sensor marches along the ray at a fixed resolution until it hits an occupied cell or reaches max range, returning the measured distance. Configurable parameters: beam count, angular limits, range limits, range resolution.

Integrated with the `gym-neu-racing` mobile robot simulator and validated against ground truth state — the sensor output matched expected geometry across a range of robot poses.

### Gaussian Noise

Extended to include zero-mean Gaussian noise on range readings. This is the standard simplification used in many robotics textbooks — useful as a baseline, but not representative of how real LiDAR sensors actually fail.

### Realistic 4-Component Noise Model

The more interesting extension was implementing the beam model from Thrun's *Probabilistic Robotics* — a weighted mixture of four physically motivated noise sources:

| Component | Physical Source |
|---|---|
| **p_hit** (Gaussian) | Correct measurement with sensor noise |
| **p_short** (exponential decay) | Short readings from dynamic obstacles, other robots |
| **p_max** (spike at max range) | Missed detections — glass, fog, specular surfaces |
| **p_rand** (uniform) | Random electronic interference, crosstalk |

The final range measurement is sampled from this mixture. The weights (w_hit, w_short, w_max, w_rand) must sum to 1.0 and are tunable per sensor.

The difference from simple Gaussian noise is significant in practice. The heavy tails from p_rand and p_max produce outliers that a downstream localization algorithm has to handle robustly — a Gaussian model would never generate them, so a filter designed against Gaussian noise can fail badly on real sensor data.

---

## Part 2 — Point Cloud SLAM on Real Velodyne Data

### Dataset

180 Velodyne HDL-64E point cloud scans (~88,000 points/scan) from Argo AI's autonomous driving dataset, captured in Miami over 18 seconds. The vehicle starts stationary at a T-intersection, waits for traffic, then navigates through the intersection. Real urban driving data — moving vehicles, pedestrians, varying point density.

### Downsampling

88,000 points per scan is too dense for practical scan matching. Before running ICP, each scan is downsampled using a voxel grid implemented from scratch — no Open3D or external point cloud libraries.

The idea: divide 3D space into cubes of a fixed leaf size, and replace all points in each occupied cube with their centroid. At leaf_size=0.5m, this reduces 88k points to roughly 3–5k while preserving the geometry of walls, vehicles, and road surfaces. Tested across leaf sizes from 0.1m to 2.0m to understand the tradeoff between density and shape preservation.

### Scan Matching with ICP

With downsampled scans, consecutive frames are aligned using Iterative Closest Point — implemented fully from scratch using NumPy and GTSAM.

The three core components:

**Point cloud transformation** — Given a gtsam.Pose3, apply it to each point using homogeneous coordinates. Vectorized entirely with NumPy matrix operations — no loops over individual points.

**Closest pair assignment** — For each point in the source cloud, find its nearest neighbor in the target cloud using a KD-Tree (sklearn.NearestNeighbors). A distance threshold rejects pairs that are too far apart — important for robustness in early iterations when the clouds are still significantly misaligned.

**Transform estimation** — Given the matched pairs, estimate the rigid transform (rotation R, translation t) that minimizes the sum of squared distances between matched points. Uses SVD of the cross-covariance matrix of the centered point clouds — the standard closed-form solution for point-to-point ICP.

The ICP loop iterates: transform → match → estimate → repeat, until convergence or max iterations. Validated first on synthetic triangle point clouds before running on the Velodyne data.

### Trajectory Estimation with GTSAM

ICP gives a relative transform between each pair of consecutive scans. Chaining these gives an odometry estimate — but errors accumulate. Over 180 scans, small per-frame errors compound into significant drift.

GTSAM pose graph optimization distributes this error globally:

- Each scan contributes a **node** (robot pose, gtsam.Pose3)
- Each ICP result contributes an **edge** (odometry factor between consecutive poses)
- A **prior factor** anchors the first pose
- **Levenberg-Marquardt** solves the resulting nonlinear least squares problem

The optimized trajectory recovers the vehicle's path through the Miami intersection. The pose graph backend is what separates LiDAR odometry (just chaining ICP) from SLAM — it provides a principled framework for incorporating loop closures as additional graph edges if they're available.

---

## What Was Hard

**Nearest neighbor efficiency** — A naive double loop over 88k points is O(n²) and completely intractable. Getting to O(n log n) with a KD-Tree was the first practical requirement before anything else could work at real scale.

**Distance threshold tuning for ICP** — Too tight and ICP rejects valid matches during early iterations when clouds are far apart. Too loose and spurious matches corrupt the transform estimate. The right threshold depends on how much motion occurred between scans, which varies with vehicle speed.

**Coordinate frame conventions** — GTSAM uses a specific right-hand, z-up Pose3 convention. Getting frame conventions consistent across scan loading, ICP output, and GTSAM factor construction required careful attention — a sign error anywhere silently produces wrong trajectories.

---

## What I Learned

A realistic sensor model is more than noise on top of a perfect measurement. The 4-component beam model captures physically distinct failure modes — dynamic obstacles, max-range misses, interference — that have very different statistical signatures. Designing localization algorithms that are robust in practice means understanding what the sensor is actually doing, not just assuming Gaussian error.

ICP is sensitive to initialization. On well-aligned consecutive scans it converges reliably. On poorly initialized scans it gets stuck in local minima. Real SLAM systems need either good initial alignment, outlier rejection, or both.

The frontend/backend separation in SLAM (ICP for local matching, pose graph for global consistency) is a clean architectural pattern. The frontend can be swapped out — better scan matching improves trajectory accuracy directly — and loop closures drop in naturally as additional graph edges without changing the backend structure.
