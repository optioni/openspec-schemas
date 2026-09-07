# Where the tokens and the hours went — the herdr-openspec run

Analysis of the `herdr-openspec` build (2026-09-03 19:58 → 2026-09-05 17:09, ~45h wall,
14 changes shipped) as evidence for editing the `tdd` schema and the four agents in this
repository. Source: the session transcript and its 124 subagent transcripts under
`~/.claude/projects/-Users-juusopiikkila-Code-herdr-openspec/`.

Finding 7 is later evidence — the `gate-integrity` change of 2026-09-07, plus a direct probe
run in this repository. It is filed here because it bounds a proposed extension of Finding 1
rather than standing on its own.

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

> **Corrected.** This section first claimed "8 of 14 changes carry those markers." That
> was wrong, and the error is instructive: the grep matched the planners' *prose*
> explaining why they had **not** used the marker. The v0.2.0 commit and tag messages
> carry the wrong number and cannot be edited; this is the record.

`schema.yaml` instructs the planner to mark independent groups
`<!-- parallel-after: N -->`. `apply-orchestrator.md` contained no handling for them and
forbade the fan-out that would use them ("never split a normal change into per-group
subagents by default"). That contradiction inside one shipped bundle was real and is
fixed in v0.2.0.

What it cost, though, was nothing. **The whole corpus contains exactly one marker** —
`list-view` group 3, `parallel-after: 1` — and a single marked group has no sibling to run
beside, so even that one yields no parallelism. The planners were right to decline, and
said so explicitly, five times in near-identical words:

> No group carries a `parallel-after` marker. Groups 2 through 8 all edit
> `src/changes.rs`, and one file is shared mutable state.

This is a single-crate Rust project with eleven source files and a whole-tree gate. Its
groups are sequential by construction. So the applier discarded an analysis the planner
had correctly concluded was empty — no wall-clock was lost, and `parallel-after` has yet
to earn its place here even though the machinery now works.

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

## Finding 7 — a dispatch prompt is not cached across sibling agents

Finding 1's fix — one implementer per task group, each starting cold — invites an obvious
follow-on. The groups of one change share a proposal and a design, so if that shared text
opened every dispatch in identical bytes, only the first group would pay for it. On the
`gate-integrity` apply the opportunity looked real: the common prefix across all sixteen
dispatch prompts is **0 characters** — every one opens "You are implementing task group
N" — while each child was already reusing a constant 28 819-token warm prefix and creating
roughly 28 600 more on top of it.

It does not work. Three subagents, same type, same model, dispatched back to back:

| probe | prompt | cache_creation | cache_read | total |
|---|---|---|---|---|
| A | 4 274 chars | 29 344 | 0 | 29 354 |
| B | 4 274 chars, identical to A | 10 459 | **18 885** | 29 354 |
| C | 561 chars, sharing nothing with A | 9 625 | **18 885** | 28 520 |

C read the same 18 885 tokens as B while sharing nothing with A beyond the system prompt and
the tool definitions. That number is the fixed system-and-tools prefix, independent of the
prompt, and B's four thousand identical characters bought none of it. The arithmetic agrees:
B's cache_creation exceeds C's by 834 tokens, and B's prompt exceeds C's by 3 713 characters,
about 928 tokens. The whole dispatch prompt falls after the breakpoint and is re-created for
every group. There is no cache breakpoint inside the first user message.

> **Shipped, then corrected.** `97d9f59` wrote the caching rationale into
> `apply-orchestrator.md` and `schema.yaml`, naming the measurement as outstanding. The
> measurement then ran and disproved it; `c1e72fb`, `738538b` and `c75bb2e` removed the claim.
> It is recorded here because the recommendation outlived its justification, and a reader who
> finds only the surviving half would reasonably re-propose the dead one.

What survives is smaller and true. Quoting proposal.md and design.md into the dispatch is
**token-neutral** — the implementer pays for them as prompt instead of as a tool result —
but it removes the fetch and the searching that follows. Group 5 of `gate-integrity` opened with
`cat design.md`, `cat proposal.md`, `cat spec.md`, then four greps and an awk hunting the two
paragraphs that belonged to it: ten of its first twenty tool calls, 3.7 minutes before its
first edit. Across the groups that orientation phase ran 3 to 12 minutes each, 19% to 57% of
a group's wall time. The prize is turns — roughly 15 to 25 minutes on a run of this size —
and not the halved cold start the prefix theory promised.

Two adjacent ideas fall with it. Forking each group off the orchestrator inherits the
orchestrator's *current* context, which on this run measured 129 864 tokens at the first
dispatch and 236 830 at the last, against a 58 000-token cold start — reintroducing Finding 1
in a new costume. Priming implementers ahead of their turn is sound in principle, since
orientation is read-only and cannot violate the attributability criterion, but its ceiling is
the same orientation phase, and an agent primed more than an hour before its turn pays its
cold start again anyway.

## Recommendations, in order of payoff

*All six landed — 1 and 2 in `3f71e47`, the rest in the commit that added this line. One
part was dropped on inspection: the "thin plan-gen / archive drivers" under 3. Those two
dispatch prompts are already two lines each inside `phase-orchestrator`, so a wrapper agent
would not have prevented the reinvention — that came from the main session dispatching them
directly instead of through the phase orchestrator.*


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
