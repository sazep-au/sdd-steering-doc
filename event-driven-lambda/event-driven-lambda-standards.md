---
inclusion: always
---

# Event-Driven Lambda TypeScript Standards

This document defines the coding standards, architecture patterns, and best practices for TypeScript-based event-driven AWS Lambda functions that process events from S3, DynamoDB Streams, EventBridge, SQS, SNS, and Kinesis.

## Project Overview

- **Runtime**: Node.js 24.x (AWS Lambda, container image deployment)
- **Language**: TypeScript 5.3+
- **Framework**: AWS Lambda Powertools for TypeScript
- **Validation**: Zod schemas
- **AWS SDK**: AWS SDK v3 (modular clients)
- **Testing**: Jest with ts-jest
- **Packaging**: Docker container image (published to Amazon ECR)
- **Build Tool**: TypeScript compiler (tsc) inside multi-stage Dockerfile
- **Observability**: AWS Lambda Powertools (Tracer, Metrics) and `@sazep/sazep-logger` for structured logging

> **Infrastructure scope**: This repository owns application code and the container image. Lambda functions, event source mappings, IAM roles, DLQs, ECR repositories, and related AWS resources are provisioned from a separate infrastructure repository. This document focuses on what the application code must do; infrastructure requirements are described as contracts to coordinate with the infra repo rather than implementation code.

## Event Source Overview

| Event Source | Invocation Model | Batch | Retry Behavior | Use Case |
|--------------|------------------|-------|----------------|----------|
| S3 | Async (event notification) | No | 2 retries, then DLQ | Object processing, ETL |
| DynamoDB Streams | Poll-based | Yes | Until expiration, then DLQ | CDC, replication, audit |
| EventBridge | Async (rules) | No | 24h retries, then DLQ | Event routing, fan-out |
| SQS | Poll-based | Yes | Until visibility/expiration | Work queues, buffering |
| SNS | Async (subscription) | No | Retry policy, then DLQ | Pub/sub, fan-out |
| Kinesis | Poll-based | Yes | Until expiration | Streaming analytics |

Understanding the invocation model is critical — poll-based sources require partial batch response handling, while async sources need DLQ configuration.

## Architecture Principles

### Single Responsibility per Function

Each Lambda function should handle one event type for one purpose. Avoid multiplexing different event sources into a single function.

```
Good:  process-s3-upload.ts, process-order-stream.ts, handle-user-signup-event.ts
Bad:   event-handler.ts (handles S3 + DynamoDB + EventBridge)
```

### Layered Architecture

```
handler.ts           → Event parsing, routing, partial batch response
└── processor.ts     → Business logic orchestration (per-record)
    └── service.ts   → Domain operations
        └── dal.ts   → Data access (DynamoDB, S3, external APIs)
```

The handler never contains business logic. It owns event shape, validation, and the partial batch contract. Business logic lives in processors and services.

### Idempotency by Default

Event-driven Lambdas will be invoked more than once for the same event. Every handler must be idempotent:

- Use natural idempotency keys from the event (S3 versionId, DynamoDB sequenceNumber, message ID)
- Store idempotency tokens in DynamoDB with TTL (use Powertools Idempotency)
- Make database writes conditional where possible (ConditionExpression)
- Treat "already processed" as a success case, not an error

## Project Structure

```
project-root/
├── src/
│   ├── index.ts                           # Single Lambda entry point
│   ├── dispatcher.ts                      # Routes events to handlers
│   ├── handlers/                          # Per-event-source handler functions
│   │   ├── process-s3-upload/
│   │   │   ├── handler.ts                 # Exports processS3Upload
│   │   │   └── handler.test.ts
│   │   ├── process-order-stream/
│   │   │   ├── handler.ts                 # Exports processOrderStream
│   │   │   └── handler.test.ts
│   │   └── handle-user-event/
│   │       ├── handler.ts                 # Exports handleUserEvent
│   │       └── handler.test.ts
│   ├── processors/                        # Per-record business logic
│   │   ├── s3-upload-processor.ts
│   │   ├── order-stream-processor.ts
│   │   └── user-event-processor.ts
│   ├── services/                          # Domain services
│   │   ├── order-service.ts
│   │   └── notification-service.ts
│   ├── dals/                              # Data access layer
│   │   ├── order-dal.ts
│   │   └── user-dal.ts
│   ├── clients/                           # AWS SDK clients (singletons)
│   │   ├── dynamodb-client.ts
│   │   ├── s3-client.ts
│   │   ├── eventbridge-client.ts
│   │   └── sqs-client.ts
│   ├── schemas/                           # Zod event/payload schemas
│   │   ├── s3-event-schemas.ts
│   │   ├── order-schemas.ts
│   │   └── user-event-schemas.ts
│   ├── models/                            # Domain types
│   │   └── order.ts
│   ├── common/                            # Shared utilities
│   │   ├── logger.ts
│   │   ├── tracer.ts
│   │   ├── metrics.ts
│   │   ├── errors.ts
│   │   └── idempotency.ts
│   └── types/                             # Shared type definitions
│       └── events.ts
├── test/
│   ├── integration/
│   └── fixtures/
│       └── events/                        # Sample event payloads
├── Dockerfile                             # Multi-stage container image build
├── .dockerignore
├── package.json
├── tsconfig.json
└── jest.config.ts
```

