# PHASE 13: Coding Conventions & Best Practices

> **Mục tiêu**: Hiểu coding standards, naming conventions, file structure patterns, TypeScript best practices, và common pitfalls trong Cal.com codebase.

## Overview

Cal.com tuân theo **strict coding conventions** để đảm bảo code consistency, maintainability, và developer experience tốt hơn.

```
Code Quality Stack:
┌────────────────────────────────────────────────────────────┐
│                Code Quality & Standards                     │
├──────────────────────┬─────────────────────────────────────┤
│                      │                                     │
│  TypeScript          │  Strict mode enabled                │
│  ESLint              │  Airbnb + Next.js rules             │
│  Prettier            │  Automatic formatting               │
│  Husky               │  Pre-commit hooks                   │
│  lint-staged         │  Staged file linting                │
│  Zod                 │  Runtime validation                 │
│                      │                                     │
└──────────────────────┴─────────────────────────────────────┘
```

---

## 1. File Naming Conventions

### 1.1 General Rules

**From CONTRIBUTING.md**:

```typescript
// Services: Use .service.ts suffix
user.service.ts
booking.service.ts
payment.service.ts

// Repositories: Use .repository.ts suffix
user.repository.ts
event-type.repository.ts

// Utilities: Use descriptive names
slugify.ts
getErrorFromUnknown.ts
safeStringify.ts

// React Components: PascalCase
BookingForm.tsx
EventTypeList.tsx
AvailabilitySettings.tsx

// Hooks: Use 'use' prefix
useBooking.ts
useEventTypes.ts
useAvailability.ts

// Types: Use .d.ts or .types.ts
booking.types.ts
api.d.ts
```

**Location**: `CONTRIBUTING.md:98-100`

### 1.2 Directory Structure Patterns

```typescript
// Feature Package Structure
packages/features/bookings/
├── components/           // React components
│   ├── BookingForm.tsx
│   └── BookingList.tsx
├── lib/                 // Business logic
│   ├── handleNewBooking.ts
│   └── getBookingInfo.ts
├── hooks/               // React hooks
│   └── useBooking.ts
├── services/            // Service layer
│   └── booking.service.ts
├── repositories/        // Data access
│   └── booking.repository.ts
└── __tests__/           // Tests
    └── booking.test.ts

// App Store Integration Structure
packages/app-store/[app-name]/
├── _metadata.ts         // App metadata (required)
├── api/                 // API routes
│   ├── callback.ts      // OAuth callback
│   └── webhook.ts       // Webhooks
├── lib/                 // Integration logic
│   └── CalendarService.ts
├── components/          // UI components
│   └── InstallAppButton.tsx
├── static/              // Static assets
│   ├── icon.svg
│   └── icon-dark.svg
└── zod.ts              // Validation schemas
```

---

## 2. TypeScript Conventions

### 2.1 Type Safety

**Strict Mode Enabled**:
```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true
  }
}
```

### 2.2 Type Patterns

**Prefer Interfaces for Objects**:
```typescript
// ✅ Good
interface User {
  id: number;
  email: string;
  name: string | null;
}

// ❌ Avoid (unless you need unions/intersections)
type User = {
  id: number;
  email: string;
  name: string | null;
};
```

**Use Type for Unions and Utilities**:
```typescript
// ✅ Good
type BookingStatus = "ACCEPTED" | "PENDING" | "CANCELLED" | "REJECTED";
type UserRole = "USER" | "ADMIN";

type PartialUser = Partial<User>;
type ReadonlyBooking = Readonly<Booking>;
```

**Avoid `any`, Use `unknown`**:
```typescript
// ❌ Bad
function processData(data: any) {
  return data.value; // No type safety
}

// ✅ Good
function processData(data: unknown) {
  if (typeof data === 'object' && data !== null && 'value' in data) {
    return (data as { value: string }).value;
  }
  throw new Error('Invalid data');
}

// ✅ Even better: Use Zod
import { z } from 'zod';

const DataSchema = z.object({
  value: z.string(),
});

function processData(data: unknown) {
  const parsed = DataSchema.parse(data);
  return parsed.value; // Type-safe!
}
```

