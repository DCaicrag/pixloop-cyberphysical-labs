Session 01 — Introduction to Pixloop
October 3, 2026
SSH · Git/GitHub · ROS 2 · CARLA · Physical Sensors · Localization · Planning
Platform: Pixloop / PIX-KIT-HOOKE
Suggested duration: 3–4 hours
Format: Collaborative work
Level: Technical onboarding
1. Session Goal
Session 01 is a guided introduction to the complete Pixloop development environment.
The purpose is not to modify autonomy algorithms yet. The team should learn how to enter the system, bring up the validated software, inspect ROS 2 interfaces, use CARLA safely, identify the physical sensor pipeline, and recognize how localization and path planning fit into the complete autonomy stack.
By the end of the session, the team should be able to:
- connect to Pixloop through SSH;
- load the correct ROS 2 workspaces;
- use Git/GitHub through a branch-based workflow;
- start and explore CARLA;
- discover ROS 2 nodes, topics, message types, and QoS;
- identify the physical LiDAR, ZED, odometry, and chassis interfaces;
- observe the Pixloop physical sensor stack;
- start the existing localization/planner-only stack when a valid map is available;
- recognize the validated CAN Route B architecture;
- document evidence and open a Pull Request.
Physical motion is not a Session 01 student task unless explicitly authorized by the instructor.

2. Session Phases
Phase	Time	Main objective
0	10 min	Safety + architecture
1	25 min	SSH + ROS 2 environment
2	30 min	Git/GitHub workflow
3	40–50 min	CARLA + ROS bridge + simulated movement
4	35–45 min	Physical Pixloop sensors + KISS-ICP + chassis RX
5	30–40 min	Localization + path-planning stack observation
6	20 min	ROS graph discovery + evidence
7	15–20 min	Commit + push + Pull Request


If CARLA setup or map availability consumes extra time, Phase 5 may be reduced to architecture and node inspection.
3. Safety and Engineering Rule
Physical Pixloop
Do not send physical motion commands unless the instructor explicitly authorizes the test.
For any instructor-authorized physical-control test:
- keep the vehicle area clear;
- keep an operator at the remote/E-stop controls;
- verify the intended control mode;
- use Neutral/Park whenever motion is not required;
- never run two CAN TX paths at the same time;
- stop immediately if the reported state or physical behavior is unexpected.
Before publishing to an unknown ROS 2 topic
1. identify the topic;
2. inspect its message type;
3. inspect the message fields;
4. inspect publishers/subscribers;
5. only then construct a command.
ros2 node list
ros2 node info <NODE>

ros2 topic list
ros2 topic info <TOPIC> --verbose
ros2 topic echo <TOPIC> --once
ros2 topic hz <TOPIC>

ros2 interface show <MESSAGE_TYPE>
4. Autonomy Architecture
Physical / Simulated Sensors
LiDAR · Camera · IMU · Vehicle State
              │
              ▼
      LiDAR Odometry
          KISS-ICP
              │
              ▼
       State Estimation
             EKF
              │
              ▼
         Localization
      Persistent Map / NDT
              │
              ▼
        Path Planning
 Nav2 Map Server + Costmap
      + Planner Server
              │
              ▼
     Tracking / Control
              │
              ▼
           Vehicle
For the current planner-only stage:
PCD map ───────────────► NDT localization
                           │
                           ▼
                      map → odom

Nav2 occupancy map ───► Map Server
                           │
                           ▼
                     Global Costmap
                           │
                           ▼
                     Planner Server
                           │
                           ▼
                    ComputePathToPose
                           │
                           ▼
                /pixloop/planning/path
                           │
                           ▼
                          RViz
