This TypeScript file, `tools-utils.ts`, serves as a comprehensive utility hub for managing, configuring, and executing "tools" within the application. In this context, "tools" typically refer to external functionalities or APIs that an AI agent or a specific workflow might need to interact with. It handles both pre-defined "built-in" tools and dynamically loaded "custom" tools, providing crucial mechanisms for their definition, parameter processing, request formatting, execution, and response handling.

---

## Purpose of this File

The primary purpose of `tools-utils.ts` is to provide a standardized and robust way to:

1.  **Define and Configure Tools:** Allow the application to understand the structure and requirements of various tools, whether they are hardcoded or dynamically created by users.
2.  **Process Tool Parameters:** Handle the transformation and validation of inputs required by tools, ensuring they meet the tool's specifications. This includes merging parameters from different sources (e.g., user input, AI agent suggestions).
3.  **Format and Execute Requests:** Construct the actual HTTP requests to call external APIs based on a tool's configuration and then execute those requests.
4.  **Handle Responses:** Process the data received from external tools, transforming it into a consistent format for the application to consume, and manage potential errors.
5.  **Manage Custom Tools:** Offer specific logic for retrieving, configuring, and executing user-defined "custom tools," which often involve sending code and dynamic parameters to a backend execution environment.
6.  **Support Client-side and Server-side Operations:** Provide mechanisms that work correctly regardless of whether the code is running in a browser environment or on a server.

Essentially, this file acts as the **toolkit's operating system**, abstracting away the complexities of interacting with diverse external services and custom logic, making it easier for other parts of the application (like an AI agent or workflow engine) to utilize these capabilities.

---

## Simplified Complex Logic

Let's break down the core ideas to make the file's complexity easier to grasp:

1.  **What are "Tools"?**
    *   Imagine "tools" as specific actions your application or an AI can take, like "search the web," "send an email," or "get weather data."
    *   They are essentially wrappers around external API calls or custom code snippets.

2.  **Two Kinds of Tools:**
    *   **Built-in Tools:** These are pre-defined, hardcoded tools that come with the application (e.g., `tools` imported from `@/tools/registry`). Their configurations are static.
    *   **Custom Tools:** These are user-created tools. Their definitions (what parameters they take, what code they run) are dynamic and often stored in a database. They need to be fetched and configured on the fly.

3.  **The Lifecycle of Using a Tool:**
    *   **1. Find the Tool:** First, you need to locate the tool by its ID. This involves checking both built-in and custom tools.
    *   **2. Prepare Inputs:** The tool needs specific parameters (e.g., for a "weather" tool, `city` and `unit`). These parameters might come from a user, an AI, or other parts of a workflow. This file ensures they are correctly formatted and validated.
    *   **3. Format the Request:** Based on the tool's configuration and the prepared inputs, an HTTP request (URL, method, headers, body) is constructed.
    *   **4. Execute the Request:** The HTTP request is sent to the external service or a backend endpoint designed to run custom code.
    *   **5. Handle the Response:** The result from the external service is received. This file checks for errors, extracts the relevant data, and transforms it into a consistent output format.

4.  **Handling Custom Tools (The Most Complex Part):**
    *   Custom tools don't directly call an external API. Instead, their *code* is sent to a special **`/api/function/execute`** endpoint on your application's backend.
    *   This backend endpoint then runs the custom tool's code in a secure environment (like a serverless function or a sandbox).
    *   The request body for a custom tool execution is comprehensive: it includes the tool's code, the parameters provided, environment variables, workflow variables, and other context needed for the custom code to run correctly.
    *   Because custom tools can be fetched from a database, their retrieval needs to work both when the application is running in a web browser (client-side) and when it's processing things on a server (server-side).

5.  **Key Utility Functions:**
    *   `getTool` / `getToolAsync`: The primary entry points to look up a tool by ID. `getTool` is synchronous (mostly for client-side) while `getToolAsync` is asynchronous (for server-side or when fetching custom definitions).
    *   `formatRequestParams`: Builds the actual HTTP request details.
    *   `executeRequest`: Sends the HTTP request and processes the response.
    *   `createCustomToolRequestBody`: Specifically builds the payload for *executing custom tool code* on the backend.
    *   `validateRequiredParametersAfterMerge`: Ensures all necessary inputs are present before a tool is used.

In essence, this file provides the foundational infrastructure to integrate and manage various external capabilities, particularly focusing on the intricate details of dynamically defined, user-generated functions.

---

## Explanation of Each Line of Code

