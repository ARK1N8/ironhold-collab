# TASK: contract-2.2 — consolidate CONTRACTS.md to v2.2.0 (systems-focused)

FROM: Studio Director
DATE: 2026-07-06
SCOPE: GAME project — edits CONTRACTS.md and commits to the GAME repo (ARK1N8/ironhold),
       NOT the collab repo. Pure documentation change; no code, no Studio, no Rojo needed.

## Goal (Option A — bump now, grow later)
Bring CONTRACTS.md current. It's at 2.1.0 and no longer describes the game we're building.
Fold in the mobile-first UI/UX contract and the systems decisions settled since 2.1, bump to
2.2.0. Keep it SYSTEMS-FOCUSED: document game systems and models. Do NOT add animation /
grip-drive / Rojo-sync / ops implementation detail — that lives in code + DIRECTOR_STATUS.md,
not the contract. Where detail isn't decided yet, mark it TBD for a later version — do NOT
invent tier ladders, stat formulas, or upgrade cost curves (those are later coordinated passes).

## Steps
1. Read the current CONTRACTS.md. Report its current CONTRACT_VERSION and its section list
   (table of contents) so we know the real structure.
2. Add the two sections below using the NEXT available section numbers after the highest
   existing one. If a UI/UX section already exists in some form, RECONCILE (merge/replace)
   rather than duplicate — and report what you merged.
3. Bump the header CONTRACT_VERSION: 2.1.0 -> 2.2.0.
4. Reconcile any existing contract text that now CONTRADICTS these settled decisions. In
   particular: if RoundShield is described anywhere as an active damage-dealing weapon/Tool,
   update it to the off-hand-hold model (see the systems section). Report every such change.
5. Commit to the GAME repo with message: "docs(contract): consolidate to v2.2.0 — UI/UX + progression/inventory systems"
   (confirm you are committing to ARK1N8/ironhold, not ironhold-collab).
6. Report the final table of contents, the new version, and everything reconciled.

======================================================================
INLINE CONTENT TO ADD  (use next available section numbers; sub-numbers shown as N.x)
======================================================================

### Section N — Mobile-First UI/UX Contract (Clean Fantasy)

Ironhold is mobile-first: touch is the primary input, mouse/keyboard and gamepad are
first-class equals, and all may be active in one session. All interactive UI conforms to
this section.

N.1  Platform principle. A single detection module (`Client/Platform.luau`) is the source of
     truth: `Platform.Get() -> "Touch"|"Mouse"|"Gamepad"`, `HasTouch()`, `HasGamepad()`,
     `Platform.Changed` signal. Detect per-input, not per-device; never assume exclusivity.
     Controllers subscribe to `Changed` and re-layout; they do not cache platform at init.

N.2  Layer model (anti-overlap rule). All UI lives in exactly three layers:
     - PERSISTENT HUD — always visible (health, level, resources, hotbar, minimap).
     - MODAL PANELS — inventory, build, character sheet, structure upgrade. **Only ONE modal
       panel is open at a time; opening one closes any other.** Modals dim/scrim the game behind.
     - TRANSIENT — tooltips, toasts, floating combat text; never block input.
     This one-modal-at-a-time rule is contractual — it is what prevents the panel overlap.

N.3  Visual language — Clean Fantasy. Dark semi-transparent panels (Color3 ~= (0,0,0),
     Transparency ~= 0.4) with weathered-gold trim accent (Color3 ~= (0.83,0.68,0.22)) and
     carved-wood / dark-metal framing. One display font for headers, one clean readable font
     for numbers/body. Consistent spacing, padding, and corner-radius tokens across every
     panel so the whole UI reads as one game.

N.4  Touch target standards. Minimum interactive target 44x44 px; primary actions 56x56;
     >= 8 px spacing between targets. Thumb-reach law: primary/frequent actions in the bottom
     third of the screen; destructive actions never inside the thumb arc and always require a
     confirm.

N.5  HUD layout (touch reference; mouse/gamepad inherit). Health+level+XP top-left; resources
     (Gold/Wood/Stone) top-right; hotbar/action bar bottom-center; movement joystick bottom-left
     (touch only); camera-pan zone the right half (does not intercept hotbar); minimap a corner.
     HUD scales with viewport; no fixed-pixel positions that clip on phones.

