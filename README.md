# openspec-schemas

Shared OpenSpec workflow assets — one schema, four agents — vendored into consuming repos
by [graft](https://github.com/optioni/graft).

```
extras/openspec-schemas/tdd/   the shared OpenSpec schema  → each repo's openspec/schemas/
extras/agents/                 orchestration agents        → each repo's .claude/agents/
```

Consumers get committed copies, not forks or symlinks, so CI and cloud clones work and
each repo pins its own version.

## Using it in a repo

```sh
graft add github.com/optioni/openspec-schemas@v0.1.0 schema:tdd 'agent:*'
```

Take both: the agents implement the lifecycle the schema describes, so a repo that pins
one must pin the other. Commit the synced files along with `graft.toml` and `graft.lock`.

Then set `schema: tdd` in that repo's `openspec/config.yaml`. If the schema is missing,
openspec fails loudly (`Schema 'tdd' not found`) rather than falling back — so a broken
install is visible, not silent.

Later, `graft update` moves the pin and re-syncs. Until you run it, nothing changes.

## Publishing a change

Edit here, then push and tag — a change reaches no one until it is tagged:

```sh
git push origin main
git tag -a vX.Y.Z -m "..." && git push origin vX.Y.Z
```

`catalog.yaml` declares what this repo offers and where each item belongs. New content
needs a `provides` entry there or it is invisible.

## What belongs where

The schema is **stack-agnostic**: it names no package manager, command, or framework.
Everything repo-specific lives in each repo's `openspec/config.yaml` under `context` and
`rules`, which openspec injects around the schema instruction — test tiers and their
commands, the cost of the acceptance tier, published interfaces and agreement sites, and
that repo's own concentration points. See `groceries/api` for a worked example.

Quoting note: a `rules` entry containing `": "` parses as a YAML map and openspec drops
the whole block without complaining. Quote those entries.

## Caveats

- Synced files are derived artifacts. `graft sync` overwrites them, so edit the source
  here, never the copy in a consumer — there is no way to collect an edit back.
- graft prunes only files it wrote, so a repo's own agents can sit in `.claude/agents/`
  beside the synced four and never be touched.
