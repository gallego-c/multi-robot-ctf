# ctf_navigation — Branch: robot_real

This branch contains the complete adaptation of the **Capture the Flag (CTF)** project for deployment on **two physical TurtleBot3 Waffle Pi robots**.

The main objective is to bring the simulation's algorithmic components—collaborative SLAM, Voronoi frontier exploration, computer vision, and mutual obstacle avoidance—into a real-world environment. This involves addressing challenges such as odometry drift, clock synchronization, network bandwidth limitations, and sensor processing on resource-constrained hardware.

The state-machine flow (Exploration → Search → Capture → Pursuit) remains identical to the simulation version, while navigation, vision, and transform parameters have been carefully tuned for physical deployment.

---

## Changes and Adaptations for Real-World Deployment

The following adjustments have been introduced into the software stack to support physical robots:

1. **Network Management and Clock Synchronization (NTP)**
   The `clock_drift_check.py` script has been added to check synchronization between the main PC and the robots' Raspberry Pi clocks. In multi-robot environments, clock offsets greater than 50 ms can cause TF transform failures.

2. **Publishing Kinematic Transforms**
   Unlike Gazebo, where the TF hierarchy can be injected easily, the physical robots use `odom_tf_broadcaster.py` to maintain continuous publication of the `odom → base_footprint → base_link → base_scan` transforms.

3. **Vision and Processing (ArUco + HSV)**
   Color detection on physical robots can be strongly affected by changes in lighting. Enhanced vision support (`aruco_flag_detector.cpp` and the parameters in `vision_real.yaml`) improves robustness through both HSV segmentation for red and fiducial marker recognition.

4. **LiDAR Noise Filtering**
   When the robots operate close to each other, each scanner can detect the other robot, leaving persistent false obstacles in the SLAM map. The footprint filter parameters in `laser_obstacle_filter.py` have been adjusted to exclude the other robot's physical volume without introducing latency.

---

## Architecture and Retained Files

The autonomous control logic and main nodes retain the structure of the simulation version:

- **Launch Files:**
  - `launch/real_multi_robot_ctf.launch`: Main game launcher for two physical robots.
  - `launch/real_robot_flag_vision.launch`: Vision-only debugging mode for testing cameras and color segmentation on site before a game.
  - `launch/real_robot_tf.launch`: Coordinate and static transform setup.

- **Configuration and Parameters (Tuned for Physical Hardware):**
  - `params/*_slam.yaml`
  - `params/gmapping_robot1.yaml` and `params/gmapping_robot2.yaml`
  - `params/vision_real.yaml`

- **Control, Debugging, and Coordination Scripts:**
  - `scripts/slam_frontier_explorer_ctf.py`: Main game control logic.
  - `scripts/robot_coordinator.py`: Costmap-level yielding logic.
  - `scripts/map_merge_debug.py` and `scripts/tf_map_merge.py`: Diagnostics for checking whether collaborative SLAM correctly aligns the two robots' maps.
  - `scripts/launch_config_echo.py`: Utility for printing configuration settings at runtime.

- **Advanced Vision Nodes:**
  - `src/nodes/flag_detector_node.cpp`
  - `src/vision/aruco_flag_detector.cpp`

---

## Running the System

Running a complex multi-robot system on physical hardware requires careful preparation before launching the main scripts.

### Step 1: Prepare the Robots

1. Power on both TurtleBot3 Waffle Pi robots and connect them to the same Wi-Fi network as the ROS master PC (`ROS_MASTER_URI`).
2. Synchronize their clocks using *chrony* or *ntpdate*:
   ```bash
   sudo ntpdate -u <MASTER_PC_IP>
   ```
3. Run the basic bringup on each robot to connect to the OpenCR board, motors, and LiDAR:
   ```bash
   # On robot 1:
   ROS_NAMESPACE=robot1 roslaunch turtlebot3_bringup turtlebot3_robot.launch multi_robot_name:=robot1

   # On robot 2:
   ROS_NAMESPACE=robot2 roslaunch turtlebot3_bringup turtlebot3_robot.launch multi_robot_name:=robot2
   ```
4. Start the camera nodes (`cv_camera` or a similar driver) on each robot so that video streams are available on `/robot1/camera/image_raw` and `/robot2/camera/image_raw`.

### Step 2: Run Preflight Checks (Strongly Recommended)

Before starting autonomous exploration, verify that vision works correctly under the current lighting conditions:

```bash
roslaunch ctf_navigation real_robot_flag_vision.launch
```

*Inspect the debug topics in `rqt_image_view` to confirm that the HSV mask clearly detects the flag's color.*

### Step 3: Launch the CTF Game

Once you have verified both robots' telemetry (`/scan`, `/odom`, and camera streams), launch the main orchestrator from the PC:

```bash
roslaunch ctf_navigation real_multi_robot_ctf.launch
```

**Optional parameter:** If the robot bringup already publishes the basic kinematic chain (`base_footprint → base_link → base_scan`) as static transforms, avoid TF tree conflicts by launching with:

```bash
roslaunch ctf_navigation real_multi_robot_ctf.launch publish_kinematic_tf:=false
```

---

## Troubleshooting Physical Deployment

- **`map_merge` does not align the maps:** This usually occurs because the initial odometry estimates (`/robot1/odom` and `/robot2/odom`) have drifted significantly since startup, or because a correct `initial_pose` estimate was not provided. If the error persists, stop the robots, align them on the floor, and restart bringup.
- **Jerky Motion and Signal Loss:** If the robots move in bursts, the Wi-Fi network may be saturated by full camera data and uncompressed point clouds. Use `image_transport/compressed` whenever possible.
- **The Robots Collide Head-On:** Ensure that `robot_coordinator.py` publishes correctly to the `other_robot_cloud` topic and that the DWA `obstacle_layer` parameters are configured to read this topic with high priority.
