# Personal Fork Release Operations

This repository is maintained as a linear custom patch stack on top of
`pingdotgg/t3code`.

## Branch model

- `upstream` records the exact upstream commit used as the base of the patch stack.
- `main` contains that upstream history followed by the fork-specific commits.
- `agent/*` branches contain custom changes before they are rebased and merged into `main`.

Do not merge upstream into `main`. The `JM Upstream Rebase` workflow fetches upstream daily, rebases
the commits in `upstream..main`, verifies the rebased tree, and atomically force-updates both branches
with explicit force-with-lease guards.

Custom pull requests should use **Squash and merge** so each change lands as one commit on the
linear patch stack.

## Workflows

- `jm-ci.yml` verifies custom pull requests.
- `jm-upstream-rebase.yml` polls upstream GitHub Releases every 15 minutes, rebases onto the exact
  release tag commit, and dispatches the matching fork release.
- `jm-release.yml` publishes macOS arm64 and macOS x64 artifacts to `jmederosalvarado/t3code`
  GitHub Releases.

Each upstream release after the recorded bootstrap checkpoint is mirrored in publication order. An
upstream nightly such as `v0.0.29-nightly.20260720.858` produces
`jm-v0.0.29-nightly.20260720.858`; a stable `v0.0.29` produces `jm-v0.0.29`. The fork release uses
the same version and prerelease status, but its commit contains the custom patch stack rebased onto the
exact upstream tag commit. Custom pull-request merges do not create independent releases.
The bootstrap checkpoint is upstream nightly `v0.0.29-nightly.20260720.857`, making nightly `858`
the first release eligible for mirroring.

The upstream CI, release, relay, and mobile workflows remain checked in to minimize rebase conflicts,
but are disabled in the personal fork's GitHub Actions settings.

## Desktop identity and releases

The fork source fixes the desktop identity as:

- application ID `com.jmederosalvarado.t3code`
- product name `T3 Code JM`
- macOS passkey signing unused

macOS builds intentionally omit automatic-update metadata because unsigned macOS applications cannot
install automatic updates. Stable and nightly builds are both downloaded manually from the fork's
GitHub Releases page.

## macOS installation and updates

The macOS artifacts are unsigned and unnotarized because this fork does not use a paid Apple Developer
Program account. GitHub Releases remain the download channel, but macOS upgrades are manual: download
the new DMG and replace the existing application.

On first launch, macOS Gatekeeper will identify the application as coming from an unidentified
developer. Only after verifying the release source, use **System Settings > Privacy & Security > Open
Anyway** to approve it.

## Manual conflict recovery

When the automated rebase reports a conflict:

```sh
git fetch origin main upstream
git fetch upstream main
git switch main
git reset --hard origin/main
git rebase --onto upstream/main origin/upstream
```

Resolve conflicts, continue the rebase, run the focused fork checks, and then update `main` and
`upstream` with explicit force-with-lease protection. Never force-push without first resolving the
expected remote SHAs.
