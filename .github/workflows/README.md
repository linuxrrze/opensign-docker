# GitHub Actions workflows in this repo

One workflow per image: `opensign-client-rebuild.yml` and
`opensign-server-rebuild.yml`. Both are the "fresh clone from git" flavor:
the image's own Dockerfile (`<name>/docker/Dockerfile`) is built with a
fresh clone of the upstream OpenSign monorepo, at the tracked release tag,
as the build context.

Shared helpers live in `.github/scripts/`:

| Script | Step | What |
| --- | --- | --- |
| `base-digest.sh` | Check base image digest | Registry digest of the Dockerfile's base image (manifest HEAD, no pull). |
| `add-banner.sh` | Add startup banner | Thin layer on top of the built image that prints name + version (figlet) on container start, then execs the original entrypoint. |
| `trivy-scan.sh` | Security scan (Trivy) | Installs a pinned, checksum-verified Trivy and fails on a fixable CRITICAL/HIGH not in `<name>/.trivyignore`. |

## Step order and triggers

```
checkout -> set image name -> check release -> check base digest
  -> clone -> build -> banner -> trivy -> push -> record version
```

- **Every build-side step** uses
  `if: github.event_name != 'schedule' || steps.check.outputs.changed == 'true' || steps.base.outputs.changed == 'true'`:
  the daily cron skips when neither the upstream release nor the base digest
  moved; a push or manual run always rebuilds. The record step only commits
  when the version/digest files actually changed.
- **`paths:`** lists only what goes into (or gates) the image:
  `<name>/docker/**`, `<name>/.trivyignore`, the workflow itself. Never
  `<name>/**` -- README edits would trigger full rebuilds.
- **Rollback tag.** Push also publishes `:<version>-<YYYYMMDD>`, since a
  rebuild re-publishes the same `:<version>`.
- **Run titles.** `run-name:` shows "Scheduled check" / "Manual run";
  pushes keep GitHub's default (the commit subject).
- **Concurrency.** Each workflow has its own `concurrency:` group (queue, no
  cancel), so a manual run can't overlap the cron on the same tags.

## Authentication and GitHub specifics

- **No secrets.** Registry login uses `github.actor` + `GITHUB_TOKEN`
  (`permissions: packages: write`); the version-bump commit is pushed with
  the token `actions/checkout` persists (`permissions: contents: write`), as
  `github-actions[bot]`.
- **Image name.** ghcr.io names must be lowercase, so "Set image name"
  derives `ghcr.io/<owner>/<image>` from `GITHUB_REPOSITORY_OWNER`
  lowercased -- never hardcode the owner.
- **Package link.** The `org.opencontainers.image.source` label ties the
  ghcr.io package to this repo.
- **Release check** is authenticated with `GITHUB_TOKEN`: the anonymous API
  limit (60/h) is shared per runner IP.
- **Bot pushes don't trigger workflows.** Pushes made with `GITHUB_TOKEN`
  never start other workflow runs, so a version-bump commit can't cause a
  second build.

## Gotchas

- **pipefail + SIGPIPE.** Never truncate a pipe early (`head`, `awk` with an
  early `exit`) under `set -o pipefail` -- use `tail -1` or an awk
  done-guard. The release check uses `?per_page=1` instead of `| head`.
- **Push races.** The two workflows run 15 minutes apart but can still
  overlap (slow server build, manual runs). The record step retries
  `git push` with `fetch` + `rebase` up to 5 times; each workflow only
  touches its own directory's files, so the rebase can't conflict.
- **Secrets via step `env:`,** never `${{ secrets.* }}` inline in a script.
- **Trivy.** Accepted findings go in `<name>/.trivyignore`, one ID per line,
  each with a reason. Prune IDs Trivy no longer reports after an OpenSign
  release or base bump. To upgrade Trivy, bump `TRIVY_VERSION` and
  `TRIVY_SHA256` in `trivy-scan.sh`.
- **Verify a changed pipeline end to end.** Decrement
  `<name>/.<name>-version` (or edit the digest file) and push -- that alone
  triggers nothing, the file is outside `paths:`. Then *Run workflow*:
  confirm it reports the mismatch, rebuilds, pushes, and the bot commit
  bumps the file back. Syntax-valid but never-run workflows have hidden
  real bugs before.
