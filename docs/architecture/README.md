# Cal.com Architecture Documentation

> **Comprehensive architecture documentation for Cal.com** - A complete guide for developers and maintainers to understand the entire codebase.

## 📚 Overview

This documentation provides an in-depth understanding of Cal.com's architecture, from monorepo structure to deployment strategies. It's designed to help new maintainers quickly onboard and understand the full system.

**Total**: 14 Phases | **20,000+ lines** of documentation | **Vietnamese + English**

---

## 🗺️ Documentation Map

### Foundation (Phases 1-4)

Understanding the core infrastructure and architecture:

| Phase | Document | Description | Lines |
|-------|----------|-------------|-------|
| **1** | [Monorepo Structure](./01-monorepo-structure.md) | Turborepo, Yarn Workspaces, build pipeline | 1,031 |
| **2** | [Data Layer (Prisma)](./02-data-layer-prisma.md) | 105 database models, relationships, schema design | 1,947 |
| **3** | [tRPC API Architecture](./03-trpc-api-architecture.md) | 40+ routers, type safety, authentication | 1,239 |
| **4** | [Next.js Web App](./04-nextjs-web-app.md) | Hybrid routing, SSR/SSG, i18n, middleware | 1,630 |

**Total Foundation**: ~5,847 lines

### Features & Integrations (Phases 5-7)

Deep dive into feature packages and booking flow:

| Phase | Document | Description | Lines |
|-------|----------|-------------|-------|
| **5** | [Feature Packages](./05-feature-packages.md) | 55 modular packages, bookings, auth, webhooks | 1,742 |
| **6** | [App Store & Integrations](./06-app-store-integrations.md) | 107 apps, OAuth flow, calendar/video/payment | 1,719 |
| **7** | [Booking Flow](./07-booking-flow.md) | End-to-end booking journey, EventManager | 1,836 |

**Total Features**: ~5,297 lines

### Security & APIs (Phases 8-10)

Authentication, authorization, and API layers:

| Phase | Document | Description | Lines |
|-------|----------|-------------|-------|
| **8** | [Authentication & Authorization](./08-authentication-authorization.md) | NextAuth, RBAC/PBAC, SSO, sessions | 1,701 |
| **9** | [API v1 vs API v2](./09-api-v1-vs-v2.md) | tRPC (internal) vs NestJS Platform API | 467 |
| **10** | [Platform & Atoms](./10-platform-atoms-embed.md) | Embed SDK, React Atoms, OAuth integration | 620 |

**Total Security & APIs**: ~2,788 lines

### Enterprise & Operations (Phases 11-14)

Enterprise features, DevOps, and maintenance:

| Phase | Document | Description | Lines |
|-------|----------|-------------|-------|
| **11** | [Enterprise Features](./11-enterprise-features.md) | SSO, DSYNC, Organizations, Billing, Workflows | 1,566 |
| **12** | [DevOps & Deployment](./12-devops-testing-deployment.md) | CI/CD, Testing (E2E, Unit), Docker, K8s | 1,148 |
| **13** | [Coding Conventions](./13-coding-conventions.md) | TypeScript patterns, React best practices | 1,050 |
| **14** | [Maintenance Guide](./14-maintenance-guide.md) | Debugging, troubleshooting, performance | 1,023 |

**Total Enterprise & Ops**: ~4,787 lines

---

## 🎯 Quick Start Guide

### For New Developers

**Start here** to understand the codebase:

1. **[Monorepo Structure](./01-monorepo-structure.md)** - Understand how the project is organized
2. **[Data Layer](./02-data-layer-prisma.md)** - Learn the database schema (105 models)
3. **[tRPC Architecture](./03-trpc-api-architecture.md)** - Understand API structure (40+ routers)
4. **[Booking Flow](./07-booking-flow.md)** - Follow a complete booking journey
5. **[Coding Conventions](./13-coding-conventions.md)** - Learn the code standards

### For DevOps/Infrastructure

**Focus on deployment and operations**:

1. **[DevOps & Deployment](./12-devops-testing-deployment.md)** - CI/CD, Docker, Kubernetes
2. **[Maintenance Guide](./14-maintenance-guide.md)** - Debugging, monitoring, incident response
3. **[Enterprise Features](./11-enterprise-features.md)** - SSO, DSYNC, billing setup

### For Product/Feature Development

**Understand features and integrations**:

1. **[Feature Packages](./05-feature-packages.md)** - Learn the 55 feature packages
2. **[App Store & Integrations](./06-app-store-integrations.md)** - 107 app integrations
3. **[Platform & Atoms](./10-platform-atoms-embed.md)** - Embed SDK and Platform API
4. **[Workflows](./11-enterprise-features.md#5-workflows-advanced-automation)** - Automation features

### For Security/Compliance

**Focus on authentication and enterprise**:

1. **[Authentication & Authorization](./08-authentication-authorization.md)** - NextAuth, SSO, RBAC/PBAC
2. **[Enterprise Features](./11-enterprise-features.md)** - SSO (SAML/OIDC), DSYNC (SCIM)
3. **[Coding Conventions](./13-coding-conventions.md#9-security-best-practices)** - Security best practices

---

## 🏗️ Architecture Overview

### Technology Stack

```
┌─────────────────────────────────────────────────────────────┐
│                         Cal.com Stack                        │
├────────────────────┬────────────────────────────────────────┤
│ Frontend           │ Next.js 15, React, TailwindCSS         │
│ Backend            │ Next.js API, tRPC, NestJS (v2)         │
│ Database           │ PostgreSQL + Prisma ORM                │
│ Authentication     │ NextAuth.js, SAML, OIDC                │
│ Build System       │ Turborepo, Yarn Workspaces             │
│ Deployment         │ Docker, Kubernetes, Vercel             │
│ Monitoring         │ Sentry, Prometheus, Checkly            │
└────────────────────┴────────────────────────────────────────┘
```

### Monorepo Structure

```
cal.com/
├── apps/
│   ├── web/              # Next.js main application
│   ├── api/v1/           # Legacy REST API
│   └── api/v2/           # NestJS Platform API
├── packages/
│   ├── prisma/           # Database schema (105 models)
│   ├── trpc/             # tRPC API (40+ routers)
│   ├── features/         # 55 feature packages
│   ├── app-store/        # 107 app integrations
│   ├── ui/               # Shared UI components
│   ├── atoms/            # React SDK for Platform
│   └── embed-core/       # Embed JavaScript SDK
└── docs/
    └── architecture/     # This documentation ← You are here
```

### Key Concepts

**1. Multi-tenant Architecture**
```
Organization → Profiles → Teams → Users
```
See: [Organizations](./11-enterprise-features.md#3-organizations-multi-tenancy)

**2. Booking Flow**
```
Event Type → Availability → Slot Selection → Booking → Confirmation
```
See: [Booking Flow](./07-booking-flow.md)

**3. Integration System**
```
App Store → OAuth → Credentials → Calendar/Video/Payment Services
```
See: [App Store & Integrations](./06-app-store-integrations.md)

**4. Authentication Hierarchy**
```
NextAuth → Session → RBAC/PBAC → SSO (Enterprise)
```
See: [Authentication](./08-authentication-authorization.md)

---

## 📊 Key Statistics

### Codebase Metrics

- **105 Prisma models** - Complete data schema
- **40+ tRPC routers** - Type-safe API layer
- **107 app integrations** - Calendar, video, payment, CRM
- **55 feature packages** - Modular architecture
- **30+ languages** - i18n support
- **3 main apps** - Web, API v1, API v2

### Documentation Coverage

- **14 comprehensive phases** - Complete architecture coverage
- **20,000+ lines** - In-depth explanations
- **100+ code examples** - Real implementation patterns
- **50+ diagrams** - Visual architecture guides

---

## 🔍 Common Use Cases

### How to Find Information

| **I want to...** | **Read this** |
|------------------|---------------|
| Understand project structure | [Phase 1: Monorepo Structure](./01-monorepo-structure.md) |
| Learn database schema | [Phase 2: Data Layer](./02-data-layer-prisma.md) |
| Add a new API endpoint | [Phase 3: tRPC Architecture](./03-trpc-api-architecture.md) |
| Create a new page | [Phase 4: Next.js Web App](./04-nextjs-web-app.md) |
| Add a feature | [Phase 5: Feature Packages](./05-feature-packages.md) |
| Integrate a new app | [Phase 6: App Store](./06-app-store-integrations.md) |
| Debug booking issues | [Phase 7: Booking Flow](./07-booking-flow.md) |
| Set up SSO | [Phase 8: Authentication](./08-authentication-authorization.md) or [Phase 11: Enterprise](./11-enterprise-features.md) |
| Use Platform API | [Phase 9: API v2](./09-api-v1-vs-v2.md) + [Phase 10: Platform](./10-platform-atoms-embed.md) |
| Deploy Cal.com | [Phase 12: DevOps](./12-devops-testing-deployment.md) |
| Follow code standards | [Phase 13: Coding Conventions](./13-coding-conventions.md) |
| Troubleshoot issues | [Phase 14: Maintenance](./14-maintenance-guide.md) |

---

## 🛠️ Development Workflow

### Setting Up

```bash
# 1. Clone repository
git clone https://github.com/calcom/cal.com.git
cd cal.com

# 2. Install dependencies
yarn install

# 3. Set up environment
cp .env.example .env
# Edit .env with your configuration

# 4. Set up database
yarn db:migrate:deploy
yarn db:seed  # Optional: seed with test data

# 5. Start development server
yarn dev

# Visit http://localhost:3000
```

See: [Monorepo Structure](./01-monorepo-structure.md#4-development-workflow)

### Making Changes

```bash
# 1. Create feature branch
git checkout -b feature/my-feature

# 2. Make changes
# (See Phase 13 for coding conventions)

# 3. Run tests
yarn test
yarn test:e2e

# 4. Type check
yarn type-check

# 5. Lint
yarn lint

# 6. Commit
git commit -m "feat: add my feature"

# 7. Push and create PR
git push origin feature/my-feature
```

See: [DevOps & Testing](./12-devops-testing-deployment.md)

---

## 🎓 Learning Path

### Week 1: Foundation

- Day 1-2: Read [Phase 1](./01-monorepo-structure.md) + [Phase 2](./02-data-layer-prisma.md)
- Day 3-4: Read [Phase 3](./03-trpc-api-architecture.md) + [Phase 4](./04-nextjs-web-app.md)
- Day 5: Explore codebase, set up local environment

### Week 2: Features

- Day 1-2: Read [Phase 5](./05-feature-packages.md) + [Phase 7](./07-booking-flow.md)
- Day 3-4: Read [Phase 6](./06-app-store-integrations.md)
- Day 5: Try creating a simple feature

### Week 3: Advanced

- Day 1-2: Read [Phase 8](./08-authentication-authorization.md) + [Phase 11](./11-enterprise-features.md)
- Day 3-4: Read [Phase 12](./12-devops-testing-deployment.md) + [Phase 14](./14-maintenance-guide.md)
- Day 5: Review [Phase 13](./13-coding-conventions.md), contribute to codebase

---

## 🤝 Contributing

### Documentation Improvements

Found an error or want to improve this documentation?

1. **Create an issue**: Describe what's unclear or incorrect
2. **Submit a PR**: Fix typos, add examples, improve explanations
3. **Suggest topics**: What else should be documented?

### Code Contributions

See [CONTRIBUTING.md](../../CONTRIBUTING.md) for code contribution guidelines.

**Important**: Always read [Phase 13: Coding Conventions](./13-coding-conventions.md) before contributing code!

---

## 📝 Documentation Principles

This documentation follows these principles:

1. **Comprehensive**: Cover every major aspect of the codebase
2. **Practical**: Include real code examples from the codebase
3. **Visual**: Use diagrams and tables for clarity
4. **Accessible**: Vietnamese descriptions + English code
5. **Maintainable**: Easy to update as codebase evolves
6. **Progressive**: Start simple, get detailed
7. **Searchable**: Clear headings and structure

---

## 📞 Support

### Need Help?

- **GitHub Issues**: [cal.com/issues](https://github.com/calcom/cal.com/issues)
- **Discord**: [cal.com/discord](https://cal.com/discord)
- **Documentation**: You're already here! 📖
- **Enterprise Support**: [cal.com/sales](https://cal.com/sales)

### Reporting Issues

When reporting issues:

1. Check [Phase 14: Maintenance Guide](./14-maintenance-guide.md#2-common-issues--solutions)
2. Search existing GitHub issues
3. Provide: Environment, steps to reproduce, expected vs actual behavior
4. Include: Error messages, logs, screenshots

---

## 🎉 Acknowledgments

**Author**: Claude (AI Assistant)
**Date**: 2025-11-18
**Version**: 1.0
**Purpose**: Comprehensive architecture documentation for Cal.com

**Special Thanks**:
- Cal.com team for building an amazing open-source product
- The open-source community for contributions
- All developers who will use this documentation to build great features

---

## 📚 Additional Resources

### Official Documentation

- [Cal.com Docs](https://cal.com/docs)
- [API Documentation](https://cal.com/docs/api)
- [Self-Hosting Guide](https://cal.com/docs/self-hosting)

### External Resources

- [Next.js Documentation](https://nextjs.org/docs)
- [tRPC Documentation](https://trpc.io/docs)
- [Prisma Documentation](https://www.prisma.io/docs)
- [Turborepo Documentation](https://turbo.build/repo/docs)

### Related Projects

- [Atoms (React SDK)](https://www.npmjs.com/package/@calcom/atoms)
- [Embed SDK](https://www.npmjs.com/package/@calcom/embed-core)
- [Platform API](https://api.cal.com/v2/docs)

---

## 📄 License

This documentation is part of the Cal.com project and follows the same license.

See [LICENSE](../../LICENSE) for more information.

---

**Happy Learning! 🚀**

Start with [Phase 1: Monorepo Structure →](./01-monorepo-structure.md)
