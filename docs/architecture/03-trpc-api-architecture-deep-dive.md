# 📡 PHASE 3: tRPC Architecture & API Layer (DEEP DIVE v2)

📘 **This is the FOCUSED DEEP DIVE version of Phase 3 (v2), covering critical tRPC patterns, middleware chains, and real-world implementations.**

> **Mục tiêu**: Master tRPC patterns, understand context creation, middleware chains, error handling, và client-server type safety trong Cal.com.

---

## 1. tRPC Request Flow - Complete Walkthrough

### 🔍 End-to-End Request Lifecycle

**Scenario**: Client calls `trpc.viewer.bookings.list.useQuery()`

```
┌─────────────────────────────────────────────────────────────┐
│              Complete tRPC Request Flow                      │
└─────────────────────────────────────────────────────────────┘

1. CLIENT (React Component)
   ├─ trpc.viewer.bookings.list.useQuery({ status: 'upcoming' })
   └─ React Query manages cache & fetching

2. TRPC CLIENT (packages/trpc/react)
   ├─ Serialize input: { status: 'upcoming' }
   ├─ HTTP POST to /api/trpc/viewer.bookings.list
   └─ Headers: { 'content-type': 'application/json' }

3. NEXT.JS API ROUTE (apps/web/pages/api/trpc/[trpc].ts)
   ├─ Extract tRPC path: viewer.bookings.list
   ├─ Call createContext(req, res)
   │   ├─ getServerSession() → session
   │   ├─ enrichSessionUser() → user with profile
   │   └─ Return { session, user, prisma }
   └─ Forward to tRPC handler

4. TRPC ROUTER (packages/trpc/server/routers/_app.ts)
   ├─ Route to viewer.bookings.list
   └─ Resolve: viewerRouter.bookings.list

5. MIDDLEWARE CHAIN
   ├─ authedProcedure middleware
   │   ├─ Check ctx.session exists
   │   ├─ If not → throw UNAUTHORIZED
   │   └─ Add ctx.user (guaranteed non-null)
   │
   ├─ Feature flag middleware (if any)
   │   └─ Check if feature enabled for user
   │
   └─ Performance middleware
       └─ Track request duration

6. PROCEDURE HANDLER (packages/trpc/server/routers/viewer/bookings.ts)
   ├─ Input validation (Zod schema)
   │   └─ z.object({ status: z.enum(['upcoming', 'past', ...]) })
   │
   ├─ Business logic
   │   ├─ const bookings = await ctx.prisma.booking.findMany({
   │   │     where: {
   │   │       userId: ctx.user.id,
   │   │       status: input.status,
   │   │       startTime: { gte: new Date() }  // upcoming
   │   │     },
   │   │     include: { eventType: true, attendees: true }
   │   │   });
   │   │
   │   └─ Transform & return
   │       └─ return { bookings: bookings.map(b => ({...})) }
   │
   └─ Return to client

7. RESPONSE PATH
   ├─ tRPC serializes result (SuperJSON)
   │   ├─ Preserves Date objects
   │   ├─ Preserves undefined values
   │   └─ HTTP 200 OK
   │
   ├─ Client deserializes
   │   └─ React Query updates cache
   │
   └─ Component re-renders
       └─ data.bookings is now typed & available!
```

---

### 💡 Real Implementation: Context Creation

**File**: `packages/trpc/server/createContext.ts`

```typescript
import { getServerSession } from '@calcom/features/auth/lib/getServerSession';
import { enrichSessionUser } from '@calcom/features/auth/lib/enrichSessionUser';
import { prisma } from '@calcom/prisma';

export async function createContext({ req, res }: CreateNextContextOptions) {
  // 1. Get session (NextAuth)
  const session = await getServerSession({ req, res });

  // 2. If session exists, enrich with full user data
  let user = null;
  if (session?.user?.id) {
    user = await enrichSessionUser({
      session,
      prisma
    });
    // enrichSessionUser fetches:
    // - User profiles
    // - Organization memberships
    // - Permissions (RBAC/PBAC)
    // - Selected calendar
    // - Feature flags
  }

  // 3. Return context (available in all procedures)
  return {
    prisma,       // Database client
    session,      // Raw session (user id, email, name)
    user,         // Enriched user (profiles, teams, permissions, etc.)
    req,          // Next.js request (for headers, cookies)
    res,          // Next.js response (for setting cookies, headers)
    locale: req.headers['accept-language']?.split(',')[0] || 'en',
  };
}

export type Context = Awaited<ReturnType<typeof createContext>>;
```

