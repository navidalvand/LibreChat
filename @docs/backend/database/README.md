# Database Layer

> **MongoDB schema and query patterns for LibreChat v0.8.1-rc1**

## Overview

LibreChat uses **MongoDB** as its primary database with **Mongoose ODM** for schema definition and query building. The database layer implements **30+ schemas** across three storage systems:

- **MongoDB** - Primary data storage (conversations, messages, users, agents)
- **Redis** - Caching and session management
- **MeiliSearch** - Full-text search indexing

## Source Code References

**Schema Definitions:**
- Package: `/home/user/LibreChat/packages/data-schemas/src/schema/`
- Models: `/home/user/LibreChat/api/models/`
- Database connection: `/home/user/LibreChat/api/db/connectDb.js`

**30 Schema Files:**
```
packages/data-schemas/src/schema/
├── user.ts              # User accounts and authentication
├── convo.ts             # Conversations (chat sessions)
├── message.ts           # Individual messages
├── agent.ts             # Custom agents
├── file.ts              # File metadata
├── preset.ts            # User presets
├── prompt.ts            # Prompt library
├── balance.ts           # Token balances
├── session.ts           # User sessions
├── token.ts             # API tokens
├── share.ts             # Shared conversations
├── role.ts              # User roles (RBAC)
├── aclEntry.ts          # Access control list entries
├── accessRole.ts        # Resource-level access roles
├── action.ts            # Agent actions
├── agentCategory.ts     # Agent marketplace categories
├── assistant.ts         # OpenAI assistants
├── banner.ts            # System banners
├── categories.ts        # Content categories
├── conversationTag.ts   # Conversation tags
├── group.ts             # User groups
├── key.ts               # API keys
├── memory.ts            # Agent memory
├── pluginAuth.ts        # Plugin authentication
├── project.ts           # Projects
├── promptGroup.ts       # Prompt groups
├── toolCall.ts          # Tool/function call logs
├── transaction.ts       # Transaction records
└── defaults.ts          # Default values
```

## Core Schemas

### User Schema

**File:** `packages/data-schemas/src/schema/user.ts` (lines 26-151)

**Purpose:** User accounts, authentication, and profile data

**Key Fields:**
```typescript
{
  name: String,
  username: String (lowercase, default: ''),
  email: String (required, unique, lowercase, indexed),
  emailVerified: Boolean (default: false),
  password: String (8-128 chars, bcrypt hashed, not selected),
  avatar: String,
  provider: String (default: 'local'),
  role: String (default: SystemRoles.USER),

  // OAuth IDs (unique, sparse indexes)
  googleId: String,
  facebookId: String,
  openidId: String,
  samlId: String,
  ldapId: String,
  githubId: String,
  discordId: String,

  // 2FA
  twoFactorSecret: String (not selected),
  twoFactorEnabled: Boolean (default: false),
  twoFactorBackupCodes: [BackupCodeSchema],

  // Session management
  refreshToken: SessionSchema,

  // Terms & Privacy
  termsAccepted: Boolean (default: false),
  termsAcceptedAt: Date,

  // Timestamps
  createdAt: Date,
  updatedAt: Date
}
```

**Sub-schemas:**
```typescript
// Session schema
SessionSchema {
  refreshToken: String (default: '')
}

// Backup code schema
BackupCodeSchema {
  codeHash: String (required),
  used: Boolean (default: false),
  usedAt: Date (default: null)
}
```

**Indexes:**
- `email` (unique)
- OAuth IDs (unique, sparse)

**Validation:**
- Email: Must match `/\S+@\S+\.\S+/`
- Password: 8-128 characters

**Security:**
- Password field excluded by default (`select: false`)
- 2FA secret excluded by default
- Password hashed with bcrypt before save

### Conversation Schema

**File:** `packages/data-schemas/src/schema/convo.ts` (lines 5-51)

**Purpose:** Chat conversation/session management

