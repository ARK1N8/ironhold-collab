TASK: tier1-handlers-01 — implement PlaceBlock + HarvestTree server handlers (kill the live hangs)

FROM: Studio Director
DATE: 2026-07-06
SCOPE: GAME project. Adds the two missing OnServerInvoke handlers the client calls TODAY, so those
       features stop hanging. Sync is now stable (Host 127.0.0.1). rojo serve is running — do NOT start
       a second instance.
DEPENDS ON: server-remotes-fix-01 (found 6/12 remotes unimplemented; these are the 2 the client calls live).

## Context
From server-remotes-fix-01: PlaceBlock (build placement) and HarvestTree (tree chopping) have NO
OnServerInvoke anywhere in the codebase, but the client calls them today -> both hang forever. This is
missing implementation, not drift. The other 4 unimplemented remotes (EquipWeapon/PurchaseWeapon/
PurchaseStructure/CraftStructure) are DEFERRED — do NOT implement them here; they belong with the
progression/economy systems (contract §12) once tier ladders/costs are designed.

IMPORTANT — extend, don't duplicate: earlier combat-discovery found the tree-harvest bug lives in
existing logic (a per-hit cooldown in ResourceService/CombatService), and Weapons.luau already has
CanHarvest + Axe classes with damage values. READ the current harvest/placement logic first and wire
these handlers INTO the existing services. Do NOT create a parallel harvest system or a duplicate Net
binding (duplicate artifacts have caused 4 bugs this session).

## Part 1 — HarvestTree (hit-to-fell model)
Design decision (confirmed with Director): trees are HIT-TO-FELL.
- Each tree has HP. Each qualifying axe hit reduces tree HP by the weapon's harvest damage
  (use the existing Weapons.luau Axe damage / CanHarvest fields — do not invent new numbers).
- Tree falls (is harvested / yields wood) when HP reaches 0.
- The RESPAWN cooldown applies AFTER felling (tree removed -> regrows after cooldown), NOT per hit.
- FIX THE EXISTING BUG: the current ~300s cooldown fires on every surviving hit, so the 2nd hit is
  always rejected and no multi-hit tree can ever fell. Move the cooldown to post-fell/respawn. Confirm a
  tree taking ~7 BeardedAxe hits actually goes down on the 7th and yields wood.
- Server-authoritative: validate the hit server-side (weapon can harvest, target is a tree, in range,
  tree HP tracked on the server — never trust the client for HP or yield). Consistent with existing
  anti-cheat.
- Implement OnServerInvoke for HarvestTree, wired into ResourceService (and CombatService routing if
  that's where harvest hits are dispatched, per AUDIT-6). Report where you bound it and how tree HP is
  stored/tracked.

## Part 2 — PlaceBlock
- Implement OnServerInvoke for PlaceBlock, wired into the existing build/structure placement path
  (StructureService / BuildingService — whatever the client's build flow already targets).
- Server-authoritative placement validation: on-plot, not overlapping, resource cost (if the block/build
  system charges), grid snap per the existing build rules. Match how PlaceStructure already validates
  (it works — mirror its pattern; don't reinvent).
- Confirm the client's current PlaceBlock call now returns instead of hanging, and a block actually
  places in-world.

## Part 3 — verify no hangs remain (for these two)
- From a client in Play, call PlaceBlock and HarvestTree and confirm both RETURN (no hang).
- Confirm the other 6 previously-working remotes still work (no regression from touching the services).

## Report back (write outbox/tier1-handlers-01.result.md)
- What existing harvest/placement logic you found, and how you extended it (not duplicated).
- HarvestTree: hit-to-fell implemented? tree HP source + per-hit damage source (from Weapons.luau)?
  cooldown moved to respawn? Verified a ~7-hit tree fells on the final hit and yields wood?
- PlaceBlock: implemented + wired to existing placement path? returns instead of hanging? block places?
- Regression check: the 6 working remotes still return.
- STATUS: DONE / PARTIAL / BLOCKED + anything unexpected.

## Guardrails
- rojo serve + Connect only (already running on 127.0.0.1:34872 — do NOT start a second instance).
  NEVER rojo build.
- Extend existing services; do NOT create duplicate Net bindings or a parallel harvest/placement system.
- Do NOT implement the 4 deferred remotes (EquipWeapon/PurchaseWeapon/PurchaseStructure/CraftStructure).
- Persist to the canonical src/live-synced state; no new place-only drift.
- No secrets in the result (outbox is public).
