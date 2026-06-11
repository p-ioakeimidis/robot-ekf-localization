# Global & Local Localization 

Summary
- This document describes the two localization packages in this workspace:
  - `global_localization` — global/global EKF (coarse/global localization, fusing GPS/odom/IMU/map-frame data).
  - `local_localization` — local/local EKF (high-rate local odometry/IMU fusion for accurate short-term pose).
- Location: `src/global_localization` and `src/local_localization`.
- Goal: provide a robust, two-layer localization pipeline for simulation (Gazebo) and real robot data.

Repository layout (relevant)
- src/
  - first_robot/            (robot model, sensors, Gazebo launches, map)
    - launch/gazebo.launch  (spawn robot, start Gazebo)
    - maps/my_map.*         (occupancy map used by global localization / rviz)
  - global_localization/
    - launch/global_ekf.launch
    - config/global_ekf.yaml
  - local_localization/
    - launch/local_ekf.launch
    - config/local_ekf.yaml

Prerequisites
- ROS Noetic on Ubuntu 20.04
- Common ROS packages: `robot_state_publisher`, `joint_state_publisher`, `gazebo_ros`, `tf`, `nav_msgs`, `sensor_msgs`, `robot_localization` (EKF/UKF), `rviz`
- Build workspace:
  - cd ~/catkin_ws
  - catkin_make
  - source devel/setup.bash

Quickstart (simulation)
1. Start Gazebo and spawn the robot:
   - roslaunch first_robot gazebo.launch
2. Start the global EKF (coarse/global fusion):
   - roslaunch global_localization global_ekf.launch
3. Start the local EKF (high-rate local fusion):
   - roslaunch local_localization local_ekf.launch
4. Open RViz to visualize:
   - roslaunch first_robot rviz.launch
   - Show topics: /map, /tf, /odometry (filtered output), /scan, /robot_model
5. (Optional) Use RViz "2D Pose Estimate" to set initial pose or publish /initialpose for global EKF to converge.

Notes on ordering
- Start Gazebo / sensors first so sensor topics exist.
- Start global EKF to fuse map/GPS/low-rate sensors and produce a stable world-frame pose.
- Start local EKF after global EKF so local filter can use corrected odom/imu if configured.

Key files & parameters
- global_localization/config/global_ekf.yaml
  - Contains `frequency`, `sensor_timeout`, `ekf` process and measurement covariances, `map_frame` / `odom_frame` / `base_link_frame`, and which sensor inputs are enabled (GPS, odom, imu, etc.).
- local_localization/config/local_ekf.yaml
  - Tuned for higher update rates (wheel odometry + IMU), lower latencies; different covariances and timeout values.
- global_localization/launch/global_ekf.launch
  - Launches `robot_localization` node (ekf_localization_node or ukf) with the global config.
- local_localization/launch/local_ekf.launch
  - Launches another `robot_localization` node for local fusion; typically publishes a filtered odometry topic used by planners/controllers.

Common topics & frames (adjust to your configs)
- Sensor inputs (from robot/Gazebo):
  - /imu/data (sensor_msgs/Imu)
  - /odom (nav_msgs/Odometry) — wheel odometry
  - /gps/fix or /fix (sensor_msgs/NavSatFix) — if present
  - /scan (sensor_msgs/LaserScan) — for mapping/rviz
- EKF outputs:
  - Filtered odometry (commonly `/odometry/filtered` or a namespaced topic defined in launch)
  - published TF between `map` and `odom` (or `world`) depending on configuration
- Frames:
  - map (global map frame)
  - odom (local odometry frame)
  - base_link (robot base)
  - imu_link, gps_link, etc.

Usage tips
- Use RViz to verify TF tree (`rosrun tf tf_monitor` / `rosrun tf view_frames`).
- If EKF won't fuse a sensor: verify topic names, message types, timestamps, and frame_ids match the config.
- If filter is unstable: increase covariance on the problematic sensor (in YAML), verify IMU orientation covariances.
- To set initial global pose: publish `/initialpose` (geometry_msgs/PoseWithCovarianceStamped) from RViz or a node.

Troubleshooting
- No output from EKF:
  - Confirm the `ekf` node is running: rosnode list
  - Check topics with rostopic list / rostopic echo
  - Inspect node logs: roslaunch ... then check terminal output or `rosparam get /` for config values
- TF discontinuities / jumps:
  - Check frame_id correctness and consistent timestamps.
  - Ensure only one node is publishing `map->odom` or `odom->base_link` transforms.
- Delay/latency issues:
  - Confirm `use_sim_time` when using Gazebo (set to true in gazebo.launch).
  - Ensure `sensor_timeout` and `frequency` in YAML align with your sensor rates.

Tuning guidelines
- Start with conservative (large) covariances for noisy sensors and tighten as you validate performance.
- For global EKF: rely more on GPS/map (lower covariance) and less on wheel odometry if wheel slippage is expected.
- For local EKF: trust high-rate IMU and wheel odom; set IMU orientation covariance low if IMU is accurate.

Example roslaunch commands
- Simulation full stack:
  - source ~/catkin_ws/devel/setup.bash
  - roslaunch first_robot gazebo.launch
  - roslaunch global_localization global_ekf.launch
  - roslaunch local_localization local_ekf.launch
  - roslaunch first_robot rviz.launch

Extending / Contribution notes
- Add sensors: update robot URDF (first_robot/urdf) and adjust YAML covariances.
- Add diagnostics: publish sensor health and use `diagnostic_aggregator`.
- Add static transforms in `first_robot/launch` if some frames are missing.

Contact / References
- See `robot_localization` ROS wiki for EKF/UKF parameters and examples:
  - http://wiki.ros.org/robot_localization
