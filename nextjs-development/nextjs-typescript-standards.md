---
inclusion: always
---

# Next.js TypeScript Development Standards

This document defines the coding standards, architecture patterns, and best practices for Next.js application development using the App Router.

## Project Overview

- **Runtime**: Node.js >= 24.0.0
- **Language**: TypeScript 5.3+
- **Framework**: Next.js 14+ (App Router)
- **Styling**: Tailwind CSS
- **State Management**: React Server Components + Client state (React Context / Zustand)
- **Validation**: Zod schemas
- **Testing**: Jest + React Testing Library + Playwright
- **Linting**: ESLint with next/core-web-vitals config
- **Package Manager**: npm or pnpm

## Architecture Overview

Next.js App Router uses a file-system based routing model with React Server Components (RSC) as the default rendering strategy.

### Rendering Strategies

| Strategy | Use Case |
|----------|----------|
| Server Components (default) | Data fetching, static content, SEO-critical pages |
| Client Components (`'use client'`) | Interactivity, browser APIs, event handlers, state |
| Static Generation | Marketing pages, blog posts, documentation |
| Dynamic Rendering | Personalized content, real-time data |
| Streaming | Progressive loading, long data fetches |

## Project Structure

```
project-root/
├── src/
│   ├── app/                    # App Router pages and layouts
│   │   ├── (auth)/             # Route groups for auth pages
│   │   ├── (marketing)/        # Route groups for public pages
│   │   ├── dashboard/          # Dashboard routes
│   │   │   ├── page.tsx
│   │   │   ├── layout.tsx
│   │   │   ├── loading.tsx
│   │   │   ├── error.tsx
│   │   │   └── not-found.tsx
│   │   ├── api/                # Route Handlers (API routes)
│   │   │   └── [resource]/
│   │   │       └── route.ts
│   │   ├── layout.tsx          # Root layout
│   │   ├── page.tsx            # Home page
│   │   ├── not-found.tsx       # Global 404
│   │   └── error.tsx           # Global error boundary
│   ├── components/             # Shared components
│   │   ├── ui/                 # Primitive UI components
│   │   ├── forms/              # Form components
│   │   └── layouts/            # Layout components
│   ├── lib/                    # Utility functions and shared logic
│   │   ├── actions/            # Server Actions
│   │   ├── db/                 # Database client and queries
│   │   ├── auth/               # Authentication utilities
│   │   └── utils.ts            # General utilities
│   ├── hooks/                  # Custom React hooks
│   ├── types/                  # TypeScript type definitions
│   ├── schemas/                # Zod validation schemas
│   └── styles/                 # Global styles
├── public/                     # Static assets
├── tests/
│   ├── e2e/                    # Playwright end-to-end tests
│   └── integration/            # Integration tests
├── next.config.ts              # Next.js configuration
├── tailwind.config.ts          # Tailwind configuration
├── tsconfig.json               # TypeScript configuration
└── package.json
```

## File Naming Conventions

- Use kebab-case for files and directories: `user-profile.tsx`, `auth-utils.ts`
- Page files: `page.tsx` (required by App Router)
- Layout files: `layout.tsx`
- Loading UI: `loading.tsx`
- Error boundaries: `error.tsx`
- Not found pages: `not-found.tsx`
- Route handlers: `route.ts`
- Components: PascalCase export, kebab-case file: `user-card.tsx` exports `UserCard`
- Server Actions: `actions.ts` or grouped in `src/lib/actions/`

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
    "forceConsistentCasingInFileNames": true
  }
}
```

### Type Definitions

- Prefer interfaces for component props and object shapes
- Use type aliases for unions, intersections, and utility types
- Always define explicit return types for server actions and utility functions
- Avoid `any` — use `unknown` if the type is truly unknown
- Use `satisfies` operator for type-safe object literals

```typescript
// Props interface
interface UserCardProps {
  user: User;
  onSelect?: (userId: string) => void;
  variant?: 'compact' | 'detailed';
}

