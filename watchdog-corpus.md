# Watchdog Example Corpus — grounded in REAL board history (compiled 2026-06-25)

For card 475eea26df62 (CTO-proxy / Watchdog). Built by mining the full TODO board comment history
(`~/workspace/mypeople/todo/data/board.v2.json` → `memory/kb/ceo-feedback.md`, 463 CEO comments).
Purpose: let the Watchdog engineer build functions 2 (stale-status ping) + 3 (false-block classify
+ auto-unblock) from the CEO's REAL patterns, not guesses. Every quote below is verbatim from the board.

---

## WHY THE WATCHDOG EXISTS (the CEO's own framing of the pain)
The recurring pain across the whole board is **lack of visibility into task/work state** — he repeatedly
has to ASK what is happening, and gets angry when he has to ask twice.
- [2026-06-17 16:16, task-1b5abe792e23] "quantos substratos eu tenho que estão rodando e gerando as instalações? Esse é um baita problema que eu estou tendo: a falta de visibilidade de como tá sendo feito o processo... eu gostaria de conseguir visualizar quantos substratos eu tenho rodando e o que eles estão fazendo."
The Watchdog's job is to surface state BEFORE he has to ask, and to never let a task sit silently blocked.

---

## FUNCTION 2 — STALE/SLOW STATUS PING (the 10-min stale ping)

### Pattern
When a task has been in `working` (or sitting) without a fresh, visible update on the CARD, the CEO
pings asking for STATE. He does not wait — if ignored he re-pings and escalates (caps, "???",
"WHERE IS MY ANSWER"). The trigger is: time elapsed + no new card comment the CEO can see.

### Real phrasings — how he asks for state (verbatim, with task + timestamp)
- "What is the status of this?" — the single most common; recurs dozens of times (e.g. 2026-06-15 20:12, 2026-06-17 13:09/17:34, 2026-06-16 14:08).
- "What is the status of this??? How many one shots did we have? Where can I use them?" [2026-06-17 19:19, 1b5abe792e23]
- "WHat is the status of this is there a version I should test?" [2026-06-17 17:52]
- "what is happening on this task currently>?" [2026-06-17 22:55]
- "What is going on here?" [2026-06-17 20:42 / 22:07 — recurs many times]
- "What is being done here?" [2026-06-16 20:45, 1c30d20c3d93]
- "did we get learnings did we cleanup the substrates? Did we setup a new batch? what is the status ?" [2026-06-17 20:08]
- "Qual o estado disso aqui? O que eu, como usuário, vou ter acesso? O que está rodando?" [2026-06-11 19:29, 0afb2faee9bf]
- "Qual o status dessa tarefa?" [2026-06-10 21:13, 1a067fe01d29]
- "Qual o status dessa tarefa?" / "What is the STATUS OF THIS???" [2026-06-15 13:26/13:47, ebca51a786d5]
- "how many substrates do we have running here?" [2026-06-17 16:40] / "How many substrates do we have?" [2026-06-17 14:13]
- "where is them for me to use?" / "Where can I use them?" / "where is the video?" [recurs]
- "validados ou validando?" [2026-06-17 13:11] (is it validated, or still validating?)
- "Did we publish it yet?" [2026-06-17 13:15, b512ecd24a36]
- "what is your answer here?" [2026-06-16 14:33]
- "Hey boss! what is the status of our projects Exec summary in md formattation here in the task please" [2026-06-17 15:18]

### Escalation phrasings — what he says when a ping went UNANSWERED (the failure state to PREVENT)
- "Where is my answer?" / "Where is my ANSWER?" / "WHERE IS MY ANSWER????" [recurs: 06-15 21:53, 06-15 20:01, 06-11 12:22]
- "where is my RESPONSE???" [2026-06-15 14:59]
- "WHAT IS THE STATUS?? WHY ARE YOU NOT REPLYING ?" [2026-06-10 20:20, add834d5fd3c]
- "What is the status of this? I need a response HERE!" [2026-06-10 22:36]
- "WHERE IS MY RESPONSE I AM TIRED OF YOU REPLYING ELSEWHERE" [2026-06-17 16:18] (answered in pane, not on card)
- "WHERE IS THE THING I ASKED?" [2026-06-16 16:30]

### CEO's EXACT spec for the ping mechanism (gold — build function 2 to this)
- [2026-06-09 13:09, task-1a22850f3ca0] "I think every 3 minutes I could receive 1 message with all the tasks that are blocked on me instead of receiving 1 message per task; also lets make it run every 5 minutes for the CEO instead of 3"
  → ONE consolidated message, batching ALL his open/blocked tasks; cadence = every 5 minutes. NOT one ping per task.
- [2026-06-09 13:07, task-1a22850f3ca0] "I am receiving a message that says 'mark done to stop these pings' I am the CEO I know how it works I don't need that the ping is my reminder to do my work; Also if I am being pinged about that thing I need a LINK to open the right card to make things easier for me"
  → The ping IS his reminder — keep pinging until he acts. Do NOT nag "mark done to stop pings". DO include a direct card LINK in every ping.

