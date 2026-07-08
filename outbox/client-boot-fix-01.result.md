# RESULT: client-boot-fix-01 — kill the double-bootstrap, one entry + one HUD

FROM: Claude Code (local executor)
DATE: 2026-07-06
STATUS: **DONE** (authorized scope complete + verified) — with one **pre-existing, out-of-scope bug my
fix EXPOSED** that the Director should triage next (see ⚠ section). The boot fix itself is clean.

---

## Summary of the authorized change
- **Deleted** the place-only `IronholdClient` bootstrap (the duplicate).
- **Deleted** `HUDController` (the older duplicate HUD) — both the disk file and the live instance.
- `ClientMain → ClientBootstrap` is now the **sole** client entry, initializing **one of each** surviving
  controller (World, Combat, Build, Inventory), with **CombatHUD as the sole HUD**.
- Play verify: PlayerGui holds **one CombatHUD, one Vignette, no HUDGui, zero duplicate stacked labels**
  — exactly the step-5 pass criteria.

## Step 1 — Parity before edits
- **Confirmed the load-bearing files are in sync** so deletions applied to the intended state:
  - Live `StarterPlayerScripts.Client.ClientBootstrap` source = disk `src/Client/ClientBootstrap.luau`
    (byte-for-byte logic): inits Build/Combat/World/Inventory, **does NOT init any HUD**. ✓
  - `IronholdClient` confirmed **place-only**: no file and no references on disk
    (`find`/`grep` over `src` = empty). ✓
  - `HUDController` referenced **only** by `IronholdClient` (being deleted) and itself — verified on both
    disk (`grep -rl HUDController src` → only its own file) and live (`script_grep HUDController` → only
    IronholdClient + its own definition). No other caller. ✓
- **Note on "reconnect Rojo":** I cannot click the Studio Rojo plugin's *Connect* button from here (the
  MCP Luau sandbox lacks Plugin capability — `GetDebugId` errors confirm it). I verified parity by direct
  source comparison instead, which satisfies the intent (deletions applied to the state we think we're
  editing). **No `rojo build` was run.** Deletions were performed directly in the Edit datamodel + on disk.

## Steps 2–4 — Deletions & single-entry confirmation
| Action | Result |
|---|---|
| Delete disk `src/Client/Controllers/HUDController.luau` | removed (5717 bytes) ✓ |
| Delete live `StarterPlayerScripts.Client.Controllers.HUDController` | `DELETED` ✓ |
| Delete live `StarterPlayerScripts.IronholdClient` | `DELETED` ✓ |
| Confirm no broken references to HUDController | none — only caller was IronholdClient ✓ |
| ClientBootstrap = sole entry, one init each, no 2nd HUD | confirmed (source unchanged; already correct) ✓ |

Post-deletion Edit datamodel (persists across Play-stop):
```
IronholdClient present: false   (want false)  ✓
HUDController present:   false   (want false)  ✓
Controllers: BuildController, CombatController, InventoryController, WorldController
StarterPlayerScripts: Client, ShieldToggleClient
```

## Step 5 — Play verification (the pass/fail evidence)
Fresh Play boot, live PlayerGui (Studio's `Freecam` omitted — it's a Studio test artifact, not game UI):
```
CombatHUD [ScreenGui] DO=0 {HealthFrame, XPFrame, LevelLabel, GoldLabel, ATKButton, ACTButton}
Vignette  [ImageLabel]
--- DUPLICATES: none (PASS)
--- Vignette count = 1   (was 2 before the fix)
--- HUDGui present = false (was present before the fix)
```
- **One** HUD (CombatHUD), **one** HP bar (HealthFrame), **one** XP bar + **one** Level label, **one**
  Gold label, **one** Vignette. **No `HUDGui`**, **no duplicate stacked labels** in the corner.
- Before → after: the top-left triple-HUD overlap (two "Lv 1" indicators, "Gold" label sitting on a
  health frame, red+green bars stacked, 2 vignettes) is **gone**. The double-bootstrap is eliminated.

## Step 6 — Persistence (no new orphan)
- **HUDController**: disk file deleted → the canonical/src state no longer contains it, so a Rojo sync
  can't re-add it. Fully persisted; **no place-only orphan created**. ✓
- **IronholdClient**: it was place-only, so its removal lives only in the open Edit place. It is deleted
  in the Edit datamodel now (survives Play-stop, verified above). **To persist to the `.rbxl` on disk you
  must Save the place (Ctrl+S / File → Save)** — I cannot trigger a place-save from the MCP sandbox
  (no Plugin capability). Nothing else is needed; there is no src counterpart to update.

---

