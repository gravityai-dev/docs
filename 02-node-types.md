# Node Types: PromiseNode vs CallbackNode

**Choose the right base class for your node**

## 🎯 Quick Decision Guide

| Use Case | Node Type | Example |
|----------|-----------|---------|
| API call that returns once | **PromiseNode** | OpenAI completion, AWS Bedrock |
| Data transformation | **PromiseNode** | Text processing, format conversion |
| File operations | **PromiseNode** | Upload file, read document |
| Database operations | **PromiseNode** | Insert record, query data |
| Streaming responses | **CallbackNode** | Real-time data, chat streaming |
| Processing collections | **CallbackNode** | Loop through items, batch processing |
| User interactions | **CallbackNode** | Wait for input, multi-step workflows |
| Long-running tasks | **CallbackNode** | Data scraping, bulk operations |

## 🔄 PromiseNode - Single Execution

**Pattern**: Execute once → Return result → Done

### When to Use
- ✅ Single input → Single output
- ✅ Stateless operations
- ✅ API calls that complete immediately
- ✅ Data transformations
- ✅ Simple workflows

### Key Characteristics
- Extends `PromiseNode<ConfigType>`
- Implements `executeNode()` method
- Returns `Promise<OutputType>`
- No state management needed
- Workflow continues immediately after completion

### Method Signature
```typescript
protected async executeNode(
  inputs: Record<string, any>,
  config: ConfigType,
  context: NodeExecutionContext
): Promise<OutputType>
```

### Real Examples
- **`@gravityai-dev/aws-bedrock`** - BedrockClaude executor
- **`@gravityai-dev/openai`** - OpenAI completion
- **`@gravityai-dev/aws-s3`** - S3 file operations

## 🔄 CallbackNode - Multiple Outputs

**Pattern**: Initialize → HandleEvent → Emit → HandleEvent → Emit → Complete

### When to Use
- ✅ Multiple outputs over time
- ✅ Stateful operations
- ✅ Processing collections/arrays one by one
- ✅ Waiting for continuation signals
- ✅ Iterative workflows (like Loop node)

### Key Characteristics
- Extends `CallbackNode<ConfigType, StateType>`
- Implements `initializeState()` and `handleEvent()` methods
- Maintains state between events
- Can emit multiple outputs via `emit()` function
- Workflow waits for continuation signals (like "continue" input)

### Method Signatures
```typescript
// Initialize the node's state
initializeState(inputs: any): StateType & { isComplete?: boolean }

// Handle events and emit outputs
async handleEvent(
  event: { type: string; inputs?: any; config?: any },
  state: StateType,
  emit: (output: any) => void
): Promise<StateType>
```

### Real Examples
- **`@gravityai-dev/ingest`** - ApifyResults executor (processes items one by one)
- **`@gravityai-dev/flow`** - Loop executor (iterates through collections)

## 🔍 Detailed Comparison

### PromiseNode Execution Flow
1. Node receives inputs
2. `executeNode()` is called once
3. Returns final result
4. Workflow continues to next node

### CallbackNode Execution Flow
1. Node receives inputs
2. `initializeState()` creates initial state
3. `handleEvent()` processes first event
4. Emits output, updates state
5. Waits for continuation signal
6. `handleEvent()` processes next event
7. Repeats until `state.isComplete = true`

## 🚨 Common Mistakes

### ❌ Wrong Choice Examples

**Using PromiseNode for streaming:**
```typescript
// WRONG - PromiseNode can't emit multiple outputs
class StreamingNode extends PromiseNode<Config> {
  async executeNode() {
    // Can only return once!
    return { text: "final result" };
  }
}
```

**Using CallbackNode for simple operations:**
```typescript
// WRONG - Unnecessary complexity for single operation
class SimpleAPINode extends CallbackNode<Config, State> {
  initializeState() {
    return { isComplete: false };
  }
  
  async handleEvent(event, state, emit) {
    const result = await apiCall();
    emit({ __outputs: result });
    return { ...state, isComplete: true };
  }
}
```

### ✅ Correct Patterns

**PromiseNode for API calls:**
```typescript
class APINode extends PromiseNode<Config> {
  async executeNode(inputs, config, context) {
    const result = await apiCall(config);
    return { __outputs: result };
  }
}
```

**CallbackNode for processing collections:**
```typescript
class ProcessorNode extends CallbackNode<Config, State> {
  initializeState(inputs) {
    return {
      items: [],
      currentIndex: 0,
      isComplete: false,
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
      return {
        ...state,
        items: config.items
      };
    }
    
    return state;
  }
}
```

## 🎯 Decision Framework

Ask yourself:

1. **How many outputs?**
   - One output → PromiseNode
   - Multiple outputs → CallbackNode

2. **Do I need to wait for user input?**
   - No → PromiseNode
   - Yes → CallbackNode

3. **Am I processing a collection?**
   - No → PromiseNode
   - Yes → CallbackNode

4. **Is this a streaming operation?**
   - No → PromiseNode
   - Yes → CallbackNode

5. **Do I need to maintain state between operations?**
   - No → PromiseNode
   - Yes → CallbackNode

## 🔗 Study Real Examples

**PromiseNode Examples:**
- Browse `@gravityai-dev/aws-bedrock/src/BedrockClaude/node/executor.ts`
- Browse `@gravityai-dev/openai/src/OpenAI/node/executor.ts`

**CallbackNode Examples:**
- Browse `@gravityai-dev/ingest/src/ApifyResults/node/executor.ts`
- Browse `@gravityai-dev/flow/src/Loop/node/executor.ts`

---

**Next**: [Implementation Patterns](./03-patterns.md) - Core patterns and architecture
