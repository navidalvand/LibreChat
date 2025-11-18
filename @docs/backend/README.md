# Backend Overview

> **Complete API architecture documentation for LibreChat v0.8.1-rc1**

## Architecture Summary

LibreChat's backend implements a **layered architecture** with clear separation of concerns:

```
Routes → Middleware → Controllers → Services → Models → Database
```

## Technology Stack

- **Runtime:** Node.js v20+ (Bun compatible)
- **Framework:** Express.js
- **Database:** MongoDB (Mongoose ODM)
- **Cache:** Redis (ioredis)
- **Search:** MeiliSearch
- **Authentication:** Passport.js (11 strategies)
- **AI SDKs:** OpenAI, Anthropic, Google, AWS Bedrock

## Directory Structure

```
api/
├── server/
│   ├── index.js              # Main server entry point
│   ├── routes/               # API route definitions (20+ files)
│   ├── controllers/          # Request handlers
│   ├── middleware/           # Express middleware (40+ functions)
│   └── services/             # Business logic layer
├── app/clients/              # AI provider implementations
├── models/                   # Database query methods
├── strategies/               # Passport auth strategies (11 total)
├── db/                       # Database connection & indexing
├── cache/                    # Redis caching logic
└── lib/                      # Shared libraries
```

## Key Entry Points

**Server Initialization:** `/home/user/LibreChat/api/server/index.js`
- Lines 42-177: Complete server setup
- Lines 81-96: Global middleware stack
- Lines 103-115: Passport configuration
- Lines 116-146: Route registration

**Route Index:** `/home/user/LibreChat/api/server/routes/index.js`
- Central route registration
- Middleware application per route group

## Request Flow

```
1. Client Request
2. Global Middleware (JSON parsing, CORS, sanitization, compression)
3. Route-Specific Middleware (auth, rate limiting, validation)
4. Controller (request handling, orchestration)
5. Service Layer (business logic, AI calls)
6. Model Layer (database operations)
7. Response (JSON or SSE stream)
```

## Core Features

### 60+ API Endpoints

- **Authentication:** Login, OAuth2, LDAP, SAML, 2FA
- **Conversations:** CRUD, search, archive, fork, duplicate
- **Messages:** CRUD, streaming, feedback, artifacts
- **Agents:** Marketplace, CRUD, versioning, MCP integration
- **Files:** Upload, download, multi-strategy storage
- **Permissions:** ACL management, resource sharing
- **Admin:** User management, roles, system config

### 40+ Middleware Functions

- **Authentication:** JWT, Local, LDAP, OAuth2, SAML, OpenID
- **Authorization:** RBAC, resource-level ACL
- **Security:** Rate limiting (15+ types), ban system, NoSQL injection prevention
- **Validation:** Request schema validation, message validation
- **Processing:** Abort controllers, SSE headers, compression

### Streaming Architecture

- **Technology:** Server-Sent Events (SSE)
- **Use Cases:** AI response streaming, real-time updates
- **Features:** Abort support, progress tracking, multi-conversation

## Performance Optimizations

1. **Connection Pooling:** Configurable MongoDB/Redis connections
2. **Caching:** Redis for config, models, sessions
3. **Lean Queries:** MongoDB queries return plain objects
4. **Compression:** gzip/brotli response compression
5. **Rate Limiting:** Prevents abuse, distributed via Redis

## Security Features

1. **NoSQL Injection Prevention:** mongoSanitize middleware
2. **CORS:** Configurable cross-origin policies
3. **Rate Limiting:** IP and user-based limits
4. **Ban System:** Automated violation tracking
5. **Session Security:** httpOnly cookies, refresh token rotation
6. **Input Validation:** Zod schemas, Mongoose validation

## Related Documentation

- [API Routes](api-routes.md) - Complete endpoint reference
- [Middleware](middleware.md) - Middleware pipeline documentation
- [Database](database/README.md) - Schema and query patterns
- [Authentication](../features/authentication.md) - Auth implementation

---

**Last Updated:** 2025-11-18
**Version:** 1.0.0
