# GravityAI Plugin Node Development Guide

**Create powerful AI workflow nodes with GravityAI's plugin system**

## 🚀 Quick Navigation

- **[Quick Start](./01-quick-start.md)** - Essential templates and 5-minute setup
- **[Node Types](./02-node-types.md)** - PromiseNode vs CallbackNode decision guide  
- **[Implementation Patterns](./03-patterns.md)** - Core patterns and architecture
- **[Credential Management](./04-credentials.md)** - Authentication pattern (the one rule)
- **[Troubleshooting](./05-troubleshooting.md)** - Common issues and solutions

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

## 🎯 For LLMs

When creating nodes:
1. **Use the templates** from [Quick Start](./01-quick-start.md)
2. **Follow the credential pattern** from [Credential Management](./04-credentials.md)
3. **Choose the right base class** from [Node Types](./02-node-types.md)
4. **Reference published examples** above for real implementations

---

**Next**: Start with [Quick Start](./01-quick-start.md) for essential templates
