TASK: framework-core-01 — UI framework plumbing (layer manager + component kit + Platform.luau)

FROM: Studio Director
DATE: 2026-07-06
SCOPE: GAME project. Builds the FOUNDATION the HUD/inventory/panels will sit on. Pure plumbing +
       a throwaway demo. Do NOT rebuild the real HUD or any real panel here — that's the fan-out.
DEPENDS ON: ui-audit-01 (kit primitives + palette), contract v2.2.0 §11 (Clean Fantasy, layer model,
       §11.11 one-modal-at-a-time, §11.1 Platform.luau).
ENV: sync stable on Host 127.0.0.1; rojo serve already running on 127.0.0.1:34872 — do NOT start a
       second instance. rojo serve + Connect only, NEVER rojo build.

## Goal
A thin, custom UI framework — owned, not an external dependency — that every future panel uses:
1. A layer manager that structurally prevents overlap (the audit found NO layer/z-order/modal
   management today — that's the root cause of the overlap, not styling).
2. A Clean Fantasy component kit of the 5 primitives the audit identified.
3. Platform.luau per §11.1, replacing the boot-time GameConfig.PLATFORM read.
Prove it all with a throwaway demo. Keep it isolated from the real HUD/controllers.

## Part 1 — Layer manager
Implement a client UI layer manager with the three §11.2 layers:
- PERSISTENT (HUD-class, always visible), MODAL (panels), TRANSIENT (tooltips/toasts/combat text).
- Z-order: transient above modal above persistent.
- **§11.11 one-modal-at-a-time rule (the anti-overlap fix):** opening a modal panel automatically
  closes any other open modal. Provide an API like openModal(panel) / closeModal() / closeAllModals(),
  and a scrim/dim behind the active modal. Two modals must NOT be openable at once.
- Persistent and transient layers are unaffected by modal open/close.
Report the API surface (function signatures).

## Part 2 — Clean Fantasy component kit (5 primitives)
Build these as reusable, themeable components (the audit flagged all 5 as salvageable/needed):
- **Bar** (health/XP/progress — fill %, label, color).
- **Label / Row** (a titled value row, e.g. "Gold: 100").
- **Panel** (the modal frame — Clean Fantasy styling, header w/ title + close X, content area).
- **Button** (primary/secondary, with §11.7 hover/press/disabled states).
- **Toast** (transient notification).
Styling — per §11.3 / audit Part E:
- Dark semi-transparent panels (~Color3(0,0,0), Transparency ~0.4), weathered-gold trim
  **(212,173,56)** applied CONSISTENTLY (audit found gold inconsistent across 3 values today).
- Centralize palette + spacing + font tokens in ONE module (a theme table) — no inline colors.
  One display font (headers) + one body font (values). (Audit: Gotham-only today; pick the display/body
  split and put it in the theme.)
- **Scale-based sizing (UDim scale + AspectRatio/UIScale where needed), NOT fixed pixels** — the audit
  found almost everything fixed-pixel, which breaks on phones. All primitives must be responsive.
- Tap targets >= 44px (§11.4). Buttons/interactive elements meet the touch minimum.

## Part 3 — Platform.luau (§11.1)
- New `Client/Platform.luau`: `Platform.Get() -> "Touch"|"Mouse"|"Gamepad"`, `HasTouch()`,
  `HasGamepad()`, and a `Platform.Changed` signal that fires when the primary input changes.
- Detect per-input at runtime (not a boot-time cache); do NOT rely on GameConfig.PLATFORM.
- The kit/demo should be able to subscribe to Platform.Changed and re-layout. (Refactoring the existing
  client OFF GameConfig.PLATFORM everywhere is a separate follow-up — here, just provide Platform.luau
  and have the demo use it. Note the remaining GameConfig.PLATFORM usages for that follow-up.)

## Part 4 — Throwaway demo (proof, then removable)
A small demo (behind a temp keybind or a clearly-named demo script) that proves the framework:
- Renders one Bar, one Label/Row, one Button, and can raise a Toast — all from the kit, themed.
- Opens two different demo Panels via the layer manager and shows that opening the 2nd CLOSES the 1st
  (one-modal-at-a-time working) with the scrim behind.
- Resizes/reflows sensibly (scale-based) — sanity check at a phone-ish viewport.
- Clearly marked as throwaway/demo so it's trivially removed before the real fan-out. Do NOT wire it
  into ClientBootstrap's real controller set.

## Rider — DaneAxe harvest (one-line, confirmed with Director)
- Set `CanHarvest = true` on the **DaneAxe** in Weapons.luau (BeardedAxe already harvests; DaneAxe is an
  axe and should too). Do NOT touch the Wood/Iron/Steel tier axes (deferred to the progression pass) or
  any non-axe weapon. Verify DaneAxe can chop a tree in Play (hit-to-fell, same as BeardedAxe).

## Report back (write outbox/framework-core-01.result.md)
- Layer manager API + confirmation two modals can't coexist (one-modal-at-a-time verified) + scrim works.
- The 5 kit primitives built + the central theme module (palette/spacing/fonts). Confirm gold is the
  single (212,173,56) everywhere and sizing is scale-based (responsive), tap targets >= 44px.
- Platform.luau API + Changed signal working; list remaining GameConfig.PLATFORM usages for follow-up.
- Demo proof: two panels, opening 2nd closes 1st; toast raised; reflow at phone viewport. Confirm demo
  is isolated/removable and NOT wired into the real boot.
- Rider: DaneAxe CanHarvest=true, verified chopping a tree.
- STATUS: DONE / PARTIAL / BLOCKED + anything unexpected.

## Guardrails
- rojo serve + Connect only (already running on 127.0.0.1:34872 — no 2nd instance). NEVER rojo build.
- Framework plumbing ONLY — do NOT rebuild CombatHUD, Inventory, Build, or any real panel; do NOT restyle
  existing UI. Those are the fan-out tasks after this lands.
- Everything in canonical src (framework code is under Client -> mapped/synced). No place-only drift.
- No secrets in the result (outbox is public).
