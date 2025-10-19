```typescript
import { NextResponse } from 'next/server'
import type { McpApiResponse } from '@/lib/mcp/types'

/**
 * MCP-specific constants
 */
export const MCP_CONSTANTS = {
  EXECUTION_TIMEOUT: 60000,
  CACHE_TIMEOUT: 5 * 60 * 1000,
  DEFAULT_RETRIES: 3,
  DEFAULT_CONNECTION_TIMEOUT: 30000,
} as const

/**
 * Client-safe MCP constants
 */
export const MCP_CLIENT_CONSTANTS = {
  CLIENT_TIMEOUT: 60000,
  AUTO_REFRESH_INTERVAL: 5 * 60 * 1000,
} as const

/**
 * Create standardized MCP error response
 */
export function createMcpErrorResponse(
  error: unknown,
  defaultMessage: string,
  status = 500
): NextResponse {
  const errorMessage = error instanceof Error ? error.message : defaultMessage

  const response: McpApiResponse = {
    success: false,
    error: errorMessage,
  }

  return NextResponse.json(response, { status })
}

/**
 * Create standardized MCP success response
 */
export function createMcpSuccessResponse<T>(data: T, status = 200): NextResponse {
  const response: McpApiResponse<T> = {
    success: true,
    data,
  }

  return NextResponse.json(response, { status })
}

/**
 * Validate string parameter
 * Consolidates parameter validation logic found across routes
 */
export function validateStringParam(
  value: unknown,
  paramName: string
): { isValid: true } | { isValid: false; error: string } {
  if (!value || typeof value !== 'string') {
    return {
      isValid: false,
      error: `${paramName} is required and must be a string`,
    }
  }
  return { isValid: true }
}

/**
 * Validate required fields in request body
 */
export function validateRequiredFields(
  body: Record<string, unknown>,
  requiredFields: string[]
): { isValid: true } | { isValid: false; error: string } {
  const missingFields = requiredFields.filter((field) => !(field in body))

  if (missingFields.length > 0) {
    return {
      isValid: false,
      error: `Missing required fields: ${missingFields.join(', ')}`,
    }
  }

  return { isValid: true }
}

/**
 * Enhanced error categorization for more specific HTTP status codes
 */
export function categorizeError(error: unknown): { message: string; status: number } {
  if (!(error instanceof Error)) {
    return { message: 'Unknown error occurred', status: 500 }
  }

  const message = error.message.toLowerCase()

  if (message.includes('timeout')) {
    return { message: 'Request timed out', status: 408 }
  }

  if (message.includes('not found') || message.includes('not accessible')) {
    return { message: error.message, status: 404 }
  }

  if (message.includes('authentication') || message.includes('unauthorized')) {
    return { message: 'Authentication required', status: 401 }
  }

  if (
    message.includes('invalid') ||
    message.includes('missing required') ||
    message.includes('validation')
  ) {
    return { message: error.message, status: 400 }
  }

  return { message: error.message, status: 500 }
}

/**
 * Create standardized MCP tool ID from server ID and tool name
 */
export function createMcpToolId(serverId: string, toolName: string): string {
  const normalizedServerId = serverId.startsWith('mcp-') ? serverId : `mcp-${serverId}`
  return `${normalizedServerId}-${toolName}`
}

/**
 * Parse MCP tool ID to extract server ID and tool name
 */
export function parseMcpToolId(toolId: string): { serverId: string; toolName: string } {
  const parts = toolId.split('-')
  if (parts.length < 3 || parts[0] !== 'mcp') {
    throw new Error(`Invalid MCP tool ID format: ${toolId}. Expected: mcp-serverId-toolName`)
  }

  const serverId = `${parts[0]}-${parts[1]}`
  const toolName = parts.slice(2).join('-')

  return { serverId, toolName }
}
```

### Purpose of this file

This TypeScript file provides utility functions and constants specifically designed for interacting with the MCP (presumably, a service or platform within the application).  It offers standardized ways to:

- Handle responses (both success and error)
- Validate input parameters
- Categorize errors for better HTTP status codes
- Generate and parse MCP tool IDs

The goal is to promote consistency, reduce boilerplate code, and improve error handling across different parts of the application that interact with the MCP service.

