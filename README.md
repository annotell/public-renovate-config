# renovate-config

Centralized Renovate config at Kognic. Loaded automatically by the self-hosted
Renovate bot via `globalExtends` — repos do not need a `renovate.json` to
participate.

## How it's applied

The Renovate deployment in `k8s-platform-gitops` sets:

```js
globalExtends: ['github>annotell/public-renovate-config']
```

…which resolves to [`default.json5`](./default.json5). That preset extends the
other files in this repo. Every scanned repo inherits the resulting config.

Repo-local config (when present) is layered on top and can override anything
except entries in the bot's `force` block (currently just `addLabels: ['autoreview']`).

## Layout

| File | Purpose | Opt-out string |
|---|---|---|
| `default.json5` | Entry point. 7-day cooldown, labels, dependency dashboard. | — (don't extend this repo at all) |
| `schedule.json5` | Time windows: language deps daytime-only weekdays, GHA any hour weekdays. | `github>annotell/public-renovate-config//schedule` |
| `security.json5` | Vuln PRs bypass cooldown + schedule. | `github>annotell/public-renovate-config//security` |
| `internal.json5` | Drops the 7-day cooldown for Kognic-published packages (internal GHA, `kognic-*` Python). | `github>annotell/public-renovate-config//internal` |
| `gha.json5` | SHA-pin GitHub Actions, group non-major with automerge, one PR per major. | `github>annotell/public-renovate-config//gha` |
| `python.json5` | Group non-major with automerge, one PR per major. | `github>annotell/public-renovate-config//python` |

## Opting out of a sub-preset

Drop a `renovate.json` in the repo:

```json
{
  "extends": ["github>annotell/public-renovate-config"],
  "ignorePresets": ["github>annotell/public-renovate-config//schedule"]
}
```

`ignorePresets` strings must match the references in `default.json5` exactly —
copy-paste from the table above rather than guessing.

## Adding a new language preset

1. Create `<lang>.json5` at repo root.
2. Add it to the `extends` list in `default.json5`.
3. List it in the table above with its opt-out string.

Language presets are no-ops in repos that don't use the relevant manager, so
including them all in `default.json5` is safe.
