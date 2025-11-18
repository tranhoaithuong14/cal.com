# PHASE 11: Enterprise Features (EE)

> **Mục tiêu**: Hiểu các tính năng Enterprise Edition trong `packages/features/ee/`, bao gồm SSO, DSYNC, Organizations, Billing, Workflows, và các tính năng nâng cao khác.

## Overview

Cal.com có **phiên bản Enterprise** với các tính năng nâng cao cho doanh nghiệp lớn:

```
Enterprise Features:
┌────────────────────────────────────────────────────────────┐
│                    packages/features/ee/                   │
├──────────────────────┬─────────────────────────────────────┤
│                      │                                     │
│  SSO                 │  SAML/OIDC authentication           │
│  DSYNC               │  SCIM directory synchronization     │
│  Organizations       │  Multi-tenant hierarchy             │
│  Billing             │  Stripe subscriptions               │
│  Workflows           │  Advanced automation                │
│  API Keys            │  Programmatic access                │
│  Teams (Advanced)    │  Enhanced team features             │
│  Video (Premium)     │  Premium video integrations         │
│                      │                                     │
└──────────────────────┴─────────────────────────────────────┘
```

**Enterprise vs Free**:

| Feature | Free | Enterprise |
|---------|------|------------|
| **SSO (SAML/OIDC)** | ❌ | ✅ |
| **Directory Sync (SCIM)** | ❌ | ✅ |
| **Organizations** | ❌ | ✅ |
| **Workflows** | Limited | Advanced |
| **API Keys** | ❌ | ✅ |
| **White-labeling** | ❌ | ✅ |
| **Admin Console** | Basic | Advanced |
| **SLA Support** | Community | Priority |

---

## 1. SSO (Single Sign-On)

### 1.1 Overview

**Technology**: BoxyHQ SAML Jackson

**Supported Protocols**:
- SAML 2.0
- OIDC (OpenID Connect)

**Location**: `packages/features/ee/sso/`

### 1.2 SAML Configuration

```typescript
// Environment variables
SAML_DATABASE_URL=postgresql://user:pass@host:5432/saml_db
SAML_ADMINS=admin@company.com,it@company.com
SAML_CLIENT_SECRET_VERIFIER=your-secret-key

// Constants
export const samlTenantID = "Cal.com";
export const samlProductID = "Cal.com";
export const samlAudience = "https://saml.cal.com";
export const samlPath = "/api/auth/saml/callback";
export const oidcPath = "/api/auth/oidc";
```

**Location**: `packages/features/ee/sso/lib/saml.ts:7-15`

### 1.3 SAML Flow

```
SAML Authentication Flow:
┌──────────┐                ┌──────────┐                ┌──────────┐
│  User    │                │ Cal.com  │                │   IdP    │
│          │                │          │                │ (Okta,   │
│          │                │          │                │  Azure)  │
└────┬─────┘                └────┬─────┘                └────┬─────┘
     │                           │                           │
     │ 1. Visit Cal.com          │                           │
     ├──────────────────────────>│                           │
     │                           │                           │
     │ 2. Redirect to IdP        │                           │
     │<──────────────────────────┤                           │
     │                           │                           │
     │ 3. Login at IdP           │                           │
     ├───────────────────────────────────────────────────────>│
     │                           │                           │
     │ 4. SAML Response          │                           │
     │<───────────────────────────────────────────────────────┤
     │                           │                           │
     │ 5. POST SAML Response     │                           │
     ├──────────────────────────>│                           │
     │                           │                           │
     │                           │ 6. Validate & Create Session
     │                           │                           │
     │ 7. Authenticated          │                           │
     │<──────────────────────────┤                           │
     │                           │                           │
```

### 1.4 SAML Implementation

```typescript
// packages/features/ee/sso/lib/saml.ts

export const canAccessOrganization = async (
  user: { id: number; email: string },
  teamId: number | null
) => {
  const { id: userId, email } = user;

  // Check if SAML is enabled
  if (!isSAMLLoginEnabled) {
    return {
      message: "To enable this feature, add value for `SAML_DATABASE_URL` and `SAML_ADMINS` to your `.env`",
      access: false,
    };
  }

  // Hosted Cal.com: Check PBAC permissions
  if (HOSTED_CAL_FEATURES) {
    if (teamId === null) {
      return { message: "dont_have_permission", access: false };
    }

    const permissionCheckService = new PermissionCheckService();
    const hasPermission = await permissionCheckService.checkPermission({
      userId,
      teamId,
      permission: "organization.read",
      fallbackRoles: [MembershipRole.OWNER, MembershipRole.ADMIN],
    });

    if (!hasPermission) {
      return { message: "dont_have_permission", access: false };
    }
  }

  // Self-hosted: Check SAML_ADMINS
  if (!HOSTED_CAL_FEATURES) {
    if (!isSAMLAdmin(email)) {
      return { message: "dont_have_permission", access: false };
    }
  }

  return { message: "success", access: true };
};
```

