# RESULT: character-sheet-01 — character sheet panel + platform-aware Controls panel

FROM: Claude Code (local executor)
DATE: 2026-07-10
STATUS: **DONE** — both panels built on the Kit (zero inline styling), pure views, verified live in Play
(allocate → derived updates, cap indicators, gated + buttons, server rejects at 0; platform-aware Controls
from a single registry). No regression.

Recovery note: a power outage reverted the live place to a pre-hud-rebuild save; I restarted rojo serve +
the inbox bridge, and once the Rojo plugin reconnected (Host **127.0.0.1** — `localhost` fails per
rojo-sync-diagnose) the full src re-synced (hud-rebuild + character-stats + this task) and I tested.

## Files
```
Client/KeybindRegistry.luau            (NEW — single source of truth for bindings)
Client/Panels/CharacterSheet.luau      (NEW — stat panel, pure view over GetStats)
Client/Panels/ControlsPanel.luau       (NEW — platform-aware controls, renders from registry)
Client/MenusInit.client.luau           (NEW — binds C, wires HUD ?/Character buttons)
Client/HUD/CombatHUD.luau              (+always-visible Character + "?" menu buttons + hooks)
Client/InventoryPanelInit.client.luau  (now reads I/Esc from the registry)
Client/SprintClient.client.luau        (now reads Shift from the registry)
```

## PART 1 — Character sheet panel (verified)
- **Modal via layer manager** (§11.11): opening it closes any other modal; scrim behind. Verified opening
  the Controls panel closed the sheet (visible modals = 1). Opens on **C** (from registry), closes via
  Panel X / Esc / closeModal.
- **Built from Kit + Theme — ZERO inline values** (grep: 0 `Color3.*` in the panel; cap indicator uses the
  Theme gold token, not an inline color).
- **Content:** prominent **Unspent Points** counter at top; one row per stat with **points + derived
  effect in human terms** (from the server's `derived`, not recomputed): e.g. `Vitality 5  +5 Max HP
  (105)`, `Speed 30  +30% move speed  (MAX)`, `Defense … % dmg reduction`, `Strength … % melee damage`,
  `Endurance … max stamina, faster regen`.
- **Cap indicators:** Speed (cap +30%) and Defense (cap 40%) show **"(MAX)"** and turn **gold** when
  capped — verified Speed at 30 pts → "+30% move speed (MAX)" in gold.
- **+ button per stat** (44px), **enabled only when unspentPoints > 0** — verified all + buttons disable
  at 0. Clicking → `AllocateStat`; verified via the panel's own `Allocate` (Vitality 0→5 pts, derived
  "+0 Max HP (100)" → "+5 Max HP (105)", unspent 6→5).
- **Pure view + server-authoritative:** allocation goes to the server and the panel re-renders from the
  confirmed snapshot (`AllocateStat` response + `StatsChanged`). **Server rejects allocate at 0 points →
  false** (verified). No local stat mutation.
- **Async-safe:** GetStats/AllocateStat run in `task.spawn` with a "..." loading state — no blocking
  InvokeServer in open/render.

## PART 2 — Controls / keybind panel (verified)
- **Single KeybindRegistry** (Ground Rule 7 spirit — one source, not a stale duplicate). Both the panel
  and (where cheap) the input code read from it. Pasted below.
- **Wired to the registry** (`wired=true`, menu guaranteed accurate): Sprint (Shift), Open Inventory (I),
  Character Sheet (C), Controls, Close (Esc). These inputs now READ their KeyCode from the registry.
- **Still hardcoded** (`wired=false`, shown in the menu but binding lives in a controller — reported, not
  faked): Attack (LMB/RT), Interact (E/RB), Build (B), Rotate (R), Place Block (LMB/A), Toggle Shield (Q,
  in the place-only ShieldToggleClient). Migrating these is a small follow-up.
- **Platform-aware (§11.1):** renders desktop keys / gamepad buttons / touch legend via
  `Registry.LabelFor(entry, Platform.Get())`, subscribed to `Platform.Changed` to re-render. Verified:
  desktop shows "Keyboard / Mouse" + key labels (Attack→Left Click, Sprint→Shift, Character→C …);
  the touch legend is correct (Sprint→"Sprint button", Inventory→"Inventory button").
- **Discoverable:** always-visible **"?"** and **Character** buttons on the HUD (PERSISTENT layer, all
  platforms) open the two panels — reachable without knowing a keybind. Verified both present + visible.
- 11 bindings enumerated (attack, interact, sprint, inventory, character, controls, build, rotate, place,
  shield, close).

The registry (`Client/KeybindRegistry.luau`):
```lua
Keys = { OpenInventory=I, CharacterSheet=C, Close=Escape, Sprint={LeftShift,RightShift} }
Actions = {  -- action, desktop, touch, gamepad, wired
  Attack (Left Click / Attack button / RT)         wired=false
  Interact / Harvest (E / Use button / RB)          wired=false
  Sprint (hold) (Shift / Sprint button / —)         wired=true
  Open Inventory (I / Inventory button / —)         wired=true
  Character Sheet (C / Character button / —)         wired=true
  Controls / Help (— / ? button / —)                wired=true
  Build Mode (B / Build button / —)                 wired=false
  Rotate (build) (R / Rotate button / —)            wired=false
  Place Block (Left Click / Tap / A)                wired=false
  Toggle Shield (Q / Shield button / —)             wired=false
  Close / Cancel (Esc / X button / B)               wired=true
}
```

## Regression
Boot: 0 non-benign client/server errors; HUD present (health/XP/stamina bars + Character/"?" buttons);
inventory + sprint still work; remotes respond (4/4 spot-check incl. GetStats). One-modal holds across
inventory/character/controls.

## VERIFY (evidence)
- Sheet: modal+scrim; Unspent 30; Speed 30 → "+30% move speed (MAX)" gold; +buttons disable at 0;
  CS.Allocate(Vitality) → "+5 Max HP (105)"; server rejects at 0 → false.
- Controls: one-modal (sheet closed on open); header "Keyboard / Mouse"; registry-driven rows; touch
  labels correct; 11 bindings.
- Theme-only: 0 inline `Color3` in CharacterSheet/ControlsPanel.
- Guardrails: no `rojo build`; no 2nd serve; find-or-create registry; async open/render; pure views
  (server authoritative); Kit/Theme only; Speed attack-speed hook left disabled; no armor/upgrade UI;
  all in canonical src; no secrets.

## STATUS: DONE

## BACK TO DIRECTOR
1. **Save the place (Ctrl+S).** The outage reverted the live place to an old save; after the Rojo re-sync
   everything is correct in-session, but **save so the next restart doesn't revert again** (this is the
   recurring unsaved-place issue).
2. **6 inputs still hardcoded** (Attack/Interact/Build/Rotate/Place/Shield) — the Controls menu shows them
   correctly but they don't yet READ from the registry. Small follow-up to route them through it so the
   menu can never drift. (Shield lives in the place-only `ShieldToggleClient`, outside src.)
3. Both panels are pure views on the live stat/registry data — ready for whatever comes next
   (structure-upgrade / armor UI when those systems exist).
