# PHASE 7: Booking Flow End-to-End

> **Mục tiêu**: Hiểu complete booking flow từ lúc user chọn time slot đến booking confirmed, bao gồm UI, backend logic, calendar integration, và notifications.

## 📋 Table of Contents

- [1. Overview](#1-overview)
- [2. User Journey (Frontend Flow)](#2-user-journey-frontend-flow)
- [3. Backend Booking Flow](#3-backend-booking-flow)
- [4. EventManager & Calendar Integration](#4-eventmanager--calendar-integration)
- [5. Email & SMS Notifications](#5-email--sms-notifications)
- [6. Edge Cases & Variations](#6-edge-cases--variations)
- [7. Error Handling & Rollback](#7-error-handling--rollback)
- [8. Performance & Optimization](#8-performance--optimization)
- [9. Security Considerations](#9-security-considerations)
- [10. Testing the Booking Flow](#10-testing-the-booking-flow)

---

## 1. Overview

### 1.1 High-Level Flow

```
User Journey:
┌──────────────┐
│ Booking Page │ → User selects date/time
└──────┬───────┘
       ↓
┌──────────────┐
│ Time Slots   │ → Available times displayed
└──────┬───────┘
       ↓
┌──────────────┐
│ Booking Form │ → User fills out details
└──────┬───────┘
       ↓
┌──────────────┐
│ Confirmation │ → User confirms (or payment if paid event)
└──────┬───────┘
       ↓
Backend Processing:
┌──────────────┐
│ Validation   │ → Validate input, check conflicts
└──────┬───────┘
       ↓
┌──────────────┐
│ Create Booking│ → Save to database
└──────┬───────┘
       ↓
┌──────────────┐
│ EventManager │ → Create calendar events & video meetings
└──────┬───────┘
       ↓
┌──────────────┐
│ Notifications│ → Send emails/SMS to attendees & organizer
└──────┬───────┘
       ↓
┌──────────────┐
│ Webhooks     │ → Trigger webhooks & workflows
└──────────────┘
```

### 1.2 Key Components

| Component | Purpose | Location |
|-----------|---------|----------|
| **Booker** | Main UI component | `packages/features/bookings/Booker/Booker.tsx` |
| **handleNewBooking** | Backend booking logic | `packages/features/bookings/lib/handleNewBooking/` |
| **EventManager** | Calendar/video integration | `packages/features/bookings/lib/EventManager.ts` |
| **BookingEmailSmsHandler** | Notifications | `packages/features/bookings/lib/BookingEmailSmsHandler.ts` |

### 1.3 Database Flow

```sql
-- 1. Create booking
INSERT INTO Booking (uid, title, startTime, endTime, userId, eventTypeId, status, ...)
VALUES (...);

-- 2. Create attendees
INSERT INTO Attendee (bookingId, email, name, timeZone, locale)
VALUES (...);

-- 3. Create booking references (calendar events, video meetings)
INSERT INTO BookingReference (bookingId, type, uid, meetingId, meetingPassword, meetingUrl, ...)
VALUES (...);

-- 4. Create workflows/reminders
INSERT INTO WorkflowReminder (bookingId, workflowStepId, scheduledDate, method)
VALUES (...);

-- 5. Create payment (if paid event)
INSERT INTO Payment (bookingId, amount, currency, externalId, success, ...)
VALUES (...);
```

---

## 2. User Journey (Frontend Flow)

### 2.1 Booker Component Architecture

**Location**: `packages/features/bookings/Booker/Booker.tsx:48`

The Booker is a **state-driven** component using Zustand for state management:

```typescript
// Booker states
type BookerState =
  | "loading"           // Initial load
  | "selecting_date"    // Date picker visible
  | "selecting_time"    // Time slots visible
  | "booking"           // Booking form visible
  | "success"           // Booking confirmed
  | "error";            // Error occurred

// Booker layouts
enum BookerLayout {
  MONTH_VIEW = "month_view",         // Calendar + slots side-by-side
  WEEK_VIEW = "week_view",           // Week view + slots
  COLUMN_VIEW = "column_view",       // Vertical layout
}
```

**Component Hierarchy**:
```tsx
<Booker>
  <Header />                           {/* Event type info */}
  <EventMeta />                        {/* Duration, location */}

  {/* Layout-dependent components */}
  {layout === "month_view" && (
    <>
      <DatePicker />                   {/* Calendar month view */}
      <AvailableTimeSlots />           {/* Time slots for selected date */}
    </>
  )}

  {layout === "week_view" && (
    <LargeCalendar />                  {/* Week view with inline slots */}
  )}

  {state === "booking" && (
    <BookEventForm />                  {/* Booking form */}
  )}

  {state === "success" && (
    <BookingSuccessCard />             {/* Confirmation */}
  )}

  <PoweredBy />                        {/* Branding */}
</Booker>
```

### 2.2 Date & Time Selection Flow

```typescript
// 1. User lands on booking page
// State: "selecting_date"
// Display: DatePicker component

// 2. Fetch available dates
const { data: schedule } = trpc.viewer.public.schedule.useQuery({
  username,
  eventTypeSlug,
  month: currentMonth,
});

// schedule = {
//   "2025-11-19": true,  // Available
//   "2025-11-20": true,
//   "2025-11-21": false, // Not available
// }

// 3. User selects date
onClick={(date) => setSelectedDate(date)}
// State transitions to: "selecting_time"

// 4. Fetch available slots for selected date
const { data: slots } = trpc.viewer.public.slots.useQuery({
  eventTypeId,
  startTime: selectedDate,
  endTime: endOfDay(selectedDate),
  timeZone: userTimeZone,
});

// slots = {
//   slots: {
//     "2025-11-19T14:00:00Z": [{ time: "14:00", attendees: 1 }],
//     "2025-11-19T14:30:00Z": [{ time: "14:30", attendees: 1 }],
//     "2025-11-19T15:00:00Z": [{ time: "15:00", attendees: 1 }],
//   }
// }

// 5. Render time slots
<AvailableTimeSlots>
  {Object.entries(slots.slots).map(([time, slot]) => (
    <TimeSlot
      time={time}
      onClick={() => {
        setSelectedSlot(time);
        setBookerState("booking"); // Show booking form
      }}
    />
  ))}
</AvailableTimeSlots>
```

**Location**: `packages/features/bookings/Booker/components/AvailableTimeSlots.tsx:1`

### 2.3 Booking Form

**Location**: `packages/features/bookings/Booker/components/BookEventForm/BookEventForm.tsx:1`

```tsx
<BookEventForm>
  {/* Attendee Information */}
  <BookingFields
    fields={eventType.bookingFields}
    values={formValues}
    onChange={setFormValues}
  />

  {/* Example fields */}
  <Input name="name" label="Your Name" required />
  <Input name="email" label="Email" type="email" required />
  <Input name="phone" label="Phone Number" />
  <Textarea name="notes" label="Additional Notes" />

  {/* Custom Fields (defined by event type) */}
  {eventType.bookingFields.map(field => (
    <BookingField
      type={field.type}           // text, email, phone, select, etc.
      name={field.name}
      label={field.label}
      required={field.required}
      options={field.options}
    />
  ))}

  {/* Guest Emails (optional) */}
  {eventType.allowGuests && (
    <MultiEmail
      name="guests"
      label="Add Guests"
      placeholder="email@example.com"
    />
  )}

  {/* Location (if multiple options) */}
  {eventType.locations.length > 1 && (
    <Select name="location" label="Location">
      {eventType.locations.map(loc => (
        <option value={loc.type}>{loc.label}</option>
      ))}
    </Select>
  )}

  {/* Reschedule Reason (if rescheduling) */}
  {isRescheduling && (
    <Textarea
      name="rescheduleReason"
      label="Reason for rescheduling"
      required
    />
  )}

  {/* Submit Button */}
  <Button
    type="submit"
    loading={isSubmitting}
    disabled={!isValid || isSubmitting}
  >
    {isPaidEvent ? "Continue to Payment" : "Confirm Booking"}
  </Button>
</BookEventForm>
```

### 2.4 Form Submission

```typescript
// Submit handler
const onSubmit = async (formData: BookingFormData) => {
  try {
    setIsSubmitting(true);

    // Build booking mutation input
    const input = {
      eventTypeId: eventType.id,
      start: selectedSlot.toISOString(),
      end: addMinutes(selectedSlot, eventType.length).toISOString(),
      timeZone: Intl.DateTimeFormat().resolvedOptions().timeZone,
      language: i18n.language,

      // Attendee info
      responses: {
        name: formData.name,
        email: formData.email,
        phone: formData.phone,
        notes: formData.notes,
        location: formData.location,
        guests: formData.guests,
        ...customFieldResponses,
      },

      // Metadata
      metadata: {
        videoCallUrl: videoCallData?.url,
      },

      // Reschedule
      rescheduleUid: rescheduleUid || undefined,
      rescheduleReason: formData.rescheduleReason,

      // Hashed link (for private links)
      hashedLink: hashedLink || undefined,
    };

    // Call tRPC mutation
    const booking = await bookingMutation.mutateAsync(input);

    // Handle payment events
    if (isPaidEvent && booking.paymentUrl) {
      // Redirect to Stripe Checkout
      window.location.href = booking.paymentUrl;
      return;
    }

    // Show success message
    setBookerState("success");
    setBookingData(booking);

    // Trigger webhooks, analytics, etc.
    await onBookingSuccess(booking);

  } catch (error) {
    setBookerState("error");
    setError(error.message);
  } finally {
    setIsSubmitting(false);
  }
};
```

---

## 3. Backend Booking Flow

### 3.1 Entry Point: `handleNewBooking`

**Location**: `packages/features/bookings/lib/handleNewBooking/handleNewBooking.ts:1`

**Main Handler**:
```typescript
export async function handleNewBooking(req: NextApiRequest) {
  // 1. Parse and validate request
  const input = bookingCreateBodySchema.parse(req.body);

  // 2. Get event type with all relations
  const eventType = await getEventType(input);

  // 3. Check if booker email is blocked
  await checkIfBookerEmailIsBlocked(input.responses.email, eventType);

  // 4. Ensure users are available
  const { availableUsers, selectedUsers } = await ensureAvailableUsers(
    eventType,
    input
  );

  // 5. Check booking limits
  await checkBookingAndDurationLimits({
    eventType,
    bookerEmail: input.responses.email,
    startDate: input.start,
  });

  // 6. Check for conflicts
  const conflicts = await checkForConflicts({
    users: availableUsers,
    start: input.start,
    end: input.end,
  });

  if (conflicts.length > 0) {
    throw new Error("Selected time is no longer available");
  }

  // 7. Build calendar event
  const evt = await buildCalendarEvent({
    input,
    eventType,
    selectedUsers,
  });

  // 8. Create booking in database
  const newBooking = await createBooking({
    evt,
    eventType,
    reqBodyUser: input.user,
    reqBodyMetadata: input.metadata,
    reqBodyRecurringEventId: input.recurringEventId,
  });

  // 9. Create calendar events & video meetings
  const results = await EventManager.create(evt);

  // 10. Store booking references
  await storeBookingReferences(newBooking.id, results);

  // 11. Send notifications
  await sendScheduledEmails(evt, newBooking);

  // 12. Trigger webhooks
  await triggerWebhooks(evt, newBooking);

  // 13. Execute workflows
  await scheduleWorkflowReminders(newBooking, eventType);

  return {
    uid: newBooking.uid,
    id: newBooking.id,
    ...newBooking,
  };
}
```

### 3.2 Validation Steps

**1. Schema Validation** (`bookingCreateBodySchema`)

```typescript
const bookingCreateBodySchema = z.object({
  eventTypeId: z.number(),
  start: z.string().datetime(),
  end: z.string().datetime(),
  timeZone: z.string(),
  language: z.string(),

  responses: z.object({
    name: z.string().min(1),
    email: z.string().email(),
    phone: z.string().optional(),
    notes: z.string().optional(),
    guests: z.array(z.string().email()).optional(),
    location: z.string().optional(),
    // ... custom fields
  }),

  metadata: z.record(z.any()).optional(),
  hashedLink: z.string().optional(),
  rescheduleUid: z.string().optional(),
  rescheduleReason: z.string().optional(),
});
```

**2. Email Blocking Check**

```typescript
async function checkIfBookerEmailIsBlocked(
  email: string,
  eventType: EventType
) {
  // Check if email matches blocked patterns
  const blockedEmailPatterns = eventType.metadata?.blockedEmailPatterns || [];

  for (const pattern of blockedEmailPatterns) {
    const regex = new RegExp(pattern);
    if (regex.test(email)) {
      throw new Error(`Email ${email} is blocked from booking`);
    }
  }
}
```

**3. Availability Check**

```typescript
async function ensureAvailableUsers(
  eventType: EventType,
  input: BookingInput
) {
  // Get all hosts for this event type
  const hosts = await getEventTypeHosts(eventType);

  // Check each host's availability
  const availableHosts = await Promise.all(
    hosts.map(async (host) => {
      const isAvailable = await checkUserAvailability({
        userId: host.userId,
        startTime: input.start,
        endTime: input.end,
        eventTypeId: eventType.id,
      });

      return isAvailable ? host : null;
    })
  );

  const availableUsers = availableHosts.filter(Boolean);

  if (availableUsers.length === 0) {
    throw new Error("No available users for selected time");
  }

  // For round-robin, select user
  let selectedUsers = availableUsers;
  if (eventType.schedulingType === "ROUND_ROBIN") {
    selectedUsers = [await getLuckyUser(eventType, availableUsers)];
  }

  return { availableUsers, selectedUsers };
}
```

**4. Booking Limits**

```typescript
async function checkBookingAndDurationLimits({
  eventType,
  bookerEmail,
  startDate,
}: CheckLimitsParams) {
  // Check booking limits (per day/week/month/year)
  if (eventType.bookingLimits) {
    const bookingCount = await getBookingCountForPeriod({
      eventTypeId: eventType.id,
      bookerEmail,
      period: "DAY",
      startDate,
    });

    if (bookingCount >= eventType.bookingLimits.PER_DAY) {
      throw new Error("Daily booking limit reached");
    }
  }

  // Check duration limits (total booking duration per period)
  if (eventType.durationLimits) {
    const totalDuration = await getTotalBookingDuration({
      eventTypeId: eventType.id,
      bookerEmail,
      period: "WEEK",
      startDate,
    });

    if (totalDuration + eventType.length >= eventType.durationLimits.PER_WEEK) {
      throw new Error("Weekly duration limit reached");
    }
  }
}
```

**5. Conflict Detection**

```typescript
async function checkForConflicts({
  users,
  start,
  end,
}: ConflictCheckParams) {
  const conflicts: Conflict[] = [];

  // Check each user's calendar
  for (const user of users) {
    // Get connected calendars
    const credentials = await prisma.credential.findMany({
      where: {
        userId: user.id,
        type: { contains: "calendar" },
        invalid: false,
      },
    });

    // Fetch busy times from all calendars
    const busyTimes = await Promise.all(
      credentials.map(async (credential) => {
        const calendar = await getCalendar(credential);
        return calendar.getAvailability(start, end, []);
      })
    );

    // Check for overlaps
    const flatBusyTimes = busyTimes.flat();
    const hasConflict = flatBusyTimes.some(busy =>
      overlaps(busy.start, busy.end, start, end)
    );

    if (hasConflict) {
      conflicts.push({
        userId: user.id,
        busyTimes: flatBusyTimes,
      });
    }
  }

  return conflicts;
}

function overlaps(
  start1: string,
  end1: string,
  start2: string,
  end2: string
): boolean {
  const s1 = new Date(start1);
  const e1 = new Date(end1);
  const s2 = new Date(start2);
  const e2 = new Date(end2);

  return s1 < e2 && s2 < e1;
}
```

**Location**: `packages/features/bookings/lib/conflictChecker/checkForConflicts.ts:1`

### 3.3 Booking Creation

```typescript
async function createBooking({
  evt,
  eventType,
  reqBodyUser,
  reqBodyMetadata,
}: CreateBookingParams) {
  // Determine booking status
  const status = eventType.requiresConfirmation
    ? BookingStatus.PENDING
    : BookingStatus.ACCEPTED;

  // Create booking
  const newBooking = await prisma.booking.create({
    data: {
      uid: uuid(),
      title: evt.title,
      description: evt.additionalNotes,
      startTime: evt.startTime,
      endTime: evt.endTime,

      // Relations
      eventTypeId: eventType.id,
      userId: evt.organizer.id,

      // Attendees
      attendees: {
        create: evt.attendees.map(attendee => ({
          email: attendee.email,
          name: attendee.name,
          timeZone: attendee.timeZone,
          locale: attendee.locale || "en",
        })),
      },

      // Responses (custom fields)
      responses: evt.responses || {},

      // Metadata
      metadata: {
        ...reqBodyMetadata,
        videoCallUrl: evt.videoCallData?.url,
      },

      // Status
      status,

      // Location
      location: evt.location,

      // Timestamps
      createdAt: new Date(),
    },
    include: {
      attendees: true,
      user: true,
      eventType: true,
    },
  });

  return newBooking;
}
```

**Location**: `packages/features/bookings/lib/handleNewBooking/createBooking.ts:1`

---

## 4. EventManager & Calendar Integration

### 4.1 EventManager Overview

**Location**: `packages/features/bookings/lib/EventManager.ts:1`

The EventManager handles:
1. Creating calendar events across all connected calendars
2. Creating video meetings (Zoom, Google Meet, etc.)
3. Updating events on reschedule
4. Deleting events on cancellation

**Main Methods**:
```typescript
export default class EventManager {
  // Create calendar events + video meetings
  static async create(evt: CalendarEvent): Promise<CreateUpdateResult>;

  // Update existing events
  static async update(evt: CalendarEvent, bookingId: number): Promise<CreateUpdateResult>;

  // Delete events
  static async delete(evt: CalendarEvent, bookingId: number): Promise<void>;

  // Get all credentials for event users
  private static async getAllCredentials(evt: CalendarEvent): Promise<Credential[]>;
}
```

### 4.2 Create Flow

```typescript
static async create(evt: CalendarEvent): Promise<CreateUpdateResult> {
  // 1. Get all calendar/video credentials for organizer & team members
  const credentials = await this.getAllCredentials(evt);

  const results: CreateUpdateResult = {
    results: [],
    referencesToCreate: [],
  };

  // 2. Create video meeting (if location is video integration)
  if (isDedicatedIntegration(evt.location)) {
    const videoResult = await this.createVideoMeeting(evt, credentials);

    if (videoResult.success) {
      // Update evt with video meeting URL
      evt.videoCallData = {
        type: videoResult.type,
        id: videoResult.id,
        password: videoResult.password,
        url: videoResult.url,
      };

      results.results.push(videoResult);
      results.referencesToCreate.push({
        type: videoResult.type,
        uid: videoResult.uid,
        meetingId: videoResult.id,
        meetingPassword: videoResult.password,
        meetingUrl: videoResult.url,
        credentialId: videoResult.credentialId,
      });
    }
  }

  // 3. Create calendar events
  const calendarResults = await Promise.allSettled(
    credentials
      .filter(cred => cred.type.includes("calendar"))
      .map(async (credential) => {
        try {
          const calendar = await getCalendar(credential);

          // Get destination calendar (which calendar to create event in)
          const destinationCalendar = evt.destinationCalendar.find(
            dc => dc.credentialId === credential.id
          );

          // Create event
          const createdEvent = await calendar.createEvent({
            ...evt,
            destinationCalendar,
          });

          return {
            success: true,
            type: credential.type,
            uid: createdEvent.uid,
            credentialId: credential.id,
          };
        } catch (error) {
          logger.error("Failed to create calendar event", {
            error,
            credentialId: credential.id,
          });

          // Mark credential as invalid if auth error
          if (error.code === 401) {
            await invalidateCredential(credential.id);
          }

          return {
            success: false,
            error: error.message,
            credentialId: credential.id,
          };
        }
      })
  );

  // 4. Process results
  calendarResults.forEach((result) => {
    if (result.status === "fulfilled" && result.value.success) {
      results.results.push(result.value);
      results.referencesToCreate.push({
        type: result.value.type,
        uid: result.value.uid,
        credentialId: result.value.credentialId,
      });
    }
  });

  return results;
}
```

### 4.3 Video Meeting Creation

```typescript
private static async createVideoMeeting(
  evt: CalendarEvent,
  credentials: Credential[]
): Promise<VideoMeetingResult> {
  // Find video credential for the selected location
  const locationType = evt.location; // e.g., "integrations:zoom"

  const videoCredential = credentials.find(cred =>
    cred.type === locationType.replace("integrations:", "_video")
  );

  if (!videoCredential) {
    throw new Error(`No credential found for ${locationType}`);
  }

  // Get video adapter
  const videoAdapter = await getVideoAdapter(videoCredential);

  // Create meeting
  const meeting = await videoAdapter.createMeeting(evt);

  return {
    success: true,
    type: videoCredential.type,
    uid: meeting.id,
    id: meeting.id,
    password: meeting.password,
    url: meeting.url,
    credentialId: videoCredential.id,
  };
}
```

**Zoom Example**:
```typescript
// packages/app-store/zoomvideo/lib/VideoApiAdapter.ts
export const createMeeting = async (
  credential: Credential,
  evt: CalendarEvent
): Promise<VideoCallData> => {
  const response = await fetch("https://api.zoom.us/v2/users/me/meetings", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      Authorization: `Bearer ${credential.key.access_token}`,
    },
    body: JSON.stringify({
      topic: evt.title,
      type: 2, // Scheduled meeting
      start_time: evt.startTime,
      duration: Math.round(
        (new Date(evt.endTime).getTime() - new Date(evt.startTime).getTime()) / 60000
      ),
      timezone: evt.organizer.timeZone,
      settings: {
        host_video: true,
        participant_video: true,
        join_before_host: true,
      },
    }),
  });

  const data = await response.json();

  return {
    type: "zoom_video",
    id: data.id.toString(),
    password: data.password || "",
    url: data.join_url,
  };
};
```

### 4.4 Calendar Event Creation

```typescript
// packages/app-store/googlecalendar/lib/CalendarService.ts
async createEvent(evt: CalendarEvent): Promise<NewCalendarEventType> {
  const calendar = await this.authedCalendar();

  // Get destination calendar (default: primary)
  const calendarId = evt.destinationCalendar?.externalId || "primary";

  // Prepare attendees
  const attendees = [
    {
      email: evt.organizer.email,
      displayName: evt.organizer.name,
      responseStatus: "accepted",
      organizer: true,
    },
    ...evt.attendees.map(attendee => ({
      email: attendee.email,
      displayName: attendee.name,
      responseStatus: "accepted",
    })),
  ];

  // Create event
  const { data } = await calendar.events.insert({
    calendarId,
    conferenceDataVersion: evt.location === "integrations:google_meet" ? 1 : 0,
    requestBody: {
      summary: evt.title,
      description: getRichDescription(evt),
      location: evt.location,
      start: {
        dateTime: evt.startTime,
        timeZone: evt.organizer.timeZone,
      },
      end: {
        dateTime: evt.endTime,
        timeZone: evt.organizer.timeZone,
      },
      attendees,

      // Create Google Meet if location is Google Meet
      conferenceData: evt.location === "integrations:google_meet" ? {
        createRequest: {
          requestId: uuid(),
          conferenceSolutionKey: { type: "hangoutsMeet" },
        },
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
```

### 4.5 Store Booking References

```typescript
async function storeBookingReferences(
  bookingId: number,
  results: CreateUpdateResult
) {
  await prisma.bookingReference.createMany({
    data: results.referencesToCreate.map(ref => ({
      bookingId,
      type: ref.type,
      uid: ref.uid,
      meetingId: ref.meetingId,
      meetingPassword: ref.meetingPassword,
      meetingUrl: ref.meetingUrl,
      credentialId: ref.credentialId,
    })),
  });
}
```

---

## 5. Email & SMS Notifications

### 5.1 Email Flow

**Location**: `packages/features/bookings/lib/BookingEmailSmsHandler.ts:1`

```typescript
export async function sendScheduledEmails(
  evt: CalendarEvent,
  booking: Booking
) {
  // 1. Send confirmation email to organizer
  await sendOrganizerRequestEmail({
    calEvent: evt,
    booking,
  });

  // 2. Send confirmation email to attendees
  await Promise.all(
    evt.attendees.map(attendee =>
      sendAttendeeRequestEmail({
        calEvent: evt,
        attendee,
        booking,
      })
    )
  );

  // 3. Send to team members (if team event)
  if (evt.team?.members) {
    await Promise.all(
      evt.team.members.map(member =>
        sendTeamMemberEmail({
          calEvent: evt,
          member,
          booking,
        })
      )
    );
  }
}
```

**Email Templates**:

| Template | To | When | Variables |
|----------|-----|------|-----------|
| `organizer_request_email` | Organizer | New booking | `{EVENT_NAME}`, `{ATTENDEE}`, `{DATE}`, `{TIME}` |
| `attendee_request_email` | Attendee | New booking | `{EVENT_NAME}`, `{ORGANIZER}`, `{DATE}`, `{TIME}`, `{LOCATION}` |
| `organizer_scheduled_email` | Organizer | Booking confirmed | Same + `{CANCEL_LINK}`, `{RESCHEDULE_LINK}` |
| `attendee_scheduled_email` | Attendee | Booking confirmed | Same + `{CANCEL_LINK}`, `{RESCHEDULE_LINK}` |
| `organizer_cancelled_email` | Organizer | Booking cancelled | `{EVENT_NAME}`, `{ATTENDEE}`, `{CANCEL_REASON}` |
| `attendee_cancelled_email` | Attendee | Booking cancelled | `{EVENT_NAME}`, `{ORGANIZER}`, `{CANCEL_REASON}` |
| `organizer_rescheduled_email` | Organizer | Booking rescheduled | `{EVENT_NAME}`, `{OLD_DATE}`, `{NEW_DATE}` |
| `attendee_rescheduled_email` | Attendee | Booking rescheduled | Same |

**Email Content Example**:
```html
<!-- attendee_scheduled_email.html -->
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <title>Booking Confirmed</title>
</head>
<body>
  <h1>Your booking is confirmed</h1>

  <p>Hi {ATTENDEE},</p>

  <p>Your booking for <strong>{EVENT_NAME}</strong> with {ORGANIZER} is confirmed.</p>

  <table>
    <tr>
      <td><strong>Date:</strong></td>
      <td>{DATE}</td>
    </tr>
    <tr>
      <td><strong>Time:</strong></td>
      <td>{TIME}</td>
    </tr>
    <tr>
      <td><strong>Location:</strong></td>
      <td>{LOCATION}</td>
    </tr>
  </table>

  {VIDEO_CALL_URL && (
    <p><a href="{VIDEO_CALL_URL}">Join Video Call</a></p>
  )}

  <p><a href="{CANCEL_LINK}">Cancel</a> | <a href="{RESCHEDULE_LINK}">Reschedule</a></p>

  <p>Add to calendar: <a href="{ICS_LINK}">Download ICS</a></p>
</body>
</html>
```

### 5.2 SMS Notifications

```typescript
async function sendSMSNotification(
  booking: Booking,
  attendee: Attendee,
  type: "confirmation" | "reminder" | "cancellation"
) {
  // Only send if attendee has phone number
  if (!attendee.phone) return;

  // Get SMS provider (Twilio, etc.)
  const smsProvider = await getSMSProvider();

  const message = buildSMSMessage(booking, attendee, type);

  await smsProvider.send({
    to: attendee.phone,
    body: message,
  });
}

function buildSMSMessage(
  booking: Booking,
  attendee: Attendee,
  type: "confirmation" | "reminder" | "cancellation"
): string {
  switch (type) {
    case "confirmation":
      return `Booking confirmed: ${booking.title} on ${formatDate(booking.startTime)} at ${formatTime(booking.startTime)}. Location: ${booking.location}`;

    case "reminder":
      return `Reminder: You have ${booking.title} with ${booking.user.name} in 1 hour. Location: ${booking.location}`;

    case "cancellation":
      return `Booking cancelled: ${booking.title} on ${formatDate(booking.startTime)} has been cancelled.`;
  }
}
```

### 5.3 Calendar Invites (ICS Files)

```typescript
import ical from "ical-generator";

function generateICS(booking: Booking, attendee: Attendee): string {
  const calendar = ical({
    name: booking.title,
    prodId: "//Cal.com//Cal.com//EN",
  });

  calendar.createEvent({
    start: new Date(booking.startTime),
    end: new Date(booking.endTime),
    summary: booking.title,
    description: booking.description,
    location: booking.location,
    url: booking.metadata?.videoCallUrl,

    organizer: {
      name: booking.user.name,
      email: booking.user.email,
    },

    attendees: [
      {
        name: attendee.name,
        email: attendee.email,
        status: "ACCEPTED",
      },
    ],

    alarms: [
      {
        type: "display",
        trigger: 60 * 15, // 15 minutes before
      },
    ],
  });

  return calendar.toString();
}
```

---

## 6. Edge Cases & Variations

### 6.1 Reschedule Flow

```typescript
async function handleReschedule(req: NextApiRequest) {
  const { rescheduleUid, ...input } = req.body;

  // 1. Find original booking
  const originalBooking = await prisma.booking.findUnique({
    where: { uid: rescheduleUid },
    include: {
      attendees: true,
      references: true,
      eventType: true,
    },
  });

  if (!originalBooking) {
    throw new Error("Booking not found");
  }

  // 2. Validate new time is available
  await ensureAvailableUsers(originalBooking.eventType, input);

  // 3. Update booking
  const updatedBooking = await prisma.booking.update({
    where: { id: originalBooking.id },
    data: {
      startTime: input.start,
      endTime: input.end,
      rescheduled: true,
    },
  });

  // 4. Update calendar events
  const evt = await buildCalendarEvent({
    input,
    eventType: originalBooking.eventType,
    booking: updatedBooking,
  });

  await EventManager.update(evt, updatedBooking.id);

  // 5. Send rescheduled emails
  await sendRescheduledEmails({
    evt,
    booking: updatedBooking,
    originalBooking,
    rescheduleReason: input.rescheduleReason,
  });

  // 6. Trigger webhooks
  await triggerWebhook({
    event: "BOOKING_RESCHEDULED",
    booking: updatedBooking,
  });

  return updatedBooking;
}
```

### 6.2 Cancellation Flow

```typescript
async function handleCancellation(req: NextApiRequest) {
  const { uid, cancellationReason } = req.body;

  // 1. Find booking
  const booking = await prisma.booking.findUnique({
    where: { uid },
    include: {
      attendees: true,
      references: true,
      payment: true,
      eventType: true,
    },
  });

  // 2. Update booking status
  await prisma.booking.update({
    where: { id: booking.id },
    data: {
      status: BookingStatus.CANCELLED,
      cancellationReason,
      cancelledAt: new Date(),
    },
  });

  // 3. Delete calendar events
  const evt = await buildCalendarEvent({ booking });
  await EventManager.delete(evt, booking.id);

  // 4. Refund payment (if paid event)
  if (booking.payment?.[0]?.success) {
    await refundPayment(booking.payment[0]);
  }

  // 5. Send cancellation emails
  await sendCancelledEmails({
    evt,
    booking,
    cancellationReason,
  });

  // 6. Trigger webhooks
  await triggerWebhook({
    event: "BOOKING_CANCELLED",
    booking,
  });

  return booking;
}
```

### 6.3 Seats (Multi-Attendee Bookings)

```typescript
async function handleSeatsBooking(req: NextApiRequest) {
  const input = req.body;

  // Event type must have seatsPerTimeSlot > 1
  if (!input.eventType.seatsPerTimeSlot || input.eventType.seatsPerTimeSlot <= 1) {
    return handleNormalBooking(req);
  }

  // 1. Find existing booking for this time slot
  const existingBooking = await prisma.booking.findFirst({
    where: {
      eventTypeId: input.eventTypeId,
      startTime: input.start,
      status: BookingStatus.ACCEPTED,
    },
    include: {
      _count: {
        select: { seatsReferences: true },
      },
    },
  });

  // 2. Check if seats available
  if (existingBooking) {
    const seatsBooked = existingBooking._count.seatsReferences;
    const seatsAvailable = input.eventType.seatsPerTimeSlot - seatsBooked;

    if (seatsAvailable <= 0) {
      throw new Error("No seats available");
    }

    // Add attendee to existing booking
    await prisma.booking.update({
      where: { id: existingBooking.id },
      data: {
        attendees: {
          create: {
            email: input.responses.email,
            name: input.responses.name,
            timeZone: input.timeZone,
          },
        },
        seatsReferences: {
          create: {
            referenceUid: uuid(),
            attendeeEmail: input.responses.email,
          },
        },
      },
    });

    // Don't recreate calendar event, just update attendee list
    await updateCalendarEventAttendees(existingBooking);

    // Send confirmation to new attendee only
    await sendAttendeeConfirmation(input.responses.email, existingBooking);

    return existingBooking;
  }

  // 3. No existing booking, create new one
  return handleNormalBooking(req);
}
```

### 6.4 Recurring Events

```typescript
async function handleRecurringBooking(req: NextApiRequest) {
  const { recurringCount, ...input } = req.body;

  const bookings: Booking[] = [];

  // Create multiple bookings based on recurrence
  for (let i = 0; i < recurringCount; i++) {
    const start = addWeeks(new Date(input.start), i);
    const end = addWeeks(new Date(input.end), i);

    // Check availability for each occurrence
    const isAvailable = await checkAvailability({
      eventTypeId: input.eventTypeId,
      start,
      end,
    });

    if (!isAvailable) {
      throw new Error(`Time slot ${i + 1} is not available`);
    }

    // Create booking
    const booking = await handleNewBooking({
      ...req,
      body: {
        ...input,
        start: start.toISOString(),
        end: end.toISOString(),
        recurringEventId: bookings[0]?.recurringEventId || uuid(),
      },
    });

    bookings.push(booking);
  }

  return bookings;
}
```

### 6.5 Paid Events

```typescript
async function handlePaidEvent(req: NextApiRequest) {
  const input = req.body;

  if (!input.eventType.price || input.eventType.price <= 0) {
    return handleNormalBooking(req);
  }

  // 1. Create booking with PENDING status
  const booking = await createBooking({
    ...input,
    status: BookingStatus.PENDING,
  });

  // 2. Create payment session (Stripe Checkout)
  const paymentSession = await createPaymentSession({
    booking,
    eventType: input.eventType,
    amount: input.eventType.price,
    currency: input.eventType.currency,
  });

  // 3. Store payment record
  await prisma.payment.create({
    data: {
      bookingId: booking.id,
      amount: input.eventType.price,
      currency: input.eventType.currency,
      externalId: paymentSession.id,
      success: false,
      appId: "stripe",
    },
  });

  // 4. Return payment URL (user will be redirected)
  return {
    booking,
    paymentUrl: paymentSession.url,
  };
}

// Webhook handler (called by Stripe when payment succeeds)
async function handlePaymentSuccess(req: NextApiRequest) {
  const event = await verifyStripeWebhook(req);

  if (event.type === "checkout.session.completed") {
    const session = event.data.object;
    const bookingId = session.metadata.bookingId;

    // 1. Update payment
    await prisma.payment.updateMany({
      where: { bookingId: parseInt(bookingId), success: false },
      data: { success: true },
    });

    // 2. Confirm booking
    const booking = await prisma.booking.update({
      where: { id: parseInt(bookingId) },
      data: { status: BookingStatus.ACCEPTED },
      include: {
        attendees: true,
        eventType: true,
      },
    });

    // 3. Create calendar events
    const evt = await buildCalendarEvent({ booking });
    await EventManager.create(evt);

    // 4. Send confirmation emails
    await sendScheduledEmails(evt, booking);

    // 5. Trigger webhooks
    await triggerWebhook({ event: "BOOKING_PAID", booking });
  }
}
```

---

## 7. Error Handling & Rollback

### 7.1 Transaction Pattern

```typescript
async function handleNewBooking(req: NextApiRequest) {
  // Use Prisma transaction for atomicity
  return await prisma.$transaction(async (tx) => {
    try {
      // 1. Create booking
      const booking = await tx.booking.create({ data: bookingData });

      // 2. Create attendees
      await tx.attendee.createMany({ data: attendeesData });

      // 3. Create calendar events
      const eventResults = await EventManager.create(evt);

      // 4. Store references
      await tx.bookingReference.createMany({
        data: eventResults.referencesToCreate,
      });

      return booking;
    } catch (error) {
      // Transaction will auto-rollback on error
      logger.error("Booking creation failed", { error });
      throw error;
    }
  });
}
```

### 7.2 Cleanup on Failure

```typescript
async function cleanupFailedBooking(bookingId: number) {
  try {
    // 1. Delete booking references
    await prisma.bookingReference.deleteMany({
      where: { bookingId },
    });

    // 2. Delete calendar events (best effort)
    const references = await prisma.bookingReference.findMany({
      where: { bookingId },
    });

    await Promise.allSettled(
      references.map(ref =>
        deleteCalendarEvent(ref.uid, ref.credentialId)
      )
    );

    // 3. Delete booking
    await prisma.booking.delete({
      where: { id: bookingId },
    });
  } catch (error) {
    logger.error("Cleanup failed", { bookingId, error });
  }
}
```

---

## 8. Performance & Optimization

### 8.1 Parallel Processing

```typescript
// ✅ Good: Parallel calendar creation
const results = await Promise.allSettled(
  credentials.map(cred => createCalendarEvent(cred, evt))
);

// ❌ Bad: Sequential processing
for (const cred of credentials) {
  await createCalendarEvent(cred, evt);
}
```

### 8.2 Caching

```typescript
// Cache availability for 5 minutes
const getCachedAvailability = async (userId: number, date: string) => {
  const cacheKey = `availability:${userId}:${date}`;

  const cached = await redis.get(cacheKey);
  if (cached) return JSON.parse(cached);

  const availability = await fetchAvailability(userId, date);
  await redis.setex(cacheKey, 300, JSON.stringify(availability));

  return availability;
};
```

### 8.3 Database Optimization

```sql
-- Index for booking queries
CREATE INDEX idx_booking_user_date ON Booking(userId, startTime);
CREATE INDEX idx_booking_eventtype_date ON Booking(eventTypeId, startTime);
CREATE INDEX idx_booking_status ON Booking(status);

-- Index for availability queries
CREATE INDEX idx_availability_user ON Availability(userId);
CREATE INDEX idx_schedule_user ON Schedule(userId);
```

---

## 9. Security Considerations

### 9.1 Input Validation

```typescript
// Always validate user input
const validated = bookingSchema.parse(req.body);

// Sanitize HTML in custom fields
const sanitizedNotes = DOMPurify.sanitize(validated.responses.notes);
```

### 9.2 Authorization Checks

```typescript
// Verify user can book this event type
async function authorizeBooking(eventType: EventType, userId?: number) {
  // Public events: always allowed
  if (!eventType.requiresBookerEmailVerification) {
    return true;
  }

  // Private events: check access
  if (eventType.hashedLink && !verifyHashedLink(req.hashedLink)) {
    throw new Error("Invalid booking link");
  }

  // Team events: check team membership
  if (eventType.teamId && userId) {
    const membership = await prisma.membership.findFirst({
      where: {
        teamId: eventType.teamId,
        userId,
        accepted: true,
      },
    });

    if (!membership) {
      throw new Error("Not authorized to book this event");
    }
  }
}
```

### 9.3 Rate Limiting

```typescript
// Limit booking requests per IP
await checkRateLimitAndThrowError({
  identifier: getIP(req),
  rateLimitingType: "booking",
  // 10 bookings per hour per IP
});
```

---

## 10. Testing the Booking Flow

### 10.1 Unit Tests

```typescript
describe("handleNewBooking", () => {
  it("should create booking successfully", async () => {
    const req = mockRequest({
      body: {
        eventTypeId: 1,
        start: "2025-11-19T14:00:00Z",
        end: "2025-11-19T14:30:00Z",
        responses: {
          email: "test@example.com",
          name: "Test User",
        },
      },
    });

    const booking = await handleNewBooking(req);

    expect(booking).toBeDefined();
    expect(booking.uid).toBeDefined();
    expect(booking.status).toBe(BookingStatus.ACCEPTED);
  });

  it("should throw error if time is unavailable", async () => {
    // Mock conflict
    vi.mocked(checkForConflicts).mockResolvedValue([
      { userId: 1, busyTimes: [...] },
    ]);

    await expect(handleNewBooking(req)).rejects.toThrow(
      "Selected time is no longer available"
    );
  });
});
```

### 10.2 Integration Tests

```typescript
describe("Booking Flow E2E", () => {
  it("should complete full booking flow", async () => {
    // 1. Create event type
    const eventType = await prisma.eventType.create({
      data: { title: "Test Event", length: 30, userId: 1 },
    });

    // 2. Make booking
    const booking = await handleNewBooking({
      eventTypeId: eventType.id,
      start: "2025-11-19T14:00:00Z",
      end: "2025-11-19T14:30:00Z",
      responses: {
        email: "test@example.com",
        name: "Test User",
      },
    });

    // 3. Verify booking created
    const dbBooking = await prisma.booking.findUnique({
      where: { id: booking.id },
      include: {
        attendees: true,
        references: true,
      },
    });

    expect(dbBooking).toBeDefined();
    expect(dbBooking.attendees).toHaveLength(1);
    expect(dbBooking.references.length).toBeGreaterThan(0);

    // 4. Verify calendar event created
    const calendarRef = dbBooking.references.find(r =>
      r.type.includes("calendar")
    );
    expect(calendarRef).toBeDefined();

    // 5. Verify email sent
    expect(sendEmailMock).toHaveBeenCalledWith(
      expect.objectContaining({
        to: "test@example.com",
        subject: expect.stringContaining("Booking Confirmed"),
      })
    );
  });
});
```

---

## Summary

### Key Takeaways

1. **User Journey**: Date selection → Time slot → Booking form → Payment (if paid) → Confirmation

2. **Backend Flow**:
   - Validation (schema, email blocks, limits)
   - Conflict detection (check calendars)
   - Booking creation (database transaction)
   - Calendar/video creation (EventManager)
   - Notifications (email/SMS)
   - Webhooks & workflows

3. **EventManager**: Handles calendar event and video meeting creation across multiple integrations

4. **Edge Cases**:
   - Reschedule: Update booking + calendar events
   - Cancellation: Delete events + refund payment
   - Seats: Multiple attendees per time slot
   - Recurring: Create multiple bookings
   - Paid: Payment flow before confirmation

5. **Security**: Input validation, authorization checks, rate limiting

6. **Performance**: Parallel processing, caching, database indexes

### Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                      BOOKING FLOW                           │
└─────────────────────────────────────────────────────────────┘

Frontend (Booker.tsx)
  ↓
  1. User selects date
  ↓
  2. Fetch available slots (tRPC: viewer.public.slots)
  ↓
  3. User selects time slot
  ↓
  4. User fills booking form
  ↓
  5. Submit (tRPC: viewer.public.bookings.create)
  ↓
Backend (handleNewBooking)
  ↓
  6. Validate input (Zod schema)
  ↓
  7. Check email blocks
  ↓
  8. Ensure users available
  ↓
  9. Check booking/duration limits
  ↓
  10. Check conflicts (calendar busy times)
  ↓
  11. Create booking (Prisma transaction)
      - Booking record
      - Attendees
      - Responses (custom fields)
  ↓
  12. EventManager.create()
      - Create video meeting (if video location)
      - Create calendar events (all calendars)
      - Store BookingReferences
  ↓
  13. Send notifications
      - Email to organizer
      - Email to attendees
      - SMS (if configured)
      - Calendar invites (ICS)
  ↓
  14. Trigger webhooks
      - BOOKING_CREATED event
      - Zapier, Make, n8n, etc.
  ↓
  15. Schedule workflow reminders
      - Email reminders
      - SMS reminders
      - WhatsApp reminders
  ↓
  16. Return booking data
  ↓
Frontend
  ↓
  17. Show success page
  ↓
  18. Redirect to payment (if paid event)
```

### Next Steps

- **PHASE 8**: Authentication & Authorization
- **PHASE 9**: API v1 vs API v2
- **PHASE 10**: Platform & Atoms (Embed/SDK)

---

**Tác giả**: Claude (AI Assistant)
**Ngày tạo**: 2025-11-18
**Phiên bản**: 1.0