## File Naming Conventions

- Use kebab-case for file names: `process-s3-upload.ts`, `order-dal.ts`
- Handler directory matches the logical function name in the infra repo: `handlers/process-s3-upload/`
- Test files: `*.test.ts` co-located with source
- Event fixtures: `test/fixtures/events/s3-put-event.json`

## TypeScript Standards

### Strict Mode

All strict TypeScript compiler options must be enabled:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "commonjs",
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noUncheckedIndexedAccess": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  }
}
```

### Event Type Definitions

Always type event objects using `aws-lambda` types:

```typescript
import type {
  S3Event,
  DynamoDBStreamEvent,
  EventBridgeEvent,
  SQSEvent,
  SNSEvent,
  KinesisStreamEvent,
  SQSBatchResponse,
  Context,
} from 'aws-lambda';
```

### Import Organization

1. Node built-ins
2. AWS SDK imports
3. Third-party libraries (aws-lambda types, Powertools, Zod)
4. Internal absolute imports (from `src/`)
5. Relative imports
6. Type-only imports last

## Handler Patterns

Handlers in this service are plain functions called by the dispatcher in `src/index.ts`, not Lambda entry points themselves. This means:

- Handlers do **not** wrap themselves with Middy — observability middleware is applied once in `src/index.ts`
- Handlers do **not** call `logger.addContext(context)` — the `injectLambdaContext` middleware handles that
- Handlers accept typed events and return the event-source-specific response shape (void, `SQSBatchResponse`, `DynamoDBBatchResponse`, etc.)
- Handlers are exported under descriptive named functions, not a generic `handler`

### S3 Event Handler

```typescript
// src/handlers/process-s3-upload/handler.ts
import { logger } from '@/common/logger';
import { metrics } from '@/common/metrics';
import { s3UploadProcessor } from '@/processors/s3-upload-processor';
import { MetricUnit } from '@aws-lambda-powertools/metrics';
import type { S3Event, Context } from 'aws-lambda';

export async function processS3Upload(event: S3Event, _context: Context): Promise<void> {
  logger.info('Received S3 event', { recordCount: event.Records.length });

  for (const record of event.Records) {
    try {
      await s3UploadProcessor.process(record);
      metrics.addMetric('S3ObjectProcessed', MetricUnit.Count, 1);
    } catch (error) {
      logger.error('Failed to process S3 record', {
        error,
        bucket: record.s3.bucket.name,
        key: record.s3.object.key,
      });
      metrics.addMetric('S3ProcessingError', MetricUnit.Count, 1);
      throw error; // Re-throw to trigger Lambda retry + DLQ
    }
  }
}
```

### DynamoDB Streams Handler with Partial Batch Response

```typescript
// src/handlers/process-order-stream/handler.ts
import { logger } from '@/common/logger';
import { metrics } from '@/common/metrics';
import { orderStreamProcessor } from '@/processors/order-stream-processor';
import { MetricUnit } from '@aws-lambda-powertools/metrics';
import type {
  DynamoDBStreamEvent,
  DynamoDBBatchResponse,
  Context,
} from 'aws-lambda';

export async function processOrderStream(
  event: DynamoDBStreamEvent,
  _context: Context
): Promise<DynamoDBBatchResponse> {
  const batchItemFailures: DynamoDBBatchResponse['batchItemFailures'] = [];

  for (const record of event.Records) {
    try {
      await orderStreamProcessor.process(record);
      metrics.addMetric('OrderStreamRecordProcessed', MetricUnit.Count, 1);
    } catch (error) {
      logger.error('Failed to process stream record', {
        error,
        sequenceNumber: record.dynamodb?.SequenceNumber,
      });
      metrics.addMetric('OrderStreamError', MetricUnit.Count, 1);

      if (record.dynamodb?.SequenceNumber) {
        batchItemFailures.push({
          itemIdentifier: record.dynamodb.SequenceNumber,
        });
      }
    }
  }

  return { batchItemFailures };
}
```

### EventBridge Handler

```typescript
// src/handlers/handle-user-event/handler.ts
import { logger } from '@/common/logger';
import { userEventSchema } from '@/schemas/user-event-schemas';
import { userEventProcessor } from '@/processors/user-event-processor';
import type { EventBridgeEvent, Context } from 'aws-lambda';

type UserEventDetail = {
  userId: string;
  action: 'created' | 'updated' | 'deleted';
  timestamp: string;
};

export async function handleUserEvent(
  event: EventBridgeEvent<'UserEvent', UserEventDetail>,
  _context: Context
): Promise<void> {
  logger.info('Received EventBridge event', {
    source: event.source,
    detailType: event['detail-type'],
    eventId: event.id,
  });

  const validated = userEventSchema.parse(event.detail);

  await userEventProcessor.process(validated, {
    eventId: event.id,
    eventTime: event.time,
  });
}
```

### SQS Handler with Partial Batch Response

```typescript
// src/handlers/process-sqs-messages/handler.ts
import { logger } from '@/common/logger';
import { messageProcessor } from '@/processors/message-processor';
import type { SQSEvent, SQSBatchResponse, Context } from 'aws-lambda';