**Location**: `packages/features/ee/sso/lib/saml.ts:32-81`

### 1.5 NextAuth Integration

```typescript
// packages/features/auth/lib/next-auth-options.ts

import { SAMLProvider } from "@calcom/features/ee/sso/lib/saml";

export const providers = [
  // ... other providers

  SAMLProvider({
    id: "saml",
    name: "BoxyHQ",
    type: "oauth",
    version: "2.0",
    checks: ["pkce", "state"],
    authorization: {
      url: `${SAML_PROVIDER_URL}/api/oauth/authorize`,
      params: {
        scope: "",
        response_type: "code",
        provider: "saml",
      },
    },
    token: {
      url: `${SAML_PROVIDER_URL}/api/oauth/token`,
    },
    userinfo: {
      url: `${SAML_PROVIDER_URL}/api/oauth/userinfo`,
    },
    profile(profile) {
      return {
        id: profile.id || profile.email,
        email: profile.email,
        name: profile.firstName + " " + profile.lastName,
      };
    },
  }),
];
```

### 1.6 OIDC Support

OIDC is also supported for providers like Azure AD:

```typescript
// OIDC endpoint
export const oidcPath = "/api/auth/oidc";

// Connection type
export type SSOConnection = (SAMLSSORecord | OIDCSSORecord) & {
  type: string;
  acsUrl: string | null;
  entityId: string | null;
  callbackUrl: string | null;
};
```

**Location**: `packages/features/ee/sso/lib/saml.ts:14,83-88`

---

## 2. DSYNC (Directory Sync)

### 2.1 Overview

**DSYNC** (Directory Synchronization) sử dụng **SCIM 2.0** protocol để đồng bộ users/groups từ directory providers (Okta, Azure AD, Google Workspace) vào Cal.com.

**Technology**: BoxyHQ Directory Sync

**Supported Providers**:
- Okta
- Azure AD
- Google Workspace
- OneLogin
- JumpCloud

**Location**: `packages/features/ee/dsync/`

### 2.2 SCIM Protocol

**SCIM** (System for Cross-domain Identity Management) là chuẩn REST API để quản lý user lifecycle.

```
SCIM Operations:
┌─────────────────┬──────────────────────────────────────┐
│ Operation       │ Description                          │
├─────────────────┼──────────────────────────────────────┤
│ GET /Users      │ List all users                       │
│ GET /Users/:id  │ Get specific user                    │
│ POST /Users     │ Create new user                      │
│ PUT /Users/:id  │ Update user (replace)                │
│ PATCH /Users/:id│ Partial update user                  │
│ DELETE /Users/:id│ Delete/deprovision user             │
│                 │                                      │
│ GET /Groups     │ List all groups                      │
│ POST /Groups    │ Create group                         │
│ PATCH /Groups/:id│ Add/remove group members           │
└─────────────────┴──────────────────────────────────────┘
```

### 2.3 DSYNC Flow

```
Directory Sync Flow:
┌──────────────┐         ┌──────────────┐         ┌──────────────┐
│  Directory   │         │   Cal.com    │         │   Database   │
│  Provider    │         │   DSYNC      │         │              │
│  (Okta)      │         │              │         │              │
└──────┬───────┘         └──────┬───────┘         └──────┬───────┘
       │                        │                        │
       │ 1. User created in Okta│                        │
       │                        │                        │
       │ 2. SCIM POST /Users    │                        │
       ├───────────────────────>│                        │
       │                        │                        │
       │                        │ 3. Extract attributes  │
       │                        │    (name, email, etc.) │
       │                        │                        │
       │                        │ 4. Create Cal user     │
       │                        ├───────────────────────>│
       │                        │                        │
       │                        │ 5. Add to organization │
       │                        ├───────────────────────>│
       │                        │                        │
       │ 6. SCIM Response       │                        │
       │<───────────────────────┤                        │
       │                        │                        │
       │                        │                        │
       │ 7. Group membership    │                        │
       │    PATCH /Groups/:id   │                        │
       ├───────────────────────>│                        │
       │                        │                        │
       │                        │ 8. Add user to team    │
       │                        ├───────────────────────>│
       │                        │                        │
```

### 2.4 User Events Handler

