# RESULT: server-remotes-fix-01 — bind the live server's RemoteFunction handlers + harden boot

FROM: Claude Code (local executor)
DATE: 2026-07-06
STATUS: **DONE** — GetInventory (and 5 other remotes) now respond instead of hanging; boot chain
hardened; InventoryGui + BuildHUD both build. **The root cause was NOT stale/unbound server code — it
was duplicate `Net` folders.** Details + one follow-up (6 genuinely-unimplemented remotes) below.

---

## ⭐ Root cause correction (important)
The task hypothesised "the live server is running stale code where the entire remote layer never bound."
That's **not** what was happening. The live `InventoryService` **does** bind
`GetInventory.OnServerInvoke` (line 113), the loader requires it, and the require succeeds. The real bug:

**There were THREE `ReplicatedStorage.Net` folders at runtime (1 baked into the place + more created
each run).** `Net.luau`'s server branch does `Instance.new("Folder")` named "Net" **unconditionally
every server start**, and a stale copy was also baked into the saved place. The server binds its handlers
on the folder *it* created, but the **client's `ReplicatedStorage:WaitForChild("Net")` resolves to the
first "Net" folder** — a handler-less dead one — so **every `InvokeServer` hung forever.** Same
place-vs-src *duplication* pattern as IronholdClient / the double CONTRACTS.md, but for a runtime folder.

## Step 1 — Parity before edits
- Live `ReplicatedStorage.Shared.Net` source == disk `src/Shared/Net.luau` (verified byte-for-byte before
  editing). Live `ClientBootstrap` == disk (verified in client-boot-fix-01). ✓
- Confirmed the live services exist and load: `ServerScriptService.ServerLoader` requires
  `ServerScriptService.Server.Services.*`; `require(InventoryService)` + all its deps succeed on the live
  server (tested in Server datamodel). So the handler code runs — the folder mismatch was the block.
- **Cannot click the Rojo plugin *Connect* button from the MCP sandbox** (no Plugin capability). Verified
  parity by direct source comparison and applied fixes to **both disk (canonical src) and the live
  datamodel** so they stay in sync. **No `rojo build` run.**

## Part 1 — Fix (load-bearing)
1. **`src/Shared/Net.luau` (canonical):** server branch now **destroys any pre-existing `Net` folder(s)
   before creating exactly one**, so the client and server always share a single handler-bound folder and
   stale copies can never accumulate again. Pushed to live `Shared.Net` (parity re-verified).
   ```lua
   if IS_SERVER then
       for _, existing in ipairs(ReplicatedStorage:GetChildren()) do
           if existing.Name == "Net" and existing:IsA("Folder") then existing:Destroy() end
       end
       local folder = Instance.new("Folder"); folder.Name = "Net"; folder.Parent = ReplicatedStorage
       ...
   ```
2. **Removed the stale baked `Net` folder** from the place (Edit datamodel: 1 → 0). At runtime the fixed
   module now creates exactly one.

### Handler verification table (client `InvokeServer` probe — respond-fast = bound; hang = no handler)
Measured on a fresh Play boot **after** the fix (folder count = **1**):

| RemoteFunction | Handler in src | Live invoke result |
|---|---|---|
| **GetInventory** (InventoryService) | yes | ✅ RESPONDS (table, 0.05s) |
| **Attack** (CombatService) | yes | ✅ RESPONDS (table) |
| **ClaimPlot** (PlotService) | yes | ✅ RESPONDS (errors on nil pos → handler ran) |
| **PlaceStructure** (StructureService) | yes | ✅ RESPONDS (boolean) |
| **FireStructure** (StructureService) | yes | ✅ RESPONDS (boolean) |
| **UpgradeTownHall** (TownHallService) | yes | ✅ RESPONDS (boolean) |
| PlaceBlock | **no** | ⚠ HANG — unimplemented |
| HarvestTree | **no** | ⚠ HANG — unimplemented |
| EquipWeapon | **no** | ⚠ HANG — unimplemented |
| PurchaseWeapon | **no** | ⚠ HANG — unimplemented |
| PurchaseStructure | **no** | ⚠ HANG — unimplemented |
| CraftStructure | **no** | ⚠ HANG — unimplemented |

**Before the fix: ALL 12 hung** (client on the dead folder). **After: the 6 that have handlers in src all
respond.** The remaining 6 hang because they have **no `OnServerInvoke` anywhere in `src/Server`** —
they were never implemented. `grep OnServerInvoke src/Server` = exactly those 6 bound handlers, no more.
This is a *missing-implementation* issue, distinct from the folder drift this task fixed — see follow-up.