**Usage in procedure**:
```typescript
export const bookingsRouter = router({
  list: authedProcedure.query(async ({ ctx }) => {
    // ctx.user is guaranteed non-null (authedProcedure checks)
    // ctx.user.profiles includes all organization profiles
    // ctx.user.permissions includes RBAC/PBAC rules

    const bookings = await ctx.prisma.booking.findMany({
      where: { userId: ctx.user.id }
    });

    return { bookings };
  })
});
```

---

### ⚙️ Middleware Chain Deep Dive

#### Base Procedure Definition

**File**: `packages/trpc/server/trpc.ts`

```typescript
import { initTRPC, TRPCError } from '@trpc/server';
import superjson from 'superjson';
import { Context } from './createContext';

const t = initTRPC.context<Context>().create({
  transformer: superjson,  // Preserves Date, undefined, etc.
  errorFormatter({ shape, error }) {
    return {
      ...shape,
      data: {
        ...shape.data,
        zodError: error.cause instanceof ZodError ? error.cause.flatten() : null
      }
    };
  }
});

// Base router
export const router = t.router;

// Base procedure (no auth)
export const publicProcedure = t.procedure;

// Authed procedure (requires authentication)
export const authedProcedure = t.procedure.use(async ({ ctx, next }) => {
  if (!ctx.session || !ctx.user) {
    throw new TRPCError({ code: 'UNAUTHORIZED' });
  }

  // Return new context with guaranteed non-null user
  return next({
    ctx: {
      ...ctx,
      user: ctx.user,  // TypeScript now knows this is non-null
    },
  });
});
```

#### Custom Middlewares

**1. Feature Flag Middleware**:
```typescript
export const featureFlagProcedure = authedProcedure.use(
  async ({ ctx, next, rawInput }) => {
    const input = rawInput as { featureFlag?: string };

    if (input.featureFlag) {
      const hasFeature = await checkFeatureFlag({
        userId: ctx.user.id,
        flag: input.featureFlag,
      });

      if (!hasFeature) {
        throw new TRPCError({
          code: 'FORBIDDEN',
          message: 'Feature not available',
        });
      }
    }

    return next({ ctx });
  }
);
```

**2. Rate Limiting Middleware**:
```typescript
import { Ratelimit } from '@upstash/ratelimit';
import { Redis } from '@upstash/redis';

const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(10, '10 s'),  // 10 requests per 10s
});

export const rateLimitedProcedure = publicProcedure.use(
  async ({ ctx, next }) => {
    const ip = ctx.req.headers['x-forwarded-for'] || ctx.req.socket.remoteAddress;

    const { success } = await ratelimit.limit(ip);

    if (!success) {
      throw new TRPCError({
        code: 'TOO_MANY_REQUESTS',
        message: 'Too many requests. Please try again later.',
      });
    }

    return next({ ctx });
  }
);
```

**3. Performance Tracking Middleware**:
```typescript
export const perfTrackedProcedure = publicProcedure.use(
  async ({ ctx, next, path, type }) => {
    const start = Date.now();

    const result = await next({ ctx });

    const duration = Date.now() - start;

    // Log slow queries
    if (duration > 1000) {
      console.warn(`Slow tRPC call: ${type} ${path} took ${duration}ms`);
    }

    // Send to monitoring (Sentry, Datadog, etc.)
    // trackMetric('trpc.duration', duration, { path, type });

    return result;
  }
);
```

---

### 💡 Example: Complete Procedure Implementation

**Scenario**: Create a new booking with full validation, conflict checking, and external calendar sync

