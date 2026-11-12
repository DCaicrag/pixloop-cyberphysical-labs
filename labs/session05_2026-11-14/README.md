# Session 05 — PIXLOOP | November 14, 2026

**Cyber-Physical Systems · Integration session**  
**Teams:** A — Localization & Mapping · B — Obstacle Detection · C — Energy-Aware Navigation

**Session goal:** Move beyond isolated nodes. Each team should demonstrate a working data flow, inspect real ROS 2 interfaces, and leave reproducible evidence of what works, what fails, and what should be integrated next. This is a session milestone, **not the final grading rubric**.

## Current integration baseline

- **NUVO / ROS 2 Humble:** main integration workspace: `/data/workspaces/pixloop-sensor-ws` (additional overlays may be needed).
- **Physical control:** `keyboard_teleop_full` has been tested for forward and reverse using the ROS 2 → Route B → CAN path. This validates manual keyboard teleoperation, **not Nav2 autonomous driving**.
- **Localization:** CHCNAV IMU data is available on `/imu/data_raw`; EKF fusion, TF consistency, and persistent mapping still require verification.
- **Perception:** the existing `deteccion_obstaculos` package must be connected to the actual camera stream and produce useful obstacle information.
- **Energy:** battery/CAN monitoring and energy-decision logic already exist. **The priority is integration with a navigation component and Nav2**, not rewriting the energy algorithm.

Topic names below are examples from the current setup. **Confirm actual names, types, frames, and rates** before using them.

## Team A — Localization, IMU & Mapping

**Target:** Save and reload **at least one real map**, and verify that CHCNAV IMU measurements actually contribute to the EKF/localization pipeline.

- Verify `/imu/data_raw` (timestamps, `frame_id`, angular velocity, acceleration, covariance, Hz) and the effective EKF configuration (`imu0`, `odom0`, enabled variables, frames, covariances).
- Inspect `map → odom → base_link` and sensor TF; avoid duplicate TF publishers. Confirm that the pose/odometry output is consistent while stationary, translating, and turning.
- Produce a mapping run using **either `keyboard_teleop_full` or the physical remote controller**, as appropriate for the test. Record which control method was used and which sensors were available.
- Save the map in the format supported by the mapping implementation, then **reload it**. A live KISS-ICP trajectory alone is not a saved map.
- Demonstrate map/point cloud, trajectory, pose, and TF in RViz. Note drift, jumps, missing transforms, or sensor limitations.

**Evidence:** saved/reloadable map artifact, RViz screenshot, effective EKF config, brief terminal checks, and a short test record. **No rosbag required for Team A** this session.

```bash
ros2 topic info /imu/data_raw --verbose
ros2 topic hz /imu/data_raw
ros2 topic echo /imu/data_raw --once
ros2 run tf2_ros tf2_echo base_link imu_link
ros2 topic list -t | grep -Ei 'imu|odom|point|map'
```

## Team B — YOLO & Obstacle Detection

**Target:** Fix the existing obstacle node and show reliable detection on **dynamic scenes** with informative terminal output and a visualization suitable for diagnosis.

- Verify the **actual camera publisher/subscriber connection**, input encoding and resolution, image rate, YOLO inference rate, and output rate. Do not assume `/camera/image_raw` exists; the ZED image topic may need remapping.
- Produce an inspectable output: class, confidence, timestamp, source image/frame, and obstacle relevance. Include estimated distance/position **only if backed by depth, calibration, or another documented source**.
- Test at least three scenes: **clear path**, **obstacle in the travel corridor**, and **lateral/irrelevant object**. Include camera/vehicle motion or dynamic bag replay.
- Show the annotated image and, **if available**, obstacle markers or another compatible output in RViz. Document false positives, missed detections, latency, and frame/TF problems.
- Expose an obstacle output that navigation could consume; document its ROS 2 contract even if the downstream consumer is not ready.

**Required evidence — one useful ROS 2 bag plus terminal logs:**

| Record in the bag | Why it matters |
| --- | --- |
| **Actual input image** used by the detector | Reproduce the exact frames and confirm the camera connection |
| **YOLO detections** (boxes/classes/confidences or the actual published equivalent) | Verify inference and compare detections with the image |
| **Obstacle-node output** | Check relevance/position logic and the interface intended for navigation |
| **Depth / point cloud** if distance is calculated | Reproduce distance estimation rather than assuming 2D boxes give meters |
| **`/tf`, `/tf_static`, and odometry** if transforming/motion-tracking obstacles | Reproduce frame transformations and motion-related errors |

Record an entire **short, representative dynamic sequence**, including a clear interval and a positive detection. A bag containing only camera images is **not sufficient** to verify the full pipeline. If a required output is missing, keep the input bag and explicitly mark the missing topic as a failure rather than inventing it.

**Terminal evidence:** actual topic names/types, `ros2 topic info --verbose`, input/output `ros2 topic hz`, a sample detection message, node errors, and `ros2 bag info` (duration, message counts, sizes).

