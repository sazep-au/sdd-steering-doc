---
inclusion: always
---

# Next.js Testing Standards

This document defines the testing strategy for Next.js applications, covering unit tests, integration tests, and end-to-end tests.

## Overview

A well-tested Next.js application uses a layered testing approach:

| Layer | Tool | Scope | Speed |
|-------|------|-------|-------|
| Unit | Jest + React Testing Library | Components, hooks, utilities | Fast (ms) |
| Integration | Jest + MSW | Server Actions, Route Handlers, data flow | Medium (ms-s) |
| End-to-End | Playwright | Full user flows, cross-page navigation | Slow (s) |

## Testing Strategy

### What to Test at Each Layer

**Unit Tests** (`*.test.ts` / `*.test.tsx`):
- Individual component rendering and behavior
- Custom hooks
- Utility functions
- Zod schema validation
- Conditional rendering logic

**Integration Tests** (`*.integration.test.ts`):
- Server Actions with mocked database
- Route Handlers with request/response cycle
- Form submission flows
- Data fetching and caching behavior

**End-to-End Tests** (`tests/e2e/*.spec.ts`):
- Critical user journeys (signup, login, checkout)
- Cross-page navigation
- Authentication flows
- Form submissions with real UI

### Property-Based Testing Policy

- **DO NOT** implement property-based tests unless explicitly requested by the user
- Focus on example-based tests with specific inputs and expected outputs
- Property-based testing adds complexity and is not part of the standard testing approach

## Project Structure

```
project-root/
├── src/
│   ├── components/
│   │   ├── user-card.tsx
│   │   └── user-card.test.tsx          # Unit test (co-located)
│   ├── hooks/
│   │   ├── use-debounce.ts
│   │   └── use-debounce.test.ts        # Hook test (co-located)
│   ├── lib/
│   │   ├── actions/
│   │   │   ├── user-actions.ts
│   │   │   └── user-actions.test.ts    # Server Action test
│   │   └── utils.ts
│   └── schemas/
│       ├── user-schema.ts
│       └── user-schema.test.ts
├── tests/
│   ├── e2e/
│   │   ├── auth.spec.ts                # Playwright E2E
│   │   ├── dashboard.spec.ts
│   │   └── fixtures/
│   ├── integration/
│   │   ├── api/
│   │   │   └── users.integration.test.ts
│   │   └── setup/
│   │       └── test-server.ts
│   └── setup/
│       ├── jest-setup.ts
│       └── mocks/
│           └── handlers.ts             # MSW handlers
├── jest.config.ts
├── playwright.config.ts
└── package.json
```

## Configuration

### jest.config.ts

```typescript
import type { Config } from 'jest';
import nextJest from 'next/jest';

const createJestConfig = nextJest({
  dir: './',
});

const config: Config = {
  testEnvironment: 'jsdom',
  setupFilesAfterFramework: ['<rootDir>/tests/setup/jest-setup.ts'],
  testMatch: [
    '<rootDir>/src/**/*.test.{ts,tsx}',
    '<rootDir>/tests/integration/**/*.test.ts',
  ],
  moduleNameMapper: {
    '^@/(.*)$': '<rootDir>/src/$1',
  },
  collectCoverageFrom: [
    'src/**/*.{ts,tsx}',
    '!src/**/*.test.{ts,tsx}',
    '!src/**/*.d.ts',
    '!src/types/**',
  ],
  coverageThreshold: {
    global: {
      branches: 80,
      statements: 80,
      lines: 80,
      functions: 80,
    },
  },
};

export default createJestConfig(config);
```

### tests/setup/jest-setup.ts

```typescript
import '@testing-library/jest-dom';

// Mock next/navigation
jest.mock('next/navigation', () => ({
  useRouter: () => ({
    push: jest.fn(),
    replace: jest.fn(),
    back: jest.fn(),
    refresh: jest.fn(),
  }),
  usePathname: () => '/',
  useSearchParams: () => new URLSearchParams(),
  redirect: jest.fn(),
}));

// Mock next/headers
jest.mock('next/headers', () => ({
  cookies: () => ({
    get: jest.fn(),
    set: jest.fn(),
    delete: jest.fn(),
  }),
  headers: () => new Headers(),
}));
```

### playwright.config.ts

```typescript
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests/e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: 'html',
  use: {
    baseURL: 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
  },
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] },
    },
    {
      name: 'mobile',
      use: { ...devices['iPhone 14'] },
    },
  ],
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
});
```

