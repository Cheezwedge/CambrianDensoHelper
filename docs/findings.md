# J6 wrist rotation - investigation record

**Status: open.** Prime suspect is grasp point Symmetry set to **Cylindrical** with a
rotation axis parallel to the gripper approach axis. Next step is the Symmetry = None
test described at the bottom.

## Symptom

Running `EX26_BinPick_Auto`, the robot arrives at the part, rotates J6 slowly 180
degrees, then rotates it slowly back. Net zero. The part is still picked.

`Ex13_approach_parts` picks the same part correctly with no wrist rotation.

## Where the motion happens

Localised precisely to the two linear legs, both driven at `MoveLSpeed` (20%), which is
why they look slow:

| Step | Call | Target |
|---|---|---|
| Wind | `CamExecutePathEx` final leg | `Move L, CamGraspPose` |
| Unwind | `CamExecutePathReverse` first step | `Move L, J2P( CamPathJoints( N ) )` |

The two legs express the *same waypoint* differently. The reverse leg is internally
consistent - `J2P` computes a pose through the active tool and `Move L` executes through
the same tool - so it lands exactly on the joint solution it was derived from. The
forward leg targets a pose that came from the server.

**The asymmetry is itself the evidence.** If those two expressions described the same
pose, neither leg would move the wrist at all.

## Why Ex13 is unaffected

Ex13 never uses Cambrian's IK. It goes to a taught joint pose (`J1`) then chains
`Move L` moves; a linear move inherits its arm branch from wherever the robot already
is. There are no server-supplied joint solutions to contradict the poses.

Ex26 is the first program that mixes the two: `Move P, CamPathJoints(N)` puts the wrist
where the server says, then `Move L, CamGraspPose` pulls it elsewhere.

A consequence worth noting: **Ex13 would be equally affected by any pose-frame error and
would not show it.** With a rotationally symmetric vacuum cup the pick succeeds either
way. Ex13 working is not evidence that the frames are correct.

## Why the figure guard does not catch it

```pacscript
WantFig = Fig( J2P( CamPathJoints( N ) ) )
If CurFig <> WantFig Then ... Exit Sub
```

On a 6-axis DENSO, `Fig` encodes three branch bits - J1 front/back, J3 elbow up/down,
J5 wrist flip/non-flip. **J6 contributes nothing.** Two solutions 180 degrees apart in
J6 carry identical figures, so the guard passes and the `Move L` is free to place J6
wherever interpolation demands.

The guard also validates a joint-to-joint relationship and then commands a *pose*. The
thing verified and the thing commanded are different objects.

## Measurements

### Pick path consistency (`Chk04`)

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

**Rotation.** Converting both orientations to rotation matrices, sample 1:

| Axis | `CamGraspPose` | `J2P` |
|---|---|---|
| Z (approach direction) | (0.0335, 0.0396, -0.9987) | (0.0335, 0.0396, -0.9987) — identical |
| X | (0.6933, -0.7206, -0.0053) | (-0.6933, 0.7206, 0.0053) — negated |
| Y | (-0.7199, -0.6922, -0.0516) | (0.7199, 0.6922, 0.0516) — negated |

Same Z, X and Y flipped: a pure 180 degree rotation about the tool Z axis, nothing else.
Sample 2 shows the identical signature. The RX/RY sign flips in the raw output are just
how the Euler decomposition re-expresses that rotation.

**Position.**

| | Magnitude | Tool-frame X | Tool-frame Y | Tool-frame Z |
|---|---|---|---|---|
| Sample 1 | 2.442 mm | ~0.21 | ~2.46 | ~0 |
| Sample 2 | 2.468 mm | ~0.21 | ~2.46 | ~0.02 |
| Predicted `2 x (tool X, tool Y)` | 2.493 mm | 0.289 | 2.476 | 0 |

Constant in the tool frame across both samples, and matching twice the tool's lateral
offset. That is what a 180 degree wrist rotation does to a TCP sitting off the rotation
axis - it swings it to the other side.

`CamGraspJoints` and `CamPathJoints(last)` are identical in both samples, so nothing is
inconsistent on the joints side. The entire discrepancy lives in the returned pose.

### Tool offsets (`Chk03`)

```
controller tool  ToolPos(1)          P(-0.144509, -1.23811, 199.9489, 0, 0, 0, -1)
srv flange       CamGetFlangeFK(J)   P(476.3116, 1.20625, 500.0031, -179.9999, -6e-005, 0.00145, -1)
srv TCP          CamGetFK(J, 1)      P(476.1673, 2.44465, 300.0542, -179.9999, -6e-005, 0.00145, -1)
```

`srv flange` and `srv TCP` have *identical* orientations, so Cambrian's tool rotation is
zero. Position delta in base frame is `(-0.1443, +1.2384, -199.9489)`; the flange is at
RX ~ 180 (pointing down) so its axes are X=(1,0,0), Y=(0,-1,0), Z=(0,0,-1). Projecting:

```
Cambrian tool    (-0.1443,   -1.2384,   199.9489, 0, 0, 0)
Controller tool  (-0.144509, -1.23811,  199.9489, 0, 0, 0)
```

Match to four decimal places. **The tool is not the cause.**

## Ruled out, with the evidence

