# Session 01 — Introduction to Pixloop

## October 3, 2026

### SSH · Git/GitHub · ROS 2 · CARLA · Sensors · Localization · Planning

**Platform:** Pixloop / PIX-KIT-HOOKE  
**Duration:** 3–4 hours  
**Format:** Collaborative work  
**Level:** Technical onboarding

---

# 1. Session Goal

This session introduces the complete Pixloop development environment.

The objective is **not** to modify autonomy algorithms yet. The team should learn how to:

- connect to Pixloop through SSH;
- load the ROS 2 environment;
- work with Git/GitHub using feature branches;
- start and explore CARLA;
- discover ROS 2 nodes, topics and message types;
- identify the main physical sensors;
- recognize the localization and planning pipeline;
- collect evidence and open a Pull Request.

By the end of the session, students should understand the general flow:

```text
Sensors
  ↓
LiDAR Odometry
  ↓
State Estimation
  ↓
Localization
  ↓
Path Planning
  ↓
Tracking / Control
  ↓
Vehicle
```

> Physical vehicle motion is **not** a Session 01 student task unless explicitly authorized by the instructor.

---

# 2. Safety and Engineering Rules

## Physical vehicle

Do not send physical motion commands unless explicitly authorized.

For any physical test:

- keep the test area clear;
- keep the emergency stop available;
- verify the control mode;
- do not run multiple CAN TX paths simultaneously;
- stop immediately if the reported state or vehicle behavior is unexpected.

## Before publishing to an unknown ROS 2 topic

Always follow this sequence:

```text
Find topic
   ↓
Inspect topic type
   ↓
Inspect message structure
   ↓
Inspect publishers/subscribers
   ↓
Publish only when understood
```

Useful commands:

```bash
ros2 node list
ros2 node info <NODE>

ros2 topic list
ros2 topic info <TOPIC> --verbose
ros2 topic echo <TOPIC> --once
ros2 topic hz <TOPIC>

ros2 interface show <MESSAGE_TYPE>
```

---

# 3. Pixloop Environment

## Connect through SSH

Check connectivity:

```bash
ping <PIXLOOP_IP>
```

Connect:

```bash
ssh dc@<PIXLOOP_IP>
```

Validated Pixloop host:

```text
dc-Nuvo-6108GC
```

Inspect the machine:

```bash
hostname
whoami
pwd
uname -a
lsb_release -a
```

---

## Load ROS 2

Validated workspaces:

```text
/data/workspaces/pixloop_ros2_ws
/data/workspaces/pixloop_mapping_ws
/data/workspaces/pixloop-sensor-ws
```

Source them in this order:

```bash
source /opt/ros/humble/setup.bash
source /data/workspaces/pixloop_ros2_ws/install/setup.bash
source /data/workspaces/pixloop_mapping_ws/install/setup.bash
source /data/workspaces/pixloop-sensor-ws/install/setup.bash
```

Set the ROS runtime environment:

```bash
export ROS_DOMAIN_ID=42
export ROS_LOCALHOST_ONLY=0
export RMW_IMPLEMENTATION=rmw_fastrtps_cpp
```

Verify:

```bash
printenv ROS_DISTRO
echo "$ROS_DOMAIN_ID"
echo "$RMW_IMPLEMENTATION"
```

Expected:

```text
ROS_DISTRO=humble
ROS_DOMAIN_ID=42
RMW_IMPLEMENTATION=rmw_fastrtps_cpp
```

Search available packages:

```bash
ros2 pkg list | grep -Ei \
"pix|nav2|kiss|local|slam|planning|chassis"
```

---

# 4. Git and GitHub Workflow

Course repository:

```text
git@github.com:DCaicrag/pixloop-cyberphysical-labs.git
```

Clone once:

```bash
cd ~

git clone \
git@github.com:DCaicrag/pixloop-cyberphysical-labs.git

cd ~/pixloop-cyberphysical-labs
```

If already cloned:

```bash
cd ~/pixloop-cyberphysical-labs
```

Inspect:

```bash
git status
git branch -a
git log --oneline --decorate -10
git remote -v
```

Update `main`:

```bash
git switch main
git pull origin main
```

Create the team branch:

```bash
git switch -c feature/team-a/session01
```

Use the corresponding team name:

```text
feature/team-a/session01
feature/team-b/session01
feature/team-c/session01
```

Do **not** push student work directly to `main`.

Create the results area:

```bash
mkdir -p students/team-a/session01/evidence
touch students/team-a/session01/results.md
```

Open with:

```bash
code students/team-a/session01/results.md
```

or:

```bash
nano students/team-a/session01/results.md
```

---

# 5. CARLA Simulation

CARLA is the safe environment for the first vehicle-control experiment.

