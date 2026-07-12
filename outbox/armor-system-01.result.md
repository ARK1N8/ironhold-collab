# armor-system-01 — Result

**STATUS: DONE** — verified in Play with real numbers. Armor is stats-only; visuals deferred to armor-visuals-01 (spike below). All 12 original remotes now respond (0 hangs).

Armor system (3 slots × 3 tiers, 9 items) as data, feeding the EXISTING combined damage-reduction clamp — no second clamp, no parallel path. The last 2 dead remotes (`EquipWeapon`, `PurchaseWeapon`) are bound and functional, plus dedicated armor remotes. Armor tab shows real data with equip/unequip. Visuals spike is investigation-only.

---

## PART 1 — Armor data + service

### The 9-item registry (`src/Shared/ArmorRegistry.luau`)
Pure data, mirrors `Weapons.luau`. Costs + stats are DATA, not hardcoded in service logic. Per-piece reduction locked with Director: Leather 3% / Iron 6% / Steel 10%.

| Id | DisplayName | Slot | Tier | DamageReduction | Cost (gold) | UnlockLevel | AssetId |
|---|---|---|---|---|---|---|---|
| LeatherHelmet | Leather Helmet | Helmet | Leather | 0.03 | 40 | 1 | "" |
| IronHelmet | Iron Helmet | Helmet | Iron | 0.06 | 120 | 4 | "" |
| SteelHelmet | Steel Helmet | Helmet | Steel | 0.10 | 300 | 8 | "" |
| LeatherChest | Leather Chestpiece | Chest | Leather | 0.03 | 60 | 1 | "" |
| IronChest | Iron Chestplate | Chest | Iron | 0.06 | 180 | 4 | "" |
| SteelChest | Steel Chestplate | Chest | Steel | 0.10 | 450 | 8 | "" |
| LeatherLegs | Leather Greaves | Legs | Leather | 0.03 | 50 | 1 | "" |
| IronLegs | Iron Greaves | Legs | Iron | 0.06 | 150 | 4 | "" |
| SteelLegs | Steel Greaves | Legs | Steel | 0.10 | 375 | 8 | "" |

- **Tiered, not upgraded**: no upgrade path exists in the registry (deliberate contrast with `StructureRegistry.UPGRADES`). Better armor is acquired.
- **`AssetId = ""`** placeholder on every item so armor-visuals-01 fills it in with no schema rework.
- Full Steel set = 0.30. Helpers: `Get`, `All`, `BySlot`, `IsSlot`, `SLOTS = {Helmet,Chest,Legs}`.

### Where owned/equipped state lives
Extended PlayerData in `DataManager.buildDefaultData` (Ground Rule 7 — find-or-create; `deserializeData` already merges new top-level default keys into legacy profiles, so existing players get them empty automatically):
- `ownedArmor = {}` → `{ [armorId] = true }`
- `equippedArmor = {}` → `{ [slot] = armorId }` (absent key = empty slot)

**EconomyService remains the sole writer** of `ownedArmor`/`equippedArmor` (consistent with it owning `ownedWeapons`/`equippedWeapon`). It gained `PurchaseArmor`, `EquipArmor`, `UnequipArmor`.

**`ArmorService`** (new) does two things: (1) read-only `GetArmorReduction(player)` = sum of equipped pieces' `DamageReduction`; (2) binds the whole acquire-and-equip remote surface (delegating all writes to EconomyService). *Why a new service:* the reduction read mirrors `StatService.GetDefenseReduction` (a clean read consumed by CombatService), and grouping the acquire/equip remotes in one cohesive place keeps CombatService and EconomyService focused.

### PROOF: armor + Defense + shield compose through ONE clamp
The clamp lives in `CombatService.applyPlayerDamage`. I added exactly one line — `reduction += ArmorService.GetArmorReduction(plr)` — into the existing sum, **before** the single `math.min(reduction, StatService.COMBINED_REDUCTION_CAP)` (cap = 0.60). Real numbers from Play (`CombatService.ApplyPlayerDamage` on a live Humanoid, MaxHealth normalized):

| Setup | Reductions | Clamped | 100-dmg hit lands |
|---|---|---|---|
| Baseline (none) | 0 | 0.00 | **100.0** |
| Full Steel set | armor 0.30 | 0.30 | **70.0** |
| **Full tank**: shield + Defense 40pts + full Steel | 0.25 + 0.40 + 0.30 = **0.95** | **0.60** (capped) | **40.0** |
| 2-piece Steel (helmet unequipped) | armor 0.20 | 0.20 | **80.0** |

The full-tank build hits the 0.60 ceiling exactly as intended — a 100-dmg hit lands as 40. Shield (25%) + Defense stat (40%) + armor (30%) all stack into the same sum and the same clamp.

---

## PART 2 — The last 2 dead remotes + armor remotes

**Latent bug fixed while binding:** `EconomyService.PurchaseWeapon` checked `def.GoldCost`, but `WeaponDefinition`s carry the price in `Cost`. It would have failed 100% of the time ("has no GoldCost"). Fixed to read `def.Cost`, treat `Cost == 0` as a free starter weapon (granted, not charged), and `EquipWeapon("Fists")` is now always valid (the unarmed/unequip state).

**Final remote surface** (all bound in `ArmorService`, writes via EconomyService, each fires `InventoryChanged` on success):
- `PurchaseWeapon(weaponId)` → boolean
- `EquipWeapon(weaponId)` → boolean
- `PurchaseArmor(armorId)` → (ok, reason?)
- `EquipArmor(armorId)` → (ok, reason?)
- `UnequipArmor(slot)` → (ok, reason?)

