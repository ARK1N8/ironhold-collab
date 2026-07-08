# RESULT: framework-core-01 — UI framework plumbing (layers + kit + Platform.luau)

FROM: Claude Code (local executor)
DATE: 2026-07-07
STATUS: **DONE** — layer manager + 5-primitive Clean Fantasy kit + Platform.luau built and verified;
throwaway demo isolated & removable; DaneAxe rider done. All in canonical src, Rojo-synced to live.

---

## Files added (all under `src/Client`, Rojo-mapped → StarterPlayerScripts.Client)
```
Client/Platform.luau                     -- §11.1 runtime input + Changed signal
Client/UI/Theme.luau                     -- single source of tokens (palette/fonts/spacing)
Client/UI/LayerManager.luau              -- §11.2/§11.11 3 layers + one-modal + scrim
Client/UI/Kit.luau                       -- single require entry point for the framework
Client/UI/Components/Bar.luau            -- health/XP/progress bar
Client/UI/Components/Label.luau          -- titled value row ("Gold: 100")
Client/UI/Components/Panel.luau          -- Clean Fantasy modal frame (header + close X + content)
Client/UI/Components/Button.luau         -- primary/secondary + hover/press/disabled
Client/UI/Components/Toast.luau          -- transient notification
Client/UIFrameworkDemo.client.luau       -- THROWAWAY demo (F8), NOT wired into ClientBootstrap
```
Nothing touched in CombatHUD / Inventory / Build / any real controller. No place-only drift — everything
is in src and the connected Rojo session pushed it to live (verified the full tree present live).

## Part 1 — Layer manager (verified)
Three `ScreenGui` layers by `DisplayOrder`: **PERSISTENT=10, MODAL=20, TRANSIENT=30**
(transient > modal > persistent). API:
```
LayerManager.Init() -> self                 (idempotent; find-or-create so layers are never duplicated)
LayerManager.GetLayer(name) -> ScreenGui
LayerManager.AddPersistent(gui) -> gui
LayerManager.AddTransient(gui) -> gui
LayerManager.OpenModal(panel) -> panel      (closes any other open modal; shows scrim)
LayerManager.CloseModal(panel?)
LayerManager.CloseAllModals()
LayerManager.GetActiveModal() -> GuiObject?
```
**§11.11 one-modal-at-a-time — verified in Play:**
```
unique layers: PERSISTENT=1 MODAL=1 TRANSIENT=1     scrim count: 1
open Panel A:  visibleModals=1  scrim.Visible=true  scrim.T=0.45 (dim)
open Panel B:  visibleModals=1 (still 1)  A.Visible=false  B.Visible=true  active==B  scrim.Visible=true
closeAll:      active=nil  scrim.Visible=false
```
Opening a 2nd modal auto-closes the 1st (two modals can never coexist); a scrim dims behind the active
modal and is `Active=true` so it absorbs input to the PERSISTENT layer beneath. PERSISTENT and TRANSIENT
are unaffected by modal open/close. (During testing I hit a duplicate-layer artifact from Init running in
two contexts; hardened `Init`/scrim to **find-or-create**, after which layer/scrim counts are exactly 1.)

