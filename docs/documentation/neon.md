# Neon Database Research

This document contains comprehensive research on Neon DB integration with Cloudflare Workers and dynamic database creation patterns.

## Table of Contents

1. [Neon DB on Cloudflare Workers](#neon-db-on-cloudflare-workers)
2. [Dynamic Database Creation](#dynamic-database-creation)

---

## Neon DB on Cloudflare Workers

### Connection Methods

#### Method 1: Neon Serverless Driver (Direct Connection)

The Neon serverless driver (`@neondatabase/serverless`) is specifically designed for edge environments like Cloudflare Workers.

**Installation:**

```bash
npm install @neondatabase/serverless
```

**Basic Connection Example:**

```javascript
import { Client } from "@neondatabase/serverless";

export default {
  async fetch(request, env, ctx) {
    const client = new Client(env.DATABASE_URL);
    await client.connect();

    try {
      const { rows } = await client.query("SELECT * FROM users LIMIT 10;");
      return new Response(JSON.stringify(rows), {
        headers: { "Content-Type": "application/json" },
      });
    } finally {
      ctx.waitUntil(client.end()); // Proper connection cleanup
    }
  },
};
```

#### Method 2: Hyperdrive (Recommended for Performance)

Hyperdrive provides connection pooling and reduced latency through Cloudflare's network.

**Setup with Wrangler:**

```bash
# Create Hyperdrive configuration
npx wrangler hyperdrive create neon-hyperdrive --connection-string="postgresql://username:password@host/database"

# Update wrangler.toml
[[hyperdrive]]
binding = "HYPERDRIVE"
id = "your-hyperdrive-id"
```

**Hyperdrive Connection Example:**

```javascript
import { Client } from "pg";

export default {
  async fetch(request, env, ctx) {
    const client = new Client({
      connectionString: env.HYPERDRIVE.connectionString,
    });
    await client.connect();

    try {
      const result = await client.query("SELECT NOW()");
      return new Response(JSON.stringify(result.rows));
    } finally {
      ctx.waitUntil(client.end());
    }
  },
};
```

### Connection Pooling and Performance Considerations

#### Connection Pool Limits

- **Cloudflare Workers**: Maximum 6 concurrent TCP connections per Worker
- **Recommended pool size**: 5 connections maximum to stay within limits
- **Neon limits**: Up to 4,000 direct connections, 10,000 pooled connections

#### Performance Best Practices

**Pool Configuration for Multiple Queries:**

```javascript
import { Pool } from "@neondatabase/serverless";

const pool = new Pool({
  connectionString: env.DATABASE_URL,
  max: 5, // Stay within Workers' connection limits
});

export default {
  async fetch(request, env, ctx) {
    const client = await pool.connect();

    try {
      // Multiple parallel queries
      const [users, products] = await Promise.all([
        client.query("SELECT * FROM users"),
        client.query("SELECT * FROM products"),
      ]);

      return new Response(
        JSON.stringify({ users: users.rows, products: products.rows })
      );
    } finally {
      client.release();
      ctx.waitUntil(pool.end());
    }
  },
};
```

#### Critical Performance Notes

- **Connection Lifecycle**: Connections cannot outlive a single request in Workers
- **Always Close**: Use `ctx.waitUntil()` for proper cleanup without blocking response
- **Connection Pooling**: Enable pooled connections in Neon (add `-pooler` to connection string)

### Authentication and Security Best Practices

#### Secure Credential Management

```bash
# Store database URL as encrypted environment variable
npx wrangler secret put DATABASE_URL
```

#### Connection String Format

```
postgresql://username:password@host-pooler.region.aws.neon.tech/database?sslmode=require&channel_binding=require
```

#### Security Configuration Example

```javascript
// wrangler.toml
[env.production.vars];
NODE_ENV = "production";

// Use secrets for sensitive data
const client = new Client({
  connectionString: env.DATABASE_URL, // From Wrangler secrets
  ssl: {
    rejectUnauthorized: true,
  },
});
```

#### Built-in Security Features

- **End-to-end TLS encryption** between Workers and Neon
- **Row-level security** supported in Neon
- **DDoS protection** and request filtering via Cloudflare
- **Channel binding** for additional security (`channel_binding=require`)

### Complete Worker with Drizzle ORM

```typescript
import { Hono } from "hono";
import { drizzle } from "drizzle-orm/neon-http";
import { neon } from "@neondatabase/serverless";
import { users, products } from "./schema";

type Bindings = {
  DATABASE_URL: string;
};

const app = new Hono<{ Bindings: Bindings }>();

app.get("/users", async (c) => {
  const sql = neon(c.env.DATABASE_URL);
  const db = drizzle(sql);

  const result = await db.select().from(users).limit(50);
  return c.json({ users: result });
});

app.post("/users", async (c) => {
  const { name, email } = await c.req.json();
  const sql = neon(c.env.DATABASE_URL);
  const db = drizzle(sql);

  const result = await db.insert(users).values({ name, email }).returning();
  return c.json({ user: result[0] });
});

export default app;
```

### Database Schema (Drizzle)

```typescript
// schema.ts
import { pgTable, serial, varchar, timestamp } from "drizzle-orm/pg-core";

export const users = pgTable("users", {
  id: serial("id").primaryKey(),
  name: varchar("name", { length: 100 }).notNull(),
  email: varchar("email", { length: 255 }).notNull().unique(),
  createdAt: timestamp("created_at").defaultNow(),
});

export const products = pgTable("products", {
  id: serial("id").primaryKey(),
  name: varchar("name", { length: 200 }).notNull(),
  price: varchar("price", { length: 50 }).notNull(),
  description: varchar("description", { length: 500 }),
});
```

### Wrangler Configuration

```toml
# wrangler.toml
name = "neon-worker-api"
main = "src/index.ts"
compatibility_date = "2024-08-01"

[env.production]
vars = { NODE_ENV = "production" }

[[env.production.hyperdrive]]
binding = "HYPERDRIVE"
id = "your-hyperdrive-id"
```

### Limitations and Considerations

#### Connection Limitations

- **Request Scoped**: Connections must be created and destroyed within a single request
- **No Persistent Connections**: Cannot maintain connections across requests
- **Connection Limit**: Maximum 6 concurrent TCP connections per Worker execution
- **WebSocket Lifecycle**: WebSocket connections cannot outlive a single request

#### Performance Considerations

- **Cold Starts**: First request may have higher latency
- **Connection Overhead**: Each request creates new connections
- **Geographic Distribution**: Workers run globally, but database location matters

#### Common Issues and Solutions

```javascript
// ❌ Wrong: Creating pool outside request handler
const pool = new Pool({ connectionString: DATABASE_URL });

export default {
  async fetch(request, env, ctx) {
    // This will cause connection issues
  }
};

// ✅ Correct: Create and destroy within request
export default {
  async fetch(request, env, ctx) {
    const client = new Client(env.DATABASE_URL);
    await client.connect();

    try {
      const result = await client.query('SELECT 1');
      return new Response(JSON.stringify(result.rows));
    } finally {
      ctx.waitUntil(client.end());
    }
  }
};
```

### Recommended Libraries and SDKs

#### Core Libraries

- **@neondatabase/serverless**: Official Neon driver for serverless environments
- **drizzle-orm**: Type-safe ORM with excellent Neon support
- **hono**: Lightweight web framework optimized for Workers

#### Installation Command

```bash
npm install @neondatabase/serverless drizzle-orm hono
npm install -D drizzle-kit @types/node
```

#### Alternative Stack with Hyperdrive

```bash
npm install pg drizzle-orm
npm install -D @types/pg drizzle-kit
```

#### Development Tools

- **Wrangler CLI**: For deployment and local development
- **Drizzle Kit**: For database migrations and schema management
- **Neon CLI**: For database management

#### Migration Example

```typescript
// drizzle.config.ts
import type { Config } from "drizzle-kit";

export default {
  schema: "./src/schema.ts",
  out: "./drizzle",
  driver: "pg",
  dbCredentials: {
    connectionString: process.env.DATABASE_URL!,
  },
} satisfies Config;
```

---

## Dynamic Database Creation

### Neon API for Database Creation and Management

#### Core API Architecture

- **REST API**: Full resource-oriented URLs with JSON responses
- **Base URL**: `https://console.neon.tech/api/v2/`
- **OpenAPI Specification**: Available at `https://neon.tech/api_spec/release/v2.json`
- **TypeScript SDK**: `@neondatabase/api-client` provides convenient wrapper methods
- **Instant Provisioning**: Databases created in milliseconds using serverless architecture

#### API Rate Limits and Performance

- **Rate Limit**: 700 requests per minute (approximately 11 per second)
- **Burst Capacity**: Up to 40 requests per second per route
- **Error Handling**: Returns HTTP 429 when limits exceeded
- **Best Practice**: Implement exponential backoff retry mechanisms

#### Key API Endpoints

```typescript
// Project Management
POST /projects                           // Create new project
GET /projects                            // List all projects
GET /projects/{project_id}               // Get specific project
PATCH /projects/{project_id}             // Update project settings
DELETE /projects/{project_id}            // Delete project

// Branch Management
POST /projects/{project_id}/branches                    // Create branch
GET /projects/{project_id}/branches                     // List branches
GET /projects/{project_id}/branches/{branch_id}         // Get branch details
DELETE /projects/{project_id}/branches/{branch_id}      // Delete branch
POST /projects/{project_id}/branches/{branch_id}/set_default  // Set default branch

// Database Management
POST /projects/{project_id}/branches/{branch_id}/databases     // Create database
GET /projects/{project_id}/branches/{branch_id}/databases      // List databases
DELETE /projects/{project_id}/branches/{branch_id}/databases/{database_name}  // Delete database

// Compute Endpoint Management
POST /projects/{project_id}/endpoints        // Create compute endpoint
GET /projects/{project_id}/endpoints         // List endpoints
PATCH /projects/{project_id}/endpoints/{endpoint_id}  // Update endpoint
DELETE /projects/{project_id}/endpoints/{endpoint_id}  // Delete endpoint

// Role Management  
POST /projects/{project_id}/branches/{branch_id}/roles         // Create role
GET /projects/{project_id}/branches/{branch_id}/roles          // List roles
DELETE /projects/{project_id}/branches/{branch_id}/roles/{role_name}  // Delete role

// Connection Management
GET /projects/{project_id}/connection_uri   // Get connection URI

// API Key Management
GET /api_keys                               // List API keys
POST /api_keys                              // Create API key
DELETE /api_keys/{key_id}                   // Revoke API key
```

#### Project Creation Parameters

When creating a project, you can specify:

```typescript
interface CreateProjectRequest {
  project: {
    name?: string;                    // Project name (max 64 characters)
    region_id?: string;              // AWS region (e.g., "us-east-1", "eu-central-1")
    pg_version?: number;             // PostgreSQL version (14, 15, 16, 17)
    org_id?: string;                 // Organization ID (optional)
    history_retention_seconds?: number;  // Point-in-time recovery retention
    default_endpoint_settings?: {
      autoscaling_limit_min_cu?: number;  // Minimum compute units
      autoscaling_limit_max_cu?: number;  // Maximum compute units
      suspend_timeout_seconds?: number;   // Auto-suspend timeout
    };
  };
}
```

#### Database Creation within Branches

```typescript
interface CreateDatabaseRequest {
  database: {
    name: string;                    // Database name
    owner_name: string;              // Database owner role name
  };
}
```

#### Connection URI Structure

Neon provides multiple connection URI formats:

```typescript
// Direct connection (not pooled)
postgresql://username:password@hostname/database

// Pooled connection (recommended for serverless)
postgresql://username:password@hostname-pooler/database

// With SSL and channel binding (production recommended)
postgresql://username:password@hostname-pooler/database?sslmode=require&channel_binding=require
```

### Authentication and API Keys

#### API Key Types (2025)

1. **Personal API Keys**: Tied to individual accounts, access personal projects
2. **Organization API Keys**: Full admin access to organization projects
3. **Project-scoped Keys**: Limited to specific projects with member-level access

#### Setup Process

```bash
# Set environment variable
export NEON_API_KEY="your_64_bit_token_here"
```

#### Authentication Header

```bash
curl 'https://console.neon.tech/api/v2/projects' \
  -H 'Authorization: Bearer $NEON_API_KEY' \
  -H 'Content-Type: application/json'
```

### Code Examples for Dynamic Database Creation

#### TypeScript SDK Installation

```bash
npm install @neondatabase/api-client
```

#### Complete Implementation Example

```typescript
import { createApiClient } from "@neondatabase/api-client";

const apiClient = createApiClient({
  apiKey: process.env.NEON_API_KEY!,
});

// Create a new project for a tenant
async function createTenantProject(tenantName: string) {
  const response = await apiClient.createProject({
    project: {
      name: `tenant-${tenantName}`,
      region_id: "us-east-1",          // Choose appropriate region
      pg_version: 17,                  // Latest PostgreSQL version
      history_retention_seconds: 604800, // 7 days retention
      default_endpoint_settings: {
        autoscaling_limit_min_cu: 0.25,  // Scale to zero capability
        autoscaling_limit_max_cu: 4,     // Maximum 4 compute units
        suspend_timeout_seconds: 300,    // Auto-suspend after 5 minutes
      },
    },
  });
  return response.data.project;
}

// Create a branch for development/testing
async function createBranch(projectId: string, branchName: string, parentBranchId?: string) {
  const response = await apiClient.createProjectBranch(projectId, {
    branch: { 
      name: branchName,
      parent_id: parentBranchId,  // Optional: branch from specific parent
    },
    endpoints: [
      {
        type: "read_write",
        autoscaling_limit_min_cu: 0.25,
        autoscaling_limit_max_cu: 2,
        suspend_timeout_seconds: 300,
      },
    ],
  });
  return response.data.branch;
}

// Create role (user) within branch
async function createRole(projectId: string, branchId: string, roleName: string) {
  const response = await apiClient.createProjectBranchRole(
    projectId,
    branchId,
    {
      role: {
        name: roleName,
      },
    }
  );
  return response.data.role;
}

// Create database within branch
async function createDatabase(
  projectId: string,
  branchId: string,
  databaseName: string,
  ownerName: string
) {
  const response = await apiClient.createProjectBranchDatabase(
    projectId,
    branchId,
    {
      database: {
        name: databaseName,
        owner_name: ownerName,
      },
    }
  );
  return response.data.database;
}

// Get connection URI for project
async function getConnectionUri(
  projectId: string, 
  branchId?: string, 
  endpointId?: string,
  databaseName?: string,
  roleName?: string,
  pooled: boolean = true
) {
  const response = await apiClient.getConnectionUri({
    projectId,
    branchId,
    endpointId,
    databaseName,
    roleName,
    pooled,  // Use connection pooling (recommended for serverless)
  });
  return response.data.uri;
}

// Complete tenant setup with proper error handling
async function setupNewTenant(tenantId: string) {
  try {
    console.log(`🚀 Setting up tenant: ${tenantId}`);

    // 1. Create project
    console.log("Creating Neon project...");
    const project = await createTenantProject(tenantId);
    console.log(`✅ Project created: ${project.id}`);

    // 2. Create tenant role
    console.log("Creating tenant admin role...");
    const role = await createRole(
      project.id,
      project.default_branch_id,
      "tenant_admin"
    );
    console.log(`✅ Role created: ${role.name}`);

    // 3. Create production database
    console.log("Creating main database...");
    const database = await createDatabase(
      project.id,
      project.default_branch_id,
      "main_db",
      role.name
    );
    console.log(`✅ Database created: ${database.name}`);

    // 4. Create development branch
    console.log("Creating development branch...");
    const devBranch = await createBranch(
      project.id, 
      "development",
      project.default_branch_id
    );
    console.log(`✅ Dev branch created: ${devBranch.id}`);

    // 5. Get production connection string
    const connectionString = await getConnectionUri(
      project.id,
      project.default_branch_id,
      undefined,  // Use default endpoint
      database.name,
      role.name,
      true  // Use pooled connection
    );

    // 6. Get development connection string
    const devConnectionString = await getConnectionUri(
      project.id,
      devBranch.id,
      undefined,
      database.name,
      role.name,
      true
    );

    console.log(`🎉 Tenant setup completed for: ${tenantId}`);

    return {
      tenantId,
      neonProjectId: project.id,
      defaultBranchId: project.default_branch_id,
      devBranchId: devBranch.id,
      databaseName: database.name,
      roleName: role.name,
      connectionString,
      devConnectionString,
      region: project.region_id,
      createdAt: new Date().toISOString(),
    };

  } catch (error: any) {
    console.error(`❌ Tenant setup failed for ${tenantId}:`, error.message);
    
    // Enhanced error handling for common issues
    if (error.response?.status === 429) {
      throw new Error("Rate limit exceeded. Please retry after a delay.");
    } else if (error.response?.status === 401) {
      throw new Error("Invalid API key. Check NEON_API_KEY environment variable.");
    } else if (error.response?.status === 402) {
      throw new Error("Payment required. Check your Neon billing status.");
    } else if (error.message?.includes("project limit")) {
      throw new Error("Project limit reached. Upgrade your Neon plan or delete unused projects.");
    }
    
    throw error;
  }
}

// Utility function to clean up a tenant (delete project)
async function cleanupTenant(neonProjectId: string) {
  try {
    console.log(`🧹 Cleaning up tenant project: ${neonProjectId}`);
    await apiClient.deleteProject(neonProjectId);
    console.log(`✅ Project deleted: ${neonProjectId}`);
  } catch (error: any) {
    if (error.response?.status === 404) {
      console.log(`⚠️ Project not found: ${neonProjectId} (already deleted)`);
    } else {
      console.error(`❌ Failed to delete project ${neonProjectId}:`, error.message);
      throw error;
    }
  }
}
```

#### cURL Examples for Direct API Usage

```bash
# Create a new project
curl -X POST 'https://console.neon.tech/api/v2/projects' \
  -H 'Authorization: Bearer $NEON_API_KEY' \
  -H 'Content-Type: application/json' \
  -d '{
    "project": {
      "name": "tenant-example",
      "region_id": "us-east-1",
      "pg_version": 17,
      "default_endpoint_settings": {
        "autoscaling_limit_min_cu": 0.25,
        "autoscaling_limit_max_cu": 4,
        "suspend_timeout_seconds": 300
      }
    }
  }'

# Create a new branch
curl -X POST 'https://console.neon.tech/api/v2/projects/{project_id}/branches' \
  -H 'Authorization: Bearer $NEON_API_KEY' \
  -H 'Content-Type: application/json' \
  -d '{
    "branch": {
      "name": "development"
    },
    "endpoints": [{
      "type": "read_write",
      "autoscaling_limit_min_cu": 0.25,
      "autoscaling_limit_max_cu": 2
    }]
  }'

# Create a database
curl -X POST 'https://console.neon.tech/api/v2/projects/{project_id}/branches/{branch_id}/databases' \
  -H 'Authorization: Bearer $NEON_API_KEY' \
  -H 'Content-Type: application/json' \
  -d '{
    "database": {
      "name": "tenant_db",
      "owner_name": "neondb_owner"
    }
  }'

# Get connection URI
curl -X GET 'https://console.neon.tech/api/v2/projects/{project_id}/connection_uri?database_name=tenant_db&role_name=neondb_owner&pooled=true' \
  -H 'Authorization: Bearer $NEON_API_KEY'

# List all projects
curl -X GET 'https://console.neon.tech/api/v2/projects' \
  -H 'Authorization: Bearer $NEON_API_KEY'

# Delete a project (cleanup)
curl -X DELETE 'https://console.neon.tech/api/v2/projects/{project_id}' \
  -H 'Authorization: Bearer $NEON_API_KEY'
```

### Branch Creation and Management

#### Branch Features

- **Instant Creation**: Copy-on-write technology makes branching virtually free
- **Git-like Workflow**: Branch from any point in database history
- **Isolation**: Each branch has independent compute resources

#### Advanced Branch Management

```typescript
// Create branch from specific timestamp
async function createBranchFromTimestamp(
  projectId: string,
  parentBranchId: string,
  timestamp: string
) {
  const branch = await apiClient.createProjectBranch(projectId, {
    branch: {
      name: `restore-${Date.now()}`,
      parent_id: parentBranchId,
      parent_timestamp: timestamp,
    },
    endpoints: [{ type: "read_write" }],
  });
  return branch.data.branch;
}

// List all branches
async function getAllBranches(projectId: string) {
  const branches = await apiClient.listProjectBranches(projectId);
  return branches.data.branches;
}
```

### Automation Patterns and Best Practices

#### Multi-Tenant Architecture Pattern

```typescript
// Single-Project Multi-Database Architecture
class AgoraTenantManager {
  private apiClient: any;
  private platformDb: any; // Platform database for control plane

  constructor(apiKey: string, platformProjectId: string) {
    this.apiClient = createApiClient({ apiKey });
    this.platformProjectId = platformProjectId;
  }

  async onboardTenant(tenantConfig: {
    id: string;
    name: string;
    plan: "basic" | "premium";
  }) {
    // 1. Create tenant database in same project as platform
    const tenantDbName = `tenant_${tenantConfig.id.replace(/[^a-zA-Z0-9]/g, '_')}`;
    const database = await this.createTenantDatabase(tenantDbName);

    // 2. Apply initial schema to tenant database
    await this.applyInitialSchema(tenantDbName);

    // 3. Register in platform database with encrypted connection string
    await this.platformDb.clients.create({
      client_id: tenantConfig.id,
      organization_name: tenantConfig.name,
      // Connection string encrypted and stored securely
    });

    return database;
  }

  private async setResourceLimits(projectId: string, plan: string) {
    const limits =
      plan === "premium"
        ? {
            active_time_seconds: 86400, // 24 hours
            compute_time_seconds: 3600000, // 1000 hours
            logical_size_bytes: 10737418240, // 10GB
          }
        : {
            active_time_seconds: 3600, // 1 hour
            compute_time_seconds: 360000, // 100 hours
            logical_size_bytes: 1073741824, // 1GB
          };

    await this.apiClient.updateProject(projectId, {
      project: { quota: limits },
    });
  }
}
```

#### CI/CD Integration Pattern

```typescript
// CI/CD Pipeline Integration
async function createEphemeralEnvironment(
  prNumber: string,
  baseBranchId: string
) {
  const environmentName = `pr-${prNumber}`;

  // Create branch for PR environment
  const branch = await apiClient.createProjectBranch(projectId, {
    branch: {
      name: environmentName,
      parent_id: baseBranchId,
    },
    endpoints: [
      {
        type: "read_write",
        autoscaling_limit_max_cu: 1, // Limit resources for testing
      },
    ],
  });

  // Return connection URL for deployment
  return {
    branchId: branch.data.branch.id,
    connectionUrl: branch.data.connection_uris[0].connection_uri,
    cleanup: () =>
      apiClient.deleteProjectBranch(projectId, branch.data.branch.id),
  };
}
```

### Rate Limits and Quotas

#### API Rate Limits

- **Global Limits**: Applied across all endpoints
- **Authentication**: API token recommended for intensive usage
- **Best Practice**: Implement exponential backoff for retry logic

#### Resource Quotas (Per Project)

```typescript
// Quota configuration options
const quotaSettings = {
  active_time_seconds: 86400, // Max active compute time
  compute_time_seconds: 3600000, // Max total CPU seconds
  logical_size_bytes: 10737418240, // Max branch size (10GB)
};

// Apply quotas during project creation
await apiClient.createProject({
  project: {
    name: "tenant-project",
    quota: quotaSettings,
  },
});
```

### Error Handling and Monitoring

#### Common API Errors and Solutions

```typescript
// Comprehensive error handling for Neon API calls
async function handleNeonApiCall<T>(apiCall: () => Promise<T>): Promise<T> {
  try {
    return await apiCall();
  } catch (error: any) {
    const status = error.response?.status;
    const errorData = error.response?.data;
    
    switch (status) {
      case 400:
        throw new Error(`Bad Request: ${errorData?.message || 'Invalid request parameters'}`);
      case 401:
        throw new Error('Unauthorized: Invalid or expired API key');
      case 402:
        throw new Error('Payment Required: Billing issue or quota exceeded');
      case 403:
        throw new Error('Forbidden: Insufficient permissions for this operation');
      case 404:
        throw new Error('Not Found: Resource does not exist');
      case 409:
        throw new Error(`Conflict: ${errorData?.message || 'Resource already exists'}`);
      case 422:
        throw new Error(`Validation Error: ${errorData?.message || 'Invalid data provided'}`);
      case 429:
        const retryAfter = error.response?.headers['retry-after'] || 60;
        throw new Error(`Rate Limited: Retry after ${retryAfter} seconds`);
      case 500:
      case 502:
      case 503:
        throw new Error('Server Error: Neon service temporarily unavailable');
      default:
        throw new Error(`API Error: ${errorData?.message || error.message}`);
    }
  }
}

// Usage example with retry logic
async function createProjectWithRetry(projectData: any, maxRetries = 3): Promise<any> {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await handleNeonApiCall(() => 
        apiClient.createProject({ project: projectData })
      );
    } catch (error: any) {
      if (error.message.includes('Rate Limited') && attempt < maxRetries) {
        const delay = Math.pow(2, attempt) * 1000; // Exponential backoff
        console.log(`Rate limited, retrying in ${delay}ms... (attempt ${attempt})`);
        await new Promise(resolve => setTimeout(resolve, delay));
        continue;
      }
      throw error;
    }
  }
}
```

#### API Response Monitoring

```typescript
// Monitor API usage and performance
interface ApiMetrics {
  totalRequests: number;
  successfulRequests: number;
  errorRequests: number;
  averageResponseTime: number;
  rateLimitHits: number;
}

class NeonApiMonitor {
  private metrics: ApiMetrics = {
    totalRequests: 0,
    successfulRequests: 0,
    errorRequests: 0,
    averageResponseTime: 0,
    rateLimitHits: 0,
  };

  async monitoredApiCall<T>(apiCall: () => Promise<T>): Promise<T> {
    const startTime = Date.now();
    this.metrics.totalRequests++;

    try {
      const result = await apiCall();
      this.metrics.successfulRequests++;
      this.updateResponseTime(Date.now() - startTime);
      return result;
    } catch (error: any) {
      this.metrics.errorRequests++;
      if (error.response?.status === 429) {
        this.metrics.rateLimitHits++;
      }
      throw error;
    }
  }

  private updateResponseTime(responseTime: number) {
    this.metrics.averageResponseTime = 
      (this.metrics.averageResponseTime * (this.metrics.successfulRequests - 1) + responseTime) 
      / this.metrics.successfulRequests;
  }

  getMetrics(): ApiMetrics {
    return { ...this.metrics };
  }
}
```

### Cost Implications and Optimization

#### Pricing Tiers (2025)

- **Free**: 10 projects, 0.5GB storage, 190 compute hours
- **Launch** ($19/month): 100 projects, 10GB storage, 300 compute hours  
- **Scale** ($69/month): 1,000 projects, 50GB storage, 750 compute hours
- **Business** ($700/month): 5,000 projects, 500GB storage, 1,000 compute hours

#### Cost Optimization Strategies

```typescript
// Implement cost-conscious tenant creation
async function createCostOptimizedTenant(tenantId: string, tier: 'basic' | 'premium') {
  const config = tier === 'premium' ? {
    autoscaling_limit_min_cu: 0.25,
    autoscaling_limit_max_cu: 8,
    suspend_timeout_seconds: 600,  // 10 minutes for premium
    history_retention_seconds: 2592000,  // 30 days
  } : {
    autoscaling_limit_min_cu: 0.25,
    autoscaling_limit_max_cu: 2,
    suspend_timeout_seconds: 300,  // 5 minutes for basic
    history_retention_seconds: 604800,  // 7 days
  };

  return await createTenantProject(tenantId, config);
}

// Monitor and alert on usage
async function checkTenantUsage(neonProjectId: string) {
  try {
    const project = await apiClient.getProject(neonProjectId);
    const usage = project.data.project.quota;
    
    const alerts = [];
    
    // Check compute usage (80% threshold)
    if (usage.compute_time_seconds > usage.active_time_seconds * 0.8) {
      alerts.push({
        type: 'compute_usage_high',
        current: usage.compute_time_seconds,
        limit: usage.active_time_seconds,
        percentage: (usage.compute_time_seconds / usage.active_time_seconds) * 100
      });
    }
    
    // Check storage usage (80% threshold)  
    if (usage.logical_size_bytes > usage.data_transfer_bytes * 0.8) {
      alerts.push({
        type: 'storage_usage_high',
        current: usage.logical_size_bytes,
        limit: usage.data_transfer_bytes,
        percentage: (usage.logical_size_bytes / usage.data_transfer_bytes) * 100
      });
    }
    
    return { usage, alerts };
  } catch (error) {
    console.error(`Failed to check usage for ${neonProjectId}:`, error);
    return null;
  }
}
```

#### Cost Optimization Best Practices

- **Scale-to-Zero**: Configure short suspend timeouts for development environments
- **Resource Limits**: Set appropriate compute unit limits based on tenant tier
- **Branch Cleanup**: Regularly delete unused development branches
- **Usage Monitoring**: Implement automated alerts for quota thresholds
- **Regional Optimization**: Choose regions close to your users to reduce latency costs

### Integration Patterns for Multi-Tenant Applications

#### Database-per-Tenant Pattern (Single Project Architecture)

```typescript
// Tenant isolation strategy using separate databases
class AgoraTenantIsolationManager {
  async isolateTenant(tenantId: string, platformProjectId: string) {
    // Each tenant gets dedicated database within shared project
    const tenantDbName = `tenant_${tenantId.replace(/[^a-zA-Z0-9]/g, '_')}`;
    const database = await this.createTenantDatabase(platformProjectId, tenantDbName);

    return {
      // Complete data isolation via separate database
      databaseName: tenantDbName,
      // Shared compute resources (more cost-effective)
      projectId: platformProjectId,
      // Same regional placement as platform
      isolationLevel: 'database-level',
      // Shared scaling with platform
      costOptimized: true,
    };
  }
}
```

#### Advanced Multi-Tenant Features

- **Database-Level Isolation**: Complete data separation within shared project
- **Cost Optimization**: Shared compute resources reduce per-tenant costs
- **Encrypted Storage**: Tenant connection strings encrypted in platform database
- **Automated Management**: Step-by-step tenant creation via `/api/create-tenant-steps`
- **Scalable Architecture**: Single project can host thousands of tenant databases

#### Schema Migration Strategy

```typescript
async function applyMigrationToAllTenants(migrationSql: string) {
  const platformDb = getControlPlaneClient();
  const repositoryManager = createControlPlaneRepositoryManager(platformDb);
  
  // Get all active tenants from platform database
  const tenants = await repositoryManager.clients.findAll();

  for (const tenant of tenants) {
    try {
      // Get encrypted tenant connection string
      const connectionString = await repositoryManager.clients.getConnectionString(tenant.client_id);
      
      if (connectionString) {
        // Connect to tenant's specific database
        const tenantClient = new NeonClient(connectionString);
        
        // Apply migration to tenant database
        await tenantClient.query(migrationSql);
        
        console.log(`✅ Migration applied to tenant: ${tenant.client_id}`);
      }
    } catch (error) {
      console.error(`❌ Migration failed for tenant ${tenant.client_id}:`, error);
      // Handle migration failure (rollback, alerts, etc.)
    }
  }
}
```

---

## API Reference Quick Guide

### Essential Endpoints Summary

| Operation | Method | Endpoint | Description |
|-----------|--------|----------|-------------|
| **Projects** | | | |
| Create Project | POST | `/projects` | Create new tenant project |
| List Projects | GET | `/projects` | Get all projects |
| Get Project | GET | `/projects/{id}` | Get project details |
| Update Project | PATCH | `/projects/{id}` | Update project settings |
| Delete Project | DELETE | `/projects/{id}` | Delete project and all data |
| **Branches** | | | |
| Create Branch | POST | `/projects/{id}/branches` | Create new branch |
| List Branches | GET | `/projects/{id}/branches` | Get all branches |
| Delete Branch | DELETE | `/projects/{id}/branches/{id}` | Delete branch |
| **Databases** | | | |
| Create Database | POST | `/projects/{id}/branches/{id}/databases` | Create database in branch |
| List Databases | GET | `/projects/{id}/branches/{id}/databases` | Get all databases |
| Delete Database | DELETE | `/projects/{id}/branches/{id}/databases/{name}` | Delete database |
| **Connection** | | | |
| Get Connection URI | GET | `/projects/{id}/connection_uri` | Get connection string |

### Environment Variables

```bash
# Required for Neon API integration
NEON_API_KEY=neon_api_your_64_character_key_here
NEON_PROJECT_ID=your_default_project_id  # Optional for build-platform fallback

# Database connection (if using existing project)
NEON_DATABASE_URL=postgresql://username:password@hostname-pooler/database?sslmode=require

# Agora platform authentication
CKN_API_KEY=ckn_api_your_generated_api_key_here
```

### Integration Checklist

When implementing Neon API integration:

- [ ] **API Key Setup**: Generate and securely store Neon API key
- [ ] **Error Handling**: Implement comprehensive error handling for all status codes
- [ ] **Rate Limiting**: Add exponential backoff retry logic for 429 responses
- [ ] **Monitoring**: Track API usage metrics and response times
- [ ] **Cost Control**: Set appropriate resource limits and quotas
- [ ] **Security**: Use pooled connections with SSL and channel binding
- [ ] **Regional Placement**: Choose optimal regions for your users
- [ ] **Cleanup Strategy**: Implement automated cleanup for unused resources
- [ ] **Testing**: Test tenant lifecycle (create, use, delete) thoroughly
- [ ] **Documentation**: Document your tenant management workflows

---

## Summary

This comprehensive research document covers:

1. **Neon + Cloudflare Workers**: Production-ready integration patterns with connection pooling, security, and performance optimization for serverless environments

2. **Dynamic Database Creation**: Complete automation patterns for multi-tenant applications using the Neon API, including:
   - Project and database creation workflows
   - Error handling and retry mechanisms  
   - Cost optimization strategies
   - Multi-tenant isolation patterns
   - CI/CD integration approaches

3. **API Reference**: Detailed endpoint documentation with TypeScript examples, cURL commands, and best practices for production deployments

Both Neon and Cloudflare Workers provide enterprise-grade solutions for globally distributed, serverless database architectures with automatic scaling, cost efficiency, and developer-friendly APIs.
