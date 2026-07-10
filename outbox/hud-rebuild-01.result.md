# RESULT: hud-rebuild-01 — CombatHUD rebuilt on the framework + Platform migration

FROM: Claude Code (local executor)
DATE: 2026-07-09
STATUS: **DONE** — HUD rebuilt on the Kit in the PERSISTENT layer, live + event-driven, combat juice
preserved and verified firing, Combat/Build migrated off GameConfig.PLATFORM. Found and fixed a real
latent bug in my earlier Net.luau change along the way (details below).

---

## What changed
```
Client/HUD/CombatHUD.luau                (NEW — the HUD, built on Kit + Theme, PERSISTENT layer)
Client/Controllers/CombatController.luau (rewritten — juice only; HUD delegated to CombatHUD; Platform.luau)
Client/Controllers/BuildController.luau  (migrated off GameConfig.PLATFORM -> Platform.luau)
Client/UI/Components/Label.luau          (exposes ValueLabel so the HUD can animate the gold value)
Shared/Net.luau                          (BUGFIX — idempotent find-or-create; see below)
```

## 1. HUD on the framework
- `CombatHUD` registers a root frame in the layer manager's **PERSISTENT** layer via `LM.AddPersistent`
  (DisplayOrder 10) — always visible, never a modal. Verified: modal panels (DisplayOrder 20) draw above
  it and the HUD stays visible/untouched under modal open/close.
- Elements are all Kit primitives: **health Bar**, **XP Bar**, **Level** Kit.Label (top-left cluster);
  **Gold / Wood / Stone** Kit.Label rows (top-right cluster); touch/gamepad **Attack/Use** Kit.Buttons.
- **Zero inline colors/sizes** — 100% Theme. Verified in Play: the gold value reads exactly
  `Color3(0.831, 0.678, 0.220)` = (212,173,56).
- **Scale-based + responsive**: clusters sized by UDim scale with min/max `UISizeConstraint` clamps;
  Bars/rows fill width; buttons carry the 44px `UISizeConstraint` (MinSize 44,44). HUD never eats input
  (`Active=false` on root + clusters).
- **Top-inset fix**: the PERSISTENT layer ignores the GUI inset (so modals/scrim cover the whole screen),
  which put the HUD *behind the Roblox topbar* (measured AbsPos.Y = -41). Fixed by insetting the HUD root
  below the topbar via `GuiService:GetGuiInset()`, re-applied on `TopbarInset` change (mobile-safe). After
  the fix the top clusters sit at AbsPos.Y = 15, fully on-screen.
- HUD layout per §11.5: health+XP+level top-left, resources top-right. **Hotbar left as the default
  backpack** (not rebuilt, per scope).

## 2. Combat juice — preserved and VERIFIED firing
The old CombatHUD's feedback logic was NOT ripped out — it stays in CombatController, re-homed onto the
layer manager's TRANSIENT layer (was loose in PlayerGui). What existed and confirmed still fires:
- **Damage vignette** (§11.6): red screen flash on HP loss — verified (min ImageTransparency hit **0.09**
  on a hit, now in TRANSIENT).
- **Level-up flash + "LEVEL UP!"** — verified a full-screen flash Frame appeared in TRANSIENT on LevelUp.
- **Gold pulse** — the value label tweens on each GoldChanged (preserved via the new `Label.ValueLabel`).
- Also intact (same code, unchanged): floating **damage numbers**, **hit VFX** particle burst, **hit
  sound**, **hit-stop + camera shake** on a landed attack.

## 3. Live data, async, event-driven
- **Event-driven, not polled** (§11.8): `HealthChanged` → health bar; `GoldChanged` → gold;
  `LevelUp` → level + XP bar; `MaterialAwarded` / `InventoryChanged` → resource re-pull.
- **Async-open rule honored** (from inventory-panel-01): the only server fetch (`GetInventory` for
  Wood/Stone) runs in `task.spawn` with rows on a `...` placeholder until it lands — NO blocking
  InvokeServer in init/render. I also fixed the same footgun in CombatController's `InventoryChanged`
  handler (it previously did a blocking InvokeServer on the event thread).
