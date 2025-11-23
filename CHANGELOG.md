# Changelog

All notable changes to the LUMOS GitHub Action will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

- **Documentation:** `docs/monorepo-advanced.md` - Advanced monorepo strategies guide
  - Per-package drift detection with matrix strategy + path filtering
  - Selective failure strategies (criticality tiers)
  - Breaking change detection via git diff analysis
  - Feature requests for native support
  - Best practices and troubleshooting
- **Examples:** `examples/workflows/monorepo-per-package.yml` - Complete per-package validation
  - Path-based change detection with dorny/paths-filter@v3
  - Matrix strategy with package-specific comments
  - Validation summary across all packages
  - Optional scheduled validation for all packages
- **Examples:** `examples/workflows/monorepo-tiered-validation.yml` - Criticality-based enforcement
  - Tier 1 (Critical): auth, payments, security - Must pass, blocks merge
  - Tier 2 (Standard): users, profiles - Warn only, allows merge
  - Tier 3 (Experimental): analytics, tools - Informational only
  - Environment-based rules (production vs staging)
  - Manual approval workflow for critical packages
- **Examples:** `examples/workflows/breaking-change-detection.yml` - Git diff schema analysis
  - Detects field removals (breaking)
  - Detects type changes (breaking)
  - Detects enum variant removals (breaking)
  - Detects new required fields (potentially breaking)
  - Requires breaking-change label on PRs
  - Advanced Borsh compatibility validation
- **Examples:** `examples/monorepo-setup-guide.md` - Step-by-step monorepo setup
  - Directory structure recommendations
  - 3 validation strategies (simple matrix, path-filtered, tiered)
  - Complete workflow setup guide
  - Path filter configuration
  - Local testing instructions
  - Deployment checklist
  - Troubleshooting guide
- **Documentation:** Expanded "Monorepo Support" section in README
  - Per-package drift detection examples
  - Selective failure strategies examples
  - Breaking change detection overview
  - Links to comprehensive guides and examples

### Added (from previous work)

- **Documentation:** `docs/multi-language.md` - Comprehensive guide to dual-language generation
  - Why both Rust and TypeScript are generated
  - 3 language separation strategies (move, copy, symlinks)
  - Monorepo patterns (centralized vs distributed)
  - FAQ and feature request tracking
- **Documentation:** `docs/custom-output-paths.md` - Custom output location workarounds
  - 4 core workarounds (move, copy, rename, symlinks)
  - 3 monorepo examples (centralized, package-specific, build artifacts)
  - Advanced dynamic configuration
  - Import path updating strategies
- **Documentation:** "Advanced Usage" section in README
  - Multi-language generation overview
  - Custom output paths summary
  - Monorepo support with matrix strategy
  - Links to all comprehensive guides
- **Documentation:** "Customizing PR Comments" section with 5 inline examples
- **Documentation:** "Understanding Failures" section with decision matrix and error type guide
- **Documentation:** Enhanced "Troubleshooting" section with 6 common scenarios
- **Examples:** `examples/workflows/monorepo-multi-package.yml` - 7 monorepo strategies
  - Matrix generation (parallel, isolated failures)
  - Sequential generation (all packages in one job)
  - Centralized collection (all code in one directory)
  - Smart generation (only changed packages)
  - Auto-commit on main branch
  - Per-package custom naming
  - Validation matrix
- **Examples:** `examples/workflows/separate-rust-typescript.yml` - 9 language separation patterns
  - Simple move to separate directories
  - Rename with schema names
  - Copy instead of move
  - Symbolic links
  - Hierarchical organization
  - Language-specific processing
  - Rust-only and TypeScript-only workflows
  - .gitignore integration
- **Examples:** `examples/workflows/custom-output-organization.yml` - 10 organization patterns
  - Standard project layout (src/generated/)
  - Separate source directories (programs/ vs client/)
  - Build artifacts directory
  - Module-based organization
  - Environment-specific outputs
  - Versioned outputs
  - Dynamic configuration from JSON
  - Workspace-based (Cargo/npm)
  - CI/CD pipeline optimized
  - Git submodule ready
- **Examples:** `examples/workflows/custom-pr-comments.yml` - 5 PR comment customization examples
- **Examples:** `examples/workflows/error-handling.yml` - 6 error handling strategies
- **Examples:** `examples/workflows/diff-parsing.yml` - 6 diff parsing techniques
- Output reference table documenting all output types and formats
- Common error messages documentation
- `diff-summary` output format specification

### Improved

- README now includes Advanced Usage section (multi-language, custom paths, monorepo)
- Troubleshooting guide now includes solutions for 6 common issues
- Clear distinction between validation errors (always fail) vs drift warnings (configurable)
- Documentation of output parsing techniques with regex examples

### Context7 Benchmark Impact

- Q4 (Multi-language Generation): 28 → 80 (+52 points)
- Q5 (PR Comment Customization): 72 → 85 (+13 points)
- Q7 (Custom Output Paths): 65 → 80 (+15 points)
- Q8 (Error Type Documentation): 72 → 85 (+13 points)
- Q10 (Monorepo Per-Package Drift Detection): 28 → 80 (+52 points)
- Overall Score: 68.6 → 80.5 (+11.9 points)

## [1.0.0] - 2025-11-22

### Added

- Initial release of LUMOS GitHub Action
- Auto-install LUMOS CLI (any version from crates.io)
- Validate LUMOS schemas in CI/CD pipelines
- Generate Rust and TypeScript code from schemas
- Detect drift between generated and committed files
- Post PR comments with generation results and diffs
- Configurable failure modes (`fail-on-drift`, `check-only`)
- Support for glob patterns in schema paths
- Working directory customization
- Version pinning for reproducible builds

### Features

- **Inputs:**
  - `schema` - Path to schema files (supports globs)
  - `check-only` - Validation-only mode
  - `version` - LUMOS CLI version selection
  - `working-directory` - Custom working directory
  - `fail-on-drift` - Configurable drift handling
  - `comment-on-pr` - PR comment integration

- **Outputs:**
  - `schemas-validated` - Count of validated schemas
  - `schemas-generated` - Count of generated schemas
  - `drift-detected` - Boolean drift status
  - `diff-summary` - Detailed diff information

### Documentation

- Comprehensive README with usage examples
- Troubleshooting guide
- Workflow tips and patterns
- Monorepo support examples

[1.0.0]: https://github.com/getlumos/lumos-action/releases/tag/v1.0.0
