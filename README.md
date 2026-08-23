# Reusable GitHub Workflows

No workflow has been added yet.

## Usage

```yaml
jobs:
  build:
    uses: florian-hild/reusable-workflows/.github/workflows/<name>.yml@v1
    with:
      example: value
```

## Versioning

One version covers the whole repository. Every push to `main` that touches
`.github/workflows/` publishes three tags:

| Tag | Moves | Use it for |
|---|---|---|
| `v1.4.2` | never | exact reproducibility |
| `v1.4` | with every patch | fixes only |
| `v1` | with every compatible release | the normal choice |

The size of the bump comes from the commit **subject** lines: `(MAJOR)`,
`(MINOR)`, or patch when neither appears.

Tagging is done by
[`florian-hild/actions`](https://github.com/florian-hild/actions), see
[`release.yml`](.github/workflows/release.yml).
