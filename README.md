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
| `default.json` | Entry point. 7-day cooldown, labels, dependency dashboard. | — (don't extend this repo at all) |
| `schedule.json` | Time windows: language deps daytime-only weekdays, GHA any hour weekdays. | `github>annotell/public-renovate-config//schedule` |
| `security.json` | Vuln PRs bypass cooldown + schedule. | `github>annotell/public-renovate-config//security` |
| `internal.json` | Drops the 7-day cooldown for Kognic-published packages (internal GHA, `kognic-*` Python). | `github>annotell/public-renovate-config//internal` |
| `gha.json` | SHA-pin GitHub Actions, group non-major with automerge, one PR per major. | `github>annotell/public-renovate-config//gha` |
| `python.json` | Group non-major with automerge, one PR per major. | `github>annotell/public-renovate-config//python` |

## Why each preset exists

### `default.json` — 7-day cooldown

`minimumReleaseAge: "7 days"` is a supply-chain guard. It gives upstream
releases a week to surface regressions, yanks, or force-pushes before we offer
the bump. Security PRs bypass this via `security.json`; Kognic-published
packages bypass it via `internal.json`.

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

Scope today is GHA (`annotell/*`) and Python (`kognic-*`). Other ecosystems
(npm, helm, go, docker) will be added when we roll them out.

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
