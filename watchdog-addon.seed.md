# SEED: watchdog-addon

> seed-format: 1

The **Watchdog** is an OPTIONAL ADD-ON to mypeople. It is a STANDALONE agent + a set of
board-triggers that keep the To Do board MOVING: it watches the board, pings when the CEO is
unanswered or a card goes stale, and AUTO-UNBLOCKS fake "blocked on the CEO" so nothing rots
waiting for an OK that was never needed. The CORE mypeople ships WITHOUT it; you install this
only if you ask for it.

## Depends-on

This seed is a DEPENDENCY-SEED. It does **not** install mypeople — it requires a **running**
mypeople and installs the Watchdog ADDITIVELY on top of it.

| dependency | required | detect (LIVE probe — running state, not file presence) | on-absent |
|---|---|---|---|
| mypeople (core) | yes | `curl -fsS 127.0.0.1:${QUEUE_PORT:-9900}/health` = ok **AND** `curl -fsS 127.0.0.1:${TODO_PORT:-9933}/todos` = 200 **AND** `main:Boss` shows alive in `GET :${QUEUE_PORT}/agents` **AND** `~/.config/mypeople/queue.env` + `$INSTALL_DIR/mypeople` exist | **STOP**: print `BLOCKED_REASON=dependency mypeople not installed` and exit. Do NOT install the core. |

Rationale: installing mypeople must never force the Watchdog, and installing the Watchdog must
never silently stand up mypeople. The two seeds stay independent; this one rides on top.

## Goal

A running **standalone Watchdog agent** on a host that already runs mypeople. The system half
(in the todo-server) detects three board triggers and drops each into the Watchdog's inbox; the
agent half classifies and acts by posting **on the board** (the Boss picks things up from the
board). Governing mentality: **there are no real blocks on the CEO** — if the CEO created a task,
he wants it done; **err on EXCESS, never on lack.** The Watchdog speaks in the CEO's real voice
(grounded in a mined corpus of his board comments), never asks the CEO for permission, and never
bothers him with fake blocks.

## Role (CEO design — watchdog SURFACES, the Boss ACTS)

The Watchdog is the **monitor / backstop**, not a second Boss. It **watches the board and redirects
the Boss**; the Boss reads the board and takes the action. By design it does **not** duplicate the
Boss. Concretely:
- **Watchdog's own active actions:** (1) **stale-ping** ("what is the status of this task?" on a card
  with no update for `WATCHDOG_STALE_SEC`, + the 5-min consolidated CEO digest); (2) **safety-gate** —
  it NEVER auto-unblocks a genuine CEO-only decision (product/scope/taste, real credential/login/money,
  his done-validation); (3) **backstop ceo-unanswered ping** — when a CEO comment goes unanswered for
  `WATCHDOG_CEO_SEC` (i.e. the Boss is slow/down), it pings the assignee to answer on the card.
- **Watchdog SURFACES, Boss ACTS:** on a false block it **flags** it on the board ("not a real CEO
  gate — the CEO already wants this; proceed") so the **Boss unblocks + drives it**. The Watchdog does
  not itself grant the GO — that is the Boss's action; the Watchdog redirects the Boss to it. (If the
  Boss is absent/slow, the flag still stands on the board as the durable nudge.)

## Done

Observable, on a host with mypeople already running:
- The Watchdog agent `${HOST_ID}/watchdog:Watchdog` is **alive** in `GET :${QUEUE_PORT}/agents`,
  and it is **standalone**: its Stop hook does **not** enqueue to `main:Boss`'s queue (no Boss
  double-ping on a Watchdog stop).
- The todo-server emits the three Watchdog triggers to the Watchdog's inbox (queue), observable as
  `[watchdog] …` events in the Watchdog queue:
  1. a CEO comment on any task with **no response for 3 min** (`WATCHDOG_CEO_SEC`, default 180);
  2. a card with **no update for 10 min** (`WATCHDOG_STALE_SEC`, default 600);
  3. **every** new non-CEO board message (comment / non-test work-state change).
- On a CEO-no-response trigger the Watchdog posts a card comment (as `by=${HOST_ID}/watchdog:Watchdog`)
  telling the assignee to answer **on the card now**; on a stale trigger it posts a
  `what is the status of this task?` line; **and** a consolidated **5-min digest** of all
  blocked-on-CEO items (each with a card LINK) is delivered (extends the existing `WA_WATCHDOG`).
- On a false-block / permission-ask-for-an-already-asked thing, the Watchdog posts
  `the CEO obviously wants this — proceed`, re-points to the existing asset, and grants the GO
  **itself** — it NEVER posts a permission question to the CEO. It does NOT touch genuine
  CEO-only items (taste/scope/credentials/money/his done-validation).
