This TypeScript file defines a Next.js API route handler for creating a Stripe Billing Portal session. Its primary purpose is to allow authenticated users to manage their billing information (subscriptions, payment methods, invoices) directly through Stripe's hosted portal, either for their individual account or for an organization they are associated with.

---

### Purpose of this file and what it does

This file essentially acts as a secure gateway between your application and Stripe's customer billing portal. When a user in your application clicks a button like "Manage Billing" or "Update Subscription," a request is sent to this API route.

Here's a step-by-step breakdown of what it does:

1.  **Authenticates the User**: It first verifies if the request is coming from a logged-in user.
2.  **Determines Billing Context**: It checks if the user wants to manage billing for their personal account (`user` context) or for an organization (`organization` context).
3.  **Retrieves Stripe Customer ID**: Based on the context, it queries your application's database to find the corresponding Stripe Customer ID. This ID is how Stripe identifies the customer or organization in their system.
4.  **Creates Stripe Billing Portal Session**: Using the retrieved Stripe Customer ID, it makes a request to the Stripe API to create a unique, temporary URL for the billing portal session.
5.  **Returns Portal URL**: It sends this Stripe Billing Portal URL back to the client application, which can then redirect the user to Stripe's hosted portal.
6.  **Handles Errors**: It includes robust error handling for unauthorized access, missing data, customer not found, and general failures during the process.

In essence, it's a backend endpoint that securely fetches the necessary information and orchestrates the creation of a personalized billing management experience powered by Stripe.

---

### Simplify complex logic

The most "complex" part of this file is how it decides *whose* Stripe customer ID to fetch:

*   **Individual vs. Organization Billing**: Many applications have both personal plans (e.g., a "Pro" plan for a single user) and team/organization plans (e.g., a "Business" plan for a company). This route needs to support both scenarios.
*   **How it's handled**: The `context` variable (either `'user'` or `'organization'`) dictates this.
    *   If `context` is `'organization'`, it means the request is to manage billing for a team. The code then looks up the `stripeCustomerId` from the `subscriptionTable` using the provided `organizationId`. This assumes your subscriptions are tied to organizations in your database.
    *   If `context` is `'user'` (or not explicitly `'organization'`), it defaults to managing personal billing. In this case, it fetches the `stripeCustomerId` directly from the `user` table using the currently logged-in user's ID.

This conditional logic ensures that the correct Stripe customer is identified, whether the user is managing their own personal subscription or a subscription belonging to a team they are part of.

---

### Explain each line of code deep

Let's break down the code line by line, explaining its role, relevant types, and dependencies.

```typescript
import { db } from '@sim/db'
```
*   **Purpose**: Imports the database client instance.
*   **Explanation**: `db` is likely an instance of a database client (e.g., Drizzle ORM, Prisma, etc.) configured in your application. It provides methods to interact with your database. The `@sim/db` path suggests it's an internal module or a monorepo package.

```typescript
import { subscription as subscriptionTable, user } from '@sim/db/schema'
```
*   **Purpose**: Imports database schema definitions.
*   **Explanation**: This line imports the schema definitions for two database tables: `subscription` and `user`.
    *   `subscription as subscriptionTable`: The `subscription` schema is imported and aliased as `subscriptionTable` for clearer usage in queries, avoiding naming conflicts if `subscription` was used elsewhere. This schema likely defines the structure of your subscriptions table, including columns like `stripeCustomerId`, `referenceId`, `status`, etc.
    *   `user`: This imports the schema for your `user` table, which typically stores user details including their `id` and `stripeCustomerId`.
    *   `@sim/db/schema`: Indicates these are internal schema definitions for your application's database.

```typescript
import { and, eq } from 'drizzle-orm'
```
*   **Purpose**: Imports Drizzle ORM utility functions for query building.
*   **Explanation**:
    *   `and`: A Drizzle ORM helper function used to combine multiple conditions in a `WHERE` clause with a logical AND operator. For example, `and(condition1, condition2)` means `condition1 AND condition2`.
    *   `eq`: A Drizzle ORM helper function used to create an equality condition (e.g., `column = value`). For example, `eq(user.id, userId)` translates to `user.id = userId` in SQL.

