# TASK: consolidation-01 — reconcile, harden, cleanup

FROM: Studio Director
DATE: 2026-07-05
SCOPE: operates on the GAME project (C:\Users\Jon\Desktop\MMRPG Game), not this repo.
DO THIS BEFORE resuming weapon swing animations.

## Why
The 2026-07-05 engine bring-up left the live datamodel ahead of disk: the Rojo plugin
dropped mid-session and the last batch of fixes was applied directly to the live tree.
Disk `src/` is *claimed* canonical but not yet proven. Before building anything else we
close that drift, version-control the one place-file-only module, and remove the orphan.

## Do, in order

### 1. RECONCILE  (highest priority — this is the drift risk)
- Reconnect the Rojo plugin (`rojo serve` + Connect). NEVER `rojo build`.
- Reconcile Studio ⇄ disk so disk `src/` is authoritative.
- Verify each of the 8 bring-up fixes is actually present on disk in `src/`:
  1. DataManager — DataStore guard (in-memory fallback in Studio)
  2. DataManager.SetPlotId — exists
  3. EconomyService.SpendOwnedStructure — exists
  4. PlotService — OnLoaded registered per-player (not `:Connect`)
  5. StabilitySolver — steps by the 4-stud grid size
  6. EconomyService / DataManager.Get — nil-player guards
  7. WorldGenerator — `Enum.Material.LeafyGrass`
  8. CombatController — six listeners use `.OnClientEvent`
  Also confirm the HandshakeB B-7 fixture fix is on disk.
- REPORT a table: for each fix, "on disk ✓" or "was live-only → written back to disk".
  Any "written back" entries confirm the drift was real and is now closed.

### 2. HARDEN TwoHandHold
- Move place-file-only `ReplicatedStorage.TwoHandHold` into `src/Shared/TwoHandHold.luau`.
- Update the 3 `HoldServer` requires:
  `RS:WaitForChild("TwoHandHold")` → `RS.Shared:WaitForChild("TwoHandHold")`.
- Verify two-handed holds and the BeardedAxe swing still work after the move.

### 3. CLEANUP orphan
- Delete the stale nested `src/src/` tree entirely. Confirm it's gone.

### 4. DOC coherence
- In DIRECTOR_STATUS.md, §2's verification note still reads "all 6 pass (357/0), G–J not
  covered" — this contradicts §8 ("all 10 pass, 412/0, G–J covered"). §8 is authoritative;
  update §2 to match.

## Report back  (write outbox/consolidation-01.result.md)
- The 8-fix reconcile table (on disk vs written-back) + B-7 fixture status.
- TwoHandHold: moved? requires updated? holds + swing verified? DONE/PARTIAL/BLOCKED.
- Orphan `src/src/`: deleted + confirmed?
- Doc §2 updated?
- Anything unexpected surfaced during reconcile.

## Guardrails
- `rojo serve` + Connect ONLY — never `rojo build`.
- Do not modify the collab repo protocol; this task is game-project work.
- No secrets in the result file (outbox is public).
