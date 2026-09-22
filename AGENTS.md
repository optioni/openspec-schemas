# AGENTS.md

## Project Overview

This repository is a **content registry**, not an application. It holds one shared
OpenSpec schema and a set of Claude Code agents, and publishes them to every other repo
via [graft](https://github.com/optioni/graft), which vendors them in as committed copies.
The point is **one schema to rule them all**: consumers get copies, not forks, so the
workflow cannot drift repo to repo.

There is no source code, no package manager, no build, and no test suite. Every file
here is Markdown or YAML that some other tool reads.

```
extras/openspec-schemas/tdd/   the shared OpenSpec schema  → each repo's openspec/schemas/
extras/agents/                 orchestration agents        → each repo's .claude/agents/
catalog.yaml                   what this repo publishes, and where each item belongs
```

## Setup

Nothing to install here — clone, edit Markdown/YAML, commit. Publishing needs only `git`.
Consumers need the `graft` CLI (`brew install --cask optioni/tap/graft`); this repo does
not run it.

## Publishing

Distribution is **pull-based and version-pinned**. Nothing here reaches a consumer until
it is tagged and that consumer asks for it:

```sh
git commit ... && git push origin main
git tag -a vX.Y.Z -m "..." && git push origin vX.Y.Z
```

Tag only a pushed commit — a tag pointing at a commit no one can fetch is worse than no
tag. Current published version: `v0.4.0`.

Then, in each consumer, `graft update <source> --to vX.Y.Z` moves the pin and syncs.
Plain `graft update` does **not** find the new tag — it re-resolves the rev already in
`graft.toml` to the sha it names today, so against a pinned tag it reports "up to date"
forever. `--to` is the one path in `update` that writes `graft.toml`; run it with
`--dry-run` first to see the file list. A consumer that never runs it stays on its old
version indefinitely, which is the intended behavior: a repo upgrades its schema and
agents together, deliberately, and the diff is reviewable because the synced files are
committed.

## `catalog.yaml`

The catalog is this repo's half of the contract: it declares every published item and
where a consumer should put it. graft knows nothing about schemas or agents — the
conventions arrive as data from this file.

```yaml
kinds:
  schema:
    to: "openspec/schemas/{name}"    # {name} interpolates the item name
  agent:
    to: ".claude/agents/"            # trailing slash: the file lands in this directory
    flatten: true                    # no nesting, whatever the source layout

provides:
  - { kind: schema, name: tdd, from: extras/openspec-schemas/tdd }
```

- `to` is a path in the **consumer's** repo, and a consumer may override it in its own
  `graft.toml`. The kind name carries no guarantee; the destination is what a consumer
  actually agrees to, and `graft add` shows it before writing anything.
- An item's identity is `kind:name`, which is stable across restructuring. `from` may
  move freely inside this repo without touching any consumer's config — so reorganizing
  `extras/` is safe as long as `provides` keeps up.
- Adding content means adding its `provides` entry. An item with no entry is invisible,
  no matter where it sits.

Format rules graft enforces, so a mistake fails loudly rather than half-syncing: exactly
one YAML document, `version: 1`, unknown keys rejected, and every `from` relative and
inside the repo.

## Consuming this repo

```sh
graft add github.com/optioni/openspec-schemas@v0.4.0 schema:tdd 'agent:*'
```

Take both the schema and the agents, always: the agents implement the lifecycle the
schema describes, so a repo that pins one must pin the other. Commit the synced files
along with `graft.toml` and `graft.lock`.

Then set `schema: tdd` in that repo's `openspec/config.yaml`. A missing schema makes
openspec fail loudly (`Schema 'tdd' not found`) rather than fall back, so a broken
install is visible rather than silent.

## Repository layout

### `extras/openspec-schemas/tdd/`

`schema.yaml` defines five artifacts in order: `proposal`, `specs`, `design`, `tasks`,
`planning-review`. Each artifact has an `instruction` block (what the agent must produce)
and a `template` pointing at `templates/<name>.md`. Instruction and template are a pair —
change one and check the other still matches.

`templates/` holds the skeleton files agents fill in: `proposal.md`, `spec.md`,
`design.md`, `tasks.md`, `planning-review.md`. Templates carry HTML comments as inline
guidance; those comments are load-bearing instructions to the implementing agent, not
decoration.

### `extras/agents/`

Six Claude Code agents, flat `.md` files with YAML frontmatter (`name`, `description`,
`model`):

- `apply-orchestrator` — coordinates a change, dispatching one implementer per task group
- `outside-in-tdd-implementer` — implements one task group; the standard unit of apply work
- `outside-in-tdd-reviewer` — reviews a finished change against its plan, reports only
- `planning-reviewer` — reviews one slice of a plan before implementation, reports only
- `phase-orchestrator` — drives ff-change → apply → archive across many changes
- `repo-locator` — answers one location question, returning paths and never file contents

`name` in frontmatter must match the filename stem, and `flatten: true` in the catalog
means these land directly in the consumer's `.claude/agents/` with no directory nesting.

They install **per repo, not into `~/.claude/agents`**. Repo-level keeps agents and schema
pinned to the same version, and makes them present in CI and in cloud clones, which never
see a user-level `~/.claude`. The schema's apply instruction tests exactly this — "if this
repository defines implementer and reviewer subagent types" — so a user-level install
reads as absent.

## Editing rules

**The schema is stack-agnostic.** It must name no package manager, no command, no
framework, no language. Anything repo-specific belongs in the consuming repo's
`openspec/config.yaml` under `context` and `rules`, which openspec injects around the
schema instruction — test tiers and their commands, what an acceptance test costs there,
published interfaces and agreement sites, that repo's own concentration points. If an
edit here would only make sense in one repo, it is in the wrong file.

**YAML quoting trap.** A `rules` entry containing `": "` parses as a YAML map and openspec
drops the whole block without complaining. Quote any such entry. The same hazard applies
to `instruction` text in `schema.yaml`.

**Verification.** There is nothing to run here. Validate a change by syncing it into a
consumer repo and running `openspec validate <change> --strict` there.

## Code style

- Markdown and YAML only. Match the surrounding wrapping (~90 columns) and comment style.
- English everywhere — content, comments, commit messages.
- Template guidance goes in HTML comments so it survives into the generated artifact and
  can be deleted by the agent that fills it in.

## Commits

- Conventional Commits: `type(scope): description` (`feat`, `fix`, `docs`, `refactor`,
  `chore`). Use `!` or a `BREAKING CHANGE:` footer when a schema change forces consumers
  to update — that footer is the signal to cut a major tag.
- Commit straight to `main`; no feature branch unless the work genuinely needs isolation.
- Never push or open a PR unless explicitly asked. Since publishing *is* a push and a tag,
  a change sitting unpushed on `main` has reached nobody — that is a normal state here,
  not a mistake to fix unprompted.

## Gotchas

- Synced files in a consumer are **derived artifacts**. `graft sync` overwrites them
  without asking, so editing a repo's `.claude/agents/*.md` or `openspec/schemas/tdd/`
  edits a copy that the next sync discards. The source is here. There is no way to
  collect an edit back — reproduce it in this repo instead.
- graft prunes only files it wrote, tracked per item in the consumer's `graft.lock`, so a
  repo's own agents can sit in `.claude/agents/` beside the synced six and never be
  touched.
- Consumers hold real copied files, not symlinks, so CI and cloud clones work and each
  consumer pins its own version by committing what it synced.
- `rev = "main"` is a legal pin and will not drift, because `sync` never re-resolves —
  but `graft update` on it jumps to whatever `main` is at that moment, unreviewed. Tags
  exist so that upgrade is a decision.
