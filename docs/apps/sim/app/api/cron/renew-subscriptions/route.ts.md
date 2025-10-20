This TypeScript file defines a Next.js API route that acts as a cron job. Its primary responsibility is to automatically monitor and renew expiring Microsoft Teams chat subscriptions associated with active webhooks in the system.

Microsoft Teams subscriptions have a limited lifespan (typically around 3 days) and do not auto-renew. This cron job ensures that these subscriptions are proactively extended before they expire, preventing service interruptions for users whose workflows rely on these webhooks.

---

## Detailed Explanation

Let's break down the code step by step.

### Imports

This section imports necessary modules and utilities from various parts of the application and external libraries.

```typescript
import { db } from '@sim/db'
import { webhook as webhookTable, workflow as workflowTable } from '@sim/db/schema'
import { and, eq } from 'drizzle-orm'
import { type NextRequest, NextResponse } from 'next/server'
import { verifyCronAuth } from '@/lib/auth/internal'
import { createLogger } from '@/lib/logs/console/logger'
import { refreshAccessTokenIfNeeded } from '@/app/api/auth/oauth/utils'
```

*   `import { db } from '@sim/db'`:
    *   **Purpose**: Imports the database client instance, which is used to interact with the database.
    *   **Explanation**: `db` is likely an instance of a Drizzle ORM database connection, providing methods to run queries like `select`, `update`, etc.
*   `import { webhook as webhookTable, workflow as workflowTable } from '@sim/db/schema'`:
    *   **Purpose**: Imports the Drizzle ORM schema definitions for the `webhook` and `workflow` tables.
    *   **Explanation**: These variables (`webhookTable`, `workflowTable`) represent the structure of these tables in the database and are used to construct type-safe database queries.
*   `import { and, eq } from 'drizzle-orm'`:
    *   **Purpose**: Imports helper functions from Drizzle ORM for building query conditions.
    *   **Explanation**:
        *   `and`: Used to combine multiple conditions with a logical `AND` operator (e.g., `condition1 AND condition2`).
        *   `eq`: Used to create an "equals" comparison condition (e.g., `column = value`).
*   `import { type NextRequest, NextResponse } from 'next/server'`:
    *   **Purpose**: Imports types and classes specific to Next.js API routes.
    *   **Explanation**:
        *   `NextRequest`: A type definition for the incoming HTTP request object in Next.js API routes, providing access to headers, body, URL, etc.
        *   `NextResponse`: A class used to create and return HTTP responses from Next.js API routes, allowing you to set status codes, headers, and JSON bodies.
*   `import { verifyCronAuth } from '@/lib/auth/internal'`:
    *   **Purpose**: Imports a custom utility function to authenticate requests originating from internal cron services.
    *   **Explanation**: This function ensures that only authorized automated jobs (like this subscription renewal) can access this endpoint, typically by checking a shared secret or API key in the request headers.
*   `import { createLogger } from '@/lib/logs/console/logger'`:
    *   **Purpose**: Imports a custom utility to create a structured logger instance.
    *   **Explanation**: This allows for consistent and informative logging throughout the application, making it easier to monitor the cron job's execution and debug issues.
*   `import { refreshAccessTokenIfNeeded } from '@/app/api/auth/oauth/utils'`:
    *   **Purpose**: Imports a utility function specifically designed to handle OAuth access tokens.
    *   **Explanation**: This function is crucial for interacting with external APIs (like Microsoft Graph) that require authentication. It intelligently refreshes an access token if it's expired or about to expire, ensuring that API calls are always made with a valid token.

### Logger Initialization

```typescript
const logger = createLogger('TeamsSubscriptionRenewal')
```

*   **Purpose**: Initializes a logger instance specifically for this cron job.
*   **Explanation**: When messages are logged, they will be tagged with `'TeamsSubscriptionRenewal'`, making it easy to filter and identify logs related to this specific process in application logs.

### JSDoc and Function Definition