### Detailed Explanation

**1. Imports:**

```typescript
import { NextResponse } from 'next/server'
import type { McpApiResponse } from '@/lib/mcp/types'
```

- `NextResponse` from `next/server`:  This is a class from Next.js used to create HTTP responses within server-side functions (API routes, server actions, etc.).  It allows you to set the status code, headers, and body of the response.
- `McpApiResponse` from `'@/lib/mcp/types'`:  This imports a type definition for the structure of MCP API responses.  It's assumed that this type defines the format of data returned by MCP endpoints, likely including fields for `success`, `data` (in case of success), and `error` (in case of failure).

**2. MCP Constants:**

```typescript
/**
 * MCP-specific constants
 */
export const MCP_CONSTANTS = {
  EXECUTION_TIMEOUT: 60000,
  CACHE_TIMEOUT: 5 * 60 * 1000,
  DEFAULT_RETRIES: 3,
  DEFAULT_CONNECTION_TIMEOUT: 30000,
} as const
```

- `MCP_CONSTANTS`:  This object defines several constants related to MCP interactions.
    - `EXECUTION_TIMEOUT`: Sets a timeout for MCP operations in milliseconds (60 seconds).
    - `CACHE_TIMEOUT`: Defines how long to cache data retrieved from MCP, also in milliseconds (5 minutes).
    - `DEFAULT_RETRIES`: Specifies the number of times to retry a failed MCP request.
    - `DEFAULT_CONNECTION_TIMEOUT`: Sets the timeout for establishing a connection to the MCP service, in milliseconds (30 seconds).
- `as const`: This tells TypeScript to treat this object as a *constant* object.  This means:
    - The values of the properties are treated as literal types (e.g., `60000` is treated as the literal number 60000, rather than just `number`).
    - The object is deeply immutable (you can't change its properties). This improves type safety and allows for compiler optimizations.

**3. MCP Client Constants:**

```typescript
/**
 * Client-safe MCP constants
 */
export const MCP_CLIENT_CONSTANTS = {
  CLIENT_TIMEOUT: 60000,
  AUTO_REFRESH_INTERVAL: 5 * 60 * 1000,
} as const
```

- `MCP_CLIENT_CONSTANTS`: This object defines constants specifically intended for use on the client-side (e.g., in a browser).
    - `CLIENT_TIMEOUT`: Similar to `EXECUTION_TIMEOUT`, but applicable to client-side MCP operations.
    - `AUTO_REFRESH_INTERVAL`: Sets the interval in milliseconds (5 minutes) at which client-side data from MCP should be automatically refreshed.  This is useful for keeping data up-to-date in the user interface.
- `as const`:  Same meaning as explained above.

**4. `createMcpErrorResponse` Function:**

```typescript
/**
 * Create standardized MCP error response
 */
export function createMcpErrorResponse(
  error: unknown,
  defaultMessage: string,
  status = 500
): NextResponse {
  const errorMessage = error instanceof Error ? error.message : defaultMessage

  const response: McpApiResponse = {
    success: false,
    error: errorMessage,
  }

  return NextResponse.json(response, { status })
}
```

- This function creates a standardized error response to be sent back to the client when an error occurs during an MCP operation.
- **Parameters:**
    - `error: unknown`:  The error that occurred. Using `unknown` is a safe practice because it forces you to handle the type explicitly.
    - `defaultMessage: string`: A fallback error message to use if the `error` doesn't have a message property (e.g., if it's not an `Error` object).
    - `status = 500`: The HTTP status code for the error response.  Defaults to 500 (Internal Server Error).
- **Logic:**
    - `const errorMessage = error instanceof Error ? error.message : defaultMessage`: This line extracts the error message. It uses a ternary operator to check if the `error` is an instance of the `Error` class. If it is, it uses the `error.message` property. Otherwise, it uses the `defaultMessage` provided as an argument.
    - `const response: McpApiResponse = { success: false, error: errorMessage }`:  This creates an object conforming to the `McpApiResponse` type, setting `success` to `false` and the `error` field to the extracted error message.
    - `return NextResponse.json(response, { status })`: This uses `NextResponse.json()` to create a JSON response with the `response` object as the body and sets the HTTP status code to the specified `status`.

