# 🏗️ Shared GitHub Actions Workflows

> Centralized, reusable CI/CD workflow templates for all projects in the **vertex-ai-automations** GitHub Organization.

[![Workflows](https://img.shields.io/badge/workflows-2-blue?logo=github-actions)](/.github/workflows)
[![License](https://img.shields.io/badge/license-MIT-green)](./LICENSE)
[![GitHub Actions](https://img.shields.io/badge/GitHub%20Actions-reusable-orange?logo=github)](https://docs.github.com/en/actions/using-workflows/reusing-workflows)

---

## 📋 Table of Contents

- [Overview](#-overview)
- [Repository Structure](#-repository-structure)
- [Available Workflows](#-available-workflows)
  - [Deploy MkDocs to GitHub Pages](#-deploy-mkdocs-to-github-pages)
  - [Build and Publish Python Package to PyPI](#-build-and-publish-python-package-to-pypi)
- [Quick Start](#-quick-start)
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
│       └── python-publish.yml             ← Python package → PyPI/TestPyPI publishing
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

## 🚀 Quick Start

Create `.github/workflows/release.yml` in your project repo:

```yaml
name: 🚢 Release

on:
  push:
    branches: [main]    # docs deploy only
    tags: ["*.*.*"]     # publish only — bare SemVer e.g. 1.2.1
  workflow_dispatch:    # manual trigger — docs only

jobs:

  # 📚 Deploy docs on push to main and manual trigger
  docs:
    name: 📚 Deploy Docs
    if: github.ref == 'refs/heads/main' || github.event_name == 'workflow_dispatch'
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
  # Non-tag pushes skip all publish jobs cleanly — no if: needed here
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
| Push to `main` | ✅ runs | ⛔ all publish jobs skip (no tag) |
| Push tag `1.2.1` | ⛔ skipped | ✅ testpypi → pypi |
| Manual dispatch | ✅ runs | ⛔ all publish jobs skip (no tag) |

**To release a new version:**

```bash
# Tag and push — that's it. setuptools_scm reads the tag for the version.
git tag 1.2.1
git push origin 1.2.1
```

> **Note:** `permissions:` is only required on the `docs` job (MkDocs needs `pages: write`).
> The `publish` job has no `permissions:` block — they are declared inside `python-publish.yml`.

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
| `docs-requirements` | `string` | `"docs/requirements.txt"` | Path to pip requirements file |
| `src-layout` | `boolean` | `false` | Add `src/` to `PYTHONPATH` |
| `mkdocs-strict` | `boolean` | `true` | Fail on MkDocs warnings |
| `working-directory` | `string` | `"."` | Root directory (for monorepos) |

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
| `run-tests` | `boolean` | `true` | Run tests before publishing. Failures are non-blocking (`continue-on-error`). Set `false` to skip tests and TestPyPI entirely. |
| `test-command` | `string` | `"pytest"` | Test command to execute |
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

## 🔁 Migration Guide

If your project has existing inline workflow files, follow these steps.

**Step 1:** Complete the [Setup Guide](#-setup-guide) for your project repo.

**Step 2:** Delete old workflow files:

```bash
git rm .github/workflows/publish-mkdocs.yml
git rm .github/workflows/python-publish.yml
```

**Step 3:** Create `.github/workflows/release.yml` from the [Quick Start](#-quick-start) example above.

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
