This TypeScript file defines an API endpoint for a Next.js application. Its primary purpose is to allow authenticated users to retrieve detailed information about a specific file stored in their Microsoft OneDrive, using the Microsoft Graph API.

Here's a detailed breakdown:

---

### **Overall Purpose of this File and What it Does**

This file defines a **Next.js API route handler** (`/api/microsoft/file`). Specifically, it implements the `GET` method for this route. When a client makes a `GET` request to this endpoint with a `credentialId` (representing a user's linked Microsoft account) and a `fileId` (the ID of the file in OneDrive), the server performs the following actions:

1.  **Authenticates the User:** Ensures a valid user session exists.
2.  **Validates Input:** Checks if the provided `credentialId` and `fileId` are present and valid.
3.  **Authorizes Access:** Verifies that the requested `credentialId` belongs to the authenticated user.
4.  **Manages OAuth Tokens:** Refreshes the Microsoft access token if it's expired or close to expiring, ensuring a valid token for API calls.
5.  **Fetches File Data:** Makes a request to the Microsoft Graph API (`graph.microsoft.com`) to retrieve metadata about the specified file from the user's OneDrive.
6.  **Transforms Data:** Processes the raw response from Microsoft Graph into a more standardized and client-friendly format.
7.  **Returns File Information:** Sends the transformed file details back to the client as a JSON response.

In essence, it acts as a secure, authenticated proxy between your application's frontend and the Microsoft Graph API, abstracting away the complexities of OAuth, data storage, and API integration.

---

### **Detailed Explanation Line by Line**

#### **Imports**

```typescript
import { db } from '@sim/db'
import { account } from '@sim/db/schema'
import { eq } from 'drizzle-orm'
import { type NextRequest, NextResponse } from 'next/server'
import { getSession } from '@/lib/auth'
import { createLogger } from '@/lib/logs/console/logger'
import { validateMicrosoftGraphId } from '@/lib/security/input-validation'
import { generateRequestId } from '@/lib/utils'
import { refreshAccessTokenIfNeeded } from '@/app/api/auth/oauth/utils'
```

*   **`import { db } from '@sim/db'`**: Imports the database client instance. This `db` object is used to interact with your application's database (e.g., PostgreSQL, MySQL) using Drizzle ORM.
*   **`import { account } from '@sim/db/schema'`**: Imports the `account` schema definition from your database schema. This `account` object likely represents a table in your database that stores information about connected user accounts, including OAuth tokens for services like Microsoft.
*   **`import { eq } from 'drizzle-orm'`**: Imports the `eq` (equals) function from Drizzle ORM. This is a utility function used to construct `WHERE` clauses in database queries, specifically to check for equality.
*   **`import { type NextRequest, NextResponse } from 'next/server'`**: Imports types and classes specific to Next.js API routes.
    *   `NextRequest`: A type representing the incoming HTTP request object, extending standard `Request` with Next.js-specific features like `nextUrl`.
    *   `NextResponse`: A class used to construct and send HTTP responses from API routes.
*   **`import { getSession } from '@/lib/auth'`**: Imports a utility function `getSession` from your application's authentication library. This function is responsible for retrieving the current user's session data, which typically includes user ID and authentication status.
*   **`import { createLogger } from '@/lib/logs/console/logger'`**: Imports a function to create a logger instance. This is used for structured logging within the application, helping with debugging and monitoring.
*   **`import { validateMicrosoftGraphId } from '@/lib/security/input-validation'`**: Imports a custom validation function. This function is likely used to ensure that IDs received from Microsoft Graph (like `fileId`) conform to expected formats, preventing security vulnerabilities like injection or malformed requests.
*   **`import { generateRequestId } from '@/lib/utils'`**: Imports a utility function to generate a unique request ID. This ID is useful for tracing requests through logs and correlating different log entries related to the same operation.
*   **`import { refreshAccessTokenIfNeeded } from '@/app/api/auth/oauth/utils'`**: Imports a utility function crucial for OAuth token management. This function intelligently checks if an access token is valid and refreshes it using a refresh token if necessary, ensuring continuous access to external APIs without requiring user re-authentication.

#### **Configuration and Logger Initialization**

```typescript
export const dynamic = 'force-dynamic'

const logger = createLogger('MicrosoftFileAPI')
```

*   **`export const dynamic = 'force-dynamic'`**: This is a Next.js-specific export. It configures the behavior of this API route in serverless environments. `force-dynamic` ensures that this route is always executed dynamically at runtime (server-side) for every request, rather than being statically optimized or cached. This is critical for routes that handle user-specific data or require up-to-the-minute information.
*   **`const logger = createLogger('MicrosoftFileAPI')`**: Initializes a logger instance specifically for this API handler, named `MicrosoftFileAPI`. This allows for clear categorization of log messages originating from this file.

#### **`GET` Request Handler**

```typescript
export async function GET(request: NextRequest) {
  const requestId = generateRequestId()
  try {
    const session = await getSession()

    if (!session?.user?.id) {
      return NextResponse.json({ error: 'User not authenticated' }, { status: 401 })
    }

    const { searchParams } = new URL(request.url)
    const credentialId = searchParams.get('credentialId')
    const fileId = searchParams.get('fileId')

    if (!credentialId || !fileId) {
      return NextResponse.json({ error: 'Credential ID and File ID are required' }, { status: 400 })
    }

    const fileIdValidation = validateMicrosoftGraphId(fileId, 'fileId')
    if (!fileIdValidation.isValid) {
      logger.warn(`[${requestId}] Invalid file ID: ${fileIdValidation.error}`)
      return NextResponse.json({ error: fileIdValidation.error }, { status: 400 })
    }

    const credentials = await db.select().from(account).where(eq(account.id, credentialId)).limit(1)

    if (!credentials.length) {
      return NextResponse.json({ error: 'Credential not found' }, { status: 404 })
    }

    const credential = credentials[0]

    if (credential.userId !== session.user.id) {
      return NextResponse.json({ error: 'Unauthorized' }, { status: 403 })
    }

    const accessToken = await refreshAccessTokenIfNeeded(credentialId, session.user.id, requestId)

    if (!accessToken) {
      return NextResponse.json({ error: 'Failed to obtain valid access token' }, { status: 401 })
    }

    const response = await fetch(
      `https://graph.microsoft.com/v1.0/me/drive/items/${fileId}?$select=id,name,mimeType,webUrl,thumbnails,createdDateTime,lastModifiedDateTime,size,createdBy`,
      {
        headers: {
          Authorization: `Bearer ${accessToken}`,
        },
      }
    )

    if (!response.ok) {
      const errorData = await response.json().catch(() => ({ error: { message: 'Unknown error' } }))
      logger.error(`[${requestId}] Microsoft Graph API error`, {
        status: response.status,
        error: errorData.error?.message || 'Failed to fetch file from Microsoft OneDrive',
      })
      return NextResponse.json(
        {
          error: errorData.error?.message || 'Failed to fetch file from Microsoft OneDrive',
        },
        { status: response.status }
      )
    }

    const file = await response.json()

    const transformedFile = {
      id: file.id,
      name: file.name,
      mimeType:
        file.mimeType || 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet',
      iconLink: file.thumbnails?.[0]?.small?.url,
      webViewLink: file.webUrl,
      thumbnailLink: file.thumbnails?.[0]?.medium?.url,
      createdTime: file.createdDateTime,
      modifiedTime: file.lastModifiedDateTime,
      size: file.size?.toString(),
      owners: file.createdBy
        ? [
            {
              displayName: file.createdBy.user?.displayName || 'Unknown',
              emailAddress: file.createdBy.user?.email || '',
            },
          ]
        : [],
      downloadUrl: `https://graph.microsoft.com/v1.0/me/drive/items/${file.id}/content`,
    }

    return NextResponse.json({ file: transformedFile }, { status: 200 })
  } catch (error) {
    logger.error(`[${requestId}] Error fetching file from Microsoft OneDrive`, error)
    return NextResponse.json({ error: 'Internal server error' }, { status: 500 })
  }
}
```

*   **`export async function GET(request: NextRequest)`**: This defines the asynchronous function that handles `GET` requests to this API route. It takes `NextRequest` as its argument, which contains information about the incoming request.
*   **`const requestId = generateRequestId()`**: Generates a unique ID for the current request. This ID is logged with all related messages, making it easy to trace the flow of a single request through the server logs.
*   **`try { ... } catch (error) { ... }`**: This `try-catch` block is a crucial error handling mechanism. All the core logic is wrapped in `try` to catch any unexpected errors that might occur during execution. If an error is caught, the `catch` block will log it and return a generic 500 Internal Server Error response.

    *   **`const session = await getSession()`**: Calls the `getSession` function to retrieve the current user's authentication session. This function is typically asynchronous as it might involve database lookups or decryption of session cookies.
    *   **`if (!session?.user?.id)`**: Checks if a valid user session exists and if the user ID is available.
        *   **`return NextResponse.json({ error: 'User not authenticated' }, { status: 401 })`**: If no valid session or user ID is found, it immediately returns a JSON response with an "User not authenticated" error and an HTTP status code of `401 Unauthorized`.
    *   **`const { searchParams } = new URL(request.url)`**: Creates a `URL` object from the incoming `request.url` and destructures `searchParams` from it. `searchParams` allows easy access to query parameters (e.g., `?param=value`).
    *   **`const credentialId = searchParams.get('credentialId')`**: Extracts the value of the `credentialId` query parameter (e.g., from `.../api/microsoft/file?credentialId=xyz`). This ID uniquely identifies the user's linked Microsoft account in your database.
    *   **`const fileId = searchParams.get('fileId')`**: Extracts the value of the `fileId` query parameter. This is the ID of the specific file in Microsoft OneDrive that the user wants to fetch information about.
    *   **`if (!credentialId || !fileId)`**: Checks if both `credentialId` and `fileId` were provided in the request query.
        *   **`return NextResponse.json({ error: 'Credential ID and File ID are required' }, { status: 400 })`**: If either is missing, it returns a `400 Bad Request` error, indicating that the client's request was malformed.
    *   **`const fileIdValidation = validateMicrosoftGraphId(fileId, 'fileId')`**: Calls a security utility to validate the format of the `fileId`. This is important to prevent sending malformed IDs to the Microsoft API or potential injection attacks.
    *   **`if (!fileIdValidation.isValid)`**: Checks the result of the validation.
        *   **`logger.warn(...)`**: If the `fileId` is invalid, a warning is logged with the request ID and the validation error message.
        *   **`return NextResponse.json({ error: fileIdValidation.error }, { status: 400 })`**: Returns a `400 Bad Request` error with the specific validation message to the client.
    *   **`const credentials = await db.select().from(account).where(eq(account.id, credentialId)).limit(1)`**: This is a Drizzle ORM database query.
        *   `db.select()`: Starts a select query.
        *   `from(account)`: Specifies the `account` table as the source.
        *   `where(eq(account.id, credentialId))`: Filters the results to find an account where the `id` column matches the `credentialId` provided in the request. `eq` is the Drizzle helper for equality.
        *   `limit(1)`: Limits the result to a single record, as `credentialId` is expected to be unique.
    *   **`if (!credentials.length)`**: Checks if any credentials were found in the database for the given `credentialId`.
        *   **`return NextResponse.json({ error: 'Credential not found' }, { status: 404 })`**: If no matching credential is found, it returns a `404 Not Found` error.
    *   **`const credential = credentials[0]`**: If credentials were found, it takes the first (and only) credential object from the array.
    *   **`if (credential.userId !== session.user.id)`**: This is a crucial authorization check. It verifies that the `userId` associated with the fetched `credential` in the database matches the `id` of the currently authenticated `session.user`.
        *   **`return NextResponse.json({ error: 'Unauthorized' }, { status: 403 })`**: If the `credentialId` does not belong to the authenticated user, it returns a `403 Forbidden` error, preventing one user from accessing another user's linked accounts.
    *   **`const accessToken = await refreshAccessTokenIfNeeded(credentialId, session.user.id, requestId)`**: Calls the utility function to get a valid Microsoft access token. This function internally handles checking the token's expiry, using a refresh token if needed, and storing the updated tokens in the database.
        *   It passes the `credentialId`, the current `session.user.id` (for authorization within the utility), and the `requestId` (for logging).
    *   **`if (!accessToken)`**: Checks if `refreshAccessTokenIfNeeded` successfully returned an access token.
        *   **`return NextResponse.json({ error: 'Failed to obtain valid access token' }, { status: 401 })`**: If no valid access token could be obtained (e.g., refresh token revoked or expired), it returns a `401 Unauthorized` error.
    *   **`const response = await fetch(...)`**: This is where the actual request to the Microsoft Graph API is made.
        *   **Template Literal URL**: `https://graph.microsoft.com/v1.0/me/drive/items/${fileId}?$select=id,name,mimeType,webUrl,thumbnails,createdDateTime,lastModifiedDateTime,size,createdBy`
            *   It targets the Microsoft Graph API's `/me/drive/items/{fileId}` endpoint to get details about a specific file.
            *   `?$select=...`: This is a query parameter that explicitly tells the Microsoft Graph API to return only a specific set of fields (id, name, mimeType, etc.) to optimize the response size and prevent over-fetching.
        *   **`headers: { Authorization: `Bearer ${accessToken}` }`**: Sets the `Authorization` header with the obtained access token. This is how the Microsoft Graph API authenticates the request as belonging to the user whose `accessToken` is being used.
    *   **`if (!response.ok)`**: Checks if the `fetch` request to Microsoft Graph was successful (i.e., if the HTTP status code was in the 200-299 range).
        *   **`const errorData = await response.json().catch(() => ({ error: { message: 'Unknown error' } }))`**: If the response was not `ok`, it attempts to parse the response body as JSON to get detailed error information from Microsoft. The `.catch()` handles cases where the error response itself might not be valid JSON.
        *   **`logger.error(...)`**: Logs the error details, including the HTTP status from Microsoft Graph and the specific error message, for debugging purposes.
        *   **`return NextResponse.json(...)`**: Returns a JSON response to the client with the error message received from Microsoft Graph (or a generic one if parsing failed) and the same HTTP status code received from Microsoft Graph.
    *   **`const file = await response.json()`**: If the Microsoft Graph API request was successful, it parses the response body into a JavaScript object, which contains the raw file metadata.
    *   **`const transformedFile = { ... }`**: This block maps the raw data received from Microsoft Graph (`file`) into a more consistent and potentially simplified `transformedFile` object. This is good practice for:
        *   **Consistency**: Ensures the client always receives data in a predictable format, regardless of minor changes in the external API.
        *   **Simplicity**: Provides only the necessary fields, reducing payload size.
        *   **Client-friendliness**: Renames fields (e.g., `webUrl` to `webViewLink`) to fit internal conventions.
        *   **Default Values**: Provides fallback values (e.g., `mimeType`) if the original API response is missing data.
        *   **Computed Properties**: Constructs `downloadUrl` based on the file ID.
    *   **`return NextResponse.json({ file: transformedFile }, { status: 200 })`**: If everything succeeds, it returns a `200 OK` JSON response containing the `transformedFile` object.
*   **`catch (error)`**: This is the global error handler for the entire `GET` function.
    *   **`logger.error(...)`**: Logs any unexpected errors that occurred within the `try` block, including the `requestId` for tracing.
    *   **`return NextResponse.json({ error: 'Internal server error' }, { status: 500 })`**: Returns a generic `500 Internal Server Error` response to the client, preventing sensitive error details from being exposed.

---