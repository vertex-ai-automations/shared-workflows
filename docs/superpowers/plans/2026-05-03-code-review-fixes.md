# Code Review Fixes Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Apply all 10 code review findings across three workflow files and the README.

**Architecture:** Pure YAML edits to three reusable GitHub Actions workflow files. No build system, no tests — validation is done by reading the final YAML. Each task is one file or one logical concern.

**Tech Stack:** GitHub Actions YAML, Bash shell steps embedded in workflow `run:` blocks.

---

## Files to Modify

| File | Changes |
|---|---|
| `.github/workflows/python-publish.yml` | 7 fixes: v-prefix error job, run-tests guard on testpypi, boolean comparisons, SHA pin, id-token scope, continue-on-error input, workflow-level permissions |
| `.github/workflows/publish-mkdocs.yml` | 4 fixes: default-branch input, docs-requirements install, artifact path site/, id-token scope |
| `.github/workflows/publish-zensical.yml` | 2 fixes: default-branch input, id-token scope |
| `README.md` | 1 fix: document publish-zensical.yml |

---

## Task 1: python-publish.yml — security and permissions

**Files:** Modify `.github/workflows/python-publish.yml`

- [ ] **Step 1: Remove `id-token: write` from workflow-level permissions block (lines 115–117)**

Change:
```yaml
permissions:
  contents: read
  id-token: write
```
To:
```yaml
permissions:
  contents: read
```

- [ ] **Step 2: Verify `publish-pypi` job already has its own `permissions: id-token: write` block**

Confirm lines ~364-365 still have:
```yaml
permissions:
  id-token: write
```

- [ ] **Step 3: Pin `pypa/gh-action-pypi-publish` to commit SHA**

Change line ~383:
```yaml
uses: pypa/gh-action-pypi-publish@release/v1
```
To:
```yaml
uses: pypa/gh-action-pypi-publish@cef221092ed1bacb1cc03d23a2d87d1d172e277b  # release/v1
```

- [ ] **Step 4: Commit**
```bash
git add .github/workflows/python-publish.yml
git commit -m "security: scope id-token to publish-pypi job only; SHA-pin pypa publish action"
```

---

## Task 2: python-publish.yml — v-prefix tag guard

**Files:** Modify `.github/workflows/python-publish.yml`

- [ ] **Step 1: Add `reject-v-prefix-tag` job immediately after the `on:` block, before `validate-version`**

Insert as the first job in the `jobs:` section:
```yaml
  # ============================================================
  # JOB 0: Reject v-prefixed tags with a clear error
  # All other jobs silently skip on non-bare-semver tags.
  # This job surfaces a helpful message instead.
  # ============================================================
  reject-v-prefix-tag:
    name: ❌ Reject v-prefixed tag
    if: startsWith(github.ref, 'refs/tags/v')
    runs-on: ubuntu-latest
    steps:
      - name: Fail with guidance
        run: |
          echo "❌ Tag '$GITHUB_REF_NAME' uses a 'v' prefix."
          echo "   This workflow requires bare SemVer tags (e.g. 1.2.3, not v1.2.3)."
          echo "   Fix:"
          echo "   git tag -d $GITHUB_REF_NAME"
          echo "   git push origin :refs/tags/$GITHUB_REF_NAME"
          echo "   git tag ${GITHUB_REF_NAME#v} && git push origin ${GITHUB_REF_NAME#v}"
          exit 1
```

- [ ] **Step 2: Commit**
```bash
git add .github/workflows/python-publish.yml
git commit -m "fix: fail explicitly on v-prefixed tags instead of silent skip"
```

---

## Task 3: python-publish.yml — test job and publish-testpypi logic

**Files:** Modify `.github/workflows/python-publish.yml`

- [ ] **Step 1: Add `fail-on-test-failure` input to `workflow_call` inputs block**

After the existing `working-directory` input, add:
```yaml
      fail-on-test-failure:
        description: >
          Set true to block publishing when tests fail.
          Default false — tests are advisory (continue-on-error).
        type: boolean
        default: false
```

- [ ] **Step 2: Make `continue-on-error` on the test job conditional**

