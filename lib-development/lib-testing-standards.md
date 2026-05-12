---
inclusion: always
---

# Library Testing Standards

This document defines the testing strategy for TypeScript library development, covering unit tests, integration tests, and consumer compatibility testing.

## Overview

Libraries require a higher testing bar than application code because bugs affect all consumers. The testing strategy focuses on:
- Correctness of the public API contract
- Edge cases and error handling
- Backward compatibility across versions
- Cross-environment compatibility (ESM/CJS, different Node.js versions)

## Testing Strategy

### Test Layers

| Layer | Tool | Scope | Speed |
|-------|------|-------|-------|
| Unit | Jest + ts-jest | Individual functions, classes, utilities | Fast (ms) |
| Integration | Jest | Module interactions, real I/O (optional) | Medium (ms-s) |
| Contract | Jest | Public API behavior guarantees | Fast (ms) |
| Compatibility | CI matrix | Node.js versions, module systems | Slow (s) |

### What to Test at Each Layer

**Unit Tests** (`*.spec.ts`):
- Individual function behavior with specific inputs/outputs
- Class methods in isolation
- Error conditions and edge cases
- Input validation logic
- Internal utility functions

**Integration Tests** (`test/integration/*.test.ts`):
- Interactions between internal modules
- Real I/O operations (file system, network) when applicable
- End-to-end library workflows

**Contract Tests** (`test/contract/*.test.ts`):
- Public API surface behaves as documented
- Return types match declared types
- Error types match documented errors
- Backward-compatible behavior is preserved

### Property-Based Testing Policy

- **DO NOT** implement property-based tests unless explicitly requested by the user
- Focus on example-based tests with specific inputs and expected outputs
- Property-based testing adds complexity and is not part of the standard testing approach
- If property-based tests are marked as optional in specs (with `*`), skip them by default

## Project Structure

```
project-root/
├── src/
│   ├── core/
│   │   ├── client.ts
│   │   └── client.spec.ts          # Unit test (co-located)
│   ├── utils/
│   │   ├── retry.ts
│   │   └── retry.spec.ts           # Unit test (co-located)
│   └── errors/
│       ├── index.ts
│       └── index.spec.ts
├── test/
│   ├── integration/
│   │   ├── client.integration.test.ts
│   │   └── setup/
│   │       └── test-server.ts       # Mock HTTP server for integration
│   ├── contract/
│   │   ├── public-api.test.ts       # API contract tests
│   │   └── error-contracts.test.ts
│   ├── compatibility/
│   │   ├── esm.test.mjs            # ESM import test
│   │   └── cjs.test.cjs            # CJS require test
│   └── fixtures/
│       └── sample-data.ts
├── jest.config.ts
└── package.json
```

## Configuration

### jest.config.ts

```typescript
import type { Config } from 'jest';

const config: Config = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  roots: ['<rootDir>/src', '<rootDir>/test'],
  testMatch: [
    '<rootDir>/src/**/*.spec.ts',
    '<rootDir>/test/**/*.test.ts',
  ],
  moduleNameMapper: {
    '^@/(.*)$': '<rootDir>/src/$1',
  },
  collectCoverageFrom: [
    'src/**/*.ts',
    '!src/**/*.spec.ts',
    '!src/**/index.ts',
    '!src/types/**',
  ],
  coverageThreshold: {
    global: {
      branches: 90,
      statements: 90,
      lines: 90,
      functions: 90,
    },
  },
  clearMocks: true,
  restoreMocks: true,
};

export default config;
```

> **Note**: Libraries use a higher coverage threshold (90%) than applications because library bugs have a wider blast radius.

## Unit Testing

### Test Structure

```typescript
describe('ClassName', () => {
  describe('methodName', () => {
    it('should return expected result for valid input', () => {
      // Arrange
      // Act
      // Assert
    });

    it('should throw ValidationError when input is invalid', () => {
      // Arrange
      // Act & Assert
    });

    it('should handle edge case gracefully', () => {
      // Arrange
      // Act
      // Assert
    });
  });
});
```

### Example: Core Module Tests

