# Authentication System

> **Multi-strategy authentication for LibreChat v0.8.1-rc1**

## Overview

LibreChat implements a comprehensive authentication system with **11 authentication strategies**, JWT-based session management, optional 2FA with TOTP, and refresh token rotation.

## Authentication Strategies

### Supported Strategies (11 Total)

1. **Local** - Email/password (default)
2. **LDAP** - Active Directory integration
3. **Google OAuth2**
4. **GitHub OAuth2**
5. **Discord OAuth2**
6. **Facebook OAuth2**
7. **OpenID Connect**
8. **SAML**
9. **Apple ID**
10. **Azure AD**
11. **Custom OAuth2**

## Source Code References

**Strategy Files:** `/home/user/LibreChat/api/strategies/`
- JWT: `api/strategies/jwtStrategy.js`
- Local: `api/strategies/localStrategy.js`
- LDAP: `api/strategies/ldapStrategy.js`
- Google: `api/strategies/googleStrategy.js`
- GitHub: `api/strategies/githubStrategy.js`

**Routes:** `api/server/routes/auth.js` (lines 1-76)
**Controllers:** `api/server/controllers/AuthController.js`

## JWT Authentication (Primary Method)

### JWT Strategy

**File:** `api/strategies/jwtStrategy.js` (lines 7-33)

```javascript
const JwtStrategy = require('passport-jwt').Strategy;
const ExtractJwt = require('passport-jwt').ExtractJwt;
const User = require('~/models/User');

const options = {
  jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
  secretOrKey: process.env.JWT_SECRET
};

passport.use(
  new JwtStrategy(options, async (jwtPayload, done) => {
    try {
      const user = await User.findById(jwtPayload.id)
        .select('-password -twoFactorSecret')
        .lean();

      if (user) {
        return done(null, user);
      }
      return done(null, false);
    } catch (err) {
      return done(err, false);
    }
  })
);
```

### Token Structure

**JWT Payload:**
```json
{
  "id": "507f1f77bcf86cd799439011",
  "username": "user@example.com",
  "iat": 1700000000,
  "exp": 1700003600
}
```

**Token Expiration:**
- Access token: 1 hour (configurable via `JWT_EXPIRY`)
- Refresh token: 7 days (configurable via `REFRESH_TOKEN_EXPIRY`)

## Login Flow

### Standard Login (Local Strategy)

**Route:** `POST /api/auth/login`

**File:** `api/server/routes/auth.js` (lines 34-42)

**Flow:**
```
Client
  ↓ POST /api/auth/login { email, password }
Login Limiter (5 attempts / 15 min)
  ↓
Check Ban Status
  ↓
Local/LDAP Strategy (validate credentials)
  ↓
Login Controller
  ↓
Generate JWT Access Token
  ↓
Generate Refresh Token (stored in DB)
  ↓
Response { token, user }
```

**Request:**
```json
POST /api/auth/login
{
  "email": "user@example.com",
  "password": "securePassword123"
}
```

**Response (Success):**
```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "id": "507f1f77bcf86cd799439011",
    "username": "user@example.com",
    "name": "User Name",
    "role": "USER",
    "avatar": "https://..."
  }
}
```

### Local Strategy Implementation

**File:** `api/strategies/localStrategy.js`

```javascript
passport.use(
  new LocalStrategy(
    {
      usernameField: 'email',
      passwordField: 'password',
      session: false
    },
    async (email, password, done) => {
      try {
        const user = await User.findOne({ email })
          .select('+password')
          .lean();

        if (!user) {
          return done(null, false, { message: 'Invalid credentials' });
        }

        const isMatch = await bcrypt.compare(password, user.password);
        if (!isMatch) {
          return done(null, false, { message: 'Invalid credentials' });
        }

        return done(null, user);
      } catch (err) {
        return done(err);
      }
    }
  )
);
```

## OAuth2 Authentication

### Google OAuth2

**Configuration:**
```env
GOOGLE_CLIENT_ID=your_client_id
GOOGLE_CLIENT_SECRET=your_client_secret
GOOGLE_CALLBACK_URL=/api/oauth/google/callback
```

**Flow:**
```
User clicks "Login with Google"
  ↓
Redirect to Google OAuth
  ↓
User authorizes
  ↓
Redirect to callback URL with auth code
  ↓
Exchange code for user info
  ↓
Find or create user in database
  ↓
Generate JWT tokens
  ↓
Redirect to client with token
```

**Routes:**
```javascript
GET  /api/oauth/google           // Initiate OAuth
GET  /api/oauth/google/callback  // OAuth callback
```

### GitHub OAuth2

**Configuration:**
```env
GITHUB_CLIENT_ID=your_client_id
GITHUB_CLIENT_SECRET=your_client_secret
GITHUB_CALLBACK_URL=/api/oauth/github/callback
```