**Key Fields:**
```typescript
{
  conversationId: String (unique, required, indexed, meiliIndexed),
  title: String (default: 'New Chat', meiliIndexed),
  user: String (indexed, meiliIndexed),
  messages: [ObjectId] (ref: 'Message'),
  agentOptions: Mixed,
  agent_id: String,
  tags: [String] (default: [], meiliIndexed),
  files: [String],
  expiredAt: Date,

  // From conversationPreset defaults
  endpoint: String,
  model: String,
  modelLabel: String,
  chatGptLabel: String,
  promptPrefix: String,
  temperature: Number,
  topP: Number,
  topK: Number,
  presencePenalty: Number,
  frequencyPenalty: Number,
  maxOutputTokens: Number,
  resendFiles: Boolean,
  iconURL: String,
  greeting: String,
  spec: String,

  // Timestamps
  createdAt: Date,
  updatedAt: Date
}
```

**Indexes:**
- `conversationId` (unique)
- `user`
- `{ conversationId, user }` (compound unique)
- `{ createdAt, updatedAt }` (compound)
- `expiredAt` (TTL index with expireAfterSeconds: 0)

**TTL Expiration:**
- Conversations with `expiredAt` field are automatically deleted

**MeiliSearch Integration:**
- Fields marked `meiliIndex: true` are indexed for full-text search
- Indexed fields: conversationId, title, user, tags

### Message Schema

**File:** `packages/data-schemas/src/schema/message.ts` (lines 4-120)

**Purpose:** Individual chat messages

**Key Fields:**
```typescript
{
  messageId: String (unique, required, indexed, meiliIndexed),
  conversationId: String (required, indexed, meiliIndexed),
  user: String (required, indexed, default: null, meiliIndexed),
  model: String (default: null),
  endpoint: String,

  // Message threading
  parentMessageId: String,

  // Token tracking
  tokenCount: Number,
  summaryTokenCount: Number,

  // Content
  sender: String (meiliIndexed),
  text: String (meiliIndexed),
  summary: String,
  content: Mixed, // Structured content (artifacts, tool calls, etc.)

  // Status flags
  isCreatedByUser: Boolean (required, default: false),
  unfinished: Boolean (default: false),
  error: Boolean (default: false),
  finish_reason: String,

  // User feedback
  feedback: {
    rating: String (enum: ['thumbsUp', 'thumbsDown'], required),
    tag: Mixed,
    text: String
  },

  // MeiliSearch sync
  _meiliIndex: Boolean (default: false, not selected),

  // Timestamps
  createdAt: Date,
  updatedAt: Date
}
```

**Indexes:**
- `messageId` (unique)
- `conversationId`
- `user`
- Compound indexes for common queries

**Content Structure:**
Messages can have structured content with multiple parts:
```javascript
{
  messageId: "msg-123",
  text: "Here's the result",
  content: [
    { type: "text", text: "Response text" },
    { type: "code", code: "console.log('hello')", language: "javascript" },
    { type: "image_url", image_url: { url: "https://..." } },
    { type: "tool_call", tool_call: { id: "...", function: {...} } }
  ]
}
```

### Agent Schema

**File:** `packages/data-schemas/src/schema/agent.ts` (lines 4-120)

**Purpose:** Custom LibreChat agents with tools and MCP integration

**Key Fields:**
```typescript
{
  id: String (unique, required, indexed),
  name: String,
  description: String,
  instructions: String,
  avatar: Mixed,

  // Model configuration
  provider: String (required),
  model: String (required),
  model_parameters: Object,

  // Agent capabilities
  artifacts: String,
  access_level: Number,
  recursion_limit: Number,

  // Tools & Actions
  tools: [String],
  tool_kwargs: [Mixed],
  actions: [String],

  // Ownership & Access
  author: ObjectId (ref: 'User', required),
  authorName: String,

  // Behavior flags
  hide_sequential_outputs: Boolean,
  end_after_tools: Boolean,

  // Multi-agent workflows
  agent_ids: [String], // DEPRECATED
  edges: [Mixed], // Graph edges for agent workflows
  isCollaborative: Boolean,

  // UI & Marketplace
  conversation_starters: [String] (default: []),
  tool_resources: Mixed (default: {}),
  category: String,

  // Organization
  projectIds: [ObjectId] (ref: 'Project', indexed),

  // Versioning
  versions: [Mixed] (default: []),

  // Timestamps
  createdAt: Date,
  updatedAt: Date
}
```

