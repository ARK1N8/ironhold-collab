# RESULT: inventory-panel-01 — tabbed inventory panel (first real feature on the framework)

FROM: Claude Code (local executor)
DATE: 2026-07-09
STATUS: **DONE** — demo removed; tabbed inventory built on the Kit, opened as a managed modal, wired to
live `GetInventory`, verified in Play. Framework held up; two components added to the Kit; one real API
rough edge found (and fixed in the panel) that the fan-out must know about.

---

## Step 0 — demo removed
`UIFrameworkDemo.client.luau` deleted (disk + synced to live). Verified in Play:
`UIFrameworkDemo present: false` — the F8 keybind no longer exists and nothing referenced it.

## What was built
```
Client/UI/Components/TabBar.luau      (NEW kit component)
Client/UI/Components/ScrollList.luau  (NEW kit component)
Client/UI/Kit.luau                    (registers TabBar + ScrollList)
Client/Panels/InventoryPanel.luau     (the feature: 4 tabs, modal, pure view)
Client/InventoryPanelInit.client.luau (isolated open-trigger: I toggles, Esc closes)
```
`InventoryPanel` API: `Init()`, `Open()`, `Close()`, `Toggle()`, `SelectTab(i)` (so a future HUD button
can open straight to a tab). Not wired into ClientBootstrap; no other controller touched.

## Panel — verified in Play
Opened as a **MODAL via the layer manager** (not a free-floating ScreenGui):
```
modal: Visible=true  scrim.Visible=true  scrim.T=0.45  title='Inventory'
after opening a 2nd modal: inventory.Visible=false   visibleModals=1   active==other:true
closeAll: active=nil  scrim.Visible=false
after IP.Close(): inventory.Visible=false  scrim.Visible=false
```
§11.11 one-modal-at-a-time holds for a real panel. Closes via the Panel **X**, **Esc**, `Close()`, and
`LayerManager.CloseModal`.

## Live data — verified (all four tabs)
Seeded server-side, then read the rendered rows:
```
tab 1 Structures  rows=2 [ Cannon x2 | Town Hall x2 ]
tab 2 Weapons     rows=2 [ BeardedAxe Equipped | Seax Owned ]
tab 3 Armor       rows=0 EMPTY:'No armor yet.'
tab 4 Resources   rows=2 [ Stone x3 | Wood x7 ]
```
Pure view (§11.6): it renders the `GetInventory` snapshot and never mutates inventory.

### Snapshot structure + how items were bucketed
`GetInventory` returns:
`{ weapons = {[id]={owned,equipped,def}}, defense = {[id]={quantity,def}}, buildings = {[id]={quantity,def}},
materials = {[id]={quantity,record}}, consumables = {}, townHall = {level,placed,levelCap} }`

| Tab | Source | Notes |
|---|---|---|
| Structures | `defense` + `buildings` (quantity > 0) | merged; `def.DisplayName` used ("Town Hall") |
| Weapons | `weapons` where `owned` | shows `Equipped` / `Owned` |
| Armor | **nothing** | **GAP: the snapshot has no armor bucket** |
| Resources | `materials` (quantity > 0) | `record.DisplayName` → id fallback |

**Gap reported, not invented:** there is **no armor** anywhere in the snapshot (and `consumables` is always
empty). I did not fabricate a category — the Armor tab renders its empty state until an armor system
exists (§12). Also note `buildings` was empty in test data because both granted structures
(TownHall, Cannon) are `Category = TownHall/Defense`.

**Empty-state handling:** each tab shows a themed message ("No structures yet." / "No armor yet." …), and
"Loading..." until the first snapshot lands.

## Responsive + touch — verified
```
panel Size scale=(0.55,0.65) offset=(0,0)     [scale, not fixed px]
UIAspectRatioConstraint=1.35 (DominantAxis=Height)   UISizeConstraint Min=240,200  Max=720,900
row Size scale=(1.00,0.00) offset=(0,34)  AbsWidth=472 fills list width=472
ScrollList AutomaticCanvasSize=Enum.AutomaticSize.Y
tap targets: close=44x44   tab=97x44   (both >= 44px, §11.4)
viewport 1058x566 -> panel 496x367 (47% x 65%)
```
Sizing is scale + aspect + min/max clamps, so it reflows rather than breaking; rows fill list width. I
could not resize the Studio viewport to a literal phone size from the harness, so the phone check is by
construction + constraints rather than a device screenshot.