**Similar flow to Google OAuth2**

## LDAP Authentication

### Configuration

**Environment Variables:**
```env
LDAP_URL=ldap://ldap.example.com:389
LDAP_USER_SEARCH_BASE=ou=users,dc=example,dc=com
LDAP_BIND_DN=cn=admin,dc=example,dc=com
LDAP_BIND_CREDENTIALS=admin_password
LDAP_USERNAME_FIELD=uid
LDAP_EMAIL_FIELD=mail
```

### LDAP Strategy

**File:** `api/strategies/ldapStrategy.js`

```javascript
passport.use(
  new LdapStrategy(
    {
      server: {
        url: process.env.LDAP_URL,
        bindDN: process.env.LDAP_BIND_DN,
        bindCredentials: process.env.LDAP_BIND_CREDENTIALS,
        searchBase: process.env.LDAP_USER_SEARCH_BASE,
        searchFilter: `(${process.env.LDAP_USERNAME_FIELD}={{username}})`
      }
    },
    async (ldapUser, done) => {
      try {
        const email = ldapUser[process.env.LDAP_EMAIL_FIELD];
        let user = await User.findOne({ email });

        if (!user) {
          user = await User.create({
            email,
            name: ldapUser.cn || ldapUser.displayName,
            provider: 'ldap',
            ldapId: ldapUser.dn
          });
        }

        return done(null, user);
      } catch (err) {
        return done(err);
      }
    }
  )
);
```

## Two-Factor Authentication (2FA)

### TOTP-Based 2FA

**Algorithm:** Time-based One-Time Password (RFC 6238)

### 2FA Endpoints

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/api/auth/2fa/enable` | Generate QR code for TOTP setup |
| POST | `/api/auth/2fa/verify` | Verify TOTP code during setup |
| POST | `/api/auth/2fa/confirm` | Confirm and enable 2FA |
| POST | `/api/auth/2fa/disable` | Disable 2FA |
| POST | `/api/auth/2fa/backup/regenerate` | Regenerate backup codes |
| POST | `/api/auth/2fa/verify-temp` | Verify 2FA with temp token |

### 2FA Setup Flow

**File:** `api/server/controllers/TwoFactorController.js`

```
1. User requests 2FA enable
   GET /api/auth/2fa/enable
   ↓
2. Server generates secret
   Returns { secret, qrCode (base64), backupCodes }
   ↓
3. User scans QR code with authenticator app
   (Google Authenticator, Authy, etc.)
   ↓
4. User enters TOTP code to verify
   POST /api/auth/2fa/verify { code }
   ↓
5. Server validates code
   ↓
6. User confirms 2FA enable
   POST /api/auth/2fa/confirm { code }
   ↓
7. Server enables 2FA, stores secret (encrypted)
   Returns success + backup codes
```

### 2FA Login Flow

```
1. User submits email/password
   POST /api/auth/login
   ↓
2. Server validates credentials
   ↓
3. If 2FA enabled:
   Return { requiresTwoFactor: true, tempToken }
   ↓
4. User enters TOTP code
   POST /api/auth/2fa/verify-temp { code, tempToken }
   ↓
5. Server validates code
   ↓
6. Return full JWT tokens
```

### Backup Codes

**Purpose:** Recovery if TOTP device lost

**Format:**
- 10 backup codes
- 8-character alphanumeric
- One-time use only

**Storage:**
```typescript
{
  codeHash: String,      // bcrypt hash
  used: Boolean,         // Used flag
  usedAt: Date          // Timestamp when used
}
```

**Usage:**
- User can use backup code instead of TOTP
- Code marked as used after authentication
- Cannot be reused

## Refresh Token Rotation

### Refresh Flow

**Route:** `POST /api/auth/refresh`

**Process:**
```
1. Client sends refresh token
   POST /api/auth/refresh
   Headers: { Authorization: Bearer <refreshToken> }
   ↓
2. Server validates refresh token
   - Check signature
   - Check expiration
   - Check token exists in DB
   ↓
3. Generate new access token
   ↓
4. Generate new refresh token
   ↓
5. Invalidate old refresh token (rotation)
   ↓
6. Return new tokens
   { token: <newAccessToken>, refreshToken: <newRefreshToken> }
```

**Security Benefits:**
- Limits refresh token lifetime
- Detects token reuse (possible theft)
- Automatic logout on suspicious activity

## Session Management

### Session Storage

**Database:** User schema (MongoDB)

```typescript
{
  refreshToken: {
    type: String,
    default: ''
  }
}
```

**Redis Session Store:**
```javascript
const session = require('express-session');
const RedisStore = require('connect-redis')(session);

