# Registry for Gas City

**GitHub Issue:** TBD

Title: `feat: gc registry — local pack registry with discovery via internet registries`

Companion to [gastownhall/gascity#434](https://github.com/gastownhall/gascity/issues/434) (`gc import`), which defines the package manager that consumes packs by URL. This doc defines `gc registry`: a local store for pack content + a discovery surface for finding packs that aren't already pinned in your imports.

> **Keeping in sync:** This file is the source of truth. Same convention as doc-packman.md.

---BEGIN ISSUE---

> **This proposal is part of a three-part design** that seeks to:
>
> - **Provide clean state separation for cities and packs** (a.k.a., have `city.toml` not do three different jobs) — [gastownhall/gascity#360](https://github.com/gastownhall/gascity/issues/360) (Pack/City v.next).
> - **Provide a more robust model for importing, identifying, and versioning packages** — [gastownhall/gascity#360](https://github.com/gastownhall/gascity/issues/360) (defines the `[imports]` schema and the city-as-pack refactor) plus [gastownhall/gascity#434](https://github.com/gastownhall/gascity/issues/434) (`gc import`, which implements semver constraints, transitive resolution, and the lock file on top of that schema).
> - **Provide an initial package management mechanism for both on-machine and internet registries** — [gastownhall/gascity#434](https://github.com/gastownhall/gascity/issues/434) (`gc import`, already filed) and **this issue** (`gc registry`).
>
> The three issues are designed to be implementable in sequence. `gc import` (#434) ships first against the v1 schema. `gc registry` (this issue) ships next, paired with `gc import` and exposing the local pack store and discovery surface. Pack/City v.next (#360) lands when the gascity loader gains the small set of capabilities the new schema requires. None of the three blocks the others — but they make more sense read together than in isolation.

## Problem

`gc import` (per [gastownhall/gascity#434](https://github.com/gastownhall/gascity/issues/434)) takes a git URL and produces a working pack in a city. That answers the "I know what I want" case. It does **not** answer:

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

## Design and scoping decisions

The session that produced this doc closed several questions. Recording them here so we don't relitigate.

| Question | Decision | Why |
|---|---|---|
| Internet registries at resolve time? | **No.** Discovery only. | Builds must never depend on a remote service being up. |
| Federation in v1? | **No.** | Premature complexity; revisit when there's demand. |
| City cache (`.gc/cache/packs/`) goes away? | **No.** It stays. | The loader needs a materialized view per city. The local registry stores clones; the city cache stores materialized packs. They're different shapes for different consumers. |
| Local registry exposes the existing hidden accelerator? | **Yes, in place.** No directory rename, no new files in `~/.gc/cache/repos/`. | The thing currently at `~/.gc/cache/repos/<hash>/` stays put for v1 and is presented to users as "the local registry" via the new `gc registry` commands. Renaming the directory to `~/.gc/registry/` is tied to the Pack/City v.next refactor; we don't make stability promises about the shape of `~/.gc`, so the rename can happen later without ceremony. For v1, the verb is `gc registry list` but the bytes still live under `~/.gc/cache/repos/`. **The cache hierarchy is unchanged from the existing accelerator** — no sibling index file, no metadata, no schema. `gc registry list` walks the clones on demand and reads each one's `pack.toml`. |
| `gc import` keeps URLs as identity? | **Yes.** | URL-as-identity is what makes the resolver simple. The registry is a layer for discovery and storage, not a name resolver. |
| Wire format for internet registry protocol? | **TOML.** | Self-consistency with the rest of Gas City — same syntax our local files use, same parser, same mental model. Costs us `jq` ergonomics; gains us "one format end-to-end." Use the registered media type `application/toml`. |
| Conflict policy for two URLs claiming the same `[pack].name`? | **`gc registry add --name <alias>`** to disambiguate. Refuse the second add otherwise. | Same shape as `gc rig add` and `gc city register` — when a default name collides, the user supplies one explicitly. |
| Implicit imports list — registry-driven or hardcoded? | **Hardcoded in `gc-import` for v1.** A future "stdlib" feature could let the registry publish the implicit list, but v1 keeps it as a literal in the package manager source. See "How the implicit-imports feature intersects with the registry" below. | The implicit list is currently exactly one entry (`maintenance`); not worth a registry round-trip. |
| Where do users go to find packs? | **One canonical PR-managed registry**, hosted as a git repo. Updates are pull requests against that repo. | Curation, operational simplicity, and "one place to find packs." See "The canonical registry and adding packs to it" below. |



## Architecture: where things live

```
~/.gc/
├── cache/
│   └── repos/                      # the local registry's pack store (the existing
│       │                             "hidden accelerator", now exposed via gc registry)
│       │                             — this is what gc registry list reads
│       └── <sha256(url+commit)>/   # one git clone per (URL, commit) pair
│           ├── .git/
│           └── ... pack contents ...
├── cities.toml                     # existing — gc register's list of registered cities
├── taps.toml                       # existing — legacy v1 gc pack tap registry (separate concept)

<city>/
├── city.toml                       # gascity config + [imports] (v1) + machine-managed [packs]/includes (v1)
├── pack.toml                       # gc import — direct imports (v2)
├── pack.lock                       # gc import — resolved transitive closure
# (no ./packs/ — vendoring is cut for v1; if it comes back it'll be reframed as registry pinning)
└── .gc/
    └── cache/
        └── packs/                  # materialized packs the loader reads
            ├── gastown/
            ├── polecat/
            └── maintenance/
```

> **Note on the path.** The registry's pack store lives at `~/.gc/cache/repos/` for v1, **not** `~/.gc/registry/store/`. We don't make stability promises about the shape of `~/.gc`, so the directory rename is tied to the Pack/City v.next refactor and isn't worth doing in v1. The user-facing word is "the local registry" (verbs are `gc registry list`, `gc registry add`, etc.); the bytes still live under `~/.gc/cache/repos/`. When v.next happens, the directory moves and `gc registry` keeps working without ceremony.

The local registry is **the existing accelerator, unchanged**. `~/.gc/cache/repos/<hash>/` holds git clones keyed by `sha256(url + commit)`, populated as a side effect of `gc import add` / `upgrade` / `install` (and, new in this design, by `gc registry add`). The cache is a flat directory of hash-named clone roots. No sibling metadata file, no index, no schema. The cache is opaque; users interact with it via `gc registry` commands rather than by inspecting it.

`gc registry list` and `gc registry info` compute their output **on demand** by walking `~/.gc/cache/repos/`, opening each clone's `pack.toml`, and reading the name and version. At v1 scale (single-digit to low-tens of cached packs per machine), this is unmeasurably fast. If the cache ever grows to a size where the walk becomes slow, we'll add an index file as an internal performance optimization — not as a public design feature, and not in v1.

**Why no index file in v1.** The earlier draft of this design added a sibling `index.toml` next to the clones to make `gc registry list` O(1) instead of O(n). On reflection, that's premature: at v1 scale the walk is fine, the index would introduce a new write path on every `gc import add` / `upgrade` / `remove`, and an "index out of sync with the actual clones" failure mode would need a `gc registry rebuild` repair verb. All of that is overhead we don't need for the surgical-fixes-near-release v1. The cache hierarchy stays exactly as it is today.

## The two-cache question

You'll notice the architecture has **two stores**: `~/.gc/cache/repos/` (clones) and `<city>/.gc/cache/packs/` (materialized). This is intentional, not an oversight. Each has a different shape and a different consumer:

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

Five verbs, mirroring `gc import` in spirit (though they don't have to be the same five). Phase-1 surface:

| Command | What it does | Network? |
|---|---|---|
| `gc registry list` | Show every (name, version) currently in the local registry | No |
| `gc registry info <name>` | Show metadata for a name: URL, available versions, description, dependencies, last update | No (local) / Yes (with `--remote`) |
| `gc registry add <url> [--version <constraint>] [--name <alias>]` | Fetch a URL into the local registry without importing it into any city. Pre-warming. | Yes |
| `gc registry remove <name>[@<version>]` | Evict a name (or a specific version) from the local registry | No |
| `gc registry search <query>` | Search descriptions/tags across the canonical registry. `--local` restricts to what's already cached. | Yes by default |

### The canonical registry and adding packs to it

There is **one canonical Gas City registry**, hosted as a git repository under the `gastownhall` GitHub organization (working name: `gastownhall/gc-registry-canonical`; final name and location TBD). It is the registry every `gc-import` install talks to by default, and it is the registry every user-facing piece of documentation refers to when it says "the registry."

The canonical registry is **a git repo containing a single `registry.toml` file** at its root, plus per-pack metadata files under `packs/`. That's it — no server, no CDN, no auth, no upload pipeline. The repo is served via `https://raw.githubusercontent.com/gastownhall/gc-registry-canonical/main/` for HTTP fetches; `gc registry search` and `gc registry info` read directly from those URLs using the internet registry protocol described later in this doc.

#### How packs get added

**File a PR against the canonical registry repo.** The PR adds (or updates) one entry in `registry.toml` plus an optional `packs/<name>.toml` file with richer metadata. Reviewers merge after a quick check that the URL is reachable, the pack has at least one semver tag, and the description isn't actively misleading. There is no automated scanning, no signing, no provenance verification — the PR review is the curation step, and the merge is the publish event.

This is the same model as homebrew-core, MELPA, and a hundred other community-curated registries: the curation is the review, and review effort scales with submission volume. For an ecosystem the size of Gas City's, that's the right shape.

The canonical registry never serves pack content. It serves *pointers* — URLs into the actual pack repos. Adding a pack to the canonical registry doesn't move bytes anywhere; it just tells the registry "this name maps to this URL." The URL still has to be a git repo that `gc import` can clone, the same as any other URL.

#### How `gc registry search` actually works

`gc registry search <query>` downloads the canonical registry's `registry.toml` over HTTP, filters it locally for the query string, and prints matches. There is no setup: a fresh `gc-import` install knows where the canonical registry lives. Search it, find a pack, copy the URL into `gc import add`. That's the entire story for the common case — every user, every public pack.

> **Open question (intentionally deferred):** what's actually in the canonical registry on day one? Pack, Registry, and some combination of gastown / maintenance / dog are the obvious candidates. The decision lives in the canonical registry repo, not in this design doc — file an issue against `gastownhall/gc-registry-canonical` (or whatever the final name turns out to be) when v1 is closer to shipping.

### `gc registry list`

```
$ gc registry list
NAME         VERSIONS              SOURCE
gastown      1.4.0, 1.5.0          https://github.com/example/gastown
polecat      0.4.1                 https://github.com/example/polecat
maintenance  2.0.1                 https://github.com/example/maintenance
```

Walks `~/.gc/cache/repos/`, opens each clone's `pack.toml` to read its name and version, and prints one row per pack. No index file is consulted; the cache directory is the source of truth. The "SOURCE" column shows the URL the clone was fetched from (encoded in the hash).

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

The "pre-warm the registry without touching any city" command. Fetches a URL into `~/.gc/cache/repos/`, parses the resolved pack's `pack.toml` for name + version, and the next `gc registry list` will pick it up automatically (no index file to update — the cache directory is the source of truth).

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

- `gc import install` reads `pack.lock`, looks up each `(URL, commit)` pair in the registry (`~/.gc/cache/repos/<sha256(url+commit)>/`), and **clones-on-miss**. New clone directories appear in the cache as a side effect, so a `gc registry list` immediately after a fresh `install` will show every pack the install just pulled in (the walk picks them up automatically).
- `gc import upgrade` is the same flow with new commits chosen by re-resolving constraints. New commits → new hash directories in `cache/repos/`.
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

Discovery surface. Queries the canonical Gas City registry and prints results. `--local` restricts to what's already in the local pack store.

```
$ gc registry search auth
gastownhall/dolt-auth   0.3.1   Authentication agents for Dolt-backed cities
example/oauth-broker    1.2.0   OAuth2 broker pack with token caching

$ gc registry search auth --local
(no local matches)

$ gc registry search auth --from canonical
gastownhall/dolt-auth   0.3.1   Authentication agents for Dolt-backed cities
```

There is one canonical registry. Federation across multiple registries is out of scope for v1.

## Internet registry protocol

An internet registry is **any HTTP endpoint that speaks a small TOML protocol.** No central server, no required authentication, no upload pipeline. A registry can be as simple as a static TOML file served from S3 or a GitHub Pages site.

**Why TOML and not JSON?** Self-consistency. Every other file in Gas City is TOML — `city.toml`, `pack.toml`, `pack.lock`, `~/.gc/implicit-import.toml`. Using JSON for the wire format would introduce a second format with no benefit other than `jq` ergonomics. Same parser, same syntax, end to end. We use the registered media type `application/toml` (RFC 9618) for the responses; clients can also accept `text/x-toml` from servers that haven't been updated.

The protocol has three endpoints:

- **`GET /registry.toml`** — lists every pack the registry knows about, with name, URL, latest version, description, tags. Used by `gc registry search` to populate results.
- **`GET /packs/<name>.toml`** — richer metadata for a single pack: all versions, per-version descriptions, dependencies, maintainer info. Used by `gc registry info --remote --from <registry>`. Optional in v1; falling back to `git ls-remote --tags <url>` is acceptable for clients that don't want a second round trip.
- **`GET /search?q=<query>`** *(optional)* — server-side search; returns the same shape as `registry.toml` but filtered. If absent, the client downloads `/registry.toml` and filters locally.

That's it. Three endpoints, all read-only, all serving static-ish TOML. A registry is "the URL of an HTTP server that serves these endpoints." Static hosting works. CDNs work. A git repo with a TOML file at HEAD and `https://raw.githubusercontent.com/...` works.

#### Schema for `registry.toml`

```toml
# registry.toml — wire format for the internet registry protocol

# Top-level fields. Required unless marked optional.
schema = 1                              # integer; current version is 1; bump for breaking changes
name = "canonical"                      # short identifier for this registry; used in --from <name>
description = "Canonical Gas City pack registry"   # human-readable
updated = "2026-04-08T00:00:00Z"        # optional; RFC 3339 timestamp of last edit; informational

# Repeated [[packs]] tables. One per pack the registry knows about.
[[packs]]
name = "gastown"                        # required; the pack's [pack].name from its own pack.toml
url = "https://github.com/example/gastown"   # required; git-cloneable URL (with optional subpath for monorepos)
latest = "1.6.0"                        # required; the highest semver tag the registry knows about
description = "Multi-agent orchestration pack with mayor and polecat agents"   # optional; one-line summary
tags = ["orchestration", "stdlib"]      # optional; array of strings; used for filtering by gc registry search
maintainer = "gastown-team"             # optional; informational; not validated
homepage = "https://gastown.example.com" # optional; informational

[[packs]]
name = "maintenance"
url = "https://github.com/gastownhall/maintenance"
latest = "1.5.0"
description = "Baseline maintenance and supervision agents (default implicit import)"
tags = ["infrastructure"]

[[packs]]
name = "dog"
url = "https://github.com/example/dog"
latest = "0.7.2"
description = "Watchdog and process supervision agents"
tags = ["infrastructure"]
```

**Schema rules:**

- `schema = 1` is the only valid value in v1. Clients SHOULD warn (not error) on unknown schema versions, displaying as much as they can interpret.
- `name`, `url`, and `latest` on each `[[packs]]` entry are **required**. Clients MUST skip entries missing any of these and SHOULD warn.
- Unknown top-level fields and unknown fields inside `[[packs]]` are **ignored** without warning. This lets registries add metadata fields ahead of clients adopting them, and lets clients survive registry-side innovation without breaking.
- Tag strings have no controlled vocabulary in v1. The canonical registry may publish a tag-vocabulary convention later; for v1, tags are freeform and meant for substring search.
- `name` collisions across `[[packs]]` entries within a single `registry.toml` are an error and the registry's responsibility to prevent at PR-review time.

**Schema for `packs/<name>.toml` (optional richer metadata):**

```toml
schema = 1
name = "gastown"
url = "https://github.com/example/gastown"

# All versions the registry knows about, newest first. The registry
# is allowed to lag the upstream repo's tag list — clients should
# treat this as "the registry's best guess" and fall back to
# git ls-remote --tags <url> when they need authoritative data.
[[versions]]
version = "1.6.0"
released = "2026-03-15T00:00:00Z"   # optional; RFC 3339
description = "Adds polecat support."   # optional; per-version notes
deprecated = false                       # optional; default false

[[versions]]
version = "1.5.0"
released = "2026-02-01T00:00:00Z"

[[versions]]
version = "1.4.0"
released = "2026-01-10T00:00:00Z"
deprecated = true                        # optional; informational only — clients still resolve it
```

The `packs/<name>.toml` files are an optimization; in v1 they're optional and the canonical registry may not ship them initially. Clients that need version metadata should either fall back to `git ls-remote` or display "(no extended metadata)" in `gc registry info --remote`.

Critically: **the TOML the registry returns contains URLs**. When `gc registry search` finds gastown, the result is "gastown is at `https://github.com/example/gastown`". The user copies that URL into `gc import add` (no bare-name resolution in v1). The registry never serves pack content. It serves *pointers* to pack content.

This means:
- Registries are cheap to operate (static TOML).
- Registries are easy to fork (clone the TOML file, change one line).
- Registries are impossible to "lose your packs" — the canonical content lives in git repos, not the registry.
- Registries don't need versioning, releases, or pipelines. They're descriptions, not packages.

The v1 canonical registry is a curated `registry.toml` in a git repo: `https://github.com/gastownhall/gc-registry-canonical/blob/main/registry.toml` (working name; final location TBD). `gc-import` is built knowing where this registry lives — there is no user-side configuration needed. The canonical registry is PR-managed: contributors file pull requests against the repo to add or update entries, reviewers merge after a quick sanity check, and the merge is the publish event. No upload pipeline, no auth, no signing — just git. Future iterations could add a real search service, but the v1 design works without one.

## Relationship to `gc import`

The package manager and the registry are designed in tandem; this section spells out exactly how the two interact.

### `gc import add gastown` (bare name) — deferred to v1.5

In the URL-only world we built for `gc import` v0.1, the answer was "no — `add` takes a URL or a path, never a bare name." Names had nowhere to be resolved.

With a local registry, names *do* have somewhere to be resolved. `gc import add gastown` could walk `~/.gc/cache/repos/`, look for a clone whose `pack.toml` declares `name = "gastown"`, read the URL from there, and pass it to the existing `gc import add <url>` flow.

**v1 of `gc import` does NOT support bare-name resolution.** The package manager stays URL-and-path only. Bare-name resolution is **explicitly deferred to phase 1.5**, after both `gc import` v1 and `gc registry` v1 ship. Reasons:

1. **Avoids adding magic before users have lived with the registry.** A bare-name `add` quietly depends on registry state that's invisible from the importing city — that coupling is fine if the user understands the registry, but jarring if they don't.
2. **Keeps the v1 surface honest.** "URL or path" is a one-line rule. "URL, path, or a name that the registry might know about" is a longer rule with new failure modes.
3. **Implicit fallback to internet registries was considered and rejected** — see Alternatives considered.

In v1.5, when bare-name resolution lands, it will be **explicit-only**: `gc import add gastown` errors if `gastown` isn't already in the local registry. The user's recovery is `gc registry add gastown` first (or use the URL form). No silent hits to the network during `add`.

### How the implicit-imports feature intersects with the registry

`gc import` v1 ships with a small, hardcoded list of **implicit imports** — packs that every city imports automatically unless it opts out via `implicit_imports = false`. **The v1 list contains exactly one entry: `maintenance`.** See the `gc import` design ([gastownhall/gascity#434](https://github.com/gastownhall/gascity/issues/434)) for the full mechanics — opt-out, override-by-explicit-import, lock-file representation (`parent = "(implicit)"`), etc.

The interesting question for *this* doc: **where does the implicit list live, and how does it intersect with the registry?**

**v1: hardcoded in `gc-import`.** A literal in the package manager source. The maintenance pack lives at a known URL; that URL is baked into the resolver. Changing the list means changing the code and shipping a new `gc-import`. This is the simplest possible model and it's appropriate for a list of size one. **The registry plays no role in v1 implicit imports.**

**v.next (probably): registry-published implicit list.** Once the registry has been used in anger for a while, the implicit list could be promoted from "literal in source" to "manifest in the default internet registry." The default registry would publish something like:

```toml
# implicit.toml — served from the default internet registry alongside registry.toml
[[implicit]]
name = "maintenance"
url = "https://github.com/gastownhall/maintenance"
version = "^1.5"
```

`gc import add` / `install` would fetch this manifest from the configured default registry, merge it with the user's `[imports]`, and resolve the union. Same opt-out (`implicit_imports = false`); same lock-file representation; same override-by-explicit-import behavior. The change is purely *where* the list comes from.

**Why defer the registry-published version.** Two reasons:

1. **The list is currently size 1.** A registry round-trip is overkill for one URL we ship in code. Until the list grows past 2–3 entries, hardcoded is fine.
2. **It would couple `gc import` to a specific internet registry being reachable.** Today, `gc import` only talks to git URLs at resolve time. If the implicit list comes from `https://registry.gascity.dev/implicit.toml`, then every `gc import add` has a new failure mode ("can't reach the implicit-list registry"). The whole point of the registry being purely additive — never load-bearing for builds — gets undermined the minute the implicit list lives there.

   The mitigation is a stale-cache fallback: cache `implicit.toml` locally and only re-fetch on a successful `gc registry update`. But that's complexity we don't need to take on while the list is one entry. **Decision for v1: hardcoded; revisit when the list grows.**

**Open question for v.next:** *if* we move the implicit list to the registry, does it live in the **default** registry only, or can multiple configured registries each contribute? My lean: default only, single source, never federated. A config that lets multiple registries inject packs into every city is the kind of magic users will hate. **File against the registry repo when v.next thinking starts.**

### Implicit standard library beyond `maintenance`

The **bigger** version of "implicit imports" — a curated standard library with multiple agent packs (gastown, dog, polecat, etc.) implicitly imported into every city — is a different conversation. It's deferred for two distinct reasons:

1. **The product question.** "What goes in the stdlib?" is a product-shape decision that needs Donna, Chris, and Julian. v1 has one answer (`maintenance`); the bigger answer is theirs to make.
2. **The mechanics.** Once the stdlib is more than one entry, the registry-published-list mechanism (above) starts paying for itself. Until then, it's overkill.

**This is filed here as the right home for the discussion**; the registry doc owns the design when we're ready to do it. The shape will probably be: a `[[implicit]]` array in the default registry's `implicit.toml`, with the same opt-out and override semantics as v1's hardcoded version.

### `gc import install` and `gc import upgrade` — read-through with side-effect populate

Both commands **read through the registry as their store layer and populate it as a side effect.** This is identical to the behavior `gc import` v0.1 already has — the rename from "hidden accelerator" to "local registry" doesn't change anything about how the bytes flow:

- `gc import install` reads `pack.lock`, looks up each `(URL, commit)` pair in the registry, clones-on-miss, materializes into the city cache, verifies the hash. The new clone directories appear under `~/.gc/cache/repos/` as a side effect, so a `gc registry list` immediately after a fresh `install` shows every pack the install just pulled in (the on-demand walk picks them up automatically).
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
- **Configuration**: there is exactly one canonical registry. `gc-import` ships knowing where it lives.

The local registry shape: `~/.gc/cache/repos/<hash>/` (clones, existing path, unchanged). No new files anywhere under `~/.gc/`.

## Phasing

**Phase 1 (this proposal):** Five verbs above, against Shape A architecture (city cache stays). The pack store at `~/.gc/cache/repos/` is **unchanged** from the existing accelerator — no new files in the cache hierarchy, no index file, no schema. `gc-import` is built knowing the canonical registry's URL; users have no configuration to do. `gc registry` commands get implemented as a Gas City pack (`gc-registry`, pure Python, no Go changes). `gc import` v1 keeps its current URL-and-path-only `add` semantics — bare-name resolution is **not** in v1.

**Phase 1.5: Bare-name resolution in `gc import add`** (explicit-only mode). `gc import add gastown` works iff `gastown` is in the local registry. Adds a small lookup step before the existing URL flow.

**Phase 2:** Promote the implicit-imports list from "hardcoded in `gc-import` source" (where v1 ships it) to "registry-published `implicit.toml` from the canonical registry." Lets the implicit list grow beyond the v1 single entry (`maintenance`) without requiring a new `gc-import` release for each addition. Pairs with the canonical-registry content design — what's actually in the list is **owned by Donna, Chris, and Julian as a product-shape decision**, not by either tool's implementer.

**Phase 3:** Curated internet registry tooling (`gc registry publish`, signed indices, server-side search), if there's demand.

**Later (v.next, tied to Pack/City refactor):**
- **Shape C: collapse the city cache into the registry.** The loader reads from `~/.gc/cache/repos/<hash>/<subpath>/` directly, no `<city>/.gc/cache/packs/`. Saves disk and removes a layer, but requires loader changes that pair naturally with the v.next loader work. Logged as a v.next change rather than as an open question — the v1 model (Shape A) is fine on its own, and the loader work makes more sense to do all at once.
- **Directory rename: `~/.gc/cache/repos/` → `~/.gc/registry/`.** Cosmetic; we don't make stability promises about the shape of `~/.gc`. Ties to the same v.next work.
- Federation across multiple local registries, if there's demand. Authentication for private internet registries, if there's demand.

## Open questions (still need answers before v1 ships)

These are the questions the design hasn't closed. Each one needs a decision before v1 of `gc registry` ships.

1. **What's actually in the canonical registry on day one?** Pack, Registry, and some combination of gastown/maintenance/dog are the obvious candidates. Decision lives in the canonical registry repo, not here. **File an issue against `gastownhall/gc-registry-canonical` (or wherever the canonical registry ends up) when v1 is closer to shipping.** Also: confirm the final name and location of the canonical registry repo before v1 ships — the working name in this doc is `gastownhall/gc-registry-canonical`, but that's not committed.
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
- **Internet registry as a server-side service with auth and uploads (like npm/PyPI/crates.io).** Centralization is exactly what URL-as-identity was meant to avoid, and a hosted service is operational overhead we don't need. The canonical registry is a git repo with a TOML file; updates are PRs. Same model as homebrew-core, MELPA, and a hundred other community-curated registries.
- **JSON for the wire format.** Conventional for web APIs, but introduces a second format alongside the TOML we use everywhere else. Self-consistency wins.
- **Implicit fallback for bare-name `gc import add`.** If `gastown` isn't in the local registry, query configured internet registries to resolve the name to a URL, then proceed. Quietly couples builds to internet-registry availability for the *initial* `add`, and creates a new failure mode that's hard to debug (`why did this command hit the network?`). Rejected in favor of explicit-only resolution in v1.5.
- **Treating cities as registries.** Briefly considered — what if a city *is* a registry, and importing a pack just means "add a reference to the registry's view of it"? Rejected because cities are consumers, not catalogs. The registry concept is meant to *be* a catalog; conflating the two is the same error as taps were.
- **Skipping the local-registry-as-a-thing entirely and just adding `gc registry search` as a thin discovery wrapper over internet registries.** This is the absolute minimum design — just discovery, no local store, no `gc registry add/remove`. It's defensible but it doesn't answer the airgap/CI/curation use cases, and it doesn't give us a clean answer to "where does the existing accelerator live now?"
- **Auto-suffixing on name collision.** Two URLs both publish `gastown` → second one becomes `gastown-2` automatically. Magical, ugly, debugging nightmare. Refuse + `--name <alias>` is cleaner.

---END ISSUE---
