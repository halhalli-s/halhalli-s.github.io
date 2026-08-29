---
layout: default
title: HDVI-SLAM
---

# HDVI-SLAM: Handheld Dense Visual-Inertial SLAM

**Personal Project · Jul 2026 – Aug 2026**  
**Stack:** Python · GTSAM (iSAM2, IMU preintegration) · Open3D · ROS2 Jazzy · Orbbec Gemini 435Le · Allan Variance

---

<!-- ============================================================
     VIDEO PLACEHOLDER 1: HERO
     Rectangle walk WITH loop closure. Best-looking run.
     Replace with: ![Rectangle walk with loop closure enabled](../Assets/images/hdvi_rect_lc.gif)
     ============================================================ -->
> **[ VIDEO 1: Rectangle walk with loop closure ]**  
> *Live trajectory and dense colored map in RViz2, retro-corrected when the loop closes.*

---

## What This Is

A dense visual-inertial SLAM system built from scratch for a handheld RGBD camera. You walk around a scene holding an **Orbbec Gemini 435Le**, and the system produces two things in real time:

- a **live camera trajectory** (`nav_msgs/Path`), and
- a **dense, colored 3D point-cloud map** (`sensor_msgs/PointCloud2`),

both continuously re-derived from the current optimized estimate of a **GTSAM iSAM2 factor graph**, so when a loop closes, the entire trajectory and map retro-correct.

The Gemini 435Le has **no existing SLAM integration**. There was no reference implementation to adapt, no tutorial, and no known-good driver configuration. The camera driver, the IMU pipeline, the sensor calibration, and the estimation stack were all built and debugged from the SDK up.

This project was as much about **measurement discipline** as about the algorithm. Every result below was collected against a marked physical ground truth, on a fixed configuration, with a control arm.

---

## System Architecture

Strict layer separation is the load-bearing design invariant. The estimation core imports numpy, Open3D, and GTSAM, but **never** a sensor SDK and **never** ROS. All sensor data crosses a two-dataclass boundary (`CloudFrame`, `ImuSample`). The camera SDK is imported in exactly one file; ROS in exactly one other. The core is unit-testable with neither hardware nor a ROS installation present.

```
SDK callback thread
  IMU (accel + gyro)  --> preintegrator.integrate()          [202 Hz]
  Depth + color frame --> depth queue (maxsize=1, drop-on-full)

Depth worker thread
  depth_to_cloud (~120 ms)
    --> keyframe trigger (gravity-cancelled translation)
    --> backpressure gate
    --> ICP align (IMU rotation-only seed)
    --> iSAM2 update  (ICP BetweenFactor + ImuFactor + bias random walk)
    --> loop closure check (every Nth keyframe)

Map thread
  Transform all keyframe clouds by CURRENT optimized poses --> voxelize --> publish
```

**Factor graph per keyframe:** an ICP between-factor, an `ImuFactor` over the preintegrated interval, and a `BetweenFactorConstantBias` random-walk edge. Loop closures are applied incrementally with extra relinearization sweeps and a full solve.

**Sensor role separation** is deliberate and non-negotiable: **the IMU owns rotation, gravity direction, and bias observability; ICP owns position.** Velocity is in the graph so that bias stays observable, not to navigate with.

---

## Results

Five runs, single session, fixed configuration, room lights on. Ground truth established by returning the camera to a marked start position **and heading**. Re-seating repeatability is approximately ±1°.

### Loop closure on vs. off, 0.63 × 0.92 m rectangular path

| Run | Path length | Position error | Heading error | Closures accepted |
|---|---|---|---|---|
| Loop closure **off** | 2.99 m | **9.0 cm** | **6.82°** | 0 (gated) |
| Loop closure **on** (run 1) | 2.94 m | **1.7 cm** | **0.79°** | 3 @ fitness 0.929–0.930 |
| Loop closure **on** (run 2) | 2.92 m | **1.0 cm** | **1.89°** | 2 @ fitness 0.974–0.985 |

**Backend correction: 5 to 9× reduction in position error, 3.6 to 8.6× in heading.**

<!-- ============================================================
     VIDEO PLACEHOLDER 2: CONTROL ARM
     Rectangle walk WITHOUT loop closure. Show the drift.
     Replace with: ![Rectangle walk, loop closure disabled](../Assets/images/hdvi_rect_nolc.gif)
     ============================================================ -->
