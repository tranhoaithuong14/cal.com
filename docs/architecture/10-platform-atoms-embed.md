# PHASE 10: Platform & Atoms (Embed/SDK)

> **Mục tiêu**: Hiểu Platform SDK (@calcom/atoms), Embed functionality, và cách Cal.com được nhúng vào third-party websites.

## Overview

Cal.com provides **two main ways** to integrate scheduling into external applications:

1. **Embed**: JavaScript snippet to embed Cal.com booking flow
2. **Atoms (Platform SDK)**: React components for building custom UIs

```
Integration Methods:
┌──────────────────────────────────────────────────────────┐
│                    Cal.com Integration                    │
├────────────────────────────┬─────────────────────────────┤
│         Embed              │         Atoms (SDK)          │
│                            │                              │
│  Type: JavaScript snippet  │  Type: React components      │
│  Control: Low (preset UI)  │  Control: High (custom UI)   │
│  Setup: Copy/paste code    │  Setup: npm install          │
│  Customization: Limited    │  Customization: Full         │
│  Auth: None needed         │  Auth: OAuth 2.0             │
│  Use case: Simple embed    │  Use case: Custom app        │
└────────────────────────────┴─────────────────────────────┘
```

---

## 1. Embed

### 1.1 Embed Types

**Inline Embed**: Embedded directly in page
```html
<div
  data-cal-link="john/30min"
  data-cal-config='{"layout":"month_view"}'
  style="width:100%;height:100%;overflow:scroll"
></div>
```

**Popup Modal**: Opens in modal on button click
```html
<button data-cal-link="john/30min">Book a meeting</button>
```

**Floating Button**: Fixed button on page
```html
<div
  data-cal-link="john/30min"
  data-cal-config='{"layout":"month_view","floatingButton":{"position":"bottom-right"}}'
></div>
```

### 1.2 Embed Code

```html
<!-- 1. Include embed script -->
<script type="text/javascript">
(function (C, A, L) {
  let p = function (a, ar) {
    a.q.push(ar);
  };
  let d = C.document;
  C.Cal = C.Cal || function () {
    let cal = C.Cal;
    let ar = arguments;
    if (!cal.loaded) {
      cal.ns = {};
      cal.q = cal.q || [];
      d.head.appendChild(d.createElement("script")).src = A;
      cal.loaded = true;
    }
    if (ar[0] === L) {
      const api = function () {
        p(api, arguments);
      };
      const namespace = ar[1];
      api.q = api.q || [];
      typeof namespace === "string"
        ? (cal.ns[namespace] = api) && p(api, ar)
        : p(cal, ar);
      return;
    }
    p(cal, ar);
  };
})(window, "https://app.cal.com/embed/embed.js", "init");

Cal("init", { origin: "https://app.cal.com" });
</script>

<!-- 2. Add booking link -->
<button data-cal-link="john/30min">Schedule time with me</button>
```

### 1.3 Embed Configuration

```javascript
Cal("init", {
  origin: "https://app.cal.com",
});

// Inline embed with config
Cal("inline", {
  elementOrSelector: "#my-cal-inline",
  calLink: "john/30min",
  layout: "month_view",
  config: {
    theme: "dark",
    hideEventTypeDetails: false,
  },
});

// Popup modal
Cal("modal", {
  calLink: "john/30min",
  config: {
    layout: "column_view",
    theme: "auto",
  },
});

// Listen to events
Cal("on", {
  action: "bookingSuccessful",
  callback: (e) => {
    console.log("Booking successful:", e.detail);
    // e.detail = { booking: {...}, event: {...} }
  },
});
```

**Configuration Options**:
```typescript
interface EmbedConfig {
  layout?: "month_view" | "week_view" | "column_view";
  theme?: "light" | "dark" | "auto";
  hideEventTypeDetails?: boolean;
  hideLandingPageDetails?: boolean;

  // Styling
  styles?: {
    branding?: {
      brandColor?: string;
      darkBrandColor?: string;
    };
  };

  // UI
  ui?: {
    hideEventTypeDetails?: boolean;
    colorScheme?: "light" | "dark";
  };
}
```

### 1.4 Embed Events

```javascript
// Listen to embed events
Cal("on", {
  action: "bookingSuccessful",
  callback: (e) => {
    // Booking completed
    console.log(e.detail.booking);
  },
});

Cal("on", {
  action: "linkReady",
  callback: (e) => {
    // Embed loaded
  },
});

Cal("on", {
  action: "linkFailed",
  callback: (e) => {
    // Embed failed to load
  },
});

Cal("on", {
  action: "__routeChanged",
  callback: (e) => {
    // Route changed (date selected, form shown, etc.)
  },
});
```

