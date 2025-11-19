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

#### 🧠 Ý đồ & lý do thiết kế
- Tập trung type-safety end-to-end để giảm số lượng bug runtime giữa web ↔ server, đặc biệt khi domain cal.com mở rộng liên tục (booking, team, payments…).
- Loại bỏ codegen giúp CI/CD nhanh và hạn chế “drift” giữa schema và client, điều thường xảy ra khi dùng OpenAPI/GraphQL codegen.
- SuperJSON được cấu hình từ đầu để không phải viết custom serializer cho những entity chứa `Date/Map/Set`, vốn xuất hiện rất nhiều trong lịch đặt lịch, availability và setting.
- Phân lớp rõ ràng: client → Next API handler → router → middleware → procedure → Prisma. Mục tiêu là biết chính xác logic nằm ở đâu và debug theo lớp.

#### ⚖️ Trade-offs & lựa chọn khác
- **REST Next.js API Routes**: đơn giản hơn cho dev mới, nhưng tốn công type sharing (cần openapi types hoặc tự viết), dễ bị drift giữa client/server, khó batch và khó compose middleware type-safe.
- **GraphQL**: mạnh khi cần query linh hoạt, nhưng Cal đa phần là các hành động business rõ ràng (create booking, update availability). GraphQL thêm chi phí schema, resolver boilerplate, codegen, cache complexity. tRPC cho phép giữ type inference và code nhỏ gọn.
- **Drop-in Prisma calls trong React Query hooks**: nhanh nhưng phá vỡ boundary server/client, mất bảo vệ auth ở server, khó audit/observability vì logic tản mác per-component.
- **Không dùng SuperJSON**: JSON thuần sẽ mất fidelity với `Date/BigInt`, đòi hỏi manual serialize/parse ở nhiều nơi → dễ sai lệch timezone và rounding số dư credits/payments.

#### 🧪 Tình huống maintain thực tế
- Khi thêm feature mới (ví dụ “booking labels”), dev chỉ cần thêm router/procedure và Zod schema, client tự infer type. Không phải cập nhật swagger hay file types riêng.
- Khi refactor business logic, việc có context và middleware tách biệt giúp di chuyển code (auth, tracing) mà không chạm vào tất cả procedure handlers.
- Debug production: traceContext trong context giúp gắn trace-id vào Sentry/observability, lần theo call chain.

#### 🐛 Pitfalls & lưu ý
- Quên dùng Zod → mất validation runtime, dễ đẩy crash vào client.
- Serialize thủ công `Date`/`BigInt` trong mutation result thay vì tin tưởng SuperJSON → dễ tạo double-serialization.
- Chạm trực tiếp Prisma từ client hoặc bypass tRPC → mất auth/rate-limit, gây lộ data.

#### 🔧 Gợi ý khi mở rộng trong tương lai
- Nếu domain tăng mạnh, cân nhắc thêm lớp service (pure TS) để tách procedure mỏng và dễ reuse giữa routers khác.
- Bổ sung tracing sớm cho router hot-path (bookings/eventTypes) để tránh bottleneck khi khối lượng tăng.

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

#### 🧠 Ý đồ & lý do thiết kế
- `viewerRouter` làm “root” cho 40+ router vì mọi hành động đều gắn với “viewer” (người dùng hiện tại), kể cả khi có chế độ public. Điều này giữ namespace phẳng, dễ tìm, và dễ batch nhiều query trong một request.
- Phân tách routers theo domain (bookings, eventTypes, teams, payments…) giúp mỗi router giữ phạm vi nghiệp vụ rõ ràng; new dev chỉ cần grep domain để tìm entrypoint.
- `heavyEventTypesRouter` tách riêng để cô lập các thao tác nặng (generate slots, bulk availability) tránh làm chậm các mutations nhẹ. Đây là tín hiệu kiến trúc cho thấy performance là concern đầu tiên.
- `loggedInViewerRouter` xuất hiện để phục vụ SSR: một số page cần biết viewer ngay trong server render; router phụ này đảm bảo hook SSR an toàn mà không lẫn lộn với client-only routers.

