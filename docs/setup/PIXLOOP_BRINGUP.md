# Pixloop — Operational Bringup Guide

Quick reference for bringing up the current Pixloop physical ROS 2 stack.

> Physical CAN transmission and vehicle motion require an operator, a clear test area and an available E-stop.

---

## 1. Workspaces

Pixloop currently uses three ROS 2 workspaces.

| Workspace | Purpose |
|---|---|
| `/data/workspaces/pixloop_ros2_ws` | Physical drivers and external ROS 2 dependencies |
| `/data/workspaces/pixloop_mapping_ws` | LiDAR odometry / KISS-ICP mapping dependencies |
| `/data/workspaces/pixloop-sensor-ws` | Main Pixloop workspace: sensors, chassis, localization, planning, bringup and operational scripts |

Main Pixloop packages include:

```text
pixloop_bringup
pixloop_chassis
pixloop_data_collection
pixloop_lidar
pixloop_localization
pixloop_planning
pixloop_slam
pixloop_tf
pixloop_zed
```

### Standard ROS 2 environment

Use this environment for sensors, localization, mapping and planning:

```bash
source /opt/ros/humble/setup.bash
source /data/workspaces/pixloop_ros2_ws/install/setup.bash
source /data/workspaces/pixloop_mapping_ws/install/setup.bash
source /data/workspaces/pixloop-sensor-ws/install/setup.bash

export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
```

Optional shortcut:

```bash
alias px='source /opt/ros/humble/setup.bash && \
source /data/workspaces/pixloop_ros2_ws/install/setup.bash && \
source /data/workspaces/pixloop_mapping_ws/install/setup.bash && \
source /data/workspaces/pixloop-sensor-ws/install/setup.bash && \
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp'
```

Then:

```bash
px
```

---

# 2. Individual Physical Stack

Use separate terminals when validating components individually.

## Terminal 1 — RoboSense LiDAR Driver

```bash
px

ros2 launch \
  pixloop_lidar \
  rslidar_driver.launch.py
```

Raw output:

```text
/rslidar_points
```

---

## Terminal 2 — Pixloop LiDAR Bridge

```bash
px

ros2 launch \
  pixloop_lidar \
  lidar_launch.py
```

Main normalized topic:

```text
/pixloop/lidar/points
```

---

## Terminal 3 — Static Sensor TF

```bash
px

ros2 launch \
  pixloop_tf \
  sensor_tf_launch.py
```

---

## Terminal 4 — KISS-ICP LiDAR Odometry

```bash
px

ros2 launch \
  pixloop_slam \
  online_odometry.launch.py
```

Main output:

```text
/pixloop/lidar/odom
```

---

## Terminal 5 — ZED Physical Driver

```bash
px

ros2 launch \
  pixloop_zed \
  zed_driver.launch.py
```

Main physical topics:

```text
/zed/zed_node/rgb/color/rect/image
/zed/zed_node/point_cloud/cloud_registered
/zed/zed_node/odom
```

---

## Terminal 6 — EKF State Estimation

KISS-ICP + ZED fusion:

```bash
px

ros2 launch \
  pixloop_localization \
  state_estimation.launch.py
```

---

## Terminal 7 — Chassis CAN RX

First verify that SocketCAN exists:

```bash
ip link show emuccan0
```

Then:

```bash
px

ros2 run \
  pixloop_chassis \
  can_receiver
```

Output:

```text
/pixloop/chassis/can/raw
```

Verify:

```bash
ros2 topic hz /pixloop/chassis/can/raw
```

Expected physical traffic is approximately:

```text
325–330 Hz
```

---

# 3. Integrated Physical Bringup

Instead of starting the previous terminals individually:

```bash
cd /data/workspaces/pixloop-sensor-ws

px

./scripts/pixloop_up.sh
```

Useful checks:

```bash
ros2 node list
ros2 topic list
```

or:

```bash
./scripts/pixloop_check.sh
```

The integrated stack is intended to bring up the normal physical sensor and local-state pipeline.

---

# 4. Mapping

## Record Mapping Data

```bash
px

ros2 launch \
  pixloop_data_collection \
  data_collection_launch.py \
  profile:=mapping \
  output_dir:=/home/dc/pixloop_bags/2026-10-08 \
  bag_name:=parking_north_oct08
```

Change:

```text
output_dir
bag_name
```

for each experiment.

---

# 5. Localization + Path Planning

The current navigation stack is planner-only:

