This file defines an API endpoint for a Next.js application that allows authenticated users to fetch a list of folders from their Microsoft OneDrive account. It acts as a secure intermediary between your application's frontend and the Microsoft Graph API, handling authentication, authorization, and data transformation.

---

### **Simplified Explanation of What This Code Does**

Imagine you have an app that needs to show a user their folders from OneDrive. This code is the "worker" that makes that happen.

1.  **Gets a Request:** When your app's frontend asks for OneDrive folders, this code receives that request.
2.  **Checks Who You Are:** It first makes sure you're logged into *your* app. If not, it stops you.
3.  **Finds Your OneDrive Connection:** It looks up the specific connection you've set up for OneDrive (called a "credential"). It also verifies that this connection belongs to *you*.
4.  **Refreshes Your OneDrive Access:** Microsoft access tokens expire. This code makes sure your connection to OneDrive is still active, refreshing it if needed.
5.  **Talks to OneDrive (Microsoft Graph API):** Using the fresh access token, it sends a request to Microsoft's servers, asking for a list of folders in your OneDrive root directory. If you provided a search query, it includes that.
6.  **Handles Errors:** If anything goes wrong (e.g., OneDrive is unreachable, or your token is invalid), it catches the error and sends back a helpful message.
7.  **Filters and Formats:** OneDrive sends back a lot of data. This code filters it to *only* show folders and then simplifies the information into a format that's easier for your app to use.
8.  **Sends Back Folders:** Finally, it sends the nicely formatted list of folders back to your app's frontend.

In essence, it's a secure gateway to your OneDrive folders, ensuring only authorized users can access their own data, and presenting that data in a clean, usable way.

---

### **Detailed Line-by-Line Explanation**

Let's break down every part of the code.

```typescript
import { randomUUID } from 'crypto'
```
*   **Purpose:** Imports the `randomUUID` function from Node.js's built-in `crypto` module.
*   **Explanation:** This function generates a Universally Unique Identifier (UUID), which is a unique string. In this file, it's used to create a short, unique `requestId` to help trace logs and errors, especially in a distributed system where multiple requests might be processed concurrently.

```typescript
import { db } from '@sim/db'
import { account } from '@sim/db/schema'
```
*   **Purpose:** Imports database-related objects.
*   **Explanation:**
    *   `db`: This is likely an instance of a database connection (e.g., using Drizzle ORM, as indicated by `drizzle-orm` import later). It's used to interact with your application's database. The `@sim/db` alias suggests it's coming from a shared database utility within your project.
    *   `account`: This refers to a database schema definition, specifically for the `account` table. This table likely stores information about user credentials for external services, such as Microsoft OneDrive. The `@sim/db/schema` alias points to where your database schema definitions are located.

```typescript
import { eq } from 'drizzle-orm'
```
*   **Purpose:** Imports the `eq` function from the `drizzle-orm` library.
*   **Explanation:** Drizzle ORM is a TypeScript-first ORM (Object-Relational Mapper). The `eq` function is a utility provided by Drizzle that allows you to create an "equals" condition for database queries. For example, `eq(column, value)` means "where `column` is equal to `value`".

```typescript
import { type NextRequest, NextResponse } from 'next/server'
```
*   **Purpose:** Imports types and classes for handling API requests and responses in a Next.js application.
*   **Explanation:**
    *   `type NextRequest`: This is a TypeScript type definition for the incoming HTTP request object in a Next.js API route. It extends the standard `Request` object with Next.js-specific features, like access to `url` or `searchParams`.
    *   `NextResponse`: This is a class used to create and send HTTP responses from a Next.js API route. It provides methods to set status codes, headers, and send JSON or other data.

```typescript
import { getSession } from '@/lib/auth'
```
*   **Purpose:** Imports a utility function to retrieve the current user's session information.
*   **Explanation:** `getSession` is likely a custom function that integrates with your application's authentication system (e.g., NextAuth.js or a custom solution). It asynchronously fetches and returns the details of the currently logged-in user's session.

```typescript
import { createLogger } from '@/lib/logs/console/logger'
```
*   **Purpose:** Imports a function to create a logger instance.
*   **Explanation:** `createLogger` is a custom utility for logging messages within your application. This specific import suggests a console-based logger is being used, allowing for structured logging with different severity levels (e.g., info, warn, error) and potentially adding context like the module name.

```typescript
import { refreshAccessTokenIfNeeded } from '@/app/api/auth/oauth/utils'
```
*   **Purpose:** Imports a utility function specifically designed to manage OAuth access tokens.
*   **Explanation:** `refreshAccessTokenIfNeeded` is a crucial function for integrations with external services like Microsoft Graph. Access tokens typically have a short lifespan. This function checks if an existing access token is still valid; if not, it uses a refresh token (stored securely) to obtain a new access token without requiring the user to re-authenticate. This ensures continuous access to the external service.

