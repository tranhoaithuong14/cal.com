# PHASE 12: DevOps, Testing & Deployment

> **Mục tiêu**: Hiểu CI/CD pipelines, testing strategies (unit, integration, E2E), deployment configurations, Docker setup, và monitoring.

## Overview

Cal.com sử dụng **modern DevOps stack** để đảm bảo quality và reliability:

```
DevOps Stack:
┌────────────────────────────────────────────────────────────┐
│                    CI/CD & Testing                         │
├──────────────────────┬─────────────────────────────────────┤
│                      │                                     │
│  CI/CD               │  GitHub Actions                     │
│  Testing Framework   │  Playwright (E2E), Vitest (Unit)    │
│  Code Quality        │  ESLint, Prettier, TypeScript       │
│  Containerization    │  Docker (multi-stage builds)        │
│  Deployment          │  Vercel, Docker, Kubernetes         │
│  Monitoring          │  Sentry, Checkly                    │
│  Performance         │  Lighthouse, Web Vitals             │
│                      │                                     │
└──────────────────────┴─────────────────────────────────────┘
```

---

## 1. CI/CD Pipelines

### 1.1 GitHub Actions Overview

**Location**: `.github/workflows/`

**Main Workflows**:
```bash
all-checks.yml                    # Main PR validation
api-v1-production-build.yml       # API v1 build
api-v2-production-build.yml       # API v2 build
atoms-production-build.yml        # Atoms library build
check-types.yml                   # TypeScript type checking
e2e.yml                          # End-to-end tests
lint.yml                         # Linting
test.yml                         # Unit/integration tests
```

### 1.2 Main CI Pipeline

**File**: `.github/workflows/all-checks.yml`

```yaml
name: All Checks

on:
  pull_request:
    branches:
      - main
  push:
    branches:
      - main

jobs:
  # 1. Linting
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'yarn'
      - run: yarn install
      - run: yarn lint
      - run: yarn format:check

  # 2. Type Checking
  type-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'yarn'
      - run: yarn install
      - run: yarn type-check

  # 3. Unit Tests
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:14
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    env:
      DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'yarn'
      - run: yarn install
      - run: yarn db:migrate:deploy
      - run: yarn test

  # 4. E2E Tests
  e2e:
    runs-on: ubuntu-latest
    timeout-minutes: 60
    services:
      postgres:
        image: postgres:14
    env:
      DATABASE_URL: postgresql://postgres:postgres@localhost:5432/e2e
      NEXTAUTH_SECRET: supersecret
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'yarn'
      - run: yarn install
      - run: yarn db:migrate:deploy
      - run: npx playwright install --with-deps chromium
      - run: yarn test:e2e
      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: playwright-report
          path: playwright-report/

  # 5. Build Check
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'yarn'
      - run: yarn install
      - run: yarn build
```

### 1.3 Cron Jobs

**Scheduled workflows** cho background tasks:

```yaml
# .github/workflows/cron-bookingReminder.yml
name: Booking Reminders Cron

on:
  schedule:
    # Runs every 5 minutes
    - cron: '*/5 * * * *'

jobs:
  send-reminders:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Trigger reminders
        run: |
          curl -X POST \
            -H "x-cal-cron-api-key: ${{ secrets.CRON_API_KEY }}" \
            https://app.cal.com/api/cron/bookingReminder
```

**Other Cron Jobs**:
- `cron-bookingReminder.yml`: Send booking reminders
- `cron-scheduleEmailReminders.yml`: Schedule email reminders
- `cron-monthlyDigestEmail.yml`: Send monthly usage digest
- `cron-downgradeUsers.yml`: Downgrade expired subscriptions
- `cron-changeTimeZone.yml`: Update timezone data

### 1.4 Deployment Workflows

**Vercel Deployment** (automatic):
```yaml
# vercel.json
{
  "buildCommand": "yarn build",
  "installCommand": "yarn install",
  "framework": "nextjs",
  "regions": ["iad1"],
  "env": {
    "DATABASE_URL": "@database-url",
    "NEXTAUTH_SECRET": "@nextauth-secret"
  }
}
```