- **GetInventory end-to-end:** `Net.Function("GetInventory"):InvokeServer()` from the client now returns a
  `table` in ~0.03–0.05s (was: no return in >8s). ✓

## Part 2 — Harden the boot chain
Deferring the fetch unmasked two more latent client bugs (both pre-existing, hidden because Build used to
block at the top of Init and never reached them). Fixed all three so both UIs build:
1. **`BuildController.Init` now builds its UI first and fetches inventory asynchronously** (`task.spawn`),
   so a slow/unresponsive server can never stall the boot chain.
2. **`BuildController.bindNetEvents` called `Net.OnEvent(...)` — a method that does not exist** (Net only
   has `.Function`/`.Event`) → it threw and aborted Init. Fixed to `Net.Event("X").OnClientEvent:Connect`
   (2 occurrences), matching how CombatController does it.
3. **`ClientBootstrap` now wraps each controller's `Init` in `pcall`** (`safeInit`) so one controller's
   failure can never blank the rest of the boot — defense-in-depth insurance.
All applied to disk **and** pushed to the live scripts (parity verified: async present, no `Net.OnEvent`
left, `safeInit` present).

### Play verification (fresh boot, after all fixes)
```
Net folders = 1            (PASS)
GetInventory: ok=true type=table   (was: HANG)
  CombatHUD    x1
  InventoryGui x1     <-- now builds
  BuildHUD     x1     <-- now builds
  BuildPalette x1
  Vignette     x1
  HUDGui       x0
Duplicates: none           (PASS)
client errors in log: 0    (PASS)
```
Both `InventoryGui` and `BuildHUD` build; single HUD; no duplicates; no client errors.

## Persistence / no new drift
- All four code changes live in **canonical `src`** (`Shared/Net.luau`, `Client/Controllers/BuildController.luau`,
  `Client/ClientBootstrap.luau`) **and** were pushed to the matching live scripts — disk == live, verified.
- Baked `Net` folder removed from the place (Edit: 0). **Self-healing:** even if unsaved, the Net.luau fix
  wipes any stale `Net` folder at every server start, so the hang cannot recur.
- **Recommend Ctrl+S** to persist the place cleanly (the baked-folder removal is a place edit; the sandbox
  can't save the `.rbxl`). No new place-only orphan was created.

## VERIFY (evidence recap)
- Parity: live Shared.Net == disk pre-edit; post-edit dedupe present on both.
- Root cause: server-side count showed **3** `Net` folders (all 27 remotes each); client resolved to
  folder #1; server module bound on its own folder → mismatch → hang.
- Post-fix: Net folders = 1; 12-remote probe = 6 respond / 6 unimplemented; GetInventory returns a table.
- Boot: PlayerGui = CombatHUD/InventoryGui/BuildHUD ×1, Vignette ×1, HUDGui ×0, dups none, 0 errors.
- Edit persistence: baked Net folders = 0, Shared.Net dedupe = true, BuildController async + Net.OnEvent
  gone = true, ClientBootstrap safeInit = true.
- Guardrails: no `rojo build`; changes in canonical src + live (no new place-only drift); no UI framework
  built / nothing restyled; no secrets.

## STATUS: DONE

## BACK TO DIRECTOR
1. **Save the place (Ctrl+S)** to persist the baked-`Net`-folder removal (self-healing regardless; the code
   fixes are already canonical in src).
2. **6 RemoteFunctions are genuinely unimplemented** (no `OnServerInvoke` in `src/Server`): **PlaceBlock,
   HarvestTree, EquipWeapon, PurchaseWeapon, PurchaseStructure, CraftStructure**. Two are actively called
   by the client today — **PlaceBlock** (BuildController placement) and **HarvestTree** (CombatController
   tree harvest) — so building blocks and chopping trees will hang until handlers exist. This is missing
   server code, *not* drift; recommend a dedicated task to implement the 6 handlers (I avoided inventing
   economy/craft behavior). EquipWeapon/PurchaseWeapon/PurchaseStructure/CraftStructure aren't invoked by
   the current client UI yet.
3. **For rojo-sync-diagnose-01:** the mechanism here was *runtime folder accumulation* (Net.luau created a
   new folder each run + a stale baked copy), not stale module source. The Net.luau dedupe closes it
   permanently, but it's worth noting the place had a baked runtime artifact saved into it — same class of
   problem as the other drift instances.
4. Client↔server request layer is now functional for the implemented remotes; inventory/plot/structure/
   combat round-trips work. The UI-framework initiative can proceed on a server that actually answers.