```typescript
// src/core/client.spec.ts
import { Client } from './client';
import { ValidationError, TimeoutError } from '../errors';

describe('Client', () => {
  describe('constructor', () => {
    it('should create instance with valid options', () => {
      const client = new Client({
        baseUrl: 'https://api.example.com',
        timeout: 5000,
      });

      expect(client).toBeInstanceOf(Client);
    });

    it('should throw ValidationError when baseUrl is missing', () => {
      expect(() => new Client({ baseUrl: '' })).toThrow(ValidationError);
    });

    it('should throw ValidationError when timeout is negative', () => {
      expect(
        () => new Client({ baseUrl: 'https://api.example.com', timeout: -1 })
      ).toThrow(ValidationError);
    });

    it('should use default timeout when not specified', () => {
      const client = new Client({ baseUrl: 'https://api.example.com' });

      expect(client.getTimeout()).toBe(30000);
    });
  });

  describe('request', () => {
    let client: Client;

    beforeEach(() => {
      client = new Client({
        baseUrl: 'https://api.example.com',
        timeout: 5000,
      });
    });

    it('should return data on successful request', async () => {
      const mockFetch = jest.fn().mockResolvedValue({
        ok: true,
        json: () => Promise.resolve({ id: 1, name: 'Test' }),
      });
      global.fetch = mockFetch;

      const result = await client.get('/users/1');

      expect(result).toEqual({ id: 1, name: 'Test' });
      expect(mockFetch).toHaveBeenCalledWith(
        'https://api.example.com/users/1',
        expect.objectContaining({ method: 'GET' })
      );
    });

    it('should throw TimeoutError when request exceeds timeout', async () => {
      global.fetch = jest.fn().mockImplementation(
        () => new Promise((resolve) => setTimeout(resolve, 10000))
      );

      await expect(client.get('/slow')).rejects.toThrow(TimeoutError);
    });
  });
});
```

### Example: Utility Function Tests

```typescript
// src/utils/retry.spec.ts
import { retry, RetryOptions } from './retry';
import { RetryExhaustedError } from '../errors';

describe('retry', () => {
  beforeEach(() => {
    jest.useFakeTimers();
  });

  afterEach(() => {
    jest.useRealTimers();
  });

  it('should return result on first successful attempt', async () => {
    const fn = jest.fn().mockResolvedValue('success');

    const result = await retry(fn, { maxAttempts: 3 });

    expect(result).toBe('success');
    expect(fn).toHaveBeenCalledTimes(1);
  });

  it('should retry on failure and succeed on subsequent attempt', async () => {
    const fn = jest.fn()
      .mockRejectedValueOnce(new Error('fail'))
      .mockResolvedValue('success');

    const resultPromise = retry(fn, { maxAttempts: 3, delay: 100 });
    jest.advanceTimersByTime(100);
    const result = await resultPromise;

    expect(result).toBe('success');
    expect(fn).toHaveBeenCalledTimes(2);
  });

  it('should throw RetryExhaustedError after all attempts fail', async () => {
    const error = new Error('persistent failure');
    const fn = jest.fn().mockRejectedValue(error);

    const resultPromise = retry(fn, { maxAttempts: 3, delay: 100 });
    jest.advanceTimersByTime(300);

    await expect(resultPromise).rejects.toThrow(RetryExhaustedError);
    expect(fn).toHaveBeenCalledTimes(3);
  });

  it('should apply exponential backoff between retries', async () => {
    const fn = jest.fn()
      .mockRejectedValueOnce(new Error('fail'))
      .mockRejectedValueOnce(new Error('fail'))
      .mockResolvedValue('success');

    const resultPromise = retry(fn, {
      maxAttempts: 3,
      delay: 100,
      backoffMultiplier: 2,
    });

    // First retry after 100ms
    jest.advanceTimersByTime(100);
    await Promise.resolve();

    // Second retry after 200ms (100 * 2)
    jest.advanceTimersByTime(200);
    const result = await resultPromise;

    expect(result).toBe('success');
    expect(fn).toHaveBeenCalledTimes(3);
  });

  it('should call onRetry callback for each retry attempt', async () => {
    const onRetry = jest.fn();
    const fn = jest.fn()
      .mockRejectedValueOnce(new Error('fail'))
      .mockResolvedValue('success');

    const resultPromise = retry(fn, {
      maxAttempts: 3,
      delay: 100,
      onRetry,
    });
    jest.advanceTimersByTime(100);
    await resultPromise;

    expect(onRetry).toHaveBeenCalledTimes(1);
    expect(onRetry).toHaveBeenCalledWith(expect.any(Error), 1);
  });

  it('should not retry when shouldRetry returns false', async () => {
    const fn = jest.fn().mockRejectedValue(new Error('non-retriable'));

    const resultPromise = retry(fn, {
      maxAttempts: 3,
      delay: 100,
      shouldRetry: () => false,
    });

    await expect(resultPromise).rejects.toThrow('non-retriable');
    expect(fn).toHaveBeenCalledTimes(1);
  });
});
```