```typescript
import { type NextRequest, NextResponse } from 'next/server'
```
*   **Purpose**: Imports types and classes specific to Next.js API routes.
*   **Explanation**:
    *   `type NextRequest`: This is a TypeScript type definition provided by Next.js for the incoming HTTP request object in API routes. It extends the standard Web `Request` object with Next.js-specific features.
    *   `NextResponse`: This is a class provided by Next.js for constructing HTTP responses in API routes. It's used to send back JSON data, set status codes, and more.

```typescript
import { getSession } from '@/lib/auth'
```
*   **Purpose**: Imports a utility function to retrieve the user's session.
*   **Explanation**: `getSession` is a function from your application's authentication library (`@/lib/auth`). It's responsible for checking if a user is currently logged in and, if so, returning their session data (e.g., user ID, name, email).

```typescript
import { requireStripeClient } from '@/lib/billing/stripe-client'
```
*   **Purpose**: Imports a utility function to get an initialized Stripe client.
*   **Explanation**: `requireStripeClient` is a function that presumably initializes and returns a Stripe API client instance. This client is then used to interact with the Stripe API (e.g., creating billing portal sessions). It's likely configured with your Stripe secret key.

```typescript
import { createLogger } from '@/lib/logs/console/logger'
```
*   **Purpose**: Imports a utility function to create a logger instance.
*   **Explanation**: `createLogger` is a function from your application's logging library. It's used to create a logger instance (`logger` in this file) that can output messages (info, error, etc.) to the console or other configured logging destinations.

```typescript
import { getBaseUrl } from '@/lib/urls/utils'
```
*   **Purpose**: Imports a utility function to get the base URL of the application.
*   **Explanation**: `getBaseUrl` is a function that returns the root URL of your application (e.g., `https://www.yourdomain.com`). This is useful for constructing absolute URLs, especially for `returnUrl` parameters in external services like Stripe.

---

```typescript
const logger = createLogger('BillingPortal')
```
*   **Purpose**: Initializes a logger instance for this module.
*   **Explanation**: This line calls the `createLogger` function, passing `'BillingPortal'` as an identifier. This creates a specific logger instance named 'BillingPortal', making it easier to track logs originating from this particular API route handler.

---

```typescript
export async function POST(request: NextRequest) {
```
*   **Purpose**: Defines the main API route handler for POST requests.
*   **Explanation**: This is the core of the Next.js API route. `export async function POST(request: NextRequest)` declares an asynchronous function named `POST`. Next.js automatically calls this function when an HTTP POST request is made to the path where this file is located (e.g., if the file is `pages/api/billing/portal.ts`, a POST request to `/api/billing/portal` will trigger this function).
    *   `async`: Indicates that the function will perform asynchronous operations (like database queries or API calls).
    *   `request: NextRequest`: The incoming HTTP request object, typed as `NextRequest`, containing information like the request body, headers, and URL.

```typescript
  const session = await getSession()
```
*   **Purpose**: Retrieves the current user's session.
*   **Explanation**: This line asynchronously calls `getSession()` to obtain the authentication session data for the user making the request. `await` ensures that the code pauses here until the session data is available. The `session` object will contain details about the logged-in user, such as their ID.

```typescript
  try {
```
*   **Purpose**: Starts a `try` block for error handling.
*   **Explanation**: This block encloses the main logic of the function. Any error that occurs within this `try` block will be caught by the subsequent `catch` block, allowing for graceful error handling.

```typescript
    if (!session?.user?.id) {
```
*   **Purpose**: Checks if the user is authenticated.
*   **Explanation**: This `if` condition checks if the `session` object exists, if it has a `user` property, and if that `user` object has an `id`. If any of these are missing, it means the user is not authenticated or their session is invalid. The `?.` (optional chaining) operator safely accesses nested properties without throwing an error if an intermediate property is `null` or `undefined`.

