---
title: Kalman Filters
description: Implementing an EKF for online parameter estimation and a quaternion UKF for orientation tracking from calibrated IMU measurements.
tags:
  - Robotics
cover: /assets/images/covers/Kalman-filters.png
thumbnail: /assets/images/thumbnails/Kalman-filters.jpg
---

## Abstract

This work presents two nonlinear state-estimation implementations developed without a filtering library. The first is an Extended Kalman Filter (EKF) for joint state and parameter estimation in a simulated scalar dynamical system. The second is an Unscented Kalman Filter (UKF) for estimating three-dimensional orientation and angular velocity from accelerometer and gyroscope measurements. Vicon motion-capture measurements were used to calibrate the inertial sensors and evaluate orientation tracking. The EKF recovered an unknown system parameter from an intentionally inaccurate initialization. The UKF represented attitude with a unit quaternion and performed uncertainty calculations in a six-dimensional local error space, avoiding invalid Euclidean operations on quaternion states.

## Problem Formulation

The two experiments address complementary nonlinear-estimation problems. In the first, an unknown model parameter is incorporated directly into the state vector and estimated from nonlinear observations. In the second, the system state lies partly on the unit-quaternion manifold, so conventional vector averaging and additive state corrections are not geometrically valid. Both implementations follow the prediction-and-correction structure introduced by [Kalman](https://doi.org/10.1115/1.3662552) (1960), but differ in how nonlinear transformations are approximated. The EKF linearizes its transition and observation models with Jacobians, whereas the UKF propagates a deterministic collection of sigma points through the nonlinear functions, following the unscented-filtering framework described by [Julier and Uhlmann](https://doi.org/10.1109/JPROC.2003.823141) (2004).

## Extended Kalman Filter for Parameter Estimation

The simulated scalar system used the transition model `x[k+1] = a * x[k] + noise` and the nonlinear observation model `y[k] = sqrt(x[k]^2 + 1) + noise`. The coefficient `a` was unknown. A sequence of 100 observations was generated with the ground-truth value `a = -1`. To estimate the coefficient online, the state was augmented to include both `x` and `a`, and the parameter was modeled as constant between successive time steps. Jacobians were derived for both the augmented transition function and the nonlinear observation function.

The coefficient estimate was initialized at `-10` with variance `1`, providing a substantial initial error relative to the true value. At each time step, the EKF propagated the augmented mean and covariance, computed a Kalman gain from the linearized observation, and applied the measurement innovation. As additional observations were incorporated, the coefficient estimate approached `-1` and its variance decreased. This result demonstrates the use of state augmentation for online parameter identification in a nonlinear dynamical model.

## Quaternion Unscented Kalman Filter

The orientation experiment used recorded inertial measurement unit data containing three-axis accelerometer and gyroscope signals. Sensor calibration was performed against Vicon measurements before filtering. For the accelerometer, gravity was assumed to be the dominant acceleration. The Vicon orientation was used to transform gravity from the world frame into the body frame, after which linear regression estimated the bias and sensitivity of each axis. For the gyroscope, reference angular velocity was computed from consecutive Vicon rotations, and axis-wise bias and sensitivity were estimated with the same regression procedure.

The UKF state consisted of a unit quaternion and a three-dimensional angular-velocity vector. Its covariance was represented in a six-dimensional local space containing a three-dimensional axis-angle attitude error and a three-dimensional angular-velocity error. Twelve sigma points were generated from this covariance. For each point, the attitude perturbation was converted from axis-angle form to a quaternion and composed with the current orientation estimate. This construction follows the quaternion-UKF formulation of [Kraft](https://doi.org/10.1109/ICIF.2003.177425) (2003), retaining the unit-norm constraint while allowing covariance operations to remain Euclidean in the local error space.

During prediction, angular velocity was integrated over the measured interval between consecutive IMU timestamps. The resulting incremental quaternion propagated the orientation component of every sigma point. The predicted quaternion mean was computed iteratively: each sigma-point quaternion was expressed as an error relative to the current mean, the errors were averaged in axis-angle coordinates, and the mean was updated until the average error magnitude fell below `1e-6`.

The measurement model predicted gravity in the body frame and combined this prediction with angular velocity. The filter then computed the predicted observation covariance, the state--observation cross covariance, the Kalman gain, and the innovation from the calibrated IMU measurements. The resulting attitude correction was converted to a quaternion and composed with the predicted state rather than added component-wise.

## Experimental Results

The EKF experiment evaluated convergence of both the coefficient estimate and its uncertainty. Despite initialization at `-10`, the estimate converged toward the ground-truth coefficient `-1` over the 100-observation sequence. The accompanying reduction in variance indicates that the measurements progressively constrained the augmented parameter state.

The UKF orientation estimate was compared with Vicon in Euler-angle and quaternion representations. The Euler-angle trajectories in Figure 1 provide a direct view of the tracking response. In the quaternion comparison, several estimated components appear to have the opposite sign from the Vicon components. This difference does not necessarily indicate orientation error because a quaternion and its negation encode the same physical rotation. Evaluation based only on component-wise quaternion differences would therefore be misleading without first resolving this sign ambiguity.

![EKF parameter convergence and UKF orientation tracking results](/assets/images/posts/kalman-filters/ekf-and-orientation-results.jpg)

*Figure 1. Left: the EKF coefficient estimate converges from `-10` to the true value of `-1`. Right: the UKF orientation estimate follows the Vicon reference in Euler-angle form.*

## Limitations and Discussion

The parameter-estimation experiment used simulated data and a scalar model, so its primary purpose was to validate the augmented-state EKF formulation rather than characterize performance on a complex physical system. The orientation experiment relied on Vicon measurements for both sensor calibration and reference evaluation. Consequently, the reported comparison characterizes estimator behavior under the recorded experimental conditions but does not establish performance when external calibration data are unavailable. The implementation also emphasizes qualitative trajectory agreement; the report does not present a single aggregate attitude-error metric.

## Conclusion

The two estimators demonstrate nonlinear Kalman filtering under different structural constraints. State augmentation enabled the EKF to recover an unknown dynamics coefficient, while the quaternion UKF combined manifold-consistent attitude updates with local Euclidean covariance calculations. These implementations illustrate the importance of sensor calibration and geometry-consistent state representations in addition to covariance propagation.

## References

- R. E. Kalman. [“A New Approach to Linear Filtering and Prediction Problems.”](https://doi.org/10.1115/1.3662552) *Journal of Basic Engineering*, 1960.
- S. J. Julier and J. K. Uhlmann. [“Unscented Filtering and Nonlinear Estimation.”](https://doi.org/10.1109/JPROC.2003.823141) *Proceedings of the IEEE*, 2004.
- E. Kraft. [“A Quaternion-Based Unscented Kalman Filter for Orientation Tracking.”](https://doi.org/10.1109/ICIF.2003.177425) *Sixth International Conference of Information Fusion*, 2003.

## Project Materials

### Report

<div class="row">
    <iframe src="/assets/files/reports/kalman-filters.pdf" title="Kalman filters project report" loading="lazy" style="width:100%; height:550px"></iframe>
</div>

[Open the Kalman filters report as a PDF](/assets/files/reports/kalman-filters.pdf)