```typescript
// packages/features/ee/dsync/lib/handleUserEvents.ts

export async function handleUserEvents(
  event: {
    event: "user.created" | "user.updated" | "user.deleted";
    tenant: string;
    data: ScimUser;
  },
  orgId: number
) {
  const { event: eventType, data } = event;

  switch (eventType) {
    case "user.created":
      // Extract attributes from SCIM payload
      const attributes = getAttributesFromScimPayload(data);

      // Create user in Cal.com
      const user = await prisma.user.create({
        data: {
          email: attributes.email,
          name: attributes.name,
          organizationId: orgId,
          role: UserPermissionRole.USER,
        },
      });

      // Add to organization
      await inviteExistingUserToOrg({
        userId: user.id,
        organizationId: orgId,
      });

      break;

    case "user.updated":
      // Update user attributes
      await assignValueToUser({
        userId: data.id,
        attributes: getAttributesFromScimPayload(data),
      });
      break;

    case "user.deleted":
      // Remove from organization (soft delete)
      await removeUserFromOrg({
        userId: data.id,
        organizationId: orgId,
      });
      break;
  }
}
```

### 2.5 Group Events Handler

```typescript
// packages/features/ee/dsync/lib/handleGroupEvents.ts

export async function handleGroupEvents(
  event: {
    event: "group.created" | "group.updated" | "group.deleted" | "group.user_added" | "group.user_removed";
    tenant: string;
    data: ScimGroup;
  },
  orgId: number
) {
  const { event: eventType, data } = event;

  switch (eventType) {
    case "group.created":
      // Create team in Cal.com
      await prisma.team.create({
        data: {
          name: data.displayName,
          slug: slugify(data.displayName),
          organizationId: orgId,
        },
      });
      break;

    case "group.user_added":
      // Add user to team
      await prisma.membership.create({
        data: {
          userId: data.userId,
          teamId: data.groupId,
          role: MembershipRole.MEMBER,
          accepted: true, // Auto-accept for DSYNC
        },
      });
      break;

    case "group.user_removed":
      // Remove user from team
      await prisma.membership.delete({
        where: {
          userId_teamId: {
            userId: data.userId,
            teamId: data.groupId,
          },
        },
      });
      break;
  }
}
```

### 2.6 Directory Providers

```typescript
// packages/features/ee/dsync/lib/directoryProviders.ts

export const DIRECTORY_PROVIDERS = [
  {
    id: "okta",
    name: "Okta",
    scimVersion: "2.0",
    supportsGroups: true,
  },
  {
    id: "azure",
    name: "Azure AD",
    scimVersion: "2.0",
    supportsGroups: true,
  },
  {
    id: "google",
    name: "Google Workspace",
    scimVersion: "2.0",
    supportsGroups: true,
  },
  {
    id: "onelogin",
    name: "OneLogin",
    scimVersion: "2.0",
    supportsGroups: true,
  },
  {
    id: "jumpcloud",
    name: "JumpCloud",
    scimVersion: "2.0",
    supportsGroups: true,
  },
];
```

---

## 3. Organizations (Multi-Tenancy)

### 3.1 Organization Hierarchy

```
Organization Hierarchy:
┌──────────────────────────────────────────────────────────┐
│                      Organization                        │
│                 (acme.cal.com)                           │
│                                                          │
│  ┌────────────────────────────────────────────────────┐ │
│  │                    Profiles                        │ │
│  │  (Sub-organizations with custom domains)           │ │
│  │                                                    │ │
│  │  ┌──────────────┐  ┌──────────────┐               │ │
│  │  │ Profile 1    │  │ Profile 2    │               │ │
│  │  │ sales.acme   │  │ support.acme │               │ │
│  │  └──────┬───────┘  └──────┬───────┘               │ │
│  │         │                  │                       │ │
│  │    ┌────┴────┐        ┌────┴────┐                 │ │
│  │    │ Teams   │        │ Teams   │                 │ │
│  │    └────┬────┘        └────┬────┘                 │ │
│  │         │                  │                       │ │
│  │    ┌────┴────┐        ┌────┴────┐                 │ │
│  │    │ Users   │        │ Users   │                 │ │
│  │    └─────────┘        └─────────┘                 │ │
│  └────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘
```

**Data Model**:
```typescript
model Organization {
  id: Int
  name: String
  slug: String  // acme

  // Sub-domain routing
  requestedSlug: String?

  // Members
  users: User[]
  teams: Team[]
  profiles: Profile[]
}

model Profile {
  id: Int
  organizationId: Int
  organization: Organization

  // Profile metadata
  uid: String  // sales, support
  username: String

  // Members
  user: User
}
```

### 3.2 Organization Domain Routing