**Docker Deployment**:
```yaml
# .github/workflows/docker-build.yml
name: Docker Build

on:
  push:
    tags:
      - 'v*'

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}
      - uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: calcom/cal.com:${{ github.ref_name }}
          cache-from: type=registry,ref=calcom/cal.com:cache
          cache-to: type=registry,ref=calcom/cal.com:cache,mode=max
```

---

## 2. Testing Strategy

### 2.1 Testing Pyramid

```
Testing Pyramid:
         ┌─────────────┐
         │   E2E Tests │  ← Playwright (slow, high value)
         │   (20%)     │
         └─────────────┘
       ┌───────────────────┐
       │ Integration Tests │  ← Vitest + tRPC (medium)
       │      (30%)        │
       └───────────────────┘
    ┌─────────────────────────┐
    │     Unit Tests          │  ← Vitest (fast, many)
    │       (50%)             │
    └─────────────────────────┘
```

### 2.2 Unit Tests (Vitest)

**Configuration**: `vitest.config.ts`

```typescript
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globals: true,
    environment: 'node',
    setupFiles: ['./tests/setup.ts'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'html', 'lcov'],
      exclude: [
        'node_modules/',
        'tests/',
        '**/*.test.ts',
        '**/*.test.tsx',
      ],
    },
  },
});
```

**Example Unit Test**:
```typescript
// packages/lib/slugify.test.ts

import { describe, it, expect } from 'vitest';
import slugify from './slugify';

describe('slugify', () => {
  it('should convert string to lowercase slug', () => {
    expect(slugify('Hello World')).toBe('hello-world');
  });

  it('should remove special characters', () => {
    expect(slugify('Hello@World!')).toBe('hello-world');
  });

  it('should handle accents', () => {
    expect(slugify('Café Münchën')).toBe('cafe-munchen');
  });

  it('should truncate long slugs', () => {
    const longString = 'a'.repeat(100);
    expect(slugify(longString).length).toBeLessThanOrEqual(50);
  });
});
```

**Testing tRPC Procedures**:
```typescript
// packages/trpc/server/routers/viewer/eventTypes.test.ts

import { describe, it, expect, vi, beforeEach } from 'vitest';
import { createInnerTRPCContext } from '../../trpc';
import { appRouter } from '../_app';

describe('eventTypes router', () => {
  beforeEach(() => {
    vi.clearAllMocks();
  });

  it('should list user event types', async () => {
    const ctx = createInnerTRPCContext({
      session: {
        user: { id: 1, email: 'test@example.com' },
      },
    });

    const caller = appRouter.createCaller(ctx);

    const result = await caller.viewer.eventTypes.list();

    expect(result.eventTypeGroups).toBeDefined();
    expect(Array.isArray(result.eventTypeGroups)).toBe(true);
  });

  it('should create event type', async () => {
    const ctx = createInnerTRPCContext({
      session: { user: { id: 1 } },
    });

    const caller = appRouter.createCaller(ctx);

    const result = await caller.viewer.eventTypes.create({
      title: '30 Minute Meeting',
      slug: '30min',
      length: 30,
    });

    expect(result.eventType).toBeDefined();
    expect(result.eventType.title).toBe('30 Minute Meeting');
  });
});
```

### 2.3 Integration Tests

**API Integration Tests**:
```typescript
// apps/api/v1/test/lib/bookings/_post.test.ts

import { describe, it, expect } from 'vitest';
import { createMocks } from 'node-mocks-http';
import handler from '../../../pages/api/bookings/index';
import { prismaMock } from '../../../../tests/libs/__mocks__/prisma';

describe('POST /api/bookings', () => {
  it('should create a booking', async () => {
    const { req, res } = createMocks({
      method: 'POST',
      headers: {
        authorization: 'Bearer test-api-key',
      },
      body: {
        eventTypeId: 1,
        start: '2025-11-19T14:00:00Z',
        end: '2025-11-19T14:30:00Z',
        responses: {
          name: 'John Doe',
          email: 'john@example.com',
        },
      },
    });

    // Mock Prisma queries
    prismaMock.eventType.findUnique.mockResolvedValue({
      id: 1,
      title: '30 Minute Meeting',
      length: 30,
    });

    prismaMock.booking.create.mockResolvedValue({
      id: 1,
      uid: 'test-booking-123',
      status: 'ACCEPTED',
    });

    await handler(req, res);

    expect(res._getStatusCode()).toBe(201);
    expect(JSON.parse(res._getData())).toMatchObject({
      booking: {
        uid: 'test-booking-123',
        status: 'ACCEPTED',
      },
    });
  });

  it('should return 400 for invalid input', async () => {
    const { req, res } = createMocks({
      method: 'POST',
      body: {
        // Missing required fields
      },
    });

    await handler(req, res);

    expect(res._getStatusCode()).toBe(400);
  });
});
```

