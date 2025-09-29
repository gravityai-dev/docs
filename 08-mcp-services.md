# MCP Services Integration

## Overview

MCP (Model Context Protocol) services enable AI models like AWS Nova to interact with workflow nodes through a standardized tool interface. MCP nodes can either be pure data services or workflow nodes that trigger downstream execution.

## MCP Service Types

### 1. Pure MCP Data Services

These services provide data to AI models without affecting workflow execution.

#### Characteristics
- Stateless data retrieval
- Direct execution and immediate response
- No workflow side effects
- No edge triggering

#### Example: PostgresFetch Data Service
```typescript
// PostgresFetch provides data services via MCP
async handleServiceCall(method: string, params: any) {
  switch(method) {
    case "getChunksByQuery":
      // Direct database query and return
      const results = await MCPGetKnowledgeService.execute(params);
      return results; // Returns data directly to Nova
      
    case "getSchema":
      return MCPSchema; // Returns schema for tool discovery
  }
}
```

#### Use Case
Nova needs to retrieve credit card information from the knowledge base:
1. Nova calls `getChunksByQuery` tool
2. PostgresFetch queries database
3. Results returned directly to Nova
4. Nova uses information in conversation

### 2. MCP Workflow Nodes

These are workflow nodes that expose MCP interfaces, allowing AI models to trigger workflow execution and downstream nodes.

#### Characteristics
- Part of the workflow graph
- Have edges to downstream nodes
- Trigger workflow routing via NODE_OUTPUT events
- Return simplified responses to MCP callers

#### Example: MCPgetNeeds Workflow Node
```typescript
class MCPgetNeedsExecutor extends PromiseNode {
  // Standard node execution
  async executeNode(inputs, config, context) {
    const result = await IdentifyNeedsService.execute(inputs);
    
    // Returns __outputs for workflow routing
    return {
      __outputs: {
        needs: result.needs,
        count: result.count,
        found: result.count > 0  // Triggers conditional edges
      }
    };
  }
  
  // MCP service interface
  async handleServiceCall(method: string, params: any, context) {
    case "identifyNeeds":
      // Use workflow execution utility to trigger routing
      const { executeNodeWithRouting } = await import("NodeExecutionUtils");
      
      // Execute node and emit NODE_OUTPUT for routing
      const result = await executeNodeWithRouting(
        this.executeNode.bind(this),
        params,
        config,
        context
      );
      
      // Return simple response to MCP caller (not full data)
      return {
        success: true,
        message: `Found ${result.__outputs?.count || 0} needs`,
        triggered: true
      };
  }
}
```

#### Use Case
Nova identifies customer needs and triggers follow-up actions:
1. Nova calls `identifyNeeds` tool
2. MCPgetNeeds executes and finds needs
3. NODE_OUTPUT event triggers routing to next node
4. Code node receives full needs data and processes it
5. Nova receives simple success message

## Integration Solution

### The Challenge
MCP workflow nodes need to trigger downstream nodes while executing within the current workflow instance.

### Key Issues Solved

#### 1. Graph Reachability
**Problem**: Nodes downstream from MCP service providers were being filtered out as "unreachable" during workflow loading.

**Root Cause**: The GraphTraversal only considered nodes reachable from traditional trigger nodes (InputTrigger, etc.), not from MCP service nodes.

**Solution**: Modified GraphTraversal to include MCP service providers as starting points:
```typescript
// Check if node provides MCP services
const providesMCPService = serviceConnectors.some(
  (connector) => connector.serviceType === "mcp" && connector.isService === true
);
if (providesMCPService) {
  // Include as starting point for reachability
  reachable.add(node.id);
}
```

#### 2. Workflow Routing
**Problem**: MCP nodes executing via service calls weren't triggering downstream nodes.

**Solution**: Use `executeNodeWithRouting` utility that:
1. Executes the node
2. Emits NODE_OUTPUT event for routing
3. Returns simplified response to caller

### Complete Flow
```
Nova calls MCP tool → handleServiceCall
                           ↓
                    executeNodeWithRouting
                           ↓
                  ┌────────┴────────┐
                  ↓                 ↓
          executeNode        (returns to Nova)
                  ↓           simple response
           __outputs
                  ↓
         NODE_OUTPUT event
                  ↓
     State machine routes
                  ↓
     Next nodes execute
```

