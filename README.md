# renovate-config

Shared [Renovate](https://docs.renovatebot.com/) presets for the `opendefensecloud` organization.

## Presets

| Preset | Reference | Use when |
|---|---|---|
| `default.json` | `github>opendefensecloud/renovate-config` | All repos |
| `go.json` | `github>opendefensecloud/renovate-config:go` | Go module repos |
| `k8s.json` | `github>opendefensecloud/renovate-config:k8s` | Repos using controller-runtime's envtest |

### `default.json`
- Extends `config:recommended`
- Conventional commit messages (`semanticCommits: enabled`)
- Pins Docker images and GitHub Actions to SHA digests (`pinDigests: true`) to prevent moving-tag attacks
- Tracks `opendefensecloud/dev-kit` releases in `Makefile` (`DEV_KIT_VERSION`)
- Tracks Go tool versions in `tools.lock`

### `go.json`
- Runs `go mod tidy` after updates
- Sets `rangeStrategy: bump` for Go module dependencies
- Groups Docker and `golang-version` updates for the Go toolchain into a single PR
- Tracks the Go version in `flake.nix`

### `k8s.json`
- Tracks the Kubernetes version used by `envtest` in `Makefile` (`ENVTEST_K8S_VERSION`)
- Groups all `k8s.io/**`, `sigs.k8s.io/controller-runtime`, `sigs.k8s.io/controller-tools`, `helm.sh/helm/v4`, `ocm.software/ocm`, and `ENVTEST_K8S_VERSION` updates into a single PR — they share k8s library dependencies and must use the exact same version; bumping any one without the others breaks the build

## Dependency Update Workflow

This config implements a "trust but verify" model that automates low-risk updates while keeping humans in the loop for anything that could introduce breaking changes.

| Update type | Behaviour | Rationale |
|---|---|---|
| **Patch, Digest** | Grouped into one PR, auto-merged once CI passes (patch updates after a 3-day stability window; digest updates immediately) | Low breaking-change risk; 3-day window on patches catches most supply-chain incidents before they hit us |
| **Minor** | Grouped into one PR, auto-merged once CI passes (after 3-day stability window) | Semver guarantees backwards compatibility; CI catches regressions |
| **Major** | Individual PRs, requires manual approval | Breaking changes expected; must be reviewed in isolation |
| **Security** | Individual PR, labeled `security`, requires manual approval, no stability window | Reviewer adds real value here — understanding CVE impact; no delay warranted for known vulnerabilities |

### Supply-chain security measures

- **Digest pinning** (`pinDigests: true`) — Docker images and GitHub Actions are pinned to immutable SHA-256 digests instead of mutable tags, preventing moving-tag attacks.
- **Stability window** (`minimumReleaseAge: 3 days` for patches and minor) — Renovate will not auto-merge a patch or minor update until the package version has been published for at least 3 days. This provides time for the community to detect and flag malicious releases before they land in the codebase. Digest updates are exempt (`minimumReleaseAge: null`) since a re-pinned digest often carries no meaningful release age.

### Requirements for auto-merge

Renovate auto-merge requires:
1. The Renovate GitHub App has **write** permission on the target repository.
2. Branch protection rules require **at least one passing status check** before merge (Renovate waits for all required checks).
3. `automergeType` defaults to `"pr"` — Renovate merges the PR after all required checks pass, not by force-pushing to the branch. If branch protection requires a manual review, Renovate must be exempted.

## Usage

Minimal setup for a Go controller repo:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "github>opendefensecloud/renovate-config",
    "github>opendefensecloud/renovate-config:go",
    "github>opendefensecloud/renovate-config:k8s"
  ]
}
```

Add repo-specific rules directly in `renovate.json` alongside `extends` — local config always takes precedence over presets.
