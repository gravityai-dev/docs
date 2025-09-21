# Service Connectors & MCP Integration

**Connect nodes to share capabilities through service connectors**

## 🔌 What are Service Connectors?

Service connectors allow nodes to provide and consume services from other nodes in a workflow. This enables:
- Dynamic capability discovery
- Service composition
- Tool integration (like MCP)
- Reusable functionality

## 🎯 Quick Guide

| Connector Type | Purpose | Example |
|----------------|---------|---------|
| **Embedding Service** | Generate text embeddings | OpenAI → PostgresFetch |
| **MCP Service** | Dynamic tool discovery | PostgresFetch → Nova |
| **Custom Service** | Any node-specific service | Your custom services |

## 📋 Service Connector Definition

Add service connectors to your node definition:

```typescript
export const definition = {
  type: "YourNode",
  // ... other properties ...
  
  serviceConnectors: [
    {
      name: "embeddingService",
      description: "Embedding service connection",
      serviceType: "embedding",
      methods: ["createEmbedding"],
      isService: false, // This node CONSUMES services
    },
    {
      name: "mcpService", 
      description: "MCP service for tool discovery",
      serviceType: "mcp",
      isService: true, // This node PROVIDES services
    }
  ]
};
```

## 🔄 Service Flow

### 1. Service Provider (isService: true)
```typescript
// In your executor's handleServiceCall method
async handleServiceCall(method: string, params: any, context: any) {
  switch (method) {
    case 'getSchema':
      return this.getMCPSchema();
    
    case 'yourMethod':
      return this.executeMethod(params);
      
    default:
      throw new Error(`Unknown method: ${method}`);
  }
}
```

### 2. Service Consumer (isService: false)
```typescript
// In your executor
import { callService } from "@gravityai-dev/plugin-base";

const result = await callService("methodName", params, context);
```

## 🤖 MCP (Model Context Protocol) Integration

MCP enables dynamic tool discovery for AI models. Here's how it works:

### 1. MCP Service Provider

```typescript
// PostgresFetch as MCP provider
export const PostgresVectorSearchSchema = {
  name: "PostgresVectorSearch",
  version: "1.0.0",
  methods: {
    getChunksByQuery: {
      input: {
        type: "object",
        properties: {
          query: { type: "string", description: "Search query" },
          topK: { type: "integer", default: 10 }
        },
        required: ["query"]
      },
      output: {
        type: "object",
        properties: {
          results: { type: "array" },
          count: { type: "integer" }
        }
      }
    }
  }
};

// In executor
async handleServiceCall(method: string, params: any) {
  if (method === 'getSchema') {
    return PostgresVectorSearchSchema;
  }
  // Handle actual method calls
}
```

### 2. MCP Service Consumer (e.g., Nova)

```typescript
// Nova automatically discovers and uses MCP tools
serviceConnectors: [
  {
    name: "mcpService",
    description: "MCP service connector",
    serviceType: "mcp",
    isService: false, // Consumes MCP services
  }
]

// The workflow system automatically:
// 1. Calls getSchema on connected MCP services
// 2. Converts schemas to tool format
// 3. Injects tools into the AI model configuration
```

## 🔗 Connecting Services in Workflows

In the workflow editor:
1. Add nodes with compatible service connectors
2. Connect service outputs to service inputs:
   - Drag from provider's `service` handle
   - Drop on consumer's `serviceConsumer` handle
3. Services are automatically discovered and available

## 📊 Service Types

### Standard Service Types

| Type | Methods | Purpose |
|------|---------|---------|
| `embedding` | `createEmbedding` | Text to vector conversion |
| `mcp` | `getSchema`, dynamic methods | Tool discovery for AI |
| `storage` | `store`, `retrieve` | Data persistence |
| `search` | `search`, `index` | Search operations |

### Custom Service Types

You can define your own service types:

```typescript
serviceConnectors: [
  {
    name: "customService",
    serviceType: "myCustomType",
    methods: ["method1", "method2"],
    isService: true
  }
]
```

## 🚀 Real Examples

### Example 1: Embedding Service Chain
```
OpenAI (provides embedding) → PostgresFetch (consumes embedding)
```

OpenAI provides:
```typescript
{
  name: "embeddingService",
  serviceType: "embedding", 
  methods: ["createEmbedding"],
  isService: true
}
```

PostgresFetch consumes:
```typescript
{
  name: "embeddingService",
  serviceType: "embedding",
  methods: ["createEmbedding"], 
  isService: false
}
```

### Example 2: MCP Tool Integration
```
PostgresFetch (provides MCP) → Nova (consumes MCP)
```

PostgresFetch provides vector search tools:
```typescript
{
  name: "vectorSearchService",
  serviceType: "mcp",
  isService: true
}
```

Nova automatically discovers and uses tools:
```typescript
{
  name: "mcpService",
  serviceType: "mcp",
  isService: false
}
```

## 🎯 Best Practices

1. **Clear Service Contracts**: Define clear input/output schemas
2. **Error Handling**: Always handle service call failures gracefully
3. **Service Discovery**: Use `getSchema` for dynamic discovery
4. **Type Safety**: Use TypeScript interfaces for service methods
5. **Documentation**: Document your service methods clearly

## 🚨 Common Patterns

### Pattern 1: Service Proxy
```typescript
// Automatically created by the workflow system
const services = {
  embeddingService: {
    createEmbedding: (text) => callService("createEmbedding", { text }, context)
  }
};
```

### Pattern 2: Schema-based Discovery
```typescript
// MCP services return their capabilities
const schema = await callService("getSchema", {}, context);
const availableMethods = Object.keys(schema.methods);
```

### Pattern 3: Service Chaining
```typescript
// Service A → Service B → Service C
const embedding = await callService("createEmbedding", { text }, context);
const results = await callService("search", { embedding }, context);
const enhanced = await callService("enhance", { results }, context);
```

## 🔍 Debugging Service Connections

1. **Check Service Registration**: Look for `[ServiceRegistry] Loaded X services` in logs
2. **Verify Connections**: Ensure service edges exist in workflow
3. **Test Schema**: Call `getSchema` to verify MCP services
4. **Monitor Calls**: Watch for `Service call: methodName` in logs

---

**Next**: [Config Schema Reference](./06-config-schema.md) - UI configuration guide
