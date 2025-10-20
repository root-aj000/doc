This TypeScript file defines a Next.js API route responsible for managing API keys for authenticated users. Specifically, it handles two main operations:
1.  **Retrieving** a list of personal API keys for the current user (`GET` request).
2.  **Creating** a new personal API key for the current user (`POST` request).

It emphasizes security by encrypting API keys before storing them and providing a masked version for display, only revealing the full, unencrypted key once during creation.

Let's break down the code in detail.

---

### **Imports**

The file starts by importing necessary modules and utilities:

```typescript
import { db } from '@sim/db'
import { apiKey } from '@sim/db/schema'
import { and, eq } from 'drizzle-orm'
import { nanoid } from 'nanoid'
import { type NextRequest, NextResponse } from 'next/server'
import { createApiKey, getApiKeyDisplayFormat } from '@/lib/api-key/auth'
import { getSession } from '@/lib/auth'
import { createLogger } from '@/lib/logs/console/logger'
```

*   `import { db } from '@sim/db'`: Imports the Drizzle ORM database client instance. This `db` object is used to interact with your database (e.g., perform queries, insertions). `@sim/db` suggests it's an alias for a common database setup within the project.
*   `import { apiKey } from '@sim/db/schema'`: Imports the Drizzle schema definition for the `apiKey` table. This `apiKey` object represents the structure of your API key records in the database, allowing Drizzle to construct type-safe queries.
*   `import { and, eq } from 'drizzle-orm'`: Imports helper functions from Drizzle ORM.
    *   `and`: Used to combine multiple conditions in a `WHERE` clause with a logical "AND".
    *   `eq`: Used to check for equality (`column = value`) in a `WHERE` clause.
*   `import { nanoid } from 'nanoid'`: Imports `nanoid`, a small, fast, and secure library for generating unique string IDs. This is used for creating unique IDs for new API keys.
*   `import { type NextRequest, NextResponse } from 'next/server'`: Imports types and classes specific to Next.js API routes.
    *   `NextRequest`: Represents an incoming HTTP request, providing access to headers, body, query parameters, etc.
    *   `NextResponse`: Used to construct and send an HTTP response back to the client.
*   `import { createApiKey, getApiKeyDisplayFormat } from '@/lib/api-key/auth'`: Imports two custom utility functions related to API key management.
    *   `createApiKey`: Likely responsible for generating a new API key, encrypting it, and returning both the plain (unencrypted) and encrypted versions.
    *   `getApiKeyDisplayFormat`: Takes an API key (likely the encrypted version) and returns a masked, display-friendly format (e.g., `sim_...xyz`).
*   `import { getSession } from '@/lib/auth'`: Imports a utility function to retrieve the current user's session data. This is crucial for authentication and authorization.
*   `import { createLogger } from '@/lib/logs/console/logger'`: Imports a custom logger utility. This is used to log messages (especially errors) to the console or other configured logging services.

---

### **Logger Initialization**

```typescript
const logger = createLogger('ApiKeysAPI')
```

*   `const logger = createLogger('ApiKeysAPI')`: Initializes a logger instance specifically for this API keys module. The string `'ApiKeysAPI'` acts as a label, making it easier to identify the source of log messages.

---

### **GET /api/users/me/api-keys - Get all API keys for the current user**

This function handles `GET` requests to the API endpoint. Its purpose is to retrieve a list of API keys belonging to the authenticated user.

