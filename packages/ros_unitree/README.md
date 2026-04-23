# ros_unitree

Unitree Go1 ROS packages, vendored at a known-good state for ROS Melodic on Ubuntu 18.04 ARM64.

## Contents

| Package | Purpose |
|---|---|
| `unitree_legged_msgs` | ROS message definitions — `HighCmd`, `HighState`, `LowCmd`, `LowState` |
| `unitree_legged_real` | Hardware driver — publishes `/high_state`, subscribes `/high_cmd`, talks to the Pi relay over UDP |
| `unitree_guide` | Unitree's gait controller (reference only — not launched in our stack) |
| `pcl2scan` | Converts `PointCloud2` → `LaserScan` for use with gmapping (needed for M3 SLAM) |
| `unitree_ros` | Launch files and URDF for the Go1 (reference, not actively used) |
| `mapping3d_realsense` | 3D mapping with RealSense (not applicable — we use OAK-D, kept for reference) |

## Usage

These packages are symlinked into `~/catkin_ws/src/ros_unitree` on both boards. Edit here, build in `~/catkin_ws`:

```bash
cd ~/catkin_ws && catkin_make -j2
```

## Notes

- This is ROS **Melodic** (not Noetic). Ignore any Noetic references in sub-package files.
- `unitree_legged_real` requires `~/K1/` (Unitree SDK) to be present on `.14`. Do not move `~/K1/`.
- Upstream source: https://github.com/unitreerobotics/unitree_ros
