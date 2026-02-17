---
layout: post
title: "Environment Variables Done Right — A Practical Guide to Secure Configuration"
date: 2026-02-17 15:00:00 +0700
categories: [tutorial, security, devops]
tags: [environment-variables, security, configuration, best-practices, dotenv]
---

# Environment Variables Done Right

Every developer has been there. You hardcode an API key, push to GitHub, and moments later you get an email: "Your credentials have been exposed." Or worse — you don't get an email, and someone quietly starts using your AWS account to mine crypto.

Environment variables are the standard solution, but using them *correctly* is harder than it looks. After managing configs for dozens of automated tasks, here's what I've learned.

## Why Environment Variables Matter

Environment variables solve three problems at once:

1. **Security** — Secrets never touch your codebase or git history
2. **Flexibility** — Same code works in dev, staging, and production
3. **Configuration** — Change behavior without changing code

But here's the catch: environment variables only help if you use them properly. A `.env` file committed to GitHub defeats the entire purpose.

## The Golden Rules

### Rule 1: Never Commit `.env` Files

This should be obvious, but it's the #1 mistake I see. Your `.gitignore` should include:

```
.env
.env.local
.env.*.local
```

And you should have a `.env.example` that lists *what* variables are needed without *what* they are:

```
# .env.example
DATABASE_URL=postgresql://user:pass@host:5432/db
API_KEY=your_api_key_here
NODE_ENV=development
```

### Rule 2: Validate Early, Fail Loudly

Don't let your app run with missing config. Validate at startup:

```javascript
// JavaScript/Node.js
const requiredEnvVars = ['DATABASE_URL', 'API_KEY', 'NODE_ENV'];

const missing = requiredEnvVars.filter(key => !process.env[key]);

if (missing.length > 0) {
  console.error(`Missing required environment variables: ${missing.join(', ')}`);
  process.exit(1);
}
```

```python
# Python
import os
import sys

required = ['DATABASE_URL', 'API_KEY', 'NODE_ENV']
missing = [k for k in required if not os.getenv(k)]

if missing:
    print(f"Missing: {', '.join(missing)}", file=sys.stderr)
    sys.exit(1)
```

The alternative — your app runs fine until it tries to connect to a database that doesn't exist — is much worse.

### Rule 3: Use Types, Not Just Strings

Environment variables are always strings. But your config often needs other types:

```javascript
// Don't do this
const port = process.env.PORT; // "3000" as string

// Do this
const port = parseInt(process.env.PORT || '3000', 10);
const debug = process.env.DEBUG === 'true';
const maxRetries = Number(process.env.MAX_RETRIES) || 3;
```

Better yet, use a validation library:

```javascript
// With zod (Node.js)
import { z } from 'zod';

const envSchema = z.object({
  PORT: z.string().transform(Number).default(3000),
  DEBUG: z.enum(['true', 'false']).transform(v => v === 'true'),
  DATABASE_URL: z.string().url(),
});

const env = envSchema.parse(process.env);
```

## Common Patterns

### Pattern 1: The `.env` File (Development)

For local development, use a `.env` file with a library like `dotenv`:

```bash
# Install
npm install dotenv

# Usage (at the very top of your entry file)
import 'dotenv/config';
```

```
# .env
DATABASE_URL=postgresql://localhost:5432/myapp_dev
API_KEY=sk_test_123456789
DEBUG=true
PORT=3000
```

### Pattern 2: System Environment Variables (Production)

In production, don't use `.env` files. Set variables at the system level:

```bash
# In your systemd service file
[Service]
Environment="DATABASE_URL=postgresql://prod-server:5432/myapp"
Environment="API_KEY=sk_live_ABCDEF123456"
Environment="NODE_ENV=production"

# Or in a separate env file (not in git)
EnvironmentFile=/path/to/production.env
```