**All 12 original remotes respond (0 hangs)** — probed from the client:
`Attack, PlaceBlock, PlaceStructure, FireStructure, UpgradeTownHall, EquipWeapon, PurchaseWeapon, PurchaseStructure, CraftStructure, ClaimPlot, HarvestTree, GetInventory` → all RESPONDS. Plus the 5 newer remotes (`UpgradeStructure, GetStructureInfo, PurchaseArmor, EquipArmor, UnequipArmor`) → all RESPONDS. **TOTAL HANGS: 0.**

Functional proof (real service/remote paths):
- `PurchaseWeapon("WarHammer")` → owned=true, gold −150 (bug fix confirmed). Insufficient gold → rejected.
- `EquipWeapon("WarHammer")` → equippedWeapon="WarHammer"; `EquipWeapon("Fists")` → back to "Fists".
- `PurchaseArmor`/`EquipArmor`/`UnequipArmor` via live client remotes: buy → owned; equip → equipped; duplicate buy → "Already owned."; unequip → slot cleared.

---

## PART 3 — Armor tab UI

`src/Client/Panels/InventoryPanel.luau` — the Armor tab now renders **real owned-armor data** (was "No armor yet."). Verified in Play: all 9 items render as rows, each `"Slot · Tier · -N%"` with an equipped/owned marker and a context button (≥44px):
- Not owned → **"Buy Ng"** → `PurchaseArmor`
- Owned, not equipped → **"Equip"** → `EquipArmor`
- Equipped → **"Unequip"** → `UnequipArmor`

Pure view over the `GetInventory` snapshot (new `armor` bucket in `InventoryService`). All remote calls are async (`task.spawn`, no blocking InvokeServer in the render path), re-render from confirmed server state, and Toast on success/failure. Rows sort by slot then tier. Server-authoritative throughout (client never writes). **Kit/Theme only** — the row is a transparent layout Frame; every visual is `Kit.Label` + `Kit.Button`; zero inline colors. Confirmed ScrollList renders exactly 9 rows (no dupes) and reflows.

No new keybind added — the armor tab lives inside the existing Inventory panel (I key), already in the KeybindRegistry.

---

## PART 4 — Visuals spike (investigation only; generated/uploaded/attached NOTHING)

Inspected the live character rig in Play:
- **RigType R15**, 17 MeshParts, **0 Bones on limbs**, **0 Motor6D** (joints are constraint/runtime-driven — consistent with the KeyframeSequence weapon-hold approach noted in prior tasks), 1 Weld.
- **15 WrapTargets present** (layered-clothing cages DO exist on the body parts), **0 WrapLayer** (no clothing currently applied).
- Attachment points available: `Head.HatAttachment` (+ Hair/FaceCenter), `UpperTorso` BodyFront/BodyBack + collar/shoulder rig attachments, `LowerTorso` waist attachments, `UpperArm` shoulder attachments, leg segments with Knee/Hip rig attachments.
- **Precedent:** `ShieldHandle` is a custom MeshPart attached via a plain **Weld** (`LeftLowerArm ↔ ShieldHandle`) — the proven Meshy → Open Cloud → rigid-weld pipeline.

**Recommendation for armor-visuals-01: RIGID welded meshes (option a).**
- Helmet → weld to `Head` (HatAttachment mount). Chest → weld to `UpperTorso`. Legs → the "Legs" gameplay slot spans 4 rigid MeshParts (Upper/Lower L/R); rigid greaves weld to each segment (thigh + shin guards) and follow perfectly because each segment is rigid. Optional pauldrons → UpperArms.
- Matches the existing rigid-weld pipeline exactly; no cage authoring; predictable.

**Layered clothing (option b) is technically possible** (15 WrapTargets exist), but only worth it for cloth that must deform at the knee/elbow — and it hinges on whether Meshy can emit properly caged (WrapLayer) assets, which is **unproven** in this pipeline. For plate armor, rigid wins.

**What armor-visuals-01 will need:** (1) fill the 9 `AssetId` fields (schema already ready); (2) re-attach welds on `CharacterAdded`/respawn (same pattern as weapon holds + shield); (3) decide the Legs sub-piece split (per-segment rigid guards). No hard blockers.

---

## Regression
Play-tested clean: **0 non-benign errors** on boot (the "module error"/asset-134278420263483 chain is pre-existing and benign). GetStats responds; `GetInventory` returns all 5 buckets (weapons/armor/defense/buildings/materials); CombatHUD intact; GetStructureInfo (structure upgrades) responds; sprint/StaminaService and character sheet untouched. All temp test scripts removed (runtime-only, never on disk).

## Files
- **New:** `src/Shared/ArmorRegistry.luau`, `src/Server/Services/ArmorService.luau`
- **Modified:** `src/Server/Services/DataManager.luau` (ownedArmor/equippedArmor defaults), `src/Server/Services/EconomyService.luau` (armor writers + PurchaseWeapon `Cost` fix + Fists), `src/Server/Services/CombatService.luau` (armor into the one clamp), `src/Server/Services/InventoryService.luau` (armor bucket), `src/Server/ServerBootstrap.server.luau` (load ArmorService), `src/Shared/Net.luau` (3 armor remotes), `src/Client/Panels/InventoryPanel.luau` (Armor tab)

## Notes / guardrails honored
- ONE clamp (no second clamp, no parallel path). Armor tiered/acquired (no upgrade path). No movement/stamina penalties. Costs + stats as DATA. Ground Rule 7 (extend/find-or-create). Async rule (no blocking InvokeServer in render paths). Part 4 investigation-only. No secrets in this doc.
- Left in place (out of scope): `ClaimPlot` lacks nil-arg validation (errors if invoked without a position — surfaced only by my probe; not armor-related).
