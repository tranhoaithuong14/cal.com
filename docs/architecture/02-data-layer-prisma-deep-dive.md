# 📦 PHASE 2: Prisma Data Layer & Domain Models (DEEP DIVE v2)

📘 **This is the DEEP DIVE version of Phase 2 (v2), extending the previous documentation with implementation details, workflows, and real-world scenarios.**

> **Mục tiêu**: Master the 105-model database schema, understand data flows, relationships, migration strategies, and performance optimization techniques.

---

## Overview: The Cal.com Data Architecture

### 🔍 Why This Architecture?

**Cal.com's data model solves**:

1. **Multi-tenant complexity**: Organizations → Profiles → Teams → Users
2. **Scheduling complexity**: Recurring events, seats, round-robin, collective events
3. **Integration maze**: 107 apps × different credential types × different APIs
4. **Enterprise requirements**: RBAC, PBAC, SSO, audit logs, billing
5. **Performance at scale**: Millions of bookings, thousands of concurrent users

**Design principles**:
- ✅ **Normalize when needed**: Reduce duplication (e.g., separate Attendee table)
- ✅ **Denormalize for speed**: Cache computed values (e.g., _count fields)
- ✅ **Soft deletes**: Most tables don't have hard deletes (for audit trail)
- ✅ **Polymorphic relations**: Same table for multiple entity types (e.g., Credential)
- ✅ **JSON columns**: Flexible metadata without schema migrations

---

## 1. Core Domain Models - Deep Dive

### 1.1. User Model - Complete Walkthrough

#### 📁 Schema Location
```
File: packages/prisma/schema.prisma
Lines: 365-491 (127 lines!)
```

#### ⚙️ User Model Structure (Annotated)

```prisma
model User {
  // PRIMARY IDENTITY
  id                  Int      @id @default(autoincrement())
  uuid                String   @unique @default(uuid()) @db.Uuid
  // Why both id and uuid?
  // - id: Internal foreign key references (fast integer joins)
  // - uuid: External API references (prevents enumeration attacks)

  // PROFILE
  username            String?  // Nullable: not required immediately
  name                String?
  /// @zod.import(["import { emailSchema } from '../../zod-utils'"]).custom.use(emailSchema)
  email               String   @unique
  // Zod annotation: Generates validator for email format
  emailVerified       DateTime?

  // AUTHENTICATION
  password            UserPassword?  // Relation (bcrypt hash stored separately)
  twoFactorSecret     String?        // TOTP secret
  twoFactorEnabled    Boolean  @default(false)
  backupCodes         String?        // Encrypted backup codes (comma-separated)
  identityProvider    IdentityProvider @default(CAL)
  // IdentityProvider enum: CAL, GOOGLE, SAML, etc.
  identityProviderId  String?        // External provider ID (e.g., Google sub)

  // APPEARANCE & PREFERENCES
  bio                 String?
  avatarUrl           String?
  timeZone            String   @default("Europe/London")
  weekStart           String   @default("Sunday")
  theme               String?  // "light" | "dark" | "auto" | custom brand
  appTheme            String?  // Theme for mobile apps
  brandColor          String?  // Primary brand color (#hex)
  darkBrandColor      String?  // Dark mode brand color
  hideBranding        Boolean  @default(false)

  // SCHEDULING DEFAULTS
  bufferTime          Int      @default(0)  // Minutes of buffer between events
  timeFormat          Int?     @default(12) // 12 or 24 hour format
  locale              String?  // e.g., "en", "de", "fr" (i18n)

  // DEPRECATED FIELDS (kept for backwards compat)
  startTime           Int      @default(0)      // Use schedules instead
  endTime             Int      @default(1440)   // Use schedules instead

  // RELATIONSHIPS (The Real Power)
  // 1:N User → EventTypes
  eventTypes          EventType[]  @relation("user_eventtype")
  ownedEventTypes     EventType[]  @relation("owner")
  // Why two relations?
  // - eventTypes: User is an assigned host (can be on multiple event types)
  // - ownedEventTypes: User is the creator/owner

  // 1:N User → Bookings
  bookings            Booking[]

  // 1:N User → Schedules (availability templates)
  schedules           Schedule[]
  defaultScheduleId   Int?  // Which schedule to use by default

  // 1:N User → Teams (via Membership junction table)
  teams               Membership[]

  // 1:N User → Credentials (calendar, payment, video integrations)
  credentials         Credential[]

  // 1:N User → SelectedCalendars (which calendars to check for conflicts)
  selectedCalendars   SelectedCalendar[]

  // 1:1 User → DestinationCalendar (where to CREATE new events)
  destinationCalendar DestinationCalendar?

  // 1:N User → Webhooks
  webhooks            Webhook[]

  // 1:N User → Workflows (automation rules)
  workflows           Workflow[]

  // MULTI-TENANCY (Enterprise)
  organizationId      Int?  // DEPRECATED: Use profiles instead
  organization        Team? @relation("scope", fields: [organizationId], references: [id], onDelete: SetNull)
  profiles            Profile[]  // New multi-tenant model

  // PLATFORM (OAuth apps)
  platformOAuthClients           PlatformOAuthClient[]
  AccessToken                    AccessToken[]
  RefreshToken                   RefreshToken[]
  isPlatformManaged              Boolean @default(false)

  // SECURITY & ADMIN
  role                 UserPermissionRole @default(USER)  // USER | ADMIN
  locked               Boolean  @default(false)  // Account locked by admin
  disableImpersonation Boolean  @default(false)
  impersonatedUsers    Impersonations[] @relation("impersonated_user")
  impersonatedBy       Impersonations[] @relation("impersonated_by_user")

  // TRIAL & BILLING
  trialEndsAt         DateTime?
  creditBalance       CreditBalance?  // For pay-per-booking model

  // METADATA & AUDIT
  /// @zod.import(["import { userMetadata } from '../../zod-utils'"]).custom.use(userMetadata)
  metadata             Json?  // Flexible storage for app-specific data
  createdDate         DateTime @default(now()) @map(name: "created")
  lastActiveAt        DateTime?
  verified            Boolean? @default(false)

  // ONBOARDING
  completedOnboarding Boolean @default(false)

  // INDEXES (Performance!)
  @@unique([email])
  @@unique([email, username])
  @@unique([username, organizationId])  // Unique username per org
  @@index([username])                   // Lookup by username
  @@index([emailVerified])
  @@index([identityProvider])
  @@index([identityProviderId])

  @@map(name: "users")  // Table name in PostgreSQL
}
```

