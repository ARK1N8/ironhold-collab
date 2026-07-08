TASK: server-remotes-fix-01 — bind the live server's RemoteFunction handlers + harden boot

FROM: Studio Director
DATE: 2026-07-06
SCOPE: GAME project. HIGHEST PRIORITY — blocks gameplay-wide, not just UI.
DEPENDS ON: client-boot-fix-01 (which exposed this pre-existing server drift).

## Context (from client-boot-fix-01)
`GetInventory:InvokeServer()` hangs forever because the RUNNING SERVER has NO `OnServerInvoke`
handlers bound for ANY RemoteFunction — Attack, ClaimPlot, PlaceStructure, GetInventory, etc. —
even though the disk source binds them (e.g. InventoryService.luau:113). This is server-side
disk<->live drift: the live server is running stale code where the entire remote layer never bound.
The double-boot was masking a downstream symptom; the real bug is server-wide.

Consequence: every client->server call that expects a response hangs. Inventory, plot claiming,
structure placement, and combat attacks are all non-functional live, despite passing on disk. Building
UI on this is pointless until the server answers.

This is the 4th instance this session of the same disk<->live drift pattern (place-only IronholdClient,
two CONTRACTS.md, live-ahead-of-disk combat fixes, now unbound server remotes). The root-cause sync
issue is being handled separately (rojo-sync-diagnose-01); THIS task restores correct live state now.

## Part 1 — Fix the server drift (load-bearing)
1. RECONNECT Rojo + confirm disk==live FIRST. rojo serve + Connect only — NEVER rojo build.
2. Force a clean SERVER sync so the live ServerScriptService matches src (the code that binds the
   handlers). Confirm the services actually re-ran their binding (a stale-but-synced module that already
   ran won't rebind — you may need to restart the server-side init / re-enter Play so the binding code
   executes).
3. VERIFY the handlers are bound on the live server. For each of these RemoteFunctions, confirm
   `OnServerInvoke` is set and an invoke RETURNS (does not hang):
     - GetInventory (InventoryService)
     - Attack (CombatService)
     - ClaimPlot (PlotService)
     - PlaceStructure (StructureService)
     - plus any other RemoteFunctions the contract/Net defines — enumerate them and check each.
   Report a table: remote name -> handler bound? -> test invoke returns?
4. Confirm no NEW place-only drift is introduced — the fix must live in the canonical src/live-synced
   state, not a hand-patch that vanishes on next sync. If anything had to be applied live, ensure it's
   reflected in src.

## Part 2 — Harden the boot chain (insurance)
Even with the server fixed, a controller that hard-blocks on a network call during boot is fragile
(that's how a slow server stalled InventoryController after the single-boot reorder).
- Make `BuildController.Init` DEFER its initial data fetch (e.g. fetch after Init returns / async, or
  build its UI first then populate) so a slow or unresponsive server can NEVER stall the boot chain.
- Confirm InventoryController and BuildController both build their UI (`InventoryGui` / `BuildHUD` get
  created) even if a server fetch is slow. No controller's Init should block the chain on a blocking
  InvokeServer.

## Report back (write outbox/server-remotes-fix-01.result.md)
- Rojo parity confirmed before edits?
- The remote-handler verification table (name / bound / invoke-returns) — this is the pass/fail evidence.
- Confirmation GetInventory now returns instead of hanging (test it end-to-end from a client).
- BuildController.Init deferral implemented? InventoryGui + BuildHUD both build now (verified in Play)?
- Confirmation the fix is in the canonical state, no new place-only orphan.
- STATUS: DONE / PARTIAL / BLOCKED + anything unexpected.

## Guardrails
- rojo serve + Connect only — NEVER rojo build (would wipe VikingTier1 / StarterPack / door service).
- Persist all changes to the canonical state; do NOT create new place-only drift while fixing drift.
- Focus: restore live remote handlers + defer the boot fetch. Do NOT build the UI framework or restyle
  anything — that's the next initiative after the sync root-cause task.
- No secrets in the result (outbox is public).
