# LibreChat Architecture Overview

## Executive Summary

LibreChat is a **production-ready, enterprise-grade AI chat platform** built on a modern full-stack JavaScript architecture. The system implements a sophisticated **monorepo structure** with clear separation of concerns, supporting multiple AI providers, real-time streaming, and advanced features like agents, tools, and code artifacts.

**Key Architectural Characteristics:**
- **Monorepo:** npm workspaces with 4 shared packages
- **Backend:** Node.js + Express.js (RESTful API + Server-Sent Events)
- **Frontend:** React 18 + Vite (SPA with progressive enhancement)
- **Database:** MongoDB (primary), Redis (cache/sessions), MeiliSearch (search), pgvector (RAG)
- **Authentication:** Multi-strategy (JWT, OAuth2, LDAP, SAML, 2FA)
- **Real-time:** Server-Sent Events (SSE) for streaming AI responses
- **Deployment:** Docker-first with Kubernetes support

---

## System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           CLIENT LAYER (React)                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌────────────┐ │
│  │   Chat UI    │  │   Agents     │  │  Artifacts   │  │  Settings  │ │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └─────┬──────┘ │
│         │                 │                 │                 │        │
│  ┌──────┴─────────────────┴─────────────────┴─────────────────┴──────┐ │
│  │              State Management (Jotai + TanStack Query)            │ │
│  └──────────────────────────────┬─────────────────────────────────────┘ │
│                                 │                                       │
└─────────────────────────────────┼───────────────────────────────────────┘
                                  │ REST API + SSE
                                  ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                     API LAYER (Express.js)                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │  Middleware Pipeline                                           │    │
│  │  (Auth → Rate Limit → Validation → Sanitization)              │    │
│  └────────────────────────────┬───────────────────────────────────┘    │
│                               │                                         │
│  ┌────────────────────────────▼───────────────────────────────────┐    │
│  │  Routes (messages, convos, agents, auth, files, etc.)         │    │
│  └────────────────────────────┬───────────────────────────────────┘    │
│                               │                                         │
│  ┌────────────────────────────▼───────────────────────────────────┐    │
│  │  Controllers (request handling, response formatting)           │    │
│  └────────────────────────────┬───────────────────────────────────┘    │
│                               │                                         │
│  ┌────────────────────────────▼───────────────────────────────────┐    │
│  │  Services (business logic, AI provider integration)            │    │
│  └────────────────────────────┬───────────────────────────────────┘    │
│                               │                                         │
│  ┌────────────────────────────▼───────────────────────────────────┐    │
│  │  Models (database queries, data validation)                    │    │
│  └────────────────────────────┬───────────────────────────────────┘    │
│                               │                                         │
└───────────────────────────────┼─────────────────────────────────────────┘
                                │
                    ┌───────────┴───────────┐
                    │                       │
                    ▼                       ▼
