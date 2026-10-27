# Session 04 — Subsystem Validation and Cross-Team Integration

**Pixloop Cyber-Physical Labs**\
**Development period:** October 27 – November 6, 2026\
**Code and synthetic-test checkpoint:** **Wednesday, November 4, 2026**\
**Final deadline / physical integration laboratory:** **Saturday, November 7, 2026**

> **The November 7 session is the deadline to complete and demonstrate the activities in this README, not the day to start writing code.** Work remotely using the current repositories and ROS 2 interfaces. By November 4, each team must have runnable code and initial synthetic tests (WSL, ROS 2 mock publishers, bags, or CARLA as appropriate). Use November 4–7 for hardware validation, correction, and integration.

## 1. Starting Point

The main Pixloop stack already provides LiDAR RoboSense, ZED, CHCNAV IMU integration, KISS-ICP, an EKF configuration, CAN RX/energy reporting, Route B, and the consolidated `keyboard_teleop_full` for supervised physical operation. **Do not create another teleoperator or CAN driver.** Some integrations remain uncalibrated or unvalidated.

**Repository and workspace separation (do not confuse them):**

- **Instructor's physical stack:** [`DCaicrag/pixloop-sensor-ws`](https://github.com/DCaicrag/pixloop-sensor-ws), locally at `/data/workspaces/pixloop-sensor-ws` on the NUVO. Use it to **inspect and reuse** the actual CAN, IMU, EKF, PID, Route B, Nav2 planning and bringup interfaces. Students must not replace this workspace with their own packages.
- **Students' development workspace:** `~/plataforma_challenge/pixloop_challenge_ws` (under `~/plataforma_challenge/`). Implement and test students' code here, including `deteccion_obstaculos` and `battery_navigation_pkg`.
- Inspect existing `pixloop_heading_control`, `pixloop_localization`, `pixloop_planning`, `pixloop_chassis`, and launch/config files **in the instructor's repository** before creating duplicate functionality in the student workspace.

Quick inspection (paths shown are examples from the NUVO; students may clone the reference repository separately):

```bash
# Instructor / reference stack (NUVO)
cd /data/workspaces/pixloop-sensor-ws
find src/pixloop_planning src/pixloop_heading_control src/pixloop_chassis -maxdepth 3 -type f | sort

# Students' workspace
cd ~/plataforma_challenge/pixloop_challenge_ws
find src -maxdepth 3 -type f | sort
```


Each team must document exactly which existing files it reused, changed, or added. Prefer incremental changes and preserve existing topic contracts. **Record all commands, parameters, topics, tests and operating instructions in the team README on GitHub** so other teams can reproduce the work.

## 2. Timeline and Required Checkpoints

| Date                           | Required result                                                                                                                                                                                          |
| ------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Oct 27 – Nov 3 (remote)**    | Inspect code, define inputs/outputs, implement missing logic, and run initial synthetic tests.                                                                                                           |
| **Wed Nov 4 (checkpoint)**     | **Runnable code + repeatable synthetic tests** from each team. Demonstrate nodes, sample inputs, topics, outputs, and expected/actual results. Team B may also check ZED/LiDAR on the physical platform. |
| **Nov 5–6**                    | Fix issues from the checkpoint; prepare logs, plots, parameters and validation scripts.                                                                                                                  |
| **Sat Nov 7 (final deadline)** | Validate against hardware and/or current Nav2 stack as specified below, calibrate, integrate interfaces, and submit the evidence and remaining limitations.                                              |

Do not postpone first implementation until November 7.

## 3. Team A — Localization, Wheel Distance and Heading Control

**Goal:** Convert existing chassis speed feedback into measurable displacement, command controlled straight-line movement, and check localization readiness for Nav2.

**Existing components to reuse:** `wheel_speed_decoder_node` (CAN `0x506` wheel speeds already available), `keyboard_teleop_full` as a *reference for CAN authority/safety behavior*, `pixloop_heading_control`, KISS-ICP, CHCNAV IMU, EKF profiles and NDT localizer.

### Work to complete

1. **Wheel distance/odometry — missing component.** The existing `wheel_speed_decoder_node` already decodes CAN `0x506` and publishes measured wheel speeds. **Do not recreate or recheck this decoder as a missing deliverable.** Implement the missing integration of measured speed over time into **signed traveled distance (m)**, and only publish `nav_msgs/Odometry` if the available kinematic information supports a defensible pose estimate. Validate time integration, direction, stops and stale feedback. Wheel speed alone does not provide reliable global pose or yaw.
2. **Distance command.** Build a **supervised, closed-loop straight-line distance controller** (example request: `1.0 m`), with feedback, deceleration near the target, stop/timeout handling and measured final error. Keep control logic separate from physical CAN transmission; do not replace the teleop. Use synthetic inputs before physical authorization.
3. **Heading PI/PID.** Inspect and tune the existing `pixloop_heading_control` to correct heading deviations using valid IMU/localization feedback. Separate **distance control** from **heading control**.
4. **EKF + NDT review.** Check topics, frames, timestamps, covariances, LiDAR/ZED/IMU fusion and NDT map localization; identify what is missing for `map → odom → base_link`. Do not claim wheel odometry is already fused until confirmed in configuration.

