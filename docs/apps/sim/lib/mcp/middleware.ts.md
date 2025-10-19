```typescript
import type { NextRequest, NextResponse } from 'next/server'
import { checkHybridAuth } from '@/lib/auth/hybrid'
import { createLogger } from '@/lib/logs/console/logger'
import { createMcpErrorResponse } from '@/lib/mcp/utils'
import { getUserEntityPermissions } from '@/lib/permissions/utils'
import { generateRequestId } from '@/lib/utils'

const logger = createLogger('McpAuthMiddleware')

export type McpPermissionLevel = 'read' | 'write' | 'admin'

export interface McpAuthContext {
  userId: string
  workspaceId: string
  requestId: string
}

export type McpRouteHandler = (
  request: NextRequest,
  context: McpAuthContext,
  ...args: any[]
) => Promise<NextResponse>

interface AuthResult {
  success: true
  context: McpAuthContext
}

interface AuthFailure {
  success: false
  errorResponse: NextResponse
}

type AuthValidationResult = AuthResult | AuthFailure

/**
 * Validates MCP authentication and authorization
 */
async function validateMcpAuth(
  request: NextRequest,
  permissionLevel: McpPermissionLevel
): Promise<AuthValidationResult> {
  const requestId = generateRequestId()

  try {
    const auth = await checkHybridAuth(request, { requireWorkflowId: false })
    if (!auth.success || !auth.userId) {
      logger.warn(`[${requestId}] Authentication failed: ${auth.error}`)
      return {
        success: false,
        errorResponse: createMcpErrorResponse(
          new Error(auth.error || 'Authentication required'),
          'Authentication failed',
          401
        ),
      }
    }

    let workspaceId: string | null = null

    const { searchParams } = new URL(request.url)
    workspaceId = searchParams.get('workspaceId')

    if (!workspaceId) {
      try {
        const contentType = request.headers.get('content-type')
        if (contentType?.includes('application/json')) {
          const body = await request.json()
          workspaceId = body.workspaceId
          ;(request as any)._parsedBody = body
        }
      } catch (error) {
        logger.debug(`[${requestId}] Could not parse request body for workspaceId extraction`)
      }
    }

    if (!workspaceId) {
      return {
        success: false,
        errorResponse: createMcpErrorResponse(
          new Error('workspaceId is required'),
          'Missing required parameter',
          400
        ),
      }
    }

    const userPermissions = await getUserEntityPermissions(auth.userId, 'workspace', workspaceId)
    if (!userPermissions) {
      return {
        success: false,
        errorResponse: createMcpErrorResponse(
          new Error('Access denied to workspace'),
          'Insufficient permissions',
          403
        ),
      }
    }

    const hasRequiredPermission = checkPermissionLevel(userPermissions, permissionLevel)
    if (!hasRequiredPermission) {
      const permissionError = getPermissionErrorMessage(permissionLevel)
      return {
        success: false,
        errorResponse: createMcpErrorResponse(
          new Error(permissionError),
          'Insufficient permissions',
          403
        ),
      }
    }

    return {
      success: true,
      context: {
        userId: auth.userId,
        workspaceId,
        requestId,
      },
    }
  } catch (error) {
    logger.error(`[${requestId}] Error during MCP auth validation:`, error)
    return {
      success: false,
      errorResponse: createMcpErrorResponse(
        error instanceof Error ? error : new Error('Authentication validation failed'),
        'Authentication validation failed',
        500
      ),
    }
  }
}

/**
 * Check if user has required permission level
 */
function checkPermissionLevel(userPermission: string, requiredLevel: McpPermissionLevel): boolean {
  switch (requiredLevel) {
    case 'read':
      return ['read', 'write', 'admin'].includes(userPermission)
    case 'write':
      return ['write', 'admin'].includes(userPermission)
    case 'admin':
      return userPermission === 'admin'
    default:
      return false
  }
}

/**
 * Get appropriate error message for permission level
 */
function getPermissionErrorMessage(permissionLevel: McpPermissionLevel): string {
  switch (permissionLevel) {
    case 'read':
      return 'Workspace access required for MCP operations'
    case 'write':
      return 'Write or admin permission required for MCP server management'
    case 'admin':
      return 'Admin permission required for MCP server administration'
    default:
      return 'Insufficient permissions for MCP operation'
  }
}

/**
 * Higher-order function that wraps MCP route handlers with authentication middleware
 *
 * @param permissionLevel - Required permission level ('read', 'write', or 'admin')
 * @returns Middleware wrapper function
 *
 */
export function withMcpAuth(permissionLevel: McpPermissionLevel = 'read') {
  return function middleware(handler: McpRouteHandler) {
    return async function wrappedHandler(
      request: NextRequest,
      ...args: any[]
    ): Promise<NextResponse> {
      const authResult = await validateMcpAuth(request, permissionLevel)

      if (!authResult.success) {
        return (authResult as AuthFailure).errorResponse
      }

      try {
        return await handler(request, (authResult as AuthResult).context, ...args)
      } catch (error) {
        logger.error(
          `[${(authResult as AuthResult).context.requestId}] Error in MCP route handler:`,
          error
        )
        return createMcpErrorResponse(
          error instanceof Error ? error : new Error('Internal server error'),
          'Internal server error',
          500
        )
      }
    }
  }
}

/**
 * Utility to get parsed request body
 */
export function getParsedBody(request: NextRequest): any {
  return (request as any)._parsedBody
}
```

