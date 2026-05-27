# renovate-config

Centralized Renovate config at Kognic. Loaded automatically by the self-hosted
Renovate bot via `globalExtends` — repos do not need a `renovate.json` to
participate.

## How it's applied

The Renovate deployment in `k8s-platform-gitops` sets:

```js
globalExtends: ['github>annotell/public-renovate-config']
```

…which resolves to [`default.json`](./default.json). That preset extends the
sub-presets in this repo. Every scanned repo inherits the resulting config.

Repo-local config (when present) is layered on top and can override anything
except entries in the bot's `force` block (currently just `addLabels: ['autoreview']`).

All preset files are strict JSON — no comments, no trailing commas. Renovate's
GitHub preset resolver only looks for `default.json` (then `renovate.json`)
for an unqualified `github>org/repo` reference and only `<name>.json` for a
sub-preset `github>org/repo//<name>` — see
[renovatebot/renovate#15370](https://github.com/renovatebot/renovate/issues/15370).
Rationale for each rule lives here in the README rather than in inline
comments.

## Layout

| File | Purpose | Opt-out string |
|---|---|---|
| `default.json` | Entry point. 7-day cooldown, `fix(deps)` commit prefix, labels, dependency dashboard. | — (don't extend this repo at all) |
| `schedule.json` | Time windows: language deps daytime-only weekdays, GHA any hour weekdays. | `github>annotell/public-renovate-config//schedule` |
| `security.json` | Vuln PRs bypass cooldown + schedule. | `github>annotell/public-renovate-config//security` |
| `internal.json` | Drops the 7-day cooldown for Kognic-published packages (internal GHA, `kognic-*` Python). | `github>annotell/public-renovate-config//internal` |
| `gha.json` | SHA-pin GitHub Actions, group non-major with automerge, one PR per major. | `github>annotell/public-renovate-config//gha` |
| `python.json` | Group non-major with automerge, one PR per major. | `github>annotell/public-renovate-config//python` |
| `gradle.json` | Group non-major with automerge, one PR per major. | `github>annotell/public-renovate-config//gradle` |
| `go.json` | Group non-major with automerge, one PR per major. | `github>annotell/public-renovate-config//go` |
| `npm.json` | Group non-major, one PR per major. No automerge. | `github>annotell/public-renovate-config//npm` |
| `rust.json` | Group non-major, one PR per major. No automerge. | `github>annotell/public-renovate-config//rust` |

## Why each preset exists

### `default.json` — 7-day cooldown

`minimumReleaseAge: "7 days"` is a supply-chain guard. It gives upstream
releases a week to surface regressions, yanks, or force-pushes before we offer
the bump. Security PRs bypass this via `security.json`; Kognic-published
packages bypass it via `internal.json`.

### `default.json` — `fix(deps)` commit prefix

`semanticCommitType: "fix"` + `semanticCommitScope: "deps"` make dependency
bumps land as `fix(deps): …` rather than Renovate's default `chore(deps): …`.
Under Conventional Commits, `fix` is release-triggering (patch) while `chore`
is not, so this lets release-please / semantic-release cut a release when a dep
update merges. `semanticCommits: "enabled"` forces the prefix in every repo
rather than the default `"auto"`, which would only apply it where the commit
history already uses Conventional Commits.

### `schedule.json` — business-hours window for language deps

Language deps land between 08:00 and 16:00 on weekdays so the
image-updater-driven prod deploys happen while people are around. GHA is
low-risk and can land any time on weekdays.

### `security.json` — vuln PRs bypass cooldown and schedule

A CVE bump can land any hour. The `security` label lets `kognic-github-bot`
and humans distinguish these from regular dep updates.

OSV works without extra GitHub App scopes. GitHub Advisories additionally
require `security_events: read` on the App — verify before relying on it.

### `internal.json` — no cooldown for our own packages

The 7-day cooldown protects against third-party packages that get yanked,
force-pushed, or compromised shortly after release. None of that applies to
packages we publish ourselves — we know what we shipped.

Scope today is GHA (`annotell/*`), Python (`kognic-*`), Gradle
(`com.kognic.*`), npm (`@kognic/*`, `@annotell/*`), and Cargo git deps from
`github.com/annotell/*` and `github.com/kognic/*`. Other ecosystems (helm,
go, docker) will be added when we roll them out.

### `gha.json` — SHA-pin actions, group non-major

Every `uses:` line is SHA-pinned with the human-readable tag as a trailing
comment, e.g. `actions/checkout@a1b2c3... # v4.1.0`.

Non-major bumps (patch, minor, digest, pinDigest) are grouped into one PR with
the `autoreview` label so the AI bot can merge it if safe. Major bumps get one
PR per action because action inputs/outputs frequently break across majors and
grouping them makes triage harder.

### `python.json` — group non-major, one PR per major

Non-major Python bumps grouped into one PR with `autoreview` + `automerge`.
Major bumps get one PR per package because Python major bumps frequently
break public APIs (sqlalchemy, pydantic, fastapi, etc.) and benefit from
individual review.

### `gradle.json` — group non-major, one PR per major

Non-major Gradle bumps (minor + patch) grouped into one PR with `autoreview`
+ `automerge`. Covers both the `gradle` manager (dependencies in
`libs.versions.toml` / `build.gradle.kts`) and `gradle-wrapper` (the Gradle
distribution itself).

Major bumps get one PR per dependency because major-version upgrades of core
Scala libraries (scala-library, pekko, slick, tapir, etc.) frequently break
source compatibility and benefit from individual review before merging.

### `go.json` — group non-major, one PR per major

Non-major Go module bumps (minor + patch) grouped into one PR with `autoreview`
+ `automerge`. Go's compatibility promise within a major version means a
passing test suite is a strong signal that a minor/patch bump is safe.

Major bumps get one PR per module because Go major versions are explicit
breaking changes (the import path changes, e.g. `foo/v2`) and need code
changes to adopt — they benefit from individual review.

We don't currently publish internal Go packages, so there's no `kognic-*`
carve-out in `internal.json` for Go yet. Add one when that changes.

### `npm.json` — group non-major, one PR per major, no automerge

Non-major npm bumps (minor + patch) grouped into one PR with `autoreview`.
Major bumps get one PR per package because major bumps in the JS ecosystem
(react, next, eslint, vite, etc.) routinely ship breaking API changes and
benefit from individual review.

Unlike `go.json` and `gradle.json`, npm does **not** automerge non-major
bumps. The JS ecosystem has a higher incidence of breaking changes shipped
under minor/patch versions, so we keep a human in the loop.

Internal packages are served from the GAR registry at
`europe-west1-npm.pkg.dev/annotell-com/npm-default/` — repos configure that
via a per-repo `.npmrc` (e.g. `@kognic:registry=…`), no central host rule
needed. The `@kognic/*` and `@annotell/*` carve-outs in `internal.json` drop
the 7-day cooldown for those scopes.

The `npm` manager covers npm, pnpm, and yarn lockfiles; the `bun` manager
covers `bun.lock(b)`. Both are included in the rules. `rangeStrategy: bump`
raises the declared range in `package.json` for in-range releases (e.g.
`^18.2.0` locked at 18.2.5 → `^18.2.6` with the lockfile at 18.2.6),
mirroring the Python preset. Reviewers see the change in `package.json`,
and downstream consumers of our published packages resolve from the actual
tested floor.

### `rust.json` — group non-major, one PR per major, no automerge

Non-major Cargo bumps (minor + patch) grouped into one PR with `autoreview`.
Major bumps get one PR per crate because major-version upgrades of core
crates (tokio, serde, axum, etc.) frequently break source compatibility and
benefit from individual review.

Like `npm.json`, Cargo does **not** automerge non-major bumps. Cargo treats
`0.x.y` as semver where every `0.x` bump is breaking, and Renovate already
classifies those as major — but minor/patch bumps on `1.0+` crates still
occasionally ship subtle behavior changes in the ecosystem, so we keep a
human in the loop.

Internal crates today are pulled directly from GitHub as `git = …` Cargo
dependencies. CI authenticates with a GitHub App token and a
`git config --global url.…insteadOf` rewrite (see e.g.
`judgement-interpolator/.github/workflows/rust.yml`); the Renovate bot uses
its own GitHub token to read the same repos, so no central host rule is
needed here. The `github.com/annotell/*` and `github.com/kognic/*`
carve-out in `internal.json` drops the 7-day cooldown for those Cargo git
deps. If we later publish to a private crates registry, add a host rule in
the self-hosted bot config rather than here.

## Opting out of a sub-preset

Drop a `renovate.json` in the repo:

```json
{
  "extends": ["github>annotell/public-renovate-config"],
  "ignorePresets": ["github>annotell/public-renovate-config//schedule"]
}
```

`ignorePresets` strings must match the references in `default.json` exactly —
copy-paste from the table above rather than guessing.

## Adding a new language preset

1. Create `<lang>.json` at repo root (strict JSON — no comments, no trailing commas).
2. Add it to the `extends` list in `default.json`.
3. List it in the table above with its opt-out string.
4. Add a `### <lang>.json` section under [Why each preset exists](#why-each-preset-exists) with rationale.

Language presets are no-ops in repos that don't use the relevant manager, so
including them all in `default.json` is safe.
