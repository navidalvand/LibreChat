# Middleware Pipeline

> **Complete middleware reference for LibreChat v0.8.1-rc1**

## Overview

LibreChat's Express.js backend implements a **comprehensive middleware pipeline** with **40+ middleware functions** organized into layers:

1. **Global Middleware** - Applied to all requests (parsing, CORS, security)
2. **Route-Specific Middleware** - Applied to specific routes (auth, validation)
3. **Error Middleware** - Centralized error handling

## Source Code References

**Primary Files:**
- Middleware index: `/home/user/LibreChat/api/server/middleware/index.js` (lines 1-54)
- Server setup: `/home/user/LibreChat/api/server/index.js` (lines 81-96)
- Middleware directory: `/home/user/LibreChat/api/server/middleware/`

## Middleware Stack

### Global Middleware Layer

Applied to **all requests** in server initialization:

```javascript
// api/server/index.js (lines 81-96)

// 1. JSON parsing
app.use(express.json({ limit: '3mb' }));
app.use(express.urlencoded({ extended: true, limit: '3mb' }));

// 2. MongoDB sanitization (NoSQL injection prevention)
app.use(mongoSanitize());

// 3. CORS
app.use(cors({
  origin: process.env.DOMAIN_CLIENT,
  credentials: true
}));

// 4. Compression
app.use(compression());

// 5. Cookie parsing
app.use(cookieParser());

// 6. Session management (Passport.js)
app.use(session({ /* config */ }));
app.use(passport.initialize());
app.use(passport.session());

// 7. Security headers (Helmet)
app.use(helmet(/* config */));

// 8. Static file serving
app.use(express.static('public'));
```

**Execution Order:**
```
Request → JSON Parsing → Sanitization → CORS → Compression →
Cookies → Session → Security Headers → Route Middleware → Controller
```

## Authentication Middleware

### requireJwtAuth

**File:** `api/server/middleware/requireJwtAuth.js`

**Purpose:** Validates JWT token and loads authenticated user

**Implementation:**
```javascript
const passport = require('passport');

const requireJwtAuth = passport.authenticate('jwt', { session: false });

module.exports = requireJwtAuth;
```

**Usage:**
```javascript
router.get('/api/messages',
  requireJwtAuth,  // ← Requires valid JWT
  controller
);
```

**Behavior:**
- Extracts token from `Authorization: Bearer <token>` header
- Validates signature and expiration
- Loads user from database
- Attaches user to `req.user`
- Returns 401 if invalid/missing

**User Object:**
```javascript
req.user = {
  id: '507f1f77bcf86cd799439011',
  username: 'user@example.com',
  name: 'User Name',
  role: 'USER',
  // ... other fields
}
```

### optionalJwtAuth

**File:** `api/server/middleware/optionalJwtAuth.js`

**Purpose:** Loads user if JWT present, continues if absent

**Usage:**
```javascript
router.get('/api/public',
  optionalJwtAuth,  // ← Optional authentication
  controller
);
```

**Behavior:**
- Attempts JWT validation
- Loads user if token valid
- Continues without user if token absent/invalid
- Does NOT return 401

### requireLocalAuth

**File:** `api/server/middleware/requireLocalAuth.js`

**Purpose:** Email/password authentication via Passport LocalStrategy

**Implementation:**
```javascript
const passport = require('passport');

const requireLocalAuth = passport.authenticate('local', {
  session: false,
  failWithError: true
});

module.exports = requireLocalAuth;
```

**Usage:**
```javascript
router.post('/api/auth/login',
  requireLocalAuth,  // ← Validates email/password
  loginController
);
```

**Behavior:**
- Validates email and password
- Loads user if credentials valid
- Attaches user to `req.user`
- Returns 401 with error message if invalid

### requireLdapAuth

**File:** `api/server/middleware/requireLdapAuth.js`

**Purpose:** LDAP authentication for Active Directory integration

**Usage:**
```javascript
const ldapAuth = !!process.env.LDAP_URL;

router.post('/api/auth/login',
  ldapAuth ? requireLdapAuth : requireLocalAuth,
  loginController
);
```

**Configuration:**
- `LDAP_URL` - LDAP server URL
- `LDAP_USER_SEARCH_BASE` - Search base DN
- `LDAP_BIND_DN` - Bind DN for queries
- `LDAP_BIND_CREDENTIALS` - Bind password

## Authorization Middleware

### checkAdmin

**File:** `api/server/middleware/roles/admin.js`

