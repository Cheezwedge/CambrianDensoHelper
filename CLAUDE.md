# CLAUDE.md

Orientation for a future session — human or AI — picking this up cold.

## What this repo is

A **diagnostic and reference toolkit** for one Cambrian VISION cell running on a DENSO
robot. It is not an application and has no build. It holds:

- standalone PacScript programs that answer one question each,
- an investigation record for an unresolved wrist-rotation bug,
- the DENSO RC8 manual plus a generated command index.

The robot programs that do real work (`Ex13_approach_parts`, `EX26_BinPick_Auto`) live
on the controller, **not here**. Edits to them are described in `docs/findings.md`; the
files themselves are not version-controlled in this repo.

## The cell

| | |
|---|---|
| Robot | DENSO VS068A4A, RC8 controller |
| Cambrian VISION | 2.5.1 |
| DENSO API module | `Cambrian_API.pcs` v1.6.0 — header says *requires 2.4.x* |
| Camera rig | Flange mounted |
| End effector | Vacuum cup, `TOOL1`, Z ≈ 200 mm |
| AI model | `12x130_Kii_BLN_*` (Full Pose), 12 × 130 mm rod |
| Comms | `CamInit 8` — channel 8 |

## Architecture decisions, and why

**Diagnostic programs are standalone and read-only where possible.**
Each `programs/Chk*.pcs` answers exactly one question and prints to the pendant log.
Most never command motion. Rationale: they can be run mid-debug, on a cell in any
state, without re-teaching anything or risking a crash.

**Programs are numbered by dependency, not by topic.**
`Chk01` → `Chk02` → `Cal01` → `Chk03` → `Chk04`. Each step's assumptions are verified by
the steps above it. Running them out of order produces measurements you cannot trust —
e.g. a tool comparison means nothing if the kinematic calibration has never been run.

**Never edit `Cambrian_API.pcs`.**
It is vendor library code. Local edits are lost on the next API update, and Cambrian
support will ask for a reproduction on stock. Workarounds belong in *your* program,
exploiting the fact that the library's `Dim`s are module-scope and therefore writable
from the calling program. The `CamGraspPose = J2P( CamGraspJoints )` fix works precisely
because of this.

**Documentation is organised by confidence, not by topic.**
`docs/denso-pacscript-notes.md` separates *confirmed against the manual* from *observed
on this cell* from *inferred from library source*. This is not pedantry — several
confident-sounding assertions during the investigation turned out wrong, and the
labelling is what makes the rest trustworthy.

**The manual is one Markdown file plus a generated index.**
Not the original HTML tree. One file is greppable and `sed`-addressable;
`reference/COMMAND_INDEX.md` maps 565 commands to line numbers, which is what makes
336k lines usable. Images were deliberately **excluded** — of the nine command sections
this project works in, 90 of 93 image references were navigation chrome and only three
carried content.

## Conventions

**Units are inconsistent across the Cambrian API.** This is the single biggest footgun.

| Call | Units |
|---|---|
| `CamGetPickPath`, `CamGetPickSequence` approach/hover | **metres** (`0.010` = 10 mm) |
| `CamGetApproachPose`, `CamGetHoverPose`, `CamGetExitPose` | **millimetres** |
| `CamAutoCalibration` X/Y/Z | mm — but `StereoSeparation` is metres |
| `CamOffsetPose` | mm and degrees |
| Raw TCP API | metres and radians throughout |

**Program prefixes.** `Chk*` reads and reports. `Cal*` performs a calibration. Anything
that moves the robot says so in its header comment.

**Every program sets `ChangeTool 1` and `ChangeWork 0` explicitly** before talking to
Cambrian. Never inherit ambient controller state — see the `CamGetFlangePose` bug below.

## Gotchas

These cost real time. Each one is a thing we got wrong first.

- **`J2P()` is not a flange pose.** The manual states it reflects the current work *and*
  tool coordinates, so `J2P( CurJnt )` equals `CurPos`. For a genuine flange pose use
  `TOOL0`, or `CamGetFlangeFK()` server-side.
- **`CurPos` is expressed in the active work frame.** Select work 0 before comparing it
  against anything Cambrian returns.
- **`Fig` encodes J1 (front/back), J3 (elbow), J5 (wrist flip) — not J6.** Two IK
  solutions 180° apart in J6 carry *identical* figures, which is why the figure guard in
  `CamExecutePathEx` cannot catch a wrist winding difference.
- **Euler components do not add independently.** A 180° tool rotation about Z can surface
  in RX or RY depending on the flange orientation. Compare all six components, or compare
  rotation-matrix axes — never a single Euler component.