#### ⚖️ Trade-offs & lựa chọn khác
- **Một root router duy nhất không có namespace viewer**: rút gọn path nhưng khó phân biệt public/authed và khó tránh đụng tên khi có 40+ routers.
- **Tách appRouter thành nhiều root (adminRouter, publicRouter)**: rõ ràng nhưng phản tác dụng batching; client phải tạo nhiều TRPC client hoặc link, mất đơn giản.
- **Sử dụng Next.js API routes cho từng domain**: cho phép caching CDN tốt hơn, nhưng mất type inference và batching. Với ~40 routers thì số file endpoint sẽ nổ và middleware phải lặp lại.

#### 🧪 Tình huống maintain thực tế
- Thêm procedure mới vào `bookingsRouter`: thường phải chạm `packages/trpc/server/routers/viewer/bookings.ts`, định nghĩa Zod input, dùng `authedProcedure`. Có thể cần cập nhật React Query hooks tại `packages/trpc/react` hoặc app web để invalidate cache liên quan.
- Đổi logic auth (thêm role mới, ví dụ `SUPPORT_AGENT`): cần xem routers nào cần role này. Điều chỉnh middleware hoặc tạo middleware mới rồi dùng `.use` tại các router domain (teams/admin). Namespace rõ ràng giúp tìm nơi cắm middleware.
- Khi domain lớn dần, nếu một router vượt quá ~300-400 dòng, team thường split ra các file nhỏ trong thư mục router (ví dụ bookings/queries.ts, bookings/mutations.ts) nhưng vẫn export gộp để giữ API path ổn định.

#### 🐛 Pitfalls & lưu ý
- Đặt procedure mới sai namespace (ví dụ logic team lại đặt trong bookings) sẽ gây nhầm lẫn cache key phía client và khó tìm khi debug.
- Bỏ qua `heavyEventTypesRouter` và thêm thao tác nặng vào router thường → dễ timeout hoặc chậm toàn bộ batch request.
- Quy ước naming: lẫn lộn giữa `viewer` và `public` router có thể khiến endpoint public vô tình yêu cầu auth hoặc ngược lại.

#### 🔧 Gợi ý khi mở rộng trong tương lai
- Áp dụng “feature folder” cho router phình to: ví dụ `/viewer/bookings/` chứa `queries.ts`, `mutations.ts`, `validation.ts`, `index.ts` để export router. Điều này giữ path ổn nhưng tách file rõ hơn.
- Khi bắt đầu có nhiều tổ chức/tenant, cân nhắc namespace bổ sung (ví dụ `viewer.org`) hoặc middleware kiểm tra orgId thống nhất để giảm lặp code kiểm tra org trên từng procedure.

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

#### 🧠 Ý đồ & lý do thiết kế
- `createContext` nhận `req/res` để tích hợp Next API handler, còn `createContextInner` tách phần thuần data (session, locale, prisma) nhằm tái sử dụng cho SSR (không có req/res) và unit test.
- Đặt Prisma và `insightsDb` trong context để mọi procedure dùng cùng instance/connection pool, tránh tạo mới mỗi request. `insightsDb` read-only giúp tách workload analytics khỏi traffic chính.
- `traceContext` sinh ra sớm trong context để middleware/procedure có thể log cùng traceId, giảm chi phí debug khi batch nhiều call.
- `sourceIp` và `locale` được tính sớm để middleware (logging, rate-limit) có đủ thông tin mà không đụng vào handler.

#### ⚖️ Trade-offs & lựa chọn khác
- **Context tối giản chỉ có user**: đỡ nặng nhưng thiếu thông tin cho tracing/perf và khó mở rộng khi thêm features như insights/stats.
- **Khởi tạo Prisma mới per request**: tránh shared pool nhưng gây overhead kết nối, nguy cơ connection storm. Cal chọn singleton và rely vào connection pooling của Prisma.
- **Không tách inner context**: code đơn giản hơn nhưng test/SSR phải mock `req/res`, tăng boilerplate.

#### 🧪 Tình huống maintain thực tế
- Thêm trường mới vào `TRPCContext` (ví dụ `featureFlags`): thêm vào `createContextInner`, rồi merge trong `createContext`. Nếu field cần `req`, thêm ở `createContext` và spread vào return.
- Thay đổi cách lấy session (ví dụ migrate sang khác provider): cập nhật `sessionGetter` trong API handler và logic `getUserSession`, không phải chạm từng procedure.
- Khi thêm DB read replica mới cho một domain (analytics), có thể thêm client mới vào context và chỉ router liên quan dùng client đó.

