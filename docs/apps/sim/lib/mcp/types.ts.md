```typescript
/**
 * Model Context Protocol (MCP) Types
 *
 * Type definitions for JSON-RPC 2.0 based MCP implementation
 * Supporting HTTP/SSE and Streamable HTTP transports
 */

// JSON-RPC 2.0 Base Types
export interface JsonRpcRequest {
  jsonrpc: '2.0'
  id: string | number
  method: string
  params?: any
}

export interface JsonRpcResponse<T = any> {
  jsonrpc: '2.0'
  id: string | number
  result?: T
  error?: JsonRpcError
}

export interface JsonRpcNotification {
  jsonrpc: '2.0'
  method: string
  params?: any
}

export interface JsonRpcError {
  code: number
  message: string
  data?: any
}

// MCP Transport Types
export type McpTransport = 'http' | 'sse' | 'streamable-http'

export interface McpServerConfig {
  id: string
  name: string
  description?: string
  transport: McpTransport

  // HTTP/SSE transport config
  url?: string
  headers?: Record<string, string>

  // Common config
  timeout?: number
  retries?: number
  enabled?: boolean
  createdAt?: string
  updatedAt?: string
}

// MCP Protocol Types
export interface McpCapabilities {
  tools?: {
    listChanged?: boolean
  }
  resources?: {
    subscribe?: boolean
    listChanged?: boolean
  }
  prompts?: {
    listChanged?: boolean
  }
  logging?: Record<string, any>
}

export interface McpInitializeParams {
  protocolVersion: string
  capabilities: McpCapabilities
  clientInfo: {
    name: string
    version: string
  }
}

// Version negotiation support
export interface McpVersionInfo {
  supported: string[] // List of supported protocol versions
  preferred: string // Preferred version to use
}

export interface McpVersionNegotiationError extends JsonRpcError {
  code: -32000 // Custom error code for version negotiation failures
  message: 'Version negotiation failed'
  data: {
    clientVersions: string[]
    serverVersions: string[]
    reason: string
  }
}

export interface McpInitializeResult {
  protocolVersion: string
  capabilities: McpCapabilities
  serverInfo: {
    name: string
    version: string
  }
}

// Security and Consent Framework
export interface McpConsentRequest {
  type: 'tool_execution' | 'resource_access' | 'data_sharing'
  context: {
    serverId: string
    serverName: string
    action: string // Tool name or resource path
    description?: string // Human-readable description
    dataAccess?: string[] // Types of data being accessed
    sideEffects?: string[] // Potential side effects
  }
  expires?: number // Consent expiration timestamp
}

export interface McpConsentResponse {
  granted: boolean
  expires?: number
  restrictions?: Record<string, any> // Any access restrictions
  auditId?: string // For audit trail
}

export interface McpSecurityPolicy {
  requireConsent: boolean
  allowedOrigins?: string[]
  blockedOrigins?: string[]
  maxToolExecutionsPerHour?: number
  auditLevel: 'none' | 'basic' | 'detailed'
}

// MCP Tool Types
export interface McpToolSchema {
  type: string
  properties?: Record<string, any>
  required?: string[]
  additionalProperties?: boolean
  description?: string
}

export interface McpTool {
  name: string
  description?: string
  inputSchema: McpToolSchema
  serverId: string
  serverName: string
}

export interface McpToolCall {
  name: string
  arguments: Record<string, any>
}

// Standard MCP protocol response format
export interface McpToolResult {
  content?: Array<{
    type: 'text' | 'image' | 'resource'
    text?: string
    data?: string
    mimeType?: string
  }>
  isError?: boolean
  // Allow additional fields that some MCP servers return
  [key: string]: any
}

// MCP Resource Types
export interface McpResource {
  uri: string
  name: string
  description?: string
  mimeType?: string
}

export interface McpResourceContent {
  uri: string
  mimeType?: string
  text?: string
  blob?: string
}

// MCP Prompt Types
export interface McpPrompt {
  name: string
  description?: string
  arguments?: Array<{
    name: string
    description?: string
    required?: boolean
  }>
}

export interface McpPromptMessage {
  role: 'user' | 'assistant'
  content: {
    type: 'text' | 'image' | 'resource'
    text?: string
    data?: string
    mimeType?: string
  }
}

// Connection and Error Types
export interface McpConnectionStatus {
  connected: boolean
  lastConnected?: Date
  lastError?: string
  serverInfo?: McpInitializeResult['serverInfo']
}

export class McpError extends Error {
  constructor(
    message: string,
    public code?: number,
    public data?: any
  ) {
    super(message)
    this.name = 'McpError'
  }
}

export class McpConnectionError extends McpError {
  constructor(message: string, serverId: string) {
    super(`MCP Connection Error for server ${serverId}: ${message}`)
    this.name = 'McpConnectionError'
  }
}

export class McpTimeoutError extends McpError {
  constructor(serverId: string, timeout: number) {
    super(`MCP request to server ${serverId} timed out after ${timeout}ms`)
    this.name = 'McpTimeoutError'
  }
}

// Integration Types (for existing platform)
export interface McpToolInput {
  type: 'mcp'
  serverId: string
  toolName: string
  params: Record<string, any>
  usageControl?: 'auto' | 'force' | 'none'
}

export interface McpServerSummary {
  id: string
  name: string
  url?: string
  transport?: McpTransport
  status: 'connected' | 'disconnected' | 'error'
  toolCount: number
  resourceCount?: number
  promptCount?: number
  lastSeen?: Date
  error?: string
}

// API Response Types
export interface McpApiResponse<T = any> {
  success: boolean
  data?: T
  error?: string
}

export interface McpToolDiscoveryResponse {
  tools: McpTool[]
  totalCount: number
  byServer: Record<string, number>
}
```