**Indexes:**
- `id` (unique)
- `projectIds`

**Permissions:**
- Agents use resource-level ACL (aclEntry schema)
- Permission bits: READ (1), UPDATE (2), DELETE (4), SHARE (8)

### File Schema

**File:** `packages/data-schemas/src/schema/file.ts`

**Purpose:** File metadata and storage tracking

**Key Fields:**
```typescript
{
  file_id: String (unique, required),
  user: String (required, indexed),
  filename: String (required),
  filepath: String (required),
  type: String (required),

  // Storage metadata
  bytes: Number,
  embedded: Boolean (default: false),

  // File associations
  context: String,
  conversationId: String,

  // Image-specific
  width: Number,
  height: Number,

  // Storage strategy
  source: String (enum: ['local', 's3', 'firebase', 'azure']),

  // Timestamps
  createdAt: Date,
  updatedAt: Date
}
```

**Indexes:**
- `file_id` (unique)
- `user`
- `conversationId`

**Storage Strategies:**
- `local` - Filesystem storage
- `s3` - AWS S3 or compatible
- `firebase` - Firebase Storage
- `azure` - Azure Blob Storage

### Preset Schema

**File:** `packages/data-schemas/src/schema/preset.ts`

**Purpose:** Saved conversation presets

**Key Fields:**
```typescript
{
  presetId: String (unique, required),
  user: String (required, indexed),
  defaultPreset: Boolean (default: false),
  title: String,

  // Model configuration
  endpoint: String,
  model: String,
  modelLabel: String,

  // Generation parameters
  temperature: Number,
  topP: Number,
  topK: Number,
  maxOutputTokens: Number,
  presencePenalty: Number,
  frequencyPenalty: Number,

  // Agent configuration
  agent_id: String,
  agentOptions: Mixed,

  // Tools
  tools: [String],

  // Timestamps
  createdAt: Date,
  updatedAt: Date
}
```

### Balance Schema

**File:** `packages/data-schemas/src/schema/balance.ts`

**Purpose:** User token balance tracking

**Key Fields:**
```typescript
{
  user: String (unique, required, indexed),
  tokenCredits: Number (default: 0),

  // Timestamps
  createdAt: Date,
  updatedAt: Date
}
```

**Methods:**
- `deductBalance(amount)` - Deduct tokens
- `addBalance(amount)` - Add tokens
- `checkBalance(required)` - Check sufficient balance

### Prompt Schema

**File:** `packages/data-schemas/src/schema/prompt.ts`

**Purpose:** Prompt library for reusable prompts

**Key Fields:**
```typescript
{
  id: String (unique, required),
  title: String (required),
  prompt: String (required),
  category: String,
  author: ObjectId (ref: 'User', required),
  shared: Boolean (default: false),

  // Permissions
  permissions: [{
    user: ObjectId (ref: 'User'),
    permission: Number
  }],

  // Grouping
  groupId: String,

  // Timestamps
  createdAt: Date,
  updatedAt: Date
}
```

## Access Control Schemas

### ACL Entry Schema

**File:** `packages/data-schemas/src/schema/aclEntry.ts`

**Purpose:** Resource-level access control entries

**Key Fields:**
```typescript
{
  resourceId: String (required, indexed),
  resourceType: String (enum: ['agent', 'prompt', 'file'], required),
  userId: String (required, indexed),
  permission: Number (required), // Bit flags
  grantedBy: String,

  // Timestamps
  createdAt: Date,
  updatedAt: Date
}
```