// Type alias for unions
type ApiResponse<T> = { data: T; error: null } | { data: null; error: string };

// satisfies for type-safe config
const routes = {
  home: '/',
  dashboard: '/dashboard',
  settings: '/settings',
} satisfies Record<string, string>;
```

### Import Organization

1. React and Next.js imports
2. Third-party libraries
3. Internal absolute imports (`@/`)
4. Relative imports
5. Type-only imports last

```typescript
import { Suspense } from 'react';
import Link from 'next/link';
import { z } from 'zod';
import { db } from '@/lib/db';
import { UserCard } from './user-card';
import type { User } from '@/types';
```

## Component Standards

### Server Components (Default)

Server Components are the default in the App Router. Use them for:
- Data fetching
- Accessing backend resources directly
- Keeping sensitive information on the server
- Reducing client-side JavaScript

```typescript
// app/users/page.tsx — Server Component (no directive needed)
import { db } from '@/lib/db';
import { UserList } from '@/components/user-list';

export default async function UsersPage() {
  const users = await db.user.findMany();

  return (
    <main>
      <h1>Users</h1>
      <UserList users={users} />
    </main>
  );
}
```

### Client Components

Add `'use client'` directive only when the component needs:
- Event handlers (onClick, onChange, etc.)
- State (useState, useReducer)
- Effects (useEffect)
- Browser-only APIs
- Custom hooks that use state or effects

```typescript
'use client';

import { useState } from 'react';

interface CounterProps {
  initialCount?: number;
}

export function Counter({ initialCount = 0 }: CounterProps) {
  const [count, setCount] = useState(initialCount);

  return (
    <button onClick={() => setCount(count + 1)}>
      Count: {count}
    </button>
  );
}
```

### Component Composition Pattern

Push `'use client'` boundaries as far down the tree as possible:

```typescript
// app/dashboard/page.tsx — Server Component
import { db } from '@/lib/db';
import { DashboardChart } from './dashboard-chart'; // Client
import { StatsSummary } from './stats-summary';     // Server

export default async function DashboardPage() {
  const stats = await db.stats.getLatest();

  return (
    <div>
      <StatsSummary stats={stats} />
      <DashboardChart data={stats.chartData} />
    </div>
  );
}
```

## Data Fetching

### Server Component Data Fetching

Fetch data directly in Server Components. Next.js extends `fetch` with caching and revalidation:

```typescript
// Cached by default (static)
const data = await fetch('https://api.example.com/data');

// Revalidate every 60 seconds
const data = await fetch('https://api.example.com/data', {
  next: { revalidate: 60 },
});

// No cache (dynamic)
const data = await fetch('https://api.example.com/data', {
  cache: 'no-store',
});
```

### Database Queries

For direct database access, use query functions with `unstable_cache` or `revalidateTag`:

```typescript
import { unstable_cache } from 'next/cache';
import { db } from '@/lib/db';

const getUser = unstable_cache(
  async (userId: string) => {
    return db.user.findUnique({ where: { id: userId } });
  },
  ['user'],
  { revalidate: 3600, tags: ['user'] }
);
```

### Parallel Data Fetching

Fetch data in parallel to avoid waterfalls:

```typescript
export default async function Page({ params }: { params: { id: string } }) {
  // Parallel fetching — both start immediately
  const [user, posts] = await Promise.all([
    getUser(params.id),
    getUserPosts(params.id),
  ]);

  return (
    <div>
      <UserProfile user={user} />
      <PostList posts={posts} />
    </div>
  );
}
```

## Server Actions

### Definition

Server Actions are async functions that run on the server. Define them with the `'use server'` directive:

```typescript
// src/lib/actions/user-actions.ts
'use server';

import { revalidatePath } from 'next/cache';
import { redirect } from 'next/navigation';
import { z } from 'zod';
import { db } from '@/lib/db';