### Purpose of this file

This TypeScript file implements an authentication and authorization middleware for Next.js API routes related to an MCP (Management Control Plane) system.  It ensures that only authenticated and authorized users can access specific routes based on their permission level within a given workspace.

### Simplification of Complex Logic

The primary complexity lies in the `validateMcpAuth` function, which orchestrates several steps:

1.  **Authentication:** Verifies the user's identity.
2.  **Workspace Identification:** Extracts the `workspaceId` from either the query parameters or the request body.
3.  **Authorization:** Checks if the user has the required permission level for the specified workspace.
4.  **Error Handling:**  Gracefully handles authentication and authorization failures, returning appropriate error responses.

The code is simplified by:

*   **Modularization:**  Breaking down the logic into smaller, well-defined functions (`validateMcpAuth`, `checkPermissionLevel`, `getPermissionErrorMessage`).
*   **Type Safety:**  Using TypeScript types extensively to improve code clarity and prevent runtime errors.  The `AuthResult`, `AuthFailure`, and `AuthValidationResult` types clearly define the possible outcomes of the authentication process.
*   **Clear Error Handling:** Using try/catch blocks and `createMcpErrorResponse` to generate consistent and informative error responses.
*   **Centralized Logging:** Using a logger to track authentication attempts, errors, and other important events.
*   **Higher-Order Function:** The `withMcpAuth` function uses a higher-order function to wrap route handlers with the authentication and authorization logic, making it reusable and easy to apply to multiple routes.

### Code Explanation (Line by Line)

```typescript
import type { NextRequest, NextResponse } from 'next/server'
import { checkHybridAuth } from '@/lib/auth/hybrid'
import { createLogger } from '@/lib/logs/console/logger'
import { createMcpErrorResponse } from '@/lib/mcp/utils'
import { getUserEntityPermissions } from '@/lib/permissions/utils'
import { generateRequestId } from '@/lib/utils'
```

*   **Imports:** Imports necessary modules and types from external libraries and internal modules.
    *   `NextRequest`, `NextResponse`: Types for handling Next.js API requests and responses.
    *   `checkHybridAuth`:  A function from the project's authentication library to verify the user's identity using a hybrid authentication mechanism (likely involving both API keys and user sessions).
    *   `createLogger`:  A function to create a logger instance for the middleware.
    *   `createMcpErrorResponse`:  A function to create standardized error responses for the MCP system.
    *   `getUserEntityPermissions`:  A function to retrieve the user's permissions for a specific entity (in this case, a workspace).
    *   `generateRequestId`: A function to generate unique request IDs.

```typescript
const logger = createLogger('McpAuthMiddleware')
```

*   **Logger Initialization:** Creates a logger instance named 'McpAuthMiddleware' for logging events related to this middleware.  This allows for targeted logging and debugging.

```typescript
export type McpPermissionLevel = 'read' | 'write' | 'admin'
```

*   **Type Definition: `McpPermissionLevel`:** Defines a type `McpPermissionLevel` as a string literal type, restricting its values to 'read', 'write', or 'admin'.  This ensures that permission levels are always valid.

```typescript
export interface McpAuthContext {
  userId: string
  workspaceId: string
  requestId: string
}
```

*   **Interface Definition: `McpAuthContext`:** Defines an interface `McpAuthContext` representing the authentication context that will be passed to the route handler.  It contains the authenticated user's ID, the workspace ID, and the unique request ID.

```typescript
export type McpRouteHandler = (
  request: NextRequest,
  context: McpAuthContext,
  ...args: any[]
) => Promise<NextResponse>
```

