# Playwright Workflow Improvements: PR Tests vs Original

## Overview
The `pr-tests.yml` workflow represents a significant improvement over `playwright.yml` by adopting PR-based triggering, implementing dependency caching, and improving artifact management. This document details the line-by-line improvements and architectural changes.

---

## Line-by-Line Comparison

### 1. **Trigger Event (Lines 3-6 vs 2-4)**

**playwright.yml (Old):**
```yaml
on:
  push:
    branches: [upgrade-dev]
```

**pr-tests.yml (New):**
```yaml
on:
  pull_request:
    branches: [main, dev]
    types: [opened, synchronize, reopened]
```

**Why it's better:**
- **PR-based triggering** means tests run on pull request events, not every push to a branch
- **Explicit event types** (`opened`, `synchronize`, `reopened`) ensure tests run when:
  - A new PR is created (`opened`)
  - New commits are pushed to an existing PR (`synchronize`)
  - A closed PR is reopened (`reopened`)
- **Multiple branches** (main, dev) instead of single upgrade-dev allows testing across development workflows
- **Cleaner history** - avoids running tests on every commit; instead runs on PR lifecycle events
- **Cost reduction** - fewer workflow executions = lower GitHub Actions usage

---

### 2. **Docker Image Handling (Line 16 vs 9-11)**

**playwright.yml (Old):**
```yaml
container:
  image: mcr.microsoft.com/playwright:v1.58.1-noble
  options: --user 1001
```

**pr-tests.yml (New):**
```yaml
- name: Pull Playwright Docker image
  run: docker pull mcr.microsoft.com/playwright:v1.58.1-noble

# Later in steps:
docker run --rm \
  --user 1001 \
  -v "${{ github.workspace }}:/workspace" \
  -w /workspace \
  -e CI=true \
  mcr.microsoft.com/playwright:v1.58.1-noble \
  sh -c "..."
```

**Why it's better:**
- **Explicit pull step** ensures latest image is pulled and gives visibility in logs
- **Docker run approach** provides finer control over:
  - Volume mounts: `-v "${{ github.workspace }}:/workspace"`
  - Working directory: `-w /workspace`
  - User context: `--user 1001`
  - Environment variables: `-e CI=true`
- **`--rm` flag** automatically removes container after execution, saving disk space
- **Transparent command execution** - full Docker command is visible in logs for debugging

---

### 3. **Dependency Caching (Lines 18-26)**

**playwright.yml (Old):**
No caching strategy

**pr-tests.yml (New):**
```yaml
- name: Cache npm dependencies
  uses: actions/cache@v4
  with:
    path: |
      node_modules
      ~/.npm
    key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-npm-
```

**Why it's better:**
- **Hash-based cache keys** use `hashFiles('**/package-lock.json')` - cache only invalidates when dependencies actually change
- **Multi-path caching** caches both:
  - `node_modules/` - installed packages
  - `~/.npm` - npm cache directory
- **Restore keys fallback** (`runner.os-npm-`) allows partial cache hits even if lock file changes
- **Significant speed improvement** - avoids re-downloading all dependencies on every run
- **Cost reduction** - faster builds = lower GitHub Actions minutes consumed

---

### 4. **Dependency Installation (Lines 20-21 vs 40-41)**

**playwright.yml (Old):**
```yaml
- name: Install dependencies
  run: npm ci
```

**pr-tests.yml (New):**
```yaml
if [ -f package-lock.json ]; then npm ci; else npm install; fi
```

**Why it's better:**
- **Conditional logic** handles both scenarios:
  - `npm ci` (preferred) - installs exact versions from lock file (reproducible)
  - `npm install` (fallback) - creates new lock file if none exists
- **More robust** - doesn't fail if lock file is missing
- **Wrapped in Docker container** with proper error output

---

### 5. **Test Report Upload (Lines 45-52 vs 31-38)**

**playwright.yml (Old):**
```yaml
- name: Upload Playwright test report
  id: upload-report
  uses: actions/upload-artifact@v5
  if: always()
  with:
    name: playwright-report
    path: playwright-report/
    retention-days: 30
```

**pr-tests.yml (New):**
```yaml
- name: Upload test report
  if: always()
  uses: actions/upload-artifact@v4
  with:
    name: playwright-report-${{ github.run_id }}
    path: playwright-report/
    retention-days: 30
    if-no-files-found: warn
```

**Why it's better:**
- **Unique artifact naming** using `${{ github.run_id }}` prevents overwrites
  - Old: all reports named `playwright-report` (each new run overwrites previous)
  - New: each run gets unique name like `playwright-report-5234891234`
- **`if-no-files-found: warn`** handles edge cases gracefully:
  - If tests don't generate report, workflow won't fail
  - Shows warning instead of error for transparency