```typescript
import { createLogger } from '@/lib/logs/console/logger'
import { getBaseUrl } from '@/lib/urls/utils'
import { useCustomToolsStore } from '@/stores/custom-tools/store'
import { useEnvironmentStore } from '@/stores/settings/environment/store'
import { tools } from '@/tools/registry'
import type { TableRow, ToolConfig, ToolResponse } from '@/tools/types'
```
*   **`import { createLogger } from '@/lib/logs/console/logger'`**: Imports a function `createLogger` to create a logging instance, used for debugging and error reporting.
*   **`import { getBaseUrl } from '@/lib/urls/utils'`**: Imports `getBaseUrl`, a utility to get the base URL of the application, useful for constructing API endpoints regardless of the deployment environment.
*   **`import { useCustomToolsStore } from '@/stores/custom-tools/store'`**: Imports a Zustand store hook `useCustomToolsStore` for managing and accessing custom tools defined by users. This is typically used client-side.
*   **`import { useEnvironmentStore } from '@/stores/settings/environment/store'`**: Imports `useEnvironmentStore`, another Zustand store hook, used to access environment variables configured within the application, particularly on the client-side.
*   **`import { tools } from '@/tools/registry'`**: Imports a `tools` object, which is likely a registry (a collection/map) of all pre-defined, "built-in" tools available in the application.
*   **`import type { TableRow, ToolConfig, ToolResponse } from '@/tools/types'`**: Imports TypeScript type definitions that describe the structure of various data entities used in the tool system:
    *   `TableRow`: Represents a row in a table-like data structure, likely for UI inputs.
    *   `ToolConfig`: Defines the full configuration for a tool (ID, name, description, parameters, request details, response transformation).
    *   `ToolResponse`: Describes the expected output format after a tool has been executed (success status, output data, error details).

```typescript
const logger = createLogger('ToolsUtils')
```
*   **`const logger = createLogger('ToolsUtils')`**: Initializes a logger instance specifically named `'ToolsUtils'`. This allows for clearer log messages, indicating which part of the application is generating them.

```typescript
/**
 * Transforms a table from the store format to a key-value object
 * @param table Array of table rows from the store
 * @returns Record of key-value pairs
 */
export const transformTable = (table: TableRow[] | null): Record<string, any> => {
  if (!table) return {}

  return table.reduce(
    (acc, row) => {
      if (row.cells?.Key && row.cells?.Value !== undefined) {
        // Extract the Value cell as is - it should already be properly resolved
        // by the InputResolver based on variable type (number, string, boolean etc.)
        const value = row.cells.Value

        // Store the correctly typed value in the result object
        acc[row.cells.Key] = value
      }
      return acc
    },
    {} as Record<string, any>
  )
}
```
*   **`export const transformTable = (table: TableRow[] | null): Record<string, any> => { ... }`**: Defines an exported function `transformTable` that takes an array of `TableRow` objects (or `null`) and converts it into a plain JavaScript object where keys and values are extracted from the table rows.
    *   **`if (!table) return {}`**: If the input `table` is `null` or `undefined`, it immediately returns an empty object, preventing errors.
    *   **`return table.reduce(...)`**: It uses the `reduce` array method to iterate over each `row` in the `table` array and build up a single result object.
        *   **`(acc, row) => { ... }`**: This is the reducer function. `acc` (accumulator) is the object being built, and `row` is the current `TableRow` being processed.
        *   **`if (row.cells?.Key && row.cells?.Value !== undefined)`**: This condition checks if the current `row` has a `cells` property, and within `cells`, if both `Key` and `Value` properties exist and `Value` is not `undefined`. The `?.` (optional chaining) safely accesses `cells` and its properties.
        *   **`const value = row.cells.Value`**: If the condition is met, the `Value` from the cell is extracted. The comment explains that this value is expected to be already correctly typed (e.g., number, string, boolean) by an upstream "InputResolver."
        *   **`acc[row.cells.Key] = value`**: The extracted `Key` becomes a property name in the `acc` object, and the `value` becomes its corresponding value.
        *   **`return acc`**: The updated accumulator object is returned for the next iteration.
        *   **`{} as Record<string, any>`**: The initial value for the `acc` (accumulator) is an empty object, explicitly typed as `Record<string, any>`.

```typescript
interface RequestParams {
  url: string
  method: string
  headers: Record<string, string>
  body?: string
}
```
*   **`interface RequestParams { ... }`**: Defines a TypeScript interface `RequestParams` that specifies the structure for HTTP request parameters.
    *   **`url: string`**: The target URL for the request.
    *   **`method: string`**: The HTTP method (e.g., 'GET', 'POST', 'PUT').
    *   **`headers: Record<string, string>`**: An object mapping header names to their string values.
    *   **`body?: string`**: An optional request body, expected to be a string. The `?` makes it optional.

