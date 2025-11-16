# Getting Started with LibreChat Development

## Prerequisites

Before you begin, ensure you have the following installed:

- **Node.js** v20.x or higher ([Download](https://nodejs.org/))
- **npm** v10.x or higher (comes with Node.js)
- **MongoDB** v6.x or higher ([Download](https://www.mongodb.com/try/download/community))
- **Git** ([Download](https://git-scm.com/downloads))
- **Docker** (optional, for containerized development) ([Download](https://www.docker.com/))

**Optional but Recommended:**
- **Redis** (for caching and sessions)
- **MeiliSearch** (for full-text search)
- **Bun** (faster alternative to npm)

---

## Quick Start (5 Minutes)

### 1. Clone the Repository

```bash
git clone https://github.com/danny-avila/LibreChat.git
cd LibreChat
```

### 2. Install Dependencies

```bash
npm install
```

This installs dependencies for the entire monorepo (api, client, and packages).

### 3. Configure Environment

```bash
cp .env.example .env
```

**Minimum Required Configuration:**

Edit `.env` and set:

```bash
# MongoDB Connection
MONGO_URI=mongodb://127.0.0.1:27017/LibreChat

# Application URLs
DOMAIN_CLIENT=http://localhost:3080
DOMAIN_SERVER=http://localhost:3080

# JWT Secrets (generate with: openssl rand -base64 32)
JWT_SECRET=your-jwt-secret-here
JWT_REFRESH_SECRET=your-jwt-refresh-secret-here

# Encryption Key (generate with: openssl rand -hex 32)
CREDS_KEY=your-32-byte-hex-key
CREDS_IV=your-32-byte-hex-iv

# AI Provider API Key (at least one)
OPENAI_API_KEY=sk-...
# ANTHROPIC_API_KEY=sk-ant-...
# GOOGLE_API_KEY=...
```

**Generate Secrets:**
```bash
# JWT secrets
openssl rand -base64 32

# Encryption keys
openssl rand -hex 32
```

### 4. Start Development Servers

**Option A: npm (separate terminals)**

Terminal 1 - Backend:
```bash
npm run backend:dev
```

Terminal 2 - Frontend:
```bash
npm run frontend:dev
```

**Option B: Bun (faster)**

Terminal 1 - Backend:
```bash
npm run b:api:dev
```

Terminal 2 - Frontend:
```bash
npm run b:client:dev
```

### 5. Access the Application

Open your browser and navigate to:
```
http://localhost:3080
```

**First User Auto-Admin:**
The first user to register will automatically become an administrator.

---

## Docker Development Setup

### Using Docker Compose

```bash
# Start all services
docker compose up -d

# View logs
docker compose logs -f api

# Stop services
docker compose down
```

**Services Started:**
- LibreChat API (port 3080)
- MongoDB (port 27017)
- MeiliSearch (port 7700)
- RAG API (optional, port 8000)

---

## Project Structure Overview

```
LibreChat/
├── api/                    # Backend Node.js application
│   ├── server/
│   │   ├── index.js       # Server entry point
│   │   ├── routes/        # API route definitions
│   │   ├── controllers/   # Request handlers
│   │   ├── middleware/    # Express middleware
│   │   └── services/      # Business logic
│   ├── app/clients/       # AI provider implementations
│   ├── models/            # Database query methods
│   └── strategies/        # Authentication strategies
│
├── client/                 # Frontend React application
│   ├── src/
│   │   ├── main.jsx       # React entry point
│   │   ├── components/    # UI components
│   │   ├── hooks/         # Custom hooks
│   │   ├── store/         # State management
│   │   └── data-provider/ # API client
│   └── public/            # Static assets
│
├── packages/               # Shared monorepo packages
│   ├── data-schemas/      # Mongoose schemas
│   ├── data-provider/     # Data services
│   ├── api/               # MCP services
│   └── client/            # Shared components
│
├── e2e/                   # Playwright E2E tests
├── config/                # Configuration scripts
├── .env.example           # Environment template
└── package.json           # Root package config
```

---

## Development Workflow

### 1. Create a Feature Branch

```bash
git checkout -b feature/my-new-feature
```

### 2. Make Your Changes

**Backend Changes:**
- Modify files in `/api/`
- Server auto-restarts with nodemon

**Frontend Changes:**
- Modify files in `/client/src/`
- Hot Module Replacement (HMR) enabled

**Package Changes:**
- Modify files in `/packages/*/`
- Rebuild package: `npm run build:data-provider` (or relevant package)

### 3. Test Your Changes

**Linting:**
```bash
npm run lint
npm run lint:fix  # Auto-fix issues
```

**Unit Tests:**
```bash
npm run test:api
npm run test:client
```

**E2E Tests:**
```bash
npm run e2e
npm run e2e:headed  # With visible browser
```

### 4. Commit Your Changes

```bash
git add .
git commit -m "✨ feat: Add new feature"
```

**Commit runs automatically:**
- Prettier (formatting)
- ESLint (linting)

**Commit Message Convention:**
```
emoji type: description

Examples:
✨ feat: Add agent marketplace
🔧 fix: Fix message streaming bug
📊 chore: Update dependencies
📜 docs: Update API documentation
```

### 5. Push and Create PR

```bash
git push origin feature/my-new-feature
```

Then create a Pull Request on GitHub.

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
     res.json({ message: 'Success' });
   });

   module.exports = router;
   ```

2. **Register route:**
   ```javascript
   // api/server/routes/index.js
   const myroute = require('./myroute');

   module.exports = (app) => {
     // ...
     app.use('/api/myroute', myroute);
   };
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

2. **Export from index:**
   ```typescript
   // client/src/components/MyFeature/index.ts
   export { default as MyComponent } from './MyComponent';
   ```

### Adding Translations

1. **Add key to English:**
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

### Running Database Migrations

```bash
# Dry run first
npm run migrate:agent-permissions:dry-run

# Execute migration
npm run migrate:agent-permissions
```

### Managing Users

```bash
# Create user
npm run create-user

# Reset password
npm run reset-password

# Add token balance
npm run add-balance

# List users
npm run list-users
```

---

## Debugging

### Backend Debugging

**Enable Debug Logging:**
```bash
# In .env
DEBUG_LOGGING=true
DEBUG_CONSOLE=true
```

**Inspect with Bun:**
```bash
npm run b:api-inspect
```

Then open `chrome://inspect` in Chrome.

### Frontend Debugging

**React DevTools:**
Install the [React DevTools](https://react.dev/learn/react-developer-tools) browser extension.

**Redux DevTools:**
For Jotai atoms, use [Redux DevTools](https://github.com/reduxjs/redux-devtools).

**Console Logging:**
```typescript
import { logger } from '~/utils';

logger.log('component', 'Message', data);
logger.error('error', 'Failed to load', error);
```

### E2E Debugging

```bash
# Run with Playwright Inspector
npm run e2e:debug

# Generate test code
npm run e2e:codegen
```

---

## Environment Variables Reference

### Essential Variables

```bash
# Server
HOST=localhost
PORT=3080
MONGO_URI=mongodb://127.0.0.1:27017/LibreChat

# Security
JWT_SECRET=<generate-with-openssl>
JWT_REFRESH_SECRET=<generate-with-openssl>
CREDS_KEY=<32-byte-hex>
CREDS_IV=<32-byte-hex>

# Domains
DOMAIN_CLIENT=http://localhost:3080
DOMAIN_SERVER=http://localhost:3080

# AI Providers (at least one)
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
GOOGLE_API_KEY=...
```

### Optional but Recommended

```bash
# Redis
REDIS_URI=redis://localhost:6379

# MeiliSearch
MEILI_HOST=http://localhost:7700
MEILI_MASTER_KEY=<your-master-key>

# Email (for verification)
EMAIL_SERVICE=gmail
EMAIL_USERNAME=your-email@gmail.com
EMAIL_PASSWORD=your-app-password
EMAIL_FROM=noreply@librechat.ai

# Debug
DEBUG_LOGGING=true
DEBUG_CONSOLE=true
```

See [Environment Variables](configuration/environment-variables.md) for complete reference.

---

## Troubleshooting

### MongoDB Connection Failed

**Error:** `MongoServerError: connect ECONNREFUSED 127.0.0.1:27017`

**Solution:**
```bash
# Start MongoDB
mongod --dbpath /path/to/data/db

# Or with Docker
docker run -d -p 27017:27017 mongo:latest
```

### Port Already in Use

**Error:** `Error: listen EADDRINUSE: address already in use :::3080`

**Solution:**
```bash
# Find process using port 3080
lsof -ti:3080

# Kill the process
kill -9 <PID>
```

### Module Not Found

**Error:** `Cannot find module '@librechat/data-provider'`

**Solution:**
```bash
# Rebuild packages
npm run build:packages

# Or clean install
rm -rf node_modules package-lock.json
npm install
```

### ESLint Errors on Commit

**Solution:**
```bash
# Auto-fix linting issues
npm run lint:fix

# If issues persist, check specific errors
npm run lint
```

### Frontend Won't Start

**Solution:**
```bash
# Clear Vite cache
rm -rf client/node_modules/.vite

# Rebuild data provider
npm run build:data-provider

# Restart frontend
npm run frontend:dev
```

---

## Performance Tips

### Faster Builds with Bun

Install Bun: https://bun.sh/

```bash
# Use Bun commands
npm run b:api:dev      # Backend
npm run b:client:dev   # Frontend
npm run b:test:client  # Tests
```

### Faster npm Install

```bash
# Use npm ci for clean installs (faster)
npm ci

# Or use Bun
bun install
```

### Skip Tests During Development

```bash
# Skip E2E tests
SKIP_E2E=true npm run test

# Skip unit tests
SKIP_UNIT=true npm run test
```

---

## Next Steps

### Learn the Architecture
- Read [Architecture Overview](ARCHITECTURE.md)
- Explore [Backend Architecture](backend/README.md)
- Understand [Frontend Architecture](frontend/README.md)

### Understand Key Features
- [Authentication](features/authentication.md)
- [AI Provider Integration](providers/README.md)
- [Message Streaming](features/message-streaming.md)

### Contribute
- Review [Contributing Guide](development/contributing.md)
- Check [Open Issues](https://github.com/danny-avila/LibreChat/issues)
- Join [Discord Community](https://discord.librechat.ai)

---

## Useful Commands Cheat Sheet

```bash
# Development
npm run backend:dev         # Start backend with hot reload
npm run frontend:dev        # Start frontend with HMR
npm run b:api:dev          # Backend with Bun (faster)
npm run b:client:dev       # Frontend with Bun (faster)

# Building
npm run build:packages     # Build all shared packages
npm run frontend           # Production frontend build

# Testing
npm run lint               # Check code quality
npm run lint:fix           # Auto-fix linting issues
npm run test:api           # Backend unit tests
npm run test:client        # Frontend unit tests
npm run e2e                # E2E tests (headless)
npm run e2e:headed         # E2E tests (headed)

# Database
npm run create-user        # Create new user
npm run reset-password     # Reset user password
npm run add-balance        # Add token balance

# Docker
docker compose up -d       # Start all services
docker compose logs -f api # View API logs
docker compose down        # Stop all services

# Updates
npm run update             # Update LibreChat
npm run update:local       # Update local environment
```

---

## Additional Resources

- **Official Docs:** https://docs.librechat.ai
- **API Reference:** [API Routes](backend/api-routes.md)
- **Discord:** https://discord.librechat.ai
- **GitHub:** https://github.com/danny-avila/LibreChat

---

**Last Updated:** 2025-11-16
