# SEED: watchdog-addon

> seed-format: 1

The **Watchdog** is an OPTIONAL ADD-ON to mypeople: a standalone, **bossless** agent + a set of
board-triggers in the todo-server that keep the To Do board MOVING. It has THREE features, all on
the watchdog's OWN clock (never woken by needing a human to act first):

- **Feat A — unanswered CEO message (default 3 min).** When the CEO posts a message OR creates a
  task and it sits unanswered, the watchdog nudges the card.
- **Feat B — idle card (default 10 min).** When a card gets no comment from ANYONE for the
  threshold, the watchdog posts a fixed generic status nudge. Skips Done/Cancelled.
- **Feat C — false-block / done-without-proof.** Every non-CEO message is classified; a genuine
  false-block ("waiting for the CEO's OK" when the task already exists) or a done-claim with no
  proof gets one correction.

Division of labour (the architecture): the **SYSTEM** (todo-server) DETECTS, SCHEDULES, GATES and
DEDUPS; the **AGENT** (`watchdog:Watchdog`) does the JUDGMENT and POSTS the human-facing message.
The CORE mypeople ships WITHOUT the watchdog; you install this add-on only if you ask for it.

## Depends-on

This seed is a DEPENDENCY-SEED. It does **not** install mypeople — it requires a **running**
mypeople and installs the Watchdog ADDITIVELY on top of it.

| dependency | required | detect (LIVE probe — running state, not file presence) | on-absent |
|---|---|---|---|
| mypeople (core) | yes | `curl -fsS 127.0.0.1:${TODO_PORT:-9933}/todo/board -H "X-Queue-Secret: $QUEUE_SECRET"` = 200 **AND** `main:Boss` alive in `mp status` **AND** `~/.config/mypeople/queue.env` + `$INSTALL_DIR` exist | **STOP**: print `BLOCKED_REASON=dependency mypeople not installed` and exit. Do NOT install the core. |

Rationale: installing mypeople must never force the Watchdog, and installing the Watchdog must never
silently stand up mypeople. The two seeds stay independent; this one rides on top.

## Goal

A running **standalone, bossless** `watchdog:Watchdog` agent on a host that already runs mypeople,
plus three board-triggers folded into the running todo-server, plus a launchd keepalive that keeps
the agent durable. The watchdog keeps the board moving with the minimum number of clean, human-voiced
posts — and **cannot storm** (every trigger is event-scheduled and/or deduped).

## Architecture (locked — SYSTEM detects, AGENT composes)

- The **SYSTEM half** lives in the running todo-server. On each relevant board event it SCHEDULES a
  persisted, deferred job or routes a classify request; a background worker FIRES due jobs only when
  a gate still holds; on fire it hands an **incident** to the agent via `mp send ${WATCHDOG_AGENT}`.
  The system half never composes human text for Feat A/C and never decides taste — it only
  detects/schedules/gates/dedups. Jobs are PERSISTED to disk so they survive a todo-server restart.
- The **AGENT half** is a standalone `claude` agent that acts ONLY on the system's tagged messages
  (`[watchdog idle]`, `[watchdog incident]`, `[watchdog classify]`) and POSTS on the card. It has NO
  boss (see ## Ownership). Its voice is grounded in the mined corpus (`seeds/watchdog-corpus.md`).
- The watchdog NEVER marks a task done and NEVER creates tasks. It posts under its own id.

## Ownership (the watchdog is a PEER of the Boss, with NO boss)

An agent's boss = whoever its **Stop hook** reports to. The mypeople Stop hook (`tmux-boss-hooks`
`emit-event`) notifies `$BOSS_ID` and, if `BOSS_ID` is empty, exits without notifying anyone. A
normal engineer inherits `BOSS_ID=…/main:Boss`; the Boss itself runs with NO `BOSS_ID`.

The Watchdog MUST run like the Boss: **top-level, no boss**. So its launch must UNSET any inherited
`BOSS_ID` (otherwise a leaked `BOSS_ID` from the spawner/tmux env makes its Stop hook ping the Boss).
Fold this into the spawner so it generalises: when spawning a TOP-LEVEL agent (the Boss via
`--master`, OR any agent whose window is the Watchdog) the launch line runs `unset BOSS_ID` before
`exec`; normal engineers are untouched (they keep inheriting the Boss). Acceptance: the running
`watchdog:Watchdog` process tree has NO `BOSS_ID` in its env (verify via `ps eww`), so its Stop hook
reports to no one — exactly like the Boss.

## Config (in `~/.config/mypeople/queue.env`, read by the todo-server at start)