## Part 2 — Clean Fantasy component kit (5 primitives, verified)
Bar, Label/Row, Panel, Button, Toast — all themeable, all reading from one **Theme** module:
- **Gold is the single value everywhere:** `Color3.fromRGB(212,173,56)`. Play check swept every trim
  element (UIStroke + header Divider) across Panel/Bar/Button → **0 non-gold** (fixes the audit's 3-golds).
- **Dark semi-transparent panels:** `PanelBG = (0,0,0)` at `Transparency = 0.4` (§11.3); scrim dim 0.45.
- **Fonts:** one **display** (Garamond, headers) + one **body** (Gotham, values/buttons) — was Gotham-only.
- **Scale-based / responsive:** primitives size via UDim **scale** (e.g. Panel `Size=(0.55,0.60)scale, 0px
  offset`) + `UIAspectRatioConstraint` + `UISizeConstraint` clamps — no fixed-pixel layout (the audit's
  phone-breaking issue). Bar/Row use `TextScaled`.
- **Tap targets ≥ 44px (§11.4):** Button and Panel close-X carry a `UISizeConstraint` MinSize of 44 —
  verified Button tap-min = 44.
- **Button states (§11.7):** hover (lighten), press (darken), disabled (dim + non-interactive) via
  `SetEnabled`.

## Part 3 — Platform.luau (§11.1, verified)
`Client/Platform.luau`: `Platform.Get() -> "Touch"|"Mouse"|"Gamepad"`, `Platform.HasTouch()`,
`Platform.HasGamepad()`, and `Platform.Changed` (fires with the new platform when the primary input
changes). Detection is **runtime per-input** via `UserInputService.LastInputTypeChanged` (with a
capability-based initial guess) — **not** a boot-time `GameConfig.PLATFORM` cache. The demo subscribes to
`Platform.Changed` and live-updates its "Input:" row. Play check: `Platform.Get()="Mouse"`.

**Remaining `GameConfig.PLATFORM` usages to migrate off (separate follow-up, per task):**
- `ClientBootstrap.luau:15` (writes `GameConfig.PLATFORM`)
- `BuildController.luau:199, 485, 580`
- `CombatController.luau:191, 192, 445, 446`
These should later read `Platform.Get()` / react to `Platform.Changed`; out of scope here.

## Part 4 — Throwaway demo (verified, isolated)
`UIFrameworkDemo.client.luau` — **press F8** to toggle a demo dashboard. It is fully **inert until F8**
(only a keybind listener + a load print; `LayerManager.Init()` is lazy inside the first open) and is **not
required by ClientBootstrap** — deleting the one file removes it entirely. It renders a themed Bar
(health), a Label/Row ("Gold: 100"), a "Raise Toast" Button (raises a kit Toast in TRANSIENT), and
"Open Panel A"/"Open Panel B" buttons that prove one-modal-at-a-time via the layer manager (opening B
closes A behind the scrim). All widgets are scale-based, so they reflow at a phone-ish viewport
(verified sizing is scale/AspectRatio, 0px offsets; no fixed-pixel layout to break).

## Rider — DaneAxe harvest (verified)
`Weapons.luau` DaneAxe `CanHarvest = false → true` (only DaneAxe; Wood/Iron/Steel tier axes and non-axe
weapons untouched). Play check (server-side hit-to-fell, reusing the tier1-handlers path):
```
DaneAxe: Damage=30 CanHarvest=true
tree MaxHealth=100  hits: 70 -> 40 -> 10 -> 0  (FELLED on hit 4)   wood +3
```

## VERIFY (evidence)
- Live tree: all 10 modules present under StarterPlayerScripts.Client (synced by Rojo, Host 127.0.0.1).
- Layer/one-modal/scrim/gold/scale/tap/platform: the Play readouts quoted above.
- DaneAxe: the 4-hit fell trace above.
- Guardrails: no `rojo build`; no 2nd rojo serve; no real HUD/panel rebuilt or restyled; demo isolated &
  removable; all code in canonical src (no place-only drift); no secrets.

## STATUS: DONE

## BACK TO DIRECTOR
1. **Framework is ready for the fan-out.** Real panels do: `local Kit = require(Client.UI.Kit)`, build
   from `Kit.Panel/Bar/Button/Label/Toast`, and open via `Kit.LayerManager.OpenModal(...)`.
2. **Follow-up: migrate controllers off `GameConfig.PLATFORM`** to `Platform.Get()`/`Platform.Changed`
   (8 usages listed above) and remove the boot-time write in ClientBootstrap. Separate task.
3. **Delete the demo** (`UIFrameworkDemo.client.luau`) before/at the start of the real UI fan-out — it's a
   single self-contained file.
4. When the real HUD is rebuilt on this kit, that's the moment to retire the ad-hoc CombatHUD builder
   (audit Part D) and route its bars/labels through `Kit.Bar`/`Kit.Label` on the PERSISTENT layer.
