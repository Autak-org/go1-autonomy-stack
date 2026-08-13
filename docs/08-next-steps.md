# 08 — Next steps (M3 → M5)

M1 (platform + command path) and M2 (OAK-D camera integration) are done. Below is what remains.

---

## M3 — 2D SLAM

**Goal:** real-time 2D occupancy map of a flat indoor environment while the robot walks under teleop.

**Approach:** OAK-D PointCloud2 → virtual LaserScan via [`pointcloud_to_laserscan`](https://github.com/ros-perception/pointcloud_to_laserscan/tree/lunar-devel) → ROS 1 [`slam_toolbox`](https://github.com/SteveMacenski/slam_toolbox/tree/melodic-devel) → `/map` plus a serialized pose graph.

The existing `packages/ros_unitree/pcl2scan/` package is only a local launch/config wrapper around upstream `pointcloud_to_laserscan`; it does not contain a separate point-cloud converter. Use the linked ROS 1 branches with the robot's Melodic installation, not the repositories' ROS 2 examples.

### What's needed

1. **Forward-facing OAK-D.** Currently mounted pointing down. Re-angle before SLAM makes sense.
2. **Verify PointCloud2 output.** Enable it in the installed, patched depthai-ros 2.11.2 stack and measure the actual topic, frame ID, rate and timestamp behaviour on the Xavier. Do not assume `/oak/stereo/points` or a configuration key until confirmed on the robot.
3. **Complete the timestamped TF tree.** Publish the measured static robot-base → OAK-D mount transform, retain DepthAI's camera/optical transforms and use one consistent planar robot base frame. Synchronise the Nano and Xavier clocks, then verify TF lookup at each point-cloud/scan timestamp.
4. **Continuous odometry.** Publish `nav_msgs/Odometry` on `/odom` and the matching dynamic `odom` → robot-base TF independently of command callbacks. Validate the units, axes, translation and yaw from HighState before relying on them for mapping.
5. **Configure `pointcloud_to_laserscan`.** Adapt the existing `pcl2scan` wrapper to the verified OAK-D topic, publish `/scan`, transform into the chosen planar/z-up robot frame and tune height, field-of-view, range and scan-time parameters from recorded data. Run the converter on the Xavier beside the camera to avoid sending the raw point cloud over the robot network.
6. **Configure ROS 1 `slam_toolbox`.** Pin the Melodic-compatible package/version, use consistent `scan_topic`, `map_frame`, `odom_frame` and `base_frame` values, and save both a standard occupancy map for M4 and a serialized pose graph. Start with asynchronous online mapping on the Xavier, then confirm CPU, memory and dropped-scan behaviour on the robot.
7. **Controlled comparison.** With no mapper running, record `/scan`, `/odom`, `/tf` and `/tf_static` so the bag contains no mapper-generated `map` → `odom` transform. Replay the identical bag into clean `slam_toolbox` and `gmapping` sessions with `use_sim_time` and `rosbag play --clock`; reset all map state between trials and keep the scan/TF inputs fixed. Keep `gmapping` as the baseline/fallback. Use `hector_slam` only as an optional odometry-independent diagnostic, not as the definition of M3 completion.

### Acceptance criteria

Use three independently recorded runs around a taped 2 × 2 m square, starting and ending at the same marked pose. Each run must last at least 90 s, include stationary, straight-line and turning segments, and keep walking speed at or below 0.3 m/s.

- PointCloud2 and `/scan` each deliver at least 95% of the messages expected from their measured source rate in every run, with no output gap longer than 0.5 s.
- Nano/Xavier clock offset is at most 20 ms before each run; at least 99% of scans transform at their own timestamp, with no continuous TF failure longer than 0.5 s.
- Raw odometry uses the correct axes/signs and returns within 1.0 m and 20° of the marked start pose after the 8 m loop in at least two of three runs.
- `slam_toolbox` closes the loop; the start/end `map` → robot-base pose delta is at most 0.25 m and 10° in at least two of three runs.
- The occupancy map and serialized pose graph both save, survive a clean mapper restart and reload successfully.
- During each run, sample `tegrastats` at 1 Hz. Xavier RAM usage stays below 7 GB, and the run-average CPU — calculated by averaging utilisation across online cores at each sample, then across all samples — stays below 80%. Instrument mapper-input counters: scans lost to TF/message-filter rejection or queue overflow stay below 5% of `/scan` messages, excluding configured throttling, minimum-time and minimum-motion filtering. Report the same measurements for `gmapping`.

---

## M4 — Point-to-point navigation

**Goal:** click a goal in rviz, robot walks there avoiding obstacles.

**Approach:** `move_base` + `AMCL` on the M3 map.

### What's needed

- **`/cmd_vel` → `/high_cmd` translator.** `move_base` outputs `geometry_msgs/Twist`; write a small ROS node (new package under `packages/`) to convert it to `/high_cmd` at 50 Hz with `mode=2, gaitType=1`.
- **Costmap config.** Go1 footprint ~0.60 × 0.35 m. Inflation radius ~0.35 m. Disable `rotate_recovery` (gait switching is janky); use `clear_costmap_recovery` only.
- **Velocity cap.** Start with `max_vel_x: 0.3 m/s`; odometry drift gets bad above that.

---

## M5 — Evaluation

**What to measure:**

- SLAM accuracy: drive a known loop (tape a 2×2 m square), compare map closure error.
- Localisation drift: AMCL covariance growth over 5 min stationary.
- Navigation success rate: 20 goals, fixed environment, report success / partial / fail.
- Compute load: `tegrastats` during SLAM and navigation.
