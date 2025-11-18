# 📡 PHASE 3: tRPC Architecture & API Layer

> **Mục tiêu**: Hiểu sâu về tRPC routers, procedures, middlewares, context creation và client-side usage patterns trong cal.com.

---

## 📑 Mục lục

1. [Tổng quan tRPC Architecture](#1-tổng-quan-trpc-architecture)
2. [Router Hierarchy](#2-router-hierarchy)
3. [Context Creation](#3-context-creation)
4. [Procedures](#4-procedures)
5. [Middlewares](#5-middlewares)
6. [Sub-Routers Deep Dive](#6-sub-routers-deep-dive)
7. [Error Handling](#7-error-handling)
8. [Client-Side Usage](#8-client-side-usage)
9. [Best Practices](#9-best-practices)

---

## 1. Tổng quan tRPC Architecture

### 1.1. Why tRPC?

**tRPC** = TypeScript Remote Procedure Call

**Benefits**:
- ✅ **End-to-end type safety**: Types từ server → client tự động
- ✅ **No codegen**: Không cần generate code
- ✅ **Auto-completion**: IDE support đầy đủ
- ✅ **Runtime validation**: Kết hợp với Zod schemas
- ✅ **Small bundle size**: Chỉ export types, không export runtime code

---

### 1.2. Tech Stack

**tRPC Version**: `11.0.0-next-beta.222`

**Key Dependencies**:
```json
{
  "@trpc/client": "11.0.0-next-beta.222",
  "@trpc/server": "11.0.0-next-beta.222",
  "@trpc/react-query": "11.0.0-next-beta.222",
  "@trpc/next": "11.0.0-next-beta.222",
  "@tanstack/react-query": "^5.17.15",
  "superjson": "1.9.1",
  "zod": "^3.22.4"
}
```

**File Locations**:
```
packages/trpc/
├── server/
│   ├── routers/           # 40+ routers
│   ├── procedures/        # Base procedures
│   ├── middlewares/       # Auth, perf middlewares
│   ├── createContext.ts   # Context builder
│   ├── trpc.ts            # tRPC instance
│   └── errorFormatter.ts  # Error formatting
├── react/                 # React Query hooks
└── index.ts
```

---

### 1.3. Architecture Overview

```
Client (React)
    ↓ (tRPC client + React Query)
Next.js API Route (/api/trpc/[trpc].ts)
    ↓ (createContext)
tRPC Router
    ↓ (middleware chain)
Procedure (query/mutation)
    ↓ (business logic)
Prisma Client → PostgreSQL
```

**Key Concepts**:
1. **Router**: Nhóm procedures theo domain (users, bookings, teams...)
2. **Procedure**: Một endpoint API (query = GET, mutation = POST/PUT/DELETE)
3. **Middleware**: Xử lý trước/sau procedure (auth, logging, rate-limit...)
4. **Context**: Shared data giữa procedures (user, session, prisma...)

---

## 2. Router Hierarchy

### 2.1. Root Router

**File**: `packages/trpc/server/routers/_app.ts`

```typescript
import { router } from "../trpc";
import { viewerRouter } from "./viewer/_router";

export const appRouter = router({
  viewer: viewerRouter,
});

export type AppRouter = typeof appRouter;
```

**Structure**:
```
appRouter
└── viewer
    ├── apps
    ├── me
    ├── auth
    ├── bookings
    ├── calendars
    ├── eventTypes
    ├── teams
    ├── ... (40+ sub-routers)
```

**Usage**:
```typescript
// Client side
trpc.viewer.bookings.list.useQuery({ ... });
trpc.viewer.eventTypes.create.useMutation();
```

---

### 2.2. Viewer Router (Main Router)

**File**: `packages/trpc/server/routers/viewer/_router.tsx`

```typescript
export const viewerRouter = router({
  // Authentication & Session
  loggedInViewerRouter,    // SSR-safe session check
  auth: authRouter,

  // User & Profile
  me: meRouter,
  public: publicViewerRouter,

  // Core Features
  bookings: bookingsRouter,
  eventTypes: eventTypesRouter,
  eventTypesHeavy: heavyEventTypesRouter,
  availability: availabilityRouter,
  calendars: calendarsRouter,
  credentials: credentialsRouter,

  // Team & Organization
  teams: viewerTeamsRouter,
  organizations: viewerOrganizationsRouter,

  // Integrations
  apps: appsRouter,
  webhook: webhookRouter,
  workflows: workflowsRouter,

  // Enterprise
  saml: ssoRouter,
  dsync: dsyncRouter,
  pbac: permissionsRouter,

  // Admin
  admin: adminRouter,
  users: userAdminRouter,

  // Other
  slots: slotsRouter,
  payments: paymentsRouter,
  insights: insightsRouter,
  features: featureFlagRouter,
  credits: creditsRouter,
  ooo: oooRouter,
  i18n: i18nRouter,
  // ... 40+ routers total
});
```

**Total Sub-Routers**: 40+

---

### 2.3. Router Naming Conventions

**Pattern**: `{domain}Router`

Examples:
- `bookingsRouter` → `/api/trpc/viewer.bookings.*`
- `eventTypesRouter` → `/api/trpc/viewer.eventTypes.*`
- `viewerTeamsRouter` → `/api/trpc/viewer.teams.*`

**Special Routers**:
- `loggedInViewerRouter`: SSR-safe session check
- `publicViewerRouter`: Public endpoints (no auth)
- `heavyEventTypesRouter`: Heavy operations (separate to avoid timeout)

---

## 3. Context Creation

### 3.1. Context Types

**File**: `packages/trpc/server/createContext.ts`

```typescript
export type TRPCContext = {
  // Request/Response
  req: NextApiRequest | GetServerSidePropsContext["req"];
  res: NextApiResponse | GetServerSidePropsContext["res"];

  // Session & User
  session?: Session | null;
  user?: UserFromSession;

  // Database
  prisma: typeof prisma;
  insightsDb: typeof readonlyPrisma;

  // Metadata
  locale: string;
  sourceIp?: string;
  i18n?: Awaited<ReturnType<typeof serverSideTranslations>>;

  // Tracing
  traceContext: TraceContext;
};
```

---

### 3.2. Context Creation Flow

```typescript
export const createContext = async (
  { req, res }: CreateContextOptions,
  sessionGetter?: GetSessionFn
): Promise<TRPCContext> => {
  // 1. Get locale from request
  const locale = await getLocale(req);

  // 2. Get source IP
  const sourceIp = getIP(req as NextApiRequest);

  // 3. Get session (if sessionGetter provided)
  const session = sessionGetter
    ? await sessionGetter({ req, res })
    : null;

  // 4. Create inner context
  const contextInner = await createContextInner({
    locale,
    session,
    sourceIp
  });

  // 5. Return full context
  return {
    ...contextInner,
    req,
    res,
  };
};
```

**Called from**: `apps/web/pages/api/trpc/[trpc].ts`

```typescript
export default trpcNext.createNextApiHandler({
  router: appRouter,
  createContext: async ({ req, res }) => {
    return await createContext({ req, res }, getServerSession);
  },
});
```

---

### 3.3. Inner Context

```typescript
export async function createContextInner(
  opts: CreateInnerContextOptions
): Promise<InnerContext> {
  // Create distributed tracing context
  const traceContext = distributedTracing.createTrace("trpc_request", {
    meta: {
      userId: opts.session?.user?.id?.toString() || "anonymous",
    },
  });

  return {
    prisma,                    // Main Prisma client
    insightsDb: readonlyPrisma, // Read-only Prisma (for analytics)
    ...opts,
    traceContext,
  };
}
```

**Why Inner Context?**
- **Testing**: Don't need to mock `req`/`res`
- **SSR Helpers**: `createServerSideHelpers` doesn't have `req`/`res`
- **Reusability**: Can be used outside of Next.js

---

## 4. Procedures

### 4.1. Base Procedure

**File**: `packages/trpc/server/trpc.ts`

```typescript
import superjson from "superjson";
import { initTRPC } from "@trpc/server";

export const tRPCContext = initTRPC
  .context<typeof createContextInner>()
  .create({
    transformer: superjson,  // Serialize Date, Map, Set, etc.
    errorFormatter,          // Custom error formatting
  });

export const router = tRPCContext.router;
export const procedure = tRPCContext.procedure;
export const middleware = tRPCContext.middleware;
```

**SuperJSON**: Cho phép serialize/deserialize:
- `Date` objects
- `Map`, `Set`
- `undefined`
- `BigInt`

---

### 4.2. Public Procedure

**File**: `packages/trpc/server/procedures/publicProcedure.ts`

```typescript
import { procedure } from "../trpc";

// No authentication required
const publicProcedure = procedure;

export default publicProcedure;
```

**Usage**:
```typescript
export const publicRouter = router({
  health: publicProcedure.query(() => {
    return { status: "ok" };
  }),

  timezone: publicProcedure
    .input(z.object({ timezone: z.string() }))
    .query(({ input }) => {
      return { timezone: input.timezone };
    }),
});
```

---

### 4.3. Authenticated Procedure

**File**: `packages/trpc/server/procedures/authedProcedure.ts`

```typescript
import perfMiddleware from "../middlewares/perfMiddleware";
import { isAuthed } from "../middlewares/sessionMiddleware";
import { procedure } from "../trpc";

// Requires authentication
const authedProcedure = procedure
  .use(perfMiddleware)  // Performance monitoring
  .use(isAuthed);       // Session check

export default authedProcedure;
```

**Middleware Chain**:
```
procedure → perfMiddleware → isAuthed → your handler
```

**Usage**:
```typescript
export const meRouter = router({
  get: authedProcedure.query(async ({ ctx }) => {
    // ctx.user is guaranteed to exist
    const user = ctx.user;
    return user;
  }),

  update: authedProcedure
    .input(z.object({ name: z.string() }))
    .mutation(async ({ ctx, input }) => {
      return await ctx.prisma.user.update({
        where: { id: ctx.user.id },
        data: { name: input.name },
      });
    }),
});
```

---

### 4.4. Admin Procedures

**File**: `packages/trpc/server/procedures/authedProcedure.ts`

```typescript
// Instance ADMIN only
export const authedAdminProcedure = publicProcedure
  .use(isAdminMiddleware);

// Organization ADMIN only
export const authedOrgAdminProcedure = publicProcedure
  .use(isOrgAdminMiddleware);
```

**Usage**:
```typescript
export const adminRouter = router({
  // Only instance admins can call this
  getAllUsers: authedAdminProcedure.query(async ({ ctx }) => {
    return await ctx.prisma.user.findMany();
  }),
});

export const orgRouter = router({
  // Only org admins can call this
  getOrgSettings: authedOrgAdminProcedure.query(async ({ ctx }) => {
    return await ctx.prisma.organizationSettings.findUnique({
      where: { organizationId: ctx.user.organizationId },
    });
  }),
});
```

---

## 5. Middlewares

### 5.1. Session Middleware

**File**: `packages/trpc/server/middlewares/sessionMiddleware.ts`

```typescript
export const isAuthed = middleware(async ({ ctx, next }) => {
  const middlewareStart = performance.now();

  // 1. Get user from session
  const { user, session } = await getUserSession(ctx);

  const middlewareEnd = performance.now();
  logger.debug("Perf:t.isAuthed", middlewareEnd - middlewareStart);

  // 2. Check if authenticated
  if (!user || !session) {
    throw new TRPCError({ code: "UNAUTHORIZED" });
  }

  // 3. Set Sentry user
  SentrySetUser({ id: user.id });

  // 4. Pass user to next middleware/procedure
  return next({
    ctx: { user, session },
  });
});
```

**Key Functions**:

#### getUserSession
```typescript
export const getUserSession = async (ctx: TRPCContextInner) => {
  // 1. Get session from context or request
  const session = ctx.session || (await getSession(ctx));

  // 2. Get user from session
  const user = session ? await getUserFromSession(ctx, session) : null;

  // 3. Check profile authorization
  if (session?.profileId && user?.id) {
    const foundProfile = await ProfileRepository.findByUserIdAndProfileId({
      userId: user.id,
      profileId: session.profileId,
    });

    if (!foundProfile) {
      throw new TRPCError({
        code: "UNAUTHORIZED",
        message: "Profile not found or not authorized"
      });
    }
  }

  // 4. Add upId to session
  let upId = session.upId;
  if (!upId) {
    upId = foundProfile?.upId ?? `usr-${user?.id}`;
  }

  return {
    user,
    session: { ...session, upId }
  };
};
```

#### getUserFromSession
```typescript
export async function getUserFromSession(
  ctx: TRPCContextInner,
  session: Maybe<Session>
) {
  if (!session?.user?.id) return null;

  // 1. Get user from database
  const userRepo = new UserRepository(prisma);
  const userFromDb = await userRepo.findUnlockedUserForSession({
    userId: session.user.id
  });

  if (!userFromDb) return null;

  // 2. Enrich with profile data
  const user = await userRepo.enrichUserWithTheProfile({
    user: userFromDb,
    upId: session.upId,
  });

  // 3. Parse metadata
  const userMetaData = userMetadata.parse(user.metadata || {});
  const orgMetadata = teamMetadataSchema.parse(
    user.profile?.organization?.metadata || {}
  );

  // 4. Check if org admin
  const { members = [], ..._organization } = user.profile?.organization || {};
  const isOrgAdmin = members.some(
    (member) => ["OWNER", "ADMIN"].includes(member.role)
  );

  // 5. Return enriched user
  return {
    ...user,
    avatar: `${WEBAPP_URL}/${user.username}/avatar.png${
      organization.id ? `?orgId=${organization.id}` : ""
    }`,
    organization: {
      ..._organization,
      id: user.profile?.organization?.id ?? null,
      isOrgAdmin,
      metadata: orgMetadata,
    },
    organizationId: organization.id,
    locale: user.locale ?? ctx.locale,
  };
}
```

---

### 5.2. Admin Middlewares

```typescript
// Instance admin middleware
export const isAdminMiddleware = isAuthed.unstable_pipe(
  ({ ctx, next }) => {
    const { user } = ctx;
    if (user?.role !== "ADMIN") {
      throw new TRPCError({ code: "UNAUTHORIZED" });
    }
    return next({ ctx: { user } });
  }
);

// Organization admin middleware
export const isOrgAdminMiddleware = isAuthed.unstable_pipe(
  ({ ctx, next }) => {
    const { user } = ctx;
    if (!user?.organization?.isOrgAdmin) {
      throw new TRPCError({ code: "UNAUTHORIZED" });
    }
    return next({ ctx: { user } });
  }
);
```

**Pipe Pattern**:
- `unstable_pipe`: Chain middlewares
- Each middleware can extend context
- Type-safe context passed to next middleware

---

### 5.3. Performance Middleware

**File**: `packages/trpc/server/middlewares/perfMiddleware.ts`

```typescript
import { middleware } from "../trpc";

const perfMiddleware = middleware(async ({ path, next }) => {
  const start = performance.now();
  const result = await next();
  const duration = performance.now() - start;

  console.log(`[tRPC] ${path} took ${duration.toFixed(2)}ms`);

  return result;
});

export default perfMiddleware;
```

**Purpose**: Monitor procedure execution time

---

## 6. Sub-Routers Deep Dive

### 6.1. Bookings Router

**File**: `packages/trpc/server/routers/viewer/bookings/_router.tsx`

**Key Procedures**:
```typescript
export const bookingsRouter = router({
  // Queries (GET)
  list: authedProcedure
    .input(z.object({ /* ... */ }))
    .query(async ({ ctx, input }) => { /* ... */ }),

  get: authedProcedure
    .input(z.object({ bookingUid: z.string() }))
    .query(async ({ ctx, input }) => { /* ... */ }),

  // Mutations (POST/PUT/DELETE)
  create: authedProcedure
    .input(bookingCreateSchema)
    .mutation(async ({ ctx, input }) => { /* ... */ }),

  reschedule: authedProcedure
    .input(bookingRescheduleSchema)
    .mutation(async ({ ctx, input }) => { /* ... */ }),

  cancel: authedProcedure
    .input(z.object({ id: z.number(), reason: z.string().optional() }))
    .mutation(async ({ ctx, input }) => { /* ... */ }),

  confirm: authedProcedure
    .input(z.object({ bookingId: z.number() }))
    .mutation(async ({ ctx, input }) => { /* ... */ }),
});
```

---

### 6.2. EventTypes Router

**File**: `packages/trpc/server/routers/viewer/eventTypes/_router.ts`

**Structure**:
```typescript
export const eventTypesRouter = router({
  // Basic CRUD
  list: authedProcedure.query(async ({ ctx }) => { /* ... */ }),
  get: authedProcedure
    .input(z.object({ id: z.number() }))
    .query(async ({ ctx, input }) => { /* ... */ }),
  create: authedProcedure
    .input(eventTypeCreateSchema)
    .mutation(async ({ ctx, input }) => { /* ... */ }),
  update: authedProcedure
    .input(eventTypeUpdateSchema)
    .mutation(async ({ ctx, input }) => { /* ... */ }),
  delete: authedProcedure
    .input(z.object({ id: z.number() }))
    .mutation(async ({ ctx, input }) => { /* ... */ }),

  // Sub-routers
  schedule: scheduleRouter,
  hosts: hostsRouter,
  workflows: workflowsOnEventTypesRouter,
});
```

**Heavy Router** (separate to avoid timeout):
```typescript
// packages/trpc/server/routers/viewer/eventTypes/heavy/_router.ts
export const heavyEventTypesRouter = router({
  // Expensive operations
  getByViewer: authedProcedure
    .input(eventTypeByViewerSchema)
    .query(async ({ ctx, input }) => {
      // Heavy DB queries with many joins
      // Can take 1-2 seconds
    }),
});
```

---

### 6.3. Teams Router

**File**: `packages/trpc/server/routers/viewer/teams/_router.tsx`

**Nested Structure**:
```typescript
export const viewerTeamsRouter = router({
  // Team management
  list: authedProcedure.query(async ({ ctx }) => { /* ... */ }),
  get: authedProcedure
    .input(z.object({ teamId: z.number() }))
    .query(async ({ ctx, input }) => { /* ... */ }),
  create: authedProcedure
    .input(teamCreateSchema)
    .mutation(async ({ ctx, input }) => { /* ... */ }),

  // Member management
  inviteMember: authedProcedure
    .input(inviteMemberSchema)
    .mutation(async ({ ctx, input }) => { /* ... */ }),
  removeMember: authedProcedure
    .input(z.object({ teamId: z.number(), memberId: z.number() }))
    .mutation(async ({ ctx, input }) => { /* ... */ }),
  updateMembership: authedProcedure
    .input(updateMembershipSchema)
    .mutation(async ({ ctx, input }) => { /* ... */ }),

  // Nested routers
  webhooks: teamWebhooksRouter,
  workflows: teamWorkflowsRouter,
  attributes: teamAttributesRouter,
});
```

---

## 7. Error Handling

### 7.1. Error Formatter

**File**: `packages/trpc/server/errorFormatter.ts`

```typescript
export function errorFormatter({
  shape,
  error
}: ErrorFormatterOptions) {
  return {
    ...shape,
    data: {
      ...shape.data,
      zodError:
        error.code === "BAD_REQUEST" && error.cause instanceof ZodError
          ? error.cause.flatten()
          : null,
    },
  };
}
```

**Purpose**: Format Zod validation errors

---

### 7.2. Error Codes

```typescript
// TRPCError codes
const errorCodes = {
  // 400
  BAD_REQUEST: "BAD_REQUEST",
  PARSE_ERROR: "PARSE_ERROR",

  // 401
  UNAUTHORIZED: "UNAUTHORIZED",

  // 403
  FORBIDDEN: "FORBIDDEN",

  // 404
  NOT_FOUND: "NOT_FOUND",

  // 408
  TIMEOUT: "TIMEOUT",

  // 429
  TOO_MANY_REQUESTS: "TOO_MANY_REQUESTS",

  // 500
  INTERNAL_SERVER_ERROR: "INTERNAL_SERVER_ERROR",
};
```

---

### 7.3. Throwing Errors

```typescript
import { TRPCError } from "@trpc/server";

// In procedure
export const deleteBooking = authedProcedure
  .input(z.object({ id: z.number() }))
  .mutation(async ({ ctx, input }) => {
    const booking = await ctx.prisma.booking.findUnique({
      where: { id: input.id },
    });

    if (!booking) {
      throw new TRPCError({
        code: "NOT_FOUND",
        message: "Booking not found",
      });
    }

    if (booking.userId !== ctx.user.id) {
      throw new TRPCError({
        code: "FORBIDDEN",
        message: "Not authorized to delete this booking",
      });
    }

    await ctx.prisma.booking.delete({
      where: { id: input.id },
    });

    return { success: true };
  });
```

---

## 8. Client-Side Usage

### 8.1. Setup tRPC Client

**File**: `apps/web/app/_trpc/client.tsx`

```typescript
import { createTRPCReact } from "@trpc/react-query";
import type { AppRouter } from "@calcom/trpc/server/routers/_app";

export const trpc = createTRPCReact<AppRouter>();
```

**Provider Setup**:
```typescript
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { httpBatchLink } from "@trpc/client";
import superjson from "superjson";

const queryClient = new QueryClient();

const trpcClient = trpc.createClient({
  links: [
    httpBatchLink({
      url: "/api/trpc",
      transformer: superjson,
    }),
  ],
});

export function TRPCProvider({ children }) {
  return (
    <trpc.Provider client={trpcClient} queryClient={queryClient}>
      <QueryClientProvider client={queryClient}>
        {children}
      </QueryClientProvider>
    </trpc.Provider>
  );
}
```

---

### 8.2. Queries

```typescript
import { trpc } from "@/app/_trpc/client";

function BookingsList() {
  // useQuery hook
  const { data, isLoading, error } = trpc.viewer.bookings.list.useQuery({
    status: ["ACCEPTED", "PENDING"],
    take: 10,
  });

  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;

  return (
    <div>
      {data?.bookings.map((booking) => (
        <div key={booking.id}>{booking.title}</div>
      ))}
    </div>
  );
}
```

**Features**:
- Auto-refetch on window focus
- Cache management
- Retry on error
- Optimistic updates

---

### 8.3. Mutations

```typescript
function CreateEventTypeForm() {
  const utils = trpc.useUtils();

  // useMutation hook
  const createEventType = trpc.viewer.eventTypes.create.useMutation({
    onSuccess: () => {
      // Invalidate cache
      utils.viewer.eventTypes.list.invalidate();
    },
    onError: (error) => {
      toast.error(error.message);
    },
  });

  const handleSubmit = (data) => {
    createEventType.mutate({
      title: data.title,
      slug: data.slug,
      length: data.length,
    });
  };

  return (
    <form onSubmit={handleSubmit}>
      {/* ... */}
      <button disabled={createEventType.isLoading}>
        {createEventType.isLoading ? "Creating..." : "Create"}
      </button>
    </form>
  );
}
```

---

### 8.4. Optimistic Updates

```typescript
function UpdateBookingStatus() {
  const utils = trpc.useUtils();

  const confirmBooking = trpc.viewer.bookings.confirm.useMutation({
    onMutate: async (newData) => {
      // Cancel outgoing refetches
      await utils.viewer.bookings.get.cancel({
        bookingUid: newData.bookingUid
      });

      // Snapshot the previous value
      const previousBooking = utils.viewer.bookings.get.getData({
        bookingUid: newData.bookingUid,
      });

      // Optimistically update
      utils.viewer.bookings.get.setData(
        { bookingUid: newData.bookingUid },
        (old) => ({ ...old, status: "ACCEPTED" })
      );

      return { previousBooking };
    },
    onError: (err, newData, context) => {
      // Rollback on error
      utils.viewer.bookings.get.setData(
        { bookingUid: newData.bookingUid },
        context.previousBooking
      );
    },
    onSettled: () => {
      // Refetch after mutation
      utils.viewer.bookings.get.invalidate();
    },
  });

  return (
    <button onClick={() => confirmBooking.mutate({ bookingUid: "abc123" })}>
      Confirm
    </button>
  );
}
```

---

### 8.5. Infinite Queries

```typescript
function InfiniteBookingsList() {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage
  } = trpc.viewer.bookings.list.useInfiniteQuery(
    { limit: 10 },
    {
      getNextPageParam: (lastPage) => lastPage.nextCursor,
    }
  );

  return (
    <div>
      {data?.pages.map((page) =>
        page.bookings.map((booking) => (
          <div key={booking.id}>{booking.title}</div>
        ))
      )}

      {hasNextPage && (
        <button
          onClick={() => fetchNextPage()}
          disabled={isFetchingNextPage}
        >
          {isFetchingNextPage ? "Loading..." : "Load More"}
        </button>
      )}
    </div>
  );
}
```

---

### 8.6. SSR with tRPC

```typescript
import { createServerSideHelpers } from "@trpc/react-query/server";
import { appRouter } from "@calcom/trpc/server/routers/_app";
import { createContextInner } from "@calcom/trpc/server/createContext";

export async function getServerSideProps(context) {
  const helpers = createServerSideHelpers({
    router: appRouter,
    ctx: await createContextInner({
      session: await getServerSession({ req: context.req }),
      locale: context.locale,
    }),
  });

  // Prefetch
  await helpers.viewer.eventTypes.list.prefetch();

  return {
    props: {
      trpcState: helpers.dehydrate(),
    },
  };
}
```

---

## 9. Best Practices

### 9.1. Input Validation

**Always use Zod schemas**:
```typescript
import { z } from "zod";

const createEventTypeSchema = z.object({
  title: z.string().min(1).max(255),
  slug: z.string().min(1).regex(/^[a-z0-9-]+$/),
  length: z.number().min(1).max(1440),
  description: z.string().optional(),
});

export const eventTypesRouter = router({
  create: authedProcedure
    .input(createEventTypeSchema)
    .mutation(async ({ ctx, input }) => {
      // input is fully typed and validated
      return await ctx.prisma.eventType.create({
        data: input,
      });
    }),
});
```

---

### 9.2. Context Usage

**Access user safely**:
```typescript
// ❌ Bad: Might be null
const userId = ctx.user?.id;

// ✅ Good: Use authedProcedure (user guaranteed)
export const myRouter = router({
  myProcedure: authedProcedure.query(({ ctx }) => {
    const userId = ctx.user.id; // Never null
  }),
});
```

---

### 9.3. Performance

**1. Use batch requests**:
```typescript
// Client automatically batches these
const user = trpc.viewer.me.get.useQuery();
const bookings = trpc.viewer.bookings.list.useQuery();
const teams = trpc.viewer.teams.list.useQuery();

// All sent in single HTTP request
```

**2. Select only needed fields**:
```typescript
const booking = await ctx.prisma.booking.findUnique({
  where: { id: input.id },
  select: {
    id: true,
    title: true,
    startTime: true,
    // Don't fetch everything
  },
});
```

**3. Use indexes**:
```prisma
model Booking {
  @@index([userId, startTime])
}
```

---

### 9.4. Error Handling

**Provide context in errors**:
```typescript
if (!booking) {
  throw new TRPCError({
    code: "NOT_FOUND",
    message: `Booking with ID ${input.id} not found`,
    cause: new Error("Booking not found in database"),
  });
}
```

---

### 9.5. Testing

**Test procedures**:
```typescript
import { appRouter } from "@calcom/trpc/server/routers/_app";
import { createContextInner } from "@calcom/trpc/server/createContext";

describe("Bookings Router", () => {
  it("should create booking", async () => {
    const ctx = await createContextInner({
      session: mockSession,
      locale: "en",
    });

    const caller = appRouter.createCaller(ctx);

    const booking = await caller.viewer.bookings.create({
      eventTypeId: 1,
      start: new Date(),
      end: new Date(),
    });

    expect(booking).toBeDefined();
  });
});
```

---

## 📝 Thay đổi trong PHASE 3

✅ **Đã phân tích**:
- Router hierarchy: appRouter → viewer → 40+ sub-routers
- Context creation: Request → Context → User/Session enrichment
- Procedures: publicProcedure, authedProcedure, admin procedures
- Middlewares: isAuthed, isAdminMiddleware, perfMiddleware
- Error handling: TRPCError codes, error formatter
- Client-side usage: Queries, mutations, optimistic updates, SSR

**Files covered**:
- ✅ `packages/trpc/server/routers/_app.ts`
- ✅ `packages/trpc/server/routers/viewer/_router.tsx`
- ✅ `packages/trpc/server/createContext.ts`
- ✅ `packages/trpc/server/procedures/`
- ✅ `packages/trpc/server/middlewares/`
- ✅ `packages/trpc/server/trpc.ts`

**Key insights**:
1. **Type-safe end-to-end**: Types flow từ server → client tự động
2. **40+ routers**: Organized by domain (bookings, eventTypes, teams...)
3. **Middleware chain**: perfMiddleware → isAuthed → procedure
4. **Context enrichment**: Session → User → Profile → Organization
5. **React Query integration**: Auto caching, refetch, optimistic updates

---

## 👉 Phase tiếp theo

**PHASE 4: Next.js App Structure (Web Frontend)**

Sẽ đi sâu vào:
- App Router vs Pages Router hybrid
- Route groups pattern
- Middleware logic
- SSR/SSG/ISR strategy
- i18n implementation
- Booking flow pages
