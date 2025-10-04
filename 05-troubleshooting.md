# Troubleshooting

**Common issues and solutions for GravityAI plugin node development**

## 🚨 Critical Errors

### "Node X is not a PromiseNode but was executed as one"

**Cause**: Incorrect base class import

**✅ Correct Pattern:**
```typescript
// For PromiseNode:
import { PromiseNode, type NodeExecutionContext } from "@gravityai-dev/plugin-base";

export default class MyExecutor extends PromiseNode {
  constructor() {
    super("MyNode");
  }
}

// For CallbackNode:
import { getPlatformDependencies } from "@gravityai-dev/plugin-base";
const { CallbackNode } = getPlatformDependencies();

export default class MyCallbackExecutor extends CallbackNode {
  constructor() {
    super("MyNode");
  }
}
```

**Why**: The workflow system validates node types. PromiseNode can be imported directly, but CallbackNode still needs `getPlatformDependencies()`.

### "Cannot find name 'PromiseNode'"

**Cause**: Missing import

**✅ Correct:**
```typescript
import { PromiseNode, type NodeExecutionContext } from "@gravityai-dev/plugin-base";

export default class MyExecutor extends PromiseNode {
  constructor() {
    super("MyNode");
  }
}
```

### "Startup Freeze" / Plugin Loading Hangs

**Cause**: Module-level calls to platform dependencies

**✅ Correct - Use Direct Imports:**
```typescript
import { PromiseNode, type NodeExecutionContext } from "@gravityai-dev/plugin-base";
import { myService } from "./service";

export default class MyExecutor extends PromiseNode {
  // No module-level getPlatformDependencies() needed
}
```

**Note**: Only CallbackNode still needs `getPlatformDependencies()` at module level.

## 🔧 Build & Import Errors

### "Module cannot have multiple default exports"

**Cause**: Duplicate default exports in executor file

**❌ Wrong:**
```typescript
export default class MyExecutor extends PromiseNode<Config> {
  // ...
}

// Later in same file
export default MyExecutor; // Duplicate!
```

**✅ Correct:**
```typescript
export default class MyExecutor extends PromiseNode<Config> {
  // ...
}

// OR use named export
export { MyExecutor };
```

### "Cannot find module '@gravityai-dev/plugin-base'"

**Cause**: Missing dependency in package.json

**Solution**: Add to package.json:
```json
{
  "dependencies": {
    "@gravityai-dev/plugin-base": "^1.0.0"
  }
}
```

### TypeScript Compilation Errors

**Common Issues:**
- Missing type imports
- Incorrect generic types
- Missing return types

**✅ Correct Types:**
```typescript
import { 
  PromiseNode,
  type NodeExecutionContext, 
  type ValidationResult 
} from "@gravityai-dev/plugin-base";

export default class MyExecutor extends PromiseNode {
  protected async validateConfig(config: MyConfig): Promise<ValidationResult> {
    return { success: true };
  }

  protected async executeNode(
    inputs: Record<string, any>,
    config: MyConfig,
    context: NodeExecutionContext
  ): Promise<MyOutput> {
    // Use injected API
    const logger = context.api?.createLogger?.(this.nodeType) || console;
    const result = await myService(config, credentialContext, context.api);
    return { __outputs: result };
  }
}
```

## 🔐 Credential Issues

### "Credentials are required" Error

**Cause**: Node definition doesn't declare credential requirements

**✅ Solution:**
```typescript
export const MyNode: EnhancedNodeDefinition = {
  type: "MyNode",
  // ... other properties
  credentials: [
    {
      name: "myCredential",
      type: "myCredentialType",
      required: true
    }
  ],
};
```

### "Credential not found" Error

**Cause**: Node config doesn't have credential ID stored

**Check Config Structure:**
```json
{
  "credentials": {
    "myCredential": "cred_xxxxx"
  },
  "otherConfig": "value"
}
```

**Debug Steps:**
1. Check if credential exists in database
2. Verify credential ID in node config
3. Ensure credential type matches node definition

### Service Can't Access Credentials

**❌ Wrong Pattern:**
```typescript
// Don't access credentials directly
const apiKey = context.credentials.myCredential.apiKey;
```