```typescript
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
```
*   **Purpose**: Returns an unauthorized error response.
*   **Explanation**: If the user is not authenticated, this line immediately returns an HTTP response.
    *   `NextResponse.json(...)`: Creates a JSON response.
    *   `{ error: 'Unauthorized' }`: The JSON payload, indicating the error.
    *   `{ status: 401 }`: Sets the HTTP status code to 401, which signifies "Unauthorized."

```typescript
    }
```
*   **Explanation**: Closes the `if` block.

```typescript
    const body = await request.json().catch(() => ({}))
```
*   **Purpose**: Parses the request body as JSON, handling potential parsing errors.
*   **Explanation**:
    *   `await request.json()`: Asynchronously parses the incoming request body, assuming it's JSON, and converts it into a JavaScript object.
    *   `.catch(() => ({}))`: This is an error-handling mechanism. If `request.json()` fails (e.g., if the request body is not valid JSON or is empty), instead of throwing an error, it catches it and returns an empty object `{}`. This prevents the entire request from crashing due to malformed input and provides a safe default.

```typescript
    const context: 'user' | 'organization' =
      body?.context === 'organization' ? 'organization' : 'user'
```
*   **Purpose**: Determines the billing context (user or organization).
*   **Explanation**: This line defines a constant `context` with a TypeScript type of a string literal union `'user' | 'organization'`.
    *   `body?.context === 'organization'`: It checks if the `context` property exists in the parsed request `body` and if its value is exactly the string `'organization'`. The `?.` ensures it doesn't crash if `body` or `body.context` is `undefined`.
    *   `? 'organization' : 'user'`: This is a ternary operator. If the condition is true, `context` is set to `'organization'`. Otherwise, it defaults to `'user'`.

```typescript
    const organizationId: string | undefined = body?.organizationId || undefined
```
*   **Purpose**: Extracts the `organizationId` from the request body.
*   **Explanation**: This line defines `organizationId` as a string or `undefined`.
    *   `body?.organizationId`: Attempts to access the `organizationId` property from the `body` object.
    *   `|| undefined`: If `body.organizationId` is `null`, `undefined`, `0`, `false`, or an empty string (falsy values), `organizationId` will explicitly be set to `undefined`. Otherwise, it will take the value from `body.organizationId`.

```typescript
    const returnUrl: string = body?.returnUrl || `${getBaseUrl()}/workspace?billing=updated`
```
*   **Purpose**: Determines the URL Stripe should redirect to after the billing portal session.
*   **Explanation**: This line defines `returnUrl` as a string.
    *   `body?.returnUrl`: It tries to get a `returnUrl` from the request body. This allows the client to specify where the user should be redirected after managing their billing on Stripe.
    *   `|| `${getBaseUrl()}/workspace?billing=updated``: If `body.returnUrl` is not provided (or is falsy), it defaults to a constructed URL.
        *   `` `${getBaseUrl()}/workspace?billing=updated` ``: Uses a template literal to build a default URL by calling `getBaseUrl()` (e.g., `https://app.yourdomain.com`) and appending `/workspace?billing=updated`. This provides a sensible fallback, usually directing the user back to a relevant section of the application with a status indicator.

```typescript
    const stripe = requireStripeClient()
```
*   **Purpose**: Initializes the Stripe API client.
*   **Explanation**: Calls the `requireStripeClient()` function to get an authenticated Stripe client object. This `stripe` object will be used to make API calls to Stripe.

```typescript
    let stripeCustomerId: string | null = null
```
*   **Purpose**: Declares a variable to store the Stripe Customer ID.
*   **Explanation**: Initializes `stripeCustomerId` as a mutable variable (`let`) that can hold either a `string` (the Stripe Customer ID) or `null`. It's initially set to `null` because the ID hasn't been fetched yet.

```typescript
    if (context === 'organization') {
```
*   **Purpose**: Starts a conditional block for organization-specific billing logic.
*   **Explanation**: This `if` statement checks if the `context` variable (determined earlier from the request body) is `'organization'`. If true, the code inside this block will execute to fetch the customer ID for an organization.

