# structure-upgrades-01 — Result

**Status:** ✅ Complete — verified in Play with real numbers.

Instant (no timers / no queue / no offline progression) 5-level upgrade system for all 8 non-TownHall structures, gated by Town Hall level. All costs and per-level benefits live as **data in `StructureRegistry`**, not as logic in the service. The two previously-dead remotes (`PurchaseStructure`, `CraftStructure`) are now bound and functional. Upgrade UI is a pure, async Kit panel.

---

## STEP 0 — Registry read (design driven by REAL fields, nothing invented)

The 9 canonical structures and the base fields the curves were derived from:

| Structure | Cat | Base HP | AutoAtk Dmg / Range | PlayerActivated Dmg | Base cost |
|---|---|---|---|---|---|
| TownHall | TownHall | 2000 | — | — | Wood 200 + Stone 100 (gate; not upgradable here) |
| Wall | Building | 400 | — | — | Stone (Craft) |
| WallGate | Building | 300 | — | — | Stone (Craft) |
| ArcherTower | Defense | 600 | 15 / 30 | — | Gold 150 |
| Cannon | Defense | 800 | 40 | 60 | Gold 300 |
| Catapult | Defense | 700 | 30 | 80 | Wood 50 + Iron 20 |
| ArcherLodge | Defense | 500 | 20 | — | Wood 80 + Stone 30 |
| WizardTower | Defense | 750 | 25 | 50 | Gold 500 |
| Ballista | Defense | 650 | 35 | — | Wood 60 + Iron 40 |

**Curve design (applied to the real fields above):**
- **HP:** ×1.5 per level → +50% HP each level, as specified. (L1→L5 = ×5.06 total.)
- **AutoAttack / PlayerActivated damage:** ×1.5 per level (role-specific — only structures that have the field get it scaled).
- **AutoAttack range:** ×~1.15 per level (towers only; keeps range growth gentle so it never trivializes the map).
- **Cost:** geometric ×1.6 per level, paid in the structure's **native resource** (Gold for gold-bought defenses, the Craft materials for crafted ones). Rounded to whole units.

TownHall is intentionally **not** in the upgrade table — it keeps its own path via the existing `TownHallService` / `UpgradeTownHall`, and it is the gate every other upgrade checks against (you can never upgrade a structure above your TownHall level).

### Full per-level upgrade table (exact registry values)

**HP-only (Building):**

| Structure | L1→L2 cost / HP | L2→L3 | L3→L4 | L4→L5 |
|---|---|---|---|---|
| Wall | Stone 16 / 600 | Stone 26 / 900 | Stone 41 / 1350 | Stone 66 / 2025 |
| WallGate | Stone 24 / 450 | Stone 38 / 675 | Stone 61 / 1013 | Stone 98 / 1519 |

**Ranged tower (HP + Dmg + Range):**

| ArcherTower | Cost / HP / Dmg / Range |
|---|---|
| L2 | Gold 240 / 900 / 23 / 35 |
| L3 | Gold 384 / 1350 / 34 / 40 |
| L4 | Gold 614 / 2025 / 51 / 45 |
| L5 | Gold 983 / 3038 / 76 / 53 |

**Auto + PlayerActivated damage:**

| Structure | L2 | L3 | L4 | L5 |
|---|---|---|---|---|
| Cannon | Gold 480 / HP1200 / A60 / PA90 | Gold 768 / 1800 / 90 / 135 | Gold 1229 / 2700 / 135 / 203 | Gold 1966 / 4050 / 203 / 304 |
| Catapult | W80+I32 / 1050 / A45 / PA120 | W128+I51 / 1575 / 68 / 180 | W205+I82 / 2363 / 101 / 270 | W328+I131 / 3544 / 152 / 405 |
| WizardTower | Gold 800 / 1125 / A38 / PA75 | Gold 1280 / 1688 / 56 / 113 | Gold 2048 / 2531 / 84 / 169 | Gold 3277 / 3797 / 127 / 253 |

**Auto damage only:**

| Structure | L2 | L3 | L4 | L5 |
|---|---|---|---|---|
| ArcherLodge | W128+S48 / HP750 / A30 | W205+S77 / 1125 / 45 | W328+S123 / 1688 / 68 | W524+S197 / 2531 / 101 |
| Ballista | W96+I64 / 975 / A53 | W154+I102 / 1463 / 79 | W246+I164 / 2194 / 118 | W393+I262 / 3291 / 177 |

---

## Implementation