No physical controller is launched by the planner-only stack.
Phase 1 — SSH and ROS 2 Environment
5. Connect to Pixloop
Check connectivity:
ping <PIXLOOP_IP>
Connect:
ssh dc@<PIXLOOP_IP>
Validated host:
dc-Nuvo-6108GC
Inspect:
hostname
whoami
pwd
uname -a
lsb_release -a
6. Load the complete Pixloop ROS 2 environment
Validated workspaces:
/data/workspaces/pixloop_ros2_ws
/data/workspaces/pixloop_mapping_ws
/data/workspaces/pixloop-sensor-ws
Source in this order:
source /opt/ros/humble/setup.bash
source /data/workspaces/pixloop_ros2_ws/install/setup.bash
source /data/workspaces/pixloop_mapping_ws/install/setup.bash
source /data/workspaces/pixloop-sensor-ws/install/setup.bash
Runtime environment:
export ROS_DOMAIN_ID=42
export ROS_LOCALHOST_ONLY=0
export RMW_IMPLEMENTATION=rmw_fastrtps_cpp
Verify:
printenv ROS_DISTRO
echo "$ROS_DOMAIN_ID"
echo "$RMW_IMPLEMENTATION"
ros2 pkg list | grep -Ei "pix|nav2|kiss|local|slam|planning|chassis"
Phase 2 — Git and GitHub
7. Course repository
Repository:
git@github.com:DCaicrag/pixloop-cyberphysical-labs.git
Clone once:
cd ~
git clone git@github.com:DCaicrag/pixloop-cyberphysical-labs.git
cd ~/pixloop-cyberphysical-labs
If already cloned:
cd ~/pixloop-cyberphysical-labs
Inspect:
git status
git branch -a
git log --oneline --decorate -10
git remote -v
Create a team branch:
git switch main
git pull origin main
git switch -c feature/team-a/session01
Create the evidence area:
mkdir -p students/team-a/session01/evidence
touch students/team-a/session01/results.md
code students/team-a/session01/results.md
Do not push directly to main.
Phase 3 — CARLA Simulation
CARLA is a central part of Session 01 because it is the safe environment for the first control experiment.
The original Session 01 workflow requires students to:
Start CARLA
    ↓
Start the ROS bridge
    ↓
Discover the simulated ROS graph
    ↓
Identify sensors and vehicle control
    ↓
Inspect the control message
    ↓
Send a short forward command
    ↓
Send STOP
8. Start CARLA
The exact local CARLA installation path has not yet been recovered from the validated Pixloop notes. Keep it explicit rather than guessing:
cd <TO_CONFIRM_CARLA_PATH>
Inspect:
pwd
ls
Typical executable name to look for:
CarlaUE4.sh
Once the local installation is confirmed:
<CARLA_START_COMMAND>
Record:
CARLA version:
Map:
Server:
Port:
9. Start the CARLA ROS 2 bridge
In another terminal:
source /opt/ros/humble/setup.bash
source <TO_CONFIRM_CARLA_ROS_WS>/install/setup.bash
Start the bridge with the command validated on the workstation:
<TO_CONFIRM_CARLA_ROS_BRIDGE_COMMAND>
Keep the server and bridge terminals open.
10. Discover the CARLA graph
ros2 node list
ros2 topic list | grep -i carla
Search simulated sensors:
ros2 topic list | grep -Ei "camera|image"
ros2 topic list | grep -Ei "lidar|point|scan"
ros2 topic list | grep -Ei "imu"
ros2 topic list | grep -Ei "odom|odometry"
Search vehicle control:
ros2 topic list | grep -Ei "control|cmd|vehicle|ego|throttle|steer"
For the discovered control topic:
ros2 topic info <CARLA_CONTROL_TOPIC> --verbose
ros2 interface show <CARLA_CONTROL_MESSAGE_TYPE>
Identify the fields corresponding to:
throttle
steering
brake
reverse
hand_brake
11. First movement — CARLA only
Before moving, prepare the validated STOP command:
<TO_CONFIRM_CARLA_STOP_COMMAND>
Then perform one short low-speed forward command:
<TO_CONFIRM_CARLA_FORWARD_COMMAND>
Immediately stop:
<TO_CONFIRM_CARLA_STOP_COMMAND>
Confirm visually that the simulated vehicle stops.
These CARLA values remain intentionally marked TO_CONFIRM because no validated exact local path, bridge command, role name, or tested forward/STOP payload appears in the recovered Pixloop records. Do not replace them with invented values.

