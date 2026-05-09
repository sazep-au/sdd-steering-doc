---
inclusion: always
---

# Integration Testing Standards

This document defines the integration testing strategy for the chimbuk-service-site API, including local development and CI pipeline execution.

## Overview

Integration tests validate the interaction between multiple layers of the application (Routes → Services → DALs → Database) without mocking the database layer. This ensures that:
- Database queries work correctly
- Data transformations are accurate
- API endpoints return expected responses
- Error handling works end-to-end

## Testing Strategy

### Unit Tests vs Integration Tests

**Unit Tests** (`*.spec.ts`):
- Test individual functions/classes in isolation
- Mock external dependencies (AWS SDK, database)
- Fast execution (milliseconds)
- High coverage of edge cases
- Located: Co-located with source files

**Integration Tests** (`*.integration.test.ts`):
- Test multiple layers together
- Use real database (DynamoDB Local)
- Slower execution (seconds)
- Validate real-world scenarios
- Located: `test/integration/`

### Property-Based Testing Policy

**DO NOT** implement property-based tests unless explicitly requested by the user:
- Property-based testing adds unnecessary complexity
- Example-based tests with specific inputs/outputs are preferred
- Focus on concrete test cases that are easy to understand and debug
- If specs include optional property-based tests (marked with `*`), skip them
- Only implement property-based tests when the user specifically asks for them

## DynamoDB Local Setup

### Why DynamoDB Local?

- **Official AWS tool** - Behaves identically to production DynamoDB
- **No mocking** - Tests use real DynamoDB operations
- **Fast** - Runs in-memory for quick test execution
- **CI-friendly** - Easy to run in GitHub Actions, GitLab CI, etc.
- **Isolated** - Each test suite can use a fresh database

### Installation

```bash
# Install DynamoDB Local via npm
npm install --save-dev @shelf/jest-dynamodb

# Or use Docker
docker pull amazon/dynamodb-local
```

## Project Structure

```
project-root/
├── src/
│   ├── routes/
│   ├── services/
│   ├── dals/
│   └── *.spec.ts              # Unit tests (co-located)
├── test/
│   ├── integration/
│   │   ├── setup/
│   │   │   ├── dynamodb-setup.ts      # DynamoDB Local configuration
│   │   │   ├── table-schemas.ts       # Table definitions
│   │   │   └── test-data.ts           # Seed data helpers
│   │   ├── routes/
│   │   │   ├── profile-routes.integration.test.ts
│   │   │   ├── site-routes.integration.test.ts
│   │   │   └── page-routes.integration.test.ts
│   │   ├── services/
│   │   │   └── *.integration.test.ts
│   │   └── dals/
│   │       └── *.integration.test.ts
│   └── fixtures.ts
├── jest.config.js              # Unit test config
├── jest.integration.config.js  # Integration test config
└── package.json
```

## Configuration

### jest.integration.config.js

```javascript
module.exports = {
  preset: '@shelf/jest-dynamodb',
  testEnvironment: 'node',
  testMatch: ['**/*.integration.test.ts'],
  transform: {
    '^.+\\.ts$': ['ts-jest', {
      tsconfig: 'tsconfig.dev.json'
    }]
  },
  setupFilesAfterEnv: ['<rootDir>/test/integration/setup/jest-setup.ts'],
  testTimeout: 30000,
  maxConcurrency: 1, // Run integration tests sequentially
  collectCoverageFrom: [
    'src/**/*.ts',
    '!src/**/*.spec.ts',
    '!src/**/*.integration.test.ts'
  ]
};
```

### jest-dynamodb-config.js

```javascript
module.exports = {
  tables: [
    {
      TableName: 'Sites',
      KeySchema: [
        { AttributeName: 'PK', KeyType: 'HASH' },
        { AttributeName: 'SK', KeyType: 'RANGE' }
      ],
      AttributeDefinitions: [
        { AttributeName: 'PK', AttributeType: 'S' },
        { AttributeName: 'SK', AttributeType: 'S' }
      ],
      BillingMode: 'PAY_PER_REQUEST'
    }
    // Add more tables as needed
  ],
  port: 8000 // DynamoDB Local port
};
```

## Integration Test Structure

### Example: DAL Integration Test