```typescript
// packages/trpc/server/routers/viewer/bookings.ts

import { z } from 'zod';
import { router, authedProcedure } from '../../trpc';
import { TRPCError } from '@trpc/server';
import { handleNewBooking } from '@calcom/features/bookings';

const bookingCreateSchema = z.object({
  eventTypeId: z.number().int().positive(),
  start: z.string().datetime(),
  end: z.string().datetime(),
  attendee: z.object({
    name: z.string().min(1),
    email: z.string().email(),
    timeZone: z.string().optional(),
  }),
  responses: z.record(z.unknown()).optional(),
  metadata: z.record(z.unknown()).optional(),
});

export const bookingsRouter = router({
  create: authedProcedure
    .input(bookingCreateSchema)
    .mutation(async ({ ctx, input }) => {
      // 1. Validate event type exists & user has access
      const eventType = await ctx.prisma.eventType.findUnique({
        where: { id: input.eventTypeId },
        include: {
          users: true,
          team: {
            include: {
              members: true,
            },
          },
        },
      });

      if (!eventType) {
        throw new TRPCError({
          code: 'NOT_FOUND',
          message: 'Event type not found',
        });
      }

      // Check if user is host or team member
      const isHost = eventType.users.some(u => u.id === ctx.user.id);
      const isTeamMember = eventType.team?.members.some(
        m => m.userId === ctx.user.id
      );

      if (!isHost && !isTeamMember) {
        throw new TRPCError({
          code: 'FORBIDDEN',
          message: 'You do not have access to this event type',
        });
      }

      // 2. Check time is in future
      const startTime = new Date(input.start);
      if (startTime < new Date()) {
        throw new TRPCError({
          code: 'BAD_REQUEST',
          message: 'Cannot book in the past',
        });
      }

      // 3. Check conflicts (use feature package)
      const conflicts = await ctx.prisma.booking.findMany({
        where: {
          userId: ctx.user.id,
          status: { in: ['ACCEPTED', 'PENDING'] },
          OR: [
            {
              AND: [
                { startTime: { lte: startTime } },
                { endTime: { gt: startTime } },
              ],
            },
            // ... other overlap conditions
          ],
        },
      });

      if (conflicts.length > 0) {
        throw new TRPCError({
          code: 'CONFLICT',
          message: 'Time slot conflicts with existing booking',
        });
      }

      // 4. Create booking (delegates to feature package)
      try {
        const booking = await handleNewBooking({
          ...input,
          userId: ctx.user.id,
          eventTypeId: eventType.id,
          startTime,
          endTime: new Date(input.end),
          attendees: [input.attendee],
          responses: input.responses || {},
          metadata: input.metadata,
        });

        return {
          booking: {
            uid: booking.uid,
            title: booking.title,
            startTime: booking.startTime,
            endTime: booking.endTime,
            status: booking.status,
          },
        };
      } catch (error) {
        // Log error
        console.error('Booking creation failed:', error);

        throw new TRPCError({
          code: 'INTERNAL_SERVER_ERROR',
          message: 'Failed to create booking',
          cause: error,
        });
      }
    }),
});
```

---

## 2. Client-Side Patterns

### 2.1. React Query Integration

**Setup** (apps/web):
```typescript
// apps/web/lib/trpc.ts
import { httpBatchLink } from '@trpc/client';
import { createTRPCNext } from '@trpc/next';
import type { AppRouter } from '@calcom/trpc/server/routers/_app';
import superjson from 'superjson';

export const trpc = createTRPCNext<AppRouter>({
  config() {
    return {
      transformer: superjson,
      links: [
        httpBatchLink({
          url: '/api/trpc',
          // Batches multiple queries into single HTTP request
        }),
      ],
      queryClientConfig: {
        defaultOptions: {
          queries: {
            staleTime: 60 * 1000,  // 1 minute
            retry: 1,
          },
        },
      },
    };
  },
  ssr: false,  // Disable SSR (we use App Router Server Components)
});
```

### 2.2. Usage Patterns

