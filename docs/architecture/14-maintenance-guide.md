# PHASE 14: Maintenance Guide

> **Mục tiêu**: Hướng dẫn debugging, troubleshooting, database maintenance, performance optimization, và giải quyết các vấn đề thường gặp khi maintain Cal.com.

## Overview

Maintenance guide này giúp developers và DevOps engineers:
- Debug và troubleshoot issues
- Optimize performance
- Maintain database health
- Monitor system health
- Handle common errors

```
Maintenance Stack:
┌────────────────────────────────────────────────────────────┐
│                  Maintenance & Operations                  │
├──────────────────────┬─────────────────────────────────────┤
│                      │                                     │
│  Debugging           │  Chrome DevTools, React DevTools    │
│  Logging             │  Winston, Pino                      │
│  Monitoring          │  Sentry, Checkly, Prometheus        │
│  Database            │  Prisma Studio, pg_stat_statements  │
│  Performance         │  Lighthouse, k6                     │
│  Backup              │  PostgreSQL backups, S3             │
│                      │                                     │
└──────────────────────┴─────────────────────────────────────┘
```

---

## 1. Debugging Techniques

### 1.1 Browser DevTools

**React DevTools**:
```bash
# Install extension
# Chrome: https://chrome.google.com/webstore/detail/react-developer-tools/fmkadmapgofadopljbjfkapdkoienihi

# Features:
- Component tree inspection
- Props and state inspection
- Hooks debugging
- Performance profiling
```

**Network Tab**:
```typescript
// Debug tRPC calls
1. Open Network tab
2. Filter by "trpc"
3. Look for failed requests
4. Check request/response payload

// Example tRPC call
POST /api/trpc/viewer.bookings.create
Request: { eventTypeId: 123, start: "2025-11-19T14:00:00Z" }
Response: { error: { code: "CONFLICT", message: "Time slot unavailable" } }
```

**Console Debugging**:
```typescript
// ✅ Good: Use structured logging

console.group('Booking Flow');
console.log('Event Type ID:', eventTypeId);
console.log('Selected Date:', selectedDate);
console.log('Available Slots:', slots);
console.groupEnd();

// ✅ Good: Use debugger statement

function createBooking(input: BookingInput) {
  debugger;  // Execution will pause here

  const booking = await prisma.booking.create({
    data: input,
  });
}

// ❌ Bad: console.log everywhere

console.log('1');
console.log('2');
console.log(data);
```

### 1.2 Server-Side Debugging

**Node.js Debugger**:
```bash
# Run with inspect flag
node --inspect-brk node_modules/.bin/next dev

# Or in package.json
"debug": "NODE_OPTIONS='--inspect' next dev"

# Then attach Chrome DevTools
# Open chrome://inspect
```

**VS Code Debugging**:
```json
// .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Next.js: debug server-side",
      "type": "node-terminal",
      "request": "launch",
      "command": "yarn dev"
    },
    {
      "name": "Next.js: debug client-side",
      "type": "chrome",
      "request": "launch",
      "url": "http://localhost:3000"
    }
  ]
}
```

**Prisma Query Logging**:
```typescript
// Enable Prisma query logging

const prisma = new PrismaClient({
  log: [
    { level: 'query', emit: 'event' },
    { level: 'error', emit: 'stdout' },
    { level: 'warn', emit: 'stdout' },
  ],
});

prisma.$on('query', (e) => {
  console.log('Query:', e.query);
  console.log('Params:', e.params);
  console.log('Duration:', e.duration + 'ms');
});
```

### 1.3 tRPC Debugging

```typescript
// Enable tRPC logging

import { loggerLink } from '@trpc/client';

export const trpc = createTRPCNext<AppRouter>({
  config() {
    return {
      links: [
        loggerLink({
          enabled: (opts) =>
            process.env.NODE_ENV === 'development' ||
            (opts.direction === 'down' && opts.result instanceof Error),
        }),
        httpBatchLink({
          url: '/api/trpc',
        }),
      ],
    };
  },
});

// This will log:
// → query viewer.eventTypes.list started
// ✓ query viewer.eventTypes.list ended - 234ms
```

