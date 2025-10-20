This TypeScript file defines a Next.js API route that acts as a secure intermediary for fetching "items" (specifically contacts) from the Wealthbox CRM platform. It handles user authentication, authorization, refreshing access tokens, querying the Wealthbox API, and transforming the data before returning it to the client.

Let's break down its purpose, what it does, and then delve into a line-by-line explanation.

---

### Purpose of this File and What it Does

**Purpose:**
The primary purpose of this file is to provide a secure and controlled endpoint (`/api/wealthbox/items`) for a client-side application to retrieve information (like contacts) from a user's linked Wealthbox account. It ensures that:
1.  Only authenticated users can make requests.
2.  Users can only access data from Wealthbox accounts they have linked and own.
3.  OAuth access tokens for Wealthbox are managed and refreshed automatically.
4.  Data from Wealthbox is fetched, potentially filtered, and then transformed into a consistent format for the client application.

**What it Does (High-Level):**

1.  **Receives a Request:** Listens for `GET` requests from the client.
2.  **Authenticates User:** Verifies that the user making the request is logged in.
3.  **Parses Parameters:** Extracts `credentialId`, `type` (e.g., 'contact'), and `query` from the URL.
4.  **Validates Input:** Ensures essential parameters are present and valid (e.g., only 'contact' type is supported currently).
5.  **Authorizes Access:** Fetches the Wealthbox account credentials from the database and confirms they belong to the authenticated user.
6.  **Manages OAuth Tokens:** Automatically refreshes the Wealthbox access token if it's expired or invalid.
7.  **Calls Wealthbox API:** Makes a secure `GET` request to the Wealthbox CRM API using the validated and refreshed access token.
8.  **Handles API Response:** Checks for errors from Wealthbox, reads the data.
9.  **Transforms Data:** Converts the Wealthbox-specific data structure into a standardized `item` format.
10. **Filters Data:** Applies client-side filtering based on a `query` parameter if provided.
11. **Responds to Client:** Returns the transformed and filtered items or an appropriate error message.
12. **Logs Activity:** Uses a custom logger to record key events, warnings, and errors for monitoring and debugging.

---

### Simplified Complex Logic

The most complex parts involve:

1.  **OAuth Token Management (`refreshAccessTokenIfNeeded`):** Instead of directly handling token expiration and refresh logic in this file, it delegates this to a utility function. This function likely checks if the stored access token for the given `credentialId` is still valid. If not, it uses the stored refresh token to get a new access token and updates it in the database before returning it. This ensures continuous access to Wealthbox without requiring the user to re-authenticate frequently.
2.  **Data Transformation (`items = contacts.map(...)`):** The raw JSON response from Wealthbox might have a specific structure (e.g., `item.first_name`, `item.background_information`). This code takes that specific structure and maps it to a generic `item` structure (`id`, `name`, `type`, `content`, `createdAt`, `updatedAt`) that the client application expects, simplifying client-side data handling.
3.  **Error Handling & Logging:** Throughout the process, the code proactively checks for potential issues (unauthenticated user, missing parameters, credential not found, API errors) and logs them with a unique `requestId` for traceability. It then returns specific HTTP status codes and error messages to the client, making it easier to diagnose problems.

---

### Line-by-Line Explanation

```typescript
import { db } from '@sim/db'
```

*   **`import { db } from '@sim/db'`**: This line imports the Drizzle ORM database client instance, named `db`, from a module located at `@sim/db`. This `db` object is used to interact with your application's database (e.g., fetching user credentials).

```typescript
import { account } from '@sim/db/schema'
```

*   **`import { account } from '@sim/db/schema'`**: This imports the `account` schema definition from your database schema. In Drizzle ORM, schema definitions represent your database tables. The `account` table likely stores information about linked external accounts, including Wealthbox credentials and their associated user ID.

```typescript
import { eq } from 'drizzle-orm'
```

