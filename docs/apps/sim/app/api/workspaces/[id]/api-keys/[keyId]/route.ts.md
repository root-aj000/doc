This TypeScript file defines two API endpoints for managing API keys within a "workspace" context in a Next.js application. It specifically handles `PUT` (update) and `DELETE` (remove) requests for existing workspace API keys.

The core functionality includes:
1.  **Authentication**: Ensuring the request comes from an authenticated user.
2.  **Authorization**: Verifying that the authenticated user has the necessary permissions (admin or write access) for the specific workspace associated with the API key.
3.  **Input Validation**: Using Zod to validate the request body for `PUT` operations.
4.  **Database Operations**: Interacting with a PostgreSQL database (via Drizzle ORM) to fetch, update, or delete API key records.
5.  **Error Handling**: Providing appropriate HTTP status codes and error messages for various scenarios (unauthorized, forbidden, not found, bad request, internal server error).
6.  **Logging**: Recording important events and errors for debugging and monitoring.

Let's break down the code step by step.

---

### **Dependencies and Imports**

This section brings in all the necessary modules and types from various parts of the application and external libraries.

```typescript
import { db } from '@sim/db' // Drizzle ORM database connection instance
import { apiKey } from '@sim/db/schema' // Drizzle ORM schema definition for the 'apiKey' table
import { and, eq, not } from 'drizzle-orm' // Drizzle ORM functions for query building (AND, EQUAL, NOT)
import { type NextRequest, NextResponse } from 'next/server' // Next.js types and utilities for API routes
import { z } from 'zod' // Zod library for runtime schema validation
import { getSession } from '@/lib/auth' // Utility to retrieve the current user's session
import { createLogger } from '@/lib/logs/console/logger' // Custom logger utility
import { getUserEntityPermissions } from '@/lib/permissions/utils' // Utility to check user permissions for an entity (like a workspace)
import { generateRequestId } from '@/lib/utils' // Utility to generate a unique request ID
```

*   **`db` and `apiKey`**: These are crucial for interacting with the database. `db` is the configured Drizzle ORM client, and `apiKey` is a Drizzle schema object representing the `apiKey` table in your database.
*   **`drizzle-orm` functions (`and`, `eq`, `not`)**: These are building blocks for constructing SQL `WHERE` clauses using Drizzle's fluent API.
    *   `and`: Combines multiple conditions with a logical AND.
    *   `eq`: Checks for equality (`column = value`).
    *   `not`: Negates a condition (`NOT (condition)`).
*   **`NextRequest`, `NextResponse`**: Standard types and classes provided by Next.js for handling incoming HTTP requests and sending HTTP responses in API routes.
*   **`z` (Zod)**: A powerful schema declaration and validation library. It's used here to define the expected structure and types of the request body.
*   **`getSession`**: This function is responsible for retrieving the current user's authentication session, typically containing user ID and other session data. It's fundamental for authentication.
*   **`createLogger`**: A custom logging utility. It allows for structured logging messages (info, warn, error) that can be easily parsed and monitored.
*   **`getUserEntityPermissions`**: This is an authorization utility. It determines what level of access (`admin`, `write`, `read`, or `none`) a specific user has to a given entity (in this case, a `workspace`).
*   **`generateRequestId`**: A utility function to generate a unique identifier for each incoming request. This is extremely helpful for tracing logs across different parts of a request's lifecycle.

---

### **Logger Initialization and Schema Definition**

These are shared utilities used by both `PUT` and `DELETE` handlers.

```typescript
const logger = createLogger('WorkspaceApiKeyAPI')
```

*   **`const logger = createLogger('WorkspaceApiKeyAPI')`**: Initializes a logger instance specifically named 'WorkspaceApiKeyAPI'. This name helps in identifying which part of the application is generating a particular log message.

```typescript
const UpdateKeySchema = z.object({
  name: z.string().min(1, 'Name is required'),
})
```

*   **`const UpdateKeySchema = z.object({...})`**: This defines a Zod schema for validating the input when updating an API key.
    *   `z.object({...})`: Specifies that the input should be an object.
    *   `name: z.string().min(1, 'Name is required')`: Defines a property `name` which must be a string and have a minimum length of 1 character. If it's empty, it will throw an error with the message 'Name is required'. This ensures that the API key's name is always provided and not empty.

---

### **`PUT` Request Handler: Update Workspace API Key**