```typescript
export async function GET(request: NextRequest) {
  try {
    const session = await getSession()
    if (!session?.user?.id) {
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
    }

    const userId = session.user.id

    const keys = await db
      .select({
        id: apiKey.id,
        name: apiKey.name,
        key: apiKey.key, // Note: This stores the encrypted key from the DB
        createdAt: apiKey.createdAt,
        lastUsed: apiKey.lastUsed,
        expiresAt: apiKey.expiresAt,
      })
      .from(apiKey)
      .where(and(eq(apiKey.userId, userId), eq(apiKey.type, 'personal')))
      .orderBy(apiKey.createdAt)

    const maskedKeys = await Promise.all(
      keys.map(async (key) => {
        const displayFormat = await getApiKeyDisplayFormat(key.key) // Uses the encrypted key to derive display format
        return {
          ...key,
          key: key.key, // Still includes the encrypted key in the object
          displayKey: displayFormat, // Adds the masked version for display
        }
      })
    )

    return NextResponse.json({ keys: maskedKeys })
  } catch (error) {
    logger.error('Failed to fetch API keys', { error })
    return NextResponse.json({ error: 'Failed to fetch API keys' }, { status: 500 })
  }
}
```

#### **Line-by-line Explanation:**

*   `export async function GET(request: NextRequest)`:
    *   `export async function GET`: Defines an asynchronous function named `GET`. In Next.js API routes, functions named after HTTP methods (`GET`, `POST`, `PUT`, `DELETE`) are automatically called when a request with that method hits the route.
    *   `(request: NextRequest)`: The function receives a `NextRequest` object, which contains details about the incoming request.
*   `try { ... } catch (error) { ... }`: This is a standard `try-catch` block for error handling. Any errors that occur within the `try` block will be caught, logged, and a 500 status response will be sent.

#### **Inside the `try` block:**

1.  **Authentication:**
    ```typescript
    const session = await getSession()
    if (!session?.user?.id) {
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
    }
    const userId = session.user.id
    ```
    *   `const session = await getSession()`: Calls the `getSession` utility to retrieve the current user's session information. This is an asynchronous operation, so `await` is used.
    *   `if (!session?.user?.id)`: Checks if a session exists (`session?`), if it contains a `user` object (`user?`), and if that user has an `id`. If any of these are missing, it means the user is not authenticated.
    *   `return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })`: If not authenticated, it immediately sends a JSON response with an "Unauthorized" error message and an HTTP status code of `401`.
    *   `const userId = session.user.id`: If authenticated, extracts the `id` of the user from the session, which will be used to query their API keys.

2.  **Database Query for API Keys:**
    ```typescript
    const keys = await db
      .select({
        id: apiKey.id,
        name: apiKey.name,
        key: apiKey.key,
        createdAt: apiKey.createdAt,
        lastUsed: apiKey.lastUsed,
        expiresAt: apiKey.expiresAt,
      })
      .from(apiKey)
      .where(and(eq(apiKey.userId, userId), eq(apiKey.type, 'personal')))
      .orderBy(apiKey.createdAt)
    ```
    *   `const keys = await db`: Starts a Drizzle ORM query to fetch data from the database.
    *   `.select({...})`: Specifies the columns to retrieve from the `apiKey` table. We're selecting `id`, `name`, the `key` itself (which is the *encrypted* key stored in the DB), `createdAt`, `lastUsed`, and `expiresAt`.
    *   `.from(apiKey)`: Indicates that the query is targeting the `apiKey` table (defined by the `apiKey` schema import).
    *   `.where(and(eq(apiKey.userId, userId), eq(apiKey.type, 'personal')))`: This is the filtering condition:
        *   `eq(apiKey.userId, userId)`: Ensures that only API keys belonging to the current `userId` are fetched.
        *   `eq(apiKey.type, 'personal')`: Filters for API keys that are of type 'personal'. This implies there might be other types of keys (e.g., 'workspace', 'system') that are not intended for this endpoint.
        *   `and(...)`: Combines these two conditions, meaning both must be true for a row to be selected.
    *   `.orderBy(apiKey.createdAt)`: Sorts the results by the `createdAt` column in ascending order, so the oldest keys appear first.

