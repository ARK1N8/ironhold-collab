TASK: inventory-panel-01 — tabbed inventory panel (first real feature on the framework)

FROM: Studio Director
DATE: 2026-07-06
SCOPE: GAME project. First REAL panel built on framework-core. This is the framework's stress test —
       one careful build before we fan out HUD + shells in parallel.
DEPENDS ON: framework-core-01 (Client.UI.Kit, layer manager, Theme, Platform.luau), server-remotes-fix-01
       + tier1-handlers-01 (GetInventory returns live in ~50ms), contract v2.2.0 §11 + §12.1.
ENV: sync stable on Host 127.0.0.1; rojo serve running on 127.0.0.1:34872 — do NOT start a 2nd instance.
       rojo serve + Connect only, NEVER rojo build.

## Step 0 — remove the throwaway demo
Delete the framework-core F8 demo (the one-file-removable demo script). Confirm it's gone and the F8
keybind no longer does anything. Nothing real depended on it.

## Goal
Build the unified tabbed inventory panel per contract §12.1, on the framework — proving the Kit + layer
manager + live data all work together under one real feature.

## Requirements
1. **One panel, four tabs** (§12.1): Structures / Weapons / Armor / Resources.
   - Built on the Kit Panel (Clean Fantasy frame, header w/ title + close X) via `require(Client.UI.Kit)`.
   - A TabBar switching the four tabs (build a TabBar if the kit doesn't have one yet; if you add it,
     add it to the Kit + Theme, not inline — keep the kit the single source).
   - Tab content = a scrollable list of item rows using the Kit Label/Row (icon optional/placeholder if
     no icons yet — name + count is the floor).
2. **Opened as a MODAL via the layer manager** — so §11.11 one-modal-at-a-time applies (opening inventory
   closes any other modal; scrim behind). Provide an open trigger (keybind and/or a HUD button hook — a
   keybind is fine for now). Close via the Panel X, Esc, and closeModal.
3. **Wired to LIVE data** — populate from `GetInventory:InvokeServer()` (now returns ~50ms). The panel is
   a PURE VIEW (§11.6): it renders the server snapshot, never mutates inventory directly. Handle the
   empty case (no items in a tab) gracefully.
   - Map the inventory snapshot fields to the four tabs. Report how the snapshot is structured and how you
     bucketed items into tabs (if the snapshot doesn't cleanly separate armor/weapons/etc., report that —
     don't invent categories; bucket by what's actually there and note gaps).
4. **Responsive + touch** (§11.3/§11.4): scale-based sizing, tap targets >=44px, tabs tappable on touch.
   Sanity-check reflow at a phone-ish viewport.
5. **Isolation:** do NOT rebuild CombatHUD or the Build panel, do NOT touch other controllers. This task
   is the inventory panel only. (It MAY register itself for opening, but must not entangle other UI.)

## Framework stress-test reporting (important — this is why we're doing inventory solo first)
Explicitly report how the framework held up under a real feature:
- Did the Kit primitives cover the need, or did you have to add/patch anything (e.g. TabBar, a list
  container)? List every kit change.
- Did the layer manager modal flow work for a real panel (open/close/scrim/one-modal)?
- Did the Theme cover all styling, or were any inline values needed (there should be none — report if so)?
- Any rough edges in the framework API that the HUD/shell fan-out should know about before it runs.

## Report back (write outbox/inventory-panel-01.result.md)
- Demo deleted + F8 inert confirmed.
- Panel built on Kit, four tabs, opens as modal via layer manager (one-modal + scrim verified in Play).
- Live GetInventory wired; snapshot structure + how items bucketed into tabs; empty-tab handling.
- Responsive/touch verified (phone viewport reflow, >=44px targets).
- FRAMEWORK STRESS-TEST: kit changes made (if any), API rough edges, anything the fan-out should know.
- STATUS: DONE / PARTIAL / BLOCKED + anything unexpected.

## Guardrails
- rojo serve + Connect only (running on 127.0.0.1:34872 — no 2nd instance). NEVER rojo build.
- Inventory panel ONLY — do not rebuild HUD/Build or touch other controllers (parallel fan-out comes next).
- Any new reusable component goes into the Kit + Theme, never inline — keep the kit canonical.
- Panel is a pure view over the server snapshot; never mutate inventory client-side.
- Everything in canonical src (Client is mapped/synced). No place-only drift.
- No secrets in the result (outbox is public).
