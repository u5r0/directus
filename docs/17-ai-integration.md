# AI Integration

This document covers the AI features in Directus, including the AI chat interface and Model Context Protocol (MCP) integration.

## Overview

Directus includes built-in AI capabilities that allow users to interact with AI models directly within the admin interface. The system supports multiple AI providers and implements the Model Context Protocol (MCP) for standardized AI tool integration.

**Key Features:**
- AI chat interface in the admin app
- Model Context Protocol (MCP) server
- Multiple AI provider support (Anthropic, OpenAI)
- Custom AI tools for Directus operations
- System prompts and custom prompts
- Tool approval system for security

## AI Architecture

### Directory Structure

```
api/src/ai/
├── chat/              # AI chat functionality
│   ├── controllers/   # Chat API controllers
│   ├── lib/           # Chat utilities
│   ├── middleware/    # Chat middleware
│   ├── models/        # Request/response models
│   ├── utils/         # Helper functions
│   └── router.ts      # Chat routes
├── mcp/               # Model Context Protocol
│   ├── server.ts      # MCP server implementation
│   ├── transport.ts   # MCP transport layer
│   └── types.ts       # MCP types
└── tools/             # AI tools
    ├── items/         # Item operations
    ├── files/         # File operations
    ├── collections/   # Collection operations
    ├── fields/        # Field operations
    ├── flows/         # Flow operations
    └── ...            # Other tools
```

## AI Configuration

### Environment Variables

```bash
# Enable AI features
AI_ENABLED=true

# AI Provider Configuration
AI_PROVIDER=anthropic  # or openai

# Anthropic
AI_ANTHROPIC_API_KEY=your-api-key
AI_ANTHROPIC_MODEL=claude-3-5-sonnet-20241022

# OpenAI
AI_OPENAI_API_KEY=your-api-key
AI_OPENAI_MODEL=gpt-4-turbo

# MCP Configuration
MCP_ENABLED=true
MCP_ALLOW_DELETES=false
MCP_PROMPTS_COLLECTION=ai_prompts
MCP_SYSTEM_PROMPT_ENABLED=true
MCP_SYSTEM_PROMPT="You are a helpful assistant for Directus."
```

## AI Chat

### Chat API Endpoint

```
POST /ai/chat
```

### Chat Request

```typescript
interface ChatRequest {
  provider: 'anthropic' | 'openai';
  model: string;
  messages: Message[];
  tools?: (string | ToolRequest)[];
  toolApprovals?: Record<string, boolean>;
}

interface Message {
  role: 'user' | 'assistant' | 'system';
  content: string;
  toolInvocations?: ToolInvocation[];
}

interface ToolRequest {
  name: string;
  args?: Record<string, any>;
}
```

### Chat Response

The chat endpoint returns a Server-Sent Events (SSE) stream:

```typescript
// Text chunk
data: {"type":"text-delta","textDelta":"Hello"}

// Tool call
data: {"type":"tool-call","toolCallId":"call_123","toolName":"items","args":{...}}

// Tool result
data: {"type":"tool-result","toolCallId":"call_123","result":{...}}

// Usage data
data: {"type":"data-usage","data":{"promptTokens":100,"completionTokens":50}}

// Stream end
data: [DONE]
```

### Chat Controller (`api/src/ai/chat/controllers/chat.post.ts`)

```typescript
export const aiChatPostHandler: RequestHandler = async (req, res) => {
  // Validate request
  const { provider, model, messages, tools, toolApprovals } = req.body;
  
  // Convert tools to AI SDK format
  const aiTools = tools.reduce((acc, tool) => {
    acc[tool.name] = chatRequestToolToAiSdkTool({
      chatRequestTool: tool,
      accountability: req.accountability,
      schema: req.schema,
      toolApprovals,
    });
    return acc;
  }, {});
  
  // Create UI stream
  const stream = createUiStream(messages, {
    provider,
    model,
    tools: aiTools,
    apiKeys: res.locals.ai.apiKeys,
    systemPrompt: res.locals.ai.systemPrompt,
  });
  
  // Pipe to response
  stream.pipeUIMessageStreamToResponse(res);
};
```

## Model Context Protocol (MCP)

### MCP Server

Directus implements an MCP server that exposes Directus operations as AI tools.

**MCP Endpoint:**
```
POST /mcp
```

### MCP Server Implementation (`api/src/ai/mcp/server.ts`)

