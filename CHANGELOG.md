# Changelog

All notable changes to the reusable workflows. Versions follow [semver](https://semver.org/); see [README.md](README.md#versioning).

## v1.1.0

Web delivery workflows, extracted from the Concurso.homebrewer.cordoba pipelines:

- `node-quality-gate.yml`: audit, lint, unit tests and build; uploads the build and coverage artifacts.
- `node-playwright-e2e.yml`: Playwright E2E tests against the build artifact.
- `web-lighthouse-audit.yml`: Lighthouse CI audit of the static build.
- `firebase-hosting-deploy-preview.yml`: deploy to a Firebase Hosting preview channel and comment the URL on the PR.
- `firebase-hosting-deploy-live.yml`: deploy to the Firebase Hosting live channel (production).
- `sonarcloud-analysis.yml`: SonarCloud scan with coverage.
- `http-smoke-test.yml`: check that a list of pages responds after a deploy.

Repository:
- `lint-workflows.yml`: actionlint on every PR and push to `main`.
- Dependabot keeps the pinned actions up to date.
- README: catalog, conventions, versioning and release steps.

No changes to the existing Docker and label workflows.

## v1.0.2

- `build-push-docker-image.yml`, `get-latest-version-docker-image.yml`: fix secrets parameters.

## v1.0.1

- Remove unused parameter.

## v.1.0.0

- First release: Docker image build/push, latest Docker Hub version, version increase.
