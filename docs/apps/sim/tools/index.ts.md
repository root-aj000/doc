This TypeScript file is a core component responsible for executing various "tools" within an application. These tools can represent anything from calling external APIs (like social media platforms, project management tools, etc.) to interacting with internal services or even custom, user-defined functions.

The file's primary goal is to provide a unified, robust, and secure way to invoke these tools, handling concerns such as:
1.  **Tool Discovery:** Finding the correct tool configuration based on an ID.
2.  **Parameter Management:** Formatting and validating input parameters for the tool.
3.  **Authentication:** Handling credential management and token acquisition for secure access to APIs.
4.  **Execution Strategy:** Determining whether to call an internal API directly or route the request through a proxy for external services.
5.  **Error Handling:** Capturing and standardizing error responses from diverse APIs.
6.  **Post-processing:** Applying custom logic after a tool's primary execution.
7.  **File Output Handling:** Processing any file-related outputs a tool might generate.
8.  **Logging and Timing:** Providing insights into tool execution.

Essentially, it acts as an **orchestrator for tool invocations**, abstracting away the complexities of dealing with different tool types and external integrations.

---

### Simplified Complex Logic

The most complex part of this file is the `executeTool` function, which acts as the main entry point. Here's a simplified breakdown of its logic:

1.  **Start Timing & ID:** Records when the execution begins and generates a unique request ID for logging.
2.  **Find the Tool:**
    *   If the `toolId` starts with `custom_`, it fetches a *custom tool* definition, potentially from a database or a workflow.
    *   If `toolId` starts with `mcp-`, it's an **MCP (Microservice Control Plane) tool**, which has its own specialized execution path (`executeMcpTool`).
    *   Otherwise, it's a *built-in tool* and fetches its definition synchronously.
    *   **Crucially, if the tool isn't found, it throws an error.**
3.  **Validate Parameters:** It checks if all required parameters for the found tool are present.
4.  **Handle Credentials (OAuth/Authentication):**
    *   If the tool's parameters include a `credential` (e.g., an OAuth connection ID), it makes an internal API call to `/api/auth/oauth/token` to fetch an `accessToken` using that credential.
    *   This ensures that sensitive credential details are not passed directly to tools or proxies.
5.  **Determine Execution Path:**
    *   **Internal Route/Skip Proxy:** If the tool's configured URL starts with `/api/` (indicating an internal service) or `skipProxy` is explicitly set to `true`, it calls the `handleInternalRequest` function. This function makes a direct `fetch` call to the internal service.
    *   **External API (via Proxy):** Otherwise (for external APIs), it calls the `handleProxyRequest` function. This function sends a request to a dedicated `/api/proxy` endpoint, which then safely forwards the request to the external API.
6.  **Post-Processing:** After the tool executes, if the tool configuration has a `postProcess` function and the execution was successful, it runs this function to potentially modify or enhance the tool's output.
7.  **File Output Processing:** If the tool's configuration indicates it might produce files and an `executionContext` is available, it attempts to process these file outputs (e.g., upload them).
8.  **Final Result & Timing:** It combines the tool's output, success/error status, and records the total execution time.
9.  **Centralized Error Handling:** Any errors encountered during this entire process are caught, logged, and transformed into a standardized `ToolResponse` object indicating failure, with a clear error message.

This structure allows the system to manage a diverse set of tools consistently, regardless of their underlying implementation or external dependencies.

---

### Line-by-Line Explanation

```typescript
import { generateInternalToken } from '@/lib/auth/internal'
import { createLogger } from '@/lib/logs/console/logger'
import { parseMcpToolId } from '@/lib/mcp/utils'
import { getBaseUrl } from '@/lib/urls/utils'
import { generateRequestId } from '@/lib/utils'
import type { ExecutionContext } from '@/executor/types'
import type { OAuthTokenPayload, ToolConfig, ToolResponse } from '@/tools/types'
import {
  formatRequestParams,
  getTool,
  getToolAsync,
  validateRequiredParametersAfterMerge,
} from '@/tools/utils'
```

*   `import { generateInternalToken } from '@/lib/auth/internal'`: Imports a utility function to generate an internal authentication token. This token is used for server-to-server communication within the application, ensuring internal API calls are authorized.
*   `import { createLogger } from '@/lib/logs/console/logger'`: Imports a function to create a logger instance. This allows for structured logging of events and errors within this file.
*   `import { parseMcpToolId } from '@/lib/mcp/utils'`: Imports a utility to parse an MCP (Microservice Control Plane) tool ID, extracting server and tool names.
*   `import { getBaseUrl } from '@/lib/urls/utils'`: Imports a function to retrieve the base URL of the application. This is crucial for constructing full URLs for internal API calls.
*   `import { generateRequestId } from '@/lib/utils'`: Imports a function to generate unique request IDs, useful for tracing and debugging.
*   `import type { ExecutionContext } from '@/executor/types'`: Imports a TypeScript type `ExecutionContext`. This type likely defines contextual information related to the ongoing execution (e.g., workflow ID, workspace ID).
*   `import type { OAuthTokenPayload, ToolConfig, ToolResponse } from '@/tools/types'`: Imports TypeScript types:
    *   `OAuthTokenPayload`: The structure of data sent when requesting an OAuth token.
    *   `ToolConfig`: The configuration object for a tool, defining its behavior (e.g., URL, method, parameters).
    *   `ToolResponse`: The standardized structure for the result of a tool execution (success/failure, output, error).
*   `import { formatRequestParams, getTool, getToolAsync, validateRequiredParametersAfterMerge } from '@/tools/utils'`: Imports several utility functions related to tools:
    *   `formatRequestParams`: Prepares tool parameters into a format suitable for `fetch` API requests.
    *   `getTool`: Retrieves a synchronous (usually built-in) tool configuration by its ID.
    *   `getToolAsync`: Retrieves an asynchronous (usually custom) tool configuration by its ID, potentially requiring a `workflowId`.
    *   `validateRequiredParametersAfterMerge`: Validates that all required parameters for a tool are present after merging any default/contextual parameters.

```typescript
const logger = createLogger('Tools')
```

*   `const logger = createLogger('Tools')`: Creates a logger instance specifically for the 'Tools' module. All log messages from this file will be associated with this logger, making it easier to filter logs.

```typescript
/**
 * System parameters that should be filtered out when extracting tool arguments
 * These are internal parameters used by the execution framework, not tool inputs
 */
const MCP_SYSTEM_PARAMETERS = new Set([
  'serverId',
  'toolName',
  'serverName',
  '_context',
  'envVars',
  'workflowVariables',
  'blockData',
  'blockNameMapping',
])
```

*   `const MCP_SYSTEM_PARAMETERS = new Set(...)`: Defines a `Set` of string literal names. These names represent internal system parameters that are passed along with tool execution requests but are *not* actual inputs to the MCP tool itself. They need to be filtered out before sending arguments to an MCP tool. Using a `Set` allows for efficient `has` checks.

