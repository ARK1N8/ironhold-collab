# TASK: axe-chop-fix-01 — corrected two-handed chop (both axes) + shield placement diagnosis

FROM: Studio Director
DATE: 2026-07-06
SCOPE: GAME project. Implements the axe fix; DIAGNOSES the shield (does not yet fix it).
DEPENDS ON: combat-discovery-01 (root cause established: the chop's blade never leaves
broadside because Tool.Grip cant is fixed for the whole swing; also no low follow-through
and no eased recovery).

## Context — why the current chop reads wrong (from discovery)
The swing rotates only the arms. `Tool.Grip` (axe orientation in the hand) stays at the
fixed idle cant (BeardedAxe −165 / DaneAxe −75) through the entire strike, so the axe
sweeps SIDEWAYS broadside instead of turning edge-down into a vertical arc. Measured at
the strike frame the head vector was (−0.93, +0.25, −0.28) = character-left/up. Fixing arm
angles alone will NOT fix this. The swing must drive THREE channels together:
  (1) arm + torso joints (KeyframeSequence, as today),
  (2) **Tool.Grip re-orientation mid-swing** (broadside → edge-down),
  (3) a **low follow-through** frame + **eased recovery** (today it hard-cuts mid-height→idle).

## PART 1 — Implement the corrected BeardedAxe chop

Angle convention: same `deg(pitch,yaw,roll)` local-offset format the HoldServer already
uses; character faces forward (−Z), up +Y, right +X. Treat all joint numbers as targets to
tune ±10°. The **axe-head world vector** at each frame is the ground-truth cross-check —
after building, freeze each frame and confirm the head points where noted (esp. IMPACT).

6 keyframes over ~0.72s total:

| t | LeftUpperArm | LeftLowerArm | RightUpperArm | RightLowerArm | UpperTorso | Grip cant | Axe-head target |
|---|---|---|---|---|---|---|---|
| 0.00 Ready    | deg(92,12,0)  | deg(38,0,0) | deg(90,-10,0)  | deg(30,0,0) | deg(0,0,0)     | ~−165 (idle) | up-forward, head height |
| 0.18 Wind-up  | deg(145,18,0) | deg(55,0,0) | deg(158,-28,0) | deg(95,0,0) | deg(-18,28,0)  | ~−120 | high, behind/above R shoulder (max load) |
| 0.30 Commit   | deg(120,5,0)  | deg(45,0,0) | deg(130,-15,0) | deg(60,0,0) | deg(0,10,0)    | ~−40  | blade turning over, edge starts leading |
| 0.42 IMPACT   | deg(55,-8,0)  | deg(25,0,0) | deg(48,8,0)    | deg(18,0,0) | deg(22,-18,0)  | ~+5 to 0 | **≈ (0, −0.6, −0.8) down+forward** — verify this frame first |
| 0.56 Follow   | deg(30,-15,0) | deg(?,0,0)  | deg(25,15,0)   | deg(35,0,0) | deg(28,-22,0)  | edge-down | low, near left knee/shin |
| 0.72 Recover  | → idle hold   | → idle      | → idle         | → idle      | deg(0,0,0)     | → −165 (idle) | back to idle hold |

Requirements:
- **Ease-in** into wind-up (0.00→0.18), **fast** strike (0.18→0.42), **hold ~2 ticks** at
  IMPACT (the "bite"), **ease-out** recovery (0.56→0.72). The current linear snap-back is
  half of why it reads unfinished.
- **Drive `Tool.Grip` on the same timeline as the keyframes** so the blade rotates from
  broadside (idle cant) to edge-down/forward by IMPACT, then eases back to idle cant on
  recovery. Implement whatever mechanism is cleanest in HoldServer (e.g. tween Tool.Grip in
  step with the SwingPose track, or key the grip cant per-frame). The blade MUST lead
  edge-first into the target at IMPACT — that is the whole fix.
- Preserve the existing correct plumbing: stop-HoldPose → SwingPose(Action4) → resume idle
  on `.Stopped` (+ the task.delay safety net). Don't regress the resume.

## PART 2 — DaneAxe chop
Same 6-frame shape, but:
- **Slower / bigger**: scale all frame times ×1.25 (~0.90s total) for the longer haft's
  heavier, more committed arc.
- Its idle cant is **−75** (not −165) and it has Grip_LiftY 1.0 — so its grip-sweep start/end
  values differ; sweep from its idle cant to edge-down and back, same principle.
- Verify the head-vector cross-check at its IMPACT frame the same way.

## PART 3 — Shield placement DIAGNOSIS (measure, do NOT fix yet)
Jon reports the RoundShield **placement** is wrong (this is the idle/equip pose, likely the
shield's `Tool.Grip`, not the block animation). Do the same measurement approach that nailed
the axe — don't guess:
- In Play, equip the RoundShield. Report the shield's current `Tool.Grip` (pos + orientation).
- Measure and report, in world/character axes: which way the shield **face** points (the flat
  broad side with the boss/dome), the shield's **height** on the body, and where it's anchored
  (hand vs forearm). Screenshot from front and side.
- Compare to the intended target and report the delta: **shield face broadside toward the
  enemy/forward (not edge-on), boss at ~sternum height, left forearm horizontal across the
  body, covering chest-to-chin.** State exactly how the current pose deviates (e.g. "face
  points +X/right, edge-on to forward" or "boss faces up").
- Do NOT change it — just report the numbers so the Director specs the exact grip fix next.

## Report back (write outbox/axe-chop-fix-01.result.md)
- PART 1: BeardedAxe chop implemented? Paste the final SWING_FRAMES + how Grip is driven.
  Report the measured axe-head vector at the IMPACT frame (should be ~down+forward, NOT
  side). Verified in Play (equip → activate → clean resume, no errors)?
- PART 2: DaneAxe done + its IMPACT head-vector measured?
- PART 3: shield measurements (current Grip, face direction, height, anchor) + the delta vs
  intended. No change made.
- STATUS: DONE / PARTIAL / BLOCKED + anything unexpected.

## Guardrails
- `rojo serve` + Connect only — NEVER `rojo build`.
- Bake changes in Edit mode (Play-session edits vanish on stop). Persist into the StarterPack
  HoldServer templates so they survive.
- Do not touch damage values, the tree-harvest cooldown, or other weapons in this task.
- No secrets in the result (outbox is public).