```typescript
// test/integration/dals/profile-dal.integration.test.ts
import { ProfileDAL } from '../../../src/dals/profile-dal';
import { dbClient } from '../../../src/clients/dbClient';
import { seedTestData, clearTable } from '../setup/test-data';

describe('Profile Data Access Layer', () => {
  let profileDAL: ProfileDAL;

  beforeAll(async () => {
    // Configure DynamoDB client to use local endpoint
    process.env.DYNAMODB_ENDPOINT = 'http://localhost:8000';
    profileDAL = new ProfileDAL();
  });

  beforeEach(async () => {
    // Clear table before each test
    await clearTable('Sites');
  });

  afterAll(async () => {
    // Cleanup
    await clearTable('Sites');
  });

  describe('Retrieving an existing profile', () => {
    it('should return the profile with all details when profile exists in database', async () => {
      const testProfile = {
        PK: 'SITE#test-site',
        SK: 'PROFILE#test-profile',
        name: 'Test Profile',
        email: 'test@example.com'
      };
      await seedTestData('Sites', [testProfile]);

      const result = await profileDAL.getProfile('test-site', 'test-profile');

      expect(result).toBeDefined();
      expect(result.name).toBe('Test Profile');
      expect(result.email).toBe('test@example.com');
    });
  });

  describe('Retrieving a non-existent profile', () => {
    it('should return null when the requested profile does not exist', async () => {
      const result = await profileDAL.getProfile('non-existent', 'profile');

      expect(result).toBeNull();
    });
  });

  describe('Creating a new profile', () => {
    it('should successfully create and persist the profile in database', async () => {
      const newProfile = {
        siteId: 'test-site',
        profileId: 'new-profile',
        name: 'New Profile',
        email: 'new@example.com'
      };

      await profileDAL.createProfile(newProfile);

      const result = await profileDAL.getProfile('test-site', 'new-profile');
      expect(result).toBeDefined();
      expect(result.name).toBe('New Profile');
    });
  });

  describe('Creating a duplicate profile', () => {
    it('should throw an error when attempting to create a profile that already exists', async () => {
      const profile = {
        siteId: 'test-site',
        profileId: 'duplicate',
        name: 'Duplicate Profile'
      };
      await profileDAL.createProfile(profile);

      await expect(profileDAL.createProfile(profile))
        .rejects
        .toThrow('Profile already exists');
    });
  });
});
```

### Example: Route Integration Test

```typescript
// test/integration/routes/profile-routes.integration.test.ts
import request from 'supertest';
import { app } from '../../../src/app';
import { seedTestData, clearTable } from '../setup/test-data';

describe('Profile API Endpoints', () => {
  beforeAll(async () => {
    process.env.DYNAMODB_ENDPOINT = 'http://localhost:8000';
  });

  beforeEach(async () => {
    await clearTable('Sites');
  });

  describe('Retrieving an existing profile via GET endpoint', () => {
    it('should return 200 status and profile data when profile exists', async () => {
      await seedTestData('Sites', [{
        PK: 'SITE#test-site',
        SK: 'PROFILE#test-profile',
        name: 'Test Profile',
        email: 'test@example.com'
      }]);

      const response = await request(app)
        .get('/api/profiles/test-site/test-profile')
        .expect(200);

      expect(response.body).toMatchObject({
        name: 'Test Profile',
        email: 'test@example.com'
      });
    });
  });

  describe('Retrieving a non-existent profile via GET endpoint', () => {
    it('should return 404 status when the requested profile does not exist', async () => {
      await request(app)
        .get('/api/profiles/non-existent/profile')
        .expect(404);
    });
  });

  describe('Creating a new profile via POST endpoint', () => {
    it('should create the profile and return 201 status with profile data', async () => {
      const newProfile = {
        siteId: 'test-site',
        profileId: 'new-profile',
        name: 'New Profile',
        email: 'new@example.com'
      };

      const response = await request(app)
        .post('/api/profiles')
        .send(newProfile)
        .expect(201);

      expect(response.body).toMatchObject({
        name: 'New Profile',
        email: 'new@example.com'
      });

      const getResponse = await request(app)
        .get('/api/profiles/test-site/new-profile')
        .expect(200);
      
      expect(getResponse.body.name).toBe('New Profile');
    });
  });

  describe('Creating a profile with invalid data', () => {
    it('should return 400 status when request body is invalid', async () => {
      const invalidProfile = {
        invalid: 'data'
      };

      await request(app)
        .post('/api/profiles')
        .send(invalidProfile)
        .expect(400);
    });
  });
});
```

## Test Helpers

### test/integration/setup/test-data.ts