export async function processSqsMessages(
  event: SQSEvent,
  _context: Context
): Promise<SQSBatchResponse> {
  const batchItemFailures: SQSBatchResponse['batchItemFailures'] = [];

  for (const record of event.Records) {
    try {
      await messageProcessor.process(record);
    } catch (error) {
      logger.error('Failed to process SQS message', {
        error,
        messageId: record.messageId,
      });
      batchItemFailures.push({ itemIdentifier: record.messageId });
    }
  }

  return { batchItemFailures };
}
```

Requires `ReportBatchItemFailures` enabled on the event source mapping.

### SNS Handler

```typescript
// src/handlers/handle-sns-notification/handler.ts
import { notificationProcessor } from '@/processors/notification-processor';
import type { SNSEvent, Context } from 'aws-lambda';

export async function handleSnsNotification(
  event: SNSEvent,
  _context: Context
): Promise<void> {
  for (const record of event.Records) {
    const message = JSON.parse(record.Sns.Message);
    await notificationProcessor.process(message, {
      messageId: record.Sns.MessageId,
      topicArn: record.Sns.TopicArn,
    });
  }
}
```

### Kinesis Handler with Partial Batch Response

```typescript
// src/handlers/process-kinesis-stream/handler.ts
import { logger } from '@/common/logger';
import { streamProcessor } from '@/processors/stream-processor';
import type { KinesisStreamEvent, KinesisStreamBatchResponse, Context } from 'aws-lambda';

export async function processKinesisStream(
  event: KinesisStreamEvent,
  _context: Context
): Promise<KinesisStreamBatchResponse> {
  const batchItemFailures: KinesisStreamBatchResponse['batchItemFailures'] = [];

  for (const record of event.Records) {
    try {
      const payload = Buffer.from(record.kinesis.data, 'base64').toString('utf-8');
      await streamProcessor.process(JSON.parse(payload));
    } catch (error) {
      logger.error('Failed to process Kinesis record', {
        error,
        sequenceNumber: record.kinesis.sequenceNumber,
      });
      batchItemFailures.push({ itemIdentifier: record.kinesis.sequenceNumber });
    }
  }

  return { batchItemFailures };
}
```

## Event Validation

### Zod Schemas for Event Payloads

Validate all event payloads, even from trusted AWS sources. Schemas catch schema drift early and produce clear errors.

```typescript
// src/schemas/user-event-schemas.ts
import { z } from 'zod';

export const userEventSchema = z.object({
  userId: z.string().uuid(),
  action: z.enum(['created', 'updated', 'deleted']),
  timestamp: z.string().datetime(),
  metadata: z.record(z.unknown()).optional(),
});

export type UserEvent = z.infer<typeof userEventSchema>;
```

### Parser Utility from Powertools

Lambda Powertools Parser integrates Zod with common AWS event envelopes. Use the standalone `parse` function (not the handler wrapper) inside handlers so it composes with the dispatcher pattern:

```typescript
import { parse } from '@aws-lambda-powertools/parser';
import { EventBridgeEnvelope } from '@aws-lambda-powertools/parser/envelopes';
import { userEventSchema } from '@/schemas/user-event-schemas';
import type { EventBridgeEvent, Context } from 'aws-lambda';

export async function handleUserEvent(event: EventBridgeEvent<string, unknown>, _context: Context): Promise<void> {
  const detail = parse(event, EventBridgeEnvelope, userEventSchema);
  // detail is typed and validated
}
```

## AWS SDK Client Management

### Singleton Pattern

Instantiate SDK clients outside the handler to reuse connections across invocations:

```typescript
// src/clients/dynamodb-client.ts
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient } from '@aws-sdk/lib-dynamodb';

const client = new DynamoDBClient({
  region: process.env.AWS_REGION,
  maxAttempts: 3,
});

export const ddbClient = DynamoDBDocumentClient.from(client, {
  marshallOptions: {
    removeUndefinedValues: true,
    convertClassInstanceToMap: true,
  },
});
```

### Common Client Options

- Set `maxAttempts` to match the expected retry budget
- Use `requestHandler` with explicit timeouts for predictable behavior
- Wrap clients with `tracer.captureAWSv3Client()` for X-Ray tracing

```typescript
import { tracer } from '@/common/tracer';

export const ddbClient = tracer.captureAWSv3Client(
  DynamoDBDocumentClient.from(client)
);
```

## Idempotency

### Using Powertools Idempotency

```typescript
import { makeIdempotent } from '@aws-lambda-powertools/idempotency';
import { DynamoDBPersistenceLayer } from '@aws-lambda-powertools/idempotency/dynamodb';

const persistenceStore = new DynamoDBPersistenceLayer({
  tableName: process.env.IDEMPOTENCY_TABLE!,
});

