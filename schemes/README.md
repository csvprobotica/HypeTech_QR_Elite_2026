## 🔧 Mechanical Design Justification

**Chassis:** Built with LEGO-compatible Technic pieces (perforated beams, pins, and connectors), mounted on a custom-cut flat base plate that acts as a rigid platform for all subsystems *(based on photos — please confirm the exact base material: acrylic or wood)*. This hybrid approach combines the modularity of Technic pieces with the rigidity of a custom base, allowing components to be rearranged quickly without losing structural stability.

**Steering system:** The robot has 4 wheels — 2 front wheels (steering) and 2 rear wheels (propulsion), using **Ackermann steering geometry**: the outer wheel turns less than the inner wheel on every curve, which prevents tire scrub and reduces wear/friction during turns.
- **Why Ackermann:** It was evaluated against alternatives such as differential steering, but that option was ruled out because it would require independently controlling the speed of each wheel on the same axle, adding both mechanical and software complexity with no real benefit for the competition's track layout. Ackermann geometry, on the other hand, integrated naturally with the motors and parts available in the SPIKE Prime kit, and its underlying principle (inner wheel turning more than the outer wheel) reduces lateral tire scrub in curves, resulting in a cleaner turn and less wear across repeated track tests.

**Drivetrain and propulsion motor:**
- Motors: 2 in total — 1 traction motor (propulsion) and 1 steering motor.
- Gear ratio: direct drive (LEGO SPIKE Prime large motor coupled directly to the rear wheels, with no additional gear reduction).
- Torque vs. speed justification: traction speed (700°/s, see Software section) was prioritized over additional torque from a gear reduction, since the competition track does not require carrying extra weight or climbing slopes, but rather completing the circuit with consistent, repeatable turns.

**Mechanical stability:**
- Total robot weight: 591 g. Being lightweight relative to the chassis size, the center of gravity stays low because the heaviest components (hub and motors) are mounted close to the base, which favors stability through tight turns at the configured operating speed.

**Design iterations:**
- Previous version: an earlier stage identified a rear traction fault initially attributed to the code, but which turned out to be a mechanical/electrical issue in the traction system (see the "detected failures" section below).
- Current version: after fixing the traction issue, the robot maintains consistent turning behavior as expected by the navigation algorithm.



---

## ⚡ Power and Sensor Architecture

**Hub / main controller:** LEGO SPIKE Prime Hub — chosen over the LEGO Mindstorms Robot Inventor because it allows direct programming with **Pybricks** (an open-source firmware that replaces LEGO's official software and allows the robot's control logic to be written in pure Python, with more low-level control than the official block-based editor).

**Power budget:**

