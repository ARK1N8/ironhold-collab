TASK: contract-canonicalize-01 — make the git repo CONTRACTS.md the single source of truth

FROM: Studio Director
DATE: 2026-07-06
SCOPE: GAME project. Unifies the two divergent CONTRACTS.md files onto one canonical, git-tracked copy.

## Context
Two CONTRACTS.md currently exist and differ:
  (A) GIT-TRACKED, in the ARK1N8/IronHold repo — just consolidated to v2.2.0 (commit 22668ed). CANONICAL.
  (B) UNTRACKED, at  MMRPG Game\ironhold-studio\ironhold-studio\CONTRACTS.md  — 2.1-era, read by the
      day-to-day Studio workflow, NOT under version control.
Goal: make (A) the one and only contract, and make the Studio-side location resolve to it, so there is
exactly one version-controlled source of truth.

## Steps
1. DIFF FIRST (do not overwrite yet). Compare the untracked Studio-side copy (B) against the repo
   copy (A, v2.2.0). Report:
   - Whether B contains ANY content NOT present in A (unique sections, edits, notes, decisions).
   - If B is simply older/2.1-era text fully superseded by A, say so explicitly.
   - Paste any unique-to-B content so the Director can judge whether it's load-bearing before it's lost.
2. STOP IF UNIQUE CONTENT EXISTS. If the diff shows content in B that isn't in A and isn't obviously
   just older versions of the same thing, write the result as PARTIAL/BLOCKED with that content quoted,
   and do NOT overwrite. Wait for Director direction.
3. IF B IS FULLY SUPERSEDED (no unique load-bearing content): make the repo copy canonical at the
   Studio-side location. Preferred approach, in order of preference:
   a. Make the game project reference the repo clone's CONTRACTS.md directly (so there's literally one
      file), OR
   b. If the workflow needs the file physically at the Studio-side path, replace B with A's exact
      contents AND bring that path under the repo (or a documented sync), so it's no longer an
      untracked orphan.
   Use your judgment on which is cleanest for how the project is laid out; report what you chose and why.
4. Verify: confirm there is now one canonical v2.2.0 contract, and that the Studio-side path resolves
   to it (same content, and either tracked or a direct reference — not a silent untracked copy).

## Report back (write outbox/contract-canonicalize-01.result.md)
- The A-vs-B diff summary + any unique-to-B content quoted.
- Which unification approach you took (reference vs replace-and-track) and why.
- Confirmation: one canonical v2.2.0 contract; Studio-side path now resolves to it; no orphan untracked copy remains.
- STATUS: DONE / PARTIAL / BLOCKED + anything unexpected.

## Guardrails
- Do NOT overwrite the Studio-side copy until the diff is reported and it's confirmed to have no unique
  load-bearing content (step 2 gate).
- Repo is ARK1N8/IronHold (recently renamed from ironhold — use the current name).
- No secrets in the result (outbox is public).