- **Newer action version** (`@v4` vs `@v5`) has better bug fixes
- **Better artifact management** - old reports not overwritten, available for historical comparison

---

### 6. **Version Pinning**

**playwright.yml (Old):**
```yaml
uses: actions/checkout@v5
uses: actions/upload-artifact@v5
```

**pr-tests.yml (New):**
```yaml
uses: actions/checkout@v4
uses: actions/upload-artifact@v4
```

**Why it's better:**
- **Consistent action versions** - both use v4 generation (more recent, stable)
- **Reduced breaking changes** - v4 has fewer quirks than v5
- **Better maintainability** - unified version strategy across workflows

---

## Features Removed (Considerations)

The new workflow removed some features that may need re-implementation:

### 1. **Test Secrets**
**Old workflow had:**
```yaml
env:
  CI: true
  TEST_LOGIN_EMAIL: ${{ secrets.TEST_LOGIN_EMAIL }}
  TEST_LOGIN_PASSWORD: ${{ secrets.TEST_LOGIN_PASSWORD }}
  TEST_BASE_URL: ${{ secrets.TEST_BASE_URL }}
```

**Recommendation:** Add back to new workflow if integration tests require authentication:
```yaml
- name: Run Playwright tests
  env:
    CI: true
    TEST_LOGIN_EMAIL: ${{ secrets.TEST_LOGIN_EMAIL }}
    TEST_LOGIN_PASSWORD: ${{ secrets.TEST_LOGIN_PASSWORD }}
    TEST_BASE_URL: ${{ secrets.TEST_BASE_URL }}
  run: |
    docker run --rm \
      --user 1001 \
      -v "${{ github.workspace }}:/workspace" \
      -w /workspace \
      -e CI=true \
      -e TEST_LOGIN_EMAIL=$TEST_LOGIN_EMAIL \
      -e TEST_LOGIN_PASSWORD=$TEST_LOGIN_PASSWORD \
      -e TEST_BASE_URL=$TEST_BASE_URL \
      mcr.microsoft.com/playwright:v1.58.1-noble \
      sh -c "..."
```

### 2. **Node.js Version Verification**
**Old workflow had:**
```yaml
- name: Verify Node.js version
  run: |
    echo "Node version: $(node --version)"
    echo "npm version: $(npm --version)"
```

**Recommendation:** Add for debugging if version mismatches occur:
```yaml
- name: Verify Node.js version
  run: |
    docker run --rm \
      mcr.microsoft.com/playwright:v1.58.1-noble \
      sh -c "echo 'Node version:' && node --version && echo 'npm version:' && npm --version"
```

### 3. **GitHub Step Summary with Artifact URL**
**Old workflow had:**
```yaml
- name: Share report link
  if: always()
  run: |
    echo "## Playwright Test Report" >> $GITHUB_STEP_SUMMARY
    echo "[View HTML Report](${{ steps.upload-report.outputs.artifact-url }})" >> $GITHUB_STEP_SUMMARY
```

**Recommendation:** Add to new workflow for better PR visibility:
```yaml
- name: Share report link
  if: always()
  id: upload-report
  uses: actions/upload-artifact@v4
  with:
    name: playwright-report-${{ github.run_id }}
    path: playwright-report/
    retention-days: 30
    if-no-files-found: warn

- name: Add test report to summary
  if: always()
  run: |
    echo "## Playwright Test Report" >> $GITHUB_STEP_SUMMARY
    echo "📊 [View HTML Report](${{ steps.upload-report.outputs.artifact-url }})" >> $GITHUB_STEP_SUMMARY
```

---

## Summary of Major Improvements

| Aspect | Old (playwright.yml) | New (pr-tests.yml) | Benefit |
|--------|----------------------|-------------------|---------|
| **Trigger** | Push to upgrade-dev | PR events (opened, synchronize, reopened) | Cleaner workflow, cost reduction |
| **Branches** | Single (upgrade-dev) | Multiple (main, dev) | Better multi-branch support |
| **Caching** | None | Hash-based npm cache | Faster builds, cost savings |
| **Docker** | Container config | Explicit docker run | Better control, visibility |
| **Artifact naming** | Static name (overwrites) | Dynamic name with run_id | Historical tracking |
| **Dependency install** | npm ci only | Conditional (npm ci \| npm install) | More robust |
| **Error handling** | Fails if no report | Warns if no report | Graceful degradation |
| **Action versions** | v5 | v4 | Better stability |

---

## Recommended Next Steps

1. **Add back test secrets** if integration tests require them
2. **Implement GitHub Step Summary** to show report links in PRs
3. **Add Node.js verification step** for debugging version issues
4. **Set repository secrets** for TEST_LOGIN_EMAIL, TEST_LOGIN_PASSWORD, TEST_BASE_URL
5. **Monitor artifact storage** - unique naming may increase artifact count over time
