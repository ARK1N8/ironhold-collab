# RESULT: axe-chop-fix-01 — corrected two-handed chop (both axes) + shield placement diagnosis

FROM: Claude Code (local executor)
DATE: 2026-07-06
STATUS: **DONE** — both axe chops implemented & verified (head lands DOWN at impact); shield measured (no change made).

---

## Shared change — easing support in TwoHandHold
Added optional per-frame easing to `src/Shared/TwoHandHold.luau`: `buildKeyframe(t, poseMap, easing)` /
`playSequence(char, frames, looped)` where a frame may be `{time, poseMap, {style=Enum.PoseEasingStyle, dir=Enum.PoseEasingDirection}}`.
Backward-compatible (omit → Linear).
**Gotcha:** `Enum.PoseEasingStyle` does **not** include `Sine`/`Quad` — its valid styles are Linear/Constant/**Cubic**/Elastic/Bounce. Used **Cubic** (In/Out) for the ease.
**Rojo note:** live-sync was NOT pushing the disk edit into the running place this session, so I pushed the identical
TwoHandHold source into the live `ReplicatedStorage.Shared.TwoHandHold` to keep disk == live (no drift). Flagging the sync flakiness below.

---

## PART 1 — BeardedAxe corrected chop  ✅

**Implemented.** Three channels driven together, baked into `StarterPack.BeardedAxe.HoldServer`:

**(1) Joints — SWING_FRAMES** (6 keys + 1 held bite, ~0.72s; `deg(pitch,yaw,roll)` local offsets; `EO`=Cubic/Out, `EI`=Cubic/In):
```lua
{0.00, {LUp(92,12,0)  LLo(38,0,0) RUp(90,-10,0)  RLo(30,0,0) UT(0,0,0)}},
{0.18, {LUp(145,18,0) LLo(55,0,0) RUp(158,-28,0) RLo(95,0,0) UT(-18,28,0)}, EO},   -- windup (eased load)
{0.30, {LUp(120,5,0)  LLo(45,0,0) RUp(130,-15,0) RLo(60,0,0) UT(0,10,0)}},          -- commit
{0.42, {LUp(55,-8,0)  LLo(25,0,0) RUp(48,8,0)    RLo(18,0,0) UT(22,-18,0)}, EI},    -- IMPACT (accelerated)
{0.48, {  = impact pose (held ~2 ticks — the "bite") }},
{0.56, {LUp(30,-15,0) LLo(15,0,0) RUp(25,15,0)   RLo(35,0,0) UT(28,-22,0)}},        -- low follow-through
{0.72, {  = idle Ready pose }, EO},                                                -- eased recovery
```
**(2) Tool.Grip cant, same timeline** — a `RunService.Heartbeat` loop reads `SwingPose.TimePosition` and sets
`tool.Grip = CFrame.new(Grip_Pos) * CFrame.Angles(0,0,rad(interpCant(tp)))` (piecewise-linear interp). This is the actual fix — the blade rotates broadside→edge-down in step with the arms. The loop is disconnected on resume.
```lua
CANT_KEYS = { {0,-165},{0.18,-170},{0.30,-215},{0.42,-255},{0.48,-255},{0.56,-250},{0.72,-165} }
```
**(3) Held bite + eased recovery** — the 0.42→0.48 held frame + Cubic/Out on windup & recovery (replaces the old linear snap-back).
Existing plumbing preserved: stop-HoldPose → SwingPose(Action4) → resume idle on `.Stopped` (+ task.delay safety net). Grip loop cleaned up.

**IMPACT head-vector (measured live, frozen at t≈0.44):** `(−0.25, −0.97, −0.01)` — i.e. **pointing straight DOWN** (was `(−0.93, 0.25, −0.28)` = sideways-left before). Handle drops to Y≈3.1 (ground level in front). Screenshot confirms a clean vertical downward chop.

**Geometry note (important for your spec):** the literal target `(0,−0.6,−0.8)` down+**forward** is not fully reachable by a cant sweep at this impact pose. Measured at impact, the grip's roll axis (RightGripAttachment LookVector) = `(0.09,−0.23,−0.97)` ≈ **forward**. A cant sweep can only move the head *perpendicular* to the roll axis — so it can point the head up/down/left/right (the vertical plane facing forward) but **not** forward/back (that's along the axis). I therefore set the sweep to land the head **straight down** (the reachable best; `dot` to the forward-tilted target was 0.41 purely because its −Z part is unreachable). Straight-down reads as a correct chop. If you specifically want down+**forward**, that needs the impact **arm** pose changed so the hand tilts the roll axis downward — say the word and I'll add that.

**Verified in Play:** equip → `Tool:Activate()` → chop fires → resumes HoldPose (idle head-vector back to broadside `(−0.97,0.26,0.01)`), grip loop disconnected (grip static after), no HoldServer errors. (Pre-existing unrelated console noise: `BuildController:553` client bug + a campfire texture asset failing to load — not touched.)

---

## PART 2 — DaneAxe corrected chop  ✅