#### 🐛 Pitfalls & lưu ý
- Luôn ensure middleware không mutate trực tiếp `ctx.prisma` hoặc shared objects; nếu cần override, trả về `next({ ctx: { prisma: ... } })` để giữ kiểu.
- `createContext` chạy per request; tránh logic nặng (ví dụ fetch external) ở đây, đưa vào middleware/procedure nếu cần cache.
- Quên pass `locale` hoặc `session` khi dùng `createContextInner` trong tests/SSR sẽ dẫn tới fallback sai locale hoặc lỗi auth ngầm.

#### 🔧 Gợi ý khi mở rộng trong tương lai
- Chuẩn hóa schema context trong `.agents/` hoặc shared types để new dev nắm nhanh biến có sẵn.
- Có thể thêm `requestId`/`sessionFingerprint` ở context để phục vụ rate-limit hoặc audit, dùng chung cho middlewares.

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

#### 🧠 Ý đồ & lý do thiết kế
- `publicProcedure` giữ tối giản để endpoints public không bị dính middleware không cần thiết; cũng là baseline cho pipeline `.use(...)`.
- `authedProcedure` luôn kèm `perfMiddleware → isAuthed` nhằm chuẩn hóa logging/perf cho mọi call có auth. Cấu hình ở một nơi, tránh quên log/perf ở từng handler.
- `authedAdminProcedure`/`authedOrgAdminProcedure` dựng từ `publicProcedure.use(...)` thay vì copy/paste logic vào handler để đảm bảo kiểu context sau middleware được thu hẹp chính xác (`user` luôn tồn tại, role đã check).
- Pipeline middleware cho phép thêm bước mới (rate-limit, feature flags) mà không chỉnh vào handler, giảm nguy cơ lặp error handling.

#### ⚖️ Trade-offs & lựa chọn khác
- **Nhúng auth check vào handler**: linh hoạt hơn với logic tùy biến, nhưng dễ quên check hoặc trả sai code. Middleware đảm bảo tính thống nhất.
- **Một loại authed duy nhất (không tách admin/org)**: đơn giản hơn, nhưng buộc check role trong từng handler → noisy và dễ sai phạm quyền.
- **Không dùng SuperJSON**: sẽ phải manually map `Date`/`BigInt` trong procedure output, gây duplication.

#### 🧪 Tình huống maintain thực tế
- Khi thêm procedure booking mới cần auth: start từ `authedProcedure` để có `ctx.user`. Nếu logic chỉ dành cho org admin, chuyển sang `authedOrgAdminProcedure`.
- Khi thay đổi logic perf (ví dụ thêm tracing span), chỉ cần chỉnh `perfMiddleware`; mọi `authedProcedure` hưởng lợi mà không phải tìm tay 40+ routers.
- Nếu thêm role mới (e.g., `SUPPORT_AGENT`) cần quyền đọc booking: tạo middleware mới (ví dụ `isSupportAgent`) và áp dụng `.use` trong router/phương thức phù hợp.

#### 🐛 Pitfalls & lưu ý
- Dùng nhầm `publicProcedure` cho endpoint lộ dữ liệu nhạy cảm → phải review path `viewer.*` để đảm bảo auth. Quy tắc: bất kỳ endpoint đụng user data phải dùng `authedProcedure` ít nhất.
- Quên define `.input(z.object(...))` sẽ khiến client nhận type `any`, mất type safety.
- Return dữ liệu Prisma thô có thể chứa field không nên expose (token, secret). Nên `select` hoặc map kết quả.

#### 🔧 Gợi ý khi mở rộng trong tương lai
- Tạo helpers `protectedMutation`/`protectedQuery` (wrapper authedProcedure) để đặt mặc định behavior caching hoặc audit trail cho các domain nhạy cảm (payments, admin).
- Xem xét tách layer service để procedure mỏng và dễ test: `bookingService.confirm(...)` gọi trong handler, service testable độc lập.

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

#### 🧠 Ý đồ & lý do thiết kế
- Middleware gom auth/perf vào pipeline để tránh lặp trong handler và đảm bảo mọi endpoint quan trọng đều được log/tracked.
- `unstable_pipe` cho phép xây chuỗi linh hoạt: `perfMiddleware → isAuthed → isAdminMiddleware`. Type inference cập nhật context theo từng bước, giảm lỗi “ctx.user undefined”.
- `isAuthed` enrich session/user sớm, đồng thời set Sentry user để việc theo dõi lỗi chính xác theo user.