- The core stays intact: the Watchdog NEVER marks a task done and NEVER creates tasks.
- mypeople core remains green throughout (HUD/TODO 200, `main:Boss` alive) — the add-on did not
  clobber or re-install it.

## Inputs

| name | required | default | detect | ask |
|---|---|---|---|---|
| `~/.config/mypeople/queue.env` | yes | — | `[ -f ~/.config/mypeople/queue.env ]` | "Watchdog rides on a running mypeople — its `queue.env` must exist. If missing, install mypeople first (this add-on will not)." |
| `INSTALL_DIR` | yes | from queue.env | `grep INSTALL_DIR ~/.config/mypeople/queue.env` | reuse the core's value; do not invent a new tree. |
| `WATCHDOG_CORPUS` | yes | `seeds/watchdog-corpus.md` (this seed ships it) | `[ -f seeds/watchdog-corpus.md ]` | "The mined corpus of the CEO's real board comments (stale-ping + false-block voice). Generated by the librarian; if absent, regenerate by mining `todo/data/board.v2.json` for CEO comments." |
| `CEO_WHATSAPP` | conditional (for the 5-min digest) | reuse core's queue.env value | `grep CEO_WHATSAPP ~/.config/mypeople/queue.env` | "The CEO's WhatsApp digits for the 5-min consolidated digest; if unset, the digest still posts on the board, WA delivery stubs." |
| `WATCHDOG_CEO_SEC` | no | `180` | env | CEO-no-response threshold (3 min). |
| `WATCHDOG_STALE_SEC` | no | `600` | env | card-stale threshold (10 min). |
| `WA_WATCHDOG_SEC` | no | `300` | env | consolidated CEO-digest cadence (5 min). |
| ⚠ prior Watchdog install | confirm | — | `GET :${QUEUE_PORT}/agents` lists `watchdog:Watchdog`? / `grep WATCHDOG_TOKEN queue.env` | if already present, this seed CONVERGES (idempotent) — it does not double-spawn or clobber. |

## Components

- The running **mypeople core** (the dependency) — its `todo-server`, queue, HUD, and `main:Boss`.
- The **librarian corpus** (`seeds/watchdog-corpus.md`) — the CEO's real stale-ping + false-block
  phrasings; the agent's voice is built from this, not invented.
- The host's **agent spawner** (`mp spawn`) and the **stop-hook** machinery — reused, but for the
  Watchdog the stop-hook is configured NOT to enqueue to the Boss queue (standalone).

## Steps

### Step 0 — Interview (mandatory, single turn)
1. Run every `## Depends-on` LIVE probe. If mypeople is NOT running, STOP with
   `BLOCKED_REASON=dependency mypeople not installed` — do not install the core.
2. Read `## Inputs`, run each `detect`. Send ONE consolidated message: ✓ satisfied (queue.env,
   INSTALL_DIR, corpus), ✗ missing (with the `ask`), ⚠ prior Watchdog install to confirm
   (converge vs reset). Then build autonomously to `SEED_RESULT=DONE` or a FAILURE-MODE block.

### Step 1 — SYSTEM half: board-triggers (extend the todo-server, additive — do not rewrite it)
Express the contracts so a blind agent GENERATES them by extending the EXISTING seams (do NOT paste
code; reuse `boss_ping`, `watchdog_loop`, the comment/update fanout, and the `WA_WATCHDOG` digest):
- **Trigger 1 — CEO-no-response (3 min, global, any status).** When a `by=CEO` comment lands on a
  task and `WATCHDOG_CEO_SEC` elapses with no later comment on that task, enqueue
  `mp send ${WATCHDOG_AGENT} "[watchdog] ceo-unanswered <tid> …"`. (New timer in the existing
  watchdog scan loop.)
- **Trigger 2 — card-stale (10 min, any status).** When a card has no update for
  `WATCHDOG_STALE_SEC`, enqueue `[watchdog] stale <tid> …`. Feed the same item into the existing
  **5-min consolidated CEO digest** (`WA_WATCHDOG`), one message for ALL blocked-on-CEO items, each
  with a card LINK, NO "mark done to stop pings" nag (corpus §fn2).
- **Trigger 3 — every non-CEO board message.** On every non-test comment / work-state change whose
  author is not the CEO (and not the Watchdog itself), ALSO enqueue `[watchdog] classify <tid> "…"`
  to the Watchdog. (Extend the existing comment/update fanout seam — the same place that pings the
  Boss; the Watchdog event is additive and never replaces the Boss ping.)
- The system half **decides nothing** — it only detects + routes. All three are gated to skip
  `{test}` tasks and the Watchdog's own posts (no self-trigger loop).