| key | default | meaning |
|---|---|---|
| `WATCHDOG_AGENT` | `${HOST_ID}/watchdog:Watchdog` | the agent the system feeds + the author of posts |
| `WATCHDOG_TOKEN` | (generated) | the watchdog's own credential (distinct from `QUEUE_SECRET`) for any privileged write |
| `WATCHDOG_NUDGE_DELAY_MIN` | `3` | Feat A delay: minutes before an unanswered CEO message/task fires |
| `WATCHDOG_IDLE_MIN` | `10` | Feat B threshold: minutes of total silence before an idle card fires |
| `BOARD_FILE` | (the core's live board path) | PIN the explicit board path so a restart can never default to a missing path (see ## Reliability) |

## Components

- The running **mypeople core** (the dependency): its todo-server, queue, HUD, `main:Boss`.
- The **corpus** (`seeds/watchdog-corpus.md`) — the CEO's real phrasings; the agent's voice.
- The host's **spawner** (`mp spawn`) + **stop-hook** plugin — reused; patched only to make
  top-level agents bossless.
- **launchd** — the todo-server runs as a KeepAlive service; the watchdog keepalive is a second one.

## Steps

### Step 0 — Interview (mandatory, single turn)
1. Run the `## Depends-on` LIVE probe. If mypeople is NOT running, STOP with
   `BLOCKED_REASON=dependency mypeople not installed` — do not install the core.
2. Detect `queue.env`, `INSTALL_DIR`, the corpus, and a prior watchdog install (converge, do not
   double-spawn). Send ONE consolidated message (✓/✗/⚠), then build autonomously to `SEED_RESULT=DONE`.

### Step 1 — SYSTEM half: fold the triggers into the todo-server (additive; restart via launchd)
Express the contracts so a blind agent GENERATES them by extending the EXISTING comment/add fanout +
adding ONE background worker thread (do NOT paste code; reuse the store, the `mp send` helper, the
fanout seam). PERSIST jobs to a file under the core's todos dir so they survive restart.

- **Feat A — unanswered CEO MESSAGE (`WATCHDOG_NUDGE_DELAY_MIN`, default 3 min, NO status filter).**
  When a comment with `by==CEO` is added to a card, schedule a deferred job `{card, comment_id,
  fire_at=now+delay}`. A newer CEO message on the same card supersedes the older pending job. At fire,
  the ONLY gate is **STILL-MOST-RECENT**: the scheduled CEO comment is still the last comment on the
  card (no reply from anyone, no newer message). If still most recent → hand a `[watchdog incident]`
  to the agent (with the card + the ACTUAL CEO text to quote). Otherwise it's a no-op (a reply cancels
  it; a newer CEO message supersedes it). Feat A fires on ALL statuses — including Done/Cancelled —
  never filtered by status.

- **Feat A — TASK CREATION by the CEO counts as a CEO message.** When the CEO CREATES a task (the
  HUD/create path tags `by:"CEO"`), schedule the same kind of deferred job. At fire, the gate is
  **STILL-UNANSWERED**: the created task still has no comment, no assignee, and an unchanged initial
  state (nobody picked it up). If still untouched → `[watchdog incident]`; any activity (comment,
  assignment, state change) → no-op. Guarantees a new task gets owned.

- **PUBLIC-ONLY — the watchdog NEVER sends a private/Boss-directed message.** The CEO ONLY reads the
  board and CANNOT see private messages, so any private channel defeats the purpose. Every watchdog
  action is a PUBLIC card comment (Feat A quotes the CEO; Feat B posts `What is the status of this?`;
  Feat C posts its correction). There is NO watchdog→Boss inbox alert. The "mandatory reply" is purely
  the Boss's own behaviour (the Boss always replies ON the card to the public nudge); it is NEVER
  enforced via a private message. (The core todo-server's own board-activity notifications are separate
  core infrastructure and are not part of the watchdog.)