**Location**: `packages/embed-core/`

---

## 2. Atoms (Platform SDK)

### 2.1 Overview

**Atoms** are React components that provide full control over the booking UI.

**Technology**: React + Platform API (v2)

**Installation**:
```bash
npm install @calcom/atoms
```

**Authentication**: Requires OAuth 2.0 access token from Platform API

### 2.2 Basic Usage

```tsx
import {
  CalProvider,
  useAtomsContext,
  BookerLayout,
} from "@calcom/atoms";

function App() {
  return (
    <CalProvider
      accessToken="your-oauth-token"
      apiUrl="https://api.cal.com/v2"
    >
      <BookingFlow />
    </CalProvider>
  );
}

function BookingFlow() {
  const { bookerState } = useAtomsContext();

  return (
    <div>
      <BookerLayout
        username="john"
        eventSlug="30min"
        onBookingSuccess={(booking) => {
          console.log("Booking created:", booking);
        }}
      />
    </div>
  );
}
```

### 2.3 Available Components

```tsx
// Provider (required)
<CalProvider
  accessToken="..."
  apiUrl="https://api.cal.com/v2"
  clientId="your-client-id"
  options={{
    locale: "en",
    timeZone: "America/New_York",
  }}
>
  {children}
</CalProvider>

// Complete booker (all-in-one)
<BookerLayout
  username="john"
  eventSlug="30min"
  layout="month_view"
  onBookingSuccess={(booking) => {}}
  onBookingError={(error) => {}}
/>

// Individual components
<CalendarPicker
  selectedDate={date}
  onDateChange={setDate}
/>

<TimeSlotPicker
  date={selectedDate}
  eventTypeId={eventTypeId}
  onTimeSlotSelect={setSelectedSlot}
/>

<BookingForm
  eventTypeId={eventTypeId}
  selectedSlot={selectedSlot}
  onSubmit={handleBooking}
/>

<BookingConfirmation
  booking={booking}
/>
```

### 2.4 Customization

```tsx
import { BookerLayout } from "@calcom/atoms";

<BookerLayout
  username="john"
  eventSlug="30min"

  // Layout
  layout="month_view" // or "week_view", "column_view"

  // Styling
  customClassNames={{
    bookerContainer: "my-custom-container",
    datePickerContainer: "my-date-picker",
    timeSlotContainer: "my-time-slot",
  }}

  // Theme
  theme={{
    primaryColor: "#3b82f6",
    backgroundColor: "#ffffff",
    textColor: "#000000",
  }}

  // Callbacks
  onBookingSuccess={(booking) => {
    console.log("Success:", booking);
  }}

  onBookingError={(error) => {
    console.error("Error:", error);
  }}

  onDateChange={(date) => {
    console.log("Date changed:", date);
  }}
/>
```

### 2.5 Atoms Context

```tsx
import { useAtomsContext } from "@calcom/atoms";

function CustomComponent() {
  const {
    // State
    bookerState,        // "loading" | "selecting_date" | "selecting_time" | "booking" | "success"
    selectedDate,
    selectedSlot,
    bookingData,

    // Actions
    setSelectedDate,
    setSelectedSlot,
    createBooking,

    // Data
    eventTypes,
    availableSlots,
  } = useAtomsContext();

  return (
    <div>
      {bookerState === "selecting_date" && (
        <CustomDatePicker onSelect={setSelectedDate} />
      )}

      {bookerState === "selecting_time" && (
        <CustomTimeSlots
          slots={availableSlots}
          onSelect={setSelectedSlot}
        />
      )}

      {bookerState === "booking" && (
        <CustomBookingForm onSubmit={createBooking} />
      )}

      {bookerState === "success" && (
        <CustomConfirmation booking={bookingData} />
      )}
    </div>
  );
}
```

### 2.6 API Integration

Atoms use Platform API (v2) under the hood:

```typescript
// Atoms internally make these API calls:

// Get event type
GET /v2/event-types/{username}/{eventSlug}

// Get available slots
GET /v2/slots?eventTypeId=123&startTime=2025-11-19T00:00:00Z&endTime=2025-11-19T23:59:59Z

// Create booking
POST /v2/bookings
{
  "eventTypeId": 123,
  "start": "2025-11-19T14:00:00Z",
  "end": "2025-11-19T14:30:00Z",
  "responses": {
    "name": "John Doe",
    "email": "john@example.com"
  }
}
```

