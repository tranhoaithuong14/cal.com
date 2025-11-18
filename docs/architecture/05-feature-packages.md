# PHASE 5: Feature Packages Deep Dive

> **Mục tiêu**: Hiểu 55 feature packages, vai trò, dependencies, và cách chúng tương tác với nhau.

## 📋 Table of Contents

- [1. Overview](#1-overview)
- [2. Package Structure](#2-package-structure)
- [3. Core Features](#3-core-features)
- [4. Enterprise Features (EE)](#4-enterprise-features-ee)
- [5. Automation & Integration](#5-automation--integration)
- [6. UI & Component Features](#6-ui--component-features)
- [7. Utility & Infrastructure Features](#7-utility--infrastructure-features)
- [8. Feature Dependencies](#8-feature-dependencies)
- [9. Adding New Features](#9-adding-new-features)
- [10. Best Practices](#10-best-practices)

---

## 1. Overview

### 1.1 What are Feature Packages?

Feature packages in Cal.com are **modular, domain-specific packages** located in `packages/features/`. Unlike traditional monorepo packages with their own `package.json`, most feature packages are **sub-folders of the main `@calcom/features` package**.

```typescript
// Import pattern
import { handleNewBooking } from "@calcom/features/bookings/lib/handleNewBooking";
import { sendOrSchedulePayload } from "@calcom/features/webhooks/lib/sendPayload";
import { OrganizationRepository } from "@calcom/features/ee/organizations/repositories";
```

### 1.2 Package Statistics

```
Total Feature Packages: 55
├── Core Features: 18
├── Enterprise Features (ee/): 19
├── Automation Features: 4
├── UI Features: 6
└── Utility Features: 8
```

### 1.3 Main Package Configuration

**File**: `packages/features/package.json`

```json
{
  "name": "@calcom/features",
  "version": "1.0.0",
  "private": true,
  "description": "Cal.com's main collocation of features",
  "main": "index.ts",
  "dependencies": {
    "@calcom/atoms": "workspace:*",
    "@calcom/dayjs": "workspace:*",
    "@calcom/lib": "workspace:*",
    "@calcom/trpc": "workspace:*",
    "@calcom/ui": "workspace:*",
    "framer-motion": "^10.12.8",
    "recharts": "^3.0.2",
    "zustand": "^4.3.2",
    "web-push": "^3.6.7"
  }
}
```

**Key Dependencies**:
- `@calcom/atoms`: Platform SDK components
- `@calcom/lib`: Core utilities
- `@calcom/trpc`: API layer
- `@calcom/ui`: UI component library
- `framer-motion`: Animations
- `recharts`: Analytics charts
- `zustand`: State management
- `web-push`: Push notifications

---

## 2. Package Structure

### 2.1 Complete Feature List

```
packages/features/
├── Core Features (Business Logic)
│   ├── bookings/              ⭐ Booking management & creation
│   ├── auth/                  ⭐ Authentication & authorization
│   ├── calendars/             ⭐ Calendar integration layer
│   ├── eventtypes/            ⭐ Event type management
│   ├── availability/          Availability rules & schedules
│   ├── schedules/             Schedule management
│   ├── users/                 User management
│   ├── profile/               User profiles
│   ├── membership/            Team membership
│   ├── credentials/           API credentials
│   ├── hashedLink/            Private booking links
│   ├── instant-meeting/       Instant meetings
│   ├── noShow/                No-show tracking
│   └── links/                 Shareable links
│
├── Enterprise Features (ee/)
│   ├── organizations/         ⭐ Multi-tenant organizations
│   ├── teams/                 ⭐ Team management
│   ├── sso/                   ⭐ SAML SSO
│   ├── dsync/                 ⭐ Directory Sync (SCIM)
│   ├── workflows/             ⭐ Workflow automation
│   ├── payments/              Payment processing
│   ├── billing/               Subscription billing
│   ├── api-keys/              API key management
│   ├── managed-event-types/   Centralized event types
│   ├── round-robin/           Round-robin scheduling
│   ├── impersonation/         Admin impersonation
│   ├── platform/              Platform SDK
│   ├── video/                 Video conferencing
│   ├── deployment/            Deployment settings
│   ├── support/               Support features
│   ├── components/            EE UI components
│   ├── common/                Shared EE utilities
│   ├── users/                 EE user features
│   └── event-tracking/        Event analytics
│
├── Automation & Integration
│   ├── webhooks/              ⭐ Webhook management
│   ├── workflows/             Workflow orchestration (OSS)
│   ├── apps/                  ⭐ App marketplace
│   ├── conferencing/          Video conferencing
│   ├── crmManager/            CRM integration
│   ├── calendar-cache/        Calendar caching
│   └── calendar-subscription/ Calendar subscriptions
│
├── UI & Components
│   ├── components/            Shared feature components
│   ├── shell/                 App shell & layout
│   ├── form-builder/          Dynamic form builder
│   ├── form/                  Form utilities
│   ├── data-table/            Data table component
│   ├── kbar/                  Command palette (Cmd+K)
│   ├── filters/               Filter components
│   └── calendar-view/         Calendar view component
│
├── Utility & Infrastructure
│   ├── flags/                 Feature flags
│   ├── redis/                 Redis caching
│   ├── notifications/         Push notifications
│   ├── insights/              Analytics & insights
│   ├── busyTimes/             Busy time calculation
│   ├── cityTimezones/         Timezone utilities
│   ├── bot-detection/         Bot detection
│   ├── troubleshooter/        Debug tools
│   ├── tasker/                Background tasks
│   ├── watchlist/             Watchlist features
│   ├── delegation-credentials/ Delegated auth
│   ├── platform-oauth-client/ OAuth client
│   ├── di/                    Dependency injection
│   ├── pbac/                  Permission-based access control
│   ├── onboarding/            Onboarding flow
│   ├── settings/              Settings management
│   ├── routing-forms/         Routing forms
│   ├── billing/               Billing logic
│   ├── embed/                 Embed SDK
│   ├── timezone-buddy/        Timezone buddy
│   ├── tips/                  UI tips
│   ├── formbricks/            Formbricks integration
│   ├── mintlify-chat/         Mintlify chat widget
│   ├── calAIPhone/            AI phone features
│   └── video-call-guest/      Guest video call
```

### 2.2 Directory Structure Pattern

Most features follow this structure:

```
features/[feature-name]/
├── components/           # UI components specific to this feature
├── lib/                 # Business logic & utilities
│   ├── service/        # Service layer
│   ├── repository/     # Data access layer
│   ├── dto/            # Data transfer objects
│   └── interfaces/     # TypeScript interfaces
├── hooks/              # React hooks
├── pages/              # Page components (if any)
├── repositories/       # Prisma repositories
├── constants.ts        # Feature constants
├── types.ts            # TypeScript types
├── schema.ts           # Zod validation schemas
└── index.ts            # Public exports
```

---

## 3. Core Features

### 3.1 Bookings (`@calcom/features/bookings`)

**Purpose**: Handle all booking lifecycle operations.

**Key Files**:
```
bookings/
├── lib/
│   ├── handleNewBooking/            # Create new booking
│   ├── handleCancelBooking.ts       # Cancel booking
│   ├── handleConfirmation.ts        # Confirm pending booking
│   ├── EventManager.ts              # Calendar/video event creation
│   ├── BookingEmailSmsHandler.ts    # Email/SMS notifications
│   ├── checkBookingLimits.ts        # Enforce booking limits
│   ├── checkDurationLimits.ts       # Enforce duration limits
│   ├── getLuckyUser.ts              # Round-robin algorithm
│   ├── conflictChecker/             # Booking conflict detection
│   ├── handleSeats/                 # Seat-based bookings
│   └── payment/                     # Payment handling
├── Booker/                          # Main booking UI
└── components/                      # Booking UI components
```

**Main Responsibilities**:
1. **Booking Creation**: Validate, create booking, send calendar invites, send emails
2. **Booking Modification**: Reschedule, cancel, confirm
3. **Conflict Detection**: Check calendar availability
4. **Limit Enforcement**: Booking limits, duration limits, buffer times
5. **Round-Robin**: Assign bookings to team members
6. **Seats**: Multi-attendee bookings
7. **Payments**: Integration with payment providers
8. **Notifications**: Email/SMS reminders

**Example Flow**:
```typescript
// apps/web/pages/api/book/event.ts
import { handleNewBooking } from "@calcom/features/bookings/lib/handleNewBooking";

export default async function handler(req, res) {
  const booking = await handleNewBooking({
    req,
    res,
    eventTypeId: req.body.eventTypeId,
    responses: req.body.responses,
    timeZone: req.body.timeZone,
    start: req.body.start,
    end: req.body.end,
  });

  return res.status(200).json({ booking });
}
```

**Key Algorithms**:

**Round-Robin** (`getLuckyUser.ts`):
```typescript
// Weighted round-robin based on:
// 1. Manual priority
// 2. Least recently booked
// 3. Even distribution
export async function getLuckyUser(
  eventType: EventType,
  allAvailableUsers: User[]
) {
  // Filter by availability
  const availableUsers = await getAvailableUsers(eventType, allAvailableUsers);

  // Apply weights
  const weightedUsers = availableUsers.map(user => ({
    ...user,
    weight: calculateWeight(user, eventType),
  }));

  // Select user with highest weight
  return weightedUsers.sort((a, b) => b.weight - a.weight)[0];
}
```

**Conflict Detection** (`conflictChecker/checkForConflicts.ts`):
```typescript
export async function checkForConflicts({
  userId,
  start,
  end,
  eventTypeId,
}: ConflictCheckParams) {
  // 1. Get all calendar connections
  const calendars = await getConnectedCalendars(userId);

  // 2. Fetch busy times from all calendars
  const busyTimes = await Promise.all(
    calendars.map(cal => cal.getEvents(start, end))
  );

  // 3. Check for overlaps
  return busyTimes.some(event =>
    overlaps(event.start, event.end, start, end)
  );
}
```

**Location**: `packages/features/bookings/lib/handleNewBooking/handleNewBooking.ts:1`

---

### 3.2 Authentication (`@calcom/features/auth`)

**Purpose**: Handle user authentication and session management.

**Key Files**:
```
auth/
├── lib/
│   ├── next-auth-options.ts      # NextAuth configuration
│   ├── next-auth-custom-adapter.ts # Custom Prisma adapter
│   ├── getLocale.ts              # Locale detection
│   ├── getServerSession.ts       # Server-side session
│   ├── verifyPassword.ts         # Password verification
│   ├── isPasswordValid.ts        # Password validation
│   ├── signJwt.ts                # JWT signing
│   └── ErrorCode.ts              # Auth error codes
└── signup/                        # Signup flow
```

**Authentication Providers**:
1. **Email** (Magic Link)
2. **Google OAuth**
3. **SAML SSO** (Enterprise)
4. **CAL** (Custom provider)
5. **Impersonation** (Admin)

**NextAuth Configuration** (`next-auth-options.ts:1`):
```typescript
import { AuthOptions } from "next-auth";
import EmailProvider from "next-auth/providers/email";
import GoogleProvider from "next-auth/providers/google";
import CredentialsProvider from "next-auth/providers/credentials";

export const authOptions: AuthOptions = {
  adapter: CalComAdapter(prisma),
  providers: [
    // 1. Email (Magic Link)
    EmailProvider({
      server: process.env.EMAIL_SERVER,
      from: process.env.EMAIL_FROM,
      sendVerificationRequest: async ({ identifier, url }) => {
        await sendVerificationEmail({ email: identifier, url });
      },
    }),

    // 2. Google OAuth
    GoogleProvider({
      clientId: GOOGLE_CLIENT_ID,
      clientSecret: GOOGLE_CLIENT_SECRET,
      authorization: {
        params: {
          scope: GOOGLE_OAUTH_SCOPES.join(" "),
        },
      },
    }),

    // 3. Credentials (Email + Password)
    CredentialsProvider({
      credentials: {
        email: { type: "email" },
        password: { type: "password" },
      },
      authorize: async (credentials) => {
        const user = await UserRepository.findByEmail(credentials.email);
        if (!user || !user.password) return null;

        const isValid = await verifyPassword(
          credentials.password,
          user.password
        );

        if (!isValid) return null;
        return user;
      },
    }),
  ],

  session: {
    strategy: "jwt",
    maxAge: 30 * 24 * 60 * 60, // 30 days
  },

  callbacks: {
    async jwt({ token, user, account }) {
      // Enrich token with user data
      if (user) {
        token.id = user.id;
        token.email = user.email;
        token.role = user.role;
      }
      return token;
    },

    async session({ session, token }) {
      // Enrich session with token data
      session.user = {
        id: token.id,
        email: token.email,
        role: token.role,
      };
      return session;
    },

    async signIn({ user, account, profile }) {
      // Auto-link organization members
      if (ORGANIZATIONS_AUTOLINK && account.provider === "google") {
        await autoLinkToOrganization(user, profile.email);
      }
      return true;
    },
  },
};
```

**Session Flow**:
```
1. User signs in → NextAuth creates session
   ↓
2. Session stored in JWT cookie
   ↓
3. On each request → JWT decoded
   ↓
4. tRPC context enriched with session
   ↓
5. Middleware checks permissions
```

**Location**: `packages/features/auth/lib/next-auth-options.ts:1`

---

### 3.3 Event Types (`@calcom/features/eventtypes`)

**Purpose**: Manage event type configurations.

**Key Files**:
```
eventtypes/
├── components/
│   ├── EventTypeDescription/
│   ├── EventTypeSetup/
│   └── EventTypeList/
├── lib/
│   ├── getEventTypesFromDB.ts
│   ├── getEventType.ts
│   ├── validateCustomEventName.ts
│   └── getPublicEvent.ts
├── hooks/
│   └── useEventTypes.tsx
└── types.ts
```

**Event Type Schemas**:
```typescript
// Event Type Configuration
interface EventType {
  id: number;
  title: string;
  slug: string;
  length: number;                    // Duration in minutes

  // Scheduling
  schedulingType: SchedulingType;    // ROUND_ROBIN | COLLECTIVE | MANAGED
  minimumBookingNotice: number;      // Minutes before event
  slotInterval: number;              // Slot size (15, 30, 60 min)
  beforeEventBuffer: number;         // Buffer before event
  afterEventBuffer: number;          // Buffer after event

  // Limits
  periodType: PeriodType;            // UNLIMITED | ROLLING | RANGE
  periodDays: number;                // Rolling period
  periodStartDate: Date;             // Range start
  periodEndDate: Date;               // Range end
  periodCountCalendarDays: boolean;  // Include weekends

  // Booking Limits
  bookingLimits: {
    PER_DAY: number;
    PER_WEEK: number;
    PER_MONTH: number;
    PER_YEAR: number;
  };

  durationLimits: {
    PER_DAY: number;
    PER_WEEK: number;
    PER_MONTH: number;
    PER_YEAR: number;
  };

  // Location
  locations: Location[];

  // Team
  teamId: number | null;
  hosts: Host[];

  // Features
  requiresConfirmation: boolean;
  requiresBookerEmailVerification: boolean;
  hideCalendarNotes: boolean;
  lockTimeZoneToggleOnBookingPage: boolean;

  // Payments
  price: number;
  currency: string;

  // Workflows
  workflows: WorkflowsOnEventTypes[];
}
```

**Scheduling Types**:

| Type | Description | Use Case |
|------|-------------|----------|
| `ROUND_ROBIN` | Distribute bookings evenly | Sales teams, support |
| `COLLECTIVE` | All hosts must attend | Panel interviews |
| `MANAGED` | Centrally managed by admin | Enterprise teams |

**Location**: `packages/features/eventtypes/lib/getEventType.ts:1`

---

### 3.4 Calendars (`@calcom/features/calendars`)

**Purpose**: Calendar integration abstraction layer.

**Status**: Currently minimal, planning to migrate `CalendarManager` and `EventManager` here.

**Current Structure**:
```
calendars/
└── README.md    # Placeholder for future migration
```

**Note**: Calendar logic currently lives in:
- `packages/features/bookings/lib/EventManager.ts` (49KB)
- `packages/app-store/*/lib/CalendarService.ts` (per integration)

**Future Goals**:
1. Create unified calendar interface
2. Move `EventManager` to this package
3. Centralize calendar operations

---

### 3.5 Availability (`@calcom/features/availability`)

**Purpose**: Manage user availability and schedules.

**Key Files**:
```
availability/
├── lib/
│   ├── getAvailabilityFromSchedule.ts
│   ├── getBusyTimes.ts
│   ├── getDefaultSchedule.ts
│   └── getUserAvailability.ts
├── components/
│   ├── AvailabilityForm.tsx
│   └── ScheduleList.tsx
└── types.ts
```

**Availability Calculation**:
```typescript
export async function getUserAvailability({
  userId,
  eventTypeId,
  dateFrom,
  dateTo,
}: GetAvailabilityParams) {
  // 1. Get user's schedule
  const schedule = await getSchedule(userId, eventTypeId);

  // 2. Get busy times from calendars
  const busyTimes = await getBusyTimes(userId, dateFrom, dateTo);

  // 3. Calculate available slots
  const slots = calculateAvailableSlots({
    schedule,
    busyTimes,
    dateFrom,
    dateTo,
    slotInterval: eventType.slotInterval,
    beforeBuffer: eventType.beforeEventBuffer,
    afterBuffer: eventType.afterEventBuffer,
  });

  return slots;
}
```

**Schedule Structure**:
```typescript
interface Schedule {
  id: number;
  name: string;
  timeZone: string;
  availability: Availability[];
}

interface Availability {
  days: number[];        // [0, 1, 2, 3, 4] = Mon-Fri
  startTime: Time;       // "09:00"
  endTime: Time;         // "17:00"
  date?: Date;           // Override for specific date
}
```

---

## 4. Enterprise Features (EE)

### 4.1 Organizations (`@calcom/features/ee/organizations`)

**Purpose**: Multi-tenant organization management.

**Key Files**:
```
ee/organizations/
├── lib/
│   ├── orgDomains.ts                    # Subdomain routing
│   ├── onboardingStore.ts               # Onboarding state
│   ├── OrganizationPermissionService.ts # Permission checks
│   ├── OrganizationPaymentService.ts    # Billing
│   ├── getBrand.ts                      # Custom branding
│   └── server/                          # Server utilities
├── repositories/
│   └── OrganizationRepository.ts
├── pages/                               # Org settings pages
└── components/                          # Org UI components
```

**Organization Hierarchy**:
```
Organization
├── Profile (booker.cal.com)
├── Teams
│   ├── Team A
│   │   ├── Members
│   │   └── Event Types
│   └── Team B
└── Settings
    ├── Branding
    ├── SSO
    ├── SCIM
    └── Billing
```

**Subdomain Routing** (`orgDomains.ts:1`):
```typescript
export function getOrgFullOrigin(orgSlug: string | null) {
  if (!orgSlug) return WEBAPP_URL;

  // Subdomain: acme.cal.com
  if (ORGANIZATIONS_ENABLED) {
    return `https://${orgSlug}.${CAL_DOMAIN}`;
  }

  // Subpath: cal.com/org/acme
  return `${WEBAPP_URL}/org/${orgSlug}`;
}

export function isOrgRequest(req: NextRequest) {
  const hostname = req.headers.get("host");
  const isSubdomain = hostname !== CAL_DOMAIN && hostname.endsWith(`.${CAL_DOMAIN}`);
  return isSubdomain;
}
```

**Organization Features**:
- ✅ Custom subdomains (`acme.cal.com`)
- ✅ Custom branding (logo, colors)
- ✅ SSO (SAML)
- ✅ Directory Sync (SCIM)
- ✅ Centralized billing
- ✅ Team management
- ✅ Usage analytics

**Location**: `packages/features/ee/organizations/lib/orgDomains.ts:1`

---

### 4.2 Teams (`@calcom/features/ee/teams`)

**Purpose**: Team collaboration and scheduling.

**Key Files**:
```
ee/teams/
├── lib/
│   ├── getTeam.ts
│   ├── inviteTeamMember.ts
│   ├── removeTeamMember.ts
│   └── upgradeTeam.ts
├── components/
│   ├── TeamList.tsx
│   ├── TeamSettings.tsx
│   └── MemberList.tsx
└── pages/                  # Team pages
```

**Team Types**:
```typescript
enum MembershipRole {
  MEMBER = "MEMBER",
  ADMIN = "ADMIN",
  OWNER = "OWNER",
}

interface Membership {
  id: number;
  teamId: number;
  userId: number;
  role: MembershipRole;
  accepted: boolean;
  disableImpersonation: boolean;
}
```

**Team Features**:
- ✅ Collective events (all members attend)
- ✅ Round-robin events (distribute bookings)
- ✅ Managed events (admin-controlled)
- ✅ Team billing
- ✅ Shared event types
- ✅ Member permissions

**Location**: `packages/features/ee/teams/lib/getTeam.ts:1`

---

### 4.3 SSO (`@calcom/features/ee/sso`)

**Purpose**: SAML Single Sign-On for enterprise.

**Key Files**:
```
ee/sso/
├── lib/
│   ├── saml.ts                # SAML provider setup
│   ├── jackson.ts             # BoxyHQ SAML Jackson
│   └── samlAuth.ts            # SAML authentication
├── components/
│   └── SAMLConfiguration.tsx
└── pages/
    └── sso/[provider].tsx
```

**SAML Flow**:
```
1. User clicks "Sign in with SSO"
   ↓
2. Redirect to SAML IdP (Okta, Azure AD, etc.)
   ↓
3. IdP authenticates user
   ↓
4. IdP sends SAML assertion to Cal.com
   ↓
5. Cal.com validates assertion
   ↓
6. Create/update user account
   ↓
7. Create session
```

**Supported IdPs**:
- Okta
- Azure AD
- Google Workspace
- OneLogin
- Auth0
- Generic SAML 2.0

**Configuration**:
```typescript
interface SAMLConfig {
  organizationId: number;
  provider: string;              // "okta", "azure", etc.
  entityId: string;              // IdP Entity ID
  signInUrl: string;             // IdP SSO URL
  x509cert: string;              // IdP Certificate
  attributeMapping: {
    email: string;               // SAML attribute for email
    firstName: string;           // SAML attribute for first name
    lastName: string;            // SAML attribute for last name
  };
}
```

**Location**: `packages/features/ee/sso/lib/saml.ts:1`

---

### 4.4 Directory Sync (DSYNC) (`@calcom/features/ee/dsync`)

**Purpose**: SCIM 2.0 user provisioning and deprovisioning.

**Key Files**:
```
ee/dsync/
├── lib/
│   ├── scim.ts                      # SCIM handler
│   ├── users/
│   │   ├── createUsersAndConnectToOrg.ts
│   │   ├── deleteUser.ts
│   │   └── updateUser.ts
│   └── groups/
│       ├── createGroup.ts
│       └── deleteGroup.ts
└── api/                             # SCIM endpoints
```

**SCIM Operations**:
```typescript
// Create User
POST /scim/v2/Users
{
  "schemas": ["urn:ietf:params:scim:schemas:core:2.0:User"],
  "userName": "john.doe@acme.com",
  "name": {
    "givenName": "John",
    "familyName": "Doe"
  },
  "emails": [
    {
      "value": "john.doe@acme.com",
      "primary": true
    }
  ]
}

// Update User
PATCH /scim/v2/Users/{id}
{
  "schemas": ["urn:ietf:params:scim:api:messages:2.0:PatchOp"],
  "Operations": [
    {
      "op": "replace",
      "path": "active",
      "value": false
    }
  ]
}

// Delete User
DELETE /scim/v2/Users/{id}
```

**Supported IdPs**:
- Okta
- Azure AD
- OneLogin

**Location**: `packages/features/ee/dsync/lib/scim.ts:1`

---

### 4.5 Workflows (EE) (`@calcom/features/ee/workflows`)

**Purpose**: Advanced workflow automation (reminders, webhooks, actions).

**Key Files**:
```
ee/workflows/
├── lib/
│   ├── actionHelperFunctions.ts     # Workflow actions
│   ├── getAllWorkflows.ts
│   ├── getWorkflowReminders.ts
│   ├── reminders/
│   │   ├── smsReminderManager.ts   # SMS reminders
│   │   ├── emailReminderManager.ts # Email reminders
│   │   └── whatsappReminderManager.ts
│   ├── service/
│   └── test/
├── components/
│   ├── WorkflowEditor.tsx
│   └── WorkflowList.tsx
└── pages/
```

**Workflow Triggers**:
- `BEFORE_EVENT`: X hours/days before event
- `EVENT_CANCELLED`: When event is cancelled
- `NEW_EVENT`: When new event is created
- `RESCHEDULE_EVENT`: When event is rescheduled

**Workflow Actions**:
1. **Email Reminder** (`EMAIL_HOST`, `EMAIL_ATTENDEE`)
2. **SMS Reminder** (`SMS_ATTENDEE`, `SMS_NUMBER`)
3. **WhatsApp Reminder** (`WHATSAPP_ATTENDEE`, `WHATSAPP_NUMBER`)
4. **Webhook** (`WEBHOOK`)

**Workflow Schema**:
```typescript
interface Workflow {
  id: number;
  name: string;
  trigger: WorkflowTrigger;
  time: number;                      // Minutes before/after
  timeUnit: TimeUnit;                // MINUTE | HOUR | DAY
  steps: WorkflowStep[];
  activeOn: {
    eventTypeId?: number;            // Specific event type
    userId?: number;                 // All user events
    teamId?: number;                 // All team events
  };
}

interface WorkflowStep {
  id: number;
  stepNumber: number;
  action: WorkflowAction;
  template: WorkflowTemplate;        // Email/SMS template
  reminderBody: string;
  sender: string;                    // Sender ID for SMS
  senderName: string;
}

enum WorkflowAction {
  EMAIL_HOST = "EMAIL_HOST",
  EMAIL_ATTENDEE = "EMAIL_ATTENDEE",
  SMS_ATTENDEE = "SMS_ATTENDEE",
  WHATSAPP_ATTENDEE = "WHATSAPP_ATTENDEE",
}
```

**Reminder Variables**:
```
{EVENT_NAME}      - Event type title
{ORGANIZER}       - Organizer name
{ATTENDEE}        - Attendee name
{EVENT_DATE}      - Event date
{EVENT_TIME}      - Event time
{LOCATION}        - Event location
{CANCEL_LINK}     - Cancellation link
{RESCHEDULE_LINK} - Reschedule link
```

**Example Workflow**:
```typescript
// 24-hour email reminder
{
  name: "24-hour reminder",
  trigger: "BEFORE_EVENT",
  time: 24,
  timeUnit: "HOUR",
  steps: [
    {
      action: "EMAIL_ATTENDEE",
      template: "REMINDER",
      reminderBody: `
        Hi {ATTENDEE},

        This is a reminder for your {EVENT_NAME} with {ORGANIZER}
        on {EVENT_DATE} at {EVENT_TIME}.

        Location: {LOCATION}

        Need to reschedule? {RESCHEDULE_LINK}
      `,
    }
  ]
}
```

**Location**: `packages/features/ee/workflows/lib/actionHelperFunctions.ts:1`

---

### 4.6 Payments (`@calcom/features/ee/payments`)

**Purpose**: Payment processing for paid events.

**Key Files**:
```
ee/payments/
├── lib/
│   ├── stripe/
│   │   ├── createCheckoutSession.ts
│   │   ├── handlePaymentSuccess.ts
│   │   └── handleRefund.ts
│   ├── paypal/
│   └── alby/                        # Bitcoin payments
└── components/
    └── PaymentForm.tsx
```

**Payment Flow**:
```
1. User books paid event
   ↓
2. Create pending booking
   ↓
3. Redirect to payment page (Stripe Checkout)
   ↓
4. User completes payment
   ↓
5. Webhook receives payment confirmation
   ↓
6. Confirm booking
   ↓
7. Send calendar invite & emails
```

**Supported Payment Providers**:
- Stripe
- PayPal
- Alby (Bitcoin Lightning)

**Location**: `packages/features/ee/payments/lib/stripe/createCheckoutSession.ts:1`

---

## 5. Automation & Integration

### 5.1 Webhooks (`@calcom/features/webhooks`)

**Purpose**: Send event data to external systems.

**Key Files**:
```
webhooks/
├── lib/
│   ├── sendPayload.ts               # HTTP webhook delivery
│   ├── scheduleTrigger.ts           # Scheduled webhooks
│   ├── WebhookService.ts            # Webhook management
│   ├── factory/
│   │   ├── EventPayloadFactory.ts  # Payload construction
│   │   └── BookingPayloadFactory.ts
│   ├── service/
│   └── repository/
└── components/
    └── WebhookForm.tsx
```

**Webhook Events**:
```typescript
enum WebhookTriggerEvents {
  BOOKING_CREATED = "BOOKING_CREATED",
  BOOKING_RESCHEDULED = "BOOKING_RESCHEDULED",
  BOOKING_CANCELLED = "BOOKING_CANCELLED",
  BOOKING_REJECTED = "BOOKING_REJECTED",
  BOOKING_REQUESTED = "BOOKING_REQUESTED",
  BOOKING_PAYMENT_INITIATED = "BOOKING_PAYMENT_INITIATED",
  BOOKING_PAID = "BOOKING_PAID",
  MEETING_ENDED = "MEETING_ENDED",
  MEETING_STARTED = "MEETING_STARTED",
  RECORDING_READY = "RECORDING_READY",
  FORM_SUBMITTED = "FORM_SUBMITTED",
  INSTANT_MEETING = "INSTANT_MEETING",
}
```

**Webhook Payload Example**:
```json
{
  "triggerEvent": "BOOKING_CREATED",
  "createdAt": "2025-11-18T10:00:00Z",
  "payload": {
    "type": "Cal.com",
    "title": "30 Min Meeting",
    "description": "",
    "customInputs": {},
    "startTime": "2025-11-19T14:00:00Z",
    "endTime": "2025-11-19T14:30:00Z",
    "organizer": {
      "id": 1,
      "name": "John Doe",
      "email": "john@example.com",
      "timeZone": "America/New_York",
      "language": "en"
    },
    "attendees": [
      {
        "email": "jane@example.com",
        "name": "Jane Smith",
        "timeZone": "America/Los_Angeles",
        "language": "en"
      }
    ],
    "location": "Zoom",
    "uid": "abc123",
    "bookingId": 456,
    "status": "ACCEPTED",
    "metadata": {}
  }
}
```

**Webhook Delivery**:
```typescript
export async function sendPayload({
  webhook,
  payload,
  retries = 3,
}: SendPayloadParams) {
  const maxRetries = retries;
  let attempt = 0;

  while (attempt < maxRetries) {
    try {
      const response = await fetch(webhook.subscriberUrl, {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          "X-Cal-Signature-256": generateSignature(webhook.secret, payload),
        },
        body: JSON.stringify(payload),
      });

      if (response.ok) {
        await logWebhookSuccess(webhook.id);
        return;
      }

      throw new Error(`HTTP ${response.status}`);
    } catch (error) {
      attempt++;
      if (attempt >= maxRetries) {
        await logWebhookFailure(webhook.id, error);
        throw error;
      }
      await sleep(2 ** attempt * 1000); // Exponential backoff
    }
  }
}
```

**Security**:
- HMAC SHA-256 signature (`X-Cal-Signature-256` header)
- Secret validation
- Rate limiting

**Location**: `packages/features/webhooks/lib/sendPayload.ts:1`

---

### 5.2 Apps / Integrations (`@calcom/features/apps`)

**Purpose**: Manage app marketplace integrations.

**Key Files**:
```
apps/
├── lib/
│   ├── getApps.ts
│   ├── getInstalledApps.ts
│   └── getAppKeysFromSlug.ts
├── components/
│   ├── AppCard.tsx
│   └── AppStoreCategories.tsx
└── pages/
    └── apps/[slug].tsx
```

**App Categories**:
- Calendar (Google, Outlook, iCloud, CalDAV)
- Video (Zoom, Google Meet, MS Teams, Daily.co)
- Payment (Stripe, PayPal, Alby)
- CRM (Salesforce, HubSpot, Pipedrive)
- Analytics (Google Analytics, Plausible)
- Automation (Zapier, Make, n8n)
- Messaging (Slack, WhatsApp, Telegram)

**App Structure**:
```typescript
interface App {
  slug: string;
  name: string;
  description: string;
  type: AppType;                     // calendar, video, payment, etc.
  categories: AppCategory[];
  logo: string;
  publisher: string;
  email: string;
  dirName: string;
  variant: "calendar" | "conferencing" | "payment" | "other";
  extendsFeature: string;
  credentials: Credential[];
}
```

**Location**: `packages/features/apps/lib/getApps.ts:1`

---

## 6. UI & Component Features

### 6.1 Shell (`@calcom/features/shell`)

**Purpose**: App shell, navigation, and layout.

**Key Files**:
```
shell/
├── Shell.tsx                        # Main app shell
├── ShellMain.tsx                    # Content area
├── ShellSubHeading.tsx              # Page subheading
├── SettingsLayout.tsx               # Settings layout
└── navigation/
    ├── Navigation.tsx               # Sidebar navigation
    └── MobileNavigation.tsx
```

**Shell Structure**:
```tsx
<Shell
  heading="Event Types"
  subtitle="Create and manage your event types"
  CTA={<CreateEventTypeButton />}
  backPath="/dashboard"
>
  {/* Page content */}
</Shell>
```

**Location**: `packages/features/shell/Shell.tsx:1`

---

### 6.2 Form Builder (`@calcom/features/form-builder`)

**Purpose**: Dynamic form builder for booking questions.

**Key Files**:
```
form-builder/
├── FormBuilder.tsx
├── FieldEditor.tsx
├── FieldTypes.tsx
└── utils.ts
```

**Field Types**:
- Text
- Email
- Phone
- Textarea
- Number
- Select (dropdown)
- MultiSelect (checkboxes)
- Radio
- Boolean (yes/no)

**Example**:
```typescript
const bookingFields = [
  {
    type: "text",
    name: "name",
    label: "Your Name",
    required: true,
  },
  {
    type: "email",
    name: "email",
    label: "Email Address",
    required: true,
  },
  {
    type: "select",
    name: "industry",
    label: "Industry",
    options: ["Technology", "Finance", "Healthcare", "Other"],
  },
];
```

**Location**: `packages/features/form-builder/FormBuilder.tsx:1`

---

### 6.3 KBar (Command Palette) (`@calcom/features/kbar`)

**Purpose**: Keyboard-driven command palette (Cmd+K).

**Key Files**:
```
kbar/
├── Kbar.tsx
├── KbarActions.tsx
└── useKbar.tsx
```

**Commands**:
- Create event type
- View bookings
- Go to settings
- Search...

**Usage**:
- `Cmd+K` (Mac) or `Ctrl+K` (Windows) to open
- Type to search
- Arrow keys to navigate
- Enter to execute

**Location**: `packages/features/kbar/Kbar.tsx:1`

---

## 7. Utility & Infrastructure Features

### 7.1 Feature Flags (`@calcom/features/flags`)

**Purpose**: Feature flag management for gradual rollouts.

**Key Files**:
```
flags/
├── config.ts                        # Flag definitions
├── provider.tsx                     # Flag provider
└── useFlag.tsx                      # Hook to check flags
```

**Example Flags**:
```typescript
enum FeatureFlag {
  ORGANIZATIONS = "organizations",
  PLATFORM = "platform",
  TEAMS = "teams",
  WORKFLOWS = "workflows",
  WEBHOOKS_V2 = "webhooks-v2",
}

// Usage
const isOrgsEnabled = useFlag("organizations");

if (isOrgsEnabled) {
  // Show organizations UI
}
```

**Location**: `packages/features/flags/config.ts:1`

---

### 7.2 Redis Cache (`@calcom/features/redis`)

**Purpose**: Redis caching layer for performance.

**Key Files**:
```
redis/
├── client.ts                        # Redis client
├── cache.ts                         # Cache utilities
└── types.ts
```

**Use Cases**:
- Rate limiting
- Session caching
- Booking availability cache
- App credentials cache

**Example**:
```typescript
import { redis } from "@calcom/features/redis/client";

// Cache availability for 5 minutes
await redis.set(
  `availability:${userId}:${date}`,
  JSON.stringify(slots),
  "EX",
  300
);

// Get cached availability
const cached = await redis.get(`availability:${userId}:${date}`);
```

**Location**: `packages/features/redis/client.ts:1`

---

### 7.3 Insights (`@calcom/features/insights`)

**Purpose**: Analytics and reporting.

**Key Files**:
```
insights/
├── components/
│   ├── EventTypeInsights.tsx
│   ├── BookingChart.tsx
│   └── TeamInsights.tsx
├── lib/
│   ├── getInsights.ts
│   └── aggregations.ts
└── types.ts
```

**Metrics**:
- Total bookings
- Booking rate
- No-show rate
- Popular time slots
- Revenue (for paid events)
- Team performance

**Location**: `packages/features/insights/lib/getInsights.ts:1`

---

## 8. Feature Dependencies

### 8.1 Dependency Graph

```
@calcom/features
├── Depends on:
│   ├── @calcom/lib        (utilities)
│   ├── @calcom/prisma     (database)
│   ├── @calcom/trpc       (API layer)
│   ├── @calcom/ui         (UI components)
│   ├── @calcom/atoms      (SDK components)
│   └── @calcom/dayjs      (date utilities)
│
└── Used by:
    ├── apps/web           (main Next.js app)
    ├── apps/api/v2        (NestJS API)
    └── packages/app-store (integrations)
```

### 8.2 Internal Feature Dependencies

```
bookings
├── → auth            (session)
├── → calendars       (event creation)
├── → webhooks        (notifications)
├── → workflows       (reminders)
├── → payments        (paid events)
└── → ee/workflows    (advanced automation)

auth
├── → users           (user management)
├── → ee/sso          (SAML SSO)
└── → ee/organizations (org linking)

ee/organizations
├── → auth            (SSO)
├── → ee/teams        (team management)
├── → ee/dsync        (user provisioning)
└── → ee/billing      (subscription)
```

---

## 9. Adding New Features

### 9.1 Feature Scaffold

```bash
# Create new feature
mkdir packages/features/my-feature
cd packages/features/my-feature

# Create structure
mkdir -p lib/{service,repository,dto}
mkdir components
mkdir hooks
touch index.ts types.ts schema.ts constants.ts
```

### 9.2 Feature Template

```typescript
// packages/features/my-feature/index.ts
export * from "./components";
export * from "./lib";
export * from "./hooks";
export * from "./types";

// packages/features/my-feature/types.ts
import { z } from "zod";

export const myFeatureSchema = z.object({
  id: z.number(),
  name: z.string(),
});

export type MyFeature = z.infer<typeof myFeatureSchema>;

// packages/features/my-feature/lib/service/MyFeatureService.ts
import { MyFeatureRepository } from "../repository/MyFeatureRepository";

export class MyFeatureService {
  constructor(private repository: MyFeatureRepository) {}

  async getById(id: number) {
    return this.repository.findById(id);
  }

  async create(data: CreateMyFeatureDTO) {
    return this.repository.create(data);
  }
}

// packages/features/my-feature/lib/repository/MyFeatureRepository.ts
import prisma from "@calcom/prisma";

export class MyFeatureRepository {
  async findById(id: number) {
    return prisma.myFeature.findUnique({ where: { id } });
  }

  async create(data: CreateMyFeatureDTO) {
    return prisma.myFeature.create({ data });
  }
}
```

### 9.3 Integration with tRPC

```typescript
// packages/trpc/server/routers/viewer/myFeature/_router.ts
import { authedProcedure, router } from "../../../trpc";
import { myFeatureSchema } from "@calcom/features/my-feature/types";
import { MyFeatureService } from "@calcom/features/my-feature/lib/service";

export const myFeatureRouter = router({
  list: authedProcedure.query(async ({ ctx }) => {
    const service = new MyFeatureService(ctx.prisma);
    return service.list();
  }),

  get: authedProcedure
    .input(z.object({ id: z.number() }))
    .query(async ({ input, ctx }) => {
      const service = new MyFeatureService(ctx.prisma);
      return service.getById(input.id);
    }),

  create: authedProcedure
    .input(myFeatureSchema)
    .mutation(async ({ input, ctx }) => {
      const service = new MyFeatureService(ctx.prisma);
      return service.create(input);
    }),
});

// packages/trpc/server/routers/viewer/_router.tsx
import { myFeatureRouter } from "./myFeature/_router";

export const viewerRouter = router({
  // ... existing routers
  myFeature: myFeatureRouter,
});
```

---

## 10. Best Practices

### 10.1 Code Organization

```typescript
// ✅ Good: Clear separation of concerns
features/booking/
├── lib/
│   ├── service/           # Business logic
│   ├── repository/        # Data access
│   └── dto/               # Data transfer objects
├── components/            # UI components
└── hooks/                 # React hooks

// ❌ Bad: Mixed responsibilities
features/booking/
├── utils.ts              # Everything in one file
└── components.tsx
```

### 10.2 Dependency Injection

```typescript
// ✅ Good: Use dependency injection
export class BookingService {
  constructor(
    private bookingRepo: BookingRepository,
    private emailService: EmailService
  ) {}

  async create(data: CreateBookingDTO) {
    const booking = await this.bookingRepo.create(data);
    await this.emailService.sendConfirmation(booking);
    return booking;
  }
}

// ❌ Bad: Direct dependencies
export async function createBooking(data: CreateBookingDTO) {
  const booking = await prisma.booking.create({ data });
  await sendEmail({ to: booking.email });
  return booking;
}
```

### 10.3 Schema Validation

```typescript
// ✅ Good: Zod schema validation
import { z } from "zod";

export const bookingSchema = z.object({
  eventTypeId: z.number().positive(),
  start: z.string().datetime(),
  end: z.string().datetime(),
  responses: z.record(z.any()),
});

export type BookingInput = z.infer<typeof bookingSchema>;

// Validate input
const validated = bookingSchema.parse(input);

// ❌ Bad: No validation
export async function createBooking(input: any) {
  // Hope for the best
}
```

### 10.4 Error Handling

```typescript
// ✅ Good: Custom error types
export class BookingError extends Error {
  constructor(
    message: string,
    public code: BookingErrorCode,
    public statusCode: number = 400
  ) {
    super(message);
    this.name = "BookingError";
  }
}

export enum BookingErrorCode {
  EVENT_TYPE_NOT_FOUND = "EVENT_TYPE_NOT_FOUND",
  NO_AVAILABILITY = "NO_AVAILABILITY",
  BOOKING_LIMIT_EXCEEDED = "BOOKING_LIMIT_EXCEEDED",
}

// Usage
if (!eventType) {
  throw new BookingError(
    "Event type not found",
    BookingErrorCode.EVENT_TYPE_NOT_FOUND,
    404
  );
}

// ❌ Bad: Generic errors
throw new Error("Something went wrong");
```

### 10.5 Testing

```typescript
// ✅ Good: Unit tests with mocks
import { describe, it, expect, vi } from "vitest";
import { BookingService } from "./BookingService";

describe("BookingService", () => {
  it("should create booking and send email", async () => {
    const mockRepo = {
      create: vi.fn().mockResolvedValue({ id: 1 }),
    };
    const mockEmailService = {
      sendConfirmation: vi.fn(),
    };

    const service = new BookingService(mockRepo, mockEmailService);
    const booking = await service.create({ eventTypeId: 1 });

    expect(mockRepo.create).toHaveBeenCalled();
    expect(mockEmailService.sendConfirmation).toHaveBeenCalledWith(booking);
  });
});
```

---

## Summary

### Key Takeaways

1. **55 Feature Packages**: Modular organization by domain, all under `@calcom/features`

2. **Core Features**:
   - **Bookings**: Complete booking lifecycle
   - **Auth**: NextAuth-based authentication
   - **Event Types**: Event configuration
   - **Availability**: Schedule management

3. **Enterprise Features**:
   - **Organizations**: Multi-tenancy with subdomains
   - **Teams**: Collaborative scheduling
   - **SSO**: SAML integration
   - **DSYNC**: SCIM user provisioning
   - **Workflows**: Advanced automation

4. **Architecture Patterns**:
   - Service layer for business logic
   - Repository layer for data access
   - DTO for data transfer
   - Zod for validation

5. **Integration Points**:
   - tRPC for API exposure
   - Webhooks for external systems
   - Apps for marketplace integrations

### Feature Comparison Matrix

| Feature | OSS | Enterprise | Key Use Case |
|---------|-----|------------|--------------|
| Bookings | ✅ | ✅ | Core scheduling |
| Event Types | ✅ | ✅ | Event configuration |
| Teams | ✅ | ✅ | Team scheduling |
| Webhooks | ✅ | ✅ | Integration |
| Workflows (Basic) | ✅ | ✅ | Simple automation |
| Organizations | ❌ | ✅ | Multi-tenancy |
| SSO | ❌ | ✅ | Enterprise auth |
| DSYNC | ❌ | ✅ | User provisioning |
| Workflows (Advanced) | ❌ | ✅ | SMS, WhatsApp |
| Managed Events | ❌ | ✅ | Centralized control |

### Next Steps

- **PHASE 6**: App Store & Integration Architecture
- **PHASE 7**: Booking Flow End-to-End
- **PHASE 8**: Authentication & Authorization

---

**Tác giả**: Claude (AI Assistant)
**Ngày tạo**: 2025-11-18
**Phiên bản**: 1.0