**By Wednesday Nov 4:** code builds/runs; synthetic wheel speeds produce the expected integrated distance (including stop and reverse); distance-controller state/target/feedback are observable; heading correction responds sensibly to signed simulated error; EKF/NDT inspection results recorded.

**By Saturday Nov 7:** compare integrated distance against **physically measured travel** and adjust scale/sign; test a small straight-line distance command **only if the safety interface, feedback, stopping, area, and operator authorization are verified**; measure target vs actual displacement, overshoot and heading drift. Check EKF/NDT topics/TF with live data and report issues. If physical motion cannot safely be enabled, demonstrate the closed loop with recorded/simulated feedback and document the blocker.

**Nav2 interface:** validated `odom`, robot pose and the TF chain `map → odom → base_link`. The heading/distance controllers are stepping stones for tracking Nav2 commands, **not a second Nav2 planner**.

## 4. Team B — Obstacle Detection, Coordinates and Nav2 Costmap

**Goal:** Verify that existing `obstacle_localization_node` identifies obstacles at meaningful **robot-relative distances and coordinates**, especially whether they are **in front of the vehicle and relevant to its path**, rather than merely displaying a LiDAR/ZED distance.

**Existing components to reuse:** YOLO detection, `obstacle_localization_node`, ZED RGB/depth/point cloud, LiDAR, TF and any existing obstacle outputs. Inspect the actual code/interfaces first; do not create another obstacle localizer unless the current one is missing required functionality.

### Work to complete

1. **Inspect measurement logic.** Determine precisely how LiDAR and ZED distances are selected/associated with a detected obstacle; verify sensor frame, projection, synchronization, units and transforms.
2. **Expose actionable terminal output.** For each relevant obstacle, print/log at a controlled rate: **label/ID (if available), source, range (m), `(x, y, z)` in a named frame (prefer `base_link` for robot-relative checks), front/side/behind classification, and `in_path` or `path_overlap` flag**. Do not label arbitrary nearest-point ranges as object coordinates.
3. **Define forward corridor.** In `base_link`, check `x > 0` **only after confirming axis conventions**; apply configurable lateral width and minimum/maximum range. Compare obstacle location against the **actual planned path corridor** when a `nav_msgs/Path` is available. A frontal cone/corridor is an initial approximation, not a full path-collision check, especially in turns.
4. **Nav2 integration.** Confirm the obstacle output can be transformed into a supported costmap observation source (`PointCloud2`, `LaserScan`, or a verified compatible adapter). Demonstrate that adding/removing an obstacle changes the costmap and can result in re-planning. **Detection reports geometry; the navigation/controller/safety layer determines slowing, stopping or rerouting.**

**Suggested example terminal record** (illustrative format, not a claim about current topics):

```
obstacle=person  source=zed+lidar  frame=base_link
range=2.35m  x=2.30m  y=+0.46m  z=0.10m
sector=FRONT  in_forward_corridor=true  overlaps_planned_path=true
```

**By Wednesday Nov 4:** code and synthetic tests for front/side/behind, positive/negative lateral offset, range filtering, inconsistent or missing ZED/LiDAR measurements, and path intersection; publish/log coordinates and reasons for classification. Use mocked detections or recorded point clouds. If access permits, **validate the implementation with physical ZED + LiDAR on Wednesday**.

**By Saturday Nov 7:** demonstrate actual ZED/LiDAR obstacle coordinates in RViz/terminal, compare against tape-measured positions at several placements, verify false positives/out-of-path obstacles, and connect the validated source to **Nav2 costmap in simulation or planner-only mode**. Show at least one obstacle insertion/removal and the resulting route update or explain the blocking interface issue. No unsupervised vehicle movement.

**Nav2 interface:** sensor detections → correctly framed obstacle observations → costmap → planner replanning. A `path_overlap` flag may aid diagnostics; Nav2 still needs obstacle geometry in its costmap.

## 5. Team C — Nav2 Planning, Energy and Route Integration

**Goal:** Connect the **existing energy-based station selector** to **actual Nav2 paths and a real map frame**, instead of deciding from synthetic station coordinates alone. This must produce code and a working integration test, not merely an architecture review.

**What already exists (do not recreate):**

- In the **students' workspace** `~/plataforma_challenge/pixloop_challenge_ws`, `battery_navigation_pkg` contains `battery_selector_node`; it **already consumes SOC from the chassis energy interface**.
- Its current charging-station positions/coordinates are **synthetic**. They are *not* physically surveyed or validated points in the Pixloop map.
- The **instructor's** `DCaicrag/pixloop-sensor-ws` repository includes `pixloop_planning` / Nav2 planning configuration, which the team must inspect and reuse.

### What code Team C must deliver

