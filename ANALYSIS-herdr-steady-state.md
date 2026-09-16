# What the schema still gets wrong — nineteen changes at steady state

A second reading of `herdr-openspec`, covering **2026-09-08 → 2026-09-16**: 19 changes
shipped (18 archived, 1 in flight), 195 subagents, 4.22B input tokens, 23 564 subagent
turns, plus 44 main sessions at 2.70B and 11 105 turns. Source is the same corpus as
[ANALYSIS-herdr-run.md](ANALYSIS-herdr-run.md) — the session transcripts under
`~/.claude/projects/-Users-juusopiikkila-Code-herdr-openspec/` — read forward from where
that document stopped. Findings continue its numbering.

The first analysis measured a run that had no versioned agents. This one measures a run
that has them, and the shape of the spend has moved rather than gone away.

## What the earlier fixes did

Finding 1's per-group dispatch landed and is being used. Finding 3's improvised prompts
are gone: `general-purpose` dispatches fell from 170 to 2, and the 200 dispatches in this
window are 102 `outside-in-tdd-implementer`, 66 `planning-reviewer`, 18
`outside-in-tdd-reviewer`. Finding 6's planning-time execution rule is visibly working —
`tasks.md` fell from a median of ~70 kB before 2026-09-07 to ~28 kB after, while the
reviews kept finding real defects.

One structural surprise: **159 of the 200 dispatches come from a main session, not from
`apply-orchestrator`**, which was dispatched twice in nine days. The user runs apply in
the main session, so the orchestrator's context discipline reaches that session only
through `schema.yaml`'s apply instruction. Main sessions now peak at a median 309 k
context, p90 463 k, max 643 k. The agent file is not where the rule lands; the schema is.

**`herdr-openspec` is pinned to `v0.2.2` and has never synced `v0.2.3` or `v0.2.4`.** Its
vendored copies are byte-identical to that tag. So the parallelism and dispatch-prompt
evidence below measures text that has since been rewritten upstream, and is marked where
that matters.

## Finding 8 — a MODIFIED delta goes stale against the live spec, and nothing checks it

The most dangerous defect class in the corpus, and the schema has no rule aimed at it.

`foldable-spec-sections`, planning review, CRITICAL:

> **Archiving this change would have reverted `pane-chrome`.** A MODIFIED block replaces
> the live requirement wholesale, and eleven of the twenty-one carried requirements had
> bodies written against the pre-`pane-chrome` text.

`heading-sections` hit the same class twice in one change — GAP-01, a MODIFIED block that
"dropped **five** normative paragraphs present in the live spec at `:330–359`", and
GAP-06, a body "carried **byte-identical** from the live spec and therefore contradicted
this delta's own new requirement."

The structural cause is measurable. Requirements accrete: the live tree is 327
requirements over 1.65 M characters, median 3 564, p90 10 165, **max 41 672** — a single
requirement that any touching change must copy verbatim. So deltas are now mostly copies.
By bytes, MODIFIED blocks are 183 k of `pane-chrome`'s 229 k, 121 k of
`heading-sections`' 131 k, 154 k of `foldable-spec-sections`' 177 k. Specs have quietly
become the largest artifact in a change — 110–230 kB against a 30–70 kB design — and the
schema budgets `tasks.md` against `design.md` while saying nothing about either against
the specs.

A copy that large is stale the moment another change archives. The schema's staleness
machinery — `Reviewed Against`, the apply-phase drift rule — watches the *code* and the
*planning review*. Nothing watches the delta against the requirement it will overwrite.

Two repairs, both small:

- **planning-review**: for every MODIFIED requirement, diff the carried body against the
  live requirement at HEAD and account for every difference — each one is either this
  change's edit or an unnoticed revert. Recording "no unaccounted differences" is the
  check; a body carried byte-identical is a finding, not a pass.