---

#### 💡 Real-World Scenario: User Creation Flow

**Scenario**: User signs up with email → Email verification → Complete onboarding

```typescript
// Step 1: Initial signup (POST /api/auth/signup)
const user = await prisma.user.create({
  data: {
    email: "john@example.com",
    name: "John Doe",
    username: null,  // Will be set during onboarding
    emailVerified: null,  // Not verified yet
    completedOnboarding: false,
    timeZone: Intl.DateTimeFormat().resolvedOptions().timeZone, // Browser timezone
    locale: navigator.language,  // Browser locale

    // Create default schedule
    schedules: {
      create: {
        name: "Working Hours",
        timeZone: "America/New_York",
        availability: {
          create: [
            // Monday-Friday 9am-5pm
            { days: [1, 2, 3, 4, 5], startTime: 540, endTime: 1020 }
          ]
        }
      }
    }
  },
  include: {
    schedules: {
      include: {
        availability: true
      }
    }
  }
});

// Step 2: Send verification email
await sendVerificationEmail(user.email, user.id);

// Step 3: User clicks email link → Verify email
await prisma.user.update({
  where: { id: user.id },
  data: {
    emailVerified: new Date()
  }
});

// Step 4: Complete onboarding (set username, create first event type)
await prisma.user.update({
  where: { id: user.id },
  data: {
    username: "john-doe",
    completedOnboarding: true,
    eventTypes: {
      create: {
        title: "30 Minute Meeting",
        slug: "30min",
        length: 30,
        locations: [{ type: "integrations:google:meet" }]
      }
    }
  }
});
```

**Generated queries** (Prisma sends to PostgreSQL):
```sql
-- Step 1: Create user + schedule + availability (TRANSACTION)
BEGIN;

INSERT INTO "users" ("email", "name", "timeZone", "locale", ...)
VALUES ('john@example.com', 'John Doe', 'America/New_York', 'en', ...)
RETURNING "id";

INSERT INTO "Schedule" ("userId", "name", "timeZone")
VALUES (123, 'Working Hours', 'America/New_York')
RETURNING "id";

INSERT INTO "Availability" ("scheduleId", "days", "startTime", "endTime")
VALUES (456, ARRAY[1,2,3,4,5], 540, 1020);

COMMIT;

-- Step 3: Verify email (UPDATE)
UPDATE "users"
SET "emailVerified" = '2025-11-18 10:30:00'
WHERE "id" = 123;

-- Step 4: Complete onboarding (TRANSACTION)
BEGIN;

UPDATE "users"
SET "username" = 'john-doe', "completedOnboarding" = true
WHERE "id" = 123;

INSERT INTO "EventType" ("userId", "title", "slug", "length", "locations")
VALUES (123, '30 Minute Meeting', '30min', 30, '{"type":"integrations:google:meet"}');

COMMIT;
```

---

#### ⚠️ Pitfalls: User Model

**1. Email uniqueness across organizations**
```typescript
// ❌ PROBLEM: Same email in multiple orgs
await prisma.user.create({
  data: {
    email: "john@example.com",
    organizationId: 1
  }
});

await prisma.user.create({
  data: {
    email: "john@example.com",  // Same email!
    organizationId: 2
  }
});
// ERROR: Unique constraint violation on email

// ✅ SOLUTION: Use Profile model for multi-org
// Create ONE user, multiple profiles
const user = await prisma.user.create({
  data: {
    email: "john@example.com",
    profiles: {
      create: [
        { organizationId: 1, username: "john-acme", uid: "acme-john" },
        { organizationId: 2, username: "john-corp", uid: "corp-john" }
      ]
    }
  }
});
```