async function processOrderInternal(event: OrderEvent): Promise<void> {
  // Business logic here
}

export const processOrder = makeIdempotent(processOrderInternal, {
  persistenceStore,
  config: {
    eventKeyJmesPath: 'orderId',
    expiresAfterSeconds: 3600,
  },
});
```

### Manual Idempotency Pattern

For custom idempotency keys:

```typescript
import { ConditionalCheckFailedException } from '@aws-sdk/client-dynamodb';
import { PutCommand } from '@aws-sdk/lib-dynamodb';

async function processIfNew(idempotencyKey: string, eventId: string): Promise<boolean> {
  try {
    await ddbClient.send(new PutCommand({
      TableName: process.env.IDEMPOTENCY_TABLE!,
      Item: {
        pk: idempotencyKey,
        eventId,
        ttl: Math.floor(Date.now() / 1000) + 86400,
      },
      ConditionExpression: 'attribute_not_exists(pk)',
    }));
    return true; // First time seeing this event
  } catch (error) {
    if (error instanceof ConditionalCheckFailedException) {
      return false; // Already processed
    }
    throw error;
  }
}
```

## Error Handling

### Error Classification

Distinguish between transient (retriable) and permanent (poison-pill) errors:

```typescript
// src/common/errors.ts
export class RetriableError extends Error {
  readonly retriable = true;
  constructor(message: string, public readonly cause?: unknown) {
    super(message);
    this.name = 'RetriableError';
  }
}

export class NonRetriableError extends Error {
  readonly retriable = false;
  constructor(message: string, public readonly cause?: unknown) {
    super(message);
    this.name = 'NonRetriableError';
  }
}
```

### Handling Strategy

```typescript
try {
  await processor.process(record);
} catch (error) {
  if (error instanceof NonRetriableError) {
    // Log and send directly to DLQ / skip record
    logger.error('Non-retriable error, sending to DLQ', { error, record });
    await sendToDlq(record, error);
    continue;
  }

  // Retriable — report back to Lambda for retry
  batchItemFailures.push({ itemIdentifier: record.messageId });
}
```

### Dead Letter Queues

- Configure DLQs on all async invocations (S3, SNS, EventBridge)
- Configure DLQs on event source mappings (SQS, DynamoDB, Kinesis)
- Set `maxRetryAttempts` explicitly on event source mappings
- Implement DLQ replay tooling for recoverable failures
- Alarm on DLQ depth with CloudWatch

## Observability

### Structured Logging

Use `@sazep/sazep-logger` for all structured JSON logging. This is the team's shared Powertools-based logger, standardized across services for consistent log shape, redaction, and correlation-id handling:

```typescript
// src/common/logger.ts
import { Logger } from '@sazep/sazep-logger';

export const logger = new Logger({
  serviceName: process.env.SERVICE_NAME ?? 'event-processor',
  logLevel: (process.env.LOG_LEVEL ?? 'INFO') as 'DEBUG' | 'INFO' | 'WARN' | 'ERROR',
});
```

Logging rules:
- Import `logger` from `@/common/logger` everywhere — never instantiate ad-hoc loggers
- Never import directly from `@aws-lambda-powertools/logger` in application code; go through `@sazep/sazep-logger`
- Include request/event context on every log
- Never log event payloads that may contain PII or secrets
- Use `logger.appendKeys()` for context scoped to the current invocation (cleared by `injectLambdaContext({ clearState: true })`)
- Use `logger.appendPersistentKeys()` for context that should survive across invocations (for example, a tenant ID resolved during cold start). Use `persistentKeys` in the constructor options for values known at module load
- Do **not** use `addPersistentLogAttributes()` or the `persistentLogAttributes` constructor option — both are deprecated in Powertools v2 and replaced by `appendPersistentKeys()` / `persistentKeys`
- Log at INFO for normal operations, WARN for recoverable issues, ERROR for failures

### Tracing

Enable X-Ray tracing on all functions and instrument AWS SDK calls:

```typescript
// src/common/tracer.ts
import { Tracer } from '@aws-lambda-powertools/tracer';

export const tracer = new Tracer({
  serviceName: process.env.SERVICE_NAME ?? 'event-processor',
});
```

Wrap the single entry point with the Powertools middleware (Middy) once in `src/index.ts` to capture cold starts and errors automatically — see the Entry Point and Dispatcher section above. Individual handlers should **not** re-wrap themselves with Middy.

### Metrics

Emit business and operational metrics via CloudWatch EMF:

```typescript
// src/common/metrics.ts
import { Metrics } from '@aws-lambda-powertools/metrics';

