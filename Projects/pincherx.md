---
layout: default
title: PincherX Pick & Place
---

# Pick-and-Place Motion Planning with PincherX 100

**Robotics Control and Mechanics · Northeastern University · Dec 2024 – Jan 2025**  
**Stack:** MATLAB · PincherX 100 · 4-DOF Joint-Space Planning · Real Hardware

---

![PincherX pick-and-place demo](../Assets/images/pincherx_demo.gif)

---

## Overview

A pick-and-place pipeline for a 4-DOF PincherX 100 robotic manipulator, running on real hardware. The object pose and destination were provided in the robot's base frame by an existing perception pipeline. My focus was entirely on the motion planning layer — getting the robot to move safely, smoothly, and reliably between configurations.

---

## The Problem

The task sounds simple: pick up an object, move it to a target location, put it down, return home. The hard part is making that work on physical hardware without abrupt motion, joint limit violations, or IK failures mid-trajectory.

The sequence required: approach → grasp → lift → transfer → place → return. Each transition needs to be smooth, and each configuration needs to be reachable without the manipulator colliding with itself or the table.

---

## What I Built

The planning module handled the full motion sequence in joint space. Given start and goal configurations, it interpolated smooth joint trajectories, enforced maximum step limits between waypoints to prevent abrupt motion, and inserted intermediate hover configurations above the payload and destination to avoid unsafe diagonal sweeps close to the table surface.

The module integrated directly with existing gripper control and execution interfaces — I didn't reimplement low-level control or perception, but connected the planning output cleanly into the full pipeline.

---

## Key Decision — Joint Space over Cartesian Space

The initial approach used Cartesian-space planning with inverse kinematics to generate end-effector trajectories. This failed on hardware — IK solutions were inconsistent near singularities, and small Cartesian errors produced large unexpected joint motions.

Switching to joint-space planning with linear interpolation between configurations fixed this entirely. Joint-space paths are predictable, smooth, and don't involve online IK solving. For a 4-DOF manipulator on a structured pick-and-place task, this is simply the more reliable choice.

---

## What Was Hard

The joint step limit enforcement was more important than it sounds. Without it, the robot would occasionally produce large inter-waypoint jumps — either because of bad initial configuration estimates or floating-point accumulation across a long trajectory. Adding a hard cap on joint displacement per step, with a validation pass before execution, caught these cases before they reached the hardware.

Real hardware also introduced timing sensitivity that simulation didn't — the execution interface expected waypoints at a specific rate, and feeding them too fast or too slow caused the controller to drop commands.

---

## Results

Successful pick-and-place execution on the physical PincherX 100 across multiple runs. Smooth joint motion throughout the sequence, consistent grasp and release, and reliable return to home configuration. The joint step limit and hover waypoints were the two changes that made the difference between unreliable and consistent execution.
