# Session 02 — System Architecture and ROS 2 Interfaces

**Date:** October 10, 2026  
**Duration:** 3–4 hours  
**Platform:** Pixloop / PIX-KIT-HOOKE  
**Topics:** Workspace inspection · ROS 2 graph · Sensors · TF · Interface contracts

## 1. Objective

Understand the existing Pixloop architecture and define how the three teams will exchange information through ROS 2.

| Team | Subsystem | Core question |
|---|---|---|
| **A** | Localization & Mapping | Where is the vehicle? |
| **B** | Perception & Vehicle State | What surrounds the vehicle, and what is its state? |
| **C** | Planning & Decision Making | Where should it go, and can it reach the destination? |

**Expected result:** Each team identifies its packages, nodes, inputs, outputs, TF frames, dependencies, and an initial interface contract. **No full subsystem implementation is required today.**

## 2. Connect and Inspect the Workspaces

Load the ROS 2 environment in **each new terminal**:

```bash
source /opt/ros/humble/setup.bash
source /data/workspaces/pixloop_ros2_ws/install/setup.bash
source /data/workspaces/pixloop_mapping_ws/install/setup.bash
source /data/workspaces/pixloop-sensor-ws/install/setup.bash
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp

printf 'ROS: %s | RMW: %s\n' "$ROS_DISTRO" "$RMW_IMPLEMENTATION"
```

Expected: `humble` and `rmw_cyclonedds_cpp`.

### Workspace organization

| Workspace | Purpose |
|---|---|
| `pixloop_ros2_ws` | Physical drivers and external ROS 2 dependencies |
| `pixloop_mapping_ws` | LiDAR odometry / KISS-ICP dependencies |
| `pixloop-sensor-ws` | Pixloop packages, localization, planning, chassis, and bringup |

Inspect the main workspace:

```bash
cd /data/workspaces/pixloop-sensor-ws
ls
ls src
ros2 pkg list | grep pixloop | sort
find src -maxdepth 2 -type d | sort
find src -type f -name '*.launch.py' | sort
find src -type f \( -name '*.yaml' -o -name '*.yml' \) | sort
find scripts -maxdepth 1 -type f | sort
```

Relevant packages to locate include `pixloop_lidar`, `pixloop_zed`, `pixloop_ins`, `pixloop_chassis`, `pixloop_localization`, `pixloop_slam`, `pixloop_adaptive_fusion`, `pixloop_tf`, `pixloop_planning`, and `pixloop_bringup`.

Operational scripts to inspect: `pixloop_up.sh`, `pixloop_check.sh`, `pixloop_navigate.sh`, `pixloop_record.sh`, and `pixloop_build_maps.sh`.

**Task:** Identify which packages and launch/config files are relevant to your team. Inspect the launch/config files before reading entire implementations.

## 3. Inspect the Running ROS 2 Graph

With the authorized operator and the platform in a safe state, start the existing stack:

```bash
cd /data/workspaces/pixloop-sensor-ws
./scripts/pixloop_up.sh
```

Keep this terminal open. In a **second terminal**, reconnect if necessary and source the environment from Section 2.

### Discover nodes and topics

```bash
ros2 node list
ros2 topic list -t
ros2 action list -t
ros2 service list -t
```

Filter by subsystem:

```bash
# LiDAR / camera
ros2 topic list | grep -Ei 'lidar|rslidar|point|scan|zed|camera|image'

# Odometry / localization / TF
ros2 topic list | grep -Ei 'odom|localization|pose|^/tf$|^/tf_static$'

# Vehicle / energy
ros2 topic list | grep -Ei 'chassis|can|wheel|speed|energy|battery|bms'

# Planning
ros2 topic list | grep -Ei 'map|costmap|goal|planning|path'
```

For each relevant interface, inspect its producer, subscribers, type, and behavior:

```bash
ros2 node info <NODE>
ros2 topic info <TOPIC> --verbose
ros2 interface show <MESSAGE_TYPE>
ros2 topic echo <TOPIC> --once
ros2 topic hz <TOPIC>
```

`ros2 topic hz` measures an observed rate, not necessarily the configured publishing rate. Stop continuous commands with `Ctrl+C`. Check QoS compatibility if messages are not received.

