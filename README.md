# plow-seedlab-watchdog

**Single source of truth: the canonical `watchdog` add-on seed (generative).**

A **dependency add-on** for the [`mypeople`](https://github.com/delattre1/plow-seedlab-mypeople) seed:
a standalone Watchdog agent + supervisor keep-alive that monitors a mypeople node, with the Claude
Code feedback-survey suppression folded in (the survey modal otherwise freezes long-running agents
mid-action). Paste it into a mypeople-hydrated runtime to add the Watchdog.

## What's in this repo
- `watchdog-addon.seed.md` — the canonical add-on seed (depends on the `mypeople` core seed).
- `watchdog-corpus.md` — the validation corpus.
- `watchdog-addon.validate.md` — the checkpoint-substrate validation runbook (P1–P4 / G1–G5).

## Validation
Validated green on the checkpoint substrate (P1–P4 / G1–G5) — see `watchdog-addon.validate.md`.

> **Repo roles (2026-06-25):** this repo is the **canonical published Watchdog add-on seed**. Active
> **dev/build** lives in **`plow-pbc/mypeople`** at `seeds/watchdog-addon.seed.md`; after any change
> there, **re-publish** the file(s) here (the established per-product publish step). Do **not** put the
> add-on in `plow-seedlab-mypeople` — that repo is the core single-source mypeople seed.
