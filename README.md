# sootbean/soot-preview-action

GitHub Action that runs the [SootBean](https://sootbean.com) preview agent
+ branch-build pipeline for your repo. One job, ~15 lines of YAML.

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
          start: pnpm dev:all:lite
```

That's it. The action does everything else — installs Node and pnpm,
boots your dev stack in the background, dispatches to the right pipeline
based on the event type, and reports back to the
[SootBean cloud](https://sootbean.com).

## What it does

| Event | What runs |
|---|---|
| `pull_request` | Interactive **preview agent** — sticky comment with a recording of your PR exercising the app surface that changed. LLM-gated; skipped automatically when there's nothing to preview. |
| `push` (tracked branch) | **Branch build** — captures the dev=false bundle for every enabled platform and lands it at `https://sootbean.com/build/<owner>/<repo>/<branch>` so anyone with the link can open it on their phone or in the simulator. |

## Inputs

| Name | Required | Default | Description |
|---|---|---|---|
| `start` | recommended | _empty_ | Command that boots your app dev server in the background. The runner waits for it to be ready before recording. |
| `port` | no | `8081` | Port your dev server listens on (Metro/Expo default). |
| `node-version` | no | `24` | Node.js version installed via `actions/setup-node`. |
| `pnpm-version` | no | `10` | pnpm version installed via `pnpm/action-setup`. |
| `origin` | no | `https://sootbean.com` | Override the SootBean cloud origin. |

The runner derives everything else from `${{ github }}` context — repo
id, branch, SHA, PR number, run id, install token. No repo variables
needed; the SootBean cloud resolves your installation id from
`(owner, repo)` via the GitHub App installation it already tracks.

## Prerequisites

1. Install the **[SootBean GitHub App](https://github.com/apps/sootbean)**
   on the repo. The app opens a bootstrap PR with this workflow on first
   install; this README is for repos that want to write the workflow by
   hand.
2. Open the repo on [sootbean.com](https://sootbean.com) once so the
   per-repo build settings (which platforms to build, build visibility,
   tracked branches) are persisted. Defaults: iOS-only, unlisted.

## Source

The composite action's full source lives at
[`actions/soot-preview/action.yml`](https://github.com/sootbean/soot-preview-action/blob/main/action.yml)
in this repo. Issues and feature requests welcome.
