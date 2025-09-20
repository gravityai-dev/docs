# Config Schema Reference

**Complete guide to configSchema options for node configuration UI**

## 🎯 Overview

The `configSchema` defines the configuration UI for your node. It uses JSON Schema with custom UI extensions to create forms that users fill out when configuring nodes in workflows.

## 📋 Basic Structure

```typescript
configSchema: {
  type: "object",
  properties: {
    fieldName: {
      type: "string|number|boolean|object|array",
      title: "Display Name",
      description: "Help text for users",
      default: "default value",
      // UI-specific options
      "ui:field": "template|code",
      "ui:widget": "toggle|select",
      "ui:dependencies": { otherField: true }
    }
  },
  required: ["fieldName"]
}
```

## 🔤 Field Types

### String Fields

#### Basic String
```typescript
apiEndpoint: {
  type: "string",
  title: "API Endpoint",
  description: "The API endpoint URL",
  default: "https://api.example.com"
}
```

#### Template String (Handlebars)
```typescript
prompt: {
  type: "string",
  title: "Prompt",
  description: "User message/prompt. Supports template syntax like {{input.fieldName}} to reference input data.",
  default: "",
  "ui:field": "template"
}
```
- **Template Syntax**: `{{input.fieldName}}`, `{{signal.data}}`, `{{workflow.variables.userId}}`
- **Resolves to**: String value after handlebars processing

#### Enum/Dropdown String
```typescript
model: {
  type: "string",
  title: "Model",
  description: "Select the Claude model to use",
  enum: [
    "us.anthropic.claude-sonnet-4-20250514-v1:0",
    "us.anthropic.claude-3-5-sonnet-20241022-v2:0",
    "us.anthropic.claude-3-5-haiku-20241022-v1:0"
  ],
  enumNames: ["Claude Sonnet 4 (Latest)", "Claude 3.5 Sonnet", "Claude 3.5 Haiku"],
  default: "us.anthropic.claude-3-5-sonnet-20241022-v2:0"
}
```

### Number Fields

#### Basic Number
```typescript
maxTokens: {
  type: "number",
  title: "Max Tokens",
  description: "Maximum number of tokens to generate",
  default: 256,
  minimum: 1,
  maximum: 4096
}
```

#### Number with Step
```typescript
temperature: {
  type: "number",
  title: "Temperature",
  description: "Controls randomness (0-1)",
  default: 0.7,
  minimum: 0,
  maximum: 1,
  step: 0.1
}
```

### Boolean Fields

#### Toggle Widget
```typescript
includeImageUrl: {
  type: "boolean",
  title: "Include Image URL",
  description: "Enable image analysis by providing an image URL",
  default: false,
  "ui:widget": "toggle"
}
```

#### Checkbox (Default)
```typescript
generateIds: {
  type: "boolean",
  title: "Generate IDs",
  description: "Generate content IDs",
  default: false
}
```

### Object Fields

#### Template Object (JavaScript)
```typescript
code: {
  type: "object",
  title: "Code",
  description: "JS Code to transform data",
  default: "",
  "ui:field": "template"
}
```
- **Template Syntax**: JavaScript return statement
- **Example**: `return { result: input.data.map(x => x.value) }`
- **Resolves to**: JavaScript object/value after execution

#### History Object
```typescript
history: {
  type: "object",
  title: "History",
  description: "Message history [] for context",
  default: "",
  "ui:field": "template"
}
```

## 🎨 UI Extensions

### Template Fields

#### String Templates (Handlebars)
```typescript
"ui:field": "template"
// With type: "string"
// Syntax: {{input.fieldName}}, {{signal.data}}
// Result: Resolved string value
```

#### Object Templates (JavaScript)
```typescript
"ui:field": "template"
// With type: "object"  
// Syntax: return { key: input.value }
// Result: Executed JavaScript result
```

### Widgets

#### Toggle Switch
```typescript
"ui:widget": "toggle"
// Creates a toggle switch instead of checkbox
// Only for boolean fields
```

### Conditional Dependencies

#### Show Field Based on Another Field
```typescript
imageUrl: {
  type: "string",
  title: "Image URL",
  description: "URL of the image to analyze",
  default: "",
  "ui:field": "template",
  "ui:dependencies": {
    includeImageUrl: true  // Only show when includeImageUrl is true
  }
}
```

#### Multiple Dependencies
```typescript
advancedOptions: {
  type: "object",
  title: "Advanced Options",
  "ui:dependencies": {
    enableAdvanced: true,
    mode: "custom"  // Show when enableAdvanced=true AND mode="custom"
  }
}
```

## 📚 Real Examples from Working Nodes