This function handles HTTP `PUT` requests, which are typically used to update an existing resource. In this case, it updates the `name` of a specific workspace API key.

```typescript
export async function PUT(
  request: NextRequest,
  { params }: { params: Promise<{ id: string; keyId: string }> }
) {
```

*   **`export async function PUT(...)`**: This declares an asynchronous function named `PUT`. In Next.js API routes, functions named after HTTP methods (`GET`, `POST`, `PUT`, `DELETE`) are automatically invoked when a request with that method hits the route.
*   **`request: NextRequest`**: Represents the incoming HTTP request object, containing details like headers, body, and URL.
*   **`{ params }: { params: Promise<{ id: string; keyId: string }> }`**: This is object destructuring for the second argument. `params` will contain route parameters.
    *   `id: string`: This typically corresponds to the `workspaceId` (e.g., `api/workspaces/[id]/api-keys/[keyId]`).
    *   `keyId: string`: This is the ID of the specific API key to be updated.
    *   `Promise<{...}>`: Next.js may provide `params` as a promise in some contexts, so it's awaited later.

```typescript
  const requestId = generateRequestId()
  const { id: workspaceId, keyId } = await params
```

*   **`const requestId = generateRequestId()`**: Generates a unique ID for this specific request. This ID will be included in all log messages related to this request, making it easy to trace its flow.
*   **`const { id: workspaceId, keyId } = await params`**: Destructures the `params` object.
    *   `id` is aliased to `workspaceId` for clarity, indicating it refers to the workspace ID.
    *   `keyId` is directly extracted. We `await params` because, as noted earlier, `params` can be a Promise.

```typescript
  try {
```

*   **`try { ... }`**: This block encloses the main logic of the function. Any errors thrown within this block will be caught by the `catch` block, allowing for graceful error handling.

    ---

    ### **Authentication**

    ```typescript
    const session = await getSession()
    if (!session?.user?.id) {
      logger.warn(`[${requestId}] Unauthorized workspace API key update attempt`)
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
    }
    ```

*   **`const session = await getSession()`**: Retrieves the current user's session data. This function likely checks for a valid authentication token (e.g., a cookie or header) and decodes the user's information.
*   **`if (!session?.user?.id)`**: Checks if a session exists and if it contains a `user.id`.
    *   `?.` is optional chaining, safely checking for nested properties without throwing an error if a parent property is `null` or `undefined`.
    *   If no user ID is found, it means the user is not authenticated.
*   **`logger.warn(...)`**: Logs a warning message indicating an unauthorized attempt, including the `requestId` for tracing.
*   **`return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })`**: Returns a JSON response with an "Unauthorized" error message and an HTTP status code of `401`.

    ---

    ### **Authorization**

    ```typescript
    const userId = session.user.id

    const permission = await getUserEntityPermissions(userId, 'workspace', workspaceId)
    if (!permission || (permission !== 'admin' && permission !== 'write')) {
      return NextResponse.json({ error: 'Forbidden' }, { status: 403 })
    }
    ```

*   **`const userId = session.user.id`**: Extracts the authenticated user's ID from the session.
*   **`const permission = await getUserEntityPermissions(userId, 'workspace', workspaceId)`**: Calls a utility function to determine the `userId`'s permissions for the specified `workspaceId`. The `entityType` is 'workspace'. This function would return a permission level like `'admin'`, `'write'`, `'read'`, or `null`.
*   **`if (!permission || (permission !== 'admin' && permission !== 'write'))`**: Checks if the user has sufficient permissions.
    *   `!permission`: If `permission` is `null` or `undefined`, it means no permissions were found.
    *   `permission !== 'admin' && permission !== 'write'`: The user must be either an `admin` or have `write` access to perform an update operation. `read` access is not enough.
*   **`return NextResponse.json({ error: 'Forbidden' }, { status: 403 })`**: If permissions are insufficient, returns a JSON response with a "Forbidden" error message and an HTTP status code of `403`.

    ---

    ### **Input Validation**

    ```typescript
    const body = await request.json()
    const { name } = UpdateKeySchema.parse(body)
    ```