## Service Connector Configuration

### MCP Service Provider
Nodes that provide MCP services must declare their service connector:

```typescript
serviceConnectors: [
  {
    name: "mcpService",
    type: "service",
    serviceType: "mcp",
    isService: true,  // This node provides services
    methods: ["getSchema", "identifyNeeds"]
  }
]
```

### MCP Service Consumer
Nodes that consume MCP services (like Nova):

```typescript
serviceConnectors: [
  {
    name: "mcpService",
    type: "service", 
    serviceType: "mcp",
    isService: false,  // This node consumes services
    methods: ["getSchema"]
  }
]
```

## Schema Definition

MCP services must provide a schema for tool discovery:

```typescript
export const MCPgetNeedsSchema = {
  name: "identifyNeeds",
  description: "Identify customer needs from conversation",
  methods: {
    identifyNeeds: {
      description: "Analyze text to identify customer needs",
      input: {
        type: "object",
        properties: {
          query: {
            type: "string",
            description: "The text to analyze"
          }
        },
        required: ["query"]
      },
      output: {
        type: "object",
        properties: {
          needs: {
            type: "array",
            description: "Identified customer needs"
          },
          count: {
            type: "number"
          },
          found: {
            type: "boolean"
          }
        }
      }
    }
  }
};
```

## Best Practices

### 1. Service Type Selection
- Use **Pure MCP Services** for data retrieval, queries, calculations
- Use **MCP Workflow Nodes** when the action should trigger downstream nodes

### 2. Return Values
- **Internal (executeNode)**: Returns `{ __outputs: {...} }` for workflow routing
- **External (handleServiceCall)**: Returns simple status/summary for MCP caller

### 3. Error Handling
```typescript
async handleServiceCall(method: string, params: any) {
  try {
    // Handle service call
  } catch (error) {
    this.logger.error(`MCP service failed: ${method}`, error);
    // Return error in MCP format
    return {
      error: error.message,
      code: "SERVICE_ERROR"
    };
  }
}
```

### 4. Schema Validation
Always validate inputs against the schema:
```typescript
if (!params.query) {
  throw new Error("Query parameter is required");
}
```

## Integration with Nova

Nova discovers and uses MCP tools through:

1. **Schema Discovery**: Nova calls `getSchema` on connected MCP services
2. **Tool Registration**: Nova converts MCP methods to tool specifications
3. **Tool Invocation**: Nova calls tools during conversation
4. **Result Processing**: Nova receives and uses the results

Example Nova tool configuration:
```typescript
const mcpTools = Object.entries(mcpSchema.methods).map(([methodName, methodSchema]) => ({
  toolSpec: {
    name: methodName,
    description: methodSchema.description,
    inputSchema: {
      json: JSON.stringify(methodSchema.input)
    }
  }
}));
```

## Key Implementation Details

### executeNodeWithRouting Utility
A universal utility for executing nodes outside normal workflow flow:
```typescript
// From workflow/utils/NodeExecutionUtils.ts
export async function executeNodeWithRouting(
  executeNode: Function,
  params: any,
  config: any,
  context: NodeExecutionContext
): Promise<any> {
  // Execute the node
  const result = await executeNode(params, config, context);
  
  // Emit NODE_OUTPUT to trigger routing
  const actor = engine.getActor(context.executionId);
  actor.send({
    type: "NODE_OUTPUT",
    nodeId: context.nodeId,
    output: result
  });
  
  return result;
}
```

This utility is not MCP-specific and can be used by any node that needs to trigger workflow routing when called outside the normal flow.

## Troubleshooting

### Next Nodes Not Triggering
**Problem**: MCP workflow node executes but next nodes don't trigger
**Solution**: Ensure the node executes through the state machine, not directly

### Empty Results
**Problem**: MCP service returns empty results
**Solution**: Check workflow_id filtering and database queries

### Schema Not Found
**Problem**: Nova can't discover MCP tools
**Solution**: Verify service connector configuration and schema export

## Related Documentation

- [Service Connectors](./07-service-connectors.md)
- [Workflow Architecture](/services/gravity-workflow/ARCHITECTURE.md)
- [Nova Integration](/services/gravity-services/packages/aws-nova/README.md)