*   **`import { eq } from 'drizzle-orm'`**: This imports the `eq` (equals) function from the `drizzle-orm` library. `eq` is a Drizzle-specific comparison operator used to build `WHERE` clauses in database queries (e.g., `WHERE id = 'someId'`).

```typescript
import { type NextRequest, NextResponse } from 'next/server'
```

*   **`import { type NextRequest, NextResponse } from 'next/server'`**: These are essential types and classes for building Next.js API routes.
    *   `NextRequest` (type): Represents an incoming HTTP request in a Next.js server environment. It provides access to request properties like URL, headers, and body.
    *   `NextResponse` (class): Used to construct and return HTTP responses from Next.js API routes. It allows setting status codes, headers, and JSON bodies.

```typescript
import { getSession } from '@/lib/auth'
```

*   **`import { getSession } from '@/lib/auth'`**: This imports a utility function `getSession` from a local authentication library. This function is responsible for retrieving the current user's session information (e.g., user ID, authentication status), typically from a cookie or token.

```typescript
import { createLogger } from '@/lib/logs/console/logger'
```

*   **`import { createLogger } from '@/lib/logs/console/logger'`**: This imports a custom `createLogger` function. This function is used to create a logger instance that can output structured logs to the console or other destinations, often with a specified context or name.

```typescript
import { generateRequestId } from '@/lib/utils'
```

*   **`import { generateRequestId } from '@/lib/utils'`**: This imports a utility function `generateRequestId`. Its purpose is to create a unique identifier for each incoming request, which is extremely useful for tracing requests through logs and debugging.

```typescript
import { refreshAccessTokenIfNeeded } from '@/app/api/auth/oauth/utils'
```

*   **`import { refreshAccessTokenIfNeeded } from '@/app/api/auth/oauth/utils'`**: This imports a critical OAuth utility function. This function handles the logic for checking if an OAuth access token is still valid and, if not, using a refresh token to obtain a new access token, updating it in the database.

```typescript
export const dynamic = 'force-dynamic'
```

*   **`export const dynamic = 'force-dynamic'`**: This is a Next.js specific configuration for API routes. Setting it to `'force-dynamic'` ensures that this API route is always executed on demand at request time, preventing it from being statically cached. This is crucial for routes that handle dynamic data and user-specific logic like authentication and database lookups.

```typescript
const logger = createLogger('WealthboxItemsAPI')
```

*   **`const logger = createLogger('WealthboxItemsAPI')`**: This line initializes a logger instance specifically for this API route. All log messages originating from this file will be tagged with `'WealthboxItemsAPI'`, making it easier to filter and understand logs.

```typescript
/**
 * Get items (notes, contacts, tasks) from Wealthbox
 */
export async function GET(request: NextRequest) {
```

*   **`export async function GET(request: NextRequest)`**: This defines the main API route handler.
    *   `export`: Makes the function available as the handler for the API route. Next.js API routes expose functions matching HTTP methods (`GET`, `POST`, `PUT`, `DELETE`).
    *   `async`: Indicates that this function will perform asynchronous operations (like database calls, API requests, `await`ing promises).
    *   `GET`: This function will specifically handle incoming HTTP `GET` requests to this API route.
    *   `request: NextRequest`: The function accepts a `NextRequest` object as its only argument, providing access to all request details.

```typescript
  const requestId = generateRequestId()
```

*   **`const requestId = generateRequestId()`**: At the very beginning of the request, a unique ID is generated. This `requestId` will be used in all log messages for this specific request, allowing for end-to-end tracing in logs.

```typescript
  try {
```

*   **`try {`**: This marks the beginning of a `try...catch` block. Any code within this block that throws an error will be caught by the `catch` block, preventing the server from crashing and allowing for graceful error handling.

```typescript
    // Get the session
    const session = await getSession()
```

*   **`const session = await getSession()`**: This asynchronously calls the `getSession` utility to retrieve the current user's session information. This typically involves reading a session token or cookie and verifying its validity.