**Permission Bits:**
```javascript
READ   = 1  // 0001
UPDATE = 2  // 0010
DELETE = 4  // 0100
SHARE  = 8  // 1000

// Examples
READ_ONLY = 1
READ_UPDATE = 3 (READ | UPDATE)
FULL_ACCESS = 15 (READ | UPDATE | DELETE | SHARE)
```

**Indexes:**
- `{ resourceId, resourceType, userId }` (compound unique)
- `resourceId`
- `userId`

### Role Schema

**File:** `packages/data-schemas/src/schema/role.ts`

**Purpose:** User role definitions for RBAC

**Key Fields:**
```typescript
{
  name: String (unique, required, enum: ['USER', 'ADMIN', 'MODERATOR']),
  description: String,
  permissions: [String],

  // Timestamps
  createdAt: Date,
  updatedAt: Date
}
```

## Database Connections

### MongoDB Connection

**File:** `api/db/connectDb.js`

**Configuration:**
```javascript
const connectDb = async () => {
  const mongoUri = process.env.MONGO_URI;

  const options = {
    // Connection pooling
    maxPoolSize: parseInt(process.env.MONGO_MAX_POOL_SIZE) || 10,
    minPoolSize: parseInt(process.env.MONGO_MIN_POOL_SIZE) || 2,

    // Timeouts
    serverSelectionTimeoutMS: 30000,
    socketTimeoutMS: 45000,

    // Buffering
    bufferCommands: false,

    // Auto-indexing (disable in production)
    autoIndex: process.env.NODE_ENV === 'development'
  };

  await mongoose.connect(mongoUri, options);

  logger.info('MongoDB connected');
};
```

**Environment Variables:**
- `MONGO_URI` - Connection string
- `MONGO_MAX_POOL_SIZE` - Max connections (default: 10)
- `MONGO_MIN_POOL_SIZE` - Min connections (default: 2)

### Redis Connection

**File:** `api/cache/getLogStores.js`

**Configuration:**
```javascript
const redis = require('redis');
const { REDIS_URI } = process.env;

const client = redis.createClient({
  url: REDIS_URI,
  socket: {
    reconnectStrategy: (retries) => Math.min(retries * 50, 500)
  }
});

await client.connect();
```

**Usage:**
- Session storage
- Rate limiting
- Caching (config, models, temporary data)

### MeiliSearch Connection

**File:** `api/lib/db/indexSync.js`

**Configuration:**
```javascript
const { MeiliSearch } = require('meilisearch');

const client = new MeiliSearch({
  host: process.env.MEILI_HOST,
  apiKey: process.env.MEILI_MASTER_KEY
});

// Create indexes
await client.createIndex('messages', { primaryKey: 'messageId' });
await client.createIndex('conversations', { primaryKey: 'conversationId' });
```

**Indexed Collections:**
- Messages
- Conversations

## Query Patterns

### User Queries

**File:** `api/models/User.js`

**Common Queries:**
```javascript
// Find user by email
const getUser = async (email) => {
  return await User.findOne({ email }).lean();
};

// Find user by ID with password
const getUserById = async (id, selectPassword = false) => {
  const query = User.findById(id);
  if (selectPassword) {
    query.select('+password');
  }
  return await query.lean();
};

// Update user
const updateUser = async (userId, updates) => {
  return await User.findByIdAndUpdate(
    userId,
    { $set: updates },
    { new: true }
  ).lean();
};
```

### Conversation Queries

**File:** `api/models/Conversation.js`

