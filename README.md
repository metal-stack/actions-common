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

## Image tag formats

Container images and OCI artifacts published by the workflows below are tagged based on the event that triggered the workflow:

| Event           | Tag                                      |
| --------------- | ---------------------------------------- |
| `release`       | `<release-tag>`                          |
| `push` (branch) | `branch-<branch-name>`                   |
| `push` (tag)    | `<tag-name>`                             |
| `pull_request`  | `pr-<pull-request-number>-<branch-name>` |

Helm charts are stricter about semver tag names, so they use a different format:

| Event           | Tag                               |
| --------------- | --------------------------------- |
| `release`       | `<release-tag>`                   |
| `push` (branch) | `v0.0.0-<branch-name>`            |
| `push` (tag)    | `<tag-name>`                      |
| `pull_request`  | `v0.0.0-pr.<pull-request-number>` |

Some actions also publish on folders (like GCP bucket push), here we have the following push locations:

| Event           | Tag                                                             |
| --------------- | --------------------------------------------------------------- |
| `release`       | `<release-tag>`                                                 |
| `push` (branch) | `stable` for default branches, otherwise `branch-<branch-name>` |
| `push` (tag)    | `<tag-name>`                                                    |
| `pull_request`  | `pull_requests/pr-<pull-request-number>-<branch-name>`          |

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
- Embeds an SBOM into the resulting Docker image using Buildx.
- Signs the resulting container image using [cosign](https://github.com/sigstore/cosign).

The image is tagged according to the [image tag formats](#image-tag-formats).

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
      context: .
      dockerfile: Dockerfile
      artifact-files: ""
```

### Container Build `.github/workflows/container-build.yaml`

Builds a container and publishes it as a docker container image, including:

- Embeds an SBOM into the resulting Docker image using Buildx.
- Signs the resulting container image using [cosign](https://github.com/sigstore/cosign).

The image is tagged according to the [image tag formats](#image-tag-formats).

```yaml
jobs:
  go-build:
    uses: metal-stack/actions-common/.github/workflows/container-build.yaml@v1
    secrets: inherit
    with:
      registry: ghcr.io
      registry-username: ${{ github.actor }}
      image-name: ${{ github.repository }}
      context: .
      dockerfile: Dockerfile
      artifact-files: ""
```

### Release Assets `.github/workflows/release-assets.yaml`

Publishes files in a Github Release using [action-gh-release](https://github.com/softprops/action-gh-release). Only reacts on `release` pipeline triggers.

The workflow generates an SBOM for the artifacts and puts it into a file called `sbom.spdx.json`.

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

The image is tagged according to the [image tag formats](#image-tag-formats).

```yaml
jobs:
  release-assets:
    uses: metal-stack/actions-common/.github/workflows/helm-chart.yaml@v1
    with:
      lint: true
      oci-release: true
```

### OCI Release Vector Artifact `.github/workflows/oci-release-vector-artifact.yaml`

Publishes an OCI release vector artifact as specified in [OCI Artifacts | metal-stack.io](https://metal-stack.io/community/oci-artifacts).

The image is tagged according to the [image tag formats](#image-tag-formats).

```yaml
jobs:
  release-assets:
    uses: metal-stack/actions-common/.github/workflows/oci-release-vector-artifact.yaml@v1
    with:
      registry: ghcr.io
      registry-username: ${{ github.actor }}
      image-name: ${{ github.repository }}
```

### GCP Bucket Push `.github/workflows/gcp-bucket-push.yaml`

Pushes artifacts to a Google Bucket. This action requires a `GCP_SA_KEY` credentials file to be passed as a secret.

The workflow generates an SBOM for the artifacts and puts it into a file called `sbom.spdx.json`.

The folder is named according to the [image tag formats](#image-tag-formats).

```yaml
jobs:
  release-assets:
    uses: metal-stack/actions-common/.github/workflows/gcp-bucket-push.yaml@v1
    with:
      artifact-ids:
      bucket:
      path-prefix: ${{ github.event.repository.name }}
```
