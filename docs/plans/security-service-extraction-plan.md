# Security Service Extraction Plan

## Overview

Extract the comprehensive security sanitization and validation implementation from the chaos-monkey project to create a reusable security service package that can be consumed by multiple projects (web apps, APIs, Cloudflare Workers, etc.).

## Current Analysis

Based on `/workspace/workers/chaos-monkey/src/`, we have the following security components:

### Highly Reusable Components (Universal)

- **Core Sanitization Functions** (`services/sanitization.ts`)

  - `sanitizeText()` - Comprehensive text sanitization with XSS/NoSQL/command injection protection
  - `sanitizeEmail()` - Email validation and normalization using validator.js
  - `sanitizeUrl()` - URL validation with protocol restrictions
  - `sanitizeFilePath()` - Path traversal prevention
  - `sanitizeAlphanumeric()` - Alphanumeric field validation
  - `sanitizeInteger()`/`sanitizeFloat()` - Numeric validation with bounds
  - `sanitizeJSON()` - JSON validation with depth limits
  - `sanitizeUUID()` - UUID validation
  - `sanitizeDate()` - ISO date validation

- **Security Testing Utilities** (`test/security-testing-utils.ts`)

  - `wouldExecuteJavaScriptInHTML()` - Real XSS attack detection
  - `couldInjectIntoMongoQuery()` - NoSQL injection detection
  - `couldExecuteSystemCommands()` - Command injection detection
  - `couldTraverseDirectories()` - Path traversal detection
  - `containsDangerousUnicode()` - Unicode attack detection
  - `isContentSecure()` - Comprehensive security validation

- **Validation Schemas & Patterns**
  - Field-specific limits and patterns (session_uuid, git_commit_hash, etc.)
  - MongoDB operator detection patterns
  - Command injection patterns
  - Unicode attack patterns

### Framework-Specific Components (Adaptable)

- **Hono Middleware** (`middleware/sanitization.ts`)
  - `createSanitizationMiddleware()` - Auto-sanitization for Hono requests
  - Request body/query/params sanitization
  - Error handling and validation responses

### Project-Specific Components (Keep Local)

- **Chaos-Specific Functions**
  - `sanitizeChaosSession()` - Chaos monkey session validation
  - `sanitizeBreakExecution()` - Break execution validation
  - `CHAOS_FIELD_LIMITS` - Domain-specific field constraints

## Implementation Plan

### Phase 1: Create Security Service Package

1. **Create new package structure**

   ```
   /workspace/workers/security-service/
   ├── package.json
   ├── tsconfig.json
   ├── README.md
   ├── src/
   │   ├── index.ts                     # Main barrel export
   │   ├── core/
   │   │   ├── sanitizers.ts           # Core sanitization functions
   │   │   ├── validators.ts           # Validation utilities
   │   │   └── patterns.ts             # Attack detection patterns
   │   ├── testing/
   │   │   ├── security-utils.ts       # Security testing utilities
   │   │   └── attack-vectors.ts       # Test attack payloads
   │   ├── middleware/
   │   │   ├── hono.ts                 # Hono framework integration
   │   │   ├── express.ts              # Express framework integration
   │   │   └── base.ts                 # Framework-agnostic middleware
   │   ├── types/
   │   │   ├── index.ts                # Type definitions
   │   │   └── config.ts               # Configuration interfaces
   │   └── presets/
   │       ├── web-api.ts              # Web API security preset
   │       ├── user-input.ts           # User input sanitization preset
   │       └── cms.ts                  # Content management preset
   ├── test/
   │   ├── unit/
   │   ├── integration/
   │   └── security/                   # Security vulnerability tests
   └── dist/
   ```

2. **Package configuration**
   ```json
   {
     "name": "@workspace/workers/security-service",
     "version": "1.0.0",
     "description": "Comprehensive input sanitization and security validation library",
     "main": "dist/index.js",
     "types": "dist/index.d.ts",
     "exports": {
       ".": {
         "import": "./dist/index.mjs",
         "require": "./dist/index.js",
         "types": "./dist/index.d.ts"
       },
       "./hono": {
         "import": "./dist/middleware/hono.mjs",
         "require": "./dist/middleware/hono.js",
         "types": "./dist/middleware/hono.d.ts"
       },
       "./testing": {
         "import": "./dist/testing/index.mjs",
         "require": "./dist/testing/index.js",
         "types": "./dist/testing/index.d.ts"
       }
     },
     "dependencies": {
       "validator": "^13.12.0"
     },
     "peerDependencies": {
       "hono": "^4.0.0"
     },
     "keywords": ["security", "sanitization", "xss", "injection", "validation"]
   }
   ```