const createUserSchema = z.object({
  name: z.string().min(2).max(100),
  email: z.string().email(),
});

export async function createUser(formData: FormData) {
  const rawData = {
    name: formData.get('name'),
    email: formData.get('email'),
  };

  const validated = createUserSchema.safeParse(rawData);

  if (!validated.success) {
    return { error: validated.error.flatten().fieldErrors };
  }

  await db.user.create({ data: validated.data });

  revalidatePath('/users');
  redirect('/users');
}
```

### Usage in Forms

```typescript
import { createUser } from '@/lib/actions/user-actions';

export default function CreateUserPage() {
  return (
    <form action={createUser}>
      <input name="name" type="text" required />
      <input name="email" type="email" required />
      <button type="submit">Create User</button>
    </form>
  );
}
```

### Server Action Best Practices

- Always validate inputs with Zod
- Return structured error objects for form validation
- Use `revalidatePath` or `revalidateTag` after mutations
- Keep actions focused — one action per mutation
- Handle errors gracefully with try/catch
- Never trust client-side data — always validate server-side

## Route Handlers (API Routes)

### Definition

Route Handlers replace the Pages Router `api/` directory:

```typescript
// app/api/users/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { z } from 'zod';
import { db } from '@/lib/db';

const createUserSchema = z.object({
  name: z.string().min(2),
  email: z.string().email(),
});

export async function GET(request: NextRequest) {
  const searchParams = request.nextUrl.searchParams;
  const page = parseInt(searchParams.get('page') ?? '1');
  const limit = parseInt(searchParams.get('limit') ?? '10');

  const users = await db.user.findMany({
    skip: (page - 1) * limit,
    take: limit,
  });

  return NextResponse.json({ data: users, page, limit });
}

export async function POST(request: NextRequest) {
  try {
    const body = await request.json();
    const validated = createUserSchema.parse(body);

    const user = await db.user.create({ data: validated });

    return NextResponse.json(user, { status: 201 });
  } catch (error) {
    if (error instanceof z.ZodError) {
      return NextResponse.json(
        { error: error.flatten().fieldErrors },
        { status: 400 }
      );
    }
    return NextResponse.json(
      { error: 'Internal server error' },
      { status: 500 }
    );
  }
}
```

### Dynamic Route Handlers

```typescript
// app/api/users/[id]/route.ts
import { NextRequest, NextResponse } from 'next/server';

export async function GET(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  const user = await db.user.findUnique({ where: { id: params.id } });

  if (!user) {
    return NextResponse.json({ error: 'Not found' }, { status: 404 });
  }

  return NextResponse.json(user);
}
```

## Error Handling

### Error Boundaries

Use `error.tsx` files for route-level error handling:

```typescript
'use client';

interface ErrorProps {
  error: Error & { digest?: string };
  reset: () => void;
}

export default function Error({ error, reset }: ErrorProps) {
  return (
    <div role="alert">
      <h2>Something went wrong</h2>
      <p>{error.message}</p>
      <button onClick={reset}>Try again</button>
    </div>
  );
}
```

### Not Found Handling

```typescript
// app/users/[id]/page.tsx
import { notFound } from 'next/navigation';

export default async function UserPage({ params }: { params: { id: string } }) {
  const user = await getUser(params.id);

  if (!user) {
    notFound();
  }

  return <UserProfile user={user} />;
}
```

### Global Error Handling

```typescript
// app/global-error.tsx
'use client';