### 2.3 Zod Validation

**Always validate external input**:

```typescript
// ✅ Good: tRPC input validation
export const eventTypesRouter = router({
  create: authedProcedure
    .input(
      z.object({
        title: z.string().min(1).max(255),
        slug: z.string().min(1).max(255),
        length: z.number().int().positive(),
        locations: z.array(z.object({
          type: z.string(),
        })),
      })
    )
    .mutation(async ({ ctx, input }) => {
      // input is now type-safe and validated
      const eventType = await ctx.prisma.eventType.create({
        data: input,
      });

      return { eventType };
    }),
});
```

**Reuse schemas**:
```typescript
// packages/features/bookings/lib/schemas.ts

export const bookingCreateSchema = z.object({
  eventTypeId: z.number().int().positive(),
  start: z.string().datetime(),
  end: z.string().datetime(),
  responses: z.record(z.unknown()),
  timeZone: z.string(),
  language: z.string().optional(),
  metadata: z.record(z.unknown()).optional(),
});

export type BookingCreateInput = z.infer<typeof bookingCreateSchema>;

// Usage in tRPC
.input(bookingCreateSchema)
.mutation(async ({ input }) => {
  // input is BookingCreateInput
});
```

### 2.4 Prisma Types

**Use Prisma-generated types**:
```typescript
import type { Prisma } from '@calcom/prisma/client';

// ✅ Good: Use Prisma types
type UserWithBookings = Prisma.UserGetPayload<{
  include: { bookings: true };
}>;

// ✅ Good: Use Prisma enums
import { BookingStatus, UserPermissionRole } from '@calcom/prisma/enums';

function isBookingConfirmed(status: BookingStatus) {
  return status === BookingStatus.ACCEPTED;
}

// ❌ Bad: Don't hardcode strings
function isBookingConfirmed(status: string) {
  return status === "ACCEPTED"; // Typo-prone
}
```

---

## 3. React & Component Patterns

### 3.1 Component Structure

```typescript
// ✅ Good component structure

import type { FC } from 'react';
import { useState } from 'react';
import { useForm } from 'react-hook-form';
import { trpc } from '@calcom/trpc/react';
import { Button } from '@calcom/ui';

interface BookingFormProps {
  eventTypeId: number;
  onSuccess?: (booking: Booking) => void;
  onError?: (error: Error) => void;
}

export const BookingForm: FC<BookingFormProps> = ({
  eventTypeId,
  onSuccess,
  onError,
}) => {
  const [isSubmitting, setIsSubmitting] = useState(false);

  const form = useForm<BookingFormData>({
    defaultValues: {
      name: '',
      email: '',
    },
  });

  const createBookingMutation = trpc.viewer.bookings.create.useMutation({
    onSuccess: (data) => {
      onSuccess?.(data.booking);
    },
    onError: (error) => {
      onError?.(error);
    },
  });

  const handleSubmit = form.handleSubmit(async (data) => {
    setIsSubmitting(true);
    try {
      await createBookingMutation.mutateAsync({
        eventTypeId,
        ...data,
      });
    } finally {
      setIsSubmitting(false);
    }
  });

  return (
    <form onSubmit={handleSubmit}>
      {/* Form fields */}
      <Button type="submit" loading={isSubmitting}>
        Book Meeting
      </Button>
    </form>
  );
};
```

### 3.2 Hooks Best Practices

**Custom hooks naming**:
```typescript
// ✅ Good: Use 'use' prefix
export function useBooking(bookingId: number) {
  const { data, isLoading } = trpc.viewer.bookings.get.useQuery({
    id: bookingId,
  });

  return { booking: data?.booking, isLoading };
}

// ❌ Bad: Missing 'use' prefix
export function getBooking(bookingId: number) {
  // This looks like a regular function, not a hook
}
```

