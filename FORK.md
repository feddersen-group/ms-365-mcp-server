# Fork maintenance

This is a fork of [Softeria/ms-365-mcp-server](https://github.com/Softeria/ms-365-mcp-server).

It exists so that the container image can be built here rather than pulled from a
third party: the exact upstream revision is pinned and reviewable, every image is
scanned before it is published, and the image is served from our own registry.

We do not intend to diverge from upstream functionally. Everything in this fork is
build and supply-chain plumbing.

## The one rule

**The fork only ever adds files. It never edits a file that upstream owns.**

That is the entire conflict-avoidance strategy. `git merge upstream/main` cannot
produce a content conflict on a file the fork has not touched, so syncing stays a
review of upstream's diff rather than a merge exercise.

Fork-owned files, and the complete delta against upstream:

| Path                                  | Purpose                                                |
| ------------------------------------- | ------------------------------------------------------ |
| `FORK.md`                             | this document                                          |
| `.fork/upstream-version`              | the upstream release the current `main` corresponds to |
| `.github/workflows/docker.yml`        | build, scan and publish the image                      |
| `.github/workflows/upstream-sync.yml` | weekly upstream merge as a pull request                |

Everything else is upstream's, byte for byte. `git diff upstream/main main --stat`
should list only the files above. If it lists anything else, the invariant has been
broken and the next sync will conflict.

### When you need to change upstream behaviour

Do not edit the upstream file. In order of preference:

1. Configure it from a fork-owned file instead.
2. If it is a GitHub Actions workflow you want to stop running, disable it in the
   Actions UI (or `gh workflow disable <name>`). That state lives in GitHub, not in
   git, so it costs no diff.
3. If there is genuinely no other way, add a fork-owned file next to the upstream
   one (for example `docker/Dockerfile.fork`) and point our workflow at it, leaving
   upstream's copy untouched.
4. Only if none of the above work, edit the upstream file, and record here why. Every
   such edit is a conflict you will resolve on every future sync, forever.

## Disabled upstream workflows

`Release` (`.github/workflows/release.yml`) is disabled in the Actions settings. It
runs semantic-release, which publishes to npm under upstream's package name. The file
itself is left exactly as upstream ships it.

Confirm with `gh workflow list`. After enabling Actions on a fresh clone of this fork,
disable it again:

```sh
gh workflow disable Release
```

## Versioning

`.fork/upstream-version` holds the upstream release that `main` currently contains,
for example `0.148.1`. The sync workflow updates it from `git describe` against the
upstream tags, so the version bump is visible in the sync pull request.

`docker.yml` reads that file and publishes to
`ghcr.io/feddersen-group/ms-365-mcp-server` as:

- `<upstream-version>`, for example `0.148.1`, the tag to pin in deployments
- `latest`, the newest build from `main`
- `main` and `sha-<commit>`, for tracing an image back to a commit

Upstream tags are deliberately not pushed into this fork. A tag points at an upstream
commit, which does not contain `docker.yml`, so a tag-triggered build could not run.

## Syncing

`upstream-sync.yml` runs every Monday at 06:00 UTC, and on demand via
`gh workflow run "Upstream sync"`. It merges `upstream/main` into `sync/upstream`,
updates `.fork/upstream-version` and opens a pull request. Review the diff and merge.

Optional: set an `UPSTREAM_SYNC_TOKEN` repository secret to a fine-grained PAT with
contents and pull-requests write access. Pull requests opened by the default
`GITHUB_TOKEN` do not trigger other workflows, so without it the Docker build and the
scans only run after the sync PR is merged, rather than on the PR itself.

To sync by hand:

```sh
git fetch upstream --tags
git checkout -B sync/upstream origin/main
git merge upstream/main
printf '%s\n' "$(git describe --tags --abbrev=0 upstream/main | sed 's/^v//')" > .fork/upstream-version
git commit -m "chore(fork): pin upstream version ..." .fork/upstream-version
git push -u origin sync/upstream
```

### If a sync does conflict

It means the invariant was broken. Do not hand-merge and move on, because the same
conflict returns next week. Instead take upstream's version of the file wholesale and
re-express the fork's intent in a fork-owned file:

```sh
git checkout --theirs <the upstream file>
git add <the upstream file>
```

Then apply one of the four options above and note it in this document.

## Security scanning

`docker.yml` gates the image on two checks before anything is pushed:

- `npm audit --omit=dev --audit-level=high` fails the build. The release stage of the
  Dockerfile installs with `--omit=dev`, so dev-only advisories cannot reach the image
  and must not block it. The full audit, dev dependencies included, runs alongside as
  a report in the job summary.
- Trivy scans the built image. Fixable `CRITICAL` findings fail the build. Every
  fixable finding, at any severity, is uploaded as SARIF to the repository Security
  tab for triage. Note that `trivy-action` ignores its `severity` input when writing
  SARIF unless `limit-severities-for-sarif` is set, and that GitHub re-buckets Trivy's
  severities by CVSS score, so the counts in the Security tab will not match Trivy's
  own labels.

The image is built once for scanning and only pushed if the scan passes, so an image
that failed a gate never reaches the registry.

## Known findings

The scans separate two things that are easy to conflate:

- `npm audit --omit=dev` covers the application's own production dependencies.
- Trivy covers the whole image, so it also reports the base image: Alpine's OpenSSL
  and the npm CLI that the `node:*-alpine` images bundle but that this container never
  runs.

Every Trivy finding measured so far has come from the base image, none from the
application's dependencies.