export default function GlobalError({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  return (
    <html>
      <body>
        <h2>Something went wrong</h2>
        <button onClick={reset}>Try again</button>
      </body>
    </html>
  );
}
```

## Metadata and SEO

### Static Metadata

```typescript
import type { Metadata } from 'next';

export const metadata: Metadata = {
  title: 'Page Title',
  description: 'Page description for SEO',
  openGraph: {
    title: 'Page Title',
    description: 'Page description',
    images: ['/og-image.png'],
  },
};
```

### Dynamic Metadata

```typescript
import type { Metadata } from 'next';

interface PageProps {
  params: { id: string };
}

export async function generateMetadata({ params }: PageProps): Promise<Metadata> {
  const product = await getProduct(params.id);

  return {
    title: product.name,
    description: product.description,
  };
}
```

## Styling Standards

### Tailwind CSS

- Use Tailwind utility classes as the primary styling approach
- Extract repeated patterns into components, not custom CSS
- Use `cn()` utility (clsx + tailwind-merge) for conditional classes
- Follow mobile-first responsive design

```typescript
import { cn } from '@/lib/utils';

interface ButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: 'primary' | 'secondary' | 'destructive';
  size?: 'sm' | 'md' | 'lg';
}

export function Button({ variant = 'primary', size = 'md', className, ...props }: ButtonProps) {
  return (
    <button
      className={cn(
        'inline-flex items-center justify-center rounded-md font-medium transition-colors',
        'focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-offset-2',
        {
          'bg-blue-600 text-white hover:bg-blue-700': variant === 'primary',
          'bg-gray-200 text-gray-900 hover:bg-gray-300': variant === 'secondary',
          'bg-red-600 text-white hover:bg-red-700': variant === 'destructive',
        },
        {
          'h-8 px-3 text-sm': size === 'sm',
          'h-10 px-4 text-base': size === 'md',
          'h-12 px-6 text-lg': size === 'lg',
        },
        className
      )}
      {...props}
    />
  );
}
```

### CSS Modules (Alternative)

When Tailwind is insufficient, use CSS Modules:

```typescript
import styles from './component.module.css';

export function Component() {
  return <div className={styles.container}>Content</div>;
}
```

## Performance Optimization

### Image Optimization

Always use `next/image` for images:

```typescript
import Image from 'next/image';

export function Avatar({ src, alt }: { src: string; alt: string }) {
  return (
    <Image
      src={src}
      alt={alt}
      width={48}
      height={48}
      className="rounded-full"
    />
  );
}
```

### Font Optimization

Use `next/font` for optimized font loading:

```typescript
import { Inter } from 'next/font/google';

const inter = Inter({ subsets: ['latin'] });

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" className={inter.className}>
      <body>{children}</body>
    </html>
  );
}
```

### Dynamic Imports

Use `next/dynamic` for code splitting client components:

```typescript
import dynamic from 'next/dynamic';

const HeavyChart = dynamic(() => import('./heavy-chart'), {
  loading: () => <ChartSkeleton />,
  ssr: false, // Only if component uses browser APIs
});
```

### Loading UI and Streaming

Use `loading.tsx` and `<Suspense>` for progressive loading:

```typescript
// app/dashboard/loading.tsx
export default function Loading() {
  return <DashboardSkeleton />;
}

// Or inline Suspense boundaries
import { Suspense } from 'react';

export default function Page() {
  return (
    <div>
      <h1>Dashboard</h1>
      <Suspense fallback={<ChartSkeleton />}>
        <AsyncChart />
      </Suspense>
      <Suspense fallback={<TableSkeleton />}>
        <AsyncTable />
      </Suspense>
    </div>
  );
}
```

## Authentication

### Middleware-Based Auth

```typescript
// middleware.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  const token = request.cookies.get('session-token');

  if (!token && request.nextUrl.pathname.startsWith('/dashboard')) {
    return NextResponse.redirect(new URL('/login', request.url));
  }

  return NextResponse.next();
}

export const config = {
  matcher: ['/dashboard/:path*', '/settings/:path*'],
};
```

### Auth Utilities

```typescript
// src/lib/auth/session.ts
import { cookies } from 'next/headers';
import { redirect } from 'next/navigation';

export async function getSession() {
  const cookieStore = cookies();
  const token = cookieStore.get('session-token')?.value;

  if (!token) return null;

  return verifyToken(token);
}

