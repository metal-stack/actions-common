# Common Github Actions

Provides common Github Actions used in the metal-stack organization.

## Usage

We usually expect the following pipeline triggers:

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
```

Usually, we also expect to pass on secrets to the inherited workflow actions using `

## Workflows

### Generic

#### Release Drafter `.github/workflows/release-drafter.yaml`

Please note that it does not seem possible with Github to provide a shared `release-drafter.yml` configuration for all repositories. It still makes sense to use the common action in order to manage the version from one location.

```yaml
jobs:
  draft:
    uses: metal-stack/actions-common/.github/workflows/release-drafter.yaml@main
```

### Go

#### Go Lint `.github/workflows/go-lint.yaml`

Lints a Golang repository using [golangci-lint-action](https://github.com/golangci/golangci-lint-action).

```yaml
jobs:
  lint:
    uses: metal-stack/actions-common/.github/workflows/go-lint.yaml@main
```

#### Go Build `.github/workflows/go-build.yaml`

Builds a Go binary and publishes it as a docker container image, including:

- Image tag format for different release actions
- Embedding SBOM using Buildx
- Signing the container image using [cosign](https://github.com/sigstore/cosign)

```yaml
jobs:
  build:
    uses: metal-stack/actions-common/.github/workflows/go-build.yaml@main
    secrets: inherit
    with:
      build-command: make
      registry: ghcr.io
      registry_username: ${{ github.actor }}
```
