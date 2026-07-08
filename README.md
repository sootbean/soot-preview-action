<!--
  SOURCE OF TRUTH. This README (and action.yml next to it) is mirrored to the
  public github.com/sootbean/soot-preview-action repo by
  `bun scripts/ops/publish-preview-action.ts --push`. Edit here, not there.
-->

# sootbean/soot-preview-action

GitHub Action that runs the [SootBean](https://sootbean.com) preview agent
+ branch-build pipeline for your repo. One job, ~15 lines of YAML.

> **Requires a [Carbon plan](https://sootsim.com/docs/sootsim/plans).** PR
> previews and branch builds run on SootBean's cloud; the paid plan is what
> unlocks the automated preview/build flows and the hosted preview links.

## Quick start

```yaml
name: Soot
on:
  push:
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]
permissions:
  contents: read
  pull-requests: write
  issues: write
jobs:
  soot:
    if: ${{ github.event.pull_request.draft != true }}
    runs-on: ubuntu-latest
    timeout-minutes: 25
    steps:
      - uses: actions/checkout@v4
      - uses: sootbean/soot-preview-action@v1
        with:
          start: npm run dev   # whatever boots your app dev server
```

That's it. The action auto-detects your package manager, installs deps, boots
your dev stack in the background, dispatches to the right pipeline based on the
event type, and reports back to the SootBean cloud.

## Package manager

By default the action detects your package manager from the lockfile /
`package.json` `packageManager` field and installs with it:

| Detected | Install command |
|---|---|
| `npm` (`package-lock.json`) | `npm ci` |
| `yarn` classic (`yarn.lock`) | `yarn install --frozen-lockfile` |
| `yarn` berry (`.yarnrc.yml`) | `yarn install --immutable` |
| `pnpm` (`pnpm-lock.yaml`) | `pnpm install --frozen-lockfile` |
| `bun` (`bun.lock`/`bun.lockb`) | `bun install --frozen-lockfile` |

Override with `package-manager:` (`npm` \| `yarn` \| `pnpm` \| `bun`), or set a
custom `install:` command for monorepos.

**Bring your own setup.** If your workflow already checks out, sets up Node, and
installs deps (e.g. a repo `./.github/actions/setup` composite), run it *before*
this action and pass `package-manager: none` so this action skips straight to the
preview/build:

```yaml
      - uses: ./.github/actions/setup   # your own composite
      - uses: sootbean/soot-preview-action@v1
        with:
          package-manager: none
          start: pnpm dev:all:lite
```

## What it does

| Event | What runs |
|---|---|
| `pull_request` | Interactive **preview agent** — a sticky comment with a recording of your PR exercising the app surface that changed. LLM-gated; skipped automatically when there's nothing to preview. |
| `push` (tracked branch) | **Branch build** — captures the bundle for every enabled platform and lands it at `https://sootbean.com/build/<owner>/<repo>/<branch>`. |

## Inputs

| Name | Default | Description |
|---|---|---|
| `start` | _empty_ | Command that boots your app dev server in the background. The runner waits for it to be ready before recording. Leave blank if the app is already up. |
| `port` | `8081` | Port your dev server listens on (Metro/Expo default). |
| `package-manager` | `auto` | `auto` (detect) \| `npm` \| `yarn` \| `pnpm` \| `bun` \| `none` (skip setup + install; you ran your own). |
| `install` | _empty_ | Override the install command entirely (e.g. a monorepo filter). |
| `node-version` | `24` | Node.js version installed via `actions/setup-node`. Ignored when `package-manager: none`. |
| `pnpm-version` | _empty_ | pnpm version (only when the resolved manager is pnpm). Empty reads from `packageManager`. |
| `mode` | `auto` | `auto` (dispatch by event) \| `preview` \| `build`. |
| `platform` | _empty_ | When `mode=build`, the single platform (`ios`/`android`). Wire to a matrix for parallelism. |
| `maestro` | _empty_ | Maestro flow paths, or `auto` to run every `.maestro/*.yaml`. Each flow runs in SootSim against the PR's bundle and uploads its own replayable preview, listed in the sticky comment's Tests table and the "Soot Tests" check run. |
| `maestro-on` | `pull_request` | Which events run maestro flows: CSV of `pull_request,push:main,push:staging`. |
| `runner` | `local` | `local` runs everything on this workflow's runner. `hosted` keeps this runner serving the dev stack through a token-gated tunnel while a SootBean GPU recorder drives and records it — dramatically smoother videos than a CPU-only CI runner. Requires the Carbon plan. |
| `build-env` | _empty_ | Newline-separated `KEY=VALUE` pairs baked into the captured bundle (e.g. `EXPO_PUBLIC_*` / Clerk keys). Source from `${{ secrets.* }}`. |
| `origin` | `https://sootbean.com` | Override the SootBean cloud origin. |

The runner derives everything else from `${{ github }}` context — repo id,
branch, SHA, PR number, run id, install token. No repo variables needed.

## Prerequisites

1. Install the **[SootBean GitHub App](https://github.com/apps/sootbean)** on the
   repo. The app opens a bootstrap PR with this workflow on first install; this
   README is for repos that want to write the workflow by hand.
2. Open the repo on [sootbean.com](https://sootbean.com) once so per-repo build
   settings (platforms, visibility, tracked branches) are persisted.
3. Be on the [Carbon plan](https://sootsim.com/docs/sootsim/plans).

## Source

The action's source of truth lives in the private `sootbean/soot` monorepo at
`actions/soot-preview/action.yml` and is mirrored here. Issues and feature
requests welcome.
