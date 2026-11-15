# RouteSync Project Context

Use this file as the source of context when helping with the LDM course project. It summarizes what the team has said so far. Treat hardware and progress notes as reported by the team, not as independently verified facts.

## Project identity

- **Working name:** RouteSync
- **Suggested formal title:** *RouteSync: An ESP32-Based Two-Robot Warehouse Coordination System*
- **Course:** Logic Design and Microprocessor (LDM), second year
- **Current team scope:** Two mobile robots, not three
- **Demonstration setting:** A small warehouse layout on a board, with a prepared route marked using dark tape on a light surface

The project began as a decentralized swarm-robotics concept. The team is now considering a **shared coordinator** that receives status from both robots, resolves task or junction priority, and sends decisions back. Describe this as **hybrid coordination** if each ESP32 still handles line following and immediate obstacle stopping locally. Do not call it fully decentralized if a central coordinator makes the task and right-of-way decisions.

## Project goal

Demonstrate two small robots that can:

1. Follow a known route using an infrared line sensor array and differential-drive motors.
2. Stop locally when an ultrasonic sensor detects an obstacle ahead, then resume when it is clear.
3. Share their status with a common coordinator.
4. Use a shared priority rule for a task or a narrow junction so the robots do not enter the same space at the same time.
5. Display robot state, task ownership, and coordination events in an interface.

The interface and shared coordinator are current design goals. Do not claim that they are implemented or tested unless the team provides new evidence.

## Reported hardware

The shared project chat describes the following parts for each robot:

- ESP32 development board
- L298N motor driver
- Four-wheel chassis with DC gear motors
- IR line-sensor array, described as a five-channel TCRT5000-style strip
- HC-SR04 ultrasonic sensor at the front
- Battery pack, previously described as a 7.4 V two-cell lithium-ion pack, with a switch

The chat also reports that some IR channels are faulty. It reports line-following tests on two robots and an ultrasonic stop test on one robot. Confirm the actual parts, number of working sensors, wiring, and behavior on each individual robot before giving exact pin mappings or claiming parity between them. Earlier code discussions included robot-specific pin and sensor configurations, so do not assume both robots have identical wiring.

No gripper or pickup mechanism has been described. Until one is confirmed, describe pickup/drop-off as simulated, marker-based, or planned rather than physical object handling.

## How movement and sensing work

- The robot does not create its route. The team lays down dark tape on a light floor or board.
- An IR sensor shines infrared light at the surface and measures reflected light. Dark tape commonly reflects less IR than a light surface. Sensor output polarity depends on the module and wiring, so test whether the actual line appears as HIGH or LOW before changing firmware.
- The ESP32 estimates whether the line is left, centered, or right relative to the sensor array.
- The L298N switches motor current under ESP32 control. The ESP32 cannot power the motors directly from GPIO pins.
- The robot turns by changing the relative speed/direction of its left and right wheel groups. This is differential drive. Opposite wheel directions can pivot the robot, but a U-turn also requires suitable floor space and firmware that can turn and reacquire the tape.
- The ultrasonic sensor supports local obstacle stopping. It does not by itself choose a detour or plan a route around an obstacle.

## Route, position, and tasks

The IR array can follow the tape, but it does not by itself provide continuous coordinates or identify which identical junction the robot is at. For the coordinator to understand position, the route needs identifiable nodes such as `START`, `J1`, `P1` (pickup), `J2`, and `D1` (drop-off). Those may be identified using distinct tape patterns, a known junction-count/route sequence, or additional markers/sensors if needed.

A task could be represented as:

```text
task_id: T01
pickup_node: P1
dropoff_node: D1
priority: 1
status: open | assigned | in_progress | complete
```

For a simple fixed board, store a predefined sequence of turns. If the board has alternate paths and the robot/coordinator knows the current node, a graph search such as BFS or Dijkstra could select a route. Do not claim path planning is already implemented based only on the current conversation.