*   **`const body = await request.json()`**: Parses the incoming request body as JSON.
*   **`const { name } = UpdateKeySchema.parse(body)`**: Validates the parsed `body` against the `UpdateKeySchema` defined earlier.
    *   `UpdateKeySchema.parse(body)`: Zod attempts to validate the `body`. If validation fails (e.g., `name` is missing or empty), Zod will throw a `ZodError`, which will be caught by the outer `try...catch` block.
    *   `const { name }`: If validation succeeds, the `name` property is extracted from the validated object.

    ---

    ### **Verify API Key Exists**

    ```typescript
    const existingKey = await db
      .select()
      .from(apiKey)
      .where(
        and(eq(apiKey.workspaceId, workspaceId), eq(apiKey.id, keyId), eq(apiKey.type, 'workspace'))
      )
      .limit(1)

    if (existingKey.length === 0) {
      return NextResponse.json({ error: 'API key not found' }, { status: 404 })
    }
    ```

*   **`await db.select().from(apiKey)...`**: This initiates a Drizzle ORM query to select data from the `apiKey` table.
    *   `.select()`: Specifies that all columns should be selected (equivalent to `SELECT *`).
    *   `.from(apiKey)`: Specifies the table to query.
    *   `.where(and(eq(apiKey.workspaceId, workspaceId), eq(apiKey.id, keyId), eq(apiKey.type, 'workspace')))`: This is the core filtering condition:
        *   `and(...)`: Combines three conditions with a logical `AND`.
        *   `eq(apiKey.workspaceId, workspaceId)`: Ensures the key belongs to the specified `workspaceId`.
        *   `eq(apiKey.id, keyId)`: Ensures it's the specific key identified by `keyId`.
        *   `eq(apiKey.type, 'workspace')`: Adds an extra safeguard to ensure we're only operating on API keys explicitly marked as 'workspace' type.
    *   `.limit(1)`: Optimizes the query by asking the database to stop after finding the first matching record, as we only expect one (or zero).
*   **`if (existingKey.length === 0)`**: Checks if the query returned any results. If `existingKey` array is empty, it means no API key with the given `keyId` was found in the specified workspace.
*   **`return NextResponse.json({ error: 'API key not found' }, { status: 404 })`**: Returns a JSON response with "API key not found" error and an HTTP status code of `404`.

    ---

    ### **Check for Conflicting Name**

    ```typescript
    const conflictingKey = await db
      .select()
      .from(apiKey)
      .where(
        and(
          eq(apiKey.workspaceId, workspaceId),
          eq(apiKey.name, name),
          eq(apiKey.type, 'workspace'),
          not(eq(apiKey.id, keyId))
        )
      )
      .limit(1)

    if (conflictingKey.length > 0) {
      return NextResponse.json(
        { error: 'A workspace API key with this name already exists' },
        { status: 400 }
      )
    }
    ```

*   **`await db.select().from(apiKey)...`**: Another Drizzle select query to check for name uniqueness.
    *   `.where(and(...))` : The conditions here are critical for preventing name collisions:
        *   `eq(apiKey.workspaceId, workspaceId)`: Confines the search to the current workspace.
        *   `eq(apiKey.name, name)`: Checks if any *other* key already has the `new name` provided in the request.
        *   `eq(apiKey.type, 'workspace')`: Again, ensuring we only compare against workspace keys.
        *   `not(eq(apiKey.id, keyId))`: **This is crucial.** It excludes the API key *currently being updated* from the conflict check. We don't want the key to conflict with itself if its name hasn't changed or if the new name is still unique among *other* keys.
    *   `.limit(1)`: Again, for optimization, as we only need to know if *any* conflict exists.
*   **`if (conflictingKey.length > 0)`**: If a record is found, it means another API key in the same workspace already uses the proposed `name`.
*   **`return NextResponse.json({ error: 'A workspace API key with this name already exists' }, { status: 400 })`**: Returns a JSON response with a conflict error message and an HTTP status code of `400` (Bad Request).

    ---

    ### **Update API Key**

    ```typescript
    const [updatedKey] = await db
      .update(apiKey)
      .set({
        name,
        updatedAt: new Date(),
      })
      .where(
        and(eq(apiKey.workspaceId, workspaceId), eq(apiKey.id, keyId), eq(apiKey.type, 'workspace'))
      )
      .returning({
        id: apiKey.id,
        name: apiKey.name,
        createdAt: apiKey.createdAt,
        updatedAt: apiKey.updatedAt,
      })
    ```