### 2.4 E2E Tests (Playwright)

**Configuration**: `playwright.config.ts`

```typescript
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests',
  timeout: 60000,
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: process.env.CI ? 'html' : 'list',

  use: {
    baseURL: 'http://localhost:3000',
    trace: 'retain-on-failure',
    screenshot: 'only-on-failure',
  },

  projects: [
    {
      name: 'chromium',
      use: {
        ...devices['Desktop Chrome'],
        locale: 'en-US',
        timezoneId: 'Europe/London',
      },
    },
    {
      name: 'mobile',
      use: {
        ...devices['iPhone 13'],
      },
    },
  ],

  webServer: {
    command: 'yarn workspace @calcom/web start -p 3000',
    port: 3000,
    reuseExistingServer: !process.env.CI,
  },
});
```

**Location**: `playwright.config.ts:1-80`

**Example E2E Test**:
```typescript
// tests/booking-flow.e2e.ts

import { test, expect } from '@playwright/test';

test.describe('Booking Flow', () => {
  test('should complete a booking', async ({ page }) => {
    // 1. Navigate to booking page
    await page.goto('/john/30min');

    // 2. Select date
    await page.click('[data-testid="day-2025-11-19"]');

    // 3. Select time slot
    await page.click('[data-testid="time-14:00"]');

    // 4. Fill booking form
    await page.fill('[name="name"]', 'John Doe');
    await page.fill('[name="email"]', 'john@example.com');
    await page.fill('[name="notes"]', 'Looking forward to the meeting');

    // 5. Submit
    await page.click('[data-testid="confirm-booking"]');

    // 6. Verify confirmation
    await expect(page.locator('text=Booking confirmed')).toBeVisible();
    await expect(page.locator('[data-testid="booking-uid"]')).toBeVisible();

    // 7. Verify email sent (check database)
    // This would typically check a test email inbox or database
  });

  test('should show conflict error for double booking', async ({ page }) => {
    await page.goto('/john/30min');

    // Try to book a slot that's already taken
    await page.click('[data-testid="day-2025-11-19"]');
    await page.click('[data-testid="time-14:00"]'); // Already booked

    await expect(page.locator('text=This time is no longer available')).toBeVisible();
  });

  test('should reschedule booking', async ({ page }) => {
    const bookingUid = 'existing-booking-123';

    // Navigate to reschedule page
    await page.goto(`/reschedule/${bookingUid}`);

    // Select new date/time
    await page.click('[data-testid="day-2025-11-20"]');
    await page.click('[data-testid="time-15:00"]');

    // Confirm reschedule
    await page.click('[data-testid="confirm-reschedule"]');

    await expect(page.locator('text=Booking rescheduled')).toBeVisible();
  });

  test('should cancel booking', async ({ page }) => {
    const bookingUid = 'existing-booking-123';

    await page.goto(`/booking/${bookingUid}?cancel=true`);

    await page.fill('[name="cancellationReason"]', 'Schedule conflict');
    await page.click('[data-testid="confirm-cancel"]');

    await expect(page.locator('text=Booking cancelled')).toBeVisible();
  });
});
```

**Test Utilities**:
```typescript
// tests/libs/testUtils.ts

export async function createTestUser(page: Page) {
  const email = `test-${Date.now()}@example.com`;

  await page.goto('/auth/signup');
  await page.fill('[name="email"]', email);
  await page.fill('[name="password"]', 'TestPassword123!');
  await page.click('[type="submit"]');

  return { email };
}

export async function createTestEventType(page: Page) {
  await page.goto('/event-types');
  await page.click('[data-testid="new-event-type"]');

  await page.fill('[name="title"]', 'Test Meeting');
  await page.fill('[name="slug"]', 'test-meeting');
  await page.fill('[name="length"]', '30');

  await page.click('[data-testid="save-event-type"]');

  return { slug: 'test-meeting' };
}
```