---

## FRAMEWORK STRESS-TEST (the point of doing inventory solo first)

**Kit changes made — 2 new components, both in the Kit (nothing inline):**
1. **`TabBar`** — horizontal tab switcher, `{tabs, onChange}` → `SelectTab(i)/GetSelected()`. Active tab =
   gold text + gold underline; ≥44px tap height.
2. **`ScrollList`** — scrolling container with `AutomaticCanvasSize`, gold scrollbar, `AddRow/Clear/
   SetEmpty/Count`, and a built-in empty state.
Both were genuinely missing; everything else (Panel, Label/Row) covered the need as-is.

**Theme:** covered **100% of styling — zero inline values needed.** TabBar/ScrollList reuse existing
tokens (`Gold`, `TextMuted`, `ButtonSecondary`, `Spacing`, `Font`, `TapMinPx`). No new tokens required.

**Layer manager:** worked for a real panel — `OpenModal` parents + shows scrim, opening a second modal
auto-closed inventory, `CloseModal/CloseAllModals` clean up the scrim. No changes needed.

**API rough edges the fan-out must know:**
1. ⚠️ **Never do a blocking `InvokeServer` in a panel's open path.** My first version fetched
   synchronously inside `Open()`; if the panel opens before the server binds `GetInventory` (boot race),
   `InvokeServer` **yields forever** and freezes the client. Fixed: `refreshData()` now runs in
   `task.spawn`, the list shows "Loading...", then re-renders. **Every panel should fetch async.**
2. `Kit.Label` rows default to zero height — a caller must set a row height (I use 34px) before adding to
   a `ScrollList`. Worth documenting or defaulting.
3. `LayerManager.OpenModal(panel.Instance)` takes the **Instance**, not the Kit handle — easy to trip on.
4. Panels are built once and reused (`Init()` guard); `Open()` re-fetches. That pattern works well.

**Coexistence note (not touched, per guardrails):** the legacy `InventoryController` still builds its own
`InventoryGui` + top-right "Inventory" button. It's independent of this panel (different trigger, and mine
is layer-managed). Retiring it belongs to the HUD fan-out.

## Unexpected — harness artifact worth knowing
Much of my debugging time went into a false trail: **`OnServerInvoke` handlers rebound from the MCP
`execute_luau` sandbox become permanently unreachable** (the closure's environment is torn down when the
script ends), which makes that RemoteFunction hang for the rest of the session and *looks* exactly like a
poisoned remote. It is a **test-harness artifact, not a game bug** — with no rebinding, `GetInventory`
returns correctly (verified: client received `Wood=7`, `BeardedAxe owned`, 9 defense entries). Do not
diagnose remotes by rebinding handlers from the sandbox; seed data and read, or add a temporary print.

## VERIFY (evidence recap)
- Demo: `UIFrameworkDemo present: false`.
- Modal/scrim/one-modal/close: readouts above.
- Live data: the four-tab table above (server seeded Wood=7/Stone=3, BeardedAxe(eq)+Seax, TownHall/Cannon x2).
- Responsive/tap: constraints + AbsoluteSize readouts above.
- Guardrails: no `rojo build`; no 2nd rojo serve; only the inventory panel + 2 kit components added; no
  inline styling; panel never mutates inventory; all code in canonical `src` (live re-synced); no secrets.

## STATUS: DONE

## BACK TO DIRECTOR
1. **Armor has no data source.** The snapshot exposes no armor (and `consumables` is always empty). Either
   §12 needs an armor system + snapshot bucket, or the Armor tab should be dropped until it exists.
2. **Tell the fan-out: fetch async.** The blocking-`InvokeServer`-on-open footgun (rough edge #1) will bite
   every panel that reads server data. Worth a one-line rule in the UI contract.
3. **Legacy `InventoryController` should be retired** in the HUD fan-out — it now duplicates this panel's
   purpose with the old plain styling.
4. Kit now has 7 primitives (Bar, Label, Panel, Button, Toast, TabBar, ScrollList). The fan-out can build
   HUD + shells on it without adding more, except possibly an icon/slot-grid component for item icons.
