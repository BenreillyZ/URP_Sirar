# URP_Sirar — UR Robot Hand-Eye Calibration

This ROS2 workspace provides a complete **hand-eye calibration pipeline** for UR robotic arms (tested with UR10e) equipped with Intel RealSense cameras. It computes a fixed transformation matrix from the robot's end-effector to the camera using ArUco marker-based pose estimation.

## Overview

A fixed ArUco marker (ID 27, DICT_ARUCO_ORIGINAL) is placed in the robot's workspace. The robot is manually moved to ~20 different poses while keeping the marker visible to the camera. At each pose, the marker's 6-DOF pose is estimated via OpenCV's PnP solver, and the robot's joint state is recorded through TF. After all samples are collected, [easy_handeye2](https://github.com/marcoesposito1988/easy_handeye2) computes the end-effector-to-camera transformation matrix using Tsai-Lenz's algorithm.

## Repository Structure

```
URP_Sirar/
├── .gitignore
├── README.md
└── src/
    ├── easy_handeye2/            # git submodule — upstream calibration library
    └── aruco_detector/           # Custom ArUco detection ROS2 package
        ├── package.xml
        ├── setup.py
        ├── aruco_detector/
        │   └── detect_vis_node.py    # ArUco detection + pose estimation node
        └── launch/
            └── aruco_detect.launch.py
```

## Prerequisites

- **ROS2 Humble** (Ubuntu 22.04)
- **Intel RealSense SDK** + `realsense2_camera` ROS2 package
- **OpenCV** with ArUco module (`python3-opencv`)
- **UR Robot Driver** publishing TF frames

## Installation

```bash
# Clone the repository with submodules
git clone --recurse-submodules https://github.com/BenreillyZ/URP_Sirar.git
cd URP_Sirar

# Install ROS2 dependencies
rosdep install -iyr --from-paths src

# Build
colcon build
source install/setup.bash
```

## Usage

### Step 1 — Start RealSense Camera

Launch your RealSense camera node (with aligned depth enabled):

```bash
ros2 launch realsense2_camera rs_launch.py align_depth.enable:=true
```

### Step 2 — Start ArUco Detection

```bash
# Using ros2 run:
ros2 run aruco_detector detect_vis --ros-args \
  -p target_id:=27 \
  -p marker_length_m:=0.19 \
  -p publish_coordinate_convention:=optical \
  -p publish_tf_frame:=true \
  -p marker_frame_name:=aruco_tag

# Or using the launch file:
ros2 launch aruco_detector aruco_detect.launch.py target_id:=27
```

### Step 3 — Run Hand-Eye Calibration

```bash
source install/setup.bash
ros2 launch easy_handeye2 calibrate.launch.py \
  name:=ur10e_eye_in_hand \
  calibration_type:=eye_in_hand \
  robot_base_frame:=siraRbase \
  robot_effector_frame:=siraRtool0_controller \
  tracking_base_frame:=realsense_color_optical_frame \
  tracking_marker_frame:=aruco_tag
```

### Step 4 — Collect Samples

1. In the rqt calibration window, manually move the UR robot to different poses (keep the ArUco marker visible)
2. Rotate the end-effector significantly (up to 90°) in each axis
3. Click **Take Sample** after each stable pose
4. Collect 15–20 samples covering various orientations
5. Click **Compute** then **Save**

### Step 5 — Publish Calibration Result

```bash
ros2 launch easy_handeye2 publish.launch.py name:=ur10e_eye_in_hand
```

The calibration result is saved to `~/.ros2/easy_handeye2/calibrations/ur10e_eye_in_hand.calib`.

## ArUco Detector Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `target_id` | `27` | ArUco marker ID to detect |
| `marker_length_m` | `0.19` | Physical marker side length (meters) |
| `pose_method` | `aruco_pnp` | `aruco_pnp` (recommended) or `depth_center` |
| `publish_coordinate_convention` | `optical` | `optical` or `camera_link` |
| `publish_tf_frame` | `true` | Broadcast marker as TF frame |
| `marker_frame_name` | `aruco_tag` | TF frame name for the marker |
| `show_window` | `true` | Show OpenCV debug window |
| `publish_debug_image` | `true` | Publish debug image to `/aruco_tracker/debug_image` |

## ROS2 Topics

| Topic | Type | Description |
|-------|------|-------------|
| `/aruco_tracker/pose` | `geometry_msgs/PoseStamped` | Detected marker pose |
| `/aruco_tracker/debug_image` | `sensor_msgs/Image` | Annotated debug visualization |

## Acknowledgements

- **[easy_handeye2](https://github.com/marcoesposito1988/easy_handeye2)** by Marco Esposito — the core hand-eye calibration library (included as git submodule, LGPL v3 license)
- **[OpenCV ArUco](https://docs.opencv.org/4.x/d5/dae/tutorial_aruco_detection.html)** — marker detection and pose estimation

## References

Tsai, Roger Y., and Reimar K. Lenz. *"A new technique for fully autonomous and efficient 3D robotics hand/eye calibration."* Robotics and Automation, IEEE Transactions on 5.3 (1989): 345-358.