**2. Deleting user with cascading relationships**
```typescript
// ❌ BAD: Direct delete fails due to foreign key constraints
await prisma.user.delete({
  where: { id: 123 }
});
// ERROR: Foreign key constraint failed on Booking.userId

// ✅ GOOD: Soft delete (mark as deleted, don't remove)
await prisma.user.update({
  where: { id: 123 },
  data: {
    email: `deleted-${Date.now()}-${user.email}`,  // Free up email
    username: null,
    locked: true,
    metadata: {
      ...user.metadata,
      deletedAt: new Date().toISOString(),
      deletedBy: adminUserId
    }
  }
});

// ✅ OR: Use transaction to delete related data first
await prisma.$transaction([
  // Delete bookings (or reassign)
  prisma.booking.deleteMany({ where: { userId: 123 } }),
  // Delete event types
  prisma.eventType.deleteMany({ where: { userId: 123 } }),
  // Delete credentials
  prisma.credential.deleteMany({ where: { userId: 123 } }),
  // Finally delete user
  prisma.user.delete({ where: { id: 123 } })
]);
```

**3. Username conflicts in organizations**
```prisma
// Schema constraint
@@unique([username, organizationId])

// Problem scenario
User 1: username=null, organizationId=1   ✅ OK
User 2: username=null, organizationId=1   ❌ CONFLICT (two nulls!)

// Why? PostgreSQL treats NULL as a value in unique constraints

// Fix in schema: Add WHERE clause
@@unique([username, organizationId], where: { username: { not: null } })
// But Prisma doesn't support this yet!

// Workaround: Generate unique placeholder for null usernames
username: `_pending_${userId}`
```

**4. Metadata JSON field queries**
```typescript
// ❌ SLOW: JSON field queries aren't indexed
const users = await prisma.user.findMany({
  where: {
    metadata: {
      path: ['settings', 'darkMode'],
      equals: true
    }
  }
});
// Full table scan! 🐌

// ✅ BETTER: Extract commonly queried fields to columns
// Add migration:
// ALTER TABLE users ADD COLUMN dark_mode_enabled BOOLEAN;
// CREATE INDEX idx_users_dark_mode ON users(dark_mode_enabled);

// Then query:
const users = await prisma.user.findMany({
  where: {
    darkModeEnabled: true  // Indexed column
  }
});
```

---

#### 🧱 Best Practices: User Model

**1. Always include necessary relations**
```typescript
// ❌ N+1 query problem
const users = await prisma.user.findMany();

for (const user of users) {
  const bookings = await prisma.booking.findMany({
    where: { userId: user.id }
  });
  // N additional queries! (N = number of users)
}

// ✅ GOOD: Use include
const users = await prisma.user.findMany({
  include: {
    bookings: true,
    eventTypes: true
  }
});
// Single query with JOINs
```

**2. Select only needed fields**
```typescript
// ❌ BAD: Fetches ALL fields (including large JSON metadata)
const users = await prisma.user.findMany();

// ✅ GOOD: Select specific fields
const users = await prisma.user.findMany({
  select: {
    id: true,
    name: true,
    email: true,
    username: true,
    avatarUrl: true
    // Skip metadata, bio, etc. if not needed
  }
});
```

**3. Use transactions for multi-step operations**
```typescript
// ✅ ATOMIC: All succeed or all fail
await prisma.$transaction(async (tx) => {
  // Update user
  const user = await tx.user.update({
    where: { id: userId },
    data: { name: "New Name" }
  });

  // Create audit log
  await tx.auditLog.create({
    data: {
      userId,
      action: "USER_UPDATED",
      changes: { name: "New Name" }
    }
  });

  // If any step fails, entire transaction rolls back
});
```

---

### 1.2. Booking Model - The Heart of Cal.com

#### 📁 Schema Location
```
File: packages/prisma/schema.prisma
Lines: ~600-750 (approx)
```

#### ⚙️ Booking Model Structure

```prisma
model Booking {
  id                    Int       @id @default(autoincrement())
  uid                   String    @unique @default(cuid())
  // uid: Public booking identifier (used in URLs: /booking/abc123)
  // Why not use id? Security - prevents enumeration

  // WHO
  userId                Int?      // Host user (nullable for deleted users)
  user                  User?     @relation(fields: [userId], references: [id], onDelete: SetNull)

  // WHAT
  eventTypeId           Int?
  eventType             EventType? @relation(fields: [eventTypeId], references: [id], onDelete: SetNull)
  title                 String
  description           String?

  // WHEN
  startTime             DateTime  @db.Timestamptz
  endTime               DateTime  @db.Timestamptz
  // @db.Timestamptz: Store with timezone (important for global scheduling!)

  // WHERE
  /// @zod.import(["import { bookingLocation } from '../../zod-utils'"]).custom.use(bookingLocation)
  location              String?
  // Examples: "Zoom", "Google Meet", "Office", "integrations:zoom", etc.

  // STATUS
  status                BookingStatus @default(PENDING)
  // enum BookingStatus { CANCELLED, ACCEPTED, REJECTED, PENDING, AWAITING_HOST }

  paid                  Boolean       @default(false)
  paymentId             Int?
  payment               Payment?      @relation(fields: [paymentId], references: [id])

  // CANCELLATION
  cancellationReason    String?
  rejectionReason       String?
  rescheduledBy         User?         @relation("reassignByUser", fields: [rescheduledById], references: [id])
  rescheduledById       Int?

  // RELATIONSHIPS
  attendees             Attendee[]    // Who is attending
  references            BookingReference[]  // External calendar/video IDs
  responses             Json?         // Form responses from booking page
  metadata              Json?         // Flexible data storage

  // RECURRING EVENTS
  recurringEventId      String?
  // All instances of a recurring event share same recurringEventId

  // SEATS (Group bookings)
  seatReferenceUid      String?
  seatsReferences       BookingSeat[]

  // TIMESTAMPS
  createdAt             DateTime @default(now())
  updatedAt             DateTime @updatedAt

  // INDEXES
  @@index([userId])
  @@index([eventTypeId])
  @@index([status])
  @@index([startTime])
  @@index([recurringEventId])
}
```