```typescript
export const dynamic = 'force-dynamic'
```
*   **Purpose:** Configures the Next.js API route to be dynamically rendered on every request.
*   **Explanation:** In Next.js, API routes can be statically optimized by default if they don't use dynamic features. `export const dynamic = 'force-dynamic'` explicitly tells Next.js that this API route should *not* be cached and should execute server-side logic on *every* incoming request. This is important for APIs that need to fetch real-time data or handle user-specific sessions, which change frequently.

```typescript
const logger = createLogger('OneDriveFoldersAPI')
```
*   **Purpose:** Initializes a logger instance for this specific API route.
*   **Explanation:** It calls the `createLogger` function, passing `'OneDriveFoldersAPI'` as the module name. This means any log messages generated by this `logger` instance will be prefixed or tagged with "OneDriveFoldersAPI", making it easier to identify logs related to this specific endpoint in your console or log management system.

```typescript
import type { MicrosoftGraphDriveItem } from '@/tools/onedrive/types'
```
*   **Purpose:** Imports a TypeScript type definition for a Microsoft Graph Drive Item.
*   **Explanation:** This is a type-only import (`type`) which means it's only used by TypeScript for type checking and doesn't generate any JavaScript code at runtime. `MicrosoftGraphDriveItem` likely defines the structure of an object returned by the Microsoft Graph API when querying for drive items (files and folders), including properties like `id`, `name`, `folder`, `webUrl`, etc. This helps ensure type safety when working with the API response.

```typescript
/**
 * Get folders from Microsoft OneDrive
 */
export async function GET(request: NextRequest) {
```
*   **Purpose:** Defines the main asynchronous function for handling HTTP GET requests to this API route.
*   **Explanation:** In Next.js API routes, exported functions named after HTTP methods (`GET`, `POST`, `PUT`, `DELETE`, etc.) automatically handle requests for those methods. `async` indicates that this function will perform asynchronous operations (like fetching data from a database or an external API). It takes a `request` object (of type `NextRequest`) as its argument, which contains all the details about the incoming HTTP request. The JSDoc comment `/** ... */` provides a brief description of the function's purpose.

```typescript
  const requestId = randomUUID().slice(0, 8)
```
*   **Purpose:** Generates a short, unique identifier for the current request.
*   **Explanation:** `randomUUID()` generates a full UUID string (e.g., `a1b2c3d4-e5f6-7890-1234-567890abcdef`). `.slice(0, 8)` then takes only the first 8 characters of this UUID. This creates a compact, unique ID that can be used to correlate logs and errors for a specific request, making debugging easier.

```typescript
  try {
```
*   **Purpose:** Starts a `try` block to handle potential errors gracefully.
*   **Explanation:** Any code within this block that throws an error will be caught by the subsequent `catch` block, preventing the server from crashing and allowing a controlled error response to be sent back to the client.

```typescript
    const session = await getSession()
    if (!session?.user?.id) {
      return NextResponse.json({ error: 'User not authenticated' }, { status: 401 })
    }
```
*   **Purpose:** Authenticates the user.
*   **Explanation:**
    *   `await getSession()`: Calls the previously imported `getSession` function to retrieve the current user's session data. This is an asynchronous operation, so `await` is used.
    *   `if (!session?.user?.id)`: This checks if a session exists and if that session contains a user object with an `id`. The `?.` (optional chaining) safely checks for the existence of `session` and `session.user` before trying to access `id`.
    *   `return NextResponse.json({ error: 'User not authenticated' }, { status: 401 })`: If the user is not authenticated (no session or no user ID), it immediately sends a JSON response with an error message and an HTTP status code of `401 Unauthorized`.

```typescript
    const { searchParams } = new URL(request.url)
    const credentialId = searchParams.get('credentialId')
    const query = searchParams.get('query') || ''
```
*   **Purpose:** Extracts query parameters from the request URL.
*   **Explanation:**
    *   `new URL(request.url)`: Creates a `URL` object from the incoming request's URL. This object provides convenient methods to parse the URL components.
    *   `const { searchParams } = ...`: Destructures the `searchParams` property from the `URL` object. `searchParams` is a `URLSearchParams` object that allows easy access to query parameters.
    *   `const credentialId = searchParams.get('credentialId')`: Retrieves the value of the `credentialId` query parameter (e.g., from `?credentialId=some_id`). This ID identifies which specific OneDrive account connection to use.
    *   `const query = searchParams.get('query') || ''`: Retrieves the value of the `query` query parameter. If no `query` parameter is provided, it defaults to an empty string `''`. This parameter is used for searching within OneDrive folders.