The intended workflow is:

```text
Start CARLA
   ↓
Start ROS bridge
   ↓
Inspect ROS graph
   ↓
Identify sensors
   ↓
Identify control interface
   ↓
Send short movement command
   ↓
STOP
```

## Start CARLA

The installation path must be confirmed on the workstation:

```bash
cd <TO_CONFIRM_CARLA_PATH>
```

Inspect:

```bash
pwd
ls
```

Typical executable:

```text
CarlaUE4.sh
```

Start CARLA:

```bash
<TO_CONFIRM_CARLA_START_COMMAND>
```

Record:

```text
CARLA version:
Map:
Server:
Port:
```

---

## Start the CARLA ROS 2 bridge

Open another terminal:

```bash
source /opt/ros/humble/setup.bash
source <TO_CONFIRM_CARLA_ROS_WS>/install/setup.bash
```

Start the bridge:

```bash
<TO_CONFIRM_CARLA_ROS_BRIDGE_COMMAND>
```

Keep this terminal open.

---

## Discover the simulated ROS graph

```bash
ros2 node list
```

```bash
ros2 topic list | grep -i carla
```

Search simulated sensors:

```bash
ros2 topic list | grep -Ei "camera|image"
ros2 topic list | grep -Ei "lidar|point|scan"
ros2 topic list | grep -Ei "imu"
ros2 topic list | grep -Ei "odom|odometry"
```

Search vehicle-control topics:

```bash
ros2 topic list | grep -Ei \
"control|cmd|vehicle|ego|throttle|steer"
```

Inspect a possible control topic:

```bash
ros2 topic info <CARLA_CONTROL_TOPIC> --verbose
```

Inspect the message:

```bash
ros2 interface show <CARLA_CONTROL_MESSAGE_TYPE>
```

Identify fields equivalent to:

```text
throttle
steering
brake
reverse
hand_brake
```

---

## CARLA movement test

> Simulation only.

Prepare the STOP command first:

```bash
<TO_CONFIRM_CARLA_STOP_COMMAND>
```

Send one short low-speed forward command:

```bash
<TO_CONFIRM_CARLA_FORWARD_COMMAND>
```

Immediately stop:

```bash
<TO_CONFIRM_CARLA_STOP_COMMAND>
```

Confirm visually that the vehicle stops.

---

# 6. Physical Pixloop Sensor Stack

The validated physical stack includes:

```text
RoboSense
ZED
LiDAR bridge
KISS-ICP
EKF
Chassis RX
```

The complete bringup is:

```bash
ros2 launch \
pixloop_bringup \
online_bringup.launch.py
```

For a reduced sensor/odometry bringup:

```bash
ros2 launch \
pixloop_bringup \
online_bringup.launch.py \
enable_lidar_driver:=true \
enable_zed_driver:=true \
enable_chassis:=true \
enable_lidar_odometry:=true \
enable_tf:=false \
enable_ekf:=false
```

---

## RoboSense LiDAR

Validated sensor:

```text
Model: RS-Helios-16P
LiDAR IP: 192.168.1.200
IPC IP: 192.168.1.102/24
```

Pipeline:

```text
RoboSense
   ↓
/rslidar_points
   ↓
LiDAR bridge
   ↓
/pixloop/lidar/points
   ↓
KISS-ICP
   ↓
/pixloop/lidar/odom
```

Main topics:

```text
/rslidar_points
/pixloop/lidar/points
/pixloop/lidar/odom
```

Observed LiDAR frequency:

```text
~10 Hz
```

Inspect:

```bash
ros2 topic info /rslidar_points --verbose
ros2 topic info /pixloop/lidar/points --verbose
ros2 topic info /pixloop/lidar/odom --verbose
```

Measure:

```bash
ros2 topic hz /rslidar_points
ros2 topic hz /pixloop/lidar/points
ros2 topic hz /pixloop/lidar/odom
```

Read one odometry message:

```bash
ros2 topic echo /pixloop/lidar/odom --once
```

---

## ZED Camera

Validated camera:

```text
Model: Stereolabs ZED
Serial: 13709
Camera ID: 0
Device: /dev/video0
```

Physical topics:

```text
/zed/zed_node/rgb/color/rect/image
/zed/zed_node/point_cloud/cloud_registered
/zed/zed_node/odom
```

Pixloop bridge topics:

```text
/pixloop/zed/rgb/image
/pixloop/zed/points
/pixloop/zed/odom
```

Observed rates:

```text
RGB:         ~29–30 Hz
Point cloud: ~10 Hz
Odometry:    ~30 Hz
```

Inspect:

```bash
ros2 topic info /zed/zed_node/odom --verbose
ros2 topic hz /zed/zed_node/odom
ros2 topic echo /zed/zed_node/odom --once
```