- Verified live in Play (driven from a real server Script): health bar tracked to `84/100` (regen),
  Gold `1140`, Wood `12`, Stone `5`, XP bar `150/519 XP`, level row `3` on LevelUp — all updated live,
  **0 client errors**.
- Data-model note: the inventory snapshot has **no gold field** (gold only arrives via `GoldChanged`) and
  **no player-level field** — `snap.townHall.level` is the *Town Hall* level, not progression level. The
  old HUD conflated them; the rebuild does NOT seed the level row from townHall (it waits for LevelUp).

## 4. Platform migration
- **CombatController** and **BuildController** no longer read `GameConfig.PLATFORM`. Both now use
  `Platform.luau` (`Platform.Get()` / `Platform.Changed`). CombatHUD toggles its touch/gamepad action
  buttons via `Platform.Get()` and re-runs on `Platform.Changed` (live input switch, no rebinding).
- Input binding was made **branch-free**: every input path (mouse/E/gamepad/touch) is bound
  unconditionally and the input *type* gates which fires — inherently correct on any device and reactive
  to input changes with zero rebinding.
- **Remaining `GameConfig.PLATFORM` usages: 1** — only `ClientBootstrap:15` (the boot-time write),
  deliberately left for the final entry-point pass per the task. All 7 controller reads are gone.

## ⚠ Bug found + fixed: Net.luau was destroying live remotes
While wiring the HUD, GoldChanged/MaterialAwarded reached the client but the HUD's `OnClientEvent` never
fired — despite binding the correct, live event object. Root cause was **my own earlier change in
`server-remotes-fix-01`**: `Net.luau`'s server branch **destroyed the existing `Net` folder and recreated
all remotes** every time the server branch ran. Any re-run swapped the RemoteEvent/RemoteFunction
*instances*, silently orphaning every client `OnClientEvent` connection and leaving cached RemoteFunctions
dangling (this also retro-explains the intermittent "GetInventory hangs"). **Fix:** made the branch
**idempotent find-or-create** — reuse the existing folder and remotes, delete only genuinely duplicate
folders, so re-running is a no-op and client connections stay valid. Verified: after the fix, a real
server Script firing GoldChanged updated the HUD (Gold 821→1140). This is a strict improvement to the
earlier fix and touches only `Shared/Net.luau`.

## VERIFY (evidence)
- HUD: in PERSISTENT (DO=10); clusters at AbsPos.Y=15 (clear of the 58px topbar); gold trim exactly
  (212,173,56); buttons MinSize 44,44; root/clusters `Active=false`.
- Live data: server-driven Play run → HP 84/100, Gold 1140, Wood 12, Stone 5, XP 150/519, level 3; 0 errors.
- Juice: vignette min ImageTransparency 0.09 on damage; level-up flash frame appeared in TRANSIENT.
- Platform: grep shows the only live `GameConfig.PLATFORM` is ClientBootstrap:15; controllers use
  `Platform.Get()`.
- Cleanup: no `Temp*` scripts baked into the place; no debug prints in HUD; Net idempotent fix synced.
- Guardrails: no `rojo build`; no 2nd rojo serve; only HUD + Combat/Build touched (inventory panel,
  shells, hotbar untouched); all in canonical src (Rojo-synced); no secrets.

## STATUS: DONE

## BACK TO DIRECTOR
1. **Final Platform pass** can now migrate the entry point: `ClientBootstrap` still writes
   `GameConfig.PLATFORM` at boot (1 usage left). Once nothing reads it, delete the write. Small task.
2. **Legacy `InventoryController` still builds its own plain `InventoryGui`** (flagged since inventory-
   panel-01) — worth retiring in the shell fan-out now that both HUD and inventory are on the framework.
3. **Net.luau idempotency fix is load-bearing** beyond this task — it fixes silently-dropped client
   events/handlers game-wide. Worth a note in the contract that the remote registry must never
   destroy-and-recreate remotes.
4. Kit gained nothing new here except `Label.ValueLabel` (to animate a row's value). The shell fan-out
   can build on the same 7 primitives; an icon/slot component is still the likely next kit addition.