> **[ VIDEO 2: Same rectangle, loop closure gated off ]**  
> *The map does not snap back on revisit. Final pose lands 9.0 cm and 6.8° from the marked start.*

The control run is a **strict comparison, not an absence of data.** Loop-closure verification ran normally and *found* the same closures the enabled runs used, six candidates scoring 0.923 to 0.928, but the acceptance threshold was raised to 0.95 so none were applied. That is stronger evidence than disabling the module, because it demonstrates the closures were available and deliberately withheld.

**Scale verified independently:** walked perimeter 3.10 m vs. trajectory-integrated length 2.92 to 2.99 m, a 95% agreement. The 5% shortfall is straight-line summing between keyframes cutting the corners. No translation-scale error.

### 360° in-place rotation, ~0.9 m pivot path

| Run | Path length | Position error | Heading error | Closures accepted |
|---|---|---|---|---|
| Pivot 1 | 0.96 m | 0.81 cm | 1.26° | 3 @ fitness 0.976–0.989 |
| Pivot 2 | 0.87 m | 1.03 cm | 0.75° | 3 @ fitness 0.979–0.986 |

<!-- ============================================================
     VIDEO PLACEHOLDER 3: PIVOT
     Circle / in-place 360 rotation with LC.
     Replace with: ![360-degree in-place rotation with loop closure](../Assets/images/hdvi_circle_lc.gif)
     ============================================================ -->
> **[ VIDEO 3: 360° in-place rotation with loop closure ]**  
> *Isolates rotational accuracy with minimal translation. Both runs close to ~1 cm and within ~1°.*

### Measurement method

Position error is the Euclidean distance between the first and last optimized keyframe pose. Heading error is the rotation about the world vertical axis between those poses, computed as the world-Z component of `R_last · R_0ᵀ`. This was verified to agree with the reported yaw difference to two decimal places across all five runs.

**These figures measure loop-closure endpoint error, not trajectory error.** Returning to the marked start within a centimetre proves the loop closed; it does not prove the middle of the path was correct. True ATE/RPE requires external ground truth (motion capture, or a benchmark dataset such as TUM RGB-D). That distinction is stated deliberately.

---

## Loop Closure

Closure detection is a proximity search over past keyframes, followed by **seeded coarse-to-fine ICP verification** on each candidate. Stage 1 runs at a 0.3 m correspondence radius for reach; stage 2 re-runs from stage 1's transform at the normal 0.09 m radius for an honest score. **Gating uses stage 2 only**, because stage 1's fitness is inflated by construction at the loose radius.

Two decisions made closure work at all.

**Seeding with `Tᵢ⁻¹ · Tⱼ` from live optimized poses.** The verification ICP originally started from identity, which placed it a full drift-distance away with a 9 cm correspondence radius. It found nothing and returned fitness 0.000 on every candidate. ICP is a *local* search. It pairs points within the correspondence distance and moves toward those pairs only. It cannot cross a gap larger than that radius. The seed transports the cloud; ICP covers the remainder.

**Coarse-to-fine rather than a single wide pass.** Rotation error displaces points in proportion to range: 6.5° of yaw error moves a 3 m wall point 34 cm, but a 0.5 m floor point only 5.7 cm. At a 9 cm radius, near points find partners and far points do not, capping fitness near 0.6 regardless of how good the alignment actually is. Simply widening the radius does *not* fix this; it lets points on genuinely different surfaces pair up, inflating fitness without improving alignment.

**Before this change candidates topped out at 0.510 to 0.699 and nothing was ever accepted. After it, accepted closures score 0.93 to 0.99.**

The rectangle runs capture the entire progression in a single log as the operator walks back toward the start:

| Seed distance | Coarse fitness | Fine fitness | Outcome |
|---|---|---|---|
| 0.90 m | 0.000 | 0.000 | reject |
| 0.77 m | 0.000 | 0.000 | reject |
| 0.66 m | 0.436 | 0.270 | reject |
| 0.54 m | 0.908 | 0.651 | reject |
| **0.11 m** | **0.998** | **0.929** | **accept** |

The two failure modes are distinguishable by fitness alone, a diagnostic that fell out of this analysis:

- **Seed far outside the coarse radius gives fitness ≈ 0.0.** Every point is displaced equally, so all correspondences fail together.
- **Rotation-dominated failure gives fitness ≈ 0.3 to 0.7.** Near points match, far points do not.

Measured directly: a candidate with 3.3 cm of position-seed error but ~21° of yaw error scored 0.253 and was rejected. A candidate with 10.1 cm of position-seed error, three times worse, but only ~2° of yaw error scored 0.975 and was accepted. **Loop closure is far more sensitive to rotation error than to translation error.**

---

## Performance

Median per-keyframe cost, measured over a full run:

| Stage | Median | Note |
|---|---|---|
| ICP registration | ~250 ms | dominant cost |
| Map rebuild | ~80 ms | own thread, non-blocking |
| iSAM2 update | 1–2 ms | never the bottleneck |
| **Total** | **~348 ms** | |
| Closure keyframes | 1.4–1.8 s | verification ICP on each candidate |

Runs sustain ~100 keyframes over a 4-minute walk with zero to one weak ICP edge.

**Profiling redirected the optimization effort away from the intuitive target.** iSAM2 was the obvious suspect and measures at 1 to 2 ms, so "make the backend faster" would have achieved nothing. Within ICP, the registration solver itself is only 27–79 ms; **voxel downsampling and normal estimation account for roughly 80% of ICP time.** That is the next optimization target, and it is not where the search started.

**Adaptive point cap.** ICP cost is monotonic in point count. Measured: 13k points at 24 ms, 32k at 79 ms, 51k at 2116 ms, 85k at 2741 ms. Wide-open views produce 7× the voxel count of tight views from the same ~900k raw points. When a downsampled cloud exceeds the cap, the **original** cloud is re-voxelized at `voxel_size × √(n/cap)` rather than randomly thinned, because normals are estimated on this cloud and uniform spacing matters. Worst-case registration dropped from 2914 ms to ~400 ms.

---

## Key Engineering Decisions

### IMU delivery: 9.4 Hz → 202 Hz

The single largest fix in the project, and it had **three compounding causes**. Each independently capped the rate, so fixing any one alone changed nothing:

1. The SDK's frame aggregate mode was set to `FULL_FRAME_REQUIRE`, which only emits a frameset when *every* stream has a fresh frame. This pinned IMU delivery to the depth rate (~10 Hz).
2. The driver used a polling loop, where each `wait_for_frames()` call returns exactly one frameset, capping delivery at ~84 Hz regardless.
3. Depth-to-cloud conversion (~120 ms) ran on the delivery thread, blocking it.

Fixed with SDK push callbacks, `ANY_SITUATION` aggregation, and moving depth conversion to a worker thread behind a drop-on-full queue. Result: **202 Hz with an inter-sample standard deviation of 0.022 ms.** Allan variance was then re-derived from 1.46M samples over 2 hours at the corrected rate, because noise densities are rate-dependent and everything measured at 9 Hz was invalid.

### Turn-on bias is measured at startup, not assumed zero

Allan variance characterizes how bias *drifts*; it says nothing about the current offset. These are different quantities and conflating them is a common error. Zero-initialized bias forced ~0.075 m/s² of real MEMS offset into pose estimates, roughly **15 cm of phantom translation per 2-second window.** A static collection at startup now seeds both tilt (roll/pitch from gravity) and turn-on bias into keyframe 0 with an anisotropic prior.

### The IMU cannot supply position, and the graph is built around that

Gravity is 9.81 m/s² against roughly 0.3 m/s² of hand motion. After cancellation the residual is `orientation_error × gravity`, double-integrated, growing as t². At an 800 ms keyframe window, just 2° of orientation error leaves `0.5 · g·sin(2°) · 0.8² ≈ 11 cm` of residual, which **exceeds the 9 cm ICP correspondence radius**. The IMU translation estimate is therefore worse than no estimate at all as an ICP seed.

Consequences, all verified experimentally: the odometry ICP initial guess is **rotation-only**; the keyframe trigger uses a gravity-cancelled translation rather than raw preintegrated position (raw magnitude balloons to hundreds of metres over a long window); and three configuration flags that depend on preintegrated velocity remain permanently disabled.

### Failed ICP registrations become weak edges, not dropped edges

