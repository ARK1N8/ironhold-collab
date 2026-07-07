TASK: ui-audit-01 — inventory the existing UI/HUD before building the framework

FROM: Studio Director
DATE: 2026-07-06
SCOPE: GAME project. READ-ONLY discovery. Do NOT change, restyle, or refactor any UI in this task.
DEPENDS ON: contract v2.2.0 §11 (Clean Fantasy UI/UX, layer model, one-modal-at-a-time rule).

## Why
Before building the thin-custom UI framework (layer manager + Clean Fantasy component kit), we need a
true inventory of the UI that exists today — what screens there are, what overlaps, what's reusable —
so the framework is designed against reality and we know exactly what it replaces. Measure first.

## Part A — What UI exists
Enumerate every piece of UI currently in the game. For each, report:
- Name + type (ScreenGui / Frame / custom controller), and where it lives (StarterGui? a Client
  controller under StarterPlayerScripts? created at runtime? paste the path).
- What it shows / does (HUD element, inventory, build menu, character info, settings, etc.).
- Whether it's currently wired to real data or is a static/placeholder mockup.
List the Client UI controllers specifically (e.g. HUDController, InventoryController, WorldController —
whatever exists) and what each renders.

## Part B — The overlap / usability problems
Jon reports menus overlap and look plain. Concretely diagnose:
- Which UI elements overlap or collide, and under what conditions (which panels open at once, on what
  screen sizes). Screenshot the overlaps if you can reproduce them in Play.
- Is there ANY existing layer/z-order/one-panel-at-a-time management, or does each screen position
  itself independently? (This is the root-cause question.)
- How are panels opened/closed today (toggles, buttons, keybinds)? Can multiple modals be open at once?

## Part C — Responsiveness & platform
- Do current UIs use scale-based sizing (UDim scale, AspectRatioConstraints) or fixed pixel offsets?
  Fixed offsets are the thing that breaks on phones — report which elements use what.
- Is there any platform detection in the UI today? (Contract §11.1 wants Client/Platform.luau +
  Platform.Changed; discovery earlier noted the client still uses GameConfig.PLATFORM — confirm current
  state.)
- Note anything that assumes desktop (hover-only interactions, tiny tap targets < 44px, etc.).

## Part D — Salvage assessment
For each existing UI piece, a one-line verdict: KEEP (works, conforms), RESKIN (logic fine, needs
Clean Fantasy styling), REBUILD (replace on the new framework), or DELETE (dead/unused). This tells us
what the framework inherits vs. replaces.

## Part E — Visual/token inventory
- What colors, fonts, and sizing are used today (so we know the gap to the Clean Fantasy palette:
  dark panels ~0.4 transparency, weathered-gold trim ~ (0.83,0.68,0.22), one display + one body font).
- Any existing reusable UI components (a button style, a panel frame) worth formalizing into the kit?

## Report back (write outbox/ui-audit-01.result.md)
- Part A: the UI inventory (table preferred: name / type / location / shows / wired-or-mockup).
- Part B: overlap diagnosis + whether any layer management exists (the key finding).
- Part C: scale-vs-fixed sizing per element + platform-detection state.
- Part D: KEEP/RESKIN/REBUILD/DELETE verdict per piece.
- Part E: current colors/fonts/sizes + any salvageable components.
- STATUS: DONE / PARTIAL / BLOCKED + anything unexpected.
- Attach or reference screenshots of the current HUD and any overlapping menus if capturable.

## Guardrails
- READ-ONLY. Do not modify, restyle, move, or delete any UI — diagnose only.
- rojo serve + Connect only if Studio interaction is needed — NEVER rojo build.
- No secrets in the result (outbox is public).