### Purpose of this file

This TypeScript file defines the types and interfaces for the **Model Context Protocol (MCP)**. MCP facilitates communication between a client (e.g., an application) and one or more servers that expose models, tools, resources, and prompts. It's designed to standardize the way clients interact with these servers, regardless of the underlying transport mechanism. This file acts as a contract, ensuring that both clients and servers adhere to a common structure for requests, responses, notifications, errors, and data models.  The protocol is built upon the JSON-RPC 2.0 specification.

### Explanation of each line of code

The code is structured into logical sections, which I will explain in detail below.

**1. Document Header**

```typescript
/**
 * Model Context Protocol (MCP) Types
 *
 * Type definitions for JSON-RPC 2.0 based MCP implementation
 * Supporting HTTP/SSE and Streamable HTTP transports
 */
```
This is a multi-line comment that describes the purpose of the file. It states that the file defines TypeScript types for the Model Context Protocol (MCP), based on JSON-RPC 2.0. It also mentions the supported transport layers: HTTP, Server-Sent Events (SSE), and Streamable HTTP.

**2. JSON-RPC 2.0 Base Types**

These interfaces define the fundamental building blocks of the JSON-RPC 2.0 protocol, upon which MCP is built.

```typescript
export interface JsonRpcRequest {
  jsonrpc: '2.0'
  id: string | number
  method: string
  params?: any
}
```

*   `JsonRpcRequest`: Represents a JSON-RPC request from the client to the server.
    *   `jsonrpc`: A string indicating the JSON-RPC version (must be "2.0").
    *   `id`: A unique identifier for the request.  It can be a string or a number.  This ID is used to match requests to their corresponding responses.
    *   `method`: The name of the method to be invoked on the server.
    *   `params`: Optional parameters to be passed to the method.  `any` means it can be of any data type.

```typescript
export interface JsonRpcResponse<T = any> {
  jsonrpc: '2.0'
  id: string | number
  result?: T
  error?: JsonRpcError
}
```

*   `JsonRpcResponse`: Represents a JSON-RPC response from the server to the client.
    *   `jsonrpc`: A string indicating the JSON-RPC version (must be "2.0").
    *   `id`: The ID of the request this response corresponds to.
    *   `result`: The result of the method call, if successful.  The `<T = any>` syntax is a generic type, meaning the type of `result` can be specified when using the interface, but defaults to `any` if not specified.
    *   `error`: An error object, if the method call failed.

```typescript
export interface JsonRpcNotification {
  jsonrpc: '2.0'
  method: string
  params?: any
}
```