## Proposed shared-coordinator design

Recommended split:

- **On each robot:** line following, motor control, and immediate ultrasonic obstacle stop.
- **Shared coordinator:** combines both robot status reports, assigns or sequences tasks, resolves junction right-of-way, and sends grants/yield instructions.
- **Interface:** visualizes telemetry and decisions for the demo. It should monitor the system rather than secretly make robot movement decisions if the team is presenting the robot-level logic as autonomous.

Possible status fields:

```text
robot_id, timestamp_or_sequence, route_node, state,
current_task, requested_junction, obstacle_stop, battery_level
```

An example coordination event:

1. R1 and R2 both request junction J2.
2. The coordinator applies a shared deterministic rule, such as lower robot ID wins a simultaneous request.
3. R1 receives permission; R2 waits outside the junction.
4. R1 reports that it has cleared J2.
5. The coordinator releases R2 to proceed.

This lower-ID rule is an example only. The team must select and implement its actual rule. The coordinator should check message freshness. If a required peer/coordinator status becomes stale or unavailable, robots should wait safely rather than assume the junction is clear. Immediate obstacle stopping should remain local and should work even if the coordinator link is down.

### Network choice

The ESP32s can communicate through a local Wi-Fi network to a laptop or local computer running the coordinator and dashboard. A normal laptop does not directly receive ESP-NOW messages; if ESP-NOW is retained, an ESP32 gateway is needed to bridge those messages to the laptop. The team has not yet confirmed which transport or coordinator hardware/software it will use.

## Reported progress and planned work

### Reported as tested or addressed in the shared chat

- Chassis and motor setup
- Motor directions calibrated
- Line following tried on two robots, with some failed IR sensor channels
- Ultrasonic obstacle stop tested on one robot
- Battery switch added
- ESP32 flash/startup issues addressed

### Not confirmed as implemented

- Two-way status exchange between both robots and a shared coordinator
- Junction right-of-way / task-priority logic
- Route-node identification or graph-based path selection
- Live interface connected to robot telemetry
- Physical object pickup/drop-off
- U-turn behavior on the final board

## LDM connections

Common Logic Design and Microprocessor concepts that this project can demonstrate include:

- Digital inputs and outputs (sensors and motor-driver pins)
- Boolean logic and decision tables (sensor patterns to movement states)
- Finite-state machines (for example `FOLLOW_LINE`, `OBSTACLE_STOP`, `REQUEST_JUNCTION`, `WAIT`, `CROSS`)
- GPIO, timers, and PWM motor control
- Serial/wireless communication and message formats
- Feedback control (sense, compare, correct, repeat)
- Embedded hardware/software integration

The exact LDM syllabus is not available. Align these terms to the instructor’s unit names rather than presenting them as confirmed syllabus headings.

## Important NWIS context

The user said the project idea was taken from an SIH 2026 problem statement called NWIS. The SIH 2026 item found in prior research, **SIH26121 eRTMAC-NWIS**, is a nearby-wells intelligence and decision-support platform for drilling operations, not a robot-fleet coordination system. The transferable inspiration may be the shared dashboard/status/alert pattern. If the team meant a different SIH problem statement, ask them to confirm its ID or title before describing the relationship.

## Guidance for future assistance

- Explain technical ideas for a beginner, with plain language and concrete examples.
- Separate **reported tested**, **proposed**, **planned**, and **unknown** items.
- Do not invent results, performance figures, component pins, sensor counts, costs, team contributions, or features.
- If exact wiring or code behavior matters, ask for the current code or a wiring photo; earlier robots may differ.
- Prefer accurate terms: “line following,” “obstacle stopping,” “shared coordination,” and “junction priority.” Use “rerouting,” “object pickup,” “swarm,” or “decentralized” only when the implementation supports those claims.
- When proposing project changes, preserve the LDM learning goals and provide a way to demonstrate the logic and test the behavior.