*   **Type Definition: `McpRouteHandler`:** Defines a type `McpRouteHandler` for the function that will handle the actual route logic *after* authentication and authorization. It takes a `NextRequest`, an `McpAuthContext`, and any additional arguments, and returns a `Promise` that resolves to a `NextResponse`.

```typescript
interface AuthResult {
  success: true
  context: McpAuthContext
}

interface AuthFailure {
  success: false
  errorResponse: NextResponse
}

type AuthValidationResult = AuthResult | AuthFailure
```

*   **Interface Definitions: `AuthResult`, `AuthFailure`, `AuthValidationResult`:** Defines types for the result of the authentication validation.
    *   `AuthResult`: Represents a successful authentication result, containing a success flag (true) and the `McpAuthContext`.
    *   `AuthFailure`: Represents a failed authentication result, containing a success flag (false) and the `NextResponse` representing the error.
    *   `AuthValidationResult`:  A union type that can be either an `AuthResult` or an `AuthFailure`, representing the possible outcomes of the `validateMcpAuth` function.

```typescript
/**
 * Validates MCP authentication and authorization
 */
async function validateMcpAuth(
  request: NextRequest,
  permissionLevel: McpPermissionLevel
): Promise<AuthValidationResult> {
  const requestId = generateRequestId()

  try {
    const auth = await checkHybridAuth(request, { requireWorkflowId: false })
    if (!auth.success || !auth.userId) {
      logger.warn(`[${requestId}] Authentication failed: ${auth.error}`)
      return {
        success: false,
        errorResponse: createMcpErrorResponse(
          new Error(auth.error || 'Authentication required'),
          'Authentication failed',
          401
        ),
      }
    }

    let workspaceId: string | null = null

    const { searchParams } = new URL(request.url)
    workspaceId = searchParams.get('workspaceId')

    if (!workspaceId) {
      try {
        const contentType = request.headers.get('content-type')
        if (contentType?.includes('application/json')) {
          const body = await request.json()
          workspaceId = body.workspaceId
          ;(request as any)._parsedBody = body
        }
      } catch (error) {
        logger.debug(`[${requestId}] Could not parse request body for workspaceId extraction`)
      }
    }

    if (!workspaceId) {
      return {
        success: false,
        errorResponse: createMcpErrorResponse(
          new Error('workspaceId is required'),
          'Missing required parameter',
          400
        ),
      }
    }

    const userPermissions = await getUserEntityPermissions(auth.userId, 'workspace', workspaceId)
    if (!userPermissions) {
      return {
        success: false,
        errorResponse: createMcpErrorResponse(
          new Error('Access denied to workspace'),
          'Insufficient permissions',
          403
        ),
      }
    }

    const hasRequiredPermission = checkPermissionLevel(userPermissions, permissionLevel)
    if (!hasRequiredPermission) {
      const permissionError = getPermissionErrorMessage(permissionLevel)
      return {
        success: false,
        errorResponse: createMcpErrorResponse(
          new Error(permissionError),
          'Insufficient permissions',
          403
        ),
      }
    }

    return {
      success: true,
      context: {
        userId: auth.userId,
        workspaceId,
        requestId,
      },
    }
  } catch (error) {
    logger.error(`[${requestId}] Error during MCP auth validation:`, error)
    return {
      success: false,
      errorResponse: createMcpErrorResponse(
        error instanceof Error ? error : new Error('Authentication validation failed'),
        'Authentication validation failed',
        500
      ),
    }
  }
}
```

*   **`validateMcpAuth` Function:** This is the core function that performs the authentication and authorization.
    *   It takes a `NextRequest` and a `permissionLevel` as input.
    *   It generates a unique `requestId` for tracking the request.
    *   **Authentication:** It calls `checkHybridAuth` to authenticate the user. If authentication fails (either `auth.success` is false or `auth.userId` is missing), it logs a warning and returns an `AuthFailure` with a 401 error response.
    *   **Workspace ID Extraction:** It attempts to extract the `workspaceId` from the request in the following order:
        1.  From the query parameters.
        2.  From the request body (if the content type is `application/json`).  It parses the body and stores it in `(request as any)._parsedBody` for later use. It handles potential errors during body parsing.
    *   If the `workspaceId` cannot be found, it returns an `AuthFailure` with a 400 error response.
    *   **Authorization:** It calls `getUserEntityPermissions` to retrieve the user's permissions for the specified workspace. If the user has no permissions for the workspace, it returns an `AuthFailure` with a 403 error response.
    *   It calls `checkPermissionLevel` to check if the user's permissions meet the required `permissionLevel`. If not, it calls `getPermissionErrorMessage` to get a specific error message and returns an `AuthFailure` with a 403 error response.
    *   If authentication and authorization are successful, it returns an `AuthResult` with the `userId`, `workspaceId`, and `requestId` in the `context`.
    *   It includes a `try...catch` block to handle any errors that might occur during the authentication or authorization process.  If an error occurs, it logs the error and returns an `AuthFailure` with a 500 error response.

```typescript
/**
 * Check if user has required permission level
 */
function checkPermissionLevel(userPermission: string, requiredLevel: McpPermissionLevel): boolean {
  switch (requiredLevel) {
    case 'read':
      return ['read', 'write', 'admin'].includes(userPermission)
    case 'write':
      return ['write', 'admin'].includes(userPermission)
    case 'admin':
      return userPermission === 'admin'
    default:
      return false
  }
}
```

*   **`checkPermissionLevel` Function:** This function checks if the user's permission level is sufficient for the required level.
    *   It takes the user's `userPermission` and the `requiredLevel` as input.
    *   It uses a `switch` statement to determine if the user's permission level is sufficient based on the `requiredLevel`.
    *   A user with "write" or "admin" permission has implicit "read" permission. A user with "admin" permission has implicit "write" permission.

```typescript
/**
 * Get appropriate error message for permission level
 */
function getPermissionErrorMessage(permissionLevel: McpPermissionLevel): string {
  switch (permissionLevel) {
    case 'read':
      return 'Workspace access required for MCP operations'
    case 'write':
      return 'Write or admin permission required for MCP server management'
    case 'admin':
      return 'Admin permission required for MCP server administration'
    default:
      return 'Insufficient permissions for MCP operation'
  }
}
```

*   **`getPermissionErrorMessage` Function:** This function returns an appropriate error message for a given permission level.
    *   It takes the `permissionLevel` as input.
    *   It uses a `switch` statement to return a specific error message based on the `permissionLevel`.

```typescript
/**
 * Higher-order function that wraps MCP route handlers with authentication middleware
 *
 * @param permissionLevel - Required permission level ('read', 'write', or 'admin')
 * @returns Middleware wrapper function
 *
 */
export function withMcpAuth(permissionLevel: McpPermissionLevel = 'read') {
  return function middleware(handler: McpRouteHandler) {
    return async function wrappedHandler(
      request: NextRequest,
      ...args: any[]
    ): Promise<NextResponse> {
      const authResult = await validateMcpAuth(request, permissionLevel)

      if (!authResult.success) {
        return (authResult as AuthFailure).errorResponse
      }

      try {
        return await handler(request, (authResult as AuthResult).context, ...args)
      } catch (error) {
        logger.error(
          `[${(authResult as AuthResult).context.requestId}] Error in MCP route handler:`,
          error
        )
        return createMcpErrorResponse(
          error instanceof Error ? error : new Error('Internal server error'),
          'Internal server error',
          500
        )
      }
    }
  }
}
```

*   **`withMcpAuth` Function:** This is a higher-order function that acts as the middleware wrapper.
    *   It takes an optional `permissionLevel` as input (defaulting to 'read').
    *   It returns a `middleware` function that takes an `McpRouteHandler` as input.
    *   The `middleware` function returns an `wrappedHandler` function, which is the actual middleware that will be executed.
    *   The `wrappedHandler` function first calls `validateMcpAuth` to authenticate and authorize the user.
    *   If `validateMcpAuth` returns an `AuthFailure`, the `wrappedHandler` returns the `errorResponse` from the `AuthFailure`.
    *   If `validateMcpAuth` returns an `AuthResult`, the `wrappedHandler` calls the original `handler` function, passing in the `request`, the `authContext` from the `AuthResult`, and any additional arguments.
    *   It includes a `try...catch` block to handle any errors that might occur within the route handler.  If an error occurs, it logs the error and returns an `AuthFailure` with a 500 error response.

```typescript
/**
 * Utility to get parsed request body
 */
export function getParsedBody(request: NextRequest): any {
  return (request as any)._parsedBody
}
```

*   **`getParsedBody` Function:** This utility function retrieves the parsed request body, which was previously stored in `(request as any)._parsedBody` by the `validateMcpAuth` function. This avoids reparsing the body.

### In Summary

This file provides a robust and reusable authentication and authorization middleware for Next.js API routes within an MCP system. It ensures that only authorized users can access specific resources, improving security and maintainability. The code is well-structured, type-safe, and includes comprehensive error handling and logging.