- **apply**: the existing drift rule already re-reads HEAD before the first
  implementation task. Extend it — when an intervening commit archived a change, the
  MODIFIED bodies are re-derived from the live spec, not just the code inspected.

The deeper fix belongs in the consuming repo, not here: a 41 kB requirement is a design
problem. But the schema can say that a requirement large enough to make its own delta
unreviewable should be split, which is stack-agnostic and true everywhere.

## Finding 9 — a check that selects nothing exits 0, and the falsifiability rule misses it

Finding 6's fix made every check run at planning time and record its exit status. The
gap it left: a command that *selects nothing* succeeds.

`pane-chrome`, blocking:

> **123 of the verification matrix's 149 commands could not run.** Each passed two or
> three bare filters; `cargo test` takes one `TESTNAME`, so two bare filters exit 1 and
> the quoted form exits **0** having run nothing (`0 passed; 1222 filtered out`).

`foldable-spec-sections`: "Every `cargo test` filter in the verification matrix matched
**zero** tests … a filter that matches nothing exits 0." `markdown-legibility`: three
groups "verified with commands that **cannot run the tests they break**", so a group
"would report 'no regressions' while red." `tasks-emphasis`: "**The green baseline was
wrong by 85 tests and four binaries** … because piping `cargo test` into `grep` drops the
last four binaries' result lines." The class appears in 13 of the 41 changes, 5 of them in
the last 12.

The rule to add is one clause on the rule that already exists: **the evidence a check
records is what it ran, not that it exited.** A test command records the count it
selected and a zero is a failed check; a grep-style guard records its match count on both
halves of its negative control. `pane-chrome` had to invent this itself — "a conformance
check that runs every command in the matrix and fails on any running zero tests."

## Finding 10 — the implementer never stops, so the orchestrator's cheap repair never fires

Implementers are **68% of subagent input** in this window — the position apply held before
Finding 1. Their cost is concentrated the same way: **the top 20% carry 60% of it.**

| | n | med input | turns | gate runs | file reads | failing build/test outputs |
|---|---|---|---|---|---|---|
| top 20% | 19 | 90.3 M | 333 | 25 | 40 | 8 |
| the rest | 76 | 12.2 M | 95 | 11 | 7 | 2 |

The worst single implementer: **562 turns, 49 gate runs, 96 files read, 31 failing
compiler or test outputs, 205 M input tokens** — more than most whole changes cost. It is
not a big group. Task-line count barely separates the tail (median 6 against 5 overall);
what separates it is a build it could not get green and a manifest it read past four
times over.

`outside-in-tdd-implementer` has the vocabulary for this already — `BLOCKED`,
`NEEDS_CONTEXT`, "the manifest is the boundary of your exploration". Across **95
implementers it reported `NEEDS_CONTEXT` once and `BLOCKED` never.** An agent that never
returns never reaches `apply-orchestrator`'s repair path, which is the cheap one: "hand
the failure back to a fresh implementer for that group with the real output." Cost is
quadratic in turns, so a fresh implementer at turn 1 with the failure evidence is worth
several at turn 400.

What is missing is a trigger, not a status. Something observable: a third consecutive gate
run failing for the same cause, or the first file it needs that the manifest does not
name. Both are facts the implementer can see. The orchestrator side needs the matching
half — re-dispatch is the expected path, not an exception, and a group is allowed two
implementers before it counts as drift.

## Finding 11 — the best practice in the corpus is the one nobody wrote down

`help-overlay`'s planning review records six CRITICALs discovered not at review time but
at dispatch time: "Found while checking that `INVENTORY` could be `const` at all, before
group 4 was dispatched", "found at group 6's dispatch", "at group 7's dispatch by running
`detailwidths.sh`'s own scan over the finished `src/ui/help.rs`", "at group 8's dispatch
… by the **same** grep run repository-wide rather than against one file", "at group 9's
pre-dispatch check, before the group was dispatched", "at group 10's pre-dispatch check,
by running the comparison the requirement describes instead of re-reading it."

