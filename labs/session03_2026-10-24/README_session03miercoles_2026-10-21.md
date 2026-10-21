# Session 03 — Building and Connecting ROS 2 Nodes

**October 24, 2026 · Pixloop Cyber-Physical Labs**

## Goal

Each team will turn its current progress into a **ROS 2 interface**. Reuse Pixloop drivers and existing packages; start with recorded/synthetic data in WSL or CARLA when physical hardware is unavailable. **No vehicle actuation is required.**

## 1. ROS 2 vocabulary (one line each)

- **Workspace:** directory containing ROS 2 packages and build outputs.
- **Package:** organized ROS 2 code with configuration, dependencies and executable nodes.
- **Node:** program performing one specific task (IMU reader, object detector, energy decision).
- **Topic:** named channel carrying messages between nodes.
- **Publisher:** node that sends messages to a topic.
- **Subscriber:** node that receives messages from a topic.
- **Message:** defined data structure, such as `sensor_msgs/msg/Image` or `sensor_msgs/msg/Imu`.
- **Launch file:** starts one or more nodes with their parameters.
- **Parameter:** value configuring a node without changing its code.
- **TF / frame:** coordinate-frame transformations used to relate sensors, vehicle and map.
- **RViz:** ROS visualization tool for sensor data, frames, markers and paths.

```text
Sensor/Existing node → Topic → Subscriber/Your node → Output topic → RViz or next node
```

## 2. Inspect Pixloop before coding

On the **Nuvo**, load the normal ROS 2 environment:

```bash
source /opt/ros/humble/setup.bash
source /data/workspaces/pixloop_ros2_ws/install/setup.bash
source /data/workspaces/pixloop_mapping_ws/install/setup.bash
source /data/workspaces/pixloop-sensor-ws/install/setup.bash
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
```

```bash
cd /data/workspaces/pixloop-sensor-ws
ls src                                        # Existing packages
ros2 pkg list | grep pixloop                # Installed packages
ros2 node list                              # Running nodes
ros2 topic list -t                          # Topics and message types
ros2 topic info /TOPIC --verbose           # Publisher, subscriber and QoS
ros2 node info /NODE                        # Node interfaces
ros2 interface show MESSAGE_TYPE            # Message fields
ros2 topic echo /TOPIC --once              # Inspect one non-image message
```

**Note:** A topic may not appear until its driver/node is running. In WSL, use your **local** ROS 2 environment and test data; it is not automatically connected to Pixloop.

## 3. Team A — Localization & IMU

**Current task: discover the CHCNAV INS connection and build the missing ROS 2 IMU node.** Do not rebuild KISS-ICP or the EKF.

**What already exists**

- The CHCNAV INS has provided `$GPCHC` measurements via **TCP `192.168.200.1:9904`** over its Wi-Fi network (~10 Hz in the previous test). Its **Ethernet IP is not yet confirmed**.
- Pixloop packages to inspect: `pixloop_ins`, `pixloop_lidar`, `pixloop_slam`, `pixloop_localization`, `pixloop_tf` in `/data/workspaces/pixloop-sensor-ws/src/`.
- Existing LiDAR odometry: `/pixloop/lidar/odom`; existing state-estimation launch: `pixloop_localization/state_estimation.launch.py`.

**Steps**

1. **Find the INS:** inspect its network settings and physical Ethernet connection; establish which IP/interface reaches it without relying on manual Wi-Fi. **Do not change production network settings before documenting them.**
2. **Inspect existing code:** look for an INS/IMU reader in `pixloop_ins` or other packages **before** creating `chcnav_imu_node`.
3. **Build a first reader:** parse recorded `$GPCHC` messages, validate checksum/fields, confirm gyro and acceleration units/axes, and publish `sensor_msgs/msg/Imu` on `/imu/data_raw` (proposed topic).
4. **Test before fusion:** check frequency (~10 Hz), stillness, gravity, axes and timestamp/frame. Keep orientation **unavailable** until the heading-to-ROS conversion is validated; do not feed unverified measurements to the EKF.
5. **Define integration:** identify the existing EKF configuration and which IMU measurements it should subscribe to later (initial candidate: angular velocity Z).

**Useful inspection:**

```bash
find src -iname '*imu*' -o -iname '*ins*' -o -iname '*chcnav*'
ros2 topic list -t | grep -Ei 'imu|ins|odom|tf'
ros2 node list | grep -Ei 'local|ekf|slam|tf'
ip -br addr                              # Available network interfaces
ip route get 192.168.200.1            # Current route to INS Wi-Fi address
nc -vz -w 3 192.168.200.1 9904       # Only if connected to INS network
```

