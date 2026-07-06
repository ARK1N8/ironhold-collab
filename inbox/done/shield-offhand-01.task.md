# TASK: shield-offhand-01 — shield becomes a toggled passive off-hand hold + Seax coexistence

FROM: Studio Director
DATE: 2026-07-06
SCOPE: GAME project. Changes shield handling + adds incoming-damage reduction to CombatService.
DESIGN DECISION (Jon): shield is no longer an activatable Tool. It's a toggled passive
left-arm hold that coexists with a one-handed weapon in the right hand. Left-click always
strikes the RIGHT-HAND weapon. Only the Seax is one-handed right now; all other weapons are
two-handed and cannot coexist with the shield.

## FIRST: sync parity (recurring issue — do this before editing)
`rojo serve` live-sync has dropped disk->live pushes for 3 sessions running. Before changing
anything: reconnect the Rojo plugin, confirm disk `src/` == live for the weapon HoldServers
and `TwoHandHold`, and report whether they matched or had to be reconciled. We must be
editing the same state Jon sees — a prior shield "discrepancy" was likely a parity gap.

## PART 1 — Shield: Tool -> toggled passive strap-hold
Currently RoundShield is a StarterPack **Tool** anchored by a `ShieldStrap` weld to
LeftLowerArm. Problem: a character can only hold ONE Tool equipped at a time, so a Tool
shield can't coexist with a weapon Tool. Convert it:

- Remove the shield from the **Tool** system so it no longer occupies the equipped-Tool slot.
  Keep the visual/mesh + the `ShieldStrap` weld to LeftLowerArm (its placement measured
  correct last task — do NOT change the placement/pose).
- Make "shield up" a **toggled persistent state** on the character, independent of which
  weapon Tool is equipped. Up = shield mesh strapped to the left arm (current pose). Down =
  removed/hidden. State survives weapon swaps.
- **Toggle input:** bind a key (recommend **`Q`**; if taken, pick a free one and report it)
  to toggle shield up/down. Touch/gamepad can be a follow-up — keyboard toggle is enough now.
- The shield being up must NOT consume the Tool slot and must NOT intercept left-click.

## PART 2 — Coexistence rules
- Shield may be up ONLY while a **one-handed** weapon is equipped (or none). Right now the
  only one-hander is the **Seax** — tag weapons with a hand-count so this generalizes later
  (e.g. read Class / an added `Hands` field in Weapons.luau; Seax = 1, all others = 2).
- If the shield is up and the player equips a **two-handed** weapon (BeardedAxe, DaneAxe,
  Spear, HuntingBow), the shield **auto-drops** (goes down).
- While a two-hander is equipped, attempting to toggle the shield up is **rejected** (no-op,
  optionally a small UI/sound cue) until the player switches back to a one-hander.
- Server-authoritative: the server owns the real "shield up" flag per player; never trust a
  client claim (consistent with existing anti-cheat model).

## PART 3 — Left-click strikes the right-hand weapon
- Confirm/ensure left-click (`Tool.Activated`) fires the equipped RIGHT-HAND weapon's strike,
  regardless of shield state. With Seax + shield up, left-click = Seax strike. The shield
  must never eat the click or trigger its own action (its old block/attack is retired for now).

## PART 4 — Passive damage reduction while shield is up
- Add server-side incoming-damage reduction in **`CombatService`**, applied at every point
  that damages a PLAYER (PvP `humanoid:TakeDamage` and any NPC->player damage path), gated on
  the server's per-player "shield up" flag.
- Constant: `SHIELD_DAMAGE_REDUCTION = 0.25` (incoming damage x 0.75 while up). Put it where
  it's trivially tunable (GameConfig or a clearly-named CombatService constant). No active
  block/timing yet — passive as long as the shield is up.
- Do NOT alter outgoing weapon damage or other combat values in this task.

## PART 5 — Seax strike verification (we're in here anyway)
- The Seax strike animation was authored earlier this session but never reviewed. Verify it
  in Play WITH the shield up: equip Seax, toggle shield up, left-click. Confirm the strike
  plays on the right hand, the shield stays put on the left (doesn't fight the strike or
  glitch), and it resumes to idle cleanly with no errors.
- Report whether the Seax strike looks correct (a fast single-hand slash) or needs rework —
  don't fix it here if it's off, just report so the Director can spec it.

## Report back (write outbox/shield-offhand-01.result.md)
- Parity: matched or reconciled? (per-file)
- P1: shield converted off Tool system? toggle key used? up/down works + survives weapon swap?
- P2: coexistence — shield-up-only-with-Seax enforced? auto-drop on two-hander? toggle
  rejected while two-hander equipped? server-authoritative flag?
- P3: left-click strikes right-hand weapon confirmed (Seax + shield up)?
- P4: reduction added in CombatService, gated on server shield flag, constant = 0.25, tunable?
- P5: Seax strike verified with shield up — looks correct, or needs rework (describe)?
- STATUS: DONE / PARTIAL / BLOCKED + anything unexpected.

## Guardrails
- `rojo serve` + Connect only — NEVER `rojo build`.
- Bake persistent changes in Edit mode so they survive Play-stop; persist into the templates.
- Server-authoritative for shield state and damage reduction — no client-trusted flags.
- Touch ONLY: shield handling, the shield-coexistence logic, and the shield damage-reduction
  path in CombatService. Do NOT change outgoing weapon damage, the tree-harvest cooldown, the
  axe chops just fixed, or other weapons' animations.
- No secrets in the result (outbox is public).
