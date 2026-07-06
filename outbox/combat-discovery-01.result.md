# RESULT: combat-discovery-01 — animation API + chop diagnosis + damage model dump

FROM: Claude Code (local executor)
DATE: 2026-07-06
STATUS: **DONE** — discovery only; the sole change is the Part C doc edit.

---

## PART A — Animation system surface

### A.1 Runtime animation module — `ReplicatedStorage.Shared.TwoHandHold`
(Version-controlled at `src/Shared/TwoHandHold.luau`; Rojo-managed. It is what drives every hold + swing.)

**Public API (exact signatures):**
```lua
TwoHandHold.play(character: Model, poseMap: {[string]: CFrame}, duration: number?) -> AnimationTrack?
TwoHandHold.playSequence(character: Model, frames: { {number, {[string]: CFrame}} }, looped: boolean?) -> AnimationTrack?
```
- `play` — a single **looped** idle hold. Builds a 2-keyframe KeyframeSequence (t=0 and t=duration, identical poses so it holds). Track is named **"HoldPose"**, priority **Action4**. Stops any existing "HoldPose" first.
- `playSequence` — a **multi-keyframe swing/attack**. `frames` = ordered list of `{time, poseMap}`. Track named **"SwingPose"**, priority **Action4**, `looped=false` for attacks. **Stops any existing "HoldPose" first** — this is how a swing overrides the idle.

**How a swing is defined & played:** runtime `KeyframeSequence`, no uploaded assets.
- `buildKeyframe(t, poseMap)` creates a `Keyframe` at time `t` containing a **hierarchical `Pose` tree**. Each `Pose.CFrame` is a **per-joint LOCAL rotation offset** (relative to rig neutral — use `CFrame.Angles`), `Weight = 1`.
- Registered via `KeyframeSequenceProvider:RegisterKeyframeSequence(ks)` → content id → `Animator:LoadAnimation(anim)` → `track:Play(0)`.

**Addressable rig parts** (the `PARENT` hierarchy map, so poses nest correctly):
`HumanoidRootPart → LowerTorso → UpperTorso → {Head, LeftUpperArm, RightUpperArm}`, `LeftUpperArm → LeftLowerArm → LeftHand`, `RightUpperArm → RightLowerArm → RightHand`.
Keyframe fields per pose: `Name` (part), `Time`, `CFrame` (rotation offset), `Weight`. Rig is constraint-based (AnimationConstraint), but KeyframeSequence poses map by **part name**, so this works.

**Priority / blend model:** idle hold and swings **both** play at `Enum.AnimationPriority.Action4`. A swing does **not** win by priority — it wins because `playSequence` explicitly **stops the "HoldPose" track** (`Stop(0)`). After the swing, the weapon's HoldServer resumes the idle on the track's `.Stopped` event (with a `task.delay` safety net). Net flow: **stop-hold → play-swing → on `Stopped`, re-apply hold.** Verified live: hold resumes cleanly.

### A.2 Current BeardedAxe swing — lives in `StarterPack.BeardedAxe.HoldServer` (a server `Script`)
(Not in WeaponClient. WeaponClient only fires the `Attack` remote for damage; the swing visual is server-driven off `Tool.Activated`.)
```lua
-- Two-handed diagonal chop: idle -> overhead windup -> downward strike -> recover.
local SWING_FRAMES = {
  {0.00, {LeftUpperArm=deg(92,12,0),  LeftLowerArm=deg(38,0,0), RightUpperArm=deg(90,-10,0),  RightLowerArm=deg(30,0,0), UpperTorso=deg(0,0,0)}},
  {0.14, {LeftUpperArm=deg(140,10,0), LeftLowerArm=deg(50,0,0), RightUpperArm=deg(160,-20,0), RightLowerArm=deg(20,0,0), UpperTorso=deg(-15,20,0)}},
  {0.34, {LeftUpperArm=deg(40,-10,0), LeftLowerArm=deg(40,0,0), RightUpperArm=deg(30,10,0),   RightLowerArm=deg(20,0,0), UpperTorso=deg(25,-20,0)}},
  {0.50, {LeftUpperArm=deg(92,12,0),  LeftLowerArm=deg(38,0,0), RightUpperArm=deg(90,-10,0),  RightLowerArm=deg(30,0,0), UpperTorso=deg(0,0,0)}},
}
local function doSwing()
  if swinging then return end
  local char = tool.Parent
  if not char or not char:FindFirstChildOfClass("Humanoid") then return end
  swinging = true
  if track then track:Stop(0) end
  local sw = hold.playSequence(char, SWING_FRAMES, false)
  if not sw then swinging = false; apply(char); return end
  local resumed = false
  local function resume()
    if resumed then return end
    resumed = true; swinging = false
    if tool.Parent and tool.Parent:FindFirstChildOfClass("Humanoid") then apply(tool.Parent) end
  end
  sw.Stopped:Connect(resume)
  task.delay(1.0, resume) -- safety net
end
-- onEquip: apply(char) [idle hold]; connect tool.Activated -> doSwing.
-- The axe stays welded to the RIGHT hand via Tool.Grip (Grip_Pos (-2.2,0,0), Grip_Cant -165); the swing does NOT change Grip.
```

