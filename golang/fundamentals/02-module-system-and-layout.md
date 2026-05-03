---
title: "Go Module System & Project Layout"
date: 2026-05-03
updated: 2026-05-03
tags: [golang, fundamentals, modules, project-layout, dependency-management]
---

# Go Module System & Project Layout

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `golang` `fundamentals` `modules` `project-layout` `dependency-management`

---

## Table of Contents

1. [What a module is, and how it differs from npm and Maven](#1-what-a-module-is-and-how-it-differs-from-npm-and-maven)
2. [`go.mod`: every field explained](#2-gomod-every-field-explained)
3. [`go.sum` and the module proxy](#3-gosum-and-the-module-proxy)
4. [Semantic Import Versioning and the v2+ rule](#4-semantic-import-versioning-and-the-v2-rule)
5. [Project layout: `cmd/`, `internal/`, `pkg/`](#5-project-layout-cmd-internal-pkg)
6. [`replace`, `exclude`, and local development](#6-replace-exclude-and-local-development)
7. [Vendoring and reproducible builds](#7-vendoring-and-reproducible-builds)
8. [Private modules and `GOPRIVATE`](#8-private-modules-and-goprivate)

## Summary

Go's module system was introduced in Go 1.11 (August 2018) and became the only supported dependency mode in Go 1.16. It is opinionated in ways that surprise people coming from npm or Maven: there is no central registry of names — module identity **is the import path**, which is usually a URL. There is no `node_modules` per project — downloaded modules live in a global cache (`$GOPATH/pkg/mod`) and are reused across projects. Major version changes are encoded into the import path itself (`example.com/foo` for v0/v1, `example.com/foo/v2` for v2+) so that two major versions of the same library can coexist in the same build.

The layout conventions (`cmd/`, `internal/`, `pkg/`) are not a framework — they are community patterns documented in `golang-standards/project-layout`, with the single hard rule being that `internal/` is enforced by the compiler.

## 1. What a module is, and how it differs from npm and Maven

A **module** is a tree of Go packages versioned as a unit, identified by a **module path**. The module path is declared in `go.mod` and is, by convention, the URL where the module's source can be fetched.

```text
// go.mod
module github.com/example/myapp

go 1.22

require (
    github.com/google/uuid v1.6.0
    golang.org/x/sync v0.7.0
)
```

| Concept | npm | Maven | Go modules |
|---|---|---|---|
| Identity | name in `package.json` (string in registry) | `groupId:artifactId` (in central repo) | module path = URL (no central registry) |
| Per-project install | `node_modules/` directory | none — local Maven repo (`~/.m2`) | none — global cache (`$GOPATH/pkg/mod`) |
| Lock file | `package-lock.json` / `pnpm-lock.yaml` | none built-in (use enforcer plugins) | `go.sum` (hashes, not versions) |
| Major version coexistence | impossible without aliasing | `groupId` namespacing | encoded in import path: `/v2`, `/v3` |
| Default registry | `registry.npmjs.org` | Maven Central | none — fetches directly from VCS via proxy |

The "module path = URL" rule is what makes Go feel different. There is no `github.com/google/uuid` in a package registry — the module path **is** the GitHub URL, and `go get` fetches from it directly (mediated by the module proxy, see §3).

## 2. `go.mod`: every field explained

```text
module github.com/example/myapp     // 1. module path

go 1.22                              // 2. minimum Go language version

require (                            // 3. direct dependencies
    github.com/google/uuid v1.6.0
    golang.org/x/sync v0.7.0
)

require (                            // 4. indirect dependencies (added by go mod tidy)
    github.com/davecgh/go-spew v1.1.1 // indirect
)

replace github.com/old/lib => ../local-lib  // 5. local override (rarely committed)

exclude github.com/bad/lib v1.2.3            // 6. veto a specific version
```

1. **`module`** — the canonical import path. Code outside this module imports as `github.com/example/myapp/internal/foo`; code inside this module imports siblings the same way (Go does not have relative imports).
2. **`go`** — minimum language version. Pinning `go 1.22` means features added in 1.22 (e.g., the loop-variable-per-iteration semantics) compile correctly. With Go 1.21+ this also drives **toolchain selection**: `go 1.22` will refuse to build with Go 1.21.
3. **`require` (direct)** — what your code imports.
4. **`require` (indirect, marked `// indirect`)** — transitive dependencies you don't import yourself, kept here so the build is reproducible. Managed automatically by `go mod tidy`.
5. **`replace`** — substitute a module with a fork or a local path. Useful during development; usually not committed to main branches because it makes the module non-fetchable for downstream consumers.
6. **`exclude`** — refuse a specific version (e.g., a known-broken release). Rarely used.

`go mod tidy` is the canonical way to keep `go.mod` and `go.sum` in sync with what the code actually imports. CI typically runs `go mod tidy && git diff --exit-code go.mod go.sum` to catch drift.

## 3. `go.sum` and the module proxy

`go.sum` is a checksum file. It pins a cryptographic hash of each module version your build observed:

```text
github.com/google/uuid v1.6.0 h1:NIvaJDMOsjHA8n1jAhLSgzrAzy1Hgr+hNrb57e+94F0=
github.com/google/uuid v1.6.0/go.mod h1:TIyPZe4MgqvfeYDBFedMoGGpEw/LqOeaOT+nhxU+yHo=
```

Two hashes per version:
- The first hash is the module's source tarball.
- The `/go.mod` hash is the module's `go.mod` file alone, used for **minimum version selection** (MVS) without downloading the full module.

The hashes are looked up in the **checksum database** at `sum.golang.org`, an append-only Merkle tree maintained by Google. This guarantees that every developer fetching `github.com/google/uuid v1.6.0` gets bit-identical bytes — a supply-chain protection that npm took until 2021 to add (via `npm install --provenance`).

Module fetches go through the **module proxy** at `proxy.golang.org` by default. The proxy:
- Caches every public module version forever (modules cannot be unpublished — Go's response to the `left-pad` incident on npm).
- Removes the dependency on the original VCS being available at build time.
- Can be replaced via the `GOPROXY` environment variable for private or air-gapped builds.

```bash
# Default proxy chain
GOPROXY=https://proxy.golang.org,direct

# Corporate proxy fallback
GOPROXY=https://proxy.corp.example.com,https://proxy.golang.org,direct

# Disable proxy entirely (fetch from VCS directly)
GOPROXY=direct
```

## 4. Semantic Import Versioning and the v2+ rule

Go takes semver further than any other major language: **the major version is part of the import path**.

- v0 and v1: `import "github.com/foo/bar"`
- v2: `import "github.com/foo/bar/v2"`
- v3: `import "github.com/foo/bar/v3"`

The reasoning, per [Russ Cox's "Semantic Import Versioning" essay](https://research.swtch.com/vgo-import) (2018): if v2 is incompatible with v1 (which is what a major version bump means in semver), then v2 and v1 are **different packages**. Calling them the same name would be a lie. So Go encodes the major version in the path, and v1 and v2 of the same module can be imported into the same binary without conflict.

For module authors, going from v1 to v2 means:
1. Update the module path in `go.mod`: `module github.com/foo/bar/v2`
2. Tag the release `v2.0.0`
3. All imports inside the module (and in consumers) update to `github.com/foo/bar/v2/...`

For consumers, this means an upgrade from v1 to v2 is a deliberate code change, not a transparent dependency bump. That's a feature: it forces you to acknowledge breaking changes at the call site instead of finding them at runtime.

## 5. Project layout: `cmd/`, `internal/`, `pkg/`

The community convention, originally codified in [`golang-standards/project-layout`](https://github.com/golang-standards/project-layout):

```text
myapp/
├── go.mod
├── go.sum
├── cmd/
│   ├── api/                 # one binary per subdirectory
│   │   └── main.go
│   └── worker/
│       └── main.go
├── internal/                # not importable from outside this module
│   ├── auth/
│   ├── billing/
│   └── repo/
├── pkg/                     # public, importable libraries (use sparingly)
│   └── client/
├── api/                     # OpenAPI/protobuf specs
├── configs/                 # default config files
├── deployments/             # Dockerfiles, k8s manifests
├── scripts/                 # build / dev scripts
└── test/                    # integration test data, e2e
```

The one **compiler-enforced** rule is `internal/`. The Go specification states:

> An import of a path containing the element "internal" is disallowed if the importing code is outside the tree rooted at the parent of the "internal" directory.

So `myapp/internal/auth` can be imported by `myapp/cmd/api` (same tree) but **cannot** be imported by `otherapp` even if both are in the same Go workspace. This is the only access-control mechanism Go provides at the package level.

`pkg/` is **not** enforced. It's a convention to signal "this is a stable public API." The Go standard library does not use `pkg/`, and many high-quality projects (kubernetes, prometheus) have moved away from it because it adds path noise without compiler benefit. The current advice is: put public code at the module root, put internal code under `internal/`, and only introduce `pkg/` if you have a real reason.

`cmd/` is convention plus a build behaviour: `go build ./cmd/api` produces a binary named `api`. Each subdirectory under `cmd/` becomes one entry-point binary.

## 6. `replace`, `exclude`, and local development

When you want to develop two modules in tandem (e.g., a service and a library it depends on):

```text
// In the consumer's go.mod
replace github.com/example/lib => ../lib
```

This redirects imports of `github.com/example/lib` to a local directory, regardless of version. The original `require` entry is still respected for the version constraint, but the source comes from `../lib`.

For multi-module repos, prefer `go.work` (introduced in Go 1.18) over committing `replace` directives. A workspace file lets developers wire modules together locally without changing any module's `go.mod`:

```text
// go.work (not committed in some teams; committed in others)
go 1.22

use (
    ./service
    ./lib
)
```

The Go toolchain reads `go.work` automatically when present. CI usually runs without it (`GOWORK=off`) so builds match what consumers see.

`exclude` vetos a specific version — useful when a release is known broken (e.g., a bug that reaches main but isn't yanked):

```text
exclude github.com/some/lib v1.2.3
```

## 7. Vendoring and reproducible builds

`go mod vendor` copies all dependencies into a `vendor/` directory at the repo root. `go build` will use `vendor/` automatically when it exists. This:
- Eliminates the dependency on `proxy.golang.org` at build time (useful for air-gapped or hermetic builds).
- Makes the entire build self-contained in the repo (useful for security audits — every byte you ship is in your VCS).

Tradeoffs:
- Repository size grows significantly (often by 10–100 MB).
- Diffs become noisy when dependencies change.

Most modern Go projects do not vendor — the proxy plus `go.sum` already gives reproducibility, and CI caches `$GOPATH/pkg/mod` between runs. Vendoring is most common in regulated environments and in the Kubernetes ecosystem (where it's historical).

## 8. Private modules and `GOPRIVATE`

By default, Go fetches every module through the public proxy and verifies it against the public checksum database. For private repos this is wrong on two counts: the module isn't available publicly, and trying to fetch it leaks the module path to Google.

`GOPRIVATE` fixes this:

```bash
# Pattern-match private module paths
export GOPRIVATE=github.com/mycorp/*,gitlab.example.com/*

# Effects:
# - GONOSUMCHECK behaviour for matching paths (skip sum.golang.org)
# - GONOPROXY behaviour for matching paths (fetch direct from VCS)
```

For Git-based private repos, you'll typically also need an authentication mechanism:

```bash
# .netrc for HTTPS
machine github.com login my-token password x-oauth-basic

# Or rewrite to SSH
git config --global url."git@github.com:".insteadOf "https://github.com/"
```

## Related

- [Go Tour for TS + Java Developers](01-go-tour-for-ts-java-devs.md) — packages, imports, visibility
- [Types, Zero Values & Composite Literals](03-types-zero-values-composites.md) — what lives inside packages
- [Errors as Values](06-errors-as-values.md) — error wrapping across module boundaries
- [TypeScript Module Resolution Deep Dive](../../typescript/tooling/module-resolution.md) — contrast with Node's resolution algorithm

## References

- "Go 1.11 is released" — https://go.dev/blog/go1.11 (introduces modules)
- Go Modules Reference — https://go.dev/ref/mod
- Russ Cox, "Semantic Import Versioning" — https://research.swtch.com/vgo-import (2018)
- Russ Cox, "Minimum Version Selection" — https://research.swtch.com/vgo-mvs (2018)
- "Go's Module Mirror, Index, and Checksum Database" — https://go.dev/blog/module-mirror-launch (2019)
- The Go Programming Language Specification, "Import declarations" — https://go.dev/ref/spec#Import_declarations
- `golang-standards/project-layout` — https://github.com/golang-standards/project-layout (community convention; not an official Go team standard, as the README itself notes)
- "What is a Go workspace?" — https://go.dev/blog/get-familiar-with-workspaces (2022)