**Cursor-Based Pagination:**
```javascript
const getConvosByCursor = async (userId, options = {}) => {
  const {
    cursor,
    limit = 25,
    isArchived,
    tags,
    search,
    order = 'desc'
  } = options;

  const filter = { user: userId };

  if (typeof isArchived === 'boolean') {
    filter.isArchived = isArchived;
  }

  if (tags?.length) {
    filter.tags = { $in: tags };
  }

  if (search) {
    filter.title = { $regex: search, $options: 'i' };
  }

  if (cursor) {
    filter.updatedAt = order === 'desc'
      ? { $lt: cursor }
      : { $gt: cursor };
  }

  const conversations = await Conversation
    .find(filter)
    .sort({ updatedAt: order === 'desc' ? -1 : 1 })
    .limit(limit + 1)
    .lean();

  const hasMore = conversations.length > limit;
  if (hasMore) {
    conversations.pop();
  }

  return {
    conversations,
    hasMore,
    cursor: conversations[conversations.length - 1]?.updatedAt
  };
};
```

**Lean Queries:**
```javascript
// Returns plain JavaScript objects (faster, less memory)
const convos = await Conversation.find({ user }).lean();
```

### Message Queries

**File:** `api/models/Message.js`

**Get Messages:**
```javascript
const getMessages = async (filter, projection = '') => {
  return await Message
    .find(filter, projection)
    .sort({ createdAt: 1 })
    .lean();
};
```

**Save Message:**
```javascript
const saveMessage = async (req, message, options = {}) => {
  const { messageId, conversationId, ...data } = message;

  const saved = await Message.findOneAndUpdate(
    { messageId, conversationId, user: req.user.id },
    { $set: data },
    { upsert: true, new: true }
  ).lean();

  // Trigger MeiliSearch indexing
  if (!saved._meiliIndex) {
    await Message.updateOne(
      { messageId },
      { $set: { _meiliIndex: true } }
    );
  }

  return saved;
};
```

### Agent Queries

**File:** `api/models/Agent.js`

**With ACL Check:**
```javascript
const getAgentById = async (userId, agentId) => {
  const agent = await Agent.findOne({ id: agentId }).lean();

  if (!agent) {
    return null;
  }

  // Owner check
  if (agent.author.toString() === userId) {
    return agent;
  }

  // ACL check
  const access = await ACLEntry.findOne({
    resourceId: agentId,
    resourceType: 'agent',
    userId,
    permission: { $gte: 1 } // At least READ
  });

  return access ? agent : null;
};
```

## Indexing Strategy

### MongoDB Indexes

**User Collection:**
- Single: `email` (unique)
- Sparse unique: OAuth IDs (googleId, facebookId, etc.)

**Conversation Collection:**
- Single: `conversationId` (unique), `user`
- Compound: `{ conversationId, user }` (unique)
- Compound: `{ createdAt, updatedAt }`
- TTL: `expiredAt`

**Message Collection:**
- Single: `messageId` (unique), `conversationId`, `user`
- Compound: For frequent query patterns

**Agent Collection:**
- Single: `id` (unique)
- Single: `projectIds`

**ACL Entry Collection:**
- Compound: `{ resourceId, resourceType, userId }` (unique)
- Single: `resourceId`, `userId`

**Index Creation:**
```javascript
// Automatic (Mongoose schema definitions)
userSchema.index({ email: 1 }, { unique: true });

// Manual (database migrations)
db.messages.createIndex({ conversationId: 1, user: 1 });
```

### MeiliSearch Indexes

**Configuration:**
- Indexed fields marked with `meiliIndex: true` in schema
- Background sync process keeps search index updated
- Fuzzy search with typo tolerance

**Searchable Fields:**
- Messages: messageId, conversationId, user, sender, text
- Conversations: conversationId, title, user, tags

## Performance Optimizations

### 1. Lean Queries

**Always use `.lean()` for read-only queries:**
```javascript
// ✅ Good - Returns plain objects
const users = await User.find({}).lean();

// ❌ Avoid - Returns Mongoose documents (slower)
const users = await User.find({});
```

**Benefits:**
- 5-10x faster
- Less memory usage
- No hydration overhead

