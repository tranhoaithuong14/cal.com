# PHASE 9: API v1 vs API v2

> **Mục tiêu**: Hiểu sự khác biệt giữa tRPC API (v1) và NestJS Platform API (v2), use cases, và migration path.

## Overview

Cal.com has **two distinct API layers**:

1. **API v1 (tRPC)**: Internal API for web application
2. **API v2 (Platform API)**: Public REST API for third-party integrations

```
┌─────────────────────────────────────────────────────────┐
│                      Cal.com APIs                        │
├───────────────────────────┬─────────────────────────────┤
│      API v1 (tRPC)        │    API v2 (Platform API)    │
│                           │                             │
│  Technology: tRPC         │  Technology: NestJS + REST  │
│  Purpose: Internal        │  Purpose: Public/External   │
│  Auth: NextAuth session   │  Auth: OAuth 2.0            │
│  Consumer: apps/web       │  Consumer: External apps    │
│  Type-safety: Full        │  Type-safety: OpenAPI spec  │
│  Location: @calcom/trpc   │  Location: apps/api/v2      │
└───────────────────────────┴─────────────────────────────┘
```

---

## API v1: tRPC (Internal API)

### Architecture

**Already documented in PHASE 3**. Key points:

- **End-to-end type safety** with tRPC
- **40+ routers** organized by feature
- **Procedures**: `query` (GET) and `mutation` (POST/PUT/DELETE)
- **Middlewares**: `isAuthed`, `isAdmin`, performance tracking
- **Client-side**: `trpc.viewer.eventTypes.list.useQuery()`

**Entry Point**: `/api/trpc/[router]/[trpc]`

**Example**:
```
POST /api/trpc/viewer.eventTypes.create
{
  "json": {
    "title": "30 Min Meeting",
    "length": 30
  }
}
```

**Use Cases**:
- Cal.com web application (apps/web)
- Admin panel
- Internal features

---

## API v2: Platform API (Public REST API)

### Architecture

**Technology Stack**:
- **NestJS**: TypeScript framework for building scalable server apps
- **REST**: Standard REST API with JSON responses
- **OpenAPI/Swagger**: Auto-generated API documentation
- **OAuth 2.0**: Standard authentication for third-party apps

**Location**: `apps/api/v2/`

**Structure**:
```
apps/api/v2/
├── src/
│   ├── main.ts                    # Application bootstrap
│   ├── app.module.ts              # Root module
│   │
│   ├── modules/
│   │   ├── auth/                  # OAuth 2.0
│   │   ├── bookings/              # Bookings API
│   │   ├── event-types/           # Event types API
│   │   ├── schedules/             # Schedules API
│   │   ├── calendars/             # Calendar connections
│   │   ├── teams/                 # Team management
│   │   ├── users/                 # User management
│   │   └── webhooks/              # Webhook management
│   │
│   ├── filters/                   # Exception filters
│   ├── guards/                    # Auth guards
│   ├── interceptors/              # Request/response interceptors
│   ├── pipes/                     # Validation pipes
│   └── swagger/                   # API documentation
│
├── test/                          # E2E tests
└── package.json
```

### Key Modules

**Bookings Module**:
```typescript
// apps/api/v2/src/modules/bookings/bookings.controller.ts
@Controller("bookings")
@UseGuards(JwtAuthGuard)
export class BookingsController {
  @Get()
  @ApiOperation({ summary: "List bookings" })
  async listBookings(
    @Query() query: ListBookingsDto,
    @Req() req: Request
  ) {
    return this.bookingsService.findAll(req.user.id, query);
  }

  @Post()
  @ApiOperation({ summary: "Create booking" })
  async createBooking(
    @Body() dto: CreateBookingDto,
    @Req() req: Request
  ) {
    return this.bookingsService.create(req.user.id, dto);
  }

  @Get(":id")
  @ApiOperation({ summary: "Get booking by ID" })
  async getBooking(@Param("id") id: string) {
    return this.bookingsService.findOne(id);
  }

  @Patch(":id")
  @ApiOperation({ summary: "Update booking" })
  async updateBooking(
    @Param("id") id: string,
    @Body() dto: UpdateBookingDto
  ) {
    return this.bookingsService.update(id, dto);
  }

  @Delete(":id")
  @ApiOperation({ summary: "Cancel booking" })
  async cancelBooking(@Param("id") id: string) {
    return this.bookingsService.cancel(id);
  }
}
```