**Purpose:** Verify user has admin role

**Implementation:**
```javascript
const checkAdmin = (req, res, next) => {
  if (req.user.role !== 'ADMIN') {
    return res.status(403).json({
      error: 'Admin access required'
    });
  }
  next();
};
```

**Usage:**
```javascript
router.post('/api/admin/action',
  requireJwtAuth,
  checkAdmin,  // ← Admin only
  controller
);
```

### canAccessAgentResource

**File:** `api/server/middleware/accessResources/agents.js`

**Purpose:** Resource-level access control for agents using permission bits

**Implementation:**
```javascript
const canAccessAgentResource = async (req, res, next) => {
  const { id } = req.params;
  const userId = req.user.id;
  const requiredPermission = getRequiredPermission(req.method);

  const agent = await Agent.findById(id);
  if (!agent) {
    return res.status(404).json({ error: 'Agent not found' });
  }

  // Owner has full access
  if (agent.author === userId) {
    return next();
  }

  // Check ACL
  const access = agent.permissions?.find(p => p.user === userId);
  if (!access || !(access.permission & requiredPermission)) {
    return res.status(403).json({ error: 'Access denied' });
  }

  next();
};
```

**Permission Bits:**
```javascript
READ   = 1  // 1 << 0
UPDATE = 2  // 1 << 1
DELETE = 4  // 1 << 2
SHARE  = 8  // 1 << 3

// Owner permissions
FULL = 15  // READ | UPDATE | DELETE | SHARE
```

**Usage:**
```javascript
router.get('/api/agents/:id',
  requireJwtAuth,
  canAccessAgentResource,  // ← Checks READ permission
  controller
);

router.patch('/api/agents/:id',
  requireJwtAuth,
  canAccessAgentResource,  // ← Checks UPDATE permission
  controller
);
```

### canDeleteAccount

**File:** `api/server/middleware/canDeleteAccount.js`

**Purpose:** Check if user can delete their account (configurable)

**Configuration:**
- `ALLOW_ACCOUNT_DELETION` environment variable

## Validation Middleware

### validateMessageReq

**File:** `api/server/middleware/validateMessageReq.js` (lines 1-50)

**Purpose:** Validate message route parameters

**Implementation:**
```javascript
const validateMessageReq = (req, res, next) => {
  const { conversationId, messageId } = req.params;

  if (!conversationId) {
    return res.status(400).json({
      error: 'conversationId is required'
    });
  }

  // Additional validation...
  req.conversationId = conversationId;
  req.messageId = messageId;

  next();
};
```

**Usage:**
```javascript
router.get('/api/messages/:conversationId',
  requireJwtAuth,
  validateMessageReq,  // ← Validates params
  controller
);
```

### validateRegistration

**File:** `api/server/middleware/validateRegistration.js`

**Purpose:** Validate user registration data

**Validates:**
- Email format
- Password strength (min 8 chars)
- Username uniqueness
- Email uniqueness
- Required fields

### validatePasswordReset

**File:** `api/server/middleware/validatePasswordReset.js`

**Purpose:** Validate password reset requests

**Validates:**
- Email format
- Reset token validity
- New password strength

### validateEndpoint

**File:** `api/server/middleware/validateEndpoint.js`

**Purpose:** Validate AI endpoint configuration

**Validates:**
- Endpoint name is valid
- Required API keys present
- Model configuration valid
- Parameters within limits

### validateModel

**File:** `api/server/middleware/validateModel.js`

**Purpose:** Validate AI model selection

**Validates:**
- Model exists for endpoint
- User has access to model
- Model parameters valid

### validateAgent

**Purpose:** Validate agent creation/update payloads

**Validates:**
- Name length (1-100 chars)
- Description length (max 500 chars)
- Valid tool selections
- Valid model configuration
- Valid MCP server references

## Rate Limiting Middleware

**Directory:** `api/server/middleware/limiters/`

### Login Rate Limiter

**File:** `api/server/middleware/limiters/loginLimiter.js`

**Configuration:**
```javascript
const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 5, // 5 attempts
  message: 'Too many login attempts, please try again later',
  standardHeaders: true,
  legacyHeaders: false,
  store: redisStore // Redis-backed
});
```

**Behavior:**
- 5 login attempts per 15 minutes per IP
- Returns 429 when limit exceeded
- Distributed tracking via Redis

### Register Rate Limiter

**File:** `api/server/middleware/limiters/registerLimiter.js`

