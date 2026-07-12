TASK: structure-upgrades-01 — instant structure upgrade system (5 levels, all nine) + upgrade panel

FROM: Studio Director
DATE: 2026-07-09
SCOPE: GAME project. Builds the §12.2 structure UPGRADE system (upgrade-in-place) + its UI panel.
DEPENDS ON: StructureRegistry / StructureService / TownHallService (level-cap enforcement already exists)
       / EconomyService; framework-core-01 (Kit/Theme/layer manager); character-sheet-01 (panel pattern);
       contract v2.2.0 §12.2 + Ground Rule 7 (idempotent find-or-create) + the async rule.
ENV: sync stable on Host 127.0.0.1; rojo serve running on 127.0.0.1:34872 — do NOT start a 2nd instance.
       rojo serve + Connect only, NEVER rojo build.

## Design (locked with Director)
- **Instant upgrades** — spend resources, upgrade applies immediately. NO timed/queued builds (that's a
  later, additive system; do not build timers, offline progression, or a build queue).
- **5 levels per structure.** L1 = the placed base state; upgradeable through L5.
- **ALL NINE canonical structures get an upgrade path.**
- **TownHall is the gate:** no structure may exceed the TownHall's current level. TownHall itself upgrades
  via the existing UpgradeTownHall remote. Lean on TownHallService's existing level-cap enforcement — do
  NOT reimplement gating.
- **Costs: geometric, ~1.6x per level** (costs accelerate — this drives the gathering loop).
- **Benefits: ~+50% HP per level**, PLUS role-specific gains scaled similarly (e.g. Cannon damage,
  Watchtower range, storage/capacity — whatever stat each structure actually has).
- **Costs + benefits live in StructureRegistry AS DATA** (a per-structure level table), NOT hardcoded in
  service logic — tuning must be a table edit, not a code change.

## STEP 0 — Read the registry first, then design the curves from REAL fields
Before writing anything: read StructureRegistry and REPORT the nine structures and each one's actual stat
fields (HP? damage? range? capacity? production?).
Then design each structure's 5-level curve FROM ITS REAL FIELDS, following the curve rules above. Do NOT
invent stats a structure doesn't have, and do NOT give a WeaponDisplay the same curve as a Cannon — the
benefit must be role-appropriate to what the structure actually does. If a structure has no obvious
upgradeable stat, say so and propose the most sensible benefit (e.g. HP only) rather than fabricating one.
Report the full nine-structure cost/benefit table in the result so the Director can tune it.

## PART 1 — Upgrade system (server-authoritative)
- Add a `level` to each PLACED structure instance (per Ground Rule 7: EXTEND the existing structure data /
  find-or-create; do NOT recreate structure state). Existing placed structures default to L1.
- `UpgradeStructure(structureId)` remote (server-validated):
  - structure exists + owned by this player,
  - not already at max level (5),
  - **TownHall gate**: target level <= TownHall level,
  - player has the resources -> spend via EconomyService,
  - apply the new level's stats (HP and role-specific), persist the level.
  Reject cleanly with a reason on any failure (the UI shows it).
- Applying a level must update the LIVE structure (HP/stats on the actual instance), not just data.
- **Implement the two dead remotes that belong to this system:** `PurchaseStructure` and `CraftStructure`
  (currently unbound and hang if called). Wire them to EconomyService/StructureService using the registry's
  cost data. Do NOT implement EquipWeapon/PurchaseWeapon (those belong to the weapons/armor tier work).
- Read path for the UI: expose current level, next-level cost + benefits, and whether upgrade is blocked
  (and why: TownHall gate / insufficient resources / max level). Async-safe, no blocking work in render paths.

## PART 2 — Upgrade panel (on the framework)
- Clicking/selecting a placed structure opens the **structure-upgrade modal** via the layer manager
  (§11.11 one-modal + scrim). Built from the Kit + Theme — ZERO inline styling (all three prior panels held
  this bar).
- Shows: structure name, current level (e.g. "Level 2 / 5"), current stats, **next-level benefits**, the
  **cost**, and an **Upgrade button** (>=44px) that is DISABLED with a clear reason when blocked
  (insufficient resources / TownHall gate / max level). Use the Kit Button's disabled state.
- On upgrade: call the remote, re-render from the server's confirmed state (pure view; never mutate
  client-side). Raise a Toast on success/failure.
- Responsive: scale-based, phone-viewport reflow, >=44px targets. Register the keybind/entry point in the
  KeybindRegistry so it appears in the Controls panel automatically.

## Report back (write outbox/structure-upgrades-01.result.md)
- STEP 0: the nine structures + their real stat fields, and the FULL 5-level cost/benefit table you designed
  (this is the thing the Director will tune — make it a clear table).
- PART 1: level stored on placed structures (extended, not recreated); UpgradeStructure validated
  (ownership, max level, TownHall gate, resources); live structure stats actually update; PurchaseStructure
  + CraftStructure now bound and returning (no longer hanging).
- PART 2: panel on Kit (zero inline), modal via layer manager, shows level/stats/next-benefits/cost,
  Upgrade button disabled-with-reason, Toast feedback, entry point in the KeybindRegistry.
- Verified in Play with real numbers: upgrade a structure, show resources spent, HP/stat actually increased,
  TownHall gate blocks an over-level upgrade, insufficient-resources blocks with a reason.
- Regression: existing remotes/panels (HUD, inventory, character sheet, sprint) still fine.
- STATUS: DONE / PARTIAL / BLOCKED + anything unexpected.

## Guardrails
- rojo serve + Connect only (running on 127.0.0.1:34872 — no 2nd instance). NEVER rojo build.
- INSTANT upgrades only — no timers, no build queue, no offline progression.
- Costs/benefits as DATA in StructureRegistry, not hardcoded in service logic.
- Ground Rule 7: extend structure/player data; find-or-create; never destroy-and-recreate.
- Async rule: no blocking InvokeServer in any open/render path.
- Server-authoritative: never trust the client for level, cost, or resource state.
- Do NOT implement EquipWeapon/PurchaseWeapon or any armor/weapon-tier work (that's the next system).
- Everything in canonical src. No place-only drift. No secrets in the result (outbox is public).
