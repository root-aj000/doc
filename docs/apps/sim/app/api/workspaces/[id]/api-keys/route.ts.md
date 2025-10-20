This file defines a set of API routes within a Next.js application that allow users to manage API keys for a specific workspace. It provides functionality to retrieve, create, and delete API keys, with robust authentication, authorization, and input validation built in.

---

### **Detailed Explanation**

#### **Purpose of This File and What It Does**

This TypeScript file (`app/api/workspace/[id]/keys/route.ts`) acts as the backend for managing API keys associated with a particular workspace. It exposes three main API endpoints corresponding to HTTP methods:

*   **`GET` (Retrieve API Keys):** Fetches a list of all API keys belonging to a specified workspace. It displays a masked version of the keys for security.
*   **`POST` (Create API Key):** Allows a user to generate a new API key for the workspace, provided they have the necessary permissions.
*   **`DELETE` (Delete API Keys):** Enables a user to revoke one or more existing API keys from the workspace.

All operations are secured with user session checks and a fine-grained permission system to ensure only authorized users can perform actions on a given workspace's API keys.

#### **Core Logic Simplified**

1.  **Identify Workspace:** Each request targets a specific workspace using its ID provided in the URL (`/api/workspace/[id]/keys`).
2.  **Authenticate User:** Verify that a logged-in user is making the request.
3.  **Authorize User:** Check if the authenticated user has the necessary permissions (e.g., read, write, admin) for *that specific workspace*.
4.  **Validate Input:** For `POST` and `DELETE` requests, ensure the data sent in the request body (like key names or key IDs) is correctly formatted and valid.
5.  **Database Interaction:** Perform the requested operation (select, insert, delete) on the `apiKey` table in the database, ensuring it's tied to the correct `workspaceId` and `type`.
6.  **Respond Securely:** Return data (like API key lists) or confirmation messages. For sensitive data like API keys, the actual key is only returned once upon creation; otherwise, a masked version is shown.
7.  **Error Handling:** Catch any issues (e.g., invalid input, database errors, permission failures) and return appropriate HTTP status codes and error messages.
8.  **Logging:** Record significant events and errors using a custom logger, often tagged with a unique request ID for easier debugging.

---

### **Line-by-Line Code Explanation**

#### **Imports**

```typescript
import { db } from '@sim/db'
import { apiKey, workspace } from '@sim/db/schema'
import { and, eq, inArray } from 'drizzle-orm'
import { nanoid } from 'nanoid'
import { type NextRequest, NextResponse } from 'next/server'
import { z } from 'zod'
import { createApiKey, getApiKeyDisplayFormat } from '@/lib/api-key/auth'
import { getSession } from '@/lib/auth'
import { createLogger } from '@/lib/logs/console/logger'
import { getUserEntityPermissions } from '@/lib/permissions/utils'
import { generateRequestId } from '@/lib/utils'
```

*   **`import { db } from '@sim/db'`**: Imports the Drizzle ORM database instance, which is used to interact with the underlying database. `db` is your configured database client.
*   **`import { apiKey, workspace } from '@sim/db/schema'`**: Imports the Drizzle ORM schema definitions for the `apiKey` and `workspace` tables. These are essentially representations of your database tables, allowing you to build type-safe queries.
*   **`import { and, eq, inArray } from 'drizzle-orm'`**: Imports specific helper functions from Drizzle ORM.
    *   `and`: Used to combine multiple conditions in a `WHERE` clause (logical AND).
    *   `eq`: Checks for equality (`column = value`).
    *   `inArray`: Checks if a column's value is present in a given array (`column IN (value1, value2)`).
*   **`import { nanoid } from 'nanoid'`**: Imports `nanoid`, a small, secure, URL-friendly unique string ID generator, used here for generating primary keys for new API keys.
*   **`import { type NextRequest, NextResponse } from 'next/server'`**: Imports types and classes from Next.js for handling server-side API requests and responses.
    *   `NextRequest`: Represents an incoming HTTP request.
    *   `NextResponse`: Used to construct and send an HTTP response.
*   **`import { z } from 'zod'`**: Imports the `z` object from the Zod library, a powerful TypeScript-first schema validation library.
*   **`import { createApiKey, getApiKeyDisplayFormat } from '@/lib/api-key/auth'`**: Imports custom utility functions related to API key management.
    *   `createApiKey`: Likely generates a new API key and possibly encrypts it.
    *   `getApiKeyDisplayFormat`: Formats an API key for display (e.g., masking most of it).
