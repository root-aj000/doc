This file defines a Next.js API route that acts as a backend endpoint for fetching data (specifically "contacts" for now) from the Wealthbox CRM platform. It handles user authentication, authorization, secure access token management (OAuth), and transforms the external API's response into a standardized format before sending it back to the client.

In essence, it's a secure gateway that allows your application to retrieve contact information from Wealthbox on behalf of an authenticated user.

Here's a detailed breakdown:

---

### **Imports (Setting up our Tools)**

This section brings in all the necessary modules and types from various parts of the application and external libraries. Think of it as gathering all the ingredients and tools before starting to cook.

*   `import { db } from '@sim/db'`
    *   **Purpose:** Imports the initialized Drizzle ORM database client. Drizzle is used here to interact with your application's database (e.g., PostgreSQL, MySQL) to store and retrieve data.
    *   **Explanation:** `db` is the instance that allows you to run database queries like `SELECT`, `INSERT`, `UPDATE`, etc.
*   `import { account } from '@sim/db/schema'`
    *   **Purpose:** Imports the `account` schema definition from your database schema.
    *   **Explanation:** This `account` object represents the table in your database where user credentials (like OAuth tokens for Wealthbox) are stored. It defines the structure and columns of that table.
*   `import { eq } from 'drizzle-orm'`
    *   **Purpose:** Imports the `eq` (equals) function from Drizzle ORM.
    *   **Explanation:** `eq` is a helper function used in Drizzle queries to specify a "where" condition, like `WHERE account.id = credentialId`.
*   `import { type NextRequest, NextResponse } from 'next/server'`
    *   **Purpose:** Imports types and classes specific to Next.js API routes.
    *   **Explanation:**
        *   `NextRequest`: This is a TypeScript type that represents an incoming HTTP request in a Next.js API route. It extends the standard Web API `Request` object with additional Next.js specific features.
        *   `NextResponse`: This is a class used to create and send HTTP responses from a Next.js API route. It's similar to the standard Web API `Response` object but often preferred in Next.js for its convenience features.
*   `import { getSession } from '@/lib/auth'`
    *   **Purpose:** Imports a utility function to retrieve the current user's session information.
    *   **Explanation:** `getSession()` is crucial for authentication. It checks if a user is logged in and, if so, provides details about that user, like their ID.
*   `import { createLogger } from '@/lib/logs/console/logger'`
    *   **Purpose:** Imports a function to create a logger instance for structured logging.
    *   **Explanation:** This allows the API route to record informational messages, warnings, and errors to the console or other logging destinations, which is vital for debugging and monitoring.
*   `import { generateRequestId } from '@/lib/utils'`
    *   **Purpose:** Imports a utility function to generate a unique request ID.
    *   **Explanation:** `generateRequestId()` creates a unique identifier for each incoming request. This ID is used throughout the request's lifecycle in logs, making it easier to trace specific requests and their related events, especially in a distributed system.
*   `import { refreshAccessTokenIfNeeded } from '@/app/api/auth/oauth/utils'`
    *   **Purpose:** Imports a function to manage OAuth access tokens.
    *   **Explanation:** This function is critical for integrating with third-party APIs like Wealthbox. OAuth access tokens can expire, so `refreshAccessTokenIfNeeded` ensures that we always have a valid, non-expired token by automatically refreshing it if necessary, using a refresh token.

---

### **Configuration and Initialization**

*   `export const dynamic = 'force-dynamic'`
    *   **Purpose:** This Next.js configuration tells the server to treat this route as "dynamic," meaning it should not be cached statically and should always execute on every request.
    *   **Explanation:** By default, Next.js can sometimes statically optimize routes. `force-dynamic` ensures that this API route always runs server-side logic and fetches fresh data on each call, which is essential for an API that depends on user sessions and external API calls.
*   `const logger = createLogger('WealthboxItemsAPI')`
    *   **Purpose:** Initializes a logger specifically for this API route.
    *   **Explanation:** Using `createLogger('WealthboxItemsAPI')` creates a logger instance tagged with "WealthboxItemsAPI". This makes it easy to identify log messages originating from this specific part of the application.

---

### **`WealthboxItem` Interface (Defining Our Output Structure)**

