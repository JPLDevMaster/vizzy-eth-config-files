# Vizzy Humanoid Robot - ETH Board Configuration Files

[![YARP](https://img.shields.io/badge/YARP-3.x-blue)](https://www.yarp.it/)
[![Platform](https://img.shields.io/badge/Platform-Linux-lightgrey)](https://ubuntu.com/)
[![Robot](https://img.shields.io/badge/Robot-Vizzy-informational)](https://vislab.isr.tecnico.ulisboa.pt/robots/vizzy/)
[![Status](https://img.shields.io/badge/Status-Active%20Development-yellow)]()

**🚧 Under Active Development 🚧**

This repository contains the YARP `yarprobotinterface` configuration files for the Vizzy humanoid robot's Ethernet (ETH)-based motion control boards. It covers the **head** (eyes and neck) and **left arm** (shoulder) subsystems so far, providing hardware-level configuration for motor control, encoder mapping, joint coupling, calibration, and YARP port wiring.

> **Note on YARP documentation:** YARP's own documentation for `embObjMotionControl`, calibrators, and related devices is notoriously sparse. The docs in this repository attempt to bridge that gap by providing detailed explanations of every configuration parameter and its effect on robot behaviour.

---

## Table of Contents

- [Hardware Overview](#hardware-overview)
- [Repository Structure](#repository-structure)
- [Launching the Robot Interface](#launching-the-robot-interface)
- [Further Reading](#further-reading)
- [Resources](#resources)
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
├── vizzy_all.xml                             # Top-level robot configuration (entry point for full robot)
├── vizzy_left_arm.xml                        # Left-arm-only entry point (EB06 only, no head)
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
│       └── left_arm-eb06-j0_2-mc_service.xml # Arm: CAN/qenc port mapping
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
├── models/                                   # Mechanical CAD models (reference only)
│   ├── Lock_Piece_Model.dwg
│   ├── Lock_Piece_Model.stl
│   ├── Vizzy_Head_Board_Holder_Model.dwg
│   ├── Vizzy_Head_Boards_Holder_Model.stl
│   └── Drawing2.dwg
│
└── resources/                                # Reference materials (not loaded by yarprobotinterface)
    ├── docs/                                 # Detailed technical documentation
    │   ├── configuration-reference.md        # Full parameter reference for all config files
    │   ├── joints-reference.md               # Quick-lookup joint table
    │   ├── yarp-concepts.md                  # YARP device concepts and eye coupling
    │   └── board-types.md                    # MC4plus, EMS4, and 2FOC hardware overview
    │
    ├── firmware/                             # ETH board firmware hex files
    │   ├── ems_v1_23.hex                     # EMS4 board firmware (v1.23)
    │   └── mc4plus_v1_23.hex                 # MC4plus board firmware (v1.23)
    │
    ├── images/                               # Board schema diagrams (used in docs/board-types.md)
    │   ├── MC4_Plus_Board_Schema.png
    │   ├── EMS_Board_Schema.png
    │   └── 2FOC_Board_Schema.png
    │
    └── schematics/                           # Electrical wiring schematics
        ├── Left_Shoulder.dwg                 # Original left shoulder schematic (AutoCAD)
        ├── Left_Shoulder_v2.dwg              # Current left shoulder schematic (AutoCAD)
        ├── Left_Shoulder_v2-Model.pdf        # Exported PDF of current schematic
        ├── Current_Setup.pdf                 # Current overall wiring setup
        ├── Old_Setup.pdf                     # Previous wiring setup (reference)
        └── Old_Setup_2.pdf                   # Earlier wiring setup (reference)
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

## Further Reading

Detailed technical documentation is in [`resources/docs/`](resources/docs/):

| Document | Contents |
|----------|----------|
| [Configuration Reference](resources/docs/configuration-reference.md) | XInclude architecture, per-file parameter tables, PID groups, calibration parameters |
| [Joints Reference](resources/docs/joints-reference.md) | Quick-reference table for all 8 joints across both YARP parts |
| [YARP Concepts](resources/docs/yarp-concepts.md) | `embObjMotionControl`, `controlboardwrapper2`, `parametricCalibratorEth`, eye coupling matrices |
| [Board Types](resources/docs/board-types.md) | MC4plus, EMS4, and 2FOC hardware overview with connector diagrams |

---

## Resources

The `resources/` directory contains reference materials that are not loaded by `yarprobotinterface` but are useful for hardware maintenance, documentation, and development.

### `resources/docs/`

Detailed technical documentation for this repository. See [Further Reading](#further-reading) above.

### `resources/firmware/`

Pre-built firmware hex files for flashing the ETH boards. These must be flashed using the IIT firmware update tools (e.g., `FirmwareUpdater`) and are not applied automatically at runtime.

| File | Board | Version |
|------|-------|---------|
| `mc4plus_v1_23.hex` | MC4plus (EB04, EB05) | v1.23 |
| `ems_v1_23.hex` | EMS4 (EB06) | v1.23 |

> The firmware version must be compatible with the `MotioncontrolVersion` declared in the `*-mec.xml` files (currently `6`) and the protocol/firmware versions declared in `*-mc_service.xml`. Flashing mismatched firmware will prevent the board from initialising.

### `resources/images/`

Board schema diagrams embedded in [`resources/docs/board-types.md`](resources/docs/board-types.md). Not required for robot operation.

| File | Content |
|------|---------|
| `MC4_Plus_Board_Schema.png` | MC4plus connector and signal layout |
| `EMS_Board_Schema.png` | EMS4 connector and signal layout |
| `2FOC_Board_Schema.png` | 2FOC CAN board connector and signal layout |

### `resources/schematics/`

AutoCAD electrical wiring schematics for the robot's physical cabling. These document how the ETH boards, motors, and encoders are physically wired.

| File | Content |
|------|---------|
| `Left_Shoulder_v2.dwg` | Current left shoulder schematic (AutoCAD source, actively maintained) |
| `Left_Shoulder_v2-Model.pdf` | Exported PDF of the current left shoulder schematic |
| `Left_Shoulder.dwg` | Original left shoulder schematic (superseded, kept for reference) |
| `Current_Setup.pdf` | Current overall robot wiring setup |
| `Old_Setup.pdf` | Previous wiring setup (reference) |
| `Old_Setup_2.pdf` | Earlier wiring setup (reference) |

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
- [iCub (mesh) documentation](https://mesh-iit.github.io/documentation/)

---

## Reporting Issues

Please report bugs, configuration errors, or missing documentation using the [GitHub Issues tab](../../issues) of this repository.

When reporting an issue related to a specific joint or board, please include:
- The joint name and YARP part (e.g., `neck_tilt` on `/vizzy/head`)
- The board IP and type (e.g., `10.1.3.5`, EB05/MC4plus)
- Relevant `yarprobotinterface` log output
- Steps to reproduce