### A.3 Chop diagnosis (from code + live Play freeze-frame)
Ran it in Play, froze the strike keyframe (t=0.34), measured the axe orientation and screenshotted from the side.

- **PRIMARY — flat/horizontal arc, NOT a downward chop. ✅ applies.** At the strike, the axe head (handle `+X`, headward) points world **(−0.93, +0.25, −0.28)** = the character's **LEFT and slightly UP**, not down/forward. **Root cause:** the idle grip holds the axe roughly **horizontal across the chest** (Grip_Cant −165, head at the left hand). The swing only rotates the **arms**; `Tool.Grip` (axe orientation relative to the right hand) is **fixed** for the whole swing, so the blade never rotates to lead edge-down. The "chop" sweeps a broadside axe sideways rather than describing a vertical overhead→ground arc.
- **Wind-up — PRESENT (does NOT apply).** t=0.14 raises both arms overhead (LUp 140 / RUp 160) with torso lean-back −15 + twist +20.
- **Static torso — does NOT apply.** UpperTorso animates (−15/+20 windup → +25/−20 strike).
- **No follow-through — PARTIALLY applies.** The strike ends at RUp 30 / LUp 40 (mid-height, still forward-up) and then **hard-cuts back to idle at 0.50** — no low finish past impact, no ease-out. Reads as abrupt.
- **Wrong priority / idle fights swing — does NOT apply.** Stop-HoldPose → SwingPose(Action4) → resume-on-Stopped is correct; verified clean.
- **Timing — borderline.** 0.5s total; down-strike 0.14→0.34 (0.20s), snap-back 0.34→0.50 (0.16s) with **no easing**, which adds to the unfinished feel.
- **Grip/hand positions shifting wrongly — this is the ROOT.** The fixed cant grip keeps the blade broadside through the arc.
- **Same issue on DaneAxe** (identical frame shape + horizontal idle grip; Grip_Cant −75, Grip_LiftY 1.0).

**Fix direction (for your spec — NOT changed here):** the swing needs to **re-orient `Tool.Grip` during the strike** (rotate the axe from broadside to blade-down/edge-leading), OR restructure the two-handed hold so the arc naturally carries the head down; plus add a **low follow-through** frame and an **eased recovery**.

---

## PART B — current combat / damage model

### B.1 Weapons.luau — the 6 equippable Viking Tier-1 tools
| Weapon | Damage | StructuralDamage | Class | Range | Cooldown | CanHarvest | UnlockLvl | Cost |
|---|---|---|---|---|---|---|---|---|
| Seax | 12 | 6 | Pierce | 4 | 0.7 | no | 1 | 30 |
| BeardedAxe | 16 | 28 | Axe | 5 | 1.3 | **yes** | 1 | 40 |
| DaneAxe | 30 | 22 | Slash | 7 | 1.9 | no | 5 | 200 |
| Spear | 18 | 8 | Pierce | 9 | 1.3 | no | 3 | 90 |
| RoundShield | 6 | 4 | Blunt | 4 | 1.0 | no | 1 | 35 |
| HuntingBow | 14 | 4 | Pierce | 40 | 1.5 | no | 2 | 60 |

- `Damage` = applied vs characters/NPCs/players. `StructuralDamage` = applied vs blocks/structures (then × material resist). **Trees use `Damage`, not `StructuralDamage`.**
- The registry also holds non-Viking weapons (Club, WarHammer, Mace, Dagger, Crossbow, Harpoon, WoodSword, IronSword, Scimitar, Greatsword, WoodAxe, IronAxe, SteelAxe, WandOfSparks, StaffOfFrost, GrimoireOfFlames) — not equippable as tools. Say the word and I'll dump their fields too.
- **No per-target fields exist today** — one `Damage` and one `StructuralDamage` per weapon; target type is decided by CombatService routing (below), not by the weapon.

