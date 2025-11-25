# Next.js Codebase Guide for AI Assistants

This document provides a comprehensive overview of the Next.js repository structure, development workflows, and key conventions to help AI assistants effectively navigate and contribute to the codebase.

**Last Updated:** 2025-11-25
**Repository:** vercel/next.js
**Primary Branch:** canary

---

## Table of Contents

1. [Repository Overview](#repository-overview)
2. [Directory Structure](#directory-structure)
3. [Development Workflow](#development-workflow)
4. [Build System](#build-system)
5. [Testing Strategy](#testing-strategy)
6. [Code Organization](#code-organization)
7. [Key Conventions](#key-conventions)
8. [Common Commands](#common-commands)
9. [Contributing Guidelines](#contributing-guidelines)
10. [Troubleshooting](#troubleshooting)

---

## Repository Overview

Next.js is a monorepo containing the Next.js framework, CLI tools, examples, and supporting packages. It uses:

- **Package Manager:** pnpm (v9.6.0 - exact version required)
- **Build Tool:** Turbo (for monorepo orchestration)
- **Languages:** TypeScript, JavaScript, Rust
- **Node Version:** >=18.18.0
- **Compilers:** SWC (Rust-based), Babel (legacy)
- **Bundlers:** Webpack 5 (default), Turbopack (experimental), Rspack (experimental)

### Architecture

Next.js is a hybrid system with components written in both TypeScript/JavaScript and Rust:

```
┌─────────────────────────────────────────┐
│  Next.js Framework                      │
├─────────────────────────────────────────┤
│  TypeScript/JavaScript Layer            │
│  - Client-side runtime                  │
│  - Server-side runtime                  │
│  - Build orchestration                  │
│  - CLI                                  │
├─────────────────────────────────────────┤
│  Rust Layer (Performance-Critical)      │
│  - SWC compiler                         │
│  - Turbopack bundler                    │
│  - Native bindings (NAPI)               │
└─────────────────────────────────────────┘
```

---

## Directory Structure

### Root Level

```
next.js/
├── packages/               # NPM packages (main framework code)
├── crates/                 # Rust crates for performance-critical features
├── test/                   # Comprehensive test suites
├── examples/               # 230+ example projects
├── bench/                  # Performance benchmarks
├── apps/                   # Internal applications (docs, etc.)
├── docs/                   # Documentation source
├── errors/                 # Error definitions and documentation
├── scripts/                # Build and utility scripts
├── turbopack/              # Turbopack bundler source
├── contributing/           # Contribution guides
├── .github/                # GitHub workflows and actions
└── [config files]          # Root configuration files
```

### Key Packages (`packages/`)

| Package | Purpose | Size |
|---------|---------|------|
| `next/` | **Core Next.js framework** - Main package | ~300K+ lines |
| `create-next-app/` | CLI for creating new Next.js projects | Small |
| `next-swc/` | SWC integration (Rust-based compiler) | Native bindings |
| `next-codemod/` | Automated code upgrade tool | Medium |
| `eslint-plugin-next/` | ESLint plugin for Next.js rules | Small |
| `eslint-config-next/` | ESLint configuration | Small |
| `font/` | @next/font implementation | Medium |
| `next-mdx/` | MDX support | Small |
| `third-parties/` | Third-party integration | Medium |

### Rust Crates (`crates/`)

| Crate | Purpose |
|-------|---------|
| `napi/` | Node.js native bindings for Rust functionality |
| `next-api/` | Core Rust API implementations |
| `next-core/` | Core Next.js logic in Rust (Turbopack integration) |
| `next-build/` | Rust-based build system |
| `next-custom-transforms/` | Custom compiler transforms (Babel/SWC) |
| `next-error-code-swc-plugin/` | Error code SWC plugin |
| `wasm/` | WebAssembly implementations |

---

## Development Workflow

### Initial Setup

```bash
# 1. Clone repository (shallow clone for faster download)
gh repo clone vercel/next.js -- --filter=blob:none --branch canary --single-branch

# 2. Install dependencies
corepack enable pnpm
pnpm install

# 3. Build the project (required before testing)
pnpm build

# 4. Start development mode (watch for changes)
pnpm dev

# 5. In a separate terminal, compile type definitions
pnpm types
```

### Dependencies Required

- **Rust & Cargo:** Install via [rustup](https://rustup.rs)
- **GitHub CLI:** [Installation guide](https://github.com/cli/cli#installation)
- **pnpm:** Enable via `corepack enable pnpm`
- **Linux only:** LLD (LLVM linker) and Clang: `sudo apt install lld clang`

### Development Cycle

```bash
# Create a new branch from canary
git checkout -b MY_BRANCH_NAME origin/canary

# Make changes and watch for compilation
pnpm dev        # Terminal 1: Watch TypeScript compilation
pnpm types      # Terminal 2: Generate type definitions (run as needed)

# Test changes
pnpm test-dev test/e2e/app-dir/app/     # Development mode tests
pnpm test-start test/e2e/app-dir/app/   # Production mode tests
pnpm test-unit                          # Unit tests

# Lint and format
pnpm lint       # Run all linters
pnpm lint-fix   # Auto-fix issues

# Commit changes
git add .
git commit -m "type: description"

# Create pull request (always target canary)
gh pr create
```

### Testing Local Changes in External Projects

Since symlinks don't work well with Turbopack, use the pack-next workflow:

```bash
# Method 1: Create tarball and unpack
pnpm pack-next --tar && pnpm unpack-next /path/to/project

# Method 2: Without tarball (faster for development)
pnpm patch-next /path/to/project

# Skip build for faster iterations (if you're running pnpm dev)
pnpm pack-next --no-js-build --tar
```

---

## Build System

### Build Architecture

Next.js uses a multi-stage build process:

```
┌─────────────────────────────────────────────────┐
│ 1. TypeScript → JavaScript (SWC)               │
│    packages/next/src/ → packages/next/dist/    │
├─────────────────────────────────────────────────┤
│ 2. Bundle Runtime (Webpack)                    │
│    dist/ → dist/bundles/                       │
├─────────────────────────────────────────────────┤
│ 3. Generate Type Definitions (tsc)             │
│    src/ → dist/*.d.ts                          │
├─────────────────────────────────────────────────┤
│ 4. Compile Rust (cargo)                        │
│    crates/ → target/release/                   │
└─────────────────────────────────────────────────┘
```

### Build Tools

| Tool | Purpose | Config File |
|------|---------|-------------|
| **Taskr** | Build task orchestration | `packages/next/taskfile.js` |
| **SWC** | TypeScript → JavaScript compilation | `.swcrc` |
| **Webpack 5** | Runtime bundling | `next-runtime.webpack-config.js` |
| **TypeScript** | Type definition generation | `tsconfig.build.json` |
| **Turbo** | Monorepo task runner | `turbo.json` |
| **Cargo** | Rust compilation | `Cargo.toml` |

### Key Build Commands

```bash
# Full build (all packages + Rust)
pnpm build

# Build specific parts
pnpm types              # Type definitions only
pnpm swc-build-native   # Rust code only
pnpm swc-build-wasm     # WASM build

# Clean build artifacts
pnpm clean              # Clean all
pnpm sweep              # Deep clean (includes cargo cache)
```

### Build Tasks (via taskr)

```bash
cd packages/next

# Individual tasks
pnpm taskr compile      # SWC compilation
pnpm taskr ncc          # Bundle third-party dependencies
pnpm taskr webpack      # Webpack bundling
pnpm taskr release      # Full release build
```

---

## Testing Strategy

### Test Types

Next.js uses multiple test types for comprehensive coverage:

| Test Type | Location | Purpose | Runs Against |
|-----------|----------|---------|--------------|
| **E2E** | `test/e2e/` | Isolated end-to-end tests | next dev + next start + Vercel |
| **Development** | `test/development/` | Development mode specific tests | next dev |
| **Production** | `test/production/` | Production mode specific tests | next build + next start |
| **Integration** | `test/integration/` | Legacy tests (not isolated) | Various modes |
| **Unit** | `test/unit/` | Fast unit tests | No browser/server |

### Test Commands

```bash
# Run specific test suites
pnpm test-dev test/e2e/app-dir/app/       # Development mode
pnpm test-start test/e2e/app-dir/app/     # Production mode
pnpm test-unit                            # All unit tests

# Debug tests (shows browser)
pnpm testonly-dev test/e2e/app-dir/app/
pnpm testonly-start test/e2e/app-dir/app/

# Test with Turbopack instead of Webpack
pnpm test-dev-turbo test/e2e/app-dir/app/
pnpm test-start-turbo test/e2e/app-dir/app/

# Test with Rspack
pnpm test-dev-rspack test/e2e/app-dir/app/

# Run all tests (takes hours)
pnpm test
```

### Test Utilities

Tests use `nextTestSetup` utility for isolation:

```typescript
// Example test setup (see test/e2e/example.txt)
import { nextTestSetup } from 'e2e-utils'

describe('my feature', () => {
  const { next } = nextTestSetup({
    files: __dirname,
    dependencies: {
      // external dependencies
    },
  })

  it('should work', async () => {
    // Test implementation
  })
})
```

### Debugging Environment Variables

```bash
# Keep temp folder after test for debugging
NEXT_TEST_SKIP_CLEANUP=1 pnpm test-dev test/e2e/app-dir/app/

# Run in monorepo instead of temp folder (faster, limited use)
NEXT_SKIP_ISOLATE=1 pnpm test-dev test/e2e/app-dir/app/

# Use offline mode for package installation
NEXT_TEST_PREFER_OFFLINE=1 pnpm test-dev test/e2e/app-dir/app/

# Enable test profiling
NEXT_TEST_TRACE=1 pnpm test-dev test/e2e/app-dir/app/

# Specify test mode manually
NEXT_TEST_MODE=dev pnpm testonly test/e2e/app-dir/app/
```

### Creating New Tests

```bash
# Generate test from template
pnpm new-test

# Follow prompts to select test type and location
```

### Best Practices for Tests

1. **Isolation:** Use `nextTestSetup` for e2e/development/production tests
2. **Wait for conditions:** Use `browser.waitForElement` or `check` utility
3. **Verify test failures:** Ensure test fails without your fix
4. **TypeScript:** Write new tests in TypeScript (`.ts`/`.tsx`)
5. **Organized:** Add related tests to existing suites when appropriate

---

## Code Organization

### packages/next/src/ Structure

The main Next.js package is organized into focused modules:

```
packages/next/src/
├── api/                    # Public API re-exports
│   ├── link.ts            # Re-exports client/link
│   ├── image.ts           # Re-exports client/image
│   ├── navigation.ts      # Re-exports client/navigation
│   └── ...                # Other public APIs
│
├── bin/                    # CLI entry points
│   └── next.ts            # Main CLI entry (next dev/build/start/etc.)
│
├── build/                  # Build system logic (~140KB)
│   ├── index.ts           # Main build orchestration
│   ├── entries.ts         # Entry point resolution
│   ├── webpack/           # Webpack configuration
│   ├── babel/             # Babel plugins
│   ├── swc/               # SWC configuration
│   └── segment-config/    # Route segment config
│
├── client/                 # Client-side runtime
│   ├── components/        # Built-in components (Link, Image, Script)
│   ├── app-*.tsx          # App Router components
│   ├── link.tsx           # Link component implementation
│   ├── image.tsx          # Image component implementation
│   ├── dev/               # Dev overlay and error handling
│   └── lib/               # Client utilities
│
├── server/                 # Server-side runtime
│   ├── base-server.ts     # Base HTTP server
│   ├── app-render/        # App Router rendering
│   ├── route-*/           # Route handling and matching
│   ├── dev/               # Development server
│   ├── request/           # Request utilities (headers, cookies)
│   ├── response-cache/    # Response caching
│   ├── async-storage/     # Async context storage
│   ├── lib/               # Server utilities
│   └── api-utils/         # API route utilities
│
├── lib/                    # Shared utilities
│   ├── metadata/          # Metadata generation
│   ├── typescript/        # TypeScript config generation
│   ├── eslint/            # ESLint integration
│   ├── fs/                # File system utilities
│   └── helpers/           # Various helpers
│
├── shared/                 # Shared code (client + server)
│   └── lib/               # Shared utilities
│
├── compiled/               # 140+ bundled dependencies
│   ├── webpack/
│   ├── postcss/
│   └── ...                # Pre-bundled to avoid version conflicts
│
├── bundles/               # Pre-bundled runtime code
├── cli/                   # CLI utilities
├── diagnostic/            # Error and warning messages
├── telemetry/             # Analytics
└── trace/                 # Build tracing
```

### Entry Points and Public API

All public APIs are re-exported through `packages/next/src/api/`:

```typescript
// packages/next/src/api/link.ts
export { default } from '../client/link'
export * from '../client/link'

// This allows: import Link from 'next/link'
```

**Public API Surface:**
- `next/link` - Link component
- `next/image` - Image component
- `next/script` - Script component
- `next/navigation` - Navigation hooks (App Router)
- `next/router` - Router (Pages Router)
- `next/server` - Middleware and server utilities
- `next/cache` - Cache utilities
- `next/headers` - Header utilities
- `next/font/google` - Google Fonts
- `next/font/local` - Local fonts

---

## Key Conventions

### File Naming

- **Components:** PascalCase (e.g., `Link.tsx`, `Image.tsx`)
- **Utilities:** camelCase (e.g., `getPageFiles.ts`, `normalizePagePath.ts`)
- **Tests:** `*.test.ts` or `*.test.tsx`
- **Config:** kebab-case (e.g., `next.config.js`, `tsconfig.json`)

### Code Style

The repository enforces code style through:

1. **ESLint:** JavaScript/TypeScript linting
   - Config: `.eslintrc.json` (IDE) + `.eslintrc.cli.json` (CLI/CI)
   - Run: `pnpm lint-eslint`

2. **Prettier:** Code formatting
   - Config: `.prettierrc.json`
   - Run: `pnpm prettier-check` or `pnpm prettier-fix`

3. **AlexJS:** Inclusive language linting
   - Config: `.alexrc`
   - Run: `pnpm lint-language`

4. **ast-grep:** Pattern-based linting
   - Run: `pnpm lint-ast-grep`

5. **TypeScript:** Type checking
   - Config: `tsconfig.json`
   - Run: `pnpm typescript` or `pnpm lint-typescript`

### Commit Conventions

Follow conventional commit format:

```
type(scope): description

Examples:
- fix: resolve routing issue with dynamic routes
- feat: add support for custom error pages
- docs: update installation instructions
- test: add tests for image optimization
- refactor: simplify metadata generation logic
- perf: optimize bundle size
- chore: update dependencies
```

### Import Organization

Imports should be organized in this order:

1. External dependencies (React, Node.js modules)
2. Internal Next.js modules (from `next/`)
3. Relative imports (from `./` or `../`)
4. Type imports (with `import type`)

```typescript
// Example
import React from 'react'
import { readFile } from 'fs/promises'

import { getPageFiles } from '../lib/get-page-files'
import { normalizePagePath } from './normalize-page-path'

import type { PageFile } from '../types'
```

### TypeScript Conventions

- Use TypeScript for all new code
- Prefer interfaces over types for object shapes
- Use explicit return types for public APIs
- Avoid `any`; use `unknown` if type is truly unknown
- Use type guards for runtime type checking

### Error Handling

Errors are centrally defined:

1. Create error documentation in `errors/` directory
2. Define error in `packages/next/errors.json`
3. Reference error code in code: `new Error('NEXT_ERROR_CODE')`
4. Build validates all error codes exist

### Testing Patterns

1. **Isolated tests preferred:** Use `nextTestSetup` for new tests
2. **Test file naming:** `<feature>.test.ts`
3. **Describe blocks:** Group related tests
4. **Async/await:** Use for asynchronous operations
5. **Playwright:** Browser automation for e2e tests

---

## Common Commands

### Development

```bash
pnpm dev                    # Watch mode for TypeScript compilation
pnpm types                  # Generate type definitions
pnpm next                   # Run local Next.js CLI
pnpm debug                  # Run Next.js with Node inspector
pnpm debug-brk              # Run Next.js with Node inspector (break on start)
```

### Building

```bash
pnpm build                  # Build all packages
pnpm clean                  # Clean build artifacts
pnpm sweep                  # Deep clean (cargo cache, etc.)
pnpm pack-next              # Create tarballs for local testing
pnpm unpack-next <path>     # Unpack tarballs into project
pnpm patch-next <path>      # Patch local Next.js into project
```

### Testing

```bash
pnpm test                   # Run all tests (takes hours)
pnpm test-dev <path>        # Development mode tests
pnpm test-start <path>      # Production mode tests
pnpm test-unit              # Unit tests only
pnpm testonly <path>        # Run specific test file
pnpm test-turbo             # Test with Turbopack
pnpm new-test               # Generate new test from template
```

### Linting

```bash
pnpm lint                   # Run all linters
pnpm lint-fix               # Auto-fix lint issues
pnpm lint-eslint            # ESLint only
pnpm lint-typescript        # TypeScript type checking
pnpm prettier-check         # Check code formatting
pnpm prettier-fix           # Fix code formatting
pnpm lint-language          # Check inclusive language (alex)
```

### Rust/Native

```bash
pnpm swc-build-native       # Build Rust code (SWC)
pnpm swc-build-wasm         # Build WebAssembly
pnpm build-turbopack-cli    # Build Turbopack CLI
```

### Monorepo

```bash
pnpm lerna                  # Run lerna commands
pnpm check-examples         # Validate example projects
pnpm check-precompiled      # Validate pre-compiled dependencies
```

---

## Contributing Guidelines

### Before You Start

1. **Search existing issues/PRs:** Check if your bug/feature already exists
2. **Read contributing docs:** Review files in `contributing/` directory
3. **Understand the feature:** For new features, create a discussion first
4. **Start small:** Consider starting with a small bug fix or documentation improvement

### Feature Requests

Before implementing a new feature:

1. Create a [Feature Request Discussion](https://github.com/vercel/next.js/discussions/new?category=ideas)
2. Gather community feedback
3. Wait for approval from maintainers
4. Review existing [RFCs](https://github.com/vercel/next.js/discussions/categories/rfc) for examples

**Why?**
- Verify feature validity
- Understand maintenance implications
- Consider ecosystem impact
- Review historical decisions

### Pull Request Process

1. **Branch from canary:**
   ```bash
   git checkout -b my-feature origin/canary
   ```

2. **Make focused changes:**
   - Keep PRs small and focused
   - One feature/fix per PR
   - Don't refactor unrelated code

3. **Test thoroughly:**
   ```bash
   pnpm build
   pnpm test-dev test/e2e/relevant-test/
   pnpm test-unit
   pnpm lint
   ```

4. **Write good commit messages:**
   - Use conventional commit format
   - Be descriptive but concise
   - Reference issues if applicable

5. **Create PR:**
   ```bash
   gh pr create
   ```
   - Fill out the PR template completely
   - Target the `canary` branch
   - Add relevant labels
   - Request review if needed

### Code Review

- Respond to feedback promptly
- Be open to suggestions
- Ask questions if unclear
- Make requested changes
- Keep discussions focused and professional

### Documentation

When adding features:

1. Update relevant documentation in `docs/`
2. Add error documentation in `errors/` if applicable
3. Create examples in `examples/` for complex features
4. Update TypeScript types

---

## Troubleshooting

### Common Issues

#### Build Failures

```bash
# Clean everything and rebuild
pnpm clean
pnpm install
pnpm build

# Deep clean (includes cargo cache)
pnpm sweep
pnpm install
pnpm build
```

#### Type Errors

```bash
# Regenerate type definitions
pnpm types

# Check for type errors
pnpm typescript
```

#### Rust Build Issues

```bash
# Install Rust if not installed
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Rebuild native code
pnpm swc-build-native

# Linux: Install required dependencies
sudo apt install lld clang
```

#### Test Failures

```bash
# Keep test artifacts for debugging
NEXT_TEST_SKIP_CLEANUP=1 pnpm test-dev test/e2e/failing-test/

# Then debug the temp folder
cd /tmp/next-test-*
pnpm debug

# Run tests without isolation (faster for debugging)
NEXT_SKIP_ISOLATE=1 pnpm testonly-dev test/e2e/failing-test/
```

#### pnpm Issues

```bash
# Ensure correct pnpm version
corepack enable pnpm
corepack prepare pnpm@9.6.0 --activate

# Clear pnpm cache
pnpm store prune
```

#### Disk Space Issues

```bash
# Clean up Rust artifacts (can save GBs)
pnpm sweep

# macOS: Enable disk compression for node_modules and target
./scripts/LaunchAgents/install-macos-agents.sh
```

### Getting Help

- **Documentation:** Check `contributing/` directory
- **Discussions:** [GitHub Discussions](https://github.com/vercel/next.js/discussions)
- **Issues:** [GitHub Issues](https://github.com/vercel/next.js/issues)
- **Discord:** [Next.js Discord](https://nextjs.org/discord)

---

## Architecture Patterns

### Server Components vs. Client Components

```typescript
// Server Component (default in App Router)
// - Runs only on server
// - Can access backend resources
// - Cannot use hooks or browser APIs
export default async function Page() {
  const data = await fetchData()
  return <div>{data}</div>
}

// Client Component (with 'use client')
// - Runs on both server and client
// - Can use hooks and browser APIs
// - Cannot access backend resources directly
'use client'
import { useState } from 'react'
export default function Counter() {
  const [count, setCount] = useState(0)
  return <button onClick={() => setCount(count + 1)}>{count}</button>
}
```

### Routing Architecture

**App Router** (recommended):
- File-system based routing in `app/` directory
- Supports React Server Components
- Nested layouts and templates
- Advanced features: parallel routes, intercepting routes

**Pages Router** (legacy):
- File-system based routing in `pages/` directory
- All components are client components
- Simpler mental model
- Still fully supported

### Caching Strategy

Three-layer caching system:

1. **In-Memory Cache:** Fast, per-process
2. **Filesystem Cache:** Persistent between restarts
3. **Custom Cache Handler:** User-provided (Redis, etc.)

### Middleware System

- Runs on Edge Runtime
- Early request interception
- Defined in `middleware.ts` at root or in route groups
- Can modify request/response headers
- Cannot use Node.js APIs

---

## File Patterns to Recognize

### Route Files (App Router)

```
app/
├── page.tsx               # Route page
├── layout.tsx             # Layout wrapper
├── template.tsx           # Re-rendering layout
├── loading.tsx            # Loading UI
├── error.tsx              # Error UI
├── not-found.tsx          # 404 UI
├── route.ts               # API route
├── default.tsx            # Parallel route fallback
├── [param]/               # Dynamic route
├── [...slug]/             # Catch-all route
├── [[...slug]]/           # Optional catch-all
├── (group)/               # Route group (not in URL)
├── @modal/                # Parallel route
└── (.)modal/              # Intercepting route
```

### Special Files

```
next.config.js             # Next.js configuration
middleware.ts              # Edge middleware
instrumentation.ts         # Lifecycle hooks
.env.local                 # Environment variables
tsconfig.json              # TypeScript config (generated)
```

### Test Files

```
test/
└── e2e/
    └── my-feature/
        ├── my-feature.test.ts      # Test file
        ├── app/                    # Test fixture app
        │   ├── page.tsx
        │   └── layout.tsx
        └── next.config.js          # Test-specific config
```

---

## Key Files Reference

### Build & Configuration

| File | Purpose |
|------|---------|
| `packages/next/taskfile.js` | Build task definitions |
| `packages/next/next-runtime.webpack-config.js` | Webpack runtime config |
| `packages/next/tsconfig.build.json` | TypeScript build config |
| `turbo.json` | Turbo (monorepo) configuration |
| `Cargo.toml` | Rust workspace configuration |
| `pnpm-workspace.yaml` | pnpm workspace definition |

### Entry Points

| File | Purpose |
|------|---------|
| `packages/next/src/bin/next.ts` | CLI entry point |
| `packages/next/src/server/next.ts` | Server API entry |
| `packages/next/index.js` | Main package export |
| `packages/next/types/index.d.ts` | Type definitions entry |

### Core Implementation

| File | Purpose | Size |
|------|---------|------|
| `packages/next/src/build/index.ts` | Build orchestration | ~140KB |
| `packages/next/src/server/base-server.ts` | Base HTTP server | ~50KB |
| `packages/next/src/server/app-render/` | App Router rendering | ~100KB |
| `packages/next/src/build/entries.ts` | Entry resolution | ~30KB |

---

## Best Practices for AI Assistants

When working with this codebase:

1. **Always build before testing:**
   ```bash
   pnpm build
   ```

2. **Use isolated tests for new features:**
   ```bash
   pnpm new-test
   ```

3. **Run focused test suites:**
   ```bash
   pnpm test-dev test/e2e/specific-feature/
   ```

4. **Check lint before committing:**
   ```bash
   pnpm lint
   ```

5. **Update types after changes:**
   ```bash
   pnpm types
   ```

6. **Test locally in external projects:**
   ```bash
   pnpm pack-next --tar && pnpm unpack-next /path/to/test-project
   ```

7. **Keep PRs focused:** One feature or fix per PR

8. **Write tests:** Every bug fix or feature needs tests

9. **Document public APIs:** Add JSDoc comments to exported functions

10. **Consider performance:** Next.js serves millions of sites

---

## Additional Resources

- **Contributing Guide:** [contributing.md](./contributing.md)
- **Developing Guide:** [contributing/core/developing.md](./contributing/core/developing.md)
- **Testing Guide:** [contributing/core/testing.md](./contributing/core/testing.md)
- **Building Guide:** [contributing/core/building.md](./contributing/core/building.md)
- **Next.js Documentation:** https://nextjs.org/docs
- **Turbopack Documentation:** https://turbo.build/pack/docs
- **SWC Documentation:** https://swc.rs/docs

---

**Note:** This is a living document. As the repository evolves, this guide should be updated to reflect current practices and conventions.
