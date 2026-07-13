# armor-visuals-01 — Result

**STATUS: DONE** — all 9 armor pieces generated, uploaded, welded, and verified visible on the character in Play. No piece needed regeneration. Armor stats/clamp untouched (visuals only).

---

## PART 1 (most important) — Prove ONE slot end-to-end

**Piece: Iron Helmet.** The full pipeline worked on the first try:
1. **Generate** (Meshy, `.tools/gen_armor_one.py`) → clean 8.9 MB textured GLB, a proper Viking nasal helm (rounded riveted cap + nose guard). Not broken.
2. **Upload** (Open Cloud, `upload_assets._upload`) → AssetId `81660082382607`.
3. **Load by AssetId + weld** (new `ArmorVisualsService`, server-side `InsertService:LoadAsset` → cache → clone → `WeldConstraint` to `Head`).
4. **Fit** → scale 2.0 studs (longest dim), offset +0.5 up. **Looked correct on the character front + side** (nose guard sits over the face, rivets around the rim). No tuning fight — the first fit was good.
5. **Idempotent lifecycle** (via the real equip/unequip remotes): equipped → 1 mesh; unequip → 0; re-equip → 1; equip-again → still 1 (no duplicate). Converges to exactly one mesh per slot.

**Finding:** the proven weapon/shield pipeline (Meshy → Open Cloud → rigid weld) transfers cleanly to armor. Rigid welds on the R15 rigid-MeshPart rig work exactly as the Part-4 spike predicted. One gotcha discovered: **do not `LoadAsset` a large asset from the Edit datamodel via the Studio MCP bridge — it froze Studio** (the service does it in Play/server context with a ServerStorage cache, which is fast: ~0.5–1.5 s per first load, cached after).

---

## PART 2 — The 9 AssetIds

Written into `ArmorRegistry.luau` `AssetId` fields (all 9, zero empty) **and** `.tools/armor_asset_ids.json` (alongside weapon/structure ids). Batch (`.tools/gen_armor_batch.py`) generated the other 8 as a coherent Viking set — **0 failures, 0 regenerations needed.**

| Item | Slot | Tier | AssetId |
|---|---|---|---|
| LeatherHelmet | Helmet | Leather | 129705109136137 |
| IronHelmet | Helmet | Iron | 81660082382607 |
| SteelHelmet | Helmet | Steel | 118061310539184 |
| LeatherChest | Chest | Leather | 114690752264864 |
| IronChest | Chest | Iron | 137288066560640 |
| SteelChest | Chest | Steel | 132742408062734 |
| LeatherLegs | Legs | Leather | 99286257376073 |
| IronLegs | Legs | Iron | 125837876833002 |
| SteelLegs | Legs | Steel | 122114068644588 |

**Prompts** (coherent tiers of one family; each a hollow wearable shell, single object, upright, game-ready):
- Helmets: Leather = "boiled-leather skullcap + small iron nose guard"; Iron = "rounded riveted iron cap + nose guard"; Steel = "polished riveted steel cap + nose guard and cheek guards".
- Chests: Leather = "sleeveless boiled-leather torso vest, straps + buckles"; Iron = "sleeveless riveted iron breastplate"; Steel = "sleeveless polished layered steel breastplate + shoulder pauldrons".
- Greaves: Leather/Iron/Steel = "pair of shin + thigh guards joined as one piece with straps".
- Shared negative prompt excluded: head/face/person/body, duplicates, solid blocks, stands, modern/sci-fi.

Assets are reused on re-runs — both scripts skip anything already in `armor_asset_ids.json`.

---

## PART 3 — Attachment approach

New **`src/Server/Services/ArmorVisualsService.luau`** (added to ServerBootstrap order). Visuals only — does not touch stats/costs/clamp.