```typescript
// packages/features/ee/organizations/lib/orgDomains.ts

export function getOrgSlug(hostname: string, forcedSlug?: string) {
  // Single org mode (for self-hosted)
  if (SINGLE_ORG_SLUG) {
    return SINGLE_ORG_SLUG;
  }

  // Extract subdomain
  // Example: sales.acme.cal.com → "sales.acme"
  if (!hostname.includes(".")) {
    return null; // No org domain
  }

  // Find current hostname (e.g., cal.com)
  const currentHostname = ALLOWED_HOSTNAMES.find((ahn) => {
    const url = new URL(WEBAPP_URL);
    const testHostname = `${url.hostname}${url.port ? `:${url.port}` : ""}`;
    return testHostname.endsWith(`.${ahn}`);
  });

  if (!currentHostname) {
    return null;
  }

  // Extract slug
  const slug = hostname.replace(currentHostname ? `.${currentHostname}` : "", "");
  const hasNoDotInSlug = slug.indexOf(".") === -1;

  if (hasNoDotInSlug) {
    return slug; // "acme"
  }

  return null;
}
```

**Location**: `packages/features/ee/organizations/lib/orgDomains.ts:16-58`

### 3.3 Organization Middleware

```typescript
// apps/web/middleware.ts

export async function middleware(req: NextRequest) {
  const url = req.nextUrl;
  const hostname = req.headers.get("host") || "";

  // Get organization slug from subdomain
  const orgSlug = getOrgSlug(hostname);

  if (orgSlug) {
    // Rewrite to organization context
    url.pathname = `/org/${orgSlug}${url.pathname}`;
    return NextResponse.rewrite(url);
  }

  return NextResponse.next();
}
```

**Example**:
```
Request: https://acme.cal.com/john/30min
         ↓
Rewrite: /org/acme/john/30min
         ↓
Render: Organization-specific booking page
```

### 3.4 Organization Features

```typescript
// Organization-specific features:

1. **Custom Branding**
   - Logo, colors, fonts
   - Custom domain (acme.cal.com)

2. **Team Management**
   - Create teams within organization
   - Assign users to teams
   - Team-level permissions

3. **Billing**
   - Organization-wide subscription
   - Usage-based billing per seat

4. **Admin Console**
   - User management
   - SSO configuration
   - DSYNC setup
   - Audit logs

5. **White-labeling**
   - Remove Cal.com branding
   - Custom email templates
   - Custom booking flow
```

---

## 4. Billing & Subscriptions

### 4.1 Overview

**Payment Provider**: Stripe

**Billing Models**:
- **Team Plans**: Per-team subscription
- **Organization Plans**: Per-organization with seat-based pricing
- **Credits**: Pay-per-booking model

**Location**: `packages/features/ee/billing/`

### 4.2 Stripe Integration

```typescript
// packages/features/ee/billing/stripe-billing-service.ts

export class StripeBillingService {
  async createCheckoutSession({
    teamId,
    organizationId,
    plan,
    successUrl,
    cancelUrl,
  }: {
    teamId?: number;
    organizationId?: number;
    plan: "team" | "organization";
    successUrl: string;
    cancelUrl: string;
  }) {
    const stripe = await getStripe();

    // Create or get Stripe customer
    const customer = await this.getOrCreateCustomer({
      teamId,
      organizationId,
    });

    // Create checkout session
    const session = await stripe.checkout.sessions.create({
      customer: customer.id,
      mode: "subscription",
      line_items: [
        {
          price: plan === "team" ? TEAM_PRICE_ID : ORG_PRICE_ID,
          quantity: 1,
        },
      ],
      success_url: successUrl,
      cancel_url: cancelUrl,
      metadata: {
        teamId: teamId?.toString(),
        organizationId: organizationId?.toString(),
      },
    });

    return session;
  }

  async handleWebhook(event: Stripe.Event) {
    switch (event.type) {
      case "checkout.session.completed":
        await this.handleCheckoutCompleted(event);
        break;

      case "customer.subscription.updated":
        await this.handleSubscriptionUpdated(event);
        break;

      case "customer.subscription.deleted":
        await this.handleSubscriptionDeleted(event);
        break;

      case "invoice.paid":
        await this.handleInvoicePaid(event);
        break;
    }
  }
}
```

### 4.3 Webhook Handlers

**Checkout Completed**:
```typescript
// packages/features/ee/billing/api/webhook/_checkout.session.completed.ts

export async function handleCheckoutCompleted(
  session: Stripe.Checkout.Session
) {
  const { teamId, organizationId } = session.metadata;

  if (teamId) {
    // Activate team subscription
    await prisma.team.update({
      where: { id: parseInt(teamId) },
      data: {
        subscription: {
          create: {
            stripeCustomerId: session.customer as string,
            stripeSubscriptionId: session.subscription as string,
            status: "active",
          },
        },
      },
    });
  }

  if (organizationId) {
    // Activate organization subscription
    await prisma.organization.update({
      where: { id: parseInt(organizationId) },
      data: {
        subscription: {
          create: {
            stripeCustomerId: session.customer as string,
            stripeSubscriptionId: session.subscription as string,
            status: "active",
          },
        },
      },
    });
  }
}
```

