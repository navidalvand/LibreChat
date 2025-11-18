# LibreChat - AI Assistant Development Guide

> **Version:** 0.8.1-rc1
> **Last Updated:** 2025-11-16
> **Purpose:** Comprehensive guide for AI assistants working with the LibreChat codebase

---

## Table of Contents

1. [Repository Overview](#repository-overview)
2. [Codebase Structure](#codebase-structure)
3. [Technology Stack](#technology-stack)
4. [Development Workflows](#development-workflows)
5. [Coding Conventions](#coding-conventions)
6. [Testing Standards](#testing-standards)
7. [Git Workflow](#git-workflow)
8. [Key Architectural Patterns](#key-architectural-patterns)
9. [Common Development Tasks](#common-development-tasks)
10. [Important Notes for AI Assistants](#important-notes-for-ai-assistants)

---

## Repository Overview

**LibreChat** is a sophisticated, production-ready open-source AI chat platform that brings together multiple AI models (OpenAI, Anthropic, Google, Azure, AWS Bedrock, and more) into a unified interface inspired by ChatGPT.

### Key Features
- Multi-provider AI integration (10+ providers)
- LibreChat Agents with MCP (Model Context Protocol) support
- Code Interpreter API with sandboxed execution
- Web Search with customizable reranking
- Image generation & editing
- Multi-user authentication (OAuth2, LDAP, SAML)
- Enterprise-ready with RBAC, 2FA
- i18n support (30+ languages)
- Progressive Web App capabilities

### Architecture Type
- **Monorepo** using npm workspaces
- **Full-stack JavaScript/TypeScript**
- **Backend:** Node.js + Express
- **Frontend:** React + Vite
- **Databases:** MongoDB (primary), Redis (cache/sessions), MeiliSearch (search), pgvector (RAG)

---

## Codebase Structure

```
LibreChat/
├── api/                          # Backend Node.js/Express server
│   ├── server/
│   │   ├── controllers/          # Request handlers
│   │   ├── middleware/           # Express middleware
│   │   ├── routes/              # API routes (auth, messages, agents, etc.)
│   │   ├── services/            # Business logic (MCP, Files, Endpoints, etc.)
│   │   └── index.js             # Main server entry point
│   ├── app/clients/             # AI client implementations (OpenAI, Ollama, etc.)
│   ├── models/                  # Mongoose models (Agent, Conversation, Message, etc.)
│   ├── db/                      # Database connection and indexing
│   ├── strategies/              # Passport authentication strategies
│   ├── lib/                     # Shared libraries
│   ├── cache/                   # Caching logic
│   └── test/                    # Backend tests
│
├── client/                       # Frontend React application
│   ├── src/
│   │   ├── components/          # UI components (Chat, Messages, Agents, etc.)
│   │   ├── routes/              # React Router routes
│   │   ├── hooks/               # Custom React hooks
│   │   ├── store/               # State management (Jotai atoms)
│   │   ├── Providers/           # Context providers
│   │   ├── locales/             # i18n translations
│   │   ├── data-provider/       # API data fetching
│   │   ├── utils/               # Utility functions
│   │   ├── constants/           # App constants
│   │   └── @types/              # TypeScript type definitions
│   ├── public/                  # Static assets
│   └── test/                    # Frontend tests
│
├── packages/                     # Shared monorepo packages
│   ├── api/                     # MCP services for LibreChat
│   ├── client/                  # Shared client components
│   ├── data-provider/           # Data services for LibreChat apps
│   └── data-schemas/            # Mongoose schemas and models
│
├── e2e/                         # End-to-end Playwright tests
│   ├── specs/                   # Test specifications
│   └── playwright.config.*.ts   # Multiple test configurations
│
├── config/                      # Configuration scripts & utilities
│   ├── create-user.js          # User management scripts
│   ├── add-balance.js          # Token balance management
│   ├── update.js               # Update automation
│   └── translations/           # Translation utilities
│
├── .github/workflows/           # CI/CD workflows (20+ workflows)
├── .husky/                      # Git hooks (pre-commit)
├── helm/                        # Kubernetes Helm charts
└── utils/                       # Utility scripts
```

---

## Technology Stack

### Backend (API)

**Core:**
- Node.js v20+ (Bun support available)
- Express.js
- MongoDB with Mongoose ODM
- Redis (ioredis) for caching/sessions
- MeiliSearch for full-text search

**Authentication:**
- Passport.js (multi-strategy)
- JWT, Local, LDAP, OAuth2 (Google, GitHub, Discord, Facebook, Apple)
- SAML support
- 2FA integration

**AI SDKs:**
- OpenAI SDK (5.8.2)
- Anthropic SDK
- Google Generative AI
- AWS SDK (S3, Bedrock)
- Azure (Identity, Storage, Search)
- LangChain Core
- Model Context Protocol (MCP) SDK
- LibreChat Agents

**File Storage:**
- Strategies: Local, S3, Firebase, Azure Blob
- Image processing: Sharp
- Upload handling: Multer v2.0.0+

**Other Key Dependencies:**
- Winston (logging)
- Tiktoken (token counting)
- Socket.io alternatives (SSE for streaming)

### Frontend (Client)

**Core:**
- React 18
- Vite 6 (build tool with extensive chunking)
- JavaScript/TypeScript (mixed, gradual migration)

**State Management:**
- Jotai (primary, atom-based)
- Recoil (legacy, still in use)
- TanStack Query (React Query) for server state

**Routing & Navigation:**
- React Router DOM v6

**UI Components:**
- Radix UI (headless components)
- Headless UI
- Ariakit
- Tailwind CSS with custom theme
- Class Variance Authority (CVA)

**Forms & Validation:**
- React Hook Form

**Internationalization:**
- i18next + react-i18next
- Locize for translation management

**Code/Markdown Rendering:**
- React Markdown
- Remark/Rehype plugins
- KaTeX for math rendering
- Highlight.js for syntax highlighting
- CodeSandbox Sandpack for live code editing

**Other Features:**
- PWA support (VitePWA)
- Drag & Drop (react-dnd)
- Animations (Framer Motion, react-spring)
- Speech Recognition
- HEIC image conversion

### Testing

**Unit Testing:**
- Jest 29
- @testing-library/react
- MongoDB Memory Server (for API tests)
- jsdom (for client tests)

**E2E Testing:**
- Playwright
- Visual regression testing
- Accessibility testing (axe-core)

### Build Tools

- npm workspaces (monorepo management)
- Bun (optional, faster builds)
- Rollup (package bundling)
- Babel + ts-jest (transforms)

---

## Development Workflows

### Quick Start Commands

```bash
# Install dependencies
npm install

# Development
npm run backend:dev          # Start backend with nodemon
npm run frontend:dev         # Start frontend dev server
npm run b:api:dev           # Bun: backend with watch mode
npm run b:client:dev        # Bun: frontend dev server

# Production
npm run backend             # Start production backend
npm run frontend            # Build production frontend
npm run b:client            # Bun: build frontend

# Building
npm run build:packages      # Build all shared packages
npm run build:client        # Build client only
npm run build:api           # Build API package

# Testing
npm run test:client         # Run client unit tests
npm run test:api            # Run API unit tests
npm run e2e                 # Run Playwright E2E tests
npm run e2e:headed          # Run E2E with visible browser
npm run e2e:a11y            # Run accessibility tests
npm run e2e:debug           # Debug E2E tests

# Code Quality
npm run lint                # Run ESLint
npm run lint:fix            # Auto-fix linting issues
npm run format              # Format with Prettier

# Database & User Management
npm run create-user         # Create new user
npm run add-balance         # Add token balance
npm run list-users          # List all users
npm run reset-password      # Reset user password

# Docker
npm run start:deployed      # Start with deploy-compose.yml
npm run stop:deployed       # Stop deployed containers
npm run update:deployed     # Update deployed instance
```

### CI/CD Workflows

**20+ GitHub Actions workflows covering:**

- **Testing:** ESLint, client tests, API tests, E2E tests, a11y tests
- **Build:** Main/dev/tag Docker images, Azure containers
- **Deploy:** Production and development deployments
- **Maintenance:** i18n sync, unused package cleanup, Helm chart validation
- **Code Review:** Backend/frontend automated reviews

**Key Workflows:**
- `eslint-ci.yml` - Linting with SARIF output
- `client.yml` / `data-provider.yml` / `data-schemas.yml` - Unit tests
- `playwright.yml` - E2E testing
- `a11y.yml` - Accessibility testing
- `main-image-workflow.yml` - Docker image builds

---

## Coding Conventions

### Code Style

**Enforced via Prettier + ESLint:**
- **Line width:** 100 characters
- **Quotes:** Single quotes (JS/TS), double quotes (JSX)
- **Semicolons:** Always
- **Trailing commas:** Always
- **Arrow parens:** Always
- **Tab width:** 2 spaces
- **No tabs:** Spaces only

**File locations:**
- `.prettierrc` - Prettier configuration
- `eslint.config.mjs` - ESLint 9 flat config

### Linting Rules

**Key ESLint Rules:**
- `import/no-cycle: error` - **Import cycles are forbidden**
- `import/no-self-import: error` - No self-imports
- `no-console: off` - Console allowed (Winston logger preferred)
- `prettier/prettier: error` - Prettier violations are errors
- `react-hooks/rules-of-hooks: error` - Hooks rules enforced
- `react-hooks/exhaustive-deps: warn` - Dependency warnings
- `i18next/no-literal-string: error` - No hardcoded strings in JSX (i18n required)

**TypeScript-specific:**
- `@typescript-eslint/no-explicit-any: off` - `any` allowed (gradual migration)
- `@typescript-eslint/ban-ts-comment: off` - ts-ignore allowed
- Unused vars with `_` prefix are ignored

**Accessibility:**
- jsx-a11y rules enabled but many are warnings (gradual improvement)

### File Organization

**Naming Conventions:**
- **Components:** PascalCase (`AgentCard.tsx`)
- **Utilities:** camelCase (`formatDate.js`)
- **Constants:** SCREAMING_SNAKE_CASE (`API_TIMEOUT`)
- **Files:** Mixed (kebab-case or camelCase, be consistent within directories)

**Directory Structure:**
- Feature-based folders (`Agents/`, `Messages/`, `Chat/`)
- Co-located tests (`__tests__/`, `*.spec.ts`, `*.test.ts`)
- Barrel exports via `index.js`/`index.ts`
- Types in `@types/` or co-located `types.ts`

### TypeScript Usage

**Current State:** Mixed JS/TS, gradual migration
- API: Mostly JavaScript with some TypeScript
- Client: Heavy TypeScript usage
- Packages: Full TypeScript

**Best Practices:**
- Use TypeScript for new code when possible
- Add JSDoc comments for JavaScript code
- Export types from shared packages
- Multiple tsconfig.json files for different contexts

### Import Conventions

**Import Order (enforced by ESLint):**
1. Node built-ins (`fs`, `path`)
2. External packages (`react`, `express`)
3. Internal packages (`@librechat/api`, `librechat-data-provider`)
4. Relative imports (`~/models`, `./components`)

**Module Aliases:**
- `~/` maps to API root (`api/`)
- `@/` (client) maps to `client/src/`
- Package scopes: `@librechat/api`, `@librechat/data-schemas`, etc.

### Environment Variables

**Configuration:**
- `.env.example` - Template with 500+ lines of documentation
- `librechat.example.yaml` - Application configuration schema
- Load via `dotenv` at server startup

**Key Variables:**
- `MONGO_URI` - MongoDB connection
- `REDIS_URI` - Redis connection
- `DOMAIN_CLIENT` / `DOMAIN_SERVER` - Application domains
- `DEBUG_LOGGING` - Enable verbose logging
- Provider API keys (OpenAI, Anthropic, etc.)

---

## Testing Standards

### Unit Testing

**Framework:** Jest 29

**API Tests:**
- Location: `/api/test/`
- MongoDB Memory Server for isolation
- Custom mocks in `__mocks__/`
- Timeout: 30 seconds
- Run: `npm run test:api`

**Client Tests:**
- Location: `/client/test/`
- @testing-library/react
- jsdom environment
- jest-canvas-mock for canvas elements
- Run: `npm run test:client`

**Package Tests:**
- Each package has its own Jest config
- `data-provider`: `/packages/data-provider/specs/`
- `data-schemas`: `/packages/data-schemas/specs/`
- `api`: `/packages/api/specs/`

**Testing Patterns:**
```javascript
// API test example
describe('MessageController', () => {
  beforeEach(async () => {
    // Setup with MongoDB Memory Server
  });

  it('should create a message', async () => {
    // Test implementation
  });
});

// Client test example
import { render, screen } from '@testing-library/react';

describe('AgentCard', () => {
  it('renders agent name', () => {
    render(<AgentCard agent={mockAgent} />);
    expect(screen.getByText('Test Agent')).toBeInTheDocument();
  });
});
```

### E2E Testing

**Framework:** Playwright

**Test Locations:**
- `/e2e/specs/` - Test files
- `landing.spec.ts`, `messages.spec.ts`, `nav.spec.ts`, etc.

**Configurations:**
- `playwright.config.local.ts` - Local development
- `playwright.config.ts` - CI environment
- `playwright.config.a11y.ts` - Accessibility testing

**Features:**
- Visual regression (screenshot comparison)
- Authentication state management
- Parallel execution
- Mobile/tablet/desktop viewports

**Running Tests:**
```bash
npm run e2e              # Headless
npm run e2e:headed       # With browser
npm run e2e:debug        # Debug mode
npm run e2e:codegen      # Record interactions
npm run e2e:a11y         # Accessibility
```

### Accessibility Testing

**Tools:**
- @axe-core/playwright
- eslint-plugin-jsx-a11y

**A11y Test Suite:**
- `/e2e/specs/a11y.spec.ts`
- Automated WCAG compliance checks
- Run: `npm run e2e:a11y`

**Guidelines:**
- Use semantic HTML
- Provide ARIA labels where needed
- Ensure keyboard navigation
- Test with screen readers when possible

---

## Git Workflow

### Branch Strategy

**Main Branches:**
- `main` - Production-ready code
- `dev` - Development branch (if exists)

**Feature Branches:**
- Create from main/dev
- Naming: `feature/description`, `fix/issue-number`, etc.

### Commit Messages

**Convention:** Gitmoji + conventional commits

**Format:** `emoji type: description`

**Examples:**
```
✨ feat: Add MCP support for Streamable HTTP Transport
🔒 feat: Add Content Security Policy using Helmet middleware
💬 fix: update aria-label for accessibility in ConvoLink component
📊 chore: Remove Old Helm Chart
🌍 i18n: Add Danish and Czech localization support
```

**Common Emojis:**
- ✨ `:sparkles:` - New features
- 🔧 `:wrench:` - Bug fixes
- 📊 `:bar_chart:` - Chores
- 📜 `:scroll:` - Documentation
- 🌍 `:earth_africa:` - Internationalization
- 🔒 `:lock:` - Security

### Pre-commit Hooks

**Managed by Husky + lint-staged:**

```javascript
// .husky/lint-staged.config.js
{
  '*.{js,jsx,ts,tsx}': [
    'prettier --write',
    'eslint --fix',
    'eslint'
  ],
  '*.json': ['prettier --write']
}
```

**Process:**
1. Stage files (`git add`)
2. Commit triggers pre-commit hook
3. Lint-staged runs prettier + eslint on staged files
4. If checks pass, commit proceeds
5. If checks fail, commit is blocked

**Bypass (not recommended):**
```bash
git commit --no-verify
```

### Making Commits

**Standard Workflow:**
```bash
# Make changes
git add .
git status                    # Review changes

# Auto-formatted by pre-commit hook
git commit -m "✨ feat: Add new feature"

# Push to remote
git push origin your-branch
```

**Important:**
- Never commit secrets (.env files, credentials)
- Review `git diff` before committing
- Write clear, descriptive commit messages
- Reference issue numbers when applicable

### Creating Pull Requests

**Before Creating PR:**
1. Ensure all tests pass locally
2. Run linting: `npm run lint`
3. Run relevant tests: `npm run test:client` / `npm run test:api`
4. Update documentation if needed

**PR Guidelines:**
1. Write clear title and description
2. Link related issues
3. Add screenshots for UI changes
4. Ensure CI passes
5. Request reviews from maintainers

---

## Key Architectural Patterns

### Backend Architecture

**Pattern:** Layered Architecture (Routes → Controllers → Services → Models)

**Request Flow:**
```
Client Request
    ↓
Express Router (/api/server/routes/)
    ↓
Middleware (auth, validation)
    ↓
Controller (handle request)
    ↓
Service Layer (business logic)
    ↓
Model/Database (Mongoose)
    ↓
Response (REST or SSE stream)
```

**Key Routes:**
- `/api/server/routes/auth.js` - Authentication
- `/api/server/routes/messages.js` - Chat messages
- `/api/server/routes/convos.js` - Conversations
- `/api/server/routes/agents/` - Agent management
- `/api/server/routes/files/` - File operations
- `/api/server/routes/mcp.js` - Model Context Protocol

**Middleware:**
- `requireJwtAuth` - JWT authentication
- `validateMessageReq` - Message validation
- `mongoSanitize` - NoSQL injection prevention
- `compression` - Response compression
- `cors` - CORS handling

**Services:**
- `ActionService` - Tool/action execution
- `ModelService` - AI model management
- `FileService` - File storage operations
- `MCPService` - MCP server integration

**Client Implementations:**
- Base pattern: `BaseClient` class
- Implementations: `OpenAIClient`, `AnthropicClient`, `OllamaClient`, etc.
- Location: `/api/app/clients/`

### Frontend Architecture

**Pattern:** Component-based with centralized state management

**State Management:**
```
User Interaction
    ↓
Component Event
    ↓
Jotai Atom Update (or TanStack Query mutation)
    ↓
Re-render affected components
```

**Jotai Atoms (Primary State):**
- `/client/src/store/agents.ts` - Agent state
- `/client/src/store/conversation.ts` - Conversation state
- `/client/src/store/endpoints.ts` - Endpoint configurations
- `/client/src/store/settings.ts` - User settings
- `/client/src/store/artifacts.ts` - Code artifacts
- `/client/src/store/mcp.ts` - MCP state

**TanStack Query (Server State):**
- Cache management for API calls
- Automatic refetching
- Optimistic updates
- Query invalidation

**Component Organization:**
```
components/
├── Agents/              # Feature: Agent marketplace
│   ├── AgentCard.tsx
│   ├── AgentGrid.tsx
│   └── tests/
├── Chat/               # Feature: Chat interface
│   ├── ChatView.tsx
│   ├── Input/
│   └── Messages/
└── Artifacts/          # Feature: Code artifacts
    ├── Artifact.tsx
    └── ArtifactPreview.tsx
```

### API Streaming

**Technology:** Server-Sent Events (SSE)

**Flow:**
1. Client opens SSE connection
2. Server streams AI responses chunk-by-chunk
3. Client updates UI in real-time
4. Connection closes on completion

**Implementation:**
- Backend: Custom SSE implementation
- Frontend: EventSource API with polyfills

### Authentication Flow

**Multi-strategy via Passport.js:**

1. **Local (Email/Password):**
   - User submits credentials
   - Passport LocalStrategy validates
   - JWT token issued

2. **OAuth2 (Google, GitHub, etc.):**
   - Redirect to provider
   - Callback with authorization code
   - Exchange for user info
   - JWT token issued

3. **LDAP:**
   - User submits credentials
   - Passport LDAPStrategy validates against LDAP server
   - JWT token issued

4. **2FA (Optional):**
   - After initial auth, 2FA challenge
   - Verify TOTP code
   - Full JWT token issued

**Session Management:**
- Redis for session storage
- JWT for API authentication
- Configurable session expiry

### File Storage Strategy

**Pattern:** Strategy Pattern with multiple backends

**Strategies:**
- `local` - Local filesystem
- `s3` - AWS S3 or compatible
- `firebase` - Firebase Storage
- `azure` - Azure Blob Storage

**Granular Configuration (librechat.yaml):**
```yaml
fileStrategy:
  avatar: "s3"        # User/agent avatars
  image: "firebase"   # Chat images
  document: "local"   # Document uploads
```

**Implementation:**
- Service layer abstracts storage details
- Sharp for image processing
- Presigned URLs for secure access

### Search Architecture

**Technology:** MeiliSearch

**Features:**
- Full-text search across messages and conversations
- Fuzzy matching
- Typo tolerance
- Faceted filtering

**Indexing:**
- Background sync via `indexSync.js`
- Automatic index updates on message save
- Reset capability via `npm run reset-meili-sync`

### Database Schema Patterns

**MongoDB with Mongoose:**

**Key Models:**
- `User` - User accounts
- `Conversation` - Chat conversations
- `Message` - Individual messages
- `Agent` - Custom agents
- `File` - File metadata
- `Preset` - Saved presets
- `Prompt` - Prompt library

**Patterns:**
- Embedded documents for related data
- References for relationships
- Indexes for query performance
- Soft deletes (user-scoped queries)
- Timestamps (createdAt, updatedAt)

**Location:** `/api/models/` and `/packages/data-schemas/`

---

## Common Development Tasks

### Adding a New API Route

1. **Create route file:**
   ```javascript
   // api/server/routes/myroute.js
   const express = require('express');
   const { requireJwtAuth } = require('~/server/middleware');

   const router = express.Router();
   router.use(requireJwtAuth);

   router.get('/', async (req, res) => {
     // Implementation
     res.json({ success: true });
   });

   module.exports = router;
   ```

2. **Register in index.js:**
   ```javascript
   // api/server/routes/index.js
   const myroute = require('./myroute');
   app.use('/api/myroute', myroute);
   ```

3. **Add tests:**
   ```javascript
   // api/test/myroute.test.js
   describe('MyRoute', () => {
     it('should return success', async () => {
       // Test implementation
     });
   });
   ```

### Adding a New React Component

1. **Create component:**
   ```typescript
   // client/src/components/MyFeature/MyComponent.tsx
   import React from 'react';

   interface MyComponentProps {
     title: string;
   }

   export default function MyComponent({ title }: MyComponentProps) {
     return <div>{title}</div>;
   }
   ```

2. **Add to feature index:**
   ```typescript
   // client/src/components/MyFeature/index.ts
   export { default as MyComponent } from './MyComponent';
   ```

3. **Create tests:**
   ```typescript
   // client/src/components/MyFeature/__tests__/MyComponent.spec.tsx
   import { render, screen } from '@testing-library/react';
   import MyComponent from '../MyComponent';

   describe('MyComponent', () => {
     it('renders title', () => {
       render(<MyComponent title="Test" />);
       expect(screen.getByText('Test')).toBeInTheDocument();
     });
   });
   ```

### Adding Internationalization

1. **Add translation key:**
   ```json
   // client/src/locales/en-US/translation.json
   {
     "com_ui_my_feature": "My Feature"
   }
   ```

2. **Use in component:**
   ```typescript
   import { useTranslation } from 'react-i18next';

   function MyComponent() {
     const { t } = useTranslation();
     return <div>{t('com_ui_my_feature')}</div>;
   }
   ```

3. **Sync translations:**
   - Translations are synced via Locize
   - CI workflow updates translations automatically
   - Run `npm run lint` to catch hardcoded strings

### Adding a Database Model

1. **Create schema:**
   ```javascript
   // packages/data-schemas/src/MyModel.js
   const mongoose = require('mongoose');

   const myModelSchema = mongoose.Schema({
     name: { type: String, required: true },
     userId: { type: String, required: true, index: true },
   }, { timestamps: true });

   module.exports = myModelSchema;
   ```

2. **Register model:**
   ```javascript
   // packages/data-schemas/src/index.js
   const MyModel = mongoose.model('MyModel', myModelSchema);
   module.exports = { MyModel };
   ```

3. **Create queries:**
   ```javascript
   // api/models/MyModel.js
   const { MyModel } = require('@librechat/data-schemas');

   const getMyModels = async (userId) => {
     return await MyModel.find({ userId }).lean();
   };

   module.exports = { getMyModels };
   ```

### Running Database Migrations

```bash
# Agent permissions migration
npm run migrate:agent-permissions:dry-run    # Test first
npm run migrate:agent-permissions            # Run migration

# Prompt permissions migration
npm run migrate:prompt-permissions:dry-run
npm run migrate:prompt-permissions

# With custom batch size
npm run migrate:agent-permissions:batch
```

### User Management

```bash
# Create user
npm run create-user

# Invite user (sends email)
npm run invite-user

# List all users
npm run list-users

# Reset password
npm run reset-password

# Ban user
npm run ban-user

# Delete user
npm run delete-user
```

### Managing Token Balances

```bash
# Add balance to user
npm run add-balance

# Set specific balance
npm run set-balance

# List all balances
npm run list-balances

# User statistics
npm run user-stats
```

### Updating LibreChat

```bash
# Standard update
npm run update

# Local environment
npm run update:local

# Docker environment
npm run update:docker

# Deployed instance
npm run update:deployed

# With sudo (if needed)
npm run update:sudo

# Reinstall dependencies
npm run reinstall
```

---

## Important Notes for AI Assistants

### Do's

✅ **Always:**
- Read existing code before making changes
- Follow the established code style (Prettier + ESLint)
- Use TypeScript for new code when possible
- Add i18n keys instead of hardcoded strings in React components
- Write tests for new features
- Update documentation when changing behavior
- Use Jotai for new client state (not Recoil)
- Use TanStack Query for server state
- Check for import cycles (they're forbidden)
- Use the Winston logger in backend code
- Validate user input and sanitize data
- Handle errors gracefully
- Use environment variables for configuration

✅ **Code Quality:**
- Run `npm run lint:fix` before committing
- Run relevant tests before submitting changes
- Check that E2E tests pass for UI changes
- Ensure accessibility standards are met
- Follow React best practices (hooks, composition)
- Avoid prop drilling (use context or state management)
- Memoize expensive computations

✅ **Security:**
- Never commit secrets or API keys
- Validate and sanitize all user inputs
- Use parameterized queries (Mongoose handles this)
- Implement proper authentication checks
- Follow OWASP security guidelines
- Use HTTPS in production
- Implement rate limiting where appropriate

### Don'ts

❌ **Never:**
- Commit directly to main branch
- Hardcode strings in React components (use i18n)
- Create import cycles
- Use `any` type unless absolutely necessary
- Ignore ESLint errors
- Skip testing
- Expose sensitive data in API responses
- Store passwords in plain text
- Use `console.log` in production (use Winston logger)
- Create duplicate code (DRY principle)
- Break existing functionality without discussion

❌ **Avoid:**
- Large monolithic components (split into smaller ones)
- Inline styles (use Tailwind classes)
- Direct DOM manipulation (use React patterns)
- Synchronous operations for I/O
- Blocking the event loop
- Unnecessary re-renders (use React.memo, useMemo, useCallback)
- Over-engineering solutions
- Breaking changes without migration path

### When to Ask for Clarification

🤔 **Ask when:**
- Requirements are ambiguous
- Breaking changes are needed
- Architecture decisions are required
- Security implications are unclear
- Multiple valid approaches exist
- Existing patterns are unclear
- Documentation is missing or outdated

### Understanding the Codebase

**Mixed JavaScript/TypeScript:**
- The codebase is gradually migrating to TypeScript
- API is mostly JavaScript
- Client is mostly TypeScript
- Packages are fully TypeScript
- When in doubt, check existing files in the same directory

**State Management Evolution:**
- Old: Recoil (still present, being phased out)
- Current: Jotai (preferred for new code)
- Server state: TanStack Query (always use for API calls)

**Multiple Configuration Files:**
- Different tsconfig.json for different parts of monorepo
- ESLint config has file-specific rules
- Check which config applies to your working directory

### Debugging Tips

**Backend:**
```bash
# Enable debug logging
DEBUG_LOGGING=true npm run backend:dev

# Run with Bun inspector
npm run b:api-inspect

# Check MongoDB connection
# Verify MONGO_URI in .env

# Check Redis connection
# Verify REDIS_URI in .env
```

**Frontend:**
```bash
# Development mode with hot reload
npm run frontend:dev

# Check browser console for errors
# Use React DevTools extension
# Use Redux DevTools for Jotai atoms
```

**E2E Tests:**
```bash
# Run in headed mode to see browser
npm run e2e:headed

# Debug with Playwright Inspector
npm run e2e:debug

# Generate new interactions
npm run e2e:codegen
```

### Performance Considerations

**Backend:**
- Use Redis caching for expensive operations
- Implement pagination for large datasets
- Use MongoDB indexes appropriately
- Stream large responses (SSE)
- Optimize database queries (use `.lean()` when possible)

**Frontend:**
- Code splitting is configured in Vite
- Lazy load heavy components
- Use React.memo for expensive renders
- Virtualize long lists (react-virtualized)
- Optimize images (use Sharp on backend)
- Minimize bundle size (check imports)

### Common Pitfalls

1. **Import Cycles:**
   - ESLint will error on import cycles
   - Refactor to break cycles (extract to separate file)

2. **i18n Violations:**
   - Hardcoded strings in JSX will fail linting
   - Use `{t('translation_key')}` pattern

3. **TypeScript Errors:**
   - Check correct tsconfig.json is being used
   - Packages have separate TypeScript configs

4. **Test Failures:**
   - MongoDB Memory Server may need manual cleanup
   - Check for hardcoded ports or URLs
   - Ensure test isolation (no shared state)

5. **Docker Issues:**
   - Check docker-compose.yml configuration
   - Verify environment variables are set
   - Use `npm run update:docker` for dependency updates

### Resources

**Official Documentation:**
- Main docs: https://docs.librechat.ai
- Configuration: https://docs.librechat.ai/docs/configuration
- API docs: https://docs.librechat.ai/docs/development/api

**Code Examples:**
- Check existing routes in `/api/server/routes/`
- Check existing components in `/client/src/components/`
- Check tests for usage patterns

**Community:**
- Discord: https://discord.librechat.ai
- GitHub Issues: https://github.com/danny-avila/LibreChat/issues
- Changelog: https://librechat.ai/changelog

---

## Quick Reference

### File Paths to Know

```
# Configuration
/.env.example                          # Environment variables template
/librechat.example.yaml               # App configuration template
/eslint.config.mjs                    # ESLint rules
/.prettierrc                          # Prettier rules

# Backend Entry
/api/server/index.js                  # Main server file
/api/server/routes/index.js           # Routes registration

# Frontend Entry
/client/src/main.tsx                  # React entry point
/client/src/routes/Root.tsx           # Root route component

# State Management
/client/src/store/                    # Jotai atoms

# Tests
/api/test/                            # Backend tests
/client/test/                         # Frontend tests
/e2e/specs/                           # E2E tests

# Build Configuration
/client/vite.config.ts                # Vite config
/package.json                         # Root package scripts
```

### Environment Setup Checklist

- [ ] Node.js v20+ installed
- [ ] MongoDB running (local or remote)
- [ ] Redis running (optional but recommended)
- [ ] MeiliSearch running (optional, for search)
- [ ] `.env` file created from `.env.example`
- [ ] Dependencies installed (`npm install`)
- [ ] Database connected successfully
- [ ] Backend starts without errors
- [ ] Frontend starts without errors
- [ ] Tests pass (`npm run test:client` && `npm run test:api`)

### Command Cheat Sheet

```bash
# Setup
npm install
cp .env.example .env

# Development
npm run backend:dev
npm run frontend:dev

# Testing
npm run lint
npm run test:client
npm run test:api
npm run e2e

# Database
npm run create-user
npm run add-balance

# Build
npm run build:packages
npm run frontend

# Docker
docker compose up -d
npm run start:deployed

# Updates
npm run update
npm run update:deployed
```

---

**Last Updated:** 2025-11-16
**Maintainer:** LibreChat Team
**Questions?** Open an issue or ask in Discord: https://discord.librechat.ai
