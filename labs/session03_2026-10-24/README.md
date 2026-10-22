
# Session 03 — October 24, 2026

**Pixloop Challenge · ROS 2 Integration · IMU · Object Detection · Energy-Aware Navigation**

## 1. Session Objective

Continue developing the existing ROS 2 nodes and validate their integration with Pixloop.

Each team must:

- Inspect the existing files and packages before creating new ones.
- Run and validate its current nodes.
- Identify the inputs, outputs and dependencies of its subsystem.
- Document what works, what is missing and what should be developed next.

## 2. ROS 2 Quick Concepts

- **Workspace:** Directory containing ROS 2 packages.
- **Package:** Organized collection of nodes, configuration and launch files.
- **Node:** Program that performs a specific task.
- **Topic:** Communication channel between nodes.
- **Publisher:** Node that sends messages to a topic.
- **Subscriber:** Node that receives messages from a topic.
- **Message:** Data structure exchanged between nodes.
- **Launch file:** Starts one or more ROS 2 nodes.
- **TF:** Coordinate transformations between reference frames.
- **RViz:** Tool for visualizing ROS 2 data.

### General Inspection

```bash
cd ~/plataforma_challenge

tree -L 4 -I 'build|install|log|.venv|.venvs|__pycache__'

cd pixloop_challenge_ws
ls src

ros2 node list
ros2 topic list -t
ros2 topic info <TOPIC> --verbose
ros2 node info <NODE>
ros2 topic echo <TOPIC> --once
```

Before creating a new node, verify whether an existing node already provides the required information.

---

## 3. Team C — Integration & Energy-Aware Navigation

**Priority: Validate the existing battery decision nodes using the real Pixloop battery SOC.**

### Existing Files

```text
pixloop_challenge_ws/src/battery_navigation_pkg/
└── battery_navigation_pkg/
    ├── battery_selector_node.py
    └── battery_nav2_client_node.py
```

### Step 1 — Inspect the Package

```bash
cd ~/plataforma_challenge/pixloop_challenge_ws

tree src/battery_navigation_pkg -L 4
```

Identify which node selects charging stations and which communicates with Nav2.

### Step 2 — Activate Physical CAN

**Terminal 1 — EMUC daemon**

```bash
ls -l /dev/ttyACM*

sudo modprobe emuc2socketcan
sudo systemctl stop ModemManager

sudo /home/dc/Downloads/Linux/emucd_64 \
  -s7 -e0 /dev/ttyACM0 emuccan0 emuccan1
```

Keep this terminal running.

**Terminal 2 — SocketCAN**

```bash
sudo ip link set emuccan0 txqueuelen 1000
sudo ip link set emuccan1 txqueuelen 1000

sudo ip link set emuccan0 up
sudo ip link set emuccan1 up

ip -details link show emuccan0
timeout 2s candump emuccan0
```

This session only requires **CAN reception**. Do not activate physical motion or CAN transmission.

### Step 3 — Start the ROS 2 Energy Nodes

In a new terminal, load the Pixloop environment:

```bash
source /opt/ros/humble/setup.bash
source /data/workspaces/pixloop_ros2_ws/install/setup.bash
source /data/workspaces/pixloop_mapping_ws/install/setup.bash
source /data/workspaces/pixloop-sensor-ws/install/setup.bash

export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp

ros2 run pixloop_chassis can_receiver
```

In another terminal, load the same environment and run:

```bash
ros2 run pixloop_chassis energy_monitor
```

Verify the battery output:

```bash
ros2 topic info /pixloop/energy/battery_state --verbose
ros2 topic echo /pixloop/energy/battery_state --once
```

**Important:** `BatteryState.percentage` uses values between `0.0` and `1.0`. For example, `0.86` represents 86%.

### Step 4 — Validate the Decision Nodes

- Confirm that `battery_selector_node.py` subscribes to `/pixloop/energy/battery_state`.
- Test charging-station selection using the existing synthetic map.
- Verify that changing the SOC changes the decision appropriately.
- Inspect `battery_nav2_client_node.py` and determine what is still missing to request a real path from Nav2.
- Confirm that the nodes are correctly located inside `pixloop_challenge_ws/src/`.

**Expected result:** A working ROS 2 battery decision node using real SOC and synthetic charging stations.

**Questions:** How does the node determine that a charging station is reachable? What happens if no station is reachable? How will it obtain actual path costs from Nav2?

---

## 4. Team A — Localization & IMU

**Priority: Validate the CHCNAV IMU node and determine how to integrate it into the existing EKF.**

### Existing Files

```text
localizacion/
└── imu.py

pixloop_challenge_ws/src/pixloop_imu/
├── pixloop_imu/
│   ├── chcnav_imu_node.py
│   └── chcnav_parser.py
├── config/
│   └── imu.yaml
└── launch/
    └── imu.launch.py
```

### Step 1 — Inspect the IMU Package

```bash
cd ~/plataforma_challenge

tree pixloop_challenge_ws/src/pixloop_imu -L 4
```

Understand how the TCP connection, parser and ROS 2 publisher work together.

### Step 2 — Connect to the CHCNAV

