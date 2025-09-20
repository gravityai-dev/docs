# Quick Start - Essential Templates

**Get a working node in 5 minutes with copy-paste templates**

## 🎯 Choose Your Node Type

- **PromiseNode**: Single execution, single result → [Template](#promisenode-template)
- **CallbackNode**: Multiple outputs over time → [Template](#callbacknode-template)

Need help deciding? See [Node Types](./02-node-types.md)

## 📦 Package Structure Template

```
@gravityai-dev/my-node/
├── package.json
├── tsconfig.json
├── src/
│   ├── index.ts              # Plugin definition
│   ├── MyNode/
│   │   ├── node/
│   │   │   ├── index.ts      # Node definition
│   │   │   └── executor.ts   # Node executor
│   │   ├── service/
│   │   │   └── index.ts      # Business logic
│   │   └── util/
│   │       └── types.ts      # TypeScript types
│   ├── shared/
│   │   └── platform.ts       # Platform dependencies
│   └── credentials/
│       └── index.ts          # Credential definitions
```

## 🚀 PromiseNode Template

### 1. Plugin Definition (`src/index.ts`)
```typescript
import { createPlugin, type GravityPluginAPI } from "@gravityai-dev/plugin-base";
import packageJson from "../package.json";

const plugin = createPlugin({
  name: packageJson.name,
  version: packageJson.version,
  description: packageJson.description,

  async setup(api: GravityPluginAPI) {
    // Initialize platform dependencies
    const { initializePlatformFromAPI } = await import("@gravityai-dev/plugin-base");
    initializePlatformFromAPI(api);

    // Import and register node
    const { MyNodeNode } = await import("./MyNode/node");
    api.registerNode(MyNodeNode);

    // Import and register credentials
    const { MyCredential } = await import("./credentials");
    api.registerCredential(MyCredential);
  },
});

export default plugin;
```

### 2. Node Definition (`src/MyNode/node/index.ts`)
```typescript
import { getPlatformDependencies, type EnhancedNodeDefinition } from "@gravityai-dev/plugin-base";
import MyNodeExecutor from "./executor";

function createNodeDefinition(): EnhancedNodeDefinition {
  const { NodeInputType } = getPlatformDependencies();
  
  return {
    type: "MyNode",
    name: "My Node",
    description: "Does something useful",
    category: "AI", // AI, Flow, Output, Storage, Ingest, etc.
    inputs: [
      { name: "input", type: NodeInputType.STRING, required: true }
    ],
    outputs: [
      { name: "output", type: NodeInputType.STRING }
    ],
    configSchema: {
      type: "object",
      properties: {
        apiEndpoint: {
          type: "string",
          title: "API Endpoint",
          default: "https://api.example.com"
        },
        prompt: {
          type: "string",
          title: "Prompt Template",
          default: "${input.text}",
          "ui:field": "template"
        }
      },
      required: ["apiEndpoint"]
    },
    capabilities: {
      isTrigger: false,
    },
  };
}

const definition = createNodeDefinition();

export const MyNodeNode = {
  definition,
  executor: MyNodeExecutor,
};

export { createNodeDefinition };
```

### 3. Node Executor (`src/MyNode/node/executor.ts`)
```typescript
import { getPlatformDependencies, type NodeExecutionContext, type ValidationResult } from "@gravityai-dev/plugin-base";
import { MyNodeConfig, MyNodeOutput } from "../util/types";
import { myService } from "../service";

const { PromiseNode } = getPlatformDependencies();

export default class MyNodeExecutor extends PromiseNode<MyNodeConfig> {
  constructor() {
    super("MyNode");
  }

  protected async validateConfig(config: MyNodeConfig): Promise<ValidationResult> {
    // Simple validation - let service handle details
    return { success: true };
  }

  protected async executeNode(
    inputs: Record<string, any>,
    config: MyNodeConfig,
    context: NodeExecutionContext
  ): Promise<MyNodeOutput> {
    const nodeId = context.nodeId;
    const startTime = Date.now();

    this.logger.info(`🚀 [MyNode] Starting execution for node: ${nodeId}`);

    // Build credential context for service
    const credentialContext = this.buildCredentialContext(context);

    // Call service with config and credentials
    const result = await myService(config, credentialContext);

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

  /**
   * Build credential context from execution context
   */
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

### 4. Service (`src/MyNode/service/index.ts`)
```typescript
import { getNodeCredentials } from "../../shared/platform";
import { MyNodeConfig } from "../util/types";

export async function myService(config: MyNodeConfig, credentialContext: any) {
  // Fetch credentials using the credential pattern
  const credentials = await getNodeCredentials(credentialContext, "myCredential");
  
  if (!credentials?.apiKey) {
    throw new Error("API key not found in credentials");
  }

  // Your business logic here
  const response = await fetch(config.apiEndpoint, {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${credentials.apiKey}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      prompt: config.prompt,
    }),
  });

  if (!response.ok) {
    throw new Error(`API call failed: ${response.statusText}`);
  }

  const data = await response.json();
  
  return {
    text: data.result,
    metadata: {
      model: data.model,
      tokens: data.usage?.total_tokens || 0,
    },
  };
}
```

### 5. Types (`src/MyNode/util/types.ts`)
```typescript
export interface MyNodeConfig {
  apiEndpoint: string;
  prompt: string;
}