*   **`import { getSession } from '@/lib/auth'`**: Imports a custom utility to retrieve the current user's session data, typically used for authentication.
*   **`import { createLogger } from '@/lib/logs/console/logger'`**: Imports a custom logging utility to log messages, warnings, and errors to the console or other configured logging targets.
*   **`import { getUserEntityPermissions } from '@/lib/permissions/utils'`**: Imports a custom utility function to check a user's permissions for a specific entity (like a workspace).
*   **`import { generateRequestId } from '@/lib/utils'`**: Imports a custom utility to generate a unique request ID, useful for tracing individual requests through logs.

#### **Logger and Zod Schemas**

```typescript
const logger = createLogger('WorkspaceApiKeysAPI')

const CreateKeySchema = z.object({
  name: z.string().trim().min(1, 'Name is required'),
})

const DeleteKeysSchema = z.object({
  keys: z.array(z.string()).min(1),
})
```

*   **`const logger = createLogger('WorkspaceApiKeysAPI')`**: Initializes a logger instance specifically for this API route, making it easier to identify logs originating from this file.
*   **`const CreateKeySchema = z.object({ ... })`**: Defines a Zod schema for validating the input when creating a new API key.
    *   `name: z.string().trim().min(1, 'Name is required')`: Specifies that the request body must contain a `name` property, which should be a string, trimmed of whitespace, and have at least one character.
*   **`const DeleteKeysSchema = z.object({ ... })`**: Defines a Zod schema for validating the input when deleting API keys.
    *   `keys: z.array(z.string()).min(1)`: Specifies that the request body must contain a `keys` property, which should be an array of strings (API key IDs), and the array must contain at least one element.

#### **`GET` Endpoint (Retrieve API Keys)**

```typescript
export async function GET(request: NextRequest, { params }: { params: Promise<{ id: string }> }) {
  const requestId = generateRequestId()
  const workspaceId = (await params).id

  try {
    const session = await getSession()
    if (!session?.user?.id) {
      logger.warn(`[${requestId}] Unauthorized workspace API keys access attempt`)
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
    }

    const userId = session.user.id

    const ws = await db.select().from(workspace).where(eq(workspace.id, workspaceId)).limit(1)
    if (!ws.length) {
      return NextResponse.json({ error: 'Workspace not found' }, { status: 404 })
    }

    const permission = await getUserEntityPermissions(userId, 'workspace', workspaceId)
    if (!permission) {
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
    }

    const workspaceKeys = await db
      .select({
        id: apiKey.id,
        name: apiKey.name,
        key: apiKey.key, // Encrypted key from DB
        createdAt: apiKey.createdAt,
        lastUsed: apiKey.lastUsed,
        expiresAt: apiKey.expiresAt,
        createdBy: apiKey.createdBy,
      })
      .from(apiKey)
      .where(and(eq(apiKey.workspaceId, workspaceId), eq(apiKey.type, 'workspace')))
      .orderBy(apiKey.createdAt)

    const formattedWorkspaceKeys = await Promise.all(
      workspaceKeys.map(async (key) => {
        const displayFormat = await getApiKeyDisplayFormat(key.key) // Decrypts/masks key
        return {
          ...key,
          key: key.key, // Original encrypted key remains in the object, not sent to client directly
          displayKey: displayFormat, // The masked version
        }
      })
    )

    return NextResponse.json({
      keys: formattedWorkspaceKeys,
    })
  } catch (error: unknown) {
    logger.error(`[${requestId}] Workspace API keys GET error`, error)
    return NextResponse.json(
      { error: error instanceof Error ? error.message : 'Failed to load API keys' },
      { status: 500 }
    )
  }
}
```

*   **`export async function GET(...)`**: Defines the asynchronous function that handles `GET` requests to this API route.
    *   `request: NextRequest`: The incoming HTTP request object.
    *   `{ params }: { params: Promise<{ id: string }> }`: Extracts URL parameters. `id` is the workspace ID from the route `[id]`. It's wrapped in a `Promise` by Next.js.
*   **`const requestId = generateRequestId()`**: Generates a unique ID for this specific request for logging purposes.
*   **`const workspaceId = (await params).id`**: Awaits the `params` promise to resolve and extracts the `id` (workspace ID) from the URL.
*   **`try { ... } catch (error: unknown) { ... }`**: A standard `try-catch` block for error handling.
*   **`const session = await getSession()`**: Retrieves the current user's session information.
*   **`if (!session?.user?.id)`**: Checks if a session exists and if a user ID is present within that session. If not, the user is not authenticated.
    *   **`logger.warn(...)`**: Logs a warning about an unauthorized access attempt.
    *   **`return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })`**: Returns a JSON response with an "Unauthorized" error and an HTTP status code of 401.
