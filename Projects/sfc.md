\# Safe Flight Corridor (SFC) Generation for Quadrotor Trajectory Planning



\## Overview



Trajectory optimization for quadrotors becomes difficult when collision avoidance is enforced directly, since free space is non-convex. This project implements Safe Flight Corridor (SFC) generation to convert non-convex free-space constraints into a sequence of overlapping convex polytopes, enabling efficient trajectory optimization.



\## Problem Setup



Minimum-snap trajectory optimization requires convex constraints. Enforcing collision avoidance directly makes the problem non-convex and difficult to solve reliably.



The goal of this project was to reformulate collision avoidance constraints so that trajectory optimization could remain convex and computationally efficient.



\## Core Idea



Instead of constraining trajectories to remain in free space directly, the trajectory is constrained to lie inside a sequence of overlapping convex polytopes (Safe Flight Corridors).



If the corridor sufficiently overlaps and covers a valid homotopy class, the resulting trajectory remains collision-free.



\## Algorithm Architecture



The pipeline follows an ADMM-based alternating optimization strategy:



1\. Generate an initial path using A\* or RRT  

2\. Initialize ellipsoids along the path  

3\. Inflate ellipsoids into convex polytopes using IRIS  

4\. Optimize waypoints within corridor constraints  

5\. Optimize ellipsoids for maximum volume  

6\. Update Lagrange multipliers  

7\. Repeat until convergence  



(Add system diagram of ADMM loop here)



\## What I Implemented



\- Full SFC algorithm implementation from pseudocode  

\- ADMM optimization loop  

\- IRIS-based polytope inflation  

\- Waypoint and ellipsoid optimization  

\- Corridor validation in randomized environments  

\- 2D to 3D extension  

\- MATLAB prototype followed by C++ ROS 2 implementation  



\## Engineering Decisions



\### Polytope Subsampling



Redundant polytopes were removed using an overlap-based subsampling strategy, reducing corridor length while preserving feasibility.



\### Bounding Box Selection



Bounding box size was tuned based on obstacle density:

\- Smaller boxes for dense environments  

\- Larger boxes for sparse environments  



This significantly improved corridor quality.



\## Results



\- Tested on 20+ randomized 2D environments  

\- Successful 3D corridor generation  

\- Consistent convergence within a few ADMM iterations  

\- High-quality overlapping corridor chains  



(Add MATLAB plots and RViz visualizations here)



\## Lessons Learned



\- Reformulating constraints simplifies optimization  

\- Corridor quality is highly sensitive to geometric parameters  

\- Redundancy reduction is important for real-time performance  



\## Next Steps



\- Real-time onboard implementation  

\- Hardware deployment on quadrotor platforms  

\- Runtime benchmarking and optimization  



