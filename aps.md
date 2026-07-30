---
layout: default
title: SimpleRemote — Web control for UR5e + Robotiq Hand-E
description: Web-based remote control and CSV file-following for a UR5e cobot running on ROS 2 Jazzy / MoveIt.
---

# SimpleRemote

**Web-based remote control and CSV file-following for a UR5e + Robotiq Hand-E**, running on a coprocessor over ROS 2 Jazzy / MoveIt. A FastAPI app serves a single-page operator UI plus a REST/WebSocket API; `podman compose` runs the UR driver, MoveIt, and the app.

![Operator UI overview]({{ '/assets/img/operator-ui.png' | relative_url }})
<!-- placeholder: full-screen screenshot of the single-page operator UI -->

---

## What it does

- **Live remote control** of a physical UR5e arm and Robotiq Hand-E gripper from any browser.
- **Jog, run, and file-follow** motion — upload a CSV of joint angles or Cartesian poses and play it back.
- **Single-operator safety lock** — one leased control session commands motion at a time; everyone else observes read-only.
- **Multi-camera video** — each V4L2 camera is opened once and re-proxied as MJPEG so several browser tiles share one device.
- **Visual trajectory planner** — RViz + MoveIt over noVNC, exporting trajectories straight into the file pipeline.

![Live jogging and video tiles]({{ '/assets/img/jogging.png' | relative_url }})
<!-- placeholder: screenshot of jog controls alongside camera video tiles -->

---

## Architecture at a glance

The FastAPI layer never talks to ROS directly — it talks to a **gateway** with a fixed async interface. Two implementations sit behind the same seam:

| Gateway | Purpose |
| --- | --- |
| `RobotGateway` (mock) | Pure-Python simulation of joints/TCP in memory. Powers the whole test suite. |
| `RosRobotGateway` (real) | rclpy node, ros2_control controller switching, Servo jogging, JTC action goals, MoveIt Cartesian paths. |

**Modes** map to ros2_control controllers, switched on `POST /api/mode`:

- `IDLE` → no controller
- `JOG` → `forward_position_controller` (+ Servo)
- `RUN` → `scaled_joint_trajectory_controller`

![Architecture diagram]({{ '/assets/img/architecture.png' | relative_url }})
<!-- placeholder: block diagram — browser → FastAPI → gateway → ROS 2 / MoveIt → UR5e -->

---

## The file-following pipeline

1. **Parse** — `parse_motion_csv` reads a CSV with a `# key: value` comment header (`type: joints|poses`, `units`, `frame`, `mode`).
2. **Retime / plan** — joints go through the retimer (respecting vel/acc limits); poses go through MoveIt's Cartesian path planner (real deployments accept only `fraction == 1.0`).
3. **Execute** — both produce a `TimedTrajectory`; the gateway runs it, splitting on blocking gripper actions.

![Trajectory planner]({{ '/assets/img/planner.png' | relative_url }})
<!-- placeholder: RViz visual planner with a draggable interactive marker -->

---

## Getting started

```bash
# Tests (no ROS/robot needed — runs against the mock gateway)
python3 -m pytest

# Local smoke app (no ROS, mock backend)
DATA_DIR=$PWD/data AUTH_TOKENS=changeme OPERATOR_PASSWORD=changeme \
  python3 -m uvicorn app.server:create_app --factory --host 127.0.0.1 --port 8443

# Full stack (reads MODE/USE_ROS/etc. from .env)
./control.sh start | stop | restart | status | logs | smoke | build
```

Deployment modes via `MODE`: `real` (physical arm) · `ursim` (simulator) · `mock` (quick checks).

---

## Documentation

Built with FastAPI · ROS 2 Jazzy · MoveIt · podman compose. Deployed on a real UR5e.
