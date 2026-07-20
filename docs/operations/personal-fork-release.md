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

Custom pull requests should use **Rebase and merge** so the patch stack stays linear.

## Workflows

- `jm-ci.yml` verifies custom pull requests.
- `jm-upstream-rebase.yml` performs the guarded upstream rebase and dispatches a nightly release.
- `jm-release.yml` publishes macOS arm64, macOS x64, and Linux x64 artifacts to
  `jmederosalvarado/t3code` GitHub Releases.

Pushes and custom pull-request merges to `main` publish stable releases such as `1.0.42`, mark them
latest on GitHub, and update the Linux `latest` channel. Successful automated upstream rebases publish
nightly prereleases such as `1.0.43-nightly.20260720.43` and update the separate Linux `nightly`
channel. Either channel can also be dispatched manually from GitHub Actions.

The upstream CI, release, relay, and mobile workflows remain checked in to minimize rebase conflicts,
but are disabled in the personal fork's GitHub Actions settings.

## Desktop identity and updater

The fork source fixes the desktop identity as:

- application ID `com.jmederosalvarado.t3code`
- product name `T3 Code JM`
- macOS passkey signing unused

Linux AppImage builds use `T3CODE_DESKTOP_UPDATE_REPOSITORY=jmederosalvarado/t3code`, pointing
automatic updates at the fork through the upstream-supported build option. macOS builds intentionally
omit update metadata because unsigned macOS applications cannot install automatic updates.

## macOS installation and updates

The macOS artifacts are unsigned and unnotarized because this fork does not use a paid Apple Developer
Program account. GitHub Releases remain the download channel, but macOS upgrades are manual: download
the new DMG and replace the existing application.

On first launch, macOS Gatekeeper will identify the application as coming from an unidentified
developer. Only after verifying the release source, use **System Settings > Privacy & Security > Open
Anyway** to approve it. Linux AppImage releases continue to support in-app automatic updates.

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