*   **`await db.update(apiKey)...`**: Initiates a Drizzle ORM update operation on the `apiKey` table.
    *   `.set({ name, updatedAt: new Date() })` : Specifies the columns to update and their new values.
        *   `name`: The `name` obtained from the validated request body.
        *   `updatedAt: new Date()`: Sets the `updatedAt` timestamp to the current time.
    *   `.where(and(eq(apiKey.workspaceId, workspaceId), eq(apiKey.id, keyId), eq(apiKey.type, 'workspace')))`: This `WHERE` clause is identical to the one used to find the `existingKey`. It ensures that only the *specific* API key identified by `keyId` within the `workspaceId` and of type 'workspace' is updated.
    *   `.returning({ id: apiKey.id, name: apiKey.name, createdAt: apiKey.createdAt, updatedAt: apiKey.updatedAt })`: Specifies which columns of the *updated* record should be returned by the database. This is very useful for getting the latest state of the updated resource.
*   **`const [updatedKey] = ...`**: Drizzle's `update().returning()` method returns an array of updated records. Since we are updating a single record (due to the `where` clause on `apiKey.id`), we expect an array with one element. Array destructuring `[updatedKey]` extracts that single element directly into the `updatedKey` variable.

    ---

    ### **Success Response**

    ```typescript
    logger.info(`[${requestId}] Updated workspace API key: ${keyId} in workspace ${workspaceId}`)
    return NextResponse.json({ key: updatedKey })
  } catch (error: unknown) {
```

*   **`logger.info(...)`**: Logs a success message, including the `requestId`, `keyId`, and `workspaceId` for auditing.
*   **`return NextResponse.json({ key: updatedKey })`**: Returns a JSON response containing the details of the successfully updated API key, with a default HTTP status code of `200` (OK).

    ---

    ### **Error Handling (for `PUT` and `DELETE`)**

    ```typescript
  } catch (error: unknown) {
    logger.error(`[${requestId}] Workspace API key PUT error`, error)
    return NextResponse.json(
      { error: error instanceof Error ? error.message : 'Failed to update workspace API key' },
      { status: 500 }
    )
  }
}
```

*   **`catch (error: unknown)`**: This block executes if any error occurs within the `try` block. `error: unknown` is a TypeScript best practice for catching errors when their type is not known beforehand.
*   **`logger.error(...)`**: Logs the error message, including the `requestId` and the `error` object itself (which can provide stack traces and more details).
*   **`return NextResponse.json(...)`**: Returns a generic error response to the client.
    *   `error instanceof Error ? error.message : 'Failed to update workspace API key'`: This checks if the caught `error` is an instance of JavaScript's `Error` class. If it is, its `message` property is used. Otherwise, a generic message "Failed to update workspace API key" is provided. This prevents exposing internal server details to the client while still providing useful information where available (e.g., Zod validation errors will have specific `error.message` strings).
    *   `{ status: 500 }`: Returns an HTTP status code of `500` (Internal Server Error).

---

### **`DELETE` Request Handler: Delete Workspace API Key**

This function handles HTTP `DELETE` requests, used to remove an existing resource.

```typescript
export async function DELETE(
  request: NextRequest,
  { params }: { params: Promise<{ id: string; keyId: string }> }
) {
```

*   **`export async function DELETE(...)`**: Similar to `PUT`, this declares an asynchronous function for HTTP `DELETE` requests.
*   **`request: NextRequest, { params }: { params: Promise<{ id: string; keyId: string }> }`**: The arguments are identical to the `PUT` function, receiving the `NextRequest` and route `params` (`workspaceId` and `keyId`).

```typescript
  const requestId = generateRequestId()
  const { id: workspaceId, keyId } = await params
```

*   **`const requestId = generateRequestId()`**: Generates a unique request ID.
*   **`const { id: workspaceId, keyId } = await params`**: Extracts `workspaceId` and `keyId` from the route parameters.

```typescript
  try {
```

*   **`try { ... }`**: Encloses the main logic for error handling.

    ---

    ### **Authentication (identical to `PUT`)**

    ```typescript
    const session = await getSession()
    if (!session?.user?.id) {
      logger.warn(`[${requestId}] Unauthorized workspace API key deletion attempt`)
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
    }
    ```