### 2.5 Performance Testing

**Lighthouse CI**:
```yaml
# .github/workflows/lighthouse.yml
name: Lighthouse CI

on:
  pull_request:
    branches:
      - main

jobs:
  lighthouse:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
      - run: yarn install
      - run: yarn build
      - run: yarn start &
      - uses: treosh/lighthouse-ci-action@v10
        with:
          urls: |
            http://localhost:3000
            http://localhost:3000/john/30min
          uploadArtifacts: true
```

**Load Testing** (k6):
```javascript
// tests/performance/booking-load.js

import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '30s', target: 20 },  // Ramp up
    { duration: '1m', target: 20 },   // Stay at 20 users
    { duration: '30s', target: 0 },   // Ramp down
  ],
};

export default function () {
  // Get available slots
  const slotsRes = http.get('http://localhost:3000/api/trpc/viewer.slots.getSchedule?input={"eventTypeId":1,"startTime":"2025-11-19T00:00:00Z"}');

  check(slotsRes, {
    'slots status is 200': (r) => r.status === 200,
    'has available slots': (r) => JSON.parse(r.body).result.data.slots.length > 0,
  });

  sleep(1);

  // Create booking
  const bookingRes = http.post(
    'http://localhost:3000/api/trpc/viewer.bookings.create',
    JSON.stringify({
      eventTypeId: 1,
      start: '2025-11-19T14:00:00Z',
      responses: {
        name: 'Load Test User',
        email: 'loadtest@example.com',
      },
    }),
    { headers: { 'Content-Type': 'application/json' } }
  );

  check(bookingRes, {
    'booking status is 200': (r) => r.status === 200,
  });

  sleep(1);
}
```

---

## 3. Docker Deployment

### 3.1 Dockerfile

**Multi-stage build** for optimized image size:

```dockerfile
# Stage 1: Builder
FROM node:20 AS builder

WORKDIR /calcom

# Build arguments
ARG DATABASE_URL
ARG NEXTAUTH_SECRET
ARG CALENDSO_ENCRYPTION_KEY

# Environment variables
ENV NEXT_PUBLIC_WEBAPP_URL=http://NEXT_PUBLIC_WEBAPP_URL_PLACEHOLDER \
    DATABASE_URL=$DATABASE_URL \
    NEXTAUTH_SECRET=$NEXTAUTH_SECRET \
    CALENDSO_ENCRYPTION_KEY=$CALENDSO_ENCRYPTION_KEY \
    NODE_OPTIONS=--max-old-space-size=4096 \
    BUILD_STANDALONE=true

# Copy source
COPY package.json yarn.lock .yarnrc.yml ./
COPY .yarn ./.yarn
COPY apps/web ./apps/web
COPY packages ./packages

# Install and build
RUN yarn install
RUN yarn workspace @calcom/web run build

# Remove dev dependencies
RUN rm -rf node_modules/.cache .yarn/cache apps/web/.next/cache

# Stage 2: Runner
FROM node:20 AS runner

WORKDIR /calcom

ENV NODE_ENV=production

COPY --from=builder /calcom ./

EXPOSE 3000

CMD ["yarn", "workspace", "@calcom/web", "start"]
```

**Location**: `Dockerfile:1-80`

### 3.2 Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  # Database
  postgres:
    image: postgres:14
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: calendso
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  # Redis (for caching)
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  # Cal.com Web App
  web:
    build:
      context: .
      args:
        DATABASE_URL: postgresql://postgres:postgres@postgres:5432/calendso
        NEXTAUTH_SECRET: supersecret
        CALENDSO_ENCRYPTION_KEY: supersecret
    environment:
      DATABASE_URL: postgresql://postgres:postgres@postgres:5432/calendso
      NEXTAUTH_SECRET: supersecret
      CALENDSO_ENCRYPTION_KEY: supersecret
      NEXT_PUBLIC_WEBAPP_URL: http://localhost:3000
      REDIS_URL: redis://redis:6379
    ports:
      - "3000:3000"
    depends_on:
      - postgres
      - redis

