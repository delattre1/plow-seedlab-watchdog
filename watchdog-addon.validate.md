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
- **G4 (add-on installed on top; 3 functions live):** run the seed's `## Verify` W1–W6 over real
  running state — `watchdog:Watchdog` alive; the 3 triggers enqueue `[watchdog] …`; the
  CEO-no-response / stale / false-block actions post on the board in the CEO's voice; W1 confirms
  the Watchdog Stop does NOT hit `main:Boss`'s queue (standalone).
- **G2 (core stayed Watchdog-free before P3):** already asserted at the checkpoint.
- **G5 (no-clobber / installed ON TOP, not from zero):** across the P3 add-on install —
  - `main:Boss` process start-time is IDENTICAL before==after (Boss never restarted from zero);
  - `INSTALL_DIR/mypeople` was NOT re-cloned (dir inode / git HEAD unchanged);
  - `:9900/dashboard` + `:9933/todos` never returned non-200 during the run (continuous heartbeat
    sampled every 5s).
- **G6 (core guards intact):** Watchdog-authed `POST /todo/status {state:done}` → rejected;
  `POST /todo/update {op:add}` → rejected.

## DONE
G1–G6 all green on the checkpoint pipeline, with a recorded cast/mp4 + the coordinator log attached
to card 475eea26df62 — same evidentiary bar as the core mypeople seed.