```bash
ros2 node list
ros2 topic list -t | grep -Ei 'image|yolo|detect|obstacle|depth|marker'
ros2 topic info <ACTUAL_IMAGE_TOPIC> --verbose
ros2 topic hz <ACTUAL_IMAGE_TOPIC>
ros2 topic hz <ACTUAL_DETECTION_TOPIC>
ros2 topic echo <ACTUAL_OBSTACLE_TOPIC> --once

# Replace the placeholders with verified topics before running.
ros2 bag record -o team_b_dynamic_test \
  <ACTUAL_IMAGE_TOPIC> <ACTUAL_DETECTION_TOPIC> <ACTUAL_OBSTACLE_TOPIC> \
  /tf /tf_static
ros2 bag info team_b_dynamic_test
```

For playback, use an **isolated ROS 2 test environment**. Never replay recorded motion/CAN commands into a live physical actuator path.

## Team C — Energy Logic ↔ Navigation / Nav2

**Target:** Integrate the **existing energy logic** into the navigation workflow. The deliverable is a working interface, not another standalone SOC calculator.

- Inspect the existing energy node and determine **where its decision should enter navigation**: direct Nav2 goal selection, an adapter node, or an update to the team's navigation node.
- Define explicit **ROS 2 contracts**: battery input (e.g. `/pixloop/energy/battery_state`), localization/pose input, energy decision output, selected destination/charging station, and Nav2 action/service/topic to consume it. Record actual types, frames, units, and ownership.
- Connect the decision to **a Nav2 planning request or goal update** and verify that a route/goal response is observable. If Nav2 is unavailable, validate the same interface against a clearly labeled test consumer/mock.
- Check the **navigation → motion-control boundary**: how Nav2 commands would be translated into the actual chassis/CAN interface. Compare the contract with the **working `keyboard_teleop_full` and Route B**. The keyboard teleop is a proven diagnostic reference, **not** the Nav2 controller or a node that should run in parallel with it.
- Show three decision outcomes using the existing logic: enough energy → original goal; insufficient energy → viable charging station; no feasible route → no-go/failure state. Focus on observing the **goal/interface change**, not reimplementing calculations.

**Evidence:** architecture/contract diagram, relevant code/launch/config, logs of one energy-driven Nav2 goal or planning request, and a table with the three cases. A **small targeted bag or trace is optional** (battery + pose + decision + goal/path); do not spend the session recording unrelated sensors. No physical autonomous motion is required.

```bash
ros2 topic info /pixloop/energy/battery_state --verbose
ros2 topic echo /pixloop/energy/battery_state --once
ros2 node list | grep -Ei 'nav|planner|energy|battery|controller'
ros2 action list -t
ros2 topic list -t | grep -Ei 'battery|goal|plan|path|cmd_vel|odom'
```

**Motion boundary:** Test message flow and planning separately from physical actuation. Do not run Nav2 motor output and keyboard teleoperation as concurrent CAN command sources. Physical motion needs an authorized, locally supervised test.

## Repository structure & AI-assisted development

Each team should maintain this **small, durable source of truth** in its own repository (or clearly owned package area):

```text
my-project/
├── AGENTS.md                 # Permanent rules for coding agents
├── README.md                 # Purpose, setup, run and test commands
├── docs/
│   ├── ARCHITECTURE.md       # Verified nodes, ROS 2 contracts and data flow
│   ├── CURRENT_STATE.md      # What works now, blockers, next action
│   ├── DECISIONS.md          # Engineering decisions and trade-offs
│   ├── EXPERIMENTS.md        # Commands, scenarios, outputs and evidence
│   └── KNOWN_ISSUES.md       # Reproducible failures and limitations
├── tasks/                    # Small scoped tasks and acceptance criteria
├── tests/                    # Automated or repeatable checks
└── src/                      # Application code
```

**Using Codex, Claude Code, or other coding agents:**

1. First ask the agent to **inspect the repository and update `CURRENT_STATE.md`** using actual code and tests, clearly marking unverified assumptions. Do not ask it to rewrite the project immediately.
2. Give it **one task at a time** from `tasks/`, with a concrete input/output contract and an observable pass condition. Have it read `AGENTS.md`, `ARCHITECTURE.md`, and `KNOWN_ISSUES.md` first.
3. Require a **minimal diff**, tests or terminal evidence, and an update to `EXPERIMENTS.md` and `CURRENT_STATE.md`. Record meaningful choices in `DECISIONS.md`.
4. Review changes before committing; do not accept “it works” without a reproducible command and output. Never let an agent silently alter safety interlocks or physical command paths.

**Why this beats isolated AI chats:** chat context is temporary and can preserve incorrect assumptions. Repository files give future agents and teammates **versioned, inspectable context**: verified architecture, current blockers, previous experiments, and explicit acceptance tests.

## Session handoff

Submit the relevant **Git commit**, the exact run command, verified ROS 2 contracts, test results (**PASS / FAIL / BLOCKED**), and the team-specific evidence above. Keep large bags outside Git unless agreed; provide their location and `ros2 bag info` output.

**Next integration boundary:** A supplies consistent pose/TF; B supplies navigation-consumable obstacle data; C turns energy decisions into navigation goals and defines how navigation eventually reaches chassis control.
