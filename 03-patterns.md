# Implementation Patterns

**Core patterns and architecture for GravityAI plugin nodes**

## 🚨 Critical Pattern: Dependency Injection

**Use `context.api` for all runtime functions - no global state.**

### ✅ CORRECT Pattern - Dependency Injection
```typescript
import { PromiseNode, type NodeExecutionContext } from "@gravityai-dev/plugin-base";

export default class MyExecutor extends PromiseNode {
  constructor() {
    super("MyNode");
  }
  
  protected async executeNode(
    inputs: any,
    config: any,
    context: NodeExecutionContext
  ) {
    // Get logger from injected API
    const logger = context.api?.createLogger?.("MyNode") || console;
    
    // Use API functions
    await context.api.gravityPublish(channel, message);
    const credentials = await context.api.getNodeCredentials(ctx, "cred");
    
    return { __outputs: result };
  }
}
```

### CallbackNode Pattern
```typescript
import { getPlatformDependencies, type NodeExecutionContext } from "@gravityai-dev/plugin-base";

const { CallbackNode } = getPlatformDependencies();

export default class MyCallbackExecutor extends CallbackNode {
  // CallbackNode still needs getPlatformDependencies for base class
  // But use context.api for runtime functions in handleEvent
  async handleEvent(event, state, emit) {
    const executionContext = (this as any).executionContext;
    // Use context.api for runtime functions
    const logger = executionContext.api?.createLogger?.("MyNode") || console;
  }
}

## 🏗️ Plugin Architecture Patterns

### 1. Plugin Definition Pattern
```typescript
// src/index.ts
import { createPlugin, type GravityPluginAPI } from "@gravityai-dev/plugin-base";

const plugin = createPlugin({
  name: packageJson.name,
  version: packageJson.version,
  description: packageJson.description,

  async setup(api: GravityPluginAPI) {
    // Import and register nodes
    const { MyNode } = await import("./MyNode/node");
    api.registerNode(MyNode);

    // Register credentials
    const { MyCredential } = await import("./credentials");
    api.registerCredential(MyCredential);
  },
});

export default plugin;
```

### 2. Node Definition Pattern
```typescript
// src/MyNode/node/index.ts
import { NodeInputType, type EnhancedNodeDefinition } from "@gravityai-dev/plugin-base";
import MyNodeExecutor from "./executor";

function createNodeDefinition(): EnhancedNodeDefinition {
  
  return {
    type: "MyNode",
    name: "My Node",
    description: "Does something useful",
    category: "AI", // AI, Flow, Output, Storage, Ingest
    inputs: [
      { name: "input", type: NodeInputType.STRING, required: true }
    ],
    outputs: [
      { name: "output", type: NodeInputType.STRING }
    ],
    configSchema: {
      type: "object",
      properties: {
        // Configuration fields
      }
    },
    capabilities: {
      isTrigger: false,
    },
  };
}

export const MyNode = {
  definition: createNodeDefinition(),
  executor: MyNodeExecutor,
};
```

### 3. Executor Pattern (PromiseNode)
```typescript
// src/MyNode/node/executor.ts
import { PromiseNode, type NodeExecutionContext, type ValidationResult } from "@gravityai-dev/plugin-base";

export default class MyExecutor extends PromiseNode {
  constructor() {
    super("MyNode");
  }

  protected async validateConfig(config: MyConfig): Promise<ValidationResult> {
    // Simple validation - let service handle details
    return { success: true };
  }

  protected async executeNode(
    inputs: Record<string, any>,
    config: MyConfig,
    context: NodeExecutionContext
  ): Promise<MyOutput> {
    const nodeId = context.nodeId;
    const startTime = Date.now();

    this.logger.info(`🚀 [MyNode] Starting execution for node: ${nodeId}`);

    // Build credential context
    const credentialContext = this.buildCredentialContext(context);

    // Call service with injected API
    const result = await myService(config, credentialContext, context.api);

    // Return with __outputs wrapper
    const finalResult = {
      __outputs: {
        output: result.text,
        metadata: result.metadata,
      },
    };

    this.logger.info(
      `🎯 [MyNode] Returning result for node: ${nodeId}, total execution: ${Date.now() - startTime}ms`
    );

    return finalResult;
  }

