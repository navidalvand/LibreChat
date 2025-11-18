# API Routes

> **Complete API endpoint reference for LibreChat v0.8.1-rc1**

## Overview

LibreChat exposes **60+ REST API endpoints** across 26 route modules, handling everything from authentication to AI conversations, file management, and agent orchestration. All routes follow RESTful conventions and use JSON for request/response payloads (except file uploads which use multipart/form-data and streaming which uses Server-Sent Events).

## Source Code References

**Primary Files:**
- Route registration: `/home/user/LibreChat/api/server/routes/index.js` (lines 1-61)
- Server setup: `/home/user/LibreChat/api/server/index.js` (lines 116-146)

**Route Modules (26 total):**
- Authentication: `api/server/routes/auth.js`
- Messages: `api/server/routes/messages.js`
- Conversations: `api/server/routes/convos.js`
- Agents: `api/server/routes/agents/` (directory)
- Files: `api/server/routes/files/` (directory)
- MCP: `api/server/routes/mcp.js`
- User: `api/server/routes/user.js`
- Config: `api/server/routes/config.js`
- And 18 more...

## Route Categories

### 1. Authentication Routes (`/api/auth`)

**File:** `api/server/routes/auth.js` (lines 1-76)

| Method | Endpoint | Middleware | Purpose |
|--------|----------|-----------|---------|
| POST | `/api/auth/login` | loginLimiter, checkBan, requireLocalAuth/requireLdapAuth | User login with email/password or LDAP |
| POST | `/api/auth/logout` | requireJwtAuth | User logout (invalidates session) |
| POST | `/api/auth/register` | registerLimiter, checkBan, checkInviteUser, validateRegistration | New user registration |
| POST | `/api/auth/refresh` | - | Refresh JWT access token |
| POST | `/api/auth/requestPasswordReset` | resetPasswordLimiter, checkBan, validatePasswordReset | Request password reset email |
| POST | `/api/auth/resetPassword` | checkBan, validatePasswordReset | Reset password with token |
| GET | `/api/auth/2fa/enable` | requireJwtAuth | Generate 2FA QR code |
| POST | `/api/auth/2fa/verify` | requireJwtAuth | Verify TOTP code |
| POST | `/api/auth/2fa/verify-temp` | checkBan | Verify 2FA with temp token |
| POST | `/api/auth/2fa/confirm` | requireJwtAuth | Confirm 2FA setup |
| POST | `/api/auth/2fa/disable` | requireJwtAuth | Disable 2FA |
| POST | `/api/auth/2fa/backup/regenerate` | requireJwtAuth | Regenerate backup codes |
| GET | `/api/auth/graph-token` | requireJwtAuth | Get Microsoft Graph token |

**Key Features:**
- Multi-strategy authentication (Local, LDAP, OAuth2)
- TOTP-based 2FA with backup codes
- Rate limiting on sensitive endpoints
- Ban system integration

**Example Request:**
```javascript
// Login
POST /api/auth/login
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "securePassword123"
}

// Response
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "id": "507f1f77bcf86cd799439011",
    "username": "user@example.com",
    "name": "User Name",
    "role": "USER"
  }
}
```

### 2. Message Routes (`/api/messages`)

**File:** `api/server/routes/messages.js` (lines 1-331)

**All routes require JWT authentication** (`requireJwtAuth` middleware)

| Method | Endpoint | Middleware | Purpose |
|--------|----------|-----------|---------|
| GET | `/api/messages/` | - | Search/query messages with pagination |
| GET | `/api/messages/:conversationId` | validateMessageReq | Get all messages in conversation |
| GET | `/api/messages/:conversationId/:messageId` | validateMessageReq | Get specific message |
| POST | `/api/messages/:conversationId` | validateMessageReq | Save new message |
| PUT | `/api/messages/:conversationId/:messageId` | validateMessageReq | Update message text/content |
| PUT | `/api/messages/:conversationId/:messageId/feedback` | validateMessageReq | Update message feedback |
| POST | `/api/messages/artifact/:messageId` | - | Edit artifact within message |
| DELETE | `/api/messages/:conversationId/:messageId` | validateMessageReq | Delete message |