```typescript
import { DynamoDBDocumentClient, PutCommand, ScanCommand, DeleteCommand } from '@aws-sdk/lib-dynamodb';
import { dbClient } from '../../../src/clients/dbClient';

export async function seedTestData(tableName: string, items: any[]): Promise<void> {
  for (const item of items) {
    await dbClient.send(new PutCommand({
      TableName: tableName,
      Item: item
    }));
  }
}

export async function clearTable(tableName: string): Promise<void> {
  const scanResult = await dbClient.send(new ScanCommand({
    TableName: tableName
  }));

  if (scanResult.Items && scanResult.Items.length > 0) {
    for (const item of scanResult.Items) {
      await dbClient.send(new DeleteCommand({
        TableName: tableName,
        Key: {
          PK: item.PK,
          SK: item.SK
        }
      }));
    }
  }
}

export async function getItemCount(tableName: string): Promise<number> {
  const result = await dbClient.send(new ScanCommand({
    TableName: tableName,
    Select: 'COUNT'
  }));
  return result.Count || 0;
}
```

### test/integration/setup/dynamodb-setup.ts

```typescript
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient } from '@aws-sdk/lib-dynamodb';

export function createTestDynamoDBClient(): DynamoDBDocumentClient {
  const client = new DynamoDBClient({
    endpoint: process.env.DYNAMODB_ENDPOINT || 'http://localhost:8000',
    region: 'local',
    credentials: {
      accessKeyId: 'fake',
      secretAccessKey: 'fake'
    }
  });

  return DynamoDBDocumentClient.from(client);
}
```

## NPM Scripts

Add to `package.json`:

```json
{
  "scripts": {
    "test": "jest --config jest.config.js",
    "test:integration": "jest --config jest.integration.config.js",
    "test:integration:watch": "jest --config jest.integration.config.js --watch",
    "test:all": "npm run test && npm run test:integration",
    "dynamodb:start": "docker run -p 8000:8000 amazon/dynamodb-local",
    "dynamodb:install": "npm install --save-dev @shelf/jest-dynamodb"
  }
}
```

## Running Tests

### Local Development

```bash
# Run unit tests only
npm test

# Run integration tests (DynamoDB Local starts automatically)
npm run test:integration

# Run all tests
npm run test:all

# Watch mode for integration tests
npm run test:integration:watch
```

### CI Pipeline (GitHub Actions)

```yaml
name: CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run unit tests
        run: npm test
      
      - name: Run integration tests
        run: npm run test:integration
        # DynamoDB Local starts automatically via @shelf/jest-dynamodb
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
```

## Best Practices

### 1. Test Isolation
- Clear tables between tests using `beforeEach`
- Use unique IDs for test data to avoid conflicts
- Don't rely on test execution order

### 2. Test Data Management
- Create reusable seed data functions
- Use factories or builders for complex test objects
- Keep test data minimal and focused

### 3. Performance
- Run integration tests sequentially (`maxConcurrency: 1`)
- Use in-memory DynamoDB Local for speed
- Only test critical paths in integration tests

### 4. Coverage
- Unit tests should cover edge cases and error handling
- Integration tests should cover happy paths and common error scenarios
- Aim for 70%+ coverage on integration tests

### 5. Debugging
- Use descriptive test names
- Log DynamoDB operations in test helpers when debugging
- Use `test.only()` to run specific tests during development

## Alternative: Testcontainers (Advanced)

For teams wanting Docker-based isolation:

```typescript
import { GenericContainer } from 'testcontainers';

let dynamoContainer: StartedTestContainer;

beforeAll(async () => {
  dynamoContainer = await new GenericContainer('amazon/dynamodb-local')
    .withExposedPorts(8000)
    .start();
  
  const endpoint = `http://${dynamoContainer.getHost()}:${dynamoContainer.getMappedPort(8000)}`;
  process.env.DYNAMODB_ENDPOINT = endpoint;
});

afterAll(async () => {
  await dynamoContainer.stop();
});
```

## S3 Integration Testing

For S3 operations, use **LocalStack** or **S3Mock**:

```bash
# Using LocalStack
docker run -p 4566:4566 localstack/localstack

# Or use s3-mock npm package
npm install --save-dev @shelf/jest-s3
```

## Troubleshooting

### DynamoDB Local won't start
- Check port 8000 is not in use: `lsof -i :8000`
- Ensure Java is installed (required for DynamoDB Local JAR)
- Try Docker version instead

### Tests are slow
- Use in-memory mode (default with @shelf/jest-dynamodb)
- Reduce test data size
- Run integration tests in parallel with separate ports

### CI failures
- Ensure DynamoDB Local has enough memory
- Check for port conflicts
- Verify table schemas match production
