# Where the tokens and the hours went — the herdr-openspec run

Analysis of the `herdr-openspec` build (2026-09-03 19:58 → 2026-09-05 17:09, ~45h wall,
14 changes shipped) as evidence for editing the `tdd` schema and the four agents in this
repository. Source: the session transcript and its 124 subagent transcripts under
`~/.claude/projects/-Users-juusopiikkila-Code-herdr-openspec/`.

## The shape of the spend

| | agents | input tokens | share | assistant turns |
|---|---|---|---|---|
| apply | 14 | 2 987M | **70.4%** | 7 698 |
| planning (artifact generation) | 17 | 576M | 13.6% | 2 609 |
| review (planning + change) | 64 | 538M | 12.7% | 4 507 |
| other (orchestrators, archive) | 29 | 142M | 3.4% | 1 676 |
| main session | 2 | 118M | — | 489 |

Subagent totals: **4.24B input tokens** (101M cache-creation, 4.14B cache-read), 3.29M
output, 16 490 turns. The top 10 agents alone are 63.4% of all input.

Cost here is not tool output volume — every tool result in the whole run is 29.8 MB. Cost
is **turns × accumulated context**. The average turn in this run carried a 251k-token
context.

## Finding 1 — the single-context apply is the whole bill

`apply-orchestrator.md` states the policy outright: work every task group "inline, in your
own context… isolating per group costs more than it saves." Measured, the opposite holds by
a wide margin.

The worst case, the `markdown-viewer` apply:

- 932 assistant turns, 97 minutes, **context grew 49k → 811k and never compacted**
- 504M cache-read tokens — **12% of the entire run in one agent**
- context composition: `Read` results 37.6%, harness attachments 18.6%, `Bash` results
  15.4%, `Edit` inputs 13.3%

Every one of those 932 turns re-read the whole accumulated context. The cost is quadratic
in conversation length, and the schema's own group structure (6–18 groups per change,
median 12) is the natural seam that was declined.

Counterfactual, charging a **full 49k cold start per group** (re-read proposal, design,
specs, tasks):

| change | inline | split at 12 groups | saving |
|---|---|---|---|
| markdown-viewer | 504M | 76M | **6.6×** |
| list-view | 476M | 75M | 6.4× |
| detail-view | 383M | 68M | 5.7× |
| tasks-tab | 320M | 57M | 5.6× |
| schema-model | 128M | 26M | 4.9× |
| repo-foundation | 95M | 28M | 3.4× |

Across the 12 heavy applies: **3.4–6.6×**, larger the bigger the change. Applied to the
70% of spend that apply represents, that is roughly a 3× cut to the whole run.

The re-derivation tax the current policy is avoiding is real but small: measured redundant
file re-reads inside heavy agents run 1–93 per agent, and the artifacts that define the
work are ~50k tokens — cheap to re-read twelve times, ruinous to carry for 900 turns.

## Finding 2 — the bundle contradicts itself on parallel groups

`schema.yaml` instructs the planner to mark independent groups
`<!-- parallel-after: N -->`. **8 of 14 changes carry those markers.**
`apply-orchestrator.md` contains no handling for them and forbids the fan-out that would
use them ("never split a normal change into per-group subagents by default").

The planner did the analysis; the applier discards it. This is wall-clock lost as well as
tokens — apply accounted for 726 agent-minutes.

## Finding 3 — the agent set has holes, so prompts got reinvented 40+ times

| dispatched role | agents | distinct hand-written prompt openings | agent definition exists? |
|---|---|---|---|
| planning-review | 60 | **24** | **no** |
| skill-driver (`opsx:ff`, archive) | 23 | 7 | **no** |
| apply | 18 | **13** | yes |
| plan-gen | 6 | 4 | **no** |
| change-review | 5 | 5 | yes |

620k characters (~155k tokens) of agent prompts were hand-authored in the main session.
The schema defines `planning-review` as a first-class artifact, but nothing in
`extras/agents/` produces it — so the reviewer contract was re-improvised 24 different
ways, which also means the 60 reviews are not comparable to each other.

Worse, `apply-orchestrator` **exists** and was still bypassed: 13 distinct bespoke apply
prompts, median 4 727 characters, one of them opening

> "## Hard rules — repeated because they have been violated before … **DO NOT create a git
> worktree.** … Apply subagents on this project have done this unprompted twice."

An agent definition that has to be overridden in every dispatch is not doing its job.

## Finding 4 — the worktree step misfires and costs a turn every time

`apply-orchestrator` Step 1.6 mandates `git worktree list` plus a judgement call ("if
another session may be active"). Measured: **24 agents ran a worktree command**, and
**3 actually created a worktree** nobody wanted, on a single-session repo where the human
had to say "do not" in the prompt three separate times.

## Finding 5 — the mandatory per-group usage check is a vendored dependency that cannot hold

The per-group `fetch-usage.sh` gate fired **170 times across 21 agents**. Each is a turn
paying a full context read — at the contexts these agents ran at, roughly 40M tokens of
pure overhead. It also hard-codes a path into the *user's* `~/.claude/skills/`, which a
repo-level vendored agent is explicitly not allowed to assume exists.

## Finding 6 — what the reviewers kept catching is a gap in the tasks instruction

223 gap findings across the 14 `planning-review.md` files. The dominant recurring
categories, by keyword: **RED / falsifiability (19)**, test (18), task (17), scenario (10),
**numeric counts (7)**.

Representative titles: *"Extracted and run, it …"*, *"measured baseline plus zero"*,
*"Measured and false"*, *"measured, not asserted"*, *"9 `content_lines` call sites counted
the definition; the real number is …"*.

The pattern is one thing: **checks written into `tasks.md` that were never executed at
planning time, or that cannot fail.** The schema's tasks instruction is 90 lines long and
says nothing about running a check before shipping it, nor about proving it can go red.
The human and the reviewers invented that discipline per change, and it caught real
defects every time.

## Recommendations, in order of payoff

1. **Replace the inline-only apply policy with a context budget.** Default to per-group
   dispatch above a threshold (group count, or observed context), keeping inline for small
   changes. This is the ~3× lever on the whole run.
2. **Teach `apply-orchestrator` to read `parallel-after`** and dispatch marked groups
   concurrently. Otherwise remove the marker from `schema.yaml` — shipping both is worse
   than shipping either.
3. **Add a `planning-reviewer` agent** (and thin `plan-gen` / `archive` drivers) to
   `extras/agents/`, so the 24 improvised contracts collapse into one versioned one.
4. **Add to the tasks instruction:** every check script must be executed at planning time
   with its exit status recorded, and must come with a negative control proving it can
   fail; every numeric claim must be produced by a command, not asserted.
5. **Demote the worktree step** to opt-in via the consumer's `config.yaml`, and drop the
   "if another session may be active" heuristic — it guessed wrong three times out of
   three.
6. **Drop the hard `fetch-usage.sh` dependency** from the agent; a vendored repo-level
   agent cannot reach into `~/.claude/skills/`.

Consumer-side (herdr-openspec, not this repo): the rules re-typed into every prompt — nvm
`PATH` for the `openspec` CLI, never edit graft-vendored directories, never push or open
PRs, work in the main checkout — belong in that repo's `openspec/config.yaml` under
`context`, where `openspec instructions apply` hands them to the agent for free.