#### ⚖️ Trade-offs & lựa chọn khác
- **Middleware global** (áp dụng ở root) thay vì gắn vào procedure: gọn hơn nhưng khó phân biệt public vs authed, và overhead cho endpoints public.
- **Logging trong handler**: linh hoạt nhưng dễ bị bỏ sót. Middleware đảm bảo coverage 100%.
- **PerfMiddleware console.log**: nhanh để debug, nhưng cho production nên gửi sang logger tập trung; Cal chọn middleware riêng để có thể swap implementation mà không đổi handler.

#### 🧪 Tình huống maintain thực tế
- Thêm rate-limit: tạo middleware mới (ví dụ `rateLimitMiddleware`) và chèn vào pipeline cần thiết (`authedProcedure.use(rateLimitMiddleware)` hoặc chỉ một số router nặng như bookings).
- Đổi rule admin: chỉnh `isAdminMiddleware` (hoặc tạo middleware khác) thay vì sửa từng handler admin.
- Khi muốn bổ sung feature flags: middleware có thể đọc `ctx.session`, `ctx.user`, thêm `ctx.flags` rồi pass xuống handler, không đổi signature procedure.

#### 🐛 Pitfalls & lưu ý
- Quên `return next({ ctx: ... })` khi cần override context → context không cập nhật, gây bug type và runtime.
- Middleware async nên bọc try/catch nếu có side effect; nếu throw, error formatter sẽ xử lý nhưng cần đảm bảo không để leak thông tin nhạy cảm.
- Đừng mutate `ctx` trực tiếp mà hãy trả object mới để tRPC merge; tránh race condition khi context shared trong test.

#### 🔧 Gợi ý khi mở rộng trong tương lai
- Chuẩn hóa logger thay vì `console.log` trong perfMiddleware; gửi traceId, userId, path, duration để dễ truy vấn.
- Thêm middleware “schema guard” cho headers (sử dụng Zod) nếu cần enforce request metadata (ví dụ version, client-id) đồng nhất.

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


#### 🧠 Ý đồ & lý do thiết kế
- **Bookings**: router trung tâm vì toàn bộ business xoay quanh đặt lịch. Chia query/mutation rõ ràng giúp React Query caching chính xác. Schema input phức tạp (time zone, recurring) đặt trong `booking*Schema` để tái dùng giữa nhiều procedure.
- **EventTypes**: tách router “heavy” để cô lập các truy vấn tốn kém (nhiều join). Mục tiêu: bảo vệ UX khỏi timeout và cho phép tối ưu riêng (caching, background preparation).
- **Teams**: router lồng `teamWebhooksRouter`, `teamWorkflowsRouter` để gom toàn bộ lifecycle theo team. Điều này phản ánh domain: mỗi team có webhook/workflow riêng, tránh nhầm lẫn với webhook toàn hệ thống.

#### ⚖️ Trade-offs & lựa chọn khác
- **Ghép bookings + eventTypes chung router**: ít file hơn nhưng domain lẫn lộn; cache keys khó tách; review khó vì logic dài.
- **Không tách heavy router**: code đơn giản hơn nhưng batch request dễ bị giữ lâu bởi một query chậm; user sẽ thấy toàn bộ batch delay.
- **Dùng service layer tách khỏi tRPC**: sạch hơn về kiến trúc, nhưng thêm abstraction. Hiện tại Cal ưu tiên tốc độ phát triển, nhưng vẫn giữ schema và helper để không “đổ logic” vào handler.

#### 🧪 Tình huống maintain thực tế
- Thêm procedure booking: cập nhật schema (Zod), handler, sau đó xem lại client invalidation (`viewer.bookings.*`) và email/notification trigger nếu có. Thường cần động vào Prisma `Booking` model khi thêm field.
- Thêm field mới vào Prisma `Booking` và expose: 
  1) Update Prisma schema + migration. 
  2) Update Zod schema và select trong `bookingsRouter` (tránh `select: {}` toàn bộ). 
  3) Update React components sử dụng, invalidate query liên quan.
- Chuyển logic permission team: thay đổi `teamWorkflowsRouter`/`teamWebhooksRouter` để check role (OWNER/ADMIN). Có thể thêm middleware riêng cho team để không lặp role check.
- Khi refactor event type scheduling logic, kiểm tra cả router thường và router heavy, vì client có thể gọi bất kỳ cái nào tùy view.