```typescript
    // Check if the user is authenticated
    if (!session?.user?.id) {
      logger.warn(`[${requestId}] Unauthenticated request rejected`)
      return NextResponse.json({ error: 'User not authenticated' }, { status: 401 })
    }
```

*   **`if (!session?.user?.id)`**: This is a crucial authentication check. It verifies if a `session` exists and if that session contains a `user` object with an `id`. If any of these are missing (meaning the user is not authenticated), the request is rejected.
*   **`logger.warn(...)`**: A warning message is logged with the `requestId`, indicating an unauthenticated attempt.
*   **`return NextResponse.json({ error: 'User not authenticated' }, { status: 401 })`**: An HTTP 401 (Unauthorized) response is returned to the client, indicating that authentication is required.

```typescript
    // Get parameters from query
    const { searchParams } = new URL(request.url)
    const credentialId = searchParams.get('credentialId')
    const type = searchParams.get('type') || 'contact'
    const query = searchParams.get('query') || ''
```

*   **`const { searchParams } = new URL(request.url)`**: This extracts the query parameters from the request URL. `new URL(request.url)` parses the full URL, and `.searchParams` gives access to its query string component (e.g., `?credentialId=abc&type=contact`).
*   **`const credentialId = searchParams.get('credentialId')`**: Retrieves the value of the `credentialId` query parameter. This ID identifies a specific Wealthbox account linked by the user.
*   **`const type = searchParams.get('type') || 'contact'`**: Retrieves the `type` of item to fetch (e.g., 'contact', 'note', 'task'). If `type` is not provided in the query, it defaults to `'contact'`.
*   **`const query = searchParams.get('query') || ''`**: Retrieves a search `query` string. If not provided, it defaults to an empty string. This will be used for client-side filtering.

```typescript
    if (!credentialId) {
      logger.warn(`[${requestId}] Missing credential ID`)
      return NextResponse.json({ error: 'Credential ID is required' }, { status: 400 })
    }
```

*   **`if (!credentialId)`**: This validates that the `credentialId` query parameter was provided. It's essential for identifying which Wealthbox account to query.
*   **`logger.warn(...)`**: Logs a warning if `credentialId` is missing.
*   **`return NextResponse.json({ error: 'Credential ID is required' }, { status: 400 })`**: Returns an HTTP 400 (Bad Request) response, indicating a missing required parameter.

```typescript
    // Validate item type - only handle contacts now
    if (type !== 'contact') {
      logger.warn(`[${requestId}] Invalid item type: ${type}`)
      return NextResponse.json(
        { error: 'Invalid item type. Only contact is supported.' },
        { status: 400 }
      )
    }
```

*   **`if (type !== 'contact')`**: This validates the `type` parameter. Currently, the API route is designed to only fetch 'contact' items from Wealthbox. If a different type is requested, it's considered an invalid request.
*   **`logger.warn(...)`**: Logs a warning about the invalid item type.
*   **`return NextResponse.json(...)`**: Returns an HTTP 400 (Bad Request) with an error message specific to the unsupported type.

```typescript
    // Get the credential from the database
    const credentials = await db.select().from(account).where(eq(account.id, credentialId)).limit(1)
```

*   **`const credentials = await db.select().from(account).where(eq(account.id, credentialId)).limit(1)`**: This is a Drizzle ORM query to fetch the Wealthbox credential details from your database.
    *   `db.select()`: Starts a `SELECT` query.
    *   `.from(account)`: Specifies that the query is against the `account` table (which stores linked external accounts).
    *   `.where(eq(account.id, credentialId))`: Filters the results to find the row where the `id` column matches the `credentialId` from the request. `eq` is the "equals" comparison operator.
    *   `.limit(1)`: Ensures that only a maximum of one matching credential is fetched.

```typescript
    if (!credentials.length) {
      logger.warn(`[${requestId}] Credential not found`, { credentialId })
      return NextResponse.json({ error: 'Credential not found' }, { status: 404 })
    }
```