**Configuration:**
```javascript
const registerLimiter = rateLimit({
  windowMs: 60 * 60 * 1000, // 1 hour
  max: 3, // 3 registrations
  message: 'Too many registration attempts'
});
```

### Message Rate Limiters

**File:** `api/server/middleware/limiters/messageLimiters.js`

**Types:**
- `messageIpLimiter` - Per IP
- `messageUserLimiter` - Per user

**Configuration:**
```javascript
const messageIpLimiter = rateLimit({
  windowMs: 60 * 1000, // 1 minute
  max: 120, // Configurable via LIMIT_MESSAGE_IP
  skipSuccessfulRequests: false
});

const messageUserLimiter = rateLimit({
  windowMs: 60 * 1000,
  max: 40, // Configurable via LIMIT_MESSAGE_USER
  keyGenerator: (req) => req.user.id
});
```

**Usage:**
```javascript
router.post('/api/messages/:conversationId',
  requireJwtAuth,
  messageIpLimiter,
  messageUserLimiter,
  controller
);
```

### Upload Rate Limiters

**File:** `api/server/middleware/limiters/uploadLimiters.js`

**Types:**
- `uploadIpLimiter`
- `uploadUserLimiter`

### Concurrent Request Limiter

**File:** `api/server/middleware/concurrentLimiter.js`

**Purpose:** Limit concurrent requests per user

**Configuration:**
```javascript
const concurrentLimiter = rateLimit({
  windowMs: 1000,
  max: 5, // Max 5 concurrent requests
  skipSuccessfulRequests: true,
  keyGenerator: (req) => req.user.id
});
```

### Tool Call Rate Limiter

**File:** `api/server/middleware/limiters/toolCallLimiter.js`

**Purpose:** Limit AI tool/function call execution rate

**Configuration:**
```javascript
const toolCallLimiter = rateLimit({
  windowMs: 60 * 1000,
  max: 50, // 50 tool calls per minute
  message: 'Too many tool calls'
});
```

### Complete Rate Limiter List

| Limiter | File | Window | Max | Purpose |
|---------|------|--------|-----|---------|
| loginLimiter | loginLimiter.js | 15 min | 5 | Login attempts |
| registerLimiter | registerLimiter.js | 1 hour | 3 | Registration |
| resetPasswordLimiter | resetPasswordLimiter.js | 1 hour | 3 | Password reset |
| verifyEmailLimiter | verifyEmailLimiter.js | 1 hour | 5 | Email verification |
| messageIpLimiter | messageLimiters.js | 1 min | 120 | Messages per IP |
| messageUserLimiter | messageLimiters.js | 1 min | 40 | Messages per user |
| uploadIpLimiter | uploadLimiters.js | 1 hour | 50 | File uploads per IP |
| uploadUserLimiter | uploadLimiters.js | 1 hour | 25 | File uploads per user |
| importIpLimiter | importLimiters.js | 1 hour | 5 | Conversation imports per IP |
| importUserLimiter | importLimiters.js | 1 hour | 3 | Conversation imports per user |
| forkIpLimiter | forkLimiters.js | 1 min | 10 | Conversation forks per IP |
| forkUserLimiter | forkLimiters.js | 1 min | 5 | Conversation forks per user |
| toolCallLimiter | toolCallLimiter.js | 1 min | 50 | Tool calls |
| concurrentLimiter | concurrentLimiter.js | 1 sec | 5 | Concurrent requests |
| createTTSLimiters | ttsLimiters.js | Varies | Varies | Text-to-speech |
| createSTTLimiters | sttLimiters.js | Varies | Varies | Speech-to-text |

## Security Middleware

### mongoSanitize

**Purpose:** Prevent NoSQL injection attacks

**Implementation:**
```javascript
const mongoSanitize = require('express-mongo-sanitize');

app.use(mongoSanitize({
  replaceWith: '_' // Replace prohibited characters
}));
```

**Protects Against:**
```javascript
// Malicious input
{ "username": { "$gt": "" } }

// Sanitized to
{ "username": { "_gt": "" } }
```

### checkBan

**File:** `api/server/middleware/checkBan.js`

**Purpose:** Check if user is banned

**Implementation:**
```javascript
const checkBan = async (req, res, next) => {
  const { email, username } = req.body;

  const banned = await BanModel.findOne({
    $or: [
      { email },
      { username },
      { ip: req.ip }
    ]
  });

  if (banned) {
    return res.status(403).json({
      error: 'Account has been banned'
    });
  }

  next();
};
```