#### 🐛 Pitfalls & lưu ý
- Đừng trả về toàn bộ record Prisma mặc định, đặc biệt với bookings/eventTypes chứa metadata/secret. Luôn dùng `select`/`pick`.
- Event type “heavy” có thể chạy lâu; tránh gọi nó trong batch chung với mutation quan trọng để không tăng latency toàn batch.
- Team routers thường yêu cầu org/team context; nếu không dùng middleware để populate org info, handler dễ truy cập nhầm team khác.

#### 🔧 Gợi ý khi mở rộng trong tương lai
- Chuẩn hóa naming schema/hàm trong từng router (`list`, `get`, `create`, `update`, `delete`) để new dev đoán được API.
- Tách `bookingsRouter` thành file nhỏ (queries/mutations/notifications) khi code vượt quá giới hạn đọc. Vẫn export router gộp để giữ path ổn định.
- Xem xét layer service dùng chung giữa bookings và eventTypes (ví dụ tính availability) để tránh duplication khi thêm loại lịch mới.

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


#### 🧠 Ý đồ & lý do thiết kế
- Error formatter chuẩn hóa Zod errors để client hiển thị field errors dễ dàng (flatten). Điều này giảm custom parsing ở frontend.
- Sử dụng `TRPCError` với code chuẩn nhằm map chính xác sang HTTP tương đương và UI state (401 → redirect login, 403 → thông báo permission).
- Đưa message ngắn gọn, không leak thông tin nhạy cảm (IDs, queries) để an toàn cho logs client.

#### ⚖️ Trade-offs & lựa chọn khác
- **Ném Error thường**: đơn giản nhưng client không biết status code, khó phân biệt validation vs auth vs internal.
- **Formatter phức tạp hơn (include stack)**: hữu ích cho dev, nhưng dễ lộ thông tin trong production; Cal giữ formatter mỏng, rely Sentry để xem stack.
- **Dùng HTTP exceptions (Next API)**: bỏ qua layer tRPC, mất lợi thế type-safety và batching.

#### 🧪 Tình huống maintain thực tế
- Khi thêm validation mới bằng Zod, kiểm tra UI đã đọc `zodError` chưa; nếu UI dùng toast generic, cân nhắc map lỗi field để form hiển thị.
- Khi đổi thông điệp lỗi (ví dụ policy mới), sửa trong handler nhưng vẫn giữ code chuẩn; tránh đổi code tùy ý vì client dựa vào code để xác định hành động.
- Nếu thêm rate-limit middleware, cân nhắc throw `TOO_MANY_REQUESTS` để UI có thể hiển thị “thử lại sau” thay vì generic error.

#### 🐛 Pitfalls & lưu ý
- Không nên return boolean để biểu đạt lỗi; luôn throw `TRPCError` để formatter và React Query error flow hoạt động chuẩn.
- Đừng embed error raw từ Prisma/HTTP third-party vào message gửi client; log nội bộ bằng logger/trace thay vì expose.
- Lỗi auth/permission phải dùng `UNAUTHORIZED/ FORBIDDEN` rõ ràng; nếu dùng `BAD_REQUEST`, UI sẽ hiểu sai.

#### 🔧 Gợi ý khi mở rộng trong tương lai
- Chuẩn hóa helper `assertOrgAdmin`/`assertOwnsBooking` để throw lỗi consistent, giảm duplication.
- Tùy route quan trọng (payments), có thể thêm error wrapper gắn mã lỗi nội bộ (`errorCode`) để hỗ trợ hỗ trợ khách hàng.

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

#### 🧠 Ý đồ & lý do thiết kế
- Gắn chặt tRPC với React Query để dùng chung cache, retry, background refetch. Với 40+ routers, cache key ổn định (`viewer.*`) giúp giảm call trùng và hỗ trợ optimistic update.
- `httpBatchLink` giảm số request HTTP khi có nhiều query song song (trang dashboard tải bookings, eventTypes, teams cùng lúc).
- SuperJSON cũng được dùng phía client để deserialize `Date`/`BigInt`, tránh thủ công `new Date(...)` trong component.
- SSR dùng `createServerSideHelpers` + `createContextInner` để prefetch data trong Next.js App Router/Pages Router mà không cần `req/res`, đồng thời giữ type-safety.

