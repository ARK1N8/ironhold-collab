# playpath-diagnosis-01 — Result

**STATUS: DONE.** Diagnosis only — nothing in the repo was changed. All five reported failures reproduced and root-caused by exercising the **real player input path** (real mouse clicks + key presses via input injection, camera raycasts, real ScreenGui census) — not by invoking remotes. One controlled runtime experiment (granting a weapon) isolated cause from symptom. The hypothesis is confirmed: **the server logic is fine; the client→server path and a couple of missing data/wiring pieces are where it breaks.**

## TL;DR — root causes
| # | Feature | Root cause | Shared? |
|---|---|---|---|
| 5 | Structures take no weapon damage | Default `equippedWeapon = "Fists"` is **not a registered weapon**; no starter weapon granted, player owns none → `onAttack` returns `failResult` before any damage routing | **A** |
| 14 | Trees take no weapon damage | Same as #5 (same `onAttack` early-out) | **A** |
| 1 | Plot claim doesn't work | The `"ClaimablePlot"` tag the client checks is **never written** anywhere → the E-interact raycast never matches a plot → `ClaimPlot` is never called | B |
| 9 | Structure placement doesn't work | **No client code ever calls `PlaceStructure`** (only `PlaceBlock` exists); and block placement is gated on `currentPlotId`, which depends on #1 | C (+B) |
| 6 | Menus overlap | Three ScreenGuis (`BuildHUD`, `BuildPalette`, `WorldGui`) are created **ad-hoc, outside the Kit LayerManager** → the one-modal scrim can't manage or cover them | D |
| 7 | Inventory items don't fit | Row title/value text **collide** on the narrow panel, and only ~4.5 of 9 rows fit the 230px viewport (scroll works but affordance is weak) | D-adjacent |

---

## PART 1 — Play it like a player (full chains)

### GROUP A — #5 structures & #14 trees take no damage (ONE root cause)

I stood next to a real, tagged Oak tree and swung with **real left-clicks** (input injected into the viewport). Full chain, instrumented with a parallel input observer + server reads:

| Chain step | Result |
|---|---|
| a) Client input fires? | **YES** — `InputBegan MouseButton1, gpe=false` fired on every click (CombatController's handler runs) |
| b) Raycast resolves target? | **YES** — ray hit `HitPart in OakTree`, tags `[HarvestableTree]` (correct target) |
| c) Remote called? | **YES** — `executeAttack` → `Attack:InvokeServer(HitPart)` |
| d) Server result? | **REJECTED** — 3 real swings → tree health stayed 100, **0 wood, 0 damage** |

**Why the server rejects:** `onAttack` reads `data.equippedWeapon`, which for every fresh player is `"Fists"`. `Weapons.Get("Fists")` → **nil** (there is no "Fists" entry in `Weapons.luau`), so `onAttack` hits its `if not weaponDef then return failResult` at the very top (CombatService.luau:108–111). Runtime confirmed: `equippedWeapon='Fists'`, `ownedWeapons={}` (owns nothing), and no code grants/equips a starter weapon on join (`DataManager.buildDefaultData` sets `equippedWeapon="Fists"` and stops there).

**Controlled experiment (isolates cause):** I granted+equipped a real `WoodAxe` **in the true runtime** (via a real server Script — see the Part 3 gotcha for why `execute_luau` was NOT enough), then swung once with a real click. Tree health **100 → 90** (−10 = WoodAxe.Damage). So input, raycast, remote, tag-routing, and `HarvestTree`/`DamageStructure` **all work** — the *only* missing piece is a usable equipped weapon. This single root cause explains both #5 and #14 (and would break all combat: NPCs, PvP too). Structures are the same story — `onAttack`'s `PlacedStructure` branch is never reached because it fails on the weapon check first.

> One fix (grant + equip a real starter weapon on spawn, e.g. a Club/WoodAxe, or make "Fists" a real zero-cost registry entry) resolves #5 and #14 together.

### GROUP B — #1 plot claim

Only one client path exists: `CombatController.executeInteract` (E key / gamepad R1) raycasts forward and checks `CollectionService:HasTag(hitPart, "ClaimablePlot")` → `ClaimPlot:InvokeServer(hitPart.Position)`.

| Chain step | Result |
|---|---|
| a) Input fires? | YES — E is bound in CombatController |
| b) Target resolves? | The raycast hits *something*, but… |
| c/d) Remote called? | **NEVER for a plot** — the `"ClaimablePlot"` tag is **only ever read** (CombatController.luau:195). A full-repo grep finds **zero** `AddTag(..., "ClaimablePlot")`, and the runtime census shows **0 parts tagged `ClaimablePlot`** (vs 717 `HarvestableTree`). So the plot branch of `executeInteract` can never be entered; `ClaimPlot` is never invoked from real input. |