| Component | Voltage | Estimated current | Consumption |
|---|---|---|---|
| Main battery | 7.3 V (2100 mAh, Li-Po — official LEGO Technic Large Hub battery) | — | — |
| Propulsion motor | 7.2 V (runs off the hub's battery) | Not specified by the manufacturer | — |
| Steering motor | 7.2 V (runs off the hub's battery) | Not specified by the manufacturer | — |
| Vision sensor (Matrix Robotics) | Not applicable currently — sensor installed but not used in the program | — | — |
| Ultrasonic and color sensors | 7.2 V (powered from the hub) | Not specified by the manufacturer | — |
| Hub / controller | 7.3 V (internal Li-Po battery, 2100 mAh) | — | — |

Estimated runtime: not formally measured; LEGO reports over 500 charge cycles for the hub's battery before its capacity drops below 40%.

**Sensors identified (confirmed from code, updated version):**
- **3 ultrasonic sensors:** left (port A), right (port B), and **front** (port F, added in this version) — the front sensor detects obstacles ahead and is now also used to determine when to stop at the end of the run.
- **1 color sensor** (port E) — installed but **no longer used** in this version (it previously detected the finish line; that role is now handled by the front sensor).
- The Matrix Robotics vision sensor ("I-Vision") remains installed but is not used in the navigation program.

**Calibration:**
- Steering: the robot self-calibrates on startup — the steering motor finds its physical end-stops and calculates the midpoint as the "center" position (0°). In this version, the **order in which the end-stops are searched is reversed depending on the mode** (clockwise/counterclockwise), so the final centering movement always ends up coming from the side the robot is about to turn toward.
- Sensor readings: a **moving-average filter** of the last 3 readings from each sensor (`TAMANO_FILTRO = 3`) is applied to smooth out the typical noise of ultrasonic sensors before making decisions.
- Front reference distance: at the start of each run, the initial front distance is stored; it is later used to determine when to stop during the final sequence.
- Recalibration frequency: steering is automatically recalibrated on every run of the program (by calling `centrar_direccion()`).


**Failure points considered:**
- If the left or right ultrasonic sensor fails, the robot would lose the distance reference on that side, affecting corner counting and path correction. If the front sensor fails, both the emergency protection and the end-of-run detection are lost. There is no redundancy between sensors in the current version (the color sensor, although installed, no longer serves as a backup). This dependency is a known risk, also documented in the risk analysis table.

**Real failure detected and resolved during testing:**
The rear traction motor was not turning as expected due to insufficient power. At first the issue was suspected to be in the code (control logic), but after investigation it was identified that the root cause was mechanical/electrical in the rear traction system, not software. Once that traction issue was fixed, turning behavior returned to normal and the failure did not reoccur.

---

## 🧠 Software Architecture (expanded)

**Programming environment:** The robot is programmed in **Python using Pybricks**, running directly on the LEGO SPIKE Prime Hub. Pybricks was chosen over LEGO's official block-based editor because it allows writing more complex control logic (nested conditionals, state handling, direct sensor management) with the flexibility of a full text-based language.

**Code structure (`src/`):**
```
src/
└── main.py — Main program: wall-following with corner counting,
    automatic steering calibration, sensor filtering, front emergency
    protection/maneuver, and a final sequence based on the front sensor.
```
*(Note: currently the whole project logic lives in a single file. If you plan to upload multiple files, it can be split into modules such as `sensors.py`, `navigation.py`, and `main.py` for better modularity — optional, but it adds points under the "code modularity" criterion. Important: make sure this file is uploaded directly to `src/` with a `.py` extension, not as a README.md.)*

**Navigation logic (rule-based, updated version — unifies clockwise/counterclockwise using a single sign variable):**
1. **Sensor filtering:** each cycle averages the last 3 readings from the 3 ultrasonic sensors (left, right, front) to smooth out noise before deciding.
2. **Front protection (new):** if the front sensor detects an obstacle closer than 10 cm, the robot executes an emergency maneuver (see below) before continuing.
3. **Straight driving** (default): if no special condition is detected, the robot keeps steering centered and drives forward.
4. **Corner detection:** if the "primary" sensor for the current mode (right in clockwise, left in counterclockwise) detects a distance greater than 198 cm, a corner is counted. Before turning hard, the robot first performs a small **"widening" maneuver** (15°, 150 ms) steering away from the opposite side, then turns hard (45°) toward the corner side — this improves the entry angle into the turn.
5. **Soft correction (wall nearby):** if the secondary sensor detects less than 16 cm, the robot softly corrects (20°) toward the corner side; if the primary sensor detects less than 8 cm, it softly corrects toward the opposite side.
6. **Final sequence:** upon completing 12 corners (3 laps of 4 corners each), the robot stops, waits, moves forward a bit more, and stops for good once the **front** sensor again reads a distance smaller than the front distance stored at the start of the run (this replaces the color-line detection used in the previous version).


**Front emergency maneuver (new in this version):**
If the front sensor detects an obstacle closer than 10 cm: the robot stops, steers the wheels toward the side with more free space (comparing left vs. right distance), reverses for 400 ms in that direction, and then turns sharply (60°, stronger than the normal 45° turn) toward the opposite side before resuming forward motion — a "three-point" maneuver to get unstuck without relying on operator intervention.

**Control algorithm:**
- Type: rule-based control with fixed distance thresholds (not PID), using filtered readings (moving average) to reduce noise.
- Thresholds/constants used:
  - Corner distance (open corridor): > 198 cm
  - Wall-nearby distance (soft correction): < 16 cm
  - Wall-very-close distance: < 8 cm
  - Critical front distance (emergency braking): < 10 cm
  - Emergency reverse time: 400 ms
  - "Widening" angle before a corner: 15° (150 ms)
  - Hard turn angle (corner): 45° — soft turn angle: 20°
  - Recovery turn angle/speed (after emergency): 60° at 500°/s (250 ms)
  - Traction speed: 700°/s — hard turn: 400°/s — soft turn: 280°/s
  - Reading filter: average of the last 3 samples per sensor
  - Goal: 12 corners detected (3 laps × 4 corners)
- Why this algorithm was chosen over another: the fixed-threshold approach (rather than PID) was kept for its simplicity of implementation and debugging, and layers of robustness were added over time (noise filtering, front protection, recovery maneuver) as real problems were discovered on the track, instead of redesigning the whole algorithm from scratch.

**Edge case handling:**
- **Sensor noise:** filtered with a 3-reading moving average before making any decision, preventing a single bad reading from triggering an incorrect turn or corner count.
- **Duplicate corner counting:** edge detection is used (`esquina_abierta and not esquina_abierta_anterior`), counting a corner only once, at the moment the gap opens up.
- **Obstacle directly ahead:** covered by the new front emergency maneuver (brake, reverse toward the side with more space, recovery turn).
- **Loss of wall reference on both sides:** the program does not yet have an explicit case for this beyond driving straight — identified as a future improvement (active search behavior).

**Testing and tuning process:**
- The distance thresholds (198 cm, 16 cm, 8 cm) and turn angles (45° and 20°) were iteratively tuned on the track, testing the robot's behavior at different corners of the circuit until corner counting was reliable and the robot did not hit walls during soft corrections.

**Validation metrics:**
- Main metric: number of corners consistently completed (12, equivalent to 3 laps). Exact lap time and success rate were not formally measured with a stopwatch — the code includes a `StopWatch` ready to be used in a future iteration to automatically capture this metric.

---

## ⚙️ Systems Thinking and Engineering Decisions

**Interaction between subsystems:**
The 3 ultrasonic sensors feed directly into the main program's decision logic, which controls the steering motor (turn angle and speed) and the traction motor (driving speed, including braking/reverse during the emergency maneuver) in real time. A delayed or incorrect reading from any sensor immediately affects the decision made, which is why the reliability of the ultrasonic sensors and the noise filter are critical for correct corner counting, path correction, and front protection.

**Project constraints:**
- Time: development and testing were limited by the competition date (WRO Panama), which directly influenced the decision not to integrate the Matrix Robotics vision sensor in this version.
- Budget: limited to components already available in the LEGO SPIKE Prime kit and the Matrix Robotics module, with no additional hardware purchased.
- Competition rules: [TO COMPLETE: maximum dimensions and weight allowed per the official WRO Future Engineers/Panama rulebook — send me the link to the rulebook if you have it handy and I'll check it right away]

**Key trade-offs:**
| Decision | Option A | Option B | We chose | Why |
|---|---|---|---|---|
| Hub/software | LEGO Mindstorms Robot Inventor (official app) | LEGO SPIKE Prime + Pybricks | SPIKE Prime + Pybricks | Allows programming in pure Python with more low-level control than the block editor, enabling more complex logic (state handling, custom thresholds) |
| Navigation sensor | Matrix Robotics vision sensor (I-Vision) | 3 ultrasonic sensors (left/right/front) | 3 ultrasonic sensors | Due to time constraints before the competition, it wasn't possible to reliably program and integrate the vision camera; it was left as a planned improvement for a future iteration, prioritizing a proven, stable solution with ultrasonic sensors |
| Chassis | Pure LEGO Technic pieces | Custom base + Technic pieces (hybrid) | Hybrid (custom base + Technic) | Combines the rigidity of a custom-cut flat base with the modularity and fast iteration of Technic pieces for mounting motors and sensors |

**Risk analysis:**
| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| Mechanical/electrical traction failure (already happened once) | Medium | High — robot doesn't move or turns incorrectly | Physical inspection of traction connections before each run; root cause was fixed once detected |
| Ultrasonic sensor failure during competition | Low-Medium | High — loss of reference on one side or in front, incorrect corner counting or braking | Moving-average filter reduces false readings from noise; no physical sensor redundancy yet (future improvement, possibly covered by the not-yet-integrated Matrix Robotics sensor) |
| Insufficient battery charge before a round | Low | High — robot could stop mid-run | Battery charge check at program startup (the code prints voltage and current on boot: `hub.battery.voltage()`) |

**Iteration cycles:**
- Iteration 1: it was discovered that the rear traction motor wasn't turning as expected due to insufficient power. A programming error was initially suspected, but after debugging it was confirmed the root cause was mechanical/electrical in the traction system; once fixed, turning behavior fully normalized.
- Iteration 2: finish-line detection via the color sensor was replaced with a **real front ultrasonic sensor** (port F), storing the initial distance and comparing it at the end of the run — more robust against variable lighting conditions that could affect the color reading.
- Iteration 3: a **moving-average filter** (3 readings) was added to the 3 ultrasonic sensors after noticing that noisy individual readings were causing false turns or corner counts.
- Iteration 4: a **front emergency maneuver** (brake, reverse toward the side with more space, recovery turn) was added after noticing the robot could get stuck head-on against an obstacle with no automatic recovery mechanism.

---

## 🔁 Reproducibility

**Bill of materials (BOM):**
| Component | Model/Specification | Quantity | Link |
|---|---|---|---|
| Controller hub | LEGO SPIKE Prime Hub (Technic Large Hub) | 1 | — |
| Battery | LEGO Technic Large Hub rechargeable battery, 7.3V / 2100 mAh Li-Po | 1 | — |
| Traction motor | LEGO SPIKE Prime large motor | 1 | — |
| Steering motor | LEGO SPIKE Prime medium motor | 1 | — |
| Ultrasonic sensor | LEGO SPIKE Prime Ultrasonic Sensor | 3 | — |
| Color sensor (not currently used) | LEGO SPIKE Prime Color Sensor | 1 | — |
| Vision sensor (not currently used) | Matrix Robotics I-Vision | 1 | — |
| Wheels | Technic wheels (2 large rear + 2 small front) | 4 | — |
| Chassis/base | LEGO Technic pieces + custom-cut flat base | — | — |
| Miscellaneous structural parts | LEGO Technic beams, pins, connectors | — | — |

**Steps to reproduce this robot:**
1. Assemble the chassis following the diagrams in `models/`.
2. Wire it up following the diagrams in `schemes/`.
3. Load the `main.py` file from `src/` onto the LEGO SPIKE Prime Hub using the **Pybricks** editor/extension (either via the Pybricks Code web app or the VS Code integration).
4. Calibrate the sensors following the method described above (steering self-calibrates on program startup; the front sensor automatically stores its reference distance at the start of each run).
5. Press the hub's right button for clockwise mode, or the left button for counterclockwise mode, and the robot will start the run automatically.

**Version control:**
- Full commit history is available in this repository.
- It's recommended to create a tag/release (e.g., `v1.0-competition`) on GitHub right before the competition, to mark the exact version of the robot used that day.

**License:** MIT License. This is the simplest and most standard open-source license: it allows anyone to freely use, copy, or modify the project, giving credit to the original team, with no additional restrictions. Recommended for being clear and widely recognized by evaluators.

---