---

## 2. Common Issues & Solutions

### 2.1 Booking Creation Failures

**Issue**: "Time slot no longer available"

```typescript
// Cause: Race condition (two users booking same slot)

// Solution 1: Database transaction with locking
async function createBooking(input: BookingInput) {
  return await prisma.$transaction(async (tx) => {
    // Lock the time slot
    const existingBooking = await tx.booking.findFirst({
      where: {
        eventTypeId: input.eventTypeId,
        startTime: input.start,
        status: { in: ['ACCEPTED', 'PENDING'] },
      },
    });

    if (existingBooking) {
      throw new TRPCError({
        code: 'CONFLICT',
        message: 'This time slot is no longer available',
      });
    }

    // Create booking
    return await tx.booking.create({
      data: input,
    });
  });
}

// Solution 2: Optimistic locking with version field
model Booking {
  id: Int
  version: Int @default(0)  // Increment on each update
  // ...
}

// Update with version check
await prisma.booking.update({
  where: {
    id: bookingId,
    version: currentVersion,  // Will fail if version changed
  },
  data: {
    version: { increment: 1 },
    // ... other updates
  },
});
```

**Issue**: "Calendar busy check failed"

```typescript
// Cause: Calendar API timeout or invalid credentials

// Solution: Add retry logic and better error handling
async function checkBusyTimes(userId: number, start: Date, end: Date) {
  const credentials = await prisma.credential.findMany({
    where: {
      userId,
      type: { in: ['google_calendar', 'office365_calendar'] },
    },
  });

  const busyTimes = await Promise.allSettled(
    credentials.map(async (credential) => {
      try {
        return await retry(
          () => getCalendarBusyTimes(credential, start, end),
          { retries: 3, delay: 1000 }
        );
      } catch (error) {
        logger.error('Failed to check calendar', {
          credentialId: credential.id,
          error: error.message,
        });
        // Don't fail entire booking, just skip this calendar
        return [];
      }
    })
  );

  return busyTimes
    .filter(result => result.status === 'fulfilled')
    .flatMap(result => result.value);
}
```

### 2.2 Authentication Issues

**Issue**: "Session expired" or "Invalid session"

```typescript
// Cause: JWT expiration or invalid signature

// Solution: Refresh session or redirect to login
import { signOut } from 'next-auth/react';

async function checkSession() {
  try {
    const session = await getSession();

    if (!session) {
      // Redirect to login
      router.push('/auth/login');
      return;
    }

    // Check if session is about to expire (< 5 minutes)
    const expiresAt = new Date(session.expires);
    const now = new Date();
    const minutesUntilExpiry = (expiresAt.getTime() - now.getTime()) / 1000 / 60;

    if (minutesUntilExpiry < 5) {
      // Refresh session
      await signOut({ redirect: false });
      router.push('/auth/login');
    }
  } catch (error) {
    logger.error('Session check failed', { error });
    await signOut({ redirect: true });
  }
}
```

**Issue**: "SSO login fails"

```bash
# Debugging steps:

1. Check SAML_DATABASE_URL is set
   echo $SAML_DATABASE_URL

2. Check SAML connection in database
   SELECT * FROM saml_connections WHERE tenant = 'your-org';

3. Enable SAML debug logging
   SAML_DEBUG=true yarn dev

4. Check IdP metadata is correct
   - Entity ID matches
   - ACS URL is correct
   - Certificate is valid

5. Test SAML flow
   curl -X POST https://app.cal.com/api/auth/saml/acs \
     -d "SAMLResponse=..." \
     -H "Content-Type: application/x-www-form-urlencoded"
```

### 2.3 Email Delivery Issues

**Issue**: Emails not sending