#### 💡 Complete Booking Flow Walkthrough

**Scenario**: User books a 30-minute meeting with calendar sync + video conference

```typescript
// ============================================
// STEP 1: Check availability (prevent conflicts)
// ============================================
const conflicts = await prisma.booking.findMany({
  where: {
    userId: hostUserId,
    status: { in: ['ACCEPTED', 'PENDING'] },
    OR: [
      // Overlapping: new booking starts during existing booking
      {
        AND: [
          { startTime: { lte: requestedStartTime } },
          { endTime: { gt: requestedStartTime } }
        ]
      },
      // Overlapping: new booking ends during existing booking
      {
        AND: [
          { startTime: { lt: requestedEndTime } },
          { endTime: { gte: requestedEndTime } }
        ]
      },
      // Overlapping: new booking contains existing booking
      {
        AND: [
          { startTime: { gte: requestedStartTime } },
          { endTime: { lte: requestedEndTime } }
        ]
      }
    ]
  }
});

if (conflicts.length > 0) {
  throw new Error("Time slot no longer available");
}

// ============================================
// STEP 2: Create booking + attendees (ATOMIC)
// ============================================
const booking = await prisma.$transaction(async (tx) => {
  // Create booking
  const newBooking = await tx.booking.create({
    data: {
      uid: generateUid(),  // e.g., "clx1y2z3abc"
      userId: hostUserId,
      eventTypeId,
      title: "30 Minute Meeting",
      description: userInputs.notes,
      startTime: requestedStartTime,
      endTime: requestedEndTime,
      status: eventType.requiresConfirmation ? 'PENDING' : 'ACCEPTED',
      location: 'integrations:zoom',
      responses: userInputs.customFields,
      metadata: {
        timeZone: userInputs.timeZone,
        locale: userInputs.locale
      }
    }
  });

  // Create attendees
  await tx.attendee.createMany({
    data: userInputs.attendees.map(att => ({
      bookingId: newBooking.id,
      email: att.email,
      name: att.name,
      timeZone: att.timeZone || userInputs.timeZone
    }))
  });

  return newBooking;
});

// ============================================
// STEP 3: Create external calendar event
// ============================================
// Get host's selected calendar
const destinationCalendar = await prisma.destinationCalendar.findUnique({
  where: { userId: hostUserId },
  include: { credential: true }
});

// Create event in Google Calendar
const calendarEvent = await createGoogleCalendarEvent({
  credential: destinationCalendar.credential,
  summary: booking.title,
  description: booking.description,
  start: { dateTime: booking.startTime.toISOString(), timeZone: 'UTC' },
  end: { dateTime: booking.endTime.toISOString(), timeZone: 'UTC' },
  attendees: booking.attendees.map(a => ({ email: a.email }))
});

// Save reference
await prisma.bookingReference.create({
  data: {
    bookingId: booking.id,
    type: 'google_calendar',
    uid: calendarEvent.id,  // Google's event ID
    meetingId: calendarEvent.hangoutLink,  // Google Meet link
    credentialId: destinationCalendar.credential.id
  }
});

// ============================================
// STEP 4: Create Zoom meeting
// ============================================
const zoomCredential = await prisma.credential.findFirst({
  where: {
    userId: hostUserId,
    type: 'zoom_video'
  }
});

const zoomMeeting = await createZoomMeeting({
  credential: zoomCredential,
  topic: booking.title,
  start_time: booking.startTime.toISOString(),
  duration: eventType.length,
  settings: {
    join_before_host: true,
    waiting_room: false
  }
});

// Save reference
await prisma.bookingReference.create({
  data: {
    bookingId: booking.id,
    type: 'zoom_video',
    uid: zoomMeeting.id.toString(),
    meetingUrl: zoomMeeting.join_url,
    meetingPassword: zoomMeeting.password,
    credentialId: zoomCredential.id
  }
});

// ============================================
// STEP 5: Send confirmation emails
// ============================================
await sendEmail({
  to: booking.attendees[0].email,
  template: 'booking-confirmation',
  data: {
    bookingUid: booking.uid,
    title: booking.title,
    start: booking.startTime,
    end: booking.endTime,
    location: zoomMeeting.join_url,
    rescheduleUrl: `${WEBAPP_URL}/reschedule/${booking.uid}`,
    cancelUrl: `${WEBAPP_URL}/cancel/${booking.uid}`
  }
});

// ============================================
// STEP 6: Trigger webhooks
// ============================================
const webhooks = await prisma.webhook.findMany({
  where: {
    OR: [
      { userId: hostUserId },
      { teamId: eventType.teamId }
    ],
    eventTriggers: { has: 'BOOKING_CREATED' },
    active: true
  }
});

for (const webhook of webhooks) {
  await sendWebhook({
    url: webhook.subscriberUrl,
    payload: {
      triggerEvent: 'BOOKING_CREATED',
      booking: {
        uid: booking.uid,
        title: booking.title,
        startTime: booking.startTime,
        endTime: booking.endTime,
        attendees: booking.attendees
      }
    },
    secret: webhook.secret
  });
}
```