### OAuth 2.0 Flow

```
1. Register OAuth App
   ↓
2. Redirect user to authorize URL
   GET /oauth/authorize?client_id=xxx&redirect_uri=yyy&scope=bookings:read
   ↓
3. User authorizes
   ↓
4. Redirect back with code
   https://yourapp.com/callback?code=abc123
   ↓
5. Exchange code for access token
   POST /oauth/token
   {
     "grant_type": "authorization_code",
     "code": "abc123",
     "client_id": "xxx",
     "client_secret": "yyy"
   }
   ↓
6. Response:
   {
     "access_token": "eyJ...",
     "refresh_token": "xyz...",
     "expires_in": 3600
   }
   ↓
7. Use access token
   GET /v2/bookings
   Authorization: Bearer eyJ...
```

### API Documentation (Swagger)

**Auto-generated from NestJS decorators**:

```typescript
@ApiTags("bookings")
@ApiSecurity("oauth2")
@Controller("bookings")
export class BookingsController {
  @Get()
  @ApiOperation({
    summary: "List bookings",
    description: "Retrieve all bookings for the authenticated user",
  })
  @ApiQuery({
    name: "status",
    enum: BookingStatus,
    required: false,
  })
  @ApiResponse({
    status: 200,
    description: "List of bookings",
    type: [BookingDto],
  })
  async listBookings(...) { ... }
}
```

**Swagger UI**: `https://api.cal.com/v2/docs`

---

## Comparison

| Feature | API v1 (tRPC) | API v2 (Platform API) |
|---------|---------------|------------------------|
| **Technology** | tRPC | NestJS + REST |
| **Type Safety** | Full (TypeScript) | OpenAPI spec |
| **Auth** | NextAuth session | OAuth 2.0 |
| **Consumers** | Internal (apps/web) | External (third-party) |
| **Documentation** | TypeScript types | Swagger/OpenAPI |
| **Base URL** | `/api/trpc` | `/v2` |
| **Versioning** | Router-based | URL-based (/v2, /v3) |
| **Rate Limiting** | Per-user session | Per API key |
| **Scopes** | Role-based | OAuth scopes |
| **Real-time** | React Query | Webhooks |
| **Testing** | Vitest + MSW | Jest + Supertest |
| **Error Format** | tRPC errors | REST HTTP codes |
| **Batching** | Built-in | Not supported |
| **Caching** | React Query | HTTP caching |

---

## Use Cases

### Use API v1 (tRPC) When:

- Building features for the Cal.com web application
- Need full TypeScript type safety
- Working with internal Cal.com codebase
- Need real-time updates via React Query
- Building admin features

**Example**:
```typescript
// apps/web/pages/event-types/index.tsx
const { data: eventTypes, isLoading } = trpc.viewer.eventTypes.list.useQuery();

if (isLoading) return <Spinner />;

return (
  <div>
    {eventTypes.map(et => (
      <EventTypeCard key={et.id} eventType={et} />
    ))}
  </div>
);
```

### Use API v2 (Platform API) When:

- Building third-party integrations
- Creating mobile apps
- Integrating Cal.com into external systems
- Building marketplace apps
- Need OAuth 2.0 authentication
- Need public REST API

**Example**:
```bash
# Get access token
curl -X POST https://api.cal.com/v2/oauth/token \
  -d grant_type=authorization_code \
  -d code=abc123 \
  -d client_id=xxx \
  -d client_secret=yyy

# Use access token
curl https://api.cal.com/v2/bookings \
  -H "Authorization: Bearer eyJ..."
```

---

## Migration Path

### From API v1 to API v2