Connect the Nuvo to the CHCNAV Wi-Fi network `GNSS-3556506` using the credentials provided privately.

Verify connectivity:

```bash
ping -c 3 192.168.200.1
```

The IMU currently uses:

```text
IP:   192.168.200.1
Port: 9904/TCP
Data: GPCHC
```

### Step 3 — Start the IMU Node

```bash
cd ~/plataforma_challenge/pixloop_challenge_ws

source /opt/ros/humble/setup.bash
source install/setup.bash

ros2 launch pixloop_imu imu.launch.py
```

In another terminal:

```bash
source /opt/ros/humble/setup.bash
source ~/plataforma_challenge/pixloop_challenge_ws/install/setup.bash

ros2 topic hz /imu/data_raw
ros2 topic echo /imu/data_raw --once
```

Expected publication frequency: approximately **10 Hz**.

Inspect individual measurements:

```bash
ros2 topic echo /imu/data_raw --field angular_velocity
ros2 topic echo /imu/data_raw --field linear_acceleration
```

Check that the gyroscope is near zero when stationary and that acceleration magnitude is approximately 9.81 m/s².

### Step 4 — Inspect the Existing EKF

```bash
cd /data/workspaces/pixloop-sensor-ws

tree src/pixloop_localization -L 4
```

Identify the configuration used by:

```bash
ros2 launch pixloop_localization state_estimation.launch.py
```

Determine:

- Which odometry and sensor topics are currently fused.
- Which EKF parameters configure IMU inputs.
- Which TF frames must be consistent with `imu_link`.
- How to incorporate angular velocity Z without duplicating existing measurements.

Do not modify the operational EKF configuration until the IMU axes, units, timestamps and covariances are validated.

**Expected result:** Validated `/imu/data_raw` and a documented proposal for incorporating the IMU into the existing EKF.

**Questions:** Which IMU measurements should be fused first? What transformations are required? How will we demonstrate that adding the IMU improves localization?

---

## 5. Team B — Object Detection & Navigation

**Priority: Validate YOLO in ROS 2, improve detection quality and investigate path replanning.**

### Existing Files

```text
deteccion_obstaculos/
├── deteccion_obstaculos/
│   └── detector_yolo.py
├── models/
├── package.xml
└── setup.py
```

### Step 1 — Inspect and Run YOLO

```bash
cd ~/plataforma_challenge

tree deteccion_obstaculos -L 4
```

Start the ROS 2 node:

```bash
cd ~/plataforma_challenge/pixloop_challenge_ws

source /opt/ros/humble/setup.bash
source ~/plataforma_challenge/.venvs/yolo_ros/bin/activate
source install/setup.bash

ros2 run deteccion_obstaculos detector_yolo
```

Current interfaces:

```text
Input:  /camera/image_raw
Output: /pixloop/perception/yolo/image
```

Verify:

```bash
ros2 topic info /camera/image_raw --verbose
ros2 topic hz /pixloop/perception/yolo/image
```

The camera publisher must already be running. The physical ZED publishes `/zed/zed_node/rgb/color/rect/image`, so verify the actual input and configure a remapping if necessary.

### Step 2 — Improve Detection and RViz

- Display `/pixloop/perception/yolo/image` in RViz using an Image display.
- Confirm that bounding boxes and labels are visible.
- Evaluate whether YOLOv8n detects the obstacles relevant to Pixloop.
- Decide whether custom training, a different model or segmentation is required.
- Propose a structured detection output that another ROS 2 node can consume, beyond the annotated image.

### Step 3 — Inspect Path Planning

Assign one team member to inspect the existing Pixloop navigation pipeline:

```bash
cd /data/workspaces/pixloop-sensor-ws

tree src/pixloop_planning -L 4
```

Existing planning interfaces:

```text
/goal_pose
/compute_path_to_pose
/pixloop/planning/path
/pixloop/planning/status
```

Identify how the system creates paths and how a new goal could trigger replanning.

Consider:

```text
Detected Obstacle
       ↓
Obstacle Representation
       ↓
Costmap Update
       ↓
Replanning
       ↓
Updated Path
```

Object detection alone does not update the Nav2 costmap. Determine which additional information and interface would be required.

**Expected result:** YOLO visualized in RViz, a detection improvement proposal, and a documented approach for integrating obstacles with the planner.

**Questions:** Is detection sufficient or is segmentation necessary? How will detected objects be represented in map coordinates? What should trigger replanning?

---

## 6. Final Session Deliverables

| Team | Required demonstration | Next step |
|---|---|---|
| **Integration** | Real battery SOC received by the station-selection node | Connect route costs and Nav2 |
| **Localization** | CHCNAV publishing valid IMU measurements | Integrate IMU into EKF |
| **Detection** | YOLO output visible in RViz | Improve detection and design obstacle-aware replanning |

Each team must document:

1. **What works:** Include the working nodes, topics and validation results.
2. **What is missing:** Identify bugs, dependencies or unavailable interfaces.
3. **What comes next:** Propose the next development task and how it will be tested.

**Final principle:** Reuse existing Pixloop nodes, keep new packages inside the ROS 2 workspace and validate each subsystem independently before full integration.
