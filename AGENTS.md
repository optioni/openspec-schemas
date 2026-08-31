# AGENTS.md

## Project Overview

This repository is a **content registry**, not an application. It holds one shared
OpenSpec schema and a set of Claude Code agents, and distributes them to every other
repo and to `~/.claude` via [skillshare](https://skillshare.runkids.cc). The point is
**one schema to rule them all**: consumers get copies, not forks, so the workflow cannot
drift repo to repo.

There is no source code, no package manager, no build, and no test suite. Every file
here is Markdown or YAML that some other tool reads.

```
extras/openspec-schemas/tdd/   the shared OpenSpec schema  → each repo's openspec/schemas/
extras/agents/                 orchestration agents        → each repo's .claude/agents/
skills/                        skillshare skills source, currently empty
catalog.yaml                   what this repo publishes, for graft consumers
config.yaml                    skillshare config — GITIGNORED, machine-local
```

This repo *is* skillshare's root: `~/.config/skillshare` is a symlink to it. `config.yaml`
at the root is skillshare's own config, not project config.

`catalog.yaml` is the pull-side equivalent: skillshare pushes from targets declared in the
gitignored `config.yaml`, while graft consumers pull by naming this repo and an item. It
declares one entry per published item — the `tdd` schema and the four agents — with the
path each one lives at here and the path a consumer puts it at (`{name}` interpolates the
item name; `flatten: true` drops the agents straight into `.claude/agents/` with no
nesting). Those destinations mirror what skillshare already writes, so the two
distribution paths agree byte for byte. Adding content means adding its `provides` entry
here too, and consumers pin a tag, so a change reaches them only once it is pushed and
tagged.

## Setup

- Requires the `skillshare` CLI (`brew install`; verify with `skillshare --version`).
- No install step. Clone, edit Markdown/YAML, sync.

## The distribution workflow

Push content to every declared target:

```sh
git -C ~/Code/openspec-schemas pull && skillshare sync extras --force
```

`--force` is **mandatory**. Without it, copy mode silently skips files whose target copy
changed and still reports success — you get "synced" with nothing updated. `skillshare
pull` does not substitute: it syncs skills only, never extras.

Add a new consumer repo:

```sh
skillshare extras openspec-schemas --add-target /path/to/repo/openspec/schemas
skillshare extras agents --add-target /path/to/repo/.claude/agents
skillshare sync extras --force
```

Both targets, always: the agents implement the lifecycle the schema describes, so a repo
that pins one must pin the other. Commit the synced files in the consumer repo.

Then set `schema: tdd` in that repo's `openspec/config.yaml`. A missing schema makes
openspec fail loudly (`Schema 'tdd' not found`) rather than fall back, so a broken target
is visible rather than silent.

To keep an edit that was made directly in a target, run `skillshare extras collect` to
pull it back into this source before the next forced sync — otherwise `--force`
overwrites it.

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

Four Claude Code agents, flat `.md` files with YAML frontmatter (`name`, `description`,
`model`):

- `apply-orchestrator` — implements a change group by group inline
- `outside-in-tdd-implementer` — escape hatch for one oversized group
- `outside-in-tdd-reviewer` — reviews a finished change, reports only
- `phase-orchestrator` — drives ff-change → apply → archive across many changes

`name` in frontmatter must match the filename stem; `flatten: true` in the target config
means these land directly in the target `.claude/agents/` with no directory nesting.

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
  to update.
- Commit straight to `main`; no feature branch unless the work genuinely needs isolation.
- Never push or open a PR unless explicitly asked.

## Gotchas

- `config.yaml` is gitignored because targets are absolute paths. Content travels between
  machines; **target declarations do not** and must be re-declared per machine.
- `skillshare --dry-run` is not side-effect free — it has created directories while
  "previewing". Do not treat it as a safe preview.
- Targets hold real copied files, not symlinks, so CI and cloud clones work and each
  consumer pins its own version by committing what it synced. A change here reaches a
  consumer only after that consumer syncs and commits.
- Editing a repo's `.claude/agents/*.md` directly edits a *copy*. The source is
  `extras/agents/`; the next forced sync overwrites it.
- A repo's `.claude/agents/` also holds its own repo-specific agents. Sync only writes the
  four names in `extras/agents/` and leaves the rest alone.
