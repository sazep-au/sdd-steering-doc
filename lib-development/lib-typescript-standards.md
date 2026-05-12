---
inclusion: always
---

# TypeScript Library Development Standards

This document defines the coding standards, architecture patterns, and best practices for developing shared TypeScript libraries (npm packages, internal SDKs, utility modules).

## Project Overview

- **Runtime**: Node.js >= 20.0.0 (target multiple LTS versions)
- **Language**: TypeScript 5.3+
- **Module System**: Dual ESM + CJS output
- **Testing**: Jest with ts-jest (or Vitest)
- **Validation**: Zod schemas (where applicable)
- **Build Tool**: tsup or tsc for compilation
- **Linting**: ESLint + Prettier
- **Package Manager**: npm or pnpm

## Architecture Principles

### Design for Consumers

Libraries exist to serve consumers. Every design decision should prioritize:
- **Minimal API surface** — expose only what consumers need
- **Tree-shakeable exports** — allow bundlers to eliminate unused code
- **Zero or minimal dependencies** — reduce supply chain risk and bundle size
- **Clear contracts** — explicit types, documented behavior, predictable errors

### Single Responsibility

Each library should have a focused purpose:

```
Good:  @org/logger, @org/http-client, @org/validation-utils
Bad:   @org/utils (kitchen-sink package)
```

### Semantic Versioning

Follow semver strictly:
- **MAJOR** — Breaking changes to public API
- **MINOR** — New features, backward-compatible
- **PATCH** — Bug fixes, backward-compatible

## Project Structure

```
project-root/
├── src/
│   ├── index.ts                # Public API barrel export
│   ├── core/                   # Core implementation
│   │   ├── client.ts
│   │   └── client.spec.ts
│   ├── utils/                  # Internal utilities
│   │   ├── retry.ts
│   │   └── retry.spec.ts
│   ├── types/                  # Public type definitions
│   │   └── index.ts
│   └── errors/                 # Custom error classes
│       └── index.ts
├── test/
│   ├── integration/            # Integration tests
│   └── fixtures/               # Test fixtures and mocks
├── docs/                       # API documentation
│   └── api.md
├── package.json
├── tsconfig.json               # Source TypeScript config
├── tsconfig.build.json         # Build-specific config (excludes tests)
├── tsup.config.ts              # Build tool config (if using tsup)
├── jest.config.ts              # Test configuration
├── .eslintrc.js                # Linting rules
├── .prettierrc                 # Formatting rules
├── CHANGELOG.md                # Version history
└── README.md                   # Usage documentation
```

## File Naming Conventions

- Use kebab-case for file names: `http-client.ts`, `retry-policy.ts`
- Test files: `*.spec.ts` co-located with source files
- Index files: `index.ts` for barrel exports (public API only)
- Type files: `*.types.ts` or grouped in `types/` directory

## TypeScript Standards

### Strict Mode

All strict TypeScript compiler options must be enabled:

```json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "forceConsistentCasingInFileNames": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  }
}
```

### Type Definitions

- Prefer interfaces for object shapes that consumers may extend
- Use type aliases for unions, intersections, and utility types
- Always define explicit return types for public functions
- Avoid `any` — use `unknown` if the type is truly unknown
- Export all public types from the package entry point
- Use generics to provide flexible, type-safe APIs

```typescript
// Public interface — consumers can extend
export interface ClientOptions {
  baseUrl: string;
  timeout?: number;
  retries?: number;
  headers?: Record<string, string>;
}

// Generic result type
export type Result<T, E = Error> =
  | { success: true; data: T }
  | { success: false; error: E };

// Constrained generics
export function parse<T extends z.ZodType>(
  schema: T,
  data: unknown
): z.infer<T> {
  return schema.parse(data);
}
```

### Import Organization

1. Node built-in modules (`node:` prefix)
2. External dependencies (node_modules)
3. Internal absolute imports (from `src/`)
4. Relative imports
5. Type-only imports last

```typescript
import { EventEmitter } from 'node:events';
import { z } from 'zod';
import { RetryPolicy } from '../utils/retry';
import type { ClientOptions, Result } from '../types';
```

### Async/Await

- Always use async/await over raw Promises
- Handle errors with try/catch blocks
- Avoid mixing callbacks with async/await
- Return typed Promises with explicit generic parameters

## Public API Design

### Barrel Exports

Only export what consumers need from `src/index.ts`:

```typescript
// src/index.ts — Public API
export { Client } from './core/client';
export { createClient } from './core/factory';
export { RetryPolicy } from './utils/retry';

// Types
export type { ClientOptions, ClientResponse } from './types';
export type { RetryOptions } from './utils/retry';

// Errors
export { ClientError, TimeoutError, ValidationError } from './errors';
```

### API Ergonomics

- Provide both class-based and functional APIs where appropriate
- Use the builder pattern for complex configuration
- Provide sensible defaults for all optional parameters
- Support method chaining where it improves readability

```typescript
// Functional API
export function createClient(options: ClientOptions): Client {
  return new Client(options);
}

// Builder pattern for complex config
export class ClientBuilder {
  private options: Partial<ClientOptions> = {};

  baseUrl(url: string): this {
    this.options.baseUrl = url;
    return this;
  }

  timeout(ms: number): this {
    this.options.timeout = ms;
    return this;
  }

  build(): Client {
    if (!this.options.baseUrl) {
      throw new ValidationError('baseUrl is required');
    }
    return new Client(this.options as ClientOptions);
  }
}
```

### Defensive Programming

- Validate all public function inputs at the boundary
- Throw descriptive errors with actionable messages
- Use custom error classes for different failure modes
- Never expose internal implementation details through errors

```typescript
export function createClient(options: ClientOptions): Client {
  if (!options.baseUrl) {
    throw new ValidationError('baseUrl is required');
  }
  if (options.timeout !== undefined && options.timeout <= 0) {
    throw new ValidationError('timeout must be a positive number');
  }
  return new Client(options);
}
```

## Error Handling

### Custom Error Classes

Define a hierarchy of errors for the library:

```typescript
// src/errors/index.ts
export class LibraryError extends Error {
  constructor(message: string, public readonly cause?: unknown) {
    super(message);
    this.name = 'LibraryError';
  }
}

export class ValidationError extends LibraryError {
  constructor(message: string) {
    super(message);
    this.name = 'ValidationError';
  }
}

export class TimeoutError extends LibraryError {
  constructor(message: string, public readonly timeoutMs: number) {
    super(message);
    this.name = 'TimeoutError';
  }
}

export class RetryExhaustedError extends LibraryError {
  constructor(
    message: string,
    public readonly attempts: number,
    cause?: unknown
  ) {
    super(message, cause);
    this.name = 'RetryExhaustedError';
  }
}
```

### Error Guidelines

- Always extend a base library error class
- Include relevant context in error properties
- Preserve the original error as `cause`
- Use `instanceof` checks for error discrimination
- Document which errors each public method can throw

## Module System and Build

### Dual ESM/CJS Output

Support both module systems for maximum compatibility:

```json
{
  "name": "@org/my-library",
  "version": "1.0.0",
  "type": "module",
  "main": "./dist/cjs/index.js",
  "module": "./dist/esm/index.js",
  "types": "./dist/types/index.d.ts",
  "exports": {
    ".": {
      "import": {
        "types": "./dist/types/index.d.ts",
        "default": "./dist/esm/index.js"
      },
      "require": {
        "types": "./dist/types/index.d.ts",
        "default": "./dist/cjs/index.js"
      }
    }
  },
  "files": ["dist", "README.md", "CHANGELOG.md"]
}
```

### tsup Configuration

```typescript
// tsup.config.ts
import { defineConfig } from 'tsup';

export default defineConfig({
  entry: ['src/index.ts'],
  format: ['cjs', 'esm'],
  dts: true,
  sourcemap: true,
  clean: true,
  splitting: false,
  treeshake: true,
  minify: false,
  target: 'node20',
});
```

### tsconfig.build.json

```json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "outDir": "./dist",
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  },
  "include": ["src/**/*"],
  "exclude": ["src/**/*.spec.ts", "src/**/*.test.ts", "test/**/*"]
}
```

## Dependency Management

### Dependency Categories

- **dependencies** — Runtime requirements shipped to consumers (minimize these)
- **peerDependencies** — Packages consumers must provide (e.g., `zod`, AWS SDK clients)
- **devDependencies** — Build, test, and development tools only

### Guidelines

- Pin exact versions for `dependencies` to avoid unexpected breakage
- Use ranges for `peerDependencies` to allow consumer flexibility
- Prefer zero dependencies when possible — inline small utilities
- Audit dependencies regularly for security vulnerabilities
- Avoid packages with excessive transitive dependencies