*   **`const userId = session.user.id`**: Extracts the authenticated user's ID.
*   **`const ws = await db.select().from(workspace).where(eq(workspace.id, workspaceId)).limit(1)`**: Queries the `workspace` table to check if a workspace with the given `workspaceId` exists.
    *   `db.select()`: Starts a select query.
    *   `from(workspace)`: Specifies the `workspace` table.
    *   `where(eq(workspace.id, workspaceId))`: Filters results to where the workspace's ID matches the `workspaceId`.
    *   `limit(1)`: Limits the result to one row (since IDs are unique).
*   **`if (!ws.length)`**: If no workspace is found (the array `ws` is empty).
    *   **`return NextResponse.json({ error: 'Workspace not found' }, { status: 404 })`**: Returns a 404 "Not Found" error.
*   **`const permission = await getUserEntityPermissions(userId, 'workspace', workspaceId)`**: Calls a utility function to get the user's permissions for the specified workspace.
*   **`if (!permission)`**: Checks if the user has any permissions for this workspace. If `permission` is null or undefined, the user is not authorized.
    *   **`return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })`**: Returns a 401 "Unauthorized" error.
*   **`const workspaceKeys = await db.select({ ... }).from(apiKey).where(...).orderBy(...)`**: This is the main database query to fetch API keys.
    *   `db.select({...})`: Selects specific columns from the `apiKey` table: `id`, `name`, `key` (this will be the *encrypted* key stored in the DB), `createdAt`, `lastUsed`, `expiresAt`, `createdBy`.
    *   `from(apiKey)`: Specifies the `apiKey` table.
    *   `where(and(eq(apiKey.workspaceId, workspaceId), eq(apiKey.type, 'workspace')))`: Filters for keys associated with the `workspaceId` and where the key's `type` is specifically 'workspace'.
    *   `orderBy(apiKey.createdAt)`: Sorts the results by creation date.
