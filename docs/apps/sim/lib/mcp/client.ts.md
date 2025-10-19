```typescript
/**
 * MCP (Model Context Protocol) JSON-RPC 2.0 Client
 *
 * Implements the client side of MCP protocol with support for:
 * - Streamable HTTP transport (MCP 2025-03-26)
 * - Connection lifecycle management
 * - Tool execution and discovery
 * - Session management with Mcp-Session-Id header
 */

import { createLogger } from '@/lib/logs/console/logger'
import {
  type JsonRpcRequest,
  type JsonRpcResponse,
  type McpCapabilities,
  McpConnectionError,
  type McpConnectionStatus,
  type McpConsentRequest,
  type McpConsentResponse,
  McpError,
  type McpInitializeParams,
  type McpInitializeResult,
  type McpSecurityPolicy,
  type McpServerConfig,
  McpTimeoutError,
  type McpTool,
  type McpToolCall,
  type McpToolResult,
  type McpVersionInfo,
} from '@/lib/mcp/types'

// Creates a logger instance specifically for this `McpClient` class.  This allows for easier debugging and filtering of logs related to the MCP client.
const logger = createLogger('McpClient')

// Defines the `McpClient` class, which is the core of this file.  This class is responsible for managing the connection to an MCP server, sending requests, and handling responses.
export class McpClient {
  // Configuration object for the MCP server.  It includes information like the server's URL, transport type (HTTP, SSE, etc.), and any custom headers.
  private config: McpServerConfig
  // Represents the current connection status to the MCP server.  It includes whether the client is connected, the last connection time, and any error messages.
  private connectionStatus: McpConnectionStatus
  // A counter used to generate unique IDs for each JSON-RPC request. This helps in matching requests with their corresponding responses.
  private requestId = 0
  // A map that stores pending JSON-RPC requests. The key is the request ID, and the value is an object containing the `resolve` and `reject` functions of the promise associated with the request, along with a timeout.
  private pendingRequests = new Map<
    string | number,
    {
      resolve: (value: JsonRpcResponse) => void
      reject: (error: Error) => void
      timeout: NodeJS.Timeout
    }
  >()
  // Stores the capabilities of the MCP server, which are retrieved during the initialization process.  Capabilities describe the features and functionalities supported by the server (e.g., tool listing, resource subscription).
  private serverCapabilities?: McpCapabilities
  // Stores the MCP session ID, which is used to maintain a session with the server. The server provides this ID in the `Mcp-Session-Id` header.
  private mcpSessionId?: string
  // Stores the negotiated protocol version between the client and the server.  This ensures that both parties are using a compatible version of the MCP protocol.
  private negotiatedVersion?: string
  // Defines the security policy for the MCP client, including whether user consent is required for tool execution, the audit level, and any blocked origins.
  private securityPolicy: McpSecurityPolicy

  // An array of supported MCP protocol versions, ordered from the latest stable version to the oldest. This is used for version negotiation with the server.
  private static readonly SUPPORTED_VERSIONS = [
    '2025-06-18', // Latest stable with elicitation and OAuth 2.1
    '2025-03-26', // Streamable HTTP support
    '2024-11-05', // Initial stable release
  ]

  // Constructor for the `McpClient` class. It takes the server configuration and an optional security policy as arguments.
  constructor(config: McpServerConfig, securityPolicy?: McpSecurityPolicy) {
    this.config = config
    this.connectionStatus = { connected: false }

    // If no security policy is provided, a default policy is used that requires consent, sets the audit level to 'basic', and limits tool executions to 1000 per hour.
    this.securityPolicy = securityPolicy ?? {
      requireConsent: true,
      auditLevel: 'basic',
      maxToolExecutionsPerHour: 1000,
    }
  }

  /**
   * Initialize connection to MCP server
   */
  // The `connect` method establishes a connection to the MCP server based on the configured transport type.
  async connect(): Promise<void> {
    // Logs an informational message indicating that the client is attempting to connect to the server.
    logger.info(`Connecting to MCP server: ${this.config.name} (${this.config.transport})`)

    try {
      // A switch statement that determines the connection method based on the `transport` property in the `config` object.
      switch (this.config.transport) {
        // If the transport is 'http', 'sse', or 'streamable-http', the `connectStreamableHttp` method is called.
        case 'http':
        case 'sse':
        case 'streamable-http':
          await this.connectStreamableHttp()
          break
        // If the transport is not supported, an `McpError` is thrown.
        default:
          throw new McpError(`Unsupported transport: ${this.config.transport}`)
      }

      // After establishing the connection, the `initialize` method is called to negotiate the protocol version and retrieve the server's capabilities.
      await this.initialize()
      // Updates the connection status to `connected: true` and records the last connection time.
      this.connectionStatus.connected = true
      this.connectionStatus.lastConnected = new Date()

      // Logs a success message indicating that the client has successfully connected to the MCP server.
      logger.info(`Successfully connected to MCP server: ${this.config.name}`)
    } catch (error) {
      // If an error occurs during the connection process, the error message is extracted.
      const errorMessage = error instanceof Error ? error.message : 'Unknown error'
      // The error message is stored in the `connectionStatus` object.
      this.connectionStatus.lastError = errorMessage
      // An error message is logged, including the error details.
      logger.error(`Failed to connect to MCP server ${this.config.name}:`, error)
      // An `McpConnectionError` is thrown, providing more context about the connection failure.
      throw new McpConnectionError(errorMessage, this.config.id)
    }
  }

  /**
   * Disconnect from MCP server
   */
  // The `disconnect` method closes the connection to the MCP server.
  async disconnect(): Promise<void> {
    // Logs an informational message indicating that the client is disconnecting from the server.
    logger.info(`Disconnecting from MCP server: ${this.config.name}`)

    // Iterates over all pending requests and rejects them with an `McpError`, indicating that the connection has been closed.
    for (const [, pending] of this.pendingRequests) {
      clearTimeout(pending.timeout)
      pending.reject(new McpError('Connection closed'))
    }
    // Clears the `pendingRequests` map.
    this.pendingRequests.clear()

    // Updates the connection status to `connected: false`.
    this.connectionStatus.connected = false
    // Logs an informational message indicating that the client has disconnected from the MCP server.
    logger.info(`Disconnected from MCP server: ${this.config.name}`)
  }

  /**
   * Get current connection status
   */
  // The `getStatus` method returns the current connection status.
  getStatus(): McpConnectionStatus {
    // Returns a copy of the `connectionStatus` object to prevent external modification of the internal state.
    return { ...this.connectionStatus }
  }

  /**
   * List all available tools from the server
   */
  // The `listTools` method retrieves a list of available tools from the MCP server.
  async listTools(): Promise<McpTool[]> {
    // Checks if the client is currently connected to the server. If not, it throws an `McpConnectionError`.
    if (!this.connectionStatus.connected) {
      throw new McpConnectionError('Not connected to server', this.config.id)
    }

    try {
      // Sends a 'tools/list' request to the server using the `sendRequest` method.
      const response = await this.sendRequest('tools/list', {})

      // Validates the response to ensure that it contains a `tools` property that is an array.
      if (!response.tools || !Array.isArray(response.tools)) {
        // If the response is invalid, a warning message is logged, and an empty array is returned.
        logger.warn(`Invalid tools response from server ${this.config.name}:`, response)
        return []
      }

      // Maps the response data to an array of `McpTool` objects.
      return response.tools.map((tool: any) => ({
        name: tool.name,
        description: tool.description,
        inputSchema: tool.inputSchema,
        serverId: this.config.id,
        serverName: this.config.name,
      }))
    } catch (error) {
      // If an error occurs during the process, an error message is logged, and the error is re-thrown.
      logger.error(`Failed to list tools from server ${this.config.name}:`, error)
      throw error
    }
  }

  /**
   * Execute a tool on the MCP server
   */
  // The `callTool` method executes a specific tool on the MCP server.
  async callTool(toolCall: McpToolCall): Promise<McpToolResult> {
    // Checks if the client is currently connected to the server. If not, it throws an `McpConnectionError`.
    if (!this.connectionStatus.connected) {
      throw new McpConnectionError('Not connected to server', this.config.id)
    }

    // Request consent for tool execution
    // Creates an `McpConsentRequest` object to request user consent for executing the tool.
    const consentRequest: McpConsentRequest = {
      type: 'tool_execution',
      context: {
        serverId: this.config.id,
        serverName: this.config.name,
        action: toolCall.name,
        description: `Execute tool '${toolCall.name}' on ${this.config.name}`,
        dataAccess: Object.keys(toolCall.arguments || {}),
        sideEffects: ['tool_execution'],
      },
      expires: Date.now() + 5 * 60 * 1000, // 5 minute consent window
    }

    // Sends the consent request using the `requestConsent` method.
    const consentResponse = await this.requestConsent(consentRequest)
    // Checks if consent was granted. If not, it throws an `McpError`.
    if (!consentResponse.granted) {
      throw new McpError(`User consent denied for tool execution: ${toolCall.name}`, -32000, {
        consentAuditId: consentResponse.auditId,
      })
    }

    try {
      // Logs an informational message indicating that the tool is being called, including the consent audit ID and protocol version.
      logger.info(`Calling tool ${toolCall.name} on server ${this.config.name}`, {
        consentAuditId: consentResponse.auditId,
        protocolVersion: this.negotiatedVersion,
      })

      // Sends a 'tools/call' request to the server using the `sendRequest` method, including the tool name and arguments.
      const response = await this.sendRequest('tools/call', {
        name: toolCall.name,
        arguments: toolCall.arguments,
      })

      // The response is the JSON-RPC 'result' field
      // Returns the response as an `McpToolResult`.
      return response as McpToolResult
    } catch (error) {
      // If an error occurs during the process, an error message is logged, and the error is re-thrown.
      logger.error(`Failed to call tool ${toolCall.name} on server ${this.config.name}:`, error)
      throw error
    }
  }

  /**
   * Send a JSON-RPC request to the server
   */
  // The `sendRequest` method sends a generic JSON-RPC request to the MCP server.
  private async sendRequest(method: string, params: any): Promise<any> {
    // Increments the `requestId` counter to generate a unique ID for the request.
    const id = ++this.requestId
    // Creates a `JsonRpcRequest` object with the method, parameters, and ID.
    const request: JsonRpcRequest = {
      jsonrpc: '2.0',
      id,
      method,
      params,
    }

    // Returns a new promise that resolves with the response from the server or rejects with an error.
    return new Promise((resolve, reject) => {
      // Sets a timeout for the request. If the server does not respond within the configured timeout period, the promise is rejected with an `McpTimeoutError`.
      const timeout = setTimeout(() => {
        this.pendingRequests.delete(id)
        reject(new McpTimeoutError(this.config.id, this.config.timeout || 30000))
      }, this.config.timeout || 30000)

      // Stores the `resolve`, `reject`, and `timeout` functions in the `pendingRequests` map, keyed by the request ID.
      this.pendingRequests.set(id, { resolve, reject, timeout })

      // Sends the HTTP request using the `sendHttpRequest` method and catches any errors that occur.
      this.sendHttpRequest(request).catch(reject)
    })
  }

  /**
   * Initialize connection with capability and version negotiation
   */
  // The `initialize` method performs the initial handshake with the MCP server, negotiating the protocol version and retrieving the server's capabilities.
  private async initialize(): Promise<void> {
    // Start with latest supported version for negotiation
    // Sets the preferred protocol version to the latest supported version.
    const preferredVersion = McpClient.SUPPORTED_VERSIONS[0]

    // Creates an `McpInitializeParams` object with the preferred protocol version, client capabilities, and client information.
    const initParams: McpInitializeParams = {
      protocolVersion: preferredVersion,
      capabilities: {
        tools: { listChanged: true },
        resources: { subscribe: true, listChanged: true },
        prompts: { listChanged: true },
        logging: { level: 'info' },
      },
      clientInfo: {
        name: 'sim-platform',
        version: '1.0.0',
      },
    }

    try {
      // Sends the 'initialize' request to the server using the `sendRequest` method.
      const result: McpInitializeResult = await this.sendRequest('initialize', initParams)

      // Handle version negotiation
      // Checks if the server proposed a different protocol version.
      if (result.protocolVersion !== preferredVersion) {
        // Server proposed a different version - check if we support it
        // Checks if the client supports the proposed version.
        if (!McpClient.SUPPORTED_VERSIONS.includes(result.protocolVersion)) {
          // Client SHOULD disconnect if it cannot support proposed version
          // If the client does not support the proposed version, it throws an `McpError`.
          throw new McpError(
            `Version negotiation failed: Server proposed unsupported version '${result.protocolVersion}'. ` +
              `This client supports versions: ${McpClient.SUPPORTED_VERSIONS.join(', ')}. ` +
              `To use this server, you may need to update your client or find a compatible version of the server.`
          )
        }

        // Logs an informational message indicating that the server proposed a different version and that the client is using the server's version.
        logger.info(
          `Version negotiation: Server proposed version '${result.protocolVersion}' ` +
            `instead of requested '${preferredVersion}'. Using server version.`
        )
      }

      // Stores the negotiated protocol version and server capabilities.
      this.negotiatedVersion = result.protocolVersion
      this.serverCapabilities = result.capabilities

      // Logs an informational message indicating that the MCP initialization was successful, including the negotiated protocol version.
      logger.info(`MCP initialization successful with protocol version '${this.negotiatedVersion}'`)
    } catch (error) {
      // Enhanced error handling
      // If the error is an `McpError`, it is re-thrown.
      if (error instanceof McpError) {
        throw error // Re-throw MCP errors as-is
      }

      // Handle network errors
      // If the error is a network error (e.g., fetch error, timeout), a more specific `McpError` is thrown.
      if (error instanceof Error) {
        if (error.message.includes('fetch') || error.message.includes('network')) {
          throw new McpError(
            `Failed to connect to MCP server '${this.config.name}': ${error.message}. ` +
              `Please check the server URL and ensure the server is running.`
          )
        }

        if (error.message.includes('timeout')) {
          throw new McpError(
            `Connection timeout to MCP server '${this.config.name}'. ` +
              `The server may be slow to respond or unreachable.`
          )
        }

        // Generic error
        // If the error is a generic error, a more general `McpError` is thrown.
        throw new McpError(
          `Connection to MCP server '${this.config.name}' failed: ${error.message}. ` +
            `Please verify the server configuration and try again.`
        )
      }

      // If the error is an unexpected error, an `McpError` is thrown with a generic message.
      throw new McpError(`Unexpected error during MCP initialization: ${String(error)}`)
    }

    // Sends a 'notifications/initialized' notification to the server to indicate that the client has been initialized.
    await this.sendNotification('notifications/initialized', {})
  }

  /**
   * Send a notification
   */
  // The `sendNotification` method sends a JSON-RPC notification to the server. Notifications are similar to requests, but they do not expect a response.
  private async sendNotification(method: string, params: any): Promise<void> {
    // Creates a notification object with the method and parameters.  Notifications do not have an `id` field.
    const notification = {
      jsonrpc: '2.0' as const,
      method,
      params,
    }

    // Sends the HTTP request using the `sendHttpRequest` method.
    await this.sendHttpRequest(notification)
  }

  /**
   * Connect using Streamable HTTP transport
   */
  // The `connectStreamableHttp` method establishes a connection to the MCP server using the Streamable HTTP transport.  This transport is used for streaming data between the client and server.
  private async connectStreamableHttp(): Promise<void> {
    // Checks if the `url` property is defined in the `config` object. If not, it throws an `McpError`.
    if (!this.config.url) {
      throw new McpError('URL required for Streamable HTTP transport')
    }

    // Logs an informational message indicating that the client is using the Streamable HTTP transport.
    logger.info(`Using Streamable HTTP transport for ${this.config.name}`)
  }

  /**
   * Send HTTP request with automatic retry
   */
  // The `sendHttpRequest` method sends an HTTP request to the MCP server. It also includes retry logic to handle potential URL format variations.
  private async sendHttpRequest(request: JsonRpcRequest | any): Promise<void> {
    // Checks if the `url` property is defined in the `config` object. If not, it throws an `McpError`.
    if (!this.config.url) {
      throw new McpError('URL required for HTTP transport')
    }

    // Creates an array of URLs to try. It includes the original URL and, if necessary, variations with and without a trailing slash.
    const urlsToTry = [this.config.url]
    if (!this.config.url.endsWith('/')) {
      urlsToTry.push(`${this.config.url}/`)
    } else {
      urlsToTry.push(this.config.url.slice(0, -1))
    }

    // Initializes a variable to store the last error encountered during the retry process.
    let lastError: Error | null = null
    // Stores the original URL for logging purposes.
    const originalUrl = this.config.url

    // Iterates over the `urlsToTry` array and attempts to send the HTTP request to each URL.
    for (const [index, url] of urlsToTry.entries()) {
      try {
        // Attempts to send the HTTP request using the `attemptHttpRequest` method.
        await this.attemptHttpRequest(request, url, index === 0)

        // If the request is successful, logs an informational message indicating that an alternative URL format was used.
        if (index > 0) {
          logger.info(
            `[${this.config.name}] Successfully used alternative URL format: ${url} (original: ${originalUrl})`
          )
        }
        return
      } catch (error) {
        // If an error occurs, stores the error in the `lastError` variable.
        lastError = error as Error

        // If the error is an `McpError` and does not indicate a "404 Not Found" error, breaks out of the loop.
        if (error instanceof McpError && !error.message.includes('404')) {
          break
        }

        // If there are more URLs to try, logs an informational message indicating that the client is retrying with a different URL format.
        if (index < urlsToTry.length - 1) {
          logger.info(
            `[${this.config.name}] Retrying with different URL format: ${urlsToTry[index + 1]}`
          )
        }
      }
    }

    // If all URL variations fail, throws the last error encountered or a generic `McpError`.
    throw lastError || new McpError('All URL variations failed')
  }

  /**
   * Attempt HTTP request
   */
  // The `attemptHttpRequest` method attempts to send an HTTP request to the specified URL.
  private async attemptHttpRequest(
    request: JsonRpcRequest | any,
    url: string,
    isOriginalUrl = true
  ): Promise<void> {
    // If the URL is not the original URL, logs an informational message indicating that the client is trying an alternative URL format.
    if (!isOriginalUrl) {
      logger.info(`[${this.config.name}] Trying alternative URL format: ${url}`)
    }

    // Creates a `headers` object with the necessary headers, including the `Content-Type`, `Accept`, custom headers from the configuration, and the `Mcp-Session-Id` if it exists.
    const headers: Record<string, string> = {
      'Content-Type': 'application/json',
      Accept: 'application/json, text/event-stream',
      ...this.config.headers,
    }

    // If the MCP session ID exists, add it to the headers.
    if (this.mcpSessionId) {
      headers['Mcp-Session-Id'] = this.mcpSessionId
    }

    // Sends the HTTP request using the `fetch` API.
    const response = await fetch(url, {
      method: 'POST',
      headers,
      body: JSON.stringify(request),
    })

    // Checks if the response status is OK (200-299). If not, it throws an `McpError`.
    if (!response.ok) {
      // Attempts to read the response text.
      const responseText = await response.text().catch(() => 'Could not read response body')
      // Logs an error message including the status code, status text, URL, and a truncated version of the response body.
      logger.error(`[${this.config.name}] HTTP request failed:`, {
        status: response.status,
        statusText: response.statusText,
        url,
        responseBody: responseText.substring(0, 500),
      })
      // Throws an `McpError` indicating that the HTTP request failed.
      throw new McpError(`HTTP request failed: ${response.status} ${response.statusText}`)
    }

    // Checks if the request was a JSON-RPC request (i.e., it has an `id` property).
    if ('id' in request) {
      // Gets the `Content-Type` header from the response.
      const contentType = response.headers.get('Content-Type')

      // If the content type is 'application/json', it parses the response as JSON and calls the `handleResponse` method.
      if (contentType?.includes('application/json')) {
        // Extracts the MCP session ID from the response headers.
        const sessionId = response.headers.get('Mcp-Session-Id')
        // If a new session ID is received, stores it and logs an informational message.
        if (sessionId && !this.mcpSessionId) {
          this.mcpSessionId = sessionId
          logger.info(`[${this.config.name}] Received MCP Session ID: ${sessionId}`)
        }

        const responseData: JsonRpcResponse = await response.json()
        this.handleResponse(responseData)
        // If the content type is 'text/event-stream', it parses the response as Server-Sent Events and calls the `handleSseResponse` method.
      } else if (contentType?.includes('text/event-stream')) {
        const responseText = await response.text()
        this.handleSseResponse(responseText, request.id)
        // If the content type is neither 'application/json' nor 'text/event-stream', it logs a warning message and throws an `McpError`.
      } else {
        // Logs a warning message with unexpected content type.
        const unexpectedType = contentType || 'unknown'
        logger.warn(`[${this.config.name}] Unexpected response content type: ${unexpectedType}`)

        // Logs the first 200 characters of the unexpected response body.
        const responseText = await response.text()
        logger.debug(
          `[${this.config.name}] Unexpected response body:`,
          responseText.substring(0, 200)
        )

        throw new McpError(
          `Unexpected response content type: ${unexpectedType}. Expected application/json or text/event-stream.`
        )
      }
    }
  }

  /**
   * Handle JSON-RPC response
   */
  // The `handleResponse` method handles a JSON-RPC response from the server.
  private handleResponse(response: JsonRpcResponse): void {
    // Retrieves the pending request information from the `pendingRequests` map using the response ID.
    const pending = this.pendingRequests.get(response.id)
    // If there is no pending request for the given ID, it logs a warning message and returns.
    if (!pending) {
      logger.warn(`Received response for unknown request ID: ${response.id}`)
      return
    }

    // Deletes the pending request from the `pendingRequests` map.
    this.pendingRequests.delete(response.id)
    // Clears the timeout associated with the pending request.
    clearTimeout(pending.timeout)

    // If the response contains an error, it creates an `McpError` and rejects the promise associated with the pending request.
    if (response.error) {
      const error = new McpError(response.error.message, response.error.code, response.error.data)
      pending.reject(error)
      // If the response is successful, it resolves the promise associated with the pending request with the response result.
    } else {
      pending.resolve(response.result)
    }
  }

  /**
   * Handle Server-Sent Events response format
   */
  // The `handleSseResponse` method handles a Server-Sent Events (SSE) response from the server.
  private handleSseResponse(responseText: string, requestId: string | number): void {
    // Retrieves the pending request information from the `pendingRequests` map using the request ID.
    const pending = this.pendingRequests.get(requestId)
    // If there is no pending request for the given ID, it logs a warning message and returns.
    if (!pending) {
      logger.warn(`Received SSE response for unknown request ID: ${requestId}`)
      return
    }

    try {
      // Parse SSE format - look for data: lines
      // Splits the response text into lines.
      const lines = responseText.split('\n')
      // Initializes an empty string to store the JSON data.
      let jsonData = ''

      // Iterates over the lines and extracts the data from the 'data: ' lines.
      for (const line of lines) {
        if (line.startsWith('data: ')) {
          const data = line.substring(6).trim()
          if (data && data !== '[DONE]') {
            jsonData += data
          }
        }
      }

      // If no data is found in the SSE response, it logs an error message and rejects the promise associated with the pending request.
      if (!jsonData) {
        logger.error(
          `[${this.config.name}] No valid data found in SSE response for request ${requestId}`
        )
        pending.reject(new McpError('No data in SSE response'))
        return
      }

      // Parse the JSON data
      // Parses the JSON data from the SSE response.
      const responseData: JsonRpcResponse = JSON.parse(jsonData)

      // Deletes the pending request from the `pendingRequests` map.
      this.pendingRequests.delete(requestId)
      // Clears the timeout associated with the pending request.
      clearTimeout(pending.timeout)

      // If the response contains an error, it creates an `McpError` and rejects the promise associated with the pending request.
      if (responseData.error) {
        const error = new McpError(
          responseData.error.message,
          responseData.error.code,
          responseData.error.data
        )
        pending.reject(error)
        // If the response is successful, it resolves the promise associated with the pending request with the response result.
      } else {
        pending.resolve(responseData.result)
      }
    } catch (error) {
      // Logs an error message including the error details and a truncated version of the response text.
      logger.error(`[${this.config.name}] Failed to parse SSE response for request ${requestId}:`, {
        error: error instanceof Error ? error.message : 'Unknown error',
        responseText: responseText.substring(0, 500),
      })

      // Deletes the pending request from the `pendingRequests` map.
      this.pendingRequests.delete(requestId)
      // Clears the timeout associated with the pending request.
      clearTimeout(pending.timeout)
      pending.reject(new McpError('Failed to parse SSE response'))
    }
  }

  /**
   * Check if server has capability
   */
  // The `hasCapability` method checks if the server has the specified capability.
  hasCapability(capability: keyof McpCapabilities): boolean {
    // Returns true if the server capabilities object exists and the specified capability is defined (and truthy) in the object.
    return !!this.serverCapabilities?.[capability]
  }

  /**
   * Get server configuration
   */
  // The `getConfig` method returns the server configuration.
  getConfig(): McpServerConfig {
    // Returns a copy of the `config` object to prevent external modification of the internal state.
    return { ...this.config }
  }

  /**
   * Get version information for this client
   */
  // The `getVersionInfo` method returns the version information for the client.
  static getVersionInfo(): McpVersionInfo {
    // Returns an object containing the list of supported protocol versions and the preferred version.
    return {
      supported: [...McpClient.SUPPORTED_VERSIONS],
      preferred: McpClient.SUPPORTED_VERSIONS[0],
    }
  }

  /**
   * Get the negotiated protocol version for this connection
   */
  // The `getNegotiatedVersion` method returns the negotiated protocol version for the connection.
  getNegotiatedVersion(): string | undefined {
    // Returns the `negotiatedVersion` property.
    return this.negotiatedVersion
  }

  /**
   * Request user consent for tool execution
   */
  // The `requestConsent` method requests user consent for tool execution.
  async requestConsent(consentRequest: McpConsentRequest): Promise<McpConsentResponse> {
    // If consent is not required according to the security policy, it returns a granted consent response.
    if (!this.securityPolicy.requireConsent) {
      return { granted: true, auditId: `audit-${Date.now()}` }
    }

    // Basic security checks
    // Extracts the server ID, server name, action, and side effects from the consent request context.
    const { serverId, serverName, action, sideEffects } = consentRequest.context

    // Check if server is in blocked
    // Checks if the server URL is in the list of blocked origins in the security policy.
    if (this.securityPolicy.blockedOrigins?.includes(this.config.url || '')) {
      // If the server is blocked, it logs a warning message and returns a denied consent response.
      logger.warn(`Tool execution blocked: Server ${serverName} is in blocked origins`)
      return {
        granted: false,
        auditId: `audit-blocked-${Date.now()}`,
      }
    }

    // For high-risk operations, log detailed audit
    // If the audit level is set to 'detailed', it logs a detailed audit message.
    if (this.securityPolicy.auditLevel === 'detailed') {
      logger.info(`Consent requested for ${action} on ${serverName}`, {
        serverId,
        action,
        sideEffects,
        timestamp: new Date().toISOString(),
      })
    }

    // Returns a granted consent response with an audit ID and expiration time.
    return {
      granted: true,
      expires: consentRequest.expires,
      auditId: `audit-${serverId}-${Date.now()}`,
    }
  }
}
```

**Purpose of this file:**

This file defines the `McpClient` class, which is a TypeScript client for interacting with an MCP (Model Context Protocol) server using JSON-RPC 2.0. The client handles connection management, request sending, response handling (including Server-Sent Events), version negotiation, capability discovery, and consent management. It provides a higher-level abstraction for interacting with an MCP server, simplifying tasks like listing available tools and executing them.

**Simplifying Complex Logic:**

- **Connection Management:** The `connect` and `disconnect` methods encapsulate the logic for establishing and closing connections to the MCP server, handling different transport types (HTTP, SSE, Streamable HTTP).
- **Request/Response Handling:** The `sendRequest` method manages the process of sending JSON-RPC requests, setting timeouts, and handling