## ⚠ EXPOSED PRE-EXISTING BUG (out of scope — needs a follow-up task)
Removing the duplicate bootstrap surfaced a latent problem: **after the fix, `InventoryGui` and
`BuildHUD` no longer appear in PlayerGui.** I traced it fully. It is **not caused by my deletions** (I
touched only client bootstrap scripts; the server is untouched), but you should know before it looks like
a regression:

**Root cause chain:**
1. **`GetInventory:InvokeServer()` hangs at runtime** — measured >8s, no return, no error, on a clean
   boot. A RemoteFunction that has no server handler bound never returns.
2. **The running server has NO `OnServerInvoke` bound for any RemoteFunction.** `script_grep OnServerInvoke`
   over the running game returns only the doc comment in `Net.luau` — none of Attack/GetInventory/
   ClaimPlot/PlaceStructure/etc. are handled at runtime. (The **disk** source *does* bind them — e.g.
   `src/Server/Services/InventoryService.luau:113` sets `Net.Function("GetInventory").OnServerInvoke`,
   plus CombatService/PlotService/StructureService/TownHallService. So this is a **server disk↔live gap**:
   the live place is running server code that lacks these handlers. `script_search "InventoryService"`
   finds no such script in the live tree, despite the disk source defining it.)
3. **`BuildController.Init()` blocks on that hang before building its UI.** Its very first line
   (`GetInventory:InvokeServer()`, `src/Client/Controllers/BuildController.luau:512`) yields forever, so
   it never builds `BuildHUD`/`BuildPalette` and never returns — which **starves `InventoryController.Init`**
   (the next controller in `ClientBootstrap`'s order: World → Combat → **Build** → Inventory).
4. **Why it looked fine before:** the now-deleted `IronholdClient` ran Inventory *before* Build, and
   `InventoryController.Init` builds its GUI **synchronously** (its server call is deferred), so
   `InventoryGui` instantiated regardless of the hang. My fix removed that path, exposing the block.

**This is really two bugs, both pre-existing:**
- **(Server) The live server is missing the RemoteFunction handlers that exist on disk** — likely the same
  place-vs-src drift pattern flagged in ui-audit-01 / the CONTRACTS work. Until the live server has
  `GetInventory.OnServerInvoke` bound, every inventory/attack/claim/place call from a client hangs. This
  is the higher-priority root cause and it affects gameplay far beyond UI.
- **(Client) `BuildController.Init` blocks the boot chain on a server round-trip before building its UI.**
  Even once the server is fixed, this is fragile: a slow/unavailable server call shouldn't prevent later
  controllers from initializing. The one-line-shaped fix is to defer that initial fetch
  (`task.spawn`/`task.defer`) so Init builds its UI and returns immediately.

I did **not** fix either — both are outside "kill the double-bootstrap, one entry + one HUD" and the
guardrail "correctness fix ONLY … report first." Recommend a follow-up task (server parity/handler
binding first, then the BuildController defer). If you want, I can take the BuildController defer as a
tiny, safe client-side correctness fix in a scoped follow-up.

## VERIFY (evidence recap)
- Parity: live ClientBootstrap == disk; IronholdClient place-only (disk find/grep empty); HUDController
  sole caller was IronholdClient (disk grep + live script_grep).
- Deletions: disk `rm` confirmed; live `Destroy()` confirmed; Edit-datamodel re-check shows both absent
  and surviving set correct.
- Play: fresh-boot PlayerGui dump → one CombatHUD, one Vignette, no HUDGui, DUPLICATES=none.
- Exposed bug: GetInventory InvokeServer timed >8s no-return (twice, incl. fresh boot); `script_grep
  OnServerInvoke` live = comment only; disk `grep OnServerInvoke src/Server` = 6 handler bindings incl.
  InventoryService:113.
- Guardrails honoured: no `rojo build`; server code untouched; nothing restyled; no framework built; no
  secrets in this file.

## STATUS: DONE (authorized scope) — with the ⚠ pre-existing server/BuildController issue flagged above.

## BACK TO DIRECTOR
1. **Save the place (Ctrl+S)** to persist the `IronholdClient` removal to the `.rbxl` (place-only; I can't
   save from the sandbox). HUDController removal is already persisted (src file deleted).
2. **Highest priority follow-up: the live server is missing its RemoteFunction handlers** (present on
   disk, absent at runtime). This blocks GetInventory/Attack/etc. — a gameplay-wide issue, same place-vs-src
   drift theme. Needs a server sync/parity task (and confirms *why* step 1 asked to reconnect Rojo first).
3. **Second follow-up (client):** make `BuildController.Init` not block the boot chain on its initial
   `GetInventory` fetch (defer it). Small, safe; I can do it in a scoped task on request.
4. The overlap the audit reported is **fixed at the source** — one bootstrap, one HUD. The layer
   manager / component-kit work (next task) now builds on a single, clean entry point.
