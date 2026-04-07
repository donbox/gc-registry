# Registry for Gas City

**GitHub Issue:** TBD

Title: `feat: gc registry — local pack registry with discovery via internet registries`

Companion to [doc-packman.md](doc-packman.md), which defines `gc import` (the package manager that consumes packs by URL). This doc defines `gc registry`: a local store for pack content + a discovery surface for finding packs that aren't already pinned in your imports.

> **Keeping in sync:** This file is the source of truth. Same convention as doc-packman.md.

---BEGIN ISSUE---

## Problem

`gc import` (per [doc-packman.md](doc-packman.md)) takes a git URL and produces a working pack in a city. That answers the "I know what I want" case. It does **not** answer:

1. **Discovery** — "what packs exist that I might want?" Today the answer is "ask around in Slack." There is no enumerated catalog of packs and no way to search by purpose, tag, or description.
2. **Curation** — "I want a vetted standard library of packs (gastown, maintenance, dog, polecat, …) that's available without each user knowing or copy-pasting URLs."
3. **Offline / airgap / CI** — "I need a known set of packs available locally without hitting the network on every build, and without each city duplicating the storage."
4. **Inspection** — "what's actually in this pack? what versions does it publish? when was it last updated? what does it depend on?" Without cloning into a city.

The mechanics that solve these are different from `gc import`'s mechanics. `gc import` is about consumption inside a city; the registry is about the content store and the discovery surface that sits next to it. They are **two commands and one artifact pool**.

## Scope

This doc defines:

- A **local registry** living under `~/.gc/registry/` — a per-machine store of pack content (git clones, version-aware), shared across all cities on the machine. This is the user's universe of "packs I have on disk."
- A **`gc registry` command surface** for managing the local registry: list, add, remove, info, search.
- The relationship between the local registry and **internet registries** — discovery surfaces (websites, search endpoints, curated index files) that the user can query to *find* packs and pull them into the local registry.
- The relationship between the local registry and **`gc import`** — specifically, whether `gc import add gastown` (bare name) resolves through the local registry or whether `gc import` continues to be URL-only and the registry sits cleanly on the side.

This doc explicitly does **not** cover:

- Federation across multiple local registries on the same machine. Postponed; revisit when there's real demand.
- Publishing flows (`gc registry publish`). Out of scope for v1; the v1 registry is consumer-side.
- Authentication, signing, or trust models for internet registries. Out of scope for v1.
- Replacing `gc import`'s URL-based identity model. Identity is still "URL+commit"; the registry is a layer on top, not a replacement.

## Two non-negotiable principles

These are settled and not up for debate in this doc.

**Internet registries never participate in resolve-time loading.** They are pure discovery. `gc registry search`, `gc registry info`, and (eventually) `gc registry add --from <internet-registry>` may hit the network. `gc import install` and the gascity loader never do — they consume the local registry (and `pack.lock`) only. If every internet registry on the planet went down tomorrow, `gc import install` would still work for any city whose `pack.lock` is committed.

**Federation is out of scope for v1.** Single local registry, period. We may add upstream pulls or registry chains in a later version once we've seen what real users actually need. Any design feature that exists *only* to support federation should be deferred.

## Settled decisions

The session that produced this doc closed several questions. Recording them here so we don't relitigate.

| Question | Decision | Why |
|---|---|---|
| Internet registries at resolve time? | **No.** Discovery only. | Builds must never depend on a remote service being up. |
| Federation in v1? | **No.** | Premature complexity; revisit when there's demand. |
| City cache (`.gc/cache/packs/`) goes away? | **No.** It stays. | The loader needs a materialized view per city. The local registry stores clones; the city cache stores materialized packs. They're different shapes for different consumers. |
| Local registry exposes the existing hidden accelerator? | **Yes, in place.** No directory rename in v1. | The thing currently at `~/.gc/cache/repos/<hash>/` stays put for v1 and is presented to users as "the local registry" via the new `gc registry` commands. Renaming the directory to `~/.gc/registry/` is tied to the Pack/City v.next refactor; we don't make stability promises about the shape of `~/.gc`, so the rename can happen later without ceremony. For v1, the verb is `gc registry list` but the bytes still live under `~/.gc/cache/repos/`. We add a sibling `index.toml` in that directory rather than creating a new directory tree. |
| `gc import` keeps URLs as identity? | **Yes.** | URL-as-identity is what makes the resolver simple. The registry is a layer for discovery and storage, not a name resolver. |
| Wire format for internet registry protocol? | **TOML.** | Self-consistency with the rest of Gas City — same syntax our local files use, same parser, same mental model. Costs us `jq` ergonomics; gains us "one format end-to-end." Use the registered media type `application/toml`. |
| Conflict policy for two URLs claiming the same `[pack].name`? | **`gc registry add --name <alias>`** to disambiguate. Refuse the second add otherwise. | Same shape as `gc rig add` and `gc city register` — when a default name collides, the user supplies one explicitly. |
| Implicit standard library? | **Deferred to v.next, design owned by the registry.** See "Explicit design decision: no implicit imports in v1" below. | Implicit imports re-introduce magic that's hard to debug; v1 ships explicit-only. |



