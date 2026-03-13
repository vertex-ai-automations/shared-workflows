# 🏗️ Shared GitHub Actions Workflows

> Centralized, reusable CI/CD workflow templates for all projects in the **your-org** GitHub Organization.

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
- [Publishing to TestPyPI](#-publishing-to-testpypi)
  - [How It Works](#how-it-works)
  - [TestPyPI Setup](#testpypi-setup)
  - [Recommended Pattern: Test Then Release](#recommended-pattern-test-then-release)
  - [TestPyPI Gotchas](#testpypi-gotchas)
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

This repository (`your-org/.github`) is the **single source of truth** for CI/CD pipelines across the organization. Instead of copy-pasting workflow YAML into every project, each project calls a versioned template from here using GitHub's [reusable workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows) feature.

**Benefits:**

- 🔒 **Security patches** propagate to all projects by updating one file
- 🔁 **DRY principle** — no duplicated workflow logic across repos
- 📌 **Version pinning** — projects can pin to a tag (e.g., `@v1.2.0`) for stability
- 🧩 **Configurable** — every workflow exposes `inputs` to customize behavior per project
- 👁️ **Auditable** — one place to review and approve CI/CD changes

---

## 📁 Repository Structure

```
your-org/.github/                   ← This repo (must be named ".github")
│
├── .github/
│   └── workflows/                  ← All reusable workflow templates live here
│       ├── publish-mkdocs.yml      ← MkDocs → GitHub Pages deployment
│       └── python-publish.yml      ← Python package → PyPI publishing
│
├── docs/                           ← Documentation for this repo itself
│   └── adr/                        ← Architecture Decision Records
│
├── README.md                       ← You are here
└── LICENSE
```

> **Why `.github/workflows/` inside a repo named `.github`?**
> GitHub specifically recognizes the path `your-org/.github/.github/workflows/` as the org-level shared workflow location. The double `.github` is intentional — the first is the repo name, the second is the folder path within it.

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
| Uses `actions/deploy-pages` instead of `peaceiris/actions-gh-pages` | Official action; avoids third-party dependency for a critical deployment step |
| `permissions: pages: write` (not `contents: write`) | Least-privilege; the workflow cannot push arbitrary commits to the repo |
| `concurrency: cancel-in-progress: false` | Prevents a fast push from cancelling an in-flight deploy; queues instead |
| Falls back gracefully if no requirements file found | Won't hard-fail new projects that haven't set up `docs/requirements.txt` yet |

---

### 📦 Build and Publish Python Package to PyPI

**File:** `.github/workflows/python-publish.yml`

A multi-job pipeline that validates, tests, builds, and publishes a Python package to PyPI (or TestPyPI).

**What it does:**

```
validate-version ──┐
                   ├──▶ build ──▶ publish-trusted  (OIDC, recommended)
test ──────────────┘         └──▶ publish-token    (API token, fallback)
```

1. **`validate-version`** — Extracts the version from `pyproject.toml` or `setup.cfg` and asserts it matches the git tag. Prevents mis-tagged releases.
2. **`test`** — Runs your test suite with a configurable command before any artifact is built.
3. **`build`** — Installs `build` + `twine`, runs `python -m build`, validates metadata with `twine check`, and uploads artifacts.
4. **`publish-trusted`** *(default)* — Uses OIDC Trusted Publishing (no secrets required). Runs inside a protected `pypi` GitHub Environment.
5. **`publish-token`** *(fallback)* — Uses `PYPI_API_TOKEN` secret for projects that haven't set up Trusted Publishing yet.

**Key design decisions:**

| Decision | Reason |
|---|---|
| Version validation job | PyPI releases cannot be deleted or overwritten; a version mismatch is caught before any artifact is built |
| Tests run before build | Catches regressions before a broken package is uploaded |
| Trusted Publishing as default | More secure than long-lived API tokens; tokens can be rotated or leaked |
| `environment: pypi` enabled | Enforces GitHub's manual approval gate and deployment protection rules |
| No `--verbose` on twine | Verbose output can expose token fragments in logs |
| Separate build and publish jobs | Artifacts are immutable between jobs; what was tested is exactly what was published |

---

## 🚀 Quick Start

Add this file to your project repo at `.github/workflows/release.yml`.

**Minimal setup** (docs + PyPI publish on tag only):

```yaml
name: 🚢 Release

on:
  push:
    branches: [main]       # triggers docs deploy
    tags: ["v*.*.*"]       # triggers PyPI publish

jobs:

  docs:
    uses: your-org/.github/.github/workflows/publish-mkdocs.yml@main
    with:
      python-version: "3.11"
      docs-requirements: "docs/requirements.txt"
      src-layout: true
    secrets: inherit

  publish:
    if: startsWith(github.ref, 'refs/tags/v')
    uses: your-org/.github/.github/workflows/python-publish.yml@main
    with:
      python-version: "3.11"
      artifact-name: "my-package-dist"
      run-tests: true
      test-command: "pytest tests/ -v"
      use-trusted-publishing: true
    secrets: inherit
```

> Want to also validate against **TestPyPI** on every merge? See [Publishing to TestPyPI → Recommended Pattern](#recommended-pattern-test-then-release) for a complete example with both jobs.

---

## 🧪 Publishing to TestPyPI

The `python-publish.yml` workflow supports publishing to [TestPyPI](https://test.pypi.org) — a fully separate sandbox registry — via the `repository-url` input. This lets you validate your packaging pipeline and package metadata before committing to a real PyPI release.

### How It Works

The `repository-url` input controls which registry receives the upload:

```yaml
# Default — publishes to the real PyPI
repository-url: "https://upload.pypi.org/legacy/"

# Override — publishes to TestPyPI sandbox instead
repository-url: "https://test.pypi.org/legacy/"
```

Everything else in the pipeline (version validation, tests, build, metadata check) runs identically regardless of target registry.

---

### TestPyPI Setup

TestPyPI and PyPI are **completely independent systems**. Accounts, packages, and Trusted Publishers are not shared between them. You must set each up separately.

#### Option A — Trusted Publishing on TestPyPI (recommended)

1. Register or log in at [test.pypi.org](https://test.pypi.org)
2. Navigate to your project → **Managing** → **Publishing** → **Add a new publisher**
3. Fill in the same fields as your PyPI publisher, but point to the `test-publish` environment:

| Field | Value |
|---|---|
| Owner | `your-org` |
| Repository name | `your-project-repo` |
| Workflow name | `release.yml` *(your caller workflow)* |
| Environment name | `test-pypi` |

4. Create a matching GitHub Environment named `test-pypi` in your project repo:
   - **Settings** → **Environments** → **New environment** → name it `test-pypi`
   - No required reviewers needed (it's just a sandbox)

#### Option B — API Token Fallback

1. Generate a token at [test.pypi.org/manage/account/token](https://test.pypi.org/manage/account/token/)
2. Add it as a repo secret named `TEST_PYPI_API_TOKEN` in your project repo
3. Use it in your caller workflow:

```yaml
with:
  use-trusted-publishing: false
  repository-url: "https://test.pypi.org/legacy/"
secrets:
  PYPI_API_TOKEN: ${{ secrets.TEST_PYPI_API_TOKEN }}
```

> Keep `TEST_PYPI_API_TOKEN` and `PYPI_API_TOKEN` as separate secrets — they are issued by different systems and should never be mixed up.

---

### Recommended Pattern: Test Then Release

The cleanest approach is two jobs in your caller workflow — one that fires on every merge to `main` (TestPyPI) and one that fires only on version tags (real PyPI). This ensures every merge proves the package is publishable before you ever cut a release.

```yaml
name: 🚢 Release

on:
  push:
    branches: [main]      # test publish on every merge
    tags: ["v*.*.*"]      # real publish only on version tags

jobs:

  docs:
    uses: your-org/.github/.github/workflows/publish-mkdocs.yml@main
    with:
      python-version: "3.11"
      docs-requirements: "docs/requirements.txt"
      src-layout: true
    secrets: inherit

  # ✅ Runs on every push to main — validates the full pipeline against TestPyPI
  test-publish:
    if: github.ref == 'refs/heads/main'
    uses: your-org/.github/.github/workflows/python-publish.yml@main
    with:
      artifact-name: "my-package-dist-test"       # must differ from prod artifact name
      run-tests: true
      test-command: "pytest tests/ -v --tb=short"
      use-trusted-publishing: true                 # or false + TEST_PYPI_API_TOKEN
      repository-url: "https://test.pypi.org/legacy/"
    secrets: inherit

  # 🚀 Runs only on version tags — publishes to real PyPI
  publish:
    if: startsWith(github.ref, 'refs/tags/v')
    uses: your-org/.github/.github/workflows/python-publish.yml@main
    with:
      artifact-name: "my-package-dist"
      run-tests: true
      test-command: "pytest tests/ -v --tb=short"
      use-trusted-publishing: true
      repository-url: "https://upload.pypi.org/legacy/"
    secrets: inherit
```

**What this gives you:**

| Event | Docs | TestPyPI | PyPI |
|---|---|---|---|
| Push to `main` | ✅ deployed | ✅ published | ⛔ skipped |
| Push `v1.2.3` tag | ⛔ skipped | ⛔ skipped | ✅ published |

---

### TestPyPI Gotchas

> **Version uniqueness** — TestPyPI rejects duplicate versions just like PyPI. Each upload must have a unique version number. Use a `.devN` suffix (e.g., `1.2.3.dev1`) for test builds, or automate it with the build number:
> ```toml
> # pyproject.toml — bump this before each test upload
> version = "1.2.3.dev1"
> ```

> **Package not installable** — TestPyPI doesn't mirror all of PyPI's dependencies. `pip install --index-url https://test.pypi.org/simple/ my-package` may fail if your dependencies aren't also on TestPyPI. Use `--extra-index-url https://pypi.org/simple/` to fall back to real PyPI for dependencies.

> **Trusted Publisher must be registered on TestPyPI separately** — even if your PyPI Trusted Publisher is set up correctly, TestPyPI will reject OIDC tokens unless you've added a publisher there too.

> **`artifact-name` must differ** — if your `test-publish` and `publish` jobs run in the same workflow, they must use different `artifact-name` values to avoid upload collisions.

---

## 🛠️ Setup Guide

### 1. Organization Settings

Allow the shared workflows to be called from other repos in the org:

1. Go to **`github.com/your-org`** → **Settings**
2. Navigate to **Actions** → **General**
3. Under **"Access"**, select:
   > ✅ Accessible from repositories in the **your-org** organization
4. Click **Save**

> If this setting is missing, your org may be on a plan that doesn't support it. Check GitHub's [pricing page](https://github.com/pricing) for reusable workflow availability.

---

### 2. PyPI Trusted Publishing (Recommended)

Trusted Publishing uses OIDC — no API token needed, no secret to rotate or leak.

**Step 1:** Log into [pypi.org](https://pypi.org) and navigate to your project.

**Step 2:** Go to **Managing** → **Publishing** → **Add a new publisher**

**Step 3:** Fill in the form:

| Field | Value |
|---|---|
| Owner | `your-org` |
| Repository name | `your-project-repo` |
| Workflow name | `release.yml` *(the caller workflow, not the template)* |
| Environment name | `pypi` |

**Step 4:** In your **project repo**, create a GitHub Environment named `pypi`:
- Go to **Settings** → **Environments** → **New environment**
- Name it `pypi`
- Optionally add **Required reviewers** for manual approval before every release

> The `environment: pypi` block in the workflow + the matching environment name in PyPI publisher settings is what makes Trusted Publishing work.

---

### 3. GitHub Pages

Enable Pages in each project repo that uses the MkDocs workflow:

1. Go to **Settings** → **Pages**
2. Under **Source**, select: **GitHub Actions**
3. Save — no branch selection needed (the workflow handles it)

> The workflow uses `permissions: pages: write` and `id-token: write`. If your organization restricts these permissions, you may need to explicitly allow them in org-level Action settings.

---

### 4. Secrets & Environments

#### If using Trusted Publishing (recommended)
No secrets needed. Only the `pypi` GitHub Environment (see above).

#### If using API token fallback

| Secret Name | Scope | Description |
|---|---|---|
| `PYPI_API_TOKEN` | Project repo | PyPI API token scoped to the specific package |

To add the secret:
1. Go to your **project repo** → **Settings** → **Secrets and variables** → **Actions**
2. Click **New repository secret**
3. Name: `PYPI_API_TOKEN`, Value: your PyPI token (starts with `pypi-`)

> Never add the PyPI token to the `.github` shared repo — it belongs in each individual project repo.

---

### 5. Docs Requirements File

Create `docs/requirements.txt` in your project with **pinned** versions for reproducible builds:

```txt
# docs/requirements.txt
mkdocs-material==9.5.18
mkdocstrings[python]==0.24.3
black==24.4.0
# Add other MkDocs plugins as needed:
# mkdocs-git-revision-date-localized-plugin==1.2.4
```

> Without a pinned requirements file, the workflow falls back to unpinned installs and prints a warning. This is acceptable for getting started but not recommended for production.

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

### `python-publish.yml`

| Input | Type | Default | Description |
|---|---|---|---|
| `python-version` | `string` | `"3.11"` | Python version for build and test |
| `artifact-name` | `string` | `"python-package-dist"` | Name of the build artifact — **must be unique per project** |
| `run-tests` | `boolean` | `true` | Run tests before publishing |
| `test-command` | `string` | `"pytest"` | Test command to execute |
| `use-trusted-publishing` | `boolean` | `true` | Use OIDC Trusted Publishing; set `false` to use token |
| `repository-url` | `string` | `https://upload.pypi.org/legacy/` | Override to `https://test.pypi.org/legacy/` to publish to TestPyPI |
| `working-directory` | `string` | `"."` | Root directory (for monorepos) |

| Secret | Required | Description |
|---|---|---|
| `PYPI_API_TOKEN` | Only if `use-trusted-publishing: false` | PyPI **or** TestPyPI API token — keep these as separate secrets (`PYPI_API_TOKEN` vs `TEST_PYPI_API_TOKEN`) |

---

## 🔁 Migration Guide

If your project currently has its own inline `publish-mkdocs.yml` or `python-publish.yml`, follow these steps:

**Step 1:** Complete the [Setup Guide](#-setup-guide) above for your project repo.

**Step 2:** Delete the old workflow files from your project:
```bash
git rm .github/workflows/publish-mkdocs.yml
git rm .github/workflows/python-publish.yml
```

**Step 3:** Create the new caller workflow at `.github/workflows/release.yml` (see [Quick Start](#-quick-start)).

**Step 4:** Push and verify the workflow runs correctly. Check the **Actions** tab in your project repo.

**Step 5:** If migrating the MkDocs workflow, verify your GitHub Pages source is set to **GitHub Actions** (not a branch).

> **Tip:** Validate against TestPyPI before your first real release. See [Publishing to TestPyPI](#-publishing-to-testpypi) for the full recommended pattern.

---

## 🔍 Troubleshooting

### ❌ `Error: caller workflow does not have permission to use 'your-org/.github'`

The organization hasn't granted access to this shared repo.
→ Follow [Organization Settings](#1-organization-settings) above.

---

### ❌ `Version mismatch! Tag v1.2.3 does not match package version 1.2.0`

The git tag you pushed doesn't match the version in `pyproject.toml`.
→ Update `version` in `pyproject.toml`, commit, then re-tag:
```bash
git tag -d v1.2.3          # delete local tag
git push origin :v1.2.3    # delete remote tag
# update pyproject.toml, commit
git tag v1.2.3
git push origin v1.2.3
```

---

### ❌ MkDocs build fails with `--strict`

A warning in your docs is being treated as an error.
→ Check the build log for the specific warning. Common causes:
- Broken internal links (`[Page](./missing.md)`)
- Missing mkdocstrings docstrings
- Plugin configuration errors

To temporarily disable strict mode while debugging:
```yaml
with:
  mkdocs-strict: false
```

---

### ❌ GitHub Pages shows old content after deploy

GitHub Pages can take 1–2 minutes to propagate. If it persists:
1. Check **Actions** → latest workflow run → `deploy-docs` job for errors
2. Confirm **Settings** → **Pages** → Source is set to **GitHub Actions** (not a branch)
3. Hard refresh your browser (`Ctrl+Shift+R` / `Cmd+Shift+R`)

---

### ❌ `artifact-name` collision between projects

If two projects use the same `artifact-name` in the same workflow run context, artifact uploads can conflict.
→ Set a unique `artifact-name` per project:
```yaml
with:
  artifact-name: "my-specific-package-dist"
```

---

### ❌ TestPyPI publish fails with 400 error

The most common causes:

- **Duplicate version** — TestPyPI rejects re-uploads of the same version. Bump to a `.devN` suffix (e.g., `1.2.3.dev2`) and try again.
- **Package not registered on TestPyPI** — unlike PyPI, TestPyPI sometimes requires the project to exist before the first upload. Upload manually once via `twine upload --repository testpypi dist/*`.
- **Trusted Publisher not set up on TestPyPI** — even if PyPI Trusted Publishing works, TestPyPI requires its own separate publisher entry. See [TestPyPI Setup](#testpypi-setup).
- **Wrong secret used** — confirm `TEST_PYPI_API_TOKEN` was generated from [test.pypi.org](https://test.pypi.org), not pypi.org.

---

## 🤝 Contributing

All changes to workflows in this repo affect every project in the org. Please follow this process:

1. **Open an issue** describing the proposed change and which workflows it affects
2. **Create a branch** from `main` (e.g., `feat/add-caching-to-mkdocs`)
3. **Test locally** where possible; use `act` ([nektos/act](https://github.com/nektos/act)) to run workflows locally
4. **Open a pull request** — require at least **one review** from a team member before merging
5. **Tag a release** after merging so consuming projects can pin to the new version (see below)

> Breaking changes (removing or renaming inputs) **must** increment the major version tag and be documented in the changelog.

---

## 📌 Versioning & Pinning

This repo uses [semantic versioning](https://semver.org/) via git tags.

### For stability, pin to a tag in your project:

```yaml
# Pinned to a specific release — won't break on upstream changes
uses: your-org/.github/.github/workflows/publish-mkdocs.yml@v1.2.0
```

### For always-latest, use `@main`:

```yaml
# Follows HEAD — gets fixes automatically but may get breaking changes
uses: your-org/.github/.github/workflows/publish-mkdocs.yml@main
```

### Tagging a new release (maintainers only):

```bash
git tag -a v1.2.0 -m "Release v1.2.0: add monorepo working-directory support"
git push origin v1.2.0
```

| Tag strategy | Use case |
|---|---|
| `@main` | Development / teams that want automatic updates |
| `@v1` | Major version floating tag — gets patches and minor updates |
| `@v1.2.0` | Fully pinned — maximum stability, manual upgrade required |

---

## 📄 License

[MIT](./LICENSE) — workflows in this repo may be freely used, modified, and distributed within your organization.