| Candidate | How it was eliminated |
|---|---|
| Robot kinematic calibration | `CamCheckRobotCalibration` returns 1; `srv flange` agrees with the controller |
| TCP / tool offset mismatch | Derived server tool matches `ToolPos(1)` to 4 dp (above) |
| Tool not active | `CurTool` = 1, pendant header shows `W0 T1`, `ToolPos(1)` reads back correctly |
| `TOOL0` redefined | Reads `P(0,0,0,0,0,0,-1)` |
| Ignore Tilt / Ignore Gamma frame override | Both confirmed never enabled |
| Joint twinning budget | Raising `approachTwinJointDeg` had no effect |
| Unconstrained arm configuration | `CamSetRobotConfigurations` made the wind *larger* and earlier - backed out |
| Hand-eye calibration quality | Acceptance numbers pass |

## Corrections made during the investigation

Recorded so the reasoning can be audited rather than trusted.

- **`SingularAvoid 0` is not backwards.** Initially called out as having inverted
  semantics. Cambrian's own DENSO doc gives the same rationale as the code comment
  ("so the wrist cannot silently reconfigure mid-line"), and the documented
  `CamGetPickSequence` pattern also drives its linear legs with it. The claim was
  withdrawn.
- **`J2P` does not return a flange pose.** An early diagnostic compared `CurPos` against
  `J2P( CurJnt )` as if they were TCP vs flange. They are the same quantity, so that
  comparison could never have shown anything. The `ctrl TCP` vs `srv flange` comparison
  was valid and is what actually carried the conclusion.
- **Position offset does not rule out rotation variants.** The 2.45 mm was initially used
  to eliminate gamma/cylindrical variants on the grounds that a variant rotates about the
  grasp axis and preserves position. Wrong: the wrist rotates about the flange Z axis and
  the TCP sits laterally off it, so a variant produces exactly this offset. Variants were
  wrongly excluded for several rounds because of this.
- **Ex13 working does not exonerate the frames.** Used early to dismiss both the TCP and
  the Ignore Gamma override. Ex13 uses no server IK, so a pose-frame error is invisible
  there.

## Current hypothesis

Grasp point **Symmetry = Cylindrical**.

Cylindrical generates rotation variants at `cylindricalRotationTestStep` (default 15
degrees) about a **separately defined axis** - configured via Rotation Center X/Y/Z and
Alignment Y/Z in the grasp point inspector. 180 degrees is exactly 12 steps.

The measured data shows the approach direction (tool Z) **unchanged** between the two
poses. A rotation about a genuine part cylinder axis would tilt the approach direction.
It doesn't - which indicates the configured rotation axis is currently parallel to the
gripper approach axis, turning every cylindrical variant into a pure wrist spin.

## Next step

1. Set grasp point **Symmetry to None**.
2. **Reload the model** - rotation variants are created at model load time, so the change
   does not take effect until then.
3. Re-run `Chk04_PickPathConsistency.pcs`.

If `CamGraspPose` and `J2P( CamPathJoints( last ) )` converge, confirmed.

### If confirmed, permanent fix

- If the part does not genuinely need cylindrical symmetry, leave it at None.
- If it does, fix the **axis** rather than disabling the feature: Rotation Center and
  Alignment should describe the part's own cylinder axis (the centreline of the 130 mm
  rod), not the gripper approach direction. Then narrow **Min/Max Rotation** to the range
  the cell can actually reach, so a 180 degree variant is never generated.

### If not confirmed

Every configuration variable on the integrator side has been eliminated, and two outputs
of a single `CamGetPickPath` contradict each other. Send `bug-report.md`.

Two things to try alongside:

- Run the same comparison through `CamGetPickSequence` (Ex20 pattern), comparing
  `CamGraspPose` against `J2P( CamGraspJoints )`. If pick sequence is self-consistent,
  the defect is isolated to the pick-path code.
- Work around it by driving on joints only, in your own program rather than the library:

  ```pacscript
  Move P, CamHoverJoints,    Speed = ( 100, 100, 50 )
  Move P, CamApproachJoints, Speed = ( 100, 100, 50 )
  Move P, CamGraspJoints,    Speed = ( 20, 20, 10 )
  ```

  Every target is a configuration the server validated, so there is nothing for the wrist
  to reconcile. Costs the guaranteed straight line into the grasp, but over a 10 mm
  approach the PTP deviation is negligible.

## Open items unrelated to the wrist

- **Tool rotation was zeroed during the investigation.** `TOOL1` RZ is currently 0. If
  the physical gripper is genuinely flange-aligned that is correct; if not, restore the
  real value and re-verify with `Chk03`.
- **Version skew.** The DENSO API doc on hand is v1.6 and references `rrtClearanceMarginM`,
  renamed to `testClearanceMarginM` in VISION 2.5. `CamSearchGridAutoInit` is also being
  called with six arguments against a documented five. Confirm `Cambrian_API.pcs` came
  from the same release as the VISION install - 2.5 rejects the connection outright if the
  robot module is older than the minimum for the robot. `CamListApiCommands` dumps the
  server's command list for comparison.
- **`rrtSplitPickPath`.** The library has no `CamGetExitPath`. If anyone enables split
  pick path on the VISION side, Ex26 will execute the approach leg and stop, with no way
  to retrieve the exit leg.