## Unit Testing

### Component Tests

```typescript
// src/components/user-card.test.tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { UserCard } from './user-card';

describe('UserCard', () => {
  const mockUser = {
    id: '1',
    name: 'Jane Doe',
    email: 'jane@example.com',
    avatar: '/avatars/jane.png',
  };

  it('renders user name and email', () => {
    render(<UserCard user={mockUser} />);

    expect(screen.getByText('Jane Doe')).toBeInTheDocument();
    expect(screen.getByText('jane@example.com')).toBeInTheDocument();
  });

  it('calls onSelect when clicked', async () => {
    const user = userEvent.setup();
    const onSelect = jest.fn();

    render(<UserCard user={mockUser} onSelect={onSelect} />);

    await user.click(screen.getByRole('button'));

    expect(onSelect).toHaveBeenCalledWith('1');
  });

  it('renders compact variant without email', () => {
    render(<UserCard user={mockUser} variant="compact" />);

    expect(screen.getByText('Jane Doe')).toBeInTheDocument();
    expect(screen.queryByText('jane@example.com')).not.toBeInTheDocument();
  });

  it('displays fallback avatar when image fails to load', () => {
    render(<UserCard user={{ ...mockUser, avatar: '' }} />);

    expect(screen.getByLabelText('User avatar')).toBeInTheDocument();
  });
});
```

### Hook Tests

```typescript
// src/hooks/use-debounce.test.ts
import { renderHook, act } from '@testing-library/react';
import { useDebounce } from './use-debounce';

describe('useDebounce', () => {
  beforeEach(() => {
    jest.useFakeTimers();
  });

  afterEach(() => {
    jest.useRealTimers();
  });

  it('returns initial value immediately', () => {
    const { result } = renderHook(() => useDebounce('hello', 500));

    expect(result.current).toBe('hello');
  });

  it('updates value after delay', () => {
    const { result, rerender } = renderHook(
      ({ value, delay }) => useDebounce(value, delay),
      { initialProps: { value: 'hello', delay: 500 } }
    );

    rerender({ value: 'world', delay: 500 });

    // Value should not have changed yet
    expect(result.current).toBe('hello');

    // Fast-forward time
    act(() => {
      jest.advanceTimersByTime(500);
    });

    expect(result.current).toBe('world');
  });

  it('cancels previous timeout on rapid changes', () => {
    const { result, rerender } = renderHook(
      ({ value, delay }) => useDebounce(value, delay),
      { initialProps: { value: 'a', delay: 500 } }
    );

    rerender({ value: 'b', delay: 500 });
    act(() => { jest.advanceTimersByTime(200); });

    rerender({ value: 'c', delay: 500 });
    act(() => { jest.advanceTimersByTime(500); });

    expect(result.current).toBe('c');
  });
});
```

### Schema Validation Tests

```typescript
// src/schemas/user-schema.test.ts
import { createUserSchema, updateUserSchema } from './user-schema';

describe('createUserSchema', () => {
  it('validates a correct user object', () => {
    const result = createUserSchema.safeParse({
      name: 'Jane Doe',
      email: 'jane@example.com',
      password: 'securePassword123!',
    });

    expect(result.success).toBe(true);
  });

  it('rejects missing required fields', () => {
    const result = createUserSchema.safeParse({
      name: 'Jane Doe',
    });

    expect(result.success).toBe(false);
    if (!result.success) {
      expect(result.error.issues).toHaveLength(2);
    }
  });

  it('rejects invalid email format', () => {
    const result = createUserSchema.safeParse({
      name: 'Jane Doe',
      email: 'not-an-email',
      password: 'securePassword123!',
    });

    expect(result.success).toBe(false);
  });

  it('rejects password shorter than 8 characters', () => {
    const result = createUserSchema.safeParse({
      name: 'Jane Doe',
      email: 'jane@example.com',
      password: 'short',
    });

    expect(result.success).toBe(false);
  });
});
```

## Integration Testing

### Server Action Tests

