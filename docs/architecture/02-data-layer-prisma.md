# 📦 PHASE 2: Prisma Data Layer & Domain Models

> **Mục tiêu**: Hiểu sâu về database schema, 105 Prisma models, relationships, và domain models của cal.com.

---

## 📑 Mục lục

1. [Tổng quan Database Schema](#1-tổng-quan-database-schema)
2. [Generators & Generated Artifacts](#2-generators--generated-artifacts)
3. [Core Domain Models](#3-core-domain-models)
4. [Scheduling Domain](#4-scheduling-domain)
5. [Integration Domain](#5-integration-domain)
6. [Enterprise Features](#6-enterprise-features)
7. [Workflow & Automation](#7-workflow--automation)
8. [Payment & Billing](#8-payment--billing)
9. [Platform & OAuth](#9-platform--oauth)
10. [Calendar Sync & Caching](#10-calendar-sync--caching)
11. [Enums Reference](#11-enums-reference)
12. [Database Indexes & Performance](#12-database-indexes--performance)
13. [Best Practices](#13-best-practices)

---

## 1. Tổng quan Database Schema

### 1.1. Stats

**File**: `packages/prisma/schema.prisma` (2780 dòng)

```
📊 Database Stats:
- 105 models
- 25+ enums
- 2 views (BookingTimeStatus, BookingTimeStatusDenormalized)
- 1 datasource (PostgreSQL)
- 4 generators (Prisma Client, Zod, Kysely, Enums)
```

### 1.2. Datasource Configuration

```prisma
datasource db {
  provider  = "postgresql"
  url       = env("DATABASE_URL")
  directUrl = env("DATABASE_DIRECT_URL")
}
```

**Connection modes**:
- `DATABASE_URL`: Connection pooler (e.g., PgBouncer, Supabase pooler)
- `DATABASE_DIRECT_URL`: Direct PostgreSQL connection (for migrations)

---

## 2. Generators & Generated Artifacts

### 2.1. Prisma Client Generator

```prisma
generator client {
  provider            = "prisma-client"
  previewFeatures     = ["views"]
  output              = "./generated/prisma"
  engineType          = "client"
  moduleFormat        = "cjs"
  importFileExtension = ""
}
```

**Output**: `packages/prisma/generated/prisma/`
- Prisma Client với type-safe query API
- Support cho database views
- CommonJS module format

**Usage**:
```typescript
import { PrismaClient } from '@calcom/prisma';

const prisma = new PrismaClient();

// Type-safe queries
const user = await prisma.user.findUnique({
  where: { email: 'user@example.com' },
  include: { teams: true, bookings: true }
});
```

---

### 2.2. Zod Generator

```prisma
generator zod {
  provider                 = "zod-prisma-types"
  output                   = "./zod"
  useMultipleFiles         = true
  createInputTypes         = false
  addIncludeType           = false
  addSelectType            = false
  validateWhereUniqueInput = false
  prismaClientPath         = "../../generated/prisma/client"
  moduleFormat             = "cjs"
}
```

**Output**: `packages/prisma/zod/`
- Zod schemas cho mỗi model
- Validation cho inputs
- Type inference từ Zod schemas

**Generated files** (examples):
```
zod/
├── user.ts         # User model schema
├── booking.ts      # Booking model schema
├── eventtype.ts    # EventType model schema
└── ...
```

**Usage**:
```typescript
import { userSchema } from '@calcom/prisma/zod';

// Validate user input
const result = userSchema.safeParse(inputData);

if (result.success) {
  const validUser = result.data;
} else {
  console.error(result.error);
}
```

**Custom Zod validators** (trong schema):
```prisma
model User {
  /// @zod.import(["import { emailSchema } from '../../zod-utils'"]).custom.use(emailSchema)
  email String
}
```

---

### 2.3. Kysely Generator

```prisma
generator kysely {
  provider = "prisma-kysely"
  output   = "../kysely"
  fileName = "types.ts"
}
```

**Output**: `packages/kysely/types.ts`
- TypeScript types cho Kysely query builder
- Alternative query API (SQL-like)

**Usage**:
```typescript
import { Kysely } from 'kysely';
import { DB } from '@calcom/kysely/types';

const db = new Kysely<DB>({ /* config */ });

// Type-safe SQL queries
const users = await db
  .selectFrom('users')
  .where('email', '=', 'user@example.com')
  .selectAll()
  .execute();
```

---

### 2.4. Custom Enums Generator

```prisma
generator enums {
  provider = "ts-node --transpile-only ./enum-generator.ts"
}
```

**Output**: Generated TypeScript enums
- Export enums as TypeScript types
- Sync với database enums

---

## 3. Core Domain Models

### 3.1. User Model

**Model**: `User` (365-491)

```prisma
model User {
  id                  Int                  @id @default(autoincrement())
  uuid                String               @unique @default(uuid()) @db.Uuid
  username            String?
  name                String?
  email               String               @unique
  emailVerified       DateTime?
  password            UserPassword?
  bio                 String?
  avatarUrl           String?
  timeZone            String               @default("Europe/London")
  weekStart           String               @default("Sunday")

  // Preferences
  timeFormat          Int?                 @default(12)
  locale              String?
  theme               String?
  appTheme            String?
  hideBranding        Boolean              @default(false)

  // Authentication
  identityProvider    IdentityProvider     @default(CAL)
  identityProviderId  String?
  twoFactorEnabled    Boolean              @default(false)
  twoFactorSecret     String?
  backupCodes         String?

  // Permissions
  role                UserPermissionRole   @default(USER)
  verified            Boolean?             @default(false)
  locked              Boolean              @default(false)

  // Relationships
  eventTypes          EventType[]          @relation("user_eventtype")
  ownedEventTypes     EventType[]          @relation("owner")
  teams               Membership[]
  bookings            Booking[]
  schedules           Schedule[]
  credentials         Credential[]
  webhooks            Webhook[]
  workflows           Workflow[]
  apiKeys             ApiKey[]
  profiles            Profile[]

  // Organization
  organizationId      Int?
  organization        Team?                @relation("scope", fields: [organizationId], references: [id])

  // Metadata
  metadata            Json?
  createdDate         DateTime             @default(now())
  completedOnboarding Boolean              @default(false)

  @@unique([email, username])
  @@unique([username, organizationId])
  @@index([username])
  @@index([emailVerified])
}
```

**Key Fields**:
- `uuid`: Public identifier (không expose `id`)
- `username`: Unique per organization
- `email`: Global unique (primary identity)
- `identityProvider`: CAL | GOOGLE | SAML
- `role`: USER | ADMIN
- `organizationId`: Organization membership (deprecated, use Profile)

**Relationships**:
- **1-to-Many**: EventTypes, Bookings, Schedules, Credentials
- **Many-to-Many**: Teams (via Membership)
- **1-to-1**: UserPassword, DestinationCalendar, CreditBalance

---

### 3.2. Team Model

**Model**: `Team` (526-615)

```prisma
model Team {
  id                  Int                     @id @default(autoincrement())
  name                String
  slug                String?
  logoUrl             String?
  bio                 String?

  // Settings
  hideBranding        Boolean                 @default(false)
  isPrivate           Boolean                 @default(false)
  hideBookATeamMember Boolean                 @default(false)

  // Organization features
  isOrganization      Boolean                 @default(false)
  parentId            Int?
  parent              Team?                   @relation("organization", fields: [parentId], references: [id])
  children            Team[]                  @relation("organization")

  // Relationships
  members             Membership[]
  eventTypes          EventType[]
  workflows           Workflow[]
  webhooks            Webhook[]
  credentials         Credential[]
  verifiedNumbers     VerifiedNumber[]
  verifiedEmails      VerifiedEmail[]
  orgProfiles         Profile[]

  // Organization-specific
  organizationSettings OrganizationSettings?

  // Metadata
  metadata            Json?
  theme               String?
  timeZone            String                  @default("Europe/London")
  weekStart           String                  @default("Sunday")
  createdAt           DateTime                @default(now())

  @@unique([slug, parentId])
  @@index([parentId])
}
```

**Key Concepts**:
- **Organization**: `parentId = null` và `isOrganization = true`
- **Team**: `parentId != null` (sub-team của organization)
- **Slug**: Unique per organization (teams có thể trùng slug across orgs)

**Hierarchy**:
```
Organization (parentId = null, isOrganization = true)
├── Team A (parentId = org.id)
├── Team B (parentId = org.id)
└── Team C (parentId = org.id)
```

---

### 3.3. Profile Model

**Model**: `Profile` (503-524)

```prisma
model Profile {
  id             Int         @id @default(autoincrement())
  uid            String      // Custom identifier
  userId         Int
  user           User        @relation(fields: [userId], references: [id])
  organizationId Int
  organization   Team        @relation(fields: [organizationId], references: [id])
  username       String
  eventTypes     EventType[]

  @@unique([userId, organizationId])
  @@unique([username, organizationId])
}
```

**Purpose**: Multi-tenant user profiles
- User có nhiều profiles trong các organizations khác nhau
- Mỗi profile có `username` unique trong organization
- Replace deprecated `User.organizationId`

**Example**:
```typescript
// User john@example.com
// - Profile 1: username="john" in Organization A
// - Profile 2: username="johnsmith" in Organization B
```

---

### 3.4. Membership Model

**Model**: `Membership` (698-720)

```prisma
model Membership {
  id                   Int               @id @default(autoincrement())
  teamId               Int
  userId               Int
  accepted             Boolean           @default(false)
  role                 MembershipRole    // MEMBER | ADMIN | OWNER
  customRoleId         String?
  customRole           Role?
  team                 Team              @relation(fields: [teamId], references: [id])
  user                 User              @relation(fields: [userId], references: [id])
  disableImpersonation Boolean           @default(false)

  @@unique([userId, teamId])
  @@index([accepted])
}
```

**Roles**:
- `MEMBER`: Basic team member
- `ADMIN`: Team admin (manage members, settings)
- `OWNER`: Team owner (full control)
- `customRole`: PBAC (Permission-Based Access Control)

---

### 3.5. OrganizationSettings Model

**Model**: `OrganizationSettings` (669-690)

```prisma
model OrganizationSettings {
  id                                  Int     @id @default(autoincrement())
  organizationId                      Int     @unique
  organization                        Team    @relation(fields: [organizationId], references: [id])

  isOrganizationConfigured            Boolean @default(false)
  isOrganizationVerified              Boolean @default(false)
  orgAutoAcceptEmail                  String  // Domain for auto-accept (e.g., "acme.com")

  lockEventTypeCreationForUsers       Boolean @default(false)
  isAdminReviewed                     Boolean @default(false)
  isAdminAPIEnabled                   Boolean @default(false)
  allowSEOIndexing                    Boolean @default(false)
  orgProfileRedirectsToVerifiedDomain Boolean @default(false)
}
```

**Features**:
- **Auto-accept**: Users với email domain match được auto-join org
- **Lock event types**: Prevent users from creating personal event types
- **Admin review**: Required for sensitive operations (impersonation)
- **Admin API**: Enable organization-level API access

---

## 4. Scheduling Domain

### 4.1. EventType Model

**Model**: `EventType` (129-274)

```prisma
model EventType {
  id                  Int     @id @default(autoincrement())
  title               String
  slug                String
  description         String?
  length              Int     // Duration in minutes

  // Ownership
  userId              Int?
  owner               User?   @relation("owner", fields: [userId], references: [id])
  teamId              Int?
  team                Team?   @relation(fields: [teamId], references: [id])
  profileId           Int?
  profile             Profile? @relation(fields: [profileId], references: [id])

  // Hosts (Round Robin / Collective)
  hosts               Host[]
  schedulingType      SchedulingType?  // ROUND_ROBIN | COLLECTIVE | MANAGED

  // Booking settings
  locations           Json?
  requiresConfirmation Boolean @default(false)
  disableGuests       Boolean @default(false)
  minimumBookingNotice Int    @default(120)  // minutes
  beforeEventBuffer   Int     @default(0)
  afterEventBuffer    Int     @default(0)

  // Availability
  scheduleId          Int?
  schedule            Schedule? @relation(fields: [scheduleId], references: [id])

  // Period settings
  periodType          PeriodType @default(UNLIMITED)
  periodStartDate     DateTime?
  periodEndDate       DateTime?
  periodDays          Int?

  // Seats
  seatsPerTimeSlot    Int?
  seatsShowAttendees  Boolean? @default(false)

  // Recurring events
  recurringEvent      Json?

  // Metadata
  metadata            Json?
  bookingFields       Json?
  customInputs        EventTypeCustomInput[]

  // Relationships
  bookings            Booking[]
  webhooks            Webhook[]
  workflows           WorkflowsOnEventTypes[]

  @@unique([userId, slug])
  @@unique([teamId, slug])
}
```

**Scheduling Types**:

1. **ROUND_ROBIN**: Distribute bookings evenly among hosts
   - Uses `Host.weight` and `Host.priority`
   - Round-robin algorithm

2. **COLLECTIVE**: All hosts must be available
   - Find common availability
   - All hosts join meeting

3. **MANAGED**: Parent-child event types
   - Organization-managed event types
   - Children inherit settings from parent

**Period Types**:
- `UNLIMITED`: Always bookable
- `ROLLING`: X days into future
- `ROLLING_WINDOW`: X days from now
- `RANGE`: Specific date range

---

### 4.2. Host Model

**Model**: `Host` (61-84)

```prisma
model Host {
  userId           Int
  user             User        @relation(fields: [userId], references: [id])
  eventTypeId      Int
  eventType        EventType   @relation(fields: [eventTypeId], references: [id])

  isFixed          Boolean     @default(false)
  priority         Int?        // Lower = higher priority
  weight           Int?        // For weighted round robin

  scheduleId       Int?
  schedule         Schedule?   @relation(fields: [scheduleId], references: [id])

  groupId          String?
  group            HostGroup?  @relation(fields: [groupId], references: [id])

  memberId         Int?
  member           Membership? @relation(fields: [memberId], references: [id])

  @@id([userId, eventTypeId])
  @@index([userId])
  @@index([eventTypeId])
}
```

**Key Features**:
- **Fixed hosts**: `isFixed = true` - always included
- **Priority**: Lower value = higher priority
- **Weight**: For weighted distribution (e.g., senior gets 2x bookings)
- **Custom schedule**: Override default availability

---

### 4.3. Schedule Model

**Model**: `Schedule` (899-912)

```prisma
model Schedule {
  id           Int            @id @default(autoincrement())
  userId       Int
  user         User           @relation(fields: [userId], references: [id])
  name         String
  timeZone     String?
  availability Availability[]

  // Used by EventTypes
  eventType    EventType[]

  @@index([userId])
}
```

**Purpose**: Named availability schedules
- User có nhiều schedules (e.g., "Work Hours", "Weekend Only")
- EventType reference schedule
- Default schedule: `User.defaultScheduleId`

---

### 4.4. Availability Model

**Model**: `Availability` (914-930)

```prisma
model Availability {
  id          Int        @id @default(autoincrement())
  userId      Int?
  user        User?      @relation(fields: [userId], references: [id])
  eventTypeId Int?
  eventType   EventType? @relation(fields: [eventTypeId], references: [id])
  scheduleId  Int?
  schedule    Schedule?  @relation(fields: [scheduleId], references: [id])

  days        Int[]      // [0,1,2,3,4] = Mon-Fri
  startTime   DateTime   @db.Time
  endTime     DateTime   @db.Time
  date        DateTime?  @db.Date  // For date overrides

  @@index([userId])
  @@index([eventTypeId])
  @@index([scheduleId])
}
```

**Hierarchy**:
1. **Schedule-level**: `scheduleId != null`
2. **EventType-level**: `eventTypeId != null` (override schedule)
3. **User-level**: `userId != null` (deprecated, use schedule)

**Days encoding**:
```typescript
// Days: [0,1,2,3,4] = Monday to Friday
// 0 = Sunday, 1 = Monday, ..., 6 = Saturday
```

**Time format**:
```typescript
// startTime/endTime: stored as PostgreSQL TIME type
// Example: "09:00:00" = 9 AM
```

---

### 4.5. Booking Model

**Model**: `Booking` (807-884)

```prisma
model Booking {
  id              Int           @id @default(autoincrement())
  uid             String        @unique
  idempotencyKey  String?       @unique  // Prevent duplicates

  // Booker
  userId          Int?
  user            User?         @relation(fields: [userId], references: [id])
  userPrimaryEmail String?

  // Event
  eventTypeId     Int?
  eventType       EventType?    @relation(fields: [eventTypeId], references: [id])
  title           String
  description     String?

  // Time
  startTime       DateTime
  endTime         DateTime

  // Attendees
  attendees       Attendee[]

  // Location
  location        String?

  // Status
  status          BookingStatus @default(ACCEPTED)
  paid            Boolean       @default(false)

  // Responses
  customInputs    Json?
  responses       Json?         // Booking form responses

  // References (calendar events, video meetings)
  references      BookingReference[]

  // Cancellation/Rescheduling
  cancellationReason String?
  rejectionReason    String?
  rescheduled        Boolean?
  fromReschedule     String?      // Original booking uid

  // Recurring
  recurringEventId   String?

  // Metadata
  metadata        Json?

  // Workflows
  workflowReminders WorkflowReminder[]

  // Ratings
  rating          Int?
  ratingFeedback  String?
  noShowHost      Boolean? @default(false)

  // Destination calendar
  destinationCalendarId Int?
  destinationCalendar   DestinationCalendar?

  @@index([eventTypeId])
  @@index([userId])
  @@index([status])
  @@index([startTime, endTime, status])
}
```

**Status Flow**:
```
PENDING → ACCEPTED
        → REJECTED
        → CANCELLED

AWAITING_HOST (requires confirmation)
```

**Idempotency**:
- `idempotencyKey`: Hash của (email + startTime + endTime)
- Prevent duplicate bookings khi user submit form nhiều lần

---

### 4.6. Attendee Model

**Model**: `Attendee` (783-797)

```prisma
model Attendee {
  id          Int     @id @default(autoincrement())
  email       String
  name        String
  timeZone    String
  phoneNumber String?
  locale      String? @default("en")
  bookingId   Int?
  booking     Booking? @relation(fields: [bookingId], references: [id])
  noShow      Boolean? @default(false)

  @@index([email])
  @@index([bookingId])
}
```

**Purpose**: Participants trong booking
- Có thể nhiều attendees cho 1 booking
- Store timezone, locale của mỗi attendee
- `noShow` tracking

---

### 4.7. BookingReference Model

**Model**: `BookingReference` (756-781)

```prisma
model BookingReference {
  id                String   @id @default(autoincrement())
  type              String   // "google_calendar", "zoom_video", etc.
  uid               String
  meetingId         String?
  meetingPassword   String?
  meetingUrl        String?

  bookingId         Int?
  booking           Booking? @relation(fields: [bookingId], references: [id])

  externalCalendarId String?
  deleted           Boolean?

  credentialId      Int?
  credential        Credential?

  @@index([bookingId])
  @@index([type])
  @@index([uid])
}
```

**Purpose**: Link booking với external services
- **Calendar events**: Google Calendar, Outlook, Apple Calendar
- **Video meetings**: Zoom, Google Meet, MS Teams
- **Payment**: Stripe checkout sessions

**Example**:
```typescript
// Booking có 2 references:
// 1. Google Calendar event (type="google_calendar", uid="event123")
// 2. Zoom meeting (type="zoom_video", meetingUrl="https://zoom.us/j/...")
```

---

## 5. Integration Domain

### 5.1. Credential Model

**Model**: `Credential` (276-306)

```prisma
model Credential {
  id     Int     @id @default(autoincrement())
  type   String  // "google_calendar", "zoom_video", etc. (deprecated)
  key    Json    // Encrypted credentials (OAuth tokens, API keys)

  userId Int?
  user   User? @relation(fields: [userId], references: [id])
  teamId Int?
  team   Team? @relation(fields: [teamId], references: [id])

  appId  String?
  app    App?   @relation(fields: [appId], references: [slug])

  // Paid apps
  subscriptionId    String?
  paymentStatus     String?
  billingCycleStart Int?

  invalid           Boolean? @default(false)

  // Relationships
  destinationCalendars DestinationCalendar[]
  selectedCalendars    SelectedCalendar[]
  references           BookingReference[]

  @@index([appId])
  @@index([invalid])
}
```

**Key Fields**:
- `key`: Encrypted JSON với OAuth tokens, refresh tokens, API keys
- `appId`: Link to `App` model (e.g., "google-calendar", "zoom")
- `invalid`: Mark credential as invalid (OAuth revoked, expired...)

**Encryption**:
```typescript
// key is encrypted with CALENDSO_ENCRYPTION_KEY
const encryptedKey = encrypt(JSON.stringify({
  access_token: "...",
  refresh_token: "...",
  expires_at: 1234567890
}));
```

---

### 5.2. App Model

**Model**: `App` (1228-1247)

```prisma
model App {
  slug               String          @id
  dirName            String          // Directory name in app-store
  keys               Json?           // App-wide keys (not user-specific)
  categories         AppCategories[]
  createdAt          DateTime        @default(now())
  updatedAt          DateTime        @updatedAt
  enabled            Boolean         @default(false)

  // Relationships
  credentials        Credential[]
  payments           Payment[]
  webhooks           Webhook[]
  apiKeys            ApiKey[]
}
```

**App Categories** (enum):
```prisma
enum AppCategories {
  calendar
  messaging
  other
  payment
  video
  web3
  automation
  analytics
  conferencing
  crm
}
```

**App Registry**:
- Generated by `@calcom/app-store-cli`
- File: `packages/app-store/apps.generated.ts`
- 108 apps total

---

### 5.3. DestinationCalendar Model

**Model**: `DestinationCalendar` (314-337)

```prisma
model DestinationCalendar {
  id           Int         @id @default(autoincrement())
  integration  String      // "google_calendar", "office365_calendar"
  externalId   String      // Calendar ID in external service
  primaryEmail String?

  userId       Int?        @unique
  user         User?       @relation(fields: [userId], references: [id])

  eventTypeId  Int?        @unique
  eventType    EventType?  @relation(fields: [eventTypeId], references: [id])

  credentialId Int?
  credential   Credential? @relation(fields: [credentialId], references: [id])

  @@index([userId])
  @@index([eventTypeId])
  @@index([credentialId])
}
```

**Purpose**: Where to create calendar events
- User-level: Default calendar cho user
- EventType-level: Override per event type

**Example**:
```typescript
// User có 2 calendars: personal@gmail.com, work@company.com
// DestinationCalendar points to: work@company.com
// → All bookings tạo event trong work calendar
```

---

### 5.4. SelectedCalendar Model

**Model**: `SelectedCalendar` (932-1001)

```prisma
model SelectedCalendar {
  id           String      @id @default(uuid())
  userId       Int
  user         User        @relation(fields: [userId], references: [id])
  integration  String      // "google_calendar"
  externalId   String      // Calendar ID

  credentialId Int?
  credential   Credential? @relation(fields: [credentialId], references: [id])

  eventTypeId  Int?
  eventType    EventType?  @relation(fields: [eventTypeId], references: [id])

  // Calendar sync
  syncToken    String?
  syncedAt     DateTime?

  // Watch channel (Google Calendar push notifications)
  channelId          String?
  channelExpiration  DateTime?

  @@unique([userId, integration, externalId, eventTypeId])
  @@index([userId])
}
```

**Purpose**: Which calendars to check for availability
- User selects multiple calendars (personal, work, etc.)
- System checks all selected calendars for conflicts

**Difference với DestinationCalendar**:
- **DestinationCalendar**: WHERE to create events (write)
- **SelectedCalendar**: WHICH calendars to check (read)

---

### 5.5. Webhook Model

**Model**: `Webhook` (1096-1123)

```prisma
model Webhook {
  id                    String                     @id @unique
  userId                Int?
  teamId                Int?
  eventTypeId           Int?
  platformOAuthClientId String?

  subscriberUrl         String
  payloadTemplate       String?
  active                Boolean                    @default(true)
  eventTriggers         WebhookTriggerEvents[]
  secret                String?

  user                  User?                      @relation(fields: [userId], references: [id])
  team                  Team?                      @relation(fields: [teamId], references: [id])
  eventType             EventType?                 @relation(fields: [eventTypeId], references: [id])

  @@unique([userId, subscriberUrl])
}
```

**Trigger Events**:
```prisma
enum WebhookTriggerEvents {
  BOOKING_CREATED
  BOOKING_RESCHEDULED
  BOOKING_CANCELLED
  BOOKING_REJECTED
  BOOKING_REQUESTED
  BOOKING_NO_SHOW_UPDATED
  FORM_SUBMITTED
  MEETING_ENDED
  MEETING_STARTED
  RECORDING_READY
  // ... more events
}
```

**Levels**:
1. **User-level**: `userId != null`
2. **Team-level**: `teamId != null`
3. **EventType-level**: `eventTypeId != null`
4. **Platform-level**: `platformOAuthClientId != null`

---

## 6. Enterprise Features

### 6.1. RBAC (Role-Based Access Control)

#### Role Model

**Model**: `Role` (2598-2614)

```prisma
model Role {
  id          String       @id @default(uuid())
  name        String
  teamId      Int
  team        Team         @relation(fields: [teamId], references: [id])
  memberships Membership[]
  permissions RolePermission[]
}
```

#### RolePermission Model

**Model**: `RolePermission` (2615-2639)

```prisma
model RolePermission {
  id         String @id @default(uuid())
  roleId     String
  role       Role   @relation(fields: [roleId], references: [id])
  resource   String // "event-type", "booking", "team-member"
  action     String // "create", "read", "update", "delete"
  conditions Json?  // Additional conditions
}
```

**Example**:
```typescript
// Role: "Event Manager"
// Permissions:
// - { resource: "event-type", action: "create" }
// - { resource: "event-type", action: "update" }
// - { resource: "booking", action: "read" }
```

---

### 6.2. PBAC (Permission-Based Access Control)

#### Attribute Model

**Model**: `Attribute` (2149-2175)

```prisma
model Attribute {
  id                 String            @id @default(uuid())
  name               String
  type               String            // "TEXT", "NUMBER", "SELECT", "MULTI_SELECT"
  slug               String
  options            AttributeOption[]
  teamId             Int
  team               Team              @relation(fields: [teamId], references: [id])
  AttributeToUser    AttributeToUser[]

  @@unique([slug, teamId])
}
```

#### AttributeOption Model

**Model**: `AttributeOption` (2134-2148)

```prisma
model AttributeOption {
  id          String    @id @default(uuid())
  value       String
  slug        String
  attributeId String
  attribute   Attribute @relation(fields: [attributeId], references: [id])

  @@unique([slug, attributeId])
}
```

#### AttributeToUser Model

**Model**: `AttributeToUser` (2176-2214)

```prisma
model AttributeToUser {
  id                    String         @id @default(uuid())
  memberId              Int
  member                Membership     @relation(fields: [memberId], references: [id])
  attributeId           String
  attribute             Attribute      @relation(fields: [attributeId], references: [id])
  attributeOptionId     String?

  // Conditional assignment
  enabled               Boolean        @default(true)
  value                 String?

  @@unique([memberId, attributeId])
}
```

**Purpose**: Assign attributes to users
- Example: `department="Engineering"`, `location="San Francisco"`
- Use in routing logic (route booking based on attributes)

---

### 6.3. SSO (Single Sign-On)

**Identity Providers**:
```prisma
enum IdentityProvider {
  CAL      // Cal.com native auth
  GOOGLE   // Google OAuth
  SAML     // SAML SSO (enterprise)
}
```

**User fields**:
- `identityProvider`: Which provider used
- `identityProviderId`: External user ID

**SAML Implementation**:
- Package: `@boxyhq/saml-jackson`
- Location: `apps/web` dependency
- Enterprise feature

---

### 6.4. SCIM (System for Cross-domain Identity Management)

#### DSyncData Model

**Model**: `DSyncData` (2020-2031)

```prisma
model DSyncData {
  id                   Int                     @id @default(autoincrement())
  organizationId       Int                     @unique
  organization         OrganizationSettings    @relation(fields: [organizationId], references: [id])
  tenant               String
  dSyncTeamGroupMapping DSyncTeamGroupMapping[]
}
```

#### DSyncTeamGroupMapping Model

**Model**: `DSyncTeamGroupMapping` (2032-2043)

```prisma
model DSyncTeamGroupMapping {
  id         Int       @id @default(autoincrement())
  dsyncId    Int
  dsync      DSyncData @relation(fields: [dsyncId], references: [id])
  teamId     Int
  team       Team      @relation(fields: [teamId], references: [id])
  directoryId String
  groupId    String
}
```

**Purpose**: Sync users/groups từ external directory (Azure AD, Okta...)

---

## 7. Workflow & Automation

### 7.1. Workflow Model

**Model**: `Workflow` (1469-1489)

```prisma
model Workflow {
  id              Int                       @id @default(autoincrement())
  name            String
  userId          Int?
  user            User?                     @relation(fields: [userId], references: [id])
  teamId          Int?
  team            Team?                     @relation(fields: [teamId], references: [id])

  trigger         WorkflowTriggerEvents     // BEFORE_EVENT, AFTER_EVENT, etc.
  time            Int?                      // Minutes before/after
  timeUnit        TimeUnit?                 // MINUTE, HOUR, DAY

  steps           WorkflowStep[]
  activeOn        WorkflowsOnEventTypes[]
  isActiveOnAll   Boolean                   @default(false)

  @@index([userId])
  @@index([teamId])
}
```

**Triggers**:
```prisma
enum WorkflowTriggerEvents {
  BEFORE_EVENT           // X minutes before booking
  AFTER_EVENT            // X minutes after booking
  NEW_EVENT              // When booking created
  EVENT_CANCELLED        // When booking cancelled
  RESCHEDULE_EVENT       // When booking rescheduled
  BOOKING_REQUESTED      // Requires confirmation
  FORM_SUBMITTED         // Routing form submitted
  // ... more events
}
```

---

### 7.2. WorkflowStep Model

**Model**: `WorkflowStep` (1444-1467)

```prisma
model WorkflowStep {
  id              Int                @id @default(autoincrement())
  stepNumber      Int
  action          WorkflowActions    // EMAIL_HOST, SMS_ATTENDEE, etc.
  workflowId      Int
  workflow        Workflow           @relation(fields: [workflowId], references: [id])

  // Content
  sendTo          String?            // Email or phone
  reminderBody    String?
  emailSubject    String?
  template        WorkflowTemplates  @default(REMINDER)

  // SMS verification
  numberVerificationPending Boolean @default(true)
  verifiedAt      DateTime?

  @@index([workflowId])
}
```

**Actions**:
```prisma
enum WorkflowActions {
  EMAIL_HOST
  EMAIL_ATTENDEE
  SMS_ATTENDEE
  SMS_NUMBER
  EMAIL_ADDRESS
  WHATSAPP_ATTENDEE
  WHATSAPP_NUMBER
  CAL_AI_PHONE_CALL
}
```

**Example Workflow**:
```typescript
// Workflow: "24h Reminder"
// Trigger: BEFORE_EVENT (24 hours)
// Steps:
//   1. EMAIL_ATTENDEE (subject: "Reminder: Your meeting tomorrow")
//   2. SMS_ATTENDEE (body: "Meeting tomorrow at {start_time}")
```

---

## 8. Payment & Billing

### 8.1. Payment Model

**Model**: `Payment` (1049-1067)

```prisma
model Payment {
  id          Int            @id @default(autoincrement())
  uid         String         @unique
  appId       String?
  app         App?           @relation(fields: [appId], references: [slug])
  bookingId   Int
  booking     Booking        @relation(fields: [bookingId], references: [id])

  amount      Int            // in cents
  fee         Int            // platform fee
  currency    String
  success     Boolean
  refunded    Boolean
  externalId  String         @unique  // Stripe payment intent ID

  paymentOption PaymentOption? @default(ON_BOOKING)

  @@index([bookingId])
  @@index([externalId])
}
```

**Payment Options**:
```prisma
enum PaymentOption {
  ON_BOOKING  // Pay when booking
  HOLD        // Hold payment (charge later)
}
```

---

### 8.2. Credit System

#### CreditBalance Model

**Model**: `CreditBalance` (617-629)

```prisma
model CreditBalance {
  id                String              @id @default(uuid())
  teamId            Int?                @unique
  team              Team?               @relation(fields: [teamId], references: [id])
  userId            Int?                @unique
  user              User?               @relation(fields: [userId], references: [id])

  additionalCredits Int                 @default(0)
  limitReachedAt    DateTime?
  warningSentAt     DateTime?

  expenseLogs       CreditExpenseLog[]
  purchaseLogs      CreditPurchaseLog[]
}
```

#### CreditExpenseLog Model

**Model**: `CreditExpenseLog` (644-662)

```prisma
model CreditExpenseLog {
  id              String           @id @default(uuid())
  creditBalanceId String
  creditBalance   CreditBalance    @relation(fields: [creditBalanceId], references: [id])

  bookingUid      String?
  booking         Booking?         @relation(fields: [bookingUid], references: [uid])

  credits         Int?
  creditType      CreditType       // MONTHLY | ADDITIONAL
  creditFor       CreditUsageType? // SMS | CAL_AI_PHONE_CALL

  // SMS specific
  smsSid          String?
  smsSegments     Int?
  phoneNumber     String?

  // Call specific
  callDuration    Int?

  date            DateTime
}
```

**Credit Types**:
```prisma
enum CreditType {
  MONTHLY      // Monthly org credits
  ADDITIONAL   // Purchased credits
}

enum CreditUsageType {
  SMS
  CAL_AI_PHONE_CALL
}
```

---

### 8.3. Platform Billing

#### PlatformBilling Model

**Model**: `PlatformBilling` (2106-2133)

```prisma
model PlatformBilling {
  id                        Int      @id @default(autoincrement())
  teamId                    Int      @unique
  team                      Team     @relation(fields: [teamId], references: [id])

  subscriptionId            String?  @unique
  subscriptionItemId        String?
  status                    String?

  // Stripe
  stripeCustomerId          String?
  stripeSubscriptionId      String?
  stripeSubscriptionStatus  String?

  // Usage
  creditsUsed               Int      @default(0)
  creditsLimit              Int?

  createdAt                 DateTime @default(now())
  updatedAt                 DateTime @updatedAt
}
```

---

## 9. Platform & OAuth

### 9.1. PlatformOAuthClient Model

**Model**: `PlatformOAuthClient` (1951-1977)

```prisma
model PlatformOAuthClient {
  id                String   @id @default(uuid())
  name              String
  secret            String
  redirectUris      String[]

  organizationId    Int
  organization      Team     @relation(fields: [organizationId], references: [id])

  createdByUserId   Int
  createdBy         User     @relation(fields: [createdByUserId], references: [id])

  webhooks          Webhook[]
  createdTeams      Team[]   @relation("CreatedByOAuthClient")

  createdAt         DateTime @default(now())
  updatedAt         DateTime @updatedAt
}
```

**Purpose**: OAuth clients for Platform API
- Organization creates OAuth clients
- Used to authenticate API requests
- Can create teams on behalf of users

---

### 9.2. Access Tokens

#### AccessToken Model

**Model**: `AccessToken` (1992-2005)

```prisma
model AccessToken {
  id         String   @id @default(uuid())
  secret     String   @unique
  userId     Int
  user       User     @relation(fields: [userId], references: [id])
  expiresAt  DateTime

  createdAt  DateTime @default(now())
}
```

#### RefreshToken Model

**Model**: `RefreshToken` (2006-2019)

```prisma
model RefreshToken {
  id         String   @id @default(uuid())
  secret     String   @unique
  userId     Int
  user       User     @relation(fields: [userId], references: [id])
  expiresAt  DateTime

  createdAt  DateTime @default(now())
}
```

---

## 10. Calendar Sync & Caching

### 10.1. CalendarCache Model

**Model**: `CalendarCache` (1859-1884)

```prisma
model CalendarCache {
  key              String      @id
  credentialId     Int
  credential       Credential  @relation(fields: [credentialId], references: [id])

  externalCalendarId String
  value            Json

  expiresAt        DateTime    @db.Timestamp(3)

  @@index([credentialId])
  @@index([expiresAt])
}
```

**Purpose**: Cache calendar events để giảm API calls
- Key: hash của (credentialId, startDate, endDate)
- Value: Array of calendar events
- TTL: Configurable expiration

---

### 10.2. CalendarCacheEvent Model

**Model**: `CalendarCacheEvent` (2750-2771)

```prisma
model CalendarCacheEvent {
  id                  String           @id @default(uuid())
  selectedCalendarId  String
  selectedCalendar    SelectedCalendar @relation(fields: [selectedCalendarId], references: [id])

  externalEventId     String
  eventStart          DateTime
  eventEnd            DateTime

  // Event details
  summary             String?
  description         String?
  location            String?
  status              String?

  @@index([selectedCalendarId])
  @@index([eventStart, eventEnd])
  @@unique([selectedCalendarId, externalEventId])
}
```

**Purpose**: Normalized calendar events for fast availability check

---

## 11. Enums Reference

### Authentication & Permissions

```prisma
enum IdentityProvider {
  CAL
  GOOGLE
  SAML
}

enum UserPermissionRole {
  USER
  ADMIN
}

enum MembershipRole {
  MEMBER
  ADMIN
  OWNER
}
```

### Scheduling

```prisma
enum SchedulingType {
  ROUND_ROBIN
  COLLECTIVE
  MANAGED
}

enum PeriodType {
  UNLIMITED
  ROLLING
  ROLLING_WINDOW
  RANGE
}

enum BookingStatus {
  CANCELLED
  ACCEPTED
  REJECTED
  PENDING
  AWAITING_HOST
}
```

### Workflow

```prisma
enum WorkflowTriggerEvents {
  BEFORE_EVENT
  AFTER_EVENT
  NEW_EVENT
  EVENT_CANCELLED
  RESCHEDULE_EVENT
  FORM_SUBMITTED
  // ... 15+ events
}

enum WorkflowActions {
  EMAIL_HOST
  EMAIL_ATTENDEE
  SMS_ATTENDEE
  SMS_NUMBER
  EMAIL_ADDRESS
  WHATSAPP_ATTENDEE
  WHATSAPP_NUMBER
  CAL_AI_PHONE_CALL
}

enum TimeUnit {
  MINUTE
  HOUR
  DAY
}
```

### Payments

```prisma
enum PaymentOption {
  ON_BOOKING
  HOLD
}

enum CreditType {
  MONTHLY
  ADDITIONAL
}

enum CreditUsageType {
  SMS
  CAL_AI_PHONE_CALL
}
```

### App Store

```prisma
enum AppCategories {
  calendar
  messaging
  other
  payment
  video
  web3
  automation
  analytics
  conferencing
  crm
}
```

---

## 12. Database Indexes & Performance

### 12.1. Key Indexes

**Booking queries** (most critical):
```prisma
model Booking {
  @@index([eventTypeId])
  @@index([userId])
  @@index([status])
  @@index([recurringEventId])
  @@index([startTime, endTime, status])  // Composite for availability
}
```

**User lookups**:
```prisma
model User {
  @@index([username])
  @@index([emailVerified])
  @@index([identityProvider])
  @@unique([email])
  @@unique([email, username])
  @@unique([username, organizationId])
}
```

**EventType queries**:
```prisma
model EventType {
  @@unique([userId, slug])
  @@unique([teamId, slug])
  @@index([userId])
  @@index([teamId])
  @@index([profileId])
}
```

---

### 12.2. Performance Tips

1. **Always use indexes** cho WHERE clauses
2. **Composite indexes** cho multi-column queries
3. **Partial indexes** với `@@partial_index` (commented out, not supported yet)
4. **Select only needed fields**:
   ```typescript
   // ❌ Bad: Fetch all fields
   const booking = await prisma.booking.findUnique({ where: { id } });

   // ✅ Good: Select specific fields
   const booking = await prisma.booking.findUnique({
     where: { id },
     select: { id: true, startTime: true, endTime: true }
   });
   ```

5. **Use `include` sparingly**:
   ```typescript
   // ❌ Bad: Deep nesting
   const user = await prisma.user.findUnique({
     where: { id },
     include: {
       teams: {
         include: {
           team: {
             include: { members: true, eventTypes: true }
           }
         }
       }
     }
   });

   // ✅ Good: Fetch only what you need
   const user = await prisma.user.findUnique({
     where: { id },
     include: { teams: { select: { teamId: true, role: true } } }
   });
   ```

6. **Batch queries** với `findMany`:
   ```typescript
   // ❌ Bad: N+1 queries
   for (const userId of userIds) {
     const user = await prisma.user.findUnique({ where: { id: userId } });
   }

   // ✅ Good: Single query
   const users = await prisma.user.findMany({
     where: { id: { in: userIds } }
   });
   ```

---

## 13. Best Practices

### 13.1. Migrations

**Create migration**:
```bash
cd packages/prisma
yarn prisma migrate dev --name add_new_field
```

**Deploy to production**:
```bash
yarn prisma migrate deploy
```

**Reset database** (dev only):
```bash
yarn prisma migrate reset
```

---

### 13.2. Seed Data

**Location**: `packages/prisma/seed.ts`

**Run seed**:
```bash
yarn workspace @calcom/prisma db-seed
```

**Custom seeds**:
```bash
yarn workspace @calcom/prisma seed-app-store    # Seed app store
yarn workspace @calcom/prisma seed-insights     # Seed insights data
```

---

### 13.3. Schema Changes

1. **Always add indexes** cho foreign keys
2. **Use `@default`** cho new fields (avoid breaking changes)
3. **Deprecate, don't delete** (comment as deprecated)
4. **Test migrations** trên staging trước khi deploy
5. **Backup database** trước major migrations

**Example deprecation**:
```prisma
model EventType {
  // price is deprecated. It has now moved to metadata.apps.stripe.price.
  // Plan to drop this column.
  price    Int @default(0)

  // New field
  metadata Json?  // { apps: { stripe: { price: 100 } } }
}
```

---

### 13.4. Query Patterns

**Find unique with relations**:
```typescript
const eventType = await prisma.eventType.findUnique({
  where: { id: eventTypeId },
  include: {
    users: true,
    team: true,
    hosts: { include: { user: true } },
    schedule: { include: { availability: true } }
  }
});
```

**Nested creates** (transaction):
```typescript
const booking = await prisma.booking.create({
  data: {
    uid: generateUid(),
    title: "Meeting",
    startTime: new Date(),
    endTime: new Date(),
    eventType: { connect: { id: eventTypeId } },
    user: { connect: { id: userId } },
    attendees: {
      create: [
        { email: "john@example.com", name: "John", timeZone: "UTC" }
      ]
    },
    references: {
      create: [
        { type: "google_calendar", uid: "event123" }
      ]
    }
  }
});
```

**Complex filters**:
```typescript
const bookings = await prisma.booking.findMany({
  where: {
    AND: [
      { userId: currentUserId },
      { status: { in: ["ACCEPTED", "PENDING"] } },
      { startTime: { gte: new Date() } },
      {
        OR: [
          { eventType: { teamId: teamId } },
          { eventType: { userId: currentUserId } }
        ]
      }
    ]
  },
  orderBy: { startTime: "asc" },
  take: 10
});
```

---

### 13.5. Zod Validation

**Use generated schemas**:
```typescript
import { bookingCreateBodySchema } from '@calcom/prisma/zod';

// Validate input
const result = bookingCreateBodySchema.safeParse(req.body);
if (!result.success) {
  return res.status(400).json({ error: result.error });
}

// Use validated data
const validatedData = result.data;
```

**Custom validators** (in schema):
```prisma
model User {
  /// @zod.import(["import { emailSchema } from '../../zod-utils'"]).custom.use(emailSchema)
  email String
}
```

---

## 📝 Tổng kết PHASE 2

✅ **Đã phân tích**:
- 105 Prisma models
- 25+ enums
- 4 generators (Prisma Client, Zod, Kysely, Enums)
- Core domain: User, Team, Profile, Membership
- Scheduling: EventType, Schedule, Booking, Host
- Integrations: Credential, App, Webhook
- Enterprise: RBAC, PBAC, SSO, SCIM
- Workflow & automation
- Payment & billing
- Platform OAuth
- Calendar sync & caching

**Files covered**:
- ✅ `packages/prisma/schema.prisma` (2780 dòng)
- ✅ Generated artifacts (Zod, Kysely)

**Key insights**:
1. **Multi-tenancy**: Organizations → Teams → Users (via Profiles)
2. **Scheduling complexity**: Round-robin, Collective, Managed event types
3. **Integration architecture**: 108 apps với credential management
4. **Enterprise-grade**: RBAC, PBAC, SSO, SCIM
5. **Performance**: Strategic indexes, caching, denormalized views

---

## 👉 Gợi ý PHASE tiếp theo

**PHASE 3: tRPC Architecture & API Layer**

Sẽ đi sâu vào:
- Router hierarchy (viewer, loggedInViewer, publicViewer)
- 35+ sub-routers trong viewer
- Procedures & middlewares
- Context creation
- Error handling
- Client-side usage

**Sẵn sàng chạy PHASE 3?**
```
User nói: "OK, chạy PHASE 3"
```