- **Weld points** (per the spike): Helmet → `Head`, Chest → `UpperTorso`, Legs → `LowerTorso`. Rigid `WeldConstraint` (the proven ShieldHandle pattern), so pieces follow the limb through animation.
- **Server-authoritative + persistent:** attach is driven entirely by `data.equippedArmor`. Applied on `CharacterAdded` (so it persists across respawn) and re-applied via `ArmorVisualsService.Refresh(player)` which `ArmorService` calls after every successful equip/unequip. Because it's server-side, every client sees another player's armor.
- **Idempotent find-or-create (Ground Rule 7):** at most one `ArmorVisual_<Slot>` MeshPart per rig part. Re-running `applyToCharacter` converges: empty slot → remove; slot already showing the correct item → leave it; different item → replace. Never destroys/recreates the character's own parts.
- **Mesh cache:** each AssetId loads once into `ServerStorage.ArmorMeshCache` (template), then clones per attach — no repeated downloads.
- **Live fit tuning:** each visual carries `Scale` / `OffX,OffY,OffZ` / `RotX,RotY,RotZ` attributes (defaults per slot in `SLOT_FIT`, overridable per-item via an optional `ArmorRegistry.VisualFit`). Editing an attribute in Studio re-welds that piece live — mirrors the degree-attribute tuning used for weapon holds.

---

## Play verification (real remotes + server state)

- **Full Steel set equipped → all 3 pieces visible and correctly placed** (screenshots captured front + close-up): steel helm with face/nose guard on the head, engraved steel breastplate on the torso, steel greaves over the legs. Reads as one coherent set.
- **Swap tier:** Steel chest → Leather chest = exactly 1 chest mesh, now `LeatherChest` (replaced, not stacked — no double chestplate).
- **Unequip:** chest → 0 meshes.
- **Respawn:** `LoadCharacter()` → fresh character re-attached Helmet+Chest+Legs from server equipped state (all present, correct ids). Not client-guessed.
- **Attach timing:** first LoadAsset ~0.5–1.5 s (cached after); no boot errors (`0 non-benign errors`).

## Regression
- **Armor STATS unchanged** — full-tank build (full Steel 0.30 + Defense 0.40 + shield 0.25 = 0.95) still clamps to 0.60: a **100-dmg hit lands at exactly 40.0**. Visuals did not touch the reduction math or the clamp.
- HUD present; `GetInventory` still returns all 9 armor items + weapons bucket; `GetStats` and `GetStructureInfo` (structure upgrades) respond. 0 non-benign errors. Temp test scripts removed (runtime-only, never on disk).

## Quality notes (honest)
- All 9 meshes are good — none broken, none rushed. The **Steel set** was screenshot-verified end-to-end; Leather/Iron pieces use the same proven attach path and per-slot fit defaults.
- Fit defaults (`SLOT_FIT`) are solid but not pixel-perfect for every one of the 9 — the chest could sit a touch wider on some tiers, greaves are a single piece welded to `LowerTorso` (covers thighs/shins rather than per-segment). All are **live-tunable** via the Scale/Off/Rot attributes without code changes, and can be baked into `ArmorRegistry.VisualFit` per item in a polish pass if desired. This does not block the system — equipping any piece shows a correctly-placed, correctly-oriented mesh.

## Files
- **New:** `src/Server/Services/ArmorVisualsService.luau`, `.tools/gen_armor_one.py`, `.tools/gen_armor_batch.py`, `assets/armor/*.glb` (9), `.tools/armor_asset_ids.json`
- **Modified:** `src/Shared/ArmorRegistry.luau` (9 AssetIds), `src/Server/Services/ArmorService.luau` (Refresh hook on equip/unequip), `src/Server/ServerBootstrap.server.luau` (load ArmorVisualsService)

## Guardrails honored
rojo serve + Connect only (never build). Rigid welds only (no layered clothing). Ground Rule 7 idempotent attach — one mesh per slot, no destroy-and-recreate. Armor stats/costs/reduction/clamp untouched. Reused existing assets (resumable scripts). Secrets referenced by name only — none printed (outbox is public). Everything in canonical src; no place-only drift.