3.  **Masking API Keys for Display:**
    ```typescript
    const maskedKeys = await Promise.all(
      keys.map(async (key) => {
        const displayFormat = await getApiKeyDisplayFormat(key.key)
        return {
          ...key,
          key: key.key,
          displayKey: displayFormat,
        }
      })
    )
    ```
    *   `keys.map(async (key) => { ... })`: Iterates over each `key` object fetched from the database. The `async` keyword here indicates that the callback function itself performs asynchronous operations.
    *   `const displayFormat = await getApiKeyDisplayFormat(key.key)`: For each key, it calls the `getApiKeyDisplayFormat` utility, passing the `key.key` (which is the *encrypted* API key from the database). This function returns a safe, masked string representation suitable for displaying in a UI (e.g., `sim_...QWERTY`).
    *   `return { ...key, key: key.key, displayKey: displayFormat }`: Creates a new object for each key:
        *   `...key`: Spreads all original properties (id, name, createdAt, etc., including the *encrypted* `key`) from the database result.
        *   `key: key.key`: Explicitly ensures the `key` property (containing the *encrypted* value) is present. While redundant with `...key` if `key.key` is already part of the spread, it explicitly highlights its inclusion.
        *   `displayKey: displayFormat`: Adds a new property `displayKey` which holds the masked, user-friendly version of the API key. This is the value that should be shown to the user.
    *   `const maskedKeys = await Promise.all(...)`: `Promise.all` waits for all the asynchronous `map` operations (each call to `getApiKeyDisplayFormat`) to complete before proceeding. The `maskedKeys` array will then contain all the API key objects with their added `displayKey` property.

4.  **Send Response:**
    ```typescript
    return NextResponse.json({ keys: maskedKeys })
    ```
    *   `return NextResponse.json({ keys: maskedKeys })`: Sends a successful JSON response back to the client. The response body will contain an object with a `keys` property, which is an array of the `maskedKeys`.

#### **Inside the `catch` block:**

```typescript
} catch (error) {
  logger.error('Failed to fetch API keys', { error })
  return NextResponse.json({ error: 'Failed to fetch API keys' }, { status: 500 })
}
```
*   `logger.error('Failed to fetch API keys', { error })`: Logs the error message along with the `error` object itself (for more details) using the `logger` instance.
*   `return NextResponse.json({ error: 'Failed to fetch API keys' }, { status: 500 })`: Sends a generic "Failed to fetch API keys" error message with an HTTP status code of `500` (Internal Server Error).

---

### **POST /api/users/me/api-keys - Create a new API key**

This function handles `POST` requests. Its purpose is to allow an authenticated user to create a new personal API key.

```typescript
export async function POST(request: NextRequest) {
  try {
    const session = await getSession()
    if (!session?.user?.id) {
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
    }

    const userId = session.user.id
    const body = await request.json()

    const { name: rawName } = body
    if (!rawName || typeof rawName !== 'string') {
      return NextResponse.json({ error: 'Invalid request. Name is required.' }, { status: 400 })
    }

    const name = rawName.trim()
    if (!name) {
      return NextResponse.json({ error: 'Name cannot be empty.' }, { status: 400 })
    }

    const existingKey = await db
      .select()
      .from(apiKey)
      .where(and(eq(apiKey.userId, userId), eq(apiKey.name, name), eq(apiKey.type, 'personal')))
      .limit(1)

    if (existingKey.length > 0) {
      return NextResponse.json(
        {
          error: `A personal API key named "${name}" already exists. Please choose a different name.`,
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
        userId,
        workspaceId: null,
        name,
        key: encryptedKey,
        type: 'personal',
        createdAt: new Date(),
        updatedAt: new Date(),
      })
      .returning({
        id: apiKey.id,
        name: apiKey.name,
        createdAt: apiKey.createdAt,
      })

    return NextResponse.json({
      key: {
        ...newKey,
        key: plainKey, // Only return the plain key once, at creation
      },
    })
  } catch (error) {
    logger.error('Failed to create API key', { error })
    return NextResponse.json({ error: 'Failed to create API key' }, { status: 500 })
  }
}
```

#### **Line-by-line Explanation:**

*   `export async function POST(request: NextRequest)`:
    *   `export async function POST`: Defines an asynchronous function for `POST` requests, similar to `GET`.
