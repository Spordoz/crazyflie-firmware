---
title: The Loco Positioning System
page_id: loco_positioning
---

The Loco Positioning System is an absolute positioning system based on Ultra Wide Band (UWB) radios.

Most of the documentation for the Loco Positioning System can be found in the [lps-node-firmware repository documentation](https://www.bitcraze.io/documentation/repository/lps-node-firmware/master/), this is where protocols, principles and modes are described.

This section contains details specific to the implementation in the Crazyflie, mainly estimator related information.

## Position estimation

When using the Loco Positioning System the [Kalman estimator](/docs/functional-areas/sensor-to-control/state_estimators.md#extended-kalman-filter) is used for position estimation. The two main ranging schemes used for UWB localization are (i) two-way ranging (TWR) and (ii) time-difference-of-arrival (TDoA). In this page, we elaborate the measurement model, Kalman filter update and robust estimation for both TWR and TDoA.

### Two-way ranging (TWR)

In TWR, the UWB tag mounted on the Crazyflie communicates with fixed UWB anchors and acquires range measurements through two-way communication. The measurement model is as follows: 

$$d_i = \sqrt{(x-x_i)^2 +(y-y_i)^2 + (z-z_i)^2}$$,

where $(x, y, z)$ is the position of the Crazyflie and $(x_i, y_i, z_i)$ is the position of the fixed anchor i. For the conventional extended Kalman filter, we have 

$$g_x = (x-x_i) / d_i$$

$$g_y = (y-y_i) / d_i$$

$$g_z = (z-z_i) / d_i$$.

The H vector is

$$H = (g_x, g_y, g_z, 0, 0, 0, 0, 0, 0)$$. 

Then, we call the function `kalmanCoreScalarUpdate()` in [kalman_core.c](https://github.com/bitcraze/crazyflie-firmware/blob/master/src/modules/src/kalman_core/kalman_core.c) to update the states and covariance matrix.

### Time-difference-of-arrival (TDoA)
In TDoA, UWB tags receive signals from anchors passively and compute the difference in distance beween two anchors as TDoA measurements. Since in TDoA scheme, UWB tags only listen to the messages from anchors, a TDoA-based localization system allows a theoretically unlimited number of robots to localize themselves with a small number of fixed anchors. However, TDoA measurements are more noisy than TWR measurements, leading to a less accurate localization performance. Two types of TDoA protocols ([TDoA2](https://www.bitcraze.io/documentation/repository/lps-node-firmware/master/protocols/tdoa2_protocol/) and [TDoA3](https://www.bitcraze.io/documentation/repository/lps-node-firmware/master/protocols/tdoa3_protocol/)) are implemented in LPS system. The main difference between the two TDoA protocols is that TDoA3 protocol achieves the scalability at the cost of localization accuracy. The readers are refer to [TDoA2 VS TDoA3](https://www.bitcraze.io/documentation/repository/lps-node-firmware/master/functional-areas/tdoa2-vs-tdoa3/) for detailed information. 

The TDoA measurement model is as follows:

$$d_{ij} = d_j - d_i = \sqrt{(x-x_j)^2 +(y-y_j)^2 + (z-z_j)^2} - \sqrt{(x-x_i)^2 +(y-y_i)^2 + (z-z_i)^2}$$,

where $(x, y, z)$ is the position of the Crazyflie and $(x_i, y_i, z_i)$ and $(x_j, y_j, z_j)$ are the positions of fixed anchor i and j, respectively. For the conventional extended Kalman filter, we have

$$g_x = (x-x_j) / d_j - (x-x_i) / d_i$$

$$g_y = (y-y_j) / d_j - (y-y_i) / d_i$$

$$g_z = (z-z_j) / d_j - (z-z_i) / d_i$$. 

The H vector is

$$H = (g_x, g_y, g_z, 0, 0, 0, 0, 0, 0)$$.

Then, we call the function `kalmanCoreScalarUpdate()` in [kalman_core.c](https://github.com/bitcraze/crazyflie-firmware/blob/master/src/modules/src/kalman_core/kalman_core.c) to update the states and covariance matrix.

### M-estimation based robust Kalman filter
UWB radio signal suffers from outlier measurements caused by radio multi-path reflection and non-line-of-sight propagation. The large erroneous measurements often deteriorate the accuracy of UWB localization. The conventional Kalman filter is sensitive to measurement outliers due to its intrinsic minimum mean-square-error (MMSE) criterion. Here, we provide a robust estiamtion approach based on M-estimation robust cost function. We will explain the general idea of the robust Kalman filter and readers are encouraged to look into the firmware [mm_distance_robust](https://github.com/bitcraze/crazyflie-firmware/blob/master/src/modules/src/kalman_core/mm_distance_robust.c) and [mm_tdoa_robust](https://github.com/bitcraze/crazyflie-firmware/blob/master/src/modules/src/kalman_core/mm_tdoa_robust.c). The implementation is based on paper [1] and please read the paper for implementation details.

From the Bayesian maximum a posteriori perspective, the Kalman filter state estimation framework can be derived by solving the following minimization problem:

<img src="/docs/images/rkf-eq1.png" alt="drawing" width="500"/>

Therein, $x_k$ and $y_k$ are the system state and measurements at timestep k, $P_k$ and $R_k$ denote the prior covariance and measurement covariance, respectively. Through Cholesky factorization of $P_k$ and $R_k$, the original optimization problem is equivalent to:

<img src="/docs/images/rkf-eq2.png" alt="drawing" width="450"/>

where $e_{x,k,i}$ and $e_{y,k,i}$ are the elements of $e_{x,k}$ and $e_{y,k}$. To reduce the influence of outliers, we incorporate a robust cost function into the Kalman filter framework as follows:

<img src="/docs/images/rkf-eq3.png" alt="drawing" width="450"/>

where $\rho()$ could be any robust function (e.g., G-M, SC-DCS, Huber, Cauchy, etc.)

By introducing a weight function for the process and measurement uncertainties---with e as input---we can translate the optimization problem into an Iterative Reweight Least-Square (IRLS) problem. Then, the optimal posterior estimate can be computed through iteratively solving the least-square problem using the robust weights computed from the previous solution. In our implementation, we use the G-M robust cost function and the maximum iteration is set to be two for computational frugality. Then, we call the function `kalmanCoreUpdateWithPKE()` in [kalman_core.c](https://github.com/bitcraze/crazyflie-firmware/blob/master/src/modules/src/kalman_core/kalman_core.c) with the weighted covariance matrix $P_w_m$, kalman gain Km, and innovation error to update the states and covariance matrix.

This functionality can be turned on through setting a parameter (robustTwr or robustTdoa) in [estimator_kalman.c](https://github.com/bitcraze/crazyflie-firmware/blob/master/src/modules/src/estimator_kalman.c).

## References
[1] Zhao, Wenda, Jacopo Panerati, and Angela P. Schoellig. "Learning-based Bias Correction for Time Difference of Arrival Ultra-wideband Localization of Resource-constrained Mobile Robots." IEEE Robotics and Automation Letters 6, no. 2 (2021): 3639-3646.