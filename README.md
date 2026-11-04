# MC3 iOS — public build orchestrator

This repository contains **one GitHub Actions workflow and this README. Nothing else.**

It exists for a single reason: GitHub Actions minutes are free and unmetered
for **public** repositories when using **standard GitHub-hosted runners**, while
the same builds on a private repository draw from a metered quota. The actual
project lives in private repositories; this workflow borrows a public trigger so
the build runs without consuming that quota.

## What is here

```
.github/workflows/build.yml   the whole build, triggered manually
README.md                     this file
```

## What is deliberately NOT here

- No source code
- No decompiled / recompiled game code
- No XBE, ISO or disc image
- No game assets, saves or textures
- No credentials, tokens or keys
- No build outputs

All of that is fetched at build time from private repositories using a
read-only token, into an ephemeral runner, and deleted at the end of the job.
Nothing proprietary is ever committed to this repository or published as an
artifact except the build result itself.

## How it works

1. You trigger the workflow manually (`workflow_dispatch` only — no `push`,
   no `pull_request`, so a fork can never start a run that receives secrets).
2. It checks out the private source repository at the requested ref.
3. It downloads the private generated-code archive from a private release,
   verifies its SHA-256, and unpacks it.
4. It configures and builds the iOS arm64 target with CMake/Xcode.
5. It uploads the build log and an unsigned `.ipa` as artifacts.

## Required secret

| Secret | Value | Minimum permissions |
|---|---|---|
| `MC3_PRIVATE_READ_TOKEN` | A token that can read the private source repository and the private build-inputs repository | **Contents: Read-only** on both. Nothing else. |

The token is passed to steps through environment variables only, masked in
logs, and never written to `.git/config` (`persist-credentials: false`). It is
never printed.

## Running a build

```bash
gh workflow run build.yml \
  --repo sugarroast/mc3-ios-public-builder \
  --ref main \
  -f private_ref=ios-port \
  -f upload_ipa=true
```

## Getting the results

```bash
# wait for the run
gh run list --repo sugarroast/mc3-ios-public-builder --workflow build.yml --limit 1

# download artifacts into ./out
gh run download <run-id> --repo sugarroast/mc3-ios-public-builder --dir out
```

Artifacts:

| Artifact | Contents |
|---|---|
| `mc3-ios-game-ipa` | Unsigned `mc3-ios-game.ipa` |
| `ios-game-build-log` | Full compiler/linker output |

> Artifacts of a public repository are downloadable by anyone who can see the
> run. Retention is capped at 3 days. Download promptly and delete the artifact
> afterwards if the build output should not stay public.

## Notes on cost

The job runs on `macos-15-intel`, a **standard** GitHub-hosted macOS runner —
not a larger runner. Larger runners are billed even for public repositories,
which would defeat the purpose of this repository. There are no larger-runner
labels in the workflow.
