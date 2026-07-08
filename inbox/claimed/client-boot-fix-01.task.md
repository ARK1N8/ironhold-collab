TASK: client-boot-fix-01 — kill the double-bootstrap, one client entry + one HUD

FROM: Studio Director
DATE: 2026-07-06
SCOPE: GAME project. Surgical correctness fix. Delete duplicate boot scripts + confirm a single
       client entry point and single HUD owner. Do NOT build the UI framework in this task.
DEPENDS ON: ui-audit-01 (found the client boots twice — the real cause of the "overlapping" HUD).

## Context (from the audit)
Two enabled bootstrap LocalScripts in StarterPlayerScripts both initialize the controllers:
  - IronholdClient      -> PLACE-ONLY (not in src; the duplicate that shouldn't exist)
  - Client.ClientMain -> ClientBootstrap  -> from Rojo src (the tracked, canonical entry)
Runtime proof: live PlayerGui held TWO Vignettes, plus BOTH HUDGui and CombatHUD drawing
HP/XP/gold/level into the same top-left corner, two "Lv 1" indicators, etc.
This is the same place-vs-src drift pattern as the two CONTRACTS.md files: a place-only artifact
duplicating a src counterpart, both active.

## Decisions (confirmed with Director)
- ClientBootstrap (from src) is the SOLE client entry point.
- CombatHUD is the SOLE surviving HUD owner (it has the combat juice / logic worth keeping).
- DELETE the place-only IronholdClient bootstrap.
- DELETE HUDController (its HUD is the older duplicate; CombatHUD replaces it).

## Steps
1. RECONNECT Rojo + confirm disk==live FIRST (sync has been flaky; verify parity so deletions apply
   to the state we think we're editing). rojo serve + Connect only — NEVER rojo build.
2. Delete the place-only `IronholdClient` bootstrap LocalScript from StarterPlayerScripts. Confirm it
   is NOT represented in src (place-only) before deleting; if it unexpectedly exists in src too, STOP
   and report.
3. Delete `HUDController` (the duplicate HUD). Confirm CombatHUD does not depend on HUDController; if
   anything references HUDController, report it before removing so we don't break a caller.
4. Ensure ClientBootstrap initializes exactly one of each controller and does NOT init a second HUD.
   The surviving controller set per the audit: CombatHUD (HUD), Inventory, Build, World, Combat — one
   instance each. Remove any lingering init of the deleted scripts.
5. Verify in Play: PlayerGui holds EXACTLY ONE of each — one Vignette, one HUD (CombatHUD), one HP bar,
   one XP/level indicator, one gold label. No duplicate labels stacked in the corner. Report the live
   PlayerGui tree as evidence.
6. Persist the change so it survives Play-stop; if any deletion was made live, ensure it's reflected in
   src / the canonical state (this fix must not itself become new place-vs-src drift).

## Report back (write outbox/client-boot-fix-01.result.md)
- Rojo parity confirmed before edits?
- IronholdClient deleted (and confirmed place-only)? HUDController deleted (and confirmed no broken
  references)?
- ClientBootstrap confirmed as sole entry, initializing one of each controller, CombatHUD as sole HUD?
- Play verification: paste the live PlayerGui tree showing ONE of everything (the pass/fail evidence).
- Confirmation the fix is persisted to the canonical state (no new place-only orphan created).
- STATUS: DONE / PARTIAL / BLOCKED + anything unexpected.

## Guardrails
- rojo serve + Connect only — NEVER rojo build (would wipe VikingTier1 / StarterPack / door service).
- This is a correctness fix ONLY — do NOT build the layer manager, component kit, or restyle anything.
  That's the next task.
- Before deleting HUDController, confirm nothing else depends on it; report first if it does.
- No secrets in the result (outbox is public).