**Ban Triggers:**
- Excessive rate limit violations
- Reported abuse
- Manual admin action

### checkInviteUser

**File:** `api/server/middleware/checkInviteUser.js`

**Purpose:** Validate invite code during registration

**Configuration:**
- `INVITE_REQUIRED` - Require invite codes
- `INVITE_EXPIRATION` - Invite expiration time

### checkDomainAllowed

**File:** `api/server/middleware/checkDomainAllowed.js`

**Purpose:** Restrict registration to specific email domains

**Configuration:**
- `ALLOWED_EMAIL_DOMAINS` - Comma-separated list

**Example:**
```
ALLOWED_EMAIL_DOMAINS=company.com,example.org
```

### moderateText

**File:** `api/server/middleware/moderateText.js`

**Purpose:** Content moderation for user messages

**Integration:**
- OpenAI Moderation API (optional)
- Custom content filters
- Profanity detection

### noIndex

**File:** `api/server/middleware/noIndex.js`

**Purpose:** Prevent search engine indexing

**Implementation:**
```javascript
const noIndex = (req, res, next) => {
  res.setHeader('X-Robots-Tag', 'noindex, nofollow');
  next();
};
```

## Request Processing Middleware

### abortMiddleware

**File:** `api/server/middleware/abortMiddleware.js`

**Purpose:** Manage abort controllers for request cancellation

**Exports:**
- `addAbortController` - Register abort controller
- `removeAbortController` - Cleanup on completion
- `abortRequest` - Cancel ongoing request

**Usage:**
```javascript
const { addAbortController, abortRequest } = require('./abortMiddleware');

router.post('/api/messages/stream',
  (req, res) => {
    const abortController = new AbortController();
    addAbortController(req.user.id, req.conversationId, abortController);

    // Stream AI response with abort capability
    streamResponse({ signal: abortController.signal });
  }
);

// Abort route
router.post('/api/messages/abort',
  (req, res) => {
    abortRequest(req.user.id, req.conversationId);
    res.status(200).json({ aborted: true });
  }
);
```

### setHeaders

**File:** `api/server/middleware/setHeaders.js`

**Purpose:** Set common response headers

**Headers:**
```javascript
res.setHeader('X-Content-Type-Options', 'nosniff');
res.setHeader('X-Frame-Options', 'DENY');
res.setHeader('X-XSS-Protection', '1; mode=block');
```

### logHeaders

**File:** `api/server/middleware/logHeaders.js`

**Purpose:** Log request headers for debugging

**Usage:**
```javascript
router.post('/api/auth/login',
  logHeaders,  // ← Debug logging
  controller
);
```

### uaParser

**File:** `api/server/middleware/uaParser.js`

**Purpose:** Parse User-Agent header

**Implementation:**
```javascript
const UAParser = require('ua-parser-js');

const uaParser = (req, res, next) => {
  const parser = new UAParser(req.headers['user-agent']);
  req.userAgent = parser.getResult();
  next();
};
```

**Parsed Data:**
```javascript
req.userAgent = {
  browser: { name: 'Chrome', version: '119.0.0.0' },
  os: { name: 'Windows', version: '10' },
  device: { type: 'desktop' }
}
```

### buildEndpointOption

**File:** `api/server/middleware/buildEndpointOption.js`

**Purpose:** Build AI endpoint configuration for request

**Constructs:**
- Model settings
- Token limits
- Tool configurations
- Prompt templates

## Configuration Middleware

### configMiddleware

**File:** `api/server/middleware/config/app.js`

**Purpose:** Load and cache application configuration

**Implementation:**
```javascript
const configMiddleware = async (req, res, next) => {
  req.appConfig = await getAppConfig();
  next();
};
```

**Config Includes:**
- Enabled endpoints
- Model configurations
- Feature flags
- File upload settings

## File Upload Middleware

### Multer Configuration

**File:** `api/server/routes/files/multer.js`

**Storage Strategies:**
```javascript
const storage = multer.diskStorage({
  destination: (req, file, cb) => {
    cb(null, 'uploads/temp/');
  },
  filename: (req, file, cb) => {
    cb(null, `${Date.now()}-${file.originalname}`);
  }
});

const upload = multer({
  storage: storage,
  limits: {
    fileSize: 10 * 1024 * 1024 // 10MB
  },
  fileFilter: (req, file, cb) => {
    // Validate file type
    if (allowedTypes.includes(file.mimetype)) {
      cb(null, true);
    } else {
      cb(new Error('Invalid file type'));
    }
  }
});
```