A rejected registration was originally discarded entirely, leaving that keyframe's position constrained by the `ImuFactor` alone, which is the one sensor that physically cannot supply position. That created a hole in the constraint chain, and loop-closure corrections **pooled at the hole**, producing a localized map tear. Rejected edges are now added with sigmas scaled 15×. A fitness-0.35 registration is a poor measurement, not a meaningless one, and a weak constraint beats leaning on the accelerometer.

### ICP rotation sigma was deliberately loosened

Tightened to 0.009 rad, then reverted to 0.03. The tight value claims every keyframe rotation is accurate to half a degree, but roughly 50 to 70 chained edges accumulate 12°+ of disagreement with a closure, so the claim is false. **A too-stiff chain cannot absorb a closure smoothly**; the correction pools wherever the graph happens to be softest instead of distributing. Loosening rotation alone moved a circle run from 15.26° final yaw with a torn map to 3.19°.

---

## Known Limits

**Residual map seam at the closure keyframe. Structural, not a bug.** The map is per-keyframe point clouds bolted rigidly to poses. A closure moves the closure keyframe's pose; its neighbours were not part of that measurement and move less; where their surfaces overlap, they disagree. Observed behaviour matches exactly: only the surfaces viewed during the revisit shift, while the rest of the room stays put. Mitigations reduce it substantially but cannot remove it. Removing it fully requires a different map representation: surfels with a deformation graph (ElasticFusion), frame de-integration and re-integration (BundleFusion), or a landmark map optimized jointly with poses via bundle adjustment. All are substantial rewrites.

**Depth quality is lighting-dependent, and it constrains rotation.** The Gemini 435Le uses active stereo, which requires visible texture. In a dim room the depth map degrades and ICP is left under-constrained in rotation **while still reporting acceptable fitness**, which is a silent failure. Measured directly: identical counter-clockwise loops in low light ended at −42°, −46°, and −21° of heading error. The same loop, same code, lights on, ended at −2.8°.

**Point-to-plane ICP is degenerate against a flat wall** for motion parallel to the plane. Fitness reads 0.95 while translation is under-measured roughly 5×. Feature-rich views with visible corners are a requirement, not a preference. Eigenvalue-based degeneracy detection on the ICP information matrix is the principled fix and is not yet implemented.

**Depth rate is a hardware ceiling.** 1280×800 @ 10 fps or 640×400 @ 20 fps, with no 30 or 60 fps mode. At 10 fps and 90°/s of rotation, there is 9° of viewing-angle change between consecutive frames and the intermediate frames do not exist.

**Scale.** Results above are on 0.6 × 0.9 m and ~0.9 m closed paths. Larger loops have not yet been characterized under this protocol.

---

## What I Learned

**Fitness is not accuracy.** The single most useful lesson in the project. Point-to-plane ICP against a flat wall reports fitness 0.95 while under-measuring translation five-fold. A loop-closure candidate with 21° of yaw error scores 0.253; one with three times the position error but 2° of yaw scores 0.975. A high score means "many points found partners," which is not the same as "the transform is right." Every gate in the system had to be designed around that gap.

**The measurement setup was harder to get right than the algorithm.** Early runs scattered 0.42° to 5.52° in heading across four configurations, a spread wider than the differences between the configurations, which meant no change could be attributed to anything. Fixing it required a marked physical ground truth and repeat runs on a *fixed* configuration. With those in place, the same system produced 0.75° to 1.89°. The scatter was never noise; it was the absence of a controlled protocol.

**Textbook SLAM teaches the shape of the problem, not the failure modes.** Landmarks, probability, and EKF derivations do not tell you that gravity is 30× your hand motion, that a bad ICP seed is worse than no seed, or that Allan variance and turn-on bias are different quantities. Those came from instrumenting the pipeline and reading logs.

---

## Future Work

- **Cache ICP preprocessing.** Voxel downsampling and normals are ~80% of ICP time, and each keyframe's target cloud is the previous keyframe's source cloud, recomputed from scratch
- **Eigenvalue-based degeneracy detection.** Inflate a between-factor's sigma when the ICP information matrix is near-singular, catching the "confident nonsense" case that fitness cannot see
- **Move loop closure to its own thread.** Verification ICP costs 1.0 to 1.4 s and currently stalls the keyframe path
- **Characterize on larger loops** and against a public benchmark for true ATE/RPE
- **Deformable map representation** to eliminate the closure seam
