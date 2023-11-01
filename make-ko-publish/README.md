# Action

This action releases docker images build with [ko](https://github.com/ko-build/ko) based on a Makefile-target in your repository. This is an example workflow, which publishes your build images to a registry and signs them with cosign, generates a provenance and uploads it to a release.

```yaml
name: Publish images
permissions: {}

on:
  push:
    tags:
      - "v*"

jobs:
  publish-images:
    runs-on: ubuntu-latest
    permissions:
      packages: write
      id-token: write 
    outputs:
      digest: ${{ steps.publish.outputs.digest }}
    steps:
      - name: Checkout
        uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11 # v4.1.1
      - name: Run Trivy vulnerability (Repo)
        uses: aquasecurity/trivy-action@fbd16365eb88e12433951383f5e99bd901fc618f # v0.12.0
        with:
          scan-type: 'fs'
          ignore-unfixed: true
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
      - name: Install Cosign
        uses: sigstore/cosign-installer@11086d25041f77fe8fe7b9ea4e48e3b9192b8f19 # v3.1.2
      - name: Publish
        id: publish
        uses: peak-scale/github-actions/make-ko-publish@v0.1.0
        with:
          makefile-target: ko-publish
          registry: ghcr.io
          registry-username: ${{ github.actor }}
          registry-password: ${{ secrets.GITHUB_TOKEN }}
          repository: ${{ github.repository_owner }}
          version: ${{ github.ref_name }}
          sign-image: true
          sbom-name: artefact-name
          sbom-repository: ghcr.io/${{ github.repository_owner }}/sbom
          signature-repository: ghcr.io/${{ github.repository_owner }}/signatures
          main-path: ./
        env:
          REPOSITORY: ${{ github.repository }}
  generate-provenance:
    needs: publish-images
    permissions:
      id-token: write   # To sign the provenance.
      packages: write   # To upload assets to release.
      actions: read     # To read the workflow path.
    uses: slsa-framework/slsa-github-generator/.github/workflows/generator_container_slsa3.yml@v1.9.0
    with:
      image: ghcr.io/${{ github.repository_owner }}/artefact-name
      digest: "${{ needs.publish-images.outputs.digest }}"
      registry-username: ${{ github.actor }}
    secrets:
      registry-password: ${{ secrets.GITHUB_TOKEN }}
```

## Makefile

This is an example for `makefile-target` in your repository. It builds the image and publishes it to a registry.

```
# Version
GIT_HEAD_COMMIT ?= $(shell git rev-parse --short HEAD)
VERSION         ?= $(or $(shell git describe --abbrev=0 --tags --match "v*" 2>/dev/null),$(GIT_HEAD_COMMIT))

# Defaults
REGISTRY        ?= ghcr.io
REPOSITORY      ?= peakscale/example
IMG_BASE        ?= $(REPOSITORY)
IMG             ?= $(IMG_BASE):$(VERSION)
IMG_FULL        ?= $(REGISTRY)/$(IMG_BASE)

####################
# -- Docker
####################

KOCACHE         ?= /tmp/ko-cache
KO_TAGS         ?= "latest"
ifdef VERSION
KO_TAGS         := $(KO_TAGS),$(VERSION)
endif

# Docker Image Build
# ------------------

.PHONY: ko-build
ko-build: ko
	@echo Building Image $(IMG_BASE) - $(KO_TAGS) >&2
	@LD_FLAGS=$(LD_FLAGS) KOCACHE=$(KOCACHE) KO_DOCKER_REPO=$(IMG_FULL) \
		$(KO) build ./ --bare --tags=$(KO_TAGS) --push=false --local


# Docker Image Publish
# ------------------

REGISTRY_PASSWORD   ?= dummy
REGISTRY_USERNAME   ?= dummy

.PHONY: ko-login
ko-login: ko
	@$(KO) login $(REGISTRY) --username $(REGISTRY_USERNAME) --password $(REGISTRY_PASSWORD)

ko-publish: ko-login
	@LD_FLAGS=$(LD_FLAGS) KOCACHE=$(KOCACHE) KO_DOCKER_REPO=$(IMG_FULL) \
		$(KO) build ./ --bare --tags=$(KO_TAGS)

####################
# -- Helpers
####################
KO = $(shell pwd)/bin/ko
KO_VERSION = v0.14.1
ko:
	$(call go-install-tool,$(KO),github.com/google/ko@$(KO_VERSION))

# go-install-tool will 'go install' any package $2 and install it to $1.
PROJECT_DIR := $(shell dirname $(abspath $(lastword $(MAKEFILE_LIST))))
define go-install-tool
@[ -f $(1) ] || { \
set -e ;\
GOBIN=$(PROJECT_DIR)/bin go install $(2) ;\
}
endef
```

With this makefile you can locally create builds 

```bash
make ko-build
```

and publish the images

```bash
make ko-publish
```