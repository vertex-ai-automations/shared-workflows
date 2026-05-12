# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

`vertex-ai-automations/shared-workflows` is a **library of reusable GitHub Actions workflows** — not a runnable application. There are no build, lint, or test commands to run locally. All "execution" happens in GitHub Actions runners when consuming repos call these workflows.

To test changes locally, use [`act`](https://github.com/nektos/act):
```bash
act -W .github/workflows/publish-mkdocs.yml
```

## Architecture

All workflow files live in `.github/workflows/`. Each is a `workflow_call` reusable workflow consumed by other repos in the `vertex-ai-automations` org via:
```yaml
uses: vertex-ai-automations/shared-workflows/.github/workflows/<file>.yml@<ref>
```

**Eight workflows:**

| File | Purpose |
|---|---|
| `python-publish.yml` | Multi-job pipeline: validate tag → test → build → publish to TestPyPI and/or PyPI |
| `publish-mkdocs.yml` | Build MkDocs site → deploy to GitHub Pages |
| `publish-zensical.yml` | Build Zensical site (MkDocs-based, Python 3.10+) → deploy to GitHub Pages |
| `test.yml` | Cross-platform, multi-Python test matrix; `os-matrix` and `python-versions` are JSON-string inputs |
| `lint.yml` | Ruff lint (`ruff check`) + format check (`ruff format --check`) |
| `typecheck.yml` | Static type checking; defaults to `mypy src/`, accepts any checker via `typecheck-command` |
| `audit.yml` | Dependency security audit via `pip-audit`; fails on any known CVE |
| `coverage.yml` | pytest with `--cov`, posts a coverage table to the Actions job summary; optional threshold gate |

## Key Design Invariants

**`python-publish.yml` job DAG:**
```
validate-version ──┐
                   ├──▶ build ──▶ publish-testpypi ──▶ publish-pypi
test ──────────────┘
```

- Docs deployment is triggered via `workflow_run: workflows: ["CI"]` so it only fires after CI passes on `main`. The docs job carries an `if: conclusion == 'success'` condition. Tag-triggered publish is unaffected.
- All publish jobs are **tag-gated** internally — callers never need `if: startsWith(github.ref, 'refs/tags/')`.
- All boolean inputs check both the boolean form **and** the string form (e.g. `!= false && != 'false'`) — `workflow_dispatch` coerces booleans to strings, which breaks strict boolean comparisons.
- User-controlled string inputs that are passed to shell `run:` steps (`pip-audit-args`, `lint-args`, `ruff-version`) are routed via environment variables, never interpolated inline, to prevent shell injection.
- `fetch-tags: true` is set alongside `fetch-depth: 0` on every checkout that mentions setuptools_scm — history alone does not guarantee tag reachability on all runner configurations.
- `concurrency: group` in docs workflows includes `${{ github.ref }}` to prevent cross-branch deployment queueing.
- Tags must be **bare SemVer** (`X.Y.Z`) — no `v` prefix, no pre-release suffixes. A `reject-v-prefix-tag` job surfaces a helpful error for `v`-prefixed tags; other non-matching tags skip silently.
- **`setuptools_scm`** derives version from the git tag at build time. All checkout steps use `fetch-depth: 0` + `fetch-tags: true` — a shallow clone always produces `0.1.dev0`.
- `publish-testpypi` is skipped when `run-tests: false` (TestPyPI is the pre-release gate; bypassing tests bypasses TestPyPI).
- `id-token: write` (OIDC) is scoped only to the `publish-pypi` job and the `deploy-docs` job — not the full workflow.
- Artifact name is auto-derived from the repo name: `{repo-name}-dist`.

**Docs workflows:**
- Use `actions/deploy-pages` (official) rather than `peaceiris/actions-gh-pages`.
- `concurrency: cancel-in-progress: false` — queues deployments instead of cancelling in-flight ones.
- MkDocs outputs to `site/`; Zensical outputs to `public/`.

## Versioning & Breaking Changes

This repo uses bare SemVer tags (no `v` prefix). Consuming repos pin via `@main` or `@X.Y.Z`.

Breaking changes (removing or renaming inputs) **must** bump the major version and be documented. The caller-facing contract is the `inputs:` block of each workflow.

## Releasing a New Version

```bash
git tag -a 1.2.0 -m "Release 1.2.0: <summary>"
git push origin 1.2.0
```

## Common Fix Patterns

**Version mismatch (`setuptools_scm` resolved a dev suffix):**
```bash
git tag -d 1.2.1
git push origin :refs/tags/1.2.1
# ensure HEAD is the release commit
git tag 1.2.1 && git push origin 1.2.1
```

**MkDocs `site/` not found:** Check that `mkdocs.yml` and `docs/index.md` exist in the caller repo, and that the `build-docs` step succeeded before the upload step.
