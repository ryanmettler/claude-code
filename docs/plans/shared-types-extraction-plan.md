# Shared Types Extraction Plan

## Overview

Extract common TypeScript types and interfaces from the chat worker to create a reusable shared package that can be consumed by multiple projects (agora, chaos-monkey, examples, etc.).

## Current Analysis

Based on `/workspace/workers/chat/src/types.ts`, we have the following types that could be shared:

### Highly Shareable Types (Universal)

- `ChatMessage` - OpenAI-compatible chat message format
- `ValidationError` - Generic validation error structure
- `ErrorResponse` - Standard API error response format
- `Agent` - Agent configuration and metadata
- `KnowledgeBase` - Knowledge base structure
- `AgentKnowledgeBase` - Agent-KB relationship mapping

### Project-Specific Types (Keep Local)

- `Env` - Cloudflare Worker environment bindings (specific to chat worker)
- `ChatRequest` - Chat API request (contains worker-specific fields like `agent_token`)

## Implementation Plan

### Phase 1: Create Shared Types Package

1. **Create new package structure**

   ```
   /workspace/shared-types/
   ├── package.json
   ├── tsconfig.json
   ├── src/
   │   ├── index.ts
   │   ├── chat.ts
   │   ├── agent.ts
   │   ├── knowledge-base.ts
   │   └── api.ts
   └── dist/
   ```

2. **Package configuration**

   - Name: `@workspace/shared-types`
   - Version: `1.0.0`
   - Main: `dist/index.js`
   - Types: `dist/index.d.ts`
   - Build target: ES2020, CommonJS + ESM

3. **Extract types by category**

   ```typescript
   // src/chat.ts
   export interface ChatMessage { ... }

   // src/agent.ts
   export interface Agent { ... }
   export interface AgentKnowledgeBase { ... }

   // src/knowledge-base.ts
   export interface KnowledgeBase { ... }

   // src/api.ts
   export interface ValidationError { ... }
   export interface ErrorResponse { ... }
   ```

### Phase 2: Update Consumer Projects

1. **Chat Worker** (`/workspace/workers/chat/`)

   - Add dependency: `"@workspace/shared-types": "git+file:../../shared-types"`
   - Import shared types: `import { ChatMessage, Agent } from '@workspace/shared-types'`
   - Remove duplicated types from `src/types.ts`
   - Keep worker-specific types: `Env`, modified `ChatRequest`

2. **Agora** (`/workspace/agora/`)

   - Add same dependency
   - Update existing type imports in:
     - `types/agent.ts`
     - `types/client.ts`
     - `composables/useApi.ts`
     - Components that use Agent/KB types

3. **Chaos Monkey** (`/workspace/chaos-monkey/`)

   - Add dependency for any shared types it might need
   - Currently uses minimal types, may not need immediate changes

4. **Examples** (`/workspace/examples/`)
   - Update `llm-chat-app` to use shared `ChatMessage` type
   - Update other examples as needed

### Phase 3: Distribution Strategy

**Option A: Git Dependencies (Recommended for monorepo)**

```json
{
  "dependencies": {
    "@workspace/shared-types": "git+file:../shared-types"
  }
}
```

**Option B: NPM Workspaces (Alternative)**

```json
// Root package.json
{
  "workspaces": ["workers/*", "agora", "chaos-monkey", "shared-types"]
}
```

### Phase 4: Build & Validation

1. **Setup build pipeline**

   - TypeScript compilation
   - Type declaration generation
   - ESLint/Prettier configuration

2. **Validation tests**

   - Unit tests for type compatibility
   - Integration tests across projects
   - Build validation in each consumer

3. **Documentation**
   - API documentation for each type
   - Migration guide for consumers
   - Version compatibility matrix

## Migration Steps

### Step 1: Create Package (1-2 hours)

- [ ] Create `/workspace/shared-types/` directory
- [ ] Setup `package.json`, `tsconfig.json`
- [ ] Extract and organize types into separate files
- [ ] Setup build configuration
- [ ] Create barrel export in `index.ts`

### Step 2: Update Chat Worker (30 mins)

- [ ] Add git dependency to `package.json`
- [ ] Import shared types
- [ ] Remove duplicated type definitions
- [ ] Update imports throughout codebase
- [ ] Run tests to validate

### Step 3: Update Agora (1 hour)

- [ ] Add dependency
- [ ] Replace existing type definitions
- [ ] Update component imports
- [ ] Test build and runtime

### Step 4: Update Other Projects (30 mins each)

- [ ] Chaos monkey (if needed)
- [ ] Examples projects
- [ ] Any other consumers

### Step 5: Documentation & Testing (1 hour)

- [ ] Create README for shared-types package
- [ ] Add integration tests
- [ ] Validate all projects build successfully

## Benefits

- **Consistency**: Single source of truth for shared data structures
- **Maintainability**: Changes propagate automatically to all consumers
- **Type Safety**: Compile-time validation across project boundaries
- **Reusability**: Easy to add new projects that need these types

## Risks & Mitigation

- **Breaking Changes**: Use semantic versioning, maintain compatibility
- **Circular Dependencies**: Keep shared types pure data structures
- **Build Complexity**: Use simple git dependencies initially

## Timeline

**Total Estimated Time: 4-5 hours**

- Phase 1: 1-2 hours
- Phase 2-3: 2-2.5 hours
- Phase 4: 1 hour
- Buffer: 0.5 hours

## Success Criteria

- [ ] All projects build successfully
- [ ] No type duplication across projects
- [ ] Shared types package has comprehensive tests
- [ ] Documentation is complete and clear
- [ ] Migration path is documented for future types