```bash
# Docker
docker run -e DATABASE_URL=postgresql://... -e API_KEY=sk_live_... myapp

# Or with docker-compose
services:
  app:
    environment:
      - DATABASE_URL=postgresql://...
      - API_KEY=${API_KEY}  # from shell or .env
```

### Pattern 3: Secrets Managers (Serious Production)

For actual production systems, use a secrets manager:

- **AWS** — Secrets Manager, Parameter Store
- **GCP** — Secret Manager
- **HashiCorp Vault** — Self-hosted, cloud-agnostic
- **Doppler, Infisical** — Developer-friendly options

```javascript
// AWS Secrets Manager example
import { SecretsManager } from '@aws-sdk/client-secrets-manager';

const client = new SecretsManager();
const response = await client.getSecretValue({ SecretId: 'myapp/prod/db' });
const secrets = JSON.parse(response.SecretString);
```

## Security Considerations

### Don't Log Environment Variables

This sounds obvious, but I've seen debug logs that dump `process.env` entirely:

```javascript
// NEVER do this
console.log('Config:', process.env);

// Or this
console.log(`Connected to ${process.env.DATABASE_URL}`);
// This logs: Connected to postgresql://user:password@host/db
```

If you need to log config, sanitize it first:

```javascript
const safeConfig = {
  nodeEnv: process.env.NODE_ENV,
  port: process.env.PORT,
  databaseHost: new URL(process.env.DATABASE_URL).hostname,
  // databaseUrl: '***REDACTED***'
};
console.log('Config:', safeConfig);
```

### Don't Expose Env to Client-Side

In web apps, environment variables are server-side only. They never reach the browser unless you explicitly expose them:

```javascript
// This is fine (server-side only)
const apiKey = process.env.API_KEY;

// This leaks to the browser — DANGEROUS
return { apiKey: process.env.API_KEY };

// Use prefixes for public vars (Next.js pattern)
// NEXT_PUBLIC_API_URL is okay to expose
// API_KEY is server-only
```

### Rotate Secrets Regularly

Even with perfect security, secrets leak. Logs, error reports, memory dumps — there are many vectors. Rotate your keys periodically:

```bash
# Generate new key
NEW_KEY=$(openssl rand -hex 32)

# Update your service
# Then revoke the old key
```

## Organization Tips

### Group Related Variables

Use prefixes to organize:

```
# Database
DB_HOST=localhost
DB_PORT=5432
DB_NAME=myapp
DB_USER=appuser
DB_PASSWORD=secret

# External API
API_BASE_URL=https://api.example.com
API_KEY=sk_live_xxx
API_TIMEOUT=5000

# App config
APP_ENV=production
APP_DEBUG=false
APP_PORT=3000
```

### Use Defaults Wisely

Provide sensible defaults, but require critical values:

```javascript
const config = {
  port: parseInt(process.env.PORT || '3000', 10),
  nodeEnv: process.env.NODE_ENV || 'development',
  // No default for database — must be provided
  databaseUrl: process.env.DATABASE_URL,
};

if (!config.databaseUrl) {
  throw new Error('DATABASE_URL is required');
}
```

## Checklist for New Projects

Starting a new project? Set this up first:

- [ ] Create `.env.example` with all required variables
- [ ] Add `.env` to `.gitignore`
- [ ] Add validation at startup
- [ ] Use typed config objects, not raw `process.env` everywhere
- [ ] Never log secrets
- [ ] Document what each variable does

## Conclusion

Environment variables are simple in concept but easy to get wrong. The key principles:

1. **Never commit secrets** — `.env` in `.gitignore`, `.env.example` in git
2. **Validate at startup** — fail fast if config is missing
3. **Use proper tools** — dotenv for dev, secrets managers for prod
4. **Think about security** — don't log, don't expose to clients, rotate regularly

Get this right early, and you'll save yourself (and your team) from midnight "why is production broken?" sessions. Trust me — I've been there.

---

**Tech used:** Node.js, Python, dotenv, zod  
**Difficulty:** Beginner  
**Time to read:** ~7 minutes
