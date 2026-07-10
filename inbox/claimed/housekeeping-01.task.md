TASK: housekeeping-01 — retire legacy InventoryController, final Platform pass, idempotency contract rule

FROM: Studio Director
DATE: 2026-07-06
SCOPE: GAME project. Three small cleanups on a clean base before the character-stats system.
DEPENDS ON: inventory-panel-01 (new InventoryPanel replaced the old controller), hud-rebuild-01
       (Platform migration down to ClientBootstrap; Net.luau idempotency bug fixed).
ENV: sync stable on Host 127.0.0.1; rojo serve running on 127.0.0.1:34872 — do NOT start a 2nd instance.
       rojo serve + Connect only, NEVER rojo build.

## Part 1 — Retire the legacy InventoryController (dependency-gated deletion)
The new InventoryPanel (inventory-panel-01) replaced the old InventoryController. Retire the old one —
but SAFELY:
1. First, search the codebase (src + live) for ALL references to InventoryController — requires, Init
   calls in ClientBootstrap, any other controller referencing it, keybinds, etc. List them.
2. If anything still depends on it, STOP and report the references before deleting (do not break a caller —
   same gate that made the HUDController removal safe). Re-point or remove those references as appropriate,
   or report for Director direction if it's non-obvious.
3. Once confirmed dead, delete InventoryController and remove its Init from ClientBootstrap's controller
   set. Confirm inventory still works end-to-end (open the InventoryPanel, live data renders) after removal.
   Verify no duplicate/orphaned inventory UI remains (this is the exact duplicate-artifact class we keep
   hitting — check the live PlayerGui for one inventory panel, no ghost).

## Part 2 — Final Platform migration (ClientBootstrap)
- Migrate the last `GameConfig.PLATFORM` usage — the ClientBootstrap boot-time write/read — onto
  `Platform.luau`. After this, GameConfig.PLATFORM should have ZERO remaining readers.
- Confirm the count is zero (grep) and that platform-dependent behavior still works (Get/HasTouch/
  HasGamepad + Changed). If GameConfig.PLATFORM is now fully unused, note whether the field itself should
  be removed (report; don't necessarily delete the config field if other data lives alongside it).

## Part 3 — Contract rule: idempotent find-or-create (no destroy-and-recreate)
Add a standing engineering constraint to CONTRACTS.md (canonical git-tracked v2.2.0 copy). This rule has
been earned by SIX destroy-and-recreate / duplicate-artifact bugs this session (Net folder x2, layer
manager, client boot, HUDController, and orphaned event connections).
- Add it as a short subsection under an appropriate existing section (or a small "Engineering Constraints"
  note) — use your judgment on placement; report where you put it. Suggested text:

    "Singleton/registry initialization MUST be idempotent (find-or-create). Init code must never destroy
     and recreate shared instances (e.g. the Net/remotes folder, UI layers, HUD, controllers) — doing so
     silently orphans existing references (client event connections, open UI) and has repeatedly caused
     drift/hang bugs. Re-running an Init must converge to one instance, not duplicate or tear down."

- This is a CONTENT addition to the contract, not a version-structure change. Bump patch or note it as a
  minor addition per how the contract versions — keep it lightweight; do NOT restructure §1–§12.
- Commit to the GAME repo (ARK1N8/IronHold), not the collab repo.

## Report back (write outbox/housekeeping-01.result.md)
- Part 1: references found to InventoryController; confirmation it's safely deleted; inventory still works;
  no orphaned/duplicate inventory UI in live PlayerGui.
- Part 2: ClientBootstrap migrated; GameConfig.PLATFORM reader count now zero; platform behavior verified.
- Part 3: idempotency rule added — where placed, exact text, committed to game repo (commit hash).
- STATUS: DONE / PARTIAL / BLOCKED + anything unexpected.

## Guardrails
- rojo serve + Connect only (running on 127.0.0.1:34872 — no 2nd instance). NEVER rojo build.
- Deletion is dependency-gated: confirm no live dependents before removing InventoryController; report first
  if any are non-trivial.
- Contract edit is the git-tracked canonical CONTRACTS.md; commit to ARK1N8/IronHold. Lightweight addition
  only — do not restructure existing sections.
- Everything in canonical src. No place-only drift.
- No secrets in the result (outbox is public).
