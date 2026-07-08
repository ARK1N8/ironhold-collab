# RESULT: ui-audit-01 — UI/HUD inventory before the framework

FROM: Claude Code (local executor)
DATE: 2026-07-06
STATUS: **DONE** — full read-only audit. Nothing modified. One headline defect found (below).

---

## ⭐ HEADLINE FINDING (read this first)
**The client boots TWICE.** There are two enabled bootstrap LocalScripts in `StarterPlayerScripts`, and
they both initialise the controllers:

| Bootstrap | Location | Inits | HUDController? |
|---|---|---|---|
| `IronholdClient` | `StarterPlayerScripts.IronholdClient` (place-only, not in Rojo src) | HUD, Combat, World, Inventory, Build | **YES** |
| `ClientMain` → `ClientBootstrap.Init()` | `StarterPlayerScripts.Client.ClientMain` (Rojo `src/Client`) | World, Combat, Build, Inventory | no |

Both are `Enabled=true, Disabled=false` (verified via inspect_instance). Consequences:
- **Every controller's `Init()` runs twice** (except HUDController once). Proven at runtime: the live
  PlayerGui contained **two `Vignette` ImageLabels** (CombatController.createVignette ran twice) — see
  runtime dump below.
- Input handlers, Net listeners, and some GUIs are **double-bound** → attacks/interacts can fire twice,
  toasts can appear twice.
- **Three** overlapping HUD systems draw the same data top-left: `HUDGui` (HUDController) + `CombatHUD`
  (CombatController), on top of each other, plus each duplicates gold/HP/XP/level.

This — not styling — is the primary cause of "menus overlap." **Recommend a dedicated fix task to delete
one bootstrap and pick a single HUD owner** (details in BACK TO DIRECTOR). It is out of scope here
(read-only audit), so I did not touch it.

Runtime dump of `PlayerGui` in Play (goblin area spawn):
```
Freecam      [ScreenGui]  DisplayOrder=0   (Studio test-only, not game UI)
HUDGui       [ScreenGui]  DisplayOrder=5   {GoldLabel, HPBG, XPBG, LevelLabel}
CombatHUD    [ScreenGui]  DisplayOrder=0   {HealthFrame, XPFrame, LevelLabel, GoldLabel, ATKButton, ACTButton}
InventoryGui [ScreenGui]  DisplayOrder=0   {ToggleInventory, Panel}
Vignette     [ImageLabel] x2  <-- duplicate; proves double CombatController.Init()
```
Screenshot captured in Play (top-left): "Gold: 100" label sitting **on top of** a green health frame,
a red "100/100" HP bar + blue XP bar stacked below, and **two "Lv 1" indicators** (one gold, one purple
chip) — the literal triple-HUD overlap. Also visible: Roblox **default backpack hotbar** at bottom
(Seax/BeardedAxe/DaneAxe/Spear/HuntingBow) and a "Goblin" world-space nameplate.

---

## Part A — UI inventory
All game UI is **created at runtime from Luau**. `StarterGui` is **completely empty** (0 children) —
there are no baked ScreenGuis to inherit. Client UI lives in
`StarterPlayerScripts.Client.Controllers.*` (Rojo `src/Client/Controllers/`).

| # | Name (runtime) | Type | Created by / location | Shows / does | Wired to data? |
|---|---|---|---|---|---|
| 1 | `HUDGui` (DisplayOrder 5) | ScreenGui, code-built | `HUDController.Init()` | Gold label, HP bar, XP bar, Level chip — top-left | **Wired** (GoldChanged, LevelUp, Humanoid.HealthChanged, GetInventory seed) |
| 2 | `CombatHUD` | ScreenGui, code-built | `CombatController.buildHUD()` | HP bar, XP bar, Level, Gold (top-right), ATK/ACT buttons (mobile/console) | **Wired** (same events + Attack, vignette, dmg numbers) |
| 3 | `Vignette` | ImageLabel (loose in PlayerGui) | `CombatController.createVignette()` | Red full-screen damage flash | **Wired** (HealthChanged) — **duplicated x2** |
| 4 | `InventoryGui` | ScreenGui, code-built | `InventoryController.buildGui()` | Toggle button (top-right) + slide panel: level, TH level, equipped weapon, materials list, structures list | **Wired** (GetInventory, InventoryChanged) |
| 5 | `BuildHUD` | ScreenGui, code-built | `BuildController.buildHUDButton()` | Bottom-center toolbar: "🏗 Build" + "↻ Rotate" buttons | **Wired** (toggles build mode) |
| 6 | `BuildPalette` | ScreenGui (Enabled=false until build) | `BuildController.buildPaletteUI()` | Bottom full-width material palette + rotation label; grid overlay + ghost part in Workspace | **Wired** (owned materials, PlaceBlock) |
| 7 | `WorldGui` | ScreenGui, code-built | `WorldController` | Transient top-center toasts: "Area loaded (N chunks)", "Plot claimed: id", "Tree respawned" | **Wired** (ChunksLoaded, PlotClaimed, TreeRespawned) |
| 8 | Enemy `HealthBar` + `NameTag` | BillboardGui (world-space) | `NPCService` (server), on each NPC model | Enemy HP fill + name over NPCs | **Wired** (server) |
| 9 | Default **Backpack hotbar** | Roblox CoreGui | engine default (tools in StarterPack) | Weapon slots 1–5 | engine |

