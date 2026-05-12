# 🏗️ Shared GitHub Actions Workflows

> Centralized, reusable CI/CD workflow templates for all projects in the **vertex-ai-automations** GitHub Organization.

[![Workflows](https://img.shields.io/badge/workflows-8-blue?logo=github-actions)](/.github/workflows)
[![License](https://img.shields.io/badge/license-MIT-green)](./LICENSE)
[![GitHub Actions](https://img.shields.io/badge/GitHub%20Actions-reusable-orange?logo=github)](https://docs.github.com/en/actions/using-workflows/reusing-workflows)

---

## 📋 Table of Contents

- [Overview](#-overview)
- [Repository Structure](#-repository-structure)
- [Available Workflows](#-available-workflows)
  - [Deploy MkDocs to GitHub Pages](#-deploy-mkdocs-to-github-pages)
  - [Deploy Zensical Docs to GitHub Pages](#-deploy-zensical-docs-to-github-pages)
  - [Build and Publish Python Package to PyPI](#-build-and-publish-python-package-to-pypi)
  - [Cross-platform Test Matrix](#-cross-platform-test-matrix)
  - [Lint with Ruff](#-lint-with-ruff)
  - [Static Type Check](#-static-type-check)
  - [Security Audit](#-security-audit)
  - [Test Coverage Report](#-test-coverage-report)
- [Release Quick Start](#-release-quick-start)
- [CI Quick Start](#-ci-quick-start)
- [Publish Target Flag](#-publish-target-flag)
- [setuptools_scm Versioning](#-setuptools_scm-versioning)
- [Setup Guide](#-setup-guide)
  - [Organization Settings](#1-organization-settings)
  - [PyPI Trusted Publishing](#2-pypi-trusted-publishing-recommended)
  - [GitHub Pages](#3-github-pages)
  - [Secrets & Environments](#4-secrets--environments)
  - [Docs Requirements File](#5-docs-requirements-file)
- [Input Reference](#-input-reference)
- [Migration Guide](#-migration-guide)
- [Troubleshooting](#-troubleshooting)
- [Contributing](#-contributing)
- [Versioning & Pinning](#-versioning--pinning)

---

## 🌐 Overview

This repository (`vertex-ai-automations/shared-workflows`) is the **single source of truth** for CI/CD pipelines across the organization. Instead of copy-pasting workflow YAML into every project, each project calls a versioned template from here using GitHub's [reusable workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows) feature.

**Benefits:**

- 🔒 **Security patches** propagate to all projects by updating one file
- 🔁 **DRY principle** — no duplicated workflow logic across repos
- 📌 **Version pinning** — projects can pin to a tag (e.g., `@1.2.0`) for stability
- 🧩 **Configurable** — every workflow exposes `inputs` to customize behavior per project
- 🤖 **Zero boilerplate** — artifact names and permissions are handled automatically
- 👁️ **Auditable** — one place to review and approve CI/CD changes

---

## 📁 Repository Structure

```
vertex-ai-automations/shared-workflows/     ← This repo
│
├── .github/
│   └── workflows/                          ← All reusable workflow templates live here
│       ├── publish-mkdocs.yml              ← MkDocs → GitHub Pages deployment
│       ├── publish-zensical.yml            ← Zensical → GitHub Pages deployment
│       ├── python-publish.yml              ← Python package → PyPI/TestPyPI publishing
│       ├── test.yml                        ← Cross-platform, multi-Python test matrix
│       ├── lint.yml                        ← Ruff lint + format check
│       ├── typecheck.yml                   ← Static type checking (mypy / pyright)
│       ├── audit.yml                       ← Dependency security audit (pip-audit)
│       └── coverage.yml                    ← Test coverage report → Actions job summary
│
├── docs/
│   └── adr/                                ← Architecture Decision Records
│
├── README.md                               ← You are here
└── LICENSE
```

> **How the `uses:` path works:**
> Reusable workflows are referenced as `owner/repo/.github/workflows/filename@ref`.
> For this repo: `vertex-ai-automations/shared-workflows/.github/workflows/python-publish.yml@main`

---

## ⚙️ Available Workflows

### 📚 Deploy MkDocs to GitHub Pages

**File:** `.github/workflows/publish-mkdocs.yml`

Builds a [MkDocs](https://www.mkdocs.org/) documentation site and deploys it to GitHub Pages using GitHub's official Pages deployment pipeline.

**What it does:**

1. Checks out the repository with full git history (needed for date plugins)
2. Installs Python and caches pip dependencies
3. Installs doc dependencies from a pinned requirements file
4. Optionally adds `src/` to `PYTHONPATH` for projects using src layout
5. Runs `mkdocs build` (with optional `--strict` flag)
6. Uploads the built site as a Pages artifact
7. Deploys to GitHub Pages via the official `actions/deploy-pages` action

**Key design decisions:**

| Decision | Reason |
|---|---|
| Uses `actions/deploy-pages` instead of `peaceiris/actions-gh-pages` | Official GitHub action — no third-party dependency for a critical deployment step |
| `permissions: pages: write` (not `contents: write`) | Least-privilege — the workflow token cannot push commits or modify branches |
| `concurrency: cancel-in-progress: false` | Prevents a fast push from cancelling an in-flight deploy; queues instead |
| Falls back gracefully if no requirements file found | Won't hard-fail new projects that haven't set up `docs/requirements.txt` yet |

---

### 🔷 Deploy Zensical Docs to GitHub Pages

**File:** `.github/workflows/publish-zensical.yml`

Builds a [Zensical](https://zensical.dev/) documentation site and deploys it to GitHub Pages. Zensical is an MkDocs-based documentation framework and requires Python 3.10+.

**What it does:**

1. Checks out the repository with full git history
2. Installs Python and caches pip dependencies
3. Installs doc dependencies from the requirements file specified by `docs-requirements`
4. Optionally adds `src/` to `PYTHONPATH` for projects using src layout
5. Runs `zensical build` (outputs to `public/` by default)
6. Uploads the built site as a Pages artifact
7. Deploys to GitHub Pages via `actions/deploy-pages`

**Key design decisions:**

| Decision | Reason |
|---|---|
| Uses `docs-requirements` input | Lets callers pin exact versions for reproducible builds |
| `id-token: write` scoped to `deploy-docs` job only | Least-privilege — the build job installs third-party packages and should not have OIDC token minting capability |
| Requires Python 3.10+ | Zensical framework constraint |

---

### 📦 Build and Publish Python Package to PyPI

**File:** `.github/workflows/python-publish.yml`

A multi-job pipeline that validates, tests, builds, and publishes a Python package. Supports publishing to PyPI, TestPyPI, or both via a single `publish-target` flag.

**Job DAG:**

```
validate-version ──┐
                   ├──▶ build ──▶ publish-testpypi ──▶ publish-pypi
test ──────────────┘               (if target includes    (if target includes
(continue-on-error)                 testpypi)              pypi)
```

**What each job does:**

1. **`validate-version`** — Validates the git tag is clean SemVer (`X.Y.Z`), installs `setuptools_scm`, resolves the version from the tag, and asserts they match exactly. Catches dirty/dev tags before any artifact is built.
2. **`test`** — Runs your test suite with a configurable command. Marked `continue-on-error: true` — failures are reported in the UI but do not block publishing.
3. **`build`** — Installs `build` + `twine`, runs `python -m build`, prints the version stamped into the wheel by `setuptools_scm`, validates metadata with `twine check`, and uploads the artifact named `{repo-name}-dist`.
4. **`publish-testpypi`** — Publishes to TestPyPI using `TEST_PYPI_API_TOKEN`. Skipped when `run-tests: false` or `publish-target: pypi`.
5. **`publish-pypi`** — Publishes to real PyPI. When `publish-target: both`, waits for `publish-testpypi` to succeed first. Uses OIDC Trusted Publishing by default; falls back to `PYPI_API_TOKEN` if `use-trusted-publishing: false`.

**Key design decisions:**

| Decision | Reason |
|---|---|
| `publish-target` flag replaces `repository-url` + `use-trusted-publishing` pair | Single input controls the entire publish flow — callers don't need to know which URL maps to which registry |
| `setuptools_scm` version validation instead of static `pyproject.toml` read | Projects using `dynamic = ["version"]` have no static version field — version is derived from the git tag at build time |
| All jobs tag-gated internally | Callers never need `if: startsWith(github.ref, 'refs/tags/')` — non-tag pushes skip all jobs cleanly |
| `permissions` declared inside the workflow | Callers do not need `permissions:` blocks on the publish job |
| `continue-on-error: true` on `test` job | Test results are visible in the UI without blocking a release — useful when publishing hotfixes |
| `publish-testpypi` skipped when `run-tests: false` | TestPyPI is the pre-release validation gate — bypassing tests bypasses the sandbox too |
| `publish-testpypi` runs before `publish-pypi` when target is `both` | TestPyPI confirms the upload works cleanly before the immutable real release |
| Trusted Publishing as default for PyPI | More secure than long-lived API tokens — no secrets to rotate or leak |
| Artifact name auto-derived from repo name | Eliminates per-project configuration — repo `custom-logger` → artifact `custom-logger-dist` |
| `fetch-depth: 0` + `fetch-tags: true` on all checkout steps | `setuptools_scm` requires full git history to resolve the version — shallow clones produce `0.1.dev0` |

---

### 🧪 Cross-platform Test Matrix

**File:** `.github/workflows/test.yml`

Runs your test suite across a configurable matrix of OS and Python version combinations.

**What it does:**

1. Spins up a matrix of runners from `os-matrix` × `python-versions`
2. Checks out the repo with full git history (needed for `setuptools_scm` during install)
3. Sets up Python with built-in pip caching (cross-platform safe)
4. Upgrades pip and runs `install-command`
5. Runs `test-command`

**Key design decisions:**

| Decision | Reason |
|---|---|
| `os-matrix` and `python-versions` are JSON-string inputs | GitHub Actions has no native array input type — `fromJson()` unpacks them into the matrix |
| `\|\|` fallback on every `fromJson()` call | `workflow_call` and `workflow_dispatch` have separate defaults; without the fallback, manual dispatch with an empty field causes a parse error |
| `shell: bash` on install and test steps | Windows runners default to PowerShell — `shell: bash` (Git Bash) ensures `install-command` syntax is consistent across all three OS runners |
| `setup-python cache: 'pip'` instead of `actions/cache` | The built-in cache resolves the correct pip cache path per OS — `~/.cache/pip` doesn't exist on Windows |
| `fetch-depth: 0` | `setuptools_scm` requires full git history to derive the package version during `pip install` |
| `fail-fast` compares both `true` and `'true'` | GitHub Actions passes boolean inputs as strings internally — comparing both forms prevents `false` from being interpreted as truthy |

---

### 🔍 Lint with Ruff

**File:** `.github/workflows/lint.yml`

Runs [Ruff](https://docs.astral.sh/ruff/) to enforce both lint rules and code formatting in a single job.

**What it does:**

1. Checks out the repository
2. Installs Python and Ruff (optionally pinned to a specific version)
3. Runs `ruff check` — enforces lint rules from your `pyproject.toml` or `ruff.toml`
4. Runs `ruff format --check` — verifies formatting without modifying files

**Key design decisions:**

| Decision | Reason |
|---|---|
| `ruff check` and `ruff format --check` in one job | Both run in seconds — combining them means all failures are visible together rather than split across two jobs |
| `ruff-version` input | Pins Ruff to a known version for reproducible CI; omit to always use the latest release |
| `lint-args` passed to both steps | A single input controls the paths checked by both the linter and the formatter — no duplication needed in the caller |

---

### 🔎 Static Type Check

**File:** `.github/workflows/typecheck.yml`

Runs a static type checker against your codebase. Defaults to [mypy](https://mypy-lang.org/) but accepts any checker via `typecheck-command`.

**What it does:**

1. Checks out the repo with full git history
2. Sets up Python with pip caching
3. Upgrades pip and runs `install-command` to install the package and its type stubs
4. Runs `typecheck-command`

**Key design decisions:**

| Decision | Reason |
|---|---|
| `typecheck-command` instead of a `tool` enum | Projects using pyright or basedpyright don't need a different workflow — one input covers all checkers |
| `install-command` defaults to `.[dev]` (not `.[test]`) | Type stubs and mypy are typically declared in the `dev` extras group, separate from the test runner |
| `fetch-depth: 0` | Same reason as `test.yml` — `setuptools_scm` needs full history during `pip install` |

---

### 🔒 Security Audit

**File:** `.github/workflows/audit.yml`

Scans all installed packages (including transitive dependencies) for known vulnerabilities using [`pip-audit`](https://github.com/pypa/pip-audit). The job fails on any unfixed CVE.

**What it does:**

1. Installs the project via `install-command` to populate the full dependency set
2. Installs `pip-audit` in the same environment
3. Runs `pip-audit` — scans every installed package against the OSV and PyPI Advisory databases
4. Fails the job if any vulnerability is found

**Recommended triggers:** every pull request + a weekly scheduled scan (catches new CVEs disclosed after merge).

**Key design decisions:**

| Decision | Reason |
|---|---|
| Installs the project before auditing | Scanning the full installed environment catches transitive dep vulnerabilities — auditing a `requirements.txt` alone can miss them |
| `pip-audit` installed separately (not part of `install-command`) | Keeps the audit tool out of the project's own dependency tree |
| `pip-audit-args` input | Provides an escape hatch for suppressing accepted advisories (`--ignore-vuln GHSA-xxxx-yyyy-zzzz`) without modifying the workflow |

---

### 📊 Test Coverage Report

**File:** `.github/workflows/coverage.yml`

Runs your test suite with `pytest-cov` and posts a per-file coverage table to the GitHub Actions job summary. No third-party service (Codecov, Coveralls) required.

**What it does:**

1. Installs the project via `install-command`, then installs `pytest-cov`
2. Runs `test-command` with `--cov`, `--cov-report=term-missing`, and `--cov-report=json`
3. Parses `coverage.json` and writes a markdown coverage table to `$GITHUB_STEP_SUMMARY`
4. Optionally fails the job if total coverage is below `coverage-threshold`

**Key design decisions:**

| Decision | Reason |
|---|---|
| `pytest-cov` installed automatically | Callers don't need to add it to their extras — the workflow always ensures it is present |
| `--cov-report=json` drives both the summary and the threshold check | A single machine-readable output feeds both steps, avoiding a second pytest run |
| Summary step runs with `if: always()` | Coverage table is posted even when tests fail — useful for diagnosing which files lack coverage |
| `coverage-threshold: 0` disables the gate by default | Opt-in enforcement — projects set their own bar without the workflow imposing a default |

---

## 🚀 Release Quick Start

Create `.github/workflows/release.yml` in your project repo:

```yaml
name: 🚢 Release

on:
  push:
    tags: ["*.*.*"]       # publish only — bare SemVer e.g. 1.2.1
  workflow_run:
    workflows: ["CI"]
    types: [completed]
    branches: [main]      # docs deploy — triggered after CI passes on main
  workflow_dispatch:      # manual trigger — docs only

jobs:

  # 📚 Deploy docs after CI passes on main (or manually)
  docs:
    name: 📚 Deploy Docs
    if: |
      (github.event_name == 'workflow_run' && github.event.workflow_run.conclusion == 'success') ||
      github.event_name == 'workflow_dispatch'
    uses: vertex-ai-automations/shared-workflows/.github/workflows/publish-mkdocs.yml@main
    permissions:
      contents: read
      pages: write
      id-token: write
    with:
      python-version: "3.11"
      docs-requirements: "docs/requirements.txt"
      src-layout: true       # set false if not using src/ layout
      mkdocs-strict: true    # set false to allow doc build warnings
    secrets: inherit

  # 📦 Build and publish — tag-gated inside the reusable workflow
  # Non-tag events skip all publish jobs cleanly — no if: needed here
  publish:
    name: 📦 Publish Package
    uses: vertex-ai-automations/shared-workflows/.github/workflows/python-publish.yml@main
    with:
      python-version: "3.11"
      run-tests: true
      test-command: "pytest tests/ -v --tb=short"
      publish-target: "both"       # 'both' | 'pypi' | 'testpypi'
      use-trusted-publishing: true # set false to use PYPI_API_TOKEN instead
    secrets: inherit
```

**Trigger matrix:**

| Event | `docs` | `publish` |
|---|---|---|
| CI passes on `main` | ✅ runs | ⛔ skipped (no tag) |
| CI fails on `main` | ⛔ skipped (`conclusion != 'success'`) | ⛔ skipped (no tag) |
| Push tag `1.2.1` | ⛔ skipped | ✅ testpypi → pypi |
| Manual dispatch | ✅ runs | ⛔ skipped (no tag) |

> **Why `workflow_run` instead of `push: branches: [main]`?**
> Using `push: branches: [main]` fires the docs job in parallel with CI — docs can deploy from a broken commit. `workflow_run` chains the two workflows so docs only deploys once CI has passed. The `if: conclusion == 'success'` condition on the docs job blocks deployment on CI failure.

**To release a new version:**

```bash
# Tag and push — that's it. setuptools_scm reads the tag for the version.
git tag 1.2.1
git push origin 1.2.1
```

> **Note:** `permissions:` is only required on the `docs` job (MkDocs needs `pages: write`).
> The `publish` job has no `permissions:` block — they are declared inside `python-publish.yml`.

---

## 🔁 CI Quick Start

Create `.github/workflows/ci.yml` in your project repo to run tests, linting, and type checking on every push and pull request:

```yaml
name: 🔬 CI

on:
  push:
    branches: [main]
    tags-ignore: ["*.*.*"]   # tag pushes are handled by release.yml
  pull_request:
    branches: [main]

jobs:

  # 🧪 Cross-platform test matrix — override os-matrix/python-versions per project
  test:
    name: 🧪 Test
    uses: vertex-ai-automations/shared-workflows/.github/workflows/test.yml@main
    with:
      python-versions: '["3.9", "3.10", "3.11", "3.12"]'
      os-matrix: '["ubuntu-latest", "macos-latest", "windows-latest"]'
      install-command: 'pip install -e ".[test]"'
      test-command: 'pytest tests/ -v --tb=short'

  # 🔍 Ruff lint + format check
  lint:
    name: 🔍 Lint
    uses: vertex-ai-automations/shared-workflows/.github/workflows/lint.yml@main
    with:
      lint-args: 'src/ tests/'

  # 🔎 Static type checking
  typecheck:
    name: 🔎 Type Check
    uses: vertex-ai-automations/shared-workflows/.github/workflows/typecheck.yml@main
    with:
      install-command: 'pip install -e ".[dev]"'
      typecheck-command: 'mypy src/ --strict'

  # 🔒 Dependency security audit
  audit:
    name: 🔒 Audit
    uses: vertex-ai-automations/shared-workflows/.github/workflows/audit.yml@main
    with:
      install-command: 'pip install -e ".[test]"'

  # 📊 Coverage report — posted to the Actions job summary
  coverage:
    name: 📊 Coverage
    uses: vertex-ai-automations/shared-workflows/.github/workflows/coverage.yml@main
    with:
      install-command: 'pip install -e ".[test]"'
      coverage-source: 'src/'
      coverage-threshold: 80
```

**Narrowing the matrix per project:**

```yaml
# Linux-only project — 4 jobs instead of 12
test:
  uses: vertex-ai-automations/shared-workflows/.github/workflows/test.yml@main
  with:
    os-matrix: '["ubuntu-latest"]'
    python-versions: '["3.11", "3.12"]'
    install-command: 'pip install -e ".[test]"'
```

**Multi-step install (extras + requirements file):**

```yaml
test:
  uses: vertex-ai-automations/shared-workflows/.github/workflows/test.yml@main
  with:
    install-command: 'pip install -e ".[tui,fastapi]" && pip install -r tests/requirements.txt'
    test-command: 'pytest tests/ -v --tb=short'
```

**Weekly scheduled security scan (audit.yml):**

```yaml
# .github/workflows/audit.yml  (in your project repo)
name: 🔒 Security Audit

on:
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 9 * * 1'   # every Monday at 09:00 UTC

jobs:
  audit:
    uses: vertex-ai-automations/shared-workflows/.github/workflows/audit.yml@main
    with:
      install-command: 'pip install -e ".[test]"'
      # suppress an advisory while waiting for upstream fix:
      # pip-audit-args: '--ignore-vuln GHSA-xxxx-yyyy-zzzz'
```

**Pinning Ruff to a specific version:**

```yaml
lint:
  uses: vertex-ai-automations/shared-workflows/.github/workflows/lint.yml@main
  with:
    ruff-version: '0.4.4'
    lint-args: 'src/'
```

**Using pyright instead of mypy:**

```yaml
typecheck:
  uses: vertex-ai-automations/shared-workflows/.github/workflows/typecheck.yml@main
  with:
    install-command: 'pip install -e ".[dev]"'   # must include pyright
    typecheck-command: 'pyright src/'
```

---

## 🎯 Publish Target Flag

The `publish-target` input controls which registries receive the built package:

| Value | Behaviour |
|---|---|
| `"both"` *(default)* | Publishes to TestPyPI first, then PyPI. PyPI only runs if TestPyPI succeeds. |
| `"pypi"` | Publishes to PyPI only. TestPyPI job is skipped. |
| `"testpypi"` | Publishes to TestPyPI only. PyPI job is skipped. |

**Interaction with `run-tests`:**

| `publish-target` | `run-tests` | `test` | `publish-testpypi` | `publish-pypi` |
|---|---|---|---|---|
| `both` | `true` | ✅ (non-blocking) | ✅ | ✅ waits for testpypi |
| `both` | `false` | ⛔ skipped | ⛔ skipped | ✅ runs directly |
| `pypi` | `true` | ✅ (non-blocking) | ⛔ skipped | ✅ |
| `pypi` | `false` | ⛔ skipped | ⛔ skipped | ✅ |
| `testpypi` | `true` | ✅ (non-blocking) | ✅ | ⛔ skipped |
| `testpypi` | `false` | ⛔ skipped | ⛔ skipped | ⛔ skipped |

> **TestPyPI uses `TEST_PYPI_API_TOKEN` always** — token auth only, no Trusted Publishing.
> **PyPI uses Trusted Publishing by default** — set `use-trusted-publishing: false` to use `PYPI_API_TOKEN` instead.

**TestPyPI gotchas:**

- TestPyPI and PyPI are completely independent systems. Accounts, packages, and tokens are not shared.
- TestPyPI rejects duplicate versions. Each upload must have a unique version — you cannot re-push the same tag.
- TestPyPI doesn't mirror all PyPI dependencies. To install from TestPyPI with fallback to PyPI:
  ```bash
  pip install --index-url https://test.pypi.org/simple/ \
              --extra-index-url https://pypi.org/simple/ \
              your-package
  ```

---

## 🏷️ setuptools_scm Versioning

These workflows are designed for projects that use [`setuptools_scm`](https://github.com/pypa/setuptools_scm) for dynamic versioning — the version is derived from the git tag at build time rather than hardcoded in `pyproject.toml`.

**Typical `pyproject.toml` setup:**

```toml
[project]
dynamic = ["version"]           # no static version = "x.x.x" here

[tool.setuptools_scm]
version_file = "src/your_package/_version.py"   # stamped at build time
```

**How the workflow handles this:**

The `validate-version` job performs three checks before anything is built:

1. **Format check** — asserts the tag matches `X.Y.Z` (no `.devN`, `rc`, or `+local` suffixes that PyPI would reject)
2. **Resolution check** — installs `setuptools_scm` and calls `get_version()` to confirm the tag is reachable
3. **Exact match check** — if the tag is not on the HEAD commit, `setuptools_scm` appends `.devN+gABCDEF` — this catches that and tells you how to fix it

All three checkout steps (`validate-version`, `test`, `build`) use `fetch-depth: 0` and `fetch-tags: true` — required so `setuptools_scm` can walk the full tag history. A shallow clone always produces `0.1.dev0`.

**If the validation fails:**

```
❌ Version mismatch!
   Tag version      : 1.2.1
   Resolved version : 1.2.1.dev3+g4f2c1a0

   setuptools_scm appended a dev suffix — the tag is likely not on HEAD.
   Delete the tag, ensure HEAD is the release commit, then re-tag:
   git tag 1.2.1 && git push origin 1.2.1
```

Fix:

```bash
git tag -d 1.2.1               # delete local tag
git push origin :refs/tags/1.2.1   # delete remote tag
# ensure the commit you want is HEAD, then:
git tag 1.2.1
git push origin 1.2.1
```

---

## 🛠️ Setup Guide

### 1. Organization Settings

Allow shared workflows to be called from other repos in the org:

1. Go to **`github.com/vertex-ai-automations`** → **Settings**
2. Navigate to **Actions** → **General**
3. Under **"Access"**, select:
   > ✅ Accessible from repositories in the **vertex-ai-automations** organization
4. Click **Save**

---

### 2. PyPI Trusted Publishing (Recommended)

Trusted Publishing uses OIDC — no API token needed, no secret to rotate or leak.

**Step 1:** Log into [pypi.org](https://pypi.org) and navigate to your project.

**Step 2:** Go to **Managing** → **Publishing** → **Add a new publisher**

**Step 3:** Fill in the form:

| Field | Value |
|---|---|
| Owner | `vertex-ai-automations` |
| Repository name | `your-project-repo` |
| Workflow name | `release.yml` |
| Environment name | `pypi` |

**Step 4:** In your project repo, create a GitHub Environment named `pypi`:
- **Settings** → **Environments** → **New environment** → name it `pypi`
- Optionally add **Required reviewers** for a manual approval gate before every release

> Trusted Publishing is only configured for the real PyPI. TestPyPI always uses `TEST_PYPI_API_TOKEN`.

---

### 3. GitHub Pages

Enable Pages in each project repo that uses the MkDocs workflow:

1. Go to **Settings** → **Pages**
2. Under **Source**, select: **GitHub Actions**
3. Save — no branch selection needed

> The `docs` job in your caller workflow must declare `permissions: pages: write` and `id-token: write`.
> This is the only job that requires caller-side permissions — the `publish` job handles its own internally.

---

### 4. Secrets & Environments

**GitHub Environments** (create once per project repo):

| Environment | Purpose | Reviewers |
|---|---|---|
| `pypi` | Real PyPI publish gate | Optional — add for manual approval |
| `test-pypi` | TestPyPI sandbox | None needed |

Settings → Environments → New environment.

**Secrets** (add to your project repo, not here):

| Secret | When required |
|---|---|
| `TEST_PYPI_API_TOKEN` | `publish-target: testpypi` or `both` |
| `PYPI_API_TOKEN` | `use-trusted-publishing: false` only |

To add a secret: **Settings** → **Secrets and variables** → **Actions** → **New repository secret**

Generate tokens at:
- [pypi.org/manage/account/token](https://pypi.org/manage/account/token/)
- [test.pypi.org/manage/account/token](https://test.pypi.org/manage/account/token/)

> Keep `TEST_PYPI_API_TOKEN` and `PYPI_API_TOKEN` as separate secrets — they are issued by different systems.

Pass both to the workflow with `secrets: inherit` in your caller — the template reads them by name.

---

### 5. Docs Requirements File

Create `docs/requirements.txt` in your project with pinned versions for reproducible builds:

```txt
# docs/requirements.txt
mkdocs-material==9.5.18
mkdocstrings[python]==0.24.3
black==24.4.0
# Add other MkDocs plugins as needed:
# mkdocs-git-revision-date-localized-plugin==1.2.4
```

> If this file is missing the workflow falls back to unpinned installs and prints a warning. Acceptable for getting started, not recommended for production.

---

## 📖 Input Reference

### `publish-mkdocs.yml`

| Input | Type | Default | Description |
|---|---|---|---|
| `python-version` | `string` | `"3.11"` | Python version for building docs |
| `docs-requirements` | `string` | `"docs/requirements.txt"` | Path to pip requirements file. If the file exists it is installed; otherwise defaults are used. |
| `src-layout` | `boolean` | `false` | Add `src/` to `PYTHONPATH` |
| `mkdocs-strict` | `boolean` | `true` | Fail on MkDocs warnings |
| `working-directory` | `string` | `"."` | Root directory (for monorepos) |
| `default-branch` | `string` | `"main"` | Branch name that triggers doc builds on push. Set to `master` or your repo's default branch if not `main`. |

**Required caller permissions:**

```yaml
permissions:
  contents: read
  pages: write
  id-token: write
```

---

### `publish-zensical.yml`

| Input | Type | Default | Description |
|---|---|---|---|
| `python-version` | `string` | `"3.11"` | Python version (must be 3.10+) |
| `docs-requirements` | `string` | `"requirements-docs.txt"` | Path to pip requirements file |
| `src-layout` | `boolean` | `false` | Add `src/` to `PYTHONPATH` |
| `working-directory` | `string` | `"."` | Root directory (for monorepos) |
| `default-branch` | `string` | `"main"` | Branch name that triggers doc builds on push |

**Required caller permissions:**

```yaml
permissions:
  contents: read
  pages: write
  id-token: write
```

---

### `python-publish.yml`

| Input | Type | Default | Description |
|---|---|---|---|
| `python-version` | `string` | `"3.11"` | Python version for build and test |
| `run-tests` | `boolean` | `true` | Run tests before publishing. Set `false` to skip tests and TestPyPI entirely. |
| `test-command` | `string` | `"pytest"` | Test command to execute |
| `fail-on-test-failure` | `boolean` | `false` | Set `true` to block publishing when tests fail. Default `false` — tests are advisory. |
| `publish-target` | `string` | `"both"` | `"both"` → testpypi then pypi · `"pypi"` → pypi only · `"testpypi"` → testpypi only |
| `use-trusted-publishing` | `boolean` | `true` | Use OIDC Trusted Publishing for PyPI. Set `false` to use `PYPI_API_TOKEN` instead. TestPyPI always uses `TEST_PYPI_API_TOKEN` regardless. |
| `working-directory` | `string` | `"."` | Root directory (for monorepos) |

**Secrets:**

| Secret | Required | Description |
|---|---|---|
| `TEST_PYPI_API_TOKEN` | When `publish-target` is `testpypi` or `both` | Token from test.pypi.org |
| `PYPI_API_TOKEN` | When `use-trusted-publishing: false` | Token from pypi.org |

Pass both via `secrets: inherit` — the workflow reads them by name.

**Permissions:** Declared inside `python-publish.yml` — **no `permissions:` block needed on the caller job**.

> **`artifact-name` is not an input.** The name is automatically set to `{repo-name}-dist` (e.g., `custom-logger-dist`).

---

### `test.yml`

| Input | Type | Default | Description |
|---|---|---|---|
| `python-versions` | `string` (JSON array) | `'["3.9","3.10","3.11","3.12"]'` | Python versions to test. Pass as a JSON array string. |
| `os-matrix` | `string` (JSON array) | `'["ubuntu-latest","macos-latest","windows-latest"]'` | Runner OS labels. Pass as a JSON array string. |
| `install-command` | `string` | `'pip install -e ".[test]"'` | Full install command. Chain steps with `&&` (bash syntax). |
| `test-command` | `string` | `'pytest'` | Command to run the test suite. |
| `fail-fast` | `boolean` | `false` | Cancel remaining matrix jobs when one fails. |
| `working-directory` | `string` | `"."` | Root directory (for monorepos). |

> **No `permissions:` block needed on the caller job** — `contents: read` is declared inside `test.yml`.

---

### `lint.yml`

| Input | Type | Default | Description |
|---|---|---|---|
| `python-version` | `string` | `"3.11"` | Python version for the runner. |
| `ruff-version` | `string` | `""` | Pin Ruff to a specific version (e.g. `"0.4.4"`). Omit for latest. |
| `lint-args` | `string` | `"."` | Paths or flags passed to both `ruff check` and `ruff format --check`. |
| `working-directory` | `string` | `"."` | Root directory (for monorepos). |

> **No `permissions:` block needed on the caller job** — `contents: read` is declared inside `lint.yml`.

---

### `typecheck.yml`

| Input | Type | Default | Description |
|---|---|---|---|
| `python-version` | `string` | `"3.11"` | Python version for the runner. |
| `typecheck-command` | `string` | `"mypy src/"` | Type checker command. Accepts any checker: `pyright`, `basedpyright src/`, etc. |
| `install-command` | `string` | `'pip install -e ".[dev]"'` | Install command. Must install the type checker and any required stubs. |
| `working-directory` | `string` | `"."` | Root directory (for monorepos). |

> **No `permissions:` block needed on the caller job** — `contents: read` is declared inside `typecheck.yml`.

---

### `audit.yml`

| Input | Type | Default | Description |
|---|---|---|---|
| `python-version` | `string` | `"3.11"` | Python version for the runner. |
| `install-command` | `string` | `'pip install -e "."'` | Installs the project so all transitive deps are present for scanning. |
| `pip-audit-args` | `string` | `""` | Extra flags passed to `pip-audit` (e.g. `--ignore-vuln GHSA-xxxx-yyyy-zzzz`). |
| `working-directory` | `string` | `"."` | Root directory (for monorepos). |

> **No `permissions:` block needed on the caller job** — `contents: read` is declared inside `audit.yml`.

---

### `coverage.yml`

| Input | Type | Default | Description |
|---|---|---|---|
| `python-version` | `string` | `"3.11"` | Python version for the runner. |
| `install-command` | `string` | `'pip install -e ".[test]"'` | Installs the project and test dependencies. `pytest-cov` is installed automatically on top. |
| `test-command` | `string` | `"pytest"` | Base pytest command. `--cov`, `--cov-report=term-missing`, and `--cov-report=json` are appended automatically. |
| `coverage-source` | `string` | `"src/"` | Path passed to `--cov` (e.g. `src/mypackage`). |
| `coverage-threshold` | `number` | `0` | Minimum total coverage %. Job fails if below this. Set `0` to disable. |
| `working-directory` | `string` | `"."` | Root directory (for monorepos). |

> **No `permissions:` block needed on the caller job** — `contents: read` is declared inside `coverage.yml`.

---

## 🔁 Migration Guide

If your project has existing inline workflow files, follow these steps.

**Step 1:** Complete the [Setup Guide](#-setup-guide) for your project repo.

**Step 2:** Delete old workflow files:

```bash
git rm .github/workflows/publish-mkdocs.yml
git rm .github/workflows/python-publish.yml
```

**Step 3:** Create `.github/workflows/release.yml` from the [Release Quick Start](#-release-quick-start) example above.

**Step 4:** Remove any `artifact-name`, `repository-url`, or `use-trusted-publishing` inputs from old callers — these have been replaced by `publish-target`.

**Step 5:** Remove `permissions:` blocks from your `publish` job — they are now declared inside the reusable workflow. Keep `permissions:` on the `docs` job.

**Step 6:** If your project uses `setuptools_scm`, remove any version comparison logic — the workflow handles this automatically. See [setuptools_scm Versioning](#-setuptools_scm-versioning).

**Step 7:** Confirm **Settings** → **Pages** → Source is **GitHub Actions** (not a branch).

**Step 8:** Push and verify in the **Actions** tab.

---

## 🔍 Troubleshooting

### ❌ `Error: caller workflow does not have permission to use 'vertex-ai-automations/shared-workflows'`

The organization hasn't granted access to this shared repo.
→ Follow [Organization Settings](#1-organization-settings) above.

---

### ❌ `The workflow is requesting 'pages: write', but is only allowed 'pages: none'`

The caller workflow is missing the required permissions on the `docs` job.
→ Add this to the `docs` job in your `release.yml`:

```yaml
docs:
  uses: vertex-ai-automations/shared-workflows/.github/workflows/publish-mkdocs.yml@main
  permissions:
    contents: read
    pages: write
    id-token: write
```

---

### ❌ `Creating Pages deployment failed — Not Found (404)`

GitHub Pages hasn't been enabled on the project repo yet.
→ Go to **Settings** → **Pages** → **Source** → select **GitHub Actions**, then re-run the failed job.

---

### ❌ `setuptools_scm could not resolve a version`

The tag hasn't been pushed yet when the workflow runs, or the repo was checked out as a shallow clone.
→ Ensure the tag is pushed before the workflow triggers, and that `fetch-depth: 0` is in your checkout step (already set in this template).

---

### ❌ `Version mismatch — setuptools_scm resolved 1.2.1.dev3+g4f2c1a0`

The tag was pushed on a commit that is not HEAD. `setuptools_scm` appended a dev suffix.
→ Delete the tag, confirm the correct commit is HEAD, and re-tag:

```bash
git tag -d 1.2.1
git push origin :refs/tags/1.2.1
git tag 1.2.1
git push origin 1.2.1
```

---

### ❌ `Tag '1.2.1rc1' is not valid SemVer (expected X.Y.Z)`

The tag contains a pre-release suffix. PyPI accepts these but the org convention is clean `X.Y.Z` tags.
→ Delete the tag and re-tag with a plain version: `git tag 1.2.1`

---

### ❌ `tar: site: Cannot open: No such file or directory`

`mkdocs build` failed or `mkdocs.yml` is missing, so no `site/` folder was produced.
→ Check:
1. Does `mkdocs.yml` exist at the repo root?
2. Does `docs/index.md` exist?
3. Did the **🔨 Build MkDocs site** step succeed in the logs?

Add this debug step after `mkdocs build` to inspect the runner filesystem:

```yaml
- name: 🐛 Debug — list files after build
  if: always()
  run: |
    echo "--- Repo root ---" && ls -la
    echo "--- Site folder ---"
    ls -la site/ 2>/dev/null || echo "⚠️ site/ folder does not exist"
    echo "--- MkDocs config ---"
    cat mkdocs.yml 2>/dev/null || echo "⚠️ mkdocs.yml not found"
```

---

### ❌ MkDocs build fails with `--strict`

A warning is being treated as a build error. Common causes: broken internal links, missing `mkdocstrings` docstrings, or plugin misconfiguration.
→ Temporarily disable strict mode while debugging:

```yaml
with:
  mkdocs-strict: false
```

---

### ❌ TestPyPI publish fails with 400 Bad Request

Common causes:

- **Duplicate version** — TestPyPI rejects re-uploads of the same version. You cannot re-push a tag that was already uploaded. Use a new tag.
- **Package not yet registered on TestPyPI** — upload manually once via `twine upload --repository testpypi dist/*` to create the project entry.
- **Wrong token** — confirm `TEST_PYPI_API_TOKEN` was generated at [test.pypi.org](https://test.pypi.org), not pypi.org.

---

### ❌ `publish-testpypi` is skipped unexpectedly

This job is skipped when `run-tests: false` OR when `publish-target: pypi`. Check both inputs in your caller workflow.

---

### ❌ Test matrix fails with `fromJson: invalid JSON`

This happens when triggering `test.yml` via `workflow_dispatch` with an empty `os-matrix` or `python-versions` field. The `workflow_dispatch` input defaults are independent from `workflow_call` defaults.
→ Always provide a valid JSON array when triggering manually, e.g. `["ubuntu-latest"]` or `["3.11"]`.

---

### ❌ Tests pass on Ubuntu but fail on Windows with install errors

The `install-command` uses bash syntax (e.g. `&&`, single quotes). Windows runners use PowerShell by default.
→ `test.yml` sets `shell: bash` on the install and test steps — this uses Git Bash on Windows. Ensure your install command uses bash syntax, not PowerShell syntax.

---

### ❌ `ruff: command not found`

The `lint.yml` workflow installs Ruff before running — this error should not occur unless the install step failed.
→ Check the **📦 Install ruff** step in the job logs for the root cause (network error, invalid `ruff-version`, etc.).

---

### ❌ `mypy: command not found` (or `pyright: command not found`)

The type checker is not included in the extras installed by `install-command`.
→ Ensure the type checker is declared as a dependency in the extras group you're installing. For example, add `mypy` to `[project.optional-dependencies] dev` in `pyproject.toml`.

---

### ❌ `pip-audit` fails with a known/accepted vulnerability

A CVE is present in a dependency but has been reviewed and accepted (e.g. it affects an unused code path, or a fix isn't available yet).
→ Pass the advisory ID to `pip-audit-args`:
```yaml
pip-audit-args: '--ignore-vuln GHSA-xxxx-yyyy-zzzz'
```
Document the suppression reason in your caller workflow as a comment.

---

### ❌ `pip-audit` reports vulnerabilities in dev/test dependencies

The `install-command` installs optional extras that pull in vulnerable packages not present in production.
→ Use a minimal install command that reflects the production dependency set:
```yaml
install-command: 'pip install -e "."'   # base deps only, no extras
```

---

### ❌ Coverage summary step is skipped or shows "coverage.json not found"

The `test-command` ran but `pytest-cov` was not loaded, so `coverage.json` was never written.
→ Ensure `test-command` does not override the `--cov` flags. The workflow appends them automatically — do not include `--no-cov` in your command.
→ Check that `coverage-source` matches a real path in the project. An incorrect path causes `pytest-cov` to produce an empty report.

---

### ❌ Coverage threshold check fails unexpectedly

`coverage-threshold` is set but the actual coverage is slightly below the value due to rounding.
→ `coverage.json` stores `percent_covered_display` as a rounded integer. If actual coverage is `79.6%` and threshold is `80`, the check fails. Lower the threshold by 1 or fix the missing coverage.

---

### ❌ mypy reports `Cannot find implementation or library stub for module`

Type stubs for a dependency are missing.
→ Install the relevant stub package (e.g. `types-requests`, `pandas-stubs`) in your `dev` extras, then pass `install-command: 'pip install -e ".[dev]"'` to `typecheck.yml`.

---

## 🤝 Contributing

All changes to workflows in this repo affect every project in the org. Please follow this process:

1. **Open an issue** describing the proposed change and which workflows it affects
2. **Create a branch** from `main` (e.g., `feat/add-caching-to-mkdocs`)
3. **Test locally** where possible — use [`act`](https://github.com/nektos/act) to run workflows locally before pushing
4. **Open a pull request** — require at least one review from a team member before merging
5. **Tag a release** after merging so consuming projects can pin to the new version

> Breaking changes (removing or renaming inputs) **must** increment the major version tag and be documented in a changelog entry.

---

## 📌 Versioning & Pinning

This repo uses bare [SemVer](https://semver.org/) tags (e.g., `1.2.0`) — no `v` prefix, consistent with the org convention.

**Pin to a tag for stability:**

```yaml
uses: vertex-ai-automations/shared-workflows/.github/workflows/publish-mkdocs.yml@1.2.0
uses: vertex-ai-automations/shared-workflows/.github/workflows/python-publish.yml@1.2.0
```

**Use `@main` for always-latest:**

```yaml
uses: vertex-ai-automations/shared-workflows/.github/workflows/publish-mkdocs.yml@main
uses: vertex-ai-automations/shared-workflows/.github/workflows/python-publish.yml@main
```

**Tag a new release (maintainers only):**

```bash
git tag -a 1.2.0 -m "Release 1.2.0: add publish-target flag, move permissions into template"
git push origin 1.2.0
```

| Strategy | Use case |
|---|---|
| `@main` | Teams that want automatic updates |
| `@1` | Major version floating tag — gets patches and minor updates |
| `@1.2.0` | Fully pinned — maximum stability, manual upgrade required |

---

## 📄 License

[MIT](./LICENSE) — workflows in this repo may be freely used, modified, and distributed within your organization.