```typescript
      if (!organizationId) {
```
*   **Purpose**: Validates that `organizationId` is provided when the context is 'organization'.
*   **Explanation**: If the `context` is `'organization'` but no `organizationId` was provided in the request body, this condition is true, indicating a bad request.

```typescript
        return NextResponse.json({ error: 'organizationId is required' }, { status: 400 })
```
*   **Purpose**: Returns a "Bad Request" error if `organizationId` is missing.
*   **Explanation**: If `organizationId` is missing for an organization context, an HTTP 400 status ("Bad Request") is returned with an appropriate error message.

```typescript
      }
```
*   **Explanation**: Closes the inner `if` block.

```typescript
      const rows = await db
        .select({ customer: subscriptionTable.stripeCustomerId })
        .from(subscriptionTable)
        .where(
          and(
            eq(subscriptionTable.referenceId, organizationId),
            eq(subscriptionTable.status, 'active')
          )
        )
        .limit(1)
```
*   **Purpose**: Queries the database for an organization's Stripe Customer ID.
*   **Explanation**: This is a Drizzle ORM query to fetch the `stripeCustomerId` for an organization.
    *   `await db`: Executes a database operation asynchronously.
    *   `.select({ customer: subscriptionTable.stripeCustomerId })`: Specifies which columns to retrieve. It selects the `stripeCustomerId` column from the `subscriptionTable` and aliases it as `customer` in the result object.
    *   `.from(subscriptionTable)`: Specifies the table to query, which is the `subscriptionTable` schema.
    *   `.where(...)`: Specifies the filtering conditions.
        *   `and(...)`: Combines multiple conditions using a logical AND.
        *   `eq(subscriptionTable.referenceId, organizationId)`: Matches rows where the `referenceId` column (which likely stores the `organizationId` in the `subscriptionTable`) equals the provided `organizationId`.
        *   `eq(subscriptionTable.status, 'active')`: Further filters to only include subscriptions that are currently `active`.
    *   `.limit(1)`: Limits the query to return at most one row, as we only need one `stripeCustomerId` for the organization.
    *   `const rows`: The result of the query will be an array of objects, where each object has a `customer` property containing the `stripeCustomerId`.

```typescript
      stripeCustomerId = rows.length > 0 ? rows[0].customer || null : null
```
*   **Purpose**: Extracts the `stripeCustomerId` from the query result.
*   **Explanation**: This line assigns the fetched `stripeCustomerId` to the `stripeCustomerId` variable.
    *   `rows.length > 0`: Checks if any rows were returned by the query.
    *   `? rows[0].customer || null`: If rows exist, it accesses the `customer` property of the first row (`rows[0].customer`). The `|| null` ensures that if `rows[0].customer` is `undefined` or `null` from the database, `stripeCustomerId` is explicitly set to `null` rather than `undefined`.
    *   `: null`: If no rows were returned (`rows.length` is 0), `stripeCustomerId` is set to `null`.