- **Feat B — IDLE card (`WATCHDOG_IDLE_MIN`, default 10 min).** A background worker scans the board
  on its own clock (idleness is the ABSENCE of events, so this one is a periodic scan, not
  event-scheduled). A card is IDLE iff NObody has posted anything for the threshold — ANY comment
  from anyone (engineer, Boss, CEO, or the watchdog's own post) resets the clock (last-activity =
  newest of all comment timestamps + task timestamps; normalise ms/sec). **SKIPS Done + Cancelled +
  Recurring** (the skip-set `{done, cancelled, recurring}` — closed or recurring tasks are not "idle
  work"; this is the ONLY status exclusion in the whole watchdog, and it applies to Feat B only — the
  `recurring` state is added to the core TODO app's state enum). On the FIRST scan after (re)start,
  BASELINE all already-idle cards silently (record them, do NOT nudge) so activation never storms a
  backlog. Dedup per card keyed on BOTH the last-comment-id AND a fire-time cooldown (≥ the idle
  threshold), so it can never post twice within a window even under a transient. When a card fires →
  hand a `[watchdog idle]` incident to the agent that instructs it to post the EXACT fixed generic
  line, verbatim (NO title, NO context, NO idle-time): `What is the status of this?`. (Re-escalation
  only after another full idle interval of total silence. PUBLIC card comment only — no private alert.)

- **Feat C — every NON-CEO message → classify.** On every non-test board comment whose author is not
  the CEO and not the watchdog itself, route a `[watchdog classify]` request to the agent with the
  card + the message text. The agent judges and posts at most one correction (below). The watchdog's
  own posts are excluded (no loop); `{test}` cards are skipped.

- The system half decides nothing for A/C — it detects/schedules/gates/dedups and routes. Feat B's
  generic line is fixed; the agent still posts it (system detects, agent posts).

### Step 2 — AGENT half: spawn the standalone, bossless Watchdog
- Spawn `mp spawn ${WATCHDOG_AGENT}` — standalone, NOT `--master`, and **bossless** (its launch
  unsets `BOSS_ID`; see ## Ownership). Brief it (from the corpus) so it acts ONLY on the three tags:
  - `[watchdog idle]` → POST exactly `What is the status of this?` verbatim — no riffing on the
    title, no idle-time, nothing appended.
  - `[watchdog incident]` → POST ONE short nudge in the CEO voice, quoting the actual unanswered
    message (Feat A may quote because it really has the message).
  - `[watchdog classify]` → judge the quoted non-CEO message: **FALSE BLOCK** (claims/implies
    blocked-on-CEO or waiting-for-CEO OK/approval/sign-off to proceed, when the task already exists =
    the CEO asking IS the go-ahead) → POST `Not blocked — the task existing IS the OK. Keep working to
    the done-condition.` ; **DONE-WITHOUT-PROOF** (claims done but attaches no video/print/artifact) →
    POST `Where is the video/print proving it's done? Attach the artifact.` ; a GENUINE
    question/decision/real blocker (taste, scope, credentials, money, access, which-option) or a
    normal status update → DO NOTHING. Post at most once per message.
- The agent posts via the core's comment API as `${WATCHDOG_AGENT}` (its own id). It NEVER marks done
  and NEVER creates tasks.

### Step 3 — Keepalive (the watchdog is bossless → nothing else revives it)
Install a keepalive so the standalone watchdog self-heals, under launchd discipline (no manual
kill+nohup anywhere):
- `bin/watchdog-keepalive.sh`: poll every ~20s; if `${WATCHDOG_AGENT}` is not alive in `mp status`,
  respawn it (idempotent `mp spawn`, which launches it bossless) and re-brief it (the same three-tag
  brief). Log to the todos dir.
- A launchd job (`co.plow.watchdog.keepalive`, `KeepAlive=true`, `RunAtLoad=true`) runs that script,
  so the SUPERVISOR itself is durable (launchd revives it if it dies). Two-level supervision.

## Reliability (folded from hard-won live incidents)
- **Single instance via launchd; restart ONLY via `launchctl kickstart -k`.** The todo-server runs
  as a launchd KeepAlive service. NEVER `kill`+`nohup` it — that races the KeepAlive respawn into TWO
  servers (two writer/worker processes) which drops freshly-created cards and double-fires nudges.
  Always restart with `launchctl kickstart -k gui/$(id -u)/<label>`.
- **PIN `BOARD_FILE` + atomic safe startup.** Pin the explicit board path in `queue.env` so a restart
  can never default to a missing path and serve an empty board. The todo-server VALIDATES the loaded
  board BEFORE binding the port: if it would be empty, or far smaller than the freshest backup, it
  ABORTS (does not serve a blank/truncated board) — set `ALLOW_EMPTY_BOARD=1` only for a genuine
  fresh install. Never let an empty in-memory board overwrite the real file (catastrophic-shrink
  guard on save).
- **Idle baseline (no backlog storm).** First scan after activation seeds existing idle cards as a
  baseline without nudging (above).

## Verify

The measurable ACCEPTANCE GATE. Agent-driven over REAL running state (judge the running system, never
file presence; never trust self-report). FUNCTIONAL gates run on throwaway cards with thresholds
driven low (never wait wall-clock). **DONE = every gate green on a FRESH from-the-seed hydrate onto a
clean mypeople, RECORDED** (an independent coordinator, not the installing agent, confirms each came
from the SEED). Full runbook: `watchdog-addon.validate.md`.

### FUNCTIONAL
- **A1 — unanswered CEO message.** Post a `by:CEO` comment on a working card; with the delay driven
  low and NO reply → after the delay the watchdog posts exactly ONE nudge. Post a reply within the
  window → NO nudge (still-most-recent gate). Post a 2nd CEO message → only the 2nd can fire.
- **A2 — Feat A fires on ANY status.** A `by:CEO` comment on a Done card and on a Cancelled card both
  nudge (no status filter).
- **A3 — task-creation arms Feat A.** Create a task as the CEO, leave it untouched → after the delay,
  ONE nudge. Assign/comment/move it within the window → no nudge.
- **B1 — idle nudge, generic.** Drive `WATCHDOG_IDLE_MIN` low; an idle Working card → the watchdog
  posts EXACTLY `What is the status of this?` (assert the card's title is NOT echoed/riffed and no
  `(quiet ~Nm)` suffix). The post is made by the AGENT (system routed `[watchdog idle]`).
- **B2 — idle skips Done/Cancelled/Recurring, nudges Working/Review.** Idle Done, Cancelled, and
  Recurring cards → NO nudge; idle Working and Review cards → nudge.
- **B3 — no double-fire / no storm.** Repeated scans of the same idle card → exactly one nudge per
  idle episode (last-id + cooldown dedup); ANY new comment resets the clock; activation baselines a
  backlog of idle cards with ZERO nudges.
- **D1 — PUBLIC-ONLY (no private channel).** Across a full Feat A + Feat B + Feat C fire, the Boss
  agent's inbox receives NO watchdog message — every watchdog action is a PUBLIC card comment by
  `${WATCHDOG_AGENT}` (assert zero `[watchdog]` sends to the Boss agent).
- **C1 — false block corrected.** An engineer "just give me the OK and I'll do it" → the watchdog
  posts the "Not blocked — the task existing IS the OK…" correction.
- **C2 — genuine left alone.** An engineer "which DB, Postgres or Mongo? need your call" → NO post.
- **C3 — done without proof.** An engineer "all finished, it's done!" (no artifact) → the watchdog
  posts "Where is the video/print proving it's done?".

### STRUCTURAL
- **S1 — bossless.** The running `watchdog:Watchdog` process has NO `BOSS_ID` in its env (`ps eww`),
  so its Stop hook reports to no one (a Watchdog Stop never lands in `main:Boss`'s queue).
- **S2 — board-mediated.** Every watchdog action is a POST ON THE CARD by `${WATCHDOG_AGENT}`.
- **S3 — never-done / never-create.** A watchdog-authored `state=done` or `op:add` is not used; the
  watchdog only comments.
- **S4 — self-healing.** Kill `watchdog:Watchdog`; the keepalive respawns it (bossless) and re-briefs
  it within ~1 min, with no manual intervention.

### INSTALL (checkpoint-substrate — see `watchdog-addon.validate.md`)
- **G1 — clean checkpoint.** Core-only hydrate serves the board API 200 + `main:Boss` alive; no
  `watchdog:Watchdog`, no watchdog config.
- **G2 — dependency enforced both ways.** On a running-core box, Step-0 → additive install →
  `SEED_RESULT=DONE`; on a BARE box, Step-0 → `BLOCKED_REASON` and installs nothing.
- **G3 — add-on installs ON TOP** → `SEED_RESULT=DONE`, and A1–A3, B1–B3, C1–C3, S1–S4 all pass.
- **G4 — no-clobber.** Across the install the core `main:Boss` start-time is unchanged, `INSTALL_DIR`
  is not re-cloned, and the board API never returned non-200 (heartbeat sampled throughout).

## Failure modes
- **dependency not running** → STOP with `BLOCKED_REASON=dependency mypeople not installed`.
- **two todo-servers / dropped cards / double-fire** → caused by `kill`+`nohup` racing launchd
  KeepAlive. Fix: restart ONLY via `launchctl kickstart -k`; assert exactly one server process.
- **empty/blank board after restart** → `BOARD_FILE` not pinned, defaulted to a missing path. Fix:
  pin `BOARD_FILE`; the startup guard must abort rather than serve empty.
- **idle nudge riffs on the title / appends "(quiet ~Nm)"** → the `[watchdog idle]` path must hand a
  FIXED verbatim line to the agent (no title in the incident); only Feat A/C read real content.
- **idle storm on activation** → must baseline existing idle cards without nudging on first scan.
- **self-trigger loop** → exempt `by==WATCHDOG_AGENT` and `{test}` in all triggers.
- **watchdog dies and stays dead** → it's bossless; the keepalive (launchd KeepAlive) must respawn it.
- **stop-hook pings the Boss** → a leaked `BOSS_ID`; the launch must `unset BOSS_ID` for top-level.

## Cleanup
To remove the add-on without touching the core: `launchctl bootout gui/$(id -u)/co.plow.watchdog.keepalive`;
`mp kill ${WATCHDOG_AGENT}`; revert the todo-server trigger additions (Feat A/B/C + the worker) and
remove the `WATCHDOG_*` keys from `queue.env`; restart the todo-server via `launchctl kickstart -k`.
The core mypeople (HUD/TODO/Boss) is unaffected — proving the add-on was additive.
