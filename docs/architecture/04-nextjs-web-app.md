# PHASE 4: Next.js Web App Architecture

> **Mục tiêu**: Hiểu rõ kiến trúc Next.js hybrid routing, cấu trúc pages/app, layouts, middleware, và flow xử lý request.

## 📋 Table of Contents

- [1. Overview](#1-overview)
- [2. Hybrid Router Strategy](#2-hybrid-router-strategy)
- [3. App Router Structure](#3-app-router-structure)
- [4. Pages Router Structure](#4-pages-router-structure)
- [5. Middleware & Request Handling](#5-middleware--request-handling)
- [6. Internationalization (i18n)](#6-internationalization-i18n)
- [7. API Routes](#7-api-routes)
- [8. Component Architecture](#8-component-architecture)
- [9. SSR/SSG/ISR Strategies](#9-ssrssgisrs-strategies)
- [10. Build & Configuration](#10-build--configuration)
- [11. Best Practices](#11-best-practices)

---

## 1. Overview

### 1.1 Tech Stack

```json
{
  "framework": "Next.js 15.5.4",
  "react": "18.2.0",
  "routing": "Hybrid (App Router + Pages Router)",
  "i18n": "next-i18next 15.4.2",
  "state": "React Query (@tanstack/react-query 5.17.15)",
  "api": "tRPC (via @calcom/trpc)",
  "styling": "Tailwind CSS 3.3.3",
  "bundler": "Turbopack (dev), Webpack (production)",
  "deployment": "Vercel"
}
```

### 1.2 Key Features

- **Hybrid Routing**: Combination of App Router (React Server Components) and Pages Router for backward compatibility
- **Multi-tenancy**: Organization-based routing with subdomain support
- **Embed Support**: Specialized routes and rendering for embeddable booking widgets
- **SEO Optimized**: Dynamic metadata generation, OpenGraph, JSON-LD schema
- **Rate Limiting**: Per-route rate limiting via middleware
- **CSP Headers**: Content Security Policy with nonce support
- **i18n**: 30+ languages with server-side translation loading

### 1.3 Directory Structure

```
apps/web/
├── app/                          # App Router (Next.js 13+)
│   ├── (booking-page-wrapper)/   # Route group for booking pages
│   ├── (use-page-wrapper)/       # Route group for main app pages
│   ├── layout.tsx                # Root layout (i18n, fonts, providers)
│   ├── page.tsx                  # Root page
│   └── providers.tsx             # Client-side providers
├── pages/                        # Pages Router (backward compatibility)
│   ├── api/                      # API routes
│   ├── _app.tsx                  # Custom App component
│   ├── _document.tsx             # Custom Document component
│   └── _error.tsx                # Custom Error page
├── components/                   # Shared UI components
├── lib/                          # Utilities, hooks, helpers
├── public/                       # Static assets
├── middleware.ts                 # Edge middleware
└── next.config.js               # Next.js configuration
```

---

## 2. Hybrid Router Strategy

Cal.com uses **both** App Router and Pages Router in the same application.

### 2.1 Why Hybrid?

```typescript
// Rationale for hybrid approach:
// 1. Gradual migration from Pages Router to App Router
// 2. Some features still rely on Pages Router APIs
// 3. API routes remain in Pages Router
// 4. New features use App Router for better performance
```

### 2.2 Router Usage Matrix

| Feature | Router | Reason |
|---------|--------|--------|
| Booking Pages | App Router | SEO, Server Components, streaming |
| Main App Pages | App Router | Better performance, RSC benefits |
| API Routes | Pages Router | Stable API, no migration needed |
| tRPC Endpoints | Pages Router | Uses `/pages/api/trpc/[router]/[trpc].ts` |
| Legacy Routes | Pages Router | Backward compatibility |
| Embed Routes | App Router | Better control over rendering |

### 2.3 Migration Status

```typescript
// App Router Routes (New)
app/
├── (booking-page-wrapper)/
│   ├── [user]/page.tsx              // User booking page
│   ├── [user]/[type]/page.tsx       // Event type booking
│   ├── booking/[uid]/page.tsx       // Booking details
│   └── team/[slug]/page.tsx         // Team booking
├── (use-page-wrapper)/
│   ├── insights/page.tsx            // Analytics dashboard
│   ├── workflows/page.tsx           // Workflow management
│   ├── settings/*/page.tsx          // Settings pages
│   └── teams/*/page.tsx             // Team management

// Pages Router Routes (Legacy)
pages/
├── router/                          // Routing forms
├── api/                            // API endpoints
└── [remaining legacy routes]       // To be migrated
```

---

## 3. App Router Structure

### 3.1 Root Layout

**File**: `apps/web/app/layout.tsx`

```typescript
// Root layout handles:
// 1. HTML document structure
// 2. i18n locale and direction
// 3. Fonts (Inter, Cal Sans)
// 4. Global providers
// 5. Embed detection
export default async function RootLayout({ children }) {
  // Extract locale from request
  const locale = await getLocale();
  const direction = dir(locale); // rtl or ltr

  // Detect embed mode
  const isEmbed = await getIsEmbed();
  const embedColorScheme = await getEmbedColorScheme();

  // Load translations for this layout
  const translations = await loadTranslations(locale, ns);

  return (
    <html lang={locale} dir={direction}>
      <head>
        <link rel="preconnect" href="https://fonts.googleapis.com" />
      </head>
      <body className={cn(inter.variable, calSans.variable)}>
        <Providers isEmbed={isEmbed} embedColorScheme={embedColorScheme}>
          <AppRouterI18nProvider translations={translations}>
            {children}
          </AppRouterI18nProvider>
        </Providers>
      </body>
    </html>
  );
}
```

**Key Responsibilities**:
- ✅ Locale detection and direction (LTR/RTL)
- ✅ Font loading (Inter + Cal Sans custom font)
- ✅ Embed mode detection for widget rendering
- ✅ Provider setup (Session, tRPC, i18n, WebPush)
- ✅ Global styles injection

**Location**: `apps/web/app/layout.tsx:1`

### 3.2 Providers Setup

**File**: `apps/web/app/providers.tsx`

```typescript
export function Providers({
  isEmbed,
  embedColorScheme,
  children,
  nonce
}: ProvidersProps) {
  return (
    <SessionProvider>
      <TrpcProvider>
        <WebPushProvider>
          {children}
        </WebPushProvider>
      </TrpcProvider>
    </SessionProvider>
  );
}
```

**Provider Hierarchy**:
1. **SessionProvider** (NextAuth): Authentication state
2. **TrpcProvider**: tRPC client setup with React Query
3. **WebPushProvider**: Web push notification context

**Location**: `apps/web/app/providers.tsx:1`

### 3.3 Route Groups

Next.js route groups `(folder)` allow **layout composition without affecting URL structure**.

#### 3.3.1 `(booking-page-wrapper)` Group

**Purpose**: Pages that require booking-specific layout (no navigation, minimal UI)

```typescript
// apps/web/app/(booking-page-wrapper)/layout.tsx
export default async function BookingPageWrapperLayout({ children }) {
  const nonce = (await headers()).get("x-csp-nonce") ?? undefined;

  return (
    <PageWrapper isBookingPage={true} requiresLicense={false} nonce={nonce}>
      {children}
    </PageWrapper>
  );
}
```

**Routes in this group**:
```
(booking-page-wrapper)/
├── [user]/page.tsx                    // /:username
├── [user]/[type]/page.tsx             // /:username/:event-type
├── [user]/[type]/embed/page.tsx       // /:username/:event-type/embed
├── booking/[uid]/page.tsx             // /booking/:uid (booking details)
├── booking-successful/[uid]/page.tsx  // /booking-successful/:uid
├── team/[slug]/page.tsx               // /team/:slug
├── team/[slug]/[type]/page.tsx        // /team/:slug/:event-type
└── d/[link]/[slug]/page.tsx           // /d/:link/:slug (dynamic links)
```

**Location**: `apps/web/app/(booking-page-wrapper)/layout.tsx:1`

#### 3.3.2 `(use-page-wrapper)` Group

**Purpose**: Main app pages with full navigation and shell UI

```typescript
// apps/web/app/(use-page-wrapper)/layout.tsx
export default async function PageWrapperLayout({ children }) {
  const nonce = (await headers()).get("x-csp-nonce") ?? undefined;
  const headScript = process.env.NEXT_PUBLIC_HEAD_SCRIPTS;
  const bodyScript = process.env.NEXT_PUBLIC_BODY_SCRIPTS;

  return (
    <PageWrapper requiresLicense={false} nonce={nonce}>
      {children}
      {/* Inject custom scripts */}
      {headScript && <Script id="head-script" nonce={nonce} dangerouslySetInnerHTML={{ __html: headScript }} />}
      {bodyScript && <Script id="body-script" nonce={nonce} dangerouslySetInnerHTML={{ __html: bodyScript }} />}
    </PageWrapper>
  );
}
```

**Routes in this group**:
```
(use-page-wrapper)/
├── insights/page.tsx            // Analytics dashboard
├── workflows/page.tsx           // Workflow automation
├── settings/                    // Settings pages
│   ├── my-account/
│   ├── organizations/
│   ├── teams/
│   └── security/
├── teams/                       // Team management
├── apps/                        // App marketplace
├── enterprise/page.tsx          // Enterprise features
├── signup/page.tsx              // Registration
└── more/page.tsx               // Additional features
```

**Location**: `apps/web/app/(use-page-wrapper)/layout.tsx:1`

### 3.4 Server Components & Data Fetching

**Example**: User booking page

**File**: `apps/web/app/(booking-page-wrapper)/[user]/page.tsx`

```typescript
// 1. Metadata Generation (for SEO)
export const generateMetadata = async ({ params, searchParams }: PageProps) => {
  // Build legacy context for compatibility with existing getServerSideProps
  const props = await getData(
    buildLegacyCtx(await headers(), await cookies(), await params, await searchParams)
  );

  const { profile, markdownStrippedBio, isOrgSEOIndexable, entity } = props;
  const allowSEOIndexing = profile.allowSEOIndexing && isOrgSEOIndexable;

  const meeting = {
    title: markdownStrippedBio,
    profile: { name: profile.name, image: profile.image },
    users: [{ username: profile.username, name: profile.name }],
  };

  const metadata = await generateMeetingMetadata(
    meeting,
    () => profile.name,
    () => markdownStrippedBio,
    false,
    getOrgFullOrigin(entity.orgSlug ?? null),
    `/${decodeParams(await params).user}`
  );

  return {
    ...metadata,
    robots: {
      follow: allowSEOIndexing,
      index: allowSEOIndexing,
    },
  };
};

// 2. Server Component (RSC)
const getData = withAppDirSsr<LegacyPageProps>(getServerSideProps);

const ServerPage = async ({ params, searchParams }: PageProps) => {
  // Fetch data on the server
  const props = await getData(
    buildLegacyCtx(await headers(), await cookies(), await params, await searchParams)
  );

  // Render legacy page component with server-fetched props
  return <LegacyPage {...props} />;
};

export default ServerPage;
```

**Key Patterns**:
- ✅ **`generateMetadata`**: SEO optimization with dynamic OpenGraph tags
- ✅ **`withAppDirSsr`**: Adapter to reuse existing `getServerSideProps` logic
- ✅ **`buildLegacyCtx`**: Convert App Router primitives to Pages Router context
- ✅ **Server Components**: Data fetching happens on server, reducing client bundle

**Location**: `apps/web/app/(booking-page-wrapper)/[user]/page.tsx:15`

### 3.5 Legacy Context Bridge

**File**: `apps/web/lib/buildLegacyCtx.ts`

```typescript
// Converts App Router APIs to Pages Router context
export function buildLegacyCtx(
  headers: Headers,
  cookies: ReadonlyRequestCookies,
  params: Params,
  searchParams: SearchParams
) {
  // Reconstruct req/res-like objects for legacy code
  return {
    req: {
      headers: Object.fromEntries(headers.entries()),
      cookies: Object.fromEntries(cookies.getAll().map(c => [c.name, c.value])),
      url: buildUrl(params, searchParams),
    },
    res: {},
    query: { ...params, ...searchParams },
    params: decodeParams(params),
  };
}
```

**Why needed?**
- Existing `getServerSideProps` functions expect `req`, `res`, `query`
- App Router uses `headers()`, `cookies()`, `params`, `searchParams`
- Bridge allows gradual migration without rewriting all data fetching logic

**Location**: `apps/web/lib/buildLegacyCtx.ts:1`

---

## 4. Pages Router Structure

### 4.1 Custom App (`_app.tsx`)

**File**: `apps/web/pages/_app.tsx`

```typescript
function MyApp(props: AppProps) {
  const { Component, pageProps } = props;

  return (
    <SessionProvider session={pageProps.session ?? undefined}>
      <WebPushProvider>
        <CacheProvider>
          {/* Support PageWrapper pattern */}
          {Component.PageWrapper ? (
            <Component.PageWrapper {...props} />
          ) : (
            <Component {...pageProps} />
          )}
        </CacheProvider>
      </WebPushProvider>
    </SessionProvider>
  );
}

// Locale detection at app level
MyApp.getInitialProps = async ({ ctx }: { ctx: NextPageContext }) => {
  const { req } = ctx;
  let newLocale = "en";

  if (req) {
    const { getLocale } = await import("@calcom/features/auth/lib/getLocale");
    newLocale = await getLocale(req);
  } else if (typeof window !== "undefined" && window.calNewLocale) {
    newLocale = window.calNewLocale;
  }

  return { pageProps: { newLocale } };
};

// Wrap with tRPC client
const WrappedMyApp = trpc.withTRPC(MyApp);
export default WrappedMyApp;
```

**Key Features**:
- ✅ **tRPC Wrapper**: `trpc.withTRPC()` injects React Query client
- ✅ **SessionProvider**: NextAuth session management
- ✅ **PageWrapper Pattern**: Pages can define custom wrapper component
- ✅ **Locale Detection**: Server-side locale detection on initial load

**Location**: `apps/web/pages/_app.tsx:14`

### 4.2 Custom Document (`_document.tsx`)

**File**: `apps/web/pages/_document.tsx`

```typescript
class MyDocument extends Document<Props> {
  static async getInitialProps(ctx: DocumentContext) {
    // 1. Locale detection
    const newLocale = ctx.req ? await getLocale(ctx.req) : "en";

    // 2. Embed detection
    const asPath = ctx.asPath || "";
    const parsedUrl = new URL(asPath, "https://dummyurl");
    const isEmbed = parsedUrl.pathname.endsWith("/embed") ||
                    parsedUrl.searchParams.get("embedType") !== null;
    const embedColorScheme = parsedUrl.searchParams.get("ui.color-scheme");

    const initialProps = await Document.getInitialProps(ctx);
    return { isEmbed, embedColorScheme, ...initialProps, newLocale };
  }

  render() {
    const { isEmbed, embedColorScheme, newLocale } = this.props;
    const newDir = dir(newLocale); // rtl or ltr

    const isDesktopApp = (() => {
      try {
        return platform.todesktop.isDesktopApp();
      } catch {
        return false;
      }
    })();

    return (
      <Html lang={newLocale} dir={newDir} style={embedColorScheme ? { colorScheme: embedColorScheme } : undefined}>
        <Head>
          {/* Inject locale and theme before hydration */}
          <script
            id="newLocale"
            dangerouslySetInnerHTML={{
              __html: `
                window.calNewLocale = "${newLocale}";
                window.calIsDesktopApp = ${isDesktopApp};
                (${applyTheme.toString()})();
                (${applyToDesktopClass.toString()})();
              `,
            }}
          />
          <link rel="apple-touch-icon" sizes="180x180" href="/api/logo?type=apple-touch-icon" />
          <link rel="icon" type="image/png" sizes="32x32" href="/api/logo?type=favicon-32" />
          <link rel="manifest" href="/site.webmanifest" />
          <meta name="theme-color" media="(prefers-color-scheme: light)" content="#F9FAFC" />
          <meta name="theme-color" media="(prefers-color-scheme: dark)" content="#1F1F1F" />
        </Head>

        <body
          className="dark:bg-default bg-subtle antialiased"
          style={isEmbed ? { background: "transparent", visibility: "hidden" } : {}}>
          <Main />
          <NextScript />
        </body>
      </Html>
    );
  }
}
```

**Key Features**:
- ✅ **Locale Injection**: Sets `window.calNewLocale` before React hydration
- ✅ **Theme Application**: Runs `applyTheme()` to prevent FOUC (Flash of Unstyled Content)
- ✅ **Embed Handling**: Hides embed until parent frame initializes it
- ✅ **Desktop App Detection**: Sets `window.calIsDesktopApp` flag
- ✅ **Dynamic Favicons**: `/api/logo` endpoint for customizable branding

**Location**: `apps/web/pages/_document.tsx:13`

### 4.3 Pages Router Routes

```
pages/
├── api/                           # API Routes
│   ├── auth/
│   │   ├── [...nextauth].ts       # NextAuth handler
│   │   └── verify-email.ts        # Email verification
│   ├── book/
│   │   ├── event.ts               # Book event
│   │   ├── instant-event.ts       # Instant booking
│   │   └── recurring-event.ts     # Recurring booking
│   ├── trpc/
│   │   ├── [router]/[trpc].ts     # Modular tRPC endpoints
│   │   └── ...                    # 40+ router endpoints
│   ├── stripe/webhook.ts          # Stripe webhook handler
│   └── integrations/*/webhook.ts  # Integration webhooks
└── router/                        # Routing Forms (legacy)
    ├── index.tsx                  # Routing form builder
    └── embed.tsx                  # Embed version
```

---

## 5. Middleware & Request Handling

### 5.1 Edge Middleware

**File**: `apps/web/middleware.ts`

```typescript
export const middleware = async (req: NextRequest) => {
  // 1. Extract request metadata
  const requestorIp = getIP(req);
  const pathname = req.nextUrl.pathname;

  // 2. Rate Limiting
  await checkRateLimitAndThrowError({
    rateLimitingType: "common",
    identifier: piiHasher.hash(`${pathname}-${requestorIp}`),
  });

  // 3. CSP Headers
  const nonce = buildNonce();
  const cspHeader = buildCspHeader(nonce, req);

  // 4. Routing Logic
  const url = req.nextUrl;

  // Handle organization subdomains
  if (isOrgRequest(req)) {
    return handleOrgRouting(req);
  }

  // Handle legacy redirects
  if (pathname === "/old-path") {
    return NextResponse.redirect(new URL("/new-path", req.url));
  }

  // 5. Add security headers
  const response = NextResponse.next();
  response.headers.set("Content-Security-Policy", cspHeader);
  response.headers.set("X-CSP-Nonce", nonce);
  response.headers.set("X-Frame-Options", "SAMEORIGIN");
  response.headers.set("X-Content-Type-Options", "nosniff");

  return response;
};

export const config = {
  matcher: [
    "/((?!api/|_next/|_static|_vercel|[\\w-]+\\.\\w+).*)",
  ],
};
```

**Middleware Responsibilities**:

| Step | Responsibility | Implementation |
|------|----------------|----------------|
| 1 | **Rate Limiting** | Hash IP + pathname, check Redis/LRU cache |
| 2 | **CSP Headers** | Generate nonce, build CSP policy |
| 3 | **Organization Routing** | Subdomain detection, rewrite to `/org/[slug]` |
| 4 | **Legacy Redirects** | Permanent redirects for old URLs |
| 5 | **Security Headers** | CSP, X-Frame-Options, X-Content-Type-Options |

**Location**: `apps/web/middleware.ts:1`

### 5.2 Rate Limiting

```typescript
// Rate limiting configuration
const rateLimitConfig = {
  common: {
    points: 10,        // 10 requests
    duration: 1,       // per 1 second
  },
  api: {
    points: 100,       // 100 requests
    duration: 60,      // per 60 seconds
  },
  booking: {
    points: 5,         // 5 requests
    duration: 10,      // per 10 seconds
  },
};

// Implementation in middleware
async function checkRateLimitAndThrowError({
  rateLimitingType,
  identifier,
}: RateLimitOptions) {
  const { isRateLimited } = await checkRateLimit({
    identifier,
    rateLimitingType,
  });

  if (isRateLimited) {
    throw new Error("Rate limit exceeded");
  }
}
```

**Storage**: Redis (production) or LRU cache (development)

**Location**: `apps/web/middleware.ts:20` (approximate)

### 5.3 CSP (Content Security Policy)

```typescript
function buildCspHeader(nonce: string, req: NextRequest): string {
  const cspDirectives = {
    "default-src": ["'self'"],
    "script-src": [
      "'self'",
      `'nonce-${nonce}'`,
      "'strict-dynamic'",
      "https://js.stripe.com",
      "https://www.googletagmanager.com",
    ],
    "style-src": [
      "'self'",
      `'nonce-${nonce}'`,
      "'unsafe-inline'", // Required for Tailwind
      "https://fonts.googleapis.com",
    ],
    "img-src": ["'self'", "data:", "https:", "blob:"],
    "font-src": ["'self'", "data:", "https://fonts.gstatic.com"],
    "connect-src": ["'self'", "https://api.cal.com", "wss://"],
    "frame-src": ["'self'", "https://js.stripe.com", "https://www.google.com"],
    "frame-ancestors": ["'self'"],
  };

  return Object.entries(cspDirectives)
    .map(([key, values]) => `${key} ${values.join(" ")}`)
    .join("; ");
}
```

**Nonce Usage**:
```tsx
// In layout/document
<script nonce={nonce}>...</script>
<style nonce={nonce}>...</style>

// In components
const nonce = headers().get("x-csp-nonce");
<Script nonce={nonce} src="..." />
```

**Location**: `apps/web/lib/buildNonce.ts:1`, `apps/web/lib/csp.ts:1`

---

## 6. Internationalization (i18n)

### 6.1 Configuration

**File**: `apps/web/next-i18next.config.js`

```javascript
module.exports = {
  i18n: {
    defaultLocale: "en",
    locales: [
      "en", "fr", "de", "es", "it", "pt", "pt-BR", "nl", "pl", "ru",
      "ja", "ko", "zh-CN", "zh-TW", "ar", "he", "vi", "tr", "cs",
      "sv", "da", "no", "fi", "hu", "ro", "uk", "sr", "el", "bg",
      // ... 30+ total languages
    ],
  },
  react: {
    useSuspense: false, // Disable suspense for SSR
  },
  interpolation: {
    escapeValue: false, // React already escapes
  },
};
```

### 6.2 Translation Loading

**Server-Side (App Router)**:
```typescript
// In layout.tsx or page.tsx
import { loadTranslations } from "@calcom/lib/i18n/loadTranslations";

export default async function Page() {
  const locale = await getLocale();
  const translations = await loadTranslations(locale, ["common", "booking"]);

  return (
    <AppRouterI18nProvider translations={translations}>
      {/* Page content */}
    </AppRouterI18nProvider>
  );
}
```

**Client-Side (Pages Router)**:
```typescript
// In pages/*.tsx
import { serverSideTranslations } from "next-i18next/serverSideTranslations";

export const getServerSideProps = async ({ locale }) => {
  return {
    props: {
      ...(await serverSideTranslations(locale, ["common", "booking"])),
    },
  };
};
```

### 6.3 Usage in Components

```typescript
import { useTranslation } from "next-i18next";

function BookingButton() {
  const { t } = useTranslation("booking");

  return (
    <button>
      {t("book_event")}
    </button>
  );
}
```

**Translation Files**: `apps/web/public/static/locales/[locale]/[namespace].json`

```json
// public/static/locales/en/booking.json
{
  "book_event": "Book Event",
  "select_date": "Select a date",
  "confirm_booking": "Confirm Booking"
}
```

### 6.4 Locale Detection Flow

```
1. Check cookie: NEXT_LOCALE
   ↓
2. Check Accept-Language header
   ↓
3. Check subdomain locale (e.g., fr.cal.com)
   ↓
4. Fallback to defaultLocale (en)
```

**Implementation**: `packages/features/auth/lib/getLocale.ts`

---

## 7. API Routes

### 7.1 tRPC API Endpoints

**Pattern**: `/pages/api/trpc/[router]/[trpc].ts`

Each tRPC router gets its own API endpoint:

```typescript
// apps/web/pages/api/trpc/bookings/[trpc].ts
import { createNextApiHandler } from "@calcom/trpc/server/createNextApiHandler";
import { bookingsRouter } from "@calcom/trpc/server/routers/viewer/bookings/_router";

export default createNextApiHandler(bookingsRouter);
```

**Available Endpoints**:
```
/api/trpc/admin/[trpc]               # Admin operations
/api/trpc/auth/[trpc]                # Authentication
/api/trpc/availability/[trpc]        # Availability
/api/trpc/bookings/[trpc]            # Bookings CRUD
/api/trpc/eventTypes/[trpc]          # Event types
/api/trpc/teams/[trpc]               # Team management
/api/trpc/payments/[trpc]            # Payment processing
/api/trpc/workflows/[trpc]           # Workflow automation
... (40+ total routers)
```

**Why modular endpoints?**
- ✅ Better cold start performance (only load needed router)
- ✅ Fine-grained caching
- ✅ Easier to debug and monitor

**Location**: `apps/web/pages/api/trpc/bookings/[trpc].ts:1`

### 7.2 NextAuth API Route

**File**: `apps/web/pages/api/auth/[...nextauth].ts`

```typescript
import NextAuth from "next-auth";
import { authOptions } from "@calcom/features/auth/lib/next-auth-options";

export default NextAuth(authOptions);
```

**Endpoints Created**:
- `GET /api/auth/signin` - Sign in page
- `POST /api/auth/signin/:provider` - Sign in with provider
- `GET /api/auth/signout` - Sign out page
- `POST /api/auth/signout` - Sign out action
- `GET /api/auth/session` - Get session
- `GET /api/auth/csrf` - CSRF token
- `GET /api/auth/providers` - List providers
- `GET /api/auth/callback/:provider` - OAuth callback

**Auth Providers**:
- Email (magic link)
- Google OAuth
- SAML SSO
- CAL (custom provider)

### 7.3 Booking API Routes

**File**: `apps/web/pages/api/book/event.ts`

```typescript
import { handleNewBooking } from "@calcom/features/bookings/lib/handleNewBooking";

export default async function handler(req, res) {
  if (req.method !== "POST") {
    return res.status(405).json({ message: "Method not allowed" });
  }

  try {
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
  } catch (error) {
    return res.status(500).json({ error: error.message });
  }
}
```

**Other booking endpoints**:
- `POST /api/book/instant-event.ts` - Instant bookings
- `POST /api/book/recurring-event.ts` - Recurring bookings

### 7.4 Webhook Handlers

```typescript
// apps/web/pages/api/stripe/webhook.ts
import { buffer } from "micro";
import Stripe from "stripe";

export const config = {
  api: {
    bodyParser: false, // Must be disabled for Stripe
  },
};

export default async function handler(req, res) {
  const buf = await buffer(req);
  const sig = req.headers["stripe-signature"];

  try {
    const event = stripe.webhooks.constructEvent(
      buf,
      sig,
      process.env.STRIPE_WEBHOOK_SECRET
    );

    switch (event.type) {
      case "checkout.session.completed":
        await handlePaymentSuccess(event.data.object);
        break;
      case "customer.subscription.deleted":
        await handleSubscriptionCancelled(event.data.object);
        break;
      // ...
    }

    res.status(200).json({ received: true });
  } catch (err) {
    res.status(400).send(`Webhook Error: ${err.message}`);
  }
}
```

**Other webhooks**:
- `/api/integrations/paypal/webhook.ts`
- `/api/integrations/alby/webhook.ts`
- `/api/integrations/btcpayserver/webhook.ts`

---

## 8. Component Architecture

### 8.1 Component Organization

```
components/
├── apps/                    # App-related components
├── auth/                    # Authentication UI
├── booking/                 # Booking flow components
│   ├── BookingPage.tsx
│   ├── DatePicker.tsx
│   ├── TimeSlots.tsx
│   └── ConfirmationPage.tsx
├── dialog/                  # Modal dialogs
├── error/                   # Error boundaries
├── eventtype/               # Event type management
├── getting-started/         # Onboarding
├── integrations/            # Integration cards
├── layouts/                 # Layout components
│   ├── Shell.tsx           # Main app shell
│   ├── SettingsLayout.tsx
│   └── BookingLayout.tsx
├── schemas/                 # Form schemas (Zod)
├── security/                # Security components (2FA, etc.)
├── settings/                # Settings panels
├── setup/                   # Setup wizards
├── team/                    # Team management
└── ui/                      # Shared UI primitives
    ├── Button.tsx
    ├── Input.tsx
    ├── Select.tsx
    └── ...
```

### 8.2 Shell Layout

**File**: `apps/web/components/layouts/Shell.tsx`

```typescript
export function Shell({
  children,
  title,
  heading,
  subtitle,
  CTA,
  backPath,
}: ShellProps) {
  const { t } = useTranslation();

  return (
    <div className="flex h-screen overflow-hidden">
      {/* Sidebar */}
      <Sidebar />

      {/* Main Content */}
      <div className="flex flex-1 flex-col overflow-y-auto">
        {/* Header */}
        <Header
          title={title}
          heading={heading}
          subtitle={subtitle}
          CTA={CTA}
          backPath={backPath}
        />

        {/* Content */}
        <main className="flex-1 p-6">
          {children}
        </main>
      </div>
    </div>
  );
}
```

**Usage**:
```tsx
<Shell
  title="Event Types"
  heading="Your Event Types"
  subtitle="Manage your events"
  CTA={<CreateEventTypeButton />}
>
  <EventTypeList />
</Shell>
```

### 8.3 PageWrapper Pattern

**File**: `apps/web/components/PageWrapperAppDir.tsx`

```typescript
export default function PageWrapper({
  children,
  isBookingPage = false,
  requiresLicense = false,
  nonce,
}: PageWrapperProps) {
  return (
    <>
      {/* License check */}
      {requiresLicense && <LicenseRequired />}

      {/* Booking-specific layout */}
      {isBookingPage ? (
        <BookingPageWrapper nonce={nonce}>
          {children}
        </BookingPageWrapper>
      ) : (
        <Shell nonce={nonce}>
          {children}
        </Shell>
      )}
    </>
  );
}
```

### 8.4 UI Component Library

Cal.com uses **Radix UI** primitives with Tailwind styling:

```typescript
// components/ui/Button.tsx
import * as React from "react";
import { cva, type VariantProps } from "class-variance-authority";

const buttonVariants = cva(
  "inline-flex items-center justify-center rounded-md text-sm font-medium transition-colors focus-visible:outline-none focus-visible:ring-2 disabled:opacity-50 disabled:pointer-events-none",
  {
    variants: {
      variant: {
        default: "bg-primary text-primary-foreground hover:bg-primary/90",
        destructive: "bg-destructive text-destructive-foreground hover:bg-destructive/90",
        outline: "border border-input hover:bg-accent hover:text-accent-foreground",
        ghost: "hover:bg-accent hover:text-accent-foreground",
      },
      size: {
        default: "h-10 py-2 px-4",
        sm: "h-9 px-3 rounded-md",
        lg: "h-11 px-8 rounded-md",
      },
    },
    defaultVariants: {
      variant: "default",
      size: "default",
    },
  }
);

export interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {}

export const Button = React.forwardRef<HTMLButtonElement, ButtonProps>(
  ({ className, variant, size, ...props }, ref) => {
    return (
      <button
        className={cn(buttonVariants({ variant, size }), className)}
        ref={ref}
        {...props}
      />
    );
  }
);
```

**Usage**:
```tsx
<Button variant="default" size="lg">
  Book Event
</Button>
```

**Component Catalog**: Maintained in `@calcom/ui` package

---

## 9. SSR/SSG/ISR Strategies

### 9.1 Server-Side Rendering (SSR)

**Use Cases**:
- User profile pages (personalized content)
- Booking pages (dynamic availability)
- Dashboard pages (user-specific data)

**Example** (App Router):
```typescript
// Default behavior in App Router - all components are Server Components
export default async function Page({ params }) {
  const data = await fetchData(params);
  return <View data={data} />;
}
```

**Example** (Pages Router):
```typescript
export const getServerSideProps = async (ctx) => {
  const data = await fetchData(ctx.params);
  return { props: { data } };
};
```

### 9.2 Static Site Generation (SSG)

**Use Cases**:
- Public team pages
- Help/docs pages
- Marketing pages

**Example** (App Router):
```typescript
// Static page with revalidation
export const revalidate = 3600; // Revalidate every hour

export default async function Page() {
  const data = await fetchStaticData();
  return <View data={data} />;
}
```

**Example** (Pages Router):
```typescript
export const getStaticProps = async () => {
  const data = await fetchStaticData();
  return {
    props: { data },
    revalidate: 3600, // ISR: revalidate every hour
  };
};

export const getStaticPaths = async () => {
  const paths = await getAllPaths();
  return {
    paths,
    fallback: "blocking", // Generate on-demand for missing paths
  };
};
```

### 9.3 Incremental Static Regeneration (ISR)

**Use Cases**:
- Team booking pages (updated on team member changes)
- Event type pages (updated on configuration changes)

**Configuration**:
```typescript
export const revalidate = 60; // Revalidate every 60 seconds

// Or time-based revalidation
export const getStaticProps = async () => {
  return {
    props: { data },
    revalidate: 60,
  };
};
```

### 9.4 Client-Side Rendering (CSR)

**Use Cases**:
- Highly interactive components
- Real-time data
- User-specific actions

**Example**:
```typescript
"use client"; // Mark as Client Component

import { trpc } from "@calcom/trpc/react";

export function BookingList() {
  const { data, isLoading } = trpc.bookings.list.useQuery();

  if (isLoading) return <Spinner />;

  return (
    <div>
      {data.map(booking => (
        <BookingCard key={booking.id} booking={booking} />
      ))}
    </div>
  );
}
```

### 9.5 Rendering Strategy Matrix

| Page Type | Strategy | Reason |
|-----------|----------|--------|
| User Booking Page | SSR | Personalized, dynamic availability |
| Team Public Page | ISR (60s) | Semi-static, updated occasionally |
| Marketing Pages | SSG | Fully static |
| Dashboard | SSR + CSR | Server shell + client interactions |
| Settings Pages | CSR | Highly interactive forms |
| Event Type Config | SSR | Requires auth, dynamic |

---

## 10. Build & Configuration

### 10.1 Next.js Config

**File**: `apps/web/next.config.js`

Key configurations:

```javascript
module.exports = {
  // Internationalization
  i18n: {
    locales: ["en", "fr", "de", ...],
    defaultLocale: "en",
  },

  // Image optimization
  images: {
    domains: [
      "avatars.githubusercontent.com",
      "lh3.googleusercontent.com",
      "cal.com",
    ],
  },

  // Rewrites for organization routing
  rewrites: async () => {
    return {
      beforeFiles: [
        // Organization subdomain rewrites
        ...nextJsOrgRewriteConfig,
      ],
    };
  },

  // Redirects for legacy URLs
  redirects: async () => {
    return [
      {
        source: "/old-path",
        destination: "/new-path",
        permanent: true,
      },
    ];
  },

  // Webpack configuration
  webpack: (config, { isServer }) => {
    // Prisma workaround for monorepo
    if (isServer) {
      config.plugins.push(new PrismaPlugin());
    }
    return config;
  },

  // Environment variables
  env: {
    NEXT_PUBLIC_CALCOM_VERSION: version,
    NEXT_PUBLIC_WEBAPP_URL: process.env.NEXT_PUBLIC_WEBAPP_URL,
  },

  // Experimental features
  experimental: {
    serverActions: true,
    turbo: {
      // Turbopack configuration
    },
  },
};
```

### 10.2 Build Process

```bash
# 1. Pre-build: Copy static assets
yarn turbo run copy-app-store-static

# 2. Build Next.js
next build

# 3. Post-build: Create Sentry release
yarn sentry:release
```

**Output**:
```
.next/
├── static/               # Static assets
│   └── chunks/          # JS chunks
├── server/              # Server-side code
│   ├── app/            # App Router pages
│   └── pages/          # Pages Router pages
└── cache/              # Build cache
```

### 10.3 Deployment

**Vercel Configuration** (`.vercel/project.json`):
```json
{
  "buildCommand": "yarn build",
  "devCommand": "yarn dev",
  "installCommand": "yarn install",
  "framework": "nextjs",
  "outputDirectory": ".next"
}
```

**Environment Variables**:
```bash
# Required
NEXTAUTH_SECRET=<secret>
CALENDSO_ENCRYPTION_KEY=<key>
DATABASE_URL=<postgres-url>

# Optional
ORGANIZATIONS_ENABLED=1
STRIPE_API_KEY=<key>
GOOGLE_API_CREDENTIALS=<json>
SENTRY_DSN=<dsn>
```

---

## 11. Best Practices

### 11.1 Performance Optimization

#### Bundle Size Optimization
```typescript
// ✅ Good: Dynamic import for heavy components
const HeavyChart = dynamic(() => import("./HeavyChart"), {
  loading: () => <Spinner />,
  ssr: false, // Skip SSR if not needed
});

// ❌ Bad: Import everything
import { HeavyChart } from "./components";
```

#### Image Optimization
```tsx
// ✅ Good: Use Next.js Image component
import Image from "next/image";

<Image
  src="/avatar.jpg"
  alt="User avatar"
  width={40}
  height={40}
  priority={false}
/>

// ❌ Bad: Regular img tag
<img src="/avatar.jpg" alt="User avatar" />
```

#### Font Loading
```typescript
// ✅ Good: Use next/font for automatic optimization
import { Inter } from "next/font/google";

const inter = Inter({
  subsets: ["latin"],
  display: "swap",
  variable: "--font-inter",
});

// ❌ Bad: Load fonts via CSS
@import url('https://fonts.googleapis.com/css2?family=Inter');
```

### 11.2 SEO Best Practices

#### Metadata Configuration
```typescript
// ✅ Good: Use generateMetadata in App Router
export async function generateMetadata({ params }): Promise<Metadata> {
  const data = await getData(params);

  return {
    title: data.title,
    description: data.description,
    openGraph: {
      title: data.title,
      description: data.description,
      images: [{ url: data.image }],
    },
    twitter: {
      card: "summary_large_image",
      title: data.title,
      description: data.description,
      images: [data.image],
    },
    robots: {
      index: data.allowIndexing,
      follow: data.allowIndexing,
    },
  };
}

// ❌ Bad: Hardcoded metadata
export const metadata = {
  title: "Cal.com",
};
```

#### Structured Data
```tsx
import { jsonLdScriptProps } from "react-schemaorg";

<script
  {...jsonLdScriptProps({
    "@context": "https://schema.org",
    "@type": "Person",
    name: user.name,
    image: user.avatar,
    url: `https://cal.com/${user.username}`,
  })}
/>
```

### 11.3 Security Best Practices

#### Input Validation
```typescript
// ✅ Good: Zod validation
import { z } from "zod";

const bookingSchema = z.object({
  eventTypeId: z.number().positive(),
  start: z.string().datetime(),
  end: z.string().datetime(),
  email: z.string().email(),
});

const validated = bookingSchema.parse(req.body);

// ❌ Bad: No validation
const { eventTypeId, start, end, email } = req.body;
```

#### XSS Prevention
```tsx
// ✅ Good: React escapes by default
<div>{user.name}</div>

// ⚠️ Use dangerouslySetInnerHTML only with sanitized HTML
import DOMPurify from "dompurify";

<div dangerouslySetInnerHTML={{
  __html: DOMPurify.sanitize(htmlContent)
}} />

// ❌ Bad: Unsanitized HTML
<div dangerouslySetInnerHTML={{ __html: userInput }} />
```

#### Authentication Checks
```typescript
// ✅ Good: Use middleware for auth
export const authedProcedure = procedure
  .use(isAuthed)
  .use(checkPermissions);

// ❌ Bad: Manual checks everywhere
if (!session || !session.user) {
  throw new Error("Unauthorized");
}
```

### 11.4 Accessibility

```tsx
// ✅ Good: Semantic HTML + ARIA
<button
  aria-label="Close dialog"
  aria-pressed={isOpen}
  onClick={onClose}
>
  <CloseIcon aria-hidden="true" />
</button>

// ✅ Good: Form labels
<label htmlFor="email">
  Email
  <input id="email" type="email" />
</label>

// ✅ Good: Focus management
const dialogRef = useRef<HTMLDivElement>(null);

useEffect(() => {
  if (isOpen) {
    dialogRef.current?.focus();
  }
}, [isOpen]);
```

### 11.5 Error Handling

#### Error Boundaries
```typescript
// apps/web/components/error/ErrorBoundary.tsx
export class ErrorBoundary extends Component {
  state = { hasError: false };

  static getDerivedStateFromError(error) {
    return { hasError: true };
  }

  componentDidCatch(error, errorInfo) {
    Sentry.captureException(error, { extra: errorInfo });
  }

  render() {
    if (this.state.hasError) {
      return <ErrorFallback />;
    }
    return this.props.children;
  }
}
```

#### API Error Handling
```typescript
// ✅ Good: Structured error responses
try {
  const result = await handleBooking(data);
  return res.status(200).json({ success: true, data: result });
} catch (error) {
  if (error instanceof ValidationError) {
    return res.status(400).json({
      error: "VALIDATION_ERROR",
      message: error.message,
      fields: error.fields,
    });
  }

  Sentry.captureException(error);
  return res.status(500).json({
    error: "INTERNAL_ERROR",
    message: "An unexpected error occurred",
  });
}
```

---

## Summary

### Key Takeaways

1. **Hybrid Router Strategy**: Cal.com uses both App Router (new features) and Pages Router (legacy/API routes) for gradual migration.

2. **Route Groups**: `(booking-page-wrapper)` and `(use-page-wrapper)` provide different layouts without affecting URLs.

3. **Server Components**: App Router pages are Server Components by default, reducing client bundle size.

4. **Middleware**: Handles rate limiting, CSP headers, organization routing, and security at the edge.

5. **i18n**: Server-side translation loading with 30+ languages, locale detection from cookies/headers/subdomain.

6. **tRPC API**: Modular endpoints (`/api/trpc/[router]/[trpc].ts`) for better performance and debugging.

7. **Component Architecture**: Feature-based organization with shared UI library (`@calcom/ui`).

8. **Rendering Strategies**: SSR for personalized pages, ISR for semi-static content, SSG for marketing pages.

9. **Build Pipeline**: Turborepo orchestrates multi-stage build with asset copying, Next.js build, and Sentry release.

10. **Security**: CSP with nonces, rate limiting, input validation with Zod, XSS prevention.

### File Reference

| File | Purpose | Location |
|------|---------|----------|
| Root Layout | i18n, fonts, providers | `apps/web/app/layout.tsx:1` |
| Providers | Session, tRPC, WebPush | `apps/web/app/providers.tsx:1` |
| Middleware | Rate limit, CSP, routing | `apps/web/middleware.ts:1` |
| Custom App | Pages Router setup | `apps/web/pages/_app.tsx:14` |
| Custom Document | Locale/theme injection | `apps/web/pages/_document.tsx:13` |
| Booking Page | App Router example | `apps/web/app/(booking-page-wrapper)/[user]/page.tsx:15` |
| tRPC Endpoint | API route pattern | `apps/web/pages/api/trpc/bookings/[trpc].ts:1` |
| Legacy Context | App/Pages Router bridge | `apps/web/lib/buildLegacyCtx.ts:1` |
| Next Config | Build configuration | `apps/web/next.config.js:1` |

### Next Steps

- **PHASE 5**: Feature Packages Deep Dive (`@calcom/features`)
- **PHASE 6**: App Store & Integration Architecture
- **PHASE 7**: Booking Flow End-to-End

---

**Tác giả**: Claude (AI Assistant)
**Ngày tạo**: 2025-11-18
**Phiên bản**: 1.0
