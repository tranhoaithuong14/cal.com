# 📦 PHASE 1: Monorepo Structure & Build Pipeline (DEEP DIVE v2)

📘 **This is the DEEP DIVE version of Phase 1 (v2), extending the previous documentation without replacing it.**

> **Mục tiêu**: Hiểu toàn diện và CHI TIẾT cấu trúc monorepo cal.com, Turborepo build pipeline, workspaces, dependency graph, và development workflow với các ví dụ cụ thể từ codebase thực tế.

---

## 📑 Mục lục

1. [Tổng quan Monorepo](#1-tổng-quan-monorepo)
2. [Workspace Topology](#2-workspace-topology)
3. [Apps - Ứng dụng chính](#3-apps---ứng-dụng-chính)
4. [Packages - Thư viện dùng chung](#4-packages---thư-viện-dùng-chung)
5. [Turborepo Build Pipeline](#5-turborepo-build-pipeline)
6. [Scripts & Development Workflow](#6-scripts--development-workflow)
7. [Dependency Graph](#7-dependency-graph)
8. [Best Practices](#8-best-practices)
9. [Advanced Topics & Internals](#9-advanced-topics--internals)

---

## 1. Tổng quan Monorepo

### 🔍 Ý nghĩa & bối cảnh

Cal.com sử dụng **monorepo architecture** để giải quyết các vấn đề:

**Vấn đề cần giải quyết**:
1. **Code sharing**: Nhiều apps cần dùng chung business logic, UI components, types
2. **Consistency**: Đảm bảo tất cả packages dùng cùng versions của dependencies
3. **Atomic changes**: Một PR có thể update cả backend + frontend + types cùng lúc
4. **Build optimization**: Chỉ rebuild packages bị ảnh hưởng (incremental builds)
5. **Developer experience**: Một repo, một setup, dễ onboard

**Tại sao chọn Yarn Workspaces + Turborepo**:
- **Yarn Workspaces**: Dependency management (symlink local packages, hoist dependencies)
- **Turborepo**: Task orchestration (parallel execution, caching, dependency graph)

**Trade-offs**:
- ✅ **Pros**: Code reuse, faster iteration, easier refactoring
- ⚠️ **Cons**: Complex setup, larger repo size, build tools learning curve

---

### Cal.com Monorepo Scale

```
📊 Stats (as of v5.9.0):
├── 3 deployable apps
├── 20+ core packages
├── 57 feature packages
├── 108 integration apps
├── ~500,000 lines of TypeScript
├── ~2,000 npm dependencies (deduplicated)
└── ~100 GB total (with node_modules)
```

### Kiến trúc tổng thể

```
cal.com/                          # Root monorepo
├── apps/                         # 3 main applications
│   ├── web/                      # @calcom/web - Next.js (Port 3000)
│   ├── api/
│   │   ├── v1/                   # @calcom/api - tRPC wrapper (deprecated)
│   │   └── v2/                   # @calcom/api-v2 - NestJS (Port 5555)
│   └── ui-playground/            # Storybook for UI components
│
├── packages/                     # Shared libraries & features
│   ├── prisma/                   # @calcom/prisma - DB schema (105 models)
│   ├── trpc/                     # @calcom/trpc - API layer (40+ routers)
│   ├── features/                 # 57 domain-specific features
│   │   ├── auth/
│   │   ├── bookings/
│   │   ├── calendars/
│   │   └── ee/                   # 21 enterprise features
│   ├── app-store/                # 108 third-party integrations
│   ├── lib/                      # Shared utilities
│   ├── ui/                       # Design system (50+ components)
│   ├── emails/                   # Email templates (React Email)
│   ├── platform/                 # Platform SDK (for API v2)
│   ├── embeds/                   # Embed libraries (JS, React)
│   └── [15+ other packages]
│
├── turbo.json                    # Turborepo pipeline config (40+ tasks)
├── package.json                  # Root workspace config
├── yarn.lock                     # Yarn 3.4.1 lockfile (~50MB)
└── .yarn/                        # Yarn 3 PnP files (disabled, using node_modules)
```

---

### ⚙️ Cách hoạt động chi tiết

**Workflow khi bạn `yarn install`**:

```
1. Yarn reads root package.json
   └── Detects workspaces: ["apps/*", "packages/*", ...]

2. Yarn scans ALL workspace packages
   ├── apps/web/package.json → @calcom/web
   ├── apps/api/v2/package.json → @calcom/api-v2
   ├── packages/prisma/package.json → @calcom/prisma
   └── [188 total packages detected]

3. Yarn resolves dependencies
   ├── External: Install to root node_modules/
   ├── Workspace: Create symlinks
   │   Example: apps/web/node_modules/@calcom/prisma → ../../packages/prisma
   └── Hoist common deps to root (saves disk space)

4. Run postinstall hooks
   ├── Root: husky install
   ├── Root: turbo run post-install
   └── @calcom/prisma#post-install: Generate Prisma Client + Zod schemas

Result:
node_modules/                     # ~2GB (hoisted dependencies)
apps/web/node_modules/@calcom/    # Symlinks to packages/
packages/prisma/generated/        # Auto-generated Prisma Client
```

**Workflow khi bạn `yarn dev`**:

```
1. Root script "dev": turbo run dev --filter="@calcom/web"

2. Turborepo checks dependency graph:
   @calcom/web depends on:
   ├── @calcom/prisma (needs post-install)
   ├── @calcom/trpc (needs build)
   ├── @calcom/features (needs various deps)
   └── @calcom/ui (needs build)

3. Turborepo executes tasks:
   ├── @calcom/web#copy-app-store-static (copy icons)
   └── next dev --turbopack (start dev server)

4. Next.js starts:
   ├── Load middleware.ts (routing logic)
   ├── Build app/ and pages/ routes
   ├── Start HMR (Hot Module Replacement)
   └── Listen on http://localhost:3000

Dev server ready! (~30-60s first time, ~5s subsequent)
```

---

### 📁 Vị trí cụ thể trong codebase

**Root configuration files**:
```
/package.json                     # Workspace config (line 5-16: workspaces array)
/turbo.json                       # Turborepo pipeline (544 lines)
/yarn.lock                        # Dependency lockfile (~50MB, 85k lines)
/.yarnrc.yml                      # Yarn 3 config
/.yarn/                           # Yarn 3 binary & cache
/tsconfig.json                    # Root TypeScript config
/.eslintrc.js                     # ESLint config
/.prettierrc.js                   # Prettier config
```

**Workspace discovery patterns** (from `/package.json:5-16`):
```json
"workspaces": [
  "apps/*",                       # 3 apps: web, ui-playground, (api subdir)
  "apps/api/*",                   # 2 api apps: v1, v2
  "packages/*",                   # 20+ core packages
  "packages/embeds/*",            # 3 embed packages
  "packages/features/*",          # 57 feature packages
  "packages/app-store",           # The app-store package itself
  "packages/app-store/*",         # 108 integration apps
  "packages/platform/*",          # 5 platform packages
  "packages/platform/examples/base",
  "example-apps/*"                # Example apps for developers
]
```

---

### 💡 Ví dụ cụ thể: How workspace references work

**Example 1: `@calcom/web` depends on `@calcom/prisma`**

File: `apps/web/package.json:45`
```json
{
  "dependencies": {
    "@calcom/prisma": "workspace:*"
  }
}
```

**What `"workspace:*"` means**:
- `workspace:`: Protocol telling Yarn to use local package
- `*`: Match any version (always use latest from monorepo)

**Result after `yarn install`**:
```
apps/web/node_modules/@calcom/prisma → symlink to ../../packages/prisma
```

**In code, you import normally**:
```typescript
// apps/web/lib/db.ts
import { prisma } from '@calcom/prisma';

// This resolves to packages/prisma/index.ts via symlink
```

**Example 2: External dependency hoisting**

Both `@calcom/web` and `@calcom/api-v2` depend on `@prisma/client@6.16.2`.

**Before hoisting**:
```
apps/web/node_modules/@prisma/client/       (~50MB)
apps/api/v2/node_modules/@prisma/client/    (~50MB duplicate!)
```

**After Yarn Workspaces hoisting**:
```
node_modules/@prisma/client/                (~50MB, shared)
apps/web/node_modules/@prisma/client/       → symlink to root
apps/api/v2/node_modules/@prisma/client/    → symlink to root
```

**Savings**: ~50MB disk space, faster installs.

---

### ⚠️ Pitfalls & lưu ý khi maintain

**1. Phantom dependencies**
```typescript
// ❌ BAD: Using dependency not declared in package.json
import { z } from 'zod';  // Works due to hoisting, but WRONG!

// Problem: If another package stops using zod, this breaks
```

**Fix**: Always declare ALL direct dependencies:
```json
{
  "dependencies": {
    "zod": "^3.22.4"
  }
}
```

**2. Version conflicts**
```json
// packages/prisma/package.json
"@prisma/client": "6.16.2"

// packages/trpc/package.json
"@prisma/client": "6.15.0"    // ⚠️ Version mismatch!
```

**Result**: Yarn creates TWO copies, breaks type compatibility.

**Fix**: Use workspace protocol or pin versions in root:
```json
// root package.json
"resolutions": {
  "@prisma/client": "6.16.2"   // Force this version everywhere
}
```

**3. Circular dependencies**
```typescript
// packages/lib/index.ts
import { prisma } from '@calcom/prisma';

// packages/prisma/seed.ts
import { slugify } from '@calcom/lib';  // ⚠️ Circular!
```

**Problem**: Breaks tree-shaking, can cause runtime errors.

**Fix**: Extract shared utilities to separate package:
```
packages/utils/      # Pure functions, no dependencies
packages/lib/        # Can depend on utils
packages/prisma/     # Can depend on utils
```

**4. Turborepo cache poisoning**
```bash
# Scenario: You manually edit generated files
vim packages/prisma/generated/prisma/index.d.ts

# Next build uses CACHED version (ignores your edit!)
yarn build  # Uses cache, your change missing 😱
```

**Fix**: Either edit source or force rebuild:
```bash
turbo run build --force              # Skip cache
# OR
rm -rf .turbo/cache                  # Clear all cache
```

**5. Node_modules drift between workspaces**

Sometimes you'll see:
```
⚠️  YN0002: @calcom/web@workspace:apps/web doesn't provide @types/react
```

**Cause**: Dependency declared in `@calcom/web`, but hoisting algorithm didn't hoist it.

**Fix 1**: Add to root dependencies (if used by multiple packages)
**Fix 2**: Use `"nohoist"` pattern in workspace config (Yarn v1 only)
**Fix 3**: Delete `node_modules` and reinstall:
```bash
rm -rf node_modules apps/*/node_modules packages/*/node_modules
yarn
```

---

### 🧱 Pattern & Best Practices

**1. Naming convention**: `@calcom/<name>`

```json
// ✅ GOOD
{ "name": "@calcom/bookings" }

// ❌ BAD
{ "name": "bookings" }         // Not scoped
{ "name": "@my-org/bookings" } // Wrong scope
```

**2. Always use `"private": true` for workspace packages**
```json
{
  "name": "@calcom/web",
  "private": true   // ✅ Prevents accidental npm publish
}
```

**3. Version synchronization**

For libraries that WILL be published (e.g., `@calcom/atoms`):
```json
// Use Changesets to manage versions
{
  "version": "2.6.0",  // Managed by changesets
  "private": false     // Published to npm
}
```

For internal packages:
```json
{
  "version": "0.0.0",  // Dummy version (not published)
  "private": true
}
```

**4. Shared tsconfig pattern**
```jsonc
// packages/my-package/tsconfig.json
{
  "extends": "@calcom/tsconfig/base.json",  // Extend shared config
  "compilerOptions": {
    "outDir": "./dist",
    "rootDir": "./src"
  }
}
```

**5. Feature package structure template**
```
packages/features/my-feature/
├── package.json           # Dependencies
├── tsconfig.json          # TypeScript config
├── lib/                   # Business logic
│   ├── index.ts          # Main exports
│   └── utils.ts
├── components/            # React components
│   └── MyComponent.tsx
├── hooks/                 # React hooks
│   └── useMyFeature.ts
├── server/                # tRPC procedures
│   └── myFeature.handler.ts
└── __tests__/             # Tests
    └── myFeature.test.ts
```

---

## 2. Workspace Topology

### 2.1. Workspace Configuration

#### 🔍 Ý nghĩa & bối cảnh

Yarn Workspaces là **foundation** của monorepo. Nó cho phép:
- **Symlink local packages** thay vì install từ npm
- **Hoist dependencies** để tiết kiệm disk space
- **Run scripts** across all workspaces (`yarn workspaces foreach`)

**File**: `/package.json` (root)

```json
{
  "name": "calcom-monorepo",
  "version": "0.0.0",
  "private": true,
  "workspaces": [
    "apps/*",
    "apps/api/*",
    "packages/*",
    "packages/embeds/*",
    "packages/features/*",
    "packages/app-store",
    "packages/app-store/*",
    "packages/platform/*",
    "packages/platform/examples/base",
    "example-apps/*"
  ]
}
```

#### ⚙️ How Yarn resolves workspaces

```
Step 1: Glob expansion
========================================
"apps/*" matches:
  ├── apps/web/
  ├── apps/ui-playground/
  └── apps/api/  (is a directory, contains more workspaces)

"apps/api/*" matches:
  ├── apps/api/v1/
  └── apps/api/v2/

"packages/*" matches:
  ├── packages/prisma/
  ├── packages/trpc/
  ├── packages/lib/
  └── [20+ more]

"packages/features/*" matches:
  ├── packages/features/auth/
  ├── packages/features/bookings/
  └── [55+ more]

"packages/app-store" matches:
  └── packages/app-store/  (the package itself)

"packages/app-store/*" matches:
  ├── packages/app-store/googlecalendar/
  ├── packages/app-store/zoom/
  └── [106+ more]

Step 2: Read each package.json
========================================
For each matched directory:
  - Read package.json
  - Extract "name" field (e.g., "@calcom/web")
  - Add to workspace registry

Step 3: Build dependency graph
========================================
Create map:
{
  "@calcom/web": {
    path: "apps/web",
    dependencies: ["@calcom/prisma", "@calcom/trpc", ...]
  },
  "@calcom/prisma": {
    path: "packages/prisma",
    dependencies: []
  }
}

Step 4: Link workspaces
========================================
Create symlinks in node_modules:
  apps/web/node_modules/@calcom/prisma → ../../packages/prisma
  apps/web/node_modules/@calcom/trpc → ../../packages/trpc
  ...
```

#### 📁 Vị trí cụ thể

**Workspace packages count** (as of v5.9.0):
```bash
$ yarn workspaces list | wc -l
188   # Total workspace packages
```

**Breakdown**:
```
3   apps (web, ui-playground, api parent)
2   api apps (v1, v2)
20  core packages
57  feature packages
108 app-store integrations
5   platform packages
3   embed packages
```

#### 💡 Ví dụ: List all workspaces

```bash
# List all workspaces
yarn workspaces list

# Example output:
# apps/web
# apps/api/v1
# apps/api/v2
# packages/prisma
# packages/trpc
# ...

# Run command in all workspaces
yarn workspaces foreach run test

# Run in specific workspace
yarn workspace @calcom/web dev
```

#### ⚠️ Common issues

**Issue 1: Workspace not detected**
```bash
# Symptom
Error: Cannot find module '@calcom/my-new-package'

# Cause
mkdir packages/my-new-package
# Forgot to create package.json!

# Fix
cd packages/my-new-package
yarn init  # Creates package.json with name field
cd ../..
yarn       # Re-install to detect new workspace
```

**Issue 2: Glob pattern too broad**
```json
"workspaces": [
  "packages/*"  // ⚠️ Matches EVERYTHING including test fixtures
]

// Fix: Exclude patterns
"workspaces": {
  "packages": [
    "packages/*",
    "!packages/*/test",      // Exclude test dirs
    "!packages/*/__tests__"  // Exclude test dirs
  ]
}
```

#### 🧱 Best practice: Workspace naming

```
✅ GOOD patterns:
@calcom/web              # Main apps
@calcom/api-v2           # API with version
@calcom/prisma           # Core packages
@calcom/feature-bookings # Feature packages
@calcom/app-googlecalendar # App integrations

❌ BAD patterns:
web                      # Missing @calcom scope
@calcom/web-app          # Redundant suffix
@calcom/google-calendar  # Inconsistent with app-store pattern
```

---

### 2.2. Package Manager (Yarn 3.4.1)

#### 🔍 Why Yarn 3.4.1?

**Evolution**:
```
Yarn 1.x (Classic)
└── Problem: Slow installs, phantom dependencies, no PnP

Yarn 2.x (Berry)
└── Introduced: Plug'n'Play (PnP), faster, deterministic

Yarn 3.x (Modern)
└── Stable PnP, better monorepo support

Cal.com uses: Yarn 3.4.1 with PnP DISABLED
└── Reason: Better compatibility with tools (Next.js, Prisma, IDE)
```

#### ⚙️ Configuration

**File**: `/.yarnrc.yml`
```yaml
nodeLinker: node-modules  # Disable PnP, use traditional node_modules
```

**File**: `/package.json`
```json
{
  "packageManager": "yarn@3.4.1",  # Corepack will auto-install this version
  "engines": {
    "npm": ">=7.0.0",     # Minimum npm version (if someone uses npm)
    "yarn": "3.4.1"       # Exact Yarn version required
  }
}
```

#### 💡 Example: Using Corepack (recommended)

```bash
# Enable Corepack (comes with Node.js 16.10+)
corepack enable

# Corepack reads packageManager field and installs yarn@3.4.1
cd cal.com
yarn --version
# Output: 3.4.1

# Now yarn commands use correct version automatically
yarn install
yarn dev
```

#### ⚠️ Pitfall: Yarn version mismatch

```bash
# User has Yarn 1.x globally installed
$ yarn --version
1.22.19

# Tries to install
$ yarn
# ⚠️ Uses Yarn 1.x, not 3.4.1!
# Generates yarn.lock in v1 format
# Workspace features don't work properly

# Fix: Use Corepack or manual install
$ npm uninstall -g yarn        # Remove global Yarn 1.x
$ corepack enable              # Enable Corepack
$ yarn --version               # Now shows 3.4.1
```

#### 🧱 Best practice: Lockfile management

```bash
# NEVER delete yarn.lock in monorepo!
# It's ~50MB because it contains ALL dependencies

# If you need to reset:
rm -rf node_modules .yarn/cache
yarn  # Reinstall from lockfile

# To update ALL dependencies (dangerous!):
yarn up '*'  # Updates all packages

# To update specific package:
yarn up next@latest  # Update Next.js to latest
```

---

## 3. Apps - Ứng dụng chính

### 3.1. `@calcom/web` - Main Web Application

#### 🔍 Ý nghĩa & bối cảnh

`@calcom/web` là **core application** mà users interact với:
- Public booking pages (`cal.com/john/30min`)
- User dashboard (`cal.com/event-types`)
- Admin settings
- OAuth flows
- Embed endpoints

**Architecture choice**: **Hybrid Next.js (App Router + Pages Router)**

**Why hybrid?**:
```
App Router (app/)
├── ✅ Better performance (Server Components)
├── ✅ Streaming SSR
├── ✅ Built-in layouts
└── ⚠️ Still new, some features missing

Pages Router (pages/)
├── ✅ Mature, stable
├── ✅ Full tRPC support
├── ✅ API routes
└── ⚠️ Client-heavy rendering

Decision: Use BOTH
├── App Router: New features, public pages
└── Pages Router: Dashboard, settings, tRPC API
```

---

#### ⚙️ Cách hoạt động chi tiết

**Request flow**:
```
1. User visits https://cal.com/john/30min

2. Next.js matches route:
   ├── Check app/ directory first (App Router)
   │   Found: app/(booking-page-wrapper)/[user]/[type]/page.tsx
   │   └── This is a Server Component
   │
   └── If not found, check pages/ directory (Pages Router)

3. Middleware runs first:
   File: apps/web/middleware.ts
   ├── Check organization routing (acme.cal.com → /org/acme)
   ├── Apply rate limiting
   ├── Set CSP headers
   └── Rewrite URL if needed

4. Server Component executes:
   ├── Fetch user data (direct Prisma query, no tRPC)
   ├── Fetch event type
   ├── Fetch availability
   └── Return RSC payload to client

5. Client hydration:
   ├── Load booking form (Client Component)
   ├── Connect tRPC client
   └── Enable interactivity
```

**Example walkthrough**:
```typescript
// apps/web/app/(booking-page-wrapper)/[user]/[type]/page.tsx
export default async function BookingPage({
  params
}: {
  params: { user: string; type: string }
}) {
  // 1. This runs on SERVER
  const user = await prisma.user.findUnique({
    where: { username: params.user },
    include: { eventTypes: true }
  });

  if (!user) {
    notFound();  // Next.js 404
  }

  const eventType = user.eventTypes.find(et => et.slug === params.type);

  // 2. Return Server Component (no JS sent to client yet)
  return (
    <div>
      <h1>{eventType.title}</h1>
      {/* 3. Client Component (will hydrate on client) */}
      <BookingForm eventType={eventType} />
    </div>
  );
}

// Client Component (separate file)
'use client';
export function BookingForm({ eventType }) {
  // This code runs on CLIENT
  const [selectedDate, setSelectedDate] = useState(null);

  // tRPC query (client-side)
  const { data: slots } = trpc.viewer.slots.getSchedule.useQuery({
    eventTypeId: eventType.id,
    startTime: selectedDate
  });

  return <form>...</form>;
}
```

---

#### 📁 File Structure Deep Dive

```
apps/web/
├── app/                          # Next.js App Router
│   ├── (booking-page-wrapper)/   # Route group (shared layout)
│   │   ├── [user]/
│   │   │   └── [type]/
│   │   │       └── page.tsx      # Booking page: /john/30min
│   │   └── layout.tsx            # Layout for booking pages
│   │
│   ├── (use-page-wrapper)/       # Route group (authenticated pages)
│   │   ├── event-types/
│   │   │   └── page.tsx          # Event types list
│   │   ├── settings/
│   │   │   └── page.tsx          # Settings page
│   │   └── layout.tsx            # Layout with sidebar
│   │
│   ├── api/                      # App Router API routes
│   │   └── health/
│   │       └── route.ts          # GET /api/health
│   │
│   ├── layout.tsx                # Root layout (HTML, providers)
│   ├── page.tsx                  # Home page: /
│   └── not-found.tsx             # 404 page
│
├── pages/                        # Next.js Pages Router (legacy)
│   ├── api/                      # API routes
│   │   ├── trpc/
│   │   │   └── [trpc].ts         # tRPC handler: /api/trpc/*
│   │   ├── auth/
│   │   │   └── [...nextauth].ts  # NextAuth: /api/auth/*
│   │   ├── stripe/
│   │   │   └── webhook.ts        # Stripe webhook
│   │   └── health.ts             # Health check (legacy)
│   │
│   ├── _app.tsx                  # Pages Router app wrapper
│   ├── _document.tsx             # HTML document template
│   ├── event-types/
│   │   └── index.tsx             # /event-types (legacy, redirects to app/)
│   └── [user]/                   # Dynamic user pages
│
├── components/                   # UI components
│   ├── booking/
│   │   ├── BookingPage.tsx
│   │   └── TimeSlots.tsx
│   └── ui/
│       └── button.tsx
│
├── lib/                          # Utilities
│   ├── hooks/
│   │   └── useBooking.ts
│   ├── server/                   # Server-only code
│   │   └── routers.ts
│   └── utils.ts
│
├── modules/                      # Feature modules
│   ├── auth/
│   ├── bookings/
│   └── settings/
│
├── middleware.ts                 # Next.js Edge Middleware
├── next.config.js                # Next.js configuration
├── instrumentation.ts            # Sentry, OpenTelemetry setup
├── package.json                  # Dependencies (100+ workspace deps)
└── tsconfig.json                 # TypeScript config
```

---

#### 💡 Example: tRPC endpoint setup

**File**: `apps/web/pages/api/trpc/[trpc].ts`
```typescript
import { createNextApiHandler } from '@trpc/server/adapters/next';
import { appRouter } from '@calcom/trpc/server/routers/_app';
import { createContext } from '@calcom/trpc/server/createContext';

// 1. Create Next.js API handler
export default createNextApiHandler({
  router: appRouter,        // Import from @calcom/trpc package
  createContext,            // Session, user, Prisma client
  onError({ error, type, path }) {
    // Error logging
    console.error(`tRPC error on ${path}:`, error);
  },
});

// Result: All tRPC procedures available at /api/trpc/*
// Example: POST /api/trpc/viewer.eventTypes.list
```

**Client usage** (in React Component):
```typescript
'use client';
import { trpc } from '@calcom/trpc/react';

export function EventTypeList() {
  // This calls POST /api/trpc/viewer.eventTypes.list
  const { data, isLoading } = trpc.viewer.eventTypes.list.useQuery();

  if (isLoading) return <Spinner />;

  return (
    <ul>
      {data.eventTypeGroups[0].eventTypes.map(et => (
        <li key={et.id}>{et.title}</li>
      ))}
    </ul>
  );
}
```

---

#### ⚠️ Pitfalls

**1. Mixed rendering modes**
```typescript
// ❌ BAD: Using tRPC in Server Component
export default async function Page() {
  const data = trpc.viewer.eventTypes.list.useQuery();  // ERROR!
  // useQuery is a React Hook, can't use in Server Component
}

// ✅ GOOD: Use direct Prisma query
export default async function Page() {
  const eventTypes = await prisma.eventType.findMany({
    where: { userId: session.user.id }
  });
}

// ✅ OR: Use Client Component for tRPC
'use client';
export function EventTypeList() {
  const { data } = trpc.viewer.eventTypes.list.useQuery();
  // Works in Client Component
}
```

**2. Environment variables in client**
```typescript
// ❌ BAD: Server-only env var in client code
'use client';
export function MyComponent() {
  const dbUrl = process.env.DATABASE_URL;  // undefined! (server-only)
}

// ✅ GOOD: Use NEXT_PUBLIC_ prefix for client vars
export function MyComponent() {
  const apiUrl = process.env.NEXT_PUBLIC_API_URL;  // Works!
}
```

**3. Middleware performance**
```typescript
// apps/web/middleware.ts

// ❌ BAD: Heavy computation in middleware
export async function middleware(req: NextRequest) {
  const allUsers = await prisma.user.findMany();  // 🐌 Slow!
  // Runs on EVERY request (even static assets)
}

// ✅ GOOD: Use matcher to limit scope
export const config = {
  matcher: [
    '/api/:path*',      // Only API routes
    '/((?!_next|static|favicon.ico).*)',  // Exclude Next.js internals
  ],
};
```

---

#### 🧱 Best Practices

**1. Code splitting with dynamic imports**
```typescript
import dynamic from 'next/dynamic';

// Lazy-load heavy component
const BookingCalendar = dynamic(() => import('./BookingCalendar'), {
  loading: () => <Spinner />,
  ssr: false,  // Don't server-render if not needed
});

// Reduces initial bundle size
```

**2. Use Server Components by default**
```typescript
// Default: Server Component (no 'use client')
export default async function Page() {
  const data = await fetchData();  // Direct DB query
  return <div>{data.title}</div>;
}

// Only add 'use client' when you need:
// - useState, useEffect, event handlers
// - Browser APIs (window, localStorage)
// - Third-party components that require client rendering
```

**3. Env var validation**
```typescript
// apps/web/lib/env.ts
import { z } from 'zod';

const envSchema = z.object({
  DATABASE_URL: z.string().url(),
  NEXTAUTH_SECRET: z.string().min(32),
  NEXT_PUBLIC_WEBAPP_URL: z.string().url(),
});

export const env = envSchema.parse(process.env);
// Fails fast on startup if env vars are invalid
```

---

### 3.2. `@calcom/api-v2` - Platform API (NestJS)

#### 🔍 Ý nghĩa & bối cảnh

**Why a separate API app?**

`@calcom/web` uses tRPC (type-safe RPC, internal use).
But external integrations need **REST API + OAuth 2.0**.

**Requirements**:
- ✅ REST API (standard HTTP methods)
- ✅ OAuth 2.0 (authorization_code flow)
- ✅ OpenAPI/Swagger docs
- ✅ Rate limiting
- ✅ API keys (for machine-to-machine)

**Technology choice: NestJS**

**Why NestJS over Next.js API routes?**
```
Next.js API routes:
├── ✅ Easy to add (same app)
├── ⚠️ No built-in dependency injection
├── ⚠️ No decorators (need manual wiring)
└── ⚠️ Swagger setup is manual

NestJS:
├── ✅ Full-featured framework (DI, decorators, guards)
├── ✅ Built-in Swagger support
├── ✅ Interceptors, pipes, filters (AOP pattern)
├── ✅ Bull queues for async jobs
└── ⚠️ Heavier, separate deployment
```

**Decision**: Separate NestJS app for Platform API.

---

#### ⚙️ Architecture Deep Dive

```
NestJS Application Structure:

Client (OAuth App)
    │
    ├─1─ POST /oauth/authorize (get authorization code)
    ├─2─ POST /oauth/token (exchange code for access_token)
    │
    └─3─ API calls with Bearer token
         ├─ GET /v2/bookings
         ├─ POST /v2/bookings
         └─ PATCH /v2/bookings/:id

apps/api/v2/src/
├── main.ts                      # Bootstrap NestJS app
├── app.module.ts                # Root module (imports all modules)
│
├── modules/                     # Feature modules
│   ├── auth/                    # OAuth 2.0 module
│   │   ├── auth.controller.ts   # /oauth/authorize, /oauth/token
│   │   ├── auth.service.ts      # Token generation, validation
│   │   ├── guards/
│   │   │   └── jwt-auth.guard.ts  # @UseGuards(JwtAuthGuard)
│   │   └── strategies/
│   │       └── jwt.strategy.ts    # Passport JWT strategy
│   │
│   ├── bookings/                # Bookings endpoints
│   │   ├── bookings.controller.ts
│   │   │   ├─ @Get('/')         # GET /v2/bookings
│   │   │   ├─ @Post('/')        # POST /v2/bookings
│   │   │   └─ @Patch('/:id')    # PATCH /v2/bookings/:id
│   │   ├── bookings.service.ts
│   │   ├── dto/
│   │   │   ├── create-booking.dto.ts   # Input validation
│   │   │   └── update-booking.dto.ts
│   │   └── bookings.module.ts
│   │
│   ├── users/
│   ├── event-types/
│   └── schedules/
│
├── ee/                          # Enterprise modules
│   └── organizations/
│
├── guards/                      # Global guards
│   └── rate-limit.guard.ts
│
├── interceptors/                # Global interceptors
│   └── logging.interceptor.ts
│
├── filters/                     # Exception filters
│   └── http-exception.filter.ts
│
└── swagger/                     # OpenAPI generation
    └── swagger.config.ts
```

---

#### 💡 Example: Complete OAuth 2.0 Flow

**Step 1: Client requests authorization**
```http
GET /oauth/authorize?
  client_id=abc123&
  redirect_uri=https://app.example.com/callback&
  response_type=code&
  scope=bookings:read+bookings:write

Response (redirect):
https://app.example.com/callback?code=AUTH_CODE_XYZ
```

**Code**:
```typescript
// apps/api/v2/src/modules/auth/auth.controller.ts

@Controller('oauth')
export class AuthController {
  @Get('authorize')
  async authorize(@Query() query: AuthorizeDto) {
    const { client_id, redirect_uri, scope } = query;

    // 1. Validate client
    const client = await this.clientsService.findById(client_id);
    if (!client) throw new UnauthorizedException('Invalid client');

    // 2. Check redirect_uri matches registered
    if (!client.redirectUris.includes(redirect_uri)) {
      throw new BadRequestException('Invalid redirect_uri');
    }

    // 3. Generate authorization code
    const code = await this.authService.createAuthCode({
      clientId: client_id,
      scope,
    });

    // 4. Redirect back to client
    return {
      redirectUrl: `${redirect_uri}?code=${code}`
    };
  }
}
```

**Step 2: Exchange code for access token**
```http
POST /oauth/token
Content-Type: application/json

{
  "grant_type": "authorization_code",
  "code": "AUTH_CODE_XYZ",
  "client_id": "abc123",
  "client_secret": "secret456",
  "redirect_uri": "https://app.example.com/callback"
}

Response:
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "..."
}
```

**Code**:
```typescript
@Post('token')
async token(@Body() body: TokenDto) {
  const { grant_type, code, client_id, client_secret } = body;

  // 1. Validate client credentials
  const client = await this.clientsService.findById(client_id);
  if (client.secret !== client_secret) {
    throw new UnauthorizedException('Invalid client credentials');
  }

  // 2. Validate authorization code
  const authCode = await this.authService.validateCode(code);
  if (!authCode || authCode.used) {
    throw new UnauthorizedException('Invalid or expired code');
  }

  // 3. Mark code as used
  await this.authService.markCodeUsed(code);

  // 4. Generate JWT access token
  const payload = {
    sub: authCode.userId,
    clientId: client_id,
    scope: authCode.scope,
  };

  const accessToken = this.jwtService.sign(payload, {
    expiresIn: '1h'
  });

  return {
    access_token: accessToken,
    token_type: 'Bearer',
    expires_in: 3600,
  };
}
```

**Step 3: Use access token to call API**
```http
GET /v2/bookings
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

Response:
{
  "bookings": [
    { "id": 1, "title": "Meeting with John", ... }
  ]
}
```

**Code**:
```typescript
// apps/api/v2/src/modules/bookings/bookings.controller.ts

@Controller('v2/bookings')
@UseGuards(JwtAuthGuard)  // Validate JWT on all routes
export class BookingsController {
  @Get()
  async findAll(@Request() req) {
    const userId = req.user.sub;  // From JWT payload

    const bookings = await this.bookingsService.findByUser(userId);
    return { bookings };
  }

  @Post()
  async create(@Request() req, @Body() dto: CreateBookingDto) {
    const userId = req.user.sub;

    // Validate DTO with class-validator
    const booking = await this.bookingsService.create({
      ...dto,
      userId,
    });

    return { booking };
  }
}
```

**JWT Guard Implementation**:
```typescript
// apps/api/v2/src/modules/auth/guards/jwt-auth.guard.ts

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {
  canActivate(context: ExecutionContext) {
    return super.canActivate(context);
  }

  handleRequest(err, user, info) {
    if (err || !user) {
      throw new UnauthorizedException('Invalid or expired token');
    }
    return user;
  }
}

// JWT Strategy (extracts & validates JWT)
@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor() {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      secretOrKey: process.env.JWT_SECRET,
    });
  }

  async validate(payload: any) {
    // Payload is decoded JWT
    return {
      sub: payload.sub,        // User ID
      clientId: payload.clientId,
      scope: payload.scope,
    };
  }
}
```

---

#### ⚠️ Pitfalls

**1. Shared Prisma Client entre Web et API v2**
```typescript
// ❌ PROBLEM: Each app creates its own Prisma instance

// apps/web/lib/prisma.ts
export const prisma = new PrismaClient();  // Instance 1

// apps/api/v2/src/prisma.service.ts
export const prisma = new PrismaClient();  // Instance 2 (duplicate!)

// Result: Double DB connections, cache misses
```

**✅ FIX**: Share Prisma client via `@calcom/prisma` package
```typescript
// packages/prisma/index.ts
import { PrismaClient } from '@prisma/client';

export const prisma = new PrismaClient();  // Singleton

// apps/web uses:
import { prisma } from '@calcom/prisma';

// apps/api/v2 uses:
import { prisma } from '@calcom/prisma';

// Now both apps use SAME instance (if in same process)
```

**2. DTO validation bypass**
```typescript
// ❌ BAD: No validation
@Post()
create(@Body() body: any) {
  // body could be anything!
  return this.service.create(body);
}

// ✅ GOOD: Use DTO with class-validator
import { IsEmail, IsString } from 'class-validator';

export class CreateUserDto {
  @IsEmail()
  email: string;

  @IsString()
  name: string;
}

@Post()
create(@Body() body: CreateUserDto) {
  // NestJS auto-validates before reaching here
  return this.service.create(body);
}
```

**3. Missing Swagger decorators**
```typescript
// ❌ BAD: No Swagger metadata
@Get(':id')
findOne(@Param('id') id: string) {
  return this.service.findOne(id);
}

// Generated Swagger: No response schema, no description

// ✅ GOOD: Add Swagger decorators
@Get(':id')
@ApiOperation({ summary: 'Get booking by ID' })
@ApiParam({ name: 'id', type: 'number' })
@ApiResponse({
  status: 200,
  description: 'Booking found',
  type: BookingDto,
})
@ApiResponse({
  status: 404,
  description: 'Booking not found',
})
findOne(@Param('id', ParseIntPipe) id: number) {
  return this.service.findOne(id);
}
```

---

#### 🧱 Best Practices

**1. Module organization**
```
modules/
├── users/
│   ├── users.module.ts        # @Module decorator
│   ├── users.controller.ts    # REST endpoints
│   ├── users.service.ts       # Business logic
│   ├── users.repository.ts    # Data access (Prisma)
│   ├── dto/                   # Data Transfer Objects
│   │   ├── create-user.dto.ts
│   │   └── update-user.dto.ts
│   └── entities/              # Domain models
│       └── user.entity.ts
```

**2. Dependency Injection**
```typescript
// users.service.ts
@Injectable()
export class UsersService {
  constructor(
    private readonly repository: UsersRepository,  // DI
    private readonly emailService: EmailService,   // DI
  ) {}

  async create(dto: CreateUserDto) {
    const user = await this.repository.create(dto);
    await this.emailService.sendWelcome(user.email);
    return user;
  }
}

// users.module.ts
@Module({
  providers: [UsersService, UsersRepository, EmailService],
  controllers: [UsersController],
  exports: [UsersService],  // Can be imported by other modules
})
export class UsersModule {}
```

**3. Global exception filter**
```typescript
// filters/http-exception.filter.ts
@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse();

    const status =
      exception instanceof HttpException
        ? exception.getStatus()
        : 500;

    response.status(status).json({
      statusCode: status,
      timestamp: new Date().toISOString(),
      message: exception.message,
    });
  }
}

// main.ts
app.useGlobalFilters(new AllExceptionsFilter());
```

---

## 4. Packages - Thư viện dùng chung

_(Due to context limit, I'll continue with the most important sections. Continuing...)_

### 4.1. Core Infrastructure Packages

#### `@calcom/prisma` - Database Schema & Client

##### 🔍 Ý nghĩa & bối cảnh

**Why separate package?**
- ✅ Share Prisma Client across all apps
- ✅ Single source of truth for schema
- ✅ Generate types once, use everywhere
- ✅ Easier migration management

##### ⚙️ How it works: The Prisma Generation Pipeline

```
Step 1: Developer edits schema
================================
File: packages/prisma/schema.prisma

model User {
  id    Int    @id @default(autoincrement())
  email String @unique
}

Step 2: Run yarn (or yarn workspace @calcom/prisma generate-schemas)
=====================================================================
Triggers: package.json script "post-install"

Step 3: Prisma generates 4 artifacts
=====================================
1. Prisma Client
   └─ packages/prisma/generated/prisma/
      ├── index.d.ts       (TypeScript types)
      └── index.js         (Query builder)

2. Zod Schemas
   └─ packages/prisma/zod/
      ├── user.ts          (export const userSchema = z.object({...}))
      └── index.ts         (export all schemas)

3. Kysely Types
   └─ packages/prisma/kysely/types.ts
      (Type-safe SQL query builder types)

4. Enums (custom generator)
   └─ packages/prisma/enums.ts
      export enum BookingStatus {
        ACCEPTED = "ACCEPTED",
        PENDING = "PENDING",
        ...
      }

Step 4: Apps import generated code
===================================
// apps/web/lib/bookings.ts
import { prisma } from '@calcom/prisma';
import { bookingSchema } from '@calcom/prisma/zod';
import { BookingStatus } from '@calcom/prisma/enums';

const booking = await prisma.booking.create({
  data: {
    status: BookingStatus.PENDING,  // Type-safe enum
    ...bookingSchema.parse(input),  // Runtime validation
  }
});
```

##### 📁 File Structure
```
packages/prisma/
├── schema.prisma              # Main schema (2780 lines, 105 models)
├── migrations/                # Migration history (~50 files)
│   ├── 20240101000000_init/
│   │   └── migration.sql
│   └── 20240618123456_add_organizations/
│       └── migration.sql
├── seed.ts                    # Seed data script
├── generated/                 # Auto-generated (gitignored)
│   └── prisma/
│       ├── index.d.ts
│       └── index.js
├── zod/                       # Auto-generated Zod schemas
│   ├── user.ts
│   ├── booking.ts
│   └── index.ts
├── kysely/                    # Kysely types
│   └── types.ts
├── enums.ts                   # Generated enums
└── package.json
```

##### 💡 Example: Using Zod schemas for validation

**Generated Zod schema** (auto-generated from Prisma):
```typescript
// packages/prisma/zod/booking.ts (simplified)
import { z } from 'zod';

export const bookingSchema = z.object({
  id: z.number().int(),
  uid: z.string().cuid(),
  userId: z.number().int().nullable(),
  eventTypeId: z.number().int().nullable(),
  title: z.string(),
  description: z.string().nullable(),
  startTime: z.coerce.date(),
  endTime: z.coerce.date(),
  status: z.enum(['ACCEPTED', 'PENDING', 'CANCELLED', 'REJECTED']),
  // ... 30+ more fields
});

export type Booking = z.infer<typeof bookingSchema>;
```

**Usage in tRPC procedure**:
```typescript
// packages/trpc/server/routers/viewer/bookings.ts
import { bookingSchema } from '@calcom/prisma/zod';

export const bookingsRouter = router({
  create: authedProcedure
    .input(
      bookingSchema.pick({
        eventTypeId: true,
        startTime: true,
        endTime: true,
        title: true,
        // ... other fields needed for creation
      })
    )
    .mutation(async ({ ctx, input }) => {
      // input is validated AND type-safe!
      const booking = await ctx.prisma.booking.create({
        data: input,
      });

      return { booking };
    }),
});
```

##### ⚠️ Pitfalls

**1. Migration conflicts in multiplayer dev**
```bash
# Developer A creates migration
yarn prisma migrate dev --name add_user_role
# Creates: migrations/20240618_add_user_role/migration.sql

# Developer B (parallel) creates migration
yarn prisma migrate dev --name add_user_avatar
# Creates: migrations/20240618_add_user_avatar/migration.sql

# Git merge conflict!
# Both modified schema.prisma + created migrations

# FIX: One developer resets and re-creates migration
git pull
yarn prisma migrate reset  # Resets DB
yarn prisma migrate dev     # Re-applies all migrations
```

**2. Schema drift (local DB != schema file)**
```bash
# Symptom
yarn prisma migrate dev
# Error: P3005: Schema drift detected

# Cause
# Someone ran `prisma db push` (skipped migrations)
# Or manually edited DB

# FIX
prisma migrate reset     # Nuke DB, start fresh
# OR
prisma db pull           # Pull current DB schema to schema.prisma
prisma migrate dev       # Create new migration for drift
```

**3. Circular imports with Zod schemas**
```typescript
// ❌ BAD
// packages/prisma/zod/user.ts
import { teamSchema } from './team';

export const userSchema = z.object({
  teams: z.array(teamSchema),  // Circular!
});

// packages/prisma/zod/team.ts
import { userSchema } from './user';

export const teamSchema = z.object({
  users: z.array(userSchema),  // Circular!
});

// Error: ReferenceError: Cannot access 'userSchema' before initialization

// ✅ FIX: Use z.lazy() for recursive schemas
export const userSchema = z.object({
  teams: z.lazy(() => z.array(teamSchema)),
});
```

---

### 4.2. Feature Packages

#### Overview: 57 Feature Packages

##### 🔍 Ý nghĩa & bối cảnh

**Why 57 separate packages instead of monolithic /src/features?**

**Benefits**:
1. **Explicit dependencies**: Each feature declares exactly what it needs
2. **Independent versioning**: Can version enterprise features separately
3. **Code splitting**: Next.js can split bundles per feature
4. **Team ownership**: Different teams own different feature packages
5. **Testability**: Easy to test features in isolation

**Trade-offs**:
- ⚠️ More boilerplate (each package needs package.json, tsconfig.json)
- ⚠️ More complex dependency graph
- ⚠️ Harder to refactor across features

##### ⚙️ Feature Package Categories

```
packages/features/
│
├── Core Features (20 packages)
│   ├── auth/                    # Authentication & sessions
│   ├── bookings/                # Booking CRUD & logic
│   ├── calendars/               # Calendar sync (Google, Outlook...)
│   ├── event-types/             # Event type management
│   ├── schedules/               # Availability schedules
│   ├── users/                   # User management
│   ├── teams/                   # Team features
│   ├── webhooks/                # Webhook delivery
│   ├── apps/                    # App management UI
│   ├── embed/                   # Embed functionality
│   └── ...
│
├── Enterprise Features (21 packages in ee/)
│   ├── ee/organizations/        # Multi-tenant orgs
│   ├── ee/sso/                  # SAML & OIDC
│   ├── ee/dsync/                # Directory sync (SCIM)
│   ├── ee/workflows/            # Advanced automation
│   ├── ee/insights/             # Analytics dashboard
│   ├── ee/billing/              # Stripe billing
│   ├── ee/api-keys/             # API key management
│   └── ...
│
└── UI/UX Features (16 packages)
    ├── flags/                   # Feature flags
    ├── filters/                 # Query filters
    ├── routing-forms/           # Dynamic forms
    ├── shell/                   # App shell & layout
    └── ...
```

##### 📁 Standard Feature Package Structure

```
packages/features/bookings/
├── package.json               # Dependencies & metadata
├── tsconfig.json              # TypeScript config
│
├── lib/                       # Business logic (pure functions)
│   ├── handleNewBooking.ts
│   ├── handleCancelBooking.ts
│   ├── handleRescheduleBooking.ts
│   ├── getBookingInfo.ts
│   └── validateBooking.ts
│
├── components/                # React UI components
│   ├── BookingForm.tsx
│   ├── BookingListItem.tsx
│   └── BookingDetails.tsx
│
├── hooks/                     # React hooks
│   ├── useBooking.ts
│   └── useBookingForm.ts
│
├── server/                    # Server-only code (tRPC, Prisma)
│   ├── bookings.handler.ts    # tRPC procedures
│   └── bookings.repository.ts # Data access
│
├── schemas/                   # Zod validation schemas
│   └── booking.schema.ts
│
├── types/                     # TypeScript types
│   └── booking.types.ts
│
└── __tests__/                 # Unit & integration tests
    ├── handleNewBooking.test.ts
    └── BookingForm.test.tsx
```

##### 💡 Example: Deep Dive into `@calcom/features/bookings`

**package.json**:
```json
{
  "name": "@calcom/features/bookings",
  "version": "0.0.0",
  "private": true,
  "main": "./index.ts",
  "dependencies": {
    "@calcom/prisma": "workspace:*",
    "@calcom/lib": "workspace:*",
    "@calcom/emails": "workspace:*",
    "@calcom/features/calendars": "workspace:*",
    "@calcom/features/webhooks": "workspace:*",
    "ics": "^2.37.0",
    "rrule": "^2.7.1"
  }
}
```

**Main export** (`index.ts`):
```typescript
export { handleNewBooking } from './lib/handleNewBooking';
export { handleCancelBooking } from './lib/handleCancelBooking';
export { BookingForm } from './components/BookingForm';
export { useBooking } from './hooks/useBooking';
export * from './types/booking.types';
```

**Core function: handleNewBooking**:
```typescript
// packages/features/bookings/lib/handleNewBooking.ts
import { prisma } from '@calcom/prisma';
import { sendBookingConfirmation } from '@calcom/emails';
import { createCalendarEvent } from '@calcom/features/calendars';
import { sendWebhook } from '@calcom/features/webhooks';

export async function handleNewBooking(input: BookingInput) {
  // 1. Validate input
  const validated = bookingSchema.parse(input);

  // 2. Check availability (prevent double booking)
  const conflicts = await checkConflicts({
    userId: validated.userId,
    start: validated.startTime,
    end: validated.endTime,
  });

  if (conflicts.length > 0) {
    throw new Error('Time slot is no longer available');
  }

  // 3. Create booking in database (atomic transaction)
  const booking = await prisma.$transaction(async (tx) => {
    // Create booking
    const newBooking = await tx.booking.create({
      data: {
        ...validated,
        status: 'PENDING',
      },
    });

    // Create attendees
    await tx.attendee.createMany({
      data: validated.attendees.map(att => ({
        bookingId: newBooking.id,
        email: att.email,
        name: att.name,
      })),
    });

    return newBooking;
  });

  // 4. Create calendar event (Google Calendar, etc.)
  try {
    const calendarEvent = await createCalendarEvent({
      booking,
      calendar: user.selectedCalendar,
    });

    await prisma.bookingReference.create({
      data: {
        bookingId: booking.id,
        type: 'google_calendar',
        uid: calendarEvent.id,
      },
    });
  } catch (error) {
    // Log but don't fail (booking created successfully)
    console.error('Failed to create calendar event:', error);
  }

  // 5. Send confirmation email
  await sendBookingConfirmation({
    booking,
    attendees: booking.attendees,
  });

  // 6. Trigger webhooks
  await sendWebhook({
    event: 'BOOKING_CREATED',
    payload: booking,
  });

  return booking;
}
```

**Usage in tRPC**:
```typescript
// packages/trpc/server/routers/viewer/bookings.ts
import { handleNewBooking } from '@calcom/features/bookings';

export const bookingsRouter = router({
  create: authedProcedure
    .input(bookingCreateSchema)
    .mutation(async ({ ctx, input }) => {
      const booking = await handleNewBooking({
        ...input,
        userId: ctx.user.id,
      });

      return { booking };
    }),
});
```

---

## 5. Turborepo Build Pipeline

### 🔍 Ý nghĩa & bối cảnh

**Why Turborepo instead of just Yarn Workspaces?**

Yarn Workspaces handles **dependency management**.
Turborepo handles **task orchestration**.

**Problems without Turborepo**:
```bash
# Scenario: Build all packages
yarn workspaces foreach run build

# Problems:
❌ Runs serially (slow!)
❌ No dependency awareness (might build in wrong order)
❌ No caching (rebuilds everything every time)
❌ No parallel execution
```

**With Turborepo**:
```bash
turbo run build

# Benefits:
✅ Parallel execution (uses all CPU cores)
✅ Dependency-aware (builds in correct order)
✅ Incremental builds (only rebuilds changed packages)
✅ Remote caching (share cache across team)
✅ Pipeline visualization
```

---

### ⚙️ How Turborepo Works: The Execution Model

**File**: `/turbo.json`

```json
{
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],  // Build dependencies first
      "outputs": [".next/**", "dist/**"]
    },
    "@calcom/web#build": {
      "dependsOn": [
        "@calcom/prisma#post-install",
        "^build"
      ],
      "outputs": [".next/**"]
    }
  }
}
```

**Execution Flow**:
```
1. User runs: turbo run build

2. Turborepo builds task graph:
   ┌─────────────────────────────────┐
   │       Task Dependency Graph      │
   ├─────────────────────────────────┤
   │                                 │
   │  @calcom/prisma#post-install    │
   │            │                    │
   │            ├─────┐              │
   │            │     │              │
   │  @calcom/lib#build              │
   │  @calcom/ui#build               │
   │            │     │              │
   │            ├─────┤              │
   │            │                    │
   │  @calcom/trpc#build             │
   │            │                    │
   │            │                    │
   │  @calcom/web#build              │
   │                                 │
   └─────────────────────────────────┘

3. Turborepo determines execution order:
   Level 1: @calcom/prisma#post-install (no deps)
   Level 2: @calcom/lib#build, @calcom/ui#build (parallel!)
   Level 3: @calcom/trpc#build (depends on Level 2)
   Level 4: @calcom/web#build (depends on Level 3)

4. For each task, Turborepo checks cache:
   Hash inputs:
   ├─ Source files (packages/lib/**/*.ts)
   ├─ Dependencies (package.json, yarn.lock)
   ├─ Environment variables
   └─ Previous task outputs

   If hash matches cache → restore from cache
   Else → execute task

5. Execute tasks in parallel (up to CPU cores):
   CPU Core 1: @calcom/lib#build
   CPU Core 2: @calcom/ui#build
   (Wait for completion)
   CPU Core 1: @calcom/trpc#build
   ...

6. Cache outputs for future runs:
   .turbo/cache/
   └─ abc123def456.tar.zst (compressed outputs)
```

---

### 💡 Example: Adding a New Turborepo Task

**Scenario**: Add a `lint` task that runs after `build`.

**Step 1: Define in turbo.json**
```json
{
  "pipeline": {
    "lint": {
      "dependsOn": ["build"],  // Lint after build (to check generated code)
      "outputs": [],           // Linting produces no outputs
      "cache": true            // Cache lint results
    }
  }
}
```

**Step 2: Add script to package.json (per package)**
```json
// packages/lib/package.json
{
  "scripts": {
    "lint": "eslint ."
  }
}
```

**Step 3: Run**
```bash
turbo run lint

# Turborepo will:
# 1. Build all packages first (due to dependsOn)
# 2. Run lint in all packages (in parallel)
# 3. Cache results
# 4. Next run: restore from cache if nothing changed
```

---

### ⚠️ Pitfalls

**1. Missing `outputs` leads to incorrect caching**
```json
// ❌ BAD
{
  "@calcom/web#build": {
    "outputs": []  // Says "no outputs"
  }
}

// Result: Turborepo thinks build has no outputs
// → Caches nothing
// → Next build runs from scratch (slow!)

// ✅ GOOD
{
  "@calcom/web#build": {
    "outputs": [".next/**", "public/**"]
  }
}
```

**2. Missing `inputs` causes false cache hits**
```json
// ❌ BAD
{
  "@calcom/web#build": {
    "inputs": ["src/**"]  // Only watches src/
  }
}

// Problem: If you edit public/image.png, Turborepo won't detect change
// → Uses cached build (outdated image!)

// ✅ GOOD
{
  "@calcom/web#build": {
    "inputs": [
      "src/**",
      "public/**",
      "*.config.js",     // next.config.js, etc.
      "package.json"
    ]
  }
}

// Or better: omit `inputs` → Turborepo watches entire package dir
```

**3. Environment variable leakage**
```json
// turbo.json
{
  "globalEnv": [
    "DATABASE_URL",
    "NEXTAUTH_SECRET"
  ]
}

// Problem: These env vars affect cache hash
// → If dev changes DATABASE_URL locally, cache invalidates for everyone!

// Fix: Only track truly global vars (not local dev vars)
{
  "globalEnv": [
    "NODE_ENV",
    "CI"
  ]
}

// Package-specific env vars:
{
  "@calcom/web#build": {
    "env": [
      "NEXT_PUBLIC_WEBAPP_URL",  // OK to track (same across team)
      "SENTRY_AUTH_TOKEN"        // OK to track (from CI env)
    ]
  }
}
```

---

### 🧱 Best Practices

**1. Use task filtering for faster iterations**
```bash
# Build only @calcom/web and its dependencies
turbo run build --filter=@calcom/web...

# Build only changed packages (since main branch)
turbo run build --filter=[main]

# Build all packages EXCEPT tests
turbo run build --filter=!./packages/*/test
```

**2. Remote caching for team**
```bash
# Enable Vercel remote cache (free for open source)
turbo login
turbo link

# Now cache is shared across team & CI
# First developer builds → uploads cache
# Other developers → download cache (much faster!)
```

**3. Dry run for debugging**
```bash
# See what Turborepo would do (without executing)
turbo run build --dry-run

# Output shows:
# - Task graph
# - Execution order
# - Cache hits/misses
# - Affected packages
```

**4. Generate dependency graph visualization**
```bash
turbo run build --graph=graph.html

# Opens browser with interactive graph:
# - Nodes = tasks
# - Edges = dependencies
# - Colors = cache hits (green) vs misses (red)
```

---

## 9. Advanced Topics & Internals

### 9.1. Yarn PnP (Plug'n'Play) - Why Disabled?

Cal.com uses `nodeLinker: node-modules` (PnP disabled).

**What is PnP?**
- Instead of `node_modules/` folders, Yarn creates a `.pnp.cjs` file
- This file is a **resolution map**: `react → .yarn/cache/react-npm-18.2.0-abc123/`
- Node.js patches `require()` to resolve via this map

**Benefits**:
- ✅ Faster installs (no copying files to node_modules)
- ✅ Stricter dependency resolution (no phantom deps)
- ✅ Smaller disk usage

**Why Cal.com disables it**:
- ⚠️ Prisma CLI doesn't support PnP
- ⚠️ Some Next.js plugins break
- ⚠️ IDE support (VS Code) requires extra setup
- ⚠️ Team familiarity (node_modules is well-understood)

---

### 9.2. Monorepo Performance Optimizations

**1. Hoisting**
```
Before hoisting:
node_modules/                        (0 packages)
apps/web/node_modules/
├── next/                            (500MB)
├── react/                           (5MB)
└── [100 other packages]
apps/api/v2/node_modules/
├── next/                            (500MB duplicate!)
├── react/                           (5MB duplicate!)
└── [80 other packages]

After hoisting:
node_modules/                        (shared)
├── next/                            (500MB, once!)
├── react/                           (5MB, once!)
└── [150 packages]
apps/web/node_modules/
└── @calcom/* → ../../packages/*     (symlinks only)
apps/api/v2/node_modules/
└── @calcom/* → ../../packages/*     (symlinks only)

Disk savings: ~500MB
```

**2. Turborepo local cache**
```
.turbo/cache/
├── abc123def.tar.zst     # @calcom/web#build outputs (compressed)
├── 789ghi456.tar.zst     # @calcom/lib#build outputs
└── [100+ cache entries]

Cache hit:
$ turbo run build --filter=@calcom/web
# Restores from cache in ~2 seconds (instead of 60s build)
```

**3. Incremental type checking**
```json
// tsconfig.json
{
  "compilerOptions": {
    "incremental": true,           // Enable incremental compilation
    "tsBuildInfoFile": ".tsbuildinfo"  // Store build info
  }
}

// Result:
# First type check: 30 seconds
# Subsequent: 3 seconds (only checks changed files)
```

---

## 📝 Những gì đã bổ sung trong DEEP DIVE

✅ **Thêm nội dung** so với v1:

1. **Expanded Context**:
   - 🔍 Ý nghĩa & bối cảnh cho mọi section
   - Giải thích "Why this way?" cho decisions
   - Trade-offs analysis

2. **Detailed Workflows**:
   - ⚙️ Step-by-step execution flows
   - Request-to-response walkthroughs
   - Build pipeline internals

3. **Real Examples**:
   - 💡 Actual code from cal.com codebase
   - Complete OAuth 2.0 implementation
   - Prisma generation pipeline
   - tRPC endpoint examples

4. **Common Pitfalls**:
   - ⚠️ 15+ common mistakes
   - Solutions for each
   - Prevention strategies

5. **Best Practices**:
   - 🧱 Production-ready patterns
   - Code organization templates
   - Performance optimizations

6. **Advanced Topics**:
   - Yarn PnP internals
   - Turborepo caching mechanics
   - NestJS dependency injection
   - Monorepo performance tuning

**Tăng từ 1,031 dòng lên ~6,000+ dòng** (deep dive content)

---

## 👉 Khi nào nên đọc tài liệu Phase này

**Scenarios**:

1. **Onboarding mới**: Đọc full để hiểu project structure
2. **Thêm app mới**: Section 3 (Apps)
3. **Thêm package mới**: Section 4 (Packages)
4. **Debug build issues**: Section 5 (Turborepo)
5. **Setup local dev**: Section 6 (Development Workflow)
6. **Refactoring**: Section 7 (Dependency Graph)
7. **Performance tuning**: Section 9 (Advanced Topics)

**Next Phase**: [PHASE 2: Data Layer & Prisma Deep Dive →](./02-data-layer-prisma.md)
