# reusable-workflows

Reusable GitHub Actions workflows.

| Workflow | Purpose |
|---|---|
| [`python-lint.yml`](.github/workflows/python-lint.yml) | Ruff over a uv-managed project |
| [`python-test.yml`](.github/workflows/python-test.yml) | pytest, plus a check that `uv.lock` is current |
| [`container-build.yml`](.github/workflows/container-build.yml) | Build, push and tag a container image |
| [`container-deploy.yml`](.github/workflows/container-deploy.yml) | Deploy a compose project onto a self-hosted runner |
| [`security-scan.yml`](.github/workflows/security-scan.yml) | Trivy scan of a filesystem, repository or image |

Workflows prefixed `ci-local-` run on this repository itself and are not meant
to be called.


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
