# RESULT: rojo-sync-diagnose-01 — root-cause the recurring rojo serve sync drop

FROM: Claude Code (local executor)
DATE: 2026-07-06
STATUS: **DONE** (diagnosis) — root cause identified with hard evidence. Fix is a one-time plugin
setting (can't be applied from this sandbox) + optional system change needing Director sign-off; nothing
risky executed.

---

## ⭐ Root cause (one line)
`localhost` resolves to **IPv6 `::1` first** on this machine, but **Rojo serves on IPv4 `127.0.0.1` only**,
so every Rojo-plugin request wastes the IPv6→IPv4 fallback delay — **~200ms normally, up to ~2s when
AVG's network filter silently drops the `::1` SYN** — and when that stall exceeds the long-poll window the
plugin treats the session as dropped, so **disk→live sync silently dies mid-session.**

## Part A — The facts
| Fact | Finding |
|---|---|
| Rojo CLI | **7.6.1**, project-local at `ironhold-studio/ironhold-studio/.tools/rojo.exe` (not on PATH — run via that path). |
| Rojo Studio plugin | **7.6.1** (`RojoManagedPlugin.rbxm` in `%LOCALAPPDATA%\Roblox\Plugins`; version string 7.6.1 embedded). |
| **Version skew?** | **NO — CLI 7.6.1 == plugin 7.6.1.** Ruled out. (protocolVersion 4 reported by the running server.) |
| Serve command | `./.tools/rojo.exe serve` from the project dir (uses `default.project.json`, project name "ironhold"). |
| Serve health | Starts fine, binds **`127.0.0.1:34872`**, `GET /api/rojo` → HTTP 200, serverVersion 7.6.1, protocolVersion 4. The CLI/port side is healthy. |
| Security software | **AVG Antivirus running** — `AVGSvc`, `avgToolsSvc`, `aswIdsAgent`, **`avgbIDSAgent`** (intrusion-detection / network filter), 4× `AVGUI`. This is the same AVG that broke git's schannel SSL during bus setup. |

## Part B — The mechanism (with evidence)
Measured localhost latency with native `curl.exe` (low-overhead), rojo serving on `127.0.0.1`:
```
localhost   connect=0.207s  total=0.213s     <- ~200ms EVERY call
127.0.0.1   connect=0.0028s total=0.009s     <- 2.8ms, instant
[::1]:34872 connect=0.000s  total=2.054s  http=000   <- IPv6 loopback STALLS 2s then FAILS
```
- `ping localhost` → **`THINK-S30 [::1]`** — "localhost" resolves to **IPv6 first**.
- `hosts` file: both `127.0.0.1 localhost` and `::1 localhost` lines are **commented out**, so Windows
  built-in resolution is used, which returns `::1` ahead of `127.0.0.1`.
- Rojo serve `LocalAddress` = **`127.0.0.1`** (IPv4 only; it does not listen on `::1`).

Chain: the Rojo plugin's default host is the string **`localhost`** (serve even prints
`Address: localhost`). So each request → resolve `localhost`→`::1` → attempt `[::1]:34872` → Rojo isn't
there → fall back to `127.0.0.1`. The whole cost is in the **connect phase**, not payload scanning
(steady-state HTTP bodies are fine once connected). Two things make the fallback expensive:
1. **The IPv6-first ordering itself** guarantees a failed `::1` attempt on every fresh connection (~200ms
   typical dual-stack fallback on Windows).
2. **AVG makes the `::1` failure pathological.** A closed loopback port should RST **instantly** (<1ms);
   here `[::1]:34872` took **2.05s to fail** (http=000). That means the SYN to `::1` is being **silently
   dropped** (timeout) rather than refused — classic behavior of a network filter/IDS in the path
   (`avgbIDSAgent`). When the fallback stalls into the multi-second range, Rojo's plugin long-poll
   (`/api/subscribe`) exceeds its window and the plugin considers the server gone → **sync drops.**

**Ruled out:** version skew (CLI==plugin), place-id mismatch (`expectedPlaceIds` empty), the port not
opening (it does), and payload scanning being the bottleneck (127.0.0.1 total = 9ms). The distinguishing
"synced-but-didn't-rebind" flavor was a *separate* issue (see the Net-folder duplication in
server-remotes-fix-01) — not this connection problem.

### Why this produced the specific drift bugs
- **Intermittent disk→live loss** (combat fixes, Weapons, the repeated "disk edits didn't reach live"
  I kept having to hand-push): the plugin was connected but on a flaky, stalling localhost long-poll that
  periodically dropped, so edits stopped propagating without an obvious error.
- **Place-only artifacts persisting** (IronholdClient, the baked `Net` folder, two CONTRACTS.md): these
  live **outside the Rojo project entirely** and can never be synced regardless of connection — see the
  structural note below.