#### ⚖️ Trade-offs & lựa chọn khác
- **SWR/Apollo**: SWR nhẹ hơn nhưng cần adapter cho tRPC; Apollo dành cho GraphQL, không phù hợp. React Query phù hợp nhất cho pattern query/mutation hiện tại.
- **Không batch**: giảm complexity nhưng tốn kết nối; với dashboards nhiều widget, batch tiết kiệm latency đáng kể.
- **useSWR direct fetch** (bỏ tRPC hooks): mất type inference và cache key thống nhất, dễ đụng nhau.

#### 🧪 Tình huống maintain thực tế
- Thêm mutation mới: sau khi tạo procedure, dùng `trpc.viewer.x.useMutation` và nhớ invalidation (`utils.viewer.x.list.invalidate()`). Nếu data ảnh hưởng nhiều query, tạo helper invalidate chung.
- Khi sửa input schema, IDE sẽ báo lỗi ở client; cập nhật hook call. Kiểm tra đặc biệt với infinite queries (`useInfiniteQuery`) về `getNextPageParam`.
- Migrate component từ pages → app router: vẫn dùng provider `TRPCProvider` ở root layout; SSR prefetch đổi sang `fetchRequestHandler` hoặc helper tương ứng.

#### 🐛 Pitfalls & lưu ý
- Dùng nhầm `useQuery` cho mutation (hoặc ngược lại) sẽ gây side effects không mong muốn và cache mismatch.
- Không invalidate cache sau mutation → UI hiển thị dữ liệu cũ (đặc biệt bookings/eventTypes list). Dùng `utils` để invalidate đúng scope.
- Khi dùng optimistic update, nhớ rollback trong `onError`; nếu không, cache sẽ sai và cần hard reload.
- Infinite query: đảm bảo server trả `nextCursor`; nếu không, client sẽ fetch vô hạn hoặc stop sai.

#### 🔧 Gợi ý khi mở rộng trong tương lai
- Chuẩn hóa `useBookingListOptions` (hook wrappers) để tái dùng options `staleTime`, `select`, `enabled`, giảm lặp cấu hình React Query.
- Thiết lập logger cho tRPC client để theo dõi batch và errors trong dev; hữu ích cho onboarding dev mới.

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


#### 🧠 Ý đồ & lý do thiết kế
- Best practices nhấn vào validation, context safe, performance, error handling, testing để new dev có checklist khi thêm/đổi procedure.
- Cấu trúc ưu tiên “fail fast” (Zod) và “least privilege” (authedProcedure, select fields) để giảm nguy cơ lộ dữ liệu.
- Testing qua `createCaller` cho phép chạy procedure như client thực sự nhưng không cần HTTP, nhanh và type-safe.

#### ⚖️ Trade-offs & lựa chọn khác
- **Bỏ validation**: code ngắn hơn nhưng bug dữ liệu (slug sai, timezone sai) sẽ len lỏi vào DB và UI.
- **Select * trong Prisma**: tiện nhưng tăng payload, có thể expose secret. Cal chọn select tối thiểu, đặc biệt với bookings/eventTypes chứa metadata.
- **Không viết test tRPC**: tiết kiệm thời gian ngắn hạn, nhưng khi refactor dễ phá flow booking; `createCaller` là compromise tốt giữa tốc độ và độ tin cậy.

#### 🧪 Tình huống maintain thực tế
- Khi thêm hook mới, kiểm tra xem nên `enabled` condition hay không (ví dụ chỉ fetch khi có `id`). Tránh query chạy vô nghĩa.
- Khi tối ưu performance, đầu tiên audit select fields và index; sau đó mới nghĩ đến caching hoặc splitting router heavy.
- Khi chạy test, nếu cần user khác roles, tạo session mock tương ứng để đảm bảo middleware role hoạt động.

#### 🐛 Pitfalls & lưu ý
- Không invalidate cache sau mutation hoặc invalidate sai scope (`viewer.bookings.list` vs `viewer.bookings.get`) dẫn đến UI stale.
- Dùng `ctx.user?.id` trong authedProcedure sẽ làm type `number | undefined`, mất lợi thế type narrowing; dựa vào context đã được middleware gán.
- Test bỏ qua middleware (tạo context với user null) khiến kết quả test khác thực tế; luôn tạo context giống production.

#### 🔧 Gợi ý khi mở rộng trong tương lai
- Tạo template snippet (VSCode) cho procedure mới bao gồm Zod schema, input typing, select fields, error handling chuẩn.
- Thêm guideline invalidation cho từng domain (bookings, eventTypes, teams) để dev mới biết phải invalidate những query nào khi mutation thành công.

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