```json
{
  "dependencies": {},
  "peerDependencies": {
    "zod": "^3.20.0"
  },
  "peerDependenciesMeta": {
    "zod": { "optional": true }
  },
  "devDependencies": {
    "zod": "3.22.4",
    "typescript": "5.3.3",
    "tsup": "8.0.1",
    "jest": "29.7.0",
    "ts-jest": "29.1.1"
  }
}
```

## Documentation

### README Structure

Every library README should include:
1. **What it does** — One-paragraph description
2. **Installation** — npm/pnpm install command
3. **Quick Start** — Minimal working example
4. **API Reference** — Key functions and classes
5. **Configuration** — Available options with defaults
6. **Error Handling** — How to handle library errors
7. **Migration Guide** — For major version bumps

### JSDoc Comments

Document all public APIs with JSDoc:

```typescript
/**
 * Creates a new HTTP client with the given configuration.
 *
 * @param options - Client configuration options
 * @returns A configured client instance
 * @throws {ValidationError} When required options are missing or invalid
 *
 * @example
 * ```typescript
 * const client = createClient({
 *   baseUrl: 'https://api.example.com',
 *   timeout: 5000,
 * });
 * ```
 */
export function createClient(options: ClientOptions): Client {
  // ...
}
```

### CHANGELOG

Maintain a CHANGELOG.md following [Keep a Changelog](https://keepachangelog.com/) format:

```markdown
# Changelog

## [1.2.0] - 2025-03-15

### Added
- Support for custom retry policies

### Fixed
- Timeout not respected on first request

## [1.1.0] - 2025-02-01

### Added
- `onRetry` callback option
```

## Performance Considerations

- Avoid unnecessary allocations in hot paths
- Use lazy initialization for expensive resources
- Provide synchronous alternatives where async is not required
- Minimize closure captures in frequently called functions
- Profile before optimizing — measure, don't guess
- Document performance characteristics for consumers

## Security Best Practices

- Never log or expose sensitive data (tokens, credentials, PII)
- Validate all inputs at public API boundaries
- Use parameterized operations — never interpolate user input into commands
- Keep dependencies minimal and audited
- Document security considerations for consumers
- Support secure defaults (e.g., TLS, timeouts, input limits)

## Versioning and Release

### Pre-release Workflow

1. Develop on feature branches
2. Run full test suite before merging
3. Update CHANGELOG.md
4. Bump version in package.json
5. Tag the release commit
6. Publish to registry

### Scripts

```json
{
  "scripts": {
    "build": "tsup",
    "dev": "tsup --watch",
    "type-check": "tsc --noEmit",
    "lint": "eslint src/ --ext .ts",
    "lint:fix": "eslint src/ --ext .ts --fix",
    "format": "prettier --write 'src/**/*.ts'",
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage",
    "prepublishOnly": "npm run build",
    "clean": "rm -rf dist"
  }
}
```

## Git Workflow

### Commits

- Use conventional commits format (enforced by commitlint)
- Format: `type(scope): message`
- Types: feat, fix, docs, style, refactor, test, chore, perf

### Branch Strategy

- `main` — Published releases
- `develop` — Integration branch (optional)
- `feature/*` — Feature branches
- `fix/*` — Bug fix branches
- `release/*` — Release preparation

## Logging

Libraries should **not** include their own logger by default. Instead:

- Accept an optional logger interface from the consumer
- Use a no-op logger as the default
- Never write to `console.*` in production code
- Provide a debug mode that consumers can opt into

```typescript
export interface LibraryLogger {
  debug(message: string, context?: Record<string, unknown>): void;
  info(message: string, context?: Record<string, unknown>): void;
  warn(message: string, context?: Record<string, unknown>): void;
  error(message: string, context?: Record<string, unknown>): void;
}

const noopLogger: LibraryLogger = {
  debug: () => {},
  info: () => {},
  warn: () => {},
  error: () => {},
};

export interface ClientOptions {
  baseUrl: string;
  logger?: LibraryLogger;
}

export class Client {
  private readonly logger: LibraryLogger;

  constructor(options: ClientOptions) {
    this.logger = options.logger ?? noopLogger;
  }
}
```

This allows consumers to plug in their own logger (e.g., `@sazep/sazep-logger`, pino, winston) without the library imposing a dependency.
