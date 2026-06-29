# plow-seedlab-watchdog

**Single source of truth: the canonical `watchdog` add-on seed (generative).**

A **dependency add-on** for the [`mypeople`](https://github.com/delattre1/plow-seedlab-mypeople) seed:
a standalone, **bossless** Watchdog agent + board-triggers in the todo-server + a launchd keepalive,
that keep the board moving. Three features (SYSTEM detects/schedules/gates, AGENT composes/posts):
- **Feat A** — unanswered CEO message OR CEO-created task (default 3 min) → nudge (event-scheduled,
  still-most-recent / still-untouched gate; fires on any status).
- **Feat B** — idle card (default 10 min, skips Done/Cancelled) → the agent posts a fixed generic
  `What is the status of this?` (no title-riffing); baseline-on-activation + dedup so it never storms.
- **Feat C** — every non-CEO message is classified → a false-block ("waiting for the CEO's OK" when
  the task already exists) or a done-claim with no proof gets one correction.

Paste it into a mypeople-hydrated runtime to add the Watchdog.

## What's in this repo
- `watchdog-addon.seed.md` — the canonical add-on seed (depends on the `mypeople` core seed).
- `watchdog-corpus.md` — the validation corpus (the CEO's mined voice).
- `watchdog-addon.validate.md` — the checkpoint-substrate blind-hydrate validation runbook.

## Validation
The acceptance gate is a blind from-the-seed hydrate onto a clean mypeople (Feat A/B/C + bossless +
keepalive all proven to come from the SEED) — see `watchdog-addon.validate.md`.

> **Repo roles (2026-06-25):** this repo is the **canonical published Watchdog add-on seed**. Active
> **dev/build** lives in **`plow-pbc/mypeople`** at `seeds/watchdog-addon.seed.md`; after any change
> there, **re-publish** the file(s) here (the established per-product publish step). Do **not** put the
> add-on in `plow-seedlab-mypeople` — that repo is the core single-source mypeople seed.
