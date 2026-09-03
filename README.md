# ci-workflows

Reusable GitHub Actions workflows shared across `bpulidodtx-7575` repositories.
Public so private repos can call them with zero extra configuration —
`workflow_call` from a private repo works against a public one without a
token; against a private one it would need `permissions` set up per-caller.
Nothing in here reads or writes anything outside the calling repo's own
checkout — no secrets live in this repo.

## Why this exists

Four repos converged on the same CI shape independently: lint, format check,
typecheck, coverage-gated tests, a dead-code gate (knip), an AI-slop score
gate (aislop), a secret scan (gitleaks), and workflow-file linting
(actionlint + zizmor). That logic was hand-copied into each repo — 1,130+
lines of near-duplicate YAML — which means a bug fixed in one repo silently
stays broken in the other three. This repo is that logic, written once.

## Workflows

### `node-build-test.yml` + `node-deadcode-slop.yml`

Together they cover the canonical script contract — lint, format:check,
typecheck, test:coverage, build, deadcode, slop:ci — against **one npm
package**. Built for a single-`package.json` repo (optionally not at the
repo root); neither fits an npm-workspaces monorepo with per-workspace jobs.

Split in two, not one, because the first started life bundled and broke the
moment a real consumer needed a Node-version matrix: matrixing a combined job
would re-run the (slower, version-independent) dead-code and slop scans once
per matrix leg for no reason.

```yaml
jobs:
  build-and-test:
    strategy:
      matrix:
        node-version: ["20.x", "22.x"]
    uses: bpulidodtx-7575/ci-workflows/.github/workflows/node-build-test.yml@v1
    with:
      working-directory: files # or "." for the repo root
      cache-dependency-path: files/package-lock.json
      node-version: ${{ matrix.node-version }}
      # run-build: true             (false if the package has no build step)

  quality:
    uses: bpulidodtx-7575/ci-workflows/.github/workflows/node-deadcode-slop.yml@v1
    with:
      working-directory: files
      cache-dependency-path: files/package-lock.json
      # run-deadcode: true
      # run-slop: true
      # upload-sarif: false         (true only on a public repo, or a
      #                              private one with GitHub Advanced
      #                              Security enabled)
```

Requires the consuming package.json to define: `lint`, `format:check`,
`typecheck`, `test:coverage`, `build` (unless `run-build: false`), `deadcode`,
`slop:ci` (unless `run-slop: false`).

### `docs-quality.yml`

markdownlint + lychee (link check) + an aggregate `summary` job, for a
docs-only repo or the docs tree of a code repo.

```yaml
jobs:
  docs:
    uses: bpulidodtx-7575/ci-workflows/.github/workflows/docs-quality.yml@v1
    # with:
    #   markdown-globs: "**/*.md"   (default)
```

Link-check tuning (excludes, retry/accept codes) lives in the consuming
repo's own `lychee.toml` at its root — auto-loaded, no input needed here.

### `security.yml`

actionlint + zizmor against the calling repo's own `.github/workflows/`, plus
a gitleaks secret scan over full history.

```yaml
jobs:
  security:
    uses: bpulidodtx-7575/ci-workflows/.github/workflows/security.yml@v1
    # with:
    #   zizmor-config-path: ""      (default: use the embedded policy —
    #                                 hash-pin third-party actions, ref-pin
    #                                 actions/* and github/*. Point this at
    #                                 a repo-local file only to diverge from
    #                                 that policy.)
```

Each of these is `workflow_call`-only — it has no `on: push` / `on:
pull_request` / `on: schedule` of its own. The calling repo owns its
triggers and just adds a job that `uses:` the file it wants.

## Versioning

Tagged `v1`. A breaking change (a renamed input, a removed default) bumps to
`v2`; existing callers pinned to `@v1` are unaffected. Never rewrite `v1` to
mean something different — that changes what every consumer's next PR runs
without their workflow file changing at all.

## Consumed by

- [`pt-tool`](https://github.com/bpulidodtx-7575/pt-tool) — the first
  consumer, validating the extraction against the workflow it was extracted
  from.
- More repos land here as they migrate off their hand-written CI.
