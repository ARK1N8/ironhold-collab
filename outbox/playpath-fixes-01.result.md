# playpath-fixes-01 — Result

**STATUS: PARTIAL** — and per this task's own rule, I'm reporting exactly what real play verified, honestly. 4 of 8 items are implemented; 2 of those are verified by GR8 real-input play, 1 is coded-not-yet-played, 1 (the contract) is done. The remaining 4 fixes are not started. I would rather hand you a trustworthy partial than a green report that repeats the exact failure this task exists to correct.

## Scorecard
| # | Fix | Code | GR8 real-play verification |
|---|---|---|---|
| 1 | Starter weapon | ✅ done | ✅ **partially verified** — fresh player equips BeardedAxe; real axe swings damage a tree (100→68). Fell/wood + structure + NPC not completed via real input (see below). |
| 3 | ClaimablePlot tag + nil-guard | ✅ done | ✅ **fully verified** — walked to pad, pressed **E** for real → plot claimed + persisted, pad removed. |
| 2 | NPC death drops | ✅ done | ⚠️ **coded, not play-verified** (server-authoritative; no real kill completed this session). |
| 8 | Ground Rule 8 in contract | ✅ done | n/a — committed + pushed. |
| 4 | PlaceStructure client path | ❌ not started | — |
| 5 | Build UI → LayerManager | ❌ not started | — |
| 6 | Inventory row reflow | ❌ not started | — |
| 7 | BeardedAxe grip 180° | ❌ not started | — |

---

## Fix 1 — Starter weapon ✅ (partially verified by real play)

**Decision (reported per the task):** I did **both** — (a) registered `"Fists"` as a *real* weapon in `Weapons.luau` (Damage 5, Blunt, Cost 0, no AssetId → bare hands) so the default is never again an unregistered sentinel that makes `onAttack` bail; and (b) the fresh-player default profile now **owns + equips `BeardedAxe`** (`DataManager.buildDefaultData`), which both harvests and fights. `onAttack` now treats `"Fists"` as an always-usable innate state (mirrors `EquipWeapon`). A legacy migration in `deserializeData` grants+equips the BeardedAxe to any saved profile that owns no weapons / is stuck on the old `"Fists"` default.

