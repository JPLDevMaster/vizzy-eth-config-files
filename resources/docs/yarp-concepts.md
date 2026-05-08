# YARP Concepts

Conceptual documentation for the YARP device types used in this repository, and an explanation of the eye coupling mechanism.

## Table of Contents

- [YARP Concepts Explained](#yarp-concepts-explained)
  - [`embObjMotionControl`](#embobjmotioncontrol)
  - [`controlboardwrapper2`](#controlboardwrapper2)
  - [`parametricCalibratorEth`](#parametriccalibratoreth)
  - [`xi:include` (XInclude)](#xiinclude-xinclude)
- [Eye Coupling Explained](#eye-coupling-explained)

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