---

## Chassis RX

Validated adapter:

```text
Innodisk EMUC-B202 USB Dual CAN
```

Active interface:

```text
emuccan0
```

RX-only pipeline:

```text
emuccan0
   ↓
pixloop_chassis/can_receiver.py
   ↓
/pixloop/chassis/can/raw
```

Observed traffic:

```text
~325–330 frames/s
```

Inspect:

```bash
ros2 topic info \
/pixloop/chassis/can/raw \
--verbose
```

Measure:

```bash
ros2 topic hz \
/pixloop/chassis/can/raw
```

Read one frame:

```bash
ros2 topic echo \
/pixloop/chassis/can/raw \
--once
```

---

# 7. Localization and Planning

The current navigation stack is **planner-only**.

It generates a global path but does not command physical motion.

Architecture:

```text
PCD Map
   +
LiDAR
   ↓
NDT Localization
   ↓
map → odom
   ↓
Map Server
   ↓
Global Costmap
   ↓
Planner Server
   ↓
ComputePathToPose
   ↓
/pixloop/planning/path
   ↓
RViz
```

Local state estimation is conceptually:

```text
KISS-ICP
   +
ZED
   ↓
EKF
   ↓
odom → base_link
```

Expected TF chain:

```text
map
 ↓
odom
 ↓
base_link
 ├── rslidar
 └── zed_camera_link
```

---

## Start navigation

Preferred wrapper:

```bash
cd /data/workspaces/pixloop-sensor-ws
```

Source:

```bash
source /opt/ros/humble/setup.bash
source /data/workspaces/pixloop_ros2_ws/install/setup.bash
source /data/workspaces/pixloop_mapping_ws/install/setup.bash
source /data/workspaces/pixloop-sensor-ws/install/setup.bash
```

Using a generated map directory:

```bash
./scripts/pixloop_navigate.sh \
map_dir:=/path/to/RUN_001/maps
```

Using explicit map files:

```bash
./scripts/pixloop_navigate.sh \
pcd:=/home/dc/pixloop_maps/parking/parking_map.pcd \
map:=/home/dc/pixloop_maps/parking/parking_nav2.yaml
```

Equivalent direct launch:

```bash
ros2 launch \
pixloop_planning \
navigation.launch.py \
map_dir:=/path/to/RUN_001/maps
```

---

## Main navigation nodes

Expected nodes:

```text
/ndt_localizer
/map_server
/planner_server
/lifecycle_manager_planning
/pixloop_goal_bridge
/pixloop_rviz
```

Verify:

```bash
ros2 node list | grep -E \
'ndt|map_server|planner_server|lifecycle|goal_bridge|rviz'
```

Check lifecycle:

```bash
ros2 lifecycle get /map_server
ros2 lifecycle get /planner_server
```

Expected:

```text
active
```

---

## Planning interfaces

Important topics:

```text
/map
/global_costmap/costmap
/pixloop/lidar/points
/pixloop/planning/vehicle
/pixloop/planning/path
/pixloop/planning/status
```

Goal topic:

```text
/goal_pose
```

Planning action:

```text
/compute_path_to_pose
```

Planned path:

```text
/pixloop/planning/path
```

---

## RViz planning test

In RViz:

1. verify that the map is visible;
2. verify that LiDAR data is visible;
3. verify that the vehicle/localization marker is visible;
4. select **2D Goal Pose**;
5. select a target point;
6. choose the desired final orientation;
7. wait for the planner;
8. verify that a non-empty path appears.

Inspect the generated path:

```bash
ros2 topic echo \
/pixloop/planning/path \
--once
```

Inspect status:

```bash
ros2 topic echo \
/pixloop/planning/status
```

Check the action:

```bash
ros2 action list | grep compute_path
```

Expected:

```text
/compute_path_to_pose
```

The current planner does **not** launch:

```text
controller_server
bt_navigator
velocity_smoother
cmd_vel
physical CAN TX
autonomous vehicle motion
```

The output of Session 01 is:

```text
Goal
 ↓
Planner
 ↓
Path
 ↓
RViz
```

not:

```text
Goal
 ↓
Planner
 ↓
Controller
 ↓
Physical Vehicle
```

---

# 8. Discovery Task and Evidence

The team should identify:

```text
LiDAR raw topic
LiDAR normalized topic
LiDAR odometry topic

ZED image topic
ZED point-cloud topic
ZED odometry topic

Chassis CAN RX topic

NDT localization node
Map server
Planner server

Goal topic
Planned-path topic
```

For relevant interfaces record:

```text
Topic
Message type
Publisher
Subscriber
QoS
Frequency
Frame ID
```