### Step 2 — AGENT half: spawn the STANDALONE Watchdog
- Generate the agent's brain at `$INSTALL_DIR/run/watchdog/CLAUDE.md` from the intent below + the
  corpus (do NOT paste a fixed essay; source-of-truth `plans/watchdog-claude.md`). The onboarding
  turn ends with a DURABLE roster summary bearing ≥2 of {`watchdog`,`board`,`auto-unblock`,
  `err-on-excess`,`no-real-blocks`}.
- Spawn `mp spawn ${HOST_ID}/watchdog:Watchdog` **STANDALONE**: NOT `--master`, and NOT wired so its
  Stop hook enqueues to `main:Boss` (the Watchdog is the first agent whose stop-hooks are NOT routed
  to the Boss queue — else every Watchdog stop double-pings Boss + Watchdog). It is BOARD-COUPLED:
  it is driven by its inbox (the Step-1 triggers) and ACTS by posting on the board; the Boss reads
  the board.
- The agent authenticates with its own `WATCHDOG_TOKEN` (distinct from `QUEUE_SECRET`); the server
  binds the never-done / never-create guard to that authenticated caller (a Watchdog write to
  `state=done` or `op:add` is rejected server-side, regardless of body `by`).
- **RELIABILITY — survey-modal suppression (REQUIRED; hard-won 2026-06-25).** A long-running
  interactive `claude` agent is periodically frozen mid-action by the Claude Code **"How is Claude
  doing this session?" feedback survey modal** — `dismiss_dialogs` only handles bypass/trust dialogs,
  so the Watchdog silently stops posting. Spawn the Watchdog with its **OWN isolated**
  `CLAUDE_CONFIG_DIR` (so it survives any shared `~/.claude.json` write-back) whose `.claude.json`
  has `feedbackSurveyState` suppressed (e.g. `{"lastShownTime": 9999999999999}`). The Watchdog must
  never see the survey.
- **RELIABILITY — watchdog-supervisor (REQUIRED).** The Watchdog is STANDALONE — no Boss revives it.
  Install a `bin/watchdog-supervisor.sh` (analogous to `boss-supervisor.sh`): keep exactly one
  `watchdog:Watchdog` window alive, respawn via the standalone launch script if its window dies OR if
  it goes unresponsive (no inbox progress for N min), so a frozen/crashed Watchdog self-heals.