```typescript
    if (!credentialId) {
      return NextResponse.json({ error: 'Credential ID is required' }, { status: 400 })
    }
```
*   **Purpose:** Validates that the `credentialId` is provided.
*   **Explanation:** If the `credentialId` (which is essential for knowing which OneDrive account to access) is missing from the query parameters, the API returns an error message with an HTTP status code of `400 Bad Request`.

```typescript
    const credentials = await db.select().from(account).where(eq(account.id, credentialId)).limit(1)
```
*   **Purpose:** Fetches the specific credential record from the database.
*   **Explanation:**
    *   `db.select().from(account)`: This is a Drizzle ORM query. It selects all columns (`select()`) from the `account` table (`from(account)`).
    *   `.where(eq(account.id, credentialId))`: Filters the results to find the row where the `id` column of the `account` table matches the `credentialId` obtained from the request.
    *   `.limit(1)`: Limits the query to return at most one result, as `id` is typically unique. `await` is used because this is an asynchronous database operation.

```typescript
    if (!credentials.length) {
      return NextResponse.json({ error: 'Credential not found' }, { status: 404 })
    }
```
*   **Purpose:** Checks if the credential was found in the database.
*   **Explanation:** If the `credentials` array returned by the database query is empty (`!credentials.length`), it means no record matched the provided `credentialId`. In this case, an error is returned with an HTTP status code of `404 Not Found`.

```typescript
    const credential = credentials[0]
    if (credential.userId !== session.user.id) {
      return NextResponse.json({ error: 'Unauthorized' }, { status: 403 })
    }
```
*   **Purpose:** Authorizes the user to access this specific credential.
*   **Explanation:**
    *   `const credential = credentials[0]`: Since `limit(1)` was used, the first (and only) element of the `credentials` array is assigned to `credential`.
    *   `if (credential.userId !== session.user.id)`: This is a critical security check. It verifies that the `userId` associated with the fetched `credential` in the database matches the `id` of the currently authenticated `session.user`. This prevents one user from accessing another user's external service credentials.
    *   If they don't match, an `403 Forbidden` error is returned, indicating that the user is authenticated but not authorized to access this specific resource.

```typescript
    const accessToken = await refreshAccessTokenIfNeeded(credentialId, session.user.id, requestId)
    if (!accessToken) {
      return NextResponse.json({ error: 'Failed to obtain valid access token' }, { status: 401 })
    }
```
*   **Purpose:** Ensures a valid access token for Microsoft Graph is available.
*   **Explanation:**
    *   `await refreshAccessTokenIfNeeded(credentialId, session.user.id, requestId)`: Calls the utility function to get a valid access token. This function will internally check if the existing token is expired and refresh it using a refresh token if necessary. It takes the `credentialId`, the authenticated `userId`, and the `requestId` for logging purposes.
    *   `if (!accessToken)`: If the `refreshAccessTokenIfNeeded` function fails to return a valid `accessToken` (e.g., the refresh token itself is invalid or expired), an error is returned with an `401 Unauthorized` status.

```typescript
    // Build URL for OneDrive folders
    let url = `https://graph.microsoft.com/v1.0/me/drive/root/children?$filter=folder ne null&$select=id,name,folder,webUrl,createdDateTime,lastModifiedDateTime&$top=50`
```
*   **Purpose:** Constructs the base URL for the Microsoft Graph API request to fetch folders.
*   **Explanation:**
    *   `https://graph.microsoft.com/v1.0/me/drive/root/children`: This is the base endpoint for listing items in the root folder of the current user's OneDrive (`me/drive/root/children`).
    *   `?$filter=folder ne null`: This query parameter filters the results to include only items where the `folder` property is *not* null. This effectively ensures that only folders (not files) are returned.
    *   `&$select=id,name,folder,webUrl,createdDateTime,lastModifiedDateTime`: This parameter specifies which properties of each drive item should be returned. This is good practice to reduce payload size and only fetch necessary data.
    *   `&$top=50`: Limits the number of results returned to 50.

```typescript
    if (query) {
      url += `&$search="${encodeURIComponent(query)}"`
    }
```
*   **Purpose:** Appends a search query to the URL if provided.
*   **Explanation:**
    *   `if (query)`: Checks if the `query` parameter (obtained from the request's search params) has a value.
    *   `url += ...`: If `query` exists, it appends an `&$search` parameter to the URL.
    *   `"${encodeURIComponent(query)}"`: The actual search term needs to be wrapped in double quotes and URL-encoded to handle special characters correctly in the URL. `encodeURIComponent` converts characters that are not allowed in a URI component (like spaces) into their percentage-encoded equivalents.

```typescript
    const response = await fetch(url, {
      headers: {
        Authorization: `Bearer ${accessToken}`,
      },
    })
```
*   **Purpose:** Sends the request to the Microsoft Graph API.
*   **Explanation:**
    *   `await fetch(url, { ... })`: Uses the browser's native `fetch` API (or Node.js's implementation when running on the server) to send an HTTP GET request to the constructed `url`.
    *   `headers: { Authorization: \`Bearer ${accessToken}\` }`: Sets the `Authorization` header. This is how the Microsoft Graph API authenticates the request. The `Bearer` scheme is used, followed by the `accessToken` obtained earlier.