N.6  Modal panel standard. Consistent frame per N.3; header with title + close (X); back/Esc
     closes; open/close animation; one-at-a-time per N.2. Panels are pure views over
     server-authoritative data — they never mutate game state directly.

N.7  Feedback / juice. Every interactive element has hover, press, success, and error states.
     Events surface as toasts; combat surfaces as floating combat text.

N.8  Performance. No per-frame table/instance allocation in UI render loops; UI updates are
     event-driven, not polled.

N.9  Improvement rubric (audit lens for any UI module; flag if < 3 on any axis):
     platform parity, touch ergonomics, visual consistency (Clean Fantasy), feedback/juice,
     performance.

### Section N+1 — Progression & Inventory Systems

(N+1).1  Unified tabbed inventory. ONE inventory UI with tabs: **Structures / Weapons / Armor /
     Resources**. Server-authoritative via InventoryService; the panel is a pure view over the
     inventory snapshot (Handshake J). The Structures tab feeds the placement flow (ghost preview
     per the UI section; server-gated per Handshake I / TownHall).

(N+1).2  Structure progression — UPGRADE model. Each structure is owned as an instance and
     upgraded in place, Lvl 1 -> N. Each level costs resources (EconomyService) and improves the
     structure's stats (e.g. HP, capacity). Max upgrade level is gated by TownHall level
     (TownHallService — existing level-cap enforcement). [TBD later version: per-structure level
     curves, stat gains, and resource costs.]

(N+1).3  Weapon & Armor progression — TIER model. Weapons and armor exist as acquired tiers /
     ladders (better tier = better stats), NOT upgraded in place. The existing Weapons.luau Axe
     class (Wood -> Iron -> Steel) is the template pattern. Optional within-tier upgrade may be
     added later. [TBD later version: full tier ladders and per-tier stats — coordinated with the
     damage matrix.]

(N+1).4  Character stat progression — ALLOCATABLE points. The character earns stat points on
     level-up (ProgressionService owns XP -> level -> points). Points are spent into stat
     categories. Provisional categories (to be finalized): **Strength** (melee damage),
     **Vitality** (max health), **Speed** (movement / attack speed), **Endurance** (stamina, if a
     stamina system is added), **Defense** (flat incoming damage reduction). [TBD later version:
     final stat list, formulas, points-per-level, and respec rules — coordinated with the damage
     matrix.]

(N+1).5  Off-hand / shield model. The RoundShield is a **toggled passive off-hand hold**, not a
     Tool. It rides on the left arm (ShieldStrap weld) and coexists with a one-handed weapon Tool
     in the right hand — currently the Seax only. Equipping any two-handed weapon (BeardedAxe,
     DaneAxe, Spear, HuntingBow) auto-drops the shield. While up, the shield grants passive,
     server-authoritative incoming-damage reduction (constant SHIELD_DAMAGE_REDUCTION, default
     0.25). An active block/parry mechanic is deferred. Left-click always strikes the right-hand
     weapon; the shield never intercepts the attack input.

(N+1).6  Damage model note. Incoming-damage reduction now exists (shield, N+1.5). The Defense
     stat (N+1.4) and the forthcoming per-weapon x per-target damage matrix (character / NPC /
     structure / tree) reference each other and will be specified together in a later version.
     [Known open item to resolve in that pass: trees are currently un-fellable due to a per-hit
     cooldown firing before a tree can be reduced to 0 HP — the harvest model needs a decision.]

======================================================================
END INLINE CONTENT
======================================================================

## Report back (write outbox/contract-2.2.result.md)
- Current version + original table of contents (before change).
- New section numbers assigned; whether any existing UI section was reconciled (and how).
- Confirmation version bumped to 2.2.0.
- Every conflict reconciled (esp. RoundShield), with before/after.
- Final table of contents.
- Confirmation committed to the GAME repo (ARK1N8/ironhold) with the commit hash.
- STATUS: DONE / PARTIAL / BLOCKED + anything unexpected.

## Guardrails
- Edit CONTRACTS.md in the GAME repo only; commit to ARK1N8/ironhold, NOT ironhold-collab.
- SYSTEMS-FOCUSED: do not add animation, grip-drive, or Rojo-sync detail to the contract.
- Do NOT invent tier ladders, stat formulas, or upgrade cost curves — mark them TBD as written.
- Preserve Sections 1 through 10 unchanged except the explicit conflict reconciliations in step 4.
- No secrets in the result (outbox is public).