```typescript
export class DirectusMCP {
  server: Server;
  promptsCollection?: string;
  systemPrompt?: string;
  allowDeletes?: boolean;
  
  constructor(options: MCPOptions = {}) {
    this.promptsCollection = options.promptsCollection;
    this.systemPrompt = options.systemPrompt;
    this.allowDeletes = options.allowDeletes ?? false;
    
    this.server = new Server(
      {
        name: 'directus-mcp',
        version: '0.1.0',
      },
      {
        capabilities: {
          tools: {},
          prompts: {},
        },
      },
    );
    
    this.registerHandlers();
  }
  
  private registerHandlers() {
    // List tools
    this.server.setRequestHandler(ListToolsRequestSchema, () => {
      return { tools: getAllMcpTools() };
    });
    
    // Call tool
    this.server.setRequestHandler(CallToolRequestSchema, async (request) => {
      const tool = findMcpTool(request.params.name);
      const result = await tool.handler({
        args: request.params.arguments,
        schema: req.schema,
        accountability: req.accountability,
      });
      return this.toToolResponse(result);
    });
    
    // List prompts
    this.server.setRequestHandler(ListPromptsRequestSchema, async () => {
      const prompts = await this.loadPrompts();
      return { prompts };
    });
    
    // Get prompt
    this.server.setRequestHandler(GetPromptRequestSchema, async (request) => {
      const prompt = await this.loadPrompt(request.params.name);
      return this.toPromptResponse(prompt);
    });
  }
}
```

### MCP Protocol

**List Tools Request:**
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/list"
}
```

**List Tools Response:**
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "tools": [
      {
        "name": "items",
        "description": "Perform CRUD operations on items",
        "inputSchema": {
          "type": "object",
          "properties": {
            "action": {
              "type": "string",
              "enum": ["create", "read", "update", "delete"]
            },
            "collection": {
              "type": "string"
            }
          }
        }
      }
    ]
  }
}
```

**Call Tool Request:**
```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "items",
    "arguments": {
      "action": "read",
      "collection": "articles",
      "query": {
        "limit": 10
      }
    }
  }
}
```

**Call Tool Response:**
```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "{\"data\":[...],\"url\":\"https://directus.com/admin/content/articles\"}"
      }
    ]
  }
}
```

## AI Tools

### Available Tools

Directus provides 12 built-in AI tools:

1. **system-prompt** - Get system prompt
2. **items** - CRUD operations on items
3. **files** - File operations
4. **folders** - Folder operations
5. **assets** - Asset operations
6. **flows** - Flow operations
7. **trigger-flow** - Trigger flow execution
8. **operations** - Operation management
9. **schema** - Schema inspection
10. **collections** - Collection management
11. **fields** - Field management
12. **relations** - Relation management

### Tool Definition

```typescript
import { defineTool } from '../define-tool.js';
import { z } from 'zod';

export const items = defineTool({
  name: 'items',
  description: 'Perform CRUD operations on collection items',
  admin: false,
  inputSchema: z.object({
    action: z.enum(['create', 'read', 'update', 'delete', 'import']),
    collection: z.string(),
    keys: z.array(z.union([z.string(), z.number()])).optional(),
    data: z.any().optional(),
    query: z.object({
      fields: z.array(z.string()).optional(),
      filter: z.record(z.any()).optional(),
      sort: z.array(z.string()).optional(),
      limit: z.number().optional(),
    }).optional(),
  }),
  handler: async ({ args, schema, accountability }) => {
    const service = new ItemsService(args.collection, {
      schema,
      accountability,
    });
    
    switch (args.action) {
      case 'create':
        const created = await service.createOne(args.data);
        return { data: created };
      
      case 'read':
        if (args.keys) {
          const items = await service.readMany(args.keys, args.query);
          return { data: items };
        } else {
          const items = await service.readByQuery(args.query);
          return { data: items };
        }
      
      case 'update':
        await service.updateMany(args.keys, args.data);
        return { data: { updated: args.keys.length } };
      
      case 'delete':
        await service.deleteMany(args.keys);
        return { data: { deleted: args.keys.length } };
    }
  },
  endpoint: ({ input, data }) => {
    if (input.action === 'read' && Array.isArray(data) && data.length === 1) {
      return ['content', input.collection, String(data[0].id)];
    }
    return ['content', input.collection];
  },
});
```

### Tool Context

Tools receive a context object:

```typescript
interface ToolContext {
  args: ToolArgs;
  schema: SchemaOverview;
  accountability: Accountability;
}
```

### Tool Result

Tools return a result object:

```typescript
interface ToolResult {
  type?: 'text' | 'image' | 'resource';
  data: any;
  url?: string;
}
```

## AI Prompts

### Prompt Collection

Store custom prompts in a collection:

```typescript
interface Prompt {
  name: string;
  description: string;
  system_prompt?: string;
  messages: Array<{
    role: 'user' | 'assistant';
    text: string;
  }>;
}
```