Three of those are one defect: a `39 → 42` content-row repair that the planning review
applied to four spec files and to neither `tasks.md` nor `design.md`. It was caught three
separate times, at groups 4, 6 and 8, because the orchestrator re-checked each group's
numbers against the tree before handing them over.

This is emergent. `apply-orchestrator`'s Step 4 pre-gather says to assemble context;
nothing says to verify it. It should: **before dispatching a group, re-run the numeric and
location claims in its task lines — counts, line numbers, "the only three places that do
X". A number that no longer reproduces is drift, and the group is not dispatched until the
owning artifact is repaired.** The orchestrator is the only actor that sees a group's
claims immediately before they are acted on, and this window says that is where a
surprising share of the surviving defects are caught.

A second-order fix belongs with it, in the planning-review instruction: **a repaired
number is repaired everywhere by a command, not by editing the artifact that raised it.**
The `39 → 42` defect was one sweep away from never existing.

## Finding 12 — `parallel-after` still has not earned its place

Across the 19 changes in this window: **5 real markers, all before 2026-09-10, none in the
last 14 changes.** Two changes proposed markers that the planning review deleted —
`help-overlay`'s group 7 "failed all three independence conditions. Its plants landed in
the **real** working tree", and `spec-emphasis`'s pair, which "contradict[ed] a decision
`archive/2026-09-15-tasks-emphasis/tasks.md:12-19` recorded one change earlier."

That is the first analysis's Finding 2 with a new cost attached: the marker now generates
review findings rather than merely being unused.

Most of this is already fixed and unsynced. `v0.2.3`'s `b4c3dcd` rewrote criterion 3 to
ask what a gate run reaches and added the repository-wide veto in as many words — "it
vetoes every pair in every change here, and it will keep doing so. State it once as the
repository fact it is." `herdr-openspec` has never received that text.

What remains is small. The schema tells the planner to state the veto once *per change*;
it could let the consumer state it once *per repository*, in `config.yaml` under `rules`,
and have the planner cite it instead of walking the pairs again. Roughly 55 lines of
per-change analysis collapse to one line, and the machinery stays intact for repositories
where parallelism actually pays.

## Finding 13 — a budget nobody keeps teaches that budgets are advisory

`proposal.md`'s instruction says "Keep it concise (under 400 words)." Across the 18
archived changes in this window: **18 of 18 over, median 1 232 words, max 1 720.** No
planning review has ever raised it.

The number is simply wrong for what the same instruction asks the proposal to contain —
Why, What Changes, Non-Goals, Capabilities and Impact, plus whatever the consuming repo's
rules add. Either raise it to what the section list needs or drop the figure.

It matters beyond itself. The tasks-versus-design ratio from `v0.2.1` is load-bearing and
is being honoured; a second, adjacent budget that is violated every single time and never
flagged is the instruction that teaches an agent which budgets are real.

## Recommendations, in order of payoff

1. **Finding 10** — a stopping rule in `outside-in-tdd-implementer` and re-dispatch as the
   expected path in `apply-orchestrator`. Implementers are 68% of the spend and their tail
   is 60% of that.
2. **Finding 8** — diff every MODIFIED body against the live requirement in
   planning-review, re-derive it in apply when an intervening change archived, and say
   that an unreviewably large requirement should be split. This is the only class here
   that silently reverts shipped work.
3. **Finding 11** — verify a group's numeric claims at dispatch, and repair a number
   everywhere with one command.
4. **Finding 9** — a check records what it selected, and selecting nothing is a failure.
5. **Finding 13** — fix the proposal word budget.
6. **Finding 12** — let `config.yaml` carry a repository-wide parallelism veto.

Consumer-side, and first: `graft update github.com/optioni/openspec-schemas --to v0.3.0`
in `herdr-openspec`. It has been on `v0.2.2` for all nineteen changes, and two of the
pains above are already fixed upstream.
