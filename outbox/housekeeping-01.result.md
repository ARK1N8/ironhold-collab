# RESULT: housekeeping-01 — retire legacy InventoryController, final Platform pass, idempotency rule

FROM: Claude Code (local executor)
DATE: 2026-07-09
STATUS: **DONE** — all three parts complete and verified. Legacy controller retired, GameConfig.PLATFORM
has zero readers, idempotency rule committed to the game repo.

---

## Part 1 — Retire the legacy InventoryController (dependency-gated)
**Reference search (src + live) — the deletion gate:**
- `src` grep: only `ClientBootstrap.luau` (require + safeInit + comment) and the file itself.
- Live scripts: only `ClientBootstrap` (5 refs) and `InventoryController` (3 self-refs).
- **No other controller, keybind, or script depends on it** — the new InventoryPanel
  (inventory-panel-01) replaced it and is opened by `InventoryPanelInit` (I key), unrelated to the old one.

**Deletion (gate passed):**
1. Removed the `require` + `safeInit("InventoryController", …)` from `ClientBootstrap`.
2. Deleted `src/Client/Controllers/InventoryController.luau` (Rojo synced the removal to live).

**Verified in Play (fresh boot):**
- `InventoryController` file gone from live; Controllers folder = `BuildController, CombatController,
  WorldController`. **0 client errors, 0 ClientBootstrap warnings** (no Init broke).
- **No legacy `InventoryGui`** in PlayerGui (the old controller's UI is gone).
- **No ghost / duplicate:** opened the panel → **exactly 1 `InventoryPanel` instance** in PlayerGui,
  in the MODAL layer, visible.
- **Inventory works end-to-end:** seeded via a real server Script, opened the panel → live data renders:
  `Weapons: Seax Equipped`, `Resources: Wood x15 | Stone`. Tabs switch; Armor empty (no armor bucket,
  known gap). CombatHUD unaffected (still in PERSISTENT).

## Part 2 — Final Platform migration (ClientBootstrap)
- `ClientBootstrap` no longer detects platform or writes `GameConfig.PLATFORM`. It now `require`s
  `Platform.luau` to prime runtime detection before controllers Init, and dropped the `GameConfig` +
  `UserInputService` requires.
- **`GameConfig.PLATFORM` reader/writer count on the client is now ZERO** — grep for `GameConfig.` in
  `src/Client` returns only comments. Confirmed at runtime: `Platform.Get()="Mouse"`, `HasTouch=false`,
  `HasGamepad=false` (working); no code path reads `GameConfig.PLATFORM`.
- **Field itself:** `GameConfig.luau:30` still defines `GameConfig.PLATFORM = "DESKTOP"` as a default
  constant. It is now **vestigial/unread**. I did **not** delete it (per the task — it's a Shared config
  field, and removing it is a separate cross-boundary change). Flagged below for a follow-up.

## Part 3 — Contract rule: idempotent find-or-create
Added to the **canonical git-tracked** `CONTRACTS.md` (ARK1N8/ironhold), as **Ground Rule 7** under
`## 0. Ground Rules` (placement chosen because it's a cross-cutting engineering constraint, alongside the
other numbered global rules — not specific to any one §). Exact text added:

> 7. **Idempotent init — find-or-create, never destroy-and-recreate.** Singleton/registry initialization
> MUST be idempotent (find-or-create). Init code must never destroy and recreate shared instances (e.g.
> the `Net`/remotes folder, UI layers, HUD, controllers) — doing so silently orphans existing references
> (client event connections, cached remotes, open UI) and has repeatedly caused drift/hang bugs.
> Re-running an Init must converge to one instance, not duplicate or tear down.

- Version footer noted lightweight: `2.2.1 — added Ground Rule 7 … No signature/data-shape changes.`
  (No restructuring of §1–§12; per Ground Rule 3 this isn't a signature/shape change, so a note rather
  than a full version bump of the module constants.)
- **Committed + pushed to the GAME repo** (ARK1N8/ironhold), `main`: **commit `c295e12`**
  (`docs(contract): add Ground Rule 7 — idempotent init (find-or-create)`), pushed `22668ed..c295e12`.

## VERIFY (evidence)
- Part 1: reference grep (src + live) both showed only ClientBootstrap + self; post-delete Play →
  0 errors/warnings, no legacy InventoryGui, 1 InventoryPanel (no ghost), live data rendered
  (Seax Equipped, Wood x15).
- Part 2: `grep GameConfig\. src/Client` → comments only; runtime Platform.Get() works; GameConfig.PLATFORM
  field is the sole leftover (a default constant, unread).
- Part 3: `c295e12` on ARK1N8/ironhold main; Ground Rule 7 present under §0.
- Cleanup: no Temp* scripts baked into the place; ClientBootstrap comment de-references the literal.
- Guardrails: no `rojo build`; no 2nd rojo serve; deletion was dependency-gated (confirmed dead first);
  contract committed to the game repo (not collab); all client code in canonical src; no secrets.

## STATUS: DONE

## BACK TO DIRECTOR
1. **`GameConfig.PLATFORM` field is now dead** (`GameConfig.luau:30`, default `"DESKTOP"`, zero readers).
   Safe to delete the field in a small follow-up — I left it because it's a Shared config module and the
   contract's §2.10 WorldConfig lives in the same file; removing a documented field is a contract-touch
   better done deliberately. (Server side: worth a quick grep of `src/Server` for `GameConfig.PLATFORM`
   before deleting — I only cleared the client.)
2. Ground Rule 7 is live in the contract — the Director's Gate can now cite it to reject any
   destroy-and-recreate init.
3. Client controller set is now lean: World / Combat / Build, plus the self-registering InventoryPanel.
   Clean base for the character-stats system.