**5. `createMcpSuccessResponse` Function:**

```typescript
/**
 * Create standardized MCP success response
 */
export function createMcpSuccessResponse<T>(data: T, status = 200): NextResponse {
  const response: McpApiResponse<T> = {
    success: true,
    data,
  }

  return NextResponse.json(response, { status })
}
```

- This function creates a standardized success response to be sent back to the client when an MCP operation is successful.
- **Parameters:**
    - `data: T`: The data to be returned in the response.  The `<T>` indicates that this is a generic function, meaning the type of `data` can be anything, and it will be inferred by TypeScript.  The `McpApiResponse<T>` type will also use this type.
    - `status = 200`: The HTTP status code for the success response. Defaults to 200 (OK).
- **Logic:**
    - `const response: McpApiResponse<T> = { success: true, data }`: Creates an object conforming to the `McpApiResponse<T>` type, setting `success` to `true` and the `data` field to the provided `data`.
    - `return NextResponse.json(response, { status })`: Creates a JSON response using `NextResponse.json()` with the `response` object as the body and sets the HTTP status code.

**6. `validateStringParam` Function:**

```typescript
/**
 * Validate string parameter
 * Consolidates parameter validation logic found across routes
 */
export function validateStringParam(
  value: unknown,
  paramName: string
): { isValid: true } | { isValid: false; error: string } {
  if (!value || typeof value !== 'string') {
    return {
      isValid: false,
      error: `${paramName} is required and must be a string`,
    }
  }
  return { isValid: true }
}
```

- This function validates that a given parameter is a non-empty string.
- **Parameters:**
    - `value: unknown`: The value to validate. `unknown` is used to ensure type safety.
    - `paramName: string`: The name of the parameter being validated (used in the error message).
- **Logic:**
    - `if (!value || typeof value !== 'string')`: This checks if the value is either falsy (e.g., `null`, `undefined`, `''`, `0`) or not a string.
    - If the validation fails, it returns an object with `isValid: false` and an `error` message.
    - If the validation passes, it returns an object with `isValid: true`.  This pattern (returning a union type) is a common way to signal success or failure with associated data.

**7. `validateRequiredFields` Function:**

```typescript
/**
 * Validate required fields in request body
 */
export function validateRequiredFields(
  body: Record<string, unknown>,
  requiredFields: string[]
): { isValid: true } | { isValid: false; error: string } {
  const missingFields = requiredFields.filter((field) => !(field in body))

  if (missingFields.length > 0) {
    return {
      isValid: false,
      error: `Missing required fields: ${missingFields.join(', ')}`,
    }
  }

  return { isValid: true }
}
```

- This function validates that all required fields are present in a request body.
- **Parameters:**
    - `body: Record<string, unknown>`:  The request body, represented as a dictionary (object) where keys are strings and values can be of any type (`unknown` is used for type safety).
    - `requiredFields: string[]`: An array of strings representing the names of the fields that are required in the body.
- **Logic:**
    - `const missingFields = requiredFields.filter((field) => !(field in body))`:  This line uses the `filter` method to create a new array containing only the fields from `requiredFields` that are *not* present as keys in the `body` object.  The `!(field in body)` expression checks if a key exists in the object.
    - If `missingFields` has any elements (meaning there are missing fields), it returns an object with `isValid: false` and an error message listing the missing fields.
    - Otherwise, it returns an object with `isValid: true`.

**8. `categorizeError` Function:**

```typescript
/**
 * Enhanced error categorization for more specific HTTP status codes
 */
export function categorizeError(error: unknown): { message: string; status: number } {
  if (!(error instanceof Error)) {
    return { message: 'Unknown error occurred', status: 500 }
  }

  const message = error.message.toLowerCase()

  if (message.includes('timeout')) {
    return { message: 'Request timed out', status: 408 }
  }

  if (message.includes('not found') || message.includes('not accessible')) {
    return { message: error.message, status: 404 }
  }

  if (message.includes('authentication') || message.includes('unauthorized')) {
    return { message: 'Authentication required', status: 401 }
  }

  if (
    message.includes('invalid') ||
    message.includes('missing required') ||
    message.includes('validation')
  ) {
    return { message: error.message, status: 400 }
  }

  return { message: error.message, status: 500 }
}
```