*   `JsonRpcNotification`: Represents a JSON-RPC notification from the server to the client.  Notifications are one-way and do not have a response.
    *   `jsonrpc`: A string indicating the JSON-RPC version (must be "2.0").
    *   `method`: The name of the method to be invoked on the client.
    *   `params`: Optional parameters to be passed to the method.

```typescript
export interface JsonRpcError {
  code: number
  message: string
  data?: any
}
```

*   `JsonRpcError`: Represents a JSON-RPC error object.
    *   `code`: A numeric error code.
    *   `message`: A short, human-readable error message.
    *   `data`: Optional data containing additional information about the error.

**3. MCP Transport Types**

These types define how the client and server communicate.

```typescript
export type McpTransport = 'http' | 'sse' | 'streamable-http'
```

*   `McpTransport`: A type alias that defines the allowed transport protocols for MCP. It can be one of the following:
    *   `http`: Standard HTTP requests.
    *   `sse`: Server-Sent Events, a unidirectional communication protocol where the server pushes updates to the client.
    *   `streamable-http`: HTTP that supports streaming of data.

```typescript
export interface McpServerConfig {
  id: string
  name: string
  description?: string
  transport: McpTransport

  // HTTP/SSE transport config
  url?: string
  headers?: Record<string, string>

  // Common config
  timeout?: number
  retries?: number
  enabled?: boolean
  createdAt?: string
  updatedAt?: string
}
```

*   `McpServerConfig`: Represents the configuration for an MCP server.
    *   `id`: A unique identifier for the server.
    *   `name`: A human-readable name for the server.
    *   `description`: An optional description of the server.
    *   `transport`: The transport protocol used by the server (`McpTransport`).
    *   `url`: The URL of the server (required for HTTP/SSE transports).
    *   `headers`: Optional HTTP headers to include in requests to the server.
    *   `timeout`: An optional timeout value (in milliseconds) for requests to the server.
    *   `retries`: An optional number of retries for failed requests.
    *   `enabled`: A boolean indicating whether the server is enabled.
    *   `createdAt`: An optional timestamp indicating when the server configuration was created.
    *   `updatedAt`: An optional timestamp indicating when the server configuration was last updated.

**4. MCP Protocol Types**

These interfaces define the core MCP protocol structures.

```typescript
export interface McpCapabilities {
  tools?: {
    listChanged?: boolean
  }
  resources?: {
    subscribe?: boolean
    listChanged?: boolean
  }
  prompts?: {
    listChanged?: boolean
  }
  logging?: Record<string, any>
}
```

*   `McpCapabilities`: Represents the capabilities of an MCP server. It indicates which features the server supports.
    *   `tools`:  Capabilities related to tools.
        *   `listChanged`:  Indicates if the server supports notifications when the list of available tools changes.
    *   `resources`: Capabilities related to resources.
        *   `subscribe`: Indicates if the client can subscribe to resource updates.
        *   `listChanged`: Indicates if the server supports notifications when the list of available resources changes.
    *   `prompts`: Capabilities related to prompts.
        *   `listChanged`: Indicates if the server supports notifications when the list of available prompts changes.
    *   `logging`:  Capabilities related to logging. `Record<string, any>` represents a dictionary of string keys and values of any type.

```typescript
export interface McpInitializeParams {
  protocolVersion: string
  capabilities: McpCapabilities
  clientInfo: {
    name: string
    version: string
  }
}
```

*   `McpInitializeParams`: Represents the parameters sent by the client during the initialization phase of the MCP connection.
    *   `protocolVersion`: The version of the MCP protocol the client wants to use.
    *   `capabilities`: The client's capabilities (`McpCapabilities`).
    *   `clientInfo`: Information about the client.
        *   `name`: The name of the client application.
        *   `version`: The version of the client application.

```typescript
// Version negotiation support
export interface McpVersionInfo {
  supported: string[] // List of supported protocol versions
  preferred: string // Preferred version to use
}
```

*   `McpVersionInfo`:  Represents information about the protocol versions supported by the server.
    *   `supported`: An array of strings representing the protocol versions supported by the server.
    *   `preferred`:  A string representing the server's preferred protocol version.

