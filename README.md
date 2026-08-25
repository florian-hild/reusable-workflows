# reusable-workflows

Reusable GitHub Actions workflows.

| Workflow | Purpose |
|---|---|
| [`python-lint.yml`](.github/workflows/python-lint.yml) | Ruff over a uv-managed project |
| [`python-test.yml`](.github/workflows/python-test.yml) | pytest, plus a check that `uv.lock` is current |
| [`container-build.yml`](.github/workflows/container-build.yml) | Build, push and tag a container image |
| [`security-scan.yml`](.github/workflows/security-scan.yml) | Trivy scan of a filesystem, repository or image |

Workflows prefixed `ci-local-` run on this repository itself and are not meant
to be called.

## Building a Python container app

Lint and tests gate the build, so a broken commit never reaches the registry:

```yaml
name: build

on:
  push:
    branches: [main]
  pull_request:

jobs:
  lint:
    uses: florian-hild/reusable-workflows/.github/workflows/python-lint.yml@v1

  test:
    uses: florian-hild/reusable-workflows/.github/workflows/python-test.yml@v1

  build:
    needs: [lint, test]
    if: github.ref == 'refs/heads/main'
    permissions:
      contents: write # release-tags pushes git tags
      packages: write # push the image
    uses: florian-hild/reusable-workflows/.github/workflows/container-build.yml@v1
    with:
      image: ghcr.io/florian-hild/repository/kinderuni-anmeldung
      publish_container: true
      publish_tags: true
      smoke_test_env: |
        TZ=Europe/Berlin
        ALLOW_INSECURE_DEV_SECRET=true
    secrets:
      registry_token: ${{ secrets.REPO_PAT_TOKEN }}
```

On `main` this publishes container tags and one git tag:

```
image:1.4.2   image:1.4   image:1   image:latest
v1.4.2
```

Container tags carry no `v` so they read naturally in a compose file, git tags
do. The size of the bump comes from the commit **subjects**: `(MAJOR)`,
`(MINOR)`, patch otherwise.

## Versioning several things in one repository

A monorepo that releases several images independently must not let every
commit bump every version. Point `change_path` at the component and turn
`bump_version` off, and a run where nothing changed there skips the build:

```yaml
    permissions:
      contents: write
      packages: write
    uses: florian-hild/reusable-workflows/.github/workflows/container-build.yml@v1
    with:
      image: ghcr.io/florian-hild/repository/jumphost-ssh
      tag_prefix: jumphost-ssh/v
      change_path: services/jumphost_ssh
      bump_version: false
```

`schedule` and `workflow_dispatch` events always bump, so periodic rebuilds
keep releasing patches. Note that `(MAJOR)`/`(MINOR)` markers are read from
all commits in a push, not only those touching `change_path`.

## Permissions

The calling job must grant what the reusable workflow's build job requests,
**whether or not the feature behind a permission is used**: a called workflow
can only restrict the caller's grant, never expand it, and any job requesting
more than the caller granted fails the whole run at startup with nothing in
the logs. The build job needs:

```yaml
    permissions:
      contents: write # the release-tags step pushes git tags
      packages: write # push the image
```

Even with `publish_tags: false` the `contents: write` grant is required,
because the permission check is static, not per feature.

## What the build does beyond pushing

- **Smoke test** — starts the pushed image and waits for its `HEALTHCHECK`.
  Catches an image that builds but does not start. Without a `HEALTHCHECK` in
  the image the step warns rather than passing silently.
- **Attestations** — SBOM and provenance are attached, so it stays auditable
  what went into an image and who built it.
- **Tags last** — git tags are published only after the build and smoke test
  pass, so a failed run leaves no version claimed.
- **Publishing is opt-in** — `publish_container` and `publish_tags` default to
  off, so a pull request build stays a build. The smoke test pulls the pushed
  image, so it is skipped while `publish_container` is off.

## Security scanning

Trivy over the working tree, or over an image that was just built:

```yaml
  scan:
    permissions:
      contents: read
      pull-requests: write
    uses: florian-hild/reusable-workflows/.github/workflows/security-scan.yml@v1
    with:
      trivy_scan_type: fs
```

- **The report is always published** — into the job summary, as a run artifact
  and, on a pull request, as a single comment that is updated in place instead
  of one per push.
- **Findings fail the job by default.** Set `trivy_scan_exit_code: 0` to report
  without gating, for instance while adopting the scan on an existing project.
  A scan that fails for any other reason still fails the job.
- **`pull-requests: write` is only needed for the comment.** It lives in its
  own job, so with `comment_on_pr` off the scan runs on `contents: read` alone.
  Pull requests from forks get a read-only token, so the comment is skipped
  there.
- **Non-public images** need `registry_login: true` plus a `registry_token`.
- **Think before narrowing `trivy_scan_severity`.** `HIGH,CRITICAL` is the
  reflex for keeping a dependency tree's LOW and MEDIUM CVEs from blocking
  every merge, and for that it is right. It also drops 61 of trivy's 106
  secret rules, among them `grafana-api-token`, `hashicorp-tf-api-token` and
  `slack-web-hook`. On a repository without dependencies the filter therefore
  costs findings and saves no noise. Narrow it when the noise is real, and
  pair it with `trivy_ignore_unfixed: true` rather than reaching for it first.

## Versioning

One version covers the whole repository. Every push to `main` that touches
`.github/workflows/` publishes `v1.4.2`, `v1.4` and `v1`. Pin the major:

```yaml
uses: florian-hild/reusable-workflows/.github/workflows/python-test.yml@v1
```

Tagging is done by [`florian-hild/actions`](https://github.com/florian-hild/actions).

## Adding a workflow

Add it under `.github/workflows/` with `on: workflow_call`, document its inputs
in the file, and list it in the table above. The next push to `main` releases
it.
