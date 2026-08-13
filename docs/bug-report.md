# Support report draft - CamGetPickPath pose/joints inconsistency

**Send only if the Symmetry = None test in `findings.md` does not resolve it.** If
setting grasp point Symmetry to None (with a model reload) makes the discrepancy vanish,
the cause is variant generation and this report should be rewritten around that instead.

---

## Summary

On a DENSO VS068A4A running Cambrian VISION 2.5.1, a single `CamGetPickPath` call
returns a grasp pose and a set of grasp joints that do not describe the same wrist pose.
They differ by exactly 180 degrees about the tool Z axis, plus a lateral offset equal to
twice the TCP's lateral offset.

Because `CamExecutePath` drives the final leg to the *pose* and `CamExecutePathReverse`
drives its first step to the *joints*, the robot winds J6 180 degrees on the way into the
grasp and unwinds it on the retract.

## Environment

| | |
|---|---|
| Robot | DENSO VS068A4A, RC8 |
| Cambrian VISION | 2.5.1 |
| DENSO API module | `Cambrian_API.pcs`, doc v1.6 |
| Camera rig | Flange mounted |
| AI model | `12x130_Kii_BLN_10Aug2026_161309_10-08-2026`, Full Pose |
| Failing program | `EX26_BinPick_Auto` |
| Working program | `Ex13_approach_parts` (uses no server IK) |

## Reproduction

Jog so the camera sees a pickable part, then run `Chk04_PickPathConsistency.pcs` (in this
repository). It calls `CamGetPickPath` once and prints `CamGraspPose`,
`J2P( CamPathJoints( last ) )`, `CamGraspJoints` and `CamPathJoints( last )` without
moving the robot.

## Observation

Two independent grasps, different parts, different poses, different arm configurations.

**Sample 1**

```
CamGraspPose               P(550.9437, 15.70601, 37.2007,  -177.0419,  0.30552,   -46.09262, -1)
J2P(CamPathJoints(last))   P(552.4468, 17.6273,  37.30177,  177.0419, -0.3054792, 133.9078,   5)
CamGraspJoints             J(0.588876, 58.2341, 84.1607, 3.85735, 35.723, -136.406, 0, 0)
CamPathJoints(last)        J(0.588876, 58.2341, 84.1607, 3.85735, 35.723, -136.406, 0, 0)
```

**Sample 2**

```
CamGraspPose               P(561.9462, 123.3042, 34.5586,  -173.7505, -4.91649,  23.91594, -1)
J2P(CamPathJoints(last))   P(563.1489, 121.166,  34.28621,  173.7505,  4.916492, -156.084,  5)
CamGraspJoints             J(8.70736, 60.8717, 79.4239, 12.1562, 37.2275, -25.0113, 0, 0)
CamPathJoints(last)        J(8.70736, 60.8717, 79.4239, 12.1562, 37.2275, -25.0113, 0, 0)
```

`CamGraspJoints` and `CamPathJoints(last)` are identical in both samples. The
inconsistency is entirely between the joints and the returned pose.

### Rotation

Converted to rotation matrices, the two orientations share an identical Z axis with X and
Y both negated - a pure 180 degree rotation about the tool Z axis. Sample 1:

| Axis | `CamGraspPose` | `J2P` |
|---|---|---|
| Z | (0.0335, 0.0396, -0.9987) | (0.0335, 0.0396, -0.9987) |
| X | (0.6933, -0.7206, -0.0053) | (-0.6933, 0.7206, 0.0053) |
| Y | (-0.7199, -0.6922, -0.0516) | (0.7199, 0.6922, 0.0516) |

Sample 2 shows the same signature.

### Position

Projected onto each grasp's own tool axes:

| | Magnitude | Tool X | Tool Y | Tool Z |
|---|---|---|---|---|
| Sample 1 | 2.442 mm | ~0.21 | ~2.46 | ~0 |
| Sample 2 | 2.468 mm | ~0.21 | ~2.46 | ~0.02 |
| `2 x (tool X, tool Y)` | 2.493 mm | 0.289 | 2.476 | 0 |

Constant in the tool frame across both samples and matching twice the TCP's lateral
offset - consistent with a 180 degree wrist rotation swinging an off-axis TCP to the
other side.

## Tool offsets verified identical

This is not an integrator-side TCP mismatch.

```
controller  ToolPos(1)                      P(-0.144509, -1.23811, 199.9489, 0, 0, 0, -1)
server      CamGetFlangeFK(J)               P(476.3116, 1.20625, 500.0031, -179.9999, -6e-005, 0.00145, -1)
server      CamGetFK(J, 1)                  P(476.1673, 2.44465, 300.0542, -179.9999, -6e-005, 0.00145, -1)
```

Server flange and server TCP have identical orientations, so the server's tool rotation
is zero. Projecting the position delta onto the flange axes (flange at RX ~ 180, so
X=(1,0,0), Y=(0,-1,0), Z=(0,0,-1)) recovers the server's tool offset as
`(-0.1443, -1.2384, 199.9489, 0, 0, 0)` - matching `ToolPos(1)` to four decimal places.

## Ruled out on our side

| Candidate | Evidence |
|---|---|
| Robot kinematic calibration | `CamCheckRobotCalibration` returns 1; server flange agrees with controller |
| TCP mismatch | Derived server tool matches `ToolPos(1)` to 4 dp |
| Tool not active | `CurTool` = 1, pendant shows `W0 T1` |
| `TOOL0` redefined | Reads all zeros |
| Ignore Tilt / Ignore Gamma | Both confirmed never enabled |
| Joint twinning budget | Raising `approachTwinJointDeg` had no effect |
| Arm configuration constraint | `CamSetRobotConfigurations` increased the rotation; backed out |
| Hand-eye calibration | Acceptance numbers pass |
| Grasp point Symmetry | *(state the result of the Symmetry = None test here before sending)* |

## Impact

`CamExecutePath` drives `Move L, CamGraspPose` for the final grasp leg;
`CamExecutePathReverse` drives `Move L, J2P( CamPathJoints( N ) )` for its first step.
Because the two disagree, every pick winds the wrist 180 degrees into the grasp and
unwinds it on the retract. The figure guard in `CamExecutePathEx` cannot catch this -
DENSO's `Fig` encodes only the J1, J3 and J5 branches, so two solutions 180 degrees apart
in J6 carry identical figures.

`Ex13_approach_parts` is unaffected because it uses no server IK, so there are no joint
solutions to contradict the pose.

## Questions

1. Should `CamGraspPose` from `CamGetPickPath` be FK-consistent with `CamGraspJoints`, as
   `CamGetIKEx` explicitly is? The DENSO API doc says `CamGetIKEx` sets `CamGraspPose`
   from `CamGetFK` of the solved joints specifically to avoid an unspecified figure;
   `CamGetPickPath` appears to set both "directly" instead.
2. If a rotation variant is selected during planning, is the returned `grasp_pose`
   expected to carry the variant's orientation or the base grasp's?
3. Is there a supported way to have `CamExecutePath` drive the final leg to
   `J2P( CamPathJoints( N ) )` rather than `CamGraspPose`, without modifying
   `Cambrian_API.pcs`?

## Attachments to include

- Both `PrintDbg` sets above, verbatim from the pendant log
- `Chk03_ToolCompare.pcs` output
- Grasp point Symmetry configuration (type, Rotation Center, Alignment, Min/Max Rotation)
- Cambrian VISION backup with PredictionLog enabled