### Example: Error Class Tests

```typescript
// src/errors/index.spec.ts
import {
  LibraryError,
  ValidationError,
  TimeoutError,
  RetryExhaustedError,
} from './index';

describe('Error Classes', () => {
  describe('LibraryError', () => {
    it('should set name and message', () => {
      const error = new LibraryError('something failed');

      expect(error.name).toBe('LibraryError');
      expect(error.message).toBe('something failed');
      expect(error).toBeInstanceOf(Error);
    });

    it('should preserve cause', () => {
      const cause = new Error('original');
      const error = new LibraryError('wrapped', cause);

      expect(error.cause).toBe(cause);
    });
  });

  describe('ValidationError', () => {
    it('should be instanceof LibraryError', () => {
      const error = new ValidationError('invalid input');

      expect(error).toBeInstanceOf(LibraryError);
      expect(error).toBeInstanceOf(ValidationError);
      expect(error.name).toBe('ValidationError');
    });
  });

  describe('TimeoutError', () => {
    it('should include timeout duration', () => {
      const error = new TimeoutError('request timed out', 5000);

      expect(error.timeoutMs).toBe(5000);
      expect(error).toBeInstanceOf(LibraryError);
    });
  });

  describe('RetryExhaustedError', () => {
    it('should include attempt count and cause', () => {
      const cause = new Error('last failure');
      const error = new RetryExhaustedError('all retries failed', 3, cause);

      expect(error.attempts).toBe(3);
      expect(error.cause).toBe(cause);
      expect(error).toBeInstanceOf(LibraryError);
    });
  });
});
```

## Contract Testing

Contract tests verify the public API behaves as documented. They serve as regression guards against accidental breaking changes.

### Public API Contract Tests

```typescript
// test/contract/public-api.test.ts
import * as lib from '../../src/index';

describe('Public API Contract', () => {
  describe('exports', () => {
    it('should export Client class', () => {
      expect(lib.Client).toBeDefined();
      expect(typeof lib.Client).toBe('function');
    });

    it('should export createClient factory', () => {
      expect(lib.createClient).toBeDefined();
      expect(typeof lib.createClient).toBe('function');
    });

    it('should export error classes', () => {
      expect(lib.ValidationError).toBeDefined();
      expect(lib.TimeoutError).toBeDefined();
      expect(lib.RetryExhaustedError).toBeDefined();
    });
  });

  describe('createClient', () => {
    it('should return a Client instance', () => {
      const client = lib.createClient({
        baseUrl: 'https://api.example.com',
      });

      expect(client).toBeInstanceOf(lib.Client);
    });

    it('should accept all documented options', () => {
      // This test ensures the options interface hasn't changed
      const client = lib.createClient({
        baseUrl: 'https://api.example.com',
        timeout: 5000,
        retries: 3,
        headers: { 'X-Custom': 'value' },
      });

      expect(client).toBeInstanceOf(lib.Client);
    });
  });

  describe('error hierarchy', () => {
    it('should maintain error inheritance chain', () => {
      const validationError = new lib.ValidationError('test');
      const timeoutError = new lib.TimeoutError('test', 1000);

      expect(validationError).toBeInstanceOf(Error);
      expect(validationError).toBeInstanceOf(lib.LibraryError);
      expect(timeoutError).toBeInstanceOf(Error);
      expect(timeoutError).toBeInstanceOf(lib.LibraryError);
    });
  });
});
```

### Error Contract Tests

```typescript
// test/contract/error-contracts.test.ts
import { createClient, ValidationError, TimeoutError } from '../../src/index';

describe('Error Contracts', () => {
  describe('ValidationError is thrown for invalid inputs', () => {
    it('should throw ValidationError for empty baseUrl', () => {
      expect(() => createClient({ baseUrl: '' })).toThrow(ValidationError);
    });

    it('should throw ValidationError for invalid timeout', () => {
      expect(
        () => createClient({ baseUrl: 'https://api.example.com', timeout: -1 })
      ).toThrow(ValidationError);
    });
  });

  describe('error messages are descriptive', () => {
    it('should include field name in validation error', () => {
      try {
        createClient({ baseUrl: '' });
      } catch (error) {
        expect((error as ValidationError).message).toContain('baseUrl');
      }
    });
  });
});
```

