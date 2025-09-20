# GravityAI Plugin Node Development Guide

**Create powerful AI workflow nodes with GravityAI's plugin system**

## 📚 Quick Navigation

1. **[Quick Start](./01-quick-start.md)** - Templates and setup (Pattern A)
2. **[Node Types](./02-node-types.md)** - PromiseNode vs CallbackNode (FIXED)
3. **[Implementation Patterns](./03-patterns.md)** - Critical patterns (Pattern A)
4. **[Credential Management](./04-credentials.md)** - The one rule
5. **[Troubleshooting](./05-troubleshooting.md)** - Common issues (FIXED) and solutions

## 🏗️ Architecture Overview

GravityAI uses a **plugin-based architecture** where nodes are distributed as npm packages:

```
@gravityai-dev/your-node/
├── src/
│   ├── index.ts              # Plugin definition
│   ├── YourNode/
│   │   ├── node/             # Node definition & executor
│   │   ├── service/          # Business logic & API calls
│   │   └── util/             # Types & utilities
│   ├── shared/
│   │   └── platform.ts       # Platform dependencies
│   └── credentials/          # Credential definitions
└── package.json
```

## 🎯 Key Principles

1. **Plugin Pattern**: Nodes are npm packages that register themselves
2. **Separation of Concerns**: Executors handle workflow logic, services handle business logic
3. **Credential Security**: Services fetch credentials internally, never exposed to nodes
4. **Type Safety**: Full TypeScript support with proper interfaces

## 📦 Real Examples

Instead of code samples, reference these published packages:

### PromiseNode Examples (Single execution)
- **`@gravityai-dev/aws-bedrock`** - BedrockClaude executor ([GitHub](https://github.com/gravityai-dev/aws-bedrock))
- **`@gravityai-dev/openai`** - OpenAI executor ([GitHub](https://github.com/gravityai-dev/openai))
- **`@gravityai-dev/aws-s3`** - S3 operations ([GitHub](https://github.com/gravityai-dev/aws-s3))

### CallbackNode Examples (Multiple outputs)
- **`@gravityai-dev/ingest`** - ApifyResults executor ([GitHub](https://github.com/gravityai-dev/ingest))
- **`@gravityai-dev/flow`** - Loop executor ([GitHub](https://github.com/gravityai-dev/flow))

### Credential Patterns
- **AWS Services**: `@gravityai-dev/aws-bedrock`, `@gravityai-dev/aws-s3`
- **API Keys**: `@gravityai-dev/openai`, `@gravityai-dev/pinecone`

## 🚨 Critical Pattern (Must Follow)

```typescript
// ✅ CORRECT: Plugin executor pattern
import { getPlatformDependencies, type NodeExecutionContext } from "@gravityai-dev/plugin-base";

const { PromiseNode } = getPlatformDependencies();

export default class MyNodeExecutor extends PromiseNode<MyConfig> {
  constructor() {
    super("MyNode");
  }

  protected async executeNode(
    inputs: Record<string, any>,
    config: MyConfig,
    context: NodeExecutionContext
  ): Promise<MyOutput> {
    const credentialContext = this.buildCredentialContext(context);
    const result = await myService(config, credentialContext);
    return { __outputs: result };
  }
}
```

## 🔄 CallbackNode Example (CORRECTED)
```typescript
// Import from shared/platform for CallbackNode
import { CallbackNode } from "../../shared/platform";

class LoopExecutor extends CallbackNode<LoopConfig, LoopState> {
  initializeState(inputs: any): LoopState {
    return { items: [], currentIndex: 0, isComplete: false };
  }
  
  async handleEvent(event, state, emit) {
    const { inputs, config } = event;
    
    // Handle continue signal to advance iteration
    if (inputs?.continue !== undefined && state.items.length > 0) {
      if (state.currentIndex >= state.items.length) {
        return { ...state, isComplete: true };
      }
      
      const item = state.items[state.currentIndex];
      emit({ 
        __outputs: { 
          item, 
          index: state.currentIndex,
          hasMore: state.currentIndex < state.items.length - 1
        } 
      });
      
      return {
        ...state,
        currentIndex: state.currentIndex + 1,
        isComplete: state.currentIndex + 1 >= state.items.length
      };
    }
    
    // Initialize with items from config
    if (state.items.length === 0 && config?.items) {
      return { ...state, items: config.items };
    }
    
    return state;
  }
}
```

## 🎯 For LLMs

When creating nodes:
1. **Use the templates** from [Quick Start](./01-quick-start.md)
2. **Follow the credential pattern** from [Credential Management](./04-credentials.md)
3. **Choose the right base class** from [Node Types](./02-node-types.md)
4. **Reference published examples** above for real implementations

---

**Next**: Start with [Quick Start](./01-quick-start.md) for essential templates