**Extract complex logic to hooks**:
```typescript
// ✅ Good: Encapsulate logic in custom hook
export function useAvailability(eventTypeId: number, date: Date) {
  const { data: slots } = trpc.viewer.slots.getSchedule.useQuery({
    eventTypeId,
    startTime: startOfDay(date).toISOString(),
    endTime: endOfDay(date).toISOString(),
  });

  const availableSlots = slots?.filter(slot => !slot.booked) || [];
  const hasAvailability = availableSlots.length > 0;

  return {
    slots: availableSlots,
    hasAvailability,
    isLoading: !slots,
  };
}

// Usage
function BookingPage() {
  const { slots, hasAvailability } = useAvailability(eventTypeId, selectedDate);

  if (!hasAvailability) {
    return <NoSlotsAvailable />;
  }

  return <TimeSlotPicker slots={slots} />;
}
```

### 3.3 Server Components (Next.js App Router)

**Use Server Components by default**:
```typescript
// app/dashboard/page.tsx (Server Component)

import { getServerSession } from '@calcom/features/auth';
import { prisma } from '@calcom/prisma';
import { EventTypeList } from './EventTypeList'; // Client Component

export default async function DashboardPage() {
  const session = await getServerSession();

  if (!session) {
    redirect('/auth/login');
  }

  // Fetch data on server
  const eventTypes = await prisma.eventType.findMany({
    where: { userId: session.user.id },
  });

  return (
    <div>
      <h1>Dashboard</h1>
      <EventTypeList eventTypes={eventTypes} />
    </div>
  );
}
```

**Mark client components with "use client"**:
```typescript
// app/dashboard/EventTypeList.tsx

'use client';

import { useState } from 'react';
import type { EventType } from '@calcom/prisma/client';

interface Props {
  eventTypes: EventType[];
}

export function EventTypeList({ eventTypes }: Props) {
  const [filter, setFilter] = useState('');

  // Client-side interactivity
  const filtered = eventTypes.filter(et =>
    et.title.toLowerCase().includes(filter.toLowerCase())
  );

  return (
    <div>
      <input
        type="text"
        value={filter}
        onChange={(e) => setFilter(e.target.value)}
        placeholder="Filter event types..."
      />

      {filtered.map(et => (
        <EventTypeCard key={et.id} eventType={et} />
      ))}
    </div>
  );
}
```

---

## 4. tRPC Best Practices

### 4.1 Router Organization

```typescript
// ✅ Good: Organize routers by domain

// packages/trpc/server/routers/viewer/eventTypes.ts
export const eventTypesRouter = router({
  list: authedProcedure.query(async ({ ctx }) => {
    // List event types
  }),

  get: authedProcedure
    .input(z.object({ id: z.number() }))
    .query(async ({ ctx, input }) => {
      // Get single event type
    }),

  create: authedProcedure
    .input(eventTypeCreateSchema)
    .mutation(async ({ ctx, input }) => {
      // Create event type
    }),

  update: authedProcedure
    .input(eventTypeUpdateSchema)
    .mutation(async ({ ctx, input }) => {
      // Update event type
    }),

  delete: authedProcedure
    .input(z.object({ id: z.number() }))
    .mutation(async ({ ctx, input }) => {
      // Delete event type
    }),
});
```

### 4.2 Context Usage

```typescript
// ✅ Good: Use context for auth and shared services

export async function createContext({ req, res }: CreateNextContextOptions) {
  const session = await getServerSession(req, res);

  const user = session
    ? await getUserFromSession(session)
    : null;

  return {
    prisma,
    session,
    user,
    req,
    res,
  };
}

// In procedures
export const authedProcedure = publicProcedure.use(({ ctx, next }) => {
  if (!ctx.user) {
    throw new TRPCError({ code: 'UNAUTHORIZED' });
  }

  return next({
    ctx: {
      ...ctx,
      user: ctx.user, // Now guaranteed to exist
    },
  });
});
```

### 4.3 Error Handling

