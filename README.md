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
- No hourly limit on new PRs (`prHourlyLimit: 0`): a scheduled window often gets a single Renovate run, and with Renovate's default of 2, unscheduled PRs (major, security, dev-kit, flake inputs) can take both slots and push a scheduled PR to the next week. The default limit of 10 open PRs per repo still applies
- Pins Docker images and GitHub Actions to SHA digests (`pinDigests: true`) to prevent moving-tag attacks
- Writes the exact version next to action and reusable-workflow pins whose upstream has `vX.Y.Z` tags (`# v4.7.0`, not `# v4`, via `helpers:pinGitHubActionDigestsToSemver`), so a release updates the comment together with the SHA, as dev-kit's `update-action-pins` check requires. Actions without such tags keep their comment, and a re-pointed tag still moves the SHA alone; the check flags that on purpose
- Tracks `opendefensecloud/dev-kit` tags in `Makefile` (`DEV_KIT_VERSION`) and in the `flake.nix` input (`github:opendefensecloud/dev-kit/vX.Y.Z` only), through `github-tags` like the GitHub Actions pins
- Groups every dev-kit pin (`Makefile`, `flake.nix`, GitHub Actions and reusable workflows) into one `dev-kit` PR (see [Nix flakes](#nix-flakes))
- Tracks Go tool versions in `tools.lock`
- Refreshes `flake.lock` in a lock file maintenance PR before 6am (UTC) on Wednesdays (see [Nix flakes](#nix-flakes))

### `go.json`
- Runs `go mod tidy` after updates
- Sets `rangeStrategy: bump` for Go module dependencies
- Groups Docker and `golang-version` updates for the Go toolchain into a single PR, with no stability window for the `golang` image: Docker Hub dates it by its last re-push, so the window would hold the Dockerfile back while `go.mod` and `flake.nix` move
- Tracks the Go version in `flake.nix`
- Relies on dev-kit's `renovate-dev-kit-lock` workflow to relock `go-overlay` on the Go PR, so it knows the Go version `flake.nix` asks for (see [Nix flakes](#nix-flakes))

### `k8s.json`
- Adds an `ok-to-helm` label to version update and lock file maintenance PRs so chart-publish CI runs
- Tracks the Kubernetes version used by `envtest` in `Makefile` (`ENVTEST_K8S_VERSION`)
- Groups all `k8s.io/**`, `sigs.k8s.io/controller-runtime`, `sigs.k8s.io/controller-tools`, `helm.sh/helm/v4`, and `ENVTEST_K8S_VERSION` updates into a single PR — they share k8s library dependencies and must use the exact same version; bumping any one without the others breaks the build
- Never proposes a `sigs.k8s.io/structured-merge-diff` major on its own: the major follows `k8s.io/apimachinery` and generated apply configurations, and arrives with the Kubernetes group bump

## Dependency Update Workflow

Every update PR is merged by a human. Low-risk updates are grouped into one weekly PR to keep that cheap.

| Update type | Behaviour | Rationale |
|---|---|---|
| **Patch, Digest** | Grouped into one weekly PR, merged by a human (patch updates after a 3-day stability window; digest updates immediately) | Low breaking-change risk; 3-day window on patches catches most supply-chain incidents before they hit us |
| **Minor** | Grouped into one PR, requires manual approval (after 3-day stability window) | Semver promises backwards compatibility, but minors still ship behaviour changes in practice; a human look is cheap on one weekly grouped PR |
| **Major** | Individual PRs, requires manual approval (after a 10-day stability window) | Breaking changes expected; must be reviewed in isolation; longer window since major releases are more prone to early regressions |
| **Security** | Individual PR, labeled `security`, requires manual approval, no stability window | Reviewer adds real value here — understanding CVE impact; no delay warranted for known vulnerabilities |
| **Lock file maintenance** | Own weekly PR on Wednesday, labeled `ok-to-image` and `ok-to-e2e` (plus `ok-to-helm` with `k8s.json`) | Flake inputs have no releases a stability window could apply to; `flake.lock` pins the toolchain that nix-based CI runs, so the refresh gets the full CI run |

### Nix flakes

`default.json` enables Renovate's nix manager (opt-in beta). Repos without a `flake.lock` are unaffected.

- **Lock file maintenance** runs `nix flake update` in its own PR on Wednesdays, two days after the Monday groups. By then the patch group is usually merged, so a repo that requires up-to-date branches doesn't rebase the refresh onto that merge (each push reruns image builds and e2e). A midweek merge that touches `flake.lock`, such as a minor Go bump, still conflicts and triggers a rebase. Flake inputs like `nixpkgs-unstable` and `go-overlay` have no releases a stability window could apply to, and an input without a tag (e.g. an unpinned dev-kit) moves to the tip of its branch.
- **Inputs pinned by `rev`** (`?rev=<sha>`) get their own update PR for every upstream commit.
- **dev-kit bumps** change the flake input through a regex manager, which can't touch `flake.lock`, and the nix manager only re-locks inputs pinned by `rev`, not by tag. So the `dev-kit` PR needs one human commit, `nix flake update dev-kit`, and its PR body says so. Until then `nix develop` rewrites the lock in CI, and jobs that check for a clean working tree (dev-kit's `diff-check`) fail. After a human's commit Renovate treats the branch as edited and stops rebasing or updating it; ticking the rebase box recreates it and drops the lock commit. Commits by `dev-kit-bot[bot]` don't count: `gitIgnoredAuthors` lists the org app that dev-kit's `renovate-dev-kit-lock` workflow commits as, so once a repo calls that workflow, Renovate keeps rebasing.
- **Go bumps relock `go-overlay` in the same PR**, through dev-kit's `renovate-dev-kit-lock` workflow (v3.1.0+): on `renovate/go-version` it runs `nix flake update go-overlay` and commits the lock as `dev-kit-bot[bot]`, so Renovate keeps rebasing the branch. `go-overlay` adds a Go release up to a day after it ships, inside the 3-day stability window; if it hasn't yet, the relock job fails and can be re-run. Repos without that workflow fall back to the weekly lock refresh: the Go bump stays red until it is merged and the bump rebased.

### Supply-chain security measures

- **Digest pinning** (`pinDigests: true`) — Docker images and GitHub Actions are pinned to immutable SHA-256 digests instead of mutable tags, preventing moving-tag attacks. Flake inputs are the exception (`nix.pinDigests: false`): `flake.lock` already pins every input to a commit, so pinning `nixpkgs` in `flake.nix` as well adds nothing.
- **Stability window** (`minimumReleaseAge: 3 days` for patches and minor, `10 days` for major) — Renovate holds back an update until the package version has been published for at least this long. This provides time for the community to detect and flag malicious releases before they land in the codebase. Major updates use a longer 10-day window since they carry more risk and are more prone to early regressions. Digest updates have no stability window since a re-pinned digest often carries no meaningful release age. Security updates (vulnerability alerts) bypass the stability window entirely. Their behaviour lives in the `vulnerabilityAlerts` object (`minimumReleaseAge: null`, `security` label), which Renovate applies on top of the normal rules for vulnerability-fix PRs. No delay is warranted for known vulnerabilities.

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