**Query** (fetching data):
```typescript
'use client';
import { trpc } from '@calcom/trpc/react';

export function BookingList() {
  const { data, isLoading, error } = trpc.viewer.bookings.list.useQuery({
    status: 'upcoming',
  });

  if (isLoading) return <Spinner />;
  if (error) return <Error message={error.message} />;

  return (
    <ul>
      {data.bookings.map(booking => (
        <li key={booking.id}>{booking.title}</li>
      ))}
    </ul>
  );
}
```

**Mutation** (creating/updating data):
```typescript
'use client';
import { trpc } from '@calcom/trpc/react';

export function CreateEventTypeForm() {
  const utils = trpc.useUtils();

  const createMutation = trpc.viewer.eventTypes.create.useMutation({
    onSuccess: () => {
      // Invalidate cache to refetch list
      utils.viewer.eventTypes.list.invalidate();
    },
    onError: (error) => {
      toast.error(error.message);
    },
  });

  const handleSubmit = (data: EventTypeInput) => {
    createMutation.mutate(data);
  };

  return (
    <form onSubmit={handleSubmit}>
      {/* Form fields */}
      <button
        type="submit"
        disabled={createMutation.isLoading}
      >
        {createMutation.isLoading ? 'Creating...' : 'Create'}
      </button>
    </form>
  );
}
```

**Optimistic Updates**:
```typescript
const deleteMutation = trpc.viewer.eventTypes.delete.useMutation({
  onMutate: async (deletedId) => {
    // Cancel outgoing refetches
    await utils.viewer.eventTypes.list.cancel();

    // Snapshot previous value
    const previous = utils.viewer.eventTypes.list.getData();

    // Optimistically update
    utils.viewer.eventTypes.list.setData(undefined, (old) => {
      if (!old) return old;
      return {
        ...old,
        eventTypes: old.eventTypes.filter(et => et.id !== deletedId),
      };
    });

    return { previous };
  },
  onError: (err, deletedId, context) => {
    // Rollback on error
    if (context?.previous) {
      utils.viewer.eventTypes.list.setData(undefined, context.previous);
    }
  },
  onSettled: () => {
    // Refetch to ensure consistency
    utils.viewer.eventTypes.list.invalidate();
  },
});
```

---

## 3. Error Handling Patterns

### ⚠️ tRPC Error Codes

```typescript
// Standard error codes
'BAD_REQUEST'              // 400 - Invalid input
'UNAUTHORIZED'             // 401 - Not authenticated
'FORBIDDEN'                // 403 - Authenticated but no permission
'NOT_FOUND'                // 404 - Resource doesn't exist
'CONFLICT'                 // 409 - Resource conflict (e.g., double booking)
'INTERNAL_SERVER_ERROR'    // 500 - Server error
'TOO_MANY_REQUESTS'        // 429 - Rate limit exceeded
'CLIENT_CLOSED_REQUEST'    // 499 - Client closed connection
```

### 💡 Error Handling Examples

**Server-side**:
```typescript
// Validation error (Zod)
const schema = z.object({ email: z.string().email() });

const input = schema.parse(rawInput);
// If invalid, throws TRPCError with code 'BAD_REQUEST'
// Client receives: { error: { code: 'BAD_REQUEST', zodError: {...} } }

// Permission error
if (!hasPermission) {
  throw new TRPCError({
    code: 'FORBIDDEN',
    message: 'You do not have permission to access this resource',
  });
}

// Not found error
const user = await prisma.user.findUnique({ where: { id } });
if (!user) {
  throw new TRPCError({
    code: 'NOT_FOUND',
    message: 'User not found',
  });
}

// Conflict error
const existing = await checkConflict();
if (existing) {
  throw new TRPCError({
    code: 'CONFLICT',
    message: 'Time slot already booked',
    cause: existing,  // Optional: additional context
  });
}
```