- **A 180° rotation about Z leaves the Z axis unchanged**, flipping only X and Y. It
  therefore cannot reverse the sense of rotation *about* Z.
- **Joint-jog and Cartesian-jog signs disagree when the tool points down**, because
  flange Z is then anti-parallel to base Z. Normal geometry, not a fault.
- **A constant offset in the *tool* frame across different robot poses** indicates a frame
  or tool-definition problem, not a planner problem.
- **`Ex13` working proves nothing about frames.** It uses no server IK, so a pose-frame
  error is invisible there — with a symmetric cup the pick succeeds regardless. This
  misled the investigation twice.

## Things we tried that did not work

Recorded so nobody repeats them.

| Attempt | Result |
|---|---|
| `CamSetRobotConfigurations` to constrain the arm branch | **Made it worse.** The rotation grew and started earlier — the solver reached the same grasp the long way round. Backed out. |
| Raising `approachTwinJointDeg` (joint twinning) | No effect. Twinning reconciles approach-vs-grasp IK; it does not touch the returned pose. |
| Zeroing the tool `RZ` | No effect — it was already zero by then. The 180° persisted. |
| Chasing the Ignore Tilt / Ignore Gamma frame override | Dead end. Both confirmed never enabled. |
| Comparing `CurPos` against `J2P( CurJnt )` as TCP vs flange | **Invalid measurement.** They are the same quantity, so the comparison could never show anything. |
| Using the 2.45 mm position offset to rule out rotation variants | **Wrong reasoning.** A wrist rotation happens about *flange* Z and the TCP sits off that axis, so a variant *does* shift position by `2 × (tool X, tool Y)`. Variants were wrongly excluded for several rounds because of this. |

## Known bugs in `Cambrian_API.pcs` v1.6

1. **`CamGetPickPath` does not FK-normalise `CamGraspPose`.** `CamGetIKEx` explicitly
   does, with a long comment explaining this exact hazard; the same fix was never applied
   to the pick-path parser. This is the root of the wrist wind on the final linear leg.
   Workaround in the calling program: `CamGraspPose = J2P( CamGraspJoints )`.
2. **`CamGetFlangePose` uses the wrong conversion** — `ConvertPosTool( pose, CurWork, 0 )`
   passes a *work* number into a *tool* conversion. Only fires when `CurWork <> 0`, but
   `CamCreateRequestString` sends that flange pose on **every** command, so a non-zero
   work frame corrupts everything at once. Mitigated by always setting `ChangeWork 0`.
3. **`CamCreateRequestString` computes `TCPOffset` and never sends it.** Harmless; the
   server derives the tool from the flange/TCP pose pair instead.

## Version skew

The module header reads `Requires CamVISION 2.4.x`; the cell runs **2.5.1**. `CamInit`
only warns when the server is *older* than required, so a 2.4-era module runs silently
against a 2.5 server. **Get the 2.5 DENSO module from Cambrian before relying on any
2.5 feature** — in particular `rrtSplitPickPath`, since this module has no
`CamGetExitPath` and a program using split pick path will execute the approach leg and
then stop, never leaving the bin.

## Open threads

1. **Transit-leg J6 rotation.** The final linear leg is fixed. The `Move P,
   CamPathJoints(N)` transit legs still wind 180°, because the server's grasp IK picks a
   wrist branch 180° from the pose it reports and the RRT plans a continuous path there.
   Not reachable from the robot program.
2. **Grasp point Symmetry is `Cylindrical`.** The configured rotation axis appears to be
   parallel to the gripper approach axis, turning every cylindrical variant into a pure
   wrist spin. Next test: set Symmetry to **None**, **reload the model** (variants are
   built at load time), re-run `Chk04`. Fix is to correct Rotation Center / Alignment to
   describe the rod's centreline, and narrow Min/Max Rotation.
3. **Model changed** from the `10Aug2026` to the `11Aug2026` build. Grasp points and
   Symmetry are per-model — re-verify on the current one.
4. **TCP length unresolved.** `TOOL1` Z is 199.9489 and `PushIn` is 20 mm. Either the
   tool is ~20 mm too long and `PushIn` compensates, or the tool is right and `PushIn` is
   genuine cup preload. A ruler from the flange mounting face to the cup face settles it.
   This matters beyond reach: the TCP is sent on every command and drives Cambrian's IK,
   collision checking, and all approach/hover offsets.

## Working style that paid off

Pasting **specific** documentation sections moved things immediately; the single most
valuable artifact was `Cambrian_API.pcs` itself, because it showed what the code does
rather than what it is documented to do. Bulk uploads were worth much less than targeted
ones.

When a measurement contradicts a hypothesis, **check the measurement before the
hypothesis** — two separate dead ends here were instrumentation bugs, not real findings.