```typescript
// ✅ Good: Use TRPCError with proper codes

import { TRPCError } from '@trpc/server';

export const deleteEventType = authedProcedure
  .input(z.object({ id: z.number() }))
  .mutation(async ({ ctx, input }) => {
    const eventType = await ctx.prisma.eventType.findUnique({
      where: { id: input.id },
    });

    if (!eventType) {
      throw new TRPCError({
        code: 'NOT_FOUND',
        message: 'Event type not found',
      });
    }

    if (eventType.userId !== ctx.user.id) {
      throw new TRPCError({
        code: 'FORBIDDEN',
        message: 'You do not have permission to delete this event type',
      });
    }

    await ctx.prisma.eventType.delete({
      where: { id: input.id },
    });

    return { success: true };
  });
```

**Error codes**:
```typescript
- 'BAD_REQUEST': Invalid input (400)
- 'UNAUTHORIZED': Not authenticated (401)
- 'FORBIDDEN': Not authorized (403)
- 'NOT_FOUND': Resource not found (404)
- 'CONFLICT': Resource conflict (409)
- 'INTERNAL_SERVER_ERROR': Server error (500)
```

---

## 5. Naming Conventions

### 5.1 Variables and Functions

```typescript
// ✅ Good: Descriptive, camelCase

const eventTypeId = 123;
const isBookingConfirmed = true;
const hasAvailableSlots = slots.length > 0;

function createBooking(input: BookingInput) {
  // ...
}

async function sendConfirmationEmail(booking: Booking) {
  // ...
}

// ❌ Bad: Unclear, abbreviations

const et = 123;  // What is 'et'?
const confirmed = true;  // Boolean should start with is/has/can
const slots2 = [];  // Don't use numbers

function create(i) {  // Too generic, unclear param
  // ...
}
```

### 5.2 Constants

```typescript
// ✅ Good: UPPER_SNAKE_CASE for true constants

export const MAX_BOOKING_DURATION_MINUTES = 480;
export const DEFAULT_EVENT_TYPE_LENGTH = 30;
export const WEBAPP_URL = process.env.NEXT_PUBLIC_WEBAPP_URL;

// ✅ Good: camelCase for config objects

export const bookingLimits = {
  maxDuration: 480,
  minDuration: 5,
  defaultDuration: 30,
};

// ❌ Bad: Inconsistent casing

export const maxBookingDuration = 480;  // Should be UPPER_CASE
export const DEFAULT_EVENT_LENGTH = 30;  // Should be UPPER_CASE
```

### 5.3 Boolean Variables

```typescript
// ✅ Good: Use is/has/can/should prefix

const isAuthenticated = !!session;
const hasAvailability = slots.length > 0;
const canCreateBooking = user.role === 'ADMIN';
const shouldSendEmail = booking.status === 'ACCEPTED';

// ❌ Bad: Unclear boolean intent

const authenticated = !!session;  // Is this a boolean?
const availability = slots.length > 0;  // Sounds like a noun
const createBooking = user.role === 'ADMIN';  // Looks like a function
```

### 5.4 Event Handlers

```typescript
// ✅ Good: Use handle/on prefix

function handleSubmit(e: FormEvent) {
  e.preventDefault();
  // ...
}

function onDateChange(date: Date) {
  setSelectedDate(date);
}

// ❌ Bad: Unclear or inconsistent

function submit(e: FormEvent) {  // Missing 'handle' prefix
  // ...
}

function dateChanged(date: Date) {  // Use 'onDateChange' instead
  // ...
}
```

---

## 6. Code Organization

### 6.1 Import Order

```typescript
// ✅ Good: Organized imports

// 1. External packages
import { useState, useEffect } from 'react';
import { z } from 'zod';
import { trpc } from '@trpc/client';

// 2. Internal packages (@calcom/*)
import { prisma } from '@calcom/prisma';
import { Button } from '@calcom/ui';
import type { Booking } from '@calcom/prisma/client';

// 3. Relative imports
import { useBooking } from '../hooks/useBooking';
import { BookingForm } from './BookingForm';
import type { BookingFormProps } from './types';

// 4. Styles (if any)
import styles from './BookingPage.module.css';
```

### 6.2 Export Patterns

