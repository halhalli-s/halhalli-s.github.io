# Pick-and-Place Motion Planning with PincherX 100 Manipulator

## Overview

This project involved implementing a pick-and-place task on a 4-DOF PincherX 100 robotic manipulator using MATLAB. The focus was on joint-space motion planning and safe execution on real hardware while leveraging existing perception and control interfaces.

## Problem Setup

The task required the robot to:
- Locate a payload and destination using an overhead camera system
- Reach and grasp the payload
- Move it to a target location
- Release the payload
- Return to a home configuration  

The object pose and destination pose were provided in the robot’s base frame using a pre-existing perception pipeline.

## What I Built

My contribution focused on the motion planning logic:

---
layout: default
title: PincherX Pick & Place
---



- Joint-space trajectory planning from start to payload and payload to destination  
- Integration with existing gripper and execution interfaces  
- Trajectory sequencing for pick, lift, move, place, and return operations  

Low-level control, perception, and gripper force logic were handled by existing functions, which I integrated into a complete manipulation pipeline.

## What Was Hard

The main challenge was achieving smooth and reliable motion on real hardware.

An initial Cartesian-space planning approach led to inverse kinematics inaccuracies and unstable execution. Transitioning to joint-space planning resolved these issues and improved robustness.

Testing on real hardware introduced additional challenges such as enforcing movement thresholds and ensuring consistent execution.

## Decisions I Made

- Switched from Cartesian-space planning to joint-space trajectory planning  
- Implemented linear interpolation between joint configurations  
- Enforced maximum joint step limits to avoid abrupt motion  
- Used a persistent transformation check to prevent large jumps between waypoints  

These decisions improved safety and execution stability.

## What Worked and What Didn’t

Joint-space trajectory planning worked reliably and produced smooth motion.

A structured sequence using intermediate checkpoints such as hovering above the payload and destination improved grasp reliability and avoided unsafe diagonal motion.

Cartesian-space planning did not work reliably due to IK instability and was abandoned.

## What I Learned

- Joint-space planning can be more reliable than Cartesian planning for low-DOF manipulators  
- Hardware constraints must drive trajectory design  
- Intermediate motion structure improves robustness  
- Integrating existing systems requires careful validation  

This project strengthened my MATLAB programming skills and provided hands-on experience with real robotic hardware.

## Results

- Successful pick-and-place execution on physical hardware  
- Smooth joint motion within hardware limits  
- Reliable return to home configuration  

(Add short hardware execution video here)