**Query Parameters (GET `/api/messages/`):**
- `conversationId` - Filter by conversation
- `messageId` - Get specific message
- `search` - Full-text search via MeiliSearch
- `cursor` - Pagination cursor
- `sortBy` - Sort field (endpoint, createdAt, updatedAt)
- `sortDirection` - asc or desc
- `pageSize` - Results per page (default: 25)

**Message Update Features:**
- Update entire message text (lines 241-250)
- Update specific content part by index (lines 251-287)
- Artifact editing with LaTeX unescaping (lines 119-194)
- Automatic token count recalculation (lines 279-284)

**Example Request:**
```javascript
// Save message
POST /api/messages/convo-123
Content-Type: application/json

{
  "messageId": "msg-456",
  "conversationId": "convo-123",
  "text": "Hello, AI!",
  "isCreatedByUser": true,
  "endpoint": "openAI"
}

// Response: 201 Created
{
  "messageId": "msg-456",
  "conversationId": "convo-123",
  "text": "Hello, AI!",
  "tokenCount": 4,
  "createdAt": "2025-11-18T10:30:00.000Z"
}
```

### 3. Conversation Routes (`/api/convos`)

**File:** `api/server/routes/convos.js` (lines 1-243)

**All routes require JWT authentication**

| Method | Endpoint | Middleware | Purpose |
|--------|----------|-----------|---------|
| GET | `/api/convos/` | - | List conversations with pagination |
| GET | `/api/convos/:conversationId` | - | Get specific conversation |
| POST | `/api/convos/gen_title` | - | Generate conversation title |
| POST | `/api/convos/update` | - | Update conversation metadata |
| POST | `/api/convos/import` | importIpLimiter, importUserLimiter, upload | Import conversations from JSON |
| POST | `/api/convos/fork` | forkIpLimiter, forkUserLimiter | Fork conversation from message |
| POST | `/api/convos/duplicate` | - | Duplicate entire conversation |
| DELETE | `/api/convos/` | - | Delete specific conversation |
| DELETE | `/api/convos/all` | - | Delete all user conversations |

**Query Parameters (GET `/api/convos/`):**
- `limit` - Results per page (default: 25)
- `cursor` - Pagination cursor
- `isArchived` - Filter archived conversations
- `tags` - Filter by tags (array)
- `search` - Search conversation titles
- `order` - Sort order (asc/desc)

**Forking Features:**
```javascript
// Fork conversation from specific message
POST /api/convos/fork
{
  "conversationId": "convo-123",
  "messageId": "msg-456",
  "option": "target", // or "branch"
  "splitAtTarget": true,
  "latestMessageId": "msg-789"
}
```

**Import/Export:**
- Import JSON file with conversation history (lines 179-195)
- Multer file upload handling
- Rate limited (IP and user-based)

### 4. Agent Routes (`/api/agents/*`)

**Directory:** `api/server/routes/agents/`

**Versions:**
- v1: `api/server/routes/agents/v1.js` - Full CRUD with ACL
- v2: `api/server/routes/agents/v2.js` - Marketplace features

#### Agent CRUD (v1)

| Method | Endpoint | Middleware | Purpose |
|--------|----------|-----------|---------|
| GET | `/api/agents/` | - | List user's agents |
| GET | `/api/agents/:id` | canAccessAgentResource | Get agent by ID |
| POST | `/api/agents/` | validateAgent | Create new agent |
| PATCH | `/api/agents/:id` | canAccessAgentResource, validateAgent | Update agent |
| DELETE | `/api/agents/:id` | canAccessAgentResource | Delete agent |
| GET | `/api/agents/:id/access` | canAccessAgentResource | Get agent access permissions |
| POST | `/api/agents/:id/share` | canAccessAgentResource | Share agent with users |
| PATCH | `/api/agents/:id/share/:shareId` | canAccessAgentResource | Update share permissions |
| DELETE | `/api/agents/:id/share/:shareId` | canAccessAgentResource | Revoke share access |