**File Filters:**
- `importFileFilter` - JSON files only
- `imageFileFilter` - Images only (JPEG, PNG, WebP, HEIC)
- `documentFileFilter` - Documents (PDF, DOCX, TXT, etc.)

**Usage:**
```javascript
router.post('/api/files/upload',
  requireJwtAuth,
  upload.array('files', 10),  // ← Max 10 files
  controller
);
```

## Error Handling Middleware

### Error Handler

**File:** `api/server/middleware/error.js`

**Implementation:**
```javascript
const errorHandler = (err, req, res, next) => {
  logger.error('Express error:', {
    error: err.message,
    stack: err.stack,
    url: req.url,
    method: req.method
  });

  // Handle specific error types
  if (err.name === 'ValidationError') {
    return res.status(400).json({
      error: 'Validation failed',
      details: err.message
    });
  }

  if (err.name === 'UnauthorizedError') {
    return res.status(401).json({
      error: 'Authentication required'
    });
  }

  // Generic error response
  res.status(err.status || 500).json({
    error: err.message || 'Internal server error'
  });
};

// Register as last middleware
app.use(errorHandler);
```

## Middleware Execution Flow

```
Client Request
    ↓
[Global Middleware]
1. JSON Parsing (express.json)
2. URL Encoding (express.urlencoded)
3. MongoDB Sanitization (mongoSanitize)
4. CORS (cors)
5. Compression (compression)
6. Cookie Parsing (cookieParser)
7. Session (express-session)
8. Passport Initialize
9. Helmet (security headers)
    ↓
[Route-Specific Middleware]
10. Authentication (requireJwtAuth, etc.)
11. Authorization (checkAdmin, canAccess*, etc.)
12. Rate Limiting (loginLimiter, etc.)
13. Validation (validateMessageReq, etc.)
14. Processing (abortMiddleware, etc.)
    ↓
[Controller]
Business logic execution
    ↓
[Error Middleware]
15. Error Handler (if error thrown)
    ↓
Response to Client
```

## Testing

**Middleware Tests:**
- Location: `/api/server/middleware/*.spec.js`
- Example: `checkPeoplePickerAccess.spec.js`

**Test Pattern:**
```javascript
const checkAdmin = require('./roles/admin');

describe('checkAdmin', () => {
  it('should allow admin users', () => {
    const req = { user: { role: 'ADMIN' } };
    const res = {};
    const next = jest.fn();

    checkAdmin(req, res, next);

    expect(next).toHaveBeenCalled();
  });

  it('should reject non-admin users', () => {
    const req = { user: { role: 'USER' } };
    const res = {
      status: jest.fn().mockReturnThis(),
      json: jest.fn()
    };
    const next = jest.fn();

    checkAdmin(req, res, next);

    expect(res.status).toHaveBeenCalledWith(403);
    expect(next).not.toHaveBeenCalled();
  });
});
```

## Performance Considerations

**Optimization Strategies:**

1. **Caching:**
   - Configuration cached in memory
   - Redis for distributed caching
   - Reduces database queries

2. **Early Exit:**
   - Fail fast on validation errors
   - Authentication before heavy processing

3. **Compression:**
   - gzip/brotli for large responses
   - Reduces bandwidth usage

4. **Rate Limiting:**
   - Redis-backed for distributed systems
   - Prevents abuse and maintains performance

## Security Best Practices

1. **Defense in Depth:**
   - Multiple security layers
   - NoSQL injection prevention
   - XSS protection
   - CSRF tokens

2. **Authentication & Authorization:**
   - JWT with expiration
   - Resource-level access control
   - Role-based permissions

3. **Rate Limiting:**
   - Prevent brute force
   - DDoS mitigation
   - Ban system for violations

4. **Input Validation:**
   - Sanitize all inputs
   - Validate before processing
   - Type checking

## Known Issues & TODOs

1. **Rate Limiter Distribution:**
   - Redis required for multi-instance deployments
   - In-memory fallback for single instance

2. **File Upload Validation:**
   - Virus scanning optional
   - Consider mandatory for production

## Related Documentation

- [API Routes](api-routes.md) - Complete endpoint reference
- [Authentication](../features/authentication.md) - Auth system details
- [Security Model](../architecture/security-model.md) - Security architecture

---

**Last Updated:** 2025-11-18
**Version:** 1.0.0