### Phase 2: Extract and Organize Core Components

**Core Sanitizers** (`src/core/sanitizers.ts`)

```typescript
export interface SanitizationOptions {
  maxLength?: number;
  allowHTML?: boolean;
  stripLow?: boolean;
  // ... other options
}

export function sanitizeText(
  input: string,
  options?: SanitizationOptions
): string | null;
export function sanitizeEmail(email: string): string | null;
export function sanitizeUrl(url: string): string | null;
export function sanitizeFilePath(path: string): string | null;
// ... other core functions
```

**Security Testing Utilities** (`src/testing/security-utils.ts`)

```typescript
export function wouldExecuteJavaScriptInHTML(content: string): boolean;
export function couldInjectIntoMongoQuery(content: string): boolean;
export function couldExecuteSystemCommands(content: string): boolean;
// ... comprehensive security testing functions
```

**Framework Middleware** (`src/middleware/hono.ts`)

```typescript
import type { Context, Next } from "hono";

export interface MiddlewareOptions {
  maxStringLength?: number;
  maxDepth?: number;
  allowedFields?: string[];
  strictMode?: boolean;
}

export function createSanitizationMiddleware(options?: MiddlewareOptions);
```

**Security Presets** (`src/presets/web-api.ts`)

```typescript
export const WEB_API_PRESET: SanitizationOptions = {
  maxStringLength: 10000,
  allowHTML: false,
  stripLow: true,
  // ... optimized for web APIs
};

export const USER_INPUT_PRESET: SanitizationOptions = {
  maxStringLength: 5000,
  allowHTML: false,
  strictMode: true,
  // ... optimized for user input
};
```

### Phase 3: Framework Integrations

**Express Middleware** (`src/middleware/express.ts`)

```typescript
import type { Request, Response, NextFunction } from "express";

export function expressSanitizer(options?: MiddlewareOptions) {
  return (req: Request, res: Response, next: NextFunction) => {
    // Express-specific implementation
  };
}
```

**Fastify Plugin** (`src/middleware/fastify.ts`)

```typescript
import type { FastifyPluginCallback } from "fastify";

export const fastifySanitizer: FastifyPluginCallback<MiddlewareOptions>;
```

### Phase 4: Update Consumer Projects

1. **Chaos Monkey** (`/workspace/workers/chaos-monkey/`)

   - Add dependency: `"@workspace/workers/security-service": "git+file:../../security-service"`
   - Replace local sanitization with: `import { sanitizeText, createSanitizationMiddleware } from '@workspace/workers/security-service'`
   - Keep chaos-specific functions: `sanitizeChaosSession()`, `sanitizeBreakExecution()`
   - Update tests to use shared testing utilities

2. **Other Projects**
   - **Chat Worker**: Add input sanitization for chat messages
   - **Agora**: Secure user input in forms and API calls
   - **Examples**: Demonstrate security best practices

### Phase 5: Advanced Features

**Content Security Policy Generator**

```typescript
export function generateCSP(options: CSPOptions): string;
```

**Rate Limiting Utilities**

```typescript
export function createRateLimiter(options: RateLimitOptions);
```

**Security Headers Helper**

```typescript
export function securityHeaders(): Record<string, string>;
// Returns: X-Frame-Options, X-Content-Type-Options, etc.
```

## API Design

### Core API

```typescript
// Simple usage
import { sanitizeText } from "@workspace/workers/security-service";
const clean = sanitizeText(userInput);

// Advanced usage
import { createSanitizer } from "@workspace/workers/security-service";
const sanitizer = createSanitizer({
  maxLength: 5000,
  allowHTML: false,
  customPatterns: [/dangerous-pattern/g],
});
const result = sanitizer.sanitize(input);

// Framework integration
import { createSanitizationMiddleware } from "@workspace/workers/security-service/hono";
app.use("/api/*", createSanitizationMiddleware());

// Security testing
import { isContentSecure } from "@workspace/workers/security-service/testing";
const securityCheck = isContentSecure(sanitizedContent);
```

### Configuration Presets