volumes:
  postgres_data:
```

### 3.3 Kubernetes Deployment

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: calcom-web
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: calcom-web
  template:
    metadata:
      labels:
        app: calcom-web
    spec:
      containers:
      - name: web
        image: calcom/cal.com:latest
        ports:
        - containerPort: 3000
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: calcom-secrets
              key: database-url
        - name: NEXTAUTH_SECRET
          valueFrom:
            secretKeyRef:
              name: calcom-secrets
              key: nextauth-secret
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /api/health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /api/health
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5

---
apiVersion: v1
kind: Service
metadata:
  name: calcom-web
  namespace: production
spec:
  selector:
    app: calcom-web
  ports:
  - port: 80
    targetPort: 3000
  type: LoadBalancer
```

---

## 4. Environment Configuration

### 4.1 Environment Variables

**Location**: `.env.example`

**Core Variables**:
```bash
# Database
DATABASE_URL="postgresql://postgres:@localhost:5450/calendso"
DATABASE_DIRECT_URL="postgresql://postgres:@localhost:5450/calendso"

# NextAuth
NEXT_PUBLIC_WEBAPP_URL='http://localhost:3000'
NEXTAUTH_URL='http://localhost:3000'
NEXTAUTH_SECRET=  # openssl rand -base64 32
NEXTAUTH_COOKIE_DOMAIN=

# Encryption
CALENDSO_ENCRYPTION_KEY=  # openssl rand -base64 32 (must be 32 bytes)

# Cron Jobs
CRON_API_KEY='your-cron-api-key'

# Telemetry
CALCOM_TELEMETRY_DISABLED=  # Set to '1' to disable

# Organizations
ALLOWED_HOSTNAMES='"localhost:3000","cal.com"'
RESERVED_SUBDOMAINS='"app","auth","docs"'
```

**Location**: `.env.example:1-100`

**Email Configuration**:
```bash
# Email Provider (choose one)

# Sendgrid
EMAIL_FROM='Cal.com <noreply@cal.com>'
EMAIL_SERVER_HOST='smtp.sendgrid.net'
EMAIL_SERVER_PORT=587
EMAIL_SERVER_USER=apikey
SENDGRID_API_KEY=

# Postmark
EMAIL_FROM='Cal.com <noreply@cal.com>'
EMAIL_SERVER_HOST='smtp.postmarkapp.com'
EMAIL_SERVER_PORT=587
EMAIL_SERVER_USER=
EMAIL_SERVER_PASSWORD=
```

**Payment Providers**:
```bash
# Stripe
STRIPE_PRIVATE_KEY=sk_test_...
STRIPE_PUBLIC_KEY=pk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...
PAYMENT_FEE_PERCENTAGE=0.005  # 0.5%
PAYMENT_FEE_FIXED=10  # 10 cents

# PayPal
PAYPAL_CLIENT_ID=
PAYPAL_SECRET_KEY=
```

**Enterprise Features**:
```bash
# License Key
CALCOM_LICENSE_KEY=
CAL_SIGNATURE_TOKEN=

# SSO
SAML_DATABASE_URL=postgresql://...
SAML_ADMINS='admin@company.com'
SAML_CLIENT_SECRET_VERIFIER=

# DSYNC
DSYNC_DATABASE_URL=postgresql://...
```

### 4.2 Environment Validation

```typescript
// packages/lib/env.ts

import { z } from 'zod';

const envSchema = z.object({
  // Database
  DATABASE_URL: z.string().url(),
  DATABASE_DIRECT_URL: z.string().url().optional(),

  // NextAuth
  NEXTAUTH_SECRET: z.string().min(32),
  NEXTAUTH_URL: z.string().url(),

  // Encryption
  CALENDSO_ENCRYPTION_KEY: z.string().length(44), // base64 of 32 bytes

  // Optional
  REDIS_URL: z.string().url().optional(),
  CRON_API_KEY: z.string().optional(),
});

export const env = envSchema.parse(process.env);
```

---

## 5. Monitoring & Observability

### 5.1 Error Tracking (Sentry)