*   `try { ... } catch (error) { ... }`: Standard `try-catch` block for error handling.

#### **Inside the `try` block:**

1.  **Authentication:**
    ```typescript
    const session = await getSession()
    if (!session?.user?.id) {
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
    }
    const userId = session.user.id
    ```
    *   This block is identical to the `GET` handler, ensuring that only authenticated users can create API keys.

2.  **Parse Request Body and Validate Input:**
    ```typescript
    const body = await request.json()

    const { name: rawName } = body
    if (!rawName || typeof rawName !== 'string') {
      return NextResponse.json({ error: 'Invalid request. Name is required.' }, { status: 400 })
    }

    const name = rawName.trim()
    if (!name) {
      return NextResponse.json({ error: 'Name cannot be empty.' }, { status: 400 })
    }
    ```
    *   `const body = await request.json()`: Parses the incoming request body as JSON. This is where the client sends the desired `name` for the new API key.
    *   `const { name: rawName } = body`: Destructures the `name` property from the `body` object and assigns it to a variable `rawName`.
    *   `if (!rawName || typeof rawName !== 'string')`: Checks if `rawName` is missing (null, undefined, empty string) or if it's not a string type. If either is true, it's an invalid request.
    *   `return NextResponse.json({ error: 'Invalid request. Name is required.' }, { status: 400 })`: Sends a "Bad Request" (400) response if the name is invalid or missing.
    *   `const name = rawName.trim()`: Trims any leading or trailing whitespace from the `rawName`.
    *   `if (!name)`: After trimming, checks if the `name` is an empty string.
    *   `return NextResponse.json({ error: 'Name cannot be empty.' }, { status: 400 })`: Sends a "Bad Request" (400) response if the name is empty after trimming.

3.  **Check for Existing Key with Same Name:**
    ```typescript
    const existingKey = await db
      .select()
      .from(apiKey)
      .where(and(eq(apiKey.userId, userId), eq(apiKey.name, name), eq(apiKey.type, 'personal')))
      .limit(1)

    if (existingKey.length > 0) {
      return NextResponse.json(
        {
          error: `A personal API key named "${name}" already exists. Please choose a different name.`,
        },
        { status: 409 }
      )
    }
    ```
    *   `const existingKey = await db.select().from(apiKey).where(...)`: Queries the database to check if an API key with the same `userId`, `name`, and `type: 'personal'` already exists.
        *   `.select()`: Selects all columns (we only care if any row matches, not specific data).
        *   `.limit(1)`: Optimizes the query by telling the database to stop searching after finding the first match.
    *   `if (existingKey.length > 0)`: If the query returns any results (meaning an existing key was found).
    *   `return NextResponse.json({ error: `A personal API key named "${name}" already exists...` }, { status: 409 })`: Sends a "Conflict" (409) response, indicating that the requested resource (an API key with that name) already exists for the user.

4.  **Generate and Encrypt API Key:**
    ```typescript
    const { key: plainKey, encryptedKey } = await createApiKey(true)

    if (!encryptedKey) {
      throw new Error('Failed to encrypt API key for storage')
    }
    ```
    *   `const { key: plainKey, encryptedKey } = await createApiKey(true)`: Calls the `createApiKey` utility. This function is expected to:
        *   Generate a new, unencrypted API key (the `plainKey`).
        *   Encrypt that `plainKey` for secure storage (the `encryptedKey`).
        *   The `true` argument likely signifies that this is a "personal" key or a key requiring encryption.
    *   `if (!encryptedKey)`: Performs a safety check to ensure that the encryption process was successful and an `encryptedKey` was actually returned.
    *   `throw new Error('Failed to encrypt API key for storage')`: If encryption fails, an error is thrown, which will be caught by the outer `try-catch` block.

