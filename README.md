# getdoover/workflows

The CI/CD pipeline for every Doover app repo, in one place.

An app repo carries **one** workflow file with no logic in it. Everything else —
what to lint, how to test, when to publish, what to release against — lives here
and is changed by moving the `v1` tag, not by touching 55 repositories.

## Using it

A repo with one app, or several apps declared in a single root
`doover_config.json` (see `examples/single-app.yml`):

```yaml
name: Doover App

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: read
  id-token: write

jobs:
  app:
    uses: getdoover/workflows/.github/workflows/app.yml@main
    secrets: inherit
```

A monorepo keeping each app in its own directory adds one copy per app, with a
`paths:` filter and `with: {app: <dir>}`, so changing one app rebuilds only that
app. See `examples/monorepo-per-app.yml`.

`on:` and `permissions:` have to be in the caller — a reusable workflow cannot
supply them. They are therefore the only part a `v1` bump cannot reach, which is
why they are deliberately broad: every decision about what to *do* on a given ref
is made here.

## No secrets

Inside Actions with `id-token: write`, the Doover CLI detects the runner and
authenticates over the trusted-publisher OIDC flow. There is no `DOOVER_API_TOKEN`
to configure, no registry credential, and no image-signing key — releases are
signed server-side by the control plane. `secrets: inherit` is there for anything
else a repo needs, not for talking to Doover.

## What it does

| Job | |
|---|---|
| `discover` | `doover app discover --json` builds the matrix, so no app name is ever hard-coded |
| `check` | lint, plus config and UI schema validation per app |
| `test` | pytest inside `spaneng/doover_device_base` |
| `smoke` | build the image, then `import <module>` *inside* it — where a missing system library actually surfaces |
| `publish` | register, log in, buildx push, then release bound to the pushed digest |
| `publish-package` | the other deployable: `./build.sh`, upload `package.zip`, release — processors, reports and integrations |

`fail-fast: false` throughout, so one broken app never masks the others. Pull
requests release as `--alpha`, so a PR version never auto-selects as Latest.

## Inputs

| Input | Default | |
|---|---|---|
| `app` | `.` | Directory to search. Monorepos pass one app dir per caller. |
| `release` | `true` | Create an immutable version. Off leaves the image unsigned. |
| `staging` | `false` | Also release to the staging control plane, against the digest already in the production registry. |
| `runs-on` | `ubuntu-latest` | `ubuntu-24.04-arm` builds arm64 natively rather than under QEMU. |
| `test-image` | `spaneng/doover_device_base` | |
| `python-version` | `3.11` | |
| `environment` | — | GitHub environment to gate publishing behind. |

Anything that varies per *app* belongs in that app's `doover_config.json`, which
`discover` surfaces — not here. Inputs are only for what varies per *invocation*.
That is what keeps the caller identical across repos.

## Versioning

Callers pin `@main`, deliberately and long-term. The point of this repo is to
change what every app repo does without editing 55 workflow files, and a tag —
even a floating `v1` — is one more thing to move before a fix reaches the fleet.

Know what that buys and what it costs. A push here is live everywhere on the next
run, with no reviewed ref to sit behind and nothing to roll back to but another
push. These jobs hold `id-token: write` and can mint Doover publish credentials.
So treat main as production: land changes by PR, and exercise `publish` /
`publish-package` against a real repo before merging.

Anyone needing determinism pins a SHA in their own caller.

See `NOTES.md` for the design decisions, the open question about `staging`, and
what still has to ship before this can run.