```text
PCD Map
   ↓
NDT Localization
   ↓
Map Server
   ↓
Global Costmap
   ↓
Planner Server
   ↓
Planned Path
   ↓
RViz
```

It does not automatically command the physical vehicle.

## Start navigation

```bash
cd /data/workspaces/pixloop-sensor-ws

px

./scripts/pixloop_navigate.sh \
  pcd:=/home/dc/pixloop_maps/parking_base_link/map.pcd \
  map:=/home/dc/pixloop_maps/parking_base_link/nav2.yaml
```

For another map, only change:

```text
pcd:=.../map.pcd
map:=.../nav2.yaml
```

Example:

```bash
./scripts/pixloop_navigate.sh \
  pcd:=/home/dc/pixloop_maps/lab_oct08/map.pcd \
  map:=/home/dc/pixloop_maps/lab_oct08/nav2.yaml
```

Expected nodes include:

```text
/ndt_localizer
/map_server
/planner_server
/lifecycle_manager_planning
/pixloop_goal_bridge
/pixloop_rviz
```

Check:

```bash
ros2 node list | grep -E \
'ndt|map_server|planner_server|lifecycle|goal_bridge|rviz'
```

Planning interface:

```text
Goal:
/goal_pose

Action:
/compute_path_to_pose

Path:
/pixloop/planning/path

Status:
/pixloop/planning/status
```

---

# 6. Wheel-Speed Decoder

Requires chassis CAN RX to already be active.

```bash
cd /data/workspaces/pixloop-sensor-ws

px

PYTHONPATH=src/pixloop_chassis:$PYTHONPATH \
python3 \
  -m pixloop_chassis.wheel_speed_decoder_node
```

---

# 7. Physical CAN Interface

The PIXLOOP chassis currently uses:

```text
Innodisk EMUC-B202
/dev/ttyACM0
emuccan0
```

## Terminal CAN-1 — Start EMUC daemon

Check hardware:

```bash
ls -l /dev/ttyACM*
```

Load the driver:

```bash
sudo modprobe emuc2socketcan
sudo systemctl stop ModemManager
```

Start EMUC:

```bash
sudo /home/dc/Downloads/Linux/emucd_64 \
  -s7 \
  -e0 \
  /dev/ttyACM0 \
  emuccan0 \
  emuccan1
```

Keep this terminal open.

---

## Terminal CAN-2 — Enable SocketCAN

```bash
sudo ip link set emuccan0 txqueuelen 1000
sudo ip link set emuccan1 txqueuelen 1000

sudo ip link set emuccan0 up
sudo ip link set emuccan1 up
```

Verify:

```bash
ip -details link show emuccan0
```

Check physical traffic:

```bash
timeout 2s candump emuccan0
```

The active chassis bus is:

```text
emuccan0
```

---

# 8. Chassis Route B — Containers

Physical command architecture:

```text
ROS 2 Humble
      ↓
ros1_bridge
      ↓
ROS 1 Noetic
      ↓
socketcan_bridge
      ↓
emuccan0
      ↓
VCU
```

Route B uses the persistent container:

```text
pixloop-route-b-test
```

Start it:

```bash
sudo docker start pixloop-route-b-test
```

Verify:

```bash
sudo docker ps \
  --filter name=pixloop-route-b-test
```

---

## Chassis Terminal A — ROS 1 Master

```bash
sudo docker exec \
  -it \
  pixloop-route-b-test \
  bash \
  --noprofile \
  --norc
```

Inside:

```bash
source /opt/ros/noetic/setup.bash

export ROS_MASTER_URI=http://localhost:11311

roscore
```

Keep open.

---

## Chassis Terminal B — ROS 1 ↔ ROS 2 Bridge

```bash
sudo docker exec \
  -u 1000:1000 \
  -e HOME=/tmp \
  -it \
  pixloop-route-b-test \
  bash \
  --noprofile \
  --norc
```

Inside:

```bash
source /opt/ros/noetic/setup.bash
source /humble_ws/install/setup.bash
source /can_msgs_ws/install/local_setup.bash
source /bridge_ws/install/local_setup.bash

export ROS_MASTER_URI=http://localhost:11311

export ROS_DOMAIN_ID=42
export ROS_LOCALHOST_ONLY=0
export RMW_IMPLEMENTATION=rmw_fastrtps_cpp
```

Start:

```bash
ros2 run \
  ros1_bridge \
  dynamic_bridge \
  -- \
  --bridge-all-2to1-topics
```

Keep open.

---

## Chassis Terminal C — ROS 1 → SocketCAN

```bash
sudo docker exec \
  -it \
  pixloop-route-b-test \
  bash \
  --noprofile \
  --norc
```

Inside:

```bash
source /opt/ros/noetic/setup.bash

export ROS_MASTER_URI=http://localhost:11311
```

Start:

```bash
rosrun \
  socketcan_bridge \
  topic_to_socketcan_node \
  _can_device:=emuccan0 \
  sent_messages:=/pixloop/chassis/can/tx_test
```

Keep open.

---

## Chassis Terminal D — Host ROS 2

This terminal intentionally uses FastDDS instead of the normal CycloneDDS environment.

```bash
cd /data/workspaces/pixloop-sensor-ws

source /opt/ros/humble/setup.bash
source install/setup.bash

export ROS_DOMAIN_ID=42
export ROS_LOCALHOST_ONLY=0
export RMW_IMPLEMENTATION=rmw_fastrtps_cpp
```

Verify the Route B endpoint:

```bash
ros2 topic info \
  /pixloop/chassis/can/tx_test \
  --verbose
```

---

# 9. Physical Keyboard Control

Only after Route B is running and the vehicle test area is ready:

```bash
cd /data/workspaces/pixloop-sensor-ws

source /opt/ros/humble/setup.bash
source install/setup.bash

export ROS_DOMAIN_ID=42
export ROS_LOCALHOST_ONLY=0
export RMW_IMPLEMENTATION=rmw_fastrtps_cpp

PYTHONPATH=src/pixloop_chassis:$PYTHONPATH \
python3 \
  -m pixloop_chassis.keyboard_motion_probe \
  --ros-args \
  -p physical_enable:=true \
  -p operator_ready:=true \
  2>&1 | tee /tmp/teleop_debug.log
```

Do not simultaneously run another physical CAN TX path.

---

# 10. Energy / BMS Monitoring

Energy monitoring is RX-only.

It uses:

```text
CAN ID 0x512
```

Start the ROS 2 CAN receiver first:

```bash
px

ros2 run \
  pixloop_chassis \
  can_receiver
```

Then in another terminal:

```bash
px

ros2 run \
  pixloop_chassis \
  energy_monitor
```

Published topics:

```text
/pixloop/energy/bms/raw
/pixloop/energy/bms/status
/pixloop/energy/battery_state
```

Verify:

```bash
ros2 topic list | grep '/pixloop/energy'
```

Battery state:

```bash
ros2 topic echo \
  /pixloop/energy/battery_state \
  --once
```

Current validated SOC interpretation:

```text
SOC [%] = CAN 0x512 byte[4]
```

`BatteryState.percentage` uses:

```text
0.0 ... 1.0
```

Example:

```text
0.86 = 86 %
```

Voltage and current decoding remain provisional until independently validated.

---

# Quick Start

## Normal physical stack

```bash
cd /data/workspaces/pixloop-sensor-ws
px
./scripts/pixloop_up.sh
```

## Navigation

```bash
px

./scripts/pixloop_navigate.sh \
  pcd:=/path/to/map.pcd \
  map:=/path/to/nav2.yaml
```

## CAN RX

```bash
px

ros2 run pixloop_chassis can_receiver
```

## Energy Monitor

```bash
px

ros2 run pixloop_chassis energy_monitor
```

## Route B

```text
1. Start EMUC daemon
2. Bring emuccan0 up
3. Start pixloop-route-b-test
4. Terminal A → roscore
5. Terminal B → ros1_bridge
6. Terminal C → socketcan_bridge
7. Terminal D → ROS 2 FastDDS endpoint
8. Only then → physical command application
```

---

# Key Topics

```text
LiDAR raw
/rslidar_points

LiDAR normalized
/pixloop/lidar/points

LiDAR odometry
/pixloop/lidar/odom

ZED RGB
/zed/zed_node/rgb/color/rect/image

ZED point cloud
/zed/zed_node/point_cloud/cloud_registered

ZED odometry
/zed/zed_node/odom

Chassis RX
/pixloop/chassis/can/raw

Chassis Route B TX
/pixloop/chassis/can/tx_test

Battery
/pixloop/energy/battery_state

Goal
/goal_pose

Planned path
/pixloop/planning/path

Planning status
/pixloop/planning/status