- This function attempts to categorize an error based on its message and assigns a more specific HTTP status code than a generic 500 error.  This can provide clients with more information about the nature of the error.
- **Parameter:**
    - `error: unknown`: The error to categorize.
- **Logic:**
    - `if (!(error instanceof Error))`:  First, it checks if the error is an instance of the `Error` class.  If it's not, it returns a default error object with a generic message and a 500 status code.
    - `const message = error.message.toLowerCase()`:  It extracts the error message and converts it to lowercase for case-insensitive matching.
    - A series of `if` statements then check if the lowercase message contains certain keywords:
        - `'timeout'`:  Returns a 408 (Request Timeout) status.
        - `'not found'` or `'not accessible'`: Returns a 404 (Not Found) status.
        - `'authentication'` or `'unauthorized'`: Returns a 401 (Unauthorized) status.
        - `'invalid'`, `'missing required'`, or `'validation'`: Returns a 400 (Bad Request) status.
    - If none of the above conditions are met, it returns a default error object with the original error message and a 500 status code.

**9. `createMcpToolId` Function:**

```typescript
/**
 * Create standardized MCP tool ID from server ID and tool name
 */
export function createMcpToolId(serverId: string, toolName: string): string {
  const normalizedServerId = serverId.startsWith('mcp-') ? serverId : `mcp-${serverId}`
  return `${normalizedServerId}-${toolName}`
}
```

- This function creates a standardized MCP tool ID by combining a server ID and a tool name. The tool ID format is `mcp-serverId-toolName`. It ensures the `serverId` begins with `mcp-`.
- **Parameters:**
    - `serverId: string`: The ID of the MCP server.
    - `toolName: string`: The name of the tool.
- **Logic:**
    - `const normalizedServerId = serverId.startsWith('mcp-') ? serverId : `mcp-${serverId}`: This line checks if the `serverId` already starts with `mcp-`. If it does, it uses the original `serverId`. Otherwise, it prepends `mcp-` to it.
    - `return `${normalizedServerId}-${toolName}`: This line concatenates the normalized server ID and the tool name, separated by a hyphen, to create the MCP tool ID.

**10. `parseMcpToolId` Function:**

```typescript
/**
 * Parse MCP tool ID to extract server ID and tool name
 */
export function parseMcpToolId(toolId: string): { serverId: string; toolName: string } {
  const parts = toolId.split('-')
  if (parts.length < 3 || parts[0] !== 'mcp') {
    throw new Error(`Invalid MCP tool ID format: ${toolId}. Expected: mcp-serverId-toolName`)
  }

  const serverId = `${parts[0]}-${parts[1]}`
  const toolName = parts.slice(2).join('-')

  return { serverId, toolName }
}
```

- This function parses an MCP tool ID (in the format `mcp-serverId-toolName`) and extracts the server ID and tool name.
- **Parameter:**
    - `toolId: string`: The MCP tool ID to parse.
- **Logic:**
    - `const parts = toolId.split('-')`:  This splits the `toolId` string into an array of substrings, using the hyphen as a delimiter.
    - `if (parts.length < 3 || parts[0] !== 'mcp')`:  This checks if the tool ID is in the correct format:
        - `parts.length < 3`:  Ensures that there are at least three parts (mcp, serverId, and toolName).
        - `parts[0] !== 'mcp'`:  Verifies that the first part is "mcp".
        - If either of these conditions is true, it throws an error indicating that the tool ID is invalid.
    - `const serverId = `${parts[0]}-${parts[1]}`: This line reconstructs the `serverId` by joining the first two parts (`mcp` and the actual server ID) with a hyphen.
    - `const toolName = parts.slice(2).join('-')`: This line extracts the `toolName` by taking all parts of the array starting from the third element (`parts.slice(2)`) and joining them back together with a hyphen. This handles cases where the tool name itself contains hyphens.
    - It returns an object containing the extracted `serverId` and `toolName`.

### Summary

This file provides a set of utility functions and constants designed to simplify and standardize interactions with an MCP service.  It handles response formatting, input validation, error categorization, and ID generation/parsing, making it easier to build and maintain code that relies on the MCP. The consistent use of types and error handling promotes robust and predictable behavior.
