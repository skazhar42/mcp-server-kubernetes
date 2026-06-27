# MCP Server for Kubernetes Management - Developer Guide

This document explains the architecture, data flow, and key concepts in this MCP (Model Context Protocol) server implementation. It's designed to help developers understand how this project works and serve as a reference for building similar MCP servers.

## Table of Contents

- [What is MCP?](#what-is-mcp)
- [Project Structure](#project-structure)
- [Data Flow](#data-flow)
- [Core Components](#core-components)
- [Key MCP Concepts](#key-mcp-concepts)
- [Tool Development Guide](#tool-development-guide)
- [Resource Management](#resource-management)
- [Environment & Configuration](#environment--configuration)

---

## What is MCP?

**Model Context Protocol (MCP)** is an open protocol that allows AI models and applications to interact with external data sources and perform operations through standardized interfaces. Think of it as a bridge between AI assistants (like Claude) and external systems (like Kubernetes).

### MCP Architecture

MCP uses a **client-server model**:

```
┌─────────────────────────────────────┐
│  MCP Client                         │
│  (Claude AI, Claude Desktop, etc.)  │
└────────────┬────────────────────────┘
             │ JSON-RPC 2.0
             │ (Stdio, SSE, WebSocket)
             │
┌────────────▼────────────────────────┐
│  MCP Server                         │
│  (This project)                     │
│  - Tools                            │
│  - Resources                        │
│  - Prompts                          │
└────────────┬────────────────────────┘
             │
             ▼
┌─────────────────────────────────────┐
│  Backend Systems                    │
│  (Kubernetes, APIs, Databases)      │
└─────────────────────────────────────┘
```

### Key MCP Capabilities

1. **Tools**: Functions that the AI can call with parameters. Similar to function calling in LLMs.
2. **Resources**: Accessible data sources and files that provide context to the AI.
3. **Prompts**: Pre-built prompt templates that guide AI behavior.
4. **Sampling**: Request completions from the model directly (server → client).

---

## Project Structure

```
mcp-server-kubernetes/
├── src/
│   ├── index.ts                    # Server initialization & setup
│   ├── config/                     # Configuration schemas & templates
│   │   ├── deployment-config.ts    # Deployment configuration schemas
│   │   ├── namespace-config.ts     # Namespace configuration
│   │   ├── container-config.ts     # Container specifications
│   │   └── cleanup-config.ts       # Cleanup operation templates
│   ├── tools/                      # MCP Tool implementations
│   │   ├── kubectl-*.ts            # kubectl operation tools
│   │   ├── helm-*.ts               # Helm operation tools
│   │   └── *-operations.ts         # Specialized Kubernetes tools
│   ├── resources/                  # MCP Resource handlers
│   │   └── handlers.ts             # Resource endpoint implementations
│   ├── utils/
│   │   └── kubernetes-manager.ts   # Core Kubernetes API wrapper
│   └── tests/                      # Test suite
├── dist/                           # Compiled JavaScript output
├── package.json                    # Dependencies & scripts
├── tsconfig.json                   # TypeScript configuration
├── vitest.config.ts                # Test configuration
└── CLAUDE.md                       # Development guidance

```

### Key Directories Explained

#### `/src/index.ts` - Server Entry Point
- Initializes the MCP server with StdioTransport
- Registers all available tools
- Sets up resource handlers
- Implements tool filtering logic for destructive/non-destructive modes

#### `/src/tools/` - MCP Tool Implementations
Each tool file exports:
- **Input Schema** (Zod): Validates tool parameters
- **Tool Handler Function**: Executes the operation
- **Example**:
```typescript
export const MyToolSchema = z.object({
  param1: z.string().describe('Description of param1'),
});

export type MyToolInput = z.infer<typeof MyToolSchema>;

export async function executeMyTool(input: MyToolInput): Promise<ToolResponse> {
  // Implementation
}
```

#### `/src/utils/kubernetes-manager.ts` - Kubernetes Manager
Centralizes all kubectl and Kubernetes API interactions:
- Manages kubeconfig loading from multiple sources
- Executes kubectl commands
- Tracks created resources
- Manages port forwards and watchers
- Handles context switching

#### `/src/config/` - Configuration Schemas
Zod schemas that define:
- Deployment specifications (replicas, images, ports, etc.)
- Namespace configurations
- Container specifications
- Cleanup operation templates

---

## Data Flow

### Request-Response Flow

```
1. CLIENT SENDS REQUEST
   ┌─────────────────────────────────────────────┐
   │ {                                           │
   │   "jsonrpc": "2.0",                         │
   │   "id": 1,                                  │
   │   "method": "tools/call",                   │
   │   "params": {                               │
   │     "name": "kubectl_get",                  │
   │     "arguments": {"resource": "pods"}       │
   │   }                                         │
   │ }                                           │
   └─────────────────────────────────────────────┘
            │
            ▼
2. SERVER PROCESSES REQUEST
   ┌─────────────────────────────────────────────┐
   │ a) Router finds "kubectl_get" tool          │
   │ b) Validate args against tool schema        │
   │ c) Call tool handler function               │
   │ d) Handler delegates to KubernetesManager   │
   └─────────────────────────────────────────────┘
            │
            ▼
3. KUBERNETES MANAGER EXECUTES
   ┌─────────────────────────────────────────────┐
   │ a) Load kubeconfig                          │
   │ b) Build kubectl command                    │
   │ c) Execute via child_process                │
   │ d) Parse output (JSON or text)              │
   │ e) Handle errors                            │
   └─────────────────────────────────────────────┘
            │
            ▼
4. RESPONSE RETURNED
   ┌─────────────────────────────────────────────┐
   │ {                                           │
   │   "jsonrpc": "2.0",                         │
   │   "id": 1,                                  │
   │   "result": {                               │
   │     "content": [{                           │
   │       "type": "text",                       │
   │       "text": "<kubectl output>"            │
   │     }]                                      │
   │   }                                         │
   │ }                                           │
   └─────────────────────────────────────────────┘
```

### Tool Execution Flow (Detailed)

```
MCP Client Request
     │
     ▼
┌─────────────────────────────────────┐
│ index.ts Tool Handler               │
│ Receives: {name, arguments}         │
└────────┬────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────┐
│ Schema Validation (Zod)             │
│ Validates arguments match schema    │
└────────┬────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────┐
│ Specific Tool File (e.g., kubectl-  │
│ get.ts)                             │
│ - Execute business logic            │
│ - Call KubernetesManager methods    │
└────────┬────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────┐
│ KubernetesManager                   │
│ - Load kubeconfig                   │
│ - Execute kubectl command           │
│ - Parse result                      │
│ - Handle errors                     │
└────────┬────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────┐
│ Child Process (kubectl)             │
│ Executes system command             │
└────────┬────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────┐
│ Kubernetes Cluster                  │
│ Returns API response                │
└────────┬────────────────────────────┘
         │
         ▼
Response flows back through the layers
to client
```

---

## Core Components

### 1. KubernetesManager (`src/utils/kubernetes-manager.ts`)

**Purpose**: Centralized wrapper around kubectl operations.

**Key Methods**:
```typescript
class KubernetesManager {
  // Execute kubectl commands
  async execKubectl(
    command: string,
    args: string[],
    parseJson?: boolean
  ): Promise<string | object>
  
  // Get current context
  async getCurrentContext(): Promise<string>
  
  // Switch contexts
  async switchContext(context: string): Promise<void>
  
  // Port forwarding
  async portForward(
    resourceType: string,
    resourceName: string,
    ports: string[]
  ): Promise<string>
  
  // Watch resources
  async watch(
    resourceType: string,
    namespace?: string
  ): Promise<string>
  
  // Track created resources
  trackResource(type: string, name: string, namespace: string): void
}
```

**Usage Pattern**:
```typescript
const manager = KubernetesManager.getInstance();
const pods = await manager.execKubectl('get', ['pods', '-o', 'json'], true);
```

### 2. Tool Schema (Zod)

Each tool has a Zod schema that validates input parameters.

**Example**:
```typescript
import { z } from 'zod';

export const KubectlGetSchema = z.object({
  resource: z.string().describe('Resource type (pods, services, deployments, etc.)'),
  namespace: z.string().optional().describe('Kubernetes namespace'),
  name: z.string().optional().describe('Optional: specific resource name'),
});

export type KubectlGetInput = z.infer<typeof KubectlGetSchema>;
```

**MCP Integration**: Schema is exposed to clients so they know what parameters each tool accepts.

### 3. Tool Handler Functions

Each tool has an async handler that:
1. Receives validated input
2. Calls KubernetesManager methods
3. Formats output for the client
4. Returns a TextContent or ToolResult

**Example**:
```typescript
export async function executeKubectlGet(input: KubectlGetInput): Promise<ToolResult> {
  const manager = KubernetesManager.getInstance();
  
  try {
    const args = ['get', input.resource];
    if (input.namespace) args.push('-n', input.namespace);
    if (input.name) args.push(input.name);
    args.push('-o', 'json');
    
    const result = await manager.execKubectl('', args, true);
    
    return {
      content: [{
        type: 'text',
        text: JSON.stringify(result, null, 2)
      }]
    };
  } catch (error) {
    return {
      content: [{
        type: 'text',
        text: `Error: ${error.message}`
      }],
      isError: true
    };
  }
}
```

### 4. Server Initialization (`src/index.ts`)

The main server file:
- Creates MCP server instance
- Registers all tools with their schemas
- Implements tool call handler
- Sets up resource handlers
- Applies destructive tool filtering

**Key Sections**:
```typescript
// 1. Create server
const server = new Server({
  name: "kubernetes-manager",
  version: "1.0.0",
});

// 2. Register tools
server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: "tool_name",
      description: "What this tool does",
      inputSchema: ToolSchema
    }
  ]
}));

// 3. Handle tool calls
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;
  
  // Validate, execute, return response
});

// 4. Connect transport
transport.connect(server);
```

---

## Key MCP Concepts

### 1. Transport Layer

MCP supports multiple transports for communication:

- **Stdio Transport**: Uses standard input/output (used in Claude Desktop)
- **SSE Transport**: Server-Sent Events over HTTP
- **WebSocket Transport**: Full-duplex WebSocket connection

```typescript
import { StdioTransport } from '@modelcontextprotocol/sdk/server/stdio';

const transport = new StdioTransport({
  command: 'node',
  args: ['dist/index.js']
});

transport.connect(server);
```

### 2. JSON-RPC 2.0

MCP uses JSON-RPC 2.0 for all communication:

```typescript
// Request format
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/list",
  "params": {}
}

// Response format
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": { /* response data */ }
}

// Error format
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32600,
    "message": "Invalid Request"
  }
}
```

### 3. Content Types

Tools return content in standardized formats:

```typescript
// Text content
{
  type: 'text',
  text: 'Plain text response'
}

// Image content (base64)
{
  type: 'image',
  mimeType: 'image/png',
  data: 'base64_encoded_image_data'
}

// Embedded resources
{
  type: 'resource',
  resource: {
    uri: 'file:///path/to/file',
    mimeType: 'text/plain',
    contents: 'file contents'
  }
}
```

### 4. Tool Categories

**Read-Only Tools**: Safe to expose to any client
- `kubectl_get`: Retrieve resources
- `kubectl_describe`: Get detailed information

**Destructive Tools**: Require explicit permission
- `kubectl_delete`: Remove resources
- `kubectl_apply`: Modify cluster state
- Environment variable `ALLOW_ONLY_NON_DESTRUCTIVE_TOOLS=true` disables these

**Example Filtering**:
```typescript
if (process.env.ALLOW_ONLY_NON_DESTRUCTIVE_TOOLS) {
  tools = tools.filter(t => !t.isDestructive);
}
```

### 5. Error Handling

MCP errors are structured:

```typescript
// Validation error
throw {
  code: -32602,
  message: 'Invalid params',
  data: { details: 'parameter validation failed' }
};

// Server error
throw {
  code: -32603,
  message: 'Internal error',
  data: { error: error.message }
};

// Tool execution error
return {
  content: [{
    type: 'text',
    text: `Error: ${error.message}`
  }],
  isError: true
};
```

### 6. Resources

MCP Resources expose read-only data to clients:

```typescript
// List available resources
server.setRequestHandler(ListResourcesRequestSchema, async () => ({
  resources: [
    {
      uri: 'kubernetes://cluster/info',
      name: 'Cluster Information',
      mimeType: 'application/json'
    }
  ]
}));

// Read a resource
server.setRequestHandler(ReadResourceRequestSchema, async (request) => {
  const { uri } = request.params;
  
  return {
    contents: [{
      uri,
      mimeType: 'application/json',
      text: JSON.stringify(data)
    }]
  };
});
```

---

## Tool Development Guide

### Step 1: Define the Schema

Create a new tool file (e.g., `src/tools/my-operation.ts`):

```typescript
import { z } from 'zod';

export const MyOperationSchema = z.object({
  param1: z.string().describe('Parameter description'),
  param2: z.number().optional().describe('Optional parameter'),
});

export type MyOperationInput = z.infer<typeof MyOperationSchema>;
```

### Step 2: Implement the Handler

```typescript
export async function executeMyOperation(
  input: MyOperationInput
): Promise<ToolResult> {
  const manager = KubernetesManager.getInstance();
  
  try {
    // Your business logic here
    const result = await manager.execKubectl(...);
    
    return {
      content: [{
        type: 'text',
        text: `Success: ${JSON.stringify(result, null, 2)}`
      }]
    };
  } catch (error) {
    return {
      content: [{
        type: 'text',
        text: `Error: ${error.message}`
      }],
      isError: true
    };
  }
}
```

### Step 3: Register in Server

In `src/index.ts`, add to the tools list:

```typescript
import { MyOperationSchema, executeMyOperation } from './tools/my-operation';

// In tools list
{
  name: 'my_operation',
  description: 'Description of what this tool does',
  inputSchema: MyOperationSchema
}

// In tool call handler
case 'my_operation':
  return executeMyOperation(args as MyOperationInput);
```

### Step 4: Add Tests

Create `src/tests/my-operation.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { MyOperationSchema } from '../tools/my-operation';

describe('MyOperation', () => {
  it('should validate input schema', () => {
    const input = { param1: 'value' };
    const result = MyOperationSchema.parse(input);
    expect(result).toEqual(input);
  });
  
  it('should execute operation', async () => {
    // Test implementation
  });
});
```

---

## Resource Management

### Resource Tracking

The KubernetesManager tracks all created resources for cleanup:

```typescript
// When creating a resource
manager.trackResource('deployment', 'my-app', 'default');

// Get all tracked resources
const tracked = manager.getTrackedResources();

// Clean up tracked resources
await manager.cleanupTrackedResources();
```

### Port Forwarding

Managed port forwards for accessing services:

```typescript
// Start port forward
const portInfo = await manager.portForward('pod', 'my-pod', ['8080:3000']);
// Returns: "Forwarding from 127.0.0.1:8080 -> 3000"

// List active forwards
const forwards = manager.getActivePortForwards();
```

### Watchers

Watch resources for changes:

```typescript
// Watch pods in namespace
const watcher = await manager.watch('pods', 'default');
// Returns stream of changes
```

---

## Environment & Configuration

### Required Environment Variables

```bash
# Kubernetes kubeconfig (optional, defaults to ~/.kube/config)
KUBECONFIG=/path/to/kubeconfig

# Or provide full kubeconfig as YAML
KUBECONFIG_YAML='...'

# Disable destructive operations
ALLOW_ONLY_NON_DESTRUCTIVE_TOOLS=true
```

### KubeConfig Loading Priority

1. `KUBECONFIG_YAML` environment variable (full kubeconfig YAML)
2. `KUBECONFIG` path environment variable
3. `~/.kube/config` default location

### Configuration Schemas

Located in `src/config/`:

- **deployment-config.ts**: Define deployment specs (replicas, images, resources)
- **namespace-config.ts**: Namespace creation templates
- **container-config.ts**: Container specifications
- **cleanup-config.ts**: Cleanup operation templates

---

## Development Workflow

### Building

```bash
bun run build
# Compiles TypeScript to dist/ and creates executables
```

### Development Mode

```bash
bun run dev
# Starts TypeScript compiler in watch mode
```

### Testing

```bash
# Run all tests
bun run test

# Run specific test
bun run test -- src/tests/my-test.ts

# Watch mode
bun run test -- --watch
```

### Local Testing

```bash
# Test with Inspector (MCP client)
npx @modelcontextprotocol/inspector node dist/index.js

# Test with chat CLI
bun run chat

# Test with Claude Desktop
# Point config to local dist/index.js
```

---

## References

- [Model Context Protocol Specification](https://spec.modelcontextprotocol.io/)
- [MCP SDK Documentation](https://github.com/modelcontextprotocol/typescript-sdk)
- [Kubernetes API Documentation](https://kubernetes.io/docs/reference/generated/kubernetes-api/)
- [kubectl Documentation](https://kubernetes.io/docs/reference/kubectl/)

---

## Summary

This MCP server demonstrates:
- **Tool-based architecture**: Exposing Kubernetes operations as callable tools
- **Schema validation**: Using Zod for type-safe parameter validation
- **Centralized management**: KubernetesManager handles all kubectl operations
- **Error handling**: Graceful error management and reporting
- **Mode filtering**: Supporting destructive/non-destructive operation modes
- **Standard transport**: Using JSON-RPC 2.0 over stdio for compatibility

When building your own MCP server, follow these patterns:
1. Define clear tool schemas with Zod
2. Centralize external system interactions (like KubernetesManager)
3. Implement proper error handling
4. Validate all inputs
5. Support multiple transport mechanisms
6. Write comprehensive tests

