# PIXLOOP — Operational Bringup Guide

**Updated:** 27 October 2026 · ROS 2 Humble · NUVO IPC

> Physical CAN transmission and movement require an operator present, a clear test area and accessible hardware E-stop. Never start two command publishers. The FULL teleop is not a safety-rated controller.

## 1. Workspaces and environment (no alias)

| Workspace | Purpose |
|---|---|
| `/data/workspaces/pixloop_ros2_ws` | RoboSense/ZED drivers and external dependencies |
| `/data/workspaces/pixloop_mapping_ws` | KISS-ICP/mapping dependencies |
| `/data/workspaces/pixloop-sensor-ws` | Main sensors, IMU, localization, chassis, planning and bringup |

**Normal ROS 2 terminals:**

```bash
cd /data/workspaces/pixloop-sensor-ws
source /opt/ros/humble/setup.bash
source /data/workspaces/pixloop_ros2_ws/install/setup.bash
source /data/workspaces/pixloop_mapping_ws/install/setup.bash
source /data/workspaces/pixloop-sensor-ws/install/setup.bash
export ROS_DOMAIN_ID=42
export ROS_LOCALHOST_ONLY=0
export RMW_IMPLEMENTATION=rmw_fastrtps_cpp
```

**Important:** The current integrated bringup and Route B use **FastDDS** (`rmw_fastrtps_cpp`). Older guide examples used CycloneDDS; keep the host settings consistent with the running Route B bridge. Source the environment anew in each SSH terminal.

## 2. Integrated physical stack (recommended)

```bash
cd /data/workspaces/pixloop-sensor-ws
./scripts/pixloop_up.sh
```

Current orchestration performed by this script:

```text
pixloop_up.sh
  ├── pixloop_can_up.sh      → EMUC + SocketCAN + can_receiver + energy_monitor
  ├── pixloop_route_b_up.sh  → Docker + ROS1 master + ros1_bridge + socketcan_bridge
  └── online_bringup.launch.py
      ├── RoboSense + LiDAR bridge
      ├── Static sensor TF
      ├── KISS-ICP
      ├── ZED
      ├── EKF (existing default profile)
      └── CHCNAV IMU node
```

The integrated bringup **does not start** `keyboard_teleop_full`; start it separately only during an authorized test. The IMU driver may be running without publishing if the NUVO is not connected to the CHCNAV network. Do **not** duplicate the CAN receiver, Energy Monitor or Route B manually when the script has already launched them.

To start only sensors/localization when CAN and Route B are running separately:

```bash
ros2 launch pixloop_bringup online_bringup.launch.py
```

To omit a disconnected IMU:

```bash
ros2 launch pixloop_bringup online_bringup.launch.py enable_imu:=false
```

## 3. Individual physical sensors (alternative to integrated bringup)

Use the full **normal ROS 2 environment from section 1 in EACH new terminal**. Keep each `ros2 launch` terminal open.

### Terminal S1 — RoboSense LiDAR

```bash
ros2 launch pixloop_lidar rslidar_driver.launch.py
```

Output: `/rslidar_points`. If the launch is not installed in the selected overlay, inspect `ros2 pkg prefix pixloop_lidar` and installed launch files; the physical driver also exists in `rslidar_sdk`.

### Terminal S2 — LiDAR bridge

```bash
ros2 launch pixloop_lidar lidar_launch.py
```

Output: `/pixloop/lidar/points`.

### Terminal S3 — Static TF

```bash
ros2 launch pixloop_tf sensor_tf_launch.py
```

Frames include `base_link → rslidar`, `base_link → zed_camera_link` and provisional `base_link → imu_link` (IMU mounting extrinsics need physical calibration).

### Terminal S4 — KISS-ICP

```bash
ros2 launch pixloop_slam online_odometry.launch.py
```

Output: `/pixloop/lidar/odom`.

### Terminal S5 — ZED

```bash
ros2 launch pixloop_zed zed_driver.launch.py
```

Outputs: `/zed/zed_node/rgb/color/rect/image`, `/zed/zed_node/point_cloud/cloud_registered`, `/zed/zed_node/odom`.

### Terminal S6 — EKF

```bash
ros2 launch pixloop_localization state_estimation.launch.py
```

Output: `/pixloop/state_estimation/odom`. Current baseline is LiDAR + ZED. A separate IMU-enabled configuration has been added at `src/pixloop_localization/config/ekf_lidar_zed_imu.yaml`; do not assume this profile is automatically selected. Inspect launch arguments and configuration before enabling it:

```bash
ros2 launch pixloop_localization state_estimation.launch.py --show-args
```

### Terminal S7 — CHCNAV IMU

Connect the NUVO to the CHCNAV network `GNSS-3556506` (credentials kept separately). Device TCP endpoint: `192.168.200.1:9904`.