app.use(session({
  store: new RedisStore({ client: redisClient }),
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
  cookie: {
    secure: process.env.NODE_ENV === 'production',
    httpOnly: true,
    maxAge: 7 * 24 * 60 * 60 * 1000 // 7 days
  }
}));
```

### Logout

**Route:** `POST /api/auth/logout`

**Process:**
```javascript
const logoutController = async (req, res) => {
  try {
    // Invalidate refresh token
    await User.findByIdAndUpdate(
      req.user.id,
      { $set: { refreshToken: '' } }
    );

    // Destroy session
    req.session.destroy();

    res.status(200).json({ message: 'Logged out successfully' });
  } catch (error) {
    res.status(500).json({ error: 'Logout failed' });
  }
};
```

## Password Management

### Password Reset Flow

**Request Reset:**
```
POST /api/auth/requestPasswordReset
{ email: 'user@example.com' }
  ↓
Generate reset token (random, time-limited)
  ↓
Store token in database
  ↓
Send reset email with token
```

**Reset Password:**
```
POST /api/auth/resetPassword
{ token: 'abc123', newPassword: 'newPassword123' }
  ↓
Validate token (exists, not expired)
  ↓
Hash new password (bcrypt)
  ↓
Update user password
  ↓
Invalidate reset token
  ↓
Return success
```

### Password Security

**Hashing:** bcrypt with 10 rounds

```javascript
userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) {
    return next();
  }

  try {
    const salt = await bcrypt.genSalt(10);
    this.password = await bcrypt.hash(this.password, salt);
    next();
  } catch (error) {
    next(error);
  }
});
```

**Requirements:**
- Minimum 8 characters
- Maximum 128 characters
- No specific complexity requirements (user choice)

## Security Features

### 1. Rate Limiting

**Login Attempts:**
- 5 attempts per 15 minutes per IP
- Middleware: `loginLimiter`

**Registration:**
- 3 attempts per hour per IP
- Middleware: `registerLimiter`

**Password Reset:**
- 3 attempts per hour per IP
- Middleware: `resetPasswordLimiter`

### 2. Ban System

**Middleware:** `checkBan`

**Checks:**
- Email banned
- Username banned
- IP banned

**Triggers:**
- Excessive rate limit violations
- Manual admin action
- Reported abuse

### 3. CSRF Protection

**Express Session:**
```javascript
app.use(session({
  cookie: {
    sameSite: 'strict',
    httpOnly: true,
    secure: true
  }
}));
```

### 4. Password Storage

- Never stored in plain text
- bcrypt hashed (10 rounds)
- Password field excluded from queries by default

### 5. Token Security

**JWT Secret:**
- Strong random secret
- Environment variable
- Never committed to repo

**Token Transmission:**
- HTTPS only in production
- httpOnly cookies for refresh tokens
- Bearer tokens in Authorization header

## Frontend Integration

### Auth Context

**Location:** `client/src/hooks/AuthContext.jsx`

```typescript
const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(null);

  const login = async (credentials) => {
    const response = await fetch('/api/auth/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(credentials)
    });
    const data = await response.json();
    setToken(data.token);
    setUser(data.user);
    localStorage.setItem('token', data.token);
  };

  const logout = () => {
    setToken(null);
    setUser(null);
    localStorage.removeItem('token');
  };

  return (
    <AuthContext.Provider value={{ user, token, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
};
```

### Protected Routes

```typescript
const ProtectedRoute = ({ children }) => {
  const { token } = useAuth();

  if (!token) {
    return <Navigate to="/login" />;
  }

  return children;
};
```

## Configuration

### Environment Variables

```env
# JWT
JWT_SECRET=your_jwt_secret_here
JWT_EXPIRY=1h
REFRESH_TOKEN_EXPIRY=7d
SESSION_SECRET=your_session_secret

# OAuth2 - Google
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
GOOGLE_CALLBACK_URL=/api/oauth/google/callback

# OAuth2 - GitHub
GITHUB_CLIENT_ID=
GITHUB_CLIENT_SECRET=

# LDAP
LDAP_URL=ldap://ldap.example.com
LDAP_USER_SEARCH_BASE=ou=users,dc=example,dc=com
LDAP_BIND_DN=cn=admin,dc=example,dc=com
LDAP_BIND_CREDENTIALS=

# Security
ALLOW_REGISTRATION=true
ALLOW_EMAIL_LOGIN=true
ALLOW_SOCIAL_LOGIN=true
SESSION_EXPIRY=1000 * 60 * 15  # 15 minutes
```

## Testing

**Authentication Tests:**
- Location: `api/test/auth/`
- Test login, logout, token refresh
- Test 2FA setup and verification
- Test password reset flow

## Related Documentation

- [API Routes](../backend/api-routes.md) - Auth endpoints
- [Middleware](../backend/middleware.md) - Auth middleware
- [Security Model](../architecture/security-model.md) - Security architecture

---

**Last Updated:** 2025-11-18
**Version:** 1.0.0