## Architecture: where things live

```
~/.gc/
├── cache/
│   └── repos/                      # the local registry's pack store (the existing
│       │                             "hidden accelerator", now exposed via gc registry)
│       ├── index.toml              # name → URL → metadata index (built locally)
│       │                             — this is what gc registry list reads
│       └── <sha256(url+commit)>/   # one git clone per (URL, commit) pair
│           ├── .git/
│           └── ... pack contents ...
├── cities.toml                     # existing — gc register's list of registered cities
├── taps.toml                       # existing — legacy v1 gc pack tap registry (separate concept)
└── registry-config.toml            # NEW — internet registry configuration (which ones to query)

<city>/
├── city.toml                       # gascity config + [imports] (v1) + machine-managed [packs]/includes (v1)
├── pack.toml                       # gc import — direct imports (v2)
├── pack.lock                       # gc import — resolved transitive closure
├── packs/                          # hand-authored sub-packs (and frozen imports — experimental)
└── .gc/
    └── cache/
        └── packs/                  # materialized packs the loader reads
            ├── gastown/
            ├── polecat/
            └── maintenance/
```

> **Note on the path.** The registry's pack store lives at `~/.gc/cache/repos/` for v1, **not** `~/.gc/registry/store/`. We don't make stability promises about the shape of `~/.gc`, so the directory rename is tied to the Pack/City v.next refactor and isn't worth doing in v1. The user-facing word is "the local registry" (verbs are `gc registry list`, `gc registry add`, etc.); the bytes still live under `~/.gc/cache/repos/`. When v.next happens, the directory moves and `gc registry` keeps working without ceremony.

The local registry has two layers:

- **`~/.gc/cache/repos/<hash>/`** — the existing accelerator. Git clones keyed by `sha256(url + commit)`. This is the pack content. Populated as a side effect of `gc import add` / `upgrade` / `install`, and (new in this design) by `gc registry add`.
- **`~/.gc/cache/repos/index.toml`** — a small TOML file the registry manages itself, mapping `name → URL → version-list → commit + hash + metadata`. This is what makes `gc registry list`, `gc registry info`, and `gc registry search --local` possible without crawling every clone in `cache/repos/`.

The index is **derived state**: it's rebuilt from the contents of `cache/repos/` plus any explicit metadata from `gc registry add` calls. Wiping `index.toml` and running `gc registry rebuild` reconstructs it. There is no existing TOML index in `~/.gc/cache/repos/` today — the accelerator is currently a flat set of clone directories with no metadata, so `gc import` v0.1 (the implementation that already exists) has no opinion about an index. This design adds one.

## The two-cache question

You'll notice the architecture has **two stores**: `~/.gc/registry/store/` (clones) and `<city>/.gc/cache/packs/` (materialized). This is intentional, not an oversight. Each has a different shape and a different consumer:

| Store | Keyed by | Format | Consumer |
|---|---|---|---|
| `~/.gc/cache/repos/<hash>/` | `sha256(url+commit)` | Full git clone (with `.git/`) | `gc import` resolver, `gc registry` commands |
| `<city>/.gc/cache/packs/<handle>/` | Local handle (`gastown`, `polecat_v2`) | Materialized directory tree (no `.git/`) | The gascity loader at startup |

**The materialization is a copy, not a symlink.** The city pack cache holds a real copy of the resolved pack at the locked version, with `.git/` stripped. Two reasons we copy instead of symlinking into the registry:

1. **Cities stay self-contained on disk.** A user can `tar -czf my-city.tar.gz my-city/` and the result has everything needed to load that city (modulo the gitignored `.gc/cache/`, which is regenerable from `pack.lock`). Symlinks would break tarballs and container builds.
2. **Hash verification has a stable target.** The content hash in `pack.lock` is computed against the materialized tree, not the clone (which has `.git/` and per-machine git metadata). Symlinks would force on-the-fly hashing of the registry clone, which is slower and harder to reason about.

The cost is disk: every city that imports gastown holds its own copy. For typical pack sizes (a few MB) that's fine; if it ever stops being fine, the right fix is hardlinks (still on the same filesystem, still self-contained for tarball purposes if the user copies the directory tree) rather than symlinks.

Could the two stores be collapsed into one? Two options on the table, one settled and one logged for v.next:

- **Settled (v1): Shape A — two stores, registry cache + city cache.** The current model. Registry holds clones; city cache holds materialized views. `gc import install` materializes from registry → city cache (copy, not symlink). Two layers because two consumers want two shapes.
- **Logged for v.next: Shape C — city cache reads from registry directly, no per-city materialization.** The city's loader reads from `~/.gc/cache/repos/<hash>/<subpath>/` (with subpath stripping for monorepo packs). No `.gc/cache/packs/`. Saves disk, removes a layer, but requires the gascity loader to learn about the registry. Also forces a question about how the loader handles the `.git/` directory inside the clone, breaks the "city is self-contained on disk" property, and complicates hash verification. We'll revisit Shape C alongside the Pack/City v.next refactor — the loader changes for one and the loader changes for the other are likely to land together. **Not blocking v1; not an open issue in this doc.**
- **Rejected — Shape B: city does not have any local store and reads everything from a single per-machine pool.** Was on the table; rejected for being neither a clean A nor a clean C, and for making cities non-portable.

## `gc registry` commands (v1 surface)

