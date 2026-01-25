---
layout: default
title: TurtleBot3 Navigation
---


# Autonomous Navigation with ROS and TurtleBot3

## Overview

This project focused on building a complete autonomous navigation pipeline for a TurtleBot3 mobile robot using ROS. The goal was to integrate mapping, localization, and path planning into a single, synchronized system that could operate reliably in both simulation and on real hardware.

Rather than implementing individual algorithms from scratch, this project emphasized system integration, parameter tuning, node synchronization, and real-world deployment, which are critical skills in practical robotics systems.

## Problem Setup

The objective was to enable a TurtleBot3 robot to autonomously navigate in an unknown environment by completing the following pipeline:

1. Mapping the environment using 2D LiDAR  
2. Localizing the robot within a saved map  
3. Planning and executing collision-free paths to user-defined goals  

The navigation system needed to:

- Work end-to-end using ROS nodes  
- Be stable in simulation  
- Transfer successfully to real hardware  
- Handle localization loss and recovery  

This required coordinating multiple ROS packages that operate asynchronously and rely heavily on consistent timing, topic synchronization, and correct parameter configuration.

## System Architecture

The navigation stack was built by integrating the following components:

- Mapping: slam_gmapping for 2D occupancy grid generation  
- Map Server: for storing and serving the static map  
- Localization: amcl (Adaptive Monte Carlo Localization)  
- Path Planning: move_base with global and local costmaps  
- Local Planner: DWA-based local planner  
- Sensors: 2D LiDAR and odometry  
- Visualization: RViz  
- Simulation: Gazebo  

These components communicate through ROS topics, services, and TF frames, forming a tightly coupled system.

(Add architecture diagram showing SLAM → map_server → AMCL → move_base → robot)

## What I Built

- A fully functional ROS navigation stack for TurtleBot3  
- Launch files and configurations for:
  - Mapping
  - Localization
  - Path planning  
- Parameter files for:
  - Global and local costmaps
  - Local planner behavior
  - Localization accuracy  
- A navigation pipeline capable of:
  - Accepting start and goal poses
  - Planning paths
  - Executing them autonomously  
- Successful deployment in both simulation and on physical hardware  

I did not reimplement SLAM, localization, or planners, but instead focused on correct integration, synchronization, and tuning of existing ROS packages.

## What Was Hard

The most challenging part of this project was integrating all nodes and synchronizing them properly so the full navigation stack behaved correctly.

One major issue occurred between AMCL and move_base. The move_base node continuously subscribed to the map topic to generate global and local costmaps, while the AMCL node initially received the map only once. This caused false localization and misalignment between the costmaps and the actual map, leading to unstable navigation.

By modifying AMCL to subscribe to the map topic in the same way as move_base, localization accuracy improved and the costmaps aligned correctly with the map.

## Decisions I Made

- Tuned AMCL parameters such as particle count and confidence thresholds to improve localization accuracy  
- Adjusted global and local costmap parameters including obstacle inflation and resolution  
- Tuned the local planner to balance adherence to the global plan with obstacle avoidance  
- Adjusted velocity and acceleration limits to remain within hardware constraints  

These decisions were made through extensive trial and error with the goal of enabling safe navigation even in narrow passages.

## What Worked and What Didn’t

After synchronizing the nodes and tuning parameters, the system worked reliably in simulation. The robot could accept start and goal poses and execute paths end-to-end.

Occasional localization loss was handled using a recovery behavior where the robot rotated in place to gather additional sensor data for relocalization.

Deploying the stack on hardware introduced new challenges due to communication latency and sensor noise. Additional tuning and trade-offs were required, but after these adjustments the system worked reliably on the real robot. A full hardware demonstration was recorded on YouTube.

## What I Learned

This project provided deep insight into how autonomous navigation systems work as a complete pipeline rather than isolated algorithms.

Key takeaways include:
- Autonomous navigation is fundamentally a systems integration problem  
- Synchronization issues can cause significant failures  
- Simulation success does not guarantee hardware success  
- Parameter tuning is critical for real-world deployment  

This experience motivated me to further focus on motion planning, leading to more advanced work in trajectory optimization and Safe Flight Corridor generation.

## Results

- End-to-end autonomous navigation in simulation  
- Successful deployment on physical TurtleBot3 hardware  
- Robust localization and planning after tuning  
- Live technical demonstration on YouTube  

(Add RViz screenshots, Gazebo simulation images, and hardware demo video here)