### AWS Bedrock Claude
```typescript
configSchema: {
  type: "object",
  properties: {
    model: {
      type: "string",
      title: "Model",
      enum: [
        "us.anthropic.claude-sonnet-4-20250514-v1:0",
        "us.anthropic.claude-3-5-sonnet-20241022-v2:0"
      ],
      enumNames: ["Claude Sonnet 4", "Claude 3.5 Sonnet"],
      default: "us.anthropic.claude-3-5-sonnet-20241022-v2:0"
    },
    maxTokens: {
      type: "number",
      title: "Max Tokens",
      default: 256,
      minimum: 1,
      maximum: 4096
    },
    temperature: {
      type: "number",
      title: "Temperature",
      default: 0.7,
      minimum: 0,
      maximum: 1,
      step: 0.1
    },
    systemPrompt: {
      type: "string",
      title: "System Prompt",
      description: "System message. Supports {{input.fieldName}} syntax.",
      default: "",
      "ui:field": "template"
    },
    prompt: {
      type: "string", 
      title: "Prompt",
      description: "User message. Supports {{input.fieldName}} syntax.",
      default: "",
      "ui:field": "template"
    },
    includeImageUrl: {
      type: "boolean",
      title: "Include Image URL",
      default: false,
      "ui:widget": "toggle"
    },
    imageUrl: {
      type: "string",
      title: "Image URL",
      description: "URL of image to analyze. Supports {{input.imageUrl}} syntax.",
      default: "",
      "ui:field": "template",
      "ui:dependencies": {
        includeImageUrl: true
      }
    }
  },
  required: ["model"]
}
```

### OpenAI Node
```typescript
configSchema: {
  type: "object",
  properties: {
    model: {
      type: "string",
      title: "Model",
      enum: ["gpt-3.5-turbo", "gpt-4", "gpt-4-turbo-preview", "gpt-4o"],
      enumNames: ["GPT-3.5 Turbo", "GPT-4", "GPT-4 Turbo", "GPT-4o"],
      default: "gpt-3.5-turbo"
    },
    temperature: {
      type: "number",
      title: "Temperature",
      description: "Controls randomness (0-2)",
      default: 0.7,
      minimum: 0,
      maximum: 2
    },
    maxTokens: {
      type: "number",
      title: "Max Tokens",
      default: 256,
      minimum: 1,
      maximum: 4096
    },
    systemPrompt: {
      type: "string",
      title: "System Prompt",
      description: "System message. Supports {{input.fieldName}} syntax.",
      default: "",
      "ui:field": "template"
    },
    prompt: {
      type: "string",
      title: "Prompt", 
      description: "User message. Supports {{input.fieldName}} syntax.",
      default: "",
      "ui:field": "template"
    },
    history: {
      type: "object",
      title: "History",
      description: "Message history [] for context",
      default: "",
      "ui:field": "template"
    }
  },
  required: ["model"]
}
```

### Code Node
```typescript
configSchema: {
  type: "object",
  properties: {
    code: {
      type: "object",
      title: "Code",
      description: "JS Code to transform data",
      default: "",
      "ui:field": "template"  // JavaScript execution
    },
    generateIds: {
      type: "boolean",
      title: "Generate IDs",
      description: "Generate content IDs",
      default: false,
      "ui:widget": "toggle"
    }
  },
  required: ["code"]
}
```

## 🎯 Template Resolution

### String Templates (type: "string")
```typescript
// Config field:
prompt: {
  type: "string",
  "ui:field": "template"
}

// User enters:
"Hello {{input.name}}, your score is {{signal.score}}"

// Resolves to:
"Hello John, your score is 95"
```

### Object Templates (type: "object")
```typescript
// Config field:
code: {
  type: "object", 
  "ui:field": "template"
}

// User enters:
return {
  filtered: input.items.filter(x => x.score > 80),
  count: input.items.length
}

// Resolves to:
{
  filtered: [...],
  count: 25
}
```

## 🚨 Critical Rules

### 1. Template Type Distinction
- **`type: "string"` + `"ui:field": "template"`** → Handlebars syntax `{{input.field}}`
- **`type: "object"` + `"ui:field": "template"`** → JavaScript return statement

### 2. Required Fields
```typescript
required: ["fieldName"]  // Makes field mandatory in UI
```

### 3. Dependencies
```typescript
"ui:dependencies": {
  parentField: expectedValue  // Only show when parent field has expected value
}
```

### 4. Validation
- Use `minimum`, `maximum` for numbers
- Use `enum` for dropdowns
- Use `required` array for mandatory fields

---

**Next**: [Troubleshooting](./05-troubleshooting.md) - Common config schema issues
