# Pixloop Cyber-Physical Labs

Hands-on laboratory activities using the **Pixloop autonomous vehicle research platform**.

The repository is organized as a sequence of seven guided sessions covering collaborative software development, ROS 2, CARLA simulation, sensors, localization, planning and vehicle control.

## Schedule

| Session | Date | Main topic |
|---|---|---|
| 01 | 2026-10-03 | SSH · Git/GitHub · ROS 2 · CARLA onboarding |
| 02 | 2026-10-10 | ROS 2 graph · Sensors · TF · Visualization |
| 03 | 2026-10-24 | Vehicle control · Odometry · Feedback |
| 04 | 2026-11-07 | KISS-ICP · EKF · State estimation |
| 05 | 2026-11-14 | Persistent map · NDT · Localization |
| 06 | 2026-11-21 | Nav2 · Path planning · Tracking |
| 07 | 2026-11-28 | Integration · Validation · Final challenge |

## Repository structure

```text
pixloop-cyberphysical-labs/
├── README.md
├── .gitignore
├── config/
├── docs/
│   ├── architecture/
│   ├── setup/
│   └── troubleshooting/
├── labs/
│   ├── session01_2026-10-03/
│   ├── session02_2026-10-10/
│   ├── session03_2026-10-24/
│   ├── session04_2026-11-07/
│   ├── session05_2026-11-14/
│   ├── session06_2026-11-21/
│   └── session07_2026-11-28/
├── scripts/
└── students/
```

## Student workflow

Student work must be performed using feature branches.

Branch naming convention:

```text
feature/<team>/sessionXX
```

Example:

```text
feature/team-a/session01
```

Typical workflow:

```bash
git switch main
git pull origin main

git switch -c feature/team-a/session01

# Work on the activity

git status
git add <files>
git commit -m "docs: complete team-a session01"

git push -u origin feature/team-a/session01
```

After pushing the branch, create a Pull Request toward:

```text
main
```

Student work should not be pushed directly to `main`.

## Laboratory sessions

### Session 01 — October 3, 2026

**Pixloop onboarding: SSH · Git/GitHub · ROS 2 · CARLA**

[Open Session 01](labs/session01_2026-10-03/README.md)

### Session 02 — October 10, 2026

**ROS 2 graph · Sensors · TF · Visualization**

Documentation will be added progressively.

### Session 03 — October 24, 2026

**Vehicle control · Odometry · Feedback**

Documentation will be added progressively.

### Session 04 — November 7, 2026

**KISS-ICP · EKF · State estimation**

Documentation will be added progressively.

### Session 05 — November 14, 2026

**Persistent map · NDT · Localization**

Documentation will be added progressively.

### Session 06 — November 21, 2026

**Nav2 · Path planning · Tracking**

Documentation will be added progressively.

### Session 07 — November 28, 2026

**Integration · Validation · Final challenge**

Documentation will be added progressively.

## General engineering workflow

The activities follow a progressive workflow:

```text
Connect
   │
   ▼
Inspect
   │
   ▼
Understand
   │
   ▼
Simulate
   │
   ▼
Develop
   │
   ▼
Validate
   │
   ▼
Document
   │
   ▼
Review
```

Students are expected to use ROS 2 inspection tools to understand the system before modifying it.

Typical tools include:

```bash
ros2 node list
ros2 node info <NODE>

ros2 topic list
ros2 topic info <TOPIC>
ros2 topic info <TOPIC> --verbose
ros2 topic echo <TOPIC>
ros2 topic hz <TOPIC>

ros2 interface show <MESSAGE_TYPE>
```

## Safety

Vehicle motion commands must be validated in simulation before being tested on the physical platform.

During laboratory activities:

- CARLA should be used for initial motion tests.
- Physical Pixloop motion requires explicit instructor authorization.
- A valid stop procedure must exist before testing motion.
- Unknown ROS 2 topics must be inspected before publishing commands.
- Students must not execute physical vehicle commands copied from external sources without validating the topic and message interface.

## Repository policy

The `main` branch represents the reviewed baseline of the laboratory repository.

Student development follows:

```text
main
 │
 ├── feature/team-a/session01
 ├── feature/team-b/session01
 └── feature/team-c/session01
```

The expected integration process is:

```text
feature branch
      │
      ▼
   commit
      │
      ▼
     push
      │
      ▼
Pull Request
      │
      ▼
    review
      │
      ▼
   changes
      │
      ▼
    merge
      │
      ▼
     main
```

Do not merge a Pull Request until it has been reviewed.

## Large files

Do not commit large generated files unless explicitly requested.

Examples include:

```text
ROS bags
CARLA recordings
datasets
point clouds
maps
compiled binaries
build directories
large model files
```

Use the repository primarily for:

```text
documentation
source code
configuration
small experiment results
scripts
laboratory evidence
```

## Repository status

Current development phase:

```text
Session 01 preparation
```

The detailed instructions for the first laboratory session are available at:

```text
labs/session01_2026-10-03/README.md
```
