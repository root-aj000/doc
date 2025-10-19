This file, `hybrid-auth.ts`, is a central utility for handling authentication in a Next.js application. Its primary purpose is to provide a flexible and robust way to authenticate incoming requests by checking for **three different types of authentication credentials** in a specific order:

1.  **Internal JSON Web Token (JWT)**: Used for internal service-to-service communication.
2.  **User Session**: Standard cookie-based authentication for logged-in users in the web UI.
3.  **API Key**: For external clients or programmatic access using a dedicated API key.

This "hybrid" approach allows the same API endpoint to be secured by multiple authentication mechanisms, adapting to different client types (internal services, web browsers, external API users) without duplicating authentication logic.

Let's break down the code in detail.

---

## Code Explanation

```typescript
import { db } from '@sim/db'
import { workflow } from '@sim/db/schema'
import { eq } from 'drizzle-orm'
import type { NextRequest } from 'next/server'
import { authenticateApiKeyFromHeader, updateApiKeyLastUsed } from '@/lib/api-key/service'
import { getSession } from '@/lib/auth'
import { verifyInternalToken } from '@/lib/auth/internal'
import { createLogger } from '@/lib/logs/console/logger'
```

**Imports:**

*   `import { db } from '@sim/db'`: Imports the Drizzle ORM database client instance. This allows the code to interact with the database.
*   `import { workflow } from '@sim/db/schema'`: Imports the Drizzle ORM schema definition for the `workflow` table. This defines the structure of the workflow data in the database.
*   `import { eq } from 'drizzle-orm'`: Imports the `eq` (equals) function from Drizzle ORM, used for creating equality conditions in database queries (e.g., `WHERE id = 'someId'`).
*   `import type { NextRequest } from 'next/server'`: Imports the `NextRequest` type from Next.js, representing an incoming HTTP request in an API route or middleware. This is used for type safety.
*   `import { authenticateApiKeyFromHeader, updateApiKeyLastUsed } from '@/lib/api-key/service'`: Imports two functions related to API key management:
    *   `authenticateApiKeyFromHeader`: Validates an API key provided in a request header.
    *   `updateApiKeyLastUsed`: Updates the `lastUsed` timestamp for a given API key in the database.
*   `import { getSession } from '@/lib/auth'`: Imports `getSession`, a utility function to retrieve the current user's session data.
*   `import { verifyInternalToken } from '@/lib/auth/internal'`: Imports `verifyInternalToken`, a function specifically designed to validate internal JWTs.
*   `import { createLogger } from '@/lib/logs/console/logger'`: Imports a utility to create a logger instance for logging messages (e.g., errors, debug info).

```typescript
const logger = createLogger('HybridAuth')
```

**Logger Initialization:**

*   `const logger = createLogger('HybridAuth')`: Creates a logger instance named 'HybridAuth'. This logger will be used to output messages to the console, specifically for events related to this authentication logic.

```typescript
export interface AuthResult {
  success: boolean
  userId?: string
  authType?: 'session' | 'api_key' | 'internal_jwt'
  error?: string
}
```

**AuthResult Interface:**

*   `export interface AuthResult { ... }`: Defines a TypeScript interface `AuthResult`. This interface specifies the structure of the object that the `checkHybridAuth` function will return.
    *   `success: boolean`: Indicates whether authentication was successful (`true`) or failed (`false`).
    *   `userId?: string`: An optional string representing the ID of the authenticated user. Present if `success` is `true`.
    *   `authType?: 'session' | 'api_key' | 'internal_jwt'`: An optional literal type specifying which authentication method was successfully used. Present if `success` is `true`.
    *   `error?: string`: An optional string containing an error message if `success` is `false`.

```typescript
/**
 * Check for authentication using any of the 3 supported methods:
 * 1. Session authentication (cookies)
 * 2. API key authentication (X-API-Key header)
 * 3. Internal JWT authentication (Authorization: Bearer header)
 *
 * For internal JWT calls, requires workflowId to determine user context
 */
export async function checkHybridAuth(
  request: NextRequest,
  options: { requireWorkflowId?: boolean } = {}
): Promise<AuthResult> {
```