*   **`const formattedWorkspaceKeys = await Promise.all(workspaceKeys.map(async (key) => { ... }))`**: Iterates over each fetched `apiKey` to format it for display.
    *   `Promise.all(...)`: Ensures all mapping operations (which are asynchronous) complete before proceeding.
    *   `workspaceKeys.map(async (key) => { ... })`: For each `key` object:
        *   `const displayFormat = await getApiKeyDisplayFormat(key.key)`: Calls a utility to get a user-friendly, likely masked, version of the API key (e.g., `sim_...xyz`). `key.key` here is the *encrypted* key from the database, which `getApiKeyDisplayFormat` must internally handle (decrypt and then mask).
        *   `return { ...key, key: key.key, displayKey: displayFormat, }`: Creates a new object. It spreads all existing properties from `key`, keeps the original (encrypted) `key` property (though this will not be directly sent to the client, it's maintained in the intermediate object), and adds a new `displayKey` property with the masked format.
*   **`return NextResponse.json({ keys: formattedWorkspaceKeys })`**: Returns the list of formatted API keys as a JSON response with an HTTP 200 OK status.
*   **`catch (error: unknown)`**: Catches any unexpected errors during the `try` block.
    *   **`logger.error(...)`**: Logs the error with the `requestId`.
    *   **`return NextResponse.json({ error: ... }, { status: 500 })`**: Returns a 500 "Internal Server Error" status with a general error message or the error's message if it's an `Error` instance.

#### **`POST` Endpoint (Create API Key)**

```typescript
export async function POST(request: NextRequest, { params }: { params: Promise<{ id: string }> }) {
  const requestId = generateRequestId()
  const workspaceId = (await params).id

  try {
    const session = await getSession()
    if (!session?.user?.id) {
      logger.warn(`[${requestId}] Unauthorized workspace API key creation attempt`)
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
    }

    const userId = session.user.id

    const permission = await getUserEntityPermissions(userId, 'workspace', workspaceId)
    if (!permission || (permission !== 'admin' && permission !== 'write')) {
      return NextResponse.json({ error: 'Forbidden' }, { status: 403 })
    }

    const body = await request.json()
    const { name } = CreateKeySchema.parse(body)

    const existingKey = await db
      .select()
      .from(apiKey)
      .where(
        and(
          eq(apiKey.workspaceId, workspaceId),
          eq(apiKey.name, name),
          eq(apiKey.type, 'workspace')
        )
      )
      .limit(1)

    if (existingKey.length > 0) {
      return NextResponse.json(
        {
          error: `A workspace API key named "${name}" already exists. Please choose a different name.`,
        },
        { status: 409 }
      )
    }

    const { key: plainKey, encryptedKey } = await createApiKey(true)

    if (!encryptedKey) {
      throw new Error('Failed to encrypt API key for storage')
    }

    const [newKey] = await db
      .insert(apiKey)
      .values({
        id: nanoid(),
        workspaceId,
        userId: userId,
        createdBy: userId,
        name,
        key: encryptedKey,
        type: 'workspace',
        createdAt: new Date(),
        updatedAt: new Date(),
      })
      .returning({
        id: apiKey.id,
        name: apiKey.name,
        createdAt: apiKey.createdAt,
      })

    logger.info(`[${requestId}] Created workspace API key: ${name} in workspace ${workspaceId}`)

    return NextResponse.json({
      key: {
        ...newKey,
        key: plainKey, // Send the plain key ONLY upon creation
      },
    })
  } catch (error: unknown) {
    logger.error(`[${requestId}] Workspace API key POST error`, error)
    return NextResponse.json(
      { error: error instanceof Error ? error.message : 'Failed to create workspace API key' },
      { status: 500 }
    )
  }
}
```

*   **`export async function POST(...)`**: Defines the asynchronous function that handles `POST` requests.
*   **`const requestId = generateRequestId()` / `const workspaceId = (await params).id`**: Same as in `GET`, initializes request ID and extracts `workspaceId`.
*   **`try { ... } catch (error: unknown) { ... }`**: Standard error handling.
*   **`const session = await getSession()` / `if (!session?.user?.id)`**: Authenticates the user, same as in `GET`.
*   **`const userId = session.user.id`**: Extracts the user's ID.
*   **`const permission = await getUserEntityPermissions(userId, 'workspace', workspaceId)`**: Checks user permissions.
*   **`if (!permission || (permission !== 'admin' && permission !== 'write'))`**: This is a stricter authorization check than `GET`. For creating a key, the user must have 'admin' or 'write' permissions on the workspace. If not, access is forbidden.
    *   **`return NextResponse.json({ error: 'Forbidden' }, { status: 403 })`**: Returns a 403 "Forbidden" error.
*   **`const body = await request.json()`**: Parses the request body as JSON.
*   **`const { name } = CreateKeySchema.parse(body)`**: Validates the parsed body against `CreateKeySchema` using Zod. If validation fails, Zod throws an error. If successful, it extracts the `name` property.
*   **`const existingKey = await db.select().from(apiKey).where(...)`**: Queries the `apiKey` table to check if a key with the same `name` and `workspaceId` (and `type: 'workspace'`) already exists. This prevents duplicate key names within a workspace.
*   **`if (existingKey.length > 0)`**: If a key with the same name already exists.
    *   **`return NextResponse.json({ error: ... }, { status: 409 })`**: Returns a 409 "Conflict" error.
*   **`const { key: plainKey, encryptedKey } = await createApiKey(true)`**: Calls `createApiKey` to generate a new key.
    *   `plainKey`: The raw, unencrypted API key string (only revealed at creation).
    *   `encryptedKey`: The encrypted version of the API key, suitable for database storage. The `true` argument likely signals that a new key should be generated.
*   **`if (!encryptedKey)`**: Checks if the encryption process failed.
    *   **`throw new Error(...)`**: Throws an error if encryption failed. This will be caught by the outer `try-catch`.
*   **`const [newKey] = await db.insert(apiKey).values({ ... }).returning(...)`**: Inserts the new API key into the database.
    *   `db.insert(apiKey)`: Specifies insertion into the `apiKey` table.
    *   `values({ ... })`: Provides the column values for the new record:
        *   `id: nanoid()`: Generates a unique ID using `nanoid`.
        *   `workspaceId`: The ID of the workspace this key belongs to.
        *   `userId`: The ID of the user who created this key (and is the key's owner).
        *   `createdBy`: Same as `userId`, explicitly noting the creator.
        *   `name`: The user-provided name for the key.
        *   `key: encryptedKey`: Stores the *encrypted* key.
        *   `type: 'workspace'`: Categorizes this as a workspace API key.
        *   `createdAt: new Date()`, `updatedAt: new Date()`: Sets creation and update timestamps.
    *   `.returning({ id: apiKey.id, name: apiKey.name, createdAt: apiKey.createdAt })`: Specifies that after insertion, Drizzle should return these specific columns of the newly inserted row. The `[newKey]` destructuring captures the first (and only) returned row.
*   **`logger.info(...)`**: Logs the successful creation of the API key.
*   **`return NextResponse.json({ key: { ...newKey, key: plainKey } })`**: Returns a JSON response containing the properties of the new key from the database (like `id`, `name`, `createdAt`) and crucially, the `plainKey`. **This is the only time the plain, unencrypted API key is returned to the client.** The HTTP status is 200 OK by default.
*   **`catch (error: unknown)`**: Catches errors, logs them, and returns a 500 status, similar to the `GET` endpoint.

#### **`DELETE` Endpoint (Delete API Keys)**

```typescript
export async function DELETE(
  request: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  const requestId = generateRequestId()
  const workspaceId = (await params).id

  try {
    const session = await getSession()
    if (!session?.user?.id) {
      logger.warn(`[${requestId}] Unauthorized workspace API key deletion attempt`)
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
    }

    const userId = session.user.id

    const permission = await getUserEntityPermissions(userId, 'workspace', workspaceId)
    if (!permission || (permission !== 'admin' && permission !== 'write')) {
      return NextResponse.json({ error: 'Forbidden' }, { status: 403 })
    }

    const body = await request.json()
    const { keys } = DeleteKeysSchema.parse(body)

    const deletedCount = await db
      .delete(apiKey)
      .where(
        and(
          eq(apiKey.workspaceId, workspaceId),
          eq(apiKey.type, 'workspace'),
          inArray(apiKey.id, keys)
        )
      )

    logger.info(
      `[${requestId}] Deleted ${deletedCount} workspace API keys from workspace ${workspaceId}`
    )
    return NextResponse.json({ success: true, deletedCount })
  } catch (error: unknown) {
    logger.error(`[${requestId}] Workspace API key DELETE error`, error)
    return NextResponse.json(
      { error: error instanceof Error ? error.message : 'Failed to delete workspace API keys' },
      { status: 500 }
    )
  }
}
```

*   **`export async function DELETE(...)`**: Defines the asynchronous function that handles `DELETE` requests.
*   **`const requestId = generateRequestId()` / `const workspaceId = (await params).id`**: Same as in `GET` and `POST`.
*   **`try { ... } catch (error: unknown) { ... }`**: Standard error handling.
*   **`const session = await getSession()` / `if (!session?.user?.id)`**: Authenticates the user.
*   **`const userId = session.user.id`**: Extracts the user's ID.
*   **`const permission = await getUserEntityPermissions(userId, 'workspace', workspaceId)` / `if (!permission || (permission !== 'admin' && permission !== 'write'))`**: Authorizes the user. Similar to `POST`, deletion requires 'admin' or 'write' permissions, returning 403 if not met.
*   **`const body = await request.json()`**: Parses the request body.
*   **`const { keys } = DeleteKeysSchema.parse(body)`**: Validates the request body using `DeleteKeysSchema` and extracts the `keys` array (which contains the IDs of the API keys to be deleted).
*   **`const deletedCount = await db.delete(apiKey).where(...)`**: Performs the database deletion.
    *   `db.delete(apiKey)`: Specifies deletion from the `apiKey` table.
    *   `where(and(eq(apiKey.workspaceId, workspaceId), eq(apiKey.type, 'workspace'), inArray(apiKey.id, keys)))`: Filters records to be deleted. It ensures that:
        1.  The key belongs to the specified `workspaceId`.
        2.  The key's `type` is 'workspace'.
        3.  The key's `id` is present in the `keys` array provided in the request body. This prevents accidental deletion of keys not specified or keys from other workspaces.
    *   Drizzle's `delete` method returns the number of rows affected/deleted.
*   **`logger.info(...)`**: Logs the number of deleted keys.
*   **`return NextResponse.json({ success: true, deletedCount })`**: Returns a JSON response indicating success and the count of keys that were deleted.
*   **`catch (error: unknown)`**: Catches errors, logs them, and returns a 500 status, similar to other endpoints.

---

This file provides a comprehensive and secure solution for managing API keys within a multi-tenant workspace environment, leveraging modern TypeScript features, validation libraries, and ORM capabilities.