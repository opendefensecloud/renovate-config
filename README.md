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
- Tracks `opendefensecloud/dev-kit` releases in `Makefile` (`DEV_KIT_VERSION`) and in the `flake.nix` input, in the form `github:opendefensecloud/dev-kit/vX.Y.Z` only (an unpinned `github:opendefensecloud/dev-kit` or a `?ref=` form is not tracked). All dev-kit pins resolve through `github-tags`, the datasource GitHub Actions pins use, so one PR never mixes versions
- Groups every dev-kit pin (`Makefile`, `flake.nix`, GitHub Actions and reusable workflows) into one `dev-kit` PR that is never automerged. Renovate bumps the flake input with a regex manager, which can't touch `flake.lock`, and the nix manager only re-locks inputs pinned by `rev`, not by tag. So the PR needs one human commit: `nix flake update dev-kit`. Until then `nix develop` rewrites the lock in CI, and jobs that check for a clean working tree (dev-kit's `diff-check`) fail. The PR body says so
- Tracks Go tool versions in `tools.lock`
- Adds an `automerge` label to digest and patch PRs (the signal the auto-approve workflow reacts to); minor, major, security and dev-kit PRs never get it
- Refreshes `flake.lock` (`nix flake update`) in a weekly lock file maintenance PR (nix manager, opt-in beta). Repos without a `flake.lock` are unaffected. A human merges it: flake inputs like nixpkgs-unstable and go-overlay have no releases a stability window could apply to, and an input without a tag (e.g. an unpinned dev-kit) moves to the tip of its branch

### `go.json`
- Runs `go mod tidy` after updates
- Sets `rangeStrategy: bump` for Go module dependencies
- Groups Docker and `golang-version` updates for the Go toolchain into a single PR
- Tracks the Go version in `flake.nix`
- Relies on the weekly `flake.lock` refresh from `default.json` so the go-overlay input knows the Go versions `flake.nix` asks for. Renovate never groups lock file maintenance with other updates, so a Go bump and its lock refresh are separate PRs. The flake-based CI is the gate: a Go bump whose version the locked go-overlay lacks fails CI and waits until someone merges the lock refresh. go-overlay adds a Go release within hours, well inside the 3-day stability window. A patch bump automerges, so Renovate rebases it once the refresh is on the base branch and CI turns green on its own; a minor bump (e.g. 1.26 → 1.27) doesn't automerge, so it stays red until someone rebases it (tick the rebase box in the PR)

### `k8s.json`
- Tracks the Kubernetes version used by `envtest` in `Makefile` (`ENVTEST_K8S_VERSION`)
- Groups all `k8s.io/**`, `sigs.k8s.io/controller-runtime`, `sigs.k8s.io/controller-tools`, `helm.sh/helm/v4`, `ocm.software/ocm`, and `ENVTEST_K8S_VERSION` updates into a single PR — they share k8s library dependencies and must use the exact same version; bumping any one without the others breaks the build

## Dependency Update Workflow

This config implements a "trust but verify" model that automates low-risk updates while keeping humans in the loop for anything that could introduce breaking changes.

| Update type | Behaviour | Rationale |
|---|---|---|
| **Patch, Digest** | Grouped into one PR, auto-merged once CI passes (patch updates after a 3-day stability window; digest updates immediately) | Low breaking-change risk; 3-day window on patches catches most supply-chain incidents before they hit us |
| **Minor** | Grouped into one PR, requires manual approval (after 3-day stability window) | Semver promises backwards compatibility, but minors still ship behaviour changes in practice; a human look is cheap on one weekly grouped PR |
| **Major** | Individual PRs, requires manual approval (after a 10-day stability window) | Breaking changes expected; must be reviewed in isolation; longer window since major releases are more prone to early regressions |
| **Security** | Individual PR, labeled `security`, requires manual approval, no stability window | Reviewer adds real value here — understanding CVE impact; no delay warranted for known vulnerabilities |

### Supply-chain security measures

- **Digest pinning** (`pinDigests: true`) — Docker images and GitHub Actions are pinned to immutable SHA-256 digests instead of mutable tags, preventing moving-tag attacks.
- **Stability window** (`minimumReleaseAge: 3 days` for patches and minor, `10 days` for major) — Renovate will not merge an update until the package version has been published for at least this long. This provides time for the community to detect and flag malicious releases before they land in the codebase. Major updates use a longer 10-day window since they carry more risk and are more prone to early regressions. Digest updates are exempt (`minimumReleaseAge: null`) since a re-pinned digest often carries no meaningful release age. Security updates (vulnerability alerts) bypass the stability window entirely. Their behaviour lives in the `vulnerabilityAlerts` object (`minimumReleaseAge: null`, `automerge: false`, `security` label), which Renovate applies on top of the normal rules for vulnerability-fix PRs. No delay is warranted for known vulnerabilities, and these PRs still need a human.

### Requirements for auto-merge

Renovate auto-merge requires:
1. The Renovate GitHub App has **write** permission on the target repository.
2. Branch protection rules require **at least one passing status check** before merge (Renovate waits for all required checks).
3. `automergeType` defaults to `"pr"` — Renovate merges the PR after all required checks pass, not by force-pushing to the branch. If branch protection requires a manual review, Renovate must be exempted.

## Enabling automerge in a consuming repo

The presets set `automerge: true` for digest and patch updates, but Renovate cannot approve its own PRs. Two things unblock it per repo.

### 1. Call the auto-approve workflow

The reusable workflow lives in [`opendefensecloud/dev-kit`](https://github.com/opendefensecloud/dev-kit). Repos that copy dev-kit's `example/` already ship this caller. Otherwise add `.github/workflows/renovate-auto-approve.yml`:

```yaml
name: renovate-auto-approve
on:
  pull_request:
    types: [opened, reopened, synchronize, labeled, unlabeled]
permissions:
  pull-requests: write
jobs:
  approve:
    uses: opendefensecloud/dev-kit/.github/workflows/renovate-auto-approve.yml@<sha-or-tag>
    permissions:
      pull-requests: write
```

Pin `@<sha-or-tag>` to a released tag or commit SHA, same discipline we apply to actions and images. The workflow approves only PRs carrying the `automerge` label and skips any PR labeled `security`, so digest/patch updates get approved automatically while minor, major, and security updates wait for a human. If a PR gains the `security` label after it was auto-approved, the workflow revokes its approval (via a `request-changes` review) so a human is required again. It triggers on the `labeled` and `unlabeled` events for this reason (so adding or removing `security` re-evaluates the PR), so keep those in the caller's `pull_request` types.

### 2. Configure GitHub settings

- **Settings -> Actions -> General -> Workflow permissions:** enable ["Allow GitHub Actions to create and approve pull requests"](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/enabling-features-for-your-repository/managing-github-actions-settings-for-a-repository#preventing-github-actions-from-creating-or-approving-pull-requests). Without this, the workflow's approval is rejected.
- **Branch ruleset on the default branch:**
  - Require the CI / e2e status checks to pass. This is the gate that replaces the human for these update types.
  - Require 1 approving review, which the workflow provides for eligible PRs.
  - Give the Renovate app write access and allow auto-merge, so the PR merges once checks pass and the approval is in.

The approval is authored by `github-actions[bot]`, so a `Require review from Code Owners` rule wont be satisfied by it. If a repo needs that, run the approval step with a dedicated GitHub App or PAT token instead.

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