```typescript
// Extract a concise, meaningful error message from diverse API error shapes
function getDeepApiErrorMessage(errorInfo?: {
  status?: number
  statusText?: string
  data?: any
}): string {
  return (
    // GraphQL errors (Linear API)
    errorInfo?.data?.errors?.[0]?.message ||
    // X/Twitter API specific pattern
    errorInfo?.data?.errors?.[0]?.detail ||
    // Generic details array
    errorInfo?.data?.details?.[0]?.message ||
    // Hunter API pattern
    errorInfo?.data?.errors?.[0]?.details ||
    // Direct errors array (when errors[0] is a string or simple object)
    (Array.isArray(errorInfo?.data?.errors)
      ? typeof errorInfo.data.errors[0] === 'string'
        ? errorInfo.data.errors[0]
        : errorInfo.data.errors[0]?.message
      : undefined) ||
    // Notion/Discord/GitHub/Twilio pattern
    errorInfo?.data?.message ||
    // SOAP/XML fault patterns
    errorInfo?.data?.fault?.faultstring ||
    errorInfo?.data?.faultstring ||
    // Microsoft/OAuth error descriptions
    errorInfo?.data?.error_description ||
    // Airtable/Google fallback pattern
    (typeof errorInfo?.data?.error === 'object'
      ? errorInfo?.data?.error?.message || JSON.stringify(errorInfo?.data?.error)
      : errorInfo?.data?.error) ||
    // HTTP status text fallback
    errorInfo?.statusText ||
    // Final fallback
    `Request failed with status ${errorInfo?.status || 'unknown'}`
  )
}
```

*   `function getDeepApiErrorMessage(...)`: This function attempts to extract a human-readable error message from various API error response formats. APIs often return errors in inconsistent structures (e.g., `data.errors[0].message`, `data.message`, `data.error.message`).
*   It uses a series of `||` (OR) operators to try different common paths to an error message. The first path that successfully finds a non-null/non-undefined value will be returned.
    *   It checks for `data.errors[0].message` (common in GraphQL like Linear)
    *   It checks for `data.errors[0].detail` (e.g., X/Twitter)
    *   It checks for `data.details[0].message` (generic details array)
    *   It checks for `data.errors[0].details` (e.g., Hunter API)
    *   It handles cases where `data.errors` is an array of strings or simple objects.
    *   It checks for `data.message` (common generic message, e.g., Notion, Discord).
    *   It checks for SOAP/XML fault strings.
    *   It checks for OAuth error descriptions.
    *   It handles `data.error` which can be an object with a message or a string.
    *   If all else fails, it falls back to the HTTP `statusText` or a generic "Request failed" message including the status code.

```typescript
// Create an Error instance from errorInfo and attach useful context
function createTransformedErrorFromErrorInfo(errorInfo?: {
  status?: number
  statusText?: string
  data?: any
}): Error {
  const message = getDeepApiErrorMessage(errorInfo)
  const transformed = new Error(message)
  Object.assign(transformed, {
    status: errorInfo?.status,
    statusText: errorInfo?.statusText,
    data: errorInfo?.data,
  })
  return transformed
}
```

*   `function createTransformedErrorFromErrorInfo(...)`: This helper function takes the raw error information (status, status text, and response data) and creates a standardized JavaScript `Error` object.
*   `const message = getDeepApiErrorMessage(errorInfo)`: It first extracts the most meaningful error message using the `getDeepApiErrorMessage` function.
*   `const transformed = new Error(message)`: A new `Error` instance is created with this message.
*   `Object.assign(transformed, { ... })`: Additional context like `status`, `statusText`, and the original `data` are attached directly to the `Error` object. This enriches the error with useful diagnostic information without needing to parse the message itself.
*   `return transformed`: The enriched `Error` object is returned.

```typescript
/**
 * Process file outputs for a tool result if execution context is available
 * Uses dynamic imports to avoid client-side bundling issues
 */
async function processFileOutputs(
  result: ToolResponse,
  tool: ToolConfig,
  executionContext?: ExecutionContext
): Promise<ToolResponse> {
  // Skip file processing if no execution context or not successful
  if (!executionContext || !result.success) {
    return result
  }

  // Skip file processing on client-side (no Node.js modules available)
  if (typeof window !== 'undefined') {
    return result
  }

  try {
    // Dynamic import to avoid client-side bundling issues
    const { FileToolProcessor } = await import('@/executor/utils/file-tool-processor')

    // Check if tool has file outputs
    if (!FileToolProcessor.hasFileOutputs(tool)) {
      return result
    }

    const processedOutput = await FileToolProcessor.processToolOutputs(
      result.output,
      tool,
      executionContext
    )

    return {
      ...result,
      output: processedOutput,
    }
  } catch (error) {
    logger.error(`Error processing file outputs for tool ${tool.id}:`, error)
    // Return original result if file processing fails
    return result
  }
}
```

*   `async function processFileOutputs(...)`: This function is designed to handle any file-related outputs a tool might produce. For example, if a tool generates a report file, this function might upload it to a storage service.
*   `if (!executionContext || !result.success)`: It immediately returns the original result if there's no `executionContext` (which holds information needed for file processing) or if the tool execution itself wasn't successful.
*   `if (typeof window !== 'undefined')`: This is a common pattern to detect if the code is running in a browser environment (`window` object exists) versus a Node.js environment (server-side). File processing often involves Node.js-specific modules, so it's skipped on the client-side to prevent bundling errors.
*   `const { FileToolProcessor } = await import(...)`: This uses a *dynamic import*. This means the `FileToolProcessor` module is only loaded when `processFileOutputs` is called, and only in a Node.js environment. This prevents the module (and its dependencies) from being included in client-side bundles, optimizing client-side performance.
*   `if (!FileToolProcessor.hasFileOutputs(tool))`: Checks if the specific `tool` configuration indicates that it's expected to produce file outputs. If not, it returns the original result.
*   `const processedOutput = await FileToolProcessor.processToolOutputs(...)`: If the tool has file outputs, it calls a method on `FileToolProcessor` to actually perform the processing (e.g., uploading files).
*   `return { ...result, output: processedOutput }`: If successful, it returns a new `ToolResponse` object with the original properties but an updated `output` containing the processed file information.
*   `catch (error)`: If any error occurs during file processing, it logs the error but returns the `original result`. This ensures that a failure in file processing doesn't necessarily block the overall tool execution result.

```typescript
// Execute a tool by calling either the proxy for external APIs or directly for internal routes
export async function executeTool(
  toolId: string,
  params: Record<string, any>,
  skipProxy = false,
  skipPostProcess = false,
  executionContext?: ExecutionContext
): Promise<ToolResponse> {
```

*   `export async function executeTool(...)`: This is the main exported function of this file, the core logic for executing any tool.
    *   `toolId: string`: The unique identifier of the tool to be executed.
    *   `params: Record<string, any>`: An object containing all the input parameters for the tool.
    *   `skipProxy = false`: A boolean flag. If `true`, the request will *not* be sent through the proxy, even if it's an external API. This is useful for internal testing or specific configurations. Defaults to `false`.
    *   `skipPostProcess = false`: A boolean flag. If `true`, any `postProcess` function defined for the tool will be skipped. Defaults to `false`.
    *   `executionContext?: ExecutionContext`: Optional context about the current execution environment (e.g., workflow ID, user ID).
    *   `Promise<ToolResponse>`: The function returns a `Promise` that resolves to a `ToolResponse` object, detailing the success or failure and the output/error.

