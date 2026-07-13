TASK: playpath-diagnosis-01 — why do "verified working" features fail in real play?

FROM: Studio Director
DATE: 2026-07-09
SCOPE: GAME project. DIAGNOSIS ONLY — do NOT fix anything in this task. Find root causes + fix our
       verification method. (Fix tasks come after, once we know what's actually broken.)
PRIORITY: HIGHEST. This blocks all further feature work.

## The problem
Jon playtested in STUDIO PLAY and reports these DON'T WORK — yet every one was reported DONE with
"verified in Play" in a prior task:
  1. Plot claim doesn't work                    (ClaimPlot was tested + passed)
  5. Structures take no damage from weapons     (structure HP scaling was verified 400->600->900)
  9. Structure placement doesn't work           (PlaceBlock verified: block placed, resources spent)
 14. Trees take no damage from weapons          (tree felled in exactly 7 hits, +3 wood — verified)
  6. Menus overlap                              (layer manager PROVED one-modal-at-a-time)

## The hypothesis (test it, don't assume it)
Most prior verification invoked REMOTES DIRECTLY (via MCP execute_luau or seeding scripts) and confirmed
the SERVER did the right thing. The server logic is probably fine. What was never tested is the path a
PLAYER actually takes:
    click/input -> client input handler -> targeting/raycast -> remote call WITH CORRECT ARGS -> server
That client-side layer is likely broken, and one root cause may explain #1/#5/#9/#14 at once (same
signature: "server function works when called; nothing happens when played").
Note: a ClaimPlot nil-arg validation gap was already logged in passing — that may be the thread to pull.
#6 is likely a DIFFERENT root cause: something renders OUTSIDE the layer manager (the Build UI was only
ever marked RESKIN, never rebuilt — prime suspect), or a legacy GUI is still being created.

## PART 1 — Play it like a player (this is the core of the task)
Enter Studio Play as an actual player. Do NOT invoke remotes directly. Use only real input — click, keys,
the UI. For EACH of #1, #5, #9, #14:
- Perform the action the way a player would (swing an axe at a tree; swing at a structure; try to claim a
  plot; try to place a structure).
- Instrument and report the FULL chain:
    a) Does the client input handler fire at all?
    b) Does targeting/raycast resolve the intended target? (What does it actually hit — the tree? a
       collision part? nil?)
    c) Is the remote CALLED? With WHAT ARGS (paste them — nil/wrong args are a prime suspect)?
    d) Does the server receive it, and what does it return/reject?
- Report exactly WHERE each chain breaks. Be specific — "client never fires the remote" vs "remote called
  with nil target" vs "server rejects with reason X" are completely different bugs.

## PART 2 — Menu overlap (#6)
- In real play, open the menus and reproduce the overlap. Screenshot it.
- Identify WHAT is overlapping: which GUIs are on screen simultaneously, and where each is created.
- Determine whether the overlapping UI goes through the layer manager or bypasses it (check the Build UI
  and any legacy/ad-hoc GUI still being created). Report every ScreenGui in PlayerGui during the overlap.
- Also note #7 (inventory items don't fit in the panel) while you're there — is it a sizing/overflow bug in
  the ScrollList/rows? Diagnose, don't fix.

## PART 3 — Why did our verification MISS this? (the most valuable output)
This is the part that protects every future task. Answer honestly:
- Review how the prior tasks "verified" these features (check the outbox results for tier1-handlers-01,
  structure-upgrades-01, framework-core-01, inventory-panel-01).
- Identify the systematic gap: what did those verifications actually prove, and what did they NOT prove?
- Was it the MCP sandbox context trap? Direct remote invocation instead of real input? Seeding scripts
  bypassing the client? Something else?
- PROPOSE A VERIFICATION PROTOCOL that would have caught these — concrete rules for future tasks (e.g.
  "a feature is not DONE until it's exercised through real player input in Play, not via execute_luau or a
  direct remote invoke"). Make it specific enough to put in the contract as a Ground Rule.

## Report back (write outbox/playpath-diagnosis-01.result.md)
- PART 1: for EACH of #1/#5/#9/#14 — the full input->target->remote->server chain, and exactly where it
  breaks. Group them if they share a root cause (say so explicitly — one fix may resolve several).
- PART 2: the overlap reproduced + which GUI bypasses the layer manager + the #7 sizing diagnosis.
- PART 3: why verification missed it + a concrete proposed verification protocol (Ground Rule candidate).
- A recommended FIX ORDER for the Director (what's one root cause vs. separate bugs; what's cheapest/
  highest-impact first).
- STATUS: DONE / PARTIAL / BLOCKED.

## Guardrails
- DIAGNOSE ONLY — change nothing. No fixes, no refactors, no "while I was in there."
- Do NOT verify by invoking remotes directly — that is the exact blind spot we're investigating. Use real
  player input in Studio Play.
- rojo serve + Connect only (no 2nd instance). NEVER rojo build.
- No secrets in the result (outbox is public).
