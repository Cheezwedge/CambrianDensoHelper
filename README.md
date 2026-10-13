# CambrianDensoHelper

Diagnostic PacScript programs and setup notes for a Cambrian VISION cell running on a
DENSO robot.

**Cell this was built against**

| | |
|---|---|
| Robot | DENSO VS068A4A (RC8 controller) |
| Cambrian VISION | 2.5.1 |
| DENSO API module | `Cambrian_API.pcs` (doc v1.6) |
| Camera rig | Flange mounted |
| End effector | Vacuum, `TOOL1`, ~200 mm |
| AI model | `12x130_Kii_BLN_10Aug2026_161309_10-08-2026` (Full Pose) |

## Layout

```
programs/   Diagnostic PacScript programs. Each is standalone - load, run, read the log.
docs/       Setup checklist, investigation record, and a support report draft.
```

## Start here

If something is wrong and you don't know where, work
[`docs/troubleshooting-checklist.md`](docs/troubleshooting-checklist.md) top to bottom.
It is ordered so that each step's assumptions are verified by the steps above it.

If you are chasing the J6 wrist rotation specifically, read
[`docs/findings.md`](docs/findings.md) - it records what has already been ruled out and
how, so you don't repeat the work.

If you are new to this cell entirely, read [`CLAUDE.md`](CLAUDE.md) first. It covers the
architecture decisions and why they were made, the conventions, the gotchas, and the
approaches that were tried and failed - so you don't spend time re-deriving any of it.
[`CHANGELOG.md`](CHANGELOG.md) is the dated history.

## Programs

| Program | Answers |
|---|---|
| `Chk01_RobotCalibration.pcs` | Has the robot kinematic calibration ever been run? |
| `Chk02_RigPoseIterate.pcs` | Fast iteration on the camera rig pose without a full calibration sweep |
| `Chk03_ToolCompare.pcs` | Do the controller and Cambrian agree about the TCP? |
| `Chk04_PickPathConsistency.pcs` | Do `CamGraspPose` and `CamPathJoints` from one `CamGetPickPath` agree? |
| `Cal01_AutoCalibration.pcs` | Hand-eye calibration with the setup steps that are easy to forget |

Every program prints to the teach pendant message log via `PrintMsg` / `PrintDbg`.

## Conventions worth remembering

**Units are not consistent across this API.** The two that bite:

| Call | Units |
|---|---|
| `CamGetPickPath`, `CamGetPickSequence` approach/hover distance | **metres** (`0.010` = 10 mm) |
| `CamGetApproachPose`, `CamGetHoverPose`, `CamGetExitPose` | **millimetres** |
| `CamAutoCalibration` X/Y/Z | mm, but `StereoSeparation` is metres |
| `CamOffsetPose` | mm and degrees |
| Raw TCP API (bypassing the wrapper) | metres and radians throughout |

**`J2P()` is not a flange pose.** It converts joints to a Cartesian position through the
*currently active* tool and work coordinates. `J2P( CurJnt )` and `CurPos` are the same
quantity. To read a genuine flange pose, switch to `TOOL0` first, or use
`CamGetFlangeFK()` on the server side.

**Do not edit `Cambrian_API.pcs`.** `CamExecutePath`, `CamExecutePathEx`,
`CamExecutePathReverse` and friends are vendor library code. Local edits are lost on the
next API update, and support will ask you to reproduce on stock.

## Reference

`reference/DensoRC8_manual.md` is the DENSO RC8 VS-series manual (VS-050/060/068/087),
converted to Markdown. It is vendor documentation kept here for lookup convenience —
DENSO WAVE's copyright, not ours. Do not redistribute it outside this repository.

`reference/COMMAND_INDEX.md` maps 565 PacScript commands and functions to line numbers
in that file. To read one:

```bash
sed -n '15663,15722p' reference/DensoRC8_manual.md     # J2P
```

`docs/denso-pacscript-notes.md` is the distilled version — the calls this project
actually uses, organised by how well each one is confirmed.