```typescript
  // Capture start time for precise timing
  const startTime = new Date()
  const startTimeISO = startTime.toISOString()
  const requestId = generateRequestId()
```

*   `const startTime = new Date()`: Records the current time when the tool execution begins.
*   `const startTimeISO = startTime.toISOString()`: Converts the start time to an ISO 8601 string, a standard format for timestamps.
*   `const requestId = generateRequestId()`: Generates a unique ID for this specific tool execution request, used for logging and tracking.

```typescript
  try {
    let tool: ToolConfig | undefined
```

*   `try { ... }`: The entire tool execution logic is wrapped in a `try...catch` block to handle any errors gracefully and return a standardized `ToolResponse` in case of failure.
*   `let tool: ToolConfig | undefined`: Declares a variable `tool` which will hold the configuration object for the identified tool. It's initially `undefined`.

```typescript
    // If it's a custom tool, use the async version with workflowId
    if (toolId.startsWith('custom_')) {
      const workflowId = params._context?.workflowId
      tool = await getToolAsync(toolId, workflowId)
      if (!tool) {
        logger.error(`[${requestId}] Custom tool not found: ${toolId}`)
      }
    } else if (toolId.startsWith('mcp-')) {
      return await executeMcpTool(toolId, params, executionContext, requestId, startTimeISO)
    } else {
      // For built-in tools, use the synchronous version
      tool = getTool(toolId)
      if (!tool) {
        logger.error(`[${requestId}] Built-in tool not found: ${toolId}`)
      }
    }
```

*   This block determines which type of tool is being executed and retrieves its configuration.
*   `if (toolId.startsWith('custom_'))`: Checks if the `toolId` indicates a custom tool (e.g., defined by a user or a specific workflow).
    *   `const workflowId = params._context?.workflowId`: Extracts the `workflowId` from the parameters, as custom tools are often tied to specific workflows.
    *   `tool = await getToolAsync(toolId, workflowId)`: Calls the asynchronous `getToolAsync` function to retrieve the custom tool's configuration. This might involve a database lookup.
    *   `if (!tool)`: Logs an error if the custom tool isn't found.
*   `else if (toolId.startsWith('mcp-'))`: Checks if the `toolId` indicates an MCP tool.
    *   `return await executeMcpTool(...)`: If it's an MCP tool, it immediately calls the `executeMcpTool` function with all relevant parameters and returns its result directly. MCP tools have a distinct execution path.
*   `else`: If it's neither a custom nor an MCP tool, it's assumed to be a built-in tool.
    *   `tool = getTool(toolId)`: Calls the synchronous `getTool` function to retrieve the built-in tool's configuration.
    *   `if (!tool)`: Logs an error if the built-in tool isn't found.

```typescript
    // Ensure context is preserved if it exists
    const contextParams = { ...params }

    // Validate the tool and its parameters
    validateRequiredParametersAfterMerge(toolId, tool, contextParams)

    // After validation, we know tool exists
    if (!tool) {
      throw new Error(`Tool not found: ${toolId}`)
    }
```