```typescript
    if (!response.ok) {
      const errorData = await response.json().catch(() => ({ error: { message: 'Unknown error' } }))
      return NextResponse.json(
        { error: errorData.error?.message || 'Failed to fetch folders from OneDrive' },
        { status: response.status }
      )
    }
```
*   **Purpose:** Handles non-successful responses from the Microsoft Graph API.
*   **Explanation:**
    *   `if (!response.ok)`: The `response.ok` property is `true` for HTTP status codes in the 200-299 range, and `false` otherwise. This checks if the API call was *not* successful.
    *   `const errorData = await response.json().catch(() => ({ error: { message: 'Unknown error' } }))`: If the response is not `ok`, it tries to parse the response body as JSON to get detailed error information from Microsoft Graph. The `.catch()` is a safeguard in case the response body isn't valid JSON, providing a fallback "Unknown error" message.
    *   `return NextResponse.json(...)`: Sends an error response back to the client. It tries to use the error message provided by Microsoft Graph (`errorData.error?.message`) or falls back to a generic message (`'Failed to fetch folders from OneDrive'`). The HTTP status code of the original Microsoft Graph response (`response.status`) is forwarded.

```typescript
    const data = await response.json()
    const folders = (data.value || [])
      .filter((item: MicrosoftGraphDriveItem) => item.folder) // Only folders
      .map((folder: MicrosoftGraphDriveItem) => ({
        id: folder.id,
        name: folder.name,
        mimeType: 'application/vnd.microsoft.graph.folder',
        webViewLink: folder.webUrl,
        createdTime: folder.createdDateTime,
        modifiedTime: folder.lastModifiedDateTime,
      }))
```
*   **Purpose:** Processes the successful response from Microsoft Graph, filters, and formats the folder data.
*   **Explanation:**
    *   `const data = await response.json()`: Parses the successful API response body as JSON. Microsoft Graph often returns data in a `value` array.
    *   `(data.value || [])`: Accesses the `value` property from the `data` object. If `data.value` is null or undefined (e.g., if the API returned no results), it defaults to an empty array `[]` to prevent errors in the subsequent `.filter()` and `.map()` calls.
    *   `.filter((item: MicrosoftGraphDriveItem) => item.folder)`: Although the initial URL filter already requested only folders, this acts as a final safeguard. It iterates through the items and keeps only those where the `folder` property exists (indicating it's a folder, not a file). The `item: MicrosoftGraphDriveItem` provides type checking.
    *   `.map((folder: MicrosoftGraphDriveItem) => ({ ... }))`: Transforms each `folder` object into a new, simplified object with specific properties.
        *   `id: folder.id`: The unique ID of the folder.
        *   `name: folder.name`: The name of the folder.
        *   `mimeType: 'application/vnd.microsoft.graph.folder'`: A hardcoded MIME type indicating it's a Microsoft Graph folder. This might be useful for frontend rendering or icon display.
        *   `webViewLink: folder.webUrl`: The URL to view the folder directly in OneDrive.
        *   `createdTime: folder.createdDateTime`: The creation timestamp of the folder.
        *   `modifiedTime: folder.lastModifiedDateTime`: The last modification timestamp of the folder.
    *   The result of this `map` operation is an array of simplified folder objects.

```typescript
    return NextResponse.json({ files: folders }, { status: 200 })
```
*   **Purpose:** Sends the processed folder data back to the client.
*   **Explanation:** If everything was successful, it returns a JSON response containing an object with a `files` property (which holds the `folders` array) and an HTTP status code of `200 OK`. The property is named `files` even though it contains folders, likely for consistency with other file listing APIs.

```typescript
  } catch (error) {
    logger.error(`[${requestId}] Error fetching folders from OneDrive`, error)
    return NextResponse.json({ error: 'Internal server error' }, { status: 500 })
  }
}
```
*   **Purpose:** Catches any unexpected errors that occurred during the `try` block.
*   **Explanation:**
    *   `catch (error)`: This block executes if any unhandled error is thrown within the `try` block. The `error` variable contains the thrown exception.
    *   `logger.error(...)`: Logs the error using the `logger` instance. It includes the `requestId` for context and passes the `error` object itself, which might contain a stack trace or other useful debugging information.
    *   `return NextResponse.json({ error: 'Internal server error' }, { status: 500 })`: Sends a generic "Internal server error" message with an HTTP status code of `500` back to the client. This is good practice to avoid exposing sensitive internal error details to the public.