### Structural note (independent of the connection bug)
`default.project.json` maps only: `ReplicatedStorage.Shared → src/Shared`,
`ServerScriptService.Server → src/Server`, `ServerScriptService.Tests → src/Tests`,
`StarterPlayer.StarterPlayerScripts.Client → src/Client`. **Everything else is place-only by design** —
`StarterPack` tools, `Workspace.VikingTier1`, `ServerScriptService.ServerLoader / IronholdDoorService /
ShieldService` (directly under ServerScriptService, *not* under `.Server`), and any `StarterPlayerScripts`
script outside the `Client` folder (that's exactly where the duplicate `IronholdClient` sat). So a whole
class of "drift" isn't a sync failure at all — it's content that Rojo doesn't manage and that only lives
in the saved `.rbxl`. This is also why `rojo build` is hazardous (it would wipe all of the above).

## Part C — Recommended fix (+ what I did)
**Primary fix (zero-risk, durable, RECOMMENDED):** point the Rojo Studio plugin at the IPv4 address so it
never tries `::1`. In Studio → Rojo plugin panel → the connection **Host** field → change **`localhost`**
to **`127.0.0.1`** (Port stays `34872`), then Connect. Evidence this fixes it: `127.0.0.1` connects in
**2.8ms vs 207ms**, with no failed IPv6 attempt and no AVG-stall exposure. This is a one-time plugin
setting; **I could not apply it — it's Studio plugin UI, not a file, and the MCP sandbox lacks plugin
capability.**

**Alternative / belt-and-suspenders (need Director sign-off — system-wide):**
- **Hosts file:** uncomment `127.0.0.1  localhost` in `C:\Windows\System32\drivers\etc\hosts` so
  *everything* resolves `localhost`→IPv4 first. Fixes it for all localhost tooling, but requires admin
  elevation and is a system-wide change (could affect other software that expects `::1`). Recommend, did
  NOT apply.
- **AVG exclusion:** add `.tools\rojo.exe` (and/or loopback / port 34872) to AVG's firewall/IDS
  exceptions so the `::1` SYN gets a fast RST instead of a 2s drop. Reduces the stall but doesn't remove
  the wasted IPv6 attempt — the plugin-host fix is cleaner. (Same AVG that needed the schannel exclusion
  for git.)
- **CLI note:** `rojo serve --address 127.0.0.1` is already the default; there is no single flag to also
  listen on `::1`, so binding-side changes don't help — the fix belongs on the plugin's host string.

**What I executed:** nothing risky. I only started `rojo serve` to reproduce/measure (confirmed healthy)
and then **stopped it** (cleanup). No `rojo build`. No hosts/AVG changes. server-remotes-fix-01 untouched.

**Recommended "sync ritual" until the host fix is set:** run `./.tools/rojo.exe serve` from
`ironhold-studio/ironhold-studio`, Connect with plugin **Host = 127.0.0.1**, and after any change glance
at the Rojo panel to confirm it still says Connected (the drop is silent). For place-only content
(StarterPack / VikingTier1 / door service / non-`Client` StarterPlayerScripts), edits must be made live
and the place **Saved** — Rojo will never manage those.

## VERIFY (evidence recap)
- CLI 7.6.1 (`rojo --version`); plugin 7.6.1 (rbxm string) → no skew.
- `curl` timings: localhost connect 207ms vs 127.0.0.1 2.8ms (5/5 consistent); `[::1]:34872` = 2.05s fail.
- `ping localhost` → `::1`; hosts localhost lines commented; rojo LocalAddress = 127.0.0.1.
- AVG services enumerated (incl. `avgbIDSAgent` network filter).
- Test serve started, measured, and stopped; no other changes.

## STATUS: DONE

## BACK TO DIRECTOR
1. **Do this now (30 seconds, zero risk):** set the Rojo Studio plugin **Host = `127.0.0.1`** (not
   `localhost`), Port `34872`, Connect. That alone should end the mid-session drops. If you want it
   machine-wide instead, uncomment `127.0.0.1 localhost` in the hosts file (admin; sign-off).
2. This also retires the "reconnect Rojo + verify parity" preamble — but keep in mind the **structural**
   reality: `StarterPack`, `Workspace.VikingTier1`, the door/loader/shield services, and any
   non-`Client` `StarterPlayerScripts` are **outside the Rojo project** and are place-only forever; those
   aren't sync failures and won't be fixed by any connection change. If you want them under src, that's a
   project-restructure task (and a `rojo build` hazard) — separate, needs sign-off.
3. Suggest verifying the fix empirically: with Host=127.0.0.1 connected, edit a `src/Client` file and
   confirm it reflects live within a second, then leave it 10–15 min of active editing to confirm it no
   longer drops. Happy to own that verification task once the host is switched.