## Integration Testing

### Mock Server Setup

For libraries that make HTTP requests, use a local mock server:

```typescript
// test/integration/setup/test-server.ts
import { createServer, IncomingMessage, ServerResponse } from 'node:http';
import type { AddressInfo } from 'node:net';

export interface MockRoute {
  method: string;
  path: string;
  status: number;
  body?: unknown;
  delay?: number;
}

export function createTestServer(routes: MockRoute[]) {
  const server = createServer(
    async (req: IncomingMessage, res: ServerResponse) => {
      const route = routes.find(
        (r) => r.method === req.method && r.path === req.url
      );

      if (!route) {
        res.writeHead(404);
        res.end(JSON.stringify({ error: 'Not found' }));
        return;
      }

      if (route.delay) {
        await new Promise((resolve) => setTimeout(resolve, route.delay));
      }

      res.writeHead(route.status, { 'Content-Type': 'application/json' });
      res.end(JSON.stringify(route.body ?? {}));
    }
  );

  return {
    start: (): Promise<string> =>
      new Promise((resolve) => {
        server.listen(0, () => {
          const { port } = server.address() as AddressInfo;
          resolve(`http://localhost:${port}`);
        });
      }),
    close: (): Promise<void> =>
      new Promise((resolve) => {
        server.close(() => resolve());
      }),
  };
}
```

### Integration Test Example

```typescript
// test/integration/client.integration.test.ts
import { createClient } from '../../src/index';
import { createTestServer, MockRoute } from './setup/test-server';

describe('Client Integration', () => {
  let baseUrl: string;
  let server: ReturnType<typeof createTestServer>;

  const routes: MockRoute[] = [
    { method: 'GET', path: '/users/1', status: 200, body: { id: 1, name: 'Alice' } },
    { method: 'GET', path: '/users/999', status: 404, body: { error: 'Not found' } },
    { method: 'POST', path: '/users', status: 201, body: { id: 2, name: 'Bob' } },
    { method: 'GET', path: '/slow', status: 200, body: { ok: true }, delay: 5000 },
  ];

  beforeAll(async () => {
    server = createTestServer(routes);
    baseUrl = await server.start();
  });

  afterAll(async () => {
    await server.close();
  });

  it('should fetch data from a real HTTP endpoint', async () => {
    const client = createClient({ baseUrl });

    const result = await client.get('/users/1');

    expect(result).toEqual({ id: 1, name: 'Alice' });
  });

  it('should handle 404 responses', async () => {
    const client = createClient({ baseUrl });

    await expect(client.get('/users/999')).rejects.toThrow();
  });

  it('should timeout on slow responses', async () => {
    const client = createClient({ baseUrl, timeout: 100 });

    await expect(client.get('/slow')).rejects.toThrow('timeout');
  });

  it('should send POST requests with body', async () => {
    const client = createClient({ baseUrl });

    const result = await client.post('/users', { name: 'Bob' });

    expect(result).toEqual({ id: 2, name: 'Bob' });
  });
});
```

## Compatibility Testing

### Module System Tests

Verify the library works in both ESM and CJS environments:

```javascript
// test/compatibility/cjs.test.cjs
const assert = require('node:assert');
const { createClient, ValidationError } = require('../../dist/cjs/index.js');

// Verify CJS import works
assert(typeof createClient === 'function', 'createClient should be a function');
assert(typeof ValidationError === 'function', 'ValidationError should be a function');

console.log('CJS compatibility: PASS');
```

```javascript
// test/compatibility/esm.test.mjs
import assert from 'node:assert';
import { createClient, ValidationError } from '../../dist/esm/index.js';

// Verify ESM import works
assert(typeof createClient === 'function', 'createClient should be a function');
assert(typeof ValidationError === 'function', 'ValidationError should be a function');

console.log('ESM compatibility: PASS');
```

### CI Matrix Testing

Test across multiple Node.js versions:

```yaml
name: CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [20, 22, 24]

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Type check
        run: npm run type-check

      - name: Lint
        run: npm run lint

      - name: Unit tests
        run: npm test -- --coverage

      - name: Build
        run: npm run build

      - name: Module compatibility tests
        run: |
          node test/compatibility/cjs.test.cjs
          node test/compatibility/esm.test.mjs

      - name: Upload coverage
        if: matrix.node-version == 22
        uses: codecov/codecov-action@v4