**Subscription Updated**:
```typescript
// packages/features/ee/billing/api/webhook/_customer.subscription.updated.ts

export async function handleSubscriptionUpdated(
  subscription: Stripe.Subscription
) {
  const customerId = subscription.customer as string;

  // Find team or org with this customer ID
  const team = await prisma.team.findFirst({
    where: {
      subscription: {
        stripeCustomerId: customerId,
      },
    },
  });

  if (team) {
    // Update subscription status
    await prisma.teamSubscription.update({
      where: { teamId: team.id },
      data: {
        status: subscription.status, // active, past_due, canceled, etc.
        currentPeriodEnd: new Date(subscription.current_period_end * 1000),
      },
    });
  }
}
```

**Invoice Paid**:
```typescript
// packages/features/ee/billing/api/webhook/_invoice.paid.ts

export async function handleInvoicePaid(invoice: Stripe.Invoice) {
  const organizationId = invoice.metadata?.organizationId;

  if (organizationId) {
    // Add credits to organization
    const amount = invoice.amount_paid / 100; // Convert cents to dollars
    const credits = Math.floor(amount / CREDIT_PRICE);

    await prisma.organizationCredit.create({
      data: {
        organizationId: parseInt(organizationId),
        credits,
        purchaseAmount: amount,
        stripeInvoiceId: invoice.id,
      },
    });
  }
}
```

### 4.4 Credit System

```typescript
// packages/features/ee/billing/credit-service.ts

export class CreditService {
  async deductCredits({
    organizationId,
    amount,
    reason,
  }: {
    organizationId: number;
    amount: number;
    reason: string;
  }) {
    // Check balance
    const org = await prisma.organization.findUnique({
      where: { id: organizationId },
      include: { credits: true },
    });

    const totalCredits = org.credits.reduce((sum, c) => sum + c.credits, 0);

    if (totalCredits < amount) {
      throw new Error("Insufficient credits");
    }

    // Deduct credits
    await prisma.creditTransaction.create({
      data: {
        organizationId,
        amount: -amount,
        reason,
      },
    });

    return { success: true, remainingCredits: totalCredits - amount };
  }

  async addCredits({
    organizationId,
    amount,
    reason,
  }: {
    organizationId: number;
    amount: number;
    reason: string;
  }) {
    await prisma.creditTransaction.create({
      data: {
        organizationId,
        amount,
        reason,
      },
    });
  }
}
```

**Usage**:
```typescript
// Deduct credits when booking is created
await creditService.deductCredits({
  organizationId: booking.organizationId,
  amount: 1,
  reason: `Booking ${booking.uid}`,
});
```

---

## 5. Workflows (Advanced Automation)

### 5.1 Overview

**Workflows** cho phép tự động hóa thông báo và actions dựa trên booking events.

**Trigger Types**:
- `BEFORE_EVENT`: X minutes/hours before booking
- `AFTER_EVENT`: X minutes/hours after booking
- `EVENT_CANCELLED`: When booking is cancelled
- `NEW_EVENT`: When new booking is created
- `RESCHEDULE_EVENT`: When booking is rescheduled

**Action Types**:
- Email reminder
- SMS reminder
- Webhook call
- Slack notification
- WhatsApp message

**Location**: `packages/features/ee/workflows/`

### 5.2 Workflow Schema

```typescript
// packages/prisma/schema.prisma

model Workflow {
  id: Int
  name: String
  userId: Int
  teamId: Int?

  // Trigger configuration
  trigger: WorkflowTriggerEvents  // BEFORE_EVENT, AFTER_EVENT, etc.
  time: Int?  // Minutes before/after
  timeUnit: TimeUnit?  // MINUTE, HOUR, DAY

  // Actions
  steps: WorkflowStep[]

  // Active/inactive
  activeOn: EventType[]
  active: Boolean
}

model WorkflowStep {
  id: Int
  workflowId: Int
  workflow: Workflow

  // Step configuration
  stepNumber: Int
  action: WorkflowActions  // EMAIL_HOST, EMAIL_ATTENDEE, SMS_ATTENDEE, etc.

  // Template
  template: WorkflowTemplates  // REMINDER, CUSTOM, etc.
  emailSubject: String?
  reminderBody: String?

  // Sender configuration
  sender: String?  // Email or phone number

  // Scheduling
  numberRequired: Boolean?
  sendTo: String?  // EMAIL, PHONE, WEBHOOK_URL
}

enum WorkflowTriggerEvents {
  BEFORE_EVENT
  AFTER_EVENT
  EVENT_CANCELLED
  NEW_EVENT
  RESCHEDULE_EVENT
}

enum WorkflowActions {
  EMAIL_HOST
  EMAIL_ATTENDEE
  SMS_ATTENDEE
  SMS_HOST
  WHATSAPP_ATTENDEE
  WHATSAPP_HOST
  EMAIL_ADDRESS
  SMS_NUMBER
  WEBHOOK
}
```

