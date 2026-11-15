# RouteSync

**An ESP32-Based Two-Robot Warehouse Coordination System**

Course project for **Logic Design and Microprocessor (LDM)**, second year.

RouteSync is two small line-following robots that run a route on a mock warehouse board, report their status to a shared coordinator, and use a deterministic priority rule so they never occupy the same junction at the same time. A monitoring interface shows what each robot is doing and why the coordinator made each decision.

> **Status: in development.** This README separates what has been **tested**, what is **proposed**, and what is **planned**. Nothing here should be read as a finished, benchmarked system.

---

## What the system is meant to do

1. **Follow a known route** using an IR line-sensor array and differential-drive motors. The robots do not plan the route themselves — the route is dark tape laid on a light surface.
2. **Stop locally on obstacles** when the front ultrasonic sensor sees something ahead, and resume when it clears.
3. **Report status** from each robot to a common coordinator.
4. **Resolve contention** for a task or a narrow junction using a shared priority rule, so two robots never enter the same space together.
5. **Visualize** robot state, task ownership, and coordination events in an interface.

### Coordination model: hybrid, not fully decentralized

The project started as a decentralized swarm concept and is now moving to a **shared coordinator**:

| Runs on each ESP32 (local, autonomous) | Runs on the coordinator (shared) |
| --- | --- |
| Line following | Combining both robots' status reports |
| Motor / PWM control | Task assignment and sequencing |
| Immediate ultrasonic obstacle stop | Junction right-of-way decisions |
| Reporting status upstream | Issuing grant / yield instructions |

This is **hybrid coordination** — safety-critical reflexes stay on the robot, while task and right-of-way decisions are centralized. It should not be described as fully decentralized swarm behaviour.

**Safety rule:** immediate obstacle stopping must keep working with the coordinator link down. If a coordinator or peer status message goes stale, a robot **waits** rather than assuming a junction is clear.

---

## Hardware (per robot, as reported by the team)

- ESP32 development board
- L298N motor driver
- Four-wheel chassis with DC gear motors
- IR line-sensor array (five-channel, TCRT5000-style strip)
- HC-SR04 ultrasonic sensor, front-facing
- Battery pack (7.4 V 2-cell Li-ion) with a switch

**Caveats that matter:**

- Some IR channels are reported faulty. Working channel count per robot must be confirmed.
- The two robots may **not** have identical wiring or pin maps. Do not assume parity.
- IR output polarity (line = HIGH or LOW) depends on the module and wiring — test before changing firmware.
- The ESP32 cannot drive the motors from GPIO; all motor current goes through the L298N.
- **There is no gripper or pickup mechanism.** Pickup and drop-off are marker-based / simulated, not physical object handling.

---

## Route, nodes, and tasks

An IR array can follow tape, but it cannot by itself tell you *where* the robot is or which of several identical junctions it has reached. So the route needs **identifiable nodes**:

```
START → J1 → P1 (pickup) → J2 → D1 (drop-off)
```

Nodes can be identified by distinct tape patterns, a known junction-count sequence along a fixed route, or extra markers/sensors. Which method is used is **not yet decided**.

A task is represented as:

```text
task_id:      T01
pickup_node:  P1
dropoff_node: D1
priority:     1
status:       open | assigned | in_progress | complete
```

For a fixed board, a predefined sequence of turns is enough. Graph search (BFS / Dijkstra) only becomes relevant if the board gains alternate paths **and** the robot knows its current node. Path planning is **not implemented**.

---

## Status message format (proposed)

Each robot periodically reports:

```text
robot_id, timestamp_or_sequence, route_node, state,
current_task, requested_junction, obstacle_stop, battery_level
```

The coordinator checks **message freshness** — a stale report must not be treated as a live one.

### Example coordination event

1. R1 and R2 both request junction `J2`.
2. The coordinator applies a shared deterministic rule (*example:* lower robot ID wins a simultaneous request).
3. R1 is granted the junction; R2 holds outside it.
4. R1 reports `J2` cleared.
5. The coordinator releases R2.

The lower-ID rule is an **example**. The team still has to choose and implement the real rule.

---

## The interface

The interface is a **monitor**, not a hidden brain. It visualizes telemetry and coordinator decisions. If the robot-level logic is being presented as autonomous, the interface must not be quietly making movement decisions.

Intended views:

- **Board map** — the route graph with both robots' current node, and junction lock state (free / granted / contested).
- **Robot panels** — one per robot: state (`FOLLOW_LINE`, `OBSTACLE_STOP`, `REQUEST_JUNCTION`, `WAIT`, `CROSS`), current task, obstacle flag, battery, last-seen age.
- **Task board** — tasks by status: open → assigned → in progress → complete.
- **Event log** — timestamped coordination decisions with the rule that fired, so a grant or a yield can be explained during the demo.
- **Link health** — per-robot staleness indicator, making a dropped connection visible instead of silent.

Status: **design goal, not built.**

---

## Communication

The ESP32s can talk over a local Wi-Fi network to a laptop running the coordinator and interface.

Important: a normal laptop does **not** receive ESP-NOW frames directly. If ESP-NOW is kept for robot-to-robot messaging, a third ESP32 is needed as a **gateway** to bridge those frames to the laptop.

Transport and coordinator host are **not yet confirmed**.

---

## Current status

### Reported as tested or addressed

- Chassis and motor setup
- Motor directions calibrated
- Line following attempted on both robots (with some failed IR channels)
- Ultrasonic obstacle stop tested on **one** robot
- Battery switch added
- ESP32 flash / startup issues addressed

### Not implemented or not confirmed

- Two-way status exchange between both robots and a coordinator
- Junction right-of-way and task-priority logic
- Route-node identification
- Graph-based path selection
- Live interface connected to real telemetry
- Physical object pickup / drop-off
- U-turn behaviour on the final board

---

## LDM concepts demonstrated

- Digital input/output — sensors and motor-driver pins
- Boolean logic and decision tables — sensor patterns mapped to movement states
- Finite-state machines — `FOLLOW_LINE`, `OBSTACLE_STOP`, `REQUEST_JUNCTION`, `WAIT`, `CROSS`
- GPIO, timers, and PWM motor control
- Serial / wireless communication and message formats
- Feedback control — sense, compare, correct, repeat
- Embedded hardware and software integration

Terminology should be aligned with the instructor's actual unit names rather than presented as confirmed syllabus headings.

---

## Repository layout

```
RouteSync_Claude_Context.md   Project context and ground truth for assistance
main.py                       Placeholder — coordinator entry point to come
README.md                     This file
```

Firmware, coordinator, and interface code are not in the repository yet.

---

## Origin note

The idea was inspired by an SIH 2026 problem statement the team referred to as **NWIS**. The closest match found, *SIH26121 eRTMAC-NWIS*, is a nearby-wells intelligence and decision-support platform for drilling operations — not robot-fleet coordination. The transferable inspiration is the **shared dashboard / status / alert pattern**, not the domain. If a different problem statement was meant, its ID should be confirmed.

---

## Terminology

Use: *line following*, *obstacle stopping*, *shared coordination*, *junction priority*.

Avoid until the implementation supports it: *rerouting*, *object pickup*, *swarm*, *decentralized*.