### Prompt Variables

Use mustache syntax for variables:

```typescript
{
  name: 'summarize-article',
  description: 'Summarize an article',
  messages: [
    {
      role: 'user',
      text: 'Please summarize this article: {{article_content}}'
    }
  ]
}
```

### Using Prompts

**List Prompts:**
```json
{
  "method": "prompts/list"
}
```

**Get Prompt:**
```json
{
  "method": "prompts/get",
  "params": {
    "name": "summarize-article",
    "arguments": {
      "article_content": "Article text here..."
    }
  }
}
```

## Tool Approval System

### Security

For security, tools can require approval:

```typescript
interface ToolApprovals {
  [toolName: string]: boolean;
}

// Request with approvals
{
  "tools": ["items", "files"],
  "toolApprovals": {
    "items": true,
    "files": false  // Will prompt user
  }
}
```

### Admin-Only Tools

Some tools require admin access:

```typescript
export const schema = defineTool({
  name: 'schema',
  description: 'Inspect database schema',
  admin: true,  // Requires admin
  // ...
});
```

## Frontend Integration

### AI Chat Component

```vue
<template>
  <div class="ai-chat">
    <div class="messages">
      <div
        v-for="message in messages"
        :key="message.id"
        :class="['message', message.role]"
      >
        {{ message.content }}
      </div>
    </div>
    
    <div class="input">
      <textarea
        v-model="input"
        @keydown.enter.prevent="sendMessage"
        placeholder="Ask AI..."
      />
      <button @click="sendMessage">Send</button>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue';
import { useApi } from '@directus/composables';

const api = useApi();
const messages = ref([]);
const input = ref('');

async function sendMessage() {
  if (!input.value.trim()) return;
  
  // Add user message
  messages.value.push({
    id: Date.now(),
    role: 'user',
    content: input.value,
  });
  
  const userMessage = input.value;
  input.value = '';
  
  // Call AI API
  const response = await fetch('/ai/chat', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      provider: 'anthropic',
      model: 'claude-3-5-sonnet-20241022',
      messages: messages.value,
      tools: ['items', 'files'],
    }),
  });
  
  // Handle SSE stream
  const reader = response.body.getReader();
  const decoder = new TextDecoder();
  let assistantMessage = '';
  
  while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    
    const chunk = decoder.decode(value);
    const lines = chunk.split('\n');
    
    for (const line of lines) {
      if (line.startsWith('data: ')) {
        const data = JSON.parse(line.slice(6));
        
        if (data.type === 'text-delta') {
          assistantMessage += data.textDelta;
        }
      }
    }
  }
  
  // Add assistant message
  messages.value.push({
    id: Date.now(),
    role: 'assistant',
    content: assistantMessage,
  });
}
</script>
```

### AI Store

```typescript
import { defineStore } from 'pinia';

export const useAiStore = defineStore('ai', {
  state: () => ({
    enabled: false,
    provider: 'anthropic',
    model: 'claude-3-5-sonnet-20241022',
    systemPrompt: '',
  }),
  
  actions: {
    async chat(messages, tools = []) {
      const response = await fetch('/ai/chat', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({
          provider: this.provider,
          model: this.model,
          messages,
          tools,
        }),
      });
      
      return response;
    },
    
    onSystemToolResult(callback) {
      // Listen for tool results
      this.toolResultCallbacks.push(callback);
    },
  },
});
```

## Best Practices

### Security

- Always validate tool inputs
- Use accountability context
- Implement tool approvals
- Restrict admin-only tools
- Sanitize AI responses

### Performance

- Stream responses for better UX
- Cache prompts
- Limit tool execution time
- Monitor API usage
- Implement rate limiting

### User Experience

- Provide clear tool descriptions
- Show tool execution progress
- Handle errors gracefully
- Allow tool approval/rejection
- Display usage metrics

### Cost Management

- Monitor token usage
- Implement usage limits
- Cache common queries
- Use appropriate models
- Track API costs

## Troubleshooting

### Common Issues

**AI not enabled:**
```bash
AI_ENABLED=true
AI_PROVIDER=anthropic
AI_ANTHROPIC_API_KEY=your-key
```

**MCP not working:**
```bash
MCP_ENABLED=true
MCP_PROMPTS_COLLECTION=ai_prompts
```

**Tool execution fails:**
- Check accountability permissions
- Verify tool arguments
- Review error logs
- Test tool independently

## Next Steps

- [Workflow Engine](./18-workflow-engine.md) - Automation with flows
- [Extension System](./15-extension-system.md) - Custom AI tools
- [Authentication & Security](./06-auth-security.md) - AI security