Change:
```yaml
    continue-on-error: true
```
To:
```yaml
    continue-on-error: ${{ inputs.fail-on-test-failure != true && inputs.fail-on-test-failure != 'true' }}
```

- [ ] **Step 3: Add `inputs.run-tests != false` to `publish-testpypi` `if:` condition**

Change:
```yaml
    if: |
      startsWith(github.ref, 'refs/tags/') &&
      !startsWith(github.ref, 'refs/tags/v') &&
      (inputs.publish-target == 'testpypi' || inputs.publish-target == 'both')
```
To:
```yaml
    if: |
      startsWith(github.ref, 'refs/tags/') &&
      !startsWith(github.ref, 'refs/tags/v') &&
      inputs.run-tests != false &&
      (inputs.publish-target == 'testpypi' || inputs.publish-target == 'both')
```

- [ ] **Step 4: Commit**
```bash
git add .github/workflows/python-publish.yml
git commit -m "fix: add fail-on-test-failure input; enforce run-tests gate on publish-testpypi"
```

---

## Task 4: python-publish.yml — boolean/string comparison for use-trusted-publishing

**Files:** Modify `.github/workflows/python-publish.yml`

- [ ] **Step 1: Fix `if:` on Trusted Publishing step (~line 382)**

Change:
```yaml
      if: inputs.use-trusted-publishing != false
```
To:
```yaml
      if: inputs.use-trusted-publishing != false && inputs.use-trusted-publishing != 'false'
```

- [ ] **Step 2: Fix `if:` on token-path setup step (~line 389)**

Change:
```yaml
      if: inputs.use-trusted-publishing == false
```
To:
```yaml
      if: inputs.use-trusted-publishing == false || inputs.use-trusted-publishing == 'false'
```

- [ ] **Step 3: Fix `if:` on `Install twine (token path)` step (~line 395)**

Change:
```yaml
      if: inputs.use-trusted-publishing == false
```
To:
```yaml
      if: inputs.use-trusted-publishing == false || inputs.use-trusted-publishing == 'false'
```

- [ ] **Step 4: Fix `if:` on `Publish to PyPI (API Token)` step (~line 399)**

Change:
```yaml
      if: inputs.use-trusted-publishing == false
```
To:
```yaml
      if: inputs.use-trusted-publishing == false || inputs.use-trusted-publishing == 'false'
```

- [ ] **Step 5: Commit**
```bash
git add .github/workflows/python-publish.yml
git commit -m "fix: handle boolean/string use-trusted-publishing for workflow_dispatch"
```

---

## Task 5: publish-mkdocs.yml — all four fixes

**Files:** Modify `.github/workflows/publish-mkdocs.yml`

- [ ] **Step 1: Add `default-branch` input to `workflow_call` inputs**

After the `working-directory` input, add:
```yaml
      default-branch:
        description: "Default branch name that triggers doc builds on push"
        type: string
        default: "main"
```

- [ ] **Step 2: Update `build-docs` job `if:` to use `default-branch` input**

Change:
```yaml
    if: github.ref == 'refs/heads/main' || github.event_name == 'workflow_dispatch'
```
To:
```yaml
    if: github.ref == format('refs/heads/{0}', inputs.default-branch || 'main') || github.event_name == 'workflow_dispatch'
```

- [ ] **Step 3: Fix `docs-requirements` install step to use the input with graceful fallback**

Change the hardcoded install step (~line 96–99):
```yaml
      - name: 📦 Install doc dependencies
        run: |
          pip install --upgrade pip
          pip install mkdocs-material mkdocstrings-python mkdocs-git-revision-date-localized-plugin mkdocs-git-authors-plugin ruff
```
To:
```yaml
      - name: 📦 Install doc dependencies
        run: |
          pip install --upgrade pip
          REQ_FILE="${{ inputs.docs-requirements || 'docs/requirements.txt' }}"
          if [ -f "$REQ_FILE" ]; then
            echo "Installing from $REQ_FILE"
            pip install -r "$REQ_FILE"
          else
            echo "⚠️  $REQ_FILE not found — installing default packages"
            pip install mkdocs-material mkdocstrings-python mkdocs-git-revision-date-localized-plugin mkdocs-git-authors-plugin
          fi
```

