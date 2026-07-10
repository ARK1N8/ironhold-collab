TASK: character-stats-01 — stat system + stamina/sprint system (server-authoritative engine)

FROM: Studio Director
DATE: 2026-07-06
SCOPE: GAME project. Builds the STAT + STAMINA systems (the engine). The character-sheet PANEL and the
       stat-allocation UI are a SEPARATE follow-up task once these systems return live data.
DEPENDS ON: ProgressionService (owns XP->level), hud-rebuild-01 (HUD on Kit; Bar primitive available),
       shield-offhand-01 (established server-side damage reduction pattern), contract v2.2.0 §12.4 +
       Ground Rule 7 (idempotent find-or-create).
ENV: sync stable on Host 127.0.0.1; rojo serve running on 127.0.0.1:34872 — do NOT start a 2nd instance.
       rojo serve + Connect only, NEVER rojo build.

## Design (locked with Director) — all values are tunable constants
Five allocatable stats. Character earns 3 stat points per level-up (via ProgressionService).
Effects are capped so uncapped point investment can't break the game. Respec DEFERRED (not this task).

  STAT       PER-POINT EFFECT                              CAP / NOTES
  Strength   +2% melee weapon damage                       (no effect cap for now)
  Vitality   +5 max HP (base 100)                          (no cap for now)
  Speed      +1% movement speed                            cap ~+30% total. MOVEMENT ONLY now —
                                                           design an attack-speed hook but leave it
                                                           DISABLED (flip on later, no rework).
  Defense    +1% incoming damage reduction                 cap ~40%. Stacks with shield's 25%, but
                                                           TOTAL combined incoming reduction clamped ~60%.
  Endurance  +10 max stamina (base 100) AND faster regen   drives the stamina system below.

Server-authoritative: stats live on the player's server data (DataManager/ProgressionService), never
trusted from client. All stat effects applied server-side.

## Part 1 — Stat system
- Store the 5 stats + unspent stat points on the player's server data (extend the existing PlayerData
  template — per Ground Rule 7, extend/find-or-create; do not recreate the data structure).
- On level-up, grant STAT_POINTS_PER_LEVEL (=3) unspent points.
- A server RemoteFunction/Event to ALLOCATE a point to a stat (server validates: has unspent points,
  valid stat name). Re-applies derived effects on change.
- Apply derived effects server-side:
  - Strength -> melee damage multiplier used by CombatService when computing weapon damage.
  - Vitality -> Humanoid.MaxHealth (base 100 + 5*points), preserving current-health sensibly on change.
  - Speed -> Humanoid.WalkSpeed multiplier (base + up to +30%).
  - Defense -> incoming-damage reduction in CombatService, stacking with the shield reduction but the
    COMBINED reduction clamped at ~60% (implement one clamp so shield+Defense can't exceed it).
  - Endurance -> feeds stamina max + regen (Part 2).
- Expose the stats + unspent points via a read path (e.g. a GetStats remote, or fold into the existing
  inventory/stats snapshot) so the future character-sheet panel can render them. ASYNC-SAFE: no blocking
  work in any init/render path (the async rule from inventory/HUD).

## Part 2 — Stamina + sprint system (sprint-only for now)
Create a StaminaService (server-authoritative stamina) + client sprint input + HUD stamina bar.
- Max stamina = 100 + 10*Endurance (STAMINA_BASE=100, STAMINA_PER_ENDURANCE=10).
- Sprint on a held input (Shift on desktop; a touch sprint button on mobile). While sprinting:
  movement gets SPRINT_SPEED_BONUS (~+40% on top of base+Speed), and stamina drains at
  STAMINA_DRAIN_PER_SEC (~20/s) -> ~5s at base, ~15s at 20 Endurance (300 stamina).
- Regen: when NOT sprinting, after REGEN_DELAY (~1s) stamina regenerates at STAMINA_REGEN_PER_SEC (~15/s),
  scaling faster with Endurance.
- Empty behavior: at 0 stamina, sprint disabled until stamina regens above SPRINT_RESUME_THRESHOLD (~15%)
  — prevents stutter-sprinting on empty.
- Server-authoritative: the server owns the stamina value and validates sprint (never trust client for
  stamina amount). Client sends sprint intent; server drains/gates.
- HUD: add a stamina Bar to CombatHUD using the existing Kit Bar + Theme (third bar alongside health/XP).
  Themed, scale-based, no inline styling. Bar reflects live server stamina.

## Part 3 — verify (systems only; no character-sheet panel this task)
- Allocate points to each stat and confirm the effect applies live (e.g. +Vitality raises MaxHealth;
  +Speed raises WalkSpeed up to the cap; +Defense reduces incoming damage and the shield+Defense combo
  clamps at ~60%; +Strength raises dealt melee damage).
- Sprint drains stamina, stops at 0, regens after delay; Endurance raises both max and regen; the
  stamina bar tracks live.
- Confirm all 6 previously-working remotes + inventory/HUD still work (no regression).

## Report back (write outbox/character-stats-01.result.md)
- Stat storage + points-per-level + allocate remote (server-validated). Per-point effects wired to
  CombatService/Humanoid as specified; the ~60% combined-reduction clamp implemented.
- Stamina/sprint: max/drain/regen/empty behavior + the tunable constants used; sprint input (desktop +
  touch); stamina bar on HUD (Kit Bar, no inline styling).
- Read path for stats/points exposed for the future panel (async-safe).
- Verification evidence per Part 3 (numbers: MaxHealth change, WalkSpeed change, damage reduction clamp,
  sprint seconds at base vs high Endurance).
- STATUS: DONE / PARTIAL / BLOCKED + anything unexpected.

## Guardrails
- rojo serve + Connect only (running on 127.0.0.1:34872 — no 2nd instance). NEVER rojo build.
- Ground Rule 7: extend PlayerData / find-or-create; do NOT destroy-and-recreate data structures or the
  Net registry. Idempotent init.
- Async rule: no blocking InvokeServer in any init/render path.
- Server-authoritative for stats AND stamina — never trust the client for values.
- Systems ONLY this task — do NOT build the character-sheet panel or the allocation UI (next task).
  The stamina bar on the HUD is in-scope; the stat-allocation panel is not.
- Everything in canonical src. No place-only drift. No secrets in the result (outbox is public).
