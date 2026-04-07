# gc registry

Local pack store + discovery surface for [`gc import`](https://github.com/donbox/gc-import), shipped as a Gas City pack.

> **🚧 Status: design only.** This repo currently contains the design doc and a stub layout. The implementation is in progress. The companion `gc import` package is shipping first; `gc registry` v1 will follow once the design has had more bake time.
>
> The full design lives in [docs/design.md](docs/design.md). Read that first.

## What it does (the short version)

`gc registry` answers two questions that `gc import` deliberately doesn't:

1. **Where does pack content live on this machine?** A user-visible local registry at `~/.gc/cache/repos/` that stores git clones of every pack any city on this machine has imported. Equivalent to `~/.cargo/registry` or `$GOPATH/pkg/mod`, but TOML-indexed and inspectable.
2. **How do users find packs they don't already know about?** A discovery surface that queries one or more **internet registries** — pure pointer services that publish a TOML catalog of name → URL → metadata. Internet registries never participate in build-time resolution; they're just phone books.

The two are paired in this repo because they share an index file and the same set of CLI commands manages both.

## Six verbs (planned)

| Command | What it does |
|---|---|
| `gc registry list` | Show every (name, version) currently in the local registry |
| `gc registry info <name>` | Show metadata for a name (`--remote` to also query the URL for fresh tags) |
| `gc registry add <url> [--version <c>] [--name <alias>]` | Pre-warm the registry — fetch a URL into `~/.gc/cache/repos/` without touching any city. Optional `--version` constraint and `--name` alias for collisions. |
| `gc registry remove <name>[@<version>]` | Evict a name (or a specific version). Errors if any city's `pack.lock` references the version; override with `--force`. |
| `gc registry search <query>` | Discovery — queries configured internet registries via the TOML protocol. `--local` restricts to what's already cached. `--from <name>` restricts to one configured registry. |
| `gc registry rebuild` | Repair operation — walks `~/.gc/cache/repos/`, parses each clone's `pack.toml`, rewrites `index.toml`. |

## How it relates to `gc import`

- `gc import` operates on a **city** — adds, removes, installs, upgrades, lists, freezes packs in the city's `pack.lock` and `[imports]` section.
- `gc registry` operates on the **machine-wide pack store** — the user's universe of "what packs do I have on disk."
- They share `~/.gc/cache/repos/` as the underlying store. `gc import add` populates the registry as a side effect; `gc registry add` populates it explicitly without touching any city.
- Internet registries (the discovery layer) **never participate in resolve-time loading**. `gc import install` only ever talks to git, never to a registry server.

## Status of the design

Per [docs/design.md](docs/design.md):

- **Settled:** internet registries are pure discovery, federation is out of scope for v1, the pack store stays at `~/.gc/cache/repos/` (no directory rename in v1), URLs remain identity for `gc import`, wire format is TOML (`application/toml`), name collisions are resolved with `--name <alias>`.
- **Open (still need answers):** what's actually in the default `gascity-stdlib` internet registry on day one, reverse-reference tracking for `gc registry remove`, default network behavior of `gc registry info`. See "Open questions" in the design doc.
- **Deferred to later phases:** bare-name resolution in `gc import add` (phase 1.5), implicit standard library (phase 2, jointly with `gc import`), Shape C cache collapse (v.next, with the Pack/City refactor).

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
│   ├── list.py
│   ├── info.py
│   ├── add.py
│   ├── remove.py
│   ├── search.py
│   └── rebuild.py
├── lib/
│   ├── __init__.py
│   ├── index.py               # ~/.gc/cache/repos/index.toml read/write
│   ├── store.py               # ~/.gc/cache/repos/<hash>/ management
│   ├── config.py              # ~/.gc/registry-config.toml read/write
│   ├── protocol.py            # internet registry HTTP/TOML client
│   └── ui.py                  # consistent output formatting
└── tests/
    └── ...                    # integration tests
```

## License

Same as Gas City.

## See also

- [`gc-import`](https://github.com/donbox/gc-import) — the package manager that consumes packs by URL, shipped as a separate Gas City pack. `gc registry` is its companion.
- [Gas City](https://github.com/gastownhall/gascity) — the main project.
