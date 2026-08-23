# openspec-schemas

Shared OpenSpec workflow assets, distributed by [skillshare](https://github.com/runkids/skillshare).

This repo is skillshare's root: `~/.config/skillshare` is a symlink to it.

```
extras/openspec-schemas/tdd/   the shared OpenSpec schema  → each repo's openspec/schemas/
extras/agents/                 orchestration agents        → ~/.claude/agents
```

## Updating everywhere

```sh
git -C ~/Code/openspec-schemas pull && skillshare sync extras --force
```

`--force` is **required**. Without it, copy mode silently skips files that changed and
reports success — you get "synced" with nothing updated. `skillshare pull` does not help
here either; it syncs skills only, not extras.

Targets hold real copied files, not symlinks, so CI and cloud clones work and each repo
pins its own version by committing the synced files.

## Adding a repo

```sh
skillshare extras openspec-schemas --add-target /path/to/repo/openspec/schemas
skillshare sync extras --force
```

Then set `schema: tdd` in that repo's `openspec/config.yaml`. If the schema is missing,
openspec fails loudly (`Schema 'tdd' not found`) rather than falling back — so a broken
target is visible, not silent.

## What belongs where

The schema is **stack-agnostic**: it names no package manager, command, or framework.
Everything repo-specific lives in each repo's `openspec/config.yaml` under `context` and
`rules`, which openspec injects around the schema instruction — test tiers and their
commands, the cost of the acceptance tier, published interfaces and agreement sites, and
that repo's own concentration points. See `groceries/api` for a worked example.

Quoting note: a `rules` entry containing `": "` parses as a YAML map and openspec drops
the whole block without complaining. Quote those entries.

## Caveats

- `config.yaml` is gitignored (targets are absolute paths), so target declarations do not
  travel between machines. Content does; targets are re-declared per machine.
- `skillshare --dry-run` is not side-effect free — it has created directories while
  "previewing".
- To keep a local edit made in a target, `skillshare extras collect` it back into the
  source before the next forced sync.