```typescript
export interface McpVersionNegotiationError extends JsonRpcError {
  code: -32000 // Custom error code for version negotiation failures
  message: 'Version negotiation failed'
  data: {
    clientVersions: string[]
    serverVersions: string[]
    reason: string
  }
}
```

*   `McpVersionNegotiationError`: Represents an error that occurs during version negotiation.  It extends `JsonRpcError`, adding specific information related to version negotiation failures.
    *   `code`:  A custom error code (-32000) indicating a version negotiation failure.
    *   `message`: A standard error message: "Version negotiation failed".
    *   `data`:  Provides details about the failure:
        *   `clientVersions`:  The protocol versions supported by the client.
        *   `serverVersions`: The protocol versions supported by the server.
        *   `reason`:  A string explaining why the version negotiation failed.

```typescript
export interface McpInitializeResult {
  protocolVersion: string
  capabilities: McpCapabilities
  serverInfo: {
    name: string
    version: string
  }
}
```

*   `McpInitializeResult`: Represents the result returned by the server after a successful initialization.
    *   `protocolVersion`: The protocol version that was negotiated and agreed upon.
    *   `capabilities`: The server's capabilities (`McpCapabilities`).
    *   `serverInfo`: Information about the server.
        *   `name`: The name of the server application.
        *   `version`: The version of the server application.

**5. Security and Consent Framework**

These interfaces define structures related to security and user consent.

```typescript
export interface McpConsentRequest {
  type: 'tool_execution' | 'resource_access' | 'data_sharing'
  context: {
    serverId: string
    serverName: string
    action: string // Tool name or resource path
    description?: string // Human-readable description
    dataAccess?: string[] // Types of data being accessed
    sideEffects?: string[] // Potential side effects
  }
  expires?: number // Consent expiration timestamp
}
```

*   `McpConsentRequest`: Represents a request for user consent before performing an action.
    *   `type`: The type of consent being requested (tool execution, resource access, or data sharing).
    *   `context`: Information about the action requiring consent.
        *   `serverId`: The ID of the server performing the action.
        *   `serverName`: The name of the server performing the action.
        *   `action`:  The name of the tool being executed or the path of the resource being accessed.
        *   `description`:  A human-readable description of the action.
        *   `dataAccess`:  An array of strings indicating the types of data being accessed.
        *   `sideEffects`:  An array of strings indicating the potential side effects of the action.
    *   `expires`:  An optional timestamp indicating when the consent expires.

```typescript
export interface McpConsentResponse {
  granted: boolean
  expires?: number
  restrictions?: Record<string, any> // Any access restrictions
  auditId?: string // For audit trail
}
```

*   `McpConsentResponse`: Represents the user's response to a consent request.
    *   `granted`: A boolean indicating whether the user granted consent.
    *   `expires`: An optional timestamp indicating when the consent expires.
    *   `restrictions`: Optional access restrictions applied to the action.
    *   `auditId`: An optional identifier for auditing purposes.

```typescript
export interface McpSecurityPolicy {
  requireConsent: boolean
  allowedOrigins?: string[]
  blockedOrigins?: string[]
  maxToolExecutionsPerHour?: number
  auditLevel: 'none' | 'basic' | 'detailed'
}
```

*   `McpSecurityPolicy`: Defines the security policy for an MCP server.
    *   `requireConsent`: A boolean indicating whether user consent is required for certain actions.
    *   `allowedOrigins`: An optional array of allowed origins.
    *   `blockedOrigins`: An optional array of blocked origins.
    *   `maxToolExecutionsPerHour`: An optional maximum number of tool executions per hour.
    *   `auditLevel`: The level of auditing to be performed (none, basic, or detailed).

**6. MCP Tool Types**

These interfaces define structures related to tools.

```typescript
export interface McpToolSchema {
  type: string
  properties?: Record<string, any>
  required?: string[]
  additionalProperties?: boolean
  description?: string
}
```

*   `McpToolSchema`: Represents the schema for the input of an MCP tool. It's based on JSON Schema.
    *   `type`: The data type of the input (e.g., "object", "string", "number").
    *   `properties`:  A dictionary defining the properties of the input (if the type is "object").
    *   `required`: An array of strings listing the required properties (if the type is "object").
    *   `additionalProperties`: A boolean indicating whether additional properties are allowed (if the type is "object").
    *   `description`: A description of the input.