export async function requireAuth() {
  const session = await getSession();

  if (!session) {
    redirect('/login');
  }

  return session;
}
```

## Environment Variables

### Naming Convention

- `NEXT_PUBLIC_*` — Exposed to the browser (public)
- All others — Server-only (private)

```env
# .env.local
DATABASE_URL=postgresql://...
AUTH_SECRET=your-secret-key
NEXT_PUBLIC_APP_URL=http://localhost:3000
NEXT_PUBLIC_API_URL=http://localhost:3000/api
```

### Type-Safe Environment Variables

```typescript
// src/lib/env.ts
import { z } from 'zod';

const envSchema = z.object({
  DATABASE_URL: z.string().url(),
  AUTH_SECRET: z.string().min(32),
  NEXT_PUBLIC_APP_URL: z.string().url(),
});

export const env = envSchema.parse(process.env);
```

## Caching and Revalidation

### Cache Strategies

```typescript
// Static (cached indefinitely until redeployed)
export const revalidate = false;

// Time-based revalidation (ISR)
export const revalidate = 60; // seconds

// Dynamic (no caching)
export const dynamic = 'force-dynamic';
```

### On-Demand Revalidation

```typescript
import { revalidatePath, revalidateTag } from 'next/cache';

// Revalidate a specific path
revalidatePath('/blog');

// Revalidate by tag
revalidateTag('posts');
```

## Accessibility Standards

- Use semantic HTML elements (`<main>`, `<nav>`, `<article>`, `<section>`)
- Include proper ARIA attributes where semantic HTML is insufficient
- Ensure keyboard navigation works for all interactive elements
- Provide alt text for all images
- Maintain color contrast ratios (WCAG AA minimum)
- Use `role="alert"` for error messages
- Test with screen readers during development

## Security Best Practices

- Never expose server-only secrets via `NEXT_PUBLIC_*` prefix
- Validate all inputs in Server Actions and Route Handlers with Zod
- Use CSRF protection (built into Server Actions)
- Sanitize user-generated content before rendering
- Set appropriate security headers in `next.config.ts`
- Use `HttpOnly` cookies for session tokens
- Implement rate limiting on API routes

### Security Headers

```typescript
// next.config.ts
const nextConfig = {
  headers: async () => [
    {
      source: '/(.*)',
      headers: [
        { key: 'X-Frame-Options', value: 'DENY' },
        { key: 'X-Content-Type-Options', value: 'nosniff' },
        { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
        { key: 'Permissions-Policy', value: 'camera=(), microphone=()' },
      ],
    },
  ],
};

export default nextConfig;
```

## Build and Development

### Scripts

```json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint",
    "type-check": "tsc --noEmit",
    "test": "jest",
    "test:watch": "jest --watch",
    "test:e2e": "playwright test"
  }
}
```

### Next.js Configuration

```typescript
// next.config.ts
import type { NextConfig } from 'next';

const nextConfig: NextConfig = {
  experimental: {
    typedRoutes: true,
  },
  images: {
    remotePatterns: [
      {
        protocol: 'https',
        hostname: 'images.example.com',
      },
    ],
  },
};

export default nextConfig;
```

## Git Workflow

### Commits

- Use conventional commits format
- Format: `type(scope): message`
- Types: feat, fix, docs, style, refactor, test, chore

### Branch Strategy

- `main` — Production-ready code
- `develop` — Integration branch
- `feature/*` — Feature branches
- `fix/*` — Bug fix branches

## Logging

- Use structured logging (e.g., pino or winston) for server-side code
- Never log sensitive data (tokens, passwords, PII)
- Include request context (request ID, user ID) in logs
- Use appropriate log levels: error, warn, info, debug

## Documentation

- Add JSDoc comments for exported utilities and complex functions
- Document component props with TypeScript interfaces
- Keep README.md updated with setup instructions
- Document environment variables in `.env.example`
- Include architecture decision records (ADRs) for significant choices