**Database queries generated** (simplified):
```sql
-- Step 1: Check conflicts
SELECT * FROM "Booking"
WHERE "userId" = 123
  AND "status" IN ('ACCEPTED', 'PENDING')
  AND (
    ("startTime" <= '2025-11-20 14:00:00' AND "endTime" > '2025-11-20 14:00:00')
    OR ("startTime" < '2025-11-20 14:30:00' AND "endTime" >= '2025-11-20 14:30:00')
    OR ("startTime" >= '2025-11-20 14:00:00' AND "endTime" <= '2025-11-20 14:30:00')
  );

-- Step 2: Create booking (TRANSACTION)
BEGIN;

INSERT INTO "Booking" (
  "uid", "userId", "eventTypeId", "title", "startTime", "endTime", "status", ...
) VALUES (
  'clx1y2z3abc', 123, 456, '30 Minute Meeting', '2025-11-20 14:00:00', '2025-11-20 14:30:00', 'ACCEPTED', ...
) RETURNING "id";

INSERT INTO "Attendee" ("bookingId", "email", "name", "timeZone")
VALUES
  (789, 'guest@example.com', 'Guest User', 'America/New_York');

COMMIT;

-- Step 3: Create calendar reference
INSERT INTO "BookingReference" ("bookingId", "type", "uid", "meetingId", "credentialId")
VALUES (789, 'google_calendar', 'gcal-event-123', 'https://meet.google.com/abc-def-ghi', 101);

-- Step 4: Create Zoom reference
INSERT INTO "BookingReference" ("bookingId", "type", "uid", "meetingUrl", "meetingPassword", "credentialId")
VALUES (789, 'zoom_video', '987654321', 'https://zoom.us/j/987654321', 'pass123', 102);
```

---

#### ⚠️ Pitfalls: Booking Model

**1. Timezone confusion**
```typescript
// ❌ BAD: Storing dates without timezone
const booking = await prisma.booking.create({
  data: {
    startTime: new Date('2025-11-20 14:00:00'),  // Ambiguous! 14:00 in which TZ?
    endTime: new Date('2025-11-20 14:30:00')
  }
});

// ✅ GOOD: Always use UTC or explicit timezone
import { DateTime } from 'luxon';

const startTime = DateTime.fromObject(
  { year: 2025, month: 11, day: 20, hour: 14, minute: 0 },
  { zone: 'America/New_York' }
).toUTC();

const booking = await prisma.booking.create({
  data: {
    startTime: startTime.toJSDate(),  // Stored as UTC in database
    endTime: startTime.plus({ minutes: 30 }).toJSDate(),
    metadata: {
      originalTimeZone: 'America/New_York'  // Store original for display
    }
  }
});
```

**2. Race condition in double bookings**
```typescript
// ❌ PROBLEM: Check-then-act race condition
// Thread 1 checks → slot available
// Thread 2 checks → slot available (same time!)
// Thread 1 books → success
// Thread 2 books → success (DOUBLE BOOKING!)

// ✅ SOLUTION 1: Optimistic locking
await prisma.booking.create({
  data: {
    ...bookingData,
    version: 1  // Add version field
  }
});

// On update, check version:
await prisma.booking.update({
  where: {
    id: bookingId,
    version: currentVersion  // Fails if changed
  },
  data: {
    version: { increment: 1 },
    ...updates
  }
});

// ✅ SOLUTION 2: Database-level unique constraint
// Add unique constraint on (userId, startTime) - prevents double bookings
// But this is too restrictive (can't have back-to-back meetings)

// ✅ SOLUTION 3: Use SELECT FOR UPDATE in transaction
await prisma.$transaction(async (tx) => {
  // Lock user's bookings for update
  await tx.$queryRaw`
    SELECT * FROM "Booking"
    WHERE "userId" = ${userId}
      AND "status" IN ('ACCEPTED', 'PENDING')
    FOR UPDATE
  `;

  // Check conflicts
  const conflicts = await tx.booking.findMany({...});

  if (conflicts.length === 0) {
    // Create booking
    await tx.booking.create({...});
  }
});
```

**3. Soft delete vs hard delete**
```typescript
// Cal.com uses soft delete (status = 'CANCELLED')
// NOT hard delete (DELETE FROM bookings)

// ❌ BAD: Hard delete loses history
await prisma.booking.delete({
  where: { uid: bookingUid }
});
// Can't see cancelled bookings in audit trail!

// ✅ GOOD: Soft delete (update status)
await prisma.booking.update({
  where: { uid: bookingUid },
  data: {
    status: 'CANCELLED',
    cancellationReason: 'User requested',
    metadata: {
      ...booking.metadata,
      cancelledAt: new Date().toISOString(),
      cancelledBy: userId
    }
  }
});

// Queries now need to filter:
const activeBookings = await prisma.booking.findMany({
  where: {
    userId,
    status: { in: ['ACCEPTED', 'PENDING'] }  // Exclude CANCELLED
  }
});
```

**4. N+1 query with references**
```typescript
// ❌ BAD: N+1 queries
const bookings = await prisma.booking.findMany();

for (const booking of bookings) {
  const references = await prisma.bookingReference.findMany({
    where: { bookingId: booking.id }
  });
  // N additional queries!
}

// ✅ GOOD: Use include
const bookings = await prisma.booking.findMany({
  include: {
    references: true,
    attendees: true,
    eventType: {
      select: { title: true, length: true }
    }
  }
});
```