export const metrics = new Metrics({
  namespace: 'EventProcessing',
  serviceName: process.env.SERVICE_NAME ?? 'event-processor',
});
```

Emit metrics for:
- Records processed / failed per event source
- Processing duration (if not covered by Lambda Duration metric)
- Business-specific counters (orders created, uploads rejected, etc.)
- Throttle / retry counts

## Configuration

### Environment Variables

Required per function:
- `AWS_REGION` — Set by Lambda
- `SERVICE_NAME` — Used for logging and metrics namespacing
- `LOG_LEVEL` — DEBUG, INFO, WARN, or ERROR
- `POWERTOOLS_LOG_LEVEL` — Alternative for Powertools
- `POWERTOOLS_METRICS_NAMESPACE` — Metrics namespace

Per-function:
- Resource ARNs (queue URLs, table names, bucket names)
- Feature flags
- External endpoint URLs

### Type-Safe Configuration

```typescript
// src/common/config.ts
import { z } from 'zod';

const configSchema = z.object({
  orderTableName: z.string().min(1),
  notificationTopicArn: z.string().startsWith('arn:aws:sns:'),
  idempotencyTableName: z.string().min(1),
  maxRetries: z.coerce.number().int().min(0).max(10).default(3),
});

export const config = configSchema.parse({
  orderTableName: process.env.ORDER_TABLE_NAME,
  notificationTopicArn: process.env.NOTIFICATION_TOPIC_ARN,
  idempotencyTableName: process.env.IDEMPOTENCY_TABLE_NAME,
  maxRetries: process.env.MAX_RETRIES,
});
```

Parse configuration at module load so functions fail fast on startup rather than at first invocation.

## Event Source Mapping Best Practices

### SQS

- Set `batchSize` based on per-record processing time (default 10, max 10,000 for standard queues)
- Enable `reportBatchItemFailures` for partial batch handling
- Set `maximumBatchingWindowInSeconds` for cost efficiency
- Configure `maximumConcurrency` to protect downstream systems
- Use FIFO queues when order matters; standard queues for throughput

### DynamoDB Streams

- Enable `reportBatchItemFailures` for partial batch handling
- Set `maximumRetryAttempts` to bound retries before DLQ
- Use `filterCriteria` to reduce invocations for irrelevant changes
- Set `parallelizationFactor` for higher throughput per shard
- Choose `startingPosition: LATEST` for new deployments, `TRIM_HORIZON` for backfills

### Kinesis

- Enable `reportBatchItemFailures`
- Use `bisectBatchOnFunctionError` for poison-pill isolation
- Set `maximumRecordAgeInSeconds` to bound processing latency
- Monitor `IteratorAge` metric for lag detection

### EventBridge

- Use `InputTransformer` to reshape events at the rule level
- Add `DeadLetterConfig` to targets
- Use event pattern filtering to reduce invocations
- Set `RetryPolicy` with `MaximumRetryAttempts` and `MaximumEventAgeInSeconds`

### S3

- Prefer EventBridge notifications for S3 over direct Lambda triggers (better filtering, DLQ)
- Use prefix/suffix filters to scope which objects trigger Lambda
- Configure `DestinationConfig` with OnFailure SQS/SNS for async failures
- Be aware S3 may deliver duplicate notifications — always design for idempotency

## Performance and Cost

### Cold Start Optimization

- Use `arm64` base images for faster startup and lower cost
- Import only needed SDK clients (`@aws-sdk/client-dynamodb`, not the full `aws-sdk` meta-package)
- Initialize clients and config at module scope, not inside the handler
- Keep the final image small — prune dev dependencies, exclude source TS and tests
- Order Dockerfile layers so dependency layers are cached across builds
- Consider Lambda SnapStart for latency-sensitive workloads (where supported for container images)

### Memory Configuration

- Start at 512 MB for most event processors
- Increase memory to reduce duration — often cost-neutral or cheaper
- Use AWS Lambda Power Tuning to find the optimum

### Concurrency

- Set `reservedConcurrency` to protect downstream systems from spikes
- Set `provisionedConcurrency` only for latency-sensitive synchronous invocations
- Use SQS `maximumConcurrency` for consumer-side throttling

## Security

- Grant least-privilege IAM policies — one role per function
- Never hardcode credentials or ARNs in source
- Use AWS Secrets Manager or Parameter Store for secrets, cached via Powertools Parameters
- Validate all event payloads with Zod
- Encrypt sensitive data at rest (DynamoDB, S3 SSE) and in transit (HTTPS/TLS)
- Tag functions with `Owner`, `Environment`, `DataClassification`
- Enable CloudTrail for audit logging of Lambda invocations and configuration changes

### Parameter Caching

```typescript
import { getParameter } from '@aws-lambda-powertools/parameters/ssm';

const apiKey = await getParameter('/myapp/api-key', {
  maxAge: 900, // Cache 15 minutes
  decrypt: true,
});
```

## Container Image Packaging

Lambda functions in this project are deployed as container images to Amazon ECR, not as ZIP bundles. The service ships a **single image with a single entry point** that routes events to the appropriate handler based on the event shape. All Lambda functions in the service share this image; handler selection happens at runtime, not via different `CMD` values.

### Why a Single Entry Point

- One image to build, test, publish, and promote across environments
- Shared initialization (clients, config, Powertools instances) amortized across handlers
- Consistent observability setup — logger, tracer, metrics middleware applied once
- Simpler CI/CD — no per-handler build matrix or CMD wiring in infrastructure code

Each Lambda function provisioned by the infrastructure repo points at the same image and the same handler path (`index.handler`). The dispatcher inside the image inspects the event and forwards to the correct processor.

### Base Image

Use the official AWS Lambda Node.js base image. It includes the Lambda Runtime Interface Client (RIC) and Runtime Interface Emulator (RIE) for local testing.

```
public.ecr.aws/lambda/nodejs:20
```

Pin the major version tag. Rebuild regularly to pick up security patches.

### Multi-Stage Dockerfile

```dockerfile
# syntax=docker/dockerfile:1.6