```typescript
export interface McpTool {
  name: string
  description?: string
  inputSchema: McpToolSchema
  serverId: string
  serverName: string
}
```

*   `McpTool`: Represents an MCP tool.
    *   `name`: The name of the tool.
    *   `description`: An optional description of the tool.
    *   `inputSchema`: The schema for the tool's input (`McpToolSchema`).
    *   `serverId`: The ID of the server that hosts the tool.
    *   `serverName`: The name of the server that hosts the tool.

```typescript
export interface McpToolCall {
  name: string
  arguments: Record<string, any>
}
```

*   `McpToolCall`: Represents a call to an MCP tool.
    *   `name`: The name of the tool to be called.
    *   `arguments`: A dictionary of arguments to be passed to the tool.

```typescript
export interface McpToolResult {
  content?: Array<{
    type: 'text' | 'image' | 'resource'
    text?: string
    data?: string
    mimeType?: string
  }>
  isError?: boolean
  // Allow additional fields that some MCP servers return
  [key: string]: any
}
```

*   `McpToolResult`: Represents the result of an MCP tool call.
    *   `content`: An optional array of content items representing the result.
        *   `type`: The type of content (text, image, or resource).
        *   `text`: The text content (if the type is "text").
        *   `data`: The data content (if the type is "image" or "resource").
        *   `mimeType`: The MIME type of the data (if the type is "image" or "resource").
    *   `isError`: A boolean indicating whether the tool call resulted in an error.
    *   `[key: string]: any`: This is an index signature, which allows for additional, unspecified properties to be included in the `McpToolResult`. This caters to the variability in what MCP servers might return.

**7. MCP Resource Types**

These interfaces define structures related to resources.

```typescript
export interface McpResource {
  uri: string
  name: string
  description?: string
  mimeType?: string
}
```

*   `McpResource`: Represents an MCP resource.
    *   `uri`: The URI of the resource.
    *   `name`: The name of the resource.
    *   `description`: An optional description of the resource.
    *   `mimeType`: The MIME type of the resource.

```typescript
export interface McpResourceContent {
  uri: string
  mimeType?: string
  text?: string
  blob?: string
}
```

*   `McpResourceContent`: Represents the content of an MCP resource.
    *   `uri`: The URI of the resource.
    *   `mimeType`: The MIME type of the resource.
    *   `text`: The text content of the resource.
    *   `blob`: The binary content of the resource.

**8. MCP Prompt Types**

These interfaces define structures related to prompts.

```typescript
export interface McpPrompt {
  name: string
  description?: string
  arguments?: Array<{
    name: string
    description?: string
    required?: boolean
  }>
}
```

*   `McpPrompt`: Represents an MCP prompt.
    *   `name`: The name of the prompt.
    *   `description`: An optional description of the prompt.
    *   `arguments`: An optional array of arguments for the prompt.
        *   `name`: The name of the argument.
        *   `description`: An optional description of the argument.
        *   `required`: A boolean indicating whether the argument is required.

```typescript
export interface McpPromptMessage {
  role: 'user' | 'assistant'
  content: {
    type: 'text' | 'image' | 'resource'
    text?: string
    data?: string
    mimeType?: string
  }
}
```

*   `McpPromptMessage`: Represents a message within a prompt exchange.
    *   `role`: The role of the message sender (user or assistant).
    *   `content`: The content of the message.
        *   `type`: The type of content (text, image, or resource).
        *   `text`: The text content (if the type is "text").
        *   `data`: The data content (if the type is "image" or "resource").
        *   `mimeType`: The MIME type of the data (if the type is "image" or "resource").

**9. Connection and Error Types**

These interfaces and classes define structures related to connection status and error handling.

```typescript
export interface McpConnectionStatus {
  connected: boolean
  lastConnected?: Date
  lastError?: string
  serverInfo?: McpInitializeResult['serverInfo']
}
```

*   `McpConnectionStatus`: Represents the connection status to an MCP server.
    *   `connected`: A boolean indicating whether the client is currently connected to the server.
    *   `lastConnected`: An optional timestamp indicating when the client was last connected to the server.
    *   `lastError`: An optional error message indicating the last error that occurred.
    *   `serverInfo`: Optional server information obtained during initialization (`McpInitializeResult['serverInfo']`).