Phase 4 — Physical Pixloop Sensor Stack
12. Bring up the validated physical stack
This stack starts the physical sensors and local odometry without starting physical vehicle control.
source /opt/ros/humble/setup.bash
source /data/workspaces/pixloop_ros2_ws/install/setup.bash
source /data/workspaces/pixloop_mapping_ws/install/setup.bash
source /data/workspaces/pixloop-sensor-ws/install/setup.bash

export ROS_DOMAIN_ID=42
export ROS_LOCALHOST_ONLY=0
export RMW_IMPLEMENTATION=rmw_fastrtps_cpp

ros2 launch pixloop_bringup online_bringup.launch.py
The later validated bringup contains:
RoboSense driver
ZED driver
LiDAR bridge
KISS-ICP
sensor TF
EKF
RX-only chassis receiver
The current launch arguments can always be inspected before running:
ros2 launch pixloop_bringup online_bringup.launch.py --show-args
For an explicitly reduced sensor-only demonstration:
ros2 launch pixloop_bringup online_bringup.launch.py \
  enable_lidar_driver:=true \
  enable_zed_driver:=true \
  enable_chassis:=true \
  enable_lidar_odometry:=true \
  enable_tf:=false \
  enable_ekf:=false
Use the reduced form when the objective is only sensor/odometry discovery.
13. Expected physical interfaces
RoboSense RS-Helios-16P
/rslidar_points
    ↓
/pixloop/lidar/points
    ↓
KISS-ICP
    ↓
/pixloop/lidar/odom
Reference:
LiDAR IP: 192.168.1.200
IPC IP: 192.168.1.102/24
frame: rslidar
frequency: ~10 Hz
Stereolabs ZED
Raw physical topics:
/zed/zed_node/rgb/color/rect/image
/zed/zed_node/point_cloud/cloud_registered
/zed/zed_node/odom
Pixloop bridge topics when enabled:
/pixloop/zed/rgb/image
/pixloop/zed/points
/pixloop/zed/odom
Validated camera:
Model: ZED
Serial: 13709
Chassis RX
Innodisk EMUC-B202
      ↓
   emuccan0
      ↓
pixloop_chassis/can_receiver.py
      ↓
/pixloop/chassis/can/raw
Reference traffic:
~325–330 frames/s
11-bit standard CAN
DLC 8
14. Verify the physical graph
ros2 node list
ros2 topic list
Topics:
ros2 topic info /rslidar_points --verbose
ros2 topic info /pixloop/lidar/points --verbose
ros2 topic info /pixloop/lidar/odom --verbose
ros2 topic info /zed/zed_node/odom --verbose
ros2 topic info /pixloop/chassis/can/raw --verbose
Rates:
ros2 topic hz /rslidar_points
ros2 topic hz /pixloop/lidar/points
ros2 topic hz /pixloop/lidar/odom
ros2 topic hz /zed/zed_node/odom
ros2 topic hz /pixloop/chassis/can/raw
Phase 5 — Localization and Path Planning
This phase is an observation/demo of the autonomy stack. It does not command the vehicle.
15. Existing Pixloop planning package
The current workspace contains:
src/pixloop_planning/
├── config/planner.yaml
├── launch/planning.launch.py
├── launch/navigation.launch.py
├── pixloop_planning/goal_bridge.py
├── rviz/navigation.rviz
└── test/
The unified navigation launch starts the existing localization and planner-only components.
16. Full navigation demo with an existing map
Preferred wrapper:
cd /data/workspaces/pixloop-sensor-ws

source /opt/ros/humble/setup.bash
source /data/workspaces/pixloop_ros2_ws/install/setup.bash
source /data/workspaces/pixloop_mapping_ws/install/setup.bash
source /data/workspaces/pixloop-sensor-ws/install/setup.bash

./scripts/pixloop_navigate.sh \
  map_dir:=/path/to/RUN_001/maps
Historical-map form:
./scripts/pixloop_navigate.sh \
  pcd:=/home/dc/pixloop_maps/parking/parking_map.pcd \
  map:=/home/dc/pixloop_maps/parking/parking_nav2.yaml