### Implied rules (function 2)
1. If a task is `working`/owned and has had NO new CEO-visible card update for ~10 min, the Watchdog posts a proactive status line ON THE CARD (and into the 5-min consolidated CEO message) BEFORE he asks.
2. Every status surfaced must answer his standing four questions: what state, who is working it, what is blocked, and where can he see/use it (a LINK).
3. Consolidate: one message every 5 min covering all his open items, not one-ping-per-task.
4. Never make him ask twice — an unanswered ping escalating to "WHERE IS MY ANSWER" is the exact failure to prevent.
5. The answer goes ON THE CARD (Rule 29), not only in the pane ("tired of you replying elsewhere").

---

## FUNCTION 3 — FALSE-BLOCK CLASSIFICATION + AUTO-UNBLOCK REPLY

### Pattern
The Boss/engineer parks a task as "blocked on the CEO" (or stalls awaiting his approval) for something
that is NOT his to decide — an implementation detail, an asset that already exists, or an obvious GO he
already gave. The CEO corrects this sharply: it is NOT blocked on him; proceed and finish.

### Real FALSE-BLOCK corrections (verbatim — the CEO's actual voice)
- [2026-06-10 19:31, task-5e2864d00e0e] "Cara, não é possível que você não organizou ainda e não resolveu o problema. Que coisa! Não é pra você ficar bloqueado em mim, não, cara. Se eu falei pra você organizar, organiza e sobe lá." (THE canonical one: if I told you to do it, do it and push it — don't block on me.)
- [2026-06-10 16:33, task-af0a8f7032e4] "I am not seeing it updated what don'y you do your job until the end?"
- [2026-06-17 15:21, task-1b5abe792e23] "O que vc quer dizer com bloqueado em mim? O style guide do plow esta no repo do plow é so um eng pegar" (what do you mean blocked on me? — the asset is already in the repo, an engineer just grabs it)
- [2026-06-16 20:28, task-1c30d20c3d93] "what are you talking about; GO!"
- [2026-06-17 20:11, task-46bad8be2da0] "Can you not fall for this obviously we have — you just did not point the engineer to the right codebase"
- [2026-06-17 16:33, task-b512ecd24a36] "what do you mean obviously the YouTube seed existed before the engineer just skipped it"
- [2026-06-10 13:21, task-0f565a1abacb] "Para de delegar isso para outra pessoa. Você que é o boss, você que criou os agentes, você que é responsável por orquestrar o time... não outro agente, cara. Você é o boss!" (stop delegating your own orchestration job back out)
- [2026-06-16 13:38, task-637dfdbb0f31] "You can have the agent with browser control to record 1); 2; and 3) you don't need me to record those B-Rolls right" (don't put me in the loop — an agent can do it)
- [2026-06-09 15:13, task-0afb2faee9bf] "why would I need to tell you the name of the profile that sent the message? what is the point of the automation if I need to tell you the name you should receive that at the payload right?" (rejecting a fabricated dependency on him)

### Classification — signals that a "blocked on CEO" is FALSE (auto-unblock)
- The "blocker" is an asset/answer that ALREADY EXISTS (in the repo / seed / docs / payload): "style guide is in the plow repo, just grab it"; "the YouTube seed existed"; "you should receive that in the payload."
- The CEO ALREADY gave the directive to do the thing: "if I said organize it, organize it and push it."
- It is an implementation / HOW decision, not a product / intent / scope decision (doctrine Rule 11 / 31).
- The Boss is asking the CEO to do work an engineer or the browser-agent can do (record B-roll, test in browser, fetch a name).
- The Boss is handing its OWN orchestration job back to the CEO or to "another agent."

### TRUE block — what IS legitimately his (do NOT auto-unblock these)
- Genuine product/intent/scope decisions ("green light the features", which voice, publish-to-main).
- Real human-only credential/login he must personally approve, money he has not authorized.
- His manual validation of a RUNNING thing before he marks DONE (he is the sole "done" authority).

### Auto-unblock reply language (model the CEO's own tone — short, direct, finish-it)
- Internal classification line: "Not blocked on the CEO — he already gave the GO / the asset already exists / this is a how-decision. Unblocking."
- Action: re-point the engineer to the existing asset (name the repo/seed/path), give the GO, drive to live-on-remote, THEN show the result on the card.
- Reply voice the CEO rewards: "GO.", "doing it — will post the result here", "you are the boss, proceed" — never "do you want me to…" for a how-decision.
- Pairs with doctrine: Rule 26 (don't re-gate on the CEO after he said DO it), Rule 31 (Boss owns the engineering GO), Rule 11 (CEO is the user, not the approver of how).

---

## NOTES FOR THE ENGINEER
- Source data: 463 CEO comments mined from board.v2.json (via memory/kb/ceo-feedback.md). Heaviest task threads: 1b5abe792e23 (mypeople seed), 2f58c8f0cb47 (airbnb), add834d5fd3c (teleprompter), 637dfdbb0f31 (autoresearch video).
- Function 2 cadence is SPECCED by the CEO himself: 5-minute consolidated digest, one message for all blocked-on-CEO items, each with a card LINK, no "mark done to stop pings" nag.
- Function 3 must distinguish false-block (auto-unblock + GO) from true-block (surface to CEO) using the signals above — the cost of getting it wrong is either nagging him with non-decisions or silently sitting on work he already greenlit.
