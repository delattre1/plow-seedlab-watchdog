# Watchdog add-on — checkpoint-substrate validation runbook (P1–P4 / G1–G5)

Proves `watchdog-addon.seed.md` is a real DEPENDENCY-SEED: it installs ONLY on top of an
already-installed mypeople, validated against a SAVED "mypeople-installed" checkpoint (not a
fresh-from-zero box). Runs on the **seedbed Docker grid** (`seedbed.seed.md` recipe) — needs a
docker daemon, the `seedlab-test:latest` image, a per-node claude-auth volume, and the tailnet
key. NOT runnable on the Mac daily-driver (no docker daemon). Owner of the grid: seed-conductor.

Every gate is judged by an INDEPENDENT coordinator over REAL running state (HTTP / `/agents` /
process facts), never agent self-report.

## P1 — hydrate CORE-only mypeople (fresh substrate)
- Spin a fresh node per `seedbed.seed.md` (IMAGE=`seedlab-test:latest`, per-node
  `AUTH_VOLUME=claude-auth-wdck-1`, JOIN mode, ttyd).
- `mp spawn <node>/main:Boss --boss <central-boss>`; hand it ONLY `mypeople.seed.md`; it hydrates
  to `SEED_RESULT=DONE`.
- Coordinator verify: `:9900/dashboard`=200, `:9933/todos`=200, `:9900/health`=ok, node
  `main:Boss` alive in `/agents`.
- **G1 (assert ZERO Watchdog):** no `watchdog:Watchdog` in `/agents`; `:9933/watchdog/inbound` →
  404; `grep -c WATCHDOG_TOKEN queue.env` = 0. → core proven independent of the Watchdog.

## P2 — SAVE THE CHECKPOINT
- `docker commit <node-container> mypeople-installed:checkpoint` (+ snapshot the node's
  queue/auth volume). This is the saved "mypeople-installed substrate."
- **G1 cont. (re-spinnable):** `docker run` a throwaway FROM `mypeople-installed:checkpoint`;
  confirm `:9900/dashboard`=200 + `:9933/todos`=200 + `main:Boss` alive. Tear the throwaway down.

## P3 — hydrate the ADD-ON ON TOP of the checkpoint
- Spin a NEW node **FROM `mypeople-installed:checkpoint`** (NOT a fresh image) = a user who
  already has mypeople. Re-join the tailnet/queue.
- Hand the node's agent ONLY `watchdog-addon.seed.md` (+ `watchdog-corpus.md`).
- **G3 (dependency enforced, pos+neg):**
  - positive: on the checkpoint node, Step-0 LIVE probe passes → "dependency satisfied" →
    additive install; reaches `SEED_RESULT=DONE`.
  - negative: on a BARE `seedlab-test:latest` node (no mypeople), the SAME seed's Step-0 →
    `BLOCKED_REASON=dependency mypeople not installed`, and installs NOTHING.

## P4 — independent verify of the result (on the P3 checkpoint node)
Run the seed's full `## Verify` over REAL running state (an INDEPENDENT coordinator, not the
installing agent, confirms each behaviour came FROM the seed). Drive thresholds low; never wait
wall-clock. The gates (mirror the seed):
- **Feat A — A1/A2/A3:** unanswered CEO message → ONE nudge; reply within window → none; 2nd CEO
  message → only it fires; fires on Done/Cancelled too (no status filter); CEO task-creation
  untouched past the delay → nudge, picked-up → none.
- **Feat B — B1/B2/B3:** idle Working card → the AGENT posts EXACTLY `What is the status of this?`
  (no title-riff, no `(quiet ~Nm)`); idle Done/Cancelled → none, idle Working/Review → nudge;
  repeated scans → one nudge per idle episode (last-id + cooldown dedup); activation baselines a
  backlog with ZERO nudges.
- **Feat C — C1/C2/C3:** false-block → correction; genuine question → none; done-without-proof →
  proof challenge.
- **Structural — S1/S2/S3/S4:** `watchdog:Watchdog` has NO `BOSS_ID` (Stop hook reports to no one);
  every action is a board post by the watchdog; it never marks done / creates; KILL it → the
  launchd keepalive respawns + re-briefs it (bossless) within ~1 min.
- **G2 (core stayed Watchdog-free before P3):** already asserted at the checkpoint.
- **G4 (no-clobber / installed ON TOP, not from zero):** across the P3 install — `main:Boss`
  start-time IDENTICAL before==after; `INSTALL_DIR` NOT re-cloned (inode / git HEAD unchanged);
  the board API never returned non-200 (5s heartbeat). Restart the todo-server ONLY via
  `launchctl kickstart -k` (never kill+nohup → two-server race).

## DONE
Every Feat A/B/C, Structural, and G gate green on the checkpoint pipeline, with a recorded cast/mp4 +
the coordinator log attached to card 475eea26df62 — same evidentiary bar as the core mypeople seed.