```typescript
    } else {
```
*   **Purpose**: Starts the block for user-specific billing logic.
*   **Explanation**: If the `context` is *not* `'organization'` (i.e., it's `'user'` or defaults to it), the code inside this `else` block executes to fetch the customer ID for the individual user.

```typescript
      const rows = await db
        .select({ customer: user.stripeCustomerId })
        .from(user)
        .where(eq(user.id, session.user.id))
        .limit(1)
```
*   **Purpose**: Queries the database for the user's Stripe Customer ID.
*   **Explanation**: This is a Drizzle ORM query to fetch the `stripeCustomerId` for the currently authenticated user.
    *   `.select({ customer: user.stripeCustomerId })`: Selects the `stripeCustomerId` column from the `user` table and aliases it as `customer`.
    *   `.from(user)`: Specifies the `user` table schema.
    *   `.where(eq(user.id, session.user.id))`: Filters to find the row where the `id` column of the `user` table matches the ID of the current authenticated user (`session.user.id`).
    *   `.limit(1)`: Limits the query to return at most one row, as a user has only one Stripe Customer ID associated with their account.

```typescript
      stripeCustomerId = rows.length > 0 ? rows[0].customer || null : null
```
*   **Purpose**: Extracts the `stripeCustomerId` from the query result for the user.
*   **Explanation**: This line performs the same logic as the organization context to safely extract the `stripeCustomerId` from the `rows` array.

```typescript
    }
```
*   **Explanation**: Closes the `else` block.

```typescript
    if (!stripeCustomerId) {
```
*   **Purpose**: Checks if a Stripe Customer ID was successfully found.
*   **Explanation**: If `stripeCustomerId` is still `null` after attempting to fetch it (meaning no customer ID was found in either the `subscriptionTable` for an organization or the `user` table for the user), this condition is true.

```typescript
      logger.error('Stripe customer not found for portal session', {
        context,
        organizationId,
        userId: session.user.id,
      })
```
*   **Purpose**: Logs an error if the Stripe Customer ID is missing.
*   **Explanation**: If `stripeCustomerId` is not found, an error message is logged using the `logger` instance. It includes contextual information (`context`, `organizationId`, `userId`) to help debug why the customer ID was not found.

```typescript
      return NextResponse.json({ error: 'Stripe customer not found' }, { status: 404 })
```
*   **Purpose**: Returns a "Not Found" error response.
*   **Explanation**: If `stripeCustomerId` is missing, an HTTP 404 status ("Not Found") is returned to the client, indicating that the requested customer couldn't be located.

```typescript
    }
```
*   **Explanation**: Closes the `if` block.

```typescript
    const portal = await stripe.billingPortal.sessions.create({
```
*   **Purpose**: Creates a new Stripe Billing Portal session.
*   **Explanation**: This is the crucial call to the Stripe API.
    *   `await stripe.billingPortal.sessions.create(...)`: Asynchronously calls the `create` method on the `billingPortal.sessions` object of the Stripe client. This method initiates the creation of a new session for the Stripe Billing Portal.
    *   The argument is an object containing the required parameters:

```typescript
      customer: stripeCustomerId,
```
*   **Purpose**: Specifies the Stripe customer for whom the portal session is created.
*   **Explanation**: This property tells Stripe which customer's billing information should be managed in the portal. Its value is the `stripeCustomerId` that was previously fetched from the database.

```typescript
      return_url: returnUrl,
```
*   **Purpose**: Specifies the URL Stripe should redirect to after the portal session.
*   **Explanation**: This property provides the `returnUrl` (determined earlier) to Stripe. After the user finishes their actions in the Stripe Billing Portal, they will be redirected back to this URL in your application.

```typescript
    })
```
*   **Explanation**: Closes the object passed to `create()` and the `create()` method call. The result of this call, which includes the URL for the portal, is stored in the `portal` variable.

```typescript
    return NextResponse.json({ url: portal.url })
```
*   **Purpose**: Returns the URL of the created Stripe Billing Portal session.
*   **Explanation**: If the Stripe API call is successful, this line returns an HTTP 200 OK response with a JSON payload containing the `url` property from the `portal` object. This URL is the link to Stripe's hosted billing portal that the client-side application will use to redirect the user.

```typescript
  } catch (error) {
```
*   **Purpose**: Catches any errors that occurred in the `try` block.
*   **Explanation**: If any unhandled exception occurs within the `try` block, execution jumps here. The `error` variable will contain the thrown error object.

```typescript
    logger.error('Failed to create billing portal session', { error })
```
*   **Purpose**: Logs the error.
*   **Explanation**: The `logger` instance logs an error message indicating a failure to create the billing portal session, along with the actual `error` object for debugging purposes.

```typescript
    return NextResponse.json({ error: 'Failed to create billing portal session' }, { status: 500 })
```
*   **Purpose**: Returns a generic server error response.
*   **Explanation**: If an unexpected error occurs, an HTTP 500 status ("Internal Server Error") is returned to the client with a generic error message, protecting sensitive internal error details.

```typescript
  }
```
*   **Explanation**: Closes the `catch` block.

```typescript
}
```
*   **Explanation**: Closes the `POST` function definition.