# ---------- Build stage ----------
FROM public.ecr.aws/lambda/nodejs:20 AS build

WORKDIR /build

# Install all dependencies (including dev) for the TypeScript build
COPY package*.json tsconfig.json ./
RUN --mount=type=cache,target=/root/.npm npm ci

# Copy source and compile
COPY src ./src
RUN npx tsc

# Prune dev dependencies for the runtime image
RUN --mount=type=cache,target=/root/.npm npm ci --omit=dev

# ---------- Runtime stage ----------
FROM public.ecr.aws/lambda/nodejs:20

# Copy compiled output and production node_modules
COPY --from=build /build/dist ${LAMBDA_TASK_ROOT}
COPY --from=build /build/node_modules ${LAMBDA_TASK_ROOT}/node_modules

# Single entry point for all functions in the service
CMD [ "index.handler" ]
```

### Entry Point and Dispatcher

The single entry point lives at `src/index.ts` and dispatches based on the event shape:

```typescript
// src/index.ts
import middy from '@middy/core';
import { injectLambdaContext } from '@sazep/sazep-logger/middleware';
import { captureLambdaHandler } from '@aws-lambda-powertools/tracer/middleware';
import { logMetrics } from '@aws-lambda-powertools/metrics/middleware';
import { logger } from '@/common/logger';
import { tracer } from '@/common/tracer';
import { metrics } from '@/common/metrics';
import { dispatch } from '@/dispatcher';
import type { Context } from 'aws-lambda';

async function baseHandler(event: unknown, context: Context): Promise<unknown> {
  return dispatch(event, context);
}

export const handler = middy(baseHandler)
  .use(injectLambdaContext(logger, { clearState: true }))
  .use(captureLambdaHandler(tracer))
  .use(logMetrics(metrics, { captureColdStartMetric: true }));
```

> `@sazep/sazep-logger` re-exports the Powertools `injectLambdaContext` middleware so the same package supplies both the `Logger` class and its Middy middleware. If the shared package doesn't re-export the middleware, import it from `@aws-lambda-powertools/logger/middleware` instead — the `Logger` instance from `@sazep/sazep-logger` is directly assignable to it.

```typescript
// src/dispatcher.ts
import { processS3Upload } from '@/handlers/process-s3-upload/handler';
import { processOrderStream } from '@/handlers/process-order-stream/handler';
import { processSqsMessages } from '@/handlers/process-sqs-messages/handler';
import { handleSnsNotification } from '@/handlers/handle-sns-notification/handler';
import { processKinesisStream } from '@/handlers/process-kinesis-stream/handler';
import { handleUserEvent } from '@/handlers/handle-user-event/handler';
import { logger } from '@/common/logger';
import type { Context } from 'aws-lambda';

type EventWithRecords = { Records?: Array<Record<string, unknown>> };
type EventBridgeShape = { source?: string; 'detail-type'?: string; detail?: unknown };

export async function dispatch(event: unknown, context: Context): Promise<unknown> {
  const eventType = identifyEventType(event);
  logger.appendKeys({ eventType });

  switch (eventType) {
    case 's3':
      return processS3Upload(event as never, context);
    case 'dynamodb':
      return processOrderStream(event as never, context);
    case 'sqs':
      return processSqsMessages(event as never, context);
    case 'sns':
      return handleSnsNotification(event as never, context);
    case 'kinesis':
      return processKinesisStream(event as never, context);
    case 'eventbridge':
      return handleUserEvent(event as never, context);
    default:
      logger.error('Unrecognized event shape', { event });
      throw new Error(`Unrecognized event type: ${eventType}`);
  }
}