*   **`if (!credentials.length)`**: After querying the database, this checks if any credential was found. If the `credentials` array is empty, it means no record matches the provided `credentialId`.
*   **`logger.warn(...)`**: Logs a warning indicating the credential wasn't found, including the `credentialId` for context.
*   **`return NextResponse.json(...)`**: Returns an HTTP 404 (Not Found) response if the credential doesn't exist.

```typescript
    const credential = credentials[0]
```

*   **`const credential = credentials[0]`**: Since `limit(1)` was used, if `credentials.length` is not zero, `credentials[0]` will contain the single credential object that was fetched. This line extracts that object for easier access.

```typescript
    // Check if the credential belongs to the user
    if (credential.userId !== session.user.id) {
      logger.warn(`[${requestId}] Unauthorized credential access attempt`, {
        credentialUserId: credential.userId,
        requestUserId: session.user.id,
      })
      return NextResponse.json({ error: 'Unauthorized' }, { status: 403 })
    }
```

*   **`if (credential.userId !== session.user.id)`**: This is an authorization check. It compares the `userId` associated with the fetched credential (from the database) against the `id` of the currently authenticated user (from the session). This prevents a user from accessing or manipulating credentials linked by another user.
*   **`logger.warn(...)`**: Logs a warning about the unauthorized access attempt, providing both the credential's `userId` and the requesting user's `id` for debugging.
*   **`return NextResponse.json(...)`**: Returns an HTTP 403 (Forbidden) response if the user is not authorized to access this credential.

```typescript
    // Refresh access token if needed
    const accessToken = await refreshAccessTokenIfNeeded(credentialId, session.user.id, requestId)
```

