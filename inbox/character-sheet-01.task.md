TASK: character-sheet-01 — character sheet panel + platform-aware Controls panel

FROM: Studio Director
DATE: 2026-07-09
SCOPE: GAME project. Two modal panels on the existing framework, both pure views over live data.
DEPENDS ON: character-stats-01 (GetStats / StatsChanged / AllocateStat all live + async-safe),
       framework-core-01 (Kit: Panel/Bar/Label/Button/TabBar/ScrollList, Theme, layer manager),
       inventory-panel-01 (the panel pattern + the async-open rule), Platform.luau (§11.1).
ENV: sync stable on Host 127.0.0.1; rojo serve running on 127.0.0.1:34872 — do NOT start a 2nd instance.
       rojo serve + Connect only, NEVER rojo build.

## PART 1 — Character sheet panel
Build the character-sheet modal. Everything it needs is already live.
- **Modal via the layer manager** (§11.11 one-modal-at-a-time + scrim). Open on a keybind (suggest **C**;
  inventory already uses I). Close via Panel X, Esc, closeModal.
- **Built from the Kit** (Panel + Label/Row + Button + ScrollList), 100% Theme styling — ZERO inline values
  (inventory and HUD both held this bar; hold it).
- **Content — show raw points AND derived effects (Director confirmed):**
  - An **unspent points** counter at the top (prominent — it's the call to action).
  - One row per stat: name, allocated points, and the DERIVED effect in human terms. `GetStats` already
    returns derived {maxHealth, meleeMult, speedMult, defenseReduction} — use it, don't recompute client-side.
    e.g.  "Vitality  10   +50 Max HP (150)"
          "Strength  10   +20% melee damage"
          "Speed     30   +30% move speed  (MAX)"
          "Defense   40   40% dmg reduction (MAX)"
          "Endurance 10   200 max stamina, faster regen"
  - **Cap indicators:** Speed (cap +30%) and Defense (cap 40%) must clearly show when capped/at max so
    players don't waste points. Visually distinct (Theme token, not an inline color).
  - A **+ button per stat** (>=44px tap target), ENABLED only when unspentPoints > 0, disabled otherwise
    (Kit Button already has a disabled state). Clicking calls `AllocateStat(statName)`.
- **Pure view + server-authoritative:** the panel never mutates stats locally. On click -> AllocateStat ->
  re-render from the server's response / `StatsChanged` push. Optimistic UI is NOT required; prefer
  rendering what the server confirms.
- **Async-safe:** no blocking InvokeServer in the open/render path (task.spawn + a brief loading state, as
  inventory-panel-01 established).
- Responsive: scale-based sizing, phone-viewport reflow, >=44px targets.

## PART 2 — Controls / keybind panel (platform-aware)
Players currently have NO way to discover the controls (I = inventory, Shift = sprint, C = character, plus
build/combat inputs). Fix that.
- **Single keybind registry (important — avoid a stale hand-maintained list):** create ONE shared registry
  module that maps action -> binding per platform, e.g.
      { action="Open Inventory", key=Enum.KeyCode.I, touch="Inventory button", gamepad=... }
  The CONTROLS PANEL RENDERS FROM THIS REGISTRY. Where cheap, have the actual input code READ its binding
  from the registry too, so the menu can never drift from reality (this is Ground Rule 7's spirit applied
  to docs: one source of truth, not a duplicate that goes stale).
  If wiring every existing input through the registry is a large refactor, DO NOT blow up scope: build the
  registry, wire what's cheap (the new/simple ones), and REPORT which inputs are still hardcoded so we can
  migrate them later. Do not fake entries.
- **Platform-aware rendering (§11.1):** use `Platform.Get()` / `HasTouch()` / `HasGamepad()` and subscribe
  to `Platform.Changed`. Show KEYBOARD bindings on desktop, GAMEPAD bindings on controller, and the TOUCH
  button legend on mobile (a key list is meaningless on touch). Re-render on Platform.Changed.
- **Modal on the Kit**, same rules as Part 1 (layer manager, Theme-only, responsive, >=44px).
- **Discoverable:** it must be reachable without already knowing a keybind — e.g. a small "?" / Controls
  button on the HUD (persistent layer) that opens it, in addition to any keybind. On touch especially, a
  visible button is the only discoverable path.
- Enumerate every current binding you can find (inventory I, sprint Shift + touch button, character C,
  build controls, attack/left-click, shield toggle if present, etc.) and include them.

## Report back (write outbox/character-sheet-01.result.md)
- PART 1: panel built on Kit (Theme-only? confirm zero inline), opens as modal (one-modal + scrim verified),
  shows points + derived + cap indicators, + button gated on unspentPoints, AllocateStat wired and
  server-confirmed. Verified in Play with real numbers (allocate a point -> derived value updates).
- PART 2: the keybind registry (paste it), which inputs now READ from it vs. still hardcoded (be honest),
  platform-aware rendering verified (desktop keys vs touch legend), discoverable HUD entry point.
- Regression: HUD + inventory + sprint still fine; remotes still respond.
- STATUS: DONE / PARTIAL / BLOCKED + anything unexpected.

## Guardrails
- rojo serve + Connect only (running on 127.0.0.1:34872 — no 2nd instance). NEVER rojo build.
- Async rule: no blocking InvokeServer in any open/render path.
- Ground Rule 7: idempotent find-or-create for any registry/singleton (the keybind registry included).
- Panels are PURE VIEWS; server remains authoritative for stats. Never mutate stats client-side.
- Any new reusable component goes into the Kit + Theme, never inline.
- Do NOT build structure-upgrade or armor UI (later systems). Do NOT re-enable Speed's attack-speed hook.
- Everything in canonical src. No place-only drift. No secrets in the result (outbox is public).
