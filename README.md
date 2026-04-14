# Vizzy Humanoid Robot - ETH Board Configuration Files

[![YARP](https://img.shields.io/badge/YARP-3.x-blue)](https://www.yarp.it/)
[![Platform](https://img.shields.io/badge/Platform-Linux-lightgrey)](https://ubuntu.com/)
[![Robot](https://img.shields.io/badge/Robot-Vizzy-informational)](https://vislab.isr.tecnico.ulisboa.pt/robots/vizzy/)
[![Status](https://img.shields.io/badge/Status-Active%20Development-yellow)]()

**🚧 Under Active Development 🚧**

This repository contains the YARP `yarprobotinterface` configuration files for the Vizzy humanoid robot's Ethernet (ETH)-based motion control boards. It covers the **head** (eyes and neck) and **left arm** (shoulder) subsystems, providing hardware-level configuration for motor control, encoder mapping, joint coupling, calibration, and YARP port wiring.

> **Note on YARP documentation:** YARP's own documentation for `embObjMotionControl`, calibrators, and related devices is notoriously sparse. This README attempts to bridge that gap by providing detailed explanations of every configuration parameter and its effect on robot behaviour.

---

## Table of Contents

- [Hardware Overview](#hardware-overview)
- [Repository Structure](#repository-structure)
- [Configuration Architecture](#configuration-architecture)
- [Launching the Robot Interface](#launching-the-robot-interface)
- [Configuration Files Reference](#configuration-files-reference)
  - [Top-Level Files](#top-level-files)
  - [General Parameters](#general-parameters-generalxml)
  - [Electronics (ETH Board)](#electronics-eth-board-hardwareelectronics)
  - [Mechanicals (Joint Physics)](#mechanicals-joint-physics-hardwaremechanicals)
  - [Motor Control](#motor-control-hardwaremotorcontrol)
  - [Control Board Wrappers](#control-board-wrappers-wrappersmotorcontrol)
  - [Calibrators](#calibrators-calibrators)
- [Joints Reference](#joints-reference)
- [YARP Concepts Explained](#yarp-concepts-explained)
- [Eye Coupling Explained](#eye-coupling-explained)
- [Board Types: MC4plus vs EMS4](#board-types-mc4plus-vs-ems4)
- [Documentation & Citation](#documentation--citation)
- [Reporting Issues](#reporting-issues)

---

## Hardware Overview

Vizzy's motion control relies on IIT-developed Ethernet motion control boards, each responsible for a subset of joints. Every board is reachable over a local Ethernet network at a fixed IP address.

| Board ID | Board Type | IP Address | Part | Joints Controlled | Wrapper Joint Indices |
|----------|-----------|-----------|------|-------------------|-----------------------|
| EB04 | MC4plus | `10.1.3.4` | Head (Eyes) | eye_tilt, version, vergence | 2, 3, 4 |
| EB05 | MC4plus | `10.1.3.5` | Head (Neck) | neck_pan, neck_tilt | 0, 1 |
| EB06 | EMS4 | `10.1.3.6` | Left Arm (Shoulder) | left_shldr_scapula, left_shldr_flection, left_shldr_abduction | 0, 1, 2 |

**PC104 Host** (control PC) communicates with all boards at `10.1.3.104:12345`.

---

## Repository Structure

```
vizzy-eth-config-files/
│
├── vizzy_all.xml                             # Top-level robot configuration (entry point)
├── general.xml                               # Global motion control parameters
├── yarprobotinterface.ini                    # yarprobotinterface module config
├── yarpmotorgui.ini                          # yarpmotorgui config
│
├── hardware/
│   ├── electronics/                          # ETH board network & communication parameters
│   │   ├── vizzy-desktop.xml                 # Host PC (PC104) communication settings
│   │   ├── head-eb04-j2_4-eln.xml            # EB04 board (eyes): Ethernet config
│   │   ├── head-eb05-j0_1-eln.xml            # EB05 board (neck): Ethernet config
│   │   └── left_arm-eb06-j0_2-eln.xml        # EB06 board (left arm): Ethernet config
│   │
│   ├── mechanicals/                          # Joint physical properties & encoder mapping
│   │   ├── head-eb04-j2_4-mec.xml            # Eye joints: axes, encoders, coupling matrices
│   │   ├── head-eb05-j0_1-mec.xml            # Neck joints: axes, encoders, limits
│   │   └── left_arm-eb06-j0_2-mec.xml        # Shoulder joints: axes, encoders, 2FOC config
│   │
│   └── motorControl/                         # Motor control devices (PID gains, limits, mode)
│       ├── head-eb04-j2_4-mc.xml             # Eyes: PID gains, operating limits, control modes
│       ├── head-eb04-j2_4-mc_service.xml     # Eyes: actuator/encoder port mapping
│       ├── head-eb05-j0_1-mc.xml             # Neck: PID gains, operating limits, control modes
│       ├── head-eb05-j0_1-mc_service.xml     # Neck: actuator/encoder port mapping
│       ├── left_arm-eb06-j0_2-mc.xml         # Arm: PID gains, operating limits, 2FOC PIDs
│       └── left_arm-eb06-j0_2-mc_service.xml # Arm: CAN/AEA/ROIE port mapping
│
├── wrappers/
│   └── motorControl/                         # YARP controlboardwrapper2 devices
│       ├── head-mc_wrapper.xml               # Aggregates EB04+EB05 into /vizzy/head port
│       └── left_arm-mc_wrapper.xml           # Exposes EB06 as /vizzy/left_arm port
│
├── calibrators/                              # Joint calibration routines
│   ├── head-calib.xml                        # Head (5 joints) calibration
│   └── left_arm-calib.xml                    # Left arm (3 joints) calibration
│
└── models/                                   # Mechanical CAD models (reference only)
    ├── Lock_Piece_Model.dwg
    ├── Lock_Piece_Model.stl
    ├── Vizzy_Head_Boards_Holder_Model.stl
    └── Drawing2.dwg
```

### File Naming Convention

All hardware files follow the convention:

```
<part>-<board_id>-<joint_range>-<type>.xml
```

- **`<part>`**: Robot subsystem (e.g., `head`, `left_arm`)
- **`<board_id>`**: Board identifier (e.g., `eb04`, `eb05`, `eb06`)
- **`<joint_range>`**: Joint indices this board controls within the YARP part (e.g., `j0_1` = joints 0 and 1, `j2_4` = joints 2 to 4)
- **`<type>`**: File role -- `eln` (electronics), `mec` (mechanicals), `mc` (motor control), `mc_service` (service mapping)

---

## Configuration Architecture

YARP's `yarprobotinterface` loads `vizzy_all.xml`, which uses XInclude (`xi:include`) to assemble the full robot configuration from modular sub-files. The dependency tree is:

```
yarprobotinterface.ini
└── vizzy_all.xml
    ├── hardware/electronics/vizzy-desktop.xml    (global PC host params)
    │
    ├── hardware/motorControl/head-eb04-j2_4-mc.xml
    │   ├── general.xml
    │   ├── hardware/electronics/head-eb04-j2_4-eln.xml
    │   ├── hardware/mechanicals/head-eb04-j2_4-mec.xml
    │   └── hardware/motorControl/head-eb04-j2_4-mc_service.xml
    │
    ├── hardware/motorControl/head-eb05-j0_1-mc.xml
    │   ├── general.xml
    │   ├── hardware/electronics/head-eb05-j0_1-eln.xml
    │   ├── hardware/mechanicals/head-eb05-j0_1-mec.xml
    │   └── hardware/motorControl/head-eb05-j0_1-mc_service.xml
    │
    ├── hardware/motorControl/left_arm-eb06-j0_2-mc.xml
    │   ├── general.xml
    │   ├── hardware/electronics/left_arm-eb06-j0_2-eln.xml
    │   ├── hardware/mechanicals/left_arm-eb06-j0_2-mec.xml
    │   └── hardware/motorControl/left_arm-eb06-j0_2-mc_service.xml
    │
    ├── wrappers/motorControl/head-mc_wrapper.xml
    ├── wrappers/motorControl/left_arm-mc_wrapper.xml
    ├── calibrators/head-calib.xml
    └── calibrators/left_arm-calib.xml
```

Each `embObjMotionControl` device is self-contained: it includes its own general settings, board-specific network config, joint mechanical properties, and service/port mapping. The `controlboardwrapper2` devices then aggregate individual board devices into unified, user-facing YARP ports.

---

## Launching the Robot Interface

Ensure YARP is running and the robot's Ethernet network is reachable before launching.

**1. Start the YARP name server** (if not already running):
```bash
yarpserver
```

**2. Launch the robot interface** from the repository root:
```bash
yarprobotinterface
```

This will:
- Connect to all ETH boards via UDP/Ethernet
- Instantiate all `embObjMotionControl` devices
- Create YARP control ports via `controlboardwrapper2`
- Run the calibration routines (unless `skipCalibration` is set to `true`)

**3. Open the motor GUI** (optional, useful for manual joint control and inspection):
```bash
yarpmotorgui
```

The GUI will display and allow control of the `head` and `left_arm` parts as configured in `yarpmotorgui.ini`.

**YARP ports created after startup:**

| Port | Description |
|------|-------------|
| `/vizzy/head/state:o` | Head joint state publisher |
| `/vizzy/head/command:i` | Head joint command subscriber |
| `/vizzy/head/rpc:i` | Head RPC control interface |
| `/vizzy/left_arm/state:o` | Left arm joint state publisher |
| `/vizzy/left_arm/command:i` | Left arm joint command subscriber |
| `/vizzy/left_arm/rpc:i` | Left arm RPC control interface |

---

## Configuration Files Reference

### Top-Level Files

#### `vizzy_all.xml` -- Robot Configuration Root

The root configuration file. Defines the robot identity and includes all device configurations via XInclude.

| Parameter | Value | Description |
|-----------|-------|-------------|
| `name` | `vizzy` | Robot name, used to identify the robot in YARP |
| `portprefix` | `vizzy` | Prefix for all YARP ports (e.g., `/vizzy/head`) |
| `build` | `1` | Configuration build version |

Currently active subsystems: HEAD (EB04 + EB05), LEFT ARM (EB06), and their respective calibrators. Cartesian control modules are present but commented out.

---

#### `yarprobotinterface.ini` -- Module Entry Point

Points `yarprobotinterface` to the main XML:

```ini
config ./vizzy_all.xml
```

#### `yarpmotorgui.ini` -- Motor GUI Configuration

| Parameter | Value | Description |
|-----------|-------|-------------|
| `robot` | `vizzy` | Robot name to connect to |
| `parts` | `(head left_arm)` | Parts displayed in the GUI |

---

### General Parameters (`general.xml`)

Included by every `embObjMotionControl` device. Sets system-wide motion control behaviour.

| Parameter | Value | Description |
|-----------|-------|-------------|
| `skipCalibration` | `false` | If `true`, skips joint calibration on startup. **Use with extreme caution** -- only set to `true` when you are certain the robot is already at a known safe position. |
| `useRawEncoderData` | `false` | If `true`, bypasses encoder scaling and coupling matrices, using raw encoder counts directly. Set to `false` for normal operation with physical units (degrees). |
| `useLimitedPWM` | `false` | If `true`, applies an additional software cap on PWM output, reducing motor effort. Useful for debugging or safe operation with fragile hardware. |
| `verbose` | `true` | Enables verbose debug logging from the motion control library. Produces detailed output useful during tuning or fault diagnosis. |

---

### Electronics (ETH Board) -- `hardware/electronics/`

These files configure the Ethernet communication parameters for each board. All boards share the same port (12345) on the `10.1.3.x` subnet.

#### `vizzy-desktop.xml` -- Host PC Parameters

Defines the control PC's identity on the network.

| Parameter | Value | Description |
|-----------|-------|-------------|
| `PC104IpAddress` | `10.1.3.104` | IP address of the host control PC |
| `PC104IpPort` | `12345` | UDP port used for communication with ETH boards |
| `PC104TXrate` | `1` | Packet transmit rate multiplier for the host |
| `PC104RXrate` | `5` | Packet receive rate multiplier for the host |

#### `head-eb04-j2_4-eln.xml` / `head-eb05-j0_1-eln.xml` / `left_arm-eb06-j0_2-eln.xml`

| Parameter | EB04 (Eyes) | EB05 (Neck) | EB06 (Arm) | Description |
|-----------|------------|------------|------------|-------------|
| `IpAddress` | `10.1.3.4` | `10.1.3.5` | `10.1.3.6` | Board IP address on the robot's internal Ethernet network |
| `IpPort` | `12345` | `12345` | `12345` | UDP port for host↔board communication |
| `type` | `mc4plus` | `mc4plus` | `ems4` | ETH board hardware type (see [Board Types](#board-types-mc4plus-vs-ems4)) |
| `maxSizeRXpacket` | `768` | `768` | `768` | Maximum size (bytes) of a UDP receive packet |
| `maxSizeROP` | `384` | `384` | `384` | Maximum size (bytes) of a single ROP (Remote Object Protocol) message within a packet |
| `period` | `1000` | `1000` | `1000` | Communication cycle period in microseconds (1 ms) |
| `maxTimeRXactivity` | `400` | `400` | `400` | Maximum time (µs) allowed for receive processing per cycle |
| `maxTimeDigitalOutput` | `300` | `300` | `300` | Maximum time (µs) for digital output processing per cycle |
| `maxTimeTXactivity` | `300` | `300` | `300` | Maximum time (µs) allowed for transmit processing per cycle |
| `TXrateOfRegularROPs` | `5` | `5` | `5` | How many communication cycles elapse between regular ROP transmissions (lower = more frequent updates) |
| `monitorEnable` | `true` | `true` | `true` | Enables the board's communication watchdog monitoring |
| `monitorTimeout` | `20` | `20` | `20` | Watchdog timeout (ms): if no packet is received within this window, the board flags a communication error |
| `monitorMissingReportPeriod` | `60` | `60` | `60` | Period (seconds) at which missed-packet reports are printed to the log |

> **ROP (Remote Object Protocol)**: YARP's low-level binary protocol used to read/write named objects on ETH boards in real time. Each ROP encodes a request or reply for a specific data field (e.g., motor position, PID output).

---

### Mechanicals (Joint Physics) -- `hardware/mechanicals/`

These files define the physical and kinematic properties of each joint: axis names, encoder resolutions, gear ratios, joint limits, and coupling between motors and joints.

#### Common Parameters (all boards)

| Parameter | Description |
|-----------|-------------|
| `MotioncontrolVersion` | Version of the motion control API. Must match the firmware on the ETH board. Currently `6`. |
| `Joints` | Number of controlled joints on this board |
| `AxisMap` | Sequential joint indices (typically `0 1 2 ...`) mapping this board's joints to the YARP device's axis list |
| `AxisName` | String names for each joint axis, used in YARP interfaces and the motor GUI |
| `AxisType` | Kinematic type of each axis. `"revolute"` = rotational joint (degrees). |
| `Encoder` | Encoder resolution in **counts per motor revolution**. For all Vizzy joints: **73255.5 counts/rev** with 512 CPR quadrature encoders and internal multiplication. |
| `fullscalePWM` | PWM value corresponding to 100% motor effort (maximum allowed duty cycle). Used internally for scaling. |
| `ampsToSensor` | Conversion factor from amperes to the current sensor's raw units (1000 = mA). |
| `Gearbox_M2J` | Gear ratio from motor shaft to joint. `1` = direct drive (no reduction between motor and encoder). |
| `Gearbox_E2J` | Gear ratio from encoder shaft to joint. `1` = encoder is on the joint axis directly. |
| `JointEncoderType` | Encoder technology. `"OPTICAL_QUAD"` = quadrature optical incremental encoder. |
| `useMotorSpeedFbk` | `1` = use motor speed (from encoder derivative) as additional feedback for velocity control. |
| `MotorType` | Motor technology. `"DC"` = brushed DC motor. |
| `Verbose` | Per-axis verbose flag (echoes the global `verbose` in `general.xml`). |

#### Joint Limits

Two sets of limits are defined per joint:

| Parameter | Description |
|-----------|-------------|
| `jntPosMin` / `jntPosMax` | **Software limits** (degrees): the operating range enforced by the firmware. The controller will not command the joint beyond these values. Defined in `*-mc.xml`. |
| `hardwareJntPosMin` / `hardwareJntPosMax` | **Hardware limits** (degrees): the physical range defined by mechanical hard stops. These are wider than software limits and represent the absolute physical boundary. Defined in `*-mec.xml`. |
| `rotorPosMin` / `rotorPosMax` | Rotor position limits (used with FOC/absolute encoders). `0 0` means unconstrained at the rotor level. |

#### `head-eb04-j2_4-mec.xml` -- Eye Joints

This board controls 3 coupled eye joints. See [Eye Coupling Explained](#eye-coupling-explained) for details on the coupling matrices.

| Joint | Axis Name | Soft Min (°) | Soft Max (°) | Hard Min (°) | Hard Max (°) |
|-------|-----------|-------------|-------------|-------------|-------------|
| 0 | `eye_tilt` | -30 | +30 | -40 | +40 |
| 1 | `version` | -30 | +30 | -40 | +40 |
| 2 | `vergence` | 0 | +30 | -40 | +40 |

> **version** = combined horizontal gaze direction (both eyes point the same way). **vergence** = eye convergence angle (how much the eyes converge toward a near object). See [Eye Coupling Explained](#eye-coupling-explained).

#### `head-eb05-j0_1-mec.xml` -- Neck Joints

| Joint | Axis Name | Soft Min (°) | Soft Max (°) | Hard Min (°) | Hard Max (°) |
|-------|-----------|-------------|-------------|-------------|-------------|
| 0 | `neck_pan` | -53 | +53 | -55 | +55 |
| 1 | `neck_tilt` | -18 | +37 | -40 | +40 |

No coupling (identity J2M/M2J matrices).

#### `left_arm-eb06-j0_2-mec.xml` -- Left Shoulder Joints

| Joint | Axis Name | Soft Min (°) | Soft Max (°) | Hard Min (°) | Hard Max (°) |
|-------|-----------|-------------|-------------|-------------|-------------|
| 0 | `left_shldr_scapula` | -18 | +18 | -95.5 | +8 |
| 1 | `left_shldr_flection` | -75 | +135 | +15 | +160 |
| 2 | `left_shldr_abduction` | 0 | +70 | -32 | +80 |

> Note: The hardware limits for the shoulder are asymmetric and reflect physical constraints of the scapula mechanism. The wide discrepancy between hardware and software limits on joints 0 and 1 indicates the arm has not yet been fully tuned -- software limits are more conservative until reliable operation is confirmed.

**2FOC Parameters** (left arm only, EMS4 board):

| Parameter | Value | Description |
|-----------|-------|-------------|
| `HasHallSensor` | `1` | Motors have Hall effect sensors for commutation detection |
| `HasTempSensor` | `0` | No motor temperature sensors installed |
| `HasRotorEncoder` | `1` | Rotor incremental encoder present (used for FOC speed feedback) |
| `HasRotorEncoderIndex` | `0` | No index pulse on the rotor encoder |
| `HasSpeedEncoder` | `0` | No dedicated speed encoder (speed is derived from rotor encoder) |
| `MotorPoles` | `8` | Number of magnetic pole pairs in each motor. Required for correct FOC commutation. |

---

### Motor Control -- `hardware/motorControl/`

Each `*-mc.xml` file is the main `embObjMotionControl` device definition. It assembles the general, electronics, mechanicals, and service configurations, then adds operating limits, control mode assignments, and PID gains.

#### `LIMITS` Group

| Parameter | Unit | Description |
|-----------|------|-------------|
| `jntPosMin` / `jntPosMax` | degrees | Software joint position limits. The firmware refuses commands outside this range. |
| `jntVelMax` | deg/s | Maximum joint velocity. Commands exceeding this are clamped. |
| `motorOverloadCurrents` | mA | Current threshold above which the motor is considered overloaded. Triggers a fault. |
| `motorNominalCurrents` | mA | Rated continuous current for the motor. Used in thermal models. |
| `motorPeakCurrents` | mA | Maximum instantaneous current allowed (short-term peak). |
| `motorPwmLimit` | PWM units | Maximum PWM duty cycle output to the motor driver. Hard limit regardless of PID output. |

**Limits summary by part:**

| Joint | jntPosMin (°) | jntPosMax (°) | jntVelMax (°/s) | Nominal I (mA) | Peak I (mA) | PWM Limit |
|-------|--------------|--------------|----------------|---------------|------------|-----------|
| eye_tilt | -30 | +30 | 1000 | 700 | 1500 | 3360 |
| version | -30 | +30 | 1000 | 700 | 1500 | 3360 |
| vergence | 0 | +30 | 1000 | 700 | 1500 | 3360 |
| neck_pan | -53 | +53 | 1000 | 800 | 1000 | 2000 |
| neck_tilt | -18 | +37 | 1000 | 800 | 1000 | 1333 |
| left_shldr_scapula | -18 | +18 | 1000 | 800 | 1000 | 2000 |
| left_shldr_flection | -75 | +135 | 1000 | 800 | 1000 | 2000 |
| left_shldr_abduction | 0 | +70 | 1000 | 800 | 1000 | 2000 |

#### `TIMEOUTS` Group

| Parameter | Unit | Description |
|-----------|------|-------------|
| `velocity` | ms | Time after which a velocity command is considered stale and the joint is stopped. Prevents runaway if the commanding process crashes or stops sending commands. |

#### `IMPEDANCE` Group

| Parameter | Unit | Description |
|-----------|------|-------------|
| `stiffness` | N·m/rad | Spring stiffness for impedance control mode. `0` = pure position mode (no spring). |
| `damping` | N·m·s/rad | Viscous damping for impedance control mode. `0` = no damping. |

All joints currently have `stiffness = 0` and `damping = 0`, meaning impedance control is disabled.

#### `CONTROLS` Group

Assigns a named PID group to each control mode per joint. Allows different joints or modes to use different tunings.

| Parameter | Description |
|-----------|-------------|
| `positionControl` | PID group used when the joint is in position control mode |
| `velocityControl` | PID group used when the joint is in velocity control mode |
| `mixedControl` | PID group used in mixed position/velocity mode |
| `torqueControl` | PID group used in torque control mode (left arm only) |
| `currentPid` | Low-level current controller (left arm only, runs inside the FOC loop) |
| `speedPid` | Low-level speed controller (left arm only, runs inside the FOC loop) |

#### PID Groups

Each named PID group (e.g., `POS_PID_DEFAULT`) defines a complete controller configuration:

| Parameter | Unit | Description |
|-----------|------|-------------|
| `controlLaw` | -- | Algorithm used: `minjerk` = minimum-jerk trajectory following, `torque` = direct torque control, `low_lev_current` / `low_lev_speed` = inner FOC loop controllers |
| `outputType` | -- | Type of output signal: `pwm` = direct PWM duty cycle to the motor driver |
| `fbkControlUnits` | -- | Units of the feedback signal: `metric_units` = physical units (degrees, N·m); `machine_units` = raw encoder counts or ADC values |
| `outputControlUnits` | -- | Units of the output: `metric_units` or `machine_units` |
| `kp` | PWM/deg | Proportional gain. Higher = stronger correction for position error, but risks oscillation. |
| `kd` | PWM/(deg/s) | Derivative gain. Damps oscillations by opposing velocity of error. |
| `ki` | PWM/(deg·s) | Integral gain. Eliminates steady-state error by accumulating past error. |
| `maxOutput` | PWM units | Absolute cap on PID output, independent of `motorPwmLimit`. |
| `maxInt` | PWM units | Anti-windup cap on the integral term accumulator. Prevents the integrator from winding up when saturated. |
| `stictionUp` | PWM units | Static friction compensation added when commanding in the positive direction. |
| `stictionDown` | PWM units | Static friction compensation added when commanding in the negative direction. |
| `kff` | -- | Feed-forward gain. Adds a fraction of the velocity reference directly to the output to improve tracking. |

**PID tunings by part:**

*Head -- Eyes (`POS_PID_DEFAULT`, all in metric/machine units):*

| Joint | kp | kd | ki | maxOutput | maxInt |
|-------|----|----|----|-----------|--------|
| eye_tilt | +500 | +10 | +10 | 2000 | 1000 |
| version | +500 | +10 | +50 | 2000 | 1000 |
| vergence | +500 | +10 | +50 | 2000 | 1000 |

*Head -- Neck (`POS_PID_DEFAULT`):*

| Joint | kp | kd | ki | maxOutput | maxInt |
|-------|----|----|----|-----------|--------|
| neck_pan | +50 | +50 | +10 | 2000 | 1000 |
| neck_tilt | +100 | +50 | +10 | 3000 | 1000 |

> `neck_tilt` has a higher `maxOutput` (3000 vs 2000) to overcome the gravitational load of the head.

*Left Arm -- Shoulder (`POS_PID_DEFAULT`):*

| Joint | kp | kd | ki | maxOutput | maxInt |
|-------|----|----|----|-----------|--------|
| left_shldr_scapula | +5 | 0 | 0 | 1333 | 1000 |
| left_shldr_flection | +1 | 0 | 0 | 1333 | 1000 |
| left_shldr_abduction | +750 | +2 | +3 | 1333 | 1000 |

> **Tuning note:** The arm PID gains have a `TODO` note in the source -- these values are imported from a previous robot version and have not been validated for Vizzy. They are likely to need significant re-tuning.

#### Additional PID Groups (Left Arm Only)

**`TRQ_PID_DEFAULT`** -- Torque control (currently unused, imported from iCub Lisboa v2.14.0):

| Parameter | Description |
|-----------|-------------|
| `ko` | Output offset (constant additive term to PID output) |
| `viscousPos` / `viscousNeg` | Viscous friction compensation coefficients (positive/negative motion direction) |
| `coulombPos` / `coulombNeg` | Coulomb (static) friction compensation |
| `velocityThres` | Velocity threshold (deg/s) below which Coulomb friction compensation is applied |
| `filterType` | Type of filter applied to the torque feedback signal |
| `ktau` | Torque constant: converts motor current to joint torque (N·m/A). Values: scapula=180, flection=464, abduction=463. |

**`2FOC_CUR_CONTROL`** -- Low-level current controller (runs on EMS4/FOC hardware):

| Parameter | Value | Description |
|-----------|-------|-------------|
| `controlLaw` | `low_lev_current` | Inner current loop for FOC. Runs at very high frequency inside the motor driver. |
| `kp` | 8 | Current loop proportional gain |
| `ki` | 2 | Current loop integral gain |
| `shift` | 10 | Fixed-point binary shift for internal gain representation on the DSP |
| `maxOutput` | 32000 | Maximum output in machine units (full scale of the current driver) |

**`2FOC_VEL_CONTROL`** -- Low-level speed controller (runs on EMS4/FOC hardware):

| Parameter | Value | Description |
|-----------|-------|-------------|
| `controlLaw` | `low_lev_speed` | Inner speed loop for FOC. |
| `kp` | 12 | Speed loop proportional gain |
| `ki` | 16 | Speed loop integral gain |
| `shift` | 10 | Fixed-point binary shift |
| `maxOutput` | 32000 | Maximum output in machine units |

#### `*-mc_service.xml` -- Actuator & Encoder Port Mapping

These files tell the firmware which physical connectors and CAN addresses each joint's actuator and encoders are wired to.

**Head boards (`mc4plus`)** -- `SERVICE` type: `eomn_serv_MC_mc4plus`

| Joint | Actuator Port | Encoder (ENCODER2) Port | Encoder Resolution |
|-------|--------------|------------------------|-------------------|
| eye_tilt | PWM @ `CONN:P5` | `qenc` @ `CONN:P5` | 512 CPR |
| version | PWM @ `CONN:P4` | `qenc` @ `CONN:P4` | 512 CPR |
| vergence | PWM @ `CONN:P3` | `qenc` @ `CONN:P3` | 512 CPR |
| neck_pan | PWM @ `CONN:P3` | `qenc` @ `CONN:P3` | 512 CPR |
| neck_tilt | PWM @ `CONN:P2` | `qenc` @ `CONN:P2` | 512 CPR |

> **`qenc`**: Quadrature encoder. The MC4plus board reads A/B phase signals and counts edges. Resolution of 512 CPR gives 2048 counts per revolution (×4 edge counting), further multiplied internally to reach the `Encoder` value of 73255.5.

> `ENCODER1` is listed as `none` for head joints because the MC4plus only uses one encoder per axis (the quadrature encoder on `ENCODER2`). ENCODER1 would be used for an additional absolute encoder at the joint level, which is not present on the head.

**Left arm board (`ems4`)** -- `SERVICE` type: `eomn_serv_MC_foc`

| Joint | Actuator | ENCODER1 (AEA) Port | ENCODER2 (ROIE) Port | AEA Resolution |
|-------|----------|--------------------|--------------------|---------------|
| left_shldr_scapula | `foc` @ `CAN1:1:0` | `aea` @ `CONN:P6` | `roie` @ `CAN1:1:0` | -4096 |
| left_shldr_flection | `foc` @ `CAN1:2:0` | `aea` @ `CONN:P7` | `roie` @ `CAN1:2:0` | -4096 |
| left_shldr_abduction | `foc` @ `CAN1:3:0` | `aea` @ `CONN:P8` | `roie` @ `CAN1:3:0` | +4096 |

- **`foc`**: Field Oriented Control driver board, addressed over CAN bus. `CAN1:X:0` = CAN bus 1, node ID X, channel 0.
- **`aea`** (Absolute Encoder with Analogue output): Absolute position encoder. Provides joint-level position without calibration. Resolution of ±4096 → 13-bit absolute. Negative sign indicates reversed counting direction for scapula and flection.
- **`roie`** (Rotor Index Encoder): Incremental encoder on the motor rotor, used for FOC speed/commutation feedback. Resolution of -14400 counts/rev (negative = reversed direction).
- **`tolerance`**: `0.703` degrees -- maximum acceptable discrepancy between ENCODER1 (AEA) and ENCODER2 (ROIE) readings. If the two encoders disagree by more than this, a fault is triggered.

**FOC CAN board firmware version:**

| Parameter | Value |
|-----------|-------|
| Protocol major/minor | 1.6 |
| Firmware major/minor/build | 3.3.3 |

> The EMS4 checks that connected FOC boards report matching protocol and firmware versions at startup. Mismatched firmware will prevent the arm from initializing.

---

### Control Board Wrappers -- `wrappers/motorControl/`

`controlboardwrapper2` devices aggregate multiple low-level `embObjMotionControl` devices into a single unified YARP interface port. This is the layer that clients (e.g., `yarpmotorgui`, cartesian controllers, user applications) connect to.

#### `head-mc_wrapper.xml`

| Parameter | Value | Description |
|-----------|-------|-------------|
| `device` | `controlboardwrapper2` | YARP wrapper device type |
| `name` | `/vizzy/head` | Root YARP port name for this part |
| `period` | `10` ms | Wrapper update period (how often state is broadcast) |
| `joints` | `5` | Total number of joints in this part |

**Network mapping** (how sub-devices are joined):

| Wrapper Joint | Axis Name | Source Device | Source Joint |
|--------------|-----------|---------------|-------------|
| 0 | `neck_pan` | `head-eb05-j0_1-mc` | 0 |
| 1 | `neck_tilt` | `head-eb05-j0_1-mc` | 1 |
| 2 | `eye_tilt` | `head-eb04-j2_4-mc` | 0 |
| 3 | `version` | `head-eb04-j2_4-mc` | 1 |
| 4 | `vergence` | `head-eb04-j2_4-mc` | 2 |

The wrapper attaches the `head-calibrator` at startup level 10 (runs after all devices are initialized).

#### `left_arm-mc_wrapper.xml`

| Parameter | Value | Description |
|-----------|-------|-------------|
| `name` | `/vizzy/left_arm` | Root YARP port name |
| `period` | `10` ms | Update period |
| `joints` | `3` | Total joints |

| Wrapper Joint | Axis Name | Source Device |
|--------------|-----------|---------------|
| 0 | `left_shldr_scapula` | `left_arm-eb06-j0_2-mc` |
| 1 | `left_shldr_flection` | `left_arm-eb06-j0_2-mc` |
| 2 | `left_shldr_abduction` | `left_arm-eb06-j0_2-mc` |

Attaches `left_arm-calibrator` at startup.

---

### Calibrators -- `calibrators/`

`parametricCalibratorEth` devices define the automatic joint calibration routine that runs at startup (unless `skipCalibration = true`). The calibration establishes the zero-reference (home) position for each joint.

#### How Calibration Works (Type 5)

All joints use **calibration type 5**: the motor is driven at a fixed PWM until it hits a mechanical hard stop (detected by stall or current/PWM saturation). Once the stop is detected, the current encoder count is mapped to the known physical angle of that stop (`calibrationDelta`), establishing the joint's absolute zero.

The process:
1. Motor drives toward the hard stop at the configured search PWM (`calibration1`)
2. Firmware detects stall/saturation
3. Encoder is zeroed at `stop_position + calibrationDelta`
4. Joint moves to the startup position at a safe velocity (and using the PID controller!)

#### `head-calib.xml` -- Head Calibration (5 joints)

| Parameter | Joint 0 (neck_pan) | Joint 1 (neck_tilt) | Joint 2 (eye_tilt) | Joint 3 (version) | Joint 4 (vergence) |
|-----------|-------------------|--------------------|--------------------|------------------|-------------------|
| `calibration1` (search PWM) | 1000 | 800 | 2000 | 2000 | 2000 |
| `calibrationDelta` (°) | -55.0 | -40.0 | -40.0 | -40.0 | -40.0 |
| `startupPosition` (°) | 0 | 0 | 0 | 0 | 0 |
| `startupVelocity` (°/s) | 10 | 10 | 20 | 10 | 10 |
| `startupMaxPwm` (PWM) | 1600 | 1400 | 1400 | 1400 | 1400 |
| `startupPosThreshold` (°) | 1.0 | 1.0 | 1.0 | 1.0 | 1.0 |

- **Home position**: All joints → 0°, reached at `homeVelocities`: 20 20 10 10 10 (deg/s)
- **Calibration order**: `(0)` → `(1)` → `(2)` → `(3 4)` -- joints 3 and 4 calibrate simultaneously
- `calibrationDelta` is negative for all joints, meaning the hard stop is in the negative direction

#### `left_arm-calib.xml` -- Left Arm Calibration (3 joints)

| Parameter | Joint 0 (scapula) | Joint 1 (flection) | Joint 2 (abduction) |
|-----------|------------------|-------------------|---------------------|
| `calibration1` (search PWM) | 1000 | 1000 | -- |
| `calibrationDelta` (°) | -20.0 | 84.0 | -75.0 |
| `startupPosition` (°) | 0 | 0 | 0 |
| `startupVelocity` (°/s) | 10 | 10 | 10 |
| `startupMaxPwm` (PWM) | 1400 | 1400 | 1400 |
| `startupPosThreshold` (°) | 1.0 | 1.0 | 1.0 |

- **Home position**: All joints → 0°, at `homeVelocities`: 20 20 20 (deg/s)
- **Calibration order**: `(0)` → `(1)` → `(2)` -- sequential

---

## Joints Reference

Complete joint reference table including all boards:

| YARP Part | Part Index | Axis Name | Min (°) | Max (°) | Motor | Encoder | Board |
|-----------|-----------|-----------|---------|---------|-------|---------|-------|
| `/vizzy/head` | 0 | `neck_pan` | -53 | +53 | DC | OPTICAL_QUAD | EB05 (MC4plus) |
| `/vizzy/head` | 1 | `neck_tilt` | -18 | +37 | DC | OPTICAL_QUAD | EB05 (MC4plus) |
| `/vizzy/head` | 2 | `eye_tilt` | -30 | +30 | DC | OPTICAL_QUAD | EB04 (MC4plus) |
| `/vizzy/head` | 3 | `version` | -30 | +30 | DC | OPTICAL_QUAD | EB04 (MC4plus) |
| `/vizzy/head` | 4 | `vergence` | 0 | +30 | DC | OPTICAL_QUAD | EB04 (MC4plus) |
| `/vizzy/left_arm` | 0 | `left_shldr_scapula` | -18 | +18 | DC (FOC) | OPTICAL_QUAD + AEA | EB06 (EMS4) |
| `/vizzy/left_arm` | 1 | `left_shldr_flection` | -75 | +135 | DC (FOC) | OPTICAL_QUAD + AEA | EB06 (EMS4) |
| `/vizzy/left_arm` | 2 | `left_shldr_abduction` | 0 | +70 | DC (FOC) | OPTICAL_QUAD + AEA | EB06 (EMS4) |

---

## YARP Concepts Explained

### `embObjMotionControl`

The core YARP device that communicates with IIT's Ethernet motion control boards (MC4plus, EMS4). It:
- Establishes UDP/Ethernet communication with the physical board
- Implements YARP's `IControlMode`, `IPositionControl`, `IVelocityControl`, `IEncoders`, and related interfaces
- Executes PID control loops (either locally or delegated to the board's embedded DSP)
- Manages joint limits, safety timeouts, and calibration hooks

Each instance controls one physical ETH board and its associated joints.

### `controlboardwrapper2`

A YARP multiplexer device that:
- Aggregates multiple `embObjMotionControl` devices into a single logical part
- Exposes standard YARP control board ports (`/state:o`, `/command:i`, `/rpc:i`)
- Handles joint index remapping between sub-devices and the unified part

Without this wrapper, each ETH board would need to be addressed separately by client applications.

### `parametricCalibratorEth`

Implements automatic calibration for Ethernet-based motion control boards. At startup, it:
1. Communicates with each joint's board to run the configured calibration procedure
2. Parks joints at `startupPosition` after calibration
3. Reports success/failure to `yarprobotinterface`

It is attached to a `controlboardwrapper2` rather than directly to the low-level device, so it operates through the same joint-index mapping as client applications.

### `xi:include` (XInclude)

XML inclusion mechanism used throughout these configs. Allows parameter files to be split into logical units (electronics, mechanicals, motor control) and reused across different configurations. The YARP DTD for `yarprobotinterface` fully supports XInclude, and all included files are merged at parse time before device instantiation.

---

## Eye Coupling Explained

The two eyes are mechanically linked through a differential drive: each motor moves both eyes simultaneously. Because of this, the firmware works in a **virtual joint space** (version + vergence) rather than directly commanding left and right eye motors.

**Coupling matrices** in `head-eb04-j2_4-mec.xml` define the relationship:

**Motor → Joint (M2J):**
```
version  = 0.5 × motor_R + 0.5 × motor_L     (average of both motors = gaze direction)
vergence = motor_L − motor_R                  (difference = convergence angle)
```

**Joint → Motor (J2M):**
```
motor_R = version − 0.5 × vergence
motor_L = version + 0.5 × vergence
```

This means:
- Commanding **version = +20°, vergence = 0°** → both eyes rotate 20° to the right together
- Commanding **version = 0°, vergence = +20°** → eyes converge by 20° (looking at a near object)

The `eye_tilt` joint (axis 0) is uncoupled and controls vertical gaze directly with a 1:1 motor relationship.

**Why vergence ≥ 0°:** Vergence represents convergence (eyes turning inward). A positive vergence means the eyes converge; negative vergence (divergence beyond parallel) is physiologically impossible and mechanically harmful, hence the lower limit of 0°.

---

## Board Types: MC4plus vs EMS4

| Feature | MC4plus (EB04, EB05) | EMS4 (EB06) |
|---------|---------------------|------------|
| Axes per board | Up to 4 | Up to 4 |
| Motor type supported | Brushed DC | Brushed DC with FOC |
| Actuator interface | Direct PWM | CAN bus (FOC drivers) |
| Primary encoder | Quadrature (on-board) | AEA (absolute, at joint) |
| Secondary encoder | None | ROIE (rotor incremental, via CAN) |
| Joint-level absolute position | No (requires calibration) | Yes (AEA encoder) |
| Torque control | No | Yes (via 2FOC) |
| Suitable for | Head joints (light, low-inertia) | Arm joints (heavy, precision needed) |

The MC4plus is a simpler board suited for the head's light DC motors where quadrature encoders and position control are sufficient. The EMS4 supports CAN-connected FOC driver boards, enabling torque control and absolute position feedback via AEA encoders -- essential for the arm's heavier joints.

---

## Documentation & Citation

For more details on the Vizzy platform, please refer to the following publication:

```bibtex
@inproceedings{moreno2016vizzy,
  title={Vizzy: A humanoid on wheels for assistive robotics},
  author={Moreno, Plinio and Nunes, Ricardo and Figueiredo, Rui and Ferreira, Ricardo
          and Bernardino, Alexandre and Santos-Victor, Jos{\'e} and Beira, Ricardo
          and Vargas, Lu{\'\i}s and Arag{\~a}o, Duarte and Arag{\~a}o, Miguel},
  booktitle={Robot 2015: Second Iberian Robotics Conference},
  pages={17--28},
  year={2016},
  organization={Springer}
}
```

For additional YARP documentation (sparse as it may be):
- [YARP documentation](https://www.yarp.it/latest/)
- [yarprobotinterface guide](https://www.yarp.it/latest/yarprobotinterface.html)
- [embObjMotionControl source (iCub)](https://github.com/robotology/icub-main)

---

## Reporting Issues

Please report bugs, configuration errors, or missing documentation using the [GitHub Issues tab](../../issues) of this repository.

When reporting an issue related to a specific joint or board, please include:
- The joint name and YARP part (e.g., `neck_tilt` on `/vizzy/head`)
- The board IP and type (e.g., `10.1.3.5`, EB05/MC4plus)
- Relevant `yarprobotinterface` log output
- Steps to reproduce