```typescript
/**
 * Format request parameters based on tool configuration and provided params
 */
export function formatRequestParams(tool: ToolConfig, params: Record<string, any>): RequestParams {
  // Process URL
  const url = typeof tool.request.url === 'function' ? tool.request.url(params) : tool.request.url

  // Process method
  const method =
    typeof tool.request.method === 'function'
      ? tool.request.method(params)
      : params.method || tool.request.method || 'GET'

  // Process headers
  const headers = tool.request.headers ? tool.request.headers(params) : {}

  // Process body
  const hasBody = method !== 'GET' && method !== 'HEAD' && !!tool.request.body
  const bodyResult = tool.request.body ? tool.request.body(params) : undefined

  // Special handling for NDJSON content type or 'application/x-www-form-urlencoded'
  const isPreformattedContent =
    headers['Content-Type'] === 'application/x-ndjson' ||
    headers['Content-Type'] === 'application/x-www-form-urlencoded'
  const body = hasBody
    ? isPreformattedContent && typeof bodyResult === 'string'
      ? bodyResult
      : JSON.stringify(bodyResult)
    : undefined

  return { url, method, headers, body }
}
```
*   **`export function formatRequestParams(...)`**: An exported function that takes a `ToolConfig` (the tool's definition) and `params` (the specific inputs for this execution) and returns a `RequestParams` object.
    *   **`const url = ...`**: Determines the request URL. It checks if `tool.request.url` is a function. If it is, it calls that function with the `params` to dynamically generate the URL; otherwise, it uses the static `tool.request.url` string.
    *   **`const method = ...`**: Determines the HTTP method. Similar dynamic logic: if `tool.request.method` is a function, call it. Otherwise, it prioritizes `params.method`, then `tool.request.method`, and finally defaults to `'GET'`.
    *   **`const headers = tool.request.headers ? tool.request.headers(params) : {}`**: Determines request headers. If `tool.request.headers` is defined (and assumed to be a function), it calls it with `params` to get dynamic headers; otherwise, it defaults to an empty object.
    *   **`const hasBody = method !== 'GET' && method !== 'HEAD' && !!tool.request.body`**: A boolean flag indicating if the request should have a body. It's true if the method is not 'GET' or 'HEAD' AND the `tool.request.body` configuration exists.
    *   **`const bodyResult = tool.request.body ? tool.request.body(params) : undefined`**: If `tool.request.body` is defined (and assumed to be a function), it calls it with `params` to generate the raw body content; otherwise, it's `undefined`.
    *   **`const isPreformattedContent = ...`**: A boolean flag that checks if the `Content-Type` header is either `application/x-ndjson` or `application/x-www-form-urlencoded`. These content types typically require the body to be a raw string, not JSON stringified.
    *   **`const body = hasBody ? (isPreformattedContent && typeof bodyResult === 'string' ? bodyResult : JSON.stringify(bodyResult)) : undefined`**: This is the core logic for the request body:
        *   If `hasBody` is true:
            *   If `isPreformattedContent` is true AND `bodyResult` is already a string, use `bodyResult` directly (it's already formatted).
            *   Otherwise (for most other content types like `application/json`), `JSON.stringify(bodyResult)` is used to convert the body object into a JSON string.
        *   If `hasBody` is false, the `body` is `undefined`.
    *   **`return { url, method, headers, body }`**: Returns the constructed `RequestParams` object.

```typescript
/**
 * Execute the actual request and transform the response
 */
export async function executeRequest(
  toolId: string,
  tool: ToolConfig,
  requestParams: RequestParams
): Promise<ToolResponse> {
  try {
    const { url, method, headers, body } = requestParams

    const externalResponse = await fetch(url, { method, headers, body })

    if (!externalResponse.ok) {
      let errorContent
      try {
        errorContent = await externalResponse.json()
      } catch (_e) {
        errorContent = { message: externalResponse.statusText }
      }

      const error = errorContent.message || `${toolId} API error: ${externalResponse.statusText}`
      logger.error(`${toolId} error:`, { error })
      throw new Error(error)
    }

    const transformResponse =
      tool.transformResponse ||
      (async (resp: Response) => ({
        success: true,
        output: await resp.json(),
      }))

    return await transformResponse(externalResponse)
  } catch (error: any) {
    return {
      success: false,
      output: {},
      error: error.message || 'Unknown error',
    }
  }
}
```
*   **`export async function executeRequest(...)`**: An exported `async` function responsible for executing the HTTP request and processing its response. It returns a `Promise<ToolResponse>`.
    *   **`try { ... } catch (error: any) { ... }`**: A `try...catch` block to gracefully handle any errors during the network request or response processing.
    *   **`const { url, method, headers, body } = requestParams`**: Destructures the `requestParams` object to easily access its properties.
    *   **`const externalResponse = await fetch(url, { method, headers, body })`**: Makes the actual HTTP request using the native `fetch` API. It `await`s the response.
    *   **`if (!externalResponse.ok) { ... }`**: Checks if the HTTP response status is *not* `ok` (i.e., status code 4xx or 5xx).
        *   **`let errorContent; try { errorContent = await externalResponse.json(); } catch (_e) { errorContent = { message: externalResponse.statusText }; }`**: Attempts to parse the error response body as JSON to get more details. If parsing fails (e.g., non-JSON error, no body), it falls back to using the HTTP `statusText`.
        *   **`const error = errorContent.message || ...`**: Constructs an error message, prioritizing any message from the `errorContent` JSON, otherwise using a generic API error message with the `toolId` and `statusText`.
        *   **`logger.error(...)`**: Logs the error using the previously initialized logger.
        *   **`throw new Error(error)`**: Throws a new `Error` object, which will be caught by the outer `catch` block.
    *   **`const transformResponse = tool.transformResponse || (async (resp: Response) => ({ ... }))`**: Determines the function to transform the successful response.
        *   It prioritizes `tool.transformResponse` (a custom transformation function defined in the tool's configuration).
        *   If `tool.transformResponse` is not provided, it uses a default asynchronous function that parses the response as JSON (`await resp.json()`) and wraps it in a `{ success: true, output: ... }` object.
    *   **`return await transformResponse(externalResponse)`**: Calls the chosen `transformResponse` function with the raw `externalResponse` and returns its result, which is expected to be a `ToolResponse`.
    *   **`catch (error: any) { ... }`**: If any error occurs in the `try` block (network error, custom error thrown), this block catches it.
        *   **`return { success: false, output: {}, error: error.message || 'Unknown error', }`**: Returns a `ToolResponse` indicating failure, with an empty `output` and an error message derived from the caught error.

```typescript
/**
 * Formats a parameter name for user-friendly error messages
 * Converts parameter names and descriptions to more readable format
 */
function formatParameterNameForError(paramName: string): string {
  // Split camelCase and snake_case/kebab-case into words, then capitalize first letter of each word
  return paramName
    .split(/(?=[A-Z])|[_-]/)
    .filter(Boolean)
    .map((word) => word.charAt(0).toUpperCase() + word.slice(1).toLowerCase())
    .join(' ')
}
```
*   **`function formatParameterNameForError(paramName: string): string { ... }`**: A private helper function (not `export`ed) that takes a technical parameter name (e.g., `someParamName`, `some_param_name`, `some-param-name`) and converts it into a more human-readable, title-cased format (e.g., "Some Param Name").
    *   **`paramName.split(/(?=[A-Z])|[_-]/)`**: Splits the string:
        *   `(?=[A-Z])`: Splits just before any uppercase letter (for camelCase).
        *   `|[_-]`: Splits on underscores `_` or hyphens `-` (for snake_case/kebab-case).
    *   **`.filter(Boolean)`**: Removes any empty strings that might result from the split (e.g., if there are consecutive delimiters).
    *   **`.map((word) => word.charAt(0).toUpperCase() + word.slice(1).toLowerCase())`**: Iterates over each word:
        *   `word.charAt(0).toUpperCase()`: Capitalizes the first letter.
        *   `word.slice(1).toLowerCase()`: Converts the rest of the word to lowercase.
    *   **`.join(' ')`**: Joins the processed words back together with spaces.

```typescript
/**
 * Validates required parameters after LLM and user params have been merged
 * This is the final validation before tool execution - ensures all required
 * user-or-llm parameters are present after the merge process
 */
export function validateRequiredParametersAfterMerge(
  toolId: string,
  tool: ToolConfig | undefined,
  params: Record<string, any>,
  parameterNameMap?: Record<string, string>
): void {
  if (!tool) {
    throw new Error(`Tool not found: ${toolId}`)
  }

  // Validate all required user-or-llm parameters after merge
  // user-only parameters should have been validated earlier during serialization
  for (const [paramName, paramConfig] of Object.entries(tool.params)) {
    if (
      (paramConfig as any).visibility === 'user-or-llm' &&
      paramConfig.required &&
      (!(paramName in params) ||
        params[paramName] === null ||
        params[paramName] === undefined ||
        params[paramName] === '')
    ) {
      // Create a more user-friendly error message
      const toolName = tool.name || toolId
      const friendlyParamName =
        parameterNameMap?.[paramName] || formatParameterNameForError(paramName)
      throw new Error(`"${friendlyParamName}" is required for ${toolName}`)
    }
  }
}
```
*   **`export function validateRequiredParametersAfterMerge(...)`**: An exported function that performs final validation on tool parameters *after* all potential sources (user input, AI model output) have been merged. It ensures all critically required parameters are present before a tool is executed.
    *   **`toolId: string, tool: ToolConfig | undefined, params: Record<string, any>, parameterNameMap?: Record<string, string>`**: Inputs include the tool's ID, its configuration (which might be `undefined`), the merged parameters, and an optional map for user-friendly parameter names.
    *   **`if (!tool) { throw new Error(...) }`**: If the `tool` configuration is not found, it throws an error.
    *   **`for (const [paramName, paramConfig] of Object.entries(tool.params)) { ... }`**: It iterates through each parameter defined in the `tool.params` configuration.
    *   **`if ((paramConfig as any).visibility === 'user-or-llm' && paramConfig.required && (!(...)))`**: This is the core validation condition:
        *   `(paramConfig as any).visibility === 'user-or-llm'`: Checks if the parameter's visibility is `user-or-llm`, meaning it can be provided by either the user or the Large Language Model (LLM). "User-only" parameters are assumed to have been validated earlier.
        *   `paramConfig.required`: Checks if the parameter is explicitly marked as required.
        *   `(!(paramName in params) || params[paramName] === null || params[paramName] === undefined || params[paramName] === '')`: Checks if the parameter is missing (`!in params`), or if its value is `null`, `undefined`, or an empty string.
    *   **`const toolName = tool.name || toolId`**: Gets a user-friendly tool name, preferring `tool.name` if available, otherwise falling back to `toolId`.
    *   **`const friendlyParamName = parameterNameMap?.[paramName] || formatParameterNameForError(paramName)`**: Gets a user-friendly parameter name, prioritizing a lookup in `parameterNameMap` (if provided) and falling back to `formatParameterNameForError`.
    *   **`throw new Error(...)`**: If a required parameter is missing or invalid, it throws a descriptive error message.

```typescript
/**
 * Creates parameter schema from custom tool schema
 */
export function createParamSchema(customTool: any): Record<string, any> {
  const params: Record<string, any> = {}

  if (customTool.schema.function?.parameters?.properties) {
    const properties = customTool.schema.function.parameters.properties
    const required = customTool.schema.function.parameters.required || []

    Object.entries(properties).forEach(([key, config]: [string, any]) => {
      const isRequired = required.includes(key)

      // Create the base parameter configuration
      const paramConfig: Record<string, any> = {
        type: config.type || 'string',
        required: isRequired,
        description: config.description || '',
      }

      // Set visibility based on whether it's required
      if (isRequired) {
        paramConfig.visibility = 'user-or-llm'
      } else {
        paramConfig.visibility = 'user-only'
      }

      params[key] = paramConfig
    })
  }

  return params
}
```
*   **`export function createParamSchema(customTool: any): Record<string, any> { ... }`**: An exported function that takes a raw `customTool` object (likely fetched from a database with its schema) and transforms its parameter definitions into the internal `Record<string, any>` format expected by the `ToolConfig`.
    *   **`const params: Record<string, any> = {}`**: Initializes an empty object to store the processed parameters.
    *   **`if (customTool.schema.function?.parameters?.properties) { ... }`**: Checks if the `customTool` has a well-defined schema path for parameters.
        *   **`const properties = customTool.schema.function.parameters.properties`**: Extracts the properties object, which defines each parameter.
        *   **`const required = customTool.schema.function.parameters.required || []`**: Extracts the array of required parameter names, defaulting to an empty array if not present.
        *   **`Object.entries(properties).forEach(([key, config]: [string, any]) => { ... })`**: Iterates over each parameter in the `properties` object.
            *   **`const isRequired = required.includes(key)`**: Checks if the current parameter's `key` is in the `required` array.
            *   **`const paramConfig: Record<string, any> = { ... }`**: Creates a base configuration object for the parameter:
                *   `type`: Defaults to `'string'` if not specified in `config`.
                *   `required`: Set based on `isRequired`.
                *   `description`: Defaults to `''` if not specified.
            *   **`if (isRequired) { paramConfig.visibility = 'user-or-llm' } else { paramConfig.visibility = 'user-only' }`**: Assigns the `visibility` property. If a parameter is required, it's considered `user-or-llm` (meaning an LLM can provide it if the user doesn't). Otherwise, it's `user-only`.
            *   **`params[key] = paramConfig`**: Adds the constructed `paramConfig` to the `params` object using its `key` as the property name.
    *   **`return params`**: Returns the fully constructed parameters object.

```typescript
/**
 * Get environment variables from store (client-side only)
 * @param getStore Optional function to get the store (useful for testing)
 */
export function getClientEnvVars(getStore?: () => any): Record<string, string> {
  if (typeof window === 'undefined') return {}

  try {
    // Allow injecting the store for testing
    const envStore = getStore ? getStore() : useEnvironmentStore.getState()
    const allEnvVars = envStore.getAllVariables()

    // Convert environment variables to a simple key-value object
    return Object.entries(allEnvVars).reduce(
      (acc, [key, variable]: [string, any]) => {
        acc[key] = variable.value
        return acc
      },
      {} as Record<string, string>
    )
  } catch (_error) {
    // In case of any errors (like in testing), return empty object
    return {}
  }
}
```
*   **`export function getClientEnvVars(...)`**: An exported function to retrieve environment variables specifically from the client-side store.
    *   **`if (typeof window === 'undefined') return {}`**: This is a crucial check. If `window` is `undefined`, it means the code is running on the server (Node.js environment), where the client-side store is not available. In this case, it returns an empty object.
    *   **`try { ... } catch (_error) { ... }`**: A `try...catch` block to handle potential errors, especially in unusual scenarios or testing.
    *   **`const envStore = getStore ? getStore() : useEnvironmentStore.getState()`**: Gets the environment store. It allows `getStore` to be injected (e.g., for mocking in tests); otherwise, it directly accesses the current state of `useEnvironmentStore`.
    *   **`const allEnvVars = envStore.getAllVariables()`**: Calls a method on the store to retrieve all environment variables.
    *   **`return Object.entries(allEnvVars).reduce(...)`**: Converts the structured `allEnvVars` (which might contain objects with `value` properties) into a flat `Record<string, string>` (key-value pairs) where each key maps to its `value`.
    *   **`catch (_error) { return {} }`**: If any error occurs (e.g., store not initialized), it returns an empty object.

```typescript
/**
 * Creates the request body configuration for custom tools
 * @param customTool The custom tool configuration
 * @param isClient Whether running on client side
 * @param workflowId Optional workflow ID for server-side
 * @param getStore Optional function to get the store (useful for testing)
 */
export function createCustomToolRequestBody(
  customTool: any,
  isClient = true,
  workflowId?: string,
  getStore?: () => any
) {
  return (params: Record<string, any>) => {
    // Get environment variables - try multiple sources in order of preference:
    // 1. envVars parameter (passed from provider/agent context)
    // 2. Client-side store (if running in browser)
    // 3. Empty object (fallback)
    const envVars = params.envVars || (isClient ? getClientEnvVars(getStore) : {})

    // Get workflow variables from params (passed from execution context)
    const workflowVariables = params.workflowVariables || {}

    // Get block data and mapping from params (passed from execution context)
    const blockData = params.blockData || {}
    const blockNameMapping = params.blockNameMapping || {}

    // Include everything needed for execution
    return {
      code: customTool.code,
      params: params, // These will be available in the VM context
      schema: customTool.schema.function.parameters, // For validation
      envVars: envVars, // Environment variables
      workflowVariables: workflowVariables, // Workflow variables for <variable.name> resolution
      blockData: blockData, // Runtime block outputs for <block.field> resolution
      blockNameMapping: blockNameMapping, // Block name to ID mapping
      workflowId: workflowId, // Pass workflowId for server-side context
      isCustomTool: true, // Flag to indicate this is a custom tool execution
    }
  }
}
```
*   **`export function createCustomToolRequestBody(...)`**: An exported factory function that *returns another function*. The returned function will then construct the actual request body for executing a custom tool on the backend `/api/function/execute` endpoint.
    *   **`customTool: any, isClient = true, workflowId?: string, getStore?: () => any`**: Parameters to the factory function. `isClient` defaults to `true`.
    *   **`return (params: Record<string, any>) => { ... }`**: This is the function that gets returned. It takes `params` (the specific inputs for *this* custom tool execution) as an argument.
    *   **`const envVars = params.envVars || (isClient ? getClientEnvVars(getStore) : {})`**: Determines environment variables to pass to the custom tool. It prioritizes `envVars` already present in the `params` (e.g., passed from an agent context). If not found, and `isClient` is true, it calls `getClientEnvVars`. Otherwise (server-side, or `isClient` is false), it defaults to an empty object.
    *   **`const workflowVariables = params.workflowVariables || {}`**: Extracts workflow-specific variables from the `params` (context).
    *   **`const blockData = params.blockData || {}`**: Extracts runtime outputs from other blocks in a workflow.
    *   **`const blockNameMapping = params.blockNameMapping || {}`**: Extracts mapping of block names to IDs.
    *   **`return { ... }`**: Constructs the comprehensive payload object that will be sent to the backend `/api/function/execute` endpoint for custom tool execution.
        *   **`code: customTool.code`**: The actual JavaScript/Python code of the custom tool.
        *   **`params: params`**: The specific input parameters for this tool call.
        *   **`schema: customTool.schema.function.parameters`**: The parameter schema, likely used by the backend for validation.
        *   **`envVars: envVars`**: Environment variables required by the custom tool's code.
        *   **`workflowVariables: workflowVariables`**: Variables from the workflow context.
        *   **`blockData: blockData`**: Data from previous blocks in a workflow.
        *   **`blockNameMapping: blockNameMapping`**: Block name-to-ID mappings for resolving data.
        *   **`workflowId: workflowId`**: The ID of the workflow, particularly relevant for server-side execution context.
        *   **`isCustomTool: true`**: A flag to indicate that this execution request is specifically for a custom tool.

```typescript
// Get a tool by its ID
export function getTool(toolId: string): ToolConfig | undefined {
  // Check for built-in tools
  const builtInTool = tools[toolId]
  if (builtInTool) return builtInTool

  // Check if it's a custom tool
  if (toolId.startsWith('custom_') && typeof window !== 'undefined') {
    // Only try to use the sync version on the client
    const customToolsStore = useCustomToolsStore.getState()
    const identifier = toolId.replace('custom_', '')

    // Try to find the tool directly by ID first
    let customTool = customToolsStore.getTool(identifier)

    // If not found by ID, try to find by title (for backward compatibility)
    if (!customTool) {
      const allTools = customToolsStore.getAllTools()
      customTool = allTools.find((tool) => tool.title === identifier)
    }

    if (customTool) {
      return createToolConfig(customTool, toolId)
    }
  }

  // If not found or running on the server, return undefined
  return undefined
}
```
*   **`export function getTool(toolId: string): ToolConfig | undefined { ... }`**: An exported *synchronous* function to retrieve a `ToolConfig` by its `toolId`. This version is primarily for client-side use where custom tool data might be immediately available from a local store.
    *   **`const builtInTool = tools[toolId]; if (builtInTool) return builtInTool`**: First, it checks if `toolId` matches any of the `built-in` tools in the `tools` registry. If found, it returns that configuration.
    *   **`if (toolId.startsWith('custom_') && typeof window !== 'undefined') { ... }`**: If the `toolId` starts with `custom_` (indicating a custom tool) AND the code is running in a browser environment (`typeof window !== 'undefined'`):
        *   **`const customToolsStore = useCustomToolsStore.getState()`**: Accesses the current state of the client-side `customToolsStore`.
        *   **`const identifier = toolId.replace('custom_', '')`**: Extracts the actual custom tool identifier by removing the `custom_` prefix.
        *   **`let customTool = customToolsStore.getTool(identifier)`**: Tries to find the custom tool by its ID directly from the store.
        *   **`if (!customTool) { ... }`**: If not found by ID, it tries to find it by its `title` from all available custom tools. This is for backward compatibility.
        *   **`if (customTool) { return createToolConfig(customTool, toolId) }`**: If a custom tool is found, it calls `createToolConfig` (a helper function defined later) to convert the raw custom tool definition into a standardized `ToolConfig` object, then returns it.
    *   **`return undefined`**: If the tool is not found as a built-in or a client-side custom tool, or if it's a custom tool lookup attempting to run on the server, it returns `undefined`.

```typescript
// Get a tool by its ID asynchronously (supports server-side)
export async function getToolAsync(
  toolId: string,
  workflowId?: string
): Promise<ToolConfig | undefined> {
  // Check for built-in tools
  const builtInTool = tools[toolId]
  if (builtInTool) return builtInTool

  // Check if it's a custom tool
  if (toolId.startsWith('custom_')) {
    return getCustomTool(toolId, workflowId)
  }

  return undefined
}
```
*   **`export async function getToolAsync(...)`**: An exported `async` function to retrieve a `ToolConfig` by its `toolId`, designed to work asynchronously and support server-side environments for custom tools.
    *   **`const builtInTool = tools[toolId]; if (builtInTool) return builtInTool`**: Same as `getTool`, checks for built-in tools first.
    *   **`if (toolId.startsWith('custom_')) { return getCustomTool(toolId, workflowId) }`**: If it's a custom tool, it delegates the fetching to the `getCustomTool` function (defined later), which handles fetching custom tool definitions from an API. It passes the optional `workflowId` for server context.
    *   **`return undefined`**: If not found.

```typescript
// Helper function to create a tool config from a custom tool
function createToolConfig(customTool: any, customToolId: string): ToolConfig {
  // Create a parameter schema from the custom tool schema
  const params = createParamSchema(customTool)

  // Create a tool config for the custom tool
  return {
    id: customToolId,
    name: customTool.title,
    description: customTool.schema.function?.description || '',
    version: '1.0.0',
    params,

    // Request configuration - for custom tools we'll use the execute endpoint
    request: {
      url: '/api/function/execute',
      method: 'POST',
      headers: () => ({ 'Content-Type': 'application/json' }),
      body: createCustomToolRequestBody(customTool, true),
    },

    // Standard response handling for custom tools
    transformResponse: async (response: Response) => {
      const data = await response.json()

      if (!data.success) {
        throw new Error(data.error || 'Custom tool execution failed')
      }

      return {
        success: true,
        output: data.output.result || data.output,
        error: undefined,
      }
    },
  }
}
```
*   **`function createToolConfig(customTool: any, customToolId: string): ToolConfig { ... }`**: A private helper function that takes a raw `customTool` definition and its full `customToolId` and constructs a complete `ToolConfig` object for it. This is specifically used by `getTool` for *client-side* custom tools.
    *   **`const params = createParamSchema(customTool)`**: Calls `createParamSchema` to generate the structured parameters configuration.
    *   **`return { ... }`**: Returns the `ToolConfig` object:
        *   **`id: customToolId`**: The full ID (e.g., `custom_mytool`).
        *   **`name: customTool.title`**: The user-friendly name.
        *   **`description: customTool.schema.function?.description || ''`**: Description from the schema.
        *   **`version: '1.0.0'`**: A default version.
        *   **`params`**: The generated parameters configuration.
        *   **`request: { ... }`**: The HTTP request configuration for *executing* this custom tool.
            *   **`url: '/api/function/execute'`**: Custom tools are executed by calling a dedicated backend endpoint.
            *   **`method: 'POST'`**: Always a POST request for execution.
            *   **`headers: () => ({ 'Content-Type': 'application/json' })`**: Sets the content type to JSON.
            *   **`body: createCustomToolRequestBody(customTool, true)`**: Calls `createCustomToolRequestBody` to get the function that will generate the request body for execution. `true` is passed for `isClient`.
        *   **`transformResponse: async (response: Response) => { ... }`**: Defines how the response from the `/api/function/execute` endpoint should be handled.
            *   **`const data = await response.json()`**: Parses the response as JSON.
            *   **`if (!data.success) { throw new Error(...) }`**: If the execution on the backend was not `success`ful, it throws an error.
            *   **`return { success: true, output: data.output.result || data.output, error: undefined, }`**: If successful, it returns a `ToolResponse`, extracting the output (prioritizing `data.output.result` if available, otherwise `data.output`).

```typescript
// Create a tool config from a custom tool definition
async function getCustomTool(
  customToolId: string,
  workflowId?: string
): Promise<ToolConfig | undefined> {
  const identifier = customToolId.replace('custom_', '')

  try {
    const baseUrl = getBaseUrl()
    const url = new URL('/api/tools/custom', baseUrl)

    // Add workflowId as a query parameter if available
    if (workflowId) {
      url.searchParams.append('workflowId', workflowId)
    }

    const response = await fetch(url.toString())

    if (!response.ok) {
      logger.error(`Failed to fetch custom tools: ${response.statusText}`)
      return undefined
    }

    const result = await response.json()

    if (!result.data || !Array.isArray(result.data)) {
      logger.error(`Invalid response when fetching custom tools: ${JSON.stringify(result)}`)
      return undefined
    }

    // Try to find the tool by ID or title
    const customTool = result.data.find(
      (tool: any) => tool.id === identifier || tool.title === identifier
    )

    if (!customTool) {
      logger.error(`Custom tool not found: ${identifier}`)
      return undefined
    }

    // Create a parameter schema
    const params = createParamSchema(customTool)

    // Create a tool config for the custom tool
    return {
      id: customToolId,
      name: customTool.title,
      description: customTool.schema.function?.description || '',
      version: '1.0.0',
      params,

      // Request configuration - for custom tools we'll use the execute endpoint
      request: {
        url: '/api/function/execute',
        method: 'POST',
        headers: () => ({ 'Content-Type': 'application/json' }),
        body: createCustomToolRequestBody(customTool, false, workflowId),
      },

      // Same response handling as client-side
      transformResponse: async (response: Response) => {
        const data = await response.json()

        if (!data.success) {
          throw new Error(data.error || 'Custom tool execution failed')
        }

        return {
          success: true,
          output: data.output.result || data.output,
          error: undefined,
        }
      },
    }
  } catch (error) {
    logger.error(`Error fetching custom tool ${identifier} from API:`, error)
    return undefined
  }
}
```
*   **`async function getCustomTool(...)`**: A private `async` function responsible for fetching a custom tool's definition from a backend API. This is used by `getToolAsync` to support both client and server environments.
    *   **`const identifier = customToolId.replace('custom_', '')`**: Extracts the clean identifier.
    *   **`try { ... } catch (error) { ... }`**: Error handling for network requests.
    *   **`const baseUrl = getBaseUrl()`**: Gets the application's base URL.
    *   **`const url = new URL('/api/tools/custom', baseUrl)`**: Constructs the URL for the custom tools API endpoint.
    *   **`if (workflowId) { url.searchParams.append('workflowId', workflowId) }`**: If a `workflowId` is provided, it's added as a query parameter. This is useful for backend context when fetching custom tools that might be specific to a workflow.
    *   **`const response = await fetch(url.toString())`**: Makes the API call to fetch custom tool definitions.
    *   **`if (!response.ok) { ... }`**: Handles non-OK HTTP responses, logging an error and returning `undefined`.
    *   **`const result = await response.json()`**: Parses the API response as JSON.
    *   **`if (!result.data || !Array.isArray(result.data)) { ... }`**: Validates the structure of the API response, expecting an array of `data`. Logs an error and returns `undefined` if invalid.
    *   **`const customTool = result.data.find(...)`**: Searches the fetched array of custom tool definitions to find the one matching the `identifier` (by `id` or `title` for backward compatibility).
    *   **`if (!customTool) { ... }`**: If the specific custom tool is not found in the fetched list, it logs an error and returns `undefined`.
    *   **`const params = createParamSchema(customTool)`**: Generates the parameter schema.
    *   **`return { ... }`**: Constructs and returns the `ToolConfig` object, very similar to `createToolConfig`.
        *   **`request: { ... }`**: The `request` property is configured to point to `/api/function/execute`.
        *   **`body: createCustomToolRequestBody(customTool, false, workflowId)`**: Calls `createCustomToolRequestBody` to get the function for generating the execution body. Here, `false` is passed for `isClient` because this function (`getCustomTool`) can run on the server, and `workflowId` is explicitly passed.
        *   **`transformResponse: async (response: Response) => { ... }`**: The `transformResponse` is identical to the one in `createToolConfig`, handling the execution endpoint's response.
    *   **`catch (error) { logger.error(...); return undefined }`**: Catches any errors during the fetch or processing, logs them, and returns `undefined`.