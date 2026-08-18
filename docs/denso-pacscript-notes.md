# PacScript notes

Working reference for the calls this project leans on. Organised **by confidence**,
because several confident-sounding assertions during the J6 investigation turned out
wrong, and a note that admits uncertainty is worth more than one that doesn't.

Full manual: [`../reference/DensoRC8_manual.md`](../reference/DensoRC8_manual.md)
(VS series, covers VS-050/060/068/087). Jump to any command via
[`../reference/COMMAND_INDEX.md`](../reference/COMMAND_INDEX.md):

```bash
sed -n '15663,15722p' reference/DensoRC8_manual.md     # J2P
```

---

## Confirmed against the manual

| Call | Behaviour |
|---|---|
| `J2P( joints )` | Joint → Position. **Reflects the current work AND current tool coordinates.** Not a flange pose. Out-of-range joints still compute — use `OutRange` to check. |
| `DevH( ref, dev )` | Deviation applied in the **reference pose's own coordinate system** — origin from its X/Y/Z, orientation from its Rx/Ry/Rz. This is what makes `CamOffsetPose(p, 0,0,20, 0,0,0)` mean "20 mm along the tool Z". |
| `ToolPos( n )` | Returns tool coordinate definition `n` as a Position. |

The `J2P` line matters more than it looks: it is why `J2P( CurJnt )` equals `CurPos`
rather than giving a flange pose, and why `CamExecutePathReverse`'s
`Move L, J2P( CamPathJoints( N ) )` resolves through the active tool.

## Verified in this cell (observed on the VS068A4A)

- `ToolPos( n )` prints as `P(x, y, z, rx, ry, rz, fig)`; `ToolPos(0)` reads all zeros
  on an untouched controller.
- `PrintDbg <Position>` prints all six components plus FIG. `PrintDbg <Joint>` prints
  eight elements (`J(j1..j6, 0, 0)`).
- FIG `-1` means unspecified/auto. `CamGraspPose` from `CamGetPickPath` carries `-1`;
  `J2P( joints )` carries a concrete figure.
- `CurTool` returns the active tool number; the pendant header shows it as `W0 T1`.
- `PosX/PosY/PosZ/PosRx/PosRy/PosRz( pose )` extract components and compile fine.

## Read from `Cambrian_API.pcs` v1.6

Not manual-verified, but demonstrated by how the vendor library uses them.

| Call | Apparent behaviour |
|---|---|
| `CurPos` | TCP pose in the **active work frame** |
| `ConvertPosTool( pose, fromTool, toTool )` | Re-express a pose between tool frames |
| `ConvertPosWork( pose, fromWork, toWork )` | Re-express between work frames |
| `Dev( ref, dev )` | Deviation in the **base** frame (contrast `DevH`) — used by `CamSetHover` |
| `P2J( pose )` | IK. Wrapped in `On Error` in `CamInverseKinematics` — it throws when unreachable |
| `OutRange( pose )` | Returns 0 when the pose is inside the motion space |
| `Joint( n, jointvar )` | Extract joint *n* |
| `Fig( pose )` / `CurFig` | Read a pose's figure / the live arm figure |
| `LetF pose = n` | Set the figure element (also `LetX`/`LetY`/`LetZ`/`LetRx`/`LetRy`/`LetRz`) |
| `Comm.Open/Close/Clear/Output/Input/State` | Channel comms; `Comm.Input( ch, timeout )` |
| `Speed <n>` | **Global statement**, persists after the Sub that set it returns |

Motion syntax as used by the library:

```pacscript
Move P, [@n] target [, Speed = (a,b,c)] [, Next]
Move L, [@n] target
```

`@0` settles exactly, `@C` is the settled/continuous form, `@<n>` is a **pass-blend
radius in millimetres — not a figure index**. `Next` makes the move non-blocking.

## Learned the hard way

These cost real time during the J6 investigation.

- **`Fig` encodes J1 (front/back), J3 (elbow), J5 (wrist flip) — not J6.** Two IK
  solutions 180° apart in J6 carry *identical* figures, so a figure guard cannot catch
  a wrist winding difference.
- **Euler components do not add independently.** A 180° tool rotation about Z can
  surface in RX or RY instead of RZ depending on the flange orientation. Never compare
  a single Euler component across two poses — compare all six, or compare rotation
  matrix axes.
- **A 180° rotation about Z leaves the Z axis unchanged** and flips X and Y. So it
  cannot reverse the sense of rotation *about* Z — only the direction of X/Y travel.
- **Joint-mode and Cartesian-mode jog signs disagree when the flange points down**,
  because flange Z is then anti-parallel to base Z. Normal, not a fault.
- **A constant offset in the *tool* frame across different robot poses** points at a
  frame or tool definition problem, not at the planner.

## Known bugs in `Cambrian_API.pcs` v1.6

Reported upstream; see [`bug-report.md`](bug-report.md).

1. **`CamGetPickPath` does not FK-normalise `CamGraspPose`.** `CamGetIKEx` explicitly
   does — with a long comment explaining exactly this hazard — and the same fix was
   never applied to the pick-path parser. Workaround, in your own program:
   ```pacscript
   CamGraspPose = J2P( CamGraspJoints )
   ```
2. **`CamGetFlangePose` uses the wrong conversion.** It calls
   `ConvertPosTool( pose, CurWork, 0 )`, passing a *work* number into a *tool*
   conversion; should almost certainly be `ConvertPosWork`. Only fires when
   `CurWork <> 0` — and `CamCreateRequestString` sends that flange pose on **every**
   command, so a non-zero work frame poisons everything at once. Set `ChangeWork 0`.
3. **`CamCreateRequestString` computes `TCPOffset = ToolPos( CurTool )` and never
   sends it.** The server derives the tool from the flange/TCP pose pair instead.
   Harmless, but explains why the tool reaches Cambrian only implicitly.

## Version note

The module header reads `Requires CamVISION 2.4.x (GET VERSION protocol 1.6)`; this
cell runs **2.5.1**. `CamInit` only warns when the server is *older* than required, so
a 2.4-era module runs silently against a 2.5 server. Ask Cambrian for the 2.5 Denso
module before relying on 2.5-only features (split pick path in particular — this
module has no `CamGetExitPath`).
