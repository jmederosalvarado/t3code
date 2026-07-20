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
- `jm-upstream-rebase.yml` performs the guarded upstream rebase and dispatches a release.
- `jm-release.yml` publishes macOS arm64, macOS x64, and Linux x64 artifacts to
  `jmederosalvarado/t3code` GitHub Releases.

The upstream CI, release, relay, and mobile workflows remain checked in to minimize rebase conflicts,
but are disabled in the personal fork's GitHub Actions settings.

## Desktop identity and updater

The fork source fixes the desktop identity as:

- application ID `com.jmederosalvarado.t3code`
- product name `T3 Code JM`
- macOS passkeys disabled

The release workflow sets `T3CODE_DESKTOP_UPDATE_REPOSITORY=jmederosalvarado/t3code`, using the
upstream-supported build option to point Electron updates at the fork. Disabling macOS passkeys keeps
Developer ID signing independent of the upstream Clerk configuration.

## macOS signing

Unsigned artifacts are published while signing is unconfigured, but macOS automatic updates require
a consistently signed application. Add these repository secrets before treating the channel as ready
for automatic updates:

- `CSC_LINK`
- `CSC_KEY_PASSWORD`
- `APPLE_API_KEY`
- `APPLE_API_KEY_ID`
- `APPLE_API_ISSUER`

`CSC_LINK` is the exported Developer ID Application certificate and private key. `APPLE_API_KEY` is
the raw App Store Connect API `.p8` key text used for notarization.

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