*   `const contextParams = { ...params }`: Creates a shallow copy of the input `params`. This is important because subsequent steps might modify `contextParams` (e.g., adding an access token, removing internal parameters), and we want to avoid side effects on the original `params` object.
*   `validateRequiredParametersAfterMerge(toolId, tool, contextParams)`: Calls the validation utility to ensure all `required` parameters (as defined in `tool.config`) are present in `contextParams`. If validation fails, this function will throw an error.
*   `if (!tool)`: This is a defensive check. While `validateRequiredParametersAfterMerge` *should* throw an error if `tool` is `undefined` (as it can't validate parameters for a non-existent tool), this explicit check ensures that if somehow `tool` is still `undefined` at this point, an explicit error is thrown.

```typescript
    // If we have a credential parameter, fetch the access token
    if (contextParams.credential) {
      logger.info(
        `[${requestId}] Tool ${toolId} needs access token for credential: ${contextParams.credential}`
      )
      try {
        const baseUrl = getBaseUrl()

        // Prepare the token payload
        const tokenPayload: OAuthTokenPayload = {
          credentialId: contextParams.credential,
        }

        // Add workflowId if it exists in params, context, or executionContext
        const workflowId =
          contextParams.workflowId ||
          contextParams._context?.workflowId ||
          executionContext?.workflowId
        if (workflowId) {
          tokenPayload.workflowId = workflowId
        }

        logger.info(`[${requestId}] Fetching access token from ${baseUrl}/api/auth/oauth/token`)

        // Build token URL and also include workflowId in query so server auth can read it
        const tokenUrlObj = new URL('/api/auth/oauth/token', baseUrl)
        if (workflowId) {
          tokenUrlObj.searchParams.set('workflowId', workflowId)
        }

        // Always send Content-Type; add internal auth on server-side runs
        const tokenHeaders: Record<string, string> = { 'Content-Type': 'application/json' }
        if (typeof window === 'undefined') {
          try {
            const internalToken = await generateInternalToken()
            tokenHeaders.Authorization = `Bearer ${internalToken}`
          } catch (_e) {
            // Swallow token generation errors; the request will fail and be reported upstream
          }
        }

        const response = await fetch(tokenUrlObj.toString(), {
          method: 'POST',
          headers: tokenHeaders,
          body: JSON.stringify(tokenPayload),
        })

        if (!response.ok) {
          const errorText = await response.text()
          logger.error(`[${requestId}] Token fetch failed for ${toolId}:`, {
            status: response.status,
            error: errorText,
          })
          throw new Error(`Failed to fetch access token: ${response.status} ${errorText}`)
        }

        const data = await response.json()
        contextParams.accessToken = data.accessToken

        logger.info(
          `[${requestId}] Successfully got access token for ${toolId}, length: ${data.accessToken?.length || 0}`
        )

        // Preserve credential for downstream transforms while removing it from request payload
        // so we don't leak it to external services.
        if (contextParams.credential) {
          ;(contextParams as any)._credentialId = contextParams.credential
        }
        if (workflowId) {
          ;(contextParams as any)._workflowId = workflowId
        }
        // Clean up params we don't need to pass to the actual tool
        contextParams.credential = undefined
        if (contextParams.workflowId) contextParams.workflowId = undefined
      } catch (error: any) {
        logger.error(`[${requestId}] Error fetching access token for ${toolId}:`, {
          error: error instanceof Error ? error.message : String(error),
        })
        // Re-throw the error to fail the tool execution if token fetching fails
        throw new Error(
          `Failed to obtain credential for tool ${toolId}: ${error instanceof Error ? error.message : String(error)}`
        )
      }
    }
```

*   This large block handles the authentication aspect if the tool requires a `credential` (e.g., an OAuth connection) to obtain an `accessToken`.
*   `if (contextParams.credential)`: Checks if a `credential` property exists in the tool's parameters.
*   `logger.info(...)`: Logs that an access token is needed.
*   `const baseUrl = getBaseUrl()`: Gets the application's base URL.
*   `const tokenPayload: OAuthTokenPayload = { credentialId: contextParams.credential }`: Creates a payload for the token request, including the `credentialId`.
*   `const workflowId = ...`: Attempts to find a `workflowId` from parameters, context, or `executionContext`. If found, it's added to `tokenPayload` and also as a query parameter to the token URL.
*   `const tokenUrlObj = new URL('/api/auth/oauth/token', baseUrl)`: Constructs the full URL for the internal token endpoint.
*   `const tokenHeaders: Record<string, string> = { 'Content-Type': 'application/json' }`: Initializes headers for the token request.
*   `if (typeof window === 'undefined')`: Checks if running on the server.
    *   `const internalToken = await generateInternalToken()`: If on the server, generates an internal bearer token.
    *   `tokenHeaders.Authorization = `Bearer ${internalToken}``: Adds the internal token to the authorization header for secure server-to-server communication.
    *   `try...catch (_e)`: Swallows errors during internal token generation. The request to the token endpoint will likely fail anyway, and that failure will be reported.
*   `const response = await fetch(tokenUrlObj.toString(), { ... })`: Makes the HTTP POST request to the token endpoint.
*   `if (!response.ok)`: If the token request fails (non-2xx status):
    *   Logs the error, reads the error text from the response.
    *   Throws a new `Error` indicating the failure to fetch the access token.
*   `const data = await response.json()`: Parses the successful token response.
*   `contextParams.accessToken = data.accessToken`: Stores the fetched `accessToken` in `contextParams`, making it available for the actual tool invocation.
*   `logger.info(...)`: Logs the successful acquisition of the access token.
*   **Credential Cleanup:**
    *   `;(contextParams as any)._credentialId = contextParams.credential` and `_workflowId`: Preserves the original `credentialId` and `workflowId` under new, internal-only keys (`_credentialId`, `_workflowId`). This is likely for debugging or post-processing purposes.
    *   `contextParams.credential = undefined` and `contextParams.workflowId = undefined`: Removes the original `credential` and `workflowId` from `contextParams`. This is a security measure to prevent sensitive credential IDs and potentially workflow IDs from being accidentally passed to external services or being part of the `tool.request.body` that gets sent out.
*   `catch (error: any)`: Catches any errors during the access token fetching process.
    *   Logs the error.
    *   `throw new Error(...)`: Re-throws a more specific error message indicating failure to obtain the credential, ensuring the overall tool execution fails.

```typescript
    // For internal routes or when skipProxy is true, call the API directly
    // Internal routes are automatically detected by checking if URL starts with /api/
    const endpointUrl =
      typeof tool.request.url === 'function' ? tool.request.url(contextParams) : tool.request.url
    const isInternalRoute = endpointUrl.startsWith('/api/')

    if (isInternalRoute || skipProxy) {
      const result = await handleInternalRequest(toolId, tool, contextParams)

      // Apply post-processing if available and not skipped
      let finalResult = result
      if (tool.postProcess && result.success && !skipPostProcess) {
        try {
          finalResult = await tool.postProcess(result, contextParams, executeTool)
        } catch (error) {
          logger.error(`[${requestId}] Post-processing error for ${toolId}:`, {
            error: error instanceof Error ? error.message : String(error),
          })
          finalResult = result
        }
      }

      // Process file outputs if execution context is available
      finalResult = await processFileOutputs(finalResult, tool, executionContext)

      // Add timing data to the result
      const endTime = new Date()
      const endTimeISO = endTime.toISOString()
      const duration = endTime.getTime() - startTime.getTime()
      return {
        ...finalResult,
        timing: {
          startTime: startTimeISO,
          endTime: endTimeISO,
          duration,
        },
      }
    }
```

*   This block handles the execution path for **internal routes** or when the `skipProxy` flag is `true`.
*   `const endpointUrl = ...`: Determines the actual endpoint URL for the tool. The `tool.request.url` can be either a static string or a function that dynamically generates the URL based on `contextParams`.
*   `const isInternalRoute = endpointUrl.startsWith('/api/')`: Checks if the resolved `endpointUrl` starts with `/api/`, which is a convention for internal API endpoints within the application.
*   `if (isInternalRoute || skipProxy)`: If it's an internal route or explicitly asked to skip the proxy:
    *   `const result = await handleInternalRequest(toolId, tool, contextParams)`: Calls the `handleInternalRequest` function to directly make a `fetch` call to the internal API.
    *   `let finalResult = result`: Initializes `finalResult` with the raw result.
    *   `if (tool.postProcess && result.success && !skipPostProcess)`: Checks if the tool has a `postProcess` function defined, if the initial `result` was successful, and if post-processing is not skipped.
        *   `finalResult = await tool.postProcess(result, contextParams, executeTool)`: Executes the `postProcess` function, passing the result, context, and even the `executeTool` function itself (allowing for recursive tool calls within post-processing).
        *   `try...catch (error)`: Catches errors during post-processing, logs them, but defaults `finalResult` back to the original `result` to avoid failing the entire operation due to post-processing errors.
    *   `finalResult = await processFileOutputs(finalResult, tool, executionContext)`: Calls the `processFileOutputs` function to handle any file outputs.
    *   **Timing Data:**
        *   `const endTime = new Date()`: Records the end time.
        *   `const endTimeISO = endTime.toISOString()`: Converts end time to ISO string.
        *   `const duration = endTime.getTime() - startTime.getTime()`: Calculates the total duration.
    *   `return { ...finalResult, timing: { ... } }`: Returns the `finalResult` (potentially post-processed and with file outputs handled), augmented with the timing information.

```typescript
    // For external APIs, use the proxy
    const result = await handleProxyRequest(toolId, contextParams, executionContext)

    // Apply post-processing if available and not skipped
    let finalResult = result
    if (tool.postProcess && result.success && !skipPostProcess) {
      try {
        finalResult = await tool.postProcess(result, contextParams, executeTool)
      } catch (error) {
        logger.error(`[${requestId}] Post-processing error for ${toolId}:`, {
          error: error instanceof Error ? error.message : String(error),
        })
        finalResult = result
      }
    }

    // Process file outputs if execution context is available
    finalResult = await processFileOutputs(finalResult, tool, executionContext)

    // Add timing data to the result
    const endTime = new Date()
    const endTimeISO = endTime.toISOString()
    const duration = endTime.getTime() - startTime.getTime()
    return {
      ...finalResult,
      timing: {
        startTime: startTimeISO,
        endTime: endTimeISO,
        duration,
      },
    }
```

*   This block handles the execution path for **external APIs** (which are assumed when `isInternalRoute` is `false` and `skipProxy` is `false`).
*   `const result = await handleProxyRequest(toolId, contextParams, executionContext)`: Calls the `handleProxyRequest` function, which routes the request through a server-side proxy. This is important for security (hiding API keys), CORS, and rate limiting.
*   The subsequent `postProcess`, `processFileOutputs`, and timing logic are identical to the internal request path, ensuring consistent handling of these concerns regardless of the execution strategy.

```typescript
  } catch (error: any) {
    logger.error(`[${requestId}] Error executing tool ${toolId}:`, {
      error: error instanceof Error ? error.message : String(error),
      stack: error instanceof Error ? error.stack : undefined,
    })

    // Default error handling
    let errorMessage = 'Unknown error occurred'
    let errorDetails = {}

    if (error instanceof Error) {
      errorMessage = error.message || `Error executing tool ${toolId}`
    } else if (typeof error === 'string') {
      errorMessage = error
    } else if (error && typeof error === 'object') {
      // Handle HTTP response errors
      if (error.status) {
        errorMessage = `HTTP ${error.status}: ${error.statusText || 'Request failed'}`

        if (error.data) {
          if (typeof error.data === 'string') {
            errorMessage = `${errorMessage} - ${error.data}`
          } else if (error.data.message) {
            errorMessage = `${errorMessage} - ${error.data.message}`
          } else if (error.data.error) {
            errorMessage = `${errorMessage} - ${
              typeof error.data.error === 'string'
                ? error.data.error
                : JSON.stringify(error.data.error)
            }`
          }
        }

        errorDetails = {
          status: error.status,
          statusText: error.statusText,
          data: error.data,
        }
      }
      // Handle other errors with messages
      else if (error.message) {
        // Don't pass along "undefined (undefined)" messages
        if (error.message === 'undefined (undefined)') {
          errorMessage = `Error executing tool ${toolId}`
          // Add status if available
          if (error.status) {
            errorMessage += ` (Status: ${error.status})`
          }
        } else {
          errorMessage = error.message
        }

        if ((error as any).cause) {
          errorMessage = `${errorMessage} (${(error as any).cause})`
        }
      }
    }

    // Add timing data even for errors
    const endTime = new Date()
    const endTimeISO = endTime.toISOString()
    const duration = endTime.getTime() - startTime.getTime()
    return {
      success: false,
      output: errorDetails,
      error: errorMessage,
      timing: {
        startTime: startTimeISO,
        endTime: endTimeISO,
        duration,
      },
    }
  }
```

*   `catch (error: any)`: This block catches any unhandled errors that occur anywhere within the `try` block of `executeTool`.
*   `logger.error(...)`: Logs the error, including the message and stack trace if it's an `Error` object.
*   **Error Message Normalization:** This section is dedicated to creating a comprehensive and standardized `errorMessage` and `errorDetails` object from potentially diverse `error` types (e.g., `Error` instances, plain strings, or objects thrown from `fetch` or other functions).
    *   It prioritizes `Error` messages, then string errors, then delves into object structures:
        *   If `error` has a `status` property, it tries to format it as an HTTP error, also extracting data from `error.data` (which might contain `message`, `error` properties).
        *   If `error` has a `message` property (but no `status`), it uses that message, with special handling for "undefined (undefined)" messages and potentially adding `cause` information.
*   **Timing Data for Errors:** Even on error, it calculates and includes `startTime`, `endTime`, and `duration` in the returned `ToolResponse`.
*   `return { success: false, output: errorDetails, error: errorMessage, timing: { ... } }`: Returns a `ToolResponse` indicating `success: false`, along with the extracted `errorDetails`, the `errorMessage`, and timing information. This ensures that callers always receive a `ToolResponse` object, simplifying error handling for the client.

---

### Helper Function Explanations

#### `isErrorResponse`

```typescript
/**
 * Determines if a response or result represents an error condition
 */
function isErrorResponse(
  response: Response | any,
  data?: any
): { isError: boolean; errorInfo?: { status?: number; statusText?: string; data?: any } } {
  // HTTP Response object
  if (response && typeof response === 'object' && 'ok' in response) {
    if (!response.ok) {
      return {
        isError: true,
        errorInfo: {
          status: response.status,
          statusText: response.statusText,
          data: data,
        },
      }
    }
    return { isError: false }
  }

  // ToolResponse object
  if (response && typeof response === 'object' && 'success' in response) {
    return {
      isError: !response.success,
      errorInfo: response.success ? undefined : { data: response },
    }
  }

  // Check for error indicators in data
  if (data && typeof data === 'object') {
    if (data.error || data.success === false) {
      return {
        isError: true,
        errorInfo: { data: data },
      }
    }
  }

  return { isError: false }
}
```

*   `function isErrorResponse(...)`: This function serves to universally detect if a given `response` (either a raw `Response` object from `fetch` or a parsed `ToolResponse`-like object) signifies an error.
*   **HTTP Response object check:**
    *   `if (response && typeof response === 'object' && 'ok' in response)`: Checks if `response` looks like a standard `fetch` API `Response` object (i.e., it has an `ok` property).
    *   `if (!response.ok)`: If `response.ok` is `false` (meaning HTTP status is 4xx or 5xx), it returns `isError: true` along with `status`, `statusText`, and the `data` that was parsed from the response body.
*   **ToolResponse object check:**
    *   `if (response && typeof response === 'object' && 'success' in response)`: Checks if `response` looks like a `ToolResponse` object (i.e., it has a `success` property).
    *   `return { isError: !response.success, ... }`: Returns `isError: true` if `response.success` is `false`, and provides the entire `response` object as `errorInfo.data`.
*   **Generic Data Error check:**
    *   `if (data && typeof data === 'object')`: If `response` wasn't a recognized type, it checks the `data` parameter directly.
    *   `if (data.error || data.success === false)`: If `data` contains an `error` property or `success: false`, it's considered an error.
*   `return { isError: false }`: If none of the above conditions are met, it's assumed not to be an error.

#### `handleInternalRequest`

```typescript
/**
 * Handle an internal/direct tool request
 */
async function handleInternalRequest(
  toolId: string,
  tool: ToolConfig,
  params: Record<string, any>
): Promise<ToolResponse> {
  const requestId = generateRequestId()

  // Format the request parameters
  const requestParams = formatRequestParams(tool, params)

  try {
    const baseUrl = getBaseUrl()
    // Handle the case where url may be a function or string
    const endpointUrl =
      typeof tool.request.url === 'function' ? tool.request.url(params) : tool.request.url

    const fullUrl = new URL(endpointUrl, baseUrl).toString()

    // For custom tools, validate parameters on the client side before sending
    if (toolId.startsWith('custom_') && tool.request.body) {
      const requestBody = tool.request.body(params)
      if (requestBody.schema && requestBody.params) {
        try {
          validateClientSideParams(requestBody.params, requestBody.schema)
        } catch (validationError) {
          logger.error(`[${requestId}] Custom tool validation failed for ${toolId}:`, {
            error:
              validationError instanceof Error ? validationError.message : String(validationError),
          })
          throw validationError
        }
      }
    }

    // Prepare request options
    const requestOptions = {
      method: requestParams.method,
      headers: new Headers(requestParams.headers),
      body: requestParams.body,
    }

    const response = await fetch(fullUrl, requestOptions)

    // For non-OK responses, attempt JSON first; if parsing fails, preserve legacy error expected by tests
    if (!response.ok) {
      let errorData: any
      try {
        errorData = await response.json()
      } catch (jsonError) {
        logger.error(`[${requestId}] JSON parse error for ${toolId}:`, {
          error: jsonError instanceof Error ? jsonError.message : String(jsonError),
        })
        throw new Error(`Failed to parse response from ${toolId}: ${jsonError}`)
      }

      const { isError, errorInfo } = isErrorResponse(response, errorData)
      if (isError) {
        const errorToTransform = createTransformedErrorFromErrorInfo(errorInfo)

        logger.error(`[${requestId}] Internal API error for ${toolId}:`, {
          status: errorInfo?.status,
          errorData: errorInfo?.data,
        })

        throw errorToTransform
      }
    }

    // Parse response data once with guard for empty 202 bodies
    let responseData
    const status = response.status
    if (status === 202) {
      // Many APIs (e.g., Microsoft Graph) return 202 with empty body
      responseData = { status }
    } else {
      if (tool.transformResponse) {
        responseData = null
      } else {
        try {
          responseData = await response.json()
        } catch (jsonError) {
          logger.error(`[${requestId}] JSON parse error for ${toolId}:`, {
            error: jsonError instanceof Error ? jsonError.message : String(jsonError),
          })
          throw new Error(`Failed to parse response from ${toolId}: ${jsonError}`)
        }
      }
    }

    // Check for error conditions
    const { isError, errorInfo } = isErrorResponse(response, responseData)

    if (isError) {
      // Handle error case
      const errorToTransform = createTransformedErrorFromErrorInfo(errorInfo)

      logger.error(`[${requestId}] Internal API error for ${toolId}:`, {
        status: errorInfo?.status,
        errorData: errorInfo?.data,
      })

      throw errorToTransform
    }

    // Success case: use transformResponse if available
    if (tool.transformResponse) {
      try {
        // Create a mock response object that provides the methods transformResponse needs
        const mockResponse = {
          ok: response.ok,
          status: response.status,
          statusText: response.statusText,
          headers: response.headers,
          url: fullUrl,
          json: () => response.json(),
          text: () => response.text(),
        } as Response

        const data = await tool.transformResponse(mockResponse, params)
        return data
      } catch (transformError) {
        logger.error(`[${requestId}] Transform response error for ${toolId}:`, {
          error: transformError instanceof Error ? transformError.message : String(transformError),
        })
        throw transformError
      }
    }

    // Default success response handling
    return {
      success: true,
      output: responseData.output || responseData,
      error: undefined,
    }
  } catch (error: any) {
    logger.error(`[${requestId}] Internal request error for ${toolId}:`, {
      error: error instanceof Error ? error.message : String(error),
    })

    // Let the error bubble up to be handled in the main executeTool function
    throw error
  }
}
```

*   `async function handleInternalRequest(...)`: This function is responsible for making direct HTTP requests to internal API endpoints.
*   `const requestId = generateRequestId()`: Generates a request ID for logging within this function.
*   `const requestParams = formatRequestParams(tool, params)`: Formats the input `params` into a structure suitable for `fetch` (e.g., separating headers, body, method).
*   `const endpointUrl = ...`: Resolves the tool's URL, handling both function and string definitions.
*   `const fullUrl = new URL(endpointUrl, baseUrl).toString()`: Constructs the complete URL using the base URL.
*   **Client-side parameter validation for custom tools:**
    *   `if (toolId.startsWith('custom_') && tool.request.body)`: If it's a custom tool and defines a `request.body` (which might include a schema).
    *   `validateClientSideParams(requestBody.params, requestBody.schema)`: Performs basic validation of parameters against a schema, catching errors early.
*   `const requestOptions = { ... }`: Prepares the options for the `fetch` call (method, headers, body).
*   `const response = await fetch(fullUrl, requestOptions)`: Makes the actual HTTP request.
*   **Error Handling for non-OK responses:**
    *   `if (!response.ok)`: If the HTTP response is not successful (status 4xx/5xx).
    *   Attempts to parse `response.json()` into `errorData`. If JSON parsing fails, it throws a generic JSON parsing error.
    *   `const { isError, errorInfo } = isErrorResponse(response, errorData)`: Uses the helper to determine if it's an error.
    *   `const errorToTransform = createTransformedErrorFromErrorInfo(errorInfo)`: Creates a standardized error object.
    *   Logs the error and `throw errorToTransform` to propagate it up.
*   **Parsing successful response data:**
    *   `let responseData`: Variable to hold the parsed response.
    *   `if (status === 202)`: Special handling for HTTP 202 (Accepted) status, which often has an empty body; sets `responseData` to `{ status }`.
    *   `else if (tool.transformResponse)`: If the tool defines a `transformResponse` function, it indicates that `responseData` might be handled later, so `responseData` is set to `null` initially.
    *   `else`: Otherwise, it attempts to parse the response as JSON. If JSON parsing fails, it throws an error.
*   **Post-parsing error check:**
    *   `const { isError, errorInfo } = isErrorResponse(response, responseData)`: Even after parsing, `responseData` itself might indicate an error (e.g., `{"success": false, "error": "..."}`). This checks for that.
    *   If an error is detected, it logs and throws a transformed error.
*   **Success case with `transformResponse`:**
    *   `if (tool.transformResponse)`: If the tool provides a custom `transformResponse` function.
    *   It creates a `mockResponse` object that mimics a standard `Response` object but ensures `json()` and `text()` methods correctly return the original response's content. This is a common pattern to pass the full `Response` object to a `transformResponse` callback.
    *   `const data = await tool.transformResponse(mockResponse, params)`: Calls the custom transformation function.
    *   Returns the transformed `data`.
    *   Includes `try...catch` for transformation errors.
*   **Default success response:**
    *   `return { success: true, output: responseData.output || responseData, error: undefined }`: If no `transformResponse` is defined, it returns a successful `ToolResponse` with the parsed `responseData` (or `responseData.output` if available) as its `output`.
*   `catch (error: any)`: Catches any errors within `handleInternalRequest`, logs them, and re-throws them for the main `executeTool` function to handle.

#### `validateClientSideParams`

```typescript
/**
 * Validates parameters on the client side before sending to the execute endpoint
 */
function validateClientSideParams(
  params: Record<string, any>,
  schema: {
    type: string
    properties: Record<string, any>
    required?: string[]
  }
) {
  if (!schema || schema.type !== 'object') {
    throw new Error('Invalid schema format')
  }

  // Internal parameters that should be excluded from validation
  const internalParamSet = new Set([
    '_context',
    'workflowId',
    'envVars',
    'workflowVariables',
    'blockData',
    'blockNameMapping',
  ])

  // Check required parameters
  if (schema.required) {
    for (const requiredParam of schema.required) {
      if (!(requiredParam in params)) {
        throw new Error(`Required parameter missing: ${requiredParam}`)
      }
    }
  }

  // Check parameter types (basic validation)
  for (const [paramName, paramValue] of Object.entries(params)) {
    // Skip validation for internal parameters
    if (internalParamSet.has(paramName)) {
      continue
    }

    const paramSchema = schema.properties[paramName]
    if (!paramSchema) {
      throw new Error(`Unknown parameter: ${paramName}`)
    }

    // Basic type checking
    const type = paramSchema.type
    if (type === 'string' && typeof paramValue !== 'string') {
      throw new Error(`Parameter ${paramName} should be a string`)
    }
    if (type === 'number' && typeof paramValue !== 'number') {
      throw new Error(`Parameter ${paramName} should be a number`)
    }
    if (type === 'boolean' && typeof paramValue !== 'boolean') {
      throw new Error(`Parameter ${paramName} should be a boolean`)
    }
    if (type === 'array' && !Array.isArray(paramValue)) {
      throw new Error(`Parameter ${paramName} should be an array`)
    }
    if (type === 'object' && (typeof paramValue !== 'object' || paramValue === null)) {
      throw new Error(`Parameter ${paramName} should be an object`)
    }
  }
}
```

*   `function validateClientSideParams(...)`: This function performs basic validation of tool parameters against a provided schema, typically on the client side (before sending to an API).
*   `if (!schema || schema.type !== 'object')`: Basic validation of the schema itself.
*   `const internalParamSet = new Set([...])`: Defines a set of internal parameters that should be ignored during validation, as they are not actual user inputs.
*   **Check required parameters:**
    *   `if (schema.required)`: If the schema defines `required` parameters.
    *   It iterates through `requiredParam` and checks if each is present in `params`. If not, it throws an `Error`.
*   **Check parameter types:**
    *   It iterates through all `[paramName, paramValue]` in `params`.
    *   `if (internalParamSet.has(paramName)) { continue }`: Skips validation for internal parameters.
    *   `if (!paramSchema)`: If a parameter exists in `params` but not in the `schema.properties`, it's considered an "unknown parameter" and an error is thrown.
    *   **Basic type checking:** It then performs simple `typeof` or `Array.isArray` checks based on `paramSchema.type` (e.g., `string`, `number`, `boolean`, `array`, `object`). If there's a type mismatch, it throws an `Error`.

#### `handleProxyRequest`

```typescript
/**
 * Handle a request via the proxy
 */
async function handleProxyRequest(
  toolId: string,
  params: Record<string, any>,
  executionContext?: ExecutionContext
): Promise<ToolResponse> {
  const requestId = generateRequestId()

  const baseUrl = getBaseUrl()
  const proxyUrl = new URL('/api/proxy', baseUrl).toString()

  try {
    const response = await fetch(proxyUrl, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ toolId, params, executionContext }),
    })

    if (!response.ok) {
      const errorText = await response.text()
      logger.error(`[${requestId}] Proxy request failed for ${toolId}:`, {
        status: response.status,
        statusText: response.statusText,
        error: errorText.substring(0, 200), // Limit error text length
      })

      let errorMessage = `HTTP error ${response.status}: ${response.statusText}`

      try {
        // Try to parse as JSON for more details
        const errorJson = JSON.parse(errorText)
        // Enhanced error extraction to match internal API patterns
        errorMessage =
          // Primary error patterns
          errorJson.errors?.[0]?.message ||
          errorJson.errors?.[0]?.detail ||
          errorJson.error?.message ||
          (typeof errorJson.error === 'string' ? errorJson.error : undefined) ||
          errorJson.message ||
          errorJson.error_description ||
          errorJson.fault?.faultstring ||
          errorJson.faultstring ||
          // Fallback
          (typeof errorJson.error === 'object'
            ? `API Error: ${response.status} ${response.statusText}`
            : `HTTP error ${response.status}: ${response.statusText}`)
      } catch (parseError) {
        // If not JSON, use the raw text
        if (errorText) {
          errorMessage = `${errorMessage}: ${errorText}`
        }
      }

      throw new Error(errorMessage)
    }

    // Parse the successful response
    const result = await response.json()
    return result
  } catch (error: any) {
    logger.error(`[${requestId}] Proxy request error for ${toolId}:`, {
      error: error instanceof Error ? error.message : String(error),
    })

    return {
      success: false,
      output: {},
      error: error.message || 'Proxy request failed',
    }
  }
}
```

*   `async function handleProxyRequest(...)`: This function sends tool execution requests through a server-side proxy endpoint, typically for external API calls.
*   `const requestId = generateRequestId()`: Generates a request ID.
*   `const proxyUrl = new URL('/api/proxy', baseUrl).toString()`: Constructs the URL for the internal proxy endpoint.
*   `const response = await fetch(proxyUrl, { ... })`: Makes a POST request to the proxy.
    *   The `body` of this request contains the `toolId`, the tool's `params`, and the `executionContext`. The proxy server will then use this information to make the actual external API call.
*   `if (!response.ok)`: If the proxy request itself fails (e.g., the proxy server returns an error, or the external API call failed and the proxy relayed that as an error status).
    *   `const errorText = await response.text()`: Reads the error message from the proxy's response.
    *   `logger.error(...)`: Logs the error.
    *   **Enhanced error extraction:** Similar to `getDeepApiErrorMessage`, it attempts to parse `errorText` as JSON to get a more structured error message. If JSON parsing fails, it falls back to the raw `errorText`.
    *   `throw new Error(errorMessage)`: Throws a new `Error` with the extracted message.
*   `const result = await response.json()`: If the proxy request was successful, parses the response (which is expected to be a `ToolResponse` object from the proxy).
*   `return result`: Returns the `ToolResponse` from the proxy.
*   `catch (error: any)`: Catches any network or other errors during the proxy request.
    *   Logs the error.
    *   `return { success: false, output: {}, error: error.message || 'Proxy request failed' }`: Returns a `ToolResponse` indicating failure, with a generic error message or the specific error message from the caught error.

#### `executeMcpTool`

```typescript
/**
 * Execute an MCP tool via the server-side proxy
 *
 * @param toolId - MCP tool ID in format "mcp-serverId-toolName"
 * @param params - Tool parameters
 * @param executionContext - Execution context
 * @param requestId - Request ID for logging
 * @param startTimeISO - Start time for timing
 */
async function executeMcpTool(
  toolId: string,
  params: Record<string, any>,
  executionContext?: ExecutionContext,
  requestId?: string,
  startTimeISO?: string
): Promise<ToolResponse> {
  const actualRequestId = requestId || generateRequestId()
  const actualStartTime = startTimeISO || new Date().toISOString()

  try {
    logger.info(`[${actualRequestId}] Executing MCP tool: ${toolId}`)

    const { serverId, toolName } = parseMcpToolId(toolId)

    const baseUrl = getBaseUrl()

    const headers: Record<string, string> = { 'Content-Type': 'application/json' }

    if (typeof window === 'undefined') {
      try {
        const internalToken = await generateInternalToken()
        headers.Authorization = `Bearer ${internalToken}`
      } catch (error) {
        logger.error(`[${actualRequestId}] Failed to generate internal token:`, error)
      }
    }

    // Handle two different parameter structures:
    // 1. Direct MCP blocks: arguments are stored as JSON string in 'arguments' field
    // 2. Agent blocks: arguments are passed directly as top-level parameters
    let toolArguments = {}

    // First check if we have the 'arguments' field (direct MCP block usage)
    if (params.arguments) {
      if (typeof params.arguments === 'string') {
        try {
          toolArguments = JSON.parse(params.arguments)
        } catch (error) {
          logger.warn(`[${actualRequestId}] Failed to parse MCP arguments JSON:`, params.arguments)
          toolArguments = {}
        }
      } else {
        toolArguments = params.arguments
      }
    } else {
      // Agent block usage: extract MCP-specific arguments by filtering out system parameters
      toolArguments = Object.fromEntries(
        Object.entries(params).filter(([key]) => !MCP_SYSTEM_PARAMETERS.has(key))
      )
    }

    const workspaceId = params._context?.workspaceId || executionContext?.workspaceId
    const workflowId = params._context?.workflowId || executionContext?.workflowId

    if (!workspaceId) {
      return {
        success: false,
        output: {},
        error: `Missing workspaceId in execution context for MCP tool ${toolName}`,
        timing: {
          startTime: actualStartTime,
          endTime: new Date().toISOString(),
          duration: Date.now() - new Date(actualStartTime).getTime(),
        },
      }
    }

    const requestBody = {
      serverId,
      toolName,
      arguments: toolArguments,
      workflowId, // Pass workflow context for user resolution
      workspaceId, // Pass workspace context for scoping
    }

    logger.info(`[${actualRequestId}] Making MCP tool request to ${toolName} on ${serverId}`, {
      hasWorkspaceId: !!workspaceId,
      hasWorkflowId: !!workflowId,
    })

    const response = await fetch(`${baseUrl}/api/mcp/tools/execute`, {
      method: 'POST',
      headers,
      body: JSON.stringify(requestBody),
    })

    const endTime = new Date()
    const endTimeISO = endTime.toISOString()
    const duration = endTime.getTime() - new Date(actualStartTime).getTime()

    if (!response.ok) {
      let errorMessage = `MCP tool execution failed: ${response.status} ${response.statusText}`

      try {
        const errorData = await response.json()
        if (errorData.error) {
          errorMessage = errorData.error
        }
      } catch {
        // Failed to parse error response, use default message
      }

      return {
        success: false,
        output: {},
        error: errorMessage,
        timing: {
          startTime: actualStartTime,
          endTime: endTimeISO,
          duration,
        },
      }
    }

    const result = await response.json()

    if (!result.success) {
      return {
        success: false,
        output: {},
        error: result.error || 'MCP tool execution failed',
        timing: {
          startTime: actualStartTime,
          endTime: endTimeISO,
          duration,
        },
      }
    }

    logger.info(`[${actualRequestId}] MCP tool ${toolId} executed successfully`)

    return {
      success: true,
      output: result.data?.output || result.output || result.data || {},
      timing: {
        startTime: actualStartTime,
        endTime: endTimeISO,
        duration,
      },
    }
  } catch (error) {
    const endTime = new Date()
    const endTimeISO = endTime.toISOString()
    const duration = endTime.getTime() - new Date(actualStartTime).getTime()

    logger.error(`[${actualRequestId}] Error executing MCP tool ${toolId}:`, error)

    const errorMessage =
      error instanceof Error ? error.message : `Failed to execute MCP tool ${toolId}`

    return {
      success: false,
      output: {},
      error: errorMessage,
      timing: {
        startTime: actualStartTime,
        endTime: endTimeISO,
        duration,
      },
    }
  }
}
```

*   `async function executeMcpTool(...)`: This dedicated function handles the execution of MCP (Microservice Control Plane) tools. These tools are distinct in how their parameters are structured and how they are invoked.
*   `const actualRequestId = requestId || generateRequestId()` and `actualStartTime`: Uses provided `requestId` and `startTimeISO` if available, otherwise generates new ones.
*   `const { serverId, toolName } = parseMcpToolId(toolId)`: Parses the special `mcp-` prefixed `toolId` to extract the `serverId` and actual `toolName`.
*   `const headers: Record<string, string> = { 'Content-Type': 'application/json' }`: Initializes headers.
*   `if (typeof window === 'undefined')`: Checks if running on the server.
    *   `const internalToken = await generateInternalToken()`: Generates an internal token.
    *   `headers.Authorization = `Bearer ${internalToken}``: Adds internal authentication for the server-to-server call to the MCP execution endpoint.
*   **Parameter structure handling (`toolArguments`):** This is a key part of MCP tool handling.
    *   `let toolArguments = {}`: Initializes an empty object for the arguments relevant to the MCP tool itself.
    *   `if (params.arguments)`: Checks if the `params` object has an `arguments` field. This indicates a "direct MCP block" usage where arguments are explicitly nested.
        *   If `params.arguments` is a string, it attempts to `JSON.parse` it (expecting a JSON string of arguments).
        *   Otherwise, it uses `params.arguments` directly.
    *   `else`: If no `params.arguments` field, it's assumed to be "Agent block usage".
        *   `toolArguments = Object.fromEntries(Object.entries(params).filter(...))`: It extracts arguments from the top-level `params` by **filtering out** `MCP_SYSTEM_PARAMETERS`. This ensures only the actual tool inputs are passed.
*   `const workspaceId = ...` and `workflowId = ...`: Extracts `workspaceId` and `workflowId` from `params` or `executionContext`.
*   `if (!workspaceId)`: Ensures `workspaceId` is available, which is critical for MCP tools, and returns an error if missing.
*   `const requestBody = { serverId, toolName, arguments: toolArguments, workflowId, workspaceId }`: Constructs the payload for the MCP execution API.
*   `const response = await fetch(`${baseUrl}/api/mcp/tools/execute`, { ... })`: Makes the POST request to the internal `/api/mcp/tools/execute` endpoint.
*   **Timing:** Records `endTime`, `endTimeISO`, and `duration`.
*   **Error Handling (non-OK response):**
    *   `if (!response.ok)`: If the `fetch` call to the MCP execution API itself fails.
    *   Attempts to parse `response.json()` for a more specific error message, otherwise uses a default message.
    *   Returns a `ToolResponse` indicating failure with timing.
*   **Error Handling (successful response, but logical error):**
    *   `const result = await response.json()`: Parses the successful HTTP response.
    *   `if (!result.success)`: Checks if the *parsed result* from the MCP endpoint indicates `success: false`. This handles cases where the HTTP call was fine, but the MCP tool execution logically failed.
    *   Returns a `ToolResponse` indicating failure with timing.
*   **Success response:**
    *   `logger.info(...)`: Logs successful execution.
    *   `return { success: true, output: result.data?.output || result.output || result.data || {}, timing: { ... } }`: Returns a successful `ToolResponse`, extracting the output from `result.data.output`, `result.output`, or `result.data` as available, along with timing.
*   `catch (error)`: Catches any uncaught exceptions during the MCP tool execution.
    *   Logs the error, calculates timing.
    *   Returns a `ToolResponse` indicating failure with a generic error message or the specific error message from the caught error.