5.  **Insert New API Key into Database:**
    ```typescript
    const [newKey] = await db
      .insert(apiKey)
      .values({
        id: nanoid(),
        userId,
        workspaceId: null,
        name,
        key: encryptedKey,
        type: 'personal',
        createdAt: new Date(),
        updatedAt: new Date(),
      })
      .returning({
        id: apiKey.id,
        name: apiKey.name,
        createdAt: apiKey.createdAt,
      })
    ```
    *   `const [newKey] = await db.insert(apiKey)`: Starts a Drizzle ORM insertion query into the `apiKey` table. `[newKey]` is used to destructure the first element from the array returned by `returning()`.
    *   `.values({...})`: Defines the values to insert for the new API key:
        *   `id: nanoid()`: Generates a unique ID for the new key using `nanoid`.
        *   `userId`: The ID of the authenticated user.
        *   `workspaceId: null`: Indicates this is a personal key, not associated with a specific workspace.
        *   `name`: The validated name provided by the user.
        *   `key: encryptedKey`: **Crucially, the *encrypted* version of the API key is stored here, never the plain key.**
        *   `type: 'personal'`: Sets the key's type.
        *   `createdAt: new Date(), updatedAt: new Date()`: Sets the creation and update timestamps to the current date and time.
    *   `.returning({ id: apiKey.id, name: apiKey.name, createdAt: apiKey.createdAt })`: Specifies that after insertion, the database should return the `id`, `name`, and `createdAt` columns of the newly inserted row. This avoids a separate `SELECT` query.

6.  **Send Success Response:**
    ```typescript
    return NextResponse.json({
      key: {
        ...newKey,
        key: plainKey,
      },
    })
    ```
    *   `return NextResponse.json({ key: { ...newKey, key: plainKey } })`: Sends a successful JSON response.
        *   `...newKey`: Spreads the properties returned from the database insertion (`id`, `name`, `createdAt`).
        *   `key: plainKey`: **This is the *only time* the actual, unencrypted `plainKey` is returned to the client.** This is a critical security measure. The user needs to see the key once to copy it, but it should never be stored in the database in this plain format or retrieved this way again.

#### **Inside the `catch` block:**

```typescript
} catch (error) {
  logger.error('Failed to create API key', { error })
  return NextResponse.json({ error: 'Failed to create API key' }, { status: 500 })
}
```
*   `logger.error('Failed to create API key', { error })`: Logs the error message along with the `error` object.
*   `return NextResponse.json({ error: 'Failed to create API key' }, { status: 500 })`: Sends a generic "Failed to create API key" error message with an HTTP status code of `500`.

---

### **Summary of Security and Best Practices:**

*   **Authentication:** Both endpoints are protected by checking for a valid user session, preventing unauthorized access.
*   **API Key Encryption:** API keys are *encrypted* before being stored in the database (`encryptedKey`). This means if the database is compromised, the actual API keys are not immediately exposed.
*   **One-Time Plain Key Disclosure:** The unencrypted `plainKey` is only returned to the client once, immediately after creation. It is never stored in the database in plain text and never retrieved in plain text via the `GET` endpoint.
*   **Masked Display:** The `GET` endpoint provides a `displayKey` (masked representation) instead of the full API key, ensuring that even if the API response is intercepted, the full key is not revealed.
*   **Input Validation:** The `POST` endpoint rigorously validates the provided name, checking for existence, type, and non-empty values.
*   **Conflict Detection:** Before creating a new key, the `POST` endpoint checks for existing keys with the same name for the same user, preventing duplicates and providing a helpful error message.
*   **Unique IDs:** `nanoid` is used to generate strong, unique IDs for API keys.
*   **Structured Logging:** A dedicated logger is used to record errors, which is vital for monitoring and debugging issues in a production environment.
*   **Standard HTTP Status Codes:** Appropriate HTTP status codes are used (401 Unauthorized, 400 Bad Request, 409 Conflict, 500 Internal Server Error, 200 OK) for clear communication with clients.

This file demonstrates a robust and secure approach to managing user-specific API keys within a Next.js application.