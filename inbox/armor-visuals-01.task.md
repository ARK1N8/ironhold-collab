TASK: armor-visuals-01 — generate + attach visible armor (9 pieces, rigid welds on R15)

FROM: Studio Director
DATE: 2026-07-09
SCOPE: GAME project. Fills in the visuals for the WORKING armor system. Art pipeline + rig attachment.
DEPENDS ON: armor-system-01 (ArmorRegistry with 9 items + empty `AssetId` placeholders; equip/unequip
       already server-authoritative and working), the proven Meshy -> Open Cloud -> ShieldHandle weld
       pipeline, the Part-4 spike (R15 rigid MeshPart limbs, 0 bones/0 Motor6D -> RIGID WELDS, not
       layered clothing).
ENV: sync stable on Host 127.0.0.1; rojo serve running on 127.0.0.1:34872 — do NOT start a 2nd instance.
       rojo serve + Connect only, NEVER rojo build.

## Goal
Make equipped armor VISIBLE on the character. 9 pieces (Helmet/Chest/Legs x Leather/Iron/Steel), rigid-
welded per the spike: helmet -> Head, chest -> UpperTorso, greaves -> the rigid leg segments. Fill the
`AssetId` fields already waiting in ArmorRegistry. Do NOT change armor STATS or the damage-reduction
math — that system works; this is visuals only.

## CRITICAL — prove ONE slot end-to-end before batching
Generated meshes routinely come back wrong (the Spear had to be regenerated — original mesh was a broken
crossed double-shaft). Do NOT generate all 9, upload all 9, and only then discover the scale/orientation/
attachment is wrong nine times over.
1. Do **ONE piece first** (suggest Iron Helmet — a helmet is the easiest fit test): generate -> review ->
   upload -> weld -> scale/position on the rig -> verify it looks right on the character in Play, from
   front and side.
2. Only once that one piece is CORRECT and the pipeline is proven, batch the remaining 8.
3. If a generated mesh is wrong (proportions/orientation/scale), regenerate it — don't ship a broken mesh.
   Report any piece that needed regeneration.

## Requirements
- **Generate** the 9 pieces via Meshy (existing `tools/meshy_node.py` pipeline). Prompts should read as a
  coherent Viking/medieval SET — Leather / Iron / Steel tiers should look like tiers of the same armor
  family, not 9 unrelated objects. Report the prompts used.
- **Upload** via Roblox Open Cloud (existing `.tools/upload_assets.py`). Studio MCP cannot import local
  FBX — everything ships as GLB through Open Cloud, as with the weapons/structures.
- **Record the AssetIds**: write them into `ArmorRegistry`'s `AssetId` fields (that's what they're for) AND
  into the existing asset-id tracking JSON alongside the weapon/structure ids. REUSE, do not re-generate,
  on future runs.
- **Attach (rigid welds):** weld each piece to its rig part per the spike (helmet->Head, chest->UpperTorso,
  greaves->leg segments). Follow the proven ShieldHandle weld pattern. Per-piece scale/offset attributes so
  fit can be tuned live in Studio (mirror the degree-attribute tuning system used for weapon holds).
- **Equip/unequip visual lifecycle:** equipping an armor piece SHOWS its mesh; unequipping REMOVES it;
  swapping tiers in the same slot REPLACES (never stacks two chestplates). Ground Rule 7 applies — the
  attach code must be idempotent find-or-create, never destroy-and-recreate the character's parts, and
  re-running it must converge to ONE mesh per slot, not duplicates.
- **Persistence:** visuals must reflect equipped state on spawn/respawn (a player logging in with Steel
  chest equipped SEES it), driven by the server's equipped state — not client-guessed.
- Do NOT touch armor stats, the damage clamp, or the Armor tab's data logic. Visuals only.

## Report back (write outbox/armor-visuals-01.result.md)
- The prove-one-slot result FIRST: which piece, did the pipeline work, what fit/scale/orientation tuning
  was needed. (This is the finding that matters most.)
- The 9 AssetIds, written into ArmorRegistry + the tracking JSON. Any piece that needed regeneration.
- Attachment approach: weld points used, per-piece scale/offset attributes, and confirmation the attach is
  idempotent (equip/unequip/swap converge to one mesh per slot — no duplicate chestplates).
- Play verification: equip a full set -> all 3 pieces visible and correctly placed (screenshot front + side);
  unequip -> removed; swap tier -> replaced not stacked; respawn -> visuals persist from server state.
- Regression: armor STATS/reduction unchanged (the clamp still lands a 100-dmg hit at 40 for a full tank);
  HUD/inventory/character sheet/structure upgrades intact.
- STATUS: DONE / PARTIAL / BLOCKED. PARTIAL IS ACCEPTABLE — a proven pipeline + a few correct pieces beats
  9 rushed/broken ones. Be honest about which pieces look right and which need another pass.

## Guardrails
- rojo serve + Connect only (no 2nd instance). NEVER rojo build.
- RIGID WELDS per the spike — do NOT attempt layered clothing / cage-fitted assets.
- Ground Rule 7: idempotent attach (find-or-create). Never destroy-and-recreate character parts. No
  duplicate meshes per slot.
- Do NOT change armor stats, costs, the reduction math, or the clamp.
- REUSE existing uploaded assets; don't regenerate what already exists.
- Secrets (MESHY_API_KEY / ROBLOX_API_KEY / ROBLOX_CREATOR_ID) live in .env — never print them, never put
  them in the result (outbox is public). Refer to them by name only.
- Everything in canonical src. No place-only drift.