```typescript
// Debugging checklist:

// 1. Check email configuration
console.log({
  EMAIL_SERVER_HOST: process.env.EMAIL_SERVER_HOST,
  EMAIL_SERVER_PORT: process.env.EMAIL_SERVER_PORT,
  EMAIL_SERVER_USER: process.env.EMAIL_SERVER_USER,
  EMAIL_FROM: process.env.EMAIL_FROM,
});

// 2. Test email connection
import nodemailer from 'nodemailer';

async function testEmailConnection() {
  const transporter = nodemailer.createTransport({
    host: process.env.EMAIL_SERVER_HOST,
    port: parseInt(process.env.EMAIL_SERVER_PORT),
    auth: {
      user: process.env.EMAIL_SERVER_USER,
      pass: process.env.EMAIL_SERVER_PASSWORD,
    },
  });

  try {
    await transporter.verify();
    console.log('✅ Email server connection successful');
  } catch (error) {
    console.error('❌ Email server connection failed:', error);
  }
}

// 3. Check email queue
SELECT * FROM email_queue WHERE status = 'pending' ORDER BY created_at DESC LIMIT 10;

// 4. Check for rate limits
// Most providers have rate limits (e.g., SendGrid: 100 emails/second)

// 5. Verify email template rendering
import { renderEmail } from '@calcom/emails';

const html = renderEmail('BookingConfirmation', {
  booking: testBooking,
});
console.log(html);  // Check for errors
```

### 2.4 Database Performance Issues

**Issue**: Slow queries

```sql
-- Find slow queries
SELECT
  query,
  calls,
  total_time,
  mean_time,
  max_time
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 10;

-- Example slow query
SELECT * FROM bookings WHERE user_id = 123;  -- Missing index!

-- Solution: Add index
CREATE INDEX idx_bookings_user_id ON bookings(user_id);

-- Verify index usage
EXPLAIN ANALYZE SELECT * FROM bookings WHERE user_id = 123;
```

**Issue**: Connection pool exhausted

```typescript
// Cause: Too many concurrent connections

// Solution 1: Increase pool size
DATABASE_URL="postgresql://user:pass@localhost:5432/db?connection_limit=20"

// Solution 2: Use connection pooler (PgBouncer)
DATABASE_URL="postgresql://user:pass@pgbouncer:6432/db"
DATABASE_DIRECT_URL="postgresql://user:pass@localhost:5432/db"

// Solution 3: Close connections properly
async function handler(req, res) {
  try {
    const data = await prisma.booking.findMany();
    res.json(data);
  } finally {
    // Prisma handles this automatically, but make sure not to create
    // new PrismaClient instances in hot code paths
  }
}
```

---

## 3. Database Maintenance

### 3.1 Migrations

```bash
# Create new migration
npx prisma migrate dev --name add_booking_index

# Apply migrations in production
npx prisma migrate deploy

# Reset database (DANGEROUS - only in dev)
npx prisma migrate reset

# Check migration status
npx prisma migrate status

# Rollback (manual)
# Prisma doesn't support automatic rollback
# You need to create a new migration that reverses changes
```

### 3.2 Database Backup

```bash
# PostgreSQL backup
pg_dump -h localhost -U postgres -d calendso > backup_$(date +%Y%m%d).sql

# Restore from backup
psql -h localhost -U postgres -d calendso < backup_20251118.sql

# Automated daily backups (crontab)
0 2 * * * pg_dump -h localhost -U postgres -d calendso | gzip > /backups/cal_$(date +\%Y\%m\%d).sql.gz

# Keep only last 30 days
0 3 * * * find /backups -name "cal_*.sql.gz" -mtime +30 -delete
```

### 3.3 Database Cleanup

```sql
-- Clean up old booking reminders (sent > 30 days ago)
DELETE FROM workflow_reminders
WHERE scheduled_date < NOW() - INTERVAL '30 days'
  AND status = 'SENT';

-- Clean up cancelled bookings (older than 90 days)
DELETE FROM bookings
WHERE status = 'CANCELLED'
  AND updated_at < NOW() - INTERVAL '90 days';

-- Vacuum and analyze
VACUUM ANALYZE bookings;
VACUUM ANALYZE users;

-- Reindex
REINDEX TABLE bookings;
```

### 3.4 Prisma Studio