Useful commands:

```bash
ros2 node list
ros2 topic list

ros2 topic info <TOPIC> --verbose
ros2 interface show <MESSAGE_TYPE>
ros2 topic echo <TOPIC> --once
ros2 topic hz <TOPIC>
```

---

## Save evidence

Go to the course repository:

```bash
cd ~/pixloop-cyberphysical-labs
```

Create the evidence folder:

```bash
mkdir -p \
students/team-a/session01/evidence
```

Save nodes:

```bash
ros2 node list \
> students/team-a/session01/evidence/nodes.txt
```

Save topics:

```bash
ros2 topic list \
> students/team-a/session01/evidence/topics.txt
```

Save ROS environment:

```bash
env | grep -E 'ROS|RMW' \
> students/team-a/session01/evidence/ros_environment.txt
```

Save system information:

```bash
uname -a \
> students/team-a/session01/evidence/system.txt

lsb_release -a \
>> students/team-a/session01/evidence/system.txt \
2>&1
```

---

# 9. `results.md`

Recommended structure:

```markdown
# Session 01 Results

## Team

- Student 1:
- Student 2:
- Student 3:
- Student 4:
- Student 5:

## Host and ROS 2

Hostname:

Ubuntu:

ROS_DISTRO:

ROS_DOMAIN_ID:

RMW:

## Git

Branch:

Commit:

Pull Request:

## CARLA

Version:

Map:

Vehicle role:

Control topic:

Control message:

Forward test:

STOP test:

## Physical Pixloop

LiDAR raw topic:

LiDAR normalized topic:

LiDAR odometry:

ZED image topic:

ZED point-cloud topic:

ZED odometry:

Chassis RX topic:

## Localization

NDT node:

Localization pose topic:

Localization odometry topic:

## Planning

Map server:

Planner server:

Goal topic:

ComputePath action:

Path topic:

Planning status topic:

Path successfully generated:

## Observations

## Problems Found
```

---

# 10. Submission

Inspect changes:

```bash
cd ~/pixloop-cyberphysical-labs

git status
git diff
```

Add the Session 01 work:

```bash
git add \
students/team-a/session01
```

Commit:

```bash
git commit -m \
"docs: add team-a session01 results"
```

Push:

```bash
git push -u \
origin \
feature/team-a/session01
```

Open a Pull Request from:

```text
feature/team-a/session01
```

into:

```text
main
```

Do not merge the Pull Request until it has been reviewed.

---

# Session Completion Criteria

Session 01 is complete when the team can demonstrate:

```text
[ ] SSH connection to Pixloop
[ ] ROS 2 environment loaded
[ ] team Git branch created
[ ] CARLA started or inspected
[ ] CARLA ROS graph inspected
[ ] simulated control interface identified
[ ] forward / STOP test completed in CARLA
[ ] physical Pixloop stack inspected
[ ] RoboSense topics identified
[ ] LiDAR odometry identified
[ ] ZED interfaces identified
[ ] chassis RX topic identified
[ ] localization nodes identified
[ ] planning nodes identified
[ ] path-planning interface identified
[ ] evidence saved
[ ] results.md completed
[ ] commit created
[ ] branch pushed
[ ] Pull Request opened
```

---

# Validated Environment Reference

```text
Host:
dc-Nuvo-6108GC

OS:
Ubuntu 22.04

ROS 2:
Humble

ROS_DOMAIN_ID:
42

RMW:
rmw_fastrtps_cpp
```

Workspaces:

```text
/data/workspaces/pixloop_ros2_ws
/data/workspaces/pixloop_mapping_ws
/data/workspaces/pixloop-sensor-ws
```

LiDAR:

```text
RoboSense RS-Helios-16P

/rslidar_points
/pixloop/lidar/points
/pixloop/lidar/odom
```

ZED:

```text
/zed/zed_node/rgb/color/rect/image
/zed/zed_node/point_cloud/cloud_registered
/zed/zed_node/odom
```

Chassis:

```text
Innodisk EMUC-B202
emuccan0
/pixloop/chassis/can/raw
```

Planning:

```text
/ndt_localizer
/map_server
/planner_server
/lifecycle_manager_planning
/pixloop_goal_bridge

/goal_pose
/compute_path_to_pose
/pixloop/planning/path
/pixloop/planning/status
```

Repository:

```text
git@github.com:DCaicrag/pixloop-cyberphysical-labs.git
```

---

# Next Session

## Session 02 — October 10, 2026

The next session will focus on:

```text
ROS 2 graph
Publishers
Subscribers
TF
Frames
RViz
Foxglove
Sensor visualization
```

The objective will be to move from:

```text
"The components exist"
```

to:

```text
"I understand how the components are connected."
```