```typescript
// ✅ Good: Named exports for most things

export function createBooking(input: BookingInput) {
  // ...
}

export const BOOKING_STATUSES = ['ACCEPTED', 'PENDING'] as const;

export type BookingStatus = typeof BOOKING_STATUSES[number];

// ✅ Good: Default export for pages/components (when required)

export default function BookingPage() {
  // ...
}

// ❌ Bad: Mixing default and named exports unnecessarily

export default function createBooking() { ... }
export const createBooking = () => { ... }  // Confusing!
```

---

## 7. Error Handling

### 7.1 Try-Catch Patterns

```typescript
// ✅ Good: Handle errors with proper typing

import { getErrorFromUnknown } from '@calcom/lib/errors';

async function createBooking(input: BookingInput) {
  try {
    const booking = await prisma.booking.create({
      data: input,
    });

    return { success: true, booking };
  } catch (error) {
    const err = getErrorFromUnknown(error);

    logger.error('Failed to create booking', {
      error: err.message,
      input,
    });

    return { success: false, error: err.message };
  }
}

// ❌ Bad: Swallowing errors or using 'any'

async function createBooking(input: any) {
  try {
    // ...
  } catch (error) {
    console.log(error);  // No proper logging
    // No return value - caller doesn't know it failed!
  }
}
```

### 7.2 User-Facing Errors

```typescript
// ✅ Good: Clear, actionable error messages

throw new TRPCError({
  code: 'BAD_REQUEST',
  message: 'Please select a time slot before booking',
});

throw new TRPCError({
  code: 'CONFLICT',
  message: 'This time slot is no longer available. Please choose another time.',
});

// ❌ Bad: Technical jargon or unclear errors

throw new Error('Constraint violation');  // What constraint?
throw new Error('Invalid input');  // Which field is invalid?
```

---

## 8. Performance Best Practices

### 8.1 Database Queries

```typescript
// ✅ Good: Use select to fetch only needed fields

const user = await prisma.user.findUnique({
  where: { id: userId },
  select: {
    id: true,
    email: true,
    name: true,
    // Don't fetch heavy fields like 'metadata' unless needed
  },
});

// ✅ Good: Use include for relationships

const booking = await prisma.booking.findUnique({
  where: { uid: bookingUid },
  include: {
    eventType: true,
    attendees: true,
    user: {
      select: {
        email: true,
        name: true,
      },
    },
  },
});

// ❌ Bad: Over-fetching data

const user = await prisma.user.findUnique({
  where: { id: userId },
  // Fetches ALL fields including large JSON columns
});
```

### 8.2 React Optimizations

```typescript
// ✅ Good: Memoize expensive computations

import { useMemo } from 'react';

function EventTypeList({ eventTypes }: Props) {
  const sortedEventTypes = useMemo(() => {
    return [...eventTypes].sort((a, b) => a.title.localeCompare(b.title));
  }, [eventTypes]);

  return (
    <div>
      {sortedEventTypes.map(et => <EventTypeCard key={et.id} eventType={et} />)}
    </div>
  );
}

// ✅ Good: Use React.memo for expensive components

export const EventTypeCard = React.memo<EventTypeCardProps>(({ eventType }) => {
  return <div>{eventType.title}</div>;
});
```

---

## 9. Security Best Practices

### 9.1 Input Validation

```typescript
// ✅ Good: Always validate user input

import { z } from 'zod';

const emailSchema = z.string().email();
const urlSchema = z.string().url();

function sendEmail(email: string) {
  const validated = emailSchema.parse(email);
  // Now safe to use
}

// ❌ Bad: Trusting user input

function sendEmail(email: string) {
  // What if email is malicious?
  mailClient.send(email, 'Hello');
}
```

### 9.2 SQL Injection Prevention

```typescript
// ✅ Good: Use Prisma parameterized queries

const user = await prisma.user.findFirst({
  where: {
    email: userInput,  // Prisma handles escaping
  },
});

// ❌ Bad: Raw SQL with string concatenation

await prisma.$executeRaw`SELECT * FROM users WHERE email = ${userInput}`;
// This is vulnerable to SQL injection!

// ✅ If you must use raw SQL, use parameterized queries

await prisma.$executeRaw`SELECT * FROM users WHERE email = ${Prisma.sql`${userInput}`}`;
```