**Client UI controllers:** `HUDController` (bars HUD), `CombatController` (HUD + combat VFX/juice +
input), `InventoryController` (inventory panel), `BuildController` (build toolbar + palette + ghost),
`WorldController` (toasts). `ClientBootstrap` = intended single entry point; `ClientMain` runs it.
`IronholdClient` = a **second, redundant** entry point (the bug above). `ShieldToggleClient` = input only
(Q → ShieldToggle), no GUI.

## Part B — Overlap diagnosis + layer management (the key finding)
- **Is there any layer / z-order / one-panel-at-a-time manager? NO.** Every screen is its own `ScreenGui`
  parented straight to `PlayerGui`, positioning itself with absolute offsets. Ordering is ad-hoc: only
  `HUDGui` sets `DisplayOrder=5`; everything else is `DisplayOrder=0` and relies on default stacking.
  Three ScreenGuis (`BuildHUD`, `BuildPalette`, `CombatHUD`) set `ZIndexBehavior=Sibling` individually;
  the rest don't. There is **no shared layer constant, no modal stack, no "close others when I open."**
- **What overlaps, and when:**
  - **Always:** `HUDGui` (top-left HP/XP/Gold/Level) is drawn directly over `CombatHUD`'s top-left
    HP/XP/Level — same corner, slightly different Y (HUDGui HP at y=44 vs CombatHUD health at y=12).
    Two gold readouts (HUDGui top-left "Gold: N" + CombatHUD top-right "🪙 N") and **two "Lv 1"** labels.
  - **Top-right collision:** `InventoryGui` toggle button `Position=(1,-130),(0,10)` size 120×36 overlaps
    `CombatHUD` GoldLabel `Position=(1,-134),(0,12)` size 120×28 — same top-right box.
  - **Multiple modals can be open at once:** the inventory `Panel` (right side) and the build `Palette`
    (bottom) have independent toggles (Inventory button; B key / Build button) with **no mutual
    exclusion** — both can be visible together, and combat input stays live underneath. This is exactly
    the §11.11 "one modal at a time" rule not being enforced.
- **How panels open/close today:** Inventory = click toggle button (`panel.Visible = not visible`).
  Build = `B` key, Build button, or Escape to exit; palette shown via `sg.Enabled`. Shield = `Q`.
  Toasts = auto show/hide on Net events. No central router; each controller owns its own visibility.
- **Root cause statement:** overlap = (a) the double-bootstrap tripling the HUD, and (b) absence of any
  layer/anchor system — each panel hard-codes a screen corner with no knowledge of the others.

## Part C — Responsiveness & platform
- **Sizing is almost entirely FIXED PIXEL OFFSETS** (`UDim2.new(0, px, 0, px)`), which is the thing that
  breaks on phones. Per element:
  - **Fixed offset (breaks on small screens):** HUDGui GoldLabel 160×28, HP/XP bars 200×18, Level chip
    90×22; CombatHUD HealthFrame 240×28, XPFrame 240×14, GoldLabel 120×28, ATK 72×72, ACT 56×56;
    Inventory toggle 120×36 and Panel 320×480 (fixed, anchored to right); Build toolbar 220×48 and
    buttons 90×36, palette material buttons 80×80; all WorldController toasts (240×36 etc.).
  - **Scale-based (OK):** CombatController `Vignette` `fromScale(1,1)`, LevelUp flash `fromScale(1,1)`,
    Build palette `panel` width `(1,0,...)` (full width) — height still fixed 110. Bar **fills** use
    `fromScale(ratio,1)` but that's data, not layout.
  - **No `UIAspectRatioConstraint`, no `UIScale`, no `UISizeConstraint` anywhere.** Nothing adapts to
    viewport size.