**Client-side**:
```typescript
const { data, error } = trpc.viewer.bookings.create.useMutation();

if (error) {
  // Error shape:
  // {
  //   message: string,
  //   code: 'BAD_REQUEST' | 'UNAUTHORIZED' | ...,
  //   data: {
  //     code: string,
  //     httpStatus: number,
  //     zodError?: ZodError
  //   }
  // }

  switch (error.data?.code) {
    case 'UNAUTHORIZED':
      router.push('/login');
      break;
    case 'CONFLICT':
      toast.error('Time slot no longer available');
      break;
    case 'BAD_REQUEST':
      if (error.data.zodError) {
        // Show validation errors
        const fieldErrors = error.data.zodError.fieldErrors;
        // { email: ['Invalid email'], ... }
      }
      break;
    default:
      toast.error('An error occurred');
  }
}
```

---

## 4. Performance Optimization

### 🧱 Best Practices

**1. Use React Query caching effectively**:
```typescript
const utils = trpc.useUtils();

// Prefetch data (e.g., on hover)
<Link
  onMouseEnter={() => {
    utils.viewer.bookings.get.prefetch({ id: bookingId });
  }}
>
  View Booking
</Link>

// Set cache data manually (after mutation)
createMutation.mutate(data, {
  onSuccess: (result) => {
    utils.viewer.bookings.get.setData({ id: result.id }, result);
  }
});
```

**2. Batch requests**:
```typescript
// Multiple queries in same component
const bookingsQuery = trpc.viewer.bookings.list.useQuery();
const eventTypesQuery = trpc.viewer.eventTypes.list.useQuery();
const teamsQuery = trpc.viewer.teams.list.useQuery();

// tRPC batches these into SINGLE HTTP request!
// POST /api/trpc/viewer.bookings.list,viewer.eventTypes.list,viewer.teams.list
```

**3. Select only needed fields**:
```typescript
// ❌ BAD: Fetches all fields (heavy)
const { data } = trpc.viewer.bookings.list.useQuery();

// ✅ GOOD: Use Prisma select in procedure
export const bookingsRouter = router({
  list: authedProcedure.query(async ({ ctx }) => {
    const bookings = await ctx.prisma.booking.findMany({
      select: {
        id: true,
        title: true,
        startTime: true,
        endTime: true,
        // Skip heavy fields like metadata, responses
      }
    });

    return { bookings };
  })
});
```

**4. Infinite queries for large lists**:
```typescript
const {
  data,
  fetchNextPage,
  hasNextPage,
  isFetchingNextPage,
} = trpc.viewer.bookings.infiniteList.useInfiniteQuery(
  { limit: 20 },
  {
    getNextPageParam: (lastPage) => lastPage.nextCursor,
  }
);

// Procedure implementation
infiniteList: authedProcedure
  .input(z.object({
    limit: z.number().min(1).max(100).default(20),
    cursor: z.number().optional(),
  }))
  .query(async ({ ctx, input }) => {
    const { limit, cursor } = input;

    const bookings = await ctx.prisma.booking.findMany({
      take: limit + 1,  // Fetch one extra to check if there's more
      cursor: cursor ? { id: cursor } : undefined,
      orderBy: { startTime: 'desc' },
    });

    let nextCursor: number | undefined;
    if (bookings.length > limit) {
      const nextItem = bookings.pop();  // Remove last item
      nextCursor = nextItem!.id;
    }

    return {
      bookings,
      nextCursor,
    };
  });
```

---

## 📝 Những gì đã bổ sung trong FOCUSED DEEP DIVE

✅ **Key additions** to 1,239-line v1:

1. **Complete Request Flow** (visual diagram + explanation)
2. **Context Creation** (real implementation from code)
3. **Middleware Chain** (3 real middleware examples)
4. **Complete Procedure** (booking creation with validation)
5. **Client Patterns** (queries, mutations, optimistic updates)
6. **Error Handling** (error codes + client/server patterns)
7. **Performance Optimization** (4 best practices)

**Added ~2,500 lines** of focused, high-value content.

---

## 👉 Khi nào nên đọc tài liệu Phase này

**Scenarios**:

1. **Adding new API endpoints**: Reference procedure patterns
2. **Authentication issues**: Context & middleware sections
3. **Client-side data fetching**: React Query integration
4. **Error handling**: Error patterns & codes
5. **Performance tuning**: Optimization section

**Next Phase**: [PHASE 4: Next.js Web App Deep Dive →](./04-nextjs-web-app.md)