  private buildCredentialContext(context: NodeExecutionContext) {
    const { workflowId, executionId, nodeId } = this.validateAndGetContext(context);

    return {
      workflowId,
      executionId,
      nodeId,
      nodeType: this.nodeType,
      config: context.config,
      credentials: context.credentials || {},
    };
  }
}
```

### 4. Service Pattern
```typescript
// src/MyNode/service/index.ts

export async function myService(config: MyConfig, credentialContext: any, api: any) {
  // Services fetch credentials from injected API
  const credentials = await api.getNodeCredentials(credentialContext, "myCredential");
  
  // Business logic here
  const response = await externalAPI(config, credentials);
  
  return {
    text: response.data,
    metadata: { tokens: response.usage }
  };
}
```

## 🔧 API Injection Pattern

### Services Receive API Parameter
```typescript
export async function myService(config: any, credentialContext: any, api: any) {
  // Use injected API
  const credentials = await api.getNodeCredentials(credentialContext, "myCredential");
  
  // Use credentials...
  const result = await apiCall(credentials);
  
  // Save token usage if applicable
  if (result.usage) {
    await api.saveTokenUsage({
      workflowId: credentialContext.workflowId,
      executionId: credentialContext.executionId,
      nodeId: credentialContext.nodeId,
      nodeType: credentialContext.nodeType,
      model: "my-model",
      inputTokens: result.usage.input,
      outputTokens: result.usage.output,
      totalTokens: result.usage.total,
      timestamp: new Date(),
    });
  }
  
  return result;
}
```

## 🎯 Output Patterns

### PromiseNode Output
```typescript
// Always wrap outputs in __outputs
return {
  __outputs: {
    text: result.text,
    metadata: result.metadata,
    usage: result.usage,
  },
};
```

### CallbackNode Output
```typescript
// Emit outputs during processing
emit({
  __outputs: {
    item: processedItem,
    index: currentIndex,
    hasMore: currentIndex < totalItems - 1,
  },
});
```

## 🚨 Critical Rules

### 1. API Injection Pattern
- ✅ **ALWAYS** use `context.api` for runtime functions
- ✅ **ALWAYS** pass `api` parameter to services
- ❌ **NEVER** use global state or `getPlatformDependencies()` for runtime functions

### 2. Executor Rules
- ✅ **ALWAYS** extend `PromiseNode` or `CallbackNode`
- ✅ **ALWAYS** implement `executeNode()` or `handleEvent()`
- ❌ **NEVER** override `execute()` method

### 3. Credential Rules
- ✅ **ALWAYS** pass `credentialContext` to services
- ✅ **ALWAYS** let services fetch credentials
- ❌ **NEVER** access `context.credentials` directly in executors

### 4. Logger Rules
- ✅ **ALWAYS** use `this.logger` in executors
- ✅ **ALWAYS** pass logger to services
- ❌ **NEVER** create separate loggers

### 5. Output Rules
- ✅ **ALWAYS** wrap outputs in `__outputs`
- ✅ **ALWAYS** return consistent output structure
- ❌ **NEVER** return raw data without wrapper

## 🔗 Real Examples

**Study these patterns in published packages:**

### PromiseNode Patterns
- **`@gravityai-dev/aws-bedrock`** - Complete PromiseNode implementation
- **`@gravityai-dev/openai`** - API integration pattern
- **`@gravityai-dev/aws-s3`** - AWS service pattern

### CallbackNode Patterns
- **`@gravityai-dev/ingest`** - ApifyResults executor (state management)
- **`@gravityai-dev/flow`** - Loop executor (iteration pattern)

### Plugin Structure
- **Any published package** - Complete plugin structure and setup

---

**Next**: [Credential Management](./04-credentials.md) - Authentication pattern (the one rule)
