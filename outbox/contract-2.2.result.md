# RESULT: contract-2.2 — consolidate CONTRACTS.md to v2.2.0

FROM: Claude Code (local executor)
DATE: 2026-07-06
STATUS: **DONE** — committed + pushed to the GAME repo.

---

## Setup note
There was **no local clone of the GAME repo**. The `CONTRACTS.md` under the Studio project
(`MMRPG Game/ironhold-studio/ironhold-studio/`) is **not** git-tracked (no `.git` up the tree),
and it differs from the repo copy. So I cloned **`ARK1N8/ironhold`** fresh (`gh` authed as ARK1N8),
edited the repo's `CONTRACTS.md`, and committed there. (Heads-up: the repo has been **renamed
`ironhold` → `IronHold`** on GitHub — the old URL still redirects, push succeeded.)

## 1. Current version + original TOC (before change)
The doc was in a **mixed/partial state**: the base contract footer read `CONTRACT_VERSION = 2.1.0`,
but a **Section 11 (UI/UX) had already been added and labeled "v2.2"** — the base version was never
bumped to match. Effective version: **2.1.0 base + an un-consolidated 2.2 UI section.**

Original table of contents:
- 0 Ground Rules · 1 Role Ownership Map · 2 Shared Data Structures (2.1–2.15) · 3 Module Public APIs
  (3.1–3.17) · 4 Net Remote Table · 5 Anti-Cheat Rules · 6 Load-Bearing Handshakes (A–J) · 7 Rojo File
  Tree · 8 Director's Gate Checklist · 9 World & Environment Guidance · 10 Structure Definitions