Optional NDT initial-pose arguments:
initial_x
initial_y
initial_z
initial_roll
initial_pitch
initial_yaw
Equivalent direct launch:
ros2 launch pixloop_planning navigation.launch.py \
  map_dir:=/path/to/RUN_001/maps
17. Nodes launched by navigation.launch.py
The unified navigation launch brings up:
/ndt_localizer
/map_server
/planner_server
/lifecycle_manager_planning
/pixloop_goal_bridge
/pixloop_rviz
Internally:
navigation.launch.py
    │
    ├── pixloop_localization/ndt_localizer
    │
    ├── planning.launch.py
    │      ├── nav2_map_server/map_server
    │      ├── nav2_planner/planner_server
    │      └── nav2_lifecycle_manager/lifecycle_manager_planning
    │
    ├── pixloop_planning/goal_bridge
    │
    └── rviz2
Verify:
ros2 node list | grep -E \
  'ndt|map_server|planner_server|lifecycle|goal_bridge|rviz'
Check lifecycle state:
ros2 lifecycle get /map_server
ros2 lifecycle get /planner_server
Expected active nodes:
/map_server
/planner_server
18. Planning interfaces
RViz configuration:
pixloop_planning/rviz/navigation.rviz
Main visualization topics:
/map
/global_costmap/costmap
/pixloop/lidar/points
/pixloop/planning/vehicle
/pixloop/planning/path
The goal bridge listens for:
/goal_pose
and requests Nav2:
/compute_path_to_pose
Successful paths are published on:
/pixloop/planning/path
Planning status is available on:
/pixloop/planning/status
In RViz:
1. inspect the occupancy map;
2. inspect the LiDAR;
3. inspect the localized vehicle marker;
4. use 2D Goal Pose;
5. drag to specify final orientation;
6. verify that a non-empty path appears.
19. What the planning demo does not start
The planner-only stage deliberately excludes:
controller_server
bt_navigator
velocity_smoother
cmd_vel
physical CAN TX
autonomous vehicle motion
The planner calculates a geometric path only.
Phase 6 — ROS Graph Discovery and Evidence
20. Discover the complete graph
ros2 node list
ros2 topic list
Search by subsystem:
ros2 topic list | grep -Ei "lidar|point|scan"
ros2 topic list | grep -Ei "zed|camera|image"
ros2 topic list | grep -Ei "imu|ins"
ros2 topic list | grep -Ei "odom|localization"
ros2 topic list | grep -Ei "map|costmap|planning|path|goal"
ros2 topic list | grep -Ei "chassis|can"
For each selected interface:
ros2 topic info <TOPIC> --verbose
ros2 interface show <MESSAGE_TYPE>
ros2 topic echo <TOPIC> --once
ros2 topic hz <TOPIC>
21. Save evidence
cd ~/pixloop-cyberphysical-labs

mkdir -p students/team-a/session01/evidence

ros2 node list \
  > students/team-a/session01/evidence/nodes.txt

ros2 topic list \
  > students/team-a/session01/evidence/topics.txt

env | grep -E 'ROS|RMW' \
  > students/team-a/session01/evidence/ros_environment.txt

uname -a \
  > students/team-a/session01/evidence/system.txt

lsb_release -a \
  >> students/team-a/session01/evidence/system.txt 2>&1
Phase 7 — Commit and Pull Request
22. results.md
Suggested structure:
# Session 01 Results

## Team

## Host and ROS 2
Hostname:
Ubuntu:
ROS_DISTRO:
ROS_DOMAIN_ID:

## CARLA
Version:
Map:
Vehicle role:
Bridge:
Control topic:
Control message:
Forward test:
STOP test:

## Physical Pixloop
LiDAR topic:
LiDAR rate:
LiDAR odometry:
ZED topic(s):
Chassis RX topic:

## Localization
NDT node:
Pose topic:
Odometry topic:

## Planning
Map server:
Planner server:
Goal topic:
Path topic:
Path successfully generated:

## Git
Branch:
Commit:
Pull Request:

## Observations

## Problems Found
Commit:
cd ~/pixloop-cyberphysical-labs

git status
git diff

git add students/team-a/session01
git commit -m "docs: add team-a session01 environment validation"
git push -u origin feature/team-a/session01
Open a Pull Request into main.
Instructor Reference — Physical CAN Route B
The current validated command architecture is:
ROS 2 Humble
  → ros1_bridge
  → ROS 1 Noetic
  → socketcan_bridge
  → emuccan0
  → VCU
This section is for infrastructure validation, not student motion.
23. EMUC
sudo modprobe emuc2socketcan
sudo systemctl stop ModemManager

sudo /home/dc/Downloads/Linux/emucd_64 \
  -s7 -e0 /dev/ttyACM0 emuccan0 emuccan1
Then:
sudo ip link set emuccan0 up
sudo ip link set emuccan1 up
ip -details link show emuccan0
24. Bridge container
sudo docker start pixloop-route-b-test
sudo docker ps --filter name=pixloop-route-b-test
Terminal A — ROS 1 master
sudo docker exec -it pixloop-route-b-test bash --noprofile --norc
Inside:
source /opt/ros/noetic/setup.bash
export ROS_MASTER_URI=http://localhost:11311
roscore
Terminal B — ros1_bridge
sudo docker exec \
  -u 1000:1000 \
  -e HOME=/tmp \
  -it pixloop-route-b-test \
  bash --noprofile --norc
Inside:
source /opt/ros/noetic/setup.bash
source /humble_ws/install/setup.bash
source /can_msgs_ws/install/local_setup.bash
source /bridge_ws/install/local_setup.bash

export ROS_MASTER_URI=http://localhost:11311
export ROS_DOMAIN_ID=42
export ROS_LOCALHOST_ONLY=0
export RMW_IMPLEMENTATION=rmw_fastrtps_cpp

ros2 run ros1_bridge dynamic_bridge -- --bridge-all-2to1-topics
Terminal C — ROS 1 to SocketCAN
sudo docker exec -it pixloop-route-b-test bash --noprofile --norc
Inside:
source /opt/ros/noetic/setup.bash
export ROS_MASTER_URI=http://localhost:11311

rosrun socketcan_bridge topic_to_socketcan_node \
  _can_device:=emuccan0 \
  sent_messages:=/pixloop/chassis/can/tx_test
Host endpoint:
ros2 topic info /pixloop/chassis/can/tx_test --verbose
Latest stationary authority validation:
cd /data/workspaces/pixloop-sensor-ws

source /opt/ros/humble/setup.bash
source install/setup.bash

export ROS_DOMAIN_ID=42
export ROS_LOCALHOST_ONLY=0
export RMW_IMPLEMENTATION=rmw_fastrtps_cpp

PYTHONPATH=src/pixloop_chassis:$PYTHONPATH \
python3 -m pixloop_chassis.route_b_authority_probe \
  --ros-args \
  -p physical_enable:=true \
  -p sustained_stationary:=true \
  -p armed_duration_sec:=2.5 \
  -p warmup_sec:=1.0
Validated result:
FINISH: sustained stationary validation completed without safety abort;
en_states={1280: 1, 1281: 1, 1282: 1}
This validates stationary authority only. It does not by itself validate non-zero steering, propulsion, gear release, or vehicle motion.
25. Expected End-of-Session Understanding
Students should leave Session 01 able to recognize this complete flow:
CARLA
  └── simulated sensors + simulated vehicle control

PHYSICAL PIXLOOP
  ├── RoboSense
  ├── ZED
  ├── KISS-ICP
  ├── EKF
  ├── chassis RX
  │
  ├── NDT localization
  │
  ├── Nav2 map server
  ├── global costmap
  ├── planner server
  ├── goal bridge
  └── planned path in RViz

CONTROL INFRASTRUCTURE
  ROS 2
    → ROS 1 bridge
    → socketcan_bridge
    → emuccan0
    → VCU
The session does not require students to make the physical vehicle follow the planned path.
