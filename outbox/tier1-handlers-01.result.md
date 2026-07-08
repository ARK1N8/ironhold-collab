# RESULT: tier1-handlers-01 — implement PlaceBlock + HarvestTree server handlers

FROM: Claude Code (local executor)
DATE: 2026-07-07
STATUS: **DONE** — both remotes now respond and work end-to-end; the multi-hit tree bug is fixed; no
regression; no duplicated logic. Rojo synced everything to live (Host 127.0.0.1 holding).

---

## What I found (extend, not duplicate)
Both handler **functions already existed and were complete** — they were simply **never bound** to their
remotes, which is exactly why the client calls hung:
- `BuildingService.PlaceBlock(player, materialId, position, plotId)` — full validation (plot ownership,
  material exists, 4-stud grid snap, plot bounds, 500-block cap, occupancy, spends 1 material, spawns a
  tagged `HybridBlock`, awards XP). Signature already matches the client's call
  `PlaceBlock(selectedMaterial, snapped, currentPlotId)`.
- `ResourceService.HarvestTree(player, treePart, weaponId)` — full hit-to-fell model already present
  (tree HP tracked server-side in `treeHealth[treeId]`, per-hit damage from `Weapons.Get(weaponId).Damage`,
  yields wood + XP on fell, starts respawn).

So the fix was **3 small edits, zero new systems, zero duplicate Net bindings** (confirmed `PlaceBlock`
and `HarvestTree` were bound nowhere before). I mirrored `StructureService`'s existing
`Net.Function("PlaceStructure").OnServerInvoke` pattern.

## Changes (canonical src, Rojo-synced to live)
1. **`ResourceService.luau` — bug fix (the real one).** The per-player harvest cooldown was being set on
   **surviving** hits (old line ~219, `data.harvestedTrees[treeId] = os.time()` inside the `newHP > 0`
   branch). Step 5 then rejected every subsequent hit (`"already harvested recently"`), so a multi-hit
   tree could never fall. **Removed that line** — the cooldown is now set only once the tree is **felled**
   (step 11), which is where it belongs. This is the "move the cooldown to post-fell" fix.
2. **`ResourceService.luau` — bind HarvestTree.** Added `Net.Function("HarvestTree").OnServerInvoke`.
   The client sends only the tree part; the handler resolves the weapon **server-side** from
   `data.equippedWeapon` (never trusts the client for weapon/damage), type-guards the target, then calls
   the existing `ResourceService.HarvestTree`.
3. **`BuildingService.luau` — bind PlaceBlock.** Added `Net.Function("PlaceBlock").OnServerInvoke`,
   type-guarding (materialId string / position Vector3 / plotId string) so malformed input returns a
   clean error instead of erroring in the grid math, then calls the existing `BuildingService.PlaceBlock`.

## Part 1 — HarvestTree (hit-to-fell) — VERIFIED
Server-side functional test against a real `HarvestableTree` (100 HP) with an equipped **BeardedAxe**
(Damage 16, `CanHarvest = true`):
```
Tree tree_1_-6_17  MaxHealth=100  BeardedAxe.Damage=16  woodMat=Wood
  hit 1: ok=true hp=84
  hit 2: ok=true hp=68     <-- pre-fix this hit was REJECTED (the bug); now succeeds
  hit 3: ok=true hp=52
  hit 4: ok=true hp=36
  hit 5: ok=true hp=20
  hit 6: ok=true hp=4
  hit 7: ok=true hp=0  <-- TREE FELLED
wood: before=0 after=3  (+3 Wood awarded)
```
- Hit-to-fell: ✅ tree HP is server-authoritative, drops 16/hit, fells on the **7th** hit (100/16 → 7).
- Cooldown moved to fell/respawn: ✅ surviving hits 2–6 no longer rejected (the reported bug is gone).
- Yields wood: ✅ +3 Wood on fell (from MaterialMatrix `Wood.HarvestYield`).
- Tree HP source: `treeHealth[treeId]` (server); per-hit damage source: `Weapons.Get(weaponId).Damage`
  (no invented numbers). Bound in **ResourceService**; the client's existing CombatController interact
  path (`HarvestTree:InvokeServer(hitPart)`) now reaches it.

## Part 2 — PlaceBlock — VERIFIED
Server-side functional test (claimed a plot, granted Wood, placed a block):
```
ClaimPlot: ok=true   plot origin=(107,11,-273)
PlaceBlock: ok=true
HybridBlock count: 0 -> 1        (a block spawned in-world)
wood: 8 -> 7  (spent 1)          (material charged)
re-place same spot: ok=false err=Position already occupied   (validation intact)
```
- Returns instead of hanging: ✅  Block places in-world: ✅  Charges material: ✅  Occupancy/validation
  intact: ✅. Wired into the existing **BuildingService** placement path (the one the client's build flow
  already targets); mirrors PlaceStructure's validation, not reinvented.

## Part 3 — no hangs / no regression — VERIFIED
Fresh-Play client probe of all 12 RemoteFunctions (bound = responds fast; unbound = hang):
```
GetInventory ✅  Attack ✅  ClaimPlot ✅  PlaceStructure ✅  FireStructure ✅  UpgradeTownHall ✅
PlaceBlock ✅ (NEW)   HarvestTree ✅ (NEW)
EquipWeapon ⏸ hang   PurchaseWeapon ⏸ hang   PurchaseStructure ⏸ hang   CraftStructure ⏸ hang
RESPONDS 8/12
```
- The 2 target remotes now respond; the original 6 still respond (**no regression** from touching
  ResourceService/BuildingService).
- The 4 deferred remotes (EquipWeapon/PurchaseWeapon/PurchaseStructure/CraftStructure) **intentionally
  left unbound** per the task — they belong with progression/economy (§12).

## Persistence / no new drift
All three edits are in canonical `src` (`Server/Services/ResourceService.luau`,
`Server/Services/BuildingService.luau`) and Rojo pushed them to live instantly (verified the live sources
contain the bindings + the fix before testing). No duplicate Net bindings, no parallel harvest/placement
system, no new place-only instances. The Studio place can be saved to bake the synced state, but it's not
required — src is canonical and Rojo re-syncs on connect.

## STATUS: DONE

## BACK TO DIRECTOR
1. **Tree HP is currently a hardcoded 100 in `WorldGenerator` (per-tree attribute)** and BeardedAxe does
   16, giving the intended 7-hit fell. If you later want per-biome tree toughness or different axe tiers,
   that's a data change (tree `MaxHealth` attribute + Weapons `Damage`), not a code change.
2. **Only BeardedAxe has `CanHarvest = true`** among the current Viking weapons (DaneAxe/Spear/Seax/Bow do
   not). So only the BeardedAxe fells trees today — flag if other weapons should harvest.
3. The **4 deferred remotes** remain unimplemented by design; when the progression/economy tier ladders &
   costs are specced (§12), those handlers can be added the same way (functions may already exist in
   EconomyService/ProgressionService — worth checking for the same "function exists but unbound" pattern
   before writing new logic).