- **Platform detection state:** still the **old model** — `ClientBootstrap.Init()` sets
  `GameConfig.PLATFORM` ("MOBILE"/"CONSOLE"/"DESKTOP") from `UIS` boolean flags, and controllers read
  `GameConfig.PLATFORM`. **There is NO `Client/Platform.luau` and NO `Platform.Changed`** signal — the
  contract §11.1 target is not implemented. Detection runs **once at boot**; plugging in a controller or
  switching input mid-session won't update any UI. (Note: `IronholdClient` doesn't set PLATFORM at all —
  it relies on `ClientMain`'s bootstrap having run first; another fragility of the double-boot.)
- **Desktop assumptions / touch issues:** ATK/ACT buttons are hidden unless MOBILE/CONSOLE, so on a
  touch device that reports DESKTOP there'd be no attack button. Tap targets **below 44px**: Inventory
  toggle (36 high), Build/Rotate buttons (36 high), XP bar (14 high). Build/attack rely on
  `mouse.X/mouse.Y` in several spots even in shared code paths. No hover-only traps beyond button color
  changes, but nothing is sized for a phone.

## Part D — Salvage verdict (per piece)
| Piece | Verdict | Rationale |
|---|---|---|
| `IronholdClient` bootstrap | **DELETE** | Redundant second entry point; root of the double-init. Keep only `ClientMain`→`ClientBootstrap`. |
| `HUDController` (HUDGui) | **DELETE** (merge into one HUD) | 100% duplicates CombatHUD's gold/HP/XP/level. Pick ONE HUD owner; this is the weaker one (no juice). Keep its clean `makeBar` idea. |
| `CombatHUD` (bars + buttons) | **REBUILD** | Logic/juice is good and worth keeping, but rebuild on the framework so bars/buttons come from the kit and sit on managed layers. Combat **VFX/juice** (dmg numbers, hit VFX, vignette, hitstop, level-up flash) = **KEEP** (move to an effects module, not a HUD concern). |
| `InventoryController` panel | **REBUILD** | Data wiring (GetInventory/InventoryChanged) is fine; the panel is plain (flat rows, no styling) and self-positions. Rebuild as a managed modal on the kit. Keep the snapshot-render logic. |
| `BuildController` toolbar + palette | **RESKIN + rehome** | Interaction (ghost, grid, place, rotate, palette) is solid — **KEEP the logic**. RESKIN the toolbar/palette to Clean Fantasy and register them as managed layers so they stop free-floating. |
| `WorldController` toasts | **RESKIN** | Good pattern (transient notifications); logic KEEP. Should become the framework's single "toast/notification" component instead of three bespoke labels. |
| Enemy HealthBar / NameTag (NPCService) | **RESKIN** (later) | World-space, doesn't overlap screen UI; fine for now, restyle to match palette eventually. |
| Default Backpack hotbar | **KEEP** (decide) | Engine default. Works, but clashes visually with Clean Fantasy — decide later whether to replace with a custom hotbar. |
| `Vignette` (duplicate) | **REBUILD** | Keep one, as an effects-layer element; the duplication vanishes once double-init is fixed. |

## Part E — Visual / token inventory (gap to Clean Fantasy)
- **Fonts:** `Gotham` / `GothamBold` **only**, everywhere. No display font. Clean Fantasy wants **one
  display + one body** font — so we need to introduce a display face and standardise body. Sizes are a
  mix of `TextScaled=true` (bars, labels) and hard `TextSize` (12–14 on build/palette) — inconsistent.
- **Panel backgrounds:** dark, RGB ~15–40 (e.g. `(20,20,30)`, `(30,30,50)`, `(15,15,25)`),
  `BackgroundTransparency` ranging 0.2–0.4 depending on the author. Target is "dark panels ~0.4
  transparency" — current is in the ballpark but **not standardised** (every panel picks its own).
- **Accent / gold:** inconsistent — `(255,220,60)`, `(255,210,60)`, `(255,255,100)` used for gold/level
  across controllers. Clean Fantasy weathered-gold target ≈ `(212,173,56)` = `(0.83,0.68,0.22)`. So the
  gold needs to be **darkened/desaturated** and unified to one token.