```bash
# Open Prisma Studio for database browsing
npx prisma studio

# Opens at http://localhost:5555
# Features:
# - Browse all tables
# - Edit data
# - Filter and search
# - View relationships
```

---

## 4. Performance Optimization

### 4.1 Database Query Optimization

```typescript
// ❌ Bad: N+1 query problem
async function getEventTypesWithBookings(userId: number) {
  const eventTypes = await prisma.eventType.findMany({
    where: { userId },
  });

  // This runs a query for EACH event type (N+1)
  for (const et of eventTypes) {
    et.bookings = await prisma.booking.findMany({
      where: { eventTypeId: et.id },
    });
  }

  return eventTypes;
}

// ✅ Good: Use include to fetch in one query
async function getEventTypesWithBookings(userId: number) {
  return await prisma.eventType.findMany({
    where: { userId },
    include: {
      bookings: true,  // Single query with JOIN
    },
  });
}

// ✅ Even better: Use select to fetch only needed fields
async function getEventTypesWithBookings(userId: number) {
  return await prisma.eventType.findMany({
    where: { userId },
    select: {
      id: true,
      title: true,
      slug: true,
      bookings: {
        select: {
          id: true,
          startTime: true,
          status: true,
        },
        where: {
          status: 'ACCEPTED',
        },
      },
    },
  });
}
```

### 4.2 Caching Strategies

```typescript
// Redis caching for frequently accessed data

import Redis from 'ioredis';

const redis = new Redis(process.env.REDIS_URL);

async function getEventType(id: number) {
  const cacheKey = `event-type:${id}`;

  // Try cache first
  const cached = await redis.get(cacheKey);
  if (cached) {
    return JSON.parse(cached);
  }

  // Fetch from database
  const eventType = await prisma.eventType.findUnique({
    where: { id },
  });

  // Cache for 5 minutes
  await redis.setex(cacheKey, 300, JSON.stringify(eventType));

  return eventType;
}

// Invalidate cache on update
async function updateEventType(id: number, data: EventTypeUpdateInput) {
  const updated = await prisma.eventType.update({
    where: { id },
    data,
  });

  // Invalidate cache
  await redis.del(`event-type:${id}`);

  return updated;
}
```

### 4.3 React Performance

```typescript
// ✅ Good: Use React.memo for expensive components

export const BookingCard = React.memo<BookingCardProps>(
  ({ booking }) => {
    return (
      <div>
        <h3>{booking.title}</h3>
        <p>{booking.startTime}</p>
      </div>
    );
  },
  // Custom comparison function
  (prevProps, nextProps) => {
    return prevProps.booking.id === nextProps.booking.id &&
           prevProps.booking.status === nextProps.booking.status;
  }
);

// ✅ Good: Use useMemo for expensive calculations

function AvailabilityCalendar({ slots }: Props) {
  const availableDates = useMemo(() => {
    return slots
      .filter(slot => !slot.booked)
      .map(slot => slot.date)
      .sort();
  }, [slots]);

  return <Calendar dates={availableDates} />;
}

// ✅ Good: Use useCallback to stabilize function references

function BookingForm() {
  const handleSubmit = useCallback(async (data: FormData) => {
    await createBooking(data);
  }, []);  // Stable reference

  return <Form onSubmit={handleSubmit} />;
}
```

### 4.4 Next.js Optimization

```typescript
// ✅ Good: Use static generation for public pages

export async function generateStaticParams() {
  const users = await prisma.user.findMany({
    where: { username: { not: null } },
    select: { username: true },
  });

  return users.map(user => ({
    username: user.username,
  }));
}

export default async function UserPage({ params }) {
  const user = await prisma.user.findUnique({
    where: { username: params.username },
  });

  return <UserProfile user={user} />;
}

// ✅ Good: Use dynamic imports for large components

import dynamic from 'next/dynamic';

const HeavyComponent = dynamic(() => import('./HeavyComponent'), {
  loading: () => <Spinner />,
  ssr: false,  // Don't server-render if not needed
});

// ✅ Good: Optimize images

import Image from 'next/image';

<Image
  src="/avatar.jpg"
  alt="User avatar"
  width={200}
  height={200}
  placeholder="blur"
  blurDataURL="data:image/..."
/>
```