```bash
ping -c 3 192.168.200.1
ros2 launch pixloop_imu imu.launch.py
```

Output: `/imu/data_raw` (approximately 10 Hz when connected). Verify in a separate ROS 2 terminal:

```bash
ros2 topic hz /imu/data_raw
ros2 topic echo /imu/data_raw --once
```

The CHCNAV node can start without TCP connectivity; a running process alone does not prove valid IMU measurements. Yaw-rate EKF fusion and IMU frame orientation still require physical validation.

## 4. CAN RX + Energy — individual terminals

**Alternative shortcut (handles the full CAN RX/Energy setup):**

```bash
cd /data/workspaces/pixloop-sensor-ws
bash scripts/pixloop_can_up.sh
```

Use the following **manual** path when you want to inspect each stage separately. Do not run both approaches at once.

### Terminal C1 — EMUC daemon

```bash
ls -l /dev/ttyACM*
sudo modprobe emuc2socketcan
sudo systemctl stop ModemManager
sudo /home/dc/Downloads/Linux/emucd_64 \
  -s7 -e0 /dev/ttyACM0 emuccan0 emuccan1
```

Keep open. Skip starting a second daemon if an instance is already active.

### Terminal C2 — SocketCAN

```bash
sudo ip link set emuccan0 txqueuelen 1000
sudo ip link set emuccan1 txqueuelen 1000
sudo ip link set emuccan0 up
sudo ip link set emuccan1 up
ip -details link show emuccan0
timeout 2s candump emuccan0
```

### Terminal C3 — CAN receiver

Load the normal ROS 2 environment (section 1), then:

```bash
ros2 run pixloop_chassis can_receiver
```

Output: `/pixloop/chassis/can/raw`. Typical previously observed traffic: ~325 Hz.

### Terminal C4 — Energy Monitor

Load the normal ROS 2 environment (section 1), then:

```bash
ros2 run pixloop_chassis energy_monitor
```

Outputs: `/pixloop/energy/bms/raw`, `/pixloop/energy/bms/status`, `/pixloop/energy/battery_state`.

```bash
ros2 topic echo /pixloop/energy/battery_state --once
```

BMS `0x512`, `byte[4]` corresponds to SOC percent. `BatteryState.percentage` ranges 0.0–1.0 (e.g., `0.95` = 95%). Voltage and current are still provisional/NaN. Both C3 and C4 are **RX-only**.

## 5. Route B — ROS 2 → ROS 1 → SocketCAN

**Recommended independent start (already validated):**

```bash
cd /data/workspaces/pixloop-sensor-ws
bash scripts/pixloop_route_b_up.sh
```

This starts/checks container `pixloop-route-b-test`, ROS1 master, `dynamic_bridge` and `topic_to_socketcan_node` for `emuccan0`. It **does not start a motion controller**.

For troubleshooting, below is the **manual alternative**, rather than running it alongside the script.

### Terminal R1 — Docker container + ROS1 master

```bash
sudo docker start pixloop-route-b-test
sudo docker exec -it pixloop-route-b-test bash --noprofile --norc
```

Inside container:

```bash
source /opt/ros/noetic/setup.bash
export ROS_MASTER_URI=http://localhost:11311
roscore
```

### Terminal R2 — ROS1↔ROS2 bridge

```bash
sudo docker exec -u 1000:1000 -e HOME=/tmp -it \
  pixloop-route-b-test bash --noprofile --norc
```

Inside container:

```bash
source /opt/ros/noetic/setup.bash
source /humble_ws/install/setup.bash
source /can_msgs_ws/install/local_setup.bash
source /bridge_ws/install/local_setup.bash
export ROS_MASTER_URI=http://localhost:11311
export ROS_DOMAIN_ID=42
export ROS_LOCALHOST_ONLY=0
export RMW_IMPLEMENTATION=rmw_fastrtps_cpp
ros2 run ros1_bridge dynamic_bridge -- --bridge-all-2to1-topics
```

### Terminal R3 — ROS1 to SocketCAN

```bash
sudo docker exec -it pixloop-route-b-test bash --noprofile --norc
```

Inside container:

```bash
source /opt/ros/noetic/setup.bash
export ROS_MASTER_URI=http://localhost:11311
rosrun socketcan_bridge topic_to_socketcan_node \
  _can_device:=emuccan0 \
  sent_messages:=/pixloop/chassis/can/tx_test
```

### Terminal R4 — ROS2 host verification

Load the normal ROS 2 environment (section 1), then:

```bash
ros2 topic info /pixloop/chassis/can/tx_test --verbose
sudo docker ps --filter name=pixloop-route-b-test
```

Route B carries **TX commands** from ROS2 through ROS1 to `emuccan0`. It must not be confused with CAN RX or Energy.

## 6. FULL keyboard teleop — separate physical-control terminal

**Not part of automatic integrated bringup.** After CAN RX/Route B verification, with an operator present, controlled area, E-stop ready and no other command publisher:

```bash
cd /data/workspaces/pixloop-sensor-ws
source /opt/ros/humble/setup.bash
source /data/workspaces/pixloop_ros2_ws/install/setup.bash
source /data/workspaces/pixloop_mapping_ws/install/setup.bash
source /data/workspaces/pixloop-sensor-ws/install/setup.bash
export ROS_DOMAIN_ID=42
export ROS_LOCALHOST_ONLY=0
export RMW_IMPLEMENTATION=rmw_fastrtps_cpp

ros2 run pixloop_chassis keyboard_teleop_full \
  --ros-args \
  -p physical_enable:=true \
  -p operator_ready:=true
```

| Key | Action |
|---|---|
| `E` then `W` | Confirm Drive, then forward |
| `R` then `S` | Confirm Reverse, then backward |
| `A` / `D` / `C` | Steer left/right/center |
| `SPACE` / `X` | Controlled stop |
| `P` | Request Neutral + Park |
| `Q` | Stop + Park + quit |

Drive ↔ Reverse selection was validated without restarting ROS2; the vehicle must be stationary and the VCU must confirm gear (CAN `0x503`). Speed target 0.20 m/s; steering ±20°; keyboard watchdog 0.50 s. Keyboard-over-SSH is **not** a safety-rated dead-man. This is a physical command path: do not use it as a harmless connectivity test. The old keyboard probe executables have been removed; `keyboard_teleop_full` is the maintained entry point.

## 7. Verification (read-only)

In a new terminal with the normal ROS 2 environment:

```bash
ros2 node list
ros2 topic list -t
ros2 topic hz /pixloop/chassis/can/raw
ros2 topic echo /pixloop/energy/battery_state --once
ros2 topic hz /pixloop/lidar/points
ros2 topic hz /pixloop/lidar/odom
ros2 topic hz /zed/zed_node/odom
ros2 topic hz /pixloop/state_estimation/odom
ros2 topic hz /imu/data_raw
ros2 topic info /pixloop/chassis/can/tx_test --verbose
```

IMU will not publish unless its physical network is reachable. Also verify physical reports before driving:

```bash
timeout 5s candump emuccan0,503:7FF
timeout 5s candump emuccan0,504:7FF
timeout 5s candump emuccan0,506:7FF
```

## 8. Mapping / data recording

In a normal ROS 2 terminal:

```bash
ros2 launch pixloop_data_collection data_collection_launch.py \
  profile:=mapping \
  output_dir:=/home/dc/pixloop_bags/2026-10-08 \
  bag_name:=parking_north_oct08
```

Change output directory and bag name for each run.

## 9. Mapping localization + Nav2 planner-only

This pipeline computes and visualizes paths; it **does not command the physical chassis automatically**.

```text
PCD map → NDT → Map Server → Global Costmap → Planner Server → Path → RViz
```

In a normal ROS 2 terminal:

```bash
./scripts/pixloop_navigate.sh \
  pcd:=/home/dc/pixloop_maps/parking_base_link/map.pcd \
  map:=/home/dc/pixloop_maps/parking_base_link/nav2.yaml
```

Interfaces: `/goal_pose`, `/compute_path_to_pose`, `/pixloop/planning/path`, `/pixloop/planning/status`.

## 10. Optional wheel-speed decoder

Requires CAN RX, normal ROS 2 environment:

```bash
ros2 run pixloop_chassis wheel_speed_decoder_node
```

Source-run alternative if the console entry is unavailable:

```bash
PYTHONPATH=src/pixloop_chassis:$PYTHONPATH \
  python3 -m pixloop_chassis.wheel_speed_decoder_node
```

## Quick reference

| Mode | Main command | Avoid duplication |
|---|---|---|
| Complete physical bringup | `./scripts/pixloop_up.sh` | CAN/Energy/Route B/sensors already started |
| CAN + Energy only | `bash scripts/pixloop_can_up.sh` | No separate CAN RX or Energy processes |
| Route B only | `bash scripts/pixloop_route_b_up.sh` | Do not launch bridge manually again |
| Sensor stack only | `ros2 launch pixloop_bringup online_bringup.launch.py` | Do not run with complete bringup simultaneously |
| IMU only | `ros2 launch pixloop_imu imu.launch.py` | CHCNAV network required |
| Physical keyboard teleop | `ros2 run pixloop_chassis keyboard_teleop_full --ros-args -p physical_enable:=true -p operator_ready:=true` | Only one physical CAN TX controller |

**Close of session:** CAN RX + Energy, Route B, RoboSense, ZED, LiDAR odometry, EKF baseline and IMU driver were brought into the integrated stack. IMU-to-EKF fusion profile was added but requires calibration/validation; `keyboard_teleop_full` supports forward/reverse gear changes with VCU confirmation. Physical braking performance has not been fully characterized.
