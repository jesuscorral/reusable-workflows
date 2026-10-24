# reusable-workflows

Shared GitHub Actions workflows, called from other repositories with `workflow_call`. Each one does a single, clearly named job, so a project's pipeline reads as a list of stages:

```yaml
jobs:
  quality:
    uses: jesuscorral/reusable-workflows/.github/workflows/node-quality-gate.yml@<sha> # v1.1.0
  deploy:
    needs: quality
    uses: jesuscorral/reusable-workflows/.github/workflows/firebase-hosting-deploy-live.yml@<sha> # v1.1.0
```

[![Lint workflows](https://github.com/jesuscorral/reusable-workflows/actions/workflows/lint-workflows.yml/badge.svg)](https://github.com/jesuscorral/reusable-workflows/actions/workflows/lint-workflows.yml)

---

## Catalog

### Web delivery (build once, test, preview, release)

| Workflow | What it does | Needs from the caller |
|---|---|---|
| [`node-quality-gate.yml`](.github/workflows/node-quality-gate.yml) | `npm ci`, dependency audit, lint, unit tests, build. Uploads the **build** and **coverage** artifacts. | `.nvmrc`, npm scripts |
| [`node-playwright-e2e.yml`](.github/workflows/node-playwright-e2e.yml) | Playwright E2E tests against the build artifact. Uploads the report on failure. | Playwright config |
| [`web-lighthouse-audit.yml`](.github/workflows/web-lighthouse-audit.yml) | Lighthouse CI on the static build. Always uploads the reports. | `lighthouserc.json` |
| [`firebase-hosting-deploy-preview.yml`](.github/workflows/firebase-hosting-deploy-preview.yml) | Deploys the build to a temporary Firebase preview channel and comments the URL on the PR. | `firebase.json`, secret `FIREBASE_SERVICE_ACCOUNT` |
| [`firebase-hosting-deploy-live.yml`](.github/workflows/firebase-hosting-deploy-live.yml) | Deploys the build to the Firebase **live** channel (production), recorded under a GitHub environment. | `firebase.json`, secret `FIREBASE_SERVICE_ACCOUNT` |
| [`sonarcloud-analysis.yml`](.github/workflows/sonarcloud-analysis.yml) | SonarCloud scan with the coverage artifact. Skipped with a warning if no token. | `sonar-project.properties`, secret `SONAR_TOKEN` |
| [`http-smoke-test.yml`](.github/workflows/http-smoke-test.yml) | Requests a list of paths on a live site, with retries; optional expected text. | — |

The artifacts link the stages: `node-quality-gate` builds once and every later workflow downloads that same `dist` artifact, so what gets deployed is exactly what passed the tests.

### Docker images and repository housekeeping (older workflows)

| Workflow | What it does |
|---|---|
| [`build-push-docker-image.yml`](.github/workflows/build-push-docker-image.yml) | Builds a Docker image and pushes it to Docker Hub with a version tag and `latest`. |
| [`get-latest-version-docker-image.yml`](.github/workflows/get-latest-version-docker-image.yml) | Returns the highest semver tag of a Docker Hub repository. |
| [`increase-version.yml`](.github/workflows/increase-version.yml) | Bumps the patch number of a version. |
| [`setup-labels.yml`](.github/workflows/setup-labels.yml) | Creates or updates the standard issue labels. |

These keep their original names and `snake_case` inputs so existing callers don't break. `add-issue-labels.yml` and `update-default-branch.yml` are regular workflows of this repository, not reusable.

---

## Inputs, secrets and outputs

Every input has a description and, where possible, a default, so the minimal call is short and every option is visible in the workflow file. The most common ones:

| Input | Workflows | Default | Meaning |
|---|---|---|---|
| `node-version-file` | node-*, web-lighthouse | `.nvmrc` | Node.js version source |
| `build-artifact-name` / `build-path` | all web delivery | `dist` / `dist` | Artifact that carries the build between jobs, and where it is restored |
| `*-command` | node-* | the usual `npm` command | Command for each step. `''` skips audit, lint or tests |
| `firebase-project-id` | firebase-* | **required** | Firebase project |
| `firebase-tools-version` | firebase-* | `15.33.0` | Pinned Firebase CLI. 15.34+ also needs the *Service Usage Consumer* role on the deploy service account |
| `environment-name` | firebase-* | `preview` / `production` | GitHub environment of the deployment (its protection rules apply) |
| `timeout-minutes` | all | 5–20 | Job timeout |

Secrets are never inherited: the caller passes them explicitly, by the names declared in each workflow (`FIREBASE_SERVICE_ACCOUNT`, `SONAR_TOKEN`).

Outputs:
- `node-quality-gate`: `build-artifact-name`, `coverage-artifact-name`
- `firebase-hosting-deploy-preview`: `preview-url`, `expire-time`

---

## Example: pull request and production pipelines

```yaml
# .github/workflows/pull-request.yml
name: PR · Validate & preview
on:
  pull_request:
    branches: [master]
permissions:
  contents: read
jobs:
  quality:
    name: 1 · Quality gate
    uses: jesuscorral/reusable-workflows/.github/workflows/node-quality-gate.yml@<sha> # v1.1.0

  e2e:
    name: 2 · E2E tests
    needs: quality
    uses: jesuscorral/reusable-workflows/.github/workflows/node-playwright-e2e.yml@<sha> # v1.1.0

  preview:
    name: 3 · Deploy preview
    needs: [quality, e2e]
    if: github.event.pull_request.head.repo.fork == false
    permissions:
      contents: read
      checks: write
      pull-requests: write
    uses: jesuscorral/reusable-workflows/.github/workflows/firebase-hosting-deploy-preview.yml@<sha> # v1.1.0
    with:
      firebase-project-id: my-project
    secrets:
      FIREBASE_SERVICE_ACCOUNT: ${{ secrets.FIREBASE_SERVICE_ACCOUNT_MY_PROJECT }}
```

```yaml
# .github/workflows/deploy-production.yml
name: Production · Deploy
on:
  push:
    branches: [master]
permissions:
  contents: read
concurrency:
  group: production-deploy
  cancel-in-progress: false
jobs:
  quality:
    name: 1 · Quality gate
    uses: jesuscorral/reusable-workflows/.github/workflows/node-quality-gate.yml@<sha> # v1.1.0

  deploy:
    name: 2 · Deploy to production
    needs: quality
    uses: jesuscorral/reusable-workflows/.github/workflows/firebase-hosting-deploy-live.yml@<sha> # v1.1.0
    with:
      firebase-project-id: my-project
      site-url: https://www.example.com/
    secrets:
      FIREBASE_SERVICE_ACCOUNT: ${{ secrets.FIREBASE_SERVICE_ACCOUNT_MY_PROJECT }}

  smoke:
    name: 3 · Smoke test
    needs: deploy
    uses: jesuscorral/reusable-workflows/.github/workflows/http-smoke-test.yml@<sha> # v1.1.0
    with:
      base-url: https://www.example.com
      paths: / /about
```

Real-world caller: [Concurso.homebrewer.cordoba](https://github.com/jesuscorral/Concurso.homebrewer.cordoba/tree/master/.github/workflows).

---

## Conventions

**File names**: `<platform-or-tool>-<what-it-does>.yml`, kebab-case, so the file says where it runs and what it does: `node-quality-gate`, `firebase-hosting-deploy-live`, `http-smoke-test`.

**Display names**: `<Platform> · <Action>`, for example `Firebase Hosting · Deploy live channel`. The caller's job name adds the stage number (`3 · Deploy preview`), so the GitHub UI reads `3 · Deploy preview / Deploy preview channel`.

**Inputs and outputs**: kebab-case, descriptive (`build-artifact-name`, not `artifact`). Every input has a `description`; required only when there is no sensible default.

**Secrets**: `UPPER_SNAKE_CASE`, declared in `on.workflow_call.secrets` and passed explicitly by the caller. Never `secrets: inherit`.

**Security**:
- Third-party actions pinned to a full commit SHA, with the version in a comment.
- `permissions: contents: read` by default; anything else is granted per job and documented in the header.
- `actions/checkout` with `persist-credentials: false`.
- Commands and values coming from inputs reach the shell through `env:` variables, never by interpolating `${{ inputs.* }}` inside `run:` scripts.

**Header comment**: every workflow starts with what it does, what it expects from the caller and any permission it needs.

---

## Versioning

Releases are git tags following [semver](https://semver.org/): `vMAJOR.MINOR.PATCH`.

- **PATCH**: fixes, no change in inputs or behavior callers rely on.
- **MINOR**: new workflows or new optional inputs.
- **MAJOR**: renamed/removed inputs, new required inputs, changed defaults that alter behavior.

Callers should pin the **commit SHA of a tag** with the tag as a comment (`@<sha> # v1.1.0`). Dependabot (`package-ecosystem: github-actions`) recognises that format and opens a PR in each caller when a new tag is published. Changes are listed in [CHANGELOG.md](CHANGELOG.md).

Release steps:
1. Merge the change to `main` through a PR (the `Lint workflows` check must pass).
2. Add the entry to `CHANGELOG.md`.
3. Tag the merge commit and push the tag: `git tag v1.2.0 && git push origin v1.2.0`.
4. Create a GitHub release from the tag (optional, it shows the notes).

To try a change before releasing it, point a caller's PR at the branch commit (`@<sha-of-the-branch>`) and switch it to the tag once released.

---

## Adding a workflow

1. Create `.github/workflows/<platform>-<action>.yml` with `on: workflow_call`, following the conventions above.
2. Add it to the file list in [`lint-workflows.yml`](.github/workflows/lint-workflows.yml).
3. Add it to the catalog and, if it has new common inputs, to the inputs table.
4. Release a new MINOR version.
