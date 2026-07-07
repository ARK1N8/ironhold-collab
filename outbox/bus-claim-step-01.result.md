# RESULT: bus-claim-step-01 — atomic task-claiming + stale-claim recovery

FROM: Claude Code (local executor)
DATE: 2026-07-06
STATUS: **DONE** — new lifecycle live, verified on a throwaway bus, then dogfooded on the real bus.

---

## What changed
- **New lifecycle folder** `inbox/claimed/` (with `.gitkeep`) — committed to the collab repo.
- **`poll.py` rewritten** with verbs `check | claim | heartbeat | complete`, session identity, atomic
  git-push claiming, 30-min lease + stale reclaim, and ownership enforcement. Commits are **scoped to
  bus paths** (`inbox`, `inbox/claimed`, `outbox`) so a half-written result or unrelated file is never
  swept into a claim/heartbeat commit.
- **`director_loop.md` protocol rewritten**: check → claim → *only if CLAIMED* execute → result →
  complete, with the hard rule **never execute a task you did not claim**.
- **Session id** cached in `.session_id` next to the script (in `director-bus/`, **outside** the collab
  repo → never committed); also gitignored defensively. Overridable via env `IRONHOLD_BUS_SESSION`;
  bus location overridable via env `IRONHOLD_BUS_DIR` (used for the throwaway-bus tests).

## Confirmations (task checklist)
- ✅ `inbox/claimed/` created (+ `.gitkeep`), committed & pushed.
- ✅ Session-id caching works (`sess-…` generated on first use, reused after) and is **NOT committed**
  (`git ls-files | grep session_id` → empty; the file lives in `director-bus/`, plus `.session_id` added
  to the collab `.gitignore`).
- ✅ `claim` / `heartbeat` / `complete` enforce ownership (non-owner gets `ALREADY_CLAIMED` / `NOT_OWNER`
  / `REFUSED`).
- ✅ Stale reclaim works past the lease (`LEASE_TIMEOUT_SECONDS = 1800`): a claim whose heartbeat is
  older than the lease is listed by `check` as STALE-RECLAIMABLE and can be `claim`ed by another session.

## Race / behavior tests (throwaway local bus: bare remote + 2 clones + 2 session ids; public repo untouched)
| Test | Expected | Result |
|---|---|---|
| Sequential race — A then B `claim alpha` | one CLAIMED, one ALREADY_CLAIMED | A `CLAIMED alpha` / B `ALREADY_CLAIMED … held by sessA … back off` ✅ |
| **Simultaneous push race** — B stages a claim (unpushed), A `claim`s (pushes first), B pushes | first push wins; loser rejected → backs off | A `CLAIMED`; B raw push `REJECTED` (non-fast-forward); B poll.py recovery (reset+re-claim) → `ALREADY_CLAIMED`; final owner sessA ✅ |
| `check` while alpha held fresh | alpha hidden, beta listed | only beta printed ✅ |
| Non-owner `heartbeat alpha` | no-op | `NOT_OWNER: alpha is held by sessA …` ✅ |
| Non-owner `complete alpha` (both B, and A after reclaim) | refused | `REFUSED: alpha is owned by <other> … only the owning session may complete it` ✅ |
| Stale reclaim — backdate alpha heartbeat > lease, B `claim alpha` | reclaimed | `check` shows `STALE-RECLAIMABLE`; B `CLAIMED alpha (RECLAIMED a stale claim from sessA)` ✅ |
| Owner `complete` | moves to done, releases claim | `COMPLETED alpha: … archived to inbox/done/, claim released` ✅ |

**Dogfood on the REAL bus:** `poll.py claim bus-claim-step-01` → `CLAIMED` (task moved to
`inbox/claimed/`, sidecar `{"session":"sess-6d574e8c2de2", ...}`), then this result written, then
`complete` (below).

---