### 5.3 Workflow Execution

```typescript
// packages/features/ee/workflows/lib/reminders/workflows.ts

export async function scheduleWorkflowReminders({
  booking,
  eventType,
}: {
  booking: Booking;
  eventType: EventType;
}) {
  // Get active workflows for this event type
  const workflows = await prisma.workflow.findMany({
    where: {
      OR: [
        { activeOn: { some: { id: eventType.id } } },
        { teamId: eventType.teamId },
      ],
      active: true,
    },
    include: { steps: true },
  });

  for (const workflow of workflows) {
    // Calculate send time based on trigger
    const sendAt = calculateSendTime({
      bookingTime: booking.startTime,
      trigger: workflow.trigger,
      time: workflow.time,
      timeUnit: workflow.timeUnit,
    });

    // Schedule each step
    for (const step of workflow.steps) {
      await scheduleWorkflowStep({
        step,
        booking,
        sendAt,
      });
    }
  }
}

function calculateSendTime({
  bookingTime,
  trigger,
  time,
  timeUnit,
}: {
  bookingTime: Date;
  trigger: WorkflowTriggerEvents;
  time: number;
  timeUnit: TimeUnit;
}) {
  const minutes = convertToMinutes(time, timeUnit);

  switch (trigger) {
    case "BEFORE_EVENT":
      return subMinutes(bookingTime, minutes);
    case "AFTER_EVENT":
      return addMinutes(bookingTime, minutes);
    case "NEW_EVENT":
      return new Date(); // Send immediately
    default:
      return bookingTime;
  }
}
```

### 5.4 Workflow Actions

**Email Reminder**:
```typescript
// packages/features/ee/workflows/lib/reminders/emailReminderManager.ts

export async function sendEmailReminder({
  step,
  booking,
}: {
  step: WorkflowStep;
  booking: Booking;
}) {
  const { action, emailSubject, reminderBody, sendTo } = step;

  // Determine recipient
  let to: string;
  if (action === "EMAIL_ATTENDEE") {
    to = booking.attendees[0].email;
  } else if (action === "EMAIL_HOST") {
    to = booking.eventType.owner.email;
  } else if (action === "EMAIL_ADDRESS") {
    to = sendTo;
  }

  // Render template with variables
  const renderedBody = renderTemplate(reminderBody, {
    attendeeName: booking.attendees[0].name,
    eventName: booking.eventType.title,
    eventTime: booking.startTime,
    eventLocation: booking.location,
    hostName: booking.eventType.owner.name,
  });

  // Send email
  await sendEmail({
    to,
    subject: emailSubject,
    html: renderedBody,
  });
}
```

**SMS Reminder**:
```typescript
// packages/features/ee/workflows/lib/reminders/smsReminderManager.ts

export async function sendSMSReminder({
  step,
  booking,
}: {
  step: WorkflowStep;
  booking: Booking;
}) {
  const { action, reminderBody, sendTo } = step;

  // Determine recipient phone
  let to: string;
  if (action === "SMS_ATTENDEE") {
    to = booking.attendees[0].phoneNumber;
  } else if (action === "SMS_NUMBER") {
    to = sendTo;
  }

  // Render template
  const renderedBody = renderTemplate(reminderBody, {
    attendeeName: booking.attendees[0].name,
    eventTime: booking.startTime,
  });

  // Send SMS via Twilio
  await twilioClient.messages.create({
    to,
    from: TWILIO_PHONE_NUMBER,
    body: renderedBody,
  });
}
```

**Webhook Action**:
```typescript
// packages/features/ee/workflows/lib/reminders/webhookReminderManager.ts

export async function sendWebhookReminder({
  step,
  booking,
}: {
  step: WorkflowStep;
  booking: Booking;
}) {
  const { sendTo } = step; // Webhook URL

  // Prepare payload
  const payload = {
    triggerEvent: step.workflow.trigger,
    booking: {
      uid: booking.uid,
      title: booking.title,
      startTime: booking.startTime,
      endTime: booking.endTime,
      attendees: booking.attendees.map(a => ({
        name: a.name,
        email: a.email,
      })),
    },
  };

  // Send webhook
  await fetch(sendTo, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "X-Cal-Signature": generateHMAC(payload),
    },
    body: JSON.stringify(payload),
  });
}
```

### 5.5 Template Variables