Six verbs, mirroring `gc import` in spirit (though they don't have to be the same six). Phase-1 surface:

| Command | What it does | Network? |
|---|---|---|
| `gc registry list` | Show every (name, version) currently in the local registry | No |
| `gc registry info <name>` | Show metadata for a name: URL, available versions, description, dependencies, last update | No (local) / Yes (with `--remote`) |
| `gc registry add <url> [--version <constraint>] [--name <alias>]` | Fetch a URL into the local registry without importing it into any city. Pre-warming. | Yes |
| `gc registry remove <name>[@<version>]` | Evict a name (or a specific version) from the local registry | No |
| `gc registry search <query>` | Search descriptions/tags across configured internet registries. `--local` restricts to what's already cached. | Yes by default |
| `gc registry rebuild` | Rebuild `index.toml` from the contents of `~/.gc/cache/repos/`. Repair operation. | No |

### How `gc registry search` actually works

`gc registry search <query>` queries the **internet registries configured in `~/.gc/registry-config.toml`**. Each configured registry is a base URL pointing at a server (or a static-hosted directory) that serves the [internet registry protocol](#internet-registry-protocol) — a small set of TOML files at well-known paths. The client downloads `<base>/index.toml` from each configured registry, unions the results, and prints matches.

```toml
# ~/.gc/registry-config.toml — managed by gc registry config or hand-edited
[[registries]]
name = "gascity-stdlib"
url = "https://raw.githubusercontent.com/gascity/registry-stdlib/main/"

[[registries]]
name = "myteam-internal"
url = "https://internal.example.com/gc-registry/"
```

A fresh install ships with **one default registry** pre-configured (the gascity-stdlib registry) so `gc registry search` works out of the box without any setup. Users can add more with `gc registry config add <name> <url>` or remove the default with `gc registry config remove gascity-stdlib`. There is no precedence chain among multiple configured registries — results are unioned and the user disambiguates with `--from <name>` if needed. (That's not federation; it's parallel discovery.)

> **Open question (intentionally deferred):** what's actually in the gascity-stdlib registry on day one, and where does it live? Pack, Registry, and some combination of gastown / maintenance / dog are the obvious candidates. The decision lives in the registry-stdlib project, not in this design doc — file an issue against gastownhall/registry-stdlib (or wherever the stdlib repo ends up) when v1 is closer to shipping.

### `gc registry list`

```
$ gc registry list
NAME         VERSIONS              SOURCE
gastown      1.4.0, 1.5.0          https://github.com/example/gastown
polecat      0.4.1                 https://github.com/example/polecat
maintenance  2.0.1                 https://github.com/example/maintenance
```

Reads `index.toml`, prints one row per name with the versions currently available locally. The "SOURCE" column shows the canonical URL the registry recorded for this name.

### `gc registry info <name>`

```
$ gc registry info gastown
gastown
  URL:          https://github.com/example/gastown
  Local versions: 1.4.0, 1.5.0
  Latest known: 1.5.0
  Description:  Multi-agent orchestration pack with mayor and polecat agents.
  Dependencies (1.5.0):
    polecat ^0.4
  Last fetched: 2026-04-06 23:42 PDT

$ gc registry info gastown --remote
gastown
  URL:          https://github.com/example/gastown
  Local versions: 1.4.0, 1.5.0
  Available remotely: 1.0.0, 1.1.0, 1.2.0, 1.3.0, 1.4.0, 1.5.0, 1.6.0
  Latest:       1.6.0
  ...
```

Local mode reads the registry index and the cached `pack.toml` files. Remote mode also queries `git ls-remote --tags` for the recorded URL to find versions newer than what's cached locally. Internet registries are not consulted by default; use `--from <registry>` to query a specific one.

### `gc registry add <url> [--version <constraint>] [--name <alias>]`

The "pre-warm the registry without touching any city" command. Fetches a URL into `~/.gc/cache/repos/`, parses the resolved pack's `pack.toml` for name + version, and updates `index.toml`.

**`--version` works exactly like `gc import add --version`.** Same constraint syntax (`^1.2`, `~1.2.3`, `>=1.0,<2.0`, exact `1.2.3`). If `--version` is omitted, the default is `^<major>.<minor>` of the highest available tag — same default as `gc import add`. The point of the symmetry: a user who learns one verb has learned both.

**`--name <alias>` resolves a name collision** when two URLs claim the same `[pack].name` (e.g., two unrelated `gastown` packs from different repos). On the second `gc registry add`, the registry detects the collision and refuses with a clear error pointing at `--name`. The user re-runs with `--name my-gastown` (or whatever) and the second pack is indexed under that alias instead. This is the same shape as `gc rig add` and `gc city register` — when a default name collides, the user supplies one explicitly.

```
$ gc registry add https://github.com/example/gastown
Fetching https://github.com/example/gastown...
  Selected: 1.6.0 (latest, default constraint ^1.6)
  Cloned → ~/.gc/cache/repos/<hash>/
  Indexed as gastown 1.6.0

$ gc registry add https://github.com/example/gastown --version "^1.4"
Fetching https://github.com/example/gastown...
  Constraint: ^1.4
  Selected: 1.5.0
  ...

$ gc registry add https://github.com/other-org/gastown
error: name collision: 'gastown' is already registered from
  https://github.com/example/gastown
  Use --name <alias> to register this one under a different name, e.g.:
    gc registry add https://github.com/other-org/gastown --name other-gastown

$ gc registry add https://github.com/other-org/gastown --name other-gastown
Fetching https://github.com/other-org/gastown...
  Selected: 0.3.0 (latest, default constraint ^0.3)
  Cloned → ~/.gc/cache/repos/<hash>/
  Indexed as other-gastown 0.3.0  (aliased from upstream name 'gastown')
```

Useful for:
- Pre-warming a CI environment so subsequent `gc import install` runs are fully offline.
- Building a curated set of "approved" packs on a shared developer machine.
- Browsing a pack without committing to importing it into any city.
- Pulling something you found via internet-registry search into local availability.

`gc registry add` is **not** required for `gc import add` to work. `gc import add` populates the registry as a side effect. `gc registry add` is the explicit, no-side-effects-on-any-city version.

### How `gc import install` and `gc import upgrade` interact with the registry

Both commands **read through the registry as their store layer and populate it as a side effect**. This is the same behavior the existing `gc import` v0.1 already has — the rename from "hidden accelerator" to "local registry" doesn't change anything about how the bytes flow:

- `gc import install` reads `pack.lock`, looks up each `(URL, commit)` pair in the registry (`~/.gc/cache/repos/<sha256(url+commit)>/`), and **clones-on-miss**. The clone-on-miss writes a new entry to `index.toml`. So `install` IS a read-through cache update, and a `gc registry list` immediately after a fresh `install` will show every pack the install just pulled in.
- `gc import upgrade` is the same flow with new commits chosen by re-resolving constraints. New commits → new hash directories in `cache/repos/` → new `index.toml` entries.
- Neither command requires `gc registry add` first. The registry is populated lazily by import operations; explicit `gc registry add` is only useful for pre-warming (CI / offline / curation).

**The index entry's name** comes from the resolved pack's `[pack].name` (read from `pack.toml` inside the clone), **not** from the local handle the importing city uses. Two cities can call the same pack `gastown` and `gtwn` respectively in their local `[imports]` blocks; both refer to the same registry entry whose name is `gastown` (the upstream's self-declared name). This is consistent: the registry is shared across cities; using a local handle as the index key would mean each city sees its own private registry view, which is not what we want.

When a name collision happens during a side-effect populate (two cities import packs from different URLs that both name themselves `gastown`), the registry **errors the second import** with the same `--name` hint as `gc registry add`. The user's recovery path: rerun the import with an explicit alias in the importing city's `[imports]` block. (Yes, this means an import can fail with a registry-level error; this is the right tradeoff because silent renames in user-global state would be much more surprising.)

### `gc registry remove <name>[@<version>]`

```
$ gc registry remove gastown@1.4.0
Removing gastown 1.4.0 from the local registry
  Deleted ~/.gc/cache/repos/<hash>/

$ gc registry remove gastown
Removing all versions of gastown
  - 1.5.0
  - 1.6.0
Refusing: 1.5.0 is referenced by lock file in /Users/dbox/cities/my-city
  Use --force to override.
```

Eviction errors if any city's `pack.lock` on the same machine references the version being removed. Override with `--force`. The mechanism for "find cities that reference this" is itself an open question — see open questions below.

### `gc registry search <query>`

Discovery surface. By default queries every internet registry configured in `~/.gc/registry-config.toml` and prints results. `--local` restricts to what's already in the local pack store. `--from <name>` restricts to a single configured registry.

```
$ gc registry search auth
gastownhall/dolt-auth   0.3.1   Authentication agents for Dolt-backed cities
example/oauth-broker    1.2.0   OAuth2 broker pack with token caching

$ gc registry search auth --local
(no local matches)

$ gc registry search auth --from gascity-stdlib
gastownhall/dolt-auth   0.3.1   Authentication agents for Dolt-backed cities
```

Multiple configured registries get queried in parallel; results are unioned. There is no precedence or fallback chain — that's federation, which we're not building. See [How `gc registry search` actually works](#how-gc-registry-search-actually-works) above for the configuration details.

### `gc registry rebuild`

Repair operation. Walks `~/.gc/cache/repos/`, parses each clone's `pack.toml`, and rewrites `index.toml`. Useful if the index gets corrupted or out of sync, or after a manual `rm -rf` of an entry.

## Internet registry protocol

An internet registry is **any HTTP endpoint that speaks a small TOML protocol.** No central server, no required authentication, no upload pipeline. A registry can be as simple as a static TOML file served from S3 or a GitHub Pages site.

**Why TOML and not JSON?** Self-consistency. Every other file in Gas City is TOML — `city.toml`, `pack.toml`, `pack.lock`, `imports.toml` (when v1 had it), `~/.gc/cache/repos/index.toml`. Using JSON for the wire format would introduce a second format with no benefit other than `jq` ergonomics. Same parser, same syntax, end to end. We use the registered media type `application/toml` (RFC 9618) for the responses; clients can also accept `text/x-toml` from servers that haven't been updated.

The protocol has three endpoints:

- **`GET /index.toml`** — lists every pack in the registry with name, latest version, description, tags. Used by `gc registry search` to populate results.
- **`GET /packs/<name>.toml`** — full metadata for a name: URL, all versions, descriptions per version, dependencies. Used by `gc registry info --remote --from <registry>`.
- **`GET /search?q=<query>`** *(optional)* — server-side search; returns the same shape as `index.toml` but filtered. If absent, the client downloads `/index.toml` and filters locally.

That's it. Three endpoints, all read-only, all serving static-ish TOML. A registry is "the URL of an HTTP server that serves these endpoints." Static hosting works. CDNs work. A git repo with a TOML file at HEAD and `https://raw.githubusercontent.com/...` works.

```toml
# Example: index.toml served from a static-hosted internet registry

schema = 1
name = "gascity-stdlib"
description = "Curated standard packs for Gas City"

[[packs]]
name = "gastown"
url = "https://github.com/example/gastown"
latest = "1.6.0"
description = "Multi-agent orchestration pack with mayor and polecat agents"
tags = ["orchestration", "stdlib"]

[[packs]]
name = "maintenance"
url = "https://github.com/example/maintenance"
latest = "2.0.1"
description = "Infrastructure maintenance agents"
tags = ["maintenance", "stdlib"]

[[packs]]
name = "dog"
url = "https://github.com/example/dog"
latest = "0.7.2"
description = "Watchdog and supervision agents"
tags = ["maintenance", "stdlib"]
```

Critically: **the TOML the registry returns contains URLs**. When `gc registry search` finds gastown, the result is "gastown is at `https://github.com/example/gastown`". The user copies that URL into `gc import add` (no bare-name resolution in v1). The registry never serves pack content. It serves *pointers* to pack content.

This means:
- Registries are cheap to operate (static TOML).
- Registries are easy to fork (clone the TOML file, change one line).
- Registries are impossible to "lose your packs" — the canonical content lives in git repos, not the registry.
- Registries don't need versioning, releases, or pipelines. They're descriptions, not packages.

A v1 internet registry will ship as a curated TOML file in a git repo: `https://github.com/gascity/registry-stdlib/blob/main/index.toml`. Pre-configured in `~/.gc/registry-config.toml` on first run via `https://raw.githubusercontent.com/gascity/registry-stdlib/main/`. Future iterations could add a real search service, but the v1 design works without one.

## Relationship to `gc import`

The package manager and the registry are designed in tandem; this section spells out exactly how the two interact.

### `gc import add gastown` (bare name) — deferred to v1.5

In the URL-only world we built for `gc import` v0.1, the answer was "no — `add` takes a URL or a path, never a bare name." Names had nowhere to be resolved.

With a local registry, names *do* have somewhere to be resolved. `gc import add gastown` could look up `gastown` in `~/.gc/cache/repos/index.toml`, find the URL, and pass it to the existing `gc import add <url>` flow.

**v1 of `gc import` does NOT support bare-name resolution.** The package manager stays URL-and-path only. Bare-name resolution is **explicitly deferred to phase 1.5**, after both `gc import` v1 and `gc registry` v1 ship. Reasons:

1. **Avoids adding magic before users have lived with the registry.** A bare-name `add` quietly depends on registry state that's invisible from the importing city — that coupling is fine if the user understands the registry, but jarring if they don't.
2. **Keeps the v1 surface honest.** "URL or path" is a one-line rule. "URL, path, or a name that the registry might know about" is a longer rule with new failure modes.
3. **Implicit fallback to internet registries was considered and rejected** — see Alternatives considered.

In v1.5, when bare-name resolution lands, it will be **explicit-only**: `gc import add gastown` errors if `gastown` isn't already in the local registry. The user's recovery is `gc registry add gastown` first (or use the URL form). No silent hits to the network during `add`.

### Implicit imports / standard library — separate phase, registry-owned

There's a related idea: **let every city implicitly import a small standard library of packs** — probably some combination of Pack, Registry itself, and a curated set of agent packs (gastown, maintenance, dog) — unless the city opts out via `nostdlib = true` in `pack.toml`. This is genuinely powerful (extends the product surface area without touching every city) but it has consequences:

1. **The stdlib needs to live somewhere** — almost certainly as a manifest in the default internet registry, applied automatically by the resolver.
2. **It re-introduces magic** that's hard to debug ("why is `polecat.scout` available in my city? I never imported it." "It's in the stdlib.").
3. **It couples the package manager to the registry in a load-bearing way** — every city's dependency closure includes registry-managed implicit imports.
4. **`nostdlib = true` is the escape hatch**, but it has to be honored consistently across transitive imports.

**Implicit stdlib is deferred to a separate phase** that pairs registry work with package-manager work. It's not part of v1 of either project. It's filed here as the right home for the discussion; the registry doc owns the design when we're ready to do it. **Open call-out for Donna, Chris, and Julian:** the existence of an implicit stdlib (and which packs go in it) is a product-shape decision that needs all three of you. Filing a v.next issue is the right next step.

### `gc import install` and `gc import upgrade` — read-through with side-effect populate

Both commands **read through the registry as their store layer and populate it as a side effect.** This is identical to the behavior `gc import` v0.1 already has — the rename from "hidden accelerator" to "local registry" doesn't change anything about how the bytes flow:

- `gc import install` reads `pack.lock`, looks up each `(URL, commit)` pair in the registry, clones-on-miss, materializes into the city cache, verifies the hash. The clone-on-miss writes a new entry to `index.toml`, so a `gc registry list` immediately after a fresh `install` shows every pack the install just pulled in.
- `gc import upgrade` is the same flow with new commits chosen by re-resolving constraints.
- Neither command requires `gc registry add` first.

**The index entry's name comes from the pack's own `[pack].name`** (read from `pack.toml` inside the clone), **not** from the local handle the importing city uses. Two cities can call the same pack `gastown` and `gtwn` respectively in their local `[imports]` blocks; both refer to the same registry entry whose name is `gastown` (the upstream's self-declared name). This is consistent: the registry is shared across cities; using a local handle as the index key would mean each city sees its own private registry view, which is not what we want.

When a name collision happens during a side-effect populate (two cities import packs from different URLs that both name themselves `gastown`), the registry **errors the second import** with a `--name <alias>` hint, same shape as `gc registry add`. The user's recovery: rerun the import with an explicit alias in the importing city's `[imports]` block. This is the right tradeoff because silent renames in user-global state would be much more surprising than a clear collision error.

## v1 surface, summarized

- **`gc registry list`** — show what's in the local registry
- **`gc registry info <name>`** — metadata for one pack (`--remote` to also query the URL for fresh tags)
- **`gc registry add <url> [--version <c>] [--name <alias>]`** — pre-warm; fetch a URL into the local registry without touching any city
- **`gc registry remove <name>[@<version>]`** — evict
- **`gc registry search <query>`** — discovery via configured internet registries (`--local` for offline, `--from <name>` for one specific registry)
- **`gc registry rebuild`** — repair `index.toml`
- **Configuration** for internet registries lives in `~/.gc/registry-config.toml`. v1 supports hand-editing this file directly. A `gc registry config add <name> <url>` sub-verb is on the v1.5 list if hand-editing turns out to be a friction point.

The local registry shape: `~/.gc/cache/repos/<hash>/` (clones, existing path) + `~/.gc/cache/repos/index.toml` (derived metadata, new) + `~/.gc/registry-config.toml` (user config for internet registries, new).

## Phasing

**Phase 1 (this proposal):** Six verbs above, against Shape A architecture (city cache stays). The pack store stays at `~/.gc/cache/repos/` (existing path); a sibling `index.toml` gets added there; `~/.gc/registry-config.toml` is new. `gc registry` commands get implemented as a Gas City pack (`gc-registry`, pure Python, no Go changes). `gc import` v1 keeps its current URL-and-path-only `add` semantics — bare-name resolution is **not** in v1. Internet registry support ships with one default-configured registry (`gascity-stdlib`) on first run.

**Phase 1.5:** Bare-name resolution in `gc import add` (explicit-only mode). `gc import add gastown` works iff `gastown` is in the local registry. Adds a small lookup step before the existing URL flow. Optionally adds `gc registry config add/remove/list` sub-verbs if hand-editing `~/.gc/registry-config.toml` is friction.

**Phase 2:** Implicit standard library — `nostdlib = true` in `pack.toml` opts out; otherwise the resolver injects a curated set of packs. Pairs with the registry-stdlib content design. **Owned by Donna, Chris, and Julian as a product-shape decision**, not by either tool's implementer.

**Phase 3:** Curated internet registry tooling (`gc registry publish`, signed indices, server-side search), if there's demand.

**Later (v.next, tied to Pack/City refactor):**
- **Shape C: collapse the city cache into the registry.** The loader reads from `~/.gc/cache/repos/<hash>/<subpath>/` directly, no `<city>/.gc/cache/packs/`. Saves disk and removes a layer, but requires loader changes that pair naturally with the v.next loader work. Logged as a v.next change rather than as an open question — the v1 model (Shape A) is fine on its own, and the loader work makes more sense to do all at once.
- **Directory rename: `~/.gc/cache/repos/` → `~/.gc/registry/`.** Cosmetic; we don't make stability promises about the shape of `~/.gc`. Ties to the same v.next work.
- Federation across multiple local registries, if there's demand. Authentication for private internet registries, if there's demand.

## Open questions (still need answers before v1 ships)

These are the questions the design hasn't closed. Each one needs a decision before v1 of `gc registry` ships.

1. **What's actually in the gascity-stdlib internet registry on day one?** Pack, Registry, and some combination of gastown/maintenance/dog are the obvious candidates. Decision lives in the registry-stdlib project, not here. **File an issue against the gascity-stdlib repo when v1 is closer to shipping.**
2. **Reverse-reference tracking for `gc registry remove`.** To know whether a registry version is "in use," we need to find every city on the machine that has that (URL, commit) in its `pack.lock`. Two implementations:
   - Walk every directory the user has registered as a city via `gc register` (we already track these in `~/.gc/cities.toml`) at remove time. Reasonably accurate, slow only if there are many cities, no new state to maintain.
   - Keep a back-reference index in the registry: `<city path>` for every `(name, version)` it locks. Requires `gc import` to update the registry on `install`/`upgrade`/`remove`. Faster but couples the two more tightly.
   Lean: **walk `~/.gc/cities.toml` for v1.** It's already maintained by `gc register`. Cities not registered there will be missed; the warning becomes "best effort, may miss unregistered cities." `--force` is the override.
3. **Network behavior of `gc registry info <name>`.** Default to local-only (no network) and require `--remote` for the network query? Or default to network for the freshest data and add `--local` for offline? **Lean: default local; freshness is opt-in.** v1 ships this way; revisit if users complain about staleness.

## Explicit design decisions worth calling out

These aren't open questions — they're settled, but they're decisions worth being able to point at when someone asks "why."

1. **City cache is a copy, not a symlink.** See the [two-cache question section](#the-two-cache-question) above. Cities stay self-contained on disk; hash verification has a stable target. Costs disk; gains portability and reproducibility.
2. **Wire format is TOML, not JSON.** Self-consistency with the rest of Gas City. We use `application/toml` (RFC 9618) as the media type. Costs `jq` ergonomics; gains "one format end-to-end."
3. **Bare-name `gc import add gastown` is deferred to v1.5, not v1.** Avoids adding magic before users have lived with the registry. v1 stays URL-and-path-only.
4. **Implicit imports (stdlib) are deferred to phase 2.** Same reason, plus the question of *what's in the stdlib* is a product decision that needs its own conversation.
5. **The pack store stays at `~/.gc/cache/repos/` for v1.** No directory rename. The user-facing word becomes "the local registry" but the bytes are at the existing path. The rename is tied to Pack/City v.next.

## Alternatives considered (and rejected)

- **Local registry as a DB instead of TOML + filesystem.** Faster for large catalogs, but adds a binary format dependency and breaks "it's just files." For a registry of dozens or hundreds of packs, TOML + filesystem is fine.
- **Internet registry as a single canonical service (like npm).** Centralization is exactly what URL-as-identity was meant to avoid. Multiple parallel registries (each a TOML file at a URL) is the right shape.
- **JSON for the wire format.** Conventional for web APIs, but introduces a second format alongside the TOML we use everywhere else. Self-consistency wins.
- **Implicit fallback for bare-name `gc import add`.** If `gastown` isn't in the local registry, query configured internet registries to resolve the name to a URL, then proceed. Quietly couples builds to internet-registry availability for the *initial* `add`, and creates a new failure mode that's hard to debug (`why did this command hit the network?`). Rejected in favor of explicit-only resolution in v1.5.
- **Treating cities as registries.** Briefly considered — what if a city *is* a registry, and importing a pack just means "add a reference to the registry's view of it"? Rejected because cities are consumers, not catalogs. The registry concept is meant to *be* a catalog; conflating the two is the same error as taps were.
- **Skipping the local-registry-as-a-thing entirely and just adding `gc registry search` as a thin discovery wrapper over internet registries.** This is the absolute minimum design — just discovery, no local store, no `gc registry add/remove`. It's defensible but it doesn't answer the airgap/CI/curation use cases, and it doesn't give us a clean answer to "where does the existing accelerator live now?"
- **Auto-suffixing on name collision.** Two URLs both publish `gastown` → second one becomes `gastown-2` automatically. Magical, ugly, debugging nightmare. Refuse + `--name <alias>` is cleaner.

---END ISSUE---