**Location**: `packages/atoms/`

---

## 3. Comparison: Embed vs Atoms

| Feature | Embed | Atoms (SDK) |
|---------|-------|-------------|
| **Setup Complexity** | Low (copy/paste) | Medium (npm + OAuth) |
| **UI Control** | Low | High |
| **Customization** | Limited | Full |
| **Framework** | Vanilla JS | React |
| **Authentication** | None | OAuth 2.0 |
| **Bundle Size** | ~50KB | ~200KB |
| **Use Case** | Simple embedding | Custom applications |
| **Styling** | CSS overrides | Props + CSS |
| **Events** | Limited | Full control |
| **Mobile Support** | Yes | Yes |
| **TypeScript** | No | Yes |

---

## 4. Platform Features

### 4.1 OAuth Apps

Register OAuth app to use Atoms:

```typescript
// 1. Register app via Platform API
POST /v2/oauth/clients
{
  "name": "My Scheduling App",
  "redirectUris": ["https://myapp.com/callback"],
  "scopes": ["bookings:read", "bookings:write"]
}

// Response:
{
  "clientId": "xxx",
  "clientSecret": "yyy"
}

// 2. Implement OAuth flow
// (See PHASE 9 for details)

// 3. Use access token with Atoms
<CalProvider accessToken={accessToken}>
  <BookerLayout ... />
</CalProvider>
```

### 4.2 Managed Users

Platform API allows creating managed users:

```typescript
POST /v2/users
{
  "email": "user@example.com",
  "name": "John Doe",
  "timeZone": "America/New_York"
}

// Response:
{
  "id": 123,
  "email": "user@example.com",
  "username": "john-doe-xyz"
}
```

### 4.3 Managed Event Types

Create event types on behalf of users:

```typescript
POST /v2/users/{userId}/event-types
{
  "title": "30 Minute Meeting",
  "slug": "30min",
  "length": 30,
  "locations": [
    { "type": "integrations:zoom" }
  ]
}
```

---

## 5. Advanced Use Cases

### 5.1 White-Label Scheduling Platform

```tsx
import { CalProvider, BookerLayout } from "@calcom/atoms";

function WhiteLabelScheduler({ customerId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    // Get or create managed user
    async function init() {
      const customer = await getCustomer(customerId);

      let calUser = await findCalUser(customer.email);

      if (!calUser) {
        // Create managed user via Platform API
        calUser = await fetch("/api/v2/users", {
          method: "POST",
          headers: {
            "Authorization": `Bearer ${platformAccessToken}`,
          },
          body: JSON.stringify({
            email: customer.email,
            name: customer.name,
            timeZone: customer.timeZone,
          }),
        }).then(r => r.json());
      }

      setUser(calUser);
    }

    init();
  }, [customerId]);

  if (!user) return <Loading />;

  return (
    <CalProvider accessToken={platformAccessToken}>
      <BookerLayout
        username={user.username}
        eventSlug="consultation"
        theme={{
          primaryColor: "#your-brand-color",
        }}
        onBookingSuccess={async (booking) => {
          // Sync to your database
          await syncBookingToDatabase(booking);
        }}
      />
    </CalProvider>
  );
}
```

### 5.2 Multi-Calendar Marketplace

```tsx
function CalendarMarketplace() {
  const [experts, setExperts] = useState([]);

  return (
    <div>
      <h1>Book an Expert</h1>

      {experts.map(expert => (
        <ExpertCard key={expert.id}>
          <h3>{expert.name}</h3>
          <p>{expert.bio}</p>

          <CalProvider accessToken={platformAccessToken}>
            <BookerLayout
              username={expert.calUsername}
              eventSlug={expert.eventSlug}
              layout="column_view"
            />
          </CalProvider>
        </ExpertCard>
      ))}
    </div>
  );
}
```

---

## Summary

### Embed

- ✅ Use for **simple integration** (marketing sites, landing pages)
- ✅ No authentication required
- ✅ Minimal setup (copy/paste)
- ❌ Limited customization
- ❌ Cal.com branding visible

### Atoms (SDK)

- ✅ Use for **custom applications** (SaaS products, white-label)
- ✅ Full UI control
- ✅ TypeScript support
- ✅ React ecosystem
- ❌ Requires OAuth setup
- ❌ More complex integration

### Platform API

- Managed users
- Managed event types
- OAuth 2.0 authentication
- Programmatic booking creation
- Webhook integrations

---

**Tác giả**: Claude (AI Assistant)
**Ngày tạo**: 2025-11-18
**Phiên bản**: 1.0