---

## 5. Monitoring & Alerting

### 5.1 Health Checks

```typescript
// apps/web/pages/api/health.ts

export default async function handler(req, res) {
  const checks = {
    database: false,
    redis: false,
    email: false,
  };

  // Check database
  try {
    await prisma.$queryRaw`SELECT 1`;
    checks.database = true;
  } catch (error) {
    logger.error('Database health check failed', { error });
  }

  // Check Redis
  try {
    if (redis) {
      await redis.ping();
      checks.redis = true;
    }
  } catch (error) {
    logger.error('Redis health check failed', { error });
  }

  // Check email (optional)
  try {
    if (process.env.EMAIL_SERVER_HOST) {
      // Don't actually send, just verify connection
      checks.email = true;
    }
  } catch (error) {
    logger.error('Email health check failed', { error });
  }

  const isHealthy = checks.database;  // Database is critical

  res.status(isHealthy ? 200 : 503).json({
    status: isHealthy ? 'healthy' : 'unhealthy',
    checks,
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
  });
}
```

### 5.2 Metrics Collection

```typescript
// packages/lib/metrics.ts

import client from 'prom-client';

// Metrics
export const bookingCreated = new client.Counter({
  name: 'bookings_created_total',
  help: 'Total bookings created',
  labelNames: ['status', 'event_type'],
});

export const bookingDuration = new client.Histogram({
  name: 'booking_duration_seconds',
  help: 'Booking creation duration',
  buckets: [0.1, 0.5, 1, 2, 5],
});

export const activeUsers = new client.Gauge({
  name: 'active_users',
  help: 'Number of active users',
});

// Usage
async function createBooking(input: BookingInput) {
  const start = Date.now();

  try {
    const booking = await prisma.booking.create({ data: input });

    bookingCreated.inc({
      status: booking.status,
      event_type: booking.eventType.slug,
    });

    bookingDuration.observe((Date.now() - start) / 1000);

    return booking;
  } catch (error) {
    bookingCreated.inc({ status: 'failed', event_type: 'unknown' });
    throw error;
  }
}

// Expose metrics endpoint
// apps/web/pages/api/metrics.ts
import { register } from 'prom-client';

export default async function handler(req, res) {
  res.setHeader('Content-Type', register.contentType);
  res.send(await register.metrics());
}
```

### 5.3 Alerting

```yaml
# Prometheus alerts (alertmanager.yml)

groups:
  - name: cal.com
    rules:
      # High error rate
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.05
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High error rate detected"

      # Database connection issues
      - alert: DatabaseDown
        expr: up{job="postgres"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Database is down"

      # Slow booking creation
      - alert: SlowBookingCreation
        expr: histogram_quantile(0.95, booking_duration_seconds) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Booking creation is slow (p95 > 2s)"
```

---

## 6. Incident Response

### 6.1 Incident Checklist

```markdown
## Incident Response Checklist

1. **Identify**:
   - [ ] What is the issue? (error, performance, downtime)
   - [ ] What is affected? (users, feature, API)
   - [ ] When did it start?

2. **Assess**:
   - [ ] Check monitoring dashboards (Sentry, Prometheus)
   - [ ] Check logs (application, database, server)
   - [ ] Check health endpoints (/api/health)

3. **Mitigate**:
   - [ ] Can we rollback? (recent deploy)
   - [ ] Can we scale up? (increase resources)
   - [ ] Can we disable feature? (feature flag)

4. **Communicate**:
   - [ ] Update status page
   - [ ] Notify users (email, Twitter)
   - [ ] Keep team informed (Slack)

5. **Fix**:
   - [ ] Identify root cause
   - [ ] Deploy fix
   - [ ] Verify fix works

6. **Post-mortem**:
   - [ ] Document what happened
   - [ ] Document what we did
   - [ ] Identify preventive measures
```

### 6.2 Common Incident Scenarios