function identifyEventType(event: unknown): string {
  if (!event || typeof event !== 'object') return 'unknown';

  const withRecords = event as EventWithRecords;
  const firstRecord = withRecords.Records?.[0];

  if (firstRecord) {
    // S3, DynamoDB, SQS, Kinesis use lowercase `eventSource`; SNS uses `EventSource`
    const source =
      (firstRecord as Record<string, string>).eventSource ??
      (firstRecord as Record<string, string>).EventSource;

    switch (source) {
      case 'aws:s3': return 's3';
      case 'aws:dynamodb': return 'dynamodb';
      case 'aws:sqs': return 'sqs';
      case 'aws:sns': return 'sns';
      case 'aws:kinesis': return 'kinesis';
    }
  }

  // EventBridge events have no Records array; they have source + detail-type + detail
  const eb = event as EventBridgeShape;
  if (eb.source && eb['detail-type'] && 'detail' in eb) {
    return 'eventbridge';
  }

  return 'unknown';
}
```

For EventBridge events, further routing on `source` + `detail-type` happens inside `handleUserEvent` (or a dedicated EventBridge sub-router) so that multiple EventBridge rules can target the same function while still dispatching to distinct processors.

### Handler Exports for the Dispatcher

Handlers are no longer the Lambda entry point — they are plain functions the dispatcher calls. Export them under named functions rather than a generic `handler`:

```typescript
// src/handlers/process-order-stream/handler.ts
export async function processOrderStream(
  event: DynamoDBStreamEvent,
  context: Context
): Promise<DynamoDBBatchResponse> {
  // ... logic from earlier in this document
}
```

The observability middleware is applied once in `src/index.ts`, so individual handlers should not re-wrap themselves with Middy.

### CMD and Handler Path

With compiled output emitted to `dist/`, the Dockerfile `CMD` points at `index.handler` relative to `LAMBDA_TASK_ROOT`:

```dockerfile
CMD [ "index.handler" ]
```

Every Lambda function provisioned from this image uses the same `ImageConfig.Command` (or omits it entirely and relies on the Dockerfile `CMD`). The dispatcher handles the rest.

### .dockerignore

```
.git
.github
node_modules
dist
coverage
test
**/*.test.ts
**/*.spec.ts
.env*
*.md
.vscode
.idea
```

### tsconfig for Container Builds

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "commonjs",
    "outDir": "./dist",
    "rootDir": "./src",
    "sourceMap": true,
    "declaration": false,
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "resolveJsonModule": true,
    "forceConsistentCasingInFileNames": true,
    "baseUrl": "./src",
    "paths": {
      "@/*": ["*"]
    }
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.test.ts"]
}
```

If using path aliases (`@/`), run `tsc-alias` after `tsc` to rewrite them to relative paths so the compiled code resolves at runtime without a loader hook.

### Image Size Optimization

- Install production dependencies only in the runtime stage (`npm ci --omit=dev`)
- Exclude source TypeScript, tests, and dev tooling via `.dockerignore`
- Use `arm64` base images for lower cost and faster cold starts
- Enable BuildKit cache mounts for `npm ci` to speed up repeated builds

### Local Testing with RIE

The AWS base image ships with the Runtime Interface Emulator. Test the built image locally:

```bash
# Build
docker build -t event-processor .

# Run with RIE on port 9000
docker run --rm -p 9000:8080 \
  -e AWS_REGION=us-east-1 \
  -e ORDER_TABLE_NAME=orders-local \
  event-processor

# Invoke with different event payloads — dispatcher routes to the right handler
curl -XPOST "http://localhost:9000/2015-03-31/functions/function/invocations" \
  -d @test/fixtures/events/s3-put-event.json

curl -XPOST "http://localhost:9000/2015-03-31/functions/function/invocations" \
  -d @test/fixtures/events/dynamodb-stream-event.json
```

Testing every event shape against the same container is a direct validation of the dispatcher.

## Publishing to ECR

This repository's responsibility ends at building and publishing the container image to ECR. The infrastructure repository consumes the published image by digest.

### Tagging

- Tag every image with the immutable Git SHA: `event-processor:<git-sha>`
- Also publish the resolved image digest (`sha256:...`) as a build artifact for the infrastructure repo to pin against
- Do not use mutable tags (`latest`, `main`) for production deployments
- Enable ECR image scanning (`scanOnPush`) and set a lifecycle policy in the infra repo to expire untagged images

### Publish Command

```bash
aws ecr get-login-password --region "$AWS_REGION" \
  | docker login --username AWS --password-stdin \
    "$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com"

IMAGE_URI="$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/event-processor:$GIT_SHA"

docker build -t "$IMAGE_URI" .
docker push "$IMAGE_URI"

# Resolve and export the immutable digest for the infra repo
DIGEST=$(aws ecr describe-images \
  --repository-name event-processor \
  --image-ids imageTag="$GIT_SHA" \
  --query 'imageDetails[0].imageDigest' \
  --output text)

echo "image_digest=$DIGEST" >> "$GITHUB_OUTPUT"
```

### Handoff to the Infrastructure Repo

Publish the following artifacts for the infrastructure repo to consume:

- `image_uri` — Full ECR URI with SHA tag
- `image_digest` — `sha256:...` digest for immutable references
- `commit_sha` — Git commit this image was built from
- Optional: `event_source_schemas.json` — JSON Schemas derived from Zod schemas, so the infra repo can validate event source mappings against expected payloads

Typical delivery mechanisms:
- GitHub Actions workflow output consumed by a dispatch event on the infra repo
- SSM Parameter written under `/services/event-processor/image-digest` that infra reads at deploy time
- Pull request comment or commit status surfacing the digest

## Infrastructure Contract

The infrastructure repository is responsible for provisioning Lambda functions, event source mappings, IAM, DLQs, and all related AWS resources. To keep the two repos in sync, this application repo publishes the following contract.

### Image Expectations