```typescript
// src/lib/actions/user-actions.test.ts
import { createUser, updateUser } from './user-actions';

// Mock the database
jest.mock('@/lib/db', () => ({
  db: {
    user: {
      create: jest.fn(),
      update: jest.fn(),
      findUnique: jest.fn(),
    },
  },
}));

// Mock next/cache
jest.mock('next/cache', () => ({
  revalidatePath: jest.fn(),
  revalidateTag: jest.fn(),
}));

import { db } from '@/lib/db';
import { revalidatePath } from 'next/cache';

describe('createUser', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  it('creates a user with valid data', async () => {
    const formData = new FormData();
    formData.set('name', 'Jane Doe');
    formData.set('email', 'jane@example.com');

    (db.user.create as jest.Mock).mockResolvedValue({
      id: '1',
      name: 'Jane Doe',
      email: 'jane@example.com',
    });

    await createUser(formData);

    expect(db.user.create).toHaveBeenCalledWith({
      data: { name: 'Jane Doe', email: 'jane@example.com' },
    });
    expect(revalidatePath).toHaveBeenCalledWith('/users');
  });

  it('returns validation errors for invalid data', async () => {
    const formData = new FormData();
    formData.set('name', '');
    formData.set('email', 'invalid');

    const result = await createUser(formData);

    expect(result?.error).toBeDefined();
    expect(db.user.create).not.toHaveBeenCalled();
  });

  it('handles database errors gracefully', async () => {
    const formData = new FormData();
    formData.set('name', 'Jane Doe');
    formData.set('email', 'jane@example.com');

    (db.user.create as jest.Mock).mockRejectedValue(new Error('DB connection failed'));

    const result = await createUser(formData);

    expect(result?.error).toBeDefined();
  });
});
```

### Route Handler Tests

```typescript
// tests/integration/api/users.integration.test.ts
import { GET, POST } from '@/app/api/users/route';
import { NextRequest } from 'next/server';

jest.mock('@/lib/db', () => ({
  db: {
    user: {
      findMany: jest.fn(),
      create: jest.fn(),
    },
  },
}));

import { db } from '@/lib/db';

function createRequest(url: string, options?: RequestInit) {
  return new NextRequest(new URL(url, 'http://localhost:3000'), options);
}

describe('GET /api/users', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  it('returns paginated users', async () => {
    const mockUsers = [
      { id: '1', name: 'User 1', email: 'user1@test.com' },
      { id: '2', name: 'User 2', email: 'user2@test.com' },
    ];
    (db.user.findMany as jest.Mock).mockResolvedValue(mockUsers);

    const request = createRequest('http://localhost:3000/api/users?page=1&limit=10');
    const response = await GET(request);
    const body = await response.json();

    expect(response.status).toBe(200);
    expect(body.data).toHaveLength(2);
    expect(body.page).toBe(1);
    expect(body.limit).toBe(10);
  });

  it('uses default pagination when params are missing', async () => {
    (db.user.findMany as jest.Mock).mockResolvedValue([]);

    const request = createRequest('http://localhost:3000/api/users');
    const response = await GET(request);
    const body = await response.json();

    expect(body.page).toBe(1);
    expect(body.limit).toBe(10);
  });
});

describe('POST /api/users', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  it('creates a user and returns 201', async () => {
    const newUser = { name: 'Jane Doe', email: 'jane@example.com' };
    (db.user.create as jest.Mock).mockResolvedValue({ id: '1', ...newUser });

    const request = createRequest('http://localhost:3000/api/users', {
      method: 'POST',
      body: JSON.stringify(newUser),
      headers: { 'Content-Type': 'application/json' },
    });

    const response = await POST(request);
    const body = await response.json();

    expect(response.status).toBe(201);
    expect(body.name).toBe('Jane Doe');
  });

  it('returns 400 for invalid request body', async () => {
    const request = createRequest('http://localhost:3000/api/users', {
      method: 'POST',
      body: JSON.stringify({ invalid: 'data' }),
      headers: { 'Content-Type': 'application/json' },
    });

    const response = await POST(request);

    expect(response.status).toBe(400);
  });
});
```

### MSW (Mock Service Worker) Setup

For testing components that fetch external APIs:

```typescript
// tests/setup/mocks/handlers.ts
import { http, HttpResponse } from 'msw';

export const handlers = [
  http.get('https://api.example.com/users', () => {
    return HttpResponse.json([
      { id: '1', name: 'User 1' },
      { id: '2', name: 'User 2' },
    ]);
  }),

  http.post('https://api.example.com/users', async ({ request }) => {
    const body = await request.json();
    return HttpResponse.json({ id: '3', ...body }, { status: 201 });
  }),
];
```