### B.2 CombatService — damage application & routing (`Net.Function("Attack")`)
Equipped weapon is read **server-side** (never trusts client). Anti-cheat gate: owned; `level ≥ UnlockLevel`; `dist ≤ Range + ATTACK_REACH_PADDING(8)`; `os.clock()-lastAttack ≥ Cooldown×0.95`. Then routes by CollectionService tag:

| Target (tag) | Damage used | Resist? | Delegate |
|---|---|---|---|
| `HybridBlock` | `StructuralDamage` | ✅ `ComputeStructuralDamage(block.MaterialId, …, Class)` | `BuildingService.DamageBlock` |
| `HarvestableTree` | `Damage` (only if `CanHarvest`) | no | `ResourceService.HarvestTree(player, tree, weaponId)` |
| `PlacedStructure` | `StructuralDamage` | ✅ **always as material "Stone"** | `StructureService.DamageStructure` |
| Model+Humanoid, **NPC** | `Damage` | **no** | `NPCService.DamageNPC(model, Damage)` |
| Model+Humanoid, **player** | `Damage` | no | `humanoid:TakeDamage(Damage)` (PvP) |

- Single-pass resist rule holds (CombatService resists once; DamageBlock/DamageStructure apply the pre-adjusted number — Handshake A/G).
- **No crit server-side.** `CombatController` has an `isCrit` param for damage-number VFX only; CombatService applies flat damage.
- **PvP has no team check** — the code comment says "only if not same team" but no team gate is implemented; any player hit takes `Damage`.

