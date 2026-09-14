# AGENTS.md

Conventions for this repository.

## Python

Use `uv`/`uvx` for anything Python-related in workflows and locally - never
`pip` or `python -m venv` directly. Pin a version only where a workflow's
output has to be reproducible across runs (e.g. `python-lint.yml`'s
`uv sync --locked`); otherwise prefer `uvx --from <pkg>[==<version>] <tool>`
so no venv has to be created or cleaned up. See `yaml-lint.yml` for the
pattern.
