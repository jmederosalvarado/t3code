# Personal Fork Release Operations

This repository is maintained as a linear custom patch stack on top of
`pingdotgg/t3code`.

## Branch model

- `upstream` records the exact upstream commit used as the base of the patch stack.
- `main` contains that upstream history followed by the fork-specific commits.
- `agent/*` branches are optional worktree helpers for isolation; they are not a review gate.

Do not merge upstream into `main`. The `JM Upstream Rebase` workflow fetches upstream daily, rebases
the commits in `upstream..main`, verifies the rebased tree, and atomically force-updates both branches
with explicit force-with-lease guards.

Land custom changes directly on `main`. Keep `upstream..main` as small as possible: when a change
belongs with an existing stack commit (same concern, overlapping files, or a fix/revision of that
patch), amend or rewrite that commit instead of adding another. Add a new stack commit only for a
distinct fork patch. Branch protection on `main` is intentionally off so this history rewriting is
allowed. Pull requests are optional; if used, fold the result into the correct stack commit rather
than growing a long review history on `main`.

## Workflows

- `jm-ci.yml` verifies fork changes on pulls and on pushes to `main`.
- `jm-upstream-rebase.yml` polls upstream GitHub Releases every 15 minutes and rebases the patch
  stack onto the next unsynced upstream release tag commit. It updates `main` and `upstream` only; it
  does not build desktop artifacts. It requires the repo secret `JM_MIRROR_TOKEN` (fine-grained PAT
  with Contents, Workflows, Actions, and Issues write) because `GITHUB_TOKEN` cannot push commits that
  update `.github/workflows/*`.
- `jm-release.yml` publishes macOS arm64 artifacts to `jmederosalvarado/t3code` GitHub Releases via
  `workflow_dispatch` only (manual or dispatched by the mirror).

### Release model

Version identity always comes from the upstream release tag that points at `origin/upstream`. The
built commit is always the current `main` tip (that upstream base plus the fork patch stack).

1. The mirror workflow moves `upstream` (and rebases `main`) onto a new upstream release when one
   appears, then dispatches `jm-release.yml`.
2. `jm-release.yml` looks up the upstream tag for `origin/upstream` on `pingdotgg/t3code`, then
   publishes `jm-<upstream-tag>` from `main` HEAD (stable and nightly use the same path).
3. If that `jm-v*` release already points at the same commit, the release workflow skips.
4. After amending fork patches on `main` without a new upstream release, dispatch `jm-release.yml`
   manually to republish under the current synced upstream version.
5. If the upstream base is already current but the GitHub release is missing or stale, the mirror
   also dispatches `jm-release.yml` so a failed publish can retry.

Each upstream release after the recorded bootstrap checkpoint is synced in publication order. An
upstream nightly such as `v0.0.29-nightly.20260720.858` produces `jm-v0.0.29-nightly.20260720.858`;
a stable `v0.0.29` produces `jm-v0.0.29`. The bootstrap checkpoint is upstream nightly
`v0.0.29-nightly.20260720.857`, making nightly `858` the first release eligible for mirroring.

The upstream CI, release, relay, and mobile workflows remain checked in to minimize rebase conflicts,
but are disabled in the personal fork's GitHub Actions settings.

## Desktop identity and releases

The fork source fixes the desktop identity as:

- application ID `com.jmederosalvarado.t3code`
- product name `T3 Code JM` (nightlies: `T3 Code JM (Nightly)`)
- Electron user-data directory `t3code-jm` (separate from upstream `t3code`)
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
