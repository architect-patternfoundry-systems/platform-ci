# Platform CI — Organization-Level Reusable Workflows

This repository owns the centrally versioned, security-sensitive reusable
GitHub Actions workflows for the PatternFoundry organization.

## Purpose

Application repositories (ZeroTrace, application, storyloom, etc.) call
these workflows via `workflow_call` with repo-specific inputs. This
avoids copying provenance verification, registry resolution, immutability
checks, and manifest mutation logic into each repository.

## Workflows

### `secure-build.yml`

Builds, publishes, verifies, and optionally deploys a container image.

**Responsibilities:**
1. Validate trusted event/ref (protected branches or manual dispatch only)
2. Resolve registry endpoints (logical hostname, ClusterIP, NodePort)
3. Generate immutable tag (`YYYYMMDDHHMMSS-<7hex>`)
4. Build candidate image with BuildKit
5. Run runtime import/smoke check inside the built container
6. Verify tag immutability before and after push
7. Obtain OCI digest
8. Validate OCI labels (revision, source, created, base image lineage)
9. Generate provenance artifact
10. Optionally mutate a digest-pinned deployment manifest using `yq`
11. Upload provenance artifact

**Does NOT:**
- Push `:latest` or `:main` tags (mutable tags are prohibited)
- Run on pull requests from forks
- Mutate access policies, service accounts, or RBAC
- Use `sed` for manifest updates (uses `yq` for structured YAML mutation)

## Pinning

Caller workflows MUST pin to an immutable commit SHA:

```yaml
uses: architect-patternfoundry-systems/platform-ci/.github/workflows/secure-build.yml@<pinned-sha>
```

After the initial commit, update the pin in all caller workflows to
the actual commit SHA of this repository's first release.

## Runner requirements

- `self-hosted` runner with:
  - `kubectl` access to the cluster (for registry ClusterIP resolution)
  - Docker with BuildKit support
  - `yq` installed (for manifest mutation)
  - `jq` installed (for provenance artifact generation)
  - NodePort 30500 access (for local registry push)

## Secrets

Caller workflows must provide:

| Secret | Purpose |
|---|---|
| `INFRA_SSH_KEY` | SSH key for cloning/pushing manifest repos (if mutate_manifest is true) |
| `INTERNAL_PYPI_URL` | Internal PyPI index URL for build-time pip install |
