---
title: LiDAR-based Particle Filter SLAM
description: Building LiDAR-based particle-filter SLAM from motion and observation updates, stratified resampling, and log-odds occupancy mapping.
tags:
  - Robotics
cover: /assets/images/covers/Particle-Filter-SLAM.jpg
thumbnail: /assets/images/thumbnails/Particle-Filter-SLAM.jpg
---

## Abstract

This work presents a two-dimensional simultaneous localization and mapping system based on a particle filter and scanning LiDAR. The available measurements consisted of planar odometry, LiDAR ranges, and head-joint angles. Each particle represented a candidate robot pose `(x, y, yaw)`, while a shared log-odds occupancy grid represented the environment. The implementation combined an odometry-driven motion model, scan-to-map observation scoring, numerically stable weight normalization, selective stratified resampling, and ray-based occupancy updates. The complete pipeline was evaluated on four recorded datasets using 1,000 particles and a 0.05 m map resolution. The resulting maps were evaluated qualitatively by comparing raw odometry trajectories with the trajectories of the highest-weight particles.

## Problem Formulation and System Overview

Simultaneous localization and mapping is a coupled inference problem. A LiDAR scan can update an occupancy map only after the sensor pose has been estimated, while the map itself provides the spatial reference used to evaluate candidate poses. The particle-filter formulation represents this uncertainty with a weighted set of pose hypotheses, following the sequential Monte Carlo framework reviewed by [Arulampalam et al.](https://doi.org/10.1109/78.978374) (2002). Each iteration applies a motion update from odometry, evaluates the current LiDAR scan under every hypothesis, updates and normalizes the particle weights, resamples when the distribution becomes concentrated, and incorporates the scan into the occupancy map using the most likely pose.

The state was restricted to planar position and yaw. Head and neck measurements were used to transform LiDAR returns through the robot kinematic chain, but these joint values were not included in the particle state. The final implementation used 1,000 particles, balancing the number of available pose hypotheses against the cost of evaluating every scan against the map for each particle.

## Motion Model

For each pair of consecutive odometry measurements, the relative planar transformation was computed and applied to every particle. Independent Gaussian process noise was then added to produce a distribution of candidate poses. The motion model was validated separately before integrating LiDAR observations. Under very small process noise, the particle-filter trajectory closely followed the trajectory obtained directly from odometry, confirming the relative-motion calculation and pose composition.

This odometry-driven proposal distribution provides temporal continuity but accumulates drift in the absence of observation corrections. The LiDAR update is therefore required to redistribute probability toward particles whose projected scans agree with the evolving map.

## LiDAR Observation Model

LiDAR and head-joint measurements were recorded with different timestamps. Each LiDAR scan was associated with the nearest head-joint sample. Invalid ranges were removed, and the remaining scan endpoints were transformed from the LiDAR coordinate frame to the robot body frame using the measured head and neck angles. For each particle, the endpoints were subsequently transformed from the body frame into world coordinates according to the particle pose.

The occupancy map covered `-20` to `20` m in both planar directions at a resolution of `0.05` m. Observation scores were computed from the agreement between projected LiDAR endpoints and cells currently represented as occupied. A particle received a larger score when more of its transformed endpoints aligned with occupied map cells. Because direct exponentiation and normalization of scan scores can be numerically unstable, the implementation used a log-sum-exp calculation when updating the particle weights.

After normalization, the highest-weight particle supplied the pose used for map integration. Bresenham's line algorithm enumerated the cells traversed by each LiDAR ray. Cells between the sensor and the measured endpoint received a free-space log-odds increment, while the endpoint received an occupied-space increment. This representation follows the probabilistic occupancy-grid approach described by [Elfes](https://doi.org/10.1109/2.30720) (1989). The additive log-odds representation accumulated evidence from repeated measurements. For visualization, the map was converted to a binary occupancy grid using an occupancy-probability threshold of `0.6`.

## Resampling and Implementation Validation

Repeated resampling can prematurely eliminate pose hypotheses. The implementation therefore monitored the effective particle count, defined as `1 / sum(w[i]^2)`, and performed stratified resampling only when this quantity fell below 30 percent of the total particle count. After resampling, particle weights were reset to a uniform distribution; subsequent motion and observation updates again differentiated the hypotheses.

Separate execution modes were maintained for the motion model, observation model, and complete SLAM system. The observation-only validation initialized three deliberately different particles and verified that the particle whose projected scan aligned most closely with the initial map received the greatest weight. Isolating these components allowed coordinate-transformation and normalization errors to be examined before processing an entire recording.

## Experimental Results

The complete pipeline was executed on four datasets. For each recording, the system saved the final occupancy map and overlaid the raw odometry trajectory with the trajectory of the highest-weight particle. Intermediate maps and particle configurations were also recorded during execution. The same motion, scan-scoring, map-update, and resampling procedures were applied across all four recordings; Figure 1 presents the resulting maps and trajectory overlays.

![Occupancy maps and estimated trajectories from four LiDAR datasets](/assets/images/posts/particle-filter-slam/final-maps.jpg)

*Figure 1. Final maps for the four recordings. Red denotes raw odometry, and green denotes the particle-filter SLAM trajectory.*

The maps exhibit recognizable free-space and obstacle structure, while the overlaid trajectories show differences between uncorrected odometry and the particle-filter estimate. The evaluation is principally qualitative because the available results do not include an independent ground-truth trajectory for computing absolute localization error.

## Limitations and Discussion

The observation model scores endpoint agreement with occupied cells but does not explicitly optimize a continuous scan-matching objective. Map updates use only the highest-weight particle rather than maintaining a separate occupancy map for every hypothesis. This design limits computational cost but does not represent correlation between mapping uncertainty and all candidate poses. Performance is also sensitive to the selected particle count, process-noise model, map resolution, and resampling threshold. Finally, the qualitative comparison with odometry cannot determine absolute pose accuracy without an external ground-truth reference.

## Conclusion

The implemented particle-filter SLAM system integrates odometry-based proposals with LiDAR scan scoring and log-odds occupancy mapping. Timestamp association and kinematic transformations align the LiDAR data with each planar pose hypothesis, while log-sum-exp normalization and effective-sample-size-triggered resampling were used to address numerical underflow and particle degeneracy. The common processing pipeline completed the localization--mapping loop and produced occupancy maps for all four recorded datasets.

## References

- M. S. Arulampalam, S. Maskell, N. Gordon, and T. Clapp. [“A Tutorial on Particle Filters for Online Nonlinear/Non-Gaussian Bayesian Tracking.”](https://doi.org/10.1109/78.978374) *IEEE Transactions on Signal Processing*, 2002.
- A. Elfes. [“Using Occupancy Grids for Mobile Robot Perception and Navigation.”](https://doi.org/10.1109/2.30720) *Computer*, 1989.

## Project Materials

### Report

<div class="row">
    <iframe src="/assets/files/reports/particle-filter-slam-and-nerf.pdf" title="Particle Filter SLAM and NeRF project report" loading="lazy" style="width:100%; height:550px"></iframe>
</div>

[Open the Particle Filter SLAM and NeRF report as a PDF](/assets/files/reports/particle-filter-slam-and-nerf.pdf)