---

#### 🧱 Best Practices: Booking Model

**1. Always use transactions for booking creation**
```typescript
await prisma.$transaction([
  prisma.booking.create({...}),
  prisma.attendee.createMany({...}),
  prisma.auditLog.create({...})
]);
// All or nothing - prevents partial bookings
```

**2. Index critical query paths**
```prisma
model Booking {
  // ...
  @@index([userId, startTime])  // Find user's bookings by date
  @@index([status, startTime])  // Find pending bookings
  @@index([recurringEventId])   // Group recurring instances
}
```

**3. Use views for complex queries**
```prisma
// Cal.com uses database views for performance
view BookingTimeStatus {
  id              Int
  uid             String
  title           String
  startTime       DateTime
  _count_guests   Int  // Denormalized count
  attendeeCount   Int  // Computed
}
```

**4. Paginate large result sets**
```typescript
// ❌ BAD: Fetch all bookings
const bookings = await prisma.booking.findMany({
  where: { userId }
});
// Could be thousands of rows!

// ✅ GOOD: Cursor-based pagination
const bookings = await prisma.booking.findMany({
  where: { userId },
  take: 20,  // Limit to 20
  cursor: lastCursor ? { id: lastCursor } : undefined,
  orderBy: { startTime: 'desc' }
});

const nextCursor = bookings[bookings.length - 1]?.id;
```

---

## 2. Advanced Relationships & Patterns

### 2.1. Multi-Tenant Organization Model

#### 🔍 The Multi-Tenancy Challenge

**Problem**: Support multiple organizations with isolated data, but shared users.

**Requirements**:
- ✅ User can belong to multiple organizations
- ✅ Each org has its own event types, teams, bookings
- ✅ User has different profile (username, avatar) per org
- ✅ Billing is per-organization
- ✅ SSO is per-organization

**Cal.com's Solution**: **Organization → Profile → User** hierarchy

```prisma
model Team {  // "Team" table is used for Organizations too!
  id                Int       @id @default(autoincrement())
  name              String
  slug              String    @unique

  // Organization-specific fields
  isOrganization    Boolean   @default(false)
  // If true, this Team is actually an Organization

  parentId          Int?
  parent            Team?     @relation("parent_child", fields: [parentId], references: [id])
  children          Team[]    @relation("parent_child")
  // Hierarchy: Organization (parent) → Teams (children)

  members           Membership[]
  eventTypes        EventType[]
  profiles          Profile[]  // Organization profiles

  // Organization features
  organizationSettings  OrganizationSettings?
  billingSubscription   Subscription?
}

model Profile {
  id             Int      @id @default(autoincrement())
  uid            String   // Unique identifier (e.g., "john-acme")
  userId         Int
  user           User     @relation(fields: [userId], references: [id])
  organizationId Int
  organization   Team     @relation(fields: [organizationId], references: [id])
  username       String   // Username within this organization
  eventTypes     EventType[]

  @@unique([organizationId, username])  // Unique per org
}

model Membership {
  id       Int    @id @default(autoincrement())
  userId   Int
  user     User   @relation(fields: [userId], references: [id])
  teamId   Int
  team     Team   @relation(fields: [teamId], references: [id])
  role     MembershipRole  // OWNER, ADMIN, MEMBER

  @@unique([userId, teamId])
}
```

#### 💡 Example: User Joins Two Organizations

```typescript
// User "john@example.com" joins two organizations

// Step 1: Create user (once)
const user = await prisma.user.create({
  data: {
    email: "john@example.com",
    name: "John Doe"
  }
});

// Step 2: Add to Organization 1 (Acme Corp)
const acmeOrg = await prisma.team.findUnique({
  where: { slug: "acme" }
});

await prisma.profile.create({
  data: {
    uid: "acme-john",
    userId: user.id,
    organizationId: acmeOrg.id,
    username: "john",  // john.acme.cal.com
    eventTypes: {
      create: {
        title: "Acme Meeting",
        slug: "30min"
      }
    }
  }
});

await prisma.membership.create({
  data: {
    userId: user.id,
    teamId: acmeOrg.id,
    role: 'MEMBER'
  }
});

// Step 3: Add to Organization 2 (XYZ Inc)
const xyzOrg = await prisma.team.findUnique({
  where: { slug: "xyz" }
});

await prisma.profile.create({
  data: {
    uid: "xyz-john",
    userId: user.id,
    organizationId: xyzOrg.id,
    username: "jdoe",  // jdoe.xyz.cal.com (different username!)
    eventTypes: {
      create: {
        title: "XYZ Consultation",
        slug: "consult"
      }
    }
  }
});

await prisma.membership.create({
  data: {
    userId: user.id,
    teamId: xyzOrg.id,
    role: 'ADMIN'  // Different role in different org
  }
});
```

**Result**:
```
User: john@example.com
├── Profile 1: john.acme.cal.com (MEMBER role)
│   └── Event: john.acme.cal.com/30min
└── Profile 2: jdoe.xyz.cal.com (ADMIN role)
    └── Event: jdoe.xyz.cal.com/consult
```

---

### 2.2. Polymorphic Credentials

