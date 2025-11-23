# Advanced Monorepo Strategies

This guide provides advanced patterns for using LUMOS GitHub Action in monorepo environments with multiple packages. It covers per-package drift detection, selective failure strategies, and breaking change detection.

## Table of Contents

- [Per-Package Drift Detection](#per-package-drift-detection)
- [Selective Failure Strategies](#selective-failure-strategies)
- [Breaking Change Detection](#breaking-change-detection)
- [Feature Requests](#feature-requests)

---

## Per-Package Drift Detection

### Current Limitation

The action provides a **global** drift status across all schemas. It cannot output per-package drift information.

**Outputs:**
- `drift-detected`: `true` if **any** schema has drift (global)
- `diff-summary`: Combined diff for all changed files

### Workaround: Matrix Strategy with Path Filtering

Use GitHub Actions matrix to validate each package independently:

```yaml
name: Per-Package Validation

on:
  pull_request:
  push:
    branches: [main]

jobs:
  detect-changed-packages:
    name: Detect Changed Packages
    runs-on: ubuntu-latest
    outputs:
      packages: ${{ steps.filter.outputs.changes }}
    steps:
      - uses: actions/checkout@v4

      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            auth:
              - 'packages/auth/schemas/**'
              - 'packages/auth/generated/**'
            payments:
              - 'packages/payments/schemas/**'
              - 'packages/payments/generated/**'
            users:
              - 'packages/users/schemas/**'
              - 'packages/users/generated/**'
            analytics:
              - 'packages/analytics/schemas/**'
              - 'packages/analytics/generated/**'

  validate-changed-packages:
    name: Validate Changed Packages
    needs: detect-changed-packages
    if: needs.detect-changed-packages.outputs.packages != '[]'
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false  # Don't cancel other packages on failure
      max-parallel: 4   # Run 4 packages in parallel
      matrix:
        package: ${{ fromJSON(needs.detect-changed-packages.outputs.packages) }}
    steps:
      - uses: actions/checkout@v4

      - name: Validate ${{ matrix.package }} schemas
        id: lumos
        uses: getlumos/lumos-action@v1
        with:
          schema: 'packages/${{ matrix.package }}/schemas/*.lumos'
          working-directory: 'packages/${{ matrix.package }}'
          fail-on-drift: true
          comment-on-pr: true

      - name: Report package drift
        if: failure()
        uses: actions/github-script@v7
        with:
          script: |
            const pkg = '${{ matrix.package }}';
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## ❌ Package \`${pkg}\` Schema Drift

The \`${pkg}\` package has schema drift.

**To fix:**
\`\`\`bash
cd packages/${pkg}
lumos generate schemas/*.lumos
git add generated/
\`\`\`

**Affected files:** See workflow logs for details.
            `
            });
```

**Benefits:**
- ✅ Each package validated independently
- ✅ Failures isolated (one package failing doesn't block others)
- ✅ Parallel execution (faster for large monorepos)
- ✅ Only changed packages validated (efficient)

---

## Selective Failure Strategies

### Strategy 1: Package Criticality Tiers

Different rules for different packages based on importance:

```yaml
name: Tiered Package Validation

on: pull_request

jobs:
  critical-packages:
    name: Critical Packages (Must Pass)
    runs-on: ubuntu-latest
    strategy:
      fail-fast: true  # Stop all if one fails
      matrix:
        package: [auth, payments, security]
    steps:
      - uses: actions/checkout@v4

      - uses: getlumos/lumos-action@v1
        with:
          schema: 'packages/${{ matrix.package }}/schemas/*.lumos'
          working-directory: 'packages/${{ matrix.package }}'
          fail-on-drift: true  # STRICT - blocks merge

  standard-packages:
    name: Standard Packages (Should Pass)
    runs-on: ubuntu-latest
    continue-on-error: true  # Don't block CI, but report
    strategy:
      fail-fast: false
      matrix:
        package: [users, profiles, notifications, messaging]
    steps:
      - uses: actions/checkout@v4

      - uses: getlumos/lumos-action@v1
        with:
          schema: 'packages/${{ matrix.package }}/schemas/*.lumos'
          working-directory: 'packages/${{ matrix.package }}'
          fail-on-drift: true
          comment-on-pr: true  # Warn but allow merge

  experimental-packages:
    name: Experimental Packages (Informational)
    runs-on: ubuntu-latest
    continue-on-error: true
    strategy:
      matrix:
        package: [analytics, internal-tools, experimental-features]
    steps:
      - uses: actions/checkout@v4

      - uses: getlumos/lumos-action@v1
        with:
          schema: 'packages/${{ matrix.package }}/schemas/*.lumos'
          working-directory: 'packages/${{ matrix.package }}'
          fail-on-drift: false  # Never block, just inform
          comment-on-pr: true
```

**Tier Definitions:**

| Tier | Packages | fail-on-drift | continue-on-error | Blocks Merge? |
|------|----------|---------------|-------------------|---------------|
| **Critical** | auth, payments, security | `true` | `false` | ✅ Yes |
| **Standard** | users, profiles | `true` | `true` | ❌ No (warns) |
| **Experimental** | analytics, tools | `false` | `true` | ❌ No |

### Strategy 2: Environment-Based Rules

Different rules for different environments:

```yaml
jobs:
  production-validation:
    if: github.base_ref == 'main'
    strategy:
      matrix:
        package: [auth, payments, users]
    steps:
      - uses: actions/checkout@v4

      - uses: getlumos/lumos-action@v1
        with:
          schema: 'packages/${{ matrix.package }}/schemas/*.lumos'
          working-directory: 'packages/${{ matrix.package }}'
          fail-on-drift: true  # Strict for production

  staging-validation:
    if: github.base_ref == 'staging'
    strategy:
      matrix:
        package: [auth, payments, users]
    steps:
      - uses: actions/checkout@v4

      - uses: getlumos/lumos-action@v1
        with:
          schema: 'packages/${{ matrix.package }}/schemas/*.lumos'
          working-directory: 'packages/${{ matrix.package }}'
          fail-on-drift: false  # Lenient for staging
```

### Strategy 3: Manual Approval for Specific Packages

Require manual approval before allowing drift in critical packages:

```yaml
name: Critical Package Approval

on:
  pull_request:

jobs:
  validate-critical:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        package: [auth, payments]
    steps:
      - uses: actions/checkout@v4

      - name: Validate ${{ matrix.package }}
        id: lumos
        uses: getlumos/lumos-action@v1
        with:
          schema: 'packages/${{ matrix.package }}/schemas/*.lumos'
          fail-on-drift: false  # Don't fail automatically

      - name: Require approval for drift
        if: steps.lumos.outputs.drift-detected == 'true'
        uses: trstringer/manual-approval@v1
        with:
          secret: ${{ github.TOKEN }}
          approvers: team-leads,senior-devs
          minimum-approvals: 2
          issue-title: "Schema drift in ${{ matrix.package }} - Approval Required"
          issue-body: |
            Schema drift detected in `${{ matrix.package }}` package.

            **Requires 2 approvals from:**
            - @team-leads
            - @senior-devs

            Review the changes and approve if intentional.
```

---

## Breaking Change Detection

### Current Limitation

LUMOS CLI does **not** detect breaking changes automatically. The action only detects drift (generated code ≠ schema).

**Breaking changes** require manual analysis.

### Workaround: Git Diff Analysis

Analyze schema changes between branches to detect potential breaking changes:

```yaml
name: Breaking Change Detection

on:
  pull_request:

jobs:
  analyze-schema-changes:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Full history needed for diff

      - name: Get changed schema files
        id: changes
        run: |
          git diff origin/${{ github.base_ref }}...HEAD \
            --name-only \
            --diff-filter=M \
            '**/*.lumos' > changed_schemas.txt

          echo "count=$(wc -l < changed_schemas.txt)" >> $GITHUB_OUTPUT
          echo "files<<EOF" >> $GITHUB_OUTPUT
          cat changed_schemas.txt >> $GITHUB_OUTPUT
          echo "EOF" >> $GITHUB_OUTPUT

      - name: Analyze each schema for breaking changes
        if: steps.changes.outputs.count > 0
        run: |
          breaking=false

          while IFS= read -r file; do
            echo "::group::Analyzing $file"

            # Get old version
            git show origin/${{ github.base_ref }}:"$file" > old_schema.lumos

            # Check for field removals (BREAKING)
            removed_fields=$(diff old_schema.lumos "$file" | grep -c '^-.*:' || true)
            if [ "$removed_fields" -gt 0 ]; then
              echo "::warning file=$file::Potential breaking change: $removed_fields field(s) removed"
              breaking=true
            fi

            # Check for type changes (BREAKING)
            # Extract type definitions and compare
            grep ':' old_schema.lumos | sort > old_types.txt
            grep ':' "$file" | sort > new_types.txt

            changed_types=$(diff old_types.txt new_types.txt | grep -c '^[<>]' || true)
            if [ "$changed_types" -gt 0 ]; then
              echo "::warning file=$file::Type definitions changed (review for breaking changes)"
              breaking=true
            fi

            echo "::endgroup::"

          done < changed_schemas.txt

          if [ "$breaking" = true ]; then
            echo "BREAKING_DETECTED=true" >> $GITHUB_ENV
          fi

      - name: Require breaking-change label
        if: env.BREAKING_DETECTED == 'true'
        uses: actions/github-script@v7
        with:
          script: |
            const { data: pr } = await github.rest.pulls.get({
              owner: context.repo.owner,
              repo: context.repo.repo,
              pull_number: context.issue.number,
            });

            const labels = pr.labels.map(l => l.name);

            if (!labels.includes('breaking-change')) {
              core.setFailed('⚠️ Breaking changes detected. Please add "breaking-change" label and document migration path.');

              // Add comment with details
              await github.rest.issues.createComment({
                issue_number: context.issue.number,
                owner: context.repo.owner,
                repo: context.repo.repo,
                body: `## 🚨 Breaking Changes Detected

Schema changes include potential breaking modifications.

**Required Actions:**
1. Add \`breaking-change\` label to this PR
2. Document migration path in PR description
3. Update CHANGELOG.md with breaking changes section
4. Consider major version bump

**Breaking Change Checklist:**
- [ ] Migration guide added to PR description
- [ ] CHANGELOG.md updated
- [ ] Version bump planned (if applicable)
- [ ] Team notified of breaking changes
              `
              });
            }
```

### Breaking Change Checklist

**Breaking Changes:**
- ❌ Field removed from struct
- ❌ Field type changed (e.g., `u32` → `u64`)
- ❌ Field renamed
- ❌ Enum variant removed
- ❌ Required field added (non-optional)

**Non-Breaking Changes:**
- ✅ Optional field added (`Option<T>`)
- ✅ New enum variant added (at end)
- ✅ Documentation/comments updated
- ✅ Field order changed (Borsh preserves order, but verify)

### Advanced: Borsh Compatibility Validation

For stricter validation, compile both old and new schemas and compare Borsh schemas:

```yaml
- name: Compare Borsh schemas
  run: |
    # Generate from old schema
    git show origin/${{ github.base_ref }}:schema.lumos > old_schema.lumos
    lumos generate old_schema.lumos
    cp generated.ts old_generated.ts

    # Generate from new schema
    lumos generate schema.lumos

    # Compare TypeScript Borsh schemas
    node -e "
      const old = require('./old_generated.ts');
      const new = require('./generated.ts');

      // Compare Borsh schema structures
      if (JSON.stringify(old.schema) !== JSON.stringify(new.schema)) {
        console.error('⚠️ Borsh schema changed - potential breaking change');
        process.exit(1);
      }
    "
```

---

## Feature Requests

The following features are not currently supported but are documented here for tracking. Consider opening feature requests in the LUMOS Action repository.

### Requested: Per-Package Drift Outputs

**Current:**
```yaml
outputs:
  drift-detected: 'true'  # Global
  diff-summary: 'generated.rs: 5 additions'  # Global
```

**Requested:**
```yaml
outputs:
  drift-detected: 'true'
  drift-detected-packages: '["auth", "payments"]'  # NEW
  clean-packages: '["users", "analytics"]'  # NEW
  diff-summary-by-package: |  # NEW
    auth: generated.rs: 5 additions
    payments: generated.ts: 3 deletions
```

This would enable workflows to react differently based on which packages have drift.

### Requested: Breaking Change Detection

**Requested:** Built-in breaking change analysis

```yaml
outputs:
  breaking-changes-detected: 'true'
  breaking-changes-packages: '["auth"]'
  breaking-changes-summary: |
    auth: field "email" removed (line 15)
    payments: type "Amount" changed u32 → u64 (line 42)
```

This would automate the git diff analysis shown above.

### Requested: Package Configuration File

**Requested:** YAML-based per-package configuration

```yaml
- uses: getlumos/lumos-action@v1
  with:
    config: .lumos-ci.yml  # NEW parameter
```

**`.lumos-ci.yml` example:**
```yaml
packages:
  - name: auth
    schema: 'packages/auth/schemas/*.lumos'
    fail-on-drift: true
    criticality: high
    require-approval: true
    breaking-change-detection: strict

  - name: analytics
    schema: 'packages/analytics/schemas/*.lumos'
    fail-on-drift: false
    criticality: low
    require-approval: false
    breaking-change-detection: warn
```

This would simplify complex monorepo configurations.

### Requested: Drift Isolation

**Requested:** Per-package working directories and isolated validation

Currently, when using glob patterns like `packages/*/schemas/*.lumos`, all packages are validated together. Requested feature would allow:

```yaml
- uses: getlumos/lumos-action@v1
  with:
    packages: |
      - path: packages/auth
        fail-on-drift: true
      - path: packages/payments
        fail-on-drift: true
      - path: packages/analytics
        fail-on-drift: false
```

Each package would have independent drift detection and failure control.

---

## Best Practices

### 1. Start with Simple Matrix Strategy

Begin with basic per-package validation:

```yaml
strategy:
  matrix:
    package: [auth, payments, users]
steps:
  - uses: getlumos/lumos-action@v1
    with:
      schema: 'packages/${{ matrix.package }}/schemas/*.lumos'
```

### 2. Add Criticality Tiers Gradually

Once you understand your package dependencies, introduce tiered validation:
1. Identify critical packages (auth, payments)
2. Separate into jobs with different failure modes
3. Use `continue-on-error` for non-critical packages

### 3. Implement Breaking Change Detection for Stable APIs

Only add breaking change detection for packages with:
- External consumers
- Published APIs
- Strict versioning requirements

### 4. Document Package Dependencies

Maintain a `PACKAGES.md` documenting:
- Package criticality levels
- Inter-package dependencies
- Breaking change policies
- Approval requirements

### 5. Use Path Filters for Efficiency

Only run validation when schemas actually change:

```yaml
- uses: dorny/paths-filter@v3
  id: filter
  with:
    filters: |
      auth: 'packages/auth/**'
```

This reduces CI time and cost.

---

## Complete Example

See [`examples/workflows/monorepo-per-package.yml`](../examples/workflows/monorepo-per-package.yml) for a complete working example combining:
- Path-based change detection
- Per-package matrix validation
- Package-specific PR comments
- Failure isolation

---

## Related Documentation

- [Multi-Language Generation](./multi-language.md)
- [Custom Output Paths](./custom-output-paths.md)
- [Monorepo Setup Guide](../examples/monorepo-setup-guide.md)
- [Examples: Monorepo Multi-Package](../examples/workflows/monorepo-multi-package.yml)

---

**Note:** This guide documents workarounds for current limitations. Native per-package support may be added in future versions. Check the [roadmap](https://github.com/getlumos/lumos-action/issues) for updates.