**`checkHybridAuth` Function Signature:**

*   `export async function checkHybridAuth(...)`: Declares an asynchronous function named `checkHybridAuth`. The `export` keyword makes it available for other files to import.
*   `request: NextRequest`: The first parameter is the incoming `NextRequest` object, which contains information about the HTTP request (headers, URL, body, etc.).
*   `options: { requireWorkflowId?: boolean } = {}`: The second parameter is an optional `options` object.
    *   `requireWorkflowId?: boolean`: An optional boolean property. If `true`, internal JWT calls *must* provide a `workflowId` to establish a user context. Defaults to `false` if not provided (due to the `= {}`).
*   `: Promise<AuthResult>`: Specifies that the function will return a `Promise` that resolves to an `AuthResult` object.

```typescript
  try {
    // 1. Check for internal JWT token first
    const authHeader = request.headers.get('authorization')
    if (authHeader?.startsWith('Bearer ')) {
      const token = authHeader.split(' ')[1]
      const isInternalCall = await verifyInternalToken(token)

      if (isInternalCall) {
        // ... (internal JWT logic)
      }
    }
```

**Internal JWT Authentication (First Check):**

This section attempts to authenticate the request using an internal JSON Web Token.

*   `try { ... } catch (error) { ... }`: The entire function's logic is wrapped in a `try...catch` block to gracefully handle any unexpected errors during the authentication process.
*   `const authHeader = request.headers.get('authorization')`: Retrieves the value of the `Authorization` header from the incoming request.
*   `if (authHeader?.startsWith('Bearer '))`: Checks if the `authHeader` exists and starts with `Bearer `, which is the standard prefix for JWTs in the Authorization header.
*   `const token = authHeader.split(' ')[1]`: If it's a Bearer token, splits the header string by space and takes the second part, which is the actual JWT.
*   `const isInternalCall = await verifyInternalToken(token)`: Calls `verifyInternalToken` to validate the extracted JWT. This function would typically verify the token's signature, expiry, and issuer.
*   `if (isInternalCall)`: If the token is successfully verified as an internal token, proceed with internal authentication logic.

```typescript
        // For internal calls, we need workflowId to determine user context
        let workflowId: string | null = null

        // Try to get workflowId from query params or request body
        const { searchParams } = new URL(request.url)
        workflowId = searchParams.get('workflowId')

        if (!workflowId && request.method === 'POST') {
          try {
            // Clone the request to avoid consuming the original body
            const clonedRequest = request.clone()
            const bodyText = await clonedRequest.text()
            if (bodyText) {
              const body = JSON.parse(bodyText)
              workflowId = body.workflowId
            }
          } catch {
            // Ignore JSON parse errors
          }
        }
```

**Extracting `workflowId` for Internal Calls:**

For internal calls, a `workflowId` is often needed to establish the context of the user performing the action (i.e., who "owns" the workflow). This logic attempts to find `workflowId` from multiple places.

*   `let workflowId: string | null = null`: Initializes a variable to store the `workflowId`, set to `null` initially.
*   `const { searchParams } = new URL(request.url)`: Creates a `URL` object from the request's URL and destructures its `searchParams` property, which provides methods to easily access query parameters.
*   `workflowId = searchParams.get('workflowId')`: Attempts to get the `workflowId` from the URL's query parameters (e.g., `?workflowId=abc`).
*   `if (!workflowId && request.method === 'POST')`: If `workflowId` was not found in query parameters *and* the request method is `POST`, it then tries to find it in the request body.
    *   `try { ... } catch { ... }`: A nested `try...catch` block specifically for parsing the request body, to handle potential JSON parsing errors.
    *   `const clonedRequest = request.clone()`: **Important:** `request.body` (and related methods like `json()` or `text()`) can only be consumed once. To allow other middleware or the actual route handler to also read the body, the request is `cloned()`.
    *   `const bodyText = await clonedRequest.text()`: Reads the body of the cloned request as plain text.
    *   `if (bodyText)`: If there's content in the body...
    *   `const body = JSON.parse(bodyText)`: Parses the text body as JSON.
    *   `workflowId = body.workflowId`: Attempts to extract `workflowId` from the parsed JSON body.
    *   `catch { // Ignore JSON parse errors }`: If `JSON.parse` fails, it simply ignores the error, meaning `workflowId` won't be set from the body.