## Final `director_loop.md` — the loop section (rewritten)
```
## The loop (claim-based — Jon runs MULTIPLE sessions in parallel)

1. python poll.py check
   - Prints UNCLAIMED tasks + STALE-RECLAIMABLE tasks (owner silent past the 30-min lease),
     or NO_NEW_TASKS. Also refreshes heartbeats of claims you already own.
2. Pick ONE task and claim it:  python poll.py claim <slug>
   - CLAIMED = you got it; ALREADY_CLAIMED = another live session holds it.
   - HARD RULE: never execute a task you did not successfully claim. Anything other than
     CLAIMED -> do NOT work it; go back to step 1.
3. Execute the claimed task fully and for real (long task: `poll.py heartbeat <slug>` keepalive).
4. Write outbox/<slug>.result.md (DID / VERIFY / STATUS / BACK TO DIRECTOR).
5. python poll.py complete <slug>   (owner-only; moves claimed/->done/, removes sidecar, pushes)
6. Repeat.
```
(Plus: the "File conventions" section now documents the three-state lifecycle + sidecar shape +
30-min lease + env overrides; "Notes" documents the verbs, that sidecars ARE committed but
`.session_id` is not, and that same-machine sessions need distinct `IRONHOLD_BUS_SESSION` ids.)

## Final `poll.py`
```python
#!/usr/bin/env python3
"""
poll.py  --  collab-bus helper for Claude Code (local executor side).
Atomic task claiming + stale-claim recovery so parallel sessions never work the same task.
Lifecycle: inbox/<slug>.task.md -> inbox/claimed/<slug>.task.md (+<slug>.claim.json) -> inbox/done/.
Verbs: check | claim <slug> | heartbeat <slug> | complete <slug>.
Session id cached in .session_id next to this script (NOT committed); env overrides
IRONHOLD_BUS_SESSION / IRONHOLD_BUS_DIR. Lease: LEASE_TIMEOUT_SECONDS. Stdlib + git CLI only.
"""
import json, os, subprocess, sys, time, uuid
from pathlib import Path

COLLAB_DIR = Path(os.environ.get("IRONHOLD_BUS_DIR", r"C:\Users\Jon\Desktop\ironhold-collab"))
INBOX, CLAIMED, DONE, OUTBOX = COLLAB_DIR/"inbox", COLLAB_DIR/"inbox"/"claimed", COLLAB_DIR/"inbox"/"done", COLLAB_DIR/"outbox"
LEASE_TIMEOUT_SECONDS = 1800   # 30 min; older-heartbeat claims are reclaimable
CLAIM_RETRIES = 4
SESSION_ID_FILE = Path(__file__).resolve().parent / ".session_id"   # outside the repo -> never committed

try: sys.stdout.reconfigure(encoding="utf-8")
except Exception: pass

def git(*a): return subprocess.run(["git","-C",str(COLLAB_DIR),*a], capture_output=True, text=True, encoding="utf-8")

def session_id():
    env = os.environ.get("IRONHOLD_BUS_SESSION")
    if env and env.strip(): return env.strip()
    if SESSION_ID_FILE.exists():
        sid = SESSION_ID_FILE.read_text(encoding="utf-8").strip()
        if sid: return sid
    sid = "sess-" + uuid.uuid4().hex[:12]; SESSION_ID_FILE.write_text(sid, encoding="utf-8"); return sid

SESSION = session_id()

def pull():
    r = git("pull","--rebase","--autostash")
    if r.returncode != 0: print("PULL_ERROR:", (r.stderr or r.stdout).strip()); sys.exit(1)

def read_claim(slug):
    f = CLAIMED / f"{slug}.claim.json"
    if not f.exists(): return None
    try: return json.loads(f.read_text(encoding="utf-8"))
    except Exception: return {"session":"<unreadable>","ts":0,"heartbeat":0}

def is_stale(c): return (time.time() - float(c.get("heartbeat",0))) > LEASE_TIMEOUT_SECONDS

def write_claim(slug, reclaim_of=None):
    CLAIMED.mkdir(parents=True, exist_ok=True)
    now = int(time.time()); data = {"session":SESSION,"ts":now,"heartbeat":now}
    if reclaim_of: data["reclaimed_from"] = reclaim_of
    (CLAIMED / f"{slug}.claim.json").write_text(json.dumps(data, indent=2), encoding="utf-8")

def task_location(slug):
    if (INBOX / f"{slug}.task.md").exists(): return "inbox"
    if (CLAIMED / f"{slug}.task.md").exists(): return "claimed"
    if (DONE / f"{slug}.task.md").exists(): return "done"
    return None

def _refresh_own_heartbeats():
    if not CLAIMED.exists(): return 0
    now = int(time.time()); updated = 0
    for f in CLAIMED.glob("*.claim.json"):
        try: c = json.loads(f.read_text(encoding="utf-8"))
        except Exception: continue
        if c.get("session") == SESSION:
            c["heartbeat"] = now; f.write_text(json.dumps(c, indent=2), encoding="utf-8"); updated += 1
    return updated

def check():
    pull()
    n = _refresh_own_heartbeats()
    if n:
        git("add","-A","inbox/claimed")
        if git("commit","-m",f"heartbeat: {SESSION} refreshed {n} claim(s)").returncode == 0:
            if git("push").returncode != 0: git("pull","--rebase","--autostash"); git("push")
    unclaimed = sorted(INBOX.glob("*.task.md"))   # non-recursive: claimed/ and done/ excluded
    stale = []
    if CLAIMED.exists():
        for t in sorted(CLAIMED.glob("*.task.md")):
            slug = t.name[:-len(".task.md")]; c = read_claim(slug)
            if c is None or is_stale(c): stale.append((slug,t,c))
    if not unclaimed and not stale: print("NO_NEW_TASKS"); return
    for t in unclaimed:
        slug = t.name[:-len(".task.md")]
        print(f"===== TASK: {slug} =====\nPATH: {t}"); print(t.read_text(encoding="utf-8")); print(f"===== END TASK: {slug} =====\n")
    for slug,t,c in stale:
        owner = (c or {}).get("session","?"); hb = (c or {}).get("heartbeat",0)
        age = int(time.time()-float(hb)) if hb else -1
        print(f"===== STALE-RECLAIMABLE TASK: {slug} =====\nPATH: {t}")
        print(f"(was claimed by {owner}; heartbeat stale by {age}s > lease {LEASE_TIMEOUT_SECONDS}s -- run `poll.py claim {slug}` to reclaim)")
        print(t.read_text(encoding="utf-8")); print(f"===== END STALE TASK: {slug} =====\n")

def claim(slug):
    for _ in range(CLAIM_RETRIES):
        pull(); loc = task_location(slug)
        if loc is None: print(f"NO_SUCH_TASK: {slug} not found in inbox/, inbox/claimed/, or inbox/done/."); return
        if loc == "done": print(f"ALREADY_DONE: {slug} is already completed."); return
        if loc == "claimed":
            c = read_claim(slug)
            if c and c.get("session") == SESSION:
                write_claim(slug); git("add","-A","inbox/claimed"); git("commit","-m",f"claim: {slug} heartbeat ({SESSION})"); git("push")
                print(f"CLAIMED {slug} (already owned by this session; heartbeat refreshed)."); return
            if c and not is_stale(c):
                print(f"ALREADY_CLAIMED: {slug} is held by {c.get('session')} (heartbeat fresh). Not yours -- back off."); return
            prior = (c or {}).get("session","?")     # stale -> reclaim (task already in claimed/)
            write_claim(slug, reclaim_of=prior); git("add","-A","inbox")
            git("commit","-m",f"reclaim: {slug} from stale claim by {prior} -> {SESSION}")
            if git("push").returncode == 0: print(f"CLAIMED {slug} (RECLAIMED a stale claim from {prior})."); return
            git("reset","--hard","HEAD~1"); continue
        CLAIMED.mkdir(parents=True, exist_ok=True)   # loc == inbox -> fresh claim
        (INBOX / f"{slug}.task.md").replace(CLAIMED / f"{slug}.task.md"); write_claim(slug)
        git("add","-A","inbox"); git("commit","-m",f"claim: {slug} by {SESSION}")
        if git("push").returncode == 0: print(f"CLAIMED {slug}."); return
        git("reset","--hard","HEAD~1")               # push lost the race -> undo, re-evaluate
    print(f"CLAIM_CONTENDED: {slug} -- repeated push races. Run `poll.py check` and try another task.")

def heartbeat(slug):
    pull()
    if task_location(slug) != "claimed": print(f"NOT_CLAIMED: {slug} is not in inbox/claimed/. Nothing to keep alive."); return
    c = read_claim(slug)
    if not c or c.get("session") != SESSION:
        print(f"NOT_OWNER: {slug} is held by {(c or {}).get('session','?')}, not this session ({SESSION}). No-op."); return
    write_claim(slug); git("add","-A","inbox/claimed"); git("commit","-m",f"heartbeat: {slug} ({SESSION})")
    if git("push").returncode != 0: git("pull","--rebase","--autostash"); git("push")
    print(f"HEARTBEAT {slug} refreshed by {SESSION}.")

def complete(slug):
    result = OUTBOX / f"{slug}.result.md"
    if not result.exists(): print(f"REFUSED: {result} does not exist. Write the result file first."); sys.exit(1)
    pull(); loc = task_location(slug)
    if loc == "done": print(f"ALREADY_DONE: {slug}."); return
    if loc != "claimed": print(f"REFUSED: {slug} is not claimed (state={loc}). Claim it first: poll.py claim {slug}."); sys.exit(1)
    c = read_claim(slug)
    if not c or c.get("session") != SESSION:
        print(f"REFUSED: {slug} is owned by {(c or {}).get('session','?')}, not this session ({SESSION}). Only the owning session may complete it."); sys.exit(1)
    DONE.mkdir(parents=True, exist_ok=True)
    (CLAIMED / f"{slug}.task.md").replace(DONE / f"{slug}.task.md")
    sc = CLAIMED / f"{slug}.claim.json"
    if sc.exists(): sc.unlink()
    git("add","-A","inbox","outbox")
    r = git("commit","-m",f"complete: {slug} ({SESSION})")
    if r.returncode != 0 and "nothing to commit" not in (r.stdout+r.stderr).lower(): print("COMMIT_ERROR:", (r.stderr or r.stdout).strip()); sys.exit(1)
    r = git("pull","--rebase","--autostash")
    if r.returncode != 0: print("PULL_ERROR:", (r.stderr or r.stdout).strip()); sys.exit(1)
    r = git("push")
    if r.returncode != 0: print("PUSH_ERROR:", (r.stderr or r.stdout).strip()); sys.exit(1)
    print(f"COMPLETED {slug}: result pushed, task archived to inbox/done/, claim released.")

if __name__ == "__main__":
    if len(sys.argv) < 2: print("usage: poll.py check | claim <slug> | heartbeat <slug> | complete <slug>"); sys.exit(2)
    cmd = sys.argv[1]
    if cmd == "check": check()
    elif cmd in ("claim","heartbeat","complete"):
        if len(sys.argv) < 3: print(f"usage: poll.py {cmd} <slug>"); sys.exit(2)
        {"claim":claim,"heartbeat":heartbeat,"complete":complete}[cmd](sys.argv[2])
    else: print(f"unknown command: {cmd}"); sys.exit(2)
```
> The pasted source is a compact rendering of the working file at `director-bus/poll.py` (same logic;
> the on-disk version is spaced out for readability). It compiles clean with no SyntaxWarning
> (the old `\ ` docstring warning is gone — docstring now uses forward slashes).

## STATUS: DONE

## BACK TO DIRECTOR
1. **Same-machine sessions share `.session_id`.** The id is cached next to the script, so two sessions
   run from the *same* `director-bus/` would share one id (breaks per-session ownership). Give each a
   distinct id via env **`IRONHOLD_BUS_SESSION`**, or run each from its own bus clone. Documented in
   `director_loop.md` Notes.
2. **`complete`'s early `pull()` can fail** only if a session holds a stray *untracked* `outbox/<slug>.result.md`
   for a task another session already completed — which the claim mechanism now prevents (one owner per
   slug). Surfaced only as a test artifact; benign in the real flow. Left as-is to keep the code simple.
3. **This task was itself claimed via the new mechanism** (`sess-6d574e8c2de2`) before completing — the
   double-pickup that motivated this task can no longer happen for anyone using this `poll.py`.