**`src/Shared/StructureRegistry.luau`** (data + helpers — the single source of truth for the curves)
- `MAX_LEVEL = 5`; `UPGRADES` table (levels 2–5 for the 8 non-TownHall structures: absolute stat values + `Cost`).
- `StructureRegistry.MaxLevel(id)` → 5 if it has an upgrade path, else 1.
- `StructureRegistry.GetUpgradeCost(id, targetLevel)` → `{ resource = amount }` or nil.
- `StructureRegistry.GetLevelStats(id, level)` → `{ MaxHealth, AutoAttackDamage?, AutoAttackRange?, PlayerActivatedDamage? }`; level 1 returns the base def. Service logic reads stats **only** through this — no numbers are hardcoded in the service.

**`src/Server/Services/StructureService.luau`** (server-authoritative)
- Placed-structure records now carry `level = 1`; `Level` set as a Model attribute.
- `applyLevelStats(record)` — reads `GetLevelStats`, updates record `currentHealth` (an upgrade fully repairs) + Model `MaxHealth`/`CurrentHealth`/`Level` attributes.
- **Auto-attack loop** reads `GetLevelStats(id, record.level)` each tick for damage + range → upgrades actually change combat.
- **`FireStructure`** (player-activated) reads level-scaled `PlayerActivatedDamage`.
- **`UpgradeStructure(player, instanceId)`** — validates: record exists & active, not TownHall, plot owned by caller, not at max level, **target level ≤ TownHall level**, sufficient resources; spends via canonical `EconomyService.SpendGold`/`SpendMaterial`; bumps `record.level`, `applyLevelStats`, persists `data.placedStructures`, fires `StructurePlaced` + `InventoryChanged`; returns clean failure reasons.
- **`GetStructureInfo(player, instanceId)`** — read model for the panel: current level/maxLevel/stats/health + `nextLevel`, `nextStats`, `cost`, `blocked`, `reason`.
- Bound 4 remotes: `UpgradeStructure`, `GetStructureInfo`, `PurchaseStructure` → `EconomyService.PurchaseStructure`, `CraftStructure` → `EconomyService.CraftStructure` (last two also fire `InventoryChanged`).

**`src/Shared/Net.luau`** — added `UpgradeStructure`, `GetStructureInfo` functions (find-or-create, no destroy-recreate).

**`src/Client/Panels/StructureUpgradePanel.luau`** (NEW) — Kit.Panel modal, pure view over `GetStructureInfo`. Async open (`task.spawn`, never a blocking InvokeServer in the render path). Shows level, per-stat `cur → next`, cost, and an Upgrade button that disables with the server's reason when blocked. Re-renders from confirmed server state after an upgrade. Zero inline styling (Theme tokens only).

**`src/Client/StructureUpgradeInit.client.luau`** (NEW) + **`KeybindRegistry.luau`** — press **U** while aiming at a placed structure (camera raycast, not left-click, to avoid the attack input) to open its panel. Registered as a wired keybind so the Controls menu can't drift.

---

## Play verification (real numbers, driven through real remotes)

Clean boot, **0 non-benign errors**. All four remotes respond (Purchase/Craft no longer hang).

| Check | Result |
|---|---|
| Place Wall → `GetStructureInfo` | L1/5, HP 400, next cost 16 Stone → HP 600 |
| **Upgrade L1→L2** | ok — Stone 500→**484** (−16), record HP + live Model MaxHealth = **600** |
| **Upgrade L2→L3** (via panel's own remote path) | ok — server L3/5 HP **900**; panel re-rendered "Level 3/5, Health 900→1350, Upgrade to Level 4" |
| **TownHall gate** (TH=2, request L3) | blocked — *"Requires Town Hall level 3."* |
| **Insufficient resources** (Stone drained, need 26) | blocked — *"Not enough Stone (26 needed)."* |
| Panel live render | Title "Stone Wall", Level/stat/cost rows all from server; button correctly enabled/disabled with reason |
| **PurchaseStructure** (ArcherTower) | ok — ownedStructures 0→1, gold −150 |
| **CraftStructure** (Catapult) | ok — ownedStructures 0→1, Wood −50, Iron −20 |
| Regression | GetInventory / GetStats respond, CombatHUD intact, StructureUpgradeInit loaded |

Upgrades are **instant** — no timers, no build queue, no offline progression. Every level/cost/resource check is **server-authoritative**; the client never supplies level or cost. Costs and benefits are pure **data** in `StructureRegistry`.

---

## Notes / follow-ups
- `GetStructureInfo`/`UpgradeStructure` identify a structure by its Model `InstanceId` attribute. Selection is currently **U + camera-aim only**; touch/gamepad structure-select is not yet wired (noted in pending follow-ups) — the Controls entry honestly marks touch as "—".
- No secrets in this doc.
