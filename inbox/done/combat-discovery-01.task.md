# TASK: combat-discovery-01 — animation API + chop diagnosis + damage model dump

FROM: Studio Director
DATE: 2026-07-05
SCOPE: GAME project. READ-ONLY discovery, except the one small doc edit in Part C.
       Do NOT author or change any animations in this task — diagnose only.

## Why
Before I write accurate specs for (a) fixing the two axe chops, (b) authoring the other
four weapon animations, (c) weapon VFX, and (d) a per-target damage matrix, I need the
current systems dumped so my specs map onto the real API and don't conflict with existing
combat values. This task gathers that; it does not change gameplay.

## Part A — Animation system surface
Report, with actual code where noted:

1. The runtime animation module (`TwoHandHold` / whatever drives holds + swings):
   - Full public API: every function it exposes, with signatures (e.g. `playSequence(...)`,
     hold/pose functions). Paste the signatures.
   - How a swing/strike is defined and played: is it a `KeyframeSequence` built at runtime?
     What are the keyframe fields (time, per-joint CFrame/rotation), and which rig
     parts/joints are addressable on the constraint rig (arms, torso, etc.)?
   - The priority/blend model: the idle hold plays at `Action4` — how does a swing override
     or stop it, and does it resume the hold after? Paste the relevant code.

2. The current **BeardedAxe swing** implementation in full — paste the code and say which
   script it lives in (WeaponClient? HoldServer? elsewhere).

3. **Diagnose why the axe chop looks wrong.** From both the code and a live Play observation,
   report which of these apply (and anything else): flat/horizontal arc instead of a downward
   chop; no wind-up (pose jumps to raised then drops); arm-only motion with a static torso;
   no follow-through (stops at impact); wrong priority so the idle hold fights the swing;
   timing too fast/slow; grip/hand positions shifting wrongly. Be specific — this diagnosis
   drives the fix.

## Part B — Current combat / damage model
Dump the present values so I can design the weapon × target damage matrix without conflicts:

1. `Weapons.luau` — for every weapon, list all combat-relevant fields (damage, Class,
   CanHarvest, range, cooldown/attack-speed, any per-target values). Paste the table.
2. `CombatService` — how is damage applied and routed to each target type
   (character, NPC, structure, tree)? Summarize the AUDIT-6 harvest/structure routing.
3. Structure HP / resistance — from `StructureRegistry` / `StructureService` /
   `StabilitySolver`: per-structure HP and any damage-resistance model.
4. Tree / resource harvest — from `ResourceService`: tree HP, harvest amounts, respawn.
5. NPC HP model — where NPC health is defined and typical values.
6. `GameConfig` — any combat constants (base damage, crit, ranges, cooldowns).

## Part C — Trivial doc fix (only change in this task)
`DIRECTOR_STATUS.md` §5 item 1 still reads as if reconnecting Rojo is a pending action.
consolidation-01 verified that reconcile is complete. Update §5 item 1 to reflect it as
DONE/verified (mirror how §5 item 4 reads), instead of a pending step.

## Report back (write outbox/combat-discovery-01.result.md)
- Part A: the animation API signatures, the BeardedAxe swing code, and the chop diagnosis.
- Part B: the consolidated current combat/damage values (tables preferred).
- Part C: confirm §5 item 1 updated.
- STATUS: DONE / PARTIAL / BLOCKED, and anything unexpected.

## Guardrails
- Discovery only — do NOT modify animations, weapons, combat, or damage values.
- `rojo serve` + Connect only if Studio interaction is needed — NEVER `rojo build`.
- No secrets in the result (outbox is public); refer to `.env` keys by name only.
