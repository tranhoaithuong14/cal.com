# PHASE 8: Authentication & Authorization

> **Mục tiêu**: Hiểu authentication flow, session management, RBAC/PBAC permissions, SSO, và multi-tenant security.

## 📋 Table of Contents

- [1. Overview](#1-overview)
- [2. Authentication Methods](#2-authentication-methods)
- [3. NextAuth Configuration](#3-nextauth-configuration)
- [4. Session Management](#4-session-management)
- [5. Authorization & Permissions](#5-authorization--permissions)
- [6. SSO (Single Sign-On)](#6-sso-single-sign-on)
- [7. Multi-Tenant Security](#7-multi-tenant-security)
- [8. API Authentication](#8-api-authentication)
- [9. Security Best Practices](#9-security-best-practices)
- [10. Testing Auth Flow](#10-testing-auth-flow)

---

## 1. Overview

### 1.1 Authentication Stack

```
Authentication Layer:
┌────────────────┐
│   NextAuth.js  │ → Session management & OAuth
├────────────────┤
│   Providers    │
├────────────────┼──────────────────────────────────┐
│ Email (Magic)  │ → Passwordless auth             │
│ Credentials    │ → Email + Password              │
│ Google OAuth   │ → Sign in with Google           │
│ SAML SSO       │ → Enterprise SSO (Okta, Azure)  │
│ Impersonation  │ → Admin can act as user         │
└────────────────┴──────────────────────────────────┘

Authorization Layer:
┌────────────────────────────────────────────┐
│ RBAC (Role-Based Access Control)           │
│  - USER, ADMIN, INACTIVE_ADMIN             │
├────────────────────────────────────────────┤
│ PBAC (Permission-Based Access Control)     │
│  - Fine-grained permissions                │
├────────────────────────────────────────────┤
│ Organization Context                        │
│  - Organization → Profile → User           │
└────────────────────────────────────────────┘
```

### 1.2 Key Components

| Component | Purpose | Location |
|-----------|---------|----------|
| NextAuth Options | Auth configuration | `packages/features/auth/lib/next-auth-options.ts` |
| Session Middleware | tRPC auth check | `packages/trpc/server/middlewares/sessionMiddleware.ts` |
| SSO | SAML integration | `packages/features/ee/sso/lib/saml.ts` |
| PBAC | Permissions | `packages/features/pbac/` |
| Impersonation | Admin feature | `packages/features/ee/impersonation/` |

### 1.3 Database Schema

```prisma
model User {
  id                Int      @id @default(autoincrement())
  uuid              String   @unique @default(uuid())
  username          String?  @unique
  email             String   @unique
  password          String?  // bcrypt hash (nullable for OAuth users)
  emailVerified     DateTime?
  identityProvider  IdentityProvider @default(CAL)
  role              UserPermissionRole @default(USER)

  // Organization relationship
  organizationId    Int?
  organization      Team?    @relation("OrganizationToUser", fields: [organizationId], references: [id])

  // Sessions
  sessions          Session[]
  accounts          Account[]

  // Profiles (for multi-tenancy)
  allProfiles       Profile[]

  // Security
  twoFactorEnabled  Boolean @default(false)
  twoFactorSecret   String?
  backupCodes       String[]
}

model Session {
  id           String   @id @default(cuid())
  sessionToken String   @unique
  userId       Int
  expires      DateTime
  user         User     @relation(fields: [userId], references: [id])
}

model Account {
  id                String  @id @default(cuid())
  userId            Int
  type              String  // oauth, email, credentials
  provider          String  // google, cal, saml
  providerAccountId String
  refresh_token     String?
  access_token      String?
  expires_at        Int?
  token_type        String?
  scope             String?
  id_token          String?
  session_state     String?

  user              User    @relation(fields: [userId], references: [id])

  @@unique([provider, providerAccountId])
}

model Profile {
  id             Int      @id @default(autoincrement())
  uid            String   @unique @default(uuid())
  userId         Int
  organizationId Int?
  username       String

  user           User     @relation(fields: [userId], references: [id])
  organization   Team?    @relation(fields: [organizationId], references: [id])

  @@unique([organizationId, username])
}

enum IdentityProvider {
  CAL
  GOOGLE
  SAML
}

enum UserPermissionRole {
  USER
  ADMIN
  INACTIVE_ADMIN
}
```

---

## 2. Authentication Methods

### 2.1 Email (Magic Link)

**Flow**:
```
1. User enters email
   ↓
2. System generates verification token
   ↓
3. Send email with magic link
   ↓
4. User clicks link
   ↓
5. Verify token
   ↓
6. Create session
```

**Implementation**:
```typescript
// packages/features/auth/lib/next-auth-options.ts
EmailProvider({
  server: process.env.EMAIL_SERVER,
  from: process.env.EMAIL_FROM,

  sendVerificationRequest: async ({ identifier, url, provider }) => {
    const { host } = new URL(url);

    await sendEmail({
      to: identifier,
      from: provider.from,
      subject: `Sign in to ${host}`,
      html: `
        <p>Sign in to your account by clicking the link below:</p>
        <a href="${url}">Sign in</a>
        <p>If you didn't request this email, you can safely ignore it.</p>
      `,
    });
  },
}),
```

### 2.2 Credentials (Email + Password)

**Flow**:
```
1. User enters email + password
   ↓
2. Validate credentials
   ↓
3. Check bcrypt hash
   ↓
4. Create session
```

**Implementation**:
```typescript
CredentialsProvider({
  credentials: {
    email: { label: "Email", type: "email" },
    password: { label: "Password", type: "password" },
    totpCode: { label: "2FA Code", type: "text", optional: true },
  },

  authorize: async (credentials, req) => {
    // 1. Find user
    const user = await UserRepository.findByEmailAndIncludeProfilesAndPassword(
      credentials.email.toLowerCase()
    );

    if (!user) {
      throw new Error("Invalid credentials");
    }

    // 2. Verify password
    if (!user.password) {
      throw new Error("Please sign in with your identity provider");
    }

    const isValidPassword = await verifyPassword(
      credentials.password,
      user.password
    );

    if (!isValidPassword) {
      throw new Error("Invalid credentials");
    }

    // 3. Check 2FA (if enabled)
    if (user.twoFactorEnabled) {
      if (!credentials.totpCode) {
        throw new Error("2FA code required");
      }

      const isValidTOTP = verifyTOTP(credentials.totpCode, user.twoFactorSecret);

      if (!isValidTOTP) {
        throw new Error("Invalid 2FA code");
      }
    }

    // 4. Check if user is blocked
    if (user.role === "INACTIVE_ADMIN") {
      throw new Error("Your account has been suspended");
    }

    return AdapterUserPresenter.fromCalUser(user, user.role, true);
  },
}),
```

**Password Hashing**:
```typescript
import bcrypt from "bcryptjs";

// Hash password on signup
export async function hashPassword(password: string): Promise<string> {
  const salt = await bcrypt.genSalt(10);
  return bcrypt.hash(password, salt);
}

// Verify password on login
export async function verifyPassword(
  password: string,
  hashedPassword: string
): Promise<boolean> {
  return bcrypt.compare(password, hashedPassword);
}
```

**Location**: `packages/features/auth/lib/verifyPassword.ts:1`

### 2.3 Google OAuth

**Flow**:
```
1. User clicks "Sign in with Google"
   ↓
2. Redirect to Google consent page
   ↓
3. User authorizes
   ↓
4. Google redirects back with code
   ↓
5. Exchange code for tokens
   ↓
6. Get user profile from Google
   ↓
7. Find or create user
   ↓
8. Auto-link to organization (if domain matches)
   ↓
9. Create session
```

**Configuration**:
```typescript
GoogleProvider({
  clientId: GOOGLE_CLIENT_ID,
  clientSecret: GOOGLE_CLIENT_SECRET,

  authorization: {
    params: {
      scope: GOOGLE_OAUTH_SCOPES.join(" "),
      prompt: "select_account", // Force account selection
    },
  },
}),
```

**Scopes**:
```typescript
const GOOGLE_OAUTH_SCOPES = [
  "https://www.googleapis.com/auth/userinfo.profile",
  "https://www.googleapis.com/auth/userinfo.email",
  "https://www.googleapis.com/auth/calendar.readonly",
  "https://www.googleapis.com/auth/calendar.events",
];
```

**Organization Auto-Link**:
```typescript
// In signIn callback
async signIn({ user, account, profile }) {
  // Auto-link to organization if domain matches
  if (ORGANIZATIONS_AUTOLINK && account.provider === "google") {
    const email = profile.email;
    const domain = getDomainFromEmail(email);

    // Find organization with this domain
    const org = await prisma.team.findFirst({
      where: {
        metadata: {
          path: ["isOrganization"],
          equals: true,
        },
        orgDomain: domain,
      },
    });

    if (org) {
      await autoLinkToOrganization(user.id, org.id);
    }
  }

  return true;
}
```

**Location**: `packages/features/auth/lib/next-auth-options.ts:92`

### 2.4 SAML SSO (Enterprise)

**Flow**:
```
1. User clicks "Sign in with SSO"
   ↓
2. Enter organization slug or email
   ↓
3. Lookup SAML configuration
   ↓
4. Redirect to IdP (Okta, Azure AD, etc.)
   ↓
5. IdP authenticates user
   ↓
6. IdP sends SAML assertion
   ↓
7. Validate assertion signature
   ↓
8. Extract user attributes
   ↓
9. Find or create user
   ↓
10. Create session
```

**Implementation**: See **Section 6: SSO**

### 2.5 Impersonation (Admin)

**Flow**:
```
1. Admin navigates to user profile
   ↓
2. Click "Impersonate User"
   ↓
3. System creates impersonation token
   ↓
4. Redirect to auth endpoint
   ↓
5. Create session as impersonated user
   ↓
6. Admin banner shows "You are impersonating [User]"
   ↓
7. Click "Exit Impersonation" to return
```

**Implementation**:
```typescript
// packages/features/ee/impersonation/lib/ImpersonationProvider.ts
const ImpersonationProvider = CredentialsProvider({
  id: "impersonation-auth",
  name: "Impersonation",

  credentials: {
    username: { type: "text" },
    teamId: { type: "text" },
  },

  authorize: async (credentials, req) => {
    // 1. Verify admin is authenticated
    const session = await getSession({ req });
    if (!session || session.user.role !== "ADMIN") {
      throw new Error("Unauthorized");
    }

    // 2. Find target user
    const user = await UserRepository.findByUsername(
      credentials.username
    );

    if (!user) {
      throw new Error("User not found");
    }

    // 3. Check if impersonation is allowed
    if (user.disableImpersonation) {
      throw new Error("Impersonation is disabled for this user");
    }

    // 4. Log impersonation
    await prisma.impersonationLog.create({
      data: {
        impersonatorId: session.user.id,
        impersonatedUserId: user.id,
        teamId: credentials.teamId ? parseInt(credentials.teamId) : null,
      },
    });

    return user;
  },
});
```

**Location**: `packages/features/ee/impersonation/lib/ImpersonationProvider.ts:1`

---

## 3. NextAuth Configuration

### 3.1 Auth Options

**Location**: `packages/features/auth/lib/next-auth-options.ts:1`

```typescript
export const authOptions: AuthOptions = {
  // Session strategy
  session: {
    strategy: "jwt",  // Use JWT instead of database sessions
    maxAge: 30 * 24 * 60 * 60, // 30 days
  },

  // Pages (custom UI)
  pages: {
    signIn: "/auth/login",
    signOut: "/auth/logout",
    error: "/auth/error",
    verifyRequest: "/auth/verify",
    newUser: "/getting-started",
  },

  // Providers
  providers: [
    EmailProvider({ ... }),
    CredentialsProvider({ ... }),
    GoogleProvider({ ... }),
    ImpersonationProvider({ ... }),
    // SAML provider added dynamically
  ],

  // Callbacks
  callbacks: {
    async signIn({ user, account, profile }) {
      // Custom sign-in logic
    },

    async jwt({ token, user, account, trigger }) {
      // Enrich JWT token
    },

    async session({ session, token }) {
      // Enrich session
    },

    async redirect({ url, baseUrl }) {
      // Custom redirect logic
    },
  },

  // Events
  events: {
    async signIn({ user, account, profile, isNewUser }) {
      // Track sign-ins
    },
    async signOut({ token, session }) {
      // Clean up on sign-out
    },
  },

  // Custom adapter
  adapter: CalComAdapter(prisma),

  // Cookie configuration
  cookies: defaultCookies(WEBAPP_URL?.startsWith("https://") ?? false),

  // Debug
  debug: process.env.NODE_ENV === "development",
};
```

### 3.2 JWT Callback

```typescript
async jwt({ token, user, account, trigger, session }) {
  // On initial sign-in
  if (user) {
    token.id = user.id;
    token.email = user.email;
    token.role = user.role;
    token.username = user.username;
    token.organizationId = user.organizationId;
    token.belongsToActiveTeam = user.belongsToActiveTeam;
  }

  // On session update (trigger === "update")
  if (trigger === "update" && session) {
    token = { ...token, ...session };
  }

  // Refresh user data periodically
  if (token.id && Date.now() - token.iat > 24 * 60 * 60 * 1000) {
    const user = await UserRepository.findById(token.id);
    if (user) {
      token.role = user.role;
      token.organizationId = user.organizationId;
    }
  }

  return token;
}
```

### 3.3 Session Callback

```typescript
async session({ session, token }) {
  // Populate session with token data
  session.user = {
    id: token.id as number,
    email: token.email!,
    name: token.name!,
    username: token.username as string,
    role: token.role as UserPermissionRole,
    organizationId: token.organizationId as number | null,
    belongsToActiveTeam: token.belongsToActiveTeam as boolean,
  };

  // Add organization context
  if (session.user.organizationId) {
    const org = await getOrganizationById(session.user.organizationId);
    session.user.organization = {
      id: org.id,
      name: org.name,
      slug: org.slug,
    };
  }

  return session;
}
```

### 3.4 Custom Adapter

**Location**: `packages/features/auth/lib/next-auth-custom-adapter.ts:1`

```typescript
export default function CalComAdapter(prisma: PrismaClient): Adapter {
  return {
    async createUser(data) {
      const user = await prisma.user.create({
        data: {
          email: data.email.toLowerCase(),
          emailVerified: data.emailVerified,
          name: data.name,
          identityProvider: "CAL",
        },
      });

      return user;
    },

    async getUser(id) {
      const user = await prisma.user.findUnique({ where: { id } });
      return user;
    },

    async getUserByEmail(email) {
      const user = await prisma.user.findUnique({
        where: { email: email.toLowerCase() },
      });
      return user;
    },

    async getUserByAccount({ providerAccountId, provider }) {
      const account = await prisma.account.findUnique({
        where: {
          provider_providerAccountId: {
            provider,
            providerAccountId,
          },
        },
        include: { user: true },
      });

      return account?.user ?? null;
    },

    async updateUser({ id, ...data }) {
      const user = await prisma.user.update({
        where: { id },
        data,
      });
      return user;
    },

    async linkAccount(data) {
      const account = await prisma.account.create({ data });
      return account;
    },

    async createSession(data) {
      const session = await prisma.session.create({ data });
      return session;
    },

    async getSessionAndUser(sessionToken) {
      const session = await prisma.session.findUnique({
        where: { sessionToken },
        include: { user: true },
      });

      if (!session) return null;

      return {
        session,
        user: session.user,
      };
    },

    async updateSession({ sessionToken, ...data }) {
      const session = await prisma.session.update({
        where: { sessionToken },
        data,
      });
      return session;
    },

    async deleteSession(sessionToken) {
      await prisma.session.delete({
        where: { sessionToken },
      });
    },
  };
}
```

---

## 4. Session Management

### 4.1 Get Session (Server-Side)

```typescript
import { getServerSession } from "next-auth";
import { authOptions } from "@calcom/features/auth/lib/next-auth-options";

// In API route or getServerSideProps
export async function handler(req: NextApiRequest, res: NextApiResponse) {
  const session = await getServerSession(req, res, authOptions);

  if (!session) {
    return res.status(401).json({ error: "Unauthorized" });
  }

  // Access session data
  const userId = session.user.id;
  const userRole = session.user.role;

  // ...
}
```

### 4.2 Get Session (Client-Side)

```typescript
import { useSession } from "next-auth/react";

function Component() {
  const { data: session, status } = useSession();

  if (status === "loading") {
    return <Spinner />;
  }

  if (status === "unauthenticated") {
    return <SignInButton />;
  }

  return (
    <div>
      <p>Signed in as {session.user.email}</p>
      <p>Role: {session.user.role}</p>
    </div>
  );
}
```

### 4.3 tRPC Context

**Location**: `packages/trpc/server/createContext.ts:1`

```typescript
export const createContext = async ({ req, res }, sessionGetter?) => {
  // 1. Get session
  const session = sessionGetter
    ? await sessionGetter({ req, res })
    : await getServerSession(req, res, authOptions);

  // 2. Get locale
  const locale = await getLocale(req);

  // 3. Get source IP
  const sourceIp = getIP(req);

  // 4. Build context
  return {
    req,
    res,
    prisma,
    insightsDb: readonlyPrisma,
    session,
    locale,
    sourceIp,

    // Enriched user data
    user: session
      ? await getUserFromSession(session)
      : null,
  };
};

// Enrich session with user data
async function getUserFromSession(session: Session) {
  const user = await prisma.user.findUnique({
    where: { id: session.user.id },
    include: {
      allProfiles: {
        include: {
          organization: true,
        },
      },
      organization: true,
    },
  });

  return user;
}
```

### 4.4 Session Middleware (tRPC)

**Location**: `packages/trpc/server/middlewares/sessionMiddleware.ts:1`

```typescript
// Require authentication
export const isAuthed = middleware(async ({ ctx, next }) => {
  if (!ctx.session || !ctx.user) {
    throw new TRPCError({
      code: "UNAUTHORIZED",
      message: "Not authenticated",
    });
  }

  return next({
    ctx: {
      ...ctx,
      session: ctx.session,
      user: ctx.user,
    },
  });
});

// Require admin role
export const isAdmin = middleware(async ({ ctx, next }) => {
  if (!ctx.session || !ctx.user) {
    throw new TRPCError({ code: "UNAUTHORIZED" });
  }

  if (ctx.user.role !== "ADMIN") {
    throw new TRPCError({
      code: "FORBIDDEN",
      message: "Admin access required",
    });
  }

  return next({ ctx });
});

// Usage in tRPC router
export const authedProcedure = procedure.use(isAuthed);
export const adminProcedure = procedure.use(isAdmin);
```

---

## 5. Authorization & Permissions

### 5.1 RBAC (Role-Based Access Control)

**Roles**:
```typescript
enum UserPermissionRole {
  USER = "USER",              // Regular user
  ADMIN = "ADMIN",            // Platform admin
  INACTIVE_ADMIN = "INACTIVE_ADMIN", // Suspended admin
}
```

**Permission Matrix**:

| Action | USER | ADMIN |
|--------|------|-------|
| View own bookings | ✅ | ✅ |
| Create event types | ✅ | ✅ |
| Impersonate users | ❌ | ✅ |
| Manage all users | ❌ | ✅ |
| Access admin panel | ❌ | ✅ |
| Modify platform settings | ❌ | ✅ |

**Implementation**:
```typescript
// Check role
function requireAdmin(user: User) {
  if (user.role !== "ADMIN") {
    throw new Error("Admin access required");
  }
}

// In tRPC procedure
export const adminRouter = router({
  listAllUsers: adminProcedure.query(async ({ ctx }) => {
    // ctx.user.role === "ADMIN" (guaranteed by adminProcedure)
    return prisma.user.findMany();
  }),
});
```

### 5.2 Team Permissions

```typescript
enum MembershipRole {
  MEMBER = "MEMBER",    // Regular team member
  ADMIN = "ADMIN",      // Team admin
  OWNER = "OWNER",      // Team owner
}
```

**Permission Matrix**:

| Action | MEMBER | ADMIN | OWNER |
|--------|--------|-------|-------|
| View team bookings | ✅ | ✅ | ✅ |
| Manage own event types | ✅ | ✅ | ✅ |
| Manage team event types | ❌ | ✅ | ✅ |
| Invite members | ❌ | ✅ | ✅ |
| Remove members | ❌ | ✅ | ✅ |
| Delete team | ❌ | ❌ | ✅ |
| Change team settings | ❌ | ✅ | ✅ |

**Check Team Permission**:
```typescript
async function checkTeamPermission(
  userId: number,
  teamId: number,
  requiredRole: MembershipRole
) {
  const membership = await prisma.membership.findFirst({
    where: {
      userId,
      teamId,
      accepted: true,
    },
  });

  if (!membership) {
    throw new Error("Not a team member");
  }

  const roleHierarchy = {
    MEMBER: 1,
    ADMIN: 2,
    OWNER: 3,
  };

  if (roleHierarchy[membership.role] < roleHierarchy[requiredRole]) {
    throw new Error(`${requiredRole} access required`);
  }

  return membership;
}
```

### 5.3 PBAC (Permission-Based Access Control)

**Location**: `packages/features/pbac/`

Fine-grained permissions for enterprise organizations:

```typescript
// Permission model
model Permission {
  id             Int      @id @default(autoincrement())
  slug           String   @unique
  name           String
  description    String?
  createdAt      DateTime @default(now())

  // Relations
  rolePermissions RolePermission[]
}

model Role {
  id          Int      @id @default(autoincrement())
  name        String
  teamId      Int

  team        Team     @relation(fields: [teamId], references: [id])
  permissions RolePermission[]
  members     Membership[]
}

model RolePermission {
  id           Int        @id @default(autoincrement())
  roleId       Int
  permissionId Int

  role         Role       @relation(fields: [roleId], references: [id])
  permission   Permission @relation(fields: [permissionId], references: [id])

  @@unique([roleId, permissionId])
}
```

**Available Permissions**:
```typescript
enum PermissionSlug {
  // Event Types
  CREATE_EVENT_TYPE = "create_event_type",
  EDIT_EVENT_TYPE = "edit_event_type",
  DELETE_EVENT_TYPE = "delete_event_type",

  // Bookings
  VIEW_ALL_BOOKINGS = "view_all_bookings",
  CANCEL_BOOKINGS = "cancel_bookings",

  // Team Management
  INVITE_MEMBERS = "invite_members",
  REMOVE_MEMBERS = "remove_members",
  MANAGE_ROLES = "manage_roles",

  // Settings
  MANAGE_TEAM_SETTINGS = "manage_team_settings",
  MANAGE_BILLING = "manage_billing",

  // API Keys
  CREATE_API_KEYS = "create_api_keys",
  REVOKE_API_KEYS = "revoke_api_keys",
}
```

**Check Permission**:
```typescript
async function checkPermission(
  userId: number,
  teamId: number,
  permissionSlug: PermissionSlug
): Promise<boolean> {
  // Get user's membership
  const membership = await prisma.membership.findFirst({
    where: {
      userId,
      teamId,
      accepted: true,
    },
    include: {
      role: {
        include: {
          permissions: {
            include: {
              permission: true,
            },
          },
        },
      },
    },
  });

  if (!membership) return false;

  // Check if role has permission
  const hasPermission = membership.role.permissions.some(
    (rp) => rp.permission.slug === permissionSlug
  );

  return hasPermission;
}

// Usage in tRPC procedure
export const teamRouter = router({
  createEventType: authedProcedure
    .input(z.object({ teamId: z.number() }))
    .mutation(async ({ ctx, input }) => {
      const hasPermission = await checkPermission(
        ctx.user.id,
        input.teamId,
        PermissionSlug.CREATE_EVENT_TYPE
      );

      if (!hasPermission) {
        throw new TRPCError({
          code: "FORBIDDEN",
          message: "You don't have permission to create event types",
        });
      }

      // Create event type...
    }),
});
```

---

## 6. SSO (Single Sign-On)

### 6.1 SAML Configuration

**Location**: `packages/features/ee/sso/lib/saml.ts:1`

Cal.com uses **BoxyHQ SAML Jackson** for SAML SSO:

```typescript
import jackson from "@boxyhq/saml-jackson";

const samlJackson = await jackson({
  externalUrl: WEBAPP_URL,
  samlPath: "/api/auth/saml/callback",
  db: {
    engine: "sql",
    url: DATABASE_URL,
  },
});

const { apiController, oauthController } = samlJackson;
```

**Supported IdPs**:
- Okta
- Azure Active Directory
- Google Workspace
- OneLogin
- Auth0
- Generic SAML 2.0

### 6.2 SSO Setup (Organization Admin)

```typescript
// 1. Admin uploads SAML metadata
async function configureSAML(organizationId: number, metadata: SAMLMetadata) {
  await apiController.config({
    tenant: `org_${organizationId}`,
    product: "cal.com",
    defaultRedirectUrl: `${WEBAPP_URL}/auth/sso/callback`,
    redirectUrl: [
      `${WEBAPP_URL}/auth/sso/callback`,
      `${WEBAPP_URL}/*`,
    ],
    metadataUrl: metadata.metadataUrl,
    // OR raw XML
    rawMetadata: metadata.rawXml,
  });
}

// 2. Get SSO configuration
async function getSAMLConfig(organizationId: number) {
  const config = await apiController.getConfig({
    tenant: `org_${organizationId}`,
    product: "cal.com",
  });

  return config;
}
```

### 6.3 SSO Login Flow

```typescript
// 1. Initiate SSO
export async function initiateSSO(email: string) {
  // Extract domain from email
  const domain = getDomainFromEmail(email);

  // Find organization by domain
  const org = await prisma.team.findFirst({
    where: {
      metadata: {
        path: ["isOrganization"],
        equals: true,
      },
      orgDomain: domain,
    },
  });

  if (!org) {
    throw new Error("No organization found for this domain");
  }

  // Get SAML authorization URL
  const { redirect_url } = await oauthController.authorize({
    tenant: `org_${org.id}`,
    product: "cal.com",
    redirect_uri: `${WEBAPP_URL}/auth/sso/callback`,
    state: encrypt({ organizationId: org.id }),
  });

  return redirect_url;
}

// 2. Handle SSO callback
export async function handleSSOCallback(
  samlResponse: string,
  relayState: string
) {
  // Validate SAML response
  const { profile } = await oauthController.samlResponse({
    SAMLResponse: samlResponse,
    RelayState: relayState,
  });

  const { organizationId } = decrypt(relayState);

  // Extract user attributes
  const email = profile.email;
  const firstName = profile.firstName;
  const lastName = profile.lastName;

  // Find or create user
  let user = await prisma.user.findUnique({
    where: { email: email.toLowerCase() },
  });

  if (!user) {
    user = await prisma.user.create({
      data: {
        email: email.toLowerCase(),
        name: `${firstName} ${lastName}`,
        emailVerified: new Date(),
        identityProvider: "SAML",
        organizationId,
      },
    });

    // Create profile
    await createProfile(user.id, organizationId);
  }

  // Create session
  return user;
}
```

**Location**: `packages/features/ee/sso/lib/saml.ts:1`

---

## 7. Multi-Tenant Security

### 7.1 Organization Hierarchy

```
Organization (Cal.com)
├── Profile 1 (acme.cal.com)
│   ├── Team A
│   │   ├── Member 1
│   │   └── Member 2
│   └── Team B
├── Profile 2 (startup.cal.com)
│   └── Team C
└── Standalone Users
```

### 7.2 Profile Isolation

```typescript
// Each organization has separate profile namespace
model Profile {
  id             Int    @id @default(autoincrement())
  uid            String @unique
  userId         Int
  organizationId Int?
  username       String

  @@unique([organizationId, username]) // Username unique within org
}
```

**Example**:
- User ID: 123
- Profile 1: `john` @ Organization A → `john.acme.cal.com`
- Profile 2: `john` @ Organization B → `john.startup.cal.com`
- Same username, different profiles

### 7.3 Data Isolation

```typescript
// Ensure queries are scoped to organization
async function getEventTypes(userId: number, organizationId?: number) {
  // If organization context, only return event types in that org
  if (organizationId) {
    return prisma.eventType.findMany({
      where: {
        userId,
        team: {
          parentId: organizationId, // Team belongs to org
        },
      },
    });
  }

  // Otherwise, return all event types
  return prisma.eventType.findMany({
    where: { userId },
  });
}
```

### 7.4 Subdomain Routing

```typescript
// apps/web/middleware.ts
export function middleware(req: NextRequest) {
  const hostname = req.headers.get("host");

  // Check if subdomain
  if (hostname && hostname.includes(".cal.com")) {
    const subdomain = hostname.split(".")[0];

    // Find organization by slug
    const org = await getOrganizationBySlug(subdomain);

    if (org) {
      // Rewrite to organization routes
      return NextResponse.rewrite(
        new URL(`/org/${subdomain}${req.nextUrl.pathname}`, req.url)
      );
    }
  }

  return NextResponse.next();
}
```

**Location**: `apps/web/middleware.ts:1`

---

## 8. API Authentication

### 8.1 API Keys

```prisma
model ApiKey {
  id         String   @id @default(cuid())
  userId     Int
  teamId     Int?
  note       String?
  createdAt  DateTime @default(now())
  expiresAt  DateTime?
  lastUsedAt DateTime?
  hashedKey  String   @unique

  user       User     @relation(fields: [userId], references: [id])
  team       Team?    @relation(fields: [teamId], references: [id])
}
```

**Generate API Key**:
```typescript
import crypto from "crypto";

async function generateApiKey(userId: number, teamId?: number) {
  // Generate random key
  const apiKey = `cal_${crypto.randomBytes(32).toString("hex")}`;

  // Hash for storage
  const hashedKey = crypto
    .createHash("sha256")
    .update(apiKey)
    .digest("hex");

  // Store in database
  await prisma.apiKey.create({
    data: {
      userId,
      teamId,
      hashedKey,
      expiresAt: addYears(new Date(), 1), // 1 year expiration
    },
  });

  // Return unhashed key (only time it's visible)
  return apiKey;
}
```

**Verify API Key**:
```typescript
async function verifyApiKey(apiKey: string): Promise<User | null> {
  // Hash provided key
  const hashedKey = crypto
    .createHash("sha256")
    .update(apiKey)
    .digest("hex");

  // Find in database
  const key = await prisma.apiKey.findUnique({
    where: { hashedKey },
    include: { user: true },
  });

  if (!key) return null;

  // Check expiration
  if (key.expiresAt && key.expiresAt < new Date()) {
    return null;
  }

  // Update last used
  await prisma.apiKey.update({
    where: { id: key.id },
    data: { lastUsedAt: new Date() },
  });

  return key.user;
}
```

**Usage in API Route**:
```typescript
export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  const apiKey = req.headers["x-api-key"] as string;

  if (!apiKey) {
    return res.status(401).json({ error: "API key required" });
  }

  const user = await verifyApiKey(apiKey);

  if (!user) {
    return res.status(401).json({ error: "Invalid API key" });
  }

  // Continue with authenticated user...
}
```

### 8.2 OAuth Access Tokens (Platform API)

**Location**: `packages/features/ee/platform/`

For the Platform API (v2), OAuth 2.0 is used:

```typescript
// 1. Exchange authorization code for access token
POST /api/v2/oauth/token
{
  "grant_type": "authorization_code",
  "code": "abc123",
  "client_id": "xxx",
  "client_secret": "yyy",
  "redirect_uri": "https://app.example.com/callback"
}

// Response:
{
  "access_token": "eyJ...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "xyz..."
}

// 2. Use access token in API requests
GET /api/v2/bookings
Authorization: Bearer eyJ...
```

---

## 9. Security Best Practices

### 9.1 Password Requirements

```typescript
const PASSWORD_REQUIREMENTS = {
  minLength: 8,
  requireUppercase: true,
  requireLowercase: true,
  requireNumbers: true,
  requireSpecialChars: false,
};

function isPasswordValid(password: string): boolean {
  if (password.length < PASSWORD_REQUIREMENTS.minLength) {
    return false;
  }

  if (PASSWORD_REQUIREMENTS.requireUppercase && !/[A-Z]/.test(password)) {
    return false;
  }

  if (PASSWORD_REQUIREMENTS.requireLowercase && !/[a-z]/.test(password)) {
    return false;
  }

  if (PASSWORD_REQUIREMENTS.requireNumbers && !/[0-9]/.test(password)) {
    return false;
  }

  return true;
}
```

### 9.2 Two-Factor Authentication (2FA)

```typescript
import { authenticator } from "otplib";

// 1. Enable 2FA
async function enable2FA(userId: number) {
  // Generate secret
  const secret = authenticator.generateSecret();

  // Generate QR code
  const qrCode = await QRCode.toDataURL(
    authenticator.keyuri(user.email, "Cal.com", secret)
  );

  // Store secret (encrypted)
  await prisma.user.update({
    where: { id: userId },
    data: {
      twoFactorSecret: encrypt(secret),
      twoFactorEnabled: false, // Not enabled until verified
    },
  });

  return { secret, qrCode };
}

// 2. Verify 2FA setup
async function verify2FASetup(userId: number, code: string) {
  const user = await prisma.user.findUnique({ where: { id: userId } });

  const secret = decrypt(user.twoFactorSecret);
  const isValid = authenticator.verify({ token: code, secret });

  if (!isValid) {
    throw new Error("Invalid code");
  }

  // Generate backup codes
  const backupCodes = generateBackupCodes(10);

  // Enable 2FA
  await prisma.user.update({
    where: { id: userId },
    data: {
      twoFactorEnabled: true,
      backupCodes: backupCodes.map(hashBackupCode),
    },
  });

  return backupCodes;
}

// 3. Verify 2FA code on login
function verifyTOTP(code: string, secret: string): boolean {
  return authenticator.verify({
    token: code,
    secret: decrypt(secret),
  });
}
```

### 9.3 Rate Limiting

```typescript
// packages/lib/checkRateLimitAndThrowError.ts
export async function checkRateLimitAndThrowError({
  identifier,
  rateLimitingType,
}: RateLimitOptions) {
  const { isRateLimited } = await checkRateLimit({
    identifier,
    rateLimitingType,
  });

  if (isRateLimited) {
    throw new TRPCError({
      code: "TOO_MANY_REQUESTS",
      message: "Rate limit exceeded. Please try again later.",
    });
  }
}

// Rate limits
const RATE_LIMITS = {
  login: {
    points: 5,        // 5 attempts
    duration: 60 * 15, // per 15 minutes
  },
  api: {
    points: 100,      // 100 requests
    duration: 60,     // per minute
  },
  booking: {
    points: 10,       // 10 bookings
    duration: 60 * 60, // per hour
  },
};
```

### 9.4 CSRF Protection

NextAuth handles CSRF automatically:
```typescript
// CSRF token in form
<form method="post" action="/api/auth/signin/email">
  <input type="hidden" name="csrfToken" value={csrfToken} />
  <input type="email" name="email" />
  <button type="submit">Sign in</button>
</form>

// Get CSRF token
import { getCsrfToken } from "next-auth/react";

export async function getServerSideProps(context) {
  return {
    props: {
      csrfToken: await getCsrfToken(context),
    },
  };
}
```

---

## 10. Testing Auth Flow

### 10.1 Unit Tests

```typescript
describe("verifyPassword", () => {
  it("should verify correct password", async () => {
    const password = "MySecurePassword123";
    const hash = await hashPassword(password);

    const isValid = await verifyPassword(password, hash);
    expect(isValid).toBe(true);
  });

  it("should reject incorrect password", async () => {
    const hash = await hashPassword("correct");
    const isValid = await verifyPassword("wrong", hash);
    expect(isValid).toBe(false);
  });
});
```

### 10.2 Integration Tests

```typescript
describe("Auth Flow", () => {
  it("should sign in with credentials", async () => {
    const user = await createTestUser({
      email: "test@example.com",
      password: "password123",
    });

    const session = await signIn("credentials", {
      email: "test@example.com",
      password: "password123",
      redirect: false,
    });

    expect(session).toBeDefined();
    expect(session.user.email).toBe("test@example.com");
  });

  it("should reject invalid credentials", async () => {
    await expect(
      signIn("credentials", {
        email: "test@example.com",
        password: "wrong",
        redirect: false,
      })
    ).rejects.toThrow("Invalid credentials");
  });
});
```

---

## Summary

### Key Takeaways

1. **Authentication Methods**:
   - Email (magic link)
   - Credentials (email + password)
   - Google OAuth
   - SAML SSO (enterprise)
   - Impersonation (admin)

2. **Session Management**:
   - JWT-based sessions (30-day expiration)
   - NextAuth with custom adapter
   - tRPC middleware for auth checks

3. **Authorization**:
   - RBAC: USER, ADMIN roles
   - Team permissions: MEMBER, ADMIN, OWNER
   - PBAC: Fine-grained permissions (enterprise)

4. **SSO**: SAML integration via BoxyHQ Jackson

5. **Multi-Tenant**: Organization isolation via profiles

6. **API Auth**: API keys (SHA-256 hashed) + OAuth tokens

7. **Security**: 2FA, password requirements, rate limiting, CSRF protection

### Auth Flow Summary

```
Sign In Flow:
1. User enters credentials (email + password / OAuth / SSO)
2. Verify credentials
3. Create/find user in database
4. Generate JWT token
5. Set session cookie
6. Enrich session with user data
7. Redirect to dashboard

Authorization Flow:
1. Middleware extracts JWT from cookie
2. Verify JWT signature
3. Get user from database
4. Check permissions (role, team membership, PBAC)
5. Allow/deny access
```

### Next Steps

- **PHASE 9**: API v1 vs API v2
- **PHASE 10**: Platform & Atoms (Embed/SDK)
- **PHASE 11**: Enterprise Features

---

**Tác giả**: Claude (AI Assistant)
**Ngày tạo**: 2025-11-18
**Phiên bản**: 1.0
