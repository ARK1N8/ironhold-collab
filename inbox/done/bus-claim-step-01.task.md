TASK: bus-claim-step-01 — add atomic task-claiming + stale-claim recovery to the collab bus

FROM: Studio Director
DATE: 2026-07-06
SCOPE: Collaboration BUS mechanics. Edits poll.py and director_loop.md in the director-bus folder
       (C:\Users\Jon\Desktop\director-bus). Does NOT touch the game project.

## Why
Jon runs MULTIPLE Claude Code sessions in parallel on this bus. Today poll.py has no claim step:
every session's `check` sees every inbox task, so two sessions can pick up the same task at once
(this already happened once with shield-offhand-01 — resolved only by luck). Add an atomic claim
step so exactly one session owns a task, plus stale-claim recovery so a dead session never locks a
task forever.

## New task lifecycle (three states)
  inbox/<slug>.task.md            -> available, unclaimed
  inbox/claimed/<slug>.task.md    -> owned by one session (+ sidecar claim file)
  inbox/done/<slug>.task.md       -> completed
Create the inbox/claimed/ folder (with .gitkeep).

## Claim mechanism (atomicity via git)
A claim = move the task file inbox/ -> inbox/claimed/, write a sidecar
inbox/claimed/<slug>.claim.json = { "session": "<id>", "ts": <unix>, "heartbeat": <unix> },
commit, and PUSH. Git push is atomic: if two sessions race, the first push wins; the loser's push
is rejected (non-fast-forward) OR after pull it sees the task already in claimed/ -> it backs off.
A session works ONLY a task it successfully claimed.

## Session identity
poll.py generates a stable per-session id on first use, cached to a local file next to the script
(e.g. .session_id, NOT committed to the repo — add to .gitignore or keep outside the repo). All
claims/heartbeats/completes use this id.

## Stale-claim recovery
- LEASE_TIMEOUT_SECONDS default 1800 (30 min), as a tunable constant at the top of poll.py.
- A claim is "stale" if now - heartbeat > LEASE_TIMEOUT_SECONDS.
- Any session may reclaim a stale task (overwrite the sidecar with its own id/ts, push). Report when
  a reclaim happens.
- The working session updates its claim's heartbeat each time it calls `check` or a new `heartbeat`
  verb (see below), so long-but-alive tasks aren't yanked.

## poll.py verbs (update)
- `check`      -> pull; list only UNCLAIMED tasks in inbox/ (top level). Also list any STALE claimed
                  tasks as reclaimable. Refresh this session's own heartbeats while here.
- `claim <slug>` -> pull; if slug still unclaimed OR stale, move to claimed/, write sidecar with this
                  session's id, commit, push. On success print CLAIMED. On race/already-owned-by-other
                  print ALREADY_CLAIMED and do nothing.
- `heartbeat <slug>` -> update this session's claim ts on that task (optional explicit keepalive for
                  long tasks), commit+push. No-op with a clear message if not owner.
- `complete <slug>` -> require outbox/<slug>.result.md exists AND this session owns the claim; move
                  claimed/<slug>.task.md -> done/, remove the sidecar, commit, pull --rebase, push.
                  Refuse (clear message) if the caller isn't the owning session.
- Keep all existing pull --rebase --autostash safety and error handling.

## director_loop.md (update the protocol)
Rewrite the loop to: `check` -> pick an unclaimed (or stale) task -> `claim <slug>` -> ONLY IF the
claim printed CLAIMED, execute it -> write outbox/<slug>.result.md -> `complete <slug>` -> repeat.
Add the hard rule: **never execute a task you did not successfully claim.** Note the parallel-session
context and the 30-min lease.

## Report back (write outbox/bus-claim-step-01.result.md)
- Paste the final poll.py (or the diffs) and the updated director_loop.md loop section.
- Confirm: inbox/claimed/ created; session-id caching works and is NOT committed; claim/heartbeat/
  complete enforce ownership; stale reclaim works past the lease.
- Describe the race test you ran (e.g. two claims of the same slug -> one CLAIMED, one ALREADY_CLAIMED).
- STATUS: DONE / PARTIAL / BLOCKED + anything unexpected.

## Guardrails
- Bus mechanics only — do NOT touch the game project or game repo.
- The .session_id (and any per-session cache) must never be committed to the public collab repo.
- No secrets in the result (outbox is public).