```typescript
export class McpError extends Error {
  constructor(
    message: string,
    public code?: number,
    public data?: any
  ) {
    super(message)
    this.name = 'McpError'
  }
}
```

*   `McpError`: A custom error class that extends the built-in `Error` class. It provides a base class for MCP-specific errors.
    *   `message`: The error message.
    *   `code`: An optional error code.
    *   `data`: Optional data associated with the error.
    *   The constructor calls the `super(message)` constructor of the `Error` class to initialize the message, and sets the `name` property to "McpError".

```typescript
export class McpConnectionError extends McpError {
  constructor(message: string, serverId: string) {
    super(`MCP Connection Error for server ${serverId}: ${message}`)
    this.name = 'McpConnectionError'
  }
}
```

*   `McpConnectionError`: A custom error class that extends `McpError` to represent connection-related errors.
    *   It constructs the error message with specific information on which server the error occurred.

```typescript
export class McpTimeoutError extends McpError {
  constructor(serverId: string, timeout: number) {
    super(`MCP request to server ${serverId} timed out after ${timeout}ms`)
    this.name = 'McpTimeoutError'
  }
}
```

*   `McpTimeoutError`: A custom error class that extends `McpError` to represent timeout-related errors.
    *   It constructs the error message to include the specific server and the timeout duration.

**10. Integration Types (for existing platform)**

These interfaces are intended to facilitate the integration of MCP into existing platforms.

```typescript
export interface McpToolInput {
  type: 'mcp'
  serverId: string
  toolName: string
  params: Record<string, any>
  usageControl?: 'auto' | 'force' | 'none'
}
```

*   `McpToolInput`:  Represents the input required to execute an MCP tool within an existing platform.
    *   `type`:  A string literal 'mcp' indicating that this input is specifically for an MCP tool.
    *   `serverId`: The ID of the MCP server hosting the tool.
    *   `toolName`: The name of the MCP tool to execute.
    *   `params`:  A dictionary of parameters to pass to the tool.
    *   `usageControl`: An optional string controlling how usage is handled. It can be 'auto', 'force', or 'none'.

```typescript
export interface McpServerSummary {
  id: string
  name: string
  url?: string
  transport?: McpTransport
  status: 'connected' | 'disconnected' | 'error'
  toolCount: number
  resourceCount?: number
  promptCount?: number
  lastSeen?: Date
  error?: string
}
```

*   `McpServerSummary`: Represents a summary of an MCP server's status and capabilities.
    *   `id`: The ID of the server.
    *   `name`: The name of the server.
    *   `url`: The URL of the server.
    *   `transport`: The transport protocol used by the server.
    *   `status`: The connection status of the server (connected, disconnected, or error).
    *   `toolCount`: The number of tools available on the server.
    *   `resourceCount`: The number of resources available on the server.
    *   `promptCount`: The number of prompts available on the server.
    *   `lastSeen`: The last time the server was seen.
    *   `error`: An optional error message.

**11. API Response Types**

These interfaces define the structure of API responses.

```typescript
export interface McpApiResponse<T = any> {
  success: boolean
  data?: T
  error?: string
}
```

*   `McpApiResponse`: Represents a standard API response.  It's a generic interface.
    *   `success`: A boolean indicating whether the API call was successful.
    *   `data`: Optional data returned by the API call.
    *   `error`: An optional error message.

```typescript
export interface McpToolDiscoveryResponse {
  tools: McpTool[]
  totalCount: number
  byServer: Record<string, number>
}
```

*   `McpToolDiscoveryResponse`: Represents the response from a tool discovery API.
    *   `tools`: An array of `McpTool` objects representing the discovered tools.
    *   `totalCount`: The total number of tools discovered.
    *   `byServer`: A dictionary mapping server IDs to the number of tools available on that server.

### Summary

In essence, this TypeScript file defines all the necessary data structures (types and interfaces) for implementing the Model Context Protocol (MCP). These structures cover everything from the underlying JSON-RPC protocol to the specific data models for tools, resources, prompts, security, and server configuration.  The file is designed to promote interoperability between MCP clients and servers by providing a common, well-defined type system.
