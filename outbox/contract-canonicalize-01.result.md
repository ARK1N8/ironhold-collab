# RESULT: contract-canonicalize-01 — one canonical git-tracked CONTRACTS.md

FROM: Claude Code (local executor)
DATE: 2026-07-06
STATUS: **DONE** — Studio-side copy now mirrors the canonical repo v2.2.0; gate cleared first.

---

## Step 1 — DIFF FIRST (gate)
Compared **B** (untracked Studio-side `MMRPG Game\ironhold-studio\ironhold-studio\CONTRACTS.md`, 2.1.0-era,
1160 lines) against **A** (canonical `ARK1N8/IronHold` repo `CONTRACTS.md`, v2.2.0, commit `22668ed`, 1234
lines), CR-normalized.

**B contains NO unique load-bearing content.** The only lines present in B but not A (12 non-blank) are
*exactly* the pre-2.2.0 text that A deliberately supersedes:

```
*CONTRACT_VERSION = 2.1.0 — Structures, Inventory, TownHall systems added.*      (old version footer)

**Hard rule:** Platform is detected once at `ClientBootstrap.Init()` using           (old §11.1 platform
UserInputService boolean flags and written to `GameConfig.PLATFORM` (a                model — replaced in A
module-level mutable field). All controllers require GameConfig and read              by the Platform.luau /
`GameConfig.PLATFORM` — no controller performs its own platform detection.            Platform.Changed model)
`UserInputService:GetPlatform()` must NOT be called anywhere (not in
ClientBootstrap, not in controllers). Use boolean flags only:
  local isMobile  = UIS.TouchEnabled and not UIS.KeyboardEnabled
  local isConsole = UIS.GamepadEnabled and not UIS.TouchEnabled
  GameConfig.PLATFORM = isMobile and "MOBILE" or isConsole and "CONSOLE" or "DESKTOP"
Requiring ClientBootstrap from any controller is forbidden (circular dependency).

- [ ] Platform detection via GameConfig constant, not inline GetPlatform() calls (§11.1)   (old gate line)
```

These are the exact three spots reconciled in the `contract-2.2` task (version bump, §11.1 platform-model
reconcile, gate line). B has **none of A's additions** (§11.11 anti-overlap rule, Section 12 Progression &
Inventory). **Conclusion: B is fully superseded by A — an older version of the same file, safe to replace.**
Gate (step 2) cleared: no unique content → proceeded.

## Step 3 — Unification approach chosen: **3b (replace B with A's exact contents + documented sync)**
Why not the alternatives:
- **Symlink (3a "one file"):** not viable — real file symlinks don't work in this MSYS/git-bash environment
  (`ln -s` produced a non-resolving copy), and hardlinks break on `git pull`.
- **Make the whole Studio folder a git checkout (fullest 3b):** out of scope + risky. That folder is an
  untracked working copy of the entire repo (also holds `.venv/` with thousands of files, `*-backup-*`
  dirs, `.tools/`, a possibly-divergent `src/`); git-tracking it needs a big `.gitignore` and would surface
  unrelated `src/` drift — a larger change than this CONTRACTS-scoped task. Flagged as a follow-up below.

What I did (low-risk, in-scope):
1. **Overwrote B with A's exact canonical contents** — now byte-identical to `main`.
2. Added a **documented sync** so B is an explicit mirror, not a silent orphan:
   - `ironhold-studio/ironhold-studio/CONTRACTS.SYNC.md` — states CONTRACTS.md here is a MIRROR, the repo
     is canonical (v2.2.0, `22668ed`), "do not edit here — edit in the repo and re-sync."
   - `ironhold-studio/ironhold-studio/.tools/sync_contract.py` — one command re-sync: fetches the canonical
     `CONTRACTS.md` from `ARK1N8/IronHold@main` via `gh` and overwrites the mirror.

## Step 4 — Verify
- **One canonical v2.2.0 contract:** `ARK1N8/IronHold` `main` HEAD = `22668ed` (v2.2.0) — confirmed via
  `gh api`; no commit after mine.
- **Studio-side path resolves to it:** ran `python .tools/sync_contract.py` → the Studio-side `CONTRACTS.md`
  is now **byte-identical to what GitHub serves for `main`** (`gh api ... | cmp` → IDENTICAL). Its content is
  identical to the repo (CR-normalized diff vs the repo checkout = 0 lines). Version marker in the mirror =
  `CONTRACT_VERSION = 2.2.0`.
- **No silent orphan remains:** the Studio-side file is documented (CONTRACTS.SYNC.md) as a mirror and is one
  command from re-sync; it is no longer a divergent, undocumented 2.1-era copy.

## VERIFY (evidence)
- Gate: CR-normalized `diff B A` unique-to-B = the 12 pre-2.2.0 lines quoted above; zero of A's new sections
  missing from consideration.
- After canonicalize: `sync_contract.py` output "Synced canonical CONTRACTS.md from ARK1N8/IronHold@main";
  `gh api .../CONTRACTS.md | cmp - <studio CONTRACTS.md>` → IDENTICAL; repo `main` = `22668ed` v2.2.0.
- Guardrails: did not overwrite B until the diff was reported and the gate cleared; repo referenced by its
  current name `ARK1N8/IronHold`; no secrets in this file.

## STATUS: DONE

## BACK TO DIRECTOR
1. **Root issue is bigger than CONTRACTS.md:** the entire Studio-side folder
   (`MMRPG Game\ironhold-studio\ironhold-studio\`) is an **untracked working copy of the whole repo** — its
   `src/` (where the live game edits are made) is also not git-tracked and may have drifted from the repo's
   `src/`. The clean long-term fix is to make that folder the real git checkout of `ARK1N8/IronHold`
   (with a `.gitignore` for `.venv/`, backups, `.tools/` outputs). That's a larger, riskier op I left out of
   this CONTRACTS-scoped task — recommend a dedicated task for it.
2. **Sync is currently manual** (`python .tools/sync_contract.py` after a repo contract change). Automatic
   repo→mirror sync (git hook / CI) only makes sense once the folder is a real checkout (see #1).
3. **Concurrency:** two new tasks appeared together — I took this one; **`bus-claim-step-01`** (which adds the
   atomic task-claim step to prevent exactly this double-pickup) is still unclaimed and worth prioritizing.