```typescript
// tests/setup/mocks/server.ts
import { setupServer } from 'msw/node';
import { handlers } from './handlers';

export const server = setupServer(...handlers);
```

```typescript
// In jest-setup.ts, add:
import { server } from './mocks/server';

beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

## End-to-End Testing

### Playwright Tests

```typescript
// tests/e2e/auth.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Authentication', () => {
  test('user can sign up with valid credentials', async ({ page }) => {
    await page.goto('/signup');

    await page.getByLabel('Name').fill('Jane Doe');
    await page.getByLabel('Email').fill('jane@example.com');
    await page.getByLabel('Password').fill('securePassword123!');
    await page.getByRole('button', { name: 'Sign Up' }).click();

    await expect(page).toHaveURL('/dashboard');
    await expect(page.getByText('Welcome, Jane')).toBeVisible();
  });

  test('shows validation errors for invalid input', async ({ page }) => {
    await page.goto('/signup');

    await page.getByRole('button', { name: 'Sign Up' }).click();

    await expect(page.getByText('Name is required')).toBeVisible();
    await expect(page.getByText('Email is required')).toBeVisible();
  });

  test('user can log in and access protected routes', async ({ page }) => {
    await page.goto('/login');

    await page.getByLabel('Email').fill('jane@example.com');
    await page.getByLabel('Password').fill('securePassword123!');
    await page.getByRole('button', { name: 'Log In' }).click();

    await expect(page).toHaveURL('/dashboard');
  });

  test('redirects unauthenticated users to login', async ({ page }) => {
    await page.goto('/dashboard');

    await expect(page).toHaveURL('/login');
  });

  test('user can log out', async ({ page }) => {
    // Assume logged in via fixture or setup
    await page.goto('/dashboard');
    await page.getByRole('button', { name: 'Log Out' }).click();

    await expect(page).toHaveURL('/');
  });
});
```

### Page Object Pattern

```typescript
// tests/e2e/fixtures/login-page.ts
import { Page, Locator } from '@playwright/test';

export class LoginPage {
  readonly page: Page;
  readonly emailInput: Locator;
  readonly passwordInput: Locator;
  readonly submitButton: Locator;
  readonly errorMessage: Locator;

  constructor(page: Page) {
    this.page = page;
    this.emailInput = page.getByLabel('Email');
    this.passwordInput = page.getByLabel('Password');
    this.submitButton = page.getByRole('button', { name: 'Log In' });
    this.errorMessage = page.getByRole('alert');
  }

  async goto() {
    await this.page.goto('/login');
  }

  async login(email: string, password: string) {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.submitButton.click();
  }
}
```

### Auth Fixtures for E2E

```typescript
// tests/e2e/fixtures/auth.ts
import { test as base, Page } from '@playwright/test';

type AuthFixtures = {
  authenticatedPage: Page;
};

export const test = base.extend<AuthFixtures>({
  authenticatedPage: async ({ page }, use) => {
    // Login before test
    await page.goto('/login');
    await page.getByLabel('Email').fill('test@example.com');
    await page.getByLabel('Password').fill('password123');
    await page.getByRole('button', { name: 'Log In' }).click();
    await page.waitForURL('/dashboard');

    await use(page);
  },
});
```

## Testing Patterns

### Testing Loading States

```typescript
import { render, screen } from '@testing-library/react';
import { Suspense } from 'react';

it('shows loading state while data is being fetched', () => {
  render(
    <Suspense fallback={<div>Loading...</div>}>
      <AsyncComponent />
    </Suspense>
  );

  expect(screen.getByText('Loading...')).toBeInTheDocument();
});
```

### Testing Error Boundaries

```typescript
import { render, screen } from '@testing-library/react';
import { ErrorBoundary } from './error-boundary';

it('renders error UI when child throws', () => {
  const ThrowingComponent = () => {
    throw new Error('Test error');
  };

  // Suppress console.error for this test
  const spy = jest.spyOn(console, 'error').mockImplementation(() => {});

  render(
    <ErrorBoundary fallback={<div>Something went wrong</div>}>
      <ThrowingComponent />
    </ErrorBoundary>
  );

  expect(screen.getByText('Something went wrong')).toBeInTheDocument();
  spy.mockRestore();
});
```

### Testing with React Context

```typescript
import { render, screen } from '@testing-library/react';
import { ThemeProvider } from '@/components/theme-provider';
import { ThemedButton } from './themed-button';