- **Base image**: `public.ecr.aws/lambda/nodejs:20`
- **Architecture**: `arm64` (must match `Architecture.ARM_64` on the Lambda function)
- **Entry point (handler)**: `index.handler` — **same for every function** in this service
- **Runtime initialization**: All handlers share the entry point; the dispatcher in `src/index.ts` routes based on event shape

### Per-Function Requirements

The infrastructure repo should provision one `DockerImageFunction` per logical handler below. Each function uses the same image and the same `ImageConfig.Command` (`["index.handler"]`), and is differentiated only by its event source, memory, timeout, and IAM role.

| Logical Function | Event Source | Recommended Memory | Recommended Timeout | IAM Needs |
|------------------|--------------|--------------------|-----|-----------|
| process-order-stream | DynamoDB Streams | 512 MB | 30s | Read stream, write to orders table, publish to EventBridge |
| process-s3-upload | S3 → EventBridge | 1024 MB | 2m | Read source bucket, write to processed bucket |
| process-sqs-messages | SQS | 512 MB | 30s | Receive/delete from queue, write to DynamoDB |
| handle-sns-notification | SNS | 256 MB | 15s | Invoke downstream APIs |
| process-kinesis-stream | Kinesis | 1024 MB | 1m | Read shard, write to S3 |
| handle-user-event | EventBridge | 256 MB | 15s | Publish to SNS, write to DynamoDB |

Publish this table as a machine-readable `function-contract.json` artifact if the infra repo consumes it programmatically.

### Required Event Source Mapping Settings

- **SQS / DynamoDB Streams / Kinesis**: `ReportBatchItemFailures` must be enabled — handlers return `batchItemFailures` and rely on this behavior
- **DynamoDB Streams / Kinesis**: Set `BisectBatchOnFunctionError: true` for poison-pill isolation
- **S3**: Prefer EventBridge notifications over direct Lambda triggers (better filtering, DLQ, retry control)
- **EventBridge**: Targets must have `DeadLetterConfig` and an explicit `RetryPolicy`
- **Async sources (S3, SNS, EventBridge)**: Function must have `DeadLetterQueue` configured at the Lambda level

### Required Environment Variables

The infra repo must set these on every function in this service:

| Variable | Purpose | Example |
|----------|---------|---------|
| `SERVICE_NAME` | Logger/metrics namespacing | `event-processor` |
| `LOG_LEVEL` | Logger verbosity | `INFO` |
| `POWERTOOLS_LOG_LEVEL` | Powertools logger | `INFO` |
| `POWERTOOLS_METRICS_NAMESPACE` | CloudWatch EMF namespace | `EventProcessing` |
| `AWS_LAMBDA_EXEC_WRAPPER` | Only if using external instrumentation | (optional) |

Per-function environment variables (resource ARNs, table names, queue URLs, feature flags) are the infra repo's responsibility to supply. The application reads them through `src/common/config.ts`, which parses them with Zod at module load — so mis-configured functions fail fast on cold start rather than on first invocation.

### Observability Requirements

- `Tracing: Active` on every function (X-Ray)
- CloudWatch Log Retention set to the team standard (typically 30 or 90 days)
- CloudWatch alarms on: function errors, DLQ depth, `IteratorAge` (for stream sources), throttles
- Metrics namespace matches `POWERTOOLS_METRICS_NAMESPACE`

### IAM Notes

- One role per function (least privilege)
- Roles must include `AWSLambdaBasicExecutionRole` for CloudWatch Logs
- Roles must include `AWSXRayDaemonWriteAccess` (or the managed policy equivalent) since tracing is active
- Functions using Powertools Idempotency need `dynamodb:GetItem`, `PutItem`, `UpdateItem`, `DeleteItem` on the idempotency table

## Build Performance

- Order Dockerfile layers from least to most frequently changed (`package.json` before source)
- Use BuildKit cache mounts for `npm ci`
- In CI, enable Docker layer caching (GitHub Actions `docker/build-push-action` with `cache-from`/`cache-to`, or ECR's pull-through cache)
- Build the image once per commit in this repo; the infra repo deploys the same image digest across environments

## NPM Scripts

```json
{
  "scripts": {
    "build": "tsc",
    "lint": "eslint src --ext .ts",
    "test": "jest",
    "test:coverage": "jest --coverage",
    "docker:build": "docker build -t event-processor .",
    "docker:run": "docker run --rm -p 9000:8080 event-processor"
  }
}
```

## Git Workflow

### Commits

- Use conventional commits: `type(scope): message`
- Types: feat, fix, docs, style, refactor, test, chore
- Scope the function name when practical: `fix(order-stream): handle malformed sequence numbers`

### Branch Strategy

- `main` — Production-ready code
- `feature/*` — Feature branches
- `fix/*` — Bug fix branches

## Documentation

- Add JSDoc comments for handler exports and public service methods
- Document the event source, trigger configuration, and expected payload shape in the handler file header
- Keep an `ARCHITECTURE.md` describing the end-to-end event flow for each function
- Document environment variables in `.env.example`
- Include runbooks for common operational tasks (DLQ replay, backfill, rollback)