**GR8 real-play verification (fresh player, real mouse input):**
- Booted a fresh player → **`equippedWeapon = BeardedAxe`, owns it** (was `Fists`, owned nothing). 0 boot errors.
- Aimed at a tagged Oak tree, verified the camera ray hit `HitPart [HarvestableTree]`, then drove **real left-clicks**. Tree `CurrentHealth` dropped **100 → 68** — i.e. the starter weapon **damages the tree through the real player path** (in playpath-diagnosis-01 the default player did **0** damage). This is the headline fix (#5/#14 root cause) demonstrated by real input.

**Honest gap:** I did **not** finish felling the tree (→ wood), nor test structure-HP / NPC damage, *via real play* this session. Not a game bug — three input-injection limitations fought me: the §11.6 hit-stop **camera shake** nudges my scripted aim off-target after each hit; the **unanchored character drifts** out of the 10-stud reach between swings (silently out-ranging later swings); and the virtual mouse threw repeated **`duplicate button state`** errors under rapid clicking. The underlying mechanics are unchanged and were already proven end-to-end in playpath-diagnosis-01 (a real axe felling a tree, +wood; and `onAttack` routing to `DamageStructure`/`DamageNPC` once the weapon check passes — which it now does). A human tester (or steadier input) will complete these instantly; I'm flagging it rather than claiming it.

Files: `Shared/Weapons.luau` (Fists def), `Server/Services/DataManager.luau` (default + migration), `Server/Services/CombatService.luau` (Fists-innate exception).

## Fix 3 — Plot claim ✅ (fully verified by real play)

Plots were claim-**by-position** with no world markers, but the client raycasts for a `ClaimablePlot`-tagged part — which never existed. Fix: `PlotService` now spawns a **claimable pad + beacon** (both tagged `ClaimablePlot`, with a "Claim Plot [E]" billboard) near each plotless player via a race-free periodic ensure-loop; the pad is destroyed on claim (find-or-create, GR7). Added the `ClaimPlot` nil-arg guard (`typeof ~= Vector3 → clean reject`).

**GR8 real-play verification:** fresh player, pad appeared 21 studs away (2 `ClaimablePlot` tags). Walked to it, aimed at the beacon (ray confirmed `ClaimablePlot=true`), pressed **E** for real. Result (confirmed via a real server Script — not `execute_luau`, which showed a stale sandbox value): **`plotId = plot_11030761645_…`, `plotOrigin` persisted, pad destroyed.** Plot claim works end-to-end through real input.

> Note: this again caught the sandbox trap live — my `execute_luau` read said `plotId=nil` while the real runtime had it set. Exactly why GR8 bars `execute_luau` as proof.

Files: `Server/Services/PlotService.luau`.

## Fix 2 — NPC death drops ✅ code / ⚠️ not play-verified

New data-driven `Shared/NPCDrops.luau` (weighted table: Wood 3 / Iron 2 / Gold 15). On NPC death, `NPCService` spawns a **physical collectible** at the death position that awards on `Touched` by a player (`AwardMaterial`/`AwardGold`, server-authoritative) and auto-despawns in 30s. Rebalance in the data table, never in service logic.

**Honest status:** implemented and synced, but I did **not** complete a real-input NPC kill → drop → collect this session (same combat-input limits as Fix 1). Needs a GR8 pass before it can be called DONE.

Files: `Shared/NPCDrops.luau`, `Server/Services/NPCService.luau`.

## Fix 8 — Ground Rule 8 ✅ (committed + pushed)

Added GR8 to `CONTRACTS.md` §0 next to GR7, and a `2.2.2` version-history line. **Committed and pushed to ARK1N8/IronHold `main` — commit `d8c9d89`** (`c295e12..d8c9d89`). Text: "Verify by playing, not by invoking … execute_luau is observation-only … UI acceptance requires enumerating every ScreenGui … five features passed review 'verified in Play' while broken for the real player."

## Fixes 4, 5, 6, 7 — NOT STARTED
- **#4 PlaceStructure client path** — no client caller exists yet; needs a picker→ghost→`PlaceStructure` flow (server remote already works). Now unblocked since plot-claim (#3) works.
- **#5 Build UI → LayerManager** — `BuildHUD`/`BuildPalette`/`WorldGui` still bypass the manager.
- **#6 Inventory row reflow** — row title/value still collide on the narrow panel.
- **#7 BeardedAxe grip 180°** — place-side hold rotation, untouched.

## Regression
Server boots clean (0 non-benign errors) with all Fix 1–3 changes. I did not run a full real-play regression sweep of HUD/inventory/character-sheet/sprint/structure-upgrades this session — flagging that honestly rather than asserting it.

## Recommendation for the Director
The gameplay foundation (starter weapon damages things; plot claim works) is in and real-input-verified for the core. Suggest a **follow-up (`playpath-fixes-02`)** to: finish GR8 verification of Fix 1 (fell+wood, structure, NPC) and Fix 2 (kill→drop→collect) with a human/steady tester, then implement + verify #4/#5/#6/#7. The virtual-input reliability issues (camera-shake drift, character drift, duplicate-button-state) are worth noting as a standing constraint on automated GR8 testing — a brief "anchor + disable hit-shake during automated combat tests" harness helper would make future combat verification reliable.

## Guardrails honored
No `execute_luau` used as proof (only observation; and I called out where it lied). No seeded state to pass a test — the diagnostic weapon grants were done via real Scripts and clearly labeled, and the actual claims/hits were driven by real input. rojo serve + Connect only (the plugin dropped mid-task; reconnected). GR7 idempotent (pad + migration find-or-create). No new gameplay systems beyond the requested fixes. No secrets in this doc.
