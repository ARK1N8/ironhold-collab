TASK: hud-rebuild-01 — rebuild CombatHUD on the framework + Platform migration

FROM: Studio Director
DATE: 2026-07-06
SCOPE: GAME project. Rebuilds the single surviving HUD (CombatHUD) onto the UI framework, keeping its
       combat juice/logic, and migrates the controllers it touches off GameConfig.PLATFORM.
DEPENDS ON: framework-core-01 (Client.UI.Kit: Bar/Label/Panel/Button/Toast/TabBar/ScrollList, Theme,
       layer manager, Platform.luau), client-boot-fix-01 (CombatHUD is the sole HUD owner),
       inventory-panel-01 (established the async-open rule), contract v2.2.0 §11.
ENV: sync stable on Host 127.0.0.1; rojo serve running on 127.0.0.1:34872 — do NOT start a 2nd instance.
       rojo serve + Connect only, NEVER rojo build.

## Goal
Rebuild CombatHUD as a PERSISTENT-layer HUD built from the Kit + Theme, replacing its ad-hoc UI, while
preserving its existing combat feedback (the "juice" the audit said to keep). All live data — health,
XP/level, and resources already exist and work.

## Requirements
1. **HUD on the framework.**
   - Register the HUD in the layer manager's PERSISTENT layer (always visible; unaffected by modal
     open/close). It must NOT be a modal.
   - Rebuild its elements from the Kit: health Bar, XP/level Bar or Label, resource Rows (Gold/Wood/Stone)
     via Kit Label/Row. All styling from Theme — ZERO inline colors/sizes (inventory-panel-01 proved the
     Theme covers 100%; hold that bar).
   - HUD layout per §11.5: health+level+XP top-left, resources top-right, (hotbar stays the default
     backpack for now unless it's trivial to theme — do not rebuild the hotbar in this task).
   - Scale-based sizing, tap targets >=44px where interactive, phone-viewport reflow sanity check.
2. **Preserve combat juice.** Keep CombatHUD's existing combat feedback/logic (hit indicators, floating
   combat text, whatever it does today). Re-skin the presentation onto the Kit; do NOT rip out the
   behavior. Report what combat feedback existed and confirm it still fires after the rebuild.
3. **Live data, async.** HUD reads player data (health/XP/level/resources) from the existing sources.
   ENFORCE THE ASYNC-OPEN RULE from inventory-panel-01: NO blocking InvokeServer in the HUD's
   init/render path — any server fetch is async (task.spawn + a neutral/loading state) so a slow server
   can never stall the boot chain (this is the exact failure mode that starved controllers pre-boot-fix).
   Prefer event-driven updates over polling (§11.8): subscribe to the data changes, don't poll per-frame.
4. **Platform migration (the controllers this task touches).**
   - Migrate CombatHUD and the Combat + Build controllers OFF `GameConfig.PLATFORM` and ONTO
     `Platform.luau` (Get()/HasTouch()/HasGamepad() + subscribe to Platform.Changed where they branch on
     input type). That's 3 of the 8 flagged usages.
   - LEAVE the ClientBootstrap GameConfig.PLATFORM usage for a final separate pass (migrate the entry
     point last, once controllers are off it). Report how many GameConfig.PLATFORM usages remain after
     this task.
5. **Isolation.** Do NOT touch the inventory panel, do NOT build the structure/character shells, do NOT
   rebuild the hotbar. HUD + the 3-controller Platform migration only.

## Report back (write outbox/hud-rebuild-01.result.md)
- HUD rebuilt on the Kit in the PERSISTENT layer; elements + Theme-only styling confirmed (any inline
  values? there should be none).
- Combat juice: what existed, confirmed still firing post-rebuild.
- Async-open rule honored (no blocking InvokeServer in init/render); update model (event-driven?).
- Verified in Play: HP/XP/level/resources render live and update; HUD stays visible under modal
  open/close (persistent layer); phone-viewport reflow ok.
- Platform migration: CombatHUD + Combat + Build now on Platform.luau; remaining GameConfig.PLATFORM
  usage count (should be down to ~ClientBootstrap + any not-yet-touched).
- STATUS: DONE / PARTIAL / BLOCKED + anything unexpected.

## Guardrails
- rojo serve + Connect only (running on 127.0.0.1:34872 — no 2nd instance). NEVER rojo build.
- NO blocking InvokeServer in the HUD init/render path (async rule).
- Any new reusable component goes into the Kit + Theme, never inline.
- HUD + touched-controller Platform migration ONLY — no other panels/controllers, no hotbar rebuild.
- Everything in canonical src (Client mapped/synced). No place-only drift.
- No secrets in the result (outbox is public).