```typescript
// packages/features/ee/workflows/lib/variableTranslations.ts

export const WORKFLOW_VARIABLES = {
  // Attendee
  "{ATTENDEE_NAME}": "booking.attendees[0].name",
  "{ATTENDEE_EMAIL}": "booking.attendees[0].email",
  "{ATTENDEE_PHONE}": "booking.attendees[0].phoneNumber",

  // Host
  "{HOST_NAME}": "booking.eventType.owner.name",
  "{HOST_EMAIL}": "booking.eventType.owner.email",

  // Event
  "{EVENT_NAME}": "booking.eventType.title",
  "{EVENT_DATE}": "booking.startTime (formatted)",
  "{EVENT_TIME}": "booking.startTime (time only)",
  "{EVENT_DURATION}": "booking.eventType.length",
  "{EVENT_LOCATION}": "booking.location",

  // Booking
  "{BOOKING_UID}": "booking.uid",
  "{RESCHEDULE_LINK}": "reschedule URL",
  "{CANCEL_LINK}": "cancel URL",

  // Organization
  "{ORGANIZER_NAME}": "booking.eventType.team.name",
  "{ORGANIZER_LOGO}": "booking.eventType.team.logo",
};

export function renderTemplate(template: string, data: Record<string, any>) {
  let rendered = template;

  for (const [variable, value] of Object.entries(data)) {
    rendered = rendered.replace(new RegExp(variable, "g"), value);
  }

  return rendered;
}
```

---

## 6. API Keys

### 6.1 Overview

**API Keys** cho phép programmatic access to Cal.com API without OAuth flow.

**Use Cases**:
- Server-to-server integration
- CI/CD automation
- Internal tools

**Location**: `packages/features/ee/api-keys/`

### 6.2 API Key Model

```typescript
// packages/prisma/schema.prisma

model ApiKey {
  id: String @id @default(cuid())
  userId: Int
  user: User

  // Key metadata
  note: String?  // Description
  hashedKey: String  // SHA-256 hash

  // Permissions
  scopes: String[]  // ["bookings:read", "bookings:write", etc.]

  // Expiry
  expiresAt: DateTime?
  lastUsedAt: DateTime?

  createdAt: DateTime
}
```

### 6.3 API Key Generation

```typescript
// packages/features/ee/api-keys/lib/apiKeys.ts

export async function createApiKey({
  userId,
  note,
  scopes,
  expiresAt,
}: {
  userId: number;
  note?: string;
  scopes: string[];
  expiresAt?: Date;
}) {
  // Generate random key
  const key = `cal_live_${randomBytes(32).toString("hex")}`;

  // Hash for storage (never store plain text)
  const hashedKey = createHash("sha256").update(key).digest("hex");

  // Create in database
  await prisma.apiKey.create({
    data: {
      userId,
      note,
      hashedKey,
      scopes,
      expiresAt,
    },
  });

  // Return key ONCE (cannot be retrieved later)
  return { key, note, scopes };
}
```

### 6.4 API Key Authentication

```typescript
// packages/features/ee/api-keys/lib/verifyApiKey.ts

export async function verifyApiKey(key: string) {
  if (!key.startsWith("cal_live_")) {
    throw new Error("Invalid API key format");
  }

  // Hash provided key
  const hashedKey = createHash("sha256").update(key).digest("hex");

  // Find in database
  const apiKey = await prisma.apiKey.findFirst({
    where: { hashedKey },
    include: { user: true },
  });

  if (!apiKey) {
    throw new Error("Invalid API key");
  }

  // Check expiry
  if (apiKey.expiresAt && apiKey.expiresAt < new Date()) {
    throw new Error("API key expired");
  }

  // Update last used
  await prisma.apiKey.update({
    where: { id: apiKey.id },
    data: { lastUsedAt: new Date() },
  });

  return {
    user: apiKey.user,
    scopes: apiKey.scopes,
  };
}
```

### 6.5 Usage in tRPC

```typescript
// packages/trpc/server/createContext.ts

export async function createContext({ req, res }) {
  // Check for API key in Authorization header
  const authHeader = req.headers.authorization;

  if (authHeader?.startsWith("Bearer cal_live_")) {
    const apiKey = authHeader.replace("Bearer ", "");

    try {
      const { user, scopes } = await verifyApiKey(apiKey);

      return {
        prisma,
        session: null,
        user,
        apiKeyScopes: scopes,
      };
    } catch (error) {
      throw new TRPCError({
        code: "UNAUTHORIZED",
        message: error.message,
      });
    }
  }

  // Fall back to session auth
  const session = await getServerSession(req, res);
  return { prisma, session, user: null, apiKeyScopes: null };
}
```

### 6.6 Scope-Based Authorization

