# gc registry

Local pack store + discovery surface for [`gc import`](https://github.com/donbox/gc-import), shipped as a Gas City pack.

> **🚧 Status: design only.** This repo currently contains the design doc and a stub layout. The implementation is in progress; `gc registry` v1 will follow once `gc import` v1 ships and the design has had more bake time.
>
> The full design lives in [docs/design.md](docs/design.md). Read that first.

## What it does (the short version)

`gc registry` answers two questions that `gc import` deliberately doesn't:

1. **Where does pack content live on this machine?** A user-visible local registry at `~/.gc/cache/repos/` that holds git clones of every pack any city on this machine has imported. Equivalent to `~/.cargo/registry` or `$GOPATH/pkg/mod`. `gc registry list` walks it on demand and prints what it finds — no separate index file, no metadata layer, the cache directory IS the source of truth.
2. **How do users find packs they don't already know about?** A discovery surface that queries the **canonical Gas City registry** — a single curated TOML catalog of name → URL → metadata, hosted as a git repo and updated via PRs. Internet registries never participate in build-time resolution; they're pointer services that point users at git URLs to feed into `gc import add`.

## Five verbs (planned)

| Command | What it does |
|---|---|
| `gc registry list` | Walks `~/.gc/cache/repos/` on demand and prints every (name, version) currently in the local pack store |
| `gc registry info <name>` | Show metadata for a name (`--remote` to also query the URL for fresh tags) |
| `gc registry add <url> [--version <c>] [--name <alias>]` | Pre-warm the registry — fetch a URL into `~/.gc/cache/repos/` without touching any city. Optional `--version` constraint and `--name` alias for collisions. |
| `gc registry remove <name>[@<version>]` | Evict a name (or a specific version). Errors if any city's `pack.lock` references the version; override with `--force`. |
| `gc registry search <query>` | Discovery — queries the canonical Gas City registry. `--local` restricts to what's already cached. |

(Earlier drafts had a sixth verb, `gc registry rebuild`, to repair a separate index file. v1 doesn't have an index file, so there's nothing to rebuild — `gc registry list` walks the cache on demand. Five verbs.)

## How it relates to `gc import`

- `gc import` operates on a **city** — adds, removes, installs, upgrades, lists packs in the city's `pack.lock` and `[imports]` section.
- `gc registry` operates on the **machine-wide pack store** — the user's universe of "what packs do I have on disk."
- They share `~/.gc/cache/repos/` as the underlying store. `gc import add` populates it as a side effect; `gc registry add` populates it explicitly without touching any city. Either way, `gc registry list` sees the new clone the next time it walks the cache.
- Internet registries (the discovery layer) **never participate in resolve-time loading**. `gc import install` only ever talks to git, never to a registry server. If every internet registry on the planet went down tomorrow, `gc import install` would still work for any city whose `pack.lock` is committed.

## Status of the design

Per [docs/design.md](docs/design.md):

- **Settled:** internet registries are pure discovery; federation is out of scope for v1; the pack store stays at `~/.gc/cache/repos/` with no new files (no index file, no schema); URLs remain identity for `gc import`; wire format is TOML (`application/toml`, RFC 9618); name collisions are resolved with `--name <alias>`; the user-facing experience is single-canonical-registry only.
- **Open (still need answers):** what's actually in the canonical registry on day one (probably some combination of Pack, Registry, gastown, maintenance, dog), reverse-reference tracking for `gc registry remove`, default network behavior of `gc registry info`. See "Open questions" in the design doc.
- **Deferred to later phases:**
  - **v1.5**: bare-name resolution in `gc import add` (explicit-only mode — `gc import add gastown` works iff `gastown` is already in the local registry).
  - **v.next**: Shape C cache collapse (the city's loader reads from the registry directly, no per-city `.gc/cache/packs/`), tied to the Pack/City refactor.

## The canonical registry

The v1 design has **one** canonical Gas City registry, hosted as a git repo (working name: `gastownhall/gc-registry-canonical`; final name TBD). It's the registry every `gc-import` install talks to by default, and it's the registry every user-facing piece of documentation refers to when it says "the registry."

**Adding a pack** = file a PR against the canonical registry repo. The PR adds an entry to `registry.toml` plus an optional `packs/<name>.toml` file with richer metadata. Reviewers merge after a quick check that the URL is reachable, the pack has at least one semver tag, and the description isn't actively misleading. There is no automated scanning, no signing, no provenance verification — the PR review is the curation step, and the merge is the publish event.

This is the same model as homebrew-core, MELPA, and a hundred other community-curated registries. Curation scales with submission volume; for an ecosystem the size of Gas City's, that's the right shape.

## Project layout (planned)

```
gc-registry/
├── pack.toml                  # declares the [[commands]] entries
├── README.md                  # this file
├── docs/
│   └── design.md              # the canonical design doc (paired with doc-packman.md)
├── doctor/
│   └── check-python.sh        # verifies Python 3.11+ is available
├── commands/
│   ├── list.py                # walks ~/.gc/cache/repos/ on demand
│   ├── info.py                # reads a specific clone's pack.toml
│   ├── add.py                 # pre-warm: fetch a URL into the cache
│   ├── remove.py              # evict a clone (with reverse-reference check)
│   └── search.py              # query the canonical registry over HTTP
└── lib/
    ├── __init__.py
    ├── store.py               # ~/.gc/cache/repos/<hash>/ management (walks)
    ├── protocol.py            # internet registry HTTP/TOML client
    ├── canonical.py           # the URL of the canonical registry (constant)
    └── ui.py                  # consistent output formatting
```

(No `lib/index.py` or `lib/config.py` — there is no index file, and v1 has no user-facing registry config to manage.)

## License

Same as Gas City.

## See also

- [`gc-import`](https://github.com/donbox/gc-import) — the package manager that consumes packs by URL, shipped as a separate Gas City pack. `gc registry` is its companion.
- [Gas City](https://github.com/gastownhall/gascity) — the main project.
- [gastownhall/gascity#447](https://github.com/gastownhall/gascity/issues/447) — the GitHub issue tracking this design.
