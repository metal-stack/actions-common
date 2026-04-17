# Common Github Actions

Provides [reusable workflows](https://docs.github.com/en/actions/how-tos/reuse-automations/reuse-workflows) for Github Actions used in the metal-stack organization.

## Usage

```yaml
---
name: metal-stack component release
on:
  pull_request:
    branches:
      - main
  release:
    types:
      - published
  push:
    branches:
      - main

jobs:
  <see-reusable-workflows-below>
```

For certain workflows, we expect to pass on secrets to the inherited workflow actions using `secrets: inherit`.

## Reusable Workflows

### Spelling `.github/workflows/spell-check.yaml`

Runs spell checking using [typos](https://github.com/crate-ci/typos).

```yaml
jobs:
  spell-check:
    uses: metal-stack/actions-common/.github/workflows/spell-check.yaml@v1
```

### Release Drafter `.github/workflows/release-drafter.yaml`

Please note that it does not seem possible with Github to provide a shared `release-drafter.yml` configuration for all repositories. It still makes sense to use the common action in order to manage the action's version from one location.

```yaml
jobs:
  draft:
    uses: metal-stack/actions-common/.github/workflows/release-drafter.yaml@v1
```

Note that you can use this action with all pipeline triggers. There is a skip-condition that ensures it only acts on push events on the default branch.

You do not need to provide a `release-drafter.yml` explicitly. A default is used from [.github](https://github.com/metal-stack/.github/blob/main/.github/release-drafter.yml).

### Go Build `.github/workflows/go-build.yaml`

Builds a Go binary and publishes it as a docker container image, including:

- Lints a Golang repository using [golangci-lint-action](https://github.com/golangci/golangci-lint-action).
- Tests the Golang repository.
- Image tag format for different release actions:
  - `release`: `<release-tag>`
  - `push`: `branch-<branch-name>`
  - `pull_request`: `pr-<pull-request-number>-<branch-name>`
- Embeds an SBOM into the resulting Docker image using Buildx.
- Signs the resulting container image using [cosign](https://github.com/sigstore/cosign).

We encourage using the `build-command` and then using a Dockerfile that just copies over the build's binaries instead of building the artifacts directly in the Dockerfile. The advantage is that CI caches can be used properly and build times are reduced drastically.

```yaml
jobs:
  go-build:
    uses: metal-stack/actions-common/.github/workflows/go-build.yaml@v1
    secrets: inherit
    with:
      lint: true
      test: true
      build: true
      test-command: go test ./... -coverprofile=coverage.out -covermode=atomic && go tool cover -func=coverage.out
      registry: ghcr.io
      registry-username: ${{ github.actor }}
      image-name: ${{ github.repository }}
      artifact-files: ""
```

### Release Assets `.github/workflows/release-assets.yaml`

Publishes files in a Github Release using [action-gh-release](https://github.com/softprops/action-gh-release). Only reacts on `release` pipeline triggers.

```yaml
jobs:
  release-assets:
    uses: metal-stack/actions-common/.github/workflows/release-assets.yaml@v1
    with:
      files: |
        bin/*
```

### Helm Chart `.github/workflows/helm-chart.yaml`

Publishes a helm chart as an OCI artifact and lints it.

Helm is more strict about semver in tag names. So, the tags look a bit different from how the docker tags look like:

- `release`: `<release-tag>`
- `push`: `v0.0.0-<branch-name>`
- `pull_request`: `v0.0.0-pr.<pull-request-number>`

```yaml
jobs:
  release-assets:
    uses: metal-stack/actions-common/.github/workflows/helm-chart.yaml@v1
    with:
      lint: true
      oci-release: true
```