┌───────────────────────────┐   ┌───────────────────────────┐
│   DATA LAYER              │   │   INTEGRATION LAYER       │
├───────────────────────────┤   ├───────────────────────────┤
│                           │   │                           │
│  ┌──────────────────┐     │   │  ┌──────────────────┐    │
│  │    MongoDB       │     │   │  │  OpenAI API      │    │
│  │  (Conversations, │     │   │  └──────────────────┘    │
│  │   Messages,      │     │   │  ┌──────────────────┐    │
│  │   Users, etc.)   │     │   │  │  Anthropic API   │    │
│  └──────────────────┘     │   │  └──────────────────┘    │
│  ┌──────────────────┐     │   │  ┌──────────────────┐    │
│  │  Redis           │     │   │  │  Google AI       │    │
│  │  (Sessions,      │     │   │  └──────────────────┘    │
│  │   Cache)         │     │   │  ┌──────────────────┐    │
│  └──────────────────┘     │   │  │  AWS Bedrock     │    │
│  ┌──────────────────┐     │   │  └──────────────────┘    │
│  │  MeiliSearch     │     │   │  ┌──────────────────┐    │
│  │  (Full-text      │     │   │  │  Azure OpenAI    │    │
│  │   Search)        │     │   │  └──────────────────┘    │
│  └──────────────────┘     │   │  ┌──────────────────┐    │
│  ┌──────────────────┐     │   │  │  Ollama (Local)  │    │
│  │  pgvector (RAG)  │     │   │  └──────────────────┘    │
│  └──────────────────┘     │   │  ┌──────────────────┐    │
│  ┌──────────────────┐     │   │  │  MCP Servers     │    │
│  │  S3/Firebase     │     │   │  └──────────────────┘    │
│  │  (File Storage)  │     │   │                           │
│  └──────────────────┘     │   └───────────────────────────┘
└───────────────────────────┘
```

---

## Architecture Layers

### 1. Client Layer (React Application)

**Location:** `/home/user/LibreChat/client/`

**Responsibilities:**
- User interface rendering and interaction
- State management (local and server state)
- API communication
- Real-time message streaming (SSE)
- File upload handling
- Client-side routing

**Key Technologies:**
- React 18 with hooks
- Vite 6 (build tool)
- Jotai + Recoil (client state)
- TanStack Query (server state, caching)
- React Router v6 (routing)
- Radix UI + Tailwind CSS (UI components)

**Entry Point:** `client/src/main.jsx`

---

### 2. API Layer (Express Server)

**Location:** `/home/user/LibreChat/api/`

**Responsibilities:**
- Request routing and handling
- Authentication and authorization
- Business logic execution
- AI provider orchestration
- WebSocket/SSE management
- File processing

**Architecture Pattern:** **Layered Architecture**
```
Routes → Middleware → Controllers → Services → Models → Database
```

**Entry Point:** `api/server/index.js`

**Middleware Stack (in order):**
1. `noIndex` - SEO headers
2. `express.json()` - JSON parsing (3MB limit)
3. `express.urlencoded()` - Form parsing
4. `mongoSanitize()` - NoSQL injection prevention
5. `cors()` - CORS handling
6. `cookieParser()` - Cookie parsing
7. `compression()` - Response compression
8. `staticCache()` - Static file caching

---

### 3. Data Layer

**MongoDB (Primary Database)**
- **Location:** `packages/data-schemas/src/schema/`
- **Purpose:** Persistent data storage
- **Schema Types:** User, Conversation, Message, Agent, File, Session, ACLEntry, etc.
- **ORM:** Mongoose with custom methods

**Redis (Cache & Sessions)**
- **Purpose:** Session storage, response caching, rate limiting
- **Library:** ioredis + Keyv abstraction
- **Cache Types:** MODEL_QUERIES, TOKEN_CONFIG, CONFIG_STORE, FLOWS

**MeiliSearch (Search Engine)**
- **Purpose:** Full-text search across conversations and messages
- **Features:** Fuzzy matching, typo tolerance, faceted filtering
- **Integration:** Background indexing via `indexSync.js`

**pgvector (RAG Support)**
- **Purpose:** Vector embeddings for Retrieval-Augmented Generation
- **Usage:** Optional RAG API integration

**File Storage (Multi-Strategy)**
- **Strategies:** Local, S3, Firebase, Azure Blob
- **Granular Config:** Separate strategies for avatars, images, documents
- **Processing:** Sharp for image optimization

---

### 4. Integration Layer

**AI Provider Clients**
- **Location:** `api/app/clients/`
- **Pattern:** Strategy pattern with common `BaseClient` interface
- **Providers:**
  - OpenAI (incl. Azure OpenAI, OpenRouter)
  - Anthropic (Claude with prompt caching)
  - Google (Gemini via Generative AI or Vertex AI)
  - AWS Bedrock
  - Ollama (local models)
  - Custom endpoints (OpenAI-compatible)

**Authentication Providers**
- **Location:** `api/strategies/`
- **Strategies:**
  - JWT (access + refresh tokens)
  - Local (email/password with bcrypt)
  - LDAP (with TLS/StartTLS)
  - OAuth2 (Google, GitHub, Discord, Facebook, Apple)
  - OpenID Connect (with JWKS validation)
  - SAML 2.0

**External Services**
- Email (nodemailer)
- MCP (Model Context Protocol) servers
- OAuth2 identity providers
- Enterprise directories (Entra ID/Azure AD)

---

## Monorepo Structure

```
LibreChat/
├── api/                      # Backend server
├── client/                   # Frontend application
├── packages/
│   ├── data-schemas/        # Mongoose schemas (shared)
│   ├── data-provider/       # Data services (shared)
│   ├── api/                 # MCP services
│   └── client/              # Shared UI components
├── e2e/                     # Playwright tests
├── config/                  # Configuration scripts
├── utils/                   # Utility scripts
└── package.json             # Root workspace config
```

**Workspace Benefits:**
- Shared code between frontend and backend
- Consistent dependency versions
- Single-command operations (build, test, lint)
- Type sharing between packages

---

## Request Flow: Chat Message Submission

**End-to-End Request Lifecycle:**

```
1. USER ACTION
   User types message and clicks send
   ↓
