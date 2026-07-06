# RESULT: shield-offhand-01 — shield -> toggled passive off-hand + Seax coexistence + damage reduction

FROM: Claude Code (local executor)
DATE: 2026-07-06
STATUS: **DONE** — all 5 parts implemented and verified live in Play.

---

## PARITY (checked first, per task)
Rojo plugin **Connect is a manual Studio UI click I can't perform** — so I verified parity by direct content comparison and, because live-sync is still dropping disk->live pushes, I **edited disk AND pushed to live for every Rojo-managed file** so disk == live after my changes.

| File | Result |
|---|---|
| `Shared/TwoHandHold` | **MATCHED** — content identical (both have easing); byte diff is whitespace only. |
| `Server/Services/CombatService` | **MATCHED before edits** — was byte-identical modulo CRLF (213-byte diff = exactly the 213 CRLF line endings). After my edits: disk + live both updated. |
| Weapon `HoldServer`s (Seax/BeardedAxe/DaneAxe/Spear + the removed RoundShield) | **N/A for disk-parity** — these live inside StarterPack Tools, are **place-only** (not mirrored in `src/`, not Rojo-managed), so they can't drift from disk; they persist in the place file. |

**Conclusion:** no real content drift existed. The prior "shield discrepancy" was consistent with a parity/timing gap. Live-sync remained flaky this session, so I did not rely on it — all Rojo-managed edits (Weapons, CombatService, NPCService) were pushed to live directly.

---

## PART 1 — Shield: Tool -> toggled passive strap-hold  ✅
- **Removed RoundShield from the Tool system** — deleted the StarterPack `RoundShield` Tool. StarterPack now holds only the 5 weapons (Seax, BeardedAxe, DaneAxe, Spear, HuntingBow). Shield no longer occupies the equipped-Tool slot.
- **Kept the mesh + placement** — cloned the shield Handle + its `StrapC0` + arm pose (`LUpArm 25,0,8 / LLoArm 80,0,0`) into `ServerStorage.ShieldMesh` (placement unchanged, as instructed).
- **Toggled persistent state, server-authoritative** — new `ServerScriptService.ShieldService` (standalone Script, same pattern as `IronholdDoorService`, so not Rojo-managed → persists). The real "up" flag is a **server-set `Player` attribute `ShieldUp`** (only the server writes it). Up = mesh cloned onto the character + welded to `LeftLowerArm` (`ShieldStrap` weld, using the captured `StrapC0`) + left-arm pose. Down = mesh/weld/pose removed.
- **Toggle key = `Q`** (free by default; used it). Client `StarterPlayer.StarterPlayerScripts.ShieldToggleClient` LocalScript binds Q -> `ReplicatedStorage.ShieldToggle:FireServer()`. (Touch/gamepad = follow-up, as the task allows.)
- **Does not consume the Tool slot / does not intercept left-click** — the shield isn't a Tool and has no `Activated`; the right-hand weapon Tool (Seax) stays equipped while the shield is up (verified below).
- **Survives weapon swaps** — verified: shield stayed up through a weapon **unequip** (no weapon = allowed).

**Verified live:** fire toggle (Seax equipped) -> `ShieldUp=true`, `ShieldHandle` present, `ShieldStrap` weld -> LeftLowerArm, `ShieldHold` track playing, **Seax still equipped**, shield at (108.4, 5.1, -274.8) with boss face `dot(LookVector)=1.00` (broadside to enemy). Fire again -> `ShieldUp=false`, mesh removed, Seax retained. Full up/down cycle clean.

## PART 2 — Coexistence rules  ✅
- **`Hands` field added to `Weapons.luau`** (Seax=1; BeardedAxe/DaneAxe/Spear/HuntingBow=2; RoundShield=1). Disk + live. Server reads `Weapons.Get(tool.Name).Hands` (unknown Tool -> treated as 2, fail-safe).
- **Shield up only with a one-hander or none** — verified works with Seax; survives unequip.
- **Auto-drop on two-hander** — verified: equipping BeardedAxe (Hands=2) while shield up -> shield auto-dropped (`ShieldUp=false`, handle removed). (Watcher = `character.ChildAdded` for a 2-handed Tool.)
- **Toggle rejected while two-hander equipped** — verified: firing toggle with BeardedAxe equipped left `ShieldUp=false` and the client received a `"denied"` reply (hook for a future UI/sound cue).
- **Server-authoritative** — the `ShieldUp` attribute is written only by ShieldService; the client toggle only *requests*. A forged client attribute wouldn't affect the server's read (CombatService reads the server value).