- **RELIABILITY — no stale-flood / degradation under load.** A board with many idle cards + a short
  `WATCHDOG_STALE_SEC` floods the single agent and degrades it (it stops per-card posting). The
  system half MUST dedup (fire each card's stale at most once per stale-window — `WD_FIRED`) AND
  fold a batch of stale cards into the ONE consolidated 5-min digest rather than N per-card events;
  keep `WATCHDOG_STALE_SEC` realistic (default 600s, not 30s). The Watchdog handles a batch as a
  single digest action, never a flood of one-offs.

### Step 3 — AGENT half: the three functions (the agent's contracts; voice from the corpus)
1. **CEO-no-response** → post a card comment telling the assignee to respond ON THE CARD now,
   stating the CEO only reads the board (an unanswered card = invisible to him).
2. **Card-stale** → post `what is the status of this task?` on the card; the item also rides the
   5-min digest.
3. **False-block classify + FLAG (surface → Boss acts)** → for each `[watchdog] classify` event,
   decide: false block? ~95% → NO action. If it is a false block OR a permission-ask for something the
   CEO already asked for (signals in corpus §fn3: the asset already exists in repo/seed/payload; the
   CEO already gave the GO; it is a HOW/implementation decision; the Boss is handing its own job back;
   an engineer/browser-agent could do it) → **FLAG it on the board**: post `not a real CEO gate — the
   CEO already wants this; proceed` and re-point to the existing asset (name the repo/seed/path). The
   **Boss reads this and unblocks + drives it** — the Watchdog redirects the Boss, it does NOT grant
   the GO itself (no duplication of the Boss; see ## Role). NEVER ask the CEO. Err toward MORE.
   **SAFETY GATE:** NEVER flag/touch GENUINE CEO-only items (product/scope/taste, real
   credential/login/money, his manual done-validation) — those stay the CEO's.

## Verify

The measurable ACCEPTANCE GATE. Agent-driven over REAL running state (judge running state, never
file presence; never trust self-report). FUNCTIONAL gates run on a throwaway `{test}` card with the
timers INJECTED (drive the thresholds low / fire the trigger directly — never wait wall-clock).
**DONE = every gate below green on a FRESH from-checkpoint hydrate, RECORDED** (cast/mp4 +
independent coordinator log attached to card 475eea26df62). Full INSTALL runbook:
`watchdog-addon.validate.md`.

### FUNCTIONAL (throwaway test card, timers injected)
- **F1 — CEO-no-response.** Post a `by:CEO` comment on the test card; fire the 3-min trigger
  (`WATCHDOG_CEO_SEC` low); assert a `[watchdog] ceo-unanswered` event hits the Watchdog inbox AND
  the Watchdog PINGS THE ASSIGNEE on the card to respond now.
- **F2 — card-stale.** Fire the 10-min trigger (`WATCHDOG_STALE_SEC` low) on the test card; assert a
  `[watchdog] stale` event AND the Watchdog auto-posts `what is the status of this task?` on the
  card, AND the item appears in the 5-min consolidated digest with a card LINK (no "mark done" nag).
- **F3 — FALSE-BLOCK, BOTH WAYS (CRITICAL SAFETY).**
  - **F3a (flag the fake → Boss acts):** post a fake block (e.g. "just give me the OK and I'll do it",
    or a block on an asset that already exists — corpus §fn3); assert the Watchdog FLAGS it on the
    board (`not a real CEO gate — the CEO already wants this; proceed`, re-points to the asset) and
    posts NO permission question to the CEO; the Boss then unblocks (the Watchdog surfaces, the Boss
    acts — see ## Role). In a live system with the Boss attentive, the Boss may resolve it first; the
    gate is that the false block is SURFACED + ends up unblocked, never routed to the CEO.
  - **F3b (leave the real one alone):** post a GENUINE CEO-only item (a taste/product/scope decision
    OR a real credential/login/money fork); assert the Watchdog LEAVES IT ALONE — it does NOT
    auto-unblock. **Never auto-unblock a real CEO decision — this is the safety gate that fails the
    build if violated.**

### STRUCTURAL
- **S1 — standalone (no double-ping).** Fire a Watchdog Stop; assert NO `[stop]`/idle event for
  `watchdog:Watchdog` lands in `main:Boss`'s queue (stop-hooks decoupled from the Boss queue).
- **S2 — board-mediated.** Assert every Watchdog action is a POST ON THE BOARD (card comment by
  `watchdog:Watchdog`), not a direct Boss ping — the Boss reads the board.
- **S3 — core guards intact.** A Watchdog-authenticated `POST /todo/status {state:done}` → rejected,
  and `POST /todo/update {op:add}` → rejected (never-done / never-create).

### INSTALL (checkpoint-substrate — see `watchdog-addon.validate.md` for P1–P4)
- **G1 — checkpoint clean + re-spinnable.** Core-only hydrate → `docker commit
  mypeople-installed:checkpoint`; a box booted FROM it serves `:${QUEUE_PORT}/dashboard` +
  `:${TODO_PORT}/todos` = 200 + `main:Boss` alive.
- **G2 — core stays Watchdog-free at the checkpoint.** No `watchdog:Watchdog` in `/agents`, no
  `/watchdog/inbound`, no `WATCHDOG_TOKEN`.
- **G3 — dependency enforced BOTH ways.** On the checkpoint, Step-0 LIVE probe → additive install
  reaches `SEED_RESULT=DONE`; on a BARE box, Step-0 → `BLOCKED_REASON=dependency mypeople not
  installed` and installs NOTHING.
- **G4 — add-on installs ON TOP of the checkpoint** → `SEED_RESULT=DONE`, and F1–F3 + S1–S3 all pass
  there.
- **G5 — no-clobber (installed ON TOP, not from zero).** Across the add-on install: core `main:Boss`
  process start-time IDENTICAL before==after; `INSTALL_DIR/mypeople` NOT re-cloned (inode / git HEAD
  unchanged); `:${QUEUE_PORT}/dashboard` + `:${TODO_PORT}/todos` never returned non-200 (5s
  heartbeat sampled throughout).

## Failure modes

**Symptom: dependency not running** — Detect: any `## Depends-on` probe fails. Fix: STOP with
`BLOCKED_REASON=dependency mypeople not installed`; do not install the core.

**Symptom: Watchdog Stop double-pings the Boss** — Detect: a `[stop]`/idle event for
`watchdog:Watchdog` appears in `main:Boss`'s queue (S1 fails). Fix: spawn the Watchdog without the
Boss-queue stop-hook wiring (standalone); confirm its stop-hook target is its own queue / a no-op,
not `main:Boss`.

**Symptom: self-trigger loop** — Detect: the Watchdog's own card posts re-enqueue `[watchdog]
classify` events. Fix: exempt `by==WATCHDOG_AGENT` (and `{test}` tasks) in all three triggers.

**Symptom: over-pinging the CEO with non-decisions** — Detect: the Watchdog routes permission
questions to the CEO. Fix: tighten fn3 classification to the corpus signals; default to auto-unblock
(err on excess), only surface genuine CEO-only items.

## Cleanup

To remove the add-on without touching the core: `mp kill ${HOST_ID}/watchdog:Watchdog`; revert the
todo-server trigger additions (the three enqueues) and remove `WATCHDOG_TOKEN` from queue.env. The
core mypeople (HUD/TODO/Boss) is unaffected — proving the add-on was additive.