```typescript
// apps/web/sentry.client.config.ts

import * as Sentry from '@sentry/nextjs';

Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  environment: process.env.NODE_ENV,

  // Performance monitoring
  tracesSampleRate: process.env.NODE_ENV === 'production' ? 0.1 : 1.0,

  // Error filtering
  beforeSend(event, hint) {
    // Filter out known errors
    if (event.exception) {
      const error = hint.originalException;
      if (error instanceof Error) {
        if (error.message.includes('ResizeObserver')) {
          return null; // Ignore ResizeObserver errors
        }
      }
    }
    return event;
  },

  // User context
  integrations: [
    new Sentry.BrowserTracing(),
    new Sentry.Replay({
      maskAllText: true,
      blockAllMedia: true,
    }),
  ],
});
```

### 5.2 Logging

```typescript
// packages/lib/logger.ts

import winston from 'winston';

export const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  transports: [
    new winston.transports.Console(),
    new winston.transports.File({ filename: 'logs/error.log', level: 'error' }),
    new winston.transports.File({ filename: 'logs/combined.log' }),
  ],
});

// Usage
logger.info('Booking created', { bookingId: 123, userId: 456 });
logger.error('Failed to create booking', { error: error.message });
```

### 5.3 Health Checks

```typescript
// apps/web/pages/api/health.ts

import type { NextApiRequest, NextApiResponse } from 'next';
import { prisma } from '@calcom/prisma';

export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  try {
    // Check database connection
    await prisma.$queryRaw`SELECT 1`;

    // Check Redis (if configured)
    if (process.env.REDIS_URL) {
      // await redisClient.ping();
    }

    res.status(200).json({
      status: 'healthy',
      timestamp: new Date().toISOString(),
      uptime: process.uptime(),
    });
  } catch (error) {
    res.status(503).json({
      status: 'unhealthy',
      error: error.message,
    });
  }
}
```

### 5.4 Metrics (Prometheus)

```typescript
// packages/lib/metrics.ts

import client from 'prom-client';

// Create metrics
export const httpRequestDuration = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status'],
});

export const bookingCreatedCounter = new client.Counter({
  name: 'bookings_created_total',
  help: 'Total number of bookings created',
  labelNames: ['eventType', 'status'],
});

// Middleware to track request duration
export function metricsMiddleware(req, res, next) {
  const start = Date.now();

  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000;
    httpRequestDuration.observe(
      {
        method: req.method,
        route: req.route?.path || req.url,
        status: res.statusCode,
      },
      duration
    );
  });

  next();
}

// Usage
bookingCreatedCounter.inc({
  eventType: 'meeting',
  status: 'confirmed',
});
```

---

## Summary

### DevOps Best Practices

1. **CI/CD**:
   - ✅ Automated testing on every PR
   - ✅ Type checking with TypeScript
   - ✅ Linting with ESLint/Prettier
   - ✅ E2E tests with Playwright
   - ✅ Performance testing with Lighthouse

2. **Testing**:
   - ✅ 50% unit tests (Vitest)
   - ✅ 30% integration tests
   - ✅ 20% E2E tests (Playwright)
   - ✅ Load testing with k6

3. **Deployment**:
   - ✅ Docker multi-stage builds
   - ✅ Kubernetes orchestration
   - ✅ Environment variable validation
   - ✅ Health checks & readiness probes

4. **Monitoring**:
   - ✅ Error tracking (Sentry)
   - ✅ Structured logging (Winston)
   - ✅ Metrics (Prometheus)
   - ✅ Uptime monitoring (Checkly)

### Deployment Checklist

```bash
# 1. Environment Setup
cp .env.example .env
# Edit .env with production values

# 2. Database Migration
yarn db:migrate:deploy

# 3. Build
yarn build

# 4. Test
yarn test
yarn test:e2e

# 5. Docker Build
docker build -t calcom/cal.com:latest .

# 6. Deploy to Kubernetes
kubectl apply -f k8s/

# 7. Verify Health
curl https://app.cal.com/api/health

# 8. Monitor
# Check Sentry dashboard
# Check Prometheus metrics
```

---

**Tác giả**: Claude (AI Assistant)
**Ngày tạo**: 2025-11-18
**Phiên bản**: 1.0