1. **Inspect Nav2 in `DCaicrag/pixloop-sensor-ws`.** Identify the actual planner launch/configuration, required map/TF, the `ComputePathToPose` action interface, and how its output `nav_msgs/Path` is obtained. Record exact discovered interfaces, not assumed ones.
2. **Implement a small Nav2 route-query node/client** inside the students' `battery_navigation_pkg` (suggested name: `battery_nav2_client_node.py`; **new work**). Given the robot pose and candidate destinations, request paths from the existing Nav2 planner and calculate their lengths from the returned poses. Handle unavailable paths and action failures.
3. **Keep the working SOC input.** Reuse `battery_selector_node` and its real SOC subscription; replace the *synthetic straight-line route costs* with actual Nav2 path lengths. Do not create another battery monitor or station selector. The energy-per-meter model and safety reserve can remain explicitly provisional.
4. **Address the synthetic station coordinates correctly.** For November 4, use labelled **synthetic station poses in a simulated map** and document their frame. For November 7, inspect the available Pixloop map and propose/define test station locations consistently in its `map` frame **only if that map and its TF are valid**. Do not describe these test poses as surveyed physical charging stations.
5. **Expose the decision.** Log/publish each candidate path status and distance, current SOC, estimated energy requirement, selected destination/station pose, and why it was selected. This chosen goal is the output that a later Nav2 navigation client can use.

**Minimal example:** destination + two *simulated* charging stations → Nav2 path for each candidate → route lengths → existing SOC-based selection logic → one selected pose in the map frame. Include one unreachable/no-path case.

**By Wednesday Nov 4:** in WSL / ROS 2 simulation, show working route-query code, a synthetic map with labelled station poses, computed path lengths, and the existing selector reacting to different SOC values (using synthetic SOC only for tests). The actual student's selector must remain connected to its real SOC input for physical operation.

**By Saturday Nov 7:** connect the new Nav2 client and existing selector to the instructor's current Nav2 planning setup where available; show paths and distances in terminal/RViz and the chosen goal. Review map, frames and realistic candidate station placement. **Do not require autonomous physical trajectory following or CAN transmission yet**; that belongs to later integration sessions.

**Nav2 interface:** candidate destination/station poses → **new Nav2 route-query client** → `ComputePathToPose` → `nav_msgs/Path` / distance → **existing `battery_selector_node` + SOC** → selected goal. Later: selected goal → `NavigateToPose` / controller / supervised chassis adapter.

## 6. Cross-Team Integration Contract

```
Team A: pose / odom / TF ─────────────────────────────┐
                                                     ↓
Team B: obstacle observations ───────────────→ Nav2 costmap + planner
                                                     ↓
Team C: synthetic/test station poses + real SOC → Nav2 path lengths / energy feasibility
                                                     ↓
                                           selected destination / path
                                                     ↓
                                  future Nav2 controller + chassis adapter
```

- **A → C:** pose/TF validity and odometry frame; confirm exact live topic names.
- **B → C:** obstacle geometry in a common TF frame, accepted by Nav2 costmap; coordinate obstacle observations with the planned path.
- **C → A/B:** publish/record the current planned route for path-relative obstacle checks and future trajectory following.
- **Physical authority:** `keyboard_teleop_full` remains the existing *manual* verified reference; an automatic controller must have a separately reviewed arbitration/stop mechanism and must never run simultaneously with competing CAN TX publishers.

## 7. Delivery Checklist — Due Saturday, November 7

Each team must provide:

- **Document in GitHub README all commands needed to run the nodes**, including workspace sourcing, launch commands and run order.
- Document exact topics, ROS 2 message types, frames, parameters and dependencies, plus existing files reused or modified.
- Document **repeatable synthetic tests available by Nov 4** with commands, example input and expected/actual output.
- Document the **November 7 validation** with exact terminal/RViz commands, measured values and errors; clearly distinguish physical, bag and synthetic evidence.
- Include a brief **working / partially working / pending** summary and remaining issue.

GitHub is the shared **operating manual and code location**, not an exercise in requiring individual pull requests or elaborate commit workflows.

**Acceptance on Nov 7:** A measures displacement and evaluates a closed-loop distance/heading controller; B demonstrates robot-relative obstacle locations and a Nav2 costmap integration attempt/result; C **implements a new Nav2 route-query client**, integrates its path distances with the **existing SOC-connected selector**, and publishes/logs the chosen goal; station poses remain explicitly synthetic/test-only until mapped and validated. Full physical autonomous navigation is **not** the Session 04 requirement.

## 8. Safety and Remaining Sessions

For any physical movement: authorized supervisor on site, controlled test area, available E-stop, no concurrent CAN command publishers, fresh wheel/vehicle feedback, validated stop sequence. Keyboard timeout alone is not a safety-rated dead-man. Never bypass stationary/gear confirmation to expedite tests.

- **Nov 11 / Sat Nov 14 (Session 05):** persistent map/global localization, Nav2 costmap and simulated path following.
- **Nov 18 / Sat Nov 21 (Session 06):** energy reachability, obstacles, route choice and replanning integrated.
- **Nov 25 / Sat Nov 28 (Session 07):** final system demonstration, with physical operation only if validated and approved.