**Implemented** in `StarterPack.DaneAxe.HoldServer` — same 6-key shape, **times ×1.25 (~0.90s)** for the heavier, more committed arc. Idle cant **−75** + `Grip_LiftY 1.0` preserved (the lifted idle grip is restored on resume; the swing sweeps the plain cant, then `apply()` re-adds the lift).
DaneAxe's head is at **+Y** (handle UpVector), so `head = att.Right·sin(cant) + att.Up·cos(cant)`. Since the impact arm pose is identical to BeardedAxe, the attachment orientation is the same → solved for head-down → impact cant **−176**:
```lua
CANT_KEYS = { {0,-75},{0.225,-80},{0.375,-130},{0.525,-176},{0.600,-176},{0.700,-172},{0.900,-75} }
```
**IMPACT head-vector (measured, frozen t≈0.55):** `(−0.07, −1.00, +0.03)` — **straight down**. Handle drops to Y≈2.74.
**Verified in Play:** fires, resumes, lifted idle restored (both hands measured 0.94/0.95 studs off the haft axis), grip loop disconnected, no errors.

---

## PART 3 — RoundShield placement DIAGNOSIS (no change made)

Equipped RoundShield, measured the idle/equip pose. **The shield is anchored by a `ShieldStrap` weld (Part0 = LeftLowerArm), NOT by `Tool.Grip`** — the default RightGrip is killed on equip, so `Tool.Grip` (baked value `CFrame.new(0,0,-1)*...`) is **not** what positions the shield; the strap weld's C0 is.

| Measured (world / character axes) | Value |
|---|---|
| Anchor | `ShieldStrap` weld → **LeftLowerArm** (forearm-anchored) |
| Handle.Pos | (108.53, 5.25, −274.86) |
| HRP.Pos | (109.60, 5.20, −273.00) |
| Shield vs body | **1.07 studs LEFT** of centerline, **+0.05 above HRP** (≈ mid-torso/sternum), **1.86 forward** (in front of body) |
| Shield face normal (boss/dome side = −Z of the disc) | **(0, 0, −1)**, `dot(LookVector)` = **+1.00** → **broadside toward the enemy** (NOT edge-on) |
| Handle.Size | (3.74, 3.75, 1.45); disc radius ≈1.87 → covers Y≈3.4→7.1 (≈ waist to neck/chin) |
| LeftLowerArm.LookVector | **(−0.14, 0.96, 0.25)** → forearm points **UP**, not horizontal |

**Delta vs intended** (face broadside to enemy / boss ~sternum / forearm horizontal across body / covers chest-to-chin):
- Face broadside toward enemy: **MATCHES** (normal·look = +1.00, not edge-on).
- Boss ~sternum height: **~MATCHES** (relY +0.05 ≈ mid-torso).
- Forearm anchored to left forearm: **MATCHES**.
- Forearm **horizontal** across body: **DEVIATES** — the forearm actually points **up** (LookVector +Y 0.96); the correct-looking shield placement is produced by the `StrapC0` weld compensating for the arm angle, not by a horizontal forearm.
- Covers chest-to-chin: **~partial** — covers ~waist→neck, centered at sternum; head exposed above (normal for a shield).
- Extra: shield sits ~**1.07 studs left** of the body centerline (natural for a left-arm carry, but not centered).

**Honest finding:** by both measurement **and** the front + side screenshots, the current live placement reads as **correct** on the key axes (broadside to enemy, sternum height, forearm-anchored, torso covered) — I could **not** reproduce an obviously-wrong placement. The only literal spec deviation is the forearm not being horizontal (the strap weld does the positioning). See BACK TO DIRECTOR — this may already be fixed (I rebuilt the shield placement earlier this session) or Jon may be viewing a different/unsynced state.

---

## VERIFY (evidence)
- BeardedAxe & DaneAxe: baked into their StarterPack HoldServers; live Play freeze-frame at impact measured head-vectors **(−0.25,−0.97,−0.01)** and **(−0.07,−1.00,+0.03)** respectively (both straight DOWN); full-play confirmed clean resume to HoldPose + grip-drive Heartbeat disconnected + no HoldServer errors. Side screenshots show vertical downward chops.
- Shield: measured `Tool.Grip`, `ShieldStrap` weld anchor, handle pos/size, face normal·look = +1.00, forearm LookVector; front + side screenshots captured. No change made.
- Guardrails honored: `rojo serve`/live only, **no `rojo build`**; changes baked into StarterPack HoldServer templates (Edit mode) so they persist; TwoHandHold change on disk + mirrored live. Did not touch damage values, tree-harvest cooldown, or other weapons. No secrets in this file.

## STATUS: DONE

## BACK TO DIRECTOR
1. **Down vs down+forward at impact:** both chops now land the head **straight down** (verified). True down+**forward** isn't reachable by the cant sweep alone at this arm pose (roll axis points forward — see Part 1 geometry note). If you want the axe angled forward into the target, I'll adjust the impact **arm** pose to tilt the roll axis down — small follow-up.
2. **Shield placement — can't reproduce the defect.** Current live state measures correct on face (broadside, +1.00), height (sternum), and anchor (forearm); only deviation is the forearm pointing up rather than horizontal (strap weld compensates). Two likely explanations: (a) I **rebuilt the shield placement earlier this session** so it may already be fixed relative to what Jon saw, or (b) Jon is viewing a **different/unsynced state** (see #3). Please confirm what specifically looks wrong on the *current* live shield, or share a screenshot — then I can target the exact fix.
3. **Rojo live-sync was flaky again this session** — disk edits to `src/Shared/*` did not auto-appear in the running place; I pushed the TwoHandHold change into live directly to keep parity. If Jon and I are on different Studio states, that would explain the shield discrepancy. Worth a clean reconnect + parity check before the next placement task.
4. **Concurrent editing:** these HoldServers live in the shared Studio place that the collab bus also touches — flagging in case another session edits the same weapons.
