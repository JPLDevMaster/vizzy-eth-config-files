# Configuration Reference

Detailed parameter documentation for every configuration file in this repository.

## Table of Contents

- [Configuration Architecture](#configuration-architecture)
- [Top-Level Files](#top-level-files)
- [General Parameters (`general.xml`)](#general-parameters-generalxml)
- [Electronics (ETH Board) -- `hardware/electronics/`](#electronics-eth-board----hardwareelectronics)
- [Mechanicals (Joint Physics) -- `hardware/mechanicals/`](#mechanicals-joint-physics----hardwaremechanicals)
- [Motor Control -- `hardware/motorControl/`](#motor-control----hardwaremotorcontrol)
- [Control Board Wrappers -- `wrappers/motorControl/`](#control-board-wrappers----wrappersmotorcontrol)
- [Calibrators -- `calibrators/`](#calibrators----calibrators)

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

## Top-Level Files

### `vizzy_all.xml` -- Robot Configuration Root

The root configuration file. Defines the robot identity and includes all device configurations via XInclude.

| Parameter | Value | Description |
|-----------|-------|-------------|
| `name` | `vizzy` | Robot name, used to identify the robot in YARP |
| `portprefix` | `vizzy` | Prefix for all YARP ports (e.g., `/vizzy/head`) |
| `build` | `1` | Configuration build version |

Currently active subsystems: HEAD (EB04 + EB05), LEFT ARM (EB06), and their respective calibrators. Cartesian control modules are present but commented out.

---

### `yarprobotinterface.ini` -- Module Entry Point

Points `yarprobotinterface` to the main XML:

```ini
config ./vizzy_all.xml
```

### `yarpmotorgui.ini` -- Motor GUI Configuration

| Parameter | Value | Description |
|-----------|-------|-------------|
| `robot` | `vizzy` | Robot name to connect to |
| `parts` | `(head left_arm)` | Parts displayed in the GUI |

---

### `vizzy_left_arm.xml` -- Left-Arm-Only Entry Point

An alternative root configuration that includes only the left arm subsystem (EB06 + `left_arm-mc_wrapper` + `left_arm-calibrator`). Useful during development or debugging when the head boards are not needed or not available. Launch with:

```bash
yarprobotinterface --config vizzy_left_arm.xml
```

---

## General Parameters (`general.xml`)

Included by every `embObjMotionControl` device. Sets system-wide motion control behaviour.

| Parameter | Value | Description |
|-----------|-------|-------------|
| `skipCalibration` | `false` | If `true`, skips joint calibration on startup. **Use with extreme caution** -- only set to `true` when you are certain the robot is already at a known safe position. |
| `useRawEncoderData` | `false` | If `true`, bypasses encoder scaling and coupling matrices, using raw encoder counts directly. Set to `false` for normal operation with physical units (degrees). |
| `useLimitedPWM` | `false` | If `true`, applies an additional software cap on PWM output, reducing motor effort. Useful for debugging or safe operation with fragile hardware. |
| `verbose` | `true` | Enables verbose debug logging from the motion control library. Produces detailed output useful during tuning or fault diagnosis. |

---

## Electronics (ETH Board) -- `hardware/electronics/`

These files configure the Ethernet communication parameters for each board. All boards share the same port (12345) on the `10.1.3.x` subnet.

### `vizzy-desktop.xml` -- Host PC Parameters

Defines the control PC's identity on the network.

| Parameter | Value | Description |
|-----------|-------|-------------|
| `PC104IpAddress` | `10.1.3.104` | IP address of the host control PC |
| `PC104IpPort` | `12345` | UDP port used for communication with ETH boards |
| `PC104TXrate` | `1` | Packet transmit rate multiplier for the host |
| `PC104RXrate` | `5` | Packet receive rate multiplier for the host |

### `head-eb04-j2_4-eln.xml` / `head-eb05-j0_1-eln.xml` / `left_arm-eb06-j0_2-eln.xml`

| Parameter | EB04 (Eyes) | EB05 (Neck) | EB06 (Arm) | Description |
|-----------|------------|------------|------------|-------------|
| `IpAddress` | `10.1.3.4` | `10.1.3.5` | `10.1.3.6` | Board IP address on the robot's internal Ethernet network |
| `IpPort` | `12345` | `12345` | `12345` | UDP port for host↔board communication |
| `type` | `mc4plus` | `mc4plus` | `ems4` | ETH board hardware type (see [Board Types](board-types.md)) |
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

## Mechanicals (Joint Physics) -- `hardware/mechanicals/`

These files define the physical and kinematic properties of each joint: axis names, encoder resolutions, gear ratios, joint limits, and coupling between motors and joints.

### Common Parameters (all boards)

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

### Joint Limits

Two sets of limits are defined per joint:

| Parameter | Description |
|-----------|-------------|
| `jntPosMin` / `jntPosMax` | **Software limits** (degrees): the operating range enforced by the firmware. The controller will not command the joint beyond these values. Defined in `*-mc.xml`. |
| `hardwareJntPosMin` / `hardwareJntPosMax` | **Hardware limits** (degrees): the physical range defined by mechanical hard stops. These are wider than software limits and represent the absolute physical boundary. Defined in `*-mec.xml`. |
| `rotorPosMin` / `rotorPosMax` | Rotor position limits (used with FOC/absolute encoders). `0 0` means unconstrained at the rotor level. |

### `head-eb04-j2_4-mec.xml` -- Eye Joints

This board controls 3 coupled eye joints. See [Eye Coupling Explained](yarp-concepts.md#eye-coupling-explained) for details on the coupling matrices.

| Joint | Axis Name | Soft Min (°) | Soft Max (°) | Hard Min (°) | Hard Max (°) |
|-------|-----------|-------------|-------------|-------------|-------------|
| 0 | `eye_tilt` | -30 | +30 | -40 | +40 |
| 1 | `version` | -30 | +30 | -40 | +40 |
| 2 | `vergence` | 0 | +30 | -40 | +40 |

> **version** = combined horizontal gaze direction (both eyes point the same way). **vergence** = eye convergence angle (how much the eyes converge toward a near object). See [Eye Coupling Explained](yarp-concepts.md#eye-coupling-explained).

### `head-eb05-j0_1-mec.xml` -- Neck Joints

| Joint | Axis Name | Soft Min (°) | Soft Max (°) | Hard Min (°) | Hard Max (°) |
|-------|-----------|-------------|-------------|-------------|-------------|
| 0 | `neck_pan` | -53 | +53 | -55 | +55 |
| 1 | `neck_tilt` | -18 | +37 | -40 | +40 |

No coupling (identity J2M/M2J matrices).

### `left_arm-eb06-j0_2-mec.xml` -- Left Shoulder Joints

| Joint | Axis Name | Soft Min (°) | Soft Max (°) | Hard Min (°) | Hard Max (°) |
|-------|-----------|-------------|-------------|-------------|-------------|
| 0 | `left_shldr_scapula` | -18 | +18 | -95.5 | +20 |
| 1 | `left_shldr_flection` | -75 | +135 | -80 | +160 |
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

## Motor Control -- `hardware/motorControl/`

Each `*-mc.xml` file is the main `embObjMotionControl` device definition. It assembles the general, electronics, mechanicals, and service configurations, then adds operating limits, control mode assignments, and PID gains.

### `LIMITS` Group

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

### `TIMEOUTS` Group

| Parameter | Unit | Description |
|-----------|------|-------------|
| `velocity` | ms | Time after which a velocity command is considered stale and the joint is stopped. Prevents runaway if the commanding process crashes or stops sending commands. |

### `IMPEDANCE` Group

| Parameter | Unit | Description |
|-----------|------|-------------|
| `stiffness` | N·m/rad | Spring stiffness for impedance control mode. `0` = pure position mode (no spring). |
| `damping` | N·m·s/rad | Viscous damping for impedance control mode. `0` = no damping. |

All joints currently have `stiffness = 0` and `damping = 0`, meaning impedance control is disabled.

### `CONTROLS` Group

Assigns a named PID group to each control mode per joint. Allows different joints or modes to use different tunings.

| Parameter | Description |
|-----------|-------------|
| `positionControl` | PID group used when the joint is in position control mode |
| `velocityControl` | PID group used when the joint is in velocity control mode |
| `mixedControl` | PID group used in mixed position/velocity mode |
| `torqueControl` | PID group used in torque control mode (left arm only) |
| `currentPid` | Low-level current controller (left arm only, runs inside the FOC loop) |
| `speedPid` | Low-level speed controller (left arm only, runs inside the FOC loop) |

### PID Groups

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

### Additional PID Groups (Left Arm Only)

**`TRQ_PID_DEFAULT`** -- Torque control (currently unused, imported from iCub Lisboa v2.14.0):

| Parameter | Description |
|-----------|-------------|
| `ko` | Output offset (constant additive term to PID output) |
| `viscousPos` / `viscousNeg` | Viscous friction compensation coefficients (positive/negative motion direction) |
| `coulombPos` / `coulombNeg` | Coulomb (static) friction compensation |
| `velocityThres` | Velocity threshold (deg/s) below which Coulomb friction compensation is applied |
| `filterType` | Type of filter applied to the torque feedback signal |
| `ktau` | Torque constant: converts motor current to joint torque (N·m/A). Values: scapula=180, flection=464, abduction=463. |
| `kbemf` | Back-EMF compensation coefficient. Currently `0` for all joints (disabled). Added to satisfy a required-parameter check in the firmware; omitting it produces a runtime error even when torque control is not active. |

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

### `*-mc_service.xml` -- Actuator & Encoder Port Mapping

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

| Slot | Actuator | ENCODER1 | ENCODER2 (qenc) Port | ENCODER2 Resolution |
|------|----------|----------|----------------------|---------------------|
| 0 — left_shldr_scapula | `foc` @ `CAN1:1:0` | `none` | `qenc` @ `CAN1:1:0` (atmotor) | 512 CPR |
| 1 — left_shldr_flection | `foc` @ `CAN1:2:0` | `none` | `qenc` @ `CAN1:2:0` (atmotor) | 512 CPR |
| 2 — left_shldr_abduction | `foc` @ `CAN1:3:0` | `none` | `qenc` @ `CAN1:3:0` (atmotor) | 512 CPR |
| 3 — (reserved) | `foc` @ `CAN1:4:0` | `none` | `qenc` @ `CAN1:4:0` (atmotor) | 512 CPR |

- **`foc`**: Field Oriented Control driver board, addressed over CAN bus. `CAN1:X:0` = CAN bus 1, node ID X, channel 0.
- **ENCODER1 `none`**: No joint-level absolute encoder is currently used.
- **`qenc` @ `atmotor`**: Quadrature encoder mounted at the motor rotor, read through the FOC board over CAN. This mirrors the setup used on the MC4plus head boards.
- **Slot 3 (reserved)**: A 4th actuator/encoder slot is pre-allocated in the service file (`CAN1:4:0`) to account for the Torso motors. The mec.xml currently defines only 3 active joints.
- **No dual-encoder cross-check**: With ENCODER1 absent, the encoder cross-check tolerance is `0` for all slots and no fault can be triggered by encoder disagreement.

**FOC CAN board firmware version:**

| Parameter | Value |
|-----------|-------|
| Protocol major/minor | 1.6 |
| Firmware major/minor/build | 3.3.10 |

> The EMS4 checks that connected FOC boards report matching protocol and firmware versions at startup. Mismatched firmware will prevent the arm from initializing.

---

## Control Board Wrappers -- `wrappers/motorControl/`

`controlboardwrapper2` devices aggregate multiple low-level `embObjMotionControl` devices into a single unified YARP interface port. This is the layer that clients (e.g., `yarpmotorgui`, cartesian controllers, user applications) connect to.

### `head-mc_wrapper.xml`

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

### `left_arm-mc_wrapper.xml`

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

## Calibrators -- `calibrators/`

`parametricCalibratorEth` devices define the automatic joint calibration routine that runs at startup (unless `skipCalibration = true`). The calibration establishes the zero-reference (home) position for each joint.

### How Calibration Works (Type 5)

All joints use **calibration type 5**: the motor is driven at a fixed PWM until it hits a mechanical hard stop (detected by stall or current/PWM saturation). Once the stop is detected, the current encoder count is mapped to the known physical angle of that stop (`calibrationDelta`), establishing the joint's absolute zero.

The process:
1. Motor drives toward the hard stop at the configured search PWM (`calibration1`)
2. Firmware detects stall/saturation
3. Encoder is zeroed at `stop_position + calibrationDelta`
4. Joint moves to the startup position at a safe velocity (and using the PID controller!)

### `head-calib.xml` -- Head Calibration (5 joints)

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

### `left_arm-calib.xml` -- Left Arm Calibration (3 joints)

| Parameter | Joint 0 (scapula) | Joint 1 (flection) | Joint 2 (abduction) |
|-----------|------------------|-------------------|---------------------|
| `calibration1` (search PWM) | 1000 | 1000 | 1000 |
| `calibrationDelta` (°) | -20.0 | 84.0 | -75.0 |
| `startupPosition` (°) | 0 | 0 | 0 |
| `startupVelocity` (°/s) | 10 | 10 | 10 |
| `startupMaxPwm` (PWM) | 1400 | 1400 | 1400 |
| `startupPosThreshold` (°) | 1.0 | 1.0 | 1.0 |

- **Home position**: All joints → 0°, at `homeVelocities`: 20 20 20 (deg/s)
- **Calibration order**: `(0)` → `(1)` → `(2)` -- sequential
- All 3 joints use hard-stop (type 5) calibration.
