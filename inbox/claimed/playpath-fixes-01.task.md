TASK: playpath-fixes-01 — fix the 6 confirmed player-path bugs + codify Ground Rule 8

FROM: Studio Director
DATE: 2026-07-09
SCOPE: GAME project. Fixes the bugs playpath-diagnosis-01 root-caused. Adds Ground Rule 8 to the contract.
DEPENDS ON: playpath-diagnosis-01 (root causes confirmed with real input, not remote invocation).
ENV: sync stable on Host 127.0.0.1; rojo serve running on 127.0.0.1:34872 — do NOT start a 2nd instance.
       rojo serve + Connect only, NEVER rojo build.

## ⚠ VERIFICATION IS THE POINT OF THIS TASK
Every fix below MUST be verified under **Ground Rule 8**: as a FRESH PLAYER, using REAL INPUT (click, keys,
the actual UI), in Studio Play. `execute_luau` is for OBSERVATION ONLY — never as proof a feature works.
Do NOT invoke remotes directly to "verify." Do NOT seed state with scripts to make a test pass. If you
cannot demonstrate it by playing it, it is NOT done.
This rule exists because prior tasks (including combat, structure-upgrades, and armor) were "verified" by
invoking the server directly — which is exactly why five features passed review while being broken in play.

## Fix in this order (diagnosed order — early ones unblock later ones)

### 1. Starter weapon  → fixes #5, #14, AND #4 (NPC damage)
Root cause: default `equippedWeapon = "Fists"` is NOT a registered weapon, and no starter weapon is ever
granted, so `onAttack` bails before any damage is computed. (Proven: 3 real swings = 0 damage; grant a real
axe -> tree 100->90 in one swing. Input/raycast/routing all already work.)
- Grant a real starter weapon on spawn/join (a BeardedAxe is the sensible default — it harvests AND fights).
- Decide and report: fix the "Fists" default properly (either register Fists as a real weapon def with
  low damage, or ensure a real weapon is always equipped). Prefer NOT leaving an unregistered sentinel
  value in player data.
- VERIFY (GR8): as a fresh player, swing at a TREE (damage + fell + wood), a STRUCTURE (HP drops), and an
  NPC (#4 — NPC takes damage). Report actual numbers observed in play.

### 2. NPC death drops  (#4, second half)
- On NPC death, drop a RANDOM resource (Wood / Iron / Gold) that the player collects.
- Keep it data-driven (drop table, not hardcoded logic). Server-authoritative.
- VERIFY (GR8): kill an NPC with real swings, see the drop, collect it, see inventory increase.

### 3. ClaimablePlot tag  → fixes #1, unblocks #9
Root cause: the "ClaimablePlot" tag the client checks is NEVER written (0 tagged at runtime), so ClaimPlot
never fires.
- Write the tag on claimable plots (server-side, at world/plot generation).
- Also fix the `ClaimPlot` nil-arg validation gap logged earlier.
- VERIFY (GR8): as a fresh player, walk up to a plot and CLAIM IT through real input. Confirm the plot is
  claimed and persisted.

### 4. PlaceStructure client path  (#9)
Root cause: NO client code calls `PlaceStructure` (only `PlaceBlock` exists client-side).
- Add the client path so a player can actually place a STRUCTURE (the server remote already works).
- Use the existing ghost-preview placement flow per contract §11.5 if present; if not, wire the simplest
  correct path and report what's missing for the full ghost flow.
- VERIFY (GR8): as a fresh player, claim a plot, then PLACE A STRUCTURE via real input. It appears in-world.

### 5. Build UI into the LayerManager  (#6 — the REAL overlap fix)
Root cause: `BuildHUD`, `BuildPalette`, and `WorldGui` are ad-hoc ScreenGuis created OUTSIDE the
LayerManager — the scrim can't cover siblings it doesn't own. The layer manager was never broken; this UI
never joined it.
- Route these through the LayerManager + Kit/Theme (BuildHUD -> persistent or modal as appropriate;
  BuildPalette -> modal). Zero inline styling (every rebuilt panel has held this bar).
- Ground Rule 7: idempotent find-or-create; do not leave a legacy ScreenGui also being created.
- VERIFY (GR8): enumerate EVERY ScreenGui in PlayerGui during play, open all panels, and confirm NO overlap
  and one-modal-at-a-time actually holds for the player. Paste the ScreenGui census.

### 6. Inventory row reflow  (#7)
Only ~4.5 of 9 rows fit and row title/value text collide on the narrow panel.
- Fix row sizing/overflow so rows fit and text doesn't collide; scroll properly. Kit/Theme only.
- VERIFY (GR8): open inventory as a player with many items; all rows readable, no text collision, scroll works.

### 7. BeardedAxe grip  (#8)
Held upside down — needs a 180° rotation about the HANDLE-LENGTH axis (this is a `Tool.Grip` cant fix; the
mechanism is known from the chop work).
- Fix the grip so it's held correctly. Confirm the CHOP still reads correctly after (the swing's grip-drive
  sweeps from idle cant -> edge-down; adjust the sweep endpoints consistently if the idle cant changes).
- VERIFY (GR8): equip and swing as a player; axe held right-way-up AND the chop still lands edge-first.

## 8. Contract — add Ground Rule 8
Add to the canonical git-tracked CONTRACTS.md (§0 Ground Rules, next to Ground Rule 7):
  "GR8 — Verify by playing, not by invoking. A feature is not DONE until it is exercised through REAL
   PLAYER INPUT as a fresh player in Play (click/keys/UI). Invoking a remote directly, seeding state via
   script, or testing through execute_luau proves only that the server function exists — it does NOT prove
   the player can reach it. execute_luau is for OBSERVATION only. UI acceptance requires enumerating every
   ScreenGui in PlayerGui. This rule exists because five features passed review while being broken in play."
Commit to the GAME repo (ARK1N8/IronHold).

## Report back (write outbox/playpath-fixes-01.result.md)
- Each of the 7 fixes: what changed, and the GR8 PLAY VERIFICATION (real input, fresh player, actual numbers
  observed). Be explicit that you played it — if any item was NOT verified by real play, say so plainly.
- The ScreenGui census during the overlap test (proof #6 is actually fixed).
- Ground Rule 8 committed (commit hash).
- Regression: HUD/inventory/character sheet/stats/sprint/structure upgrades still work — VIA REAL PLAY.
- STATUS: DONE / PARTIAL / BLOCKED. Partial is fine — an honestly-verified subset beats a green report we
  can't trust. That's the whole lesson of this task.

## Guardrails
- GR8 governs all verification in this task. No execute_luau as proof. No seeded state to pass a test.
- rojo serve + Connect only (no 2nd instance). NEVER rojo build.
- Ground Rule 7: idempotent find-or-create; never destroy-and-recreate.
- Do NOT build new systems (mine, smelting, chests, defenses, art) — fixes only. Those come after.
- Everything in canonical src. No place-only drift. No secrets in the result (outbox is public).
