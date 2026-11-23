# RoboSAPIENS-CrossModalAnomalyDetection: INTO-CPS Case Study for RGB–LiDAR Cross-Modal Anomaly Detection

**RGB–LiDAR Cross-Modal Anomaly Detection Using Monocular Depth Estimation**

This INTO-CPS case-study repository presents **CrossModalAnomalyDetection**, an open-source ROS 2 Jazzy tool for monitoring geometric consistency between planar LiDAR measurements and monocular depth estimates derived from RGB images. The tool was developed within the [RoboSAPIENS](https://robosapiens-eu.tech/) research project and is made available through the [INTO-CPS Association](https://github.com/INTO-CPS-Association).

The implementation was developed and evaluated using a TurtleBot 4 equipped with an OAK-D Pro RGB camera and a 2D LiDAR. Sensor topics, camera calibration, and TF frames are configurable for other ROS 2 platforms. The pipeline comprises RGB rectification and border cleaning, relative monocular depth estimation, LiDAR-to-camera projection, cross-modal consistency monitoring, and a motion/anomaly status image for RViz. An optional fault-injection component publishes modified *copies* of RGB images and LiDAR scans for controlled testing without changing the original sensor streams.

---

## Relation to INTO-CPS

This repository is part of the INTO-CPS ecosystem and was developed within the **RoboSAPIENS** project (*Robotic Safe Adaptation In Unprecedented Situations*).

It supports trustworthy and dependable cyber-physical systems through runtime monitoring of robotic sensing and perception. By assessing cross-modal consistency between LiDAR measurements and monocular depth estimates from RGB images, the tool helps identify potentially unreliable sensor measurements and perception inconsistencies that may affect autonomous operation.

---

## Relationship to the research implementation

This INTO-CPS case-study presentation is based on the original **CrossModalAnomalyDetection** research implementation. The ROS 2 packages, node entry points, launch scripts, parameter names, and execution commands below are intentionally unchanged. The adaptations concern the README and repository presentation within the RoboSAPIENS and INTO-CPS tool collection.

---

## Method Overview

The pipeline rectifies and cleans the incoming RGB images, estimates relative monocular depth using Depth Anything V2, and projects planar LiDAR measurements into the camera image. It then monitors consistency between the LiDAR-derived measurements and the RGB-derived depth estimates, publishing anomaly information and a status visualization for RViz. An optional fault-injection workflow supports controlled evaluation using separate modified sensor streams.

---

## Repository Structure

```text
robosapiens-CrossModalAnomalyDetection/
├── src/
│   ├── rgb_preprocessing/             # ROS image rectification + border cleaner
│   ├── depth_estimation/              # Depth Anything V2 adapter + node
│   ├── lidar_projection/              # camera/LiDAR geometry, expected points (C++)
│   ├── cross_modal_anomaly_detection/ # geometric/irregularity/coverage checks
│   ├── motion_status_rviz/            # status image for RViz
│   └── sensor_fault_simulation/       # OPTIONAL injector and GUI controller
├── scripts/                           # launch wrappers; ROS topic overrides accepted
├── tests/                             # source/static and controller regression tests
├── LICENSE                            # Apache License 2.0
└── README.md
```

Build products (`build/`, `install/`, `log/`), trained model weights, rosbag recordings, and previous experimental versions are not included in the source distribution. Keep generated files out of version control when publishing the repository.

## Requirements

- **ROS 2 Jazzy** on Ubuntu (or a compatible Jazzy installation), `colcon`, and a functioning ROS graph.
- Sensor publishers for RGB `sensor_msgs/Image`, `sensor_msgs/CameraInfo`, planar `sensor_msgs/LaserScan`, odometry `nav_msgs/Odometry`, and TF (`/tf`, `/tf_static`).
- OpenCV, NumPy, `cv_bridge`, `image_proc`, `image_geometry`, `laser_geometry`, and ROS 2 TF packages. The fault GUI additionally needs Tkinter and an interactive desktop.
- **Depth Anything V2** source code and the **Small (ViT-S) relative-depth** checkpoint. Download these separately; checkpoints are not distributed here. A compatible `torch` installation is also required.

Example ROS-dependent packages on Ubuntu:

```bash
sudo apt update
sudo apt install -y python3-colcon-common-extensions python3-opencv python3-numpy python3-tk \
  ros-jazzy-image-proc ros-jazzy-cv-bridge ros-jazzy-vision-msgs \
  ros-jazzy-sensor-msgs-py ros-jazzy-image-geometry ros-jazzy-laser-geometry \
  ros-jazzy-tf2-geometry-msgs
```

### Clone and build

Clone the repository into a ROS 2 workspace of your choice, or use the repository as its own standalone `colcon` workspace:

```bash
# Clone the INTO-CPS Association repository:
git clone https://github.com/INTO-CPS-Association/robosapiens-CrossModalAnomalyDetection.git
cd robosapiens-CrossModalAnomalyDetection

# Ensure launch wrappers are executable (especially after browser-based uploads).
chmod +x scripts/*.sh

# Source a compatible ROS 2 Jazzy installation.
source /opt/ros/jazzy/setup.bash
# For a custom Jazzy source build, source that installation instead.

colcon build --symlink-install
source install/local_setup.bash
ros2 pkg prefix cross_modal_anomaly_detection
ros2 pkg prefix rgb_preprocessing
```

In each new terminal, activate your ROS 2 / Python environment before running a stage. The launch scripts source this project's local overlay after the initial build. Keep the repository's `src/` and `scripts/` directories intact; do not commit generated `build/`, `install/`, or `log/` directories.

### Depth Anything V2 setup

Clone the [Depth Anything V2 implementation](https://github.com/DepthAnything/Depth-Anything-V2), install the compatible Python dependencies, and obtain its relative-depth ViT-S checkpoint from the upstream instructions. One possible layout is:

```text
~/models/Depth-Anything-V2/
├── depth_anything_v2/
└── checkpoints/
    └── depth_anything_v2_vits.pth
```

Make the source package importable in the **same Python environment** as ROS 2:

```bash
export PYTHONPATH="$HOME/models/Depth-Anything-V2:${PYTHONPATH:-}"
python3 -c 'from depth_anything_v2.dpt import DepthAnythingV2; print("Depth Anything V2 import OK")'
```

Pass `-p checkpoint_dir:="$HOME/models/Depth-Anything-V2/checkpoints"` to the depth launcher. The inference node does not automatically download weights. If you already have a functional ROS 2/PyTorch virtual environment, reuse it; do not install a second conflicting interpreter.

## Check TurtleBot 4 sensor data before starting

The examples below assume these topics and TF frames; adjust them to match your driver:

| Data | Example topic or frame |
|---|---|
| OAK-D raw RGB | `/oakd/rgb/preview/image_raw` |
| RGB camera calibration | `/oakd/rgb/preview/camera_info` |
| LiDAR scan | `/scan` |
| Odometry | `/odom` |
| Camera optical frame | `oakd_rgb_camera_optical_frame` |
| LiDAR frame | `rplidar_link` |

```bash
ros2 topic list | grep -E '(oakd|scan|odom|tf)'
ros2 topic info /scan -v
ros2 topic info /oakd/rgb/preview/camera_info -v
ros2 run tf2_ros tf2_echo oakd_rgb_camera_optical_frame rplidar_link
```

The camera-to-LiDAR TF and camera intrinsics must be valid. The example projector configuration uses a 2.5° yaw correction derived from the original experimental platform; review this setting for another robot.

## Run with real sensor measurements (no simulation)

Start the robot's camera, LiDAR, odometry and TF publishers first. Open **five terminals**, with `cd ~/robosapiens-CrossModalAnomalyDetection` and your ROS/Python environment activated in each. Run the following stages in order, one per terminal.

**1. Rectify and clean RGB** (both nodes in one launch):

```bash
./scripts/run_rgb_preprocessing.sh \
  rectify:=true \
  raw_image:=/oakd/rgb/preview/image_raw \
  camera_info:=/oakd/rgb/preview/camera_info \
  rect_image:=/oakd/rgb/preview/image_rect \
  clean_image:=/oakd/rgb/preview/image_rect_clean \
  valid_mask:=/rgb_rectification/valid_mask \
  edge_guard_px:=8
```

If your camera *already publishes a correctly rectified image*, set `rectify:=false` and point `rect_image:=/your/rectified/image` to that stream, to avoid rectifying twice.

**2. Relative monocular depth:**

```bash
./scripts/run_depth_estimation.sh \
  -p input_topic:=/oakd/rgb/preview/image_rect_clean \
  -p output_topic:=/depth_anything/depth \
  -p checkpoint_dir:="$HOME/models/Depth-Anything-V2/checkpoints"
```

**3. LiDAR projection and expected-depth points:**

```bash
./scripts/run_lidar_projection.sh \
  -p image_topic:=/oakd/rgb/preview/image_rect_clean \
  -p scan_topic:=/scan \
  -p camera_info_topic:=/oakd/rgb/preview/camera_info \
  -p relative_depth_topic:=/depth_anything/depth \
  -p expected_max_residual:=0.5 \
  -p show_rejected_expected_overlay:=false
```

The projector publishes `/cross_modal/lidar_projected_points` and `/cross_modal/lidar_expected_projection`; defaults are in `src/lidar_projection/config/projector.yaml`.

**4. Cross-modal anomaly detection:**

```bash
./scripts/run_anomaly_detection.sh \
  -p projected_points_topic:=/cross_modal/lidar_projected_points \
  -p estimated_points_topic:=/cross_modal/lidar_expected_projection \
  -p depth_topic:=/depth_anything/depth \
  -p return_coverage_scan_topic:=/scan \
  -p valid_mask_topic:=/rgb_rectification/valid_mask \
  -p odom_topic:=/odom
```

Outputs: `/cross_modal_anomaly/status` (`std_msgs/String`, JSON) and `/cross_modal_anomaly/anomaly` (`std_msgs/Bool`); the detector configuration lives in `src/cross_modal_anomaly_detection/config/detector.yaml`.

**5. Motion/anomaly status for RViz:**

```bash
./scripts/run_motion_status.sh \
  --topic /odom \
  --anomaly-topic /cross_modal_anomaly/status \
  --anomaly-type json \
  --linear-threshold 0.035 \
  --angular-threshold 0.10 \
  --stop-delay 0.75
```

In RViz, add an **Image** display with topic `/turtlebot/motion_status/image`. The fault simulator is **not needed** in this workflow.

## Run with optional simulated RGB/LiDAR faults

This mode modifies copies of sensor messages on `/faults/...` topics. **Keep navigation, control, and safety-related consumers on the original unmodified sensor topics.** For a repeatable experiment, start sensor publishers and RGB preprocessing as in stage 1 above, then launch the following nodes (separate terminals) in the shown order.

**A. Fault injector** (reads clean RGB and the *real* scan, publishes separate corrupted streams):

```bash
./scripts/run_fault_injector.sh \
  -p rgb_in:=/oakd/rgb/preview/image_rect_clean \
  -p scan_in:=/scan \
  -p rgb_out:=/faults/rgb/image_rect \
  -p scan_out:=/faults/scan \
  -p keys_topic:=/faults/keys
```

**B. Depth on faulted RGB:**

```bash
./scripts/run_depth_estimation.sh \
  -p input_topic:=/faults/rgb/image_rect \
  -p output_topic:=/depth_anything/depth \
  -p checkpoint_dir:="$HOME/models/Depth-Anything-V2/checkpoints"
```

**C. Project faulted RGB, faulted LiDAR and corresponding depth:**

```bash
./scripts/run_lidar_projection.sh \
  -p image_topic:=/faults/rgb/image_rect \
  -p scan_topic:=/faults/scan \
  -p camera_info_topic:=/oakd/rgb/preview/camera_info \
  -p relative_depth_topic:=/depth_anything/depth \
  -p expected_max_residual:=0.5 \
  -p show_rejected_expected_overlay:=false
```

**D. Anomaly detection using the *faulted* scan:**

```bash
./scripts/run_anomaly_detection.sh \
  -p projected_points_topic:=/cross_modal/lidar_projected_points \
  -p estimated_points_topic:=/cross_modal/lidar_expected_projection \
  -p depth_topic:=/depth_anything/depth \
  -p return_coverage_scan_topic:=/faults/scan \
  -p valid_mask_topic:=/rgb_rectification/valid_mask \
  -p odom_topic:=/odom
```

**E. RViz status overlay:**

```bash
./scripts/run_motion_status.sh --topic /odom \
  --anomaly-topic /cross_modal_anomaly/status --anomaly-type json \
  --linear-threshold 0.035 --angular-threshold 0.10 --stop-delay 0.75
```

**F. Keyboard/GUI fault controller** (optional; requires desktop GUI):

```bash
./scripts/run_fault_controller.sh -p keys_topic:=/faults/keys
```

**Hold-to-activate:** Press and hold a keyboard key (with the controller window focused) or a GUI button to keep its simulated fault active, then release to deactivate. The GUI also offers **Toggle GUI buttons instead of holding**, for persistent mouse-click activation. `Esc` or `Space` clears all faults. Different Linux display servers can handle keyboard auto-repeat differently; for verification or longer experiments prefer GUI toggle mode.

The controller publishes `/faults/keys` (`sensor_msgs/Joy`) at 20 Hz while open. The injector periodically publishes `/faults/status` and changes the output sensor stream for as long as its current command state remains active. The final detector can require multiple distinct LiDAR scans before it raises an alarm.

## Runtime topic and parameter overrides

- RGB preprocessing uses ROS launch arguments: `rectify:=false rect_image:=/my/rectified/image`.
- Other ROS nodes use `-p parameter:=value` appended to their `run_*.sh` wrapper; these override the installed YAML/script defaults.
- RViz status uses command-line flags, e.g. `--topic /robot/odom --anomaly-topic /cross_modal_anomaly/status`.

For example, on a robot using `/lidar/scan` instead of `/scan`:

```bash
./scripts/run_lidar_projection.sh -p scan_topic:=/lidar/scan
./scripts/run_anomaly_detection.sh -p return_coverage_scan_topic:=/lidar/scan
```

To monitor an alternative anomaly output, use `-p status_topic:=/your/status` for the detector and `--anomaly-topic /your/status` for RViz.

### Selected detector parameters

The default research profile (`src/cross_modal_anomaly_detection/config/detector.yaml`) includes:

| Parameter | Default |
|---|---:|
| Spatial irregularity threshold | `0.20` |
| Minimum distinct abnormal measurements | `8` |
| Spatial irregularity neighborhood | `7` samples |
| Irregularity regions / overlap | `7` / `0.20` |
| Temporal voting | `12` of `20` unique scans |
| Weak-evidence confirmation | `3` of `4` |
| Expected-point residual gate | `0.5` |
| RGB edge guard | `8` pixels |

Settings are empirically tuned for the experimental setup. This implementation only detects *potential inconsistencies*, not the fault's cause or a safety-certified failure.

## Diagnostics

Check the published streams and detector status:

```bash
ros2 topic hz /oakd/rgb/preview/image_rect_clean
ros2 topic hz /depth_anything/depth
ros2 topic hz /cross_modal/lidar_projected_points
ros2 topic hz /cross_modal/lidar_expected_projection
ros2 topic echo /cross_modal_anomaly/status --once
ros2 topic echo /cross_modal_anomaly/anomaly --once
```

For optional simulation, check the controller and injector state:

```bash
ros2 topic echo /faults/keys --once
ros2 topic echo /faults/status --once
ros2 topic info /faults/keys --verbose
```

If the controller or detector does not behave as expected, verify that each node subscribes to the intended sensor topics and that sensor timestamps, camera calibration, TF transforms, and ROS 2 discovery settings are valid. For sustained GUI-controlled faults, use the hold or optional toggle behavior described above.

## Tests

The repository includes ROS-independent source and UI-state regression checks:

```bash
python3 -m unittest discover -s tests -v
bash -n scripts/*.sh
```

These checks cover Python module syntax, package manifests, entry points, overlay bootstrap, default detector configuration, and GUI hold/release state handling. A full ROS 2 `colcon` build, C++ compilation, live topic synchronization, and end-to-end detection must be verified on your robot or ROS-equipped development machine. For normal operation, confirm all expected topics are active and the network's ROS domain / DDS discovery settings match across machines.

## License

This project is released under the **Apache License 2.0**. See [LICENSE](LICENSE) for the full license text. The external dataset is subject to its own usage and redistribution terms.

**Third-party components:** Depth Anything V2 and its model checkpoints are obtained separately from the [upstream project](https://github.com/DepthAnything/Depth-Anything-V2) and remain subject to their respective licensing and usage terms. ROS 2 and other dependencies are likewise governed by their own licenses. No model weights are redistributed in this repository.

## Acknowledgment

This work was developed in the context of **RoboSAPIENS — Robotic Safe Adaptation In unprecedented Situations**, funded by the European Union's **Horizon Europe** research and innovation programme under **grant agreement No. 101133807**. The project includes Aristotle University of Thessaloniki (AUTH) as a partner.

For more information, see the [RoboSAPIENS project](https://robosapiens-eu.tech/) and the [INTO-CPS Association on GitHub](https://github.com/INTO-CPS-Association).

## Citation

When using this tool in a publication, please reference the [RoboSAPIENS-CrossModalAnomalyDetection repository](https://github.com/INTO-CPS-Association/robosapiens-CrossModalAnomalyDetection) and identify the release or commit used.