```

## Mocking Patterns

### Mocking External Dependencies

```typescript
// When the library wraps an external service
jest.mock('node:https', () => ({
  request: jest.fn(),
}));
```

### Mocking Time

```typescript
describe('cache expiration', () => {
  beforeEach(() => {
    jest.useFakeTimers();
  });

  afterEach(() => {
    jest.useRealTimers();
  });

  it('should expire cached values after TTL', () => {
    const cache = new Cache({ ttl: 60000 });
    cache.set('key', 'value');

    jest.advanceTimersByTime(61000);

    expect(cache.get('key')).toBeUndefined();
  });
});
```

### Mocking File System

```typescript
import { vol } from 'memfs';

jest.mock('node:fs', () => require('memfs').fs);
jest.mock('node:fs/promises', () => require('memfs').fs.promises);

beforeEach(() => {
  vol.reset();
});

it('should read config from file', async () => {
  vol.fromJSON({ '/config.json': '{"key": "value"}' });

  const config = await loadConfig('/config.json');

  expect(config).toEqual({ key: 'value' });
});
```

## NPM Scripts

```json
{
  "scripts": {
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage",
    "test:integration": "jest --testPathPattern=test/integration",
    "test:contract": "jest --testPathPattern=test/contract",
    "test:compat": "npm run build && node test/compatibility/cjs.test.cjs && node test/compatibility/esm.test.mjs",
    "test:all": "npm run test:coverage && npm run test:compat",
    "test:ci": "jest --ci --coverage"
  }
}
```

## Running Tests

### Local Development

```bash
# Run unit tests (co-located with source)
npm test

# Watch mode for development
npm run test:watch

# Run with coverage report
npm run test:coverage

# Run integration tests only
npm run test:integration

# Run contract tests only
npm run test:contract

# Run module compatibility tests (requires build first)
npm run test:compat

# Run everything
npm run test:all
```

## Best Practices

### 1. Test the Public API

- Focus tests on the public API surface, not internal implementation
- If you need to test internals, consider whether they should be extracted into their own module
- Contract tests prevent accidental breaking changes

### 2. Test Edge Cases Thoroughly

Libraries encounter more edge cases than applications because they serve diverse consumers:
- Empty inputs, null, undefined
- Very large inputs (strings, arrays, numbers)
- Unicode and special characters
- Concurrent access patterns
- Boundary values (0, -1, MAX_SAFE_INTEGER)

### 3. Test Error Paths

- Every documented error should have a test proving it's thrown
- Error messages should be tested for clarity
- Error properties (cause, metadata) should be verified
- Ensure errors are instanceof the documented class

### 4. Avoid Test Coupling

- Each test should be independent
- Don't rely on test execution order
- Use `beforeEach` for setup, not shared mutable state
- Clean up side effects in `afterEach`

### 5. Coverage Goals

- **Unit tests**: 90%+ coverage (libraries need higher confidence)
- **Branch coverage**: Pay special attention — uncovered branches are likely bugs
- **Integration tests**: Cover all documented usage patterns
- **Contract tests**: Cover all public exports and their documented behavior

### 6. Performance Testing (Optional)

For performance-sensitive libraries, include benchmark tests:

```typescript
describe('performance', () => {
  it('should process 10,000 items in under 100ms', () => {
    const items = Array.from({ length: 10000 }, (_, i) => ({ id: i }));
    const start = performance.now();

    processItems(items);

    const duration = performance.now() - start;
    expect(duration).toBeLessThan(100);
  });
});
```

### 7. Snapshot Testing

Use sparingly for complex output structures:

```typescript
it('should generate expected configuration object', () => {
  const config = generateConfig({ environment: 'production' });

  expect(config).toMatchSnapshot();
});
```

Review snapshot changes carefully — they can mask unintended changes.

## Debugging Tests

- Use `test.only()` to isolate a single test
- Use `--verbose` flag for detailed output
- Use `--detectOpenHandles` to find hanging async operations
- Use `--bail` to stop on first failure during debugging
- Add `console.log` or use `node --inspect` with `--runInBand` for debugging

## Pre-Publish Verification

Before publishing a new version, always run:

```bash
# Full verification pipeline
npm run type-check
npm run lint
npm run test:coverage
npm run build
npm run test:compat
```

This ensures the published package is correct, well-typed, and compatible across environments.