**Deliverable:** a tested IMU parser and, if ready, `/imu/data_raw` publisher; a short connection diagram (Wi-Fi now, proposed stable connection later); and the EKF input contract.

**Think next:** Which IMU fields are trustworthy? What are their units and `frame_id`? Does the current EKF need yaw rate, orientation, acceleration, or only some of them?

## 4. Team B — YOLO Object Detection & RViz

**Current task: migrate the working YOLO script to a ROS 2 node fed by the existing ZED driver, then visualize detections in RViz.**

**What already exists**

- Camera package: `pixloop_zed`; launch: `zed_driver.launch.py`.
- RGB topic: `/zed/zed_node/rgb/color/rect/image` (`sensor_msgs/msg/Image`); verify its name and QoS while the ZED node runs.
- The team's current YOLO Python script should be reused rather than rewriting inference.

**Steps**

1. **Subscribe to camera:** create `yolo_detector_node`, consume the ZED image topic, convert images with `cv_bridge`, and run the existing YOLO model. **Do not open the ZED camera directly from YOLO.**
2. **Publish results:** publish class, confidence and 2D bounding boxes using a clearly defined ROS message (for example `vision_msgs`, if available); document output topic, timestamp and camera `frame_id`.
3. **Make it visible in RViz:** publish an **annotated image** (`sensor_msgs/msg/Image`) and show it through an RViz Image display. For spatial markers, use `visualization_msgs/msg/MarkerArray` **only after** you have valid depth/3D coordinates and matching TF; a 2D box is not yet a 3D obstacle.
4. **Test without the car:** replay recorded image messages or remap a CARLA camera topic in WSL, run the detector, and confirm the output and RViz display.

**Useful inspection:**

```bash
ros2 launch pixloop_zed zed_driver.launch.py  # On Pixloop, if not running
ros2 topic list -t | grep -Ei 'zed|camera|image|detect'
ros2 topic info /zed/zed_node/rgb/color/rect/image --verbose
ros2 node info /zed/zed_node
rviz2
```

**Deliverable:** `camera topic → YOLO node → detections topic + annotated image visible in RViz`; a screenshot and the message contract.

**Think next:** Does planning need 2D image detections or 3D obstacle positions? How would you associate a detection with ZED depth and the vehicle's `base_link` frame?

## 5. Team C — Energy & Charging-Station Integration

**Current task: migrate the working SOC/station-choice script to ROS 2 and subscribe to the existing energy node.** Keep the synthetic map for now; **A* is not a prerequisite**.

**What already exists**

- `pixloop_chassis` already provides `energy_monitor` and `/pixloop/energy/battery_state` (`sensor_msgs/msg/BatteryState`).
- `BatteryState.percentage` is a fraction from **0.0 to 1.0** (e.g., `0.35` = 35%). Do **not** decode CAN again.
- Navigation later provides a path on `/pixloop/planning/path`; it is not required for the first decision test.

**Steps**

1. **Wrap current logic in a node:** create `energy_aware_decision_node` subscribing to `/pixloop/energy/battery_state` and call the existing station-selection function with its SOC.
2. **Retain synthetic stations:** store station coordinates and the requested destination in a small config; use a documented provisional distance/energy model.
3. **Publish a decision:** selected station/goal, reachable flag and reason on defined ROS 2 topic(s). Consider the case where **no station is reachable**.
4. **Test in WSL:** simulate different `BatteryState` messages and show that the same map produces different outcomes depending on SOC; later substitute actual route lengths from Nav2.

**Useful inspection:**

```bash
ros2 topic list -t | grep -Ei 'energy|battery|bms|path'
ros2 topic info /pixloop/energy/battery_state --verbose
ros2 interface show sensor_msgs/msg/BatteryState
ros2 topic echo /pixloop/energy/battery_state --once
```

On Pixloop, if they are not already active, the **CAN receiver must run before `energy_monitor`**; energy monitoring is receive-only.

**Deliverable:** `BatteryState → energy_aware_decision_node → chosen station/goal`, demonstrated with at least three SOC test cases.

**Think next:** How do real path distance, obstacles, energy consumption per meter and a safety margin change the decision? What should happen if the destination is feasible but no charging station is?

> **Model limitation:** SOC alone is **not** a measured driving range. Label energy-per-meter and safety-margin values as provisional until they have been calibrated.

## 6. End-of-session checkpoint

Each team should show **one simple demo**, its **input/output contract** (topic + type + producer/consumer), and answer:

- What already exists that we reused?
- What did we validate today, in WSL/CARLA or on Pixloop?
- What one interface or dependency blocks the next step?

**Target:** `CHCNAV → IMU`, `ZED → YOLO → RViz`, and `Battery SOC → charging-station decision` — independently testable nodes that can be connected later.
