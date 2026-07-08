TASK: rojo-sync-diagnose-01 — root-cause the recurring rojo serve sync drop

FROM: Studio Director
DATE: 2026-07-06
SCOPE: GAME project + local dev environment. DIAGNOSIS-FIRST; implement a fix only if low-risk and clear.
RUN AFTER: server-remotes-fix-01 (don't disturb that fix mid-flight).

## Why
Every drift bug this session traces to the SAME root cause: `rojo serve` keeps losing the disk->live
sync mid-session, so the live place silently diverges from src. Instances so far:
  1. place-only IronholdClient duplicating src ClientBootstrap
  2. two divergent CONTRACTS.md (untracked Studio-side vs repo)
  3. live-ahead-of-disk combat fixes (plugin dropped mid-session)
  4. unbound server RemoteFunction handlers on the live server (server-remotes-fix-01)
We've been taping over this with "reconnect Rojo + verify parity" preambles on every task. Stop taping;
find why the sync drops and kill it at the source.

## Part A — Establish the facts
- Rojo version in use (`rojo --version`) and the plugin version installed in Studio. Do they match?
  Version skew between CLI and plugin is a known cause of silent sync failures — report both.
- How is rojo being run (command line, working dir, which default.project.json)? Paste the serve command.
- Reproduce the drop if possible: start `rojo serve`, Connect, edit a src file, confirm it reflects live;
  then observe under what conditions the connection drops or stops reflecting (time? a specific action?
  Studio losing focus? a large/binary file? a syntax error in a synced file? network/SSL — recall the
  AVG schannel issue from bus setup). Report what triggers it.
- Check for the AVG / SSL-inspection angle: does AVG (or another security tool) interfere with the
  rojo localhost connection or file watching the way it broke git SSL earlier? Report findings.
- Check the rojo output/console for errors or warnings when the sync drops. Paste them.

## Part B — Identify the mechanism
Determine which of these it is (or something else), with evidence:
- Plugin<->CLI version mismatch.
- File-watcher missing changes (e.g. editor writing atomically / temp-file rename the watcher ignores,
  or too many files, or a path outside the project).
- Connection dropping (localhost port, security software, Studio backgrounding).
- Two-way sync / manual live edits conflicting with the watcher.
- Syncing succeeds but bound code doesn't re-run (module already required -> edits don't rebind until
  server re-init) — this is a "sync worked but effect didn't" flavor, important to distinguish.

## Part C — Recommend + (if safe) implement a durable fix
Based on the mechanism, recommend the fix. Likely candidates:
- Upgrade/align rojo CLI + plugin to matching current versions.
- Exclude the interfering security tool from the rojo port / project dir, or switch approach.
- A documented, reliable "sync ritual" if the tooling genuinely requires it (e.g. use `rojo build` for a
  clean base ONCE + serve for iteration — but NOTE the build hazard: build wipes place-only content like
  VikingTier1/StarterPack/door service, so this needs care).
- If the issue is "code synced but didn't rebind," document the required server-reinit/replay step so
  future syncs actually take effect.
Implement only the low-risk, clearly-correct part (e.g. version alignment, config fix). Anything risky
(anything involving rojo build, or restructuring the project) -> RECOMMEND with tradeoffs, do NOT execute;
leave it for Director sign-off.

## Report back (write outbox/rojo-sync-diagnose-01.result.md)
- Part A facts: rojo CLI vs plugin versions, serve command, reproduction conditions, console errors,
  AVG/SSL findings.
- Part B: the identified mechanism, with evidence.
- Part C: recommended durable fix + what (if anything) you safely implemented, and what needs Director
  sign-off (esp. anything involving rojo build).
- STATUS: DONE / PARTIAL / BLOCKED + anything unexpected.

## Guardrails
- Diagnosis is the priority. Do NOT run `rojo build` against the real place as a "fix" without explicit
  Director sign-off — it wipes place-only content (VikingTier1 / StarterPack / door service).
- Don't disturb the server-remotes-fix-01 changes.
- No secrets in the result (outbox is public) — refer to any keys/tools by name only.
