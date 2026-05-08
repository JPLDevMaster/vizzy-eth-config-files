# Joints Reference

Complete joint reference table for all boards. Software position limits are the operating range enforced by the firmware; see [Configuration Reference](configuration-reference.md) for hardware limits and calibration parameters.

| YARP Part | Part Index | Axis Name | Min (°) | Max (°) | Motor | Encoder | Board |
|-----------|-----------|-----------|---------|---------|-------|---------|-------|
| `/vizzy/head` | 0 | `neck_pan` | -53 | +53 | DC | OPTICAL_QUAD | EB05 (MC4plus) |
| `/vizzy/head` | 1 | `neck_tilt` | -18 | +37 | DC | OPTICAL_QUAD | EB05 (MC4plus) |
| `/vizzy/head` | 2 | `eye_tilt` | -30 | +30 | DC | OPTICAL_QUAD | EB04 (MC4plus) |
| `/vizzy/head` | 3 | `version` | -30 | +30 | DC | OPTICAL_QUAD | EB04 (MC4plus) |
| `/vizzy/head` | 4 | `vergence` | 0 | +30 | DC | OPTICAL_QUAD | EB04 (MC4plus) |
| `/vizzy/left_arm` | 0 | `left_shldr_scapula` | -18 | +18 | DC (FOC) | OPTICAL_QUAD (via CAN) | EB06 (EMS4) |
| `/vizzy/left_arm` | 1 | `left_shldr_flection` | -75 | +135 | DC (FOC) | OPTICAL_QUAD (via CAN) | EB06 (EMS4) |
| `/vizzy/left_arm` | 2 | `left_shldr_abduction` | 0 | +70 | DC (FOC) | OPTICAL_QUAD (via CAN) | EB06 (EMS4) |