- **Other colors (all inline literals, no palette module):** health fills disagree — HUDController HP =
  red `(200,50,50)`, CombatController health = green `(60,200,60)`; XP blue `(80,120,255)` vs
  `(60,140,220)`; build blue `(50,100,200)`/`(80,160,255)`; ghost green/red `(100,220,100)`/`(220,60,60)`.
  Corner radius mostly `UDim.new(0,4..10)` via `UICorner`.
- **Salvageable components worth formalising into the kit:**
  - `makeBar(parent,name,color,y)` (HUDController) — a labeled progress bar; good basis for a **Bar**
    component (health/XP).
  - `makeLabel(name,parent)` (InventoryController) and `makeRowLabel` — a **list-row/label** primitive.
  - The repeated **Frame + UICorner + BackgroundTransparency** idiom → a single **Panel** component.
  - The repeated **TextButton + UICorner + hover color** idiom (build tabs, palette, inventory toggle) →
    one **Button** component.
  - WorldController's transient-label pattern → one **Toast** component.
  These five (Bar, Label/Row, Panel, Button, Toast) are exactly what the Clean Fantasy kit should own, so
  no controller hand-rolls Instances again.

## VERIFY (evidence)
- Read all 5 controllers + `ClientBootstrap` + `ClientMain` on disk (`src/Client/...`).
- `search_game_tree StarterGui` → single empty `StarterGui` node (0 children). `StarterPlayerScripts` →
  `IronholdClient` (LocalScript), `Client/` folder (ClientBootstrap, ClientMain, 5 controllers),
  `ShieldToggleClient`.
- `inspect_instance` on both bootstraps → both `Enabled=true, Disabled=false`; captured their full source.
- **Play test:** started Play, dumped `PlayerGui` → `HUDGui, CombatHUD, InventoryGui, Vignette×2,
  Freecam`; screenshot shows the top-left triple-HUD overlap, default hotbar, and "Goblin" nameplate.
  Stopped Play. (Console showed only pre-existing server warnings: DataStore-unavailable, a broken
  campfire fire asset `134278420263483`, and IronholdDoorService `GetDebugId` plugin-capability warnings
  — all unrelated to UI.)
- Guardrails honoured: **read-only** — no UI created/moved/deleted/restyled; **no `rojo build`**; Play
  used only for observation then stopped; no secrets in this file.

## STATUS: DONE

## BACK TO DIRECTOR
1. **Fix the double-bootstrap before/with the framework (highest priority).** Recommended: **delete
   `StarterPlayerScripts.IronholdClient`** and keep the Rojo-managed `ClientMain`→`ClientBootstrap` as the
   single entry point. But note `ClientBootstrap` currently does **not** init `HUDController` — so decide
   the single HUD owner first (see #2), then make the one surviving bootstrap init exactly the intended
   set once. This is a small, high-impact task; I left it for a dedicated fix task since this was read-only.
2. **Pick ONE HUD.** `HUDGui` and `CombatHUD` render the same gold/HP/XP/level. Recommend keeping
   CombatController's version (it has the juice) and **deleting HUDController**, or — cleaner for the
   framework — extracting HUD into a single new HUD layer and deleting both ad-hoc builders.
3. **The framework must own layering + one-modal enforcement.** Root cause of overlap is that every
   ScreenGui self-positions with no z-order/modal manager. The layer manager (with a DisplayOrder/modal
   stack and "open closes others") directly fixes §11.11 and the top-right Inventory/Gold collision.
4. **Responsiveness is a near-total gap:** almost everything is fixed-pixel with no AspectRatio/UIScale.
   The kit should default components to scale-based sizing + AspectRatioConstraints, and fix sub-44px tap
   targets (Inventory toggle, Build/Rotate, XP bar).
5. **Platform §11.1 still not implemented:** no `Client/Platform.luau`, no `Platform.Changed`; UI reads
   `GameConfig.PLATFORM` set once at boot. Worth a task to add Platform.luau and make HUD react to
   `Platform.Changed` (also removes IronholdClient's reliance on boot order for PLATFORM).
6. **Tokens are all inline literals** with real inconsistencies (two different HP colors, three golds,
   varying panel transparency). A shared palette/typography module is a prerequisite for the kit; gold in
   particular must move from `(255,220,60)` to the weathered `(212,173,56)`.