function renderWithProviders(ui: React.ReactElement) {
  return render(
    <ThemeProvider defaultTheme="light">
      {ui}
    </ThemeProvider>
  );
}

it('applies theme styles correctly', () => {
  renderWithProviders(<ThemedButton>Click me</ThemedButton>);

  const button = screen.getByRole('button');
  expect(button).toHaveClass('bg-white');
});
```

## NPM Scripts

```json
{
  "scripts": {
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage",
    "test:ci": "jest --ci --coverage",
    "test:e2e": "playwright test",
    "test:e2e:ui": "playwright test --ui",
    "test:e2e:headed": "playwright test --headed",
    "test:all": "jest --coverage && playwright test"
  }
}
```

## Running Tests

### Local Development

```bash
# Run unit and integration tests
npm test

# Watch mode for development
npm run test:watch

# Run with coverage report
npm run test:coverage

# Run E2E tests
npm run test:e2e

# Run E2E tests with browser visible
npm run test:e2e:headed

# Run all tests
npm run test:all
```

### CI Pipeline (GitHub Actions)

```yaml
name: CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Type check
        run: npx tsc --noEmit

      - name: Lint
        run: npm run lint

      - name: Unit and integration tests
        run: npm run test:ci

      - name: Install Playwright browsers
        run: npx playwright install --with-deps

      - name: E2E tests
        run: npm run test:e2e

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: |
            coverage/
            playwright-report/
```

## Best Practices

### 1. Test Isolation
- Each test should be independent and not rely on other tests
- Clean up state between tests
- Use `beforeEach` for setup, not shared mutable state

### 2. Test Naming
- Use descriptive names that explain the expected behavior
- Follow the pattern: "should [expected behavior] when [condition]"
- Group related tests with `describe` blocks

### 3. Avoid Implementation Details
- Test behavior, not implementation
- Query elements by role, label, or text — not by class or test ID
- Avoid testing internal state directly

### 4. Keep Tests Fast
- Mock external dependencies in unit tests
- Use MSW for API mocking instead of real network calls
- Reserve E2E tests for critical user journeys only

### 5. Coverage Goals
- Unit tests: 80%+ coverage
- Integration tests: Cover all Server Actions and Route Handlers
- E2E tests: Cover critical user paths (auth, core features, payments)

### 6. Accessibility Testing
- Use `@testing-library/react` queries that encourage accessible markup
- Prefer `getByRole`, `getByLabelText`, `getByText` over `getByTestId`
- Include basic a11y checks with `jest-axe` in component tests

```typescript
import { axe, toHaveNoViolations } from 'jest-axe';

expect.extend(toHaveNoViolations);

it('has no accessibility violations', async () => {
  const { container } = render(<UserCard user={mockUser} />);
  const results = await axe(container);
  expect(results).toHaveNoViolations();
});
```

### 7. Snapshot Testing
- Use sparingly — only for stable UI components
- Prefer explicit assertions over snapshots
- Review snapshot changes carefully in PRs

## Mocking Patterns

### Mocking Next.js Modules

```typescript
// Mock next/image
jest.mock('next/image', () => ({
  __esModule: true,
  default: (props: any) => <img {...props} />,
}));

// Mock next/link
jest.mock('next/link', () => ({
  __esModule: true,
  default: ({ children, href }: any) => <a href={href}>{children}</a>,
}));

// Mock next/font
jest.mock('next/font/google', () => ({
  Inter: () => ({ className: 'inter' }),
}));
```

### Mocking Environment Variables

```typescript
describe('feature with env dependency', () => {
  const originalEnv = process.env;

  beforeEach(() => {
    process.env = { ...originalEnv, FEATURE_FLAG: 'true' };
  });

  afterEach(() => {
    process.env = originalEnv;
  });

  it('enables feature when flag is set', () => {
    // test implementation
  });
});
```

## Debugging Tests

### Jest
- Use `test.only()` to run a single test
- Use `--verbose` flag for detailed output
- Use `--detectOpenHandles` to find hanging async operations
- Use `console.log` or debugger with `node --inspect`

### Playwright
- Use `--headed` flag to see the browser
- Use `--debug` flag for step-by-step execution
- Use `page.pause()` to pause execution and inspect
- Check `playwright-report/` for failure screenshots and traces
