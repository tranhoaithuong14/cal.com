# PHASE 6: App Store & Integration Architecture

> **Mục tiêu**: Hiểu app marketplace, integration flow, plugin architecture, và credential management cho 107+ integrations.

## 📋 Table of Contents

- [1. Overview](#1-overview)
- [2. App Categories](#2-app-categories)
- [3. App Structure](#3-app-structure)
- [4. Integration Types](#4-integration-types)
- [5. OAuth & Credential Flow](#5-oauth--credential-flow)
- [6. Key Integrations Deep Dive](#6-key-integrations-deep-dive)
- [7. App Installation & Management](#7-app-installation--management)
- [8. Building New Apps](#8-building-new-apps)
- [9. Testing & Quality](#9-testing--quality)
- [10. Best Practices](#10-best-practices)

---

## 1. Overview

### 1.1 What is the App Store?

The App Store is Cal.com's **plugin marketplace** that enables seamless integration with 100+ third-party services. Apps extend Cal.com's functionality by connecting to:

- Calendar providers (Google, Outlook, Apple, CalDAV)
- Video conferencing (Zoom, Google Meet, MS Teams, Daily.co)
- Payment processors (Stripe, PayPal, Bitcoin)
- CRM systems (Salesforce, HubSpot, Pipedrive)
- Analytics platforms (Google Analytics, Plausible, PostHog)
- Automation tools (Zapier, Make, n8n)
- Messaging apps (Slack, WhatsApp, Telegram)

### 1.2 Statistics

```
Total Apps: 107
├── Calendar: 11 apps
├── Video Conferencing: 22 apps
├── Payment: 5 apps
├── CRM: 7 apps
├── Analytics: 8 apps
├── Automation: 4 apps
├── Messaging: 3 apps
├── AI/Phone: 12 apps
└── Other/Utility: 35 apps
```

### 1.3 Package Structure

**Location**: `packages/app-store/`

```
app-store/
├── _components/         # Shared UI components
├── _pages/              # Shared pages
├── _utils/              # Shared utilities
│   ├── oauth/          # OAuth helpers
│   ├── getCalendar.ts  # Calendar factory
│   └── getConnectedApps.ts
├── [app-name]/          # 107 individual apps
│   ├── _metadata.ts    # App configuration
│   ├── lib/            # Business logic
│   ├── api/            # API routes
│   ├── components/     # UI components
│   ├── static/         # Icons, images
│   ├── DESCRIPTION.md  # App description
│   └── package.json    # Dependencies
└── templates/           # App scaffolding templates
```

---

## 2. App Categories

### 2.1 Complete App List by Category

#### Calendar Apps (11)

```
1. googlecalendar       - Google Calendar
2. office365calendar    - Microsoft 365 Calendar
3. applecalendar        - Apple iCloud Calendar
4. caldavcalendar       - CalDAV (generic)
5. zohocalendar         - Zoho Calendar
6. larkcalendar         - Lark Calendar
7. feishucalendar       - Feishu Calendar
8. amie                 - Amie Calendar
9. vimcal               - Vimcal
10. cron                - Cron Calendar
11. ics-feedcalendar    - ICS Feed Calendar
```

#### Video Conferencing (22)

```
1. zoomvideo            - Zoom
2. googlevideo          - Google Meet
3. office365video       - Microsoft Teams
4. dailyvideo           - Daily.co
5. whereby              - Whereby
6. jitsivideo           - Jitsi Meet
7. webex                - Cisco Webex
8. huddle01video        - Huddle01
9. tandemvideo          - Tandem
10. around              - Around
11. skype               - Skype
12. facetime            - FaceTime
13. sirius_video        - Sirius
14. demodesk            - Demodesk
15. riverside           - Riverside
16. salesroom           - Salesroom
17. shimmervideo        - Shimmer
18. sylapsvideo         - Sylaps
19. element-call        - Element Call
20. mirotalk            - MiroTalk
21. nextcloudtalk       - Nextcloud Talk
22. horizon-workrooms   - Meta Horizon Workrooms
```

#### Payment Processors (5)

```
1. stripepayment        - Stripe
2. paypal               - PayPal
3. alby                 - Alby (Bitcoin Lightning)
4. btcpayserver         - BTCPay Server
5. hitpay               - HitPay
```

#### CRM Systems (7)

```
1. salesforce           - Salesforce
2. hubspot              - HubSpot
3. pipedrive-crm        - Pipedrive
4. zohocrm              - Zoho CRM
5. zoho-bigin           - Zoho Bigin
6. closecom             - Close.com
7. attio                - Attio
```

#### Analytics (8)

```
1. ga4                  - Google Analytics 4
2. gtm                  - Google Tag Manager
3. plausible            - Plausible
4. fathom               - Fathom Analytics
5. matomo               - Matomo
6. umami                - Umami
7. twipla               - Twipla
8. metapixel            - Meta Pixel
```

#### Automation (4)

```
1. zapier               - Zapier
2. make                 - Make (Integromat)
3. n8n                  - n8n
4. pipedream            - Pipedream
```

#### Messaging (3)

```
1. whatsapp             - WhatsApp
2. telegram             - Telegram
3. signal               - Signal
```

#### AI & Phone (12)

```
1. retell-ai            - Retell AI
2. synthflow            - Synthflow
3. elevenlabs           - ElevenLabs
4. bolna                - Bolna
5. greetmate-ai         - Greetmate AI
6. fonio-ai             - Fonio AI
7. millis-ai            - Millis AI
8. monobot              - Monobot
9. telli                - Telli
10. dialpad             - Dialpad
11. eightxeight         - 88
12. chatbase            - Chatbase
```

#### Project Management & Productivity (9)

```
1. basecamp3            - Basecamp
2. linear               - Linear
3. roam                 - Roam Research
4. wordpress            - WordPress
5. raycast              - Raycast
6. framer               - Framer
7. campfire             - Campfire
8. discord              - Discord
9. intercom             - Intercom
```

#### Other Utilities (26)

```
1. routing-forms        - Routing Forms
2. qr_code              - QR Code
3. giphy                - Giphy
4. weather_in_your_calendar - Weather
5. autocheckin          - Auto Check-in
6. baa-for-hipaa        - BAA for HIPAA
7. sendgrid             - SendGrid
8. dub                  - Dub.co (Link Management)
9. clic                 - Clic
10. deel                - Deel
11. vital               - Vital
12. granola             - Granola
13. jelly               - Jelly
14. lindy               - Lindy
15. wipemycalother      - Wipe My Cal Other
16. insihts             - Insights
17. ping                - Ping
18. posthog             - PostHog
19. mock-payment-app    - Mock Payment (Testing)
20. tests               - Test utilities
... and more
```

---

## 3. App Structure

### 3.1 Standard App Structure

Every app follows this pattern:

```typescript
[app-name]/
├── _metadata.ts           # App metadata & configuration
├── DESCRIPTION.md         # User-facing description
├── index.ts               # Public exports
├── package.json           # Dependencies
├── zod.ts                 # Zod validation schemas
│
├── lib/                   # Business logic
│   ├── CalendarService.ts # (for calendar apps)
│   ├── VideoService.ts    # (for video apps)
│   ├── PaymentService.ts  # (for payment apps)
│   └── [helpers].ts
│
├── api/                   # API routes
│   ├── callback.ts        # OAuth callback
│   ├── add.ts             # Installation endpoint
│   ├── webhook.ts         # Webhook handler
│   └── [custom].ts
│
├── components/            # React components
│   ├── InstallAppButton.tsx
│   └── EventTypeAppCard.tsx
│
├── pages/                 # Full pages (if needed)
│   └── setup.tsx          # Post-install setup
│
└── static/                # Assets
    ├── icon.svg           # App icon
    └── icon-dark.svg      # Dark mode icon
```

### 3.2 App Metadata (`_metadata.ts`)

**File**: `packages/app-store/[app]/_metadata.ts`

Every app exports a `metadata` object conforming to the `AppMeta` type:

```typescript
// Example: Google Calendar
import type { AppMeta } from "@calcom/types/App";
import _package from "./package.json";

export const metadata = {
  // Basic Info
  name: "Google Calendar",
  description: _package.description,
  title: "Google Calendar",
  slug: "google-calendar",
  dirName: "googlecalendar",

  // Type & Category
  type: "google_calendar",             // Unique type identifier
  variant: "calendar",                 // calendar, conferencing, payment, other
  category: "calendar",                // Primary category
  categories: ["calendar"],            // All categories

  // OAuth
  isOAuth: true,                       // Requires OAuth?

  // Installation Check
  installed: !!(
    process.env.GOOGLE_API_CREDENTIALS &&
    validJson(process.env.GOOGLE_API_CREDENTIALS)
  ),

  // Branding
  logo: "icon.svg",
  publisher: "Cal.com",
  url: "https://cal.com/",
  email: "help@cal.com",

  // Optional
  docsUrl: "https://docs.cal.com/...",  // Documentation URL
  extendsFeature: "EventType",          // What feature it extends

  // App-specific data
  appData: {
    location: {
      type: "integrations:google_calendar",
      label: "Google Calendar",
      linkType: "static",
    },
  },

  // Advanced
  delegationCredential: {
    workspacePlatformSlug: "google",
  },
} as AppMeta;

export default metadata;
```

**Key Fields**:

| Field | Description | Required |
|-------|-------------|----------|
| `slug` | URL-safe identifier | ✅ |
| `type` | Unique type (e.g., `zoom_video`) | ✅ |
| `variant` | App variant: `calendar`, `conferencing`, `payment`, `other` | ✅ |
| `category` | Primary category | ✅ |
| `categories` | All applicable categories | ✅ |
| `isOAuth` | Whether app uses OAuth | ❌ |
| `installed` | Runtime check if app is configured | ❌ |
| `appData` | App-specific configuration | ❌ |

**Location**: `packages/app-store/googlecalendar/_metadata.ts:1`

---

## 4. Integration Types

### 4.1 Calendar Integrations

**Interface**: `Calendar` (from `@calcom/types/Calendar`)

**Required Methods**:
```typescript
interface Calendar {
  // List user's calendars
  listCalendars(): Promise<IntegrationCalendar[]>;

  // Get busy times
  getAvailability(
    dateFrom: string,
    dateTo: string,
    selectedCalendars: IntegrationCalendar[]
  ): Promise<EventBusyDate[]>;

  // Create calendar event
  createEvent(event: CalendarEvent): Promise<NewCalendarEventType>;

  // Update event
  updateEvent(uid: string, event: CalendarEvent): Promise<any>;

  // Delete event
  deleteEvent(uid: string): Promise<void>;

  // Get credential ID
  getCredentialId(): number;
}
```

**Example**: Google Calendar Service

```typescript
// packages/app-store/googlecalendar/lib/CalendarService.ts
import type { calendar_v3 } from "@googleapis/calendar";
import type { Calendar } from "@calcom/types/Calendar";

export default class GoogleCalendarService implements Calendar {
  private auth: CalendarAuth;
  private credential: CredentialForCalendarServiceWithEmail;

  constructor(credential: CredentialForCalendarServiceWithEmail) {
    this.credential = credential;
    this.auth = new CalendarAuth(credential);
  }

  async authedCalendar(): Promise<calendar_v3.Calendar> {
    return this.auth.getClient();
  }

  async listCalendars(): Promise<IntegrationCalendar[]> {
    const calendar = await this.authedCalendar();
    const { data } = await calendar.calendarList.list();

    return (data.items || []).map(cal => ({
      externalId: cal.id!,
      integration: "google_calendar",
      name: cal.summary || "Unnamed Calendar",
      primary: cal.primary || false,
      readOnly: cal.accessRole === "reader",
    }));
  }

  async getAvailability(
    dateFrom: string,
    dateTo: string,
    selectedCalendars: IntegrationCalendar[]
  ): Promise<EventBusyDate[]> {
    const calendar = await this.authedCalendar();

    const { data } = await calendar.freebusy.query({
      requestBody: {
        timeMin: dateFrom,
        timeMax: dateTo,
        items: selectedCalendars.map(cal => ({ id: cal.externalId })),
      },
    });

    const busyTimes: EventBusyDate[] = [];
    Object.values(data.calendars || {}).forEach(cal => {
      cal.busy?.forEach(busy => {
        busyTimes.push({
          start: busy.start!,
          end: busy.end!,
        });
      });
    });

    return busyTimes;
  }

  async createEvent(event: CalendarEvent): Promise<NewCalendarEventType> {
    const calendar = await this.authedCalendar();

    const attendees = this.getAttendees({ event });

    const { data } = await calendar.events.insert({
      calendarId: event.destinationCalendar?.externalId || "primary",
      conferenceDataVersion: event.location?.includes("google-meet") ? 1 : 0,
      requestBody: {
        summary: event.title,
        description: getRichDescription(event),
        start: {
          dateTime: event.startTime,
          timeZone: event.organizer.timeZone,
        },
        end: {
          dateTime: event.endTime,
          timeZone: event.organizer.timeZone,
        },
        attendees,
        conferenceData: event.location?.includes("google-meet") ? {
          createRequest: { requestId: uuid() },
        } : undefined,
        reminders: {
          useDefault: true,
        },
      },
    });

    return {
      uid: data.id!,
      id: data.id!,
      type: "google_calendar",
      password: "",
      url: data.hangoutLink || data.htmlLink || "",
    };
  }

  async updateEvent(uid: string, event: CalendarEvent): Promise<any> {
    const calendar = await this.authedCalendar();

    await calendar.events.patch({
      calendarId: event.destinationCalendar?.externalId || "primary",
      eventId: uid,
      requestBody: {
        summary: event.title,
        description: getRichDescription(event),
        start: {
          dateTime: event.startTime,
          timeZone: event.organizer.timeZone,
        },
        end: {
          dateTime: event.endTime,
          timeZone: event.organizer.timeZone,
        },
      },
    });
  }

  async deleteEvent(uid: string): Promise<void> {
    const calendar = await this.authedCalendar();

    await calendar.events.delete({
      calendarId: "primary",
      eventId: uid,
    });
  }

  getCredentialId(): number {
    return this.credential.id;
  }
}
```

**Location**: `packages/app-store/googlecalendar/lib/CalendarService.ts:56`

### 4.2 Video Conferencing Integrations

**Interface**: Similar to Calendar, but focused on video meeting creation

**Required Methods**:
```typescript
interface VideoService {
  // Create video meeting
  createMeeting(event: CalendarEvent): Promise<VideoCallData>;

  // Update meeting
  updateMeeting(uid: string, event: CalendarEvent): Promise<VideoCallData>;

  // Delete meeting
  deleteMeeting(uid: string): Promise<void>;
}
```

**Location Types**:
```typescript
// App location configuration
appData: {
  location: {
    type: "integrations:zoom",      // Location type
    label: "Zoom Video",             // Display label
    linkType: "dynamic",             // static | dynamic
    organizerInputPlaceholder: "",   // Placeholder text
    attendeeInputPlaceholder: "",    // For attendee input
    default: false,                  // Default location?
  },
}
```

**Link Types**:
- **`static`**: URL doesn't change (e.g., personal Zoom room)
- **`dynamic`**: New URL per booking (e.g., Zoom instant meeting)

### 4.3 Payment Integrations

**Interface**: Payment processor integration

**Required Methods**:
```typescript
interface PaymentService {
  // Create payment session
  create(payment: PaymentData): Promise<PaymentResponse>;

  // Update payment
  update(uid: string, data: Partial<PaymentData>): Promise<PaymentResponse>;

  // Refund payment
  refund(uid: string): Promise<PaymentResponse>;

  // Get payment details
  collectCard(payment: PaymentData): Promise<PaymentResponse>;
}
```

**Stripe Example**:

```typescript
// packages/app-store/stripepayment/lib/PaymentService.ts
import Stripe from "stripe";

export class StripePaymentService implements PaymentService {
  private stripe: Stripe;

  constructor(credential: Credential) {
    this.stripe = new Stripe(credential.key.stripe_api_key, {
      apiVersion: "2022-11-15",
    });
  }

  async create(payment: PaymentData): Promise<PaymentResponse> {
    const session = await this.stripe.checkout.sessions.create({
      mode: "payment",
      payment_method_types: ["card"],
      success_url: payment.successUrl,
      cancel_url: payment.cancelUrl,
      customer_email: payment.email,
      line_items: [
        {
          price_data: {
            currency: payment.currency,
            product_data: {
              name: payment.title,
              description: payment.description,
            },
            unit_amount: payment.amount,
          },
          quantity: 1,
        },
      ],
      metadata: {
        bookingId: payment.bookingId,
      },
    });

    return {
      id: session.id,
      url: session.url,
    };
  }

  async refund(paymentIntentId: string): Promise<PaymentResponse> {
    const refund = await this.stripe.refunds.create({
      payment_intent: paymentIntentId,
    });

    return {
      id: refund.id,
      status: refund.status,
    };
  }
}
```

**Location**: `packages/app-store/stripepayment/lib/PaymentService.ts:1`

### 4.4 CRM Integrations

**Interface**: CRM data sync

**Required Methods**:
```typescript
interface CRMService {
  // Create contact
  createContact(data: ContactData): Promise<Contact>;

  // Create lead/opportunity
  createLead(data: LeadData): Promise<Lead>;

  // Update record
  updateRecord(id: string, data: any): Promise<any>;

  // Search records
  searchRecords(query: string): Promise<any[]>;
}
```

**Webhook Integration**: Most CRM apps use webhooks to sync booking data

```typescript
// packages/app-store/hubspot/api/webhook.ts
export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  if (req.method !== "POST") {
    return res.status(405).end();
  }

  const booking = req.body;

  // Find HubSpot credential
  const credential = await prisma.credential.findFirst({
    where: {
      userId: booking.userId,
      type: "hubspot_other_calendar",
    },
  });

  if (!credential) {
    return res.status(200).json({ message: "No HubSpot integration" });
  }

  // Create contact in HubSpot
  await createHubSpotContact({
    credential,
    contact: {
      email: booking.attendees[0].email,
      firstName: booking.attendees[0].name.split(" ")[0],
      lastName: booking.attendees[0].name.split(" ").slice(1).join(" "),
    },
  });

  return res.status(200).json({ success: true });
}
```

---

## 5. OAuth & Credential Flow

### 5.1 OAuth Flow

Most integrations use **OAuth 2.0** for authentication:

```
1. User clicks "Connect [App]"
   ↓
2. Redirect to app's OAuth consent page
   ↓
3. User grants permissions
   ↓
4. Redirect back to /api/integrations/[app]/callback
   ↓
5. Exchange authorization code for access token
   ↓
6. Store encrypted credential in database
   ↓
7. Mark app as connected
```

**OAuth Callback Handler**:

```typescript
// packages/app-store/googlecalendar/api/callback.ts
import type { NextApiRequest, NextApiResponse } from "next";
import { oauth2Client } from "../lib/CalendarAuth";
import prisma from "@calcom/prisma";

export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  const { code, state } = req.query;

  if (!code || typeof code !== "string") {
    return res.status(400).json({ error: "Missing authorization code" });
  }

  // Exchange code for tokens
  const { tokens } = await oauth2Client.getToken(code);

  // Get user info
  oauth2Client.setCredentials(tokens);
  const calendar = google.calendar({ version: "v3", auth: oauth2Client });
  const { data } = await calendar.calendarList.list();
  const primaryCalendar = data.items?.find(cal => cal.primary);

  // Decrypt state to get userId
  const { userId } = JSON.parse(decryptState(state as string));

  // Store credential
  await prisma.credential.create({
    data: {
      userId,
      type: "google_calendar",
      key: {
        access_token: tokens.access_token,
        refresh_token: tokens.refresh_token,
        expiry_date: tokens.expiry_date,
        scope: tokens.scope,
        token_type: tokens.token_type,
      },
      appId: "google-calendar",
      invalid: false,
    },
  });

  // Redirect back to app settings
  return res.redirect("/apps/installed/calendar");
}
```

**Location**: `packages/app-store/googlecalendar/api/callback.ts:1`

### 5.2 Credential Storage

Credentials are stored in the `Credential` table with **AES-256 encryption**:

```prisma
model Credential {
  id        Int      @id @default(autoincrement())
  userId    Int?
  teamId    Int?
  type      String   // e.g., "google_calendar", "zoom_video"
  key       Json     // Encrypted OAuth tokens
  appId     String?
  invalid   Boolean  @default(false)

  user      User?    @relation(fields: [userId], references: [id])
  team      Team?    @relation(fields: [teamId], references: [id])

  @@unique([userId, type])
  @@index([userId])
}
```

**Encryption**:
```typescript
import { symmetricEncrypt, symmetricDecrypt } from "@calcom/lib/crypto";

// Encrypt before storing
const encryptedKey = symmetricEncrypt(
  JSON.stringify(tokens),
  process.env.CALENDSO_ENCRYPTION_KEY
);

await prisma.credential.create({
  data: {
    userId,
    type: "google_calendar",
    key: encryptedKey,
  },
});

// Decrypt when using
const credential = await prisma.credential.findFirst({ where: { userId, type: "google_calendar" } });
const tokens = JSON.parse(
  symmetricDecrypt(credential.key, process.env.CALENDSO_ENCRYPTION_KEY)
);
```

### 5.3 Credential Refresh

OAuth tokens expire and must be refreshed:

```typescript
// packages/app-store/googlecalendar/lib/CalendarAuth.ts
export class CalendarAuth {
  private credential: CredentialForCalendarServiceWithEmail;
  private oauth2Client: OAuth2Client;

  constructor(credential: CredentialForCalendarServiceWithEmail) {
    this.credential = credential;
    this.oauth2Client = new OAuth2Client(
      GOOGLE_CLIENT_ID,
      GOOGLE_CLIENT_SECRET
    );

    // Set credentials
    this.oauth2Client.setCredentials(credential.key);

    // Auto-refresh tokens
    this.oauth2Client.on("tokens", (tokens) => {
      this.updateTokens(tokens);
    });
  }

  private async updateTokens(tokens: Credentials) {
    // Merge new tokens with existing
    const updatedKey = {
      ...this.credential.key,
      ...tokens,
    };

    // Update in database
    await prisma.credential.update({
      where: { id: this.credential.id },
      data: { key: updatedKey },
    });

    this.credential.key = updatedKey;
  }

  async getClient(): Promise<calendar_v3.Calendar> {
    // OAuth client will auto-refresh if needed
    return google.calendar({ version: "v3", auth: this.oauth2Client });
  }
}
```

**Location**: `packages/app-store/googlecalendar/lib/CalendarAuth.ts:1`

---

## 6. Key Integrations Deep Dive

### 6.1 Google Calendar

**Type**: `calendar`
**OAuth**: Yes
**Webhook Support**: Yes (push notifications via Google Calendar API)

**Features**:
- ✅ List calendars
- ✅ Check availability (freebusy)
- ✅ Create/update/delete events
- ✅ Google Meet integration
- ✅ Real-time sync via webhooks
- ✅ Multiple calendar support
- ✅ Attendee management

**Configuration**:
```bash
# .env
GOOGLE_API_CREDENTIALS='{
  "web": {
    "client_id": "xxx",
    "client_secret": "yyy",
    "redirect_uris": ["https://app.cal.com/api/integrations/googlecalendar/callback"]
  }
}'
```

**Scopes**:
```typescript
const GOOGLE_CALENDAR_SCOPES = [
  "https://www.googleapis.com/auth/calendar.readonly",
  "https://www.googleapis.com/auth/calendar.events",
];
```

**Webhook Setup**:
```typescript
// Watch calendar for changes
async watchCalendar(calendarId: string) {
  const calendar = await this.authedCalendar();

  const { data } = await calendar.events.watch({
    calendarId,
    requestBody: {
      id: uuid(),
      type: "web_hook",
      address: GOOGLE_WEBHOOK_URL,
      expiration: Date.now() + 30 * 24 * 60 * 60 * 1000, // 30 days
    },
  });

  // Store channel info
  await prisma.webhook.create({
    data: {
      id: data.id!,
      subscriberUrl: data.resourceUri!,
      appId: "google-calendar",
      payloadTemplate: null,
      active: true,
    },
  });
}
```

**Location**: `packages/app-store/googlecalendar/lib/CalendarService.ts:1`

---

### 6.2 Zoom Video

**Type**: `conferencing`
**OAuth**: Yes
**Meeting Types**: Instant, Scheduled

**Features**:
- ✅ Create instant meetings
- ✅ Schedule meetings
- ✅ Custom meeting IDs
- ✅ Waiting rooms
- ✅ Passcodes
- ✅ Recording settings

**Configuration**:
```bash
# .env
ZOOM_CLIENT_ID=xxx
ZOOM_CLIENT_SECRET=yyy
```

**Meeting Creation**:
```typescript
// packages/app-store/zoomvideo/lib/VideoApiAdapter.ts
export const createMeeting = async (
  credential: Credential,
  event: CalendarEvent
): Promise<VideoCallData> => {
  const tokens = credential.key;

  const response = await fetch("https://api.zoom.us/v2/users/me/meetings", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      Authorization: `Bearer ${tokens.access_token}`,
    },
    body: JSON.stringify({
      topic: event.title,
      type: 2, // Scheduled meeting
      start_time: event.startTime,
      duration: Math.round((new Date(event.endTime).getTime() - new Date(event.startTime).getTime()) / 60000),
      timezone: event.organizer.timeZone,
      settings: {
        host_video: true,
        participant_video: true,
        join_before_host: true,
        waiting_room: false,
        mute_upon_entry: false,
      },
    }),
  });

  const data = await response.json();

  return {
    type: "zoom_video",
    id: data.id,
    password: data.password || "",
    url: data.join_url,
  };
};
```

**Location**: `packages/app-store/zoomvideo/lib/VideoApiAdapter.ts:1`

---

### 6.3 Stripe Payment

**Type**: `payment`
**OAuth**: Yes (Stripe Connect)
**Webhook Required**: Yes

**Features**:
- ✅ One-time payments
- ✅ Stripe Checkout
- ✅ Automatic refunds on cancellation
- ✅ Payment status tracking
- ✅ Multi-currency support

**Configuration**:
```bash
# .env
STRIPE_CLIENT_ID=ca_xxx
STRIPE_PRIVATE_KEY=sk_live_xxx
NEXT_PUBLIC_STRIPE_PUBLIC_KEY=pk_live_xxx
STRIPE_WEBHOOK_SECRET=whsec_xxx
```

**Payment Flow**:
```typescript
// 1. Create checkout session
export const createPayment = async (
  credential: Credential,
  booking: Booking,
  eventType: EventType
): Promise<PaymentResponse> => {
  const stripe = new Stripe(credential.key.stripe_private_key, {
    apiVersion: "2022-11-15",
  });

  const session = await stripe.checkout.sessions.create({
    mode: "payment",
    payment_method_types: ["card"],
    success_url: `${WEBAPP_URL}/booking/${booking.uid}?payment=success`,
    cancel_url: `${WEBAPP_URL}/booking/${booking.uid}?payment=cancelled`,
    customer_email: booking.attendees[0].email,
    line_items: [
      {
        price_data: {
          currency: eventType.currency.toLowerCase(),
          product_data: {
            name: eventType.title,
            description: eventType.description || "",
          },
          unit_amount: eventType.price,
        },
        quantity: 1,
      },
    ],
    payment_intent_data: {
      metadata: {
        bookingId: booking.id,
      },
    },
    metadata: {
      bookingId: booking.id,
    },
  });

  // Store payment
  await prisma.payment.create({
    data: {
      bookingId: booking.id,
      amount: eventType.price,
      currency: eventType.currency,
      externalId: session.id,
      data: session as any,
      fee: 0,
      refunded: false,
      success: false,
      appId: "stripe",
    },
  });

  return {
    url: session.url!,
  };
};

// 2. Webhook handler
export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  const sig = req.headers["stripe-signature"];
  const body = await buffer(req);

  let event: Stripe.Event;

  try {
    event = stripe.webhooks.constructEvent(
      body,
      sig!,
      process.env.STRIPE_WEBHOOK_SECRET!
    );
  } catch (err) {
    return res.status(400).send(`Webhook Error: ${err.message}`);
  }

  if (event.type === "checkout.session.completed") {
    const session = event.data.object as Stripe.Checkout.Session;
    const bookingId = parseInt(session.metadata?.bookingId || "0");

    // Update payment
    await prisma.payment.updateMany({
      where: { bookingId, success: false },
      data: { success: true },
    });

    // Confirm booking
    await prisma.booking.update({
      where: { id: bookingId },
      data: { status: BookingStatus.ACCEPTED },
    });

    // Send confirmation emails
    await sendScheduledEmails(bookingId);
  }

  res.status(200).json({ received: true });
}
```

**Location**: `packages/app-store/stripepayment/lib/PaymentService.ts:1`

---

### 6.4 HubSpot CRM

**Type**: `other` (CRM)
**OAuth**: Yes
**Sync**: Real-time via webhooks

**Features**:
- ✅ Create contacts
- ✅ Create deals
- ✅ Sync booking data
- ✅ Custom properties

**Configuration**:
```bash
# .env
HUBSPOT_CLIENT_ID=xxx
HUBSPOT_CLIENT_SECRET=yyy
```

**Contact Creation**:
```typescript
// packages/app-store/hubspot/lib/contacts.ts
export const createContact = async (
  credential: Credential,
  attendee: Person
) => {
  const tokens = credential.key;

  const response = await fetch("https://api.hubapi.com/crm/v3/objects/contacts", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      Authorization: `Bearer ${tokens.access_token}`,
    },
    body: JSON.stringify({
      properties: {
        email: attendee.email,
        firstname: attendee.name.split(" ")[0],
        lastname: attendee.name.split(" ").slice(1).join(" "),
        phone: attendee.phone || "",
        cal_booking_source: "Cal.com",
      },
    }),
  });

  return response.json();
};

// Create deal for booking
export const createDeal = async (
  credential: Credential,
  booking: Booking,
  contactId: string
) => {
  const tokens = credential.key;

  const response = await fetch("https://api.hubapi.com/crm/v3/objects/deals", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      Authorization: `Bearer ${tokens.access_token}`,
    },
    body: JSON.stringify({
      properties: {
        dealname: `Booking: ${booking.title}`,
        amount: booking.payment?.[0]?.amount || 0,
        pipeline: "default",
        dealstage: "appointmentscheduled",
        cal_booking_id: booking.id,
        cal_booking_uid: booking.uid,
      },
      associations: [
        {
          to: { id: contactId },
          types: [
            {
              associationCategory: "HUBSPOT_DEFINED",
              associationTypeId: 3, // Contact to Deal
            },
          ],
        },
      ],
    }),
  });

  return response.json();
};
```

**Location**: `packages/app-store/hubspot/lib/contacts.ts:1`

---

### 6.5 Zapier

**Type**: `automation`
**Authentication**: API Key
**Trigger**: Webhooks

**Features**:
- ✅ Booking created trigger
- ✅ Booking cancelled trigger
- ✅ Booking rescheduled trigger
- ✅ 5000+ app integrations

**Setup**:
1. User creates Zap in Zapier
2. Adds Cal.com trigger
3. Cal.com creates webhook subscription
4. Booking events trigger Zap

**Webhook Payload** (sent to Zapier):
```json
{
  "triggerEvent": "BOOKING_CREATED",
  "createdAt": "2025-11-18T10:00:00Z",
  "payload": {
    "uid": "abc123",
    "title": "30 Min Meeting",
    "startTime": "2025-11-19T14:00:00Z",
    "endTime": "2025-11-19T14:30:00Z",
    "organizer": {
      "name": "John Doe",
      "email": "john@example.com"
    },
    "attendees": [
      {
        "name": "Jane Smith",
        "email": "jane@example.com"
      }
    ],
    "location": "Zoom",
    "status": "ACCEPTED"
  }
}
```

**Location**: `packages/app-store/zapier/_metadata.ts:1`

---

## 7. App Installation & Management

### 7.1 Installation Flow

```
1. User navigates to /apps
   ↓
2. Browses app marketplace
   ↓
3. Clicks "Install" on app
   ↓
4. If OAuth: Redirect to provider consent page
   ↓
5. User authorizes app
   ↓
6. Redirect to callback: /api/integrations/[app]/callback
   ↓
7. Store credential in database
   ↓
8. Redirect to post-install setup (if needed)
   ↓
9. App appears in /apps/installed
```

### 7.2 App Discovery

**API Endpoint**: `GET /api/trpc/viewer/integrations`

**Response**:
```typescript
{
  "result": {
    "data": {
      "items": {
        "conferencing": [
          {
            "slug": "zoom",
            "name": "Zoom Video",
            "description": "Video conferencing with Zoom",
            "logo": "/api/app-store/zoomvideo/icon.svg",
            "installed": true,
            "type": "zoom_video",
            "variant": "conferencing",
            "categories": ["conferencing"],
            "credentials": [{ "id": 1, "type": "zoom_video" }],
          },
          // ... more conferencing apps
        ],
        "calendar": [ /* ... */ ],
        "payment": [ /* ... */ ],
      }
    }
  }
}
```

### 7.3 Credential Management

**Get User's Credentials**:
```typescript
// packages/app-store/_utils/getConnectedApps.ts
export const getConnectedApps = async (userId: number) => {
  const credentials = await prisma.credential.findMany({
    where: {
      userId,
      invalid: false,
    },
    include: {
      user: true,
    },
  });

  return credentials.map(credential => ({
    id: credential.id,
    type: credential.type,
    appId: credential.appId,
    invalid: credential.invalid,
  }));
};
```

**Invalidate Credential** (on error):
```typescript
// packages/app-store/_utils/invalidateCredential.ts
export const invalidateCredential = async (credentialId: number) => {
  await prisma.credential.update({
    where: { id: credentialId },
    data: { invalid: true },
  });
};
```

**Location**: `packages/app-store/_utils/getConnectedApps.ts:1`

---

## 8. Building New Apps

### 8.1 App Scaffold

Use the template to create new apps:

```bash
# Copy template
cp -r packages/app-store/templates/my-new-app packages/app-store/mynewapp
cd packages/app-store/mynewapp

# Edit metadata
vim _metadata.ts
```

### 8.2 Minimal App Structure

```typescript
// _metadata.ts
export const metadata = {
  name: "My App",
  description: "Integration with My Service",
  type: "myapp_calendar",
  title: "My App",
  variant: "calendar",
  category: "calendar",
  categories: ["calendar"],
  logo: "icon.svg",
  publisher: "Your Company",
  slug: "my-app",
  url: "https://myapp.com",
  email: "support@myapp.com",
  dirName: "mynewapp",
  isOAuth: true,
} as AppMeta;

// lib/CalendarService.ts
import type { Calendar } from "@calcom/types/Calendar";

export default class MyAppCalendarService implements Calendar {
  private credential: Credential;

  constructor(credential: Credential) {
    this.credential = credential;
  }

  async listCalendars() {
    // Implementation
  }

  async getAvailability(dateFrom: string, dateTo: string, selectedCalendars: IntegrationCalendar[]) {
    // Implementation
  }

  async createEvent(event: CalendarEvent) {
    // Implementation
  }

  async updateEvent(uid: string, event: CalendarEvent) {
    // Implementation
  }

  async deleteEvent(uid: string) {
    // Implementation
  }

  getCredentialId() {
    return this.credential.id;
  }
}

// api/callback.ts
export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  const { code } = req.query;

  // Exchange code for tokens
  const tokens = await exchangeCodeForTokens(code);

  // Store credential
  await prisma.credential.create({
    data: {
      userId: req.session.user.id,
      type: "myapp_calendar",
      key: tokens,
      appId: "my-app",
    },
  });

  res.redirect("/apps/installed/calendar");
}

// index.ts
export { default as metadata } from "./_metadata";
export { default as CalendarService } from "./lib/CalendarService";
```

### 8.3 Testing Your App

```typescript
// __tests__/CalendarService.test.ts
import { describe, it, expect, vi } from "vitest";
import CalendarService from "../lib/CalendarService";

describe("MyAppCalendarService", () => {
  it("should list calendars", async () => {
    const mockCredential = {
      id: 1,
      type: "myapp_calendar",
      key: { access_token: "test" },
    };

    const service = new CalendarService(mockCredential);
    const calendars = await service.listCalendars();

    expect(calendars).toBeDefined();
    expect(calendars.length).toBeGreaterThan(0);
  });

  it("should create event", async () => {
    const service = new CalendarService(mockCredential);

    const event = await service.createEvent({
      title: "Test Event",
      startTime: "2025-11-19T14:00:00Z",
      endTime: "2025-11-19T14:30:00Z",
      organizer: {
        email: "test@example.com",
        name: "Test User",
        timeZone: "America/New_York",
      },
      attendees: [],
    });

    expect(event.uid).toBeDefined();
  });
});
```

---

## 9. Testing & Quality

### 9.1 Test Utilities

**Location**: `packages/app-store/_utils/testUtils.ts`

```typescript
export const createMockCredential = (overrides?: Partial<Credential>): Credential => ({
  id: 1,
  userId: 1,
  type: "test_app",
  key: { access_token: "test_token" },
  appId: "test",
  invalid: false,
  ...overrides,
});

export const createMockCalendarEvent = (overrides?: Partial<CalendarEvent>): CalendarEvent => ({
  uid: "test-uid",
  title: "Test Event",
  startTime: new Date().toISOString(),
  endTime: new Date(Date.now() + 30 * 60 * 1000).toISOString(),
  organizer: {
    id: 1,
    email: "organizer@example.com",
    name: "Organizer",
    timeZone: "UTC",
  },
  attendees: [],
  ...overrides,
});
```

### 9.2 Integration Tests

Run integration tests for all apps:

```bash
# Test all calendar apps
yarn test:integration calendar

# Test specific app
yarn test packages/app-store/googlecalendar
```

---

## 10. Best Practices

### 10.1 Error Handling

```typescript
// ✅ Good: Comprehensive error handling
async createEvent(event: CalendarEvent) {
  try {
    const client = await this.getClient();
    const result = await client.createEvent(event);
    return result;
  } catch (error) {
    if (error.code === 401) {
      // Invalidate credential
      await invalidateCredential(this.credential.id);
      throw new Error("Authentication expired. Please reconnect the app.");
    }

    if (error.code === 409) {
      throw new Error("Event already exists in calendar.");
    }

    // Log error with context
    logger.error("Failed to create event", {
      error,
      credentialId: this.credential.id,
      eventTitle: event.title,
    });

    throw new Error(`Failed to create event: ${error.message}`);
  }
}

// ❌ Bad: Silent failures
async createEvent(event: CalendarEvent) {
  const client = await this.getClient();
  return client.createEvent(event).catch(() => null);
}
```

### 10.2 Rate Limiting

```typescript
// ✅ Good: Respect API rate limits
import pLimit from "p-limit";

const limit = pLimit(5); // Max 5 concurrent requests

async getAvailability(calendars: IntegrationCalendar[]) {
  const busyTimes = await Promise.all(
    calendars.map(cal =>
      limit(() => this.getCalendarBusyTimes(cal))
    )
  );

  return busyTimes.flat();
}
```

### 10.3 Caching

```typescript
// ✅ Good: Cache expensive operations
import { CalendarCache } from "@calcom/features/calendar-cache";

async getAvailability(dateFrom: string, dateTo: string, selectedCalendars: IntegrationCalendar[]) {
  const cacheKey = `availability:${this.credential.id}:${dateFrom}:${dateTo}`;

  // Check cache
  const cached = await CalendarCache.get(cacheKey);
  if (cached) return cached;

  // Fetch from API
  const busyTimes = await this.fetchBusyTimes(dateFrom, dateTo, selectedCalendars);

  // Cache for 5 minutes
  await CalendarCache.set(cacheKey, busyTimes, 300);

  return busyTimes;
}
```

### 10.4 Security

```typescript
// ✅ Good: Validate webhook signatures
export default async function webhookHandler(req: NextApiRequest, res: NextApiResponse) {
  const signature = req.headers["x-myapp-signature"];
  const body = await buffer(req);

  // Verify signature
  const expectedSignature = crypto
    .createHmac("sha256", process.env.MYAPP_WEBHOOK_SECRET!)
    .update(body)
    .digest("hex");

  if (signature !== expectedSignature) {
    return res.status(401).json({ error: "Invalid signature" });
  }

  // Process webhook
  const event = JSON.parse(body.toString());
  await handleWebhookEvent(event);

  res.status(200).json({ success: true });
}
```

---

## Summary

### Key Takeaways

1. **107 Apps**: Calendar, video, payment, CRM, analytics, automation, and more

2. **App Structure**:
   - `_metadata.ts`: App configuration
   - `lib/`: Service implementation (Calendar, Video, Payment)
   - `api/`: OAuth callbacks and webhooks
   - `components/`: UI components

3. **Integration Patterns**:
   - **Calendar**: Implement `Calendar` interface
   - **Video**: Create/update/delete meetings
   - **Payment**: Stripe Checkout + webhooks
   - **CRM**: Sync contacts via webhooks

4. **OAuth Flow**: Redirect → Consent → Callback → Store Encrypted Credential

5. **Credential Management**: AES-256 encryption, auto-refresh, invalidation on error

6. **Best Practices**: Error handling, rate limiting, caching, webhook security

### App Development Checklist

- [ ] Create `_metadata.ts` with app info
- [ ] Implement service class (`CalendarService`, `VideoService`, etc.)
- [ ] Add OAuth callback handler
- [ ] Add webhook handler (if needed)
- [ ] Create post-install setup page (if needed)
- [ ] Add app icon (`static/icon.svg`)
- [ ] Write integration tests
- [ ] Document environment variables
- [ ] Test OAuth flow end-to-end
- [ ] Test error scenarios (expired tokens, rate limits)

### Next Steps

- **PHASE 7**: Booking Flow End-to-End
- **PHASE 8**: Authentication & Authorization

---

**Tác giả**: Claude (AI Assistant)
**Ngày tạo**: 2025-11-18
**Phiên bản**: 1.0