### 9.3 XSS Prevention

```typescript
// ✅ Good: React escapes by default

function UserProfile({ user }) {
  return <div>{user.name}</div>;  // Automatically escaped
}

// ❌ Bad: Using dangerouslySetInnerHTML

function UserProfile({ user }) {
  return <div dangerouslySetInnerHTML={{ __html: user.bio }} />;
  // XSS vulnerability if bio contains <script>
}

// ✅ If you need HTML, sanitize it first

import DOMPurify from 'isomorphic-dompurify';

function UserProfile({ user }) {
  const sanitized = DOMPurify.sanitize(user.bio);
  return <div dangerouslySetInnerHTML={{ __html: sanitized }} />;
}
```

---

## 10. Common Pitfalls

### 10.1 Async/Await Issues

```typescript
// ❌ Bad: Not awaiting promises

async function createMultipleBookings(inputs: BookingInput[]) {
  inputs.forEach(input => {
    createBooking(input);  // Not awaited!
  });
}

// ✅ Good: Properly await

async function createMultipleBookings(inputs: BookingInput[]) {
  await Promise.all(
    inputs.map(input => createBooking(input))
  );
}

// ✅ Good: Sequential if order matters

async function createMultipleBookings(inputs: BookingInput[]) {
  for (const input of inputs) {
    await createBooking(input);
  }
}
```

### 10.2 State Updates

```typescript
// ❌ Bad: Mutating state directly

function addBooking(booking: Booking) {
  bookings.push(booking);  // Mutation!
  setBookings(bookings);  // React won't detect change
}

// ✅ Good: Create new array

function addBooking(booking: Booking) {
  setBookings([...bookings, booking]);
}

// ✅ Good: Use functional update

function addBooking(booking: Booking) {
  setBookings(prev => [...prev, booking]);
}
```

### 10.3 useEffect Dependencies

```typescript
// ❌ Bad: Missing dependencies

useEffect(() => {
  fetchBookings(eventTypeId);  // eventTypeId not in deps!
}, []);

// ✅ Good: Include all dependencies

useEffect(() => {
  fetchBookings(eventTypeId);
}, [eventTypeId]);

// ✅ Good: Use useCallback to stabilize function references

const fetchBookings = useCallback(async (id: number) => {
  // ...
}, []);

useEffect(() => {
  fetchBookings(eventTypeId);
}, [eventTypeId, fetchBookings]);
```

---

## Summary

### Key Takeaways

1. **TypeScript**: Use strict mode, prefer `unknown` over `any`, use Zod for validation
2. **File Naming**: Use `.service.ts`, `.repository.ts`, PascalCase for components
3. **React**: Server Components by default, "use client" when needed, extract hooks
4. **tRPC**: Organize by domain, validate with Zod, use proper error codes
5. **Naming**: camelCase for variables, UPPER_CASE for constants, is/has/can for booleans
6. **Imports**: External → Internal (@calcom) → Relative → Styles
7. **Errors**: Use TRPCError, provide clear messages, log properly
8. **Performance**: Select only needed fields, memoize expensive operations
9. **Security**: Validate input, use Prisma parameterized queries, sanitize HTML
10. **Pitfalls**: Await promises, don't mutate state, include useEffect deps

### Pre-commit Checklist

```bash
# 1. Linting
yarn lint

# 2. Type checking
yarn type-check

# 3. Formatting
yarn format

# 4. Tests
yarn test

# 5. Build check
yarn build
```

### Code Review Checklist

- [ ] Type safety: No `any`, proper Zod validation
- [ ] Error handling: Proper try-catch, clear error messages
- [ ] Performance: Optimized queries, memoization where needed
- [ ] Security: Input validation, no SQL injection, no XSS
- [ ] Tests: Unit tests for logic, E2E for flows
- [ ] Documentation: Clear comments for complex logic
- [ ] Naming: Consistent with conventions

---

**Tác giả**: Claude (AI Assistant)
**Ngày tạo**: 2025-11-18
**Phiên bản**: 1.0