For actions, use `ros2 action list -t` and `ros2 action info <ACTION>` rather than treating the action name as a topic.

### Inspect TF

```bash
ros2 run tf2_tools view_frames
ros2 run tf2_ros tf2_echo map base_link
```

The **target conceptual hierarchy** is:

```text
map
 └── odom
      └── base_link
           ├── rslidar
           └── zed_camera_link
```

These names and transforms are architectural expectations; **verify the actual frame IDs and transform publishers** in the running graph. Do not create duplicate TF publishers.

## 4. Existing Interface Reference

The following names are starting points from the Pixloop stack description. **Their live availability, exact message types, publishers, and QoS must be verified in this session.**

| Subsystem | ROS 2 interface | Role |
|---|---|---|
| LiDAR | `/rslidar_points` | Raw point cloud |
| LiDAR | `/pixloop/lidar/points` | Bridged point cloud |
| Odometry | `/pixloop/lidar/odom` | LiDAR odometry |
| Camera | `/zed/zed_node/rgb/color/rect/image` | Rectified RGB image |
| Camera | `/zed/zed_node/point_cloud/cloud_registered` | Registered point cloud |
| Camera | `/zed/zed_node/odom` | Camera odometry |
| Chassis | `/pixloop/chassis/can/raw` | Raw CAN data |
| Energy | `/pixloop/energy/bms/raw` | Raw BMS data |
| Energy | `/pixloop/energy/bms/status` | BMS status |
| Energy | `/pixloop/energy/battery_state` | Battery state |
| Planning | `/goal_pose` | Requested goal |
| Planning | `/pixloop/planning/path` | Planned path |
| Planning | `/pixloop/planning/status` | Planning status |
| TF | `/tf`, `/tf_static` | Dynamic and static transforms |
| Planning action | `/compute_path_to_pose` | Path-computation action; verify exact name/type |

Typical LiDAR flow:

```text
RoboSense → /rslidar_points → LiDAR bridge
          → /pixloop/lidar/points → KISS-ICP
          → /pixloop/lidar/odom
```

Relevant planning components to look for: `/ndt_localizer`, `/map_server`, `/planner_server`, `/lifecycle_manager_planning`, and `/pixloop_goal_bridge` (confirm their active node names).

## 5. Team Assignments

### Team A — Localization & Mapping

**Goal:** Identify how local odometry and global localization provide a consistent vehicle pose.

**Inspect:** `pixloop_lidar`, localization/fusion packages, KISS-ICP, EKF, NDT, maps, and TF.

```bash
ros2 node list | grep -Ei 'kiss|odom|ekf|local|ndt|tf'
```

| Inputs to investigate | Outputs / responsibilities to confirm |
|---|---|
| `/pixloop/lidar/points` | Local odometry |
| `/pixloop/lidar/odom` | Global vehicle pose |
| `/zed/zed_node/odom` | `map → odom` TF |
| Persistent map, `/tf`, `/tf_static` | `odom → base_link` TF (identify its actual owner) |

**Decision:** What minimum interface lets Team C obtain the vehicle pose, in which frame, and from which authoritative publisher?

### Team B — Perception & Vehicle State

**Goal:** Identify available environmental and vehicle-state data and determine what Team C actually needs.

**Inspect:** `pixloop_zed`, `pixloop_chassis`, CAN, BMS, and vehicle telemetry.

```bash
ros2 topic list | grep -Ei 'wheel|speed|vehicle|chassis|can|battery|energy'
```

| Inputs to investigate | Outputs / responsibilities to confirm or design |
|---|---|
| ZED RGB image | Detected objects (proposed) |
| ZED registered point cloud | Environment/obstacle representation (proposed) |
| `/pixloop/chassis/can/raw` | Vehicle and wheel speed (verify availability) |
| BMS interfaces | Battery SOC / vehicle state |

**Decision:** Which data and message types are sufficient for planning? The object-detection interface is **not yet defined**.

### Team C — Planning & Decision Making

**Goal:** Inspect the current planning pipeline and define the data needed for future reachability decisions.

**Inspect:** `pixloop_planning`, map server, planner server, goal bridge, and planning actions.

```bash
ros2 node list | grep -Ei 'map_server|planner_server|goal_bridge|ndt'
ros2 action list -t | grep -i compute_path
```