### 2. Field Projection

**Only select needed fields:**
```javascript
// ✅ Good - Only fetch needed fields
const messages = await Message.find({}, 'messageId text createdAt').lean();

// ❌ Avoid - Fetches all fields
const messages = await Message.find({}).lean();
```

### 3. Connection Pooling

**Configure optimal pool size:**
```javascript
maxPoolSize: 10,  // Max connections
minPoolSize: 2,   // Min connections
```

**Recommendations:**
- Development: maxPoolSize: 5
- Production: maxPoolSize: 10-20 (depends on traffic)

### 4. Cursor-Based Pagination

**More efficient than skip/limit:**
```javascript
// ✅ Good - Cursor pagination
const filter = cursor ? { updatedAt: { $lt: cursor } } : {};
const results = await Model.find(filter).limit(25);

// ❌ Avoid - Skip pagination (slow for large offsets)
const results = await Model.find({}).skip(1000).limit(25);
```

### 5. Compound Indexes

**Create indexes for frequent query patterns:**
```javascript
// Query pattern
const convos = await Conversation.find({
  user: userId,
  isArchived: false
}).sort({ updatedAt: -1 });

// Required index
convoSchema.index({ user: 1, isArchived: 1, updatedAt: -1 });
```

## Database Migrations

**Script Location:** `config/migrate-*.js`

**Available Migrations:**
```bash
# Agent permissions migration
npm run migrate:agent-permissions:dry-run
npm run migrate:agent-permissions

# Prompt permissions migration
npm run migrate:prompt-permissions:dry-run
npm run migrate:prompt-permissions
```

**Migration Pattern:**
```javascript
const migrateAgentPermissions = async () => {
  const agents = await Agent.find({}).lean();

  for (const agent of agents) {
    const permissions = [];

    // Create owner ACL entry
    permissions.push({
      resourceId: agent.id,
      resourceType: 'agent',
      userId: agent.author,
      permission: 15, // FULL_ACCESS
      grantedBy: agent.author
    });

    await ACLEntry.insertMany(permissions);
  }
};
```

## Testing

**Test Database:**
- Uses MongoDB Memory Server for isolation
- Each test suite gets fresh database
- No persistent data

**Setup:**
```javascript
const { MongoMemoryServer } = require('mongodb-memory-server');

beforeAll(async () => {
  const mongod = await MongoMemoryServer.create();
  const uri = mongod.getUri();
  await mongoose.connect(uri);
});

afterAll(async () => {
  await mongoose.disconnect();
  await mongod.stop();
});
```

## Security Considerations

**1. Input Sanitization:**
```javascript
// MongoDB injection prevention via mongoSanitize
app.use(mongoSanitize({
  replaceWith: '_'
}));
```

**2. Password Security:**
```javascript
// Bcrypt hashing before save
userSchema.pre('save', async function(next) {
  if (this.isModified('password')) {
    this.password = await bcrypt.hash(this.password, 10);
  }
  next();
});
```

**3. Sensitive Field Exclusion:**
```javascript
// Password not selected by default
password: {
  type: String,
  select: false
}
```

**4. User Scoping:**
```javascript
// Always filter by user
const messages = await Message.find({
  conversationId,
  user: req.user.id  // ← Prevent access to other users' data
});
```

## Known Issues & TODOs

1. **MeiliSearch Sync Lag:**
   - Background sync can lag under high load
   - Consider real-time sync for critical searches

2. **Index Management:**
   - Auto-indexing disabled in production
   - Manual index creation required

3. **Connection Pool Tuning:**
   - Default pool size may need adjustment for high traffic
   - Monitor connection usage

## Related Documentation

- [API Routes](../api-routes.md) - Database query usage in routes
- [Models](../../models/README.md) - Query method implementations
- [Configuration](../../configuration/environment-variables.md) - Database environment variables

---

**Last Updated:** 2025-11-18
**Version:** 1.0.0