## PART 3 — Left-click strikes the right-hand weapon  ✅
Confirmed: with **Seax + shield up**, `Tool.Activated` (left-click) fires the **Seax** strike. The shield is not a Tool, has no `Activated`, and never eats the click. Its old block/attack is **retired** (removed with the Tool's HoldServer).

## PART 4 — Passive damage reduction while shield up  ✅
- **Constant `SHIELD_DAMAGE_REDUCTION = 0.25`** in CombatService (clearly named, trivially tunable at the top).
- **Centralized gated helper** `CombatService.ApplyPlayerDamage(humanoid, rawAmount)` — resolves the target player, applies `x (1 - 0.25)` **iff** that player's `ShieldUp` attribute is set, then `TakeDamage`.
- **Routed BOTH player-damage points through it:** PvP (`CombatService.onAttack`) and **NPC->player melee** (`NPCService` line 441, via a lazy `require` inside the melee function to avoid a load-time circular dependency).
- **Verified live:** raw 40 damage -> **40 dealt** with shield down (100->60); **30 dealt** with shield up (100->70) = exactly 25% reduction, gated on the flag.

## PART 5 — Seax strike verification (with shield up)  ✅
- Verified live: equip Seax, shield up, activate -> the Seax **SwingPose plays on the right arm** WHILE the shield's **ShieldHold** track keeps the left arm posed. Key detail: the shield pose uses a **distinct track name `ShieldHold`** (not `HoldPose`), and its parent joints are weight-0 pass-through, so the weapon's `playSequence` (which stops `HoldPose`) doesn't kill the shield pose and doesn't fight the strike's torso lean.
- The shield **stayed put** (moved < 0.02 studs during the slash), stayed up after, and the arm resumed cleanly. **No errors** (console showed only pre-existing noise: door-service plugin-capability warning + a campfire texture asset failing to load + DataManager no-DataStore).
- **Seax strike itself looks correct** — a fast single-hand diagonal slash (authored/verified earlier this session); coexists cleanly with the shield. No rework needed for the shield interaction.

---

## VERIFY (evidence, all live in Play)
- Infra present: `ServerStorage.ShieldMesh`, `ReplicatedStorage.ShieldToggle`, `ServerScriptService.ShieldService`, `StarterPlayerScripts.ShieldToggleClient`; `RoundShield` gone from StarterPack; `Weapons.Seax.Hands=1`, `BeardedAxe.Hands=2`. All persist in Edit mode (created in Edit -> baked into the place file).
- Raise: ShieldUp=true, welded to LeftLowerArm, boss·look=1.00, Seax retained.
- Seax strike + shield up: SwingPose + ShieldHold both playing mid-slash; shield delta < 0.02 studs; resumes clean.
- Coexistence: survives unequip=true; equip 2-hander auto-drops=true; toggle-while-2-hander -> denied + ShieldUp stayed false.
- Damage reduction: down 40->health60; up 40->health70 (25% off).
- Guardrails honored: `rojo serve`/live only, **no `rojo build`**; persistent changes baked in Edit mode; server-authoritative flag; did not change outgoing weapon damage, tree-harvest cooldown, the axe chops, or other weapons' animations.

## STATUS: DONE

## BACK TO DIRECTOR
1. **Touched `NPCService` line 441 (that one line only)** to route NPC->player melee through `CombatService.ApplyPlayerDamage` — required to satisfy Part 4's "NPC->player damage path" (that's where NPC melee damages players; it couldn't be reduced otherwise). Flagging since NPCService wasn't in the explicit touch-list; the change is purely the shield-reduction routing.
2. **Screenshots timed out repeatedly this session** (Studio busy) — verification is numeric (shield pos, boss·look=1.00, coexistence, reduction math). The shield mesh/placement visual is unchanged from the shield-diagnosis task's front/side captures (same mesh, same `StrapC0`).
3. **Rojo live-sync still flaky** (disk->live didn't auto-push) — I pushed every Rojo-managed edit to live directly to keep parity. A clean plugin reconnect + parity pass is still advisable before the next task.
4. `Q` was free; used it. The RoundShield weapon **data** entry remains in `Weapons.luau` (Hands=1) — harmless, just no longer a Tool. The `"denied"` client reply is a ready hook if you want a UI/sound cue for rejected toggles.