```typescript
// packages/features/ee/api-keys/lib/checkScope.ts

export function checkScope(
  requiredScope: string,
  userScopes: string[]
) {
  // Check exact match
  if (userScopes.includes(requiredScope)) {
    return true;
  }

  // Check wildcard (e.g., "bookings:*" covers "bookings:read" and "bookings:write")
  const [resource, action] = requiredScope.split(":");
  const wildcardScope = `${resource}:*`;

  if (userScopes.includes(wildcardScope)) {
    return true;
  }

  // Check admin scope
  if (userScopes.includes("*:*")) {
    return true;
  }

  return false;
}
```

**Example Scopes**:
```typescript
const SCOPES = [
  "bookings:read",
  "bookings:write",
  "event-types:read",
  "event-types:write",
  "availability:read",
  "availability:write",
  "teams:read",
  "teams:write",
  "webhooks:read",
  "webhooks:write",
  "*:*", // Admin (all permissions)
];
```

---

## 7. Other Enterprise Features

### 7.1 Managed Event Types

```typescript
// Create event types on behalf of users
POST /api/trpc/eventTypes.create

{
  userId: 123,
  title: "30 Minute Meeting",
  slug: "30min",
  length: 30,
  locations: [{ type: "integrations:zoom" }],
  managedBy: "organization", // Cannot be edited by user
}
```

### 7.2 Impersonation

```typescript
// Admin can impersonate users for debugging
POST /api/auth/impersonate

{
  userId: 123,
  reason: "Customer support ticket #456",
}

// Creates temporary session as that user
// Logged in audit trail
```

### 7.3 Audit Logs

```typescript
model AuditLog {
  id: Int
  organizationId: Int
  userId: Int

  action: String  // "user.created", "booking.deleted", etc.
  resource: String  // "booking", "user", "team"
  resourceId: String

  metadata: Json
  ipAddress: String
  userAgent: String

  createdAt: DateTime
}

// Track all sensitive actions
await auditLog.create({
  organizationId: user.organizationId,
  userId: user.id,
  action: "sso.configured",
  resource: "organization",
  resourceId: organization.id,
  metadata: { provider: "okta" },
  ipAddress: req.ip,
  userAgent: req.headers["user-agent"],
});
```

### 7.4 Advanced Team Features

```typescript
// Round-robin with weights
model TeamMember {
  userId: Int
  teamId: Int

  // Scheduling weight (for round-robin)
  weight: Int  // Default: 100

  // Availability override
  availability: Availability[]
}

// Weighted distribution example:
// John (weight: 200) gets 2x more bookings than Jane (weight: 100)
```

### 7.5 License Key Management

```typescript
// packages/features/ee/deployment/licensekey/

export async function verifyLicenseKey(key: string) {
  // Call Cal.com license server
  const response = await fetch("https://api.cal.com/v2/license/verify", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ key }),
  });

  const data = await response.json();

  return {
    valid: data.valid,
    plan: data.plan, // "enterprise", "team", etc.
    features: data.features, // ["sso", "dsync", "workflows"]
    expiresAt: data.expiresAt,
  };
}
```

---

## Summary

### Enterprise Features Recap

| Feature | Technology | Use Case |
|---------|-----------|----------|
| **SSO** | BoxyHQ SAML Jackson | Single sign-on with SAML/OIDC |
| **DSYNC** | BoxyHQ Directory Sync | Auto-provision users from directory |
| **Organizations** | Multi-tenant architecture | Separate workspaces per company |
| **Billing** | Stripe | Subscriptions & credits |
| **Workflows** | Scheduled jobs | Automated reminders & webhooks |
| **API Keys** | SHA-256 hashing | Server-to-server access |
| **Audit Logs** | Database logs | Compliance & security |
| **Managed Events** | Organization control | Centralized event management |
| **Impersonation** | Session switching | Customer support |
| **License Keys** | License server | Feature gating |

### Implementation Checklist

For self-hosted enterprise deployment:

1. **Environment Setup**:
```bash
# SSO
SAML_DATABASE_URL=postgresql://...
SAML_ADMINS=admin@company.com

# DSYNC
DSYNC_DATABASE_URL=postgresql://...

# Stripe
STRIPE_SECRET_KEY=sk_live_...
STRIPE_WEBHOOK_SECRET=whsec_...

# License
LICENSE_KEY=ent_...
```

2. **Database Migration**:
```bash
npx prisma migrate deploy
```

3. **Configure SSO**:
- Add SAML provider in admin console
- Configure IdP (Okta, Azure AD)
- Test login flow

4. **Configure DSYNC**:
- Add directory in admin console
- Configure SCIM endpoint in directory provider
- Map groups to teams

5. **Configure Billing**:
- Add Stripe webhook endpoint
- Test subscription flow
- Monitor invoices

6. **Test Workflows**:
- Create test workflow
- Trigger and verify delivery
- Check scheduled jobs

---

**Tác giả**: Claude (AI Assistant)
**Ngày tạo**: 2025-11-18
**Phiên bản**: 1.0