#### Agent Marketplace (v2)

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/api/agents/marketplace` | List marketplace agents |
| GET | `/api/agents/marketplace/categories` | Get agent categories |
| POST | `/api/agents/:id/install` | Install marketplace agent |

**Access Control:**
- Resource-level permissions via `canAccessAgentResource` middleware
- Permission bits: READ (1), UPDATE (2), DELETE (4), SHARE (8)
- Owner has full permissions (15)

### 5. File Routes (`/api/files/*`)

**Directory:** `api/server/routes/files/`

**Strategies:** local, s3, firebase, azure

| Method | Endpoint | Middleware | Purpose |
|--------|----------|-----------|---------|
| POST | `/api/files/upload` | uploadLimiters, upload.array | Upload files |
| GET | `/api/files/:file_id` | - | Download file |
| DELETE | `/api/files/:file_id` | - | Delete file |
| GET | `/api/files/` | - | List user files |
| POST | `/api/files/images/upload` | uploadLimiters, upload.single | Upload image |
| DELETE | `/api/files/images/:file_id` | - | Delete image |

**File Upload Features:**
- Multi-strategy storage (local, S3, Firebase, Azure)
- Granular strategy per file type (avatar, image, document)
- Image processing with Sharp
- Virus scanning integration
- Presigned URLs for S3/Azure

**Upload Limits:**
- Rate limiting per IP and user
- File size limits (configurable)
- File type filtering

### 6. MCP Routes (`/api/mcp`)

**File:** `api/server/routes/mcp.js`

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/api/mcp/servers` | List configured MCP servers |
| POST | `/api/mcp/servers/probe` | Test MCP server connection |
| GET | `/api/mcp/tools` | List available MCP tools |
| POST | `/api/mcp/tools/call` | Execute MCP tool |

**MCP (Model Context Protocol) Features:**
- Server discovery and management
- Tool listing and execution
- Streamable HTTP transport support
- Server health probing

### 7. User Routes (`/api/user`)

**File:** `api/server/routes/user.js`

| Method | Endpoint | Middleware | Purpose |
|--------|----------|-----------|---------|
| GET | `/api/user/` | requireJwtAuth | Get user profile |
| POST | `/api/user/` | requireJwtAuth | Update user profile |
| POST | `/api/user/avatar` | requireJwtAuth, upload | Update user avatar |
| DELETE | `/api/user/avatar` | requireJwtAuth | Delete user avatar |
| DELETE | `/api/user/account` | requireJwtAuth, canDeleteAccount | Delete user account |
| POST | `/api/user/verify-email` | verifyEmailLimiter | Verify email address |

### 8. Configuration Routes (`/api/config`)

**File:** `api/server/routes/config.js`

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/api/config` | Get application configuration |

**Returns:**
- Available endpoints
- Authentication methods
- Feature flags
- Model configurations
- File upload settings

### 9. OAuth Routes (`/api/oauth`)

**File:** `api/server/routes/oauth.js`

**Supported Providers:**
- Google
- GitHub
- Discord
- Facebook
- OpenID

**Routes:**
```
GET  /api/oauth/google
GET  /api/oauth/google/callback
GET  /api/oauth/github
GET  /api/oauth/github/callback
... (similar for other providers)
```

### 10. Additional Routes

#### Balance Routes (`/api/balance`)
- GET `/api/balance` - Get user token balance
- POST `/api/balance/deduct` - Deduct tokens

#### Endpoints Routes (`/api/endpoints`)
- GET `/api/endpoints` - List available AI endpoints

#### Models Routes (`/api/models`)
- GET `/api/models` - List available AI models

#### Presets Routes (`/api/presets`)
- GET `/api/presets` - List user presets
- POST `/api/presets` - Create preset
- PUT `/api/presets/:id` - Update preset
- DELETE `/api/presets/:id` - Delete preset

#### Prompts Routes (`/api/prompts`)
- GET `/api/prompts` - List prompts library
- POST `/api/prompts` - Create prompt
- PATCH `/api/prompts/:id` - Update prompt
- DELETE `/api/prompts/:id` - Delete prompt

#### Tags Routes (`/api/tags`)
- GET `/api/tags` - List conversation tags
- POST `/api/tags` - Create tag
- DELETE `/api/tags/:tag` - Delete tag

#### Search Routes (`/api/search`)
- GET `/api/search` - Full-text search (MeiliSearch)
- POST `/api/search/enable` - Enable search for conversation

#### Roles Routes (`/api/roles`)
- GET `/api/roles` - List user roles
- POST `/api/roles` - Assign role (admin only)

## Authentication & Authorization

### Authentication Methods

**JWT Authentication:**
- Primary method: `requireJwtAuth` middleware
- Token in `Authorization: Bearer <token>` header
- Validates token and loads user into `req.user`

**Optional JWT:**
- `optionalJwtAuth` for public/private routes
- Loads user if token present, continues if absent

**Local Authentication:**
- Email/password via Passport LocalStrategy
- Used in `/api/auth/login`

**LDAP Authentication:**
- Active Directory integration
- Falls back to local if LDAP unavailable

### Authorization Patterns

**Role-Based Access Control (RBAC):**
```javascript
// Admin-only route
router.post('/api/admin/action',
  requireJwtAuth,
  checkAdmin,
  controller
);
```

**Resource-Level ACL:**
```javascript
// Agent access control
router.get('/api/agents/:id',
  requireJwtAuth,
  canAccessAgentResource,
  controller
);
```

**Permission Bits:**
```
READ   = 1 << 0 = 1
UPDATE = 1 << 1 = 2
DELETE = 1 << 2 = 4
SHARE  = 1 << 3 = 8

Owner = 15 (all permissions)
Viewer = 1 (read only)
```

## Rate Limiting

**15+ Rate Limiter Types:**

| Limiter | Routes | Limit |
|---------|--------|-------|
| loginLimiter | `/api/auth/login` | 5 attempts / 15 min |
| registerLimiter | `/api/auth/register` | 3 / hour |
| resetPasswordLimiter | `/api/auth/requestPasswordReset` | 3 / hour |
| uploadLimiters | `/api/files/upload` | Configurable |
| importLimiters | `/api/convos/import` | IP + user based |
| forkLimiters | `/api/convos/fork` | IP + user based |
| messageLimiters | Message routes | Configurable |
| toolCallLimiter | Tool execution | Configurable |
| concurrentLimiter | Concurrent requests | Max 5 concurrent |

**Implementation:**
- Redis-backed distributed rate limiting
- Per-IP and per-user tracking
- Exponential backoff on violations
- Ban system integration

## Error Handling

**Standard Error Response:**
```javascript
{
  "error": "Error message",
  "details": "Optional detailed information"
}
```

**HTTP Status Codes:**
- 200 OK - Successful GET
- 201 Created - Successful POST
- 204 No Content - Successful DELETE
- 400 Bad Request - Validation error
- 401 Unauthorized - Authentication required
- 403 Forbidden - Authorization failed
- 404 Not Found - Resource not found
- 429 Too Many Requests - Rate limit exceeded
- 500 Internal Server Error - Server error

**Error Middleware:**
```javascript
// api/server/middleware/error.js
app.use((err, req, res, next) => {
  logger.error(err);
  res.status(err.status || 500).json({
    error: err.message || 'Internal server error'
  });
});
```

## Streaming Endpoints

**Server-Sent Events (SSE):**

Used for real-time AI response streaming:

```javascript
// Client initiates SSE connection
GET /api/messages/stream/:conversationId

// Server sends events:
event: message
data: {"delta": "Hello"}

event: message
data: {"delta": " world"}

event: final
data: {"messageId": "msg-123"}
```

**SSE Headers:**
```javascript
res.setHeader('Content-Type', 'text/event-stream');
res.setHeader('Cache-Control', 'no-cache');
res.setHeader('Connection', 'keep-alive');
```

## Request Validation

**Middleware:**
- `validateMessageReq` - Validates message requests
- `validateAgent` - Validates agent payloads
- `validateRegistration` - Validates registration data
- `validatePasswordReset` - Validates password reset requests
- `validateEndpoint` - Validates AI endpoint configuration

**Example Validation:**
```javascript
// api/server/middleware/validateMessageReq.js
const validateMessageReq = (req, res, next) => {
  const { conversationId, messageId } = req.params;

  if (!conversationId) {
    return res.status(400).json({
      error: 'conversationId is required'
    });
  }

  // Additional validation...
  next();
};
```

## Pagination

**Cursor-Based Pagination:**

Used in conversations and messages:

```javascript
GET /api/convos?limit=25&cursor=2025-11-18T10:30:00.000Z

// Response
{
  "conversations": [...],
  "nextCursor": "2025-11-18T09:00:00.000Z",
  "hasMore": true
}
```

**Page-Based Pagination:**

Used in some search results:

```javascript
GET /api/search?page=1&pageSize=25
```

## CORS Configuration

**Allowed Origins:**
- Configured via `DOMAIN_CLIENT` environment variable
- Supports multiple origins
- Credentials: true (for cookies)

**CORS Headers:**
```javascript
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Credentials: true
Access-Control-Allow-Methods: GET, POST, PUT, PATCH, DELETE, OPTIONS
Access-Control-Allow-Headers: Content-Type, Authorization
```

## Testing

**Route Tests:**
- Location: `/api/test/routes/`
- Uses supertest for HTTP assertions
- MongoDB Memory Server for database
- Example: `api/test/routes/messages.test.js`

**Test Pattern:**
```javascript
const request = require('supertest');
const app = require('~/server');

describe('POST /api/messages/:conversationId', () => {
  it('should save a message', async () => {
    const response = await request(app)
      .post('/api/messages/test-convo')
      .set('Authorization', `Bearer ${token}`)
      .send({ text: 'Test message' })
      .expect(201);

    expect(response.body.messageId).toBeDefined();
  });
});
```

## Performance Considerations

**Optimization Strategies:**

1. **Lean Queries:**
   ```javascript
   // Returns plain objects, not Mongoose documents
   await Conversation.find({ user }).lean();
   ```

2. **Field Projection:**
   ```javascript
   // Only fetch needed fields
   await Message.find({}, 'messageId text createdAt');
   ```

3. **Pagination:**
   - Cursor-based for efficiency
   - Limit results per page

4. **Caching:**
   - Redis for frequently accessed data
   - Configuration caching
   - Model list caching

5. **Compression:**
   - gzip/brotli for responses
   - Applied via middleware

## Security Considerations

**Input Validation:**
- All user inputs validated
- MongoDB injection prevention via mongoSanitize
- XSS prevention via sanitization

**Authentication:**
- JWT with expiration
- Refresh token rotation
- HTTPS enforced in production

**Rate Limiting:**
- Prevents brute force attacks
- DDoS mitigation
- Ban system for violations

**File Uploads:**
- File type validation
- Size limits enforced
- Virus scanning (optional)
- Presigned URLs for direct uploads

## Known Issues & TODOs

1. **Import Optimization** (convos.js:187):
   - TODO: Optimize to return imported conversations

2. **Concurrent Request Handling**:
   - Max 5 concurrent requests per user
   - May need adjustment for high-traffic scenarios

3. **Search Performance**:
   - MeiliSearch indexing can lag
   - Background sync required

## Related Documentation

- [Middleware](middleware.md) - Middleware pipeline details
- [Controllers](controllers/README.md) - Controller implementations
- [Database](database/README.md) - Database models and queries
- [Authentication](../features/authentication.md) - Auth system details

---

**Last Updated:** 2025-11-18
**Version:** 1.0.0