- [ ] **Step 4: Fix artifact upload path from `public` to `site`**

Change (~line 117):
```yaml
          path: ${{ inputs.working-directory || '.' }}/public
```
To:
```yaml
          path: ${{ inputs.working-directory || '.' }}/site
```

- [ ] **Step 5: Move `id-token: write` from workflow level to `deploy-docs` job only**

Change workflow-level permissions (~lines 56-59):
```yaml
permissions:
  contents: read
  pages: write
  id-token: write
```
To:
```yaml
permissions:
  contents: read
  pages: write
```

Add job-level permissions block to `deploy-docs` job:
```yaml
  deploy-docs:
    name: 🚀 Deploy to GitHub Pages
    needs: build-docs
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      pages: write
    environment:
```

- [ ] **Step 6: Commit**
```bash
git add .github/workflows/publish-mkdocs.yml
git commit -m "fix: default-branch input, docs-requirements install, site/ path, id-token scope"
```

---

## Task 6: publish-zensical.yml — two fixes

**Files:** Modify `.github/workflows/publish-zensical.yml`

- [ ] **Step 1: Add `default-branch` input to `workflow_call` inputs**

After the `working-directory` input, add:
```yaml
      default-branch:
        description: "Default branch name that triggers doc builds on push"
        type: string
        default: "main"
```

- [ ] **Step 2: Update `build-docs` job `if:` to use `default-branch` input**

Change:
```yaml
    if: github.ref == 'refs/heads/main' || github.event_name == 'workflow_dispatch'
```
To:
```yaml
    if: github.ref == format('refs/heads/{0}', inputs.default-branch || 'main') || github.event_name == 'workflow_dispatch'
```

- [ ] **Step 3: Move `id-token: write` from workflow level to `deploy-docs` job only**

Change workflow-level permissions (~lines 56-59):
```yaml
permissions:
  contents: read
  pages: write
  id-token: write
```
To:
```yaml
permissions:
  contents: read
  pages: write
```

Add job-level permissions block to `deploy-docs` job:
```yaml
  deploy-docs:
    name: 🚀 Deploy to GitHub Pages
    needs: build-docs
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      pages: write
    environment:
```

- [ ] **Step 4: Commit**
```bash
git add .github/workflows/publish-zensical.yml
git commit -m "fix: default-branch input, id-token scoped to deploy job"
```

---

## Task 7: README.md — document publish-zensical.yml

**Files:** Modify `README.md`

- [ ] **Step 1: Add publish-zensical.yml to the repository structure diagram**

Add `publish-zensical.yml` line to the diagram in the Repository Structure section.

- [ ] **Step 2: Add Available Workflows section for Zensical**

Add a subsection after the MkDocs section describing inputs, triggers, and requirements.

- [ ] **Step 3: Add Zensical to the Input Reference table**

Add an input reference table for `publish-zensical.yml`.

- [ ] **Step 4: Commit**
```bash
git add README.md
git commit -m "docs: document publish-zensical.yml workflow"
```

---

## Validation Checklist

After all tasks complete, verify:

- [ ] `python-publish.yml` workflow-level `permissions` has only `contents: read`
- [ ] `publish-pypi` job has `permissions: id-token: write`
- [ ] `pypa/gh-action-pypi-publish` uses SHA `cef221092ed1bacb1cc03d23a2d87d1d172e277b`
- [ ] `reject-v-prefix-tag` job exists and fires on `startsWith(github.ref, 'refs/tags/v')`
- [ ] `publish-testpypi` `if:` includes `inputs.run-tests != false`
- [ ] `use-trusted-publishing` comparisons handle both boolean and string `'false'`
- [ ] `fail-on-test-failure` input exists; `continue-on-error` is conditional
- [ ] `publish-mkdocs.yml` uploads from `.../site` not `.../public`
- [ ] Both doc workflows have `default-branch` input and use it in `build-docs` `if:`
- [ ] Both doc workflows have `id-token: write` only on `deploy-docs`, not at workflow level
- [ ] `publish-mkdocs.yml` install step reads from `docs-requirements` input with fallback
- [ ] README documents `publish-zensical.yml`