**✅ Correct Pattern:**
```typescript
// Service fetches credentials from injected API
export async function myService(config: any, credentialContext: CredentialContext, api: any) {
  const credentials = await api.getNodeCredentials(credentialContext, 'myCredential');
  const apiKey = credentials.apiKey;
  // Use apiKey...
}
```

## 🔄 Runtime Issues

### Node Doesn't Execute

**Check List:**
1. ✅ Node is registered in plugin setup
2. ✅ Node definition is exported correctly
3. ✅ Executor extends correct base class
4. ✅ `executeNode()` method is implemented
5. ✅ No errors in plugin loading

### CallbackNode Doesn't Continue

**Common Issues:**
- Missing `isComplete` in state
- Not handling "continue" signals properly
- State not updating correctly
- Wrong import pattern for CallbackNode

**✅ Correct CallbackNode Pattern:**
```typescript
import { getPlatformDependencies } from "@gravityai-dev/plugin-base";

const { CallbackNode } = getPlatformDependencies();

initializeState(inputs: any): MyState {
  return {
    items: [], // Start empty, get from config
    currentIndex: 0,
    isComplete: false, // Important!
  };
}

async handleEvent(event, state, emit) {
  const { inputs, config } = event;
  
  // Handle continue signal to advance iteration
  if (inputs?.continue !== undefined && state.items.length > 0) {
    if (state.currentIndex >= state.items.length) {
      return { ...state, isComplete: true };
    }
    
    const item = state.items[state.currentIndex];
    const result = await processItem(item);
    
    emit({ 
      __outputs: {
        item: result,
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
```

### Logger Not Working

**✅ Correct Pattern:**
```typescript
// In executor, use this.logger
this.logger.info("Processing started");

// Or get from context.api
const logger = context.api?.createLogger?.(this.nodeType) || console;

// In service, get from injected API
export async function myService(config: any, credentialContext: any, api: any) {
  const logger = api?.createLogger?.("MyService") || console;
  logger.info("Service called");
}
```

## 🧪 Testing & Debugging

### Test Your Node

**1. Build Test:**
```bash
npm run build
```

**2. Plugin Loading Test:**
```bash
# Start server and check logs for plugin loading
npm start
```

**3. Debug Execution:**
```bash
# Use debug resolver to test node
curl -X POST http://localhost:4000/api/debug/execute-node \
  -H "Content-Type: application/json" \
  -d '{
    "nodeType": "MyNode",
    "config": { "test": "value" },
    "inputs": { "input": "test data" }
  }'
```

### Debug Logging

**Add Debug Logs:**
```typescript
protected async executeNode(inputs, config, context) {
  this.logger.info("Node execution started", { 
    nodeId: context.nodeId,
    config: config 
  });
  
  try {
    const result = await myService(config, credentialContext);
    this.logger.info("Node execution completed", { result });
    return { __outputs: result };
  } catch (error) {
    this.logger.error("Node execution failed", { error: error.message });
    throw error;
  }
}
```

### Common Debug Steps

1. **Check Plugin Registration:**
   - Look for plugin loading logs in server startup
   - Verify node appears in available nodes list

2. **Check Credential Loading:**
   - Verify credentials exist in database
   - Check credential IDs in node config
   - Test credential fetching in service

3. **Check Execution Flow:**
   - Add logging to executor methods
   - Verify service calls work independently
   - Check output structure matches expected format

## 🔗 Getting Help

### Study Working Examples

**PromiseNode Issues:**
- Compare with `@gravityai-dev/aws-bedrock`
- Check `@gravityai-dev/openai` implementation

**CallbackNode Issues:**
- Compare with `@gravityai-dev/ingest`
- Check `@gravityai-dev/flow` implementation

**Credential Issues:**
- Study any working package's credential handling
- Check credential definitions in published packages

### Error Message Patterns

| Error Message | Likely Cause | Solution |
|---------------|--------------|----------|
| "is not a PromiseNode" | Wrong import | Import PromiseNode directly |
| "Cannot find name" | Missing import | Add import statement |
| "Startup freeze" | Module-level deps call | Use direct imports |
| "Credentials are required" | Missing credential definition | Add to node definition |
| "Credential not found" | Missing credential ID | Check node config |
| "Multiple default exports" | Duplicate exports | Remove duplicate |

---

**Previous**: [Credential Management](./04-credentials.md) - Authentication pattern
