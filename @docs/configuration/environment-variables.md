# Environment Variables

> **Configuration reference for LibreChat v0.8.1-rc1**

## Overview

LibreChat uses **environment variables** for configuration, with **500+ lines** of documented options in `.env.example`. Variables control database connections, AI provider API keys, authentication, security, and feature flags.

## Source Code References

- Template: `/home/user/LibreChat/.env.example`
- Loading: `api/server/index.js` (dotenv initialization)
- Validation: `api/server/services/Config/loadConfig.js`

## Core Configuration

### Database

```env
# MongoDB
MONGO_URI=mongodb://localhost:27017/librechat
MONGO_MAX_POOL_SIZE=10
MONGO_MIN_POOL_SIZE=2

# Redis
REDIS_URI=redis://localhost:6379

# MeiliSearch
MEILI_HOST=http://localhost:7700
MEILI_MASTER_KEY=your_master_key
```

### Server

```env
# Server
HOST=0.0.0.0
PORT=3080
DOMAIN_CLIENT=http://localhost:3080
DOMAIN_SERVER=http://localhost:3080

# Node environment
NODE_ENV=production
```

### Security

```env
# JWT
JWT_SECRET=your_jwt_secret_here
JWT_EXPIRY=1h
JWT_REFRESH_SECRET=your_refresh_secret
REFRESH_TOKEN_EXPIRY=7d

# Session
SESSION_SECRET=your_session_secret
SESSION_EXPIRY=900000

# CORS
ALLOW_ORIGINS=http://localhost:3080
```

## AI Provider API Keys

### OpenAI

```env
OPENAI_API_KEY=sk-...
OPENAI_MODELS=gpt-4,gpt-4-turbo-preview,gpt-3.5-turbo
OPENAI_REVERSE_PROXY=
```

### Anthropic (Claude)

```env
ANTHROPIC_API_KEY=sk-ant-...
ANTHROPIC_MODELS=claude-3-opus,claude-3-sonnet,claude-3-haiku
ANTHROPIC_REVERSE_PROXY=
```

### Google (Gemini)

```env
GOOGLE_API_KEY=AIza...
GOOGLE_MODELS=gemini-pro,gemini-pro-vision
GOOGLE_REVERSE_PROXY=

# Vertex AI
GOOGLE_VERTEX_PROJECT_ID=
GOOGLE_VERTEX_LOCATION=us-central1
```

### Azure OpenAI

```env
AZURE_OPENAI_API_KEY=
AZURE_OPENAI_ENDPOINT=https://your-resource.openai.azure.com
AZURE_OPENAI_API_VERSION=2024-02-15-preview
AZURE_OPENAI_MODELS=gpt-4,gpt-35-turbo
```

## Authentication

### Local Auth

```env
ALLOW_EMAIL_LOGIN=true
ALLOW_REGISTRATION=true
ALLOW_UNVERIFIED_EMAIL=true
```

### OAuth2

```env
# Google
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
GOOGLE_CALLBACK_URL=/api/oauth/google/callback

# GitHub
GITHUB_CLIENT_ID=
GITHUB_CLIENT_SECRET=
GITHUB_CALLBACK_URL=/api/oauth/github/callback

# Discord
DISCORD_CLIENT_ID=
DISCORD_CLIENT_SECRET=
DISCORD_CALLBACK_URL=/api/oauth/discord/callback
```

### LDAP

```env
LDAP_URL=ldap://ldap.example.com
LDAP_USER_SEARCH_BASE=ou=users,dc=example,dc=com
LDAP_BIND_DN=cn=admin,dc=example,dc=com
LDAP_BIND_CREDENTIALS=
```

## File Storage

```env
# File strategy (local, s3, firebase, azure)
CDN_PROVIDER=local

# AWS S3
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_S3_BUCKET=
AWS_REGION=us-east-1

# Firebase
FIREBASE_API_KEY=
FIREBASE_AUTH_DOMAIN=
FIREBASE_PROJECT_ID=
FIREBASE_STORAGE_BUCKET=

# Azure Blob
AZURE_STORAGE_ACCOUNT=
AZURE_STORAGE_KEY=
AZURE_STORAGE_CONTAINER=
```

## Rate Limiting

```env
LIMIT_MESSAGE_IP=120
LIMIT_MESSAGE_USER=40
LIMIT_TOOL_CALL=50
LIMIT_CONCURRENT=5
```

## Feature Flags

```env
DEBUG_LOGGING=false
DEBUG_PLUGINS=false
DEBUG_CONSOLE=false
SEARCH_ENABLED=true
TITLE_CONVO=true
CHECK_BALANCE=false
```

## Related Documentation

- [Configuration Overview](README.md)
- [LibreChat YAML](librechat-yaml.md)
- [Deployment Configs](deployment-configs.md)

---

**Last Updated:** 2025-11-18
**Version:** 1.0.0