*   `interface WealthboxItem { ... }`
    *   **Purpose:** Defines a TypeScript interface for the standardized structure of an item (like a contact) that this API will return.
    *   **Explanation:** An interface in TypeScript is like a blueprint or a contract. It specifies the names and types of properties that an object *must* have. This `WealthboxItem` interface ensures that all items returned by this API will consistently have these properties:
        *   `id: string`: A unique identifier for the item.
        *   `name: string`: The primary name of the item (e.g., contact's full name).
        *   `type: string`: The type of item (e.g., 'contact', 'note', 'task').
        *   `content: string`: The main descriptive content or details of the item.
        *   `createdAt: string`: The timestamp when the item was created.
        *   `updatedAt: string`: The timestamp when the item was last updated.
    *   **Simplification:** It ensures that no matter how Wealthbox sends us data, our application always receives it in a predictable, easy-to-use format.

---

### **`GET` Function (The Core Logic)**

This is the main function that handles incoming HTTP `GET` requests to this API route.

*   `export async function GET(request: NextRequest) { ... }`
    *   **Purpose:** Defines the asynchronous function that responds to `GET` requests.
    *   **Explanation:**
        *   `export`: Makes this function accessible from outside the file, allowing Next.js to find and execute it.
        *   `async`: Indicates that this function will perform asynchronous operations (like database calls, external API fetches) using `await`.
        *   `GET`: By convention in Next.js, a function named `GET` handles `GET` HTTP requests.
        *   `request: NextRequest`: The function receives the incoming request object, which contains information about the request (URL, headers, query parameters, etc.).

*   `const requestId = generateRequestId()`
    *   **Purpose:** Generates a unique ID for the current request.
    *   **Explanation:** As explained above, this `requestId` is used for consistent logging and debugging throughout the execution of this `GET` request.

*   `try { ... } catch (error) { ... }`
    *   **Purpose:** Implements robust error handling for the entire API route logic.
    *   **Explanation:** Any code inside the `try` block that throws an error will be caught by the `catch` block. This prevents the server from crashing and allows us to send a meaningful error response to the client.

#### **Inside the `try` block:**

1.  **Authentication Check:**
    *   `const session = await getSession()`
        *   **Explanation:** Calls the `getSession()` utility to retrieve the current user's session data. This is an asynchronous operation.
    *   `if (!session?.user?.id) { ... }`
        *   **Explanation:** Checks if a session exists and if the session contains a user ID. If not (meaning the user is not authenticated), it logs a warning and returns an HTTP 401 (Unauthorized) response.
        *   `logger.warn(...)`: Records a warning message.
        *   `return NextResponse.json({ error: 'User not authenticated' }, { status: 401 })`: Sends a JSON response with an error message and a 401 status code.

2.  **Extracting and Validating Query Parameters:**
    *   `const { searchParams } = new URL(request.url)`
        *   **Explanation:** Extracts the `searchParams` object from the request's URL. This object makes it easy to access query parameters (e.g., `?credentialId=xyz&type=contact`).
    *   `const credentialId = searchParams.get('credentialId')`
        *   **Explanation:** Retrieves the value of the `credentialId` query parameter. This ID tells us which Wealthbox account credentials to use.
    *   `const type = searchParams.get('type') || 'contact'`
        *   **Explanation:** Retrieves the `type` query parameter, defaulting to `'contact'` if not provided. This specifies what kind of Wealthbox item we want to fetch.
    *   `const query = searchParams.get('query') || ''`
        *   **Explanation:** Retrieves the `query` parameter for search functionality, defaulting to an empty string if not provided.
    *   `if (!credentialId) { ... }`
        *   **Explanation:** Validates that `credentialId` was provided. If missing, logs a warning and returns a 400 (Bad Request) response.
    *   `if (type !== 'contact') { ... }`
        *   **Explanation:** Currently, this API only supports fetching `contact` types. If an unsupported type is requested, it logs a warning and returns a 400 (Bad Request) response. This sets the stage for potential future expansion to other types like 'notes' or 'tasks'.

3.  **Fetching Credential from the Database:**
    *   `const credentials = await db.select().from(account).where(eq(account.id, credentialId)).limit(1)`
        *   **Explanation:** This is a Drizzle ORM query:
            *   `db.select()`: Starts a select query.
            *   `.from(account)`: Specifies the `account` table.
            *   `.where(eq(account.id, credentialId))`: Filters records where the `id` column in the `account` table matches the `credentialId` from the request.
            *   `.limit(1)`: Ensures only one record is fetched, as `credentialId` should be unique.
    *   `if (!credentials.length) { ... }`
        *   **Explanation:** Checks if any credential was found. If not, logs a warning and returns a 404 (Not Found) response.
    *   `const credential = credentials[0]`
        *   **Explanation:** If a credential was found, it extracts the first (and only) credential object from the `credentials` array.

4.  **Authorization Check (Credential Ownership):**
    *   `if (credential.userId !== session.user.id) { ... }`
        *   **Explanation:** Verifies that the `userId` associated with the fetched `credential` matches the `id` of the currently authenticated `session.user`. This is a critical security step to ensure a user can only access *their own* linked Wealthbox accounts, not someone else's. If unauthorized, it logs a warning and returns a 403 (Forbidden) response.

5.  **OAuth Access Token Management:**
    *   `const accessToken = await refreshAccessTokenIfNeeded(credentialId, session.user.id, requestId)`
        *   **Explanation:** Calls the utility function to get a valid Wealthbox access token. This function internally checks if the existing token is expired and, if so, uses a refresh token to obtain a new one. It also takes the `requestId` for logging purposes within the utility.
    *   `if (!accessToken) { ... }`
        *   **Explanation:** If `refreshAccessTokenIfNeeded` fails to provide a valid `accessToken` (e.g., due to an expired refresh token or other OAuth issues), it logs an error and returns a 401 (Unauthorized) response, as we cannot proceed without a valid token.

6.  **Constructing and Making the Wealthbox API Request:**
    *   `const endpoints = { contact: 'contacts' }`
        *   **Explanation:** Defines a mapping object. If `type` is 'contact', the corresponding Wealthbox API path is 'contacts'. This makes the endpoint selection dynamic based on the `type` parameter.
    *   `const endpoint = endpoints[type as keyof typeof endpoints]`
        *   **Explanation:** Selects the appropriate API path ('contacts' in this case) based on the `type` requested.
    *   `const url = new URL(`https://api.crmworkspace.com/v1/${endpoint}`) `
        *   **Explanation:** Constructs the full URL for the Wealthbox API endpoint.
    *   `logger.info(...)`: Logs information about the upcoming API call, including the endpoint and URL.
    *   `const response = await fetch(url.toString(), { ... })`
        *   **Explanation:** Performs the actual HTTP GET request to the Wealthbox API using the standard `fetch` API:
            *   `url.toString()`: Converts the `URL` object back to a string.
            *   `headers`: Sets the necessary HTTP headers:
                *   `Authorization: `Bearer ${accessToken}``: Includes the obtained access token for authentication with the Wealthbox API. This is how Wealthbox knows who is making the request.
                *   `'Content-Type': 'application/json'`: Indicates that the request body (though empty for GET) expects JSON.

7.  **Handling Wealthbox API Response:**
    *   `if (!response.ok) { ... }`
        *   **Explanation:** Checks if the HTTP response status from Wealthbox is `not ok` (i.e., not in the 200-299 range, indicating an error on their side).
        *   `const errorText = await response.text()`: Reads the error message body from the Wealthbox response.
        *   `logger.error(...)`: Logs the detailed error from the Wealthbox API.
        *   `return NextResponse.json(...)`: Returns a JSON response to the client indicating the failure, along with the original HTTP status code from Wealthbox.
    *   `const data = await response.json()`
        *   **Explanation:** If the response was `ok`, it parses the JSON body of the Wealthbox API response into a JavaScript object.
    *   `logger.info(...)`: Logs details about the raw data received from the Wealthbox API, which is helpful for understanding the structure of their response.

8.  **Transforming and Filtering Data:**
    *   `let items: WealthboxItem[] = []`
        *   **Explanation:** Initializes an empty array that will hold the transformed `WealthboxItem` objects.
    *   `if (type === 'contact') { ... }`
        *   **Explanation:** This block specifically handles transforming 'contact' data.
        *   `const contacts = data.contacts || []`: Safely retrieves the `contacts` array from the Wealthbox response. If `data.contacts` is undefined, it defaults to an empty array.
        *   `if (!Array.isArray(contacts)) { ... }`: A defensive check to ensure that `contacts` is indeed an array. If not, logs a warning and returns an empty items array, preventing further errors.
        *   `items = contacts.map((item: any) => ({ ... }))`
            *   **Explanation:** This is the core transformation logic. It iterates over each `item` (raw contact) in the `contacts` array and maps it to the `WealthboxItem` interface:
                *   `id: item.id?.toString() || ''`: Transforms the contact's ID to a string, defaulting to empty if missing.
                *   `name: `${item.first_name || ''} ${item.last_name || ''}`.trim() || `Contact ${item.id}``: Concatenates first and last names. If both are empty, it defaults to "Contact [ID]".
                *   `type: 'contact'`: Sets the type as 'contact'.
                *   `content: item.background_information || ''`: Uses the `background_information` as the main content, defaulting to empty.
                *   `createdAt: item.created_at`: Direct mapping of creation timestamp.
                *   `updatedAt: item.updated_at`: Direct mapping of update timestamp.
    *   `if (query.trim()) { ... }`
        *   **Explanation:** If a search `query` was provided, this block filters the transformed `items`.
        *   `const searchTerm = query.trim().toLowerCase()`: Converts the search query to lowercase for case-insensitive matching.
        *   `items = items.filter(...)`: Filters the `items` array, keeping only those where the `name` or `content` (converted to lowercase) includes the `searchTerm`.

9.  **Sending the Final Response:**
    *   `logger.info(...)`: Logs the success of fetching and transforming items, including the count.
    *   `return NextResponse.json({ items }, { status: 200 })`
        *   **Explanation:** Sends the final JSON response containing the array of transformed and filtered `items` with an HTTP 200 (OK) status code.

#### **Inside the `catch` block:**

*   `catch (error) { ... }`
    *   **Explanation:** This block executes if any unhandled error occurs within the `try` block.
    *   `logger.error(...)`: Logs the error, including the `requestId` for traceability.
    *   `return NextResponse.json({ error: 'Internal server error' }, { status: 500 })`
        *   **Explanation:** Returns a generic "Internal server error" message with an HTTP 500 status code to the client. This prevents leaking sensitive error details.