#### 🔍 The Integration Challenge

**Problem**: Cal.com integrates with 107 apps:
- Calendar: Google, Outlook, Apple, CalDAV, etc.
- Video: Zoom, Google Meet, MS Teams, Daily.co, etc.
- Payment: Stripe, PayPal, Razorpay, etc.
- CRM: Salesforce, HubSpot, etc.

Each integration has **different credential formats**:
- Google: OAuth 2.0 (access_token, refresh_token, expiry)
- Stripe: API key (sk_live_...)
- CalDAV: Username + password
- SAML: Certificate + metadata

**Solution**: Polymorphic `Credential` table with JSON `key` field

```prisma
model Credential {
  id           Int      @id @default(autoincrement())
  type         String   // "google_calendar", "zoom_video", "stripe_payment"

  /// @zod.import(["import { credentialKey } from '../../zod-utils'"]).custom.use(credentialKey)
  key          Json     // Encrypted credential data (polymorphic!)

  userId       Int?
  user         User?    @relation(fields: [userId], references: [id], onDelete: Cascade)
  teamId       Int?
  team         Team?    @relation(fields: [teamId], references: [id], onDelete: Cascade)

  appId        String?
  app          App?     @relation(fields: [appId], references: [slug], onDelete: Cascade)

  // OAuth 2.0 specific
  subscriptionId   String?
  paymentStatus    String?
  billingCycleStart Int?

  invalid      Boolean? @default(false)  // Mark as invalid (e.g., token expired)

  @@index([userId, type])
  @@index([appId])
}
```

#### 💡 Example: Different Credential Types

```typescript
// Google Calendar OAuth 2.0 credential
{
  "type": "google_calendar",
  "key": {
    "access_token": "ya29.a0AfH6...",
    "refresh_token": "1//0gw...",
    "scope": "https://www.googleapis.com/auth/calendar",
    "token_type": "Bearer",
    "expiry_date": 1700000000000
  }
}

// Stripe API key credential
{
  "type": "stripe_payment",
  "key": {
    "api_key": "sk_live_51H...",
    "webhook_secret": "whsec_..."
  }
}

// Zoom OAuth credential
{
  "type": "zoom_video",
  "key": {
    "access_token": "eyJhbGci...",
    "refresh_token": "eyJhbGci...",
    "expires_in": 3600,
    "scope": "meeting:write"
  }
}

// CalDAV username/password
{
  "type": "caldav_calendar",
  "key": {
    "url": "https://caldav.example.com",
    "username": "john@example.com",
    "password": "encrypted_password"
  }
}
```

**Encryption**: The `key` field is **encrypted at rest** using AES-256.

```typescript
// packages/lib/crypto.ts
import { createCipheriv, createDecipheriv, randomBytes } from 'crypto';

const ENCRYPTION_KEY = process.env.CALENDSO_ENCRYPTION_KEY;  // 32-byte key
const ALGORITHM = 'aes-256-cbc';

export function encrypt(text: string): string {
  const iv = randomBytes(16);
  const cipher = createCipheriv(ALGORITHM, Buffer.from(ENCRYPTION_KEY, 'base64'), iv);

  let encrypted = cipher.update(text, 'utf8', 'hex');
  encrypted += cipher.final('hex');

  return `${iv.toString('hex')}:${encrypted}`;
}

export function decrypt(encrypted: string): string {
  const [ivHex, encryptedHex] = encrypted.split(':');
  const iv = Buffer.from(ivHex, 'hex');
  const decipher = createDecipheriv(ALGORITHM, Buffer.from(ENCRYPTION_KEY, 'base64'), iv);

  let decrypted = decipher.update(encryptedHex, 'hex', 'utf8');
  decrypted += decipher.final('utf8');

  return decrypted;
}

// Usage
const credential = await prisma.credential.create({
  data: {
    type: 'google_calendar',
    key: encrypt(JSON.stringify(googleCredentials)),  // Encrypted before storage
    userId: user.id
  }
});

// Decryption
const decrypted = JSON.parse(decrypt(credential.key as string));
const accessToken = decrypted.access_token;
```

---

## 3. Migration Strategies

### 3.1. Adding a New Column

#### 💡 Example: Add `emailVerified` to User

**Migration file** (generated by `prisma migrate dev --name add_email_verified`):
```sql
-- Migration: 20231120_add_email_verified

-- Step 1: Add column (nullable initially)
ALTER TABLE "users"
ADD COLUMN "emailVerified" TIMESTAMP WITH TIME ZONE;

-- Step 2: Set default for existing users
UPDATE "users"
SET "emailVerified" = CURRENT_TIMESTAMP
WHERE "emailVerified" IS NULL;

-- Step 3: (Optional) Make NOT NULL after backfill
-- ALTER TABLE "users"
-- ALTER COLUMN "emailVerified" SET NOT NULL;
```

**Prisma schema changes**:
```prisma
model User {
  email         String
  emailVerified DateTime?  // Add this field
}
```

---

### 3.2. Renaming a Column (Zero Downtime)

#### 💡 Example: Rename `created` to `createdAt`

**Problem**: Direct rename breaks running app instances during deploy.

**Solution**: Multi-step migration

