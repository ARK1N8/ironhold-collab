# RESULT: character-stats-01 — stat system + stamina/sprint (server-authoritative engine)

FROM: Claude Code (local executor)
DATE: 2026-07-09
STATUS: **DONE** — both systems built server-authoritative, all effects + the combined-reduction clamp +
stamina drain/regen/empty-gate verified live in Play with exact numbers. No regression (8/8 remotes,
0 errors). Character-sheet panel deliberately NOT built (next task).

---

## Files
```
Server/Services/StatService.luau      (NEW — sole writer of stats/points; derived effects; allocate remote)
Server/Services/StaminaService.luau   (NEW — server-authoritative stamina + WalkSpeed authority + sprint)
Server/Services/DataManager.luau      (PlayerData template: + stats, + unspentStatPoints — find-or-create)
Server/Services/ProgressionService    (grants STAT_POINTS_PER_LEVEL on each level-up)
Server/Services/CombatService.luau    (Strength -> melee dmg; shield+Defense combined clamp)
Server/ServerBootstrap.server.luau    (loads StatService + StaminaService)
Shared/Net.luau                       (+AllocateStat, GetStats; +StatsChanged, StaminaChanged, SetSprint)
Client/HUD/CombatHUD.luau             (+stamina Bar, +sprint button, StaminaChanged listener)
Client/SprintClient.client.luau       (NEW — Shift + touch sprint intent)
```
Ground Rule 7 honoured: PlayerData was **extended** (find-or-create in DataManager's default + a runtime
guard in StatService), no data structure recreated; the Net registry is the idempotent find-or-create one.

## Part 1 — Stat system (all tunable constants; verified live)
- **Storage:** `stats = {Strength,Vitality,Speed,Defense,Endurance}` + `unspentStatPoints` on PlayerData.
  StatService is the sole writer (§1). Legacy profiles gain the fields via DataManager's default-merge +
  a find-or-create guard.
- **Points per level:** ProgressionService grants **3/level** on each level gain. Verified: leveling 1→5
  (TownHall cap) granted **12 points** (3 × 4 level-ups).
- **Allocate remote:** `AllocateStat(statName)` — server-validated (unspent > 0, valid stat). Verified:
  allocates + decrements, **rejects `"Nonsense"` → false**, and **stops when points hit 0**.
- **Derived effects (server-side, verified via the real AllocateStat path):**
  | Stat | effect | verified |
  |---|---|---|
  | Vitality | +5 MaxHealth/pt (base 100) | Vit 10 → **MaxHealth 150** (health ratio preserved) |
  | Speed | +1% move /pt, cap +30% | Spd 30 → WalkSpeed **20.8** (16×1.30); 50 pts still 1.30 (capped) |
  | Strength | +2% melee dmg /pt | Str 10 → **meleeMult 1.20** (applied to melee-class weapons; Magic excluded) |
  | Defense | +1% incoming reduction /pt, cap 40% | Def 40 → **0.40** (capped) |
  | Endurance | stamina max + regen | End 10 → **max stamina 200**, faster regen |
- **Combined reduction clamp:** shield (0.25) + Defense (up to 0.40) combined and **clamped at 0.60** in
  one place (CombatService.applyPlayerDamage). Verified: shield-up + Def40 → `ApplyPlayerDamage(100)` =
  **40** (100×(1−0.60)).
- **Speed's attack-speed hook:** left DISABLED as specified — Speed only affects movement now. (No
  attack-speed code path added; flipping it on later is additive.)
- **Read path:** `GetStats()` remote returns `{stats, unspentPoints, derived{maxHealth,meleeMult,
  speedMult,defenseReduction}}` — pure read, no yields (async-safe). `StatsChanged` pushes on change.

## Part 2 — Stamina + sprint (sprint-only; verified live)
Tunables: `STAMINA_BASE=100, STAMINA_PER_ENDURANCE=10, DRAIN=20/s, REGEN=15/s (+0.5/s per Endurance),
REGEN_DELAY=1s, SPRINT_RESUME=15%, SPRINT_SPEED_BONUS=+40%`.
- **Server-authoritative:** the server owns the stamina value; the client sends **intent only**
  (`SetSprint`). StaminaService is the **sole writer of Humanoid.WalkSpeed**, composing Speed-stat ×
  sprint so they never conflict — applied **every Heartbeat tick** (race-free; a redundant set is a
  no-op). This per-tick apply fixed an event-timing race where the Speed multiplier didn't stick.
- **Verified live (observing real WalkSpeed + the HUD bar):**
  - Sprint composes: base 16 × Speed 1.30 × sprint 1.40 = **29.1** while sprinting; **20.8** on release.
  - Drain: 100 → 81 → 62 → 42 → 20 over 4s (**~20/s**). Endurance raises duration: base 100 ≈ 5s, End 10
    (200) ≈ 10s.
  - **Empty auto-stop:** at 0 stamina sprint disabled (WalkSpeed 22.4 → **16** at 5s); regen resumes after
    the 1s delay; sprint only **resumes once past the 15% threshold**.
- **HUD stamina bar:** third Kit.Bar in the vitals cluster (health/XP/stamina/level), themed, scale-based,
  **no inline styling**. Reflects live server stamina (`StaminaChanged`). Verified drains 100→0 and regens.
- **Sprint input:** desktop **Shift** (SprintClient LocalScript) + a **touch/gamepad hold-button** on the
  HUD (shown per Platform.Get(), wired via CombatHUD.OnSprintStart/End). Isolated LocalScript, not in
  ClientBootstrap.

## Part 3 — regression
8/8 RemoteFunctions respond (GetInventory, Attack, ClaimPlot, PlaceStructure, FireStructure,
UpgradeTownHall + the 2 new GetStats/AllocateStat), **0 client errors**. Inventory panel + HUD unaffected.

## VERIFY (evidence recap)
- Stat numbers: MaxHealth 150, WalkSpeed 20.8 (capped), meleeMult 1.20, defReduction 0.40 (capped),
  maxStamina 200; combined clamp → 40 dmg from 100; level→points 12 from 4 levels; invalid-stat rejected.
- Sprint: WalkSpeed 29.1 sprint / 20.8 base; drain ~20/s; empty auto-stop (→16) + resume-threshold gate;
  HUD bar drains 100→0 and regens.
- Guardrails: no `rojo build`; no 2nd rojo serve; find-or-create (no recreated data / Net registry);
  no blocking InvokeServer in init/render (GetStats/StatsChanged/StaminaChanged are event/async);
  server-authoritative throughout; character-sheet panel NOT built; all in canonical src; no secrets.

## STATUS: DONE

## BACK TO DIRECTOR
1. **Character-sheet panel is the natural next task** — the read path (`GetStats` + `StatsChanged`) is
   live and async-safe, so the panel can render stats/unspent-points and call `AllocateStat` on click.
2. **Testing note (not a bug):** the Studio MCP `execute_luau` sandbox runs a separate `require` cache, so
   calling `StaminaService.Get()` / `StatService` from the sandbox reads a *different* instance than the
   game runtime. I verified everything via the **real** paths (real `AllocateStat`/`SetSprint` remotes +
   real Humanoid.WalkSpeed + the HUD bar). Worth remembering for future service tests.
3. **Deferred hooks left in place, disabled:** Speed's attack-speed effect (movement-only for now) and
   respec — both additive later, no rework.
4. **BASE_WALKSPEED** is captured per-character from the Humanoid's spawn WalkSpeed (16 today); if you set
   a custom StarterPlayer.CharacterWalkSpeed it's picked up automatically.
