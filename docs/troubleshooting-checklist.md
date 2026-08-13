# Setup and troubleshooting checklist

Ordered so each step's assumptions are verified by the steps above it. Do not skip
ahead - a bad calibration makes every downstream measurement meaningless.

## 1. Robot kinematic calibration

Run `programs/Chk01_RobotCalibration.pcs`.

- Prints `1` - continue.
- Prints `0` - the server is using the nominal DENSO model, typically several mm off.
  Run `Ex14_ROBOT_CALIBRATION_IK.pcs` (does not move the robot), then `Ex15_TEST_CALIBRATION.pcs`
  to check the FK error, then re-run Chk01.

This is separate from the camera hand-eye calibration. Programs that use only
`CamGetPrediction` + `Move L` never touch server IK, so they can look healthy while this
is missing.

## 2. Camera rig pose

Run `programs/Chk02_RigPoseIterate.pcs`, adjusting `Rz` between runs.

**The test:** the GREEN axis of the triad in the Cambrian UI must point the same way as
"up" on the real cameras.

Tolerance is ~30 mm translation and ~10 degrees rotation. Try `Rz` = 0, 90, 180, 270.
Measure `Tx/Ty/Tz` from the flange centre to the rig centre rather than assuming the
standard-mount defaults.

A rig pose flipped 180 degrees about Z **will not calibrate** - the solver cannot recover
from an axial flip in the initial estimate.

## 3. Hand-eye calibration

Run `programs/Cal01_AutoCalibration.pcs`.

Before starting:

- [ ] Rig pose verified in step 2
- [ ] **Gripper removed** - it clips the FoV edge and occludes the board, and the images
      still look fine
- [ ] Mount type FLANGE (the program sets it explicitly)
- [ ] `CamNewCalibration` called (the program does this) so the previous principal point
      is not used as a reference
- [ ] Board tangential to the robot base, unobstructed, visible in both cameras
- [ ] Cable slack for full J6 rotation

Acceptance:

| Metric | Minimum | Good |
|---|---|---|
| Number of images | 8 | 15+ |
| Position STD | 10 mm | < 1 mm |
| Average pixel error | 10 px | < 5 px |

Then inspect `~/Desktop/Caint/Calibration/calib_X/results/unity/final_overlap.png` -
red dots good, long red lines bad.

**Do not continue if these fail.**

## 4. TCP

Run `programs/Chk03_ToolCompare.pcs`.

- [ ] `ToolPos( CurTool )` matches the real gripper, rotation included
- [ ] `CurTool` is the tool you intend - defining a tool does not select it, use `ChangeTool`
- [ ] `ToolPos( 0 )` is all zeros
- [ ] Server tool (derived from `srv TCP` minus `srv flange`) matches the controller tool

Notes:

- The 4-point touch-up method solves **position only**. It leaves RX/RY/RZ at zero. If
  your gripper's working axis is rotated relative to the flange, 4-point will never tell
  you.
- Compare all six values, not just RZ. Euler components do not add independently.
- Run with the tool 30-45 degrees off vertical; near straight-down the decomposition
  trades between RX/RY/RZ and is hard to read.
- For a rotationally symmetric vacuum cup there is no physically correct RZ. Pick one and
  make both sides use it - matching is the whole requirement.
- Changing the TCP invalidates TCP-mode waypoints. **Re-teach Home and Drop.** The `.rig`
  and kinematic calibrations are flange-based and unaffected.

## 5. Pick path consistency

Run `programs/Chk04_PickPathConsistency.pcs`.

`CamGraspPose` and `J2P( CamPathJoints( last ) )` should describe the same pose. If they
differ, see `findings.md` - that discrepancy is the J6 wrist wind, and the top suspect is
grasp point Symmetry generating rotation variants.

## Quick symptom index

| Symptom | Start at |
|---|---|
| Camera model in UI points the wrong way | Step 2 |
| Calibration diverges or scores badly | Step 2, then step 3 |
| Picks land offset by a constant amount | Step 4 |
| Wrist rotates on the way into the grasp | Step 5 |
| `Ex13` works but `Ex26` doesn't | Steps 1, 4, 5 - Ex26 uses server IK, Ex13 does not |
| Predictions in the wrong position | Rig active, rig moved, calibration stale, mount type, model depth range, TCP |

## Things that mislead

- **`Ex13` working proves nothing about frames.** It uses no server IK, so a pose-frame
  error is invisible there - with a symmetric cup the pick succeeds anyway.
- **`J2P( CurJnt )` is not a flange pose.** It goes through the active tool and work, so
  it equals `CurPos`. Use `TOOL0` or `CamGetFlangeFK()` for a genuine flange pose.
- **`CurPos` is in the active work coordinate system.** Select work 0 before comparing
  against anything Cambrian returns.
- **RZ-only comparisons are unreliable.** A 180 degree rotation about Z can land in RX or
  RY depending on the flange orientation.
- **A constant offset in the tool frame across different poses** points at a frame or
  tool definition, not at the planner.