```typescript
import {
  WEB_API_PRESET,
  USER_INPUT_PRESET,
} from "@workspace/workers/security-service";

// Pre-configured for common use cases
const webSanitizer = createSanitizer(WEB_API_PRESET);
const userSanitizer = createSanitizer(USER_INPUT_PRESET);
```

## Distribution Strategy

**Option A: Git Dependencies (Recommended for monorepo)**

```json
{
  "dependencies": {
    "@workspace/workers/security-service": "git+file:../security-service"
  }
}
```

**Option B: NPM Registry (For external distribution)**

```json
{
  "dependencies": {
    "@yourorg/security-service": "^1.0.0"
  }
}
```

## Migration Steps

### Step 1: Create Package Structure (2-3 hours)

- [ ] Create `/workspace/workers/security-service/` directory
- [ ] Setup `package.json`, `tsconfig.json`, build configuration
- [ ] Extract core sanitization functions from chaos-monkey
- [ ] Extract security testing utilities
- [ ] Create barrel exports and organize modules
- [ ] Setup comprehensive test suite

### Step 2: Framework Integrations (1-2 hours)

- [ ] Create Hono middleware adapter
- [ ] Create Express middleware adapter
- [ ] Create framework-agnostic base middleware
- [ ] Test integrations with sample applications

### Step 3: Update Chaos Monkey (1 hour)

- [ ] Add git dependency to `package.json`
- [ ] Replace local sanitization imports
- [ ] Keep chaos-specific validation functions
- [ ] Update tests to use shared utilities
- [ ] Validate all tests still pass

### Step 4: Documentation & Examples (1-2 hours)

- [ ] Create comprehensive README with examples
- [ ] Document security patterns and best practices
- [ ] Create migration guide from common libraries
- [ ] Add JSDoc comments to all public APIs
- [ ] Create example integrations for different frameworks

### Step 5: Security Validation (1 hour)

- [ ] Run comprehensive vulnerability scans
- [ ] Test against OWASP Top 10 attack vectors
- [ ] Validate performance under load
- [ ] Create security benchmarks

## Benefits

### For Security

- **Battle-tested**: Based on comprehensive vulnerability testing
- **Multi-vector Protection**: XSS, NoSQL injection, command injection, path traversal, Unicode attacks
- **Real Attack Detection**: Tests actual exploitability, not just pattern matching
- **Performance Optimized**: Sub-millisecond processing per input

### For Development

- **Framework Agnostic**: Works with Hono, Express, Fastify, vanilla Node.js
- **TypeScript First**: Full type safety and IntelliSense support
- **Easy Integration**: Drop-in middleware for common frameworks
- **Configurable**: Presets for common use cases, fully customizable
- **Testing Utilities**: Built-in security testing functions

### For Maintenance

- **Single Source of Truth**: Centralized security logic
- **Consistent Protection**: Same security rules across all projects
- **Easy Updates**: Security improvements propagate to all consumers
- **Community Driven**: Can accept contributions and security reports

## Security Features

### Input Sanitization

- HTML escaping for XSS prevention
- NoSQL operator removal for injection prevention
- Command separator neutralization
- Path traversal sequence removal
- Unicode normalization and dangerous character removal

### Validation

- Email format validation with normalization
- URL validation with protocol restrictions
- File path validation with traversal prevention
- Numeric validation with bounds checking
- JSON validation with depth limits
- UUID format validation

### Testing & Detection

- Real XSS execution testing (not just pattern matching)
- NoSQL injection payload analysis
- Command injection risk assessment
- Path traversal vulnerability detection
- Unicode attack vector identification
- Comprehensive security reporting

## Timeline

**Total Estimated Time: 6-8 hours**

- Step 1: 2-3 hours (Core extraction and organization)
- Step 2: 1-2 hours (Framework integrations)
- Step 3: 1 hour (Update chaos-monkey)
- Step 4: 1-2 hours (Documentation and examples)
- Step 5: 1 hour (Security validation and testing)

## Success Criteria

- [ ] All original tests pass with extracted service
- [ ] New security service has 100% test coverage
- [ ] Documentation includes security best practices
- [ ] Framework integrations work seamlessly
- [ ] Performance benchmarks meet or exceed original
- [ ] Security vulnerability tests all pass
- [ ] Ready for external distribution to NPM registry

## Future Enhancements

- Rate limiting middleware
- Content Security Policy generator
- Security header management
- Audit logging integration
- Plugin architecture for custom sanitizers
- Machine learning based attack detection
- Real-time security monitoring dashboard