Break point: **client targeting layer** — the tag contract between whoever creates plot parts and the client interact handler was never fulfilled. (The nil-arg `ClaimPlot` gap logged earlier is a *secondary* robustness issue; the primary bug is that the call never happens.)

### GROUP C — #9 structure placement

`BuildController` is the only build/placement code. Full read + repo grep:

- It calls **`PlaceBlock`** (BuildController.luau:231) for material blocks only.
- **`PlaceStructure` is never called anywhere in the client** — the sole occurrence is a stale comment in the file header (line 4). So placing a *purchased structure* (Wall, ArcherTower, …) has **no client entry point at all**: no UI, no input, no remote call.
- Even material-block placement is gated: `attemptPlace` early-returns unless `inBuildMode and selectedMaterial and currentPlotId`. `currentPlotId` is only set by the `PlotClaimed` event — so **block placement is transitively blocked by #1** (no plot claim → `currentPlotId` stays nil → every placement no-ops).

Break points: **client is missing the entire PlaceStructure path**, and block placement is **gated behind the broken plot-claim (#1)**.

---

## PART 2 — Menu overlap (#6) + inventory sizing (#7)

**Reproduced in real play.** ScreenGui census of `PlayerGui` (screenshot `menu_overlap2` attached in session):

| ScreenGui | DisplayOrder | Managed by LayerManager? |
|---|---|---|
| UILayer_PERSISTENT | 10 | ✅ Kit |
| UILayer_MODAL | 20 | ✅ Kit |
| UILayer_TRANSIENT | 30 | ✅ Kit |
| **BuildHUD** (Build/Rotate toolbar) | **0** | ❌ ad-hoc ScreenGui |
| **BuildPalette** (material palette) | **0** | ❌ ad-hoc ScreenGui |
| **WorldGui** (chunk/plot labels) | **0** | ❌ ad-hoc ScreenGui |

**Root cause:** `BuildController.buildHUDButton` / `buildPaletteUI` and `WorldController` each create their **own** `ScreenGui` with hardcoded inline colors, bypassing the Kit LayerManager entirely (the "Build UI reskin was never rebuilt" suspect — confirmed). The layer manager's one-modal + scrim guarantee only governs children of `UILayer_MODAL`; the scrim is a frame *inside* that layer, so it **cannot cover sibling ScreenGuis**. With the Inventory modal open, `BuildHUD`, `BuildPalette`, and the persistent MenuCluster all remain on screen — the screenshot shows the "Character/?" cluster overlapping the modal's title bar and the Build/Rotate toolbar + Gold/Wood/Stone readout bleeding around it. (These ad-hoc GUIs are also below the Kit layers at DisplayOrder 0, so they can't even be raised/hidden by the manager — they're simply unmanaged.)

> Fix direction: route the Build toolbar/palette and world labels through the Kit (PERSISTENT/MODAL layers) so the manager can hide/scrim them like every other panel.

**#7 inventory sizing.** Measured the open Inventory panel on the Armor tab (9 rows):
- Panel `AbsoluteSize` 496×368; ScrollList viewport 472×**230**; content canvas 472×**464** (auto-sized). Scrolling **is** configured (`AutomaticCanvasSize.Y`, `ScrollingEnabled=true`, `ClipsDescendants=true`), so it's not a hard clip/overflow bug — but only ~4.5 of 9 rows (48px each) fit at once, with a thin 6px scrollbar → items look "cut off."
- **Worse:** within each row the `Kit.Label` title (DisplayName) and value (detail string) are laid out left/right and **visibly collide** on the narrow row — the screenshot shows "Leather Helmet" overprinting "Helmet · Leather · -3%". So the row template doesn't reflow at this width.

> Fix direction: taller/animated scroll affordance or a bigger panel, and a row layout that wraps/truncates (or stacks title over detail) instead of overlapping.

---

## PART 3 — Why our verification missed all of this (the important part)

**What every prior "verified in Play" actually proved:** that the **server function works when called directly**, and/or that a **panel renders when opened programmatically**. Concretely, from the outbox results:
- `tier1-handlers-01`: bound `HarvestTree`/`PlaceBlock` and confirmed they "work end-to-end" by **invoking the remotes** and checking server state — never by swinging an axe or entering build mode.
- `structure-upgrades-01`: "verified in Play with real numbers" — but via `execute_luau` calling `UpgradeStructure`/`GetStructureInfo` and real seeding Scripts. Player *weapon* damage to a structure was never swung.
- `framework-core-01`: "one-modal-at-a-time verified" — true **for GUIs that go through the LayerManager**; the ad-hoc Build UI was out of scope, so nothing proved the *whole screen* has one manager.
- `inventory-panel-01`: "all four tabs verified" — by reading a `GetInventory` snapshot and opening the panel; tested with a near-empty fresh inventory, so full-inventory row overflow/collision never appeared.

**The systematic gaps (two of them):**
1. **Server-proven ≠ player-proven.** We verified `click→server` by starting at "server." The entire `input handler → targeting/raycast → correct remote with correct args` layer — where #1/#5/#9/#14 actually live — was never executed. A remote that "works when invoked" tells you nothing about whether any player input ever invokes it (or invokes it with a real target, or is even wired at all — see #9's missing `PlaceStructure`).
2. **The MCP sandbox context trap — confirmed live this session.** `execute_luau` runs in a **separate `require` cache** from the game runtime. When I set `equippedWeapon="WoodAxe"` via `execute_luau`, the real `onAttack` **did not see it** (the axe swing still did nothing); only when I set it from a **real server Script** did the tree take damage. So any prior verification that *seeded data or invoked a remote through `execute_luau`* was, in part, testing a **parallel datamodel** the real handlers never read. This is exactly how "verified" features can be genuinely broken for the real player.

### Proposed verification protocol (Ground Rule candidate)

> **Ground Rule 8 — A feature is not DONE until it is exercised through real player input in a Play session.**
> 1. **Real input, not remotes.** Verification MUST drive the feature the way a player does — injected mouse/keyboard/touch, on-screen buttons, camera raycasts — not `execute_luau` invoking the remote and not a seeding Script that calls the service directly. If you invoked a remote to test, you have tested the server, not the feature.
> 2. **Prove the whole chain.** For any input-driven feature, the result MUST state, with evidence: (a) the client input handler fired, (b) targeting/raycast resolved the intended target (paste what it hit + its tags), (c) the remote was called **with what args**, (d) the server's return/reject. "It works" without this chain is not acceptance.
> 3. **Fresh-player state.** Test as a **brand-new player** (default profile: `equippedWeapon="Fists"`, owns nothing, no plot), because that is what real players hit first. Seeding gold/weapons/plots to make a test pass hides the exact starter-experience bugs in this report.
> 4. **`execute_luau` is for OBSERVATION, never for the acceptance path.** Reading state (health, tags, data) is fine. **Mutating** state or **invoking gameplay remotes** through `execute_luau` does NOT count as verification and may not even affect the real runtime (separate require cache). If you must seed for a controlled experiment, do it from a real Script and say so.
> 5. **Whole-screen UI check.** UI acceptance MUST enumerate **every ScreenGui in PlayerGui** and confirm each intended-hidden one is actually hidden by the LayerManager. A modal "works" only if nothing outside the manager is visible over/around it.

(1) + (2) would have caught #1/#5/#9/#14 immediately; (3) surfaces the Fists/no-starter-weapon and no-plot gating; (4) is the sandbox trap that made false "verified" claims possible; (5) catches #6/#7.

---

## Recommended fix order (for the Director)

1. **Starter weapon (fixes #5 + #14 together, cheapest, highest impact).** Grant + equip a real starter weapon on spawn (e.g. Club or WoodAxe), or register "Fists" as a real zero-cost weapon. One change unblocks all combat + harvesting. *Low effort.*
2. **Tag plots `"ClaimablePlot"` (fixes #1, and unblocks #9's block-placement gate).** Whoever spawns plot parts must `AddTag(part, "ClaimablePlot")` (mirror how trees get `HarvestableTree`). Add the nil-arg guard to `ClaimPlot` while there. *Low effort, unblocks a dependency chain.*
3. **Add the `PlaceStructure` client path (fixes the rest of #9).** A real entry point (structure picker → ghost → `PlaceStructure:InvokeServer`), analogous to the existing `PlaceBlock` flow. *Medium effort — genuinely missing feature, not a regression.*
4. **Route Build UI + world labels through the Kit LayerManager (fixes #6).** Rebuild `BuildHUD`/`BuildPalette`/`WorldGui` as Kit panels on PERSISTENT/MODAL so the manager governs them. *Medium effort.*
5. **Inventory row reflow + scroll affordance (fixes #7).** Wrap/truncate row text; enlarge panel or make the scrollbar obvious. *Low-medium effort.*

Items 1 and 2 are one-liners with outsized impact and should go first. 3/4/5 are real feature/UX work. #1/#5/#9/#14 are **separate root causes** (A, B, C, D) — there is no single fix for all four — but 5 and 14 share cause A, and 9 partially shares B.

## Guardrails honored
Diagnosis only — **no repo changes**. The one runtime weapon-grant was a controlled experiment in the Play session via a temp Script, reverted immediately, never written to disk. Real player input used throughout; remotes were not invoked as the acceptance path. rojo serve + Connect only. No secrets in this doc.
