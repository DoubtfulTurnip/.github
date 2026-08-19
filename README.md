# .github

Shared, reusable GitHub Actions workflows for DoubtfulTurnip repos. This is a
public repo so any repo, public or private, can call these workflows with no
extra access configuration. The workflows themselves hold no secrets, callers
provide their own via `secrets:`.

## Workflows

### docker-build-ghcr.yml

Builds a Docker image and pushes it to GHCR (`ghcr.io/<owner>/<repo>`) on
pushes to the default branch and version tags. Skips the push on pull
requests (build-only, to catch breakages before merge). Tags follow:
`latest` (default branch only), branch name, PR number, semver, short SHA.

Usage in a caller repo, e.g. `.github/workflows/build.yml`:

```yaml
name: Build and push

on:
  push:
    branches: [main]
    tags: ["v*.*.*"]
  pull_request:
    branches: [main]

jobs:
  build:
    uses: DoubtfulTurnip/.github/.github/workflows/docker-build-ghcr.yml@main
    permissions:
      contents: read
      packages: write
```

For a multi-image repo (e.g. backend + frontend), call it once per image with
`image-suffix` and `context`/`dockerfile`:

```yaml
jobs:
  build-backend:
    uses: DoubtfulTurnip/.github/.github/workflows/docker-build-ghcr.yml@main
    with:
      context: ./backend
      dockerfile: ./backend/Dockerfile
      image-suffix: "-backend"
    permissions:
      contents: read
      packages: write

  build-frontend:
    uses: DoubtfulTurnip/.github/.github/workflows/docker-build-ghcr.yml@main
    with:
      context: ./frontend
      dockerfile: ./frontend/Dockerfile
      image-suffix: "-frontend"
    permissions:
      contents: read
      packages: write
```

### dependency-track-sbom.yml

Generates a CycloneDX SBOM with `cdxgen` and uploads it to the self-hosted
Dependency-Track instance. Runs on the `[self-hosted, dependency-track]`
runner, so the calling repo needs its own runner registered on Docker01
(see `docker-compose.runners.yml` there, one `github-runner-<repo>` service
per repo, all labelled `dependency-track`). Dependency-Track's own Trivy
analyzer (configured separately in DT's admin panel, talking to the
`trivy-server` container) handles vulnerability scanning against every SBOM
uploaded this way, so no separate Trivy step is needed in CI.

Usage:

```yaml
name: Dependency-Track SBOM

on:
  push:
    branches: [main]

jobs:
  sbom:
    uses: DoubtfulTurnip/.github/.github/workflows/dependency-track-sbom.yml@main
    secrets:
      dependencytrack-api-key: ${{ secrets.DEPENDENCYTRACK_API_KEY }}
```

The caller repo needs a `DEPENDENCYTRACK_API_KEY` repo secret with
`BOM_UPLOAD` permission in Dependency-Track.

### pr-agent.yml

Runs [The-PR-Agent/pr-agent](https://github.com/The-PR-Agent/pr-agent)
(auto review, auto describe, auto improve) on pull requests, using an
OpenRouter model rather than a direct OpenAI/Anthropic key. Only rolled out
to actively developed repos, not every repo in the account.

Usage:

```yaml
name: PR Agent

on:
  pull_request:
    types: [opened, reopened, ready_for_review, synchronize]
  issue_comment:

jobs:
  pr_agent:
    uses: DoubtfulTurnip/.github/.github/workflows/pr-agent.yml@main
    secrets:
      openrouter-key: ${{ secrets.OPENROUTER_API_KEY }}
```

The caller repo needs an `OPENROUTER_API_KEY` repo secret (personal GitHub
accounts don't support org-wide secrets, so this has to be set per repo).
Default model is `openrouter/anthropic/claude-3.5-sonnet` with
`openrouter/qwen/qwen3-coder:free` as a fallback; override via the `model`
and `fallback_model` inputs if needed.

## New repo checklist

1. Create the repo, add a `Dockerfile` if it builds a container.
2. Copy the two `uses:` blocks above into `.github/workflows/`.
3. Add a `DEPENDENCYTRACK_API_KEY` secret (same value works across repos,
   it's scoped to BOM_UPLOAD only).
4. Add a `github-runner-<repo>` service to `docker-compose.runners.yml` on
   Docker01, following the existing pattern, then
   `docker compose -f docker-compose.yml -f docker-compose.runners.yml up -d
   github-runner-<repo>`. Always pass both `-f` flags for this project, a
   single-file invocation will cause Compose to see it as a different
   project state and can recreate unrelated containers.
5. Add a `.github/dependabot.yml` (docker + github-actions ecosystems at
   minimum, plus pip/npm if applicable).

## Design choices

- **Public repo**: any caller can use these workflows with zero extra
  Actions access configuration, and there's nothing sensitive in workflow
  logic itself.
- **Trivy runs inside Dependency-Track, not CI**: scanning centrally against
  the SBOM avoids re-running Trivy per repo, keeps GitHub-hosted runner disk
  usage down, and gives one dashboard (DT's Vulnerability Audit view) instead
  of scattered per-repo results.
- **GHCR only, GITHUB_TOKEN auth**: no registry secrets to manage, scoped
  automatically to the calling repo's packages.