| Inputs to investigate | Outputs / responsibilities to confirm or design |
|---|---|
| Vehicle pose and `/map` | Planned path |
| `/goal_pose` | Planning status |
| Battery SOC | Route length/cost (proposed) |
| Charging-station locations (future input) | Reachability and destination selection (proposed) |
| Perception/vehicle state (future input) | Charging-station selection (proposed) |

**Decision:** What information and assumptions are required to determine whether a destination is reachable?

## 6. Cross-Team Interface Contract

```text
Team A: Localization & Mapping ── Pose / TF ────────┐
                                                     ▼
Team B: Perception & State ── Vehicle / Environment ─► Team C
                                                     │
                                                     ▼
                                      Path / Cost / Decision
```

| Information | Producer | Consumer | Initial interface | Status |
|---|---|---|---|---|
| Vehicle pose | A | C | Determine existing pose/odometry interface | To verify |
| Global TF | A / actual TF owner | B, C | `/tf`, `/tf_static` | To verify |
| Vehicle speed | B | C | `<TO_DEFINE>` | To define |
| Battery SOC | B | C | `/pixloop/energy/battery_state` | To verify |
| Detected objects | B | C | `<TO_DEFINE>` | To define |
| Requested goal | User/external | C | `/goal_pose` | To verify |
| Planned path | C | Integration | `/pixloop/planning/path` | To verify |
| Planning status | C | Integration | `/pixloop/planning/status` | To verify |
| Reachability decision | C | Integration | `<TO_DEFINE>` | To define |

Before proposing a new interface, check whether it already exists and who owns it. For each interface, record **name, purpose, direction, type, producer, consumer, frame, units, expected/observed frequency, QoS, and status**. Mark non-applicable fields as `N/A`; do not guess values.

Use these status values consistently:

- **Existing:** Found in code or configuration, not yet confirmed running.
- **Validated:** Observed and inspected in the running stack.
- **To define:** Interface specification has not been agreed upon.
- **To implement:** Specification agreed upon; implementation pending.

## 7. Deliverables

Each team creates these files in its own folder:

```text
students/<team>/session02/
├── architecture.md
└── interfaces.md
```

### `architecture.md`

Document the subsystem's responsibility, relevant packages/launch files/nodes, current data flow, TF frames, and cross-team dependencies. A short diagram is sufficient.

### `interfaces.md`

Create an interface table using this template:

```markdown
| Interface | Direction | Type | Producer | Consumer | Frame / Units | Rate / QoS | Status |
|---|---|---|---|---|---|---|---|
| ... | Input | ... | ... | ... | ... | ... | Existing |
| ... | Output | ... | ... | ... | ... | ... | To define |
```

At the end of the session, compare the three files and agree on one **shared cross-team contract**, explicitly noting unresolved interface owners, types, frames, and dependencies.

## 8. Git Workflow

Run these commands in your **local clone of the laboratory repository** (not in a Pixloop production workspace). Replace `team-a` with your team's folder/name:

```bash
cd ~/pixloop-cyberphysical-labs
git switch main
git pull origin main
git switch -c feature/team-a/session02

# Create/edit your team's documentation, then:
git status
git add students/team-a/session02
git commit -m "docs: define team-a session02 interfaces"
git push -u origin feature/team-a/session02
```

Open a Pull Request to `main`. Explain what your team owns, which interfaces were confirmed, what must be designed, and which other teams you depend on. Follow the fork/branch workflow from Session 01 if you do not have write access to the upstream repository.

## 9. Completion Checklist

- [ ] Main workspaces, relevant packages, launch files, and configs inspected.
- [ ] Active ROS 2 nodes, topics, actions, and TF inspected.
- [ ] Each team identified its inputs, outputs, and dependencies.
- [ ] Existing, validated, and proposed interfaces clearly distinguished.
- [ ] `architecture.md` and `interfaces.md` created.
- [ ] Shared cross-team contract reviewed; unresolved items recorded.
- [ ] Changes committed and Pull Request opened.

## Next Session — October 24, 2026

**Session 03: Control & Odometry.** Use the agreed interfaces to move toward the first functional subsystem outputs: pose/odometry (A), vehicle-state data (B), and path/route cost (C), subject to the interfaces and hardware validated in Session 02.