export interface MyNodeOutput {
  __outputs: {
    output: string;
    metadata: {
      model?: string;
      tokens?: number;
    };
  };
}
```

### 6. Platform Dependencies (`src/shared/platform.ts`)
```typescript
import { getPlatformDependencies } from "@gravityai-dev/plugin-base";

const deps = getPlatformDependencies();

export const getNodeCredentials = deps.getNodeCredentials;
export const saveTokenUsage = deps.saveTokenUsage;
export const createLogger = deps.createLogger;
export const PromiseNode = deps.PromiseNode;
export const CallbackNode = deps.CallbackNode;
```

### 7. Credentials (`src/credentials/index.ts`)
```typescript
// Import shared credentials from plugin-base
import { OpenAICredential } from "@gravityai-dev/plugin-base";

// Re-export for this package
export { OpenAICredential as MyCredential };

// Or define custom credential:
export const MyCredential = {
  name: "myCredential",
  type: "object",
  title: "My API Credentials",
  properties: {
    apiKey: {
      type: "string",
      title: "API Key",
      description: "Your API key"
    }
  },
  required: ["apiKey"]
};
```

## 🔄 CallbackNode Template

For CallbackNode (streaming/iterative), replace the executor with:

```typescript
import { getPlatformDependencies, type NodeExecutionContext, type ValidationResult } from "@gravityai-dev/plugin-base";

const { CallbackNode } = getPlatformDependencies();

export default class MyCallbackExecutor extends CallbackNode<MyConfig, MyState> {
  constructor() {
    super("MyCallbackNode");
  }

  initializeState(inputs: any): MyState {
    return {
      items: [],
      currentIndex: 0,
      isComplete: false,
    };
  }

  protected async validateConfig(config: MyConfig): Promise<ValidationResult> {
    return { success: true };
  }

  async handleEvent(
    event: { type: string; inputs?: any; config?: any },
    state: MyState,
    emit: (output: any) => void
  ): Promise<MyState> {
    const executionContext = (this as any).executionContext;
    if (!executionContext) {
      throw new Error("Execution context not available");
    }

    // Process items one by one
    if (state.currentIndex < state.items.length) {
      const item = state.items[state.currentIndex];
      
      // Process item
      const result = await processItem(item, executionContext);
      
      // Emit result
      emit({ __outputs: result });
      
      return {
        ...state,
        currentIndex: state.currentIndex + 1,
        isComplete: state.currentIndex + 1 >= state.items.length
      };
    }

    return state;
  }
}
```

## 📋 Setup Checklist

1. ✅ Copy templates above
2. ✅ Replace `MyNode` with your node name throughout
3. ✅ Update `package.json` with your package name
4. ✅ Implement your service logic
5. ✅ Define your config schema
6. ✅ Test with `npm run build`
7. ✅ Publish with `npm publish`

## 🔗 Real Examples

**Study these published packages for complete implementations:**
- **PromiseNode**: `@gravityai-dev/aws-bedrock` (BedrockClaude executor)
- **CallbackNode**: `@gravityai-dev/ingest` (ApifyResults executor)

---

**Next**: [Node Types](./02-node-types.md) - Choose PromiseNode vs CallbackNode