### B.3 Structure HP + resistance
`StructureRegistry.MaxHealth`: TownHall **2000**, Wall **400**, WallGate **300**, ArcherTower **600**, Cannon **800**, Catapult **700**, ArcherLodge **500**, WizardTower **750**, Ballista **650**.
Structures are resisted as material **"Stone"** → Stone `ResistTable`: **Blunt 1.2, Pierce 0.7, Slash 0.5, Magic 0.9, Axe 0.6**. So vs structures, **Blunt is best** (×1.2), Slash/Axe worst. `DamageStructure` applies the already-resisted number directly (no second resist). Structures do **not** use StabilitySolver (that's blocks only).

### B.3b Blocks (HybridBlock) — MaterialMatrix + StabilitySolver
Material HP / resist multipliers (`< 1` = resistant, `> 1` = vulnerable):
| Material | HP | Blunt | Pierce | Slash | Magic | Axe | HarvestYield |
|---|---|---|---|---|---|---|---|
| Wood | 100 | 0.8 | 0.9 | 1.2 | 1.1 | **2.0** | 3 |
| Stone | 300 | 1.2 | 0.7 | 0.5 | 0.9 | 0.6 | 2 |
| Iron | 600 | 0.9 | 0.6 | 0.4 | 1.0 | 0.5 | 1 |
| Obsidian | 1200 | 0.6 | 0.5 | 0.3 | 1.4 | 0.4 | — |
| Sandstone | 180 | 1.1 | 0.8 | 0.7 | 0.9 | 0.8 | 2 |
| Brick | 400 | 1.3 | 0.7 | 0.5 | 0.8 | 0.6 | — |

`ComputeStructuralDamage(matId, rawDamage, class) = rawDamage × ResistTable[class]` (unknown mat/class → ×1). `StabilitySolver.resolveCollapse` flood-fills support on a **4-stud grid** (GRID_STEP=4); any surviving block that can't reach ground (Y≤0) collapses.

### B.4 Tree / resource harvest (`ResourceService`)
- **Tree HP = 100** (set at spawn in WorldGenerator; `WoodMaterialId` per biome). Per hit reduces HP by **`weapon.Damage`** (only CanHarvest weapons — among the 6, **only BeardedAxe**).
- On fell (HP≤0): award wood × **HarvestYield** (`Wood.HarvestYield = 3`; code fallback 5 if no record) + **10 XP** (`HARVEST_XP`); remove tree; **respawn after 300s** (`TREE_RESPAWN_SEC`); fire `TreeChopped` + `MaterialAwarded`.
- ⚠️ **QUIRK TO RESOLVE:** there's a **per-player, per-tree cooldown = 300s**. Each *surviving* hit stamps `harvestedTrees[treeId]=now`, and any hit within `TREE_RESPAWN_SEC` after is rejected ("already harvested recently"). With tree HP 100 and BeardedAxe Damage 16 (~7 hits to fell) but only **1 hit allowed per 300s**, trees are effectively **un-fellable** in normal play. This directly conflicts with a hit-to-fell model — please resolve when designing the damage matrix.

### B.5 NPC HP model (`NPCService` definitions)
| NPC | MaxHealth | Damage | Speed | AggroRange | XPReward | GoldReward |
|---|---|---|---|---|---|---|
| Goblin | 60 | 8 | 14 | 24 | 15 | 5 |
| Troll | 200 | 22 | 8 | 18 | 45 | 18 |
| Stone Brute | 350 | 35 | 6 | 16 | 80 | 30 |
| Forest Sprite | 80 | 12 | 16 | 28 | 20 | 8 |
| River Wraith | 120 | 18 | 12 | 22 | 30 | 12 |

Player→NPC damage = **raw `weapon.Damage`, no resist/armor**. Kill → `AwardGold(GoldReward)` + `AwardXP(XPReward)` (once — Handshake C).

### B.6 GameConfig combat constants
GameConfig holds **no damage/crit constants** (all damage lives in Weapons.luau). Relevant to combat/harvest: `HARVEST_RANGE=10`, `TREE_RESPAWN_SEC=300`, `STRUCTURE_SNAP_GRID=4`, `MAX_STRUCTURES_PER_PLOT=30`. In CombatService: `ATTACK_REACH_PADDING=8`, cooldown tolerance `×0.95`.

---

## PART C — doc fix
`DIRECTOR_STATUS.md` §5 item 1 updated from a pending "**Reconnect the Rojo plugin once**" step to **"DONE & RECONCILED (2026-07-05)"** — notes the plugin was reconnected, Studio ⇄ disk parity confirmed clean (consolidation-01), nothing wiped, no action pending. Now mirrors the DONE/CLOSED style of §5 item 4.

---

## VERIFY (evidence)
- Animation code pasted verbatim from live `StarterPack.BeardedAxe.HoldServer` and `ReplicatedStorage.Shared.TwoHandHold` (execute_luau, Edit datamodel).
- Chop diagnosis: live Play, froze `SwingPose` at t=0.34; measured handle `+X` (headward) = **(−0.93, 0.25, −0.28)** (world) = character-left/up; side screenshot confirms a flat sideways sweep, not a downward chop.
- All Part B values read from source on disk: `Weapons.luau`, `CombatService.luau` (routing + `ATTACK_REACH_PADDING=8`), `MaterialMatrix.luau` (records + `ComputeStructuralDamage`), `StructureRegistry.luau` (MaxHealth), `ResourceService.luau` (HARVEST_XP=10, yield, cooldown), `WorldGenerator.luau` (tree MaxHealth=100), `NPCService.luau` (NPC defs), `GameConfig.luau`.
- Guardrail honored: discovery only; `rojo serve`/live only, **no `rojo build`**. No animations/weapons/combat values changed. No secrets in this file.

## BACK TO DIRECTOR
1. **The other five attacks already exist.** Earlier this session I authored attack animations for **all six** weapons (BeardedAxe + DaneAxe chops, Spear thrust, Seax slash, RoundShield raise-to-block, HuntingBow left-hand draw), all baked into their StarterPack HoldServers. So "author the other four" is largely already done — your spec may want to *reconcile with / refine* the existing ones rather than start fresh. I can dump the current state of all 6 (frames + approach) if useful.
2. **Axe chop root cause = the fixed horizontal grip cant.** A believable chop will likely need the swing to **re-orient `Tool.Grip` mid-strike** (blade rotates broadside→edge-down) plus a low follow-through + eased recovery. Flag if you want me to prototype that when the fix is specced.
3. **Tree harvest is currently un-fellable** (Part B.4: 300s per-tree cooldown vs ~7 hits needed). Needs a design decision before the damage matrix.
4. **No crit, no NPC/PvP resist, no PvP team check** exist today. If the damage matrix wants per-target multipliers, crit, armor, or friendly-fire rules, those are all new work.