```typescript
/**
 * Cron endpoint to renew Microsoft Teams chat subscriptions before they expire
 *
 * Teams subscriptions expire after ~3 days and must be renewed.
 * Configured in helm/sim/values.yaml under cronjobs.jobs.renewSubscriptions
 */
export async function GET(request: NextRequest) {
  // ... function body
}
```

*   `/** ... */`:
    *   **Purpose**: This is a JSDoc comment providing a high-level overview of the file's purpose.
    *   **Explanation**: It clarifies that this is a "cron endpoint" (meaning it's meant to be triggered by an automated scheduler) and its core task is to "renew Microsoft Teams chat subscriptions." It also notes the 3-day expiration period and where its scheduling is configured (a Helm values file).
*   `export async function GET(request: NextRequest)`:
    *   **Purpose**: Defines an asynchronous Next.js API route handler for HTTP `GET` requests.
    *   **Explanation**: In Next.js, defining an `async function GET(request: NextRequest)` (or `POST`, `PUT`, etc.) within a file like `app/api/route.ts` makes it an API endpoint. When an HTTP `GET` request hits this route, this function will be executed. The `request` object contains details about the incoming request. The `async` keyword indicates that the function will perform asynchronous operations (like database queries or external API calls) and will return a `Promise`.

### Main Execution Block and Error Handling

```typescript
export async function GET(request: NextRequest) {
  try {
    // ... main cron job logic
  } catch (error) {
    logger.error('Error in Teams subscription renewal job:', error)
    return NextResponse.json({ error: 'Internal server error' }, { status: 500 })
  }
}
```

*   `try { ... } catch (error) { ... }`:
    *   **Purpose**: This `try...catch` block is a fundamental error handling mechanism.
    *   **Explanation**: It wraps the entire cron job logic. If any unhandled error occurs during the execution of the code within the `try` block, the program flow immediately jumps to the `catch` block. Here, the error is logged using `logger.error`, and a standard HTTP 500 "Internal Server Error" response is returned to the caller, preventing the server from crashing and providing a clear error message.

### Cron Authentication

```typescript
    const authError = verifyCronAuth(request, 'Teams subscription renewal')
    if (authError) {
      return authError
    }
```

*   `const authError = verifyCronAuth(request, 'Teams subscription renewal')`:
    *   **Purpose**: Calls a utility function to verify if the incoming request is authorized as an internal cron job.
    *   **Explanation**: `verifyCronAuth` likely checks for a specific header or token in the `request` object. The second argument `'Teams subscription renewal'` might be used for logging or identifying the specific cron job being authenticated. If authentication fails, it returns an error response (`NextResponse` object), otherwise, it returns `null` or `undefined`.
*   `if (authError) { return authError }`:
    *   **Purpose**: Checks if `verifyCronAuth` returned an error response.
    *   **Explanation**: If `authError` contains a `NextResponse` object (indicating an authentication failure, e.g., a 401 Unauthorized or 403 Forbidden status), that response is immediately returned, stopping the execution of the cron job's main logic.

### Job Initialization and Counters

```typescript
    logger.info('Starting Teams subscription renewal job')

    let totalRenewed = 0
    let totalFailed = 0
    let totalChecked = 0
```

*   `logger.info('Starting Teams subscription renewal job')`:
    *   **Purpose**: Logs an informational message indicating the start of the cron job.
    *   **Explanation**: This is useful for tracking when the job runs and confirming its initiation.
*   `let totalRenewed = 0`, `let totalFailed = 0`, `let totalChecked = 0`:
    *   **Purpose**: Initializes three variables to keep track of the job's progress and outcome.
    *   **Explanation**:
        *   `totalRenewed`: Counts how many subscriptions were successfully renewed.
        *   `totalFailed`: Counts how many subscription renewal attempts failed.
        *   `totalChecked`: Counts how many relevant subscriptions were identified and processed (regardless of renewal success).
    *   These counters will be incremented as the job progresses and ultimately returned in the final response.

### Fetching Webhooks from Database

```typescript
    // Get all active Microsoft Teams webhooks with their workflows
    const webhooksWithWorkflows = await db
      .select({
        webhook: webhookTable,
        workflow: workflowTable,
      })
      .from(webhookTable)
      .innerJoin(workflowTable, eq(webhookTable.workflowId, workflowTable.id))
      .where(and(eq(webhookTable.isActive, true), eq(webhookTable.provider, 'microsoftteams')))
```

*   **Purpose**: This is a database query using Drizzle ORM to retrieve all active Microsoft Teams webhooks and their associated workflow details.
*   **Simplification**: "We're asking our database for a list of all webhooks that are currently active and specifically configured for Microsoft Teams. For each of these webhooks, we also want to get the details of the workflow they belong to."
*   `const webhooksWithWorkflows = await db`:
    *   **Explanation**: Initiates a database query using the `db` client and `await`s its result because it's an asynchronous operation.
*   `.select({ webhook: webhookTable, workflow: workflowTable })`:
    *   **Explanation**: Specifies which data columns (or entire entities in Drizzle's case) to retrieve. It creates an object where `webhook` will contain all columns from `webhookTable` and `workflow` will contain all columns from `workflowTable`.
*   `.from(webhookTable)`:
    *   **Explanation**: Specifies `webhookTable` as the primary table for the query.
*   `.innerJoin(workflowTable, eq(webhookTable.workflowId, workflowTable.id))`:
    *   **Explanation**: Performs an `INNER JOIN` operation. This links each webhook record with its corresponding workflow record based on the condition that `webhookTable.workflowId` must be equal to `workflowTable.id`. Only records that have a match in both tables are included.
*   `.where(and(eq(webhookTable.isActive, true), eq(webhookTable.provider, 'microsoftteams')))`:
    *   **Explanation**: Filters the joined records based on two conditions:
        *   `eq(webhookTable.isActive, true)`: The `isActive` column of the `webhookTable` must be `true`.
        *   `eq(webhookTable.provider, 'microsoftteams')`: The `provider` column of the `webhookTable` must be `'microsoftteams'`.
    *   `and(...)`: Ensures both conditions must be true for a record to be selected.

### Initial Log and Renewal Threshold

```typescript
    logger.info(
      `Found ${webhooksWithWorkflows.length} active Teams webhooks, checking for expiring subscriptions`
    )

    // Renewal threshold: 48 hours before expiration
    const renewalThreshold = new Date(Date.now() + 48 * 60 * 60 * 1000)
```

*   `logger.info(...)`:
    *   **Purpose**: Logs the number of relevant webhooks found in the database.
    *   **Explanation**: This provides an immediate status update on how many potential subscriptions need to be checked.
*   `const renewalThreshold = new Date(Date.now() + 48 * 60 * 60 * 1000)`:
    *   **Purpose**: Calculates a timestamp that represents the "renewal threshold."
    *   **Simplification**: "We're setting a deadline: any subscription expiring within the next 48 hours is considered 'expiring soon' and needs attention."
    *   **Explanation**:
        *   `Date.now()`: Returns the current timestamp in milliseconds since the Unix epoch.
        *   `48 * 60 * 60 * 1000`: Calculates 48 hours in milliseconds (48 hours \* 60 minutes/hour \* 60 seconds/minute \* 1000 milliseconds/second).
        *   `new Date(...)`: Creates a `Date` object from the calculated future timestamp. If a subscription's expiration date is *earlier* than this `renewalThreshold`, it means it's expiring within the next 48 hours and needs renewal.

### Iterating and Processing Webhooks

This `for...of` loop is the core logic where each webhook is inspected and renewed if necessary.

```typescript
    for (const { webhook, workflow } of webhooksWithWorkflows) {
      const config = (webhook.providerConfig as Record<string, any>) || {}

      // Check if this is a Teams chat subscription that needs renewal
      if (config.triggerId !== 'microsoftteams_chat_subscription') continue

      const expirationStr = config.subscriptionExpiration as string | undefined
      if (!expirationStr) continue

      const expiresAt = new Date(expirationStr)
      if (expiresAt > renewalThreshold) continue // Not expiring soon

      totalChecked++

      try {
        // ... renewal logic for individual webhook
      } catch (error) {
        logger.error(`Error renewing subscription for webhook ${webhook.id}:`, error)
        totalFailed++
      }
    }
```

*   `for (const { webhook, workflow } of webhooksWithWorkflows)`:
    *   **Purpose**: Iterates over each object in the `webhooksWithWorkflows` array.
    *   **Explanation**: Each object in the array contains `webhook` and `workflow` properties (as defined in the `db.select()` call). Destructuring `const { webhook, workflow }` extracts these two objects for easy access within the loop.
*   `const config = (webhook.providerConfig as Record<string, any>) || {}`:
    *   **Purpose**: Extracts the `providerConfig` from the `webhook` object.
    *   **Explanation**: `providerConfig` is likely a JSONB (JSON Binary) column in the database, storing dynamic configuration data specific to the webhook provider. It's cast `as Record<string, any>` to tell TypeScript its shape, and `|| {}` provides a fallback to an empty object if `providerConfig` is `null` or `undefined`.
*   `if (config.triggerId !== 'microsoftteams_chat_subscription') continue`:
    *   **Purpose**: Filters out webhooks that are not specifically for Microsoft Teams chat subscriptions.
    *   **Explanation**: This ensures that only the relevant type of Teams webhook (which requires subscription renewal) is processed. `continue` skips the rest of the loop for the current webhook and moves to the next one.
*   `const expirationStr = config.subscriptionExpiration as string | undefined`:
    *   **Purpose**: Extracts the subscription expiration date string from the `config`.
    *   **Explanation**: `subscriptionExpiration` is expected to be a string within the `providerConfig`. It's cast `as string | undefined` because it might not always be present.
*   `if (!expirationStr) continue`:
    *   **Purpose**: Skips if no expiration date string is found.
    *   **Explanation**: Without an expiration date, there's nothing to renew for this webhook.
*   `const expiresAt = new Date(expirationStr)`:
    *   **Purpose**: Converts the expiration date string into a `Date` object for comparison.
*   `if (expiresAt > renewalThreshold) continue`:
    *   **Purpose**: Checks if the subscription is expiring soon based on the `renewalThreshold`.
    *   **Simplification**: "If the subscription is still good for more than 48 hours, we don't need to do anything yet, so we skip this webhook."
    *   **Explanation**: If the `expiresAt` date is *after* the `renewalThreshold` (meaning it's more than 48 hours away from expiring), the loop `continue`s to the next webhook.
*   `totalChecked++`:
    *   **Purpose**: Increments the counter for webhooks that were actually checked for renewal.
    *   **Explanation**: This webhook passed all initial filters and is now considered for renewal.
*   `try { ... } catch (error) { ... }`:
    *   **Purpose**: This inner `try...catch` block handles errors specifically for the renewal process of *each individual webhook*.
    *   **Explanation**: If an error occurs during the API call, token refresh, or database update for a single webhook, it logs the error, increments `totalFailed`, and then `continue`s to the next webhook, preventing a single failure from stopping the entire cron job.

### Individual Webhook Renewal Logic

This block is inside the inner `try` statement, executed for each webhook that needs renewal.

```typescript
        logger.info(
          `Renewing Teams subscription for webhook ${webhook.id} (expires: ${expiresAt.toISOString()})`
        )

        const credentialId = config.credentialId as string | undefined
        const externalSubscriptionId = config.externalSubscriptionId as string | undefined

        if (!credentialId || !externalSubscriptionId) {
          logger.error(`Missing credentialId or externalSubscriptionId for webhook ${webhook.id}`)
          totalFailed++
          continue
        }

        // Get fresh access token
        const accessToken = await refreshAccessTokenIfNeeded(
          credentialId,
          workflow.userId,
          `renewal-${webhook.id}`
        )

        if (!accessToken) {
          logger.error(`Failed to get access token for webhook ${webhook.id}`)
          totalFailed++
          continue
        }

        // Extend subscription to maximum lifetime (4230 minutes = ~3 days)
        const maxLifetimeMinutes = 4230
        const newExpirationDateTime = new Date(
          Date.now() + maxLifetimeMinutes * 60 * 1000
        ).toISOString()

        const res = await fetch(
          `https://graph.microsoft.com/v1.0/subscriptions/${externalSubscriptionId}`,
          {
            method: 'PATCH',
            headers: {
              Authorization: `Bearer ${accessToken}`,
              'Content-Type': 'application/json',
            },
            body: JSON.stringify({ expirationDateTime: newExpirationDateTime }),
          }
        )

        if (!res.ok) {
          const error = await res.json()
          logger.error(
            `Failed to renew Teams subscription ${externalSubscriptionId} for webhook ${webhook.id}`,
            { status: res.status, error: error.error }
          )
          totalFailed++
          continue
        }

        const payload = await res.json()

        // Update webhook config with new expiration
        const updatedConfig = {
          ...config,
          subscriptionExpiration: payload.expirationDateTime,
        }

        await db
          .update(webhookTable)
          .set({ providerConfig: updatedConfig, updatedAt: new Date() })
          .where(eq(webhookTable.id, webhook.id))

        logger.info(
          `Successfully renewed Teams subscription for webhook ${webhook.id}. New expiration: ${payload.expirationDateTime}`
        )
        totalRenewed++
```

*   `logger.info(...)`:
    *   **Purpose**: Logs that a specific webhook's subscription is being renewed.
    *   **Explanation**: Includes the webhook ID and its current expiration for clear tracking.
*   `const credentialId = config.credentialId as string | undefined`
    *   `const externalSubscriptionId = config.externalSubscriptionId as string | undefined`:
    *   **Purpose**: Extracts critical identifiers needed for authentication and identifying the subscription with Microsoft.
    *   **Explanation**: `credentialId` is likely an internal ID referencing OAuth credentials, and `externalSubscriptionId` is the ID Microsoft assigned to the subscription.
*   `if (!credentialId || !externalSubscriptionId) { ... }`:
    *   **Purpose**: Checks if essential configuration data is missing.
    *   **Explanation**: If either ID is missing, renewal cannot proceed. An error is logged, `totalFailed` is incremented, and the loop moves to the next webhook.
*   `const accessToken = await refreshAccessTokenIfNeeded(...)`:
    *   **Purpose**: Obtains a valid OAuth access token to authenticate with the Microsoft Graph API.
    *   **Simplification**: "We need a temporary key (access token) to tell Microsoft to renew the subscription. This function gets or refreshes that key for us."
    *   **Explanation**: It uses the `credentialId` to locate the correct OAuth credentials, the `workflow.userId` for context (likely identifying the user who authorized the integration), and a descriptive tag for logging purposes.
*   `if (!accessToken) { ... }`:
    *   **Purpose**: Checks if an access token could be obtained.
    *   **Explanation**: If `refreshAccessTokenIfNeeded` fails to provide a token, renewal is impossible. An error is logged, `totalFailed` increments, and the loop continues.
*   `const maxLifetimeMinutes = 4230`:
    *   **Explanation**: Defines the maximum duration (in minutes) for which Microsoft allows extending a subscription in a single API call (4230 minutes is approximately 2.93 days).
*   `const newExpirationDateTime = new Date(Date.now() + maxLifetimeMinutes * 60 * 1000).toISOString()`:
    *   **Purpose**: Calculates the new expiration date and time.
    *   **Explanation**: It takes the current time (`Date.now()`) and adds the `maxLifetimeMinutes` (converted to milliseconds) to determine the absolute latest possible expiration date. `.toISOString()` formats this date into a string compatible with the Microsoft Graph API.
*   `const res = await fetch(...)`:
    *   **Purpose**: Makes an HTTP `PATCH` request to the Microsoft Graph API to renew the subscription.
    *   **Simplification**: "This is the actual command sent to Microsoft's servers: 'Please update the subscription with this specific ID and set its new expiration date to this time,' using our access token as proof of permission."
    *   **Explanation**:
        *   `https://graph.microsoft.com/v1.0/subscriptions/${externalSubscriptionId}`: The API endpoint for updating a specific subscription.
        *   `method: 'PATCH'`: Indicates that this request is modifying an existing resource.
        *   `headers`:
            *   `Authorization: Bearer ${accessToken}`: Provides the OAuth access token for authentication.
            *   `'Content-Type': 'application/json'`: Specifies that the request body is in JSON format.
        *   `body: JSON.stringify({ expirationDateTime: newExpirationDateTime })`: The JSON payload containing the new desired expiration date.
*   `if (!res.ok) { ... }`:
    *   **Purpose**: Checks if the API request to Microsoft was successful.
    *   **Explanation**: If `res.ok` is `false` (meaning the HTTP status code was not in the 2xx range), it indicates an error. The response body is parsed (`await res.json()`) to get more details about the error, which are then logged. `totalFailed` is incremented, and the loop `continue`s.
*   `const payload = await res.json()`:
    *   **Purpose**: Parses the successful JSON response from the Microsoft Graph API.
    *   **Explanation**: This `payload` typically contains the confirmed new expiration date from Microsoft, along with other subscription details.
*   `const updatedConfig = { ...config, subscriptionExpiration: payload.expirationDateTime, }`:
    *   **Purpose**: Creates a new configuration object for the webhook.
    *   **Explanation**: It takes all existing properties from the `config` object (`...config`) and specifically updates the `subscriptionExpiration` property with the new, confirmed expiration date returned by Microsoft.
*   `await db.update(webhookTable).set({ ... }).where(...)`:
    *   **Purpose**: Updates the webhook record in the database with the new expiration date.
    *   **Explanation**:
        *   `db.update(webhookTable)`: Specifies that the `webhookTable` should be updated.
        *   `.set({ providerConfig: updatedConfig, updatedAt: new Date() })`: Sets the `providerConfig` column to the `updatedConfig` object (persisting the new expiration date) and also updates the `updatedAt` timestamp to the current time.
        *   `.where(eq(webhookTable.id, webhook.id))`: Ensures that only the specific webhook (identified by its `id`) is updated.
*   `logger.info(...)`:
    *   **Purpose**: Logs a success message after a subscription has been renewed and the database updated.
    *   **Explanation**: Provides clear confirmation that the renewal was successful, including the new expiration date.
*   `totalRenewed++`:
    *   **Purpose**: Increments the counter for successfully renewed subscriptions.

### Final Summary and Response

```typescript
    logger.info(
      `Teams subscription renewal job completed. Checked: ${totalChecked}, Renewed: ${totalRenewed}, Failed: ${totalFailed}`
    )

    return NextResponse.json({
      success: true,
      checked: totalChecked,
      renewed: totalRenewed,
      failed: totalFailed,
      total: webhooksWithWorkflows.length,
    })
```

*   `logger.info(...)`:
    *   **Purpose**: Logs a summary of the entire cron job's execution.
    *   **Explanation**: This final log message provides an overview of how many webhooks were checked, successfully renewed, and failed, giving a quick status of the job's overall performance.
*   `return NextResponse.json({ ... })`:
    *   **Purpose**: Returns a JSON response to the caller (the cron scheduler).
    *   **Explanation**: This is the final output of the API route. It provides a structured JSON object indicating the `success` status of the job and includes the final counts for `checked`, `renewed`, `failed` subscriptions, and the `total` number of relevant webhooks initially found. This allows the calling service to easily parse and interpret the job's outcome.

---

This comprehensive breakdown covers the purpose, high-level logic, and line-by-line explanation of the provided TypeScript code, simplifying complex concepts for better understanding.