**Not a migration**: Both APIs coexist permanently.

- **API v1 (tRPC)**: Continues for internal web app
- **API v2 (Platform)**: New endpoint for external integrations

### Adding New Endpoints to API v2

```typescript
// 1. Create DTO
export class CreateEventTypeDto {
  @IsString()
  @ApiProperty()
  title: string;

  @IsNumber()
  @ApiProperty()
  length: number;
}

// 2. Create Service
@Injectable()
export class EventTypesService {
  async create(userId: number, dto: CreateEventTypeDto) {
    return this.prisma.eventType.create({
      data: {
        userId,
        title: dto.title,
        length: dto.length,
      },
    });
  }
}

// 3. Create Controller
@Controller("event-types")
export class EventTypesController {
  @Post()
  async create(@Req() req, @Body() dto: CreateEventTypeDto) {
    return this.eventTypesService.create(req.user.id, dto);
  }
}

// 4. Register Module
@Module({
  controllers: [EventTypesController],
  providers: [EventTypesService],
})
export class EventTypesModule {}

// 5. Import in AppModule
@Module({
  imports: [
    EventTypesModule,
    // ...
  ],
})
export class AppModule {}
```

---

## Security

### API v1 (tRPC)

- **Session-based**: NextAuth JWT
- **CSRF protection**: Built-in
- **Rate limiting**: Per user session
- **Validation**: Zod schemas

### API v2 (Platform)

- **OAuth 2.0**: Standard authorization
- **Scopes**: Fine-grained permissions
  - `bookings:read`
  - `bookings:write`
  - `event-types:read`
  - `event-types:write`
  - `webhooks:read`
  - `webhooks:write`
- **Rate limiting**: Per API key (100 req/min)
- **Validation**: class-validator decorators

---

## Testing

### API v1 (tRPC)

```typescript
import { createInnerTRPCContext } from "@calcom/trpc/server/createContext";
import { appRouter } from "@calcom/trpc/server/routers/_app";

describe("eventTypes.list", () => {
  it("should list event types", async () => {
    const ctx = await createInnerTRPCContext({ session: mockSession });
    const caller = appRouter.createCaller(ctx);

    const eventTypes = await caller.viewer.eventTypes.list();

    expect(eventTypes).toBeDefined();
    expect(eventTypes.length).toBeGreaterThan(0);
  });
});
```

### API v2 (Platform)

```typescript
import { Test } from "@nestjs/testing";
import * as request from "supertest";

describe("BookingsController (e2e)", () => {
  let app: INestApplication;

  beforeAll(async () => {
    const moduleFixture = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = moduleFixture.createNestApplication();
    await app.init();
  });

  it("/v2/bookings (GET)", () => {
    return request(app.getHttpServer())
      .get("/v2/bookings")
      .set("Authorization", `Bearer ${accessToken}`)
      .expect(200)
      .expect((res) => {
        expect(Array.isArray(res.body)).toBe(true);
      });
  });
});
```

---

## Summary

### Key Differences

1. **Purpose**:
   - v1: Internal web application
   - v2: External third-party integrations

2. **Technology**:
   - v1: tRPC (type-safe RPC)
   - v2: NestJS (REST framework)

3. **Authentication**:
   - v1: NextAuth session cookies
   - v2: OAuth 2.0 access tokens

4. **Documentation**:
   - v1: TypeScript types
   - v2: OpenAPI/Swagger

5. **Consumers**:
   - v1: apps/web (React application)
   - v2: Mobile apps, third-party services

### Best Practices

- ✅ Use **API v1 (tRPC)** for web application features
- ✅ Use **API v2 (Platform)** for public integrations
- ✅ Keep business logic in `@calcom/features`, share between both APIs
- ✅ Document API v2 endpoints with OpenAPI decorators
- ✅ Use OAuth scopes to limit API access
- ✅ Version API v2 endpoints via URL (/v2, /v3)

---

**Tác giả**: Claude (AI Assistant)
**Ngày tạo**: 2025-11-18
**Phiên bản**: 1.0
