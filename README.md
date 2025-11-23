# LUMOS Generate Action

[![GitHub Marketplace](https://img.shields.io/badge/Marketplace-LUMOS%20Generate-blue.svg?colorA=24292e&colorB=0366d6&style=flat&longCache=true&logo=github)](https://github.com/marketplace/actions/lumos-generate)

GitHub Action for automatic LUMOS schema generation and validation in CI/CD pipelines.

## Quick Start

```yaml
- uses: actions/checkout@v4
- uses: getlumos/lumos-action@v1
  with:
    schema: 'schemas/**/*.lumos'
```

## Features

- ✅ Auto-install LUMOS CLI (any version)
- ✅ Validate LUMOS schemas
- ✅ Generate Rust + TypeScript code
- ✅ Detect drift between generated and committed files
- ✅ Post PR comments with diff summaries
- ✅ Configurable failure modes

## Usage

### Basic Generation

```yaml
name: LUMOS Generate

on: [push, pull_request]

jobs:
  generate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Generate from LUMOS schemas
        uses: getlumos/lumos-action@v1
        with:
          schema: 'schemas/**/*.lumos'
```

### Validation Only (PR Checks)

```yaml
name: LUMOS Validate

on: [pull_request]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Validate LUMOS schemas
        uses: getlumos/lumos-action@v1
        with:
          schema: 'schemas/**/*.lumos'
          check-only: true
          fail-on-drift: true
```

### Advanced Configuration

```yaml
- name: Generate with custom version
  uses: getlumos/lumos-action@v1
  with:
    schema: 'programs/**/schema.lumos'
    version: '0.1.1'
    working-directory: './backend'
    fail-on-drift: false
    comment-on-pr: true
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `schema` | Path to schema files (supports globs) | Yes | - |
| `check-only` | Only validate, do not generate | No | `false` |
| `version` | LUMOS CLI version to install | No | `latest` |
| `working-directory` | Working directory for commands | No | `.` |
| `fail-on-drift` | Fail if drift detected | No | `true` |
| `comment-on-pr` | Post PR comment with results | No | `true` |

## Outputs

| Output | Description |
|--------|-------------|
| `schemas-validated` | Number of schemas validated |
| `schemas-generated` | Number of schemas generated |
| `drift-detected` | Whether drift was detected |
| `diff-summary` | Summary of differences |

## Examples

### Monorepo with Multiple Schemas

```yaml
- name: Generate all schemas
  uses: getlumos/lumos-action@v1
  with:
    schema: |
      programs/nft/schema.lumos
      programs/defi/schema.lumos
      programs/gaming/schema.lumos
```

### Specific Version Pinning

```yaml
- name: Generate with pinned version
  uses: getlumos/lumos-action@v1
  with:
    schema: 'schema.lumos'
    version: '0.1.1'
```

### Custom Failure Behavior

```yaml
- name: Generate with warnings only
  uses: getlumos/lumos-action@v1
  with:
    schema: 'schemas/*.lumos'
    fail-on-drift: false  # Only warn, don't fail
```

## Workflow Tips

### Pre-commit Hook Alternative

Use this action as a pre-commit check in CI:

```yaml
on:
  pull_request:
    paths:
      - '**/*.lumos'

jobs:
  check-schemas:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: getlumos/lumos-action@v1
        with:
          schema: '**/*.lumos'
          check-only: true
```

### Auto-commit Generated Files

```yaml
- uses: getlumos/lumos-action@v1
  with:
    schema: 'schema.lumos'
    fail-on-drift: false

- name: Commit generated files
  run: |
    git config user.name "LUMOS Bot"
    git config user.email "bot@lumos-lang.org"
    git add .
    git commit -m "chore: Update generated files" || exit 0
    git push
```

## Advanced Usage

### Multi-Language Generation

LUMOS **always generates both Rust and TypeScript** from a single schema. You cannot generate only one language - this ensures type-safe serialization compatibility between Solana programs and clients.

**What gets generated:**
```yaml
- uses: getlumos/lumos-action@v1
  with:
    schema: 'schemas/player.lumos'

# Result:
# schemas/generated.rs   ← Rust (Anchor/Borsh)
# schemas/generated.ts   ← TypeScript + Borsh schemas
```

**To separate languages into different directories:**

```yaml
- uses: getlumos/lumos-action@v1
  with:
    schema: 'schemas/**/*.lumos'

- name: Organize by language
  run: |
    mkdir -p generated/rust generated/typescript
    find schemas -name "generated.rs" -exec mv {} generated/rust/ \;
    find schemas -name "generated.ts" -exec mv {} generated/typescript/ \;
```

See [docs/multi-language.md](docs/multi-language.md) for:
- ✅ Why both languages are generated
- ✅ Workarounds for language separation
- ✅ Monorepo organization patterns
- ✅ File renaming strategies

### Custom Output Paths

Generated files appear **in the same directory** as the schema file. Custom output paths are not directly supported.

**Default behavior:**
```
schemas/
├── player.lumos
├── generated.rs    ← Generated here
└── generated.ts    ← Generated here
```

**Workaround - Post-generation move:**

```yaml
- uses: getlumos/lumos-action@v1
  with:
    schema: 'schemas/**/*.lumos'

- name: Move to custom location
  run: |
    mkdir -p custom/output/path
    find schemas -name "generated.*" -exec mv {} custom/output/path/ \;
```

See [docs/custom-output-paths.md](docs/custom-output-paths.md) for:
- ✅ Working directory vs output path
- ✅ 10+ organization patterns
- ✅ Monorepo centralized vs distributed
- ✅ Dynamic configuration examples

### Monorepo Support

For projects with multiple packages, use matrix strategy to validate each package independently.

**Basic Per-Package Validation:**

```yaml
jobs:
  generate:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        package: [auth, payments, users, analytics]
    steps:
      - uses: actions/checkout@v4

      - uses: getlumos/lumos-action@v1
        with:
          schema: 'packages/${{ matrix.package }}/schemas/*.lumos'
          working-directory: 'packages/${{ matrix.package }}'
```

**Benefits:**
- ✅ Each package validated independently
- ✅ Parallel execution (faster for large monorepos)
- ✅ Failures isolated per package
- ✅ Package-specific drift detection

**Advanced Features:**

**Per-Package Drift Detection** - Matrix strategy with path filtering:

```yaml
jobs:
  detect-changes:
    outputs:
      packages: ${{ steps.filter.outputs.changes }}
    steps:
      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            auth: 'packages/auth/**'
            payments: 'packages/payments/**'

  validate:
    needs: detect-changes
    strategy:
      matrix:
        package: ${{ fromJSON(needs.detect-changes.outputs.packages) }}
    steps:
      - uses: getlumos/lumos-action@v1
        with:
          schema: 'packages/${{ matrix.package }}/schemas/*.lumos'
```

**Selective Failure Strategies** - Criticality-based validation tiers:

```yaml
jobs:
  # Critical packages - must pass (blocks merge)
  critical:
    strategy:
      matrix:
        package: [auth, payments]
    steps:
      - uses: getlumos/lumos-action@v1
        with:
          fail-on-drift: true

  # Standard packages - warn only (allows merge)
  standard:
    continue-on-error: true
    strategy:
      matrix:
        package: [users, analytics]
    steps:
      - uses: getlumos/lumos-action@v1
        with:
          fail-on-drift: true
```

**Breaking Change Detection** - Git diff analysis for schema changes:

See [docs/monorepo-advanced.md](docs/monorepo-advanced.md) for:
- ✅ Per-package drift detection with matrix + path filters
- ✅ Selective failure strategies (criticality tiers)
- ✅ Breaking change detection (field removals, type changes)
- ✅ Package-specific build failure control
- ✅ Feature requests for native support

**Complete Examples:**
- [Monorepo Multi-Package](examples/workflows/monorepo-multi-package.yml) - 7 monorepo strategies
- [Per-Package Validation](examples/workflows/monorepo-per-package.yml) - Matrix with path filtering
- [Tiered Validation](examples/workflows/monorepo-tiered-validation.yml) - Criticality-based enforcement
- [Breaking Change Detection](examples/workflows/breaking-change-detection.yml) - Git diff analysis
- [Setup Guide](examples/monorepo-setup-guide.md) - Step-by-step monorepo setup

### Example Workflows

**Separate Rust and TypeScript:**
- [examples/workflows/separate-rust-typescript.yml](examples/workflows/separate-rust-typescript.yml)
  - 9 language separation patterns
  - Rename with schema names
  - Symbolic links
  - Language-specific processing

**Custom Output Organization:**
- [examples/workflows/custom-output-organization.yml](examples/workflows/custom-output-organization.yml)
  - 10 organization patterns
  - Module-based organization
  - Versioned outputs
  - Workspace integration (Cargo/npm)
  - CI/CD optimized structure

**Monorepo Multi-Package:**
- [examples/workflows/monorepo-multi-package.yml](examples/workflows/monorepo-multi-package.yml)
  - Matrix generation strategy
  - Centralized code collection
  - Auto-commit on main branch
  - Per-package custom naming

## Customizing PR Comments

The action provides default PR comments, but you can create custom formats using the outputs.

### Default Comment Behavior

When `comment-on-pr: true` (default), the action posts:
- Validation and generation statistics
- Drift status
- Collapsible diff summary (if drift detected)

### Output Format

The `diff-summary` output contains markdown-formatted text:

```markdown
## 📊 LUMOS Generation Drift Detected

The following files differ from their generated versions:

- `generated.rs`
- `generated.ts`

<details>
<summary>View full diff</summary>

```diff
[git diff output]
```

</details>
```

### Custom Comment Examples

**Minimal comment:**

```yaml
- uses: getlumos/lumos-action@v1
  id: lumos
  with:
    schema: 'schemas/**/*.lumos'
    comment-on-pr: false

- name: Custom minimal comment
  uses: actions/github-script@v7
  with:
    script: |
      const drift = '${{ steps.lumos.outputs.drift-detected }}' === 'true';
      const status = drift ? '⚠️ Drift detected' : '✅ All good';

      await github.rest.issues.createComment({
        issue_number: context.issue.number,
        owner: context.repo.owner,
        repo: context.repo.repo,
        body: `**LUMOS:** ${status}`
      });
```

**Team mentions on drift:**

```yaml
- name: Notify team on drift
  if: steps.lumos.outputs.drift-detected == 'true'
  uses: actions/github-script@v7
  with:
    script: |
      await github.rest.issues.createComment({
        issue_number: context.issue.number,
        owner: context.repo.owner,
        repo: context.repo.repo,
        body: '@org/schema-maintainers Schema drift detected - please review'
      });
```

**Parsed file list:**

```yaml
- name: Parse and list changed files
  uses: actions/github-script@v7
  with:
    script: |
      const diffSummary = `${{ steps.lumos.outputs.diff-summary }}`;
      const files = [...diffSummary.matchAll(/`([^`]+\.(rs|ts))`/g)]
        .map(m => m[1]);

      const comment = `**Changed files:** ${files.length}\n\n` +
        files.map(f => `- \`${f}\``).join('\n');

      await github.rest.issues.createComment({
        issue_number: context.issue.number,
        owner: context.repo.owner,
        repo: context.repo.repo,
        body: comment
      });
```

**Auto-label on drift:**

```yaml
- name: Add drift label
  if: steps.lumos.outputs.drift-detected == 'true'
  uses: actions/github-script@v7
  with:
    script: |
      await github.rest.issues.addLabels({
        issue_number: context.issue.number,
        owner: context.repo.owner,
        repo: context.repo.repo,
        labels: ['schema-drift']
      });
```

See [examples/workflows/custom-pr-comments.yml](examples/workflows/custom-pr-comments.yml) for complete examples.

## Understanding Failures

The action can fail for different reasons. Understanding the difference helps with debugging.

### Failure Types

| Type | Description | When It Fails | Configurable? |
|------|-------------|---------------|---------------|
| **Validation Error** | Schema syntax or type errors | Always | ❌ No - Must fix |
| **Generation Error** | Code generation failed | Always | ❌ No - Must fix |
| **Drift Warning** | Generated ≠ committed files | Based on `fail-on-drift` | ✅ Yes |

### Decision Matrix

```
┌─────────────────────┬──────────────────┬──────────────┐
│ Condition           │ fail-on-drift    │ Result       │
├─────────────────────┼──────────────────┼──────────────┤
│ Validation error    │ true/false       │ ❌ FAIL      │
│ Generation error    │ true/false       │ ❌ FAIL      │
│ Drift detected      │ true             │ ❌ FAIL      │
│ Drift detected      │ false            │ ⚠️  WARN     │
│ No drift            │ true/false       │ ✅ PASS      │
└─────────────────────┴──────────────────┴──────────────┘
```

### Validation Errors (Always Fail)

**Cause:** Syntax errors, undefined types, invalid attributes in `.lumos` files

**Symptoms:**
- `schemas-validated: 0` in outputs
- Error messages in logs: "expected `struct`", "undefined type", etc.

**Fix:**
```bash
# Validate locally
lumos validate schemas/**/*.lumos

# Check for errors
lumos check schemas/**/*.lumos
```

### Generation Errors (Always Fail)

**Cause:** Code generator encountered an error

**Symptoms:**
- `schemas-generated: 0` in outputs
- Schemas validated successfully, but generation failed

**Fix:**
- Check LUMOS CLI version compatibility
- Review generated code requirements
- Check for unsupported features in schemas

### Drift Warnings (Configurable)

**Cause:** Generated code differs from committed files

**Symptoms:**
- `drift-detected: true` in outputs
- Schemas validated and generated successfully
- Git diff shows changes in `generated.rs` / `generated.ts`

**Fix:**
```bash
# Regenerate code
lumos generate schemas/**/*.lumos

# Commit changes
git add generated.rs generated.ts
git commit -m "Update generated code from schemas"
```

**Control behavior:**
```yaml
# Fail on drift (strict - blocks PRs)
fail-on-drift: true

# Warn only (lenient - allows auto-commit)
fail-on-drift: false
```

### Output Reference

| Output | Type | Description | Example |
|--------|------|-------------|---------|
| `schemas-validated` | number | Count of validated schemas | `3` |
| `schemas-generated` | number | Count of generated schemas | `3` |
| `drift-detected` | boolean | Whether drift exists | `true` / `false` |
| `diff-summary` | string | Markdown-formatted diff | See format above |

### Common Error Messages

**"Schema validation failed"**
- **Type:** Validation Error
- **Action:** Fix syntax in `.lumos` files

**"Code generation failed"**
- **Type:** Generation Error
- **Action:** Check generator compatibility, review CLI version

**"Drift detected and fail-on-drift is enabled"**
- **Type:** Drift Warning (configured to fail)
- **Action:** Regenerate code or set `fail-on-drift: false`

See [examples/workflows/error-handling.yml](examples/workflows/error-handling.yml) for handling strategies.

## Troubleshooting

### Drift Always Detected

**Problem:** Drift detected even when files should match

**Causes:**
1. Line ending differences (CRLF vs LF)
2. Inconsistent rustfmt versions
3. Trailing whitespace differences
4. Generated code version mismatch

**Solutions:**
```yaml
# 1. Normalize line endings in .gitattributes
*.rs text eol=lf
*.ts text eol=lf

# 2. Pin LUMOS CLI version
- uses: getlumos/lumos-action@v1
  with:
    version: '0.1.1'  # Specific version

# 3. Format generated files consistently
- run: |
    cargo fmt
    git add generated.rs
```

### Installation Failures

**Problem:** LUMOS CLI fails to install

**Causes:**
1. Version doesn't exist on crates.io
2. Rust toolchain incompatibility
3. Network connectivity issues
4. Cargo cache corruption

**Solutions:**
```yaml
# 1. Verify version exists
- uses: getlumos/lumos-action@v1
  with:
    version: 'latest'  # Use latest stable

# 2. Clear cargo cache (in workflow)
- run: rm -rf ~/.cargo/registry/cache

# 3. Use fallback version
- uses: getlumos/lumos-action@v1
  with:
    version: '0.1.1'  # Known working version
  continue-on-error: true
```

### PR Comments Not Appearing

**Problem:** Expected PR comment doesn't appear

**Causes:**
1. Not running in PR context
2. `comment-on-pr: false` set
3. Insufficient permissions
4. Event type not `pull_request`

**Solutions:**
```yaml
# 1. Check event type
on:
  pull_request:  # Required for PR comments
    branches: [main]

# 2. Enable comments explicitly
- uses: getlumos/lumos-action@v1
  with:
    comment-on-pr: true

# 3. Grant write permissions
permissions:
  pull-requests: write
  contents: read
```

### Output Parsing Issues

**Problem:** Can't parse `diff-summary` output

**Format:** The output is markdown with this structure:
```markdown
## 📊 LUMOS Generation Drift Detected

- `file1.rs`
- `file2.ts`

<details>
...
</details>
```

**Parse files:**
```javascript
const diffSummary = `${{ steps.lumos.outputs.diff-summary }}`;
const files = [...diffSummary.matchAll(/`([^`]+\.(rs|ts))`/g)]
  .map(m => m[1]);
```

**Parse sections:**
```javascript
const hasDetails = diffSummary.includes('<details>');
const isDrift = diffSummary.includes('Drift Detected');
```

See [examples/workflows/diff-parsing.yml](examples/workflows/diff-parsing.yml) for more parsing examples.

### Permissions Errors

**Problem:** "Resource not accessible by integration"

**Cause:** Missing GitHub token permissions

**Solution:**
```yaml
permissions:
  contents: write      # For pushing commits
  pull-requests: write # For PR comments
  issues: write        # For issue comments

jobs:
  generate:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
    steps:
      - uses: getlumos/lumos-action@v1
```

### False Positive Drift

**Problem:** Drift detected on first run after setup

**Cause:** Generated files never committed initially

**Solution:**
```bash
# Initial setup - generate and commit
lumos generate schemas/**/*.lumos
git add generated.rs generated.ts
git commit -m "Initial generated code"
git push

# Now CI will compare against this baseline
```

## Versioning

This action uses semantic versioning. You can reference it in several ways:

- `getlumos/lumos-action@v1` - Latest v1.x release (recommended)
- `getlumos/lumos-action@v1.0.0` - Specific version
- `getlumos/lumos-action@main` - Latest commit (not recommended for production)

## License

Licensed under either of Apache License, Version 2.0 or MIT license at your option.

## Support

- Documentation: https://lumos-lang.org/tools/github-action
- Issues: https://github.com/getlumos/lumos/issues
- Discussions: https://github.com/getlumos/lumos/discussions