*   These lines perform the same authentication check as in the `PUT` function. If the user is not logged in, it returns a `401 Unauthorized` response.

    ---

    ### **Authorization (identical to `PUT`)**

    ```typescript
    const userId = session.user.id

    const permission = await getUserEntityPermissions(userId, 'workspace', workspaceId)
    if (!permission || (permission !== 'admin' && permission !== 'write')) {
      return NextResponse.json({ error: 'Forbidden' }, { status: 403 })
    }
    ```

*   These lines perform the same authorization check. Only users with `admin` or `write` permissions for the specific `workspaceId` are allowed to delete an API key. If not, it returns a `403 Forbidden` response.

    ---

    ### **Delete API Key**

    ```typescript
    const deletedRows = await db
      .delete(apiKey)
      .where(
        and(eq(apiKey.workspaceId, workspaceId), eq(apiKey.id, keyId), eq(apiKey.type, 'workspace'))
      )
      .returning({ id: apiKey.id })
    ```

*   **`await db.delete(apiKey)...`**: Initiates a Drizzle ORM delete operation on the `apiKey` table.
    *   `.where(and(eq(apiKey.workspaceId, workspaceId), eq(apiKey.id, keyId), eq(apiKey.type, 'workspace')))`: This `WHERE` clause is identical to the one used in `PUT` to identify the specific API key. It ensures that only the API key matching the `keyId`, `workspaceId`, and `type='workspace'` is deleted.
    *   `.returning({ id: apiKey.id })`: After deletion, Drizzle returns an array of objects, where each object contains the `id` of a deleted row. This is useful for confirming that a row was indeed deleted.

```typescript
    if (deletedRows.length === 0) {
      return NextResponse.json({ error: 'API key not found' }, { status: 404 })
    }
```

*   **`if (deletedRows.length === 0)`**: Checks if any rows were actually deleted. If `deletedRows` is empty, it means no API key matching the criteria was found to delete.
*   **`return NextResponse.json({ error: 'API key not found' }, { status: 404 })`**: Returns a JSON response with "API key not found" error and an HTTP status code of `404`.

    ---

    ### **Success Response**

    ```typescript
    logger.info(`[${requestId}] Deleted workspace API key: ${keyId} from workspace ${workspaceId}`)
    return NextResponse.json({ success: true })
  } catch (error: unknown) {
```

*   **`logger.info(...)`**: Logs a success message indicating which API key was deleted from which workspace.
*   **`return NextResponse.json({ success: true })`**: Returns a simple JSON success message with a default HTTP status code of `200` (OK).

    ---

    ### **Error Handling (identical to `PUT`)**

    ```typescript
    logger.error(`[${requestId}] Workspace API key DELETE error`, error)
    return NextResponse.json(
      { error: error instanceof Error ? error.message : 'Failed to delete workspace API key' },
      { status: 500 }
    )
  }
}
```

*   This `catch` block is identical to the one in the `PUT` function, providing robust error logging and a generic `500 Internal Server Error` response to the client if any unexpected issues arise during the deletion process.

---

### **Summary of Complex Logic Simplification**

1.  **Drizzle ORM Queries**: While SQL queries can be complex, Drizzle's fluent API (`db.select().from().where().and().eq().not()`) makes them more readable and type-safe. The explanation breaks down each chainable method and its purpose, especially `and` and `not(eq())` for unique name checking.
2.  **Authentication & Authorization**: These are crucial security layers. The code uses dedicated utility functions (`getSession`, `getUserEntityPermissions`) to abstract away the underlying implementation details, making the main handler logic cleaner. The explanation clearly distinguishes between *who* the user is (authentication) and *what* they are allowed to do (authorization) for a specific resource.
3.  **Input Validation (Zod)**: Instead of manual checks and `if` statements for each input field, Zod provides a declarative way to define expected data structures. `.parse()` centralizes validation, and if it fails, it throws an error that is caught by the `try...catch`, simplifying the flow.
4.  **Error Handling**: The consistent `try...catch` block with specific `NextResponse.json` calls for different HTTP status codes (401, 403, 404, 400, 500) creates a predictable and robust error handling strategy, preventing uncaught exceptions from crashing the server and providing clear feedback to API consumers.
5.  **Request Tracing**: `generateRequestId()` simplifies debugging by providing a single identifier to correlate all log messages related to a single API call, which is invaluable in production environments.

This file exemplifies a well-structured Next.js API route that handles common web application concerns like authentication, authorization, input validation, and database interactions with clear error handling and logging.