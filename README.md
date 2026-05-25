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
other files in this repo. Every scanned repo inherits the resulting config.

### File naming

Mixed `.json` and `.json5` is deliberate, and driven by two Renovate
constraints that pull in opposite directions:

- The GitHub preset resolver only tries `default.json` (then
  `renovate.json`) for an unqualified `github>org/repo` reference and
  only `<name>.json` for a sub-preset `github>org/repo//<name>` — see
  [renovatebot/renovate#15370][rb15370].
- A file named `.json` is expected to be strict JSON. Renovate currently
  tolerates JSON5 syntax in `.json` files but emits a deprecation
  warning and plans to remove the fallback ([renovatebot/renovate#36877][rb36877]).

So:

- `default.json` is **strict JSON** with no comments or trailing commas,
  and references the sub-presets by their full `.json5` filename so the
  resolver doesn't append `.json` itself.
- Sub-presets stay `.json5`, with comments and trailing commas, because
  they are only ever referenced with the explicit extension and the
  JSON5 contents are valuable inline documentation.

[rb15370]: https://github.com/renovatebot/renovate/issues/15370
[rb36877]: https://github.com/renovatebot/renovate/discussions/36877

### Cooldown rationale

`default.json` sets `minimumReleaseAge: "7 days"` — a supply-chain guard
giving upstream releases a week to surface regressions, yanks, or
force-pushes before we offer the bump. Security PRs bypass this via
`security.json5`; Kognic-published packages bypass it via
`internal.json5`.

Repo-local config (when present) is layered on top and can override anything
except entries in the bot's `force` block (currently just `addLabels: ['autoreview']`).

## Layout

| File | Purpose | Opt-out string |
|---|---|---|
| `default.json` | Entry point. 7-day cooldown, labels, dependency dashboard. | — (don't extend this repo at all) |
| `schedule.json5` | Time windows: language deps daytime-only weekdays, GHA any hour weekdays. | `github>annotell/public-renovate-config//schedule.json5` |
| `security.json5` | Vuln PRs bypass cooldown + schedule. | `github>annotell/public-renovate-config//security.json5` |
| `internal.json5` | Drops the 7-day cooldown for Kognic-published packages (internal GHA, `kognic-*` Python). | `github>annotell/public-renovate-config//internal.json5` |
| `gha.json5` | SHA-pin GitHub Actions, group non-major with automerge, one PR per major. | `github>annotell/public-renovate-config//gha.json5` |
| `python.json5` | Group non-major with automerge, one PR per major. | `github>annotell/public-renovate-config//python.json5` |

## Opting out of a sub-preset

Drop a `renovate.json` in the repo:

```json
{
  "extends": ["github>annotell/public-renovate-config"],
  "ignorePresets": ["github>annotell/public-renovate-config//schedule.json5"]
}
```

`ignorePresets` strings must match the references in `default.json` exactly —
copy-paste from the table above rather than guessing. Note the explicit
`.json5` extension; see [File naming](#file-naming) for why.

## Adding a new language preset

1. Create `<lang>.json5` at repo root.
2. Add it to the `extends` list in `default.json`.
3. List it in the table above with its opt-out string.

Language presets are no-ops in repos that don't use the relevant manager, so
including them all in `default.json` is safe.
