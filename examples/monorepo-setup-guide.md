# Monorepo Setup Guide

Complete step-by-step guide for setting up LUMOS GitHub Action in a monorepo with multiple packages.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Directory Structure](#directory-structure)
- [Step 1: Organize Schemas](#step-1-organize-schemas)
- [Step 2: Choose Validation Strategy](#step-2-choose-validation-strategy)
- [Step 3: Set Up Workflows](#step-3-set-up-workflows)
- [Step 4: Configure Path Filters](#step-4-configure-path-filters)
- [Step 5: Test Locally](#step-5-test-locally)
- [Step 6: Deploy](#step-6-deploy)
- [Troubleshooting](#troubleshooting)

---

## Prerequisites

- ✅ GitHub repository with monorepo structure
- ✅ Multiple packages with `.lumos` schemas
- ✅ Basic understanding of GitHub Actions
- ✅ LUMOS CLI installed locally (for testing)

## Directory Structure

**Recommended structure:**

```
monorepo/
├── packages/
│   ├── auth/
│   │   ├── schemas/
│   │   │   ├── user.lumos
│   │   │   └── session.lumos
│   │   ├── generated/
│   │   │   ├── user.rs
│   │   │   ├── user.ts
│   │   │   ├── session.rs
│   │   │   └── session.ts
│   │   ├── Cargo.toml  (optional)
│   │   └── package.json  (optional)
│   │
│   ├── payments/
│   │   ├── schemas/
│   │   │   └── transaction.lumos
│   │   └── generated/
│   │
│   └── users/
│       ├── schemas/
│       └── generated/
│
└── .github/
    └── workflows/
        └── lumos.yml
```

**Alternative: Centralized schemas**

```
monorepo/
├── schemas/
│   ├── auth/
│   │   ├── user.lumos
│   │   └── session.lumos
│   ├── payments/
│   │   └── transaction.lumos
│   └── users/
│       └── profile.lumos
│
├── generated/
│   ├── rust/
│   └── typescript/
│
└── .github/
    └── workflows/
        └── lumos.yml
```

---

## Step 1: Organize Schemas

### Option A: Per-Package Organization (Recommended)

Each package has its own schemas and generated code:

```bash
# Create structure for each package
cd packages/auth
mkdir -p schemas generated

# Add your schemas
cat > schemas/user.lumos <<EOF
#[solana]
#[account]
struct User {
    wallet: PublicKey,
    email: String,
    created_at: u64,
}
EOF

# Generate locally to verify
lumos generate schemas/user.lumos
```

### Option B: Centralized Schemas

All schemas in one directory:

```bash
mkdir -p schemas/{auth,payments,users}
mkdir -p generated/{rust,typescript}

# Organize by feature/module
cat > schemas/auth/user.lumos <<EOF
#[solana]
struct User {
    wallet: PublicKey,
}
EOF
```

**Which to choose?**
- **Per-package**: Better for independent packages with their own build systems
- **Centralized**: Better for shared schemas used across packages

---

## Step 2: Choose Validation Strategy

### Strategy 1: Simple Matrix (Easiest)

Validates all packages in parallel:

```yaml
strategy:
  matrix:
    package: [auth, payments, users]
steps:
  - uses: getlumos/lumos-action@v1
    with:
      schema: 'packages/${{ matrix.package }}/schemas/*.lumos'
```

**✅ Use when:**
- All packages equally important
- Simple failure mode (all must pass)
- Getting started

### Strategy 2: Path-Filtered Matrix (Efficient)

Only validates packages that changed:

```yaml
- uses: dorny/paths-filter@v3
  with:
    filters: |
      auth: 'packages/auth/**'
      payments: 'packages/payments/**'
```

**✅ Use when:**
- Large monorepo (5+ packages)
- Want to save CI time/cost
- Packages change independently

### Strategy 3: Tiered Validation (Advanced)

Different failure modes per package criticality:

```yaml
# Critical: Must pass
critical-packages:
  strategy:
    matrix:
      package: [auth, payments]
  steps:
    - uses: getlumos/lumos-action@v1
      with:
        fail-on-drift: true  # Blocks merge

# Standard: Warn only
standard-packages:
  continue-on-error: true  # Don't block
  strategy:
    matrix:
      package: [users, profiles]
```

**✅ Use when:**
- Packages have different criticality levels
- Some packages can tolerate drift
- Need flexible enforcement

---

## Step 3: Set Up Workflows

### Workflow 1: Basic Per-Package Validation

Create `.github/workflows/lumos-monorepo.yml`:

```yaml
name: Monorepo Validation

on:
  pull_request:
  push:
    branches: [main]

jobs:
  validate:
    name: Validate ${{ matrix.package }}
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        package:
          - auth
          - payments
          - users
    steps:
      - uses: actions/checkout@v4

      - uses: getlumos/lumos-action@v1
        with:
          schema: 'packages/${{ matrix.package }}/schemas/*.lumos'
          working-directory: 'packages/${{ matrix.package }}'
          fail-on-drift: true
          comment-on-pr: true
```

**Test this workflow:**
```bash
# Install act (https://github.com/nektos/act)
brew install act  # macOS
# or: curl https://raw.githubusercontent.com/nektos/act/master/install.sh | sudo bash

# Test locally
act pull_request -W .github/workflows/lumos-monorepo.yml
```

### Workflow 2: With Path Filtering

Create `.github/workflows/lumos-smart.yml`:

```yaml
name: Smart Monorepo Validation

on:
  pull_request:
  push:
    branches: [main]

jobs:
  detect-changes:
    outputs:
      packages: ${{ steps.filter.outputs.changes }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            auth: 'packages/auth/**'
            payments: 'packages/payments/**'
            users: 'packages/users/**'

  validate:
    needs: detect-changes
    if: needs.detect-changes.outputs.packages != '[]'
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        package: ${{ fromJSON(needs.detect-changes.outputs.packages) }}
    steps:
      - uses: actions/checkout@v4

      - uses: getlumos/lumos-action@v1
        with:
          schema: 'packages/${{ matrix.package }}/schemas/*.lumos'
          working-directory: 'packages/${{ matrix.package }}'
          fail-on-drift: true
```

---

## Step 4: Configure Path Filters

### Define Filter Patterns

Customize what triggers validation for each package:

```yaml
filters: |
  auth:
    - 'packages/auth/schemas/**/*.lumos'      # Schema changes
    - 'packages/auth/generated/**'             # Generated code changes
    - 'packages/auth/.lumos-version'           # Version file changes

  payments:
    - 'packages/payments/schemas/**'
    - 'packages/payments/generated/**'
```

### Test Path Filters

```bash
# Manually test which packages would trigger
gh api repos/:owner/:repo/pulls/:pr_number/files \
  --jq '.[].filename' | \
  grep -E '^packages/(auth|payments)/'
```

---

## Step 5: Test Locally

### Test Individual Package

```bash
cd packages/auth

# Validate schemas
lumos validate schemas/*.lumos

# Generate code
lumos generate schemas/*.lumos

# Check for drift
git status generated/
```

### Test All Packages

```bash
# Create test script: scripts/validate-all.sh
#!/bin/bash
set -e

for package in packages/*/; do
  pkg_name=$(basename "$package")
  echo "Validating $pkg_name..."

  cd "$package"

  # Validate
  lumos validate schemas/*.lumos

  # Generate
  lumos generate schemas/*.lumos

  # Check for drift
  if ! git diff --quiet generated/; then
    echo "⚠️ Drift detected in $pkg_name"
    git diff generated/
  fi

  cd ../..
done

echo "✅ All packages validated"
```

```bash
chmod +x scripts/validate-all.sh
./scripts/validate-all.sh
```

---

## Step 6: Deploy

### 1. Create PR to test workflow

```bash
# Make a schema change
cd packages/auth
echo "  // Test change" >> schemas/user.lumos

# Commit and push
git add .
git commit -m "test: Verify workflow triggers"
git push origin feature/test-workflow

# Create PR
gh pr create --title "Test workflow" --body "Testing monorepo validation"
```

### 2. Verify workflow runs

- Check Actions tab: `https://github.com/owner/repo/actions`
- Verify correct packages triggered
- Check PR comments for results

### 3. Iterate and refine

- Adjust path filters if needed
- Add/remove packages from matrix
- Configure failure modes per package

### 4. Protect main branch

```bash
# Via GitHub CLI
gh api repos/:owner/:repo/branches/main/protection \
  -X PUT \
  -f required_status_checks[strict]=true \
  -f required_status_checks[contexts][]=validate

# Or via GitHub UI:
# Settings → Branches → Branch protection rules
# Add rule for 'main'
# Check: Require status checks to pass before merging
# Select: validate (your workflow name)
```

---

## Troubleshooting

### Problem: Workflow runs on all packages even when one changes

**Solution:** Use path filters:

```yaml
- uses: dorny/paths-filter@v3
  id: filter
  with:
    filters: |
      auth: 'packages/auth/**'

# Only run if changed
if: steps.filter.outputs.auth == 'true'
```

### Problem: Matrix doesn't expand properly

**Check:**
```yaml
# ❌ Wrong
matrix:
  package: ${{ needs.job.outputs.packages }}

# ✅ Correct
matrix:
  package: ${{ fromJSON(needs.job.outputs.packages) }}
```

### Problem: Working directory not found

**Solution:** Ensure package exists:

```yaml
- name: Check package exists
  run: |
    if [ ! -d "packages/${{ matrix.package }}" ]; then
      echo "Package not found: ${{ matrix.package }}"
      exit 1
    fi
```

### Problem: Generated files in wrong location

**Workaround:** Move after generation:

```yaml
- uses: getlumos/lumos-action@v1
  with:
    schema: 'packages/${{ matrix.package }}/schemas/*.lumos'

- name: Organize generated code
  run: |
    mkdir -p packages/${{ matrix.package }}/generated
    find packages/${{ matrix.package }}/schemas \
      -name "generated.*" \
      -exec mv {} packages/${{ matrix.package }}/generated/ \;
```

### Problem: Drift detected but changes are intentional

**Options:**

1. **Regenerate and commit:**
   ```bash
   lumos generate schemas/*.lumos
   git add generated/
   git commit -m "chore: Update generated code"
   ```

2. **Disable drift check temporarily:**
   ```yaml
   - uses: getlumos/lumos-action@v1
     with:
       fail-on-drift: false  # Warning only
   ```

3. **Skip validation for specific PR:**
   Add `[skip ci]` to commit message:
   ```bash
   git commit -m "chore: Update schemas [skip ci]"
   ```

---

## Next Steps

- ✅ [Advanced Strategies](../docs/monorepo-advanced.md) - Per-package drift, tiered validation
- ✅ [Example Workflows](./workflows/) - Complete working examples
- ✅ [Breaking Change Detection](./workflows/breaking-change-detection.yml) - Git diff analysis
- ✅ [Custom PR Comments](./workflows/custom-pr-comments.yml) - Custom comment formatting

---

## Quick Reference

### Package List Management

**Static list (simplest):**
```yaml
matrix:
  package: [auth, payments, users]
```

**Dynamic from paths (efficient):**
```yaml
packages: ${{ steps.filter.outputs.changes }}
```

**From file (scalable):**
```yaml
- run: echo "packages=$(cat .github/packages.json)" >> $GITHUB_OUTPUT
```

### Common Patterns

**All packages, strict:**
```yaml
strategy:
  matrix:
    package: [auth, payments, users]
steps:
  - uses: getlumos/lumos-action@v1
    with:
      fail-on-drift: true
```

**Changed packages only:**
```yaml
- uses: dorny/paths-filter@v3
  id: filter
  with:
    filters: |
      auth: 'packages/auth/**'

matrix:
  package: ${{ fromJSON(needs.detect.outputs.packages) }}
```

**Critical packages strict, others lenient:**
```yaml
# Job 1: Critical
strategy:
  matrix:
    package: [auth, payments]
steps:
  - uses: getlumos/lumos-action@v1
    with:
      fail-on-drift: true

# Job 2: Standard
continue-on-error: true
strategy:
  matrix:
    package: [users, analytics]
```

---

**Need help?** Open an issue: https://github.com/getlumos/lumos-action/issues
