---
inclusion: always
---

# TypeScript API Development Standards

This document defines the coding standards, architecture patterns, and best practices for the chimbuk-service-site TypeScript API project.

## Project Overview

- **Runtime**: Node.js >= 22.0.0
- **Language**: TypeScript 5.3+
- **Framework**: Express.js with AWS Lambda support (@codegenie/serverless-express)
- **Testing**: Jest with ts-jest
- **Validation**: Zod schemas
- **AWS Services**: DynamoDB, S3, SSM Parameter Store
- **Build Tool**: Projen for project configuration

## Architecture Layers

The project follows a layered architecture pattern:

1. **Routes** (`src/routes/`) - HTTP endpoint definitions and request handling
2. **Services** (`src/services/`) - Business logic layer
3. **DALs** (`src/dals/`) - Data Access Layer for database and external service interactions
4. **Clients** (`src/clients/`) - AWS SDK clients (DynamoDB, S3, SSM)
5. **Models** (`src/models/`) - TypeScript type definitions
6. **Schemas** (`src/schemas/`) - Zod validation schemas
7. **Middlewares** (`src/middlewares/`) - Express middleware functions

## Code Organization Standards

### File Naming
- Use kebab-case for file names: `user-service.ts`, `profile-dal.ts`
- Test files: `*.spec.ts` co-located with source files
- Index files: `index.ts` for barrel exports

### Module Structure
Each feature should have corresponding files across layers:
```
src/
├── routes/feature-routes.ts
├── services/feature-service.ts
├── dals/feature-dal.ts
├── models/feature-models/
└── schemas/feature-schemas/
```

### Import Organization
1. External dependencies (node_modules)
2. AWS SDK imports
3. Internal absolute imports (from src/)
4. Relative imports
5. Type-only imports last

## TypeScript Standards

### Strict Mode
All strict TypeScript compiler options are enabled:
- `strict: true`
- `noImplicitAny: true`
- `strictNullChecks: true`
- `noUnusedLocals: true`
- `noUnusedParameters: true`

### Type Definitions
- Prefer interfaces for object shapes that may be extended
- Use type aliases for unions, intersections, and utility types
- Always define explicit return types for functions
- Avoid `any` - use `unknown` if type is truly unknown

### Async/Await
- Always use async/await over raw Promises
- Handle errors with try/catch blocks
- Avoid mixing callbacks with async/await

## Testing Standards

### Test Framework
- **Framework**: Jest with ts-jest
- **Coverage Requirements**:
  - Branches: 80%
  - Statements: 80%
  - Lines: 80%
  - Functions: 80%

### Property-Based Testing Policy
- **DO NOT** implement property-based tests unless explicitly requested by the user
- Focus on example-based tests with specific inputs and expected outputs
- Property-based testing adds complexity and is not part of the standard testing approach
- If property-based tests are marked as optional in specs (with `*`), skip them by default

### Test File Conventions
- Co-locate tests with source: `feature.ts` → `feature.spec.ts`
- Use descriptive test names: `describe('FeatureService')` and `it('should handle edge case')`
- Group related tests with nested `describe` blocks

### Test Structure
```typescript
describe('ServiceName', () => {
  describe('methodName', () => {
    it('should handle success case', async () => {
      // Arrange
      // Act
      // Assert
    });

    it('should handle error case', async () => {
      // Arrange
      // Act
      // Assert
    });
  });
});
```

### Mocking
- Use `jest.mock()` for module mocking
- Use `aws-sdk-client-mock` for AWS SDK mocking
- Clear mocks between tests with `clearMocks: true` (configured globally)

## API Development Standards

### Request Validation
- Use Zod schemas for all request validation
- Define schemas in `src/schemas/`
- Apply validation middleware to routes

### Error Handling
- Use `http-errors` package for HTTP errors
- Return consistent error responses
- Log errors with appropriate context using `@sazep/logger`

### Response Format
- Return consistent JSON responses
- Include appropriate HTTP status codes
- Use TypeScript types for response shapes

### Route Definitions
```typescript
router.get('/resource/:id', 
  validate(requestSchema),
  async (req, res, next) => {
    try {
      const result = await service.method(req.params.id);
      res.json(result);
    } catch (error) {
      next(error);
    }
  }
);
```

## AWS Integration Standards

### DynamoDB
- Use `@aws-sdk/lib-dynamodb` for simplified operations
- Define table names in environment variables
- Use helper functions from `src/common/dbHelpers.ts`
- Always handle conditional check failures

### S3
- Use `@aws-sdk/client-s3` for S3 operations
- Store bucket names in environment variables
- Handle missing objects gracefully

### SSM Parameter Store
- Use `@aws-sdk/client-ssm` for parameter retrieval
- Cache parameters when appropriate
- Use hierarchical parameter naming

## Logging Standards

- Use `@sazep/sazep-logger` package for all logging
- Log levels: error, warn, info, debug
- Include context in log messages
- Never log sensitive data (passwords, tokens, PII)

## Build and Development

### Scripts
- `npm run build` - Compile TypeScript
- `npm test` - Run tests with coverage
- `npm run test:watch` - Run tests in watch mode
- `npm run eslint` - Run linter
- `npm start` - Start local development server

### Projen
- Project configuration is managed by Projen
- Edit `.projenrc.ts` for configuration changes
- Run `npx projen` to regenerate configuration files
- Do not manually edit generated files (marked with "Generated by projen")

## Git Workflow

### Commits
- Use conventional commits format (enforced by commitlint)
- Format: `type(scope): message`
- Types: feat, fix, docs, style, refactor, test, chore

### Pre-commit Hooks
- Husky runs pre-commit hooks
- Linting and formatting checks run automatically

## API Documentation Route

When the `ENABLE_DOCS` environment variable is set to `"true"`, mount a `/docs` route to host the API documentation (e.g., Swagger UI). This keeps documentation accessible in development and staging environments while disabled in production by default.

### Implementation Pattern
```typescript
import express from 'express';
import swaggerUi from 'swagger-ui-express';
import swaggerDocument from '../docs/openapi.json';

const app = express();

if (process.env.ENABLE_DOCS === 'true') {
  app.use('/docs', swaggerUi.serve, swaggerUi.setup(swaggerDocument));
}
```

### Guidelines
- Only mount the docs route when `ENABLE_DOCS` is explicitly `"true"`
- Use `swagger-ui-express` to serve the OpenAPI specification
- Load the OpenAPI spec from `docs/openapi.yaml` or `docs/openapi.json`
- Place the conditional mount logic in the main app setup (`src/app.ts` or equivalent)
- Do not bundle documentation dependencies in production if the route is disabled

## Environment Variables

Define environment variables for:
- AWS region and credentials
- DynamoDB table names
- S3 bucket names
- SSM parameter paths
- `ENABLE_DOCS` - Set to `"true"` to mount the `/docs` route for API documentation
- Log levels

## Security Best Practices

- Never commit secrets or credentials
- Use IAM roles for AWS access in Lambda
- Validate all user inputs with Zod
- Sanitize data before database operations
- Use parameterized queries/operations
- Keep dependencies updated

## Performance Considerations

- Minimize cold start time in Lambda functions
- Reuse AWS SDK clients across invocations
- Use connection pooling where applicable
- Implement caching for frequently accessed data
- Optimize DynamoDB queries (use indexes, avoid scans)

## Documentation

- Add JSDoc comments for public APIs
- Document complex business logic
- Keep README.md updated for developers and devops engineers
- Document environment variables
- Maintain OpenAPI spec in `docs/openapi.yaml`
