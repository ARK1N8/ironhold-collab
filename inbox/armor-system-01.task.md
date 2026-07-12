TASK: armor-system-01 — armor system (3 slots x 3 tiers, stats-only) + last 2 dead remotes + visuals spike

FROM: Studio Director
DATE: 2026-07-09
SCOPE: GAME project. Builds the §12.3 ARMOR system, stats-only. Visible armor is a SEPARATE follow-up
       (armor-visuals-01) — but design the data so visuals drop in with no rework.
DEPENDS ON: character-stats-01 (Defense stat + the COMBINED damage-reduction clamp), shield-offhand-01
       (shield's 25% reduction — armor stacks into the same clamp), inventory-panel-01 (the Armor tab
       currently renders an honest empty state — this task gives it real data), framework-core-01 (Kit/
       Theme/layer manager), contract v2.2.0 §12.3 + Ground Rule 7 + the async rule.
ENV: sync stable on Host 127.0.0.1; rojo serve running on 127.0.0.1:34872 — do NOT start a 2nd instance.
       rojo serve + Connect only, NEVER rojo build.

## Design (locked with Director)
- **3 slots:** Helmet, Chest, Legs. (The shield is already a separate off-hand system — do NOT fold it in.)
- **3 tiers:** Leather -> Iron -> Steel. Mirrors the weapon tier ladder (§12.3 TIER model — armor is
  ACQUIRED as better tiers, NOT upgraded in place. Do not build an upgrade path for armor.)
- **9 armor items total** (3 slots x 3 tiers).
- **Effect: incoming damage reduction only**, per piece:
      Leather = 3%   |   Iron = 6%   |   Steel = 10%   (per piece)
  Full Steel set = 30%. This STACKS with the Defense stat and the shield, and MUST go through the
  EXISTING combined-reduction clamp (~60%) — do NOT add a second clamp or a parallel reduction path.
  A full tank build (shield 25% + Defense 40% + Steel set 30%) should hit the clamp ceiling — that's
  intended.
- **Movement/stamina penalties: NOT in scope** (deferred tradeoff — do not add them).
- **Stats-only:** no visible armor on the character in THIS task. But every armor item carries an
  `AssetId` field (empty/nil for now) so armor-visuals-01 fills it in with no schema rework.

## PART 1 — Armor data + service
- Define the 9 armor items as DATA (mirror how Weapons.luau is structured — a registry/table), each with:
  name, slot (Helmet/Chest/Legs), tier (Leather/Iron/Steel), damageReduction, cost (for purchase),
  and an empty `AssetId` placeholder for future visuals.
  Put costs in the data, not hardcoded in service logic (same rule as StructureRegistry).
- Extend PlayerData for owned armor + equipped-per-slot (Ground Rule 7: EXTEND / find-or-create; do NOT
  recreate the data structure). Existing players default to nothing owned/equipped.
- ArmorService (or extend an existing service if cleaner — say which and why):
  - Equip/unequip per slot, server-validated (owns the item; item matches the slot).
  - Compute total armor reduction from equipped pieces.
  - Feed that into the EXISTING combined damage-reduction path in CombatService, through the SAME clamp.
    Verify shield + Defense + armor all compose through ONE clamp.

## PART 2 — Implement the last 2 dead remotes
`EquipWeapon` and `PurchaseWeapon` are still unbound and HANG if called (the final 2 of the original 12).
They belong to this acquire-and-equip system — implement them here:
- `PurchaseWeapon` — buy a weapon (and armor, if it makes sense to share the path — your call; say which)
  using EconomyService + the registry's cost data. Server-validated (affordable, not already owned).
- `EquipWeapon` — equip an owned weapon, server-validated.
- Add the equivalent for armor (purchase + equip) — either via the same remotes or clean new ones; report
  the remote surface you ended up with.
- After this task, ALL 12 original remotes should respond (none hanging). Verify and report.

## PART 3 — UI
- **Armor tab (inventory panel):** it currently renders "No armor yet." Wire it to REAL owned-armor data —
  show owned pieces with slot/tier/reduction, and which are equipped. Pure view over the server snapshot.
- **Equipping:** provide a way to equip/unequip from the UI (a button on the armor row is fine — >=44px).
  Server-authoritative; re-render from confirmed state; Toast on success/failure.
- Kit + Theme only, ZERO inline styling (every prior panel held this bar). Responsive, phone reflow.
- If you add any entry point/keybind, register it in the KeybindRegistry so the Controls panel shows it.

## PART 4 — Visuals SPIKE (investigate only — do NOT build visuals)
Armor-visuals-01 will attach 9 meshes to the character. The rig is Roblox's CONSTRAINT-based rig (not
Motor6D) — which already forced a runtime KeyframeSequence solution for the weapon holds. Determine the
viable path BEFORE we commit to the art pipeline:
- Inspect the actual character rig. Report whether visible armor should be:
  (a) RIGID welded meshes (helmet/pauldrons welded to a bone — likely compatible with the existing
      Meshy -> Open Cloud pipeline that produced the weapons), or
  (b) LAYERED CLOTHING (cage-fitted, deforms with the body — a DIFFERENT asset pipeline; unclear whether
      Meshy can produce properly caged assets).
- Report what armor-visuals-01 will actually require: attachment points available on the rig, whether
  each slot (helmet/chest/legs) can be rigid-welded or needs deformation, and any blockers.
- This is INVESTIGATION ONLY. Do not generate meshes, do not upload assets, do not attach anything.

## Report back (write outbox/armor-system-01.result.md)
- PART 1: the 9-item armor data table (paste it), where owned/equipped state lives, and PROOF that armor +
  Defense + shield compose through ONE clamp (show numbers: e.g. full Steel + shield + Defense 40 -> a
  100-dmg hit lands as ~40, clamped).
- PART 2: EquipWeapon + PurchaseWeapon (+ armor equivalents) bound; the final remote surface; confirmation
  ALL 12 original remotes now respond (no hangs).
- PART 3: Armor tab showing real data; equip/unequip works from UI; Kit/Theme-only confirmed.
- PART 4: the visuals spike findings — rigid vs layered, attachment points, and what armor-visuals-01 needs.
- Regression: HUD, inventory, character sheet, structure upgrades, sprint all still fine.
- STATUS: DONE / PARTIAL / BLOCKED + anything unexpected.

## Guardrails
- rojo serve + Connect only (no 2nd instance). NEVER rojo build.
- Armor is TIERED (acquired), NOT upgraded in place — do not build an armor upgrade path.
- ONE combined damage-reduction clamp — do not add a second clamp or parallel reduction path.
- No movement/stamina penalties (deferred).
- Ground Rule 7: extend PlayerData / find-or-create. Async rule: no blocking InvokeServer in render paths.
- Costs + item stats as DATA, not hardcoded in service logic.
- PART 4 is investigation only — generate/upload/attach NOTHING.
- Everything in canonical src. No place-only drift. No secrets in the result (outbox is public).