- **Section 11 — UI/UX Contracts v2.2** (11.1 Platform & Input · 11.2 Touch/Tap Targets · 11.3 HUD
  Layout · 11.4 Inventory Panel · 11.5 Structure Placement · 11.6 Combat Feel · 11.7 Building Feel ·
  11.8 Improvement Rubric · 11.9 Quality References · 11.10 Director's Gate Additions for v2.2)

## 2. New section numbers / UI reconciliation
- **UI/UX ("Section N" in the task): RECONCILED into the existing Section 11 — not duplicated.**
  - Task **N.1 (platform model)** *contradicted* existing 11.1 → reconciled (see §4).
  - Task **N.2 (3-layer model + one-modal-at-a-time anti-overlap rule)** was **absent** → added as new
    **§11.11 — Modal Layer Model & Anti-Overlap Rule**, plus a Gate checkbox.
  - Task **N.3–N.9** (Clean Fantasy visual language, touch-target standards, HUD layout, feedback/juice,
    performance, improvement rubric) were **already covered** by existing 11.2 / 11.3 / 11.4 / 11.6 /
    11.8 / 11.9 → kept as-is, no duplication.
- **Progression & Inventory ("Section N+1"): ADDED as the next available number — new Section 12**
  (12.1 Unified Tabbed Inventory · 12.2 Structure UPGRADE model · 12.3 Weapon/Armor TIER model · 12.4
  Character ALLOCATABLE stat points · 12.5 Off-hand/Shield model · 12.6 Damage-model note). All
  undecided detail marked **[TBD later version]** as written — no tier ladders / stat formulas / cost
  curves invented.

## 3. Version bump
`CONTRACT_VERSION` is now **2.2.0** consistently: base footer (was 2.1.0), the Section 11 header, the
new Section 12 header, and the final footer. The only remaining "2.1.0" is the descriptive phrase
"Sections 0–10 unchanged from 2.1.0" (intentional).

## 4. Conflicts reconciled (with before/after)
**(a) §11.1 Platform detection model** — real contradiction with task N.1; reconciled to the settled
decision:
- **BEFORE:** "Platform is detected **once** at `ClientBootstrap.Init()` … written to
  `GameConfig.PLATFORM` (mutable field). Controllers read `GameConfig.PLATFORM` — no controller
  performs its own detection. `GetPlatform()` must NOT be called." (detect-once / cache model)
- **AFTER:** "A single module **`Client/Platform.luau`** is the source of truth — `Platform.Get()`,
  `HasTouch()`, `HasGamepad()`, `Platform.Changed`. Detect **per active input, not per device**; never
  assume exclusivity. Controllers subscribe to `Platform.Changed` and **re-layout on change**; they do
  NOT cache at init. `GetPlatform()` still forbidden." (Also updated the matching §11.10 Gate line.)

**(b) RoundShield** — the task asked to update any place RoundShield is described as an active
damage-dealing weapon/Tool. **Searched the whole contract for "shield"/"roundshield" → 0 hits.**
RoundShield was never in CONTRACTS.md, so there was nothing to reconcile in the contract; the correct
off-hand model is documented **fresh in §12.5** (toggled passive off-hand hold, coexists with the
one-handed Seax, two-handers auto-drop it, passive server-authoritative `SHIELD_DAMAGE_REDUCTION`
= 0.25, left-click always strikes the right-hand weapon). See BACK TO DIRECTOR for the code side.

## 5. Final table of contents
0–10 (unchanged) · **11 UI/UX Contracts v2.2** → 11.1 (reconciled) · 11.2 · 11.3 · 11.4 · 11.5 · 11.6 ·
11.7 · 11.8 · 11.9 · 11.10 (Gate, +2 items) · **11.11 Modal Layer Model & Anti-Overlap Rule (new)** ·
**12 Progression & Inventory Systems (new)** → 12.1 Unified Tabbed Inventory · 12.2 Structure UPGRADE ·
12.3 Weapon/Armor TIER · 12.4 Character ALLOCATABLE points · 12.5 Off-hand/Shield · 12.6 Damage-model
note.

## 6. Commit
- Repo: **ARK1N8/ironhold** (now `IronHold`), branch **main**. Confirmed NOT the collab repo.
- Message: `docs(contract): consolidate to v2.2.0 — UI/UX + progression/inventory systems`
- **Commit hash: `22668ed4c499ca486406ec7710d02e52da650d68`** — pushed (`21572c1..22668ed`).
- Change scope: `CONTRACTS.md` only, 86 insertions / 12 deletions (no mass line-ending rewrite).

## VERIFY
Post-edit greps confirmed: version = 2.2.0 in all 4 markers; §11.11 + Section 12 (12.1–12.6) present;
`Client/Platform.luau` referenced in 11.1 + the Gate; no stray unbumped "2.1.0" except the intentional
phrase. `git push` returned `21572c1..22668ed main -> main`. No secrets in this file.

## STATUS: DONE

## BACK TO DIRECTOR
1. **Section 11 already existed as "v2.2" UI/UX** (added by a prior pass) while the base footer was
   still 2.1.0 — the doc was inconsistent. It's now consolidated at 2.2.0. The task's "Section N" UI
   content was therefore reconciled into Section 11 rather than added as a new section.
2. **§11.1 reconciliation creates a contract↔code gap.** The contract now mandates a
   `Client/Platform.luau` module with a `Platform.Changed` signal + re-layout, superseding the current
   code (`ClientBootstrap` writes `GameConfig.PLATFORM`, controllers read the cached value). That's a
   client refactor — flagging for a follow-up code task; I changed only the contract per this task's
   docs-only scope.
3. **RoundShield code, not contract.** `Weapons.luau` still lists `RoundShield` with `Damage = 6,
   Class = "Blunt"` (a weapon entry). That's consistent with the off-hand model only once the
   `shield-offhand-01` implementation lands (which removes it from the Tool system). Contract §12.5 now
   describes the target; the code entry is the open item for that task.
4. **Scope reminder:** `shield-offhand-01` is still in the inbox, paused mid-implementation (its `Hands`
   field addition to Weapons.luau — which §12.5 references — is already done and live).