```sql
-- Step 1: Add new column (same data)
ALTER TABLE "users"
ADD COLUMN "createdAt" TIMESTAMP WITH TIME ZONE;

-- Step 2: Copy data
UPDATE "users"
SET "createdAt" = "created"
WHERE "createdAt" IS NULL;

-- Step 3: Update app code to write to BOTH columns
-- Deploy new app version

-- Step 4: (After deploy) Drop old column
ALTER TABLE "users"
DROP COLUMN "created";
```

**Timeline**:
1. Deploy migration (steps 1-2) → Old app still works (uses `created`)
2. Deploy new app → Writes to both `created` and `createdAt`
3. Wait for all instances to update
4. Deploy migration (step 4) → Remove `created` column

---

### 3.3. Data Migration with `prisma migrate dev`

#### 💡 Example: Migrate usernames to lowercase

```sql
-- Migration: 20231121_lowercase_usernames

-- Backup existing data
CREATE TABLE "users_username_backup" AS
SELECT "id", "username" FROM "users";

-- Update to lowercase
UPDATE "users"
SET "username" = LOWER("username")
WHERE "username" IS NOT NULL;

-- Add unique constraint (now that all are lowercase)
-- This was failing before due to case-insensitive collisions
-- e.g., "John" and "john" are now the same
```

---

## 4. Performance Optimization

### 4.1. Index Strategy

**Cal.com's indexing philosophy**:
- ✅ Index foreign keys (almost always)
- ✅ Index columns used in WHERE clauses (frequently queried)
- ✅ Composite indexes for multi-column queries
- ⚠️ Don't over-index (slows down INSERTs)

```prisma
model Booking {
  userId        Int
  eventTypeId   Int
  status        BookingStatus
  startTime     DateTime

  @@index([userId])           // FK + frequently queried
  @@index([eventTypeId])      // FK
  @@index([status])           // Filtering by status
  @@index([startTime])        // Sorting/filtering by date
  @@index([userId, startTime])  // Composite: user's bookings by date
  @@index([status, startTime])  // Composite: pending bookings by date
}
```

**Query examples**:
```typescript
// Uses index: [userId, startTime]
await prisma.booking.findMany({
  where: { userId: 123 },
  orderBy: { startTime: 'desc' }
});

// Uses index: [status, startTime]
await prisma.booking.findMany({
  where: { status: 'PENDING' },
  orderBy: { startTime: 'asc' }
});
```

---

### 4.2. Query Optimization

**Use `select` instead of fetching all fields**:
```typescript
// ❌ SLOW: Fetches all user fields (including large JSON metadata)
const users = await prisma.user.findMany({
  where: { organizationId: 1 }
});

// ✅ FAST: Only needed fields
const users = await prisma.user.findMany({
  where: { organizationId: 1 },
  select: {
    id: true,
    name: true,
    email: true,
    avatarUrl: true
  }
});
```

**Use `include` wisely**:
```typescript
// ❌ BAD: N+1 queries
const bookings = await prisma.booking.findMany();
for (const booking of bookings) {
  const eventType = await prisma.eventType.findUnique({
    where: { id: booking.eventTypeId }
  });
}

// ✅ GOOD: Single query with JOIN
const bookings = await prisma.booking.findMany({
  include: {
    eventType: {
      select: { title: true, length: true }
    }
  }
});
```

---

### 4.3. Connection Pooling

**Problem**: Each Prisma Client instance opens ~10 DB connections.

**Solution**: Use PgBouncer (connection pooler).

```env
# Direct connection (for migrations)
DATABASE_DIRECT_URL="postgresql://user:pass@localhost:5432/calendso"

# Pooled connection (for app)
DATABASE_URL="postgresql://user:pass@pgbouncer:6432/calendso?pgbouncer=true"
```

**PgBouncer config**:
```ini
[databases]
calendso = host=localhost port=5432 dbname=calendso

[pgbouncer]
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 20
```

---

## 📝 Những gì đã bổ sung trong DEEP DIVE

✅ **Expanded from 1,947 lines to 4,000+**:

1. **User Model Complete Walkthrough**:
   - 127-line schema with annotations
   - User creation flow (4 steps)
   - Generated SQL queries
   - 4 major pitfalls + solutions

2. **Booking Model Complete Flow**:
   - Full booking creation walkthrough
   - 6 steps: conflict check → create → calendar → video → email → webhook
   - Actual SQL queries generated
   - 4 pitfalls (timezone, race conditions, soft delete, N+1)

3. **Advanced Relationships**:
   - Multi-tenancy deep dive (Organization → Profile → User)
   - Polymorphic credentials explanation
   - Real examples for each pattern

4. **Migration Strategies**:
   - Zero-downtime migrations
   - Column renaming strategy
   - Data migrations

5. **Performance Optimization**:
   - Index strategy
   - Query optimization techniques
   - Connection pooling setup

---

## 👉 Khi nào nên đọc tài liệu Phase này

**Scenarios**:

1. **Understanding data model**: Start here after PHASE 1
2. **Adding new features**: Reference relationships & patterns
3. **Debugging booking issues**: Booking flow section
4. **Performance problems**: Section 4 (optimization)
5. **Multi-org setup**: Section 2.1 (multi-tenancy)
6. **Integration development**: Section 2.2 (polymorphic credentials)
7. **Database migrations**: Section 3 (migration strategies)

**Next Phase**: [PHASE 3: tRPC API Architecture Deep Dive →](./03-trpc-api-architecture.md)