*   **`const accessToken = await refreshAccessTokenIfNeeded(credentialId, session.user.id, requestId)`**: This calls the utility function to manage the Wealthbox OAuth access token.
    *   It takes the `credentialId` (to identify which account's tokens to manage), the `session.user.id` (for authorization within the utility), and the `requestId` (for consistent logging).
    *   This function will internally check if the existing access token is valid. If not, it will use the refresh token to get a new access token from Wealthbox and update it in your database, then return the valid access token.

```typescript
    if (!accessToken) {
      logger.error(`[${requestId}] Failed to obtain valid access token`)
      return NextResponse.json({ error: 'Failed to obtain valid access token' }, { status: 401 })
    }
```

*   **`if (!accessToken)`**: If `refreshAccessTokenIfNeeded` fails to provide a valid `accessToken` (e.g., the refresh token itself expired, or there's an issue with the OAuth flow), this condition catches it.
*   **`logger.error(...)`**: Logs an error indicating the failure to get a token.
*   **`return NextResponse.json(...)`**: Returns an HTTP 401 (Unauthorized) response, as further API calls to Wealthbox would fail without a valid token.

```typescript
    // Use correct endpoints based on documentation - only for contacts
    const endpoints = {
      contact: 'contacts',
    }
    const endpoint = endpoints[type as keyof typeof endpoints]
```

*   **`const endpoints = { contact: 'contacts', }`**: This object maps the `type` parameter (which is currently fixed to `'contact'`) to the corresponding path segment required by the Wealthbox API (e.g., for contacts, the path might be `/v1/contacts`).
*   **`const endpoint = endpoints[type as keyof typeof endpoints]`**: This safely retrieves the correct API endpoint string based on the validated `type` variable. `type as keyof typeof endpoints` is a TypeScript assertion to ensure type safety, telling the compiler that `type` will be a valid key of the `endpoints` object.

```typescript
    // Build URL - using correct API base URL
    const url = new URL(`https://api.crmworkspace.com/v1/${endpoint}`)
```

*   **`const url = new URL(`https://api.crmworkspace.com/v1/${endpoint}`) `**: This constructs the full URL for the Wealthbox API request. It uses the base URL `https://api.crmworkspace.com/v1/` and appends the `endpoint` (e.g., `contacts`).

```typescript
    logger.info(`[${requestId}] Fetching ${type}s from Wealthbox`, {
      endpoint,
      url: url.toString(),
      hasQuery: !!query.trim(),
    })
```

*   **`logger.info(...)`**: Logs an informational message indicating that the API is about to fetch items from Wealthbox. It includes useful context: the `endpoint`, the full `url`, and a boolean `hasQuery` to show if a search query was provided.

```typescript
    // Make request to Wealthbox API
    const response = await fetch(url.toString(), {
      headers: {
        Authorization: `Bearer ${accessToken}`,
        'Content-Type': 'application/json',
      },
    })
```

*   **`const response = await fetch(url.toString(), { ... })`**: This performs the actual HTTP `GET` request to the Wealthbox API.
    *   `url.toString()`: Converts the `URL` object back into a string for the `fetch` function.
    *   `headers`: Specifies the HTTP headers for the request.
        *   `Authorization: `Bearer ${accessToken}``: This is the standard way to send an OAuth 2.0 access token. The `Bearer` scheme indicates that the token is a bearer token, meaning whoever holds it can use it.
        *   `'Content-Type': 'application/json'`: Specifies that the client expects a JSON response, although for a GET request, this header is often less critical than for POST/PUT requests.

```typescript
    if (!response.ok) {
      const errorText = await response.text()
      logger.error(
        `[${requestId}] Wealthbox API error: ${response.status} ${response.statusText}`,
        {
          error: errorText,
          endpoint,
          url: url.toString(),
        }
      )
      return NextResponse.json(
        { error: `Failed to fetch ${type}s from Wealthbox` },
        { status: response.status }
      )
    }
```

*   **`if (!response.ok)`**: This checks if the HTTP response from Wealthbox was successful (i.e., status code 200-299). If `response.ok` is `false`, it indicates an error from the Wealthbox API.
*   **`const errorText = await response.text()`**: If there's an error, this attempts to read the response body as plain text, which often contains more detailed error messages from the external API.
*   **`logger.error(...)`**: An error message is logged, including the `requestId`, the Wealthbox API's HTTP status and status text, the error body, the `endpoint`, and the `url`.
*   **`return NextResponse.json(...)`**: An error response is sent back to the client, indicating a failure to fetch items from Wealthbox, using the same HTTP status code that Wealthbox returned (e.g., 400, 403, 500).

```typescript
    const data = await response.json()
```

*   **`const data = await response.json()`**: If the Wealthbox API response was successful (`response.ok` was true), this line parses the response body as JSON and stores it in the `data` variable.

```typescript
    logger.info(`[${requestId}] Wealthbox API response structure`, {
      type,
      status: response.status,
      dataKeys: Object.keys(data || {}),
      hasContacts: !!data.contacts,
      dataStructure: typeof data === 'object' ? Object.keys(data) : 'not an object',
    })
```

*   **`logger.info(...)`**: This logs detailed information about the successful Wealthbox API response, which can be very helpful for understanding the shape of the data received and debugging. It includes:
    *   `type`: The requested item type.
    *   `status`: The HTTP status code from Wealthbox.
    *   `dataKeys`: The top-level keys of the JSON response object (or an empty array if `data` is null/undefined).
    *   `hasContacts`: A boolean indicating if the `data` object contains a `contacts` property.
    *   `dataStructure`: More general information about the structure of the `data`.

```typescript
    // Transform the response based on type and correct response format
    let items: any[] = []
```

*   **`let items: any[] = []`**: An empty array named `items` is initialized. This array will store the transformed data in a standardized format that your client application expects. `any[]` is used for flexibility during transformation, but a more specific type would be ideal if the target shape is well-known.

```typescript
    if (type === 'contact') {
      const contacts = data.contacts || []
      if (!Array.isArray(contacts)) {
        logger.warn(`[${requestId}] Contacts is not an array`, {
          contacts,
          dataType: typeof contacts,
        })
        return NextResponse.json({ items: [] }, { status: 200 })
      }

      items = contacts.map((item: any) => ({
        id: item.id?.toString() || '',
        name: `${item.first_name || ''} ${item.last_name || ''}`.trim() || `Contact ${item.id}`,
        type: 'contact',
        content: item.background_information || '',
        createdAt: item.created_at,
        updatedAt: item.updated_at,
      }))
    }
```

*   **`if (type === 'contact') { ... }`**: This block executes specifically if the requested `type` was 'contact'.
    *   **`const contacts = data.contacts || []`**: Safely extracts the `contacts` array from the Wealthbox response data. If `data.contacts` is `null` or `undefined`, it defaults to an empty array to prevent errors.
    *   **`if (!Array.isArray(contacts))`**: An additional check to ensure that `contacts` is indeed an array, as expected. If not, a warning is logged, and an empty `items` array is returned.
    *   **`items = contacts.map((item: any) => ({ ... }))`**: This is the core data transformation logic. It iterates over each `contact` item received from Wealthbox and maps it to a new, standardized object structure.
        *   `id: item.id?.toString() || ''`: Transforms the `id` to a string, handling potential null/undefined `id`.
        *   `name: `${item.first_name || ''} ${item.last_name || ''}`.trim() || `Contact ${item.id}``: Creates a full name from first and last names, handles missing names, and provides a fallback name.
        *   `type: 'contact'`: Sets the generic type.
        *   `content: item.background_information || ''`: Maps Wealthbox's `background_information` to a generic `content` field.
        *   `createdAt: item.created_at`: Direct mapping for creation timestamp.
        *   `updatedAt: item.updated_at`: Direct mapping for update timestamp.

```typescript
    // Apply client-side filtering if query is provided
    if (query.trim()) {
      const searchTerm = query.trim().toLowerCase()
      items = items.filter(
        (item) =>
          item.name.toLowerCase().includes(searchTerm) ||
          item.content.toLowerCase().includes(searchTerm)
      )
    }
```

*   **`if (query.trim()) { ... }`**: This block applies filtering if the `query` parameter (from the initial request) was provided and is not just whitespace.
    *   **`const searchTerm = query.trim().toLowerCase()`**: The search query is trimmed of whitespace and converted to lowercase for case-insensitive matching.
    *   **`items = items.filter(...)`**: The `items` array (after transformation) is filtered. Only items whose `name` or `content` (both converted to lowercase) include the `searchTerm` will be kept.

```typescript
    logger.info(`[${requestId}] Successfully fetched ${items.length} ${type}s from Wealthbox`, {
      totalItems: items.length,
      hasSearchQuery: !!query.trim(),
    })
```

*   **`logger.info(...)`**: An informational message is logged upon successful completion, indicating how many items were fetched and whether a search query was applied.

```typescript
    return NextResponse.json({ items }, { status: 200 })
  } catch (error) {
```

*   **`return NextResponse.json({ items }, { status: 200 })`**: If everything succeeds, the transformed and filtered `items` array is returned to the client as a JSON response with an HTTP 200 (OK) status code.
*   **`} catch (error) {`**: This is the `catch` block for the `try...catch` statement. If any unexpected error occurs within the `try` block, execution jumps here.

```typescript
    logger.error(`[${requestId}] Error fetching Wealthbox items`, error)
    return NextResponse.json({ error: 'Internal server error' }, { status: 500 })
  }
}
```

*   **`logger.error(...)`**: An error message is logged, including the `requestId` and the actual `error` object. This is crucial for debugging unexpected issues.
*   **`return NextResponse.json({ error: 'Internal server error' }, { status: 500 })`**: A generic HTTP 500 (Internal Server Error) response is returned to the client. This is a standard practice to avoid leaking sensitive server-side error details to the client when an unhandled exception occurs.