```typescript
        if (!workflowId && options.requireWorkflowId !== false) {
          return {
            success: false,
            error: 'workflowId required for internal JWT calls',
          }
        }
```

**`workflowId` Requirement Check:**

*   `if (!workflowId && options.requireWorkflowId !== false)`: This condition checks if `workflowId` is still `null` (not found) AND if the `requireWorkflowId` option is explicitly `true` or *not explicitly `false`* (meaning it's `undefined` or `true`). This implies that by default, `workflowId` is required unless `options.requireWorkflowId` is set to `false`.
*   If `workflowId` is missing and required, it immediately returns an `AuthResult` indicating failure and an appropriate error message.

```typescript
        if (workflowId) {
          // Get workflow owner as user context
          const [workflowData] = await db
            .select({ userId: workflow.userId })
            .from(workflow)
            .where(eq(workflow.id, workflowId))
            .limit(1)

          if (!workflowData) {
            return {
              success: false,
              error: 'Workflow not found',
            }
          }

          return {
            success: true,
            userId: workflowData.userId,
            authType: 'internal_jwt',
          }
        }
        // Internal call without workflow context - still valid for some routes
        return {
          success: true,
          authType: 'internal_jwt',
        }
```

**Workflow Data Retrieval and Internal JWT Success:**

*   `if (workflowId)`: If a `workflowId` was successfully obtained (either from query or body):
    *   `const [workflowData] = await db.select({ userId: workflow.userId }).from(workflow).where(eq(workflow.id, workflowId)).limit(1)`: This is a Drizzle ORM database query:
        *   `.select({ userId: workflow.userId })`: Selects only the `userId` column from the `workflow` table, aliasing it as `userId`.
        *   `.from(workflow)`: Specifies that the query is against the `workflow` table.
        *   `.where(eq(workflow.id, workflowId))`: Filters the results to find the workflow where its `id` matches the `workflowId` found in the request.
        *   `.limit(1)`: Ensures only one record is returned (since `id` should be unique).
        *   `const [workflowData] = ...`: Destructures the result, which is an array, to get the first (and only) `workflowData` object.
    *   `if (!workflowData)`: If no workflow was found with the given ID, it returns an error.
    *   `return { success: true, userId: workflowData.userId, authType: 'internal_jwt' }`: If a workflow is found, authentication is successful. The `userId` from the workflow owner is returned as the user context, and the `authType` is set to `internal_jwt`.
*   `return { success: true, authType: 'internal_jwt' }`: If the `verifyInternalToken` was successful, but no `workflowId` was provided or required, it means the call is still considered a valid internal JWT call, but without a specific user context. This can be useful for certain internal services that don't operate on behalf of a specific user/workflow.

```typescript
    // 2. Try session auth (for web UI)
    const session = await getSession()
    if (session?.user?.id) {
      return {
        success: true,
        userId: session.user.id,
        authType: 'session',
      }
    }
```

**Session Authentication (Second Check):**

If the internal JWT check fails or is not present, the function attempts session-based authentication.

*   `const session = await getSession()`: Calls the `getSession` utility to retrieve the current user's session data from cookies.
*   `if (session?.user?.id)`: Checks if a session exists and if it contains a `user` object with an `id`.
*   If a valid session with a user ID is found, it returns `success: true`, the `userId` from the session, and `authType: 'session'`.

```typescript
    // 3. Try API key auth
    const apiKeyHeader = request.headers.get('x-api-key')
    if (apiKeyHeader) {
      const result = await authenticateApiKeyFromHeader(apiKeyHeader)
      if (result.success) {
        await updateApiKeyLastUsed(result.keyId!)
        return {
          success: true,
          userId: result.userId!,
          authType: 'api_key',
        }
      }

      return {
        success: false,
        error: 'Invalid API key',
      }
    }
```

**API Key Authentication (Third Check):**

If both internal JWT and session authentication fail, the function tries API key authentication.

*   `const apiKeyHeader = request.headers.get('x-api-key')`: Retrieves the value of the custom `X-API-Key` header.
*   `if (apiKeyHeader)`: Checks if the `X-API-Key` header is present.
*   `const result = await authenticateApiKeyFromHeader(apiKeyHeader)`: Calls the `authenticateApiKeyFromHeader` function to validate the API key. This function would typically look up the key in the database and check its validity, expiration, etc.
*   `if (result.success)`: If the API key is successfully authenticated:
    *   `await updateApiKeyLastUsed(result.keyId!)`: Updates the `lastUsed` timestamp for the authenticated API key in the database. The `!` (non-null assertion operator) is used here because `result.keyId` is guaranteed to exist if `result.success` is `true`.
    *   Returns `success: true`, the `userId` associated with the API key, and `authType: 'api_key'`.
*   `return { success: false, error: 'Invalid API key' }`: If an `X-API-Key` header was present but the authentication failed (e.g., key not found, expired, invalid), it returns an `AuthResult` with an "Invalid API key" error.

```typescript
    // No authentication found
    return {
      success: false,
      error: 'Authentication required - provide session, API key, or internal JWT',
    }
  } catch (error) {
    logger.error('Error in hybrid authentication:', error)
    return {
      success: false,
      error: 'Authentication error',
    }
  }
}
```

**No Authentication Found & Error Handling:**

*   `return { success: false, error: 'Authentication required - provide session, API key, or internal JWT' }`: If none of the three authentication methods (internal JWT, session, or API key) successfully authenticated the request, this is the final fallback return, indicating that no valid credentials were found.
*   `catch (error) { ... }`: This is the `catch` block for the `try` statement that wraps the entire function.
    *   `logger.error('Error in hybrid authentication:', error)`: Logs any unexpected errors that occur during the authentication process using the previously initialized `logger`.
    *   `return { success: false, error: 'Authentication error' }`: Returns a generic `Authentication error` result, keeping internal error details from being exposed to the client.

---

## Simplified Complex Logic

1.  **Multiple Auth Methods, Single Function:** Instead of having separate checks for each authentication type, `checkHybridAuth` consolidates them into one logical flow. It tries them in a specific order (internal JWT -> session -> API key) and stops as soon as one succeeds. This makes securing endpoints much cleaner.
2.  **`workflowId` Context for Internal Calls:** For internal requests, simply verifying a JWT isn't always enough; you might need to know *who* or *what* triggered the call. The `workflowId` acts as a proxy for the user context, fetched dynamically from query parameters or the request body. This allows internal services to act "on behalf of" a specific workflow owner.
3.  **Safe Request Body Consumption:** The `request.clone()` technique is crucial for Next.js API routes. It prevents the authentication layer from "eating" the request body, allowing subsequent route handlers to still access it without issues.
4.  **Drizzle ORM for Database Access:** The database query (`db.select().from().where().limit()`) uses Drizzle ORM's fluent API, which is type-safe and makes database interactions readable and less error-prone compared to raw SQL or less-typed ORMs.
5.  **Robust Error Handling:** The `try...catch` blocks ensure that the function doesn't crash on unexpected issues (like malformed JSON in the body or database errors) and provides clear, consistent error messages to the caller while logging detailed errors internally.

In essence, `hybrid-auth.ts` is a versatile authentication gatekeeper, designed to handle various client types and provide specific user context when necessary, all within a clear, sequential, and robust structure.