**Scenario 1: Database connection pool exhausted**

```bash
# Symptoms
- Timeouts in application
- "Too many clients" errors

# Quick fix
1. Increase connection pool
   DATABASE_URL="...?connection_limit=50"

2. Restart application
   kubectl rollout restart deployment/calcom-web

3. Monitor connection usage
   SELECT count(*) FROM pg_stat_activity;

# Long-term fix
- Implement connection pooler (PgBouncer)
- Optimize queries to reduce connection time
```

**Scenario 2: High CPU usage**

```bash
# Symptoms
- Slow response times
- CPU at 100%

# Investigate
1. Check which queries are slow
   SELECT * FROM pg_stat_statements ORDER BY total_time DESC;

2. Check for long-running queries
   SELECT pid, query, state, query_start
   FROM pg_stat_activity
   WHERE state = 'active'
   ORDER BY query_start;

3. Profile application
   node --prof server.js
   node --prof-process isolate-*.log > profile.txt

# Fix
- Add indexes to slow queries
- Optimize N+1 queries
- Scale horizontally (add more instances)
```

---

## Summary

### Maintenance Best Practices

1. **Monitoring**:
   - ✅ Set up health checks
   - ✅ Collect metrics (Prometheus)
   - ✅ Configure alerts (critical issues)
   - ✅ Use error tracking (Sentry)

2. **Database**:
   - ✅ Regular backups (daily)
   - ✅ Monitor slow queries
   - ✅ Add indexes for performance
   - ✅ Clean up old data

3. **Performance**:
   - ✅ Optimize database queries
   - ✅ Use caching (Redis)
   - ✅ Memoize expensive operations
   - ✅ Monitor Web Vitals

4. **Debugging**:
   - ✅ Enable structured logging
   - ✅ Use debugger (VS Code, Chrome)
   - ✅ Check tRPC logs
   - ✅ Inspect database queries

5. **Incident Response**:
   - ✅ Have runbook ready
   - ✅ Know how to rollback
   - ✅ Communicate with users
   - ✅ Document post-mortems

### Quick Reference Commands

```bash
# Database
npx prisma studio                    # Open database GUI
npx prisma migrate deploy            # Apply migrations
pg_dump calendso > backup.sql        # Backup database
psql calendso < backup.sql           # Restore database

# Debugging
yarn dev                             # Start with hot reload
node --inspect yarn dev              # Start with debugger
yarn type-check                      # Check TypeScript
yarn lint                            # Lint code

# Monitoring
curl http://localhost:3000/api/health  # Health check
curl http://localhost:3000/api/metrics # Prometheus metrics

# Performance
yarn build && yarn start             # Production build
npx lighthouse http://localhost:3000 # Performance audit
```

---

**Tác giả**: Claude (AI Assistant)
**Ngày tạo**: 2025-11-18
**Phiên bản**: 1.0

---

## 🎉 Hoàn thành toàn bộ 14 Phases!

Bạn vừa hoàn thành hệ thống tài liệu architecture đầy đủ cho Cal.com:

1. ✅ PHASE 0: Project Overview & Documentation Plan
2. ✅ PHASE 1: Monorepo Structure & Build Pipeline
3. ✅ PHASE 2: Data Layer (Prisma)
4. ✅ PHASE 3: tRPC API Architecture
5. ✅ PHASE 4: Next.js Web Application
6. ✅ PHASE 5: Feature Packages
7. ✅ PHASE 6: App Store & Integrations
8. ✅ PHASE 7: Booking Flow (End-to-End)
9. ✅ PHASE 8: Authentication & Authorization
10. ✅ PHASE 9: API v1 vs API v2
11. ✅ PHASE 10: Platform & Atoms (Embed/SDK)
12. ✅ PHASE 11: Enterprise Features
13. ✅ PHASE 12: DevOps, Testing & Deployment
14. ✅ PHASE 13: Coding Conventions & Best Practices
15. ✅ PHASE 14: Maintenance Guide ← Bạn đang ở đây!

**Tổng cộng**: Hơn 20,000 dòng documentation chi tiết!