2. FRONTEND (client/src/hooks/Chat/useSubmitMessage.ts)
   - Validate input
   - Attach files if any
   - Create optimistic UI update
   - Trigger mutation
   ↓
3. API CLIENT (client/src/data-provider/mutations.ts)
   - useMessageSubmission mutation
   - POST /api/messages/chat
   - Open SSE connection for streaming
   ↓
4. BACKEND ENTRY (api/server/routes/index.js)
   - Route: POST /api/ask/:endpoint
   - Apply middleware stack
   ↓
5. MIDDLEWARE PIPELINE
   - requireJwtAuth → Authenticate user
   - checkBan → Check if user is banned
   - uaParser → Parse user agent
   - configMiddleware → Inject app config
   - messageIpLimiter → Rate limit by IP
   - messageUserLimiter → Rate limit by user
   - concurrentLimiter → Limit concurrent requests
   ↓
6. CONTROLLER (api/server/controllers/AskController.js)
   - Extract request parameters
   - Create abort controller
   - Call appropriate endpoint handler
   ↓
7. SERVICE LAYER (api/server/services/Endpoints/*/initialize.js)
   - Initialize AI provider client
   - Configure client options (model, temperature, etc.)
   - Set up streaming handlers
   ↓
8. AI CLIENT (api/app/clients/OpenAIClient.js, etc.)
   - Build message history
   - Calculate token counts
   - Apply context strategy (truncate/summarize)
   - Format messages for provider
   - Send request to AI provider
   ↓
9. AI PROVIDER (External API)
   - OpenAI/Anthropic/Google processes request
   - Returns streaming response
   ↓
10. STREAM HANDLER (api/app/clients/BaseClient.js)
    - Process chunks from AI provider
    - Extract text deltas
    - Handle tool calls (if any)
    - Emit SSE events to client
    ↓
11. DATABASE OPERATIONS
    - Save user message (api/models/Message.js::saveMessage)
    - Save AI response message
    - Update conversation metadata
    - Record token usage
    - Create transaction for billing
    ↓
12. SSE EVENTS TO CLIENT
    - Event: 'message' → Text deltas
    - Event: 'created' → Message created
    - Event: 'final' → Stream complete
    - Event: 'error' → Error occurred
    ↓
13. FRONTEND SSE HANDLER (client/src/hooks/SSE/useSSE.ts)
    - Receive SSE events
    - Update message state incrementally
    - Handle errors and retries
    ↓
14. STATE UPDATE (Jotai/Recoil atoms)
    - Update messages array
    - Update conversation state
    - Trigger re-render
    ↓
15. UI RE-RENDER
    - Display streaming text in real-time
    - Show typing indicators
    - Render tool calls/artifacts
    - Update scroll position
```

**Performance Optimizations:**
- Streaming reduces perceived latency
- Optimistic UI updates
- TanStack Query caching
- MongoDB lean queries
- Redis caching for config/models

---

## Data Flow Patterns

### 1. **Synchronous REST API** (CRUD Operations)

**Example:** Fetching conversations

```
Client Request (GET /api/convos)
    ↓
API Route (api/server/routes/convos.js)
    ↓
JWT Auth Middleware
    ↓
Controller (extract query params)
    ↓
Model Query (getConvosByCursor)
    ↓
MongoDB Query (paginated with cursor)
    ↓
Response (JSON with conversations + nextCursor)
    ↓
Client (TanStack Query caches result)
```

### 2. **Server-Sent Events (SSE)** (Streaming Responses)

**Example:** Chat message streaming

```
Client Opens SSE Connection (POST /api/ask/openAI)
    ↓
Server Accepts Connection (sets headers for SSE)
    ↓
Server Streams Events:
    - res.write(`event: message\ndata: ${JSON.stringify(chunk)}\n\n`)
    ↓
Client Receives Events (EventSource API or polyfill)
    ↓
Client Processes Chunks (update UI incrementally)
    ↓
Server Sends Final Event (marks completion)
    ↓
Connection Closes
```

### 3. **File Upload** (Multipart Form Data)

```
Client Selects File
    ↓
FormData Created (file + metadata)
    ↓
POST /api/files (with progress tracking)
    ↓
Multer Middleware (parses multipart/form-data)
    ↓
File Validation (size, type)
    ↓
Storage Service (local/S3/Firebase/Azure)
    ↓
Image Processing (Sharp for images)
    ↓
Save File Metadata (MongoDB)
    ↓
Response (file ID, URL, metadata)
    ↓
Client Updates Cache
```

---

## Security Architecture

### Authentication Flow

```
1. Login Request (email + password)
   ↓
2. Local Strategy (Passport.js)
   - Find user by email
   - Verify password (bcrypt compare)
   - Check email verified
   ↓
3. 2FA Check (if enabled)
   - Generate temporary token
   - Return 2FA_REQUIRED status
   - User provides TOTP code
   - Verify TOTP (±30s window)
   ↓
4. Session Creation
   - Generate refresh token (JWT)
   - Hash and store in MongoDB Session
   - Set expiration (7 days default)
   ↓
5. Token Generation
   - Generate access token (JWT, 15 min expiry)
   - Sign with JWT_SECRET
   - Include user ID and role
   ↓
6. Set Cookies
   - refreshToken (HttpOnly, Secure, SameSite=Strict)
   - token_provider (HttpOnly, Secure, SameSite=Strict)
   ↓
7. Return to Client
   - Access token (in response body)
   - User object (sanitized, no password)
```

### Authorization (ACL System)

```
Request with Resource Access
    ↓
Get User Principals
    - User principal (userId)
    - Group principals (user's groups)
    - Role principals (user's roles)
    - Public principal (if applicable)
    ↓
Query ACL Entries
    - Match principals + resourceType + resourceId
    - Retrieve permission bits
    ↓
Calculate Effective Permissions
    - Bitwise OR of all matching ACL entries
    - effectiveBits = entry1.permBits | entry2.permBits | ...
    ↓
Check Required Permission
    - (effectiveBits & requiredPermission) === requiredPermission
    ↓
Grant or Deny Access
```

**Permission Bits:**
- `VIEW: 1` (0001)
- `EDIT: 2` (0010)
- `DELETE: 4` (0100)
- `SHARE: 8` (1000)

---

## Scalability Considerations

### Horizontal Scaling

**Stateless API Design:**
- Session data in Redis (shared across instances)
- JWT tokens (no server-side session storage for access tokens)
- Load balancer compatible (sticky sessions not required)

**Database Scaling:**
- MongoDB replica sets for high availability
- Read replicas for query distribution
- Sharding for large datasets

**Caching Strategy:**
- Redis for session and config caching
- TanStack Query client-side caching (5-30 min stale times)
- CDN for static assets

### Vertical Scaling

**Resource Optimization:**
- Connection pooling (MongoDB, Redis)
- Streaming responses (reduce memory footprint)
- Lazy loading (code splitting on frontend)
- Lean queries (exclude unnecessary fields)

---

## Deployment Architecture

### Docker Deployment

**Services:**
```yaml
services:
  api:
    image: ghcr.io/danny-avila/librechat-dev:latest
    environment:
      - MONGO_URI
      - REDIS_URI
      - MEILI_HOST
    depends_on:
      - mongodb
      - redis
      - meilisearch

  mongodb:
    image: mongo:latest
    volumes:
      - ./data-node:/data/db

  redis:
    image: redis:alpine

  meilisearch:
    image: getmeili/meilisearch:latest
    environment:
      - MEILI_MASTER_KEY
```

### Kubernetes Deployment

**Helm Chart Available:** `/home/user/LibreChat/helm/`

**Key Resources:**
- Deployment (API pods with replicas)
- StatefulSet (MongoDB with persistent volumes)
- Service (LoadBalancer for API)
- Ingress (NGINX with TLS)
- ConfigMap (environment variables)
- Secret (API keys, JWT secrets)

---

## Design Patterns

### Backend Patterns

1. **Layered Architecture** - Clear separation: Routes → Controllers → Services → Models
2. **Strategy Pattern** - AI provider abstraction (BaseClient → OpenAIClient, etc.)
3. **Repository Pattern** - Data access abstraction (Mongoose models with custom methods)
4. **Middleware Chain** - Express middleware pipeline
5. **Factory Pattern** - Client initialization based on endpoint
6. **Observer Pattern** - SSE event emitters

### Frontend Patterns

1. **Component Composition** - Small, reusable components
2. **Custom Hooks** - Logic extraction and reuse
3. **Provider Pattern** - Context API for dependency injection
4. **Render Props** - Flexible component APIs
5. **Compound Components** - Related components working together
6. **Atomic Design** - atoms → molecules → organisms → templates → pages

---

## Technology Choices & Rationale

### Why Express.js?
- **Mature ecosystem** - Extensive middleware and plugin support
- **Flexibility** - Not opinionated, allows custom architecture
- **Performance** - Lightweight, fast for I/O-bound operations
- **SSE Support** - Easy to implement Server-Sent Events

### Why MongoDB?
- **Flexible schema** - Rapid iteration on data models
- **Scalability** - Horizontal scaling with sharding
- **JSON-native** - Natural fit for JavaScript stack
- **Rich querying** - Aggregation framework for complex queries

### Why React?
- **Component model** - Reusable UI building blocks
- **Ecosystem** - Vast library ecosystem (Radix UI, TanStack Query, etc.)
- **Performance** - Virtual DOM and concurrent rendering
- **Developer experience** - Hot reload, devtools, widespread adoption

### Why Jotai + Recoil?
- **Fine-grained reactivity** - Atom-based state (minimal re-renders)
- **TypeScript support** - Strong typing for state
- **Flexibility** - Both global and scoped state
- **Migration path** - Gradually migrating from Recoil to Jotai

### Why TanStack Query?
- **Server state management** - Purpose-built for async data
- **Automatic caching** - Intelligent cache invalidation
- **Background refetching** - Keep data fresh automatically
- **Optimistic updates** - Instant UI feedback

---

## API Design Principles

### RESTful Conventions

- **GET** - Retrieve resources (idempotent)
- **POST** - Create resources or trigger actions
- **PUT/PATCH** - Update resources
- **DELETE** - Remove resources

### Endpoint Naming

```
/api/convos           → Conversation collection
/api/convos/:id       → Specific conversation
/api/messages         → Message collection
/api/messages/:id     → Specific message
/api/agents           → Agent collection
/api/ask/:endpoint    → Chat endpoint (special)
```

### Response Format

**Success Response:**
```json
{
  "data": { /* resource or collection */ },
  "message": "Operation successful",
  "nextCursor": "cursor-for-pagination"
}
```

**Error Response:**
```json
{
  "message": "Error description",
  "type": "ValidationError",
  "statusCode": 400
}
```

---

## Performance Metrics

**Target Latencies:**
- API response time: < 100ms (cached), < 500ms (uncached)
- First message token: < 2s (provider-dependent)
- Full page load: < 3s (client bundle)
- Time to Interactive: < 5s

**Optimization Strategies:**
- Query result caching (Redis)
- Database indexing (compound indexes)
- Lean queries (exclude unnecessary fields)
- Code splitting (Vite dynamic imports)
- Image optimization (Sharp)
- Response compression (gzip/brotli)

---

## Related Documentation

- [Backend Architecture](backend/README.md) - Detailed backend implementation
- [Frontend Architecture](frontend/README.md) - Detailed frontend implementation
- [AI Provider Integration](providers/README.md) - Provider abstraction layer
- [Authentication](features/authentication.md) - Auth implementation
- [Database Layer](backend/database/) - Schema and query patterns
- [Deployment](architecture/deployment-architecture.md) - Deployment strategies

---

**Last Updated:** 2025-11-16
**Version:** 1.0.0
