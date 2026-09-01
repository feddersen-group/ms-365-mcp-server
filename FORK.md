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

Upstream leaves `package.json` at `0.0.0-development` and lets semantic-release fill it
in when it publishes. We do not run semantic-release, so the build stamps the pinned
version into `package.json` and `package-lock.json` in the CI workspace before building.
`src/version.ts` reads `package.json` at runtime and the release stage copies it into
the image, so `--version` inside the container matches the image tag. The stamp only
ever touches the CI workspace, never a commit, so the invariant holds.

`docker.yml` reads that file and publishes to
`ghcr.io/feddersen-group/ms-365-mcp-server` as:

- `<upstream-version>`, for example `0.148.1`, the tag to pin in deployments
- `latest`, the newest build from `main`
- `main` and `sha-<commit>`, for tracing an image back to a commit

Upstream tags are deliberately not pushed into this fork. A tag points at an upstream
commit, which does not contain `docker.yml`, so a tag-triggered build could not run.

## Consuming the image

The image is deployed via Portainer. Pin `<upstream-version>` rather than `latest`, so
a sync does not change a running deployment without a deliberate step.

It is built for `linux/amd64` and `linux/arm64`, so it runs on either kind of VM. The
arm64 build runs under QEMU emulation on an x86 runner, which is the bulk of the build
time. If arm64 is ever known to be unnecessary, dropping it from `PLATFORMS` in
`docker.yml` roughly halves the pipeline.

GHCR packages pushed by `GITHUB_TOKEN` are created **private**, even from a public
repository. The first push therefore produces a package Portainer cannot pull
anonymously. Either set the package to public under Packages -> Package settings, or
give Portainer a registry credential with a token that has `read:packages`.

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
- Trivy scans the built image, once per target platform. Fixable `CRITICAL` findings
  fail the build on either platform. Every
  fixable finding, at any severity, is uploaded as SARIF to the repository Security
  tab for triage. Note that `trivy-action` ignores its `severity` input when writing
  SARIF unless `limit-severities-for-sarif` is set, and that GitHub re-buckets Trivy's
  severities by CVSS score, so the counts in the Security tab will not match Trivy's
  own labels.

Each platform is built and scanned in its own job, and the `push` job only runs once
every one of them passed, so an image that failed a gate never reaches the registry.
buildx can only load one platform into the docker daemon at a time, which is why this
is a matrix rather than a single multi-platform build. The push reuses the layers the
scan jobs left in the buildx cache.

SARIF is uploaded from `linux/amd64` only. The findings are the same packages on both
platforms, and uploading twice would double every alert in the Security tab. Both
platforms are still gated.

## Known findings

The scans separate two things that are easy to conflate:

- `npm audit --omit=dev` covers the application's own production dependencies.
- Trivy covers the whole image, so it also reports the base image: Alpine's OpenSSL
  and the npm CLI that the `node:*-alpine` images bundle but that this container never
  runs.

Every Trivy finding measured so far has come from the base image, none from the
application's dependencies.

These base image findings are knowingly accepted. Clearing them needs three changes to
the release stage (`node:24-alpine`, `apk --no-cache upgrade`, and deleting the bundled
npm CLI), which was measured to take the image from 29 fixable findings to 0. The
`Dockerfile` is upstream's, this image is only deployed internally via Portainer, and
we would rather keep the zero-delta invariant than carry a patched Dockerfile. Revisit
if the deployment ever becomes externally reachable.

The `CRITICAL` gate is unaffected by this and still blocks a release.
