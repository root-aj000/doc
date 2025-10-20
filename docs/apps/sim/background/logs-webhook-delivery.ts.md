As a TypeScript expert and technical writer, I'm delighted to provide a detailed, easy-to-read explanation of the provided code.

---

## Explanation: `logsWebhookDelivery.ts` - Sending Workflow Execution Logs via Webhooks

This file defines a crucial background task responsible for delivering "workflow execution completed" events as webhooks to external services. It's designed for robustness, security, and includes a sophisticated retry mechanism.

### Purpose of This File

The primary purpose of `logsWebhookDelivery.ts` is to encapsulate the logic for reliably sending webhook notifications whenever a workflow execution completes (either successfully or with an error). It handles:

1.  **Retrieving Webhook Subscription Details:** Fetching the target URL, security secret, and payload customization preferences from the database.
2.  **Constructing the Webhook Payload:** Formatting the workflow execution log data into a standardized JSON structure.
3.  **Security:** Optionally generating an HMAC signature to allow the receiving service to verify the webhook's authenticity and integrity.
4.  **Reliable Delivery:** Attempting to send the webhook via HTTP POST, and implementing a custom exponential backoff and jitter retry strategy for transient failures.
5.  **State Management:** Updating the delivery status (pending, in-progress, success, failed) and attempt count in the database throughout the process, ensuring atomicity and preventing race conditions.
6.  **Error Handling & Logging:** Comprehensive logging and error handling for various failure scenarios, including network issues, timeouts, and HTTP error responses.

In essence, it acts as a robust outbound messaging system for workflow completion events.

---

### Core Concepts & Dependencies

Before diving into the line-by-line explanation, let's understand some key components used:

*   **Trigger.dev (`@trigger.dev/sdk`):** This is a serverless platform for long-running, resilient background jobs. The `task` function defines a background job, and `wait` allows pausing execution. This file defines a Trigger.dev task.
*   **Drizzle ORM (`drizzle-orm`):** A modern TypeScript ORM used for interacting with the database (`db`). It provides type-safe ways to build SQL queries.
*   **Node.js `crypto` module:** Used for cryptographic operations, specifically `createHmac` to generate secure signatures for webhooks.
*   **`uuid`:** A library for generating universally unique identifiers (UUIDs).
*   **Custom Logging (`@/lib/logs/console/logger`):** A logger utility for structured logging within the application.
*   **Secrets Management (`@/lib/utils/decryptSecret`):** A utility to securely decrypt sensitive data, such as webhook secrets, before use.

### Retry Strategy Explained

This file implements a custom, robust retry strategy for webhook deliveries. Most webhook failures are temporary (e.g., recipient server overloaded, network glitch).

*   **`MAX_ATTEMPTS` (5):** The maximum number of times the system will try to deliver a webhook.
*   **`RETRY_DELAYS`:** An array defining increasing delays between retries (5 seconds, 15 seconds, 1 minute, 3 minutes, 10 minutes). This is an **exponential backoff** strategy, which is good practice to avoid overwhelming a struggling recipient server.
*   **`getRetryDelayWithJitter(baseDelay)`:** This function adds a small random amount (up to 10%) to the `baseDelay`. This "jitter" is crucial to prevent the "thundering herd" problem, where many failed tasks might all retry at *exactly* the same time, leading to another cascade of failures. By adding randomness, the retries are spread out.

**Why a custom retry?**
While Trigger.dev tasks have built-in retry capabilities, this task explicitly sets `maxAttempts: 1` in its definition. This means the *platform* won't automatically retry. Instead, the task implements its *own* retry logic. This gives finer-grained control, allowing the task to:
    *   Distinguish between retryable (e.g., 5xx, 429 errors, network timeout) and non-retryable (e.g., 4xx client errors) failures.
    *   Implement specific delays and jitter.
    *   Update database status with `nextAttemptAt` and `errorMessage` for better observability.
    *   Leverage Trigger.dev's `wait` and `trigger` functions to re-queue the task for future processing, benefiting from Trigger.dev's underlying reliability guarantees.

---

### Detailed Line-by-Line Explanation

```typescript
import { createHmac } from 'crypto'
import { db } from '@sim/db'
import {
  workflowLogWebhook,
  workflowLogWebhookDelivery,
  workflow as workflowTable,
} from '@sim/db/schema'
import { task, wait } from '@trigger.dev/sdk'
import { and, eq, isNull, lte, or, sql } from 'drizzle-orm'
import { v4 as uuidv4 } from 'uuid'
import { createLogger } from '@/lib/logs/console/logger'
import type { WorkflowExecutionLog } from '@/lib/logs/types'
import { decryptSecret } from '@/lib/utils'
```

*   **`import { createHmac } from 'crypto'`**: Imports the `createHmac` function from Node.js's built-in `crypto` module, used for generating cryptographic hash-based message authentication codes (HMACs) to sign webhook payloads.
*   **`import { db } from '@sim/db'`**: Imports the database client instance, likely configured with Drizzle ORM.
*   **`import { workflowLogWebhook, workflowLogWebhookDelivery, workflow as workflowTable } from '@sim/db/schema'`**: Imports Drizzle ORM schema definitions for the `workflowLogWebhook` (webhook subscriptions), `workflowLogWebhookDelivery` (individual webhook delivery attempts), and `workflow` (workflow metadata) tables.
*   **`import { task, wait } from '@trigger.dev/sdk'`**: Imports core utilities from the Trigger.dev SDK:
    *   `task`: The decorator/function used to define a background task.
    *   `wait`: A function to pause task execution for a specified duration, resilient to server restarts.
*   **`import { and, eq, isNull, lte, or, sql } from 'drizzle-orm'`**: Imports Drizzle ORM query builders:
    *   `and`: Combines multiple conditions with a logical AND.
    *   `eq`: Checks for equality.
    *   `isNull`: Checks if a column is NULL.
    *   `lte`: Checks if a column is "less than or equal to."
    *   `or`: Combines multiple conditions with a logical OR.
    *   `sql`: Allows embedding raw SQL expressions, useful for operations like incrementing a counter.
*   **`import { v4 as uuidv4 } from 'uuid'`**: Imports the UUID v4 generator for creating unique identifiers.
*   **`import { createLogger } from '@/lib/logs/console/logger'`**: Imports a custom logger utility.
*   **`import type { WorkflowExecutionLog } from '@/lib/logs/types'`**: Imports the TypeScript type definition for a `WorkflowExecutionLog`, which is the data structure representing a completed workflow execution.
*   **`import { decryptSecret } from '@/lib/utils'`**: Imports a utility function to decrypt encrypted secrets, likely stored in the database.

```typescript
const logger = createLogger('LogsWebhookDelivery')

// Quick retry strategy: 5 attempts over ~15 minutes
// Most webhook failures are transient and resolve quickly
const MAX_ATTEMPTS = 5
const RETRY_DELAYS = [
  5 * 1000, // 5 seconds (1st retry)
  15 * 1000, // 15 seconds (2nd retry)
  60 * 1000, // 1 minute (3rd retry)
  3 * 60 * 1000, // 3 minutes (4th retry)
  10 * 60 * 1000, // 10 minutes (5th and final retry)
]

// Add jitter to prevent thundering herd problem (up to 10% of delay)
function getRetryDelayWithJitter(baseDelay: number): number {
  const jitter = Math.random() * 0.1 * baseDelay
  return Math.floor(baseDelay + jitter)
}
```

*   **`const logger = createLogger('LogsWebhookDelivery')`**: Initializes a logger instance with the context `LogsWebhookDelivery` for consistent logging.
*   **`const MAX_ATTEMPTS = 5`**: Defines the constant for the maximum number of delivery attempts.
*   **`const RETRY_DELAYS = [...]`**: Defines an array of delays (in milliseconds) for each subsequent retry attempt. This implements an exponential backoff strategy.
*   **`function getRetryDelayWithJitter(baseDelay: number): number { ... }`**: A helper function that takes a base delay and adds a random "jitter" (up to 10% of the base delay) to it. This helps distribute retries over time, preventing many tasks from retrying simultaneously and overwhelming the target system.

```typescript
interface WebhookPayload {
  id: string
  type: 'workflow.execution.completed'
  timestamp: number
  data: {
    workflowId: string
    executionId: string
    status: 'success' | 'error'
    level: string
    trigger: string
    startedAt: string
    endedAt: string
    totalDurationMs: number
    cost?: any
    files?: any
    finalOutput?: any
    traceSpans?: any[]
    rateLimits?: { /* ... */ }
    usage?: { /* ... */ }
  }
  links: {
    log: string
    execution: string
  }
}
```

*   **`interface WebhookPayload { ... }`**: Defines the TypeScript interface for the structured JSON payload that will be sent in the webhook. It specifies required fields like `id`, `type`, `timestamp`, `data` (containing workflow-specific details), and `links` (for related resources). Optional fields like `cost`, `files`, `finalOutput`, `traceSpans`, `rateLimits`, and `usage` indicate that the payload can be customized based on subscription preferences.

```typescript
function generateSignature(secret: string, timestamp: number, body: string): string {
  const signatureBase = `${timestamp}.${body}`
  const hmac = createHmac('sha256', secret)
  hmac.update(signatureBase)
  return hmac.digest('hex')
}
```

*   **`function generateSignature(...)`**: This function creates a secure HMAC signature for the webhook payload.
    *   **`signatureBase = `${timestamp}.${body}``**: Concatenates the timestamp and the raw JSON body string. This forms the base string that will be signed.
    *   **`hmac = createHmac('sha256', secret)`**: Initializes an HMAC object using the SHA256 algorithm and a secret key provided by the webhook subscriber.
    *   **`hmac.update(signatureBase)`**: Feeds the `signatureBase` string into the HMAC algorithm.
    *   **`return hmac.digest('hex')`**: Computes the HMAC hash and returns it as a hexadecimal string. This signature will be sent as a header, allowing the receiving service to verify that the webhook originated from a trusted source and its content hasn't been tampered with.

```typescript
export const logsWebhookDelivery = task({
  id: 'logs-webhook-delivery',
  retry: {
    maxAttempts: 1, // We handle retries manually within the task
  },
  run: async (params: {
    deliveryId: string
    subscriptionId: string
    log: WorkflowExecutionLog
  }) => {
    const { deliveryId, subscriptionId, log } = params

    try {
```

*   **`export const logsWebhookDelivery = task({ ... })`**: Defines and exports the main Trigger.dev task.
    *   **`id: 'logs-webhook-delivery'`**: A unique identifier for this task.
    *   **`retry: { maxAttempts: 1 }`**: Configures Trigger.dev's *platform-level* retry mechanism. By setting `maxAttempts` to 1, we explicitly tell Trigger.dev *not* to automatically retry this task if it fails. This is because the task implements its own, more sophisticated retry logic internally.
    *   **`run: async (params: { ... }) => { ... }`**: The core asynchronous function that contains the task's main logic.
        *   **`params: { deliveryId: string; subscriptionId: string; log: WorkflowExecutionLog }`**: Defines the input parameters for the task:
            *   `deliveryId`: The ID of the specific webhook delivery attempt.
            *   `subscriptionId`: The ID of the webhook subscription.
            *   `log`: The `WorkflowExecutionLog` object containing details about the completed workflow.
        *   **`const { deliveryId, subscriptionId, log } = params`**: Destructures the input parameters for easier access.
        *   **`try { ... }`**: Starts a top-level `try...catch` block to handle any unexpected errors that might occur during the entire task execution.

```typescript
      const [subscription] = await db
        .select()
        .from(workflowLogWebhook)
        .where(eq(workflowLogWebhook.id, subscriptionId))
        .limit(1)

      if (!subscription || !subscription.active) {
        logger.warn(`Subscription ${subscriptionId} not found or inactive`)
        await db
          .update(workflowLogWebhookDelivery)
          .set({
            status: 'failed',
            errorMessage: 'Subscription not found or inactive',
            updatedAt: new Date(),
          })
          .where(eq(workflowLogWebhookDelivery.id, deliveryId))
        return
      }
```

*   **`const [subscription] = await db.select().from(workflowLogWebhook).where(eq(workflowLogWebhook.id, subscriptionId)).limit(1)`**: Fetches the webhook subscription details from the `workflowLogWebhook` table using Drizzle ORM. It selects the record where the `id` matches `subscriptionId` and limits the result to one (though `where` on `id` should naturally return at most one).
*   **`if (!subscription || !subscription.active)`**: Checks if the subscription was found and if it's active.
*   **`logger.warn(...)`**: Logs a warning if the subscription is not found or is inactive.
*   **`await db.update(workflowLogWebhookDelivery).set({ ... }).where(eq(workflowLogWebhookDelivery.id, deliveryId))`**: If the subscription is invalid, updates the corresponding `workflowLogWebhookDelivery` record in the database, setting its `status` to `'failed'` and adding an `errorMessage`.
*   **`return`**: Exits the task early as there's no active subscription to deliver to.

```typescript
      // Atomically claim this delivery row for processing and increment attempts
      const claimed = await db
        .update(workflowLogWebhookDelivery)
        .set({
          status: 'in_progress',
          attempts: sql`${workflowLogWebhookDelivery.attempts} + 1`,
          lastAttemptAt: new Date(),
          updatedAt: new Date(),
        })
        .where(
          and(
            eq(workflowLogWebhookDelivery.id, deliveryId),
            eq(workflowLogWebhookDelivery.status, 'pending'),
            // Only claim if not scheduled in the future or schedule has arrived
            or(
              isNull(workflowLogWebhookDelivery.nextAttemptAt),
              lte(workflowLogWebhookDelivery.nextAttemptAt, new Date())
            )
          )
        )
        .returning({ attempts: workflowLogWebhookDelivery.attempts })

      if (claimed.length === 0) {
        logger.info(`Delivery ${deliveryId} not claimable (already in progress or not due)`)
        return
      }
```

This section is crucial for **preventing race conditions** in a distributed system where multiple instances of this task might try to process the same webhook delivery.

*   **`const claimed = await db.update(...)`**: This initiates an atomic database update operation.
    *   **`.set({ status: 'in_progress', attempts: sql`${workflowLogWebhookDelivery.attempts} + 1`, lastAttemptAt: new Date(), updatedAt: new Date() })`**: Sets the status of the delivery to `'in_progress'`, increments the `attempts` counter using Drizzle's `sql` helper for raw SQL increment, and updates timestamps.
    *   **`.where(and(...))`**: This is the atomic "claiming" part. The update will *only* occur if *all* these conditions are met:
        *   `eq(workflowLogWebhookDelivery.id, deliveryId)`: Matches the specific delivery ID.
        *   `eq(workflowLogWebhookDelivery.status, 'pending')`: Ensures only deliveries that are currently `pending` can be claimed.
        *   `or(isNull(workflowLogWebhookDelivery.nextAttemptAt), lte(workflowLogWebhookDelivery.nextAttemptAt, new Date()))`: Ensures the delivery is either not scheduled for a future retry (`isNull`) or its scheduled retry time (`nextAttemptAt`) has already passed (`lte(..., new Date())`).
    *   **`.returning({ attempts: workflowLogWebhookDelivery.attempts })`**: Returns the `attempts` count *after* the update, allowing the task to know the current attempt number.
*   **`if (claimed.length === 0)`**: If `claimed` is empty, it means the `update` statement didn't affect any rows. This implies one of two things:
    *   Another instance of this task (or another process) already claimed and updated this specific `deliveryId`.
    *   The delivery was not `pending` or its `nextAttemptAt` time had not yet arrived.
*   **`logger.info(...)`**: Logs that the delivery was not claimable.
*   **`return`**: Exits the task, preventing duplicate processing.

```typescript
      const attempts = claimed[0].attempts
      const timestamp = Date.now()
      const eventId = `evt_${uuidv4()}`

      const payload: WebhookPayload = {
        id: eventId,
        type: 'workflow.execution.completed',
        timestamp,
        data: {
          workflowId: log.workflowId,
          executionId: log.executionId,
          status: log.level === 'error' ? 'error' : 'success',
          level: log.level,
          trigger: log.trigger,
          startedAt: log.startedAt,
          endedAt: log.endedAt || log.startedAt,
          totalDurationMs: log.totalDurationMs,
          cost: log.cost,
          files: (log as any).files,
        },
        links: {
          log: `/v1/logs/${log.id}`,
          execution: `/v1/logs/executions/${log.executionId}`,
        },
      }
```

*   **`const attempts = claimed[0].attempts`**: Retrieves the current attempt count from the result of the atomic update.
*   **`const timestamp = Date.now()`**: Gets the current UTC timestamp (in milliseconds) for use in the payload and signature.
*   **`const eventId = `evt_${uuidv4()}``**: Generates a unique ID for the webhook event, prefixed with `evt_`.
*   **`const payload: WebhookPayload = { ... }`**: Constructs the `WebhookPayload` object based on the `WorkflowExecutionLog` data.
    *   `id`: The generated `eventId`.
    *   `type`: Hardcoded to `'workflow.execution.completed'`.
    *   `timestamp`: The current `timestamp`.
    *   `data`: Contains detailed workflow execution information, mapped from the `log` object. The `status` is derived from `log.level` (if `error`, then `'error'`, otherwise `'success'`). `endedAt` defaults to `startedAt` if not present.
    *   `links`: Provides URLs for accessing the full log and execution details.
    *   `(log as any).files`: A type assertion indicating that `files` might not be directly on `WorkflowExecutionLog` but could exist on the underlying object.

```typescript
      if (subscription.includeFinalOutput && log.executionData) {
        payload.data.finalOutput = (log.executionData as any).finalOutput
      }

      if (subscription.includeTraceSpans && log.executionData) {
        payload.data.traceSpans = (log.executionData as any).traceSpans
      }
```

*   **Conditional Payload Fields:** These blocks conditionally add more data to the webhook payload based on the `subscription`'s preferences and whether `log.executionData` is available.
    *   **`if (subscription.includeFinalOutput && log.executionData)`**: If the subscriber wants the final output and `executionData` is present, `finalOutput` is added to `payload.data`.
    *   **`if (subscription.includeTraceSpans && log.executionData)`**: Similarly, if `includeTraceSpans` is true, `traceSpans` are added.
    *   `(log.executionData as any)`: Another type assertion, implying `executionData` is a flexible object.

```typescript
      // Fetch rate limits and usage data if requested
      if ((subscription.includeRateLimits || subscription.includeUsageData) && log.executionData) {
        const executionData = log.executionData as any

        const needsRateLimits = subscription.includeRateLimits && executionData.includeRateLimits
        const needsUsage = subscription.includeUsageData && executionData.includeUsageData
        if (needsRateLimits || needsUsage) {
          const { getUserLimits } = await import('@/app/api/v1/logs/meta')
          const workflow = await db
            .select()
            .from(workflowTable)
            .where(eq(workflowTable.id, log.workflowId))
            .limit(1)

          if (workflow.length > 0) {
            try {
              const limits = await getUserLimits(workflow[0].userId)
              if (needsRateLimits) {
                payload.data.rateLimits = limits.workflowExecutionRateLimit
              }
              if (needsUsage) {
                payload.data.usage = limits.usage
              }
            } catch (error) {
              logger.warn('Failed to fetch limits/usage for webhook', { error })
            }
          }
        }
      }
```

*   **Conditional Data Loading (Rate Limits & Usage):** This block dynamically fetches and includes rate limit and usage data if requested by the subscription.
    *   **`if ((subscription.includeRateLimits || subscription.includeUsageData) && log.executionData)`**: Checks if either `includeRateLimits` or `includeUsageData` is true for the subscription, and if `executionData` exists.
    *   **`const needsRateLimits = ...`, `const needsUsage = ...`**: Further refine whether specific data types are actually needed, considering both subscription preferences and if the `executionData` itself indicates inclusion.
    *   **`const { getUserLimits } = await import('@/app/api/v1/logs/meta')`**: Dynamically imports the `getUserLimits` function. Dynamic imports help keep the initial bundle size smaller and load code only when needed.
    *   **`const workflow = await db.select()...`**: Fetches the `workflow` details from the `workflowTable` based on `log.workflowId` to get the `userId`.
    *   **`if (workflow.length > 0)`**: Ensures a workflow was found.
    *   **`try { ... } catch (error) { ... }`**: Fetches user limits using `getUserLimits(workflow[0].userId)` and populates `payload.data.rateLimits` and/or `payload.data.usage` if requested. A `try/catch` block handles potential errors during limit fetching.

```typescript
      const body = JSON.stringify(payload)
      const headers: Record<string, string> = {
        'Content-Type': 'application/json',
        'sim-event': 'workflow.execution.completed',
        'sim-timestamp': timestamp.toString(),
        'sim-delivery-id': deliveryId,
        'Idempotency-Key': deliveryId,
      }

      if (subscription.secret) {
        const { decrypted } = await decryptSecret(subscription.secret)
        const signature = generateSignature(decrypted, timestamp, body)
        headers['sim-signature'] = `t=${timestamp},v1=${signature}`
      }
```

*   **`const body = JSON.stringify(payload)`**: Converts the `payload` object into a JSON string, which will be the HTTP request body.
*   **`const headers: Record<string, string> = { ... }`**: Defines the standard HTTP headers for the webhook request:
    *   `'Content-Type': 'application/json'`: Indicates the body is JSON.
    *   `'sim-event'`: A custom header identifying the event type.
    *   `'sim-timestamp'`: A custom header with the event's timestamp.
    *   `'sim-delivery-id'`: A custom header with the unique delivery ID.
    *   `'Idempotency-Key'`: Ensures that if the same request is sent multiple times due to retries, the receiving system can process it only once.
*   **`if (subscription.secret)`**: Checks if the subscription has a secret configured for signing.
    *   **`const { decrypted } = await decryptSecret(subscription.secret)`**: Decrypts the stored secret using the `decryptSecret` utility.
    *   **`const signature = generateSignature(decrypted, timestamp, body)`**: Generates the HMAC signature using the decrypted secret, timestamp, and JSON body.
    *   **`headers['sim-signature'] = `t=${timestamp},v1=${signature}``**: Adds the generated signature to the `sim-signature` custom header in a specific format (timestamp and signature version 1).

```typescript
      logger.info(`Attempting webhook delivery ${deliveryId} (attempt ${attempts})`, {
        url: subscription.url,
        executionId: log.executionId,
      })

      const controller = new AbortController()
      const timeoutId = setTimeout(() => controller.abort(), 30000)

      try {
        const response = await fetch(subscription.url, {
          method: 'POST',
          headers,
          body,
          signal: controller.signal,
        })

        clearTimeout(timeoutId)

        const responseBody = await response.text().catch(() => '')
        const truncatedBody = responseBody.slice(0, 1000)
```

*   **`logger.info(...)`**: Logs the attempt to deliver the webhook, including the URL and current attempt number.
*   **`const controller = new AbortController()`**: Creates an `AbortController` instance. This is used to cancel the `fetch` request if it takes too long.
*   **`const timeoutId = setTimeout(() => controller.abort(), 30000)`**: Sets a 30-second timeout. If the `fetch` request doesn't complete within this time, `controller.abort()` will be called, causing the `fetch` promise to reject with an `AbortError`.
*   **`try { ... }`**: A `try...catch` block specifically for the `fetch` request and its immediate response handling.
    *   **`const response = await fetch(subscription.url, { ... })`**: Makes the actual HTTP POST request to the `subscription.url` with the prepared headers and body. The `signal: controller.signal` links the request to the `AbortController`.
    *   **`clearTimeout(timeoutId)`**: If `fetch` completes (either resolves or rejects) *before* the timeout, clears the timeout to prevent `controller.abort()` from being called unnecessarily.
    *   **`const responseBody = await response.text().catch(() => '')`**: Attempts to read the response body as text. Uses `.catch(() => '')` to handle cases where reading the body might fail.
    *   **`const truncatedBody = responseBody.slice(0, 1000)`**: Truncates the response body to the first 1000 characters for storage and logging, preventing excessively large data in the database.

```typescript
        if (response.ok) {
          await db
            .update(workflowLogWebhookDelivery)
            .set({
              status: 'success',
              attempts,
              lastAttemptAt: new Date(),
              responseStatus: response.status,
              responseBody: truncatedBody,
              errorMessage: null,
              updatedAt: new Date(),
            })
            .where(
              and(
                eq(workflowLogWebhookDelivery.id, deliveryId),
                eq(workflowLogWebhookDelivery.status, 'in_progress')
              )
            )

          logger.info(`Webhook delivery ${deliveryId} succeeded`, {
            status: response.status,
            executionId: log.executionId,
          })

          return { success: true }
        }
```

*   **`if (response.ok)`**: Checks if the HTTP response status code is in the 200-299 range (indicating success).
    *   **`await db.update(...)`**: Updates the `workflowLogWebhookDelivery` record to reflect a successful delivery.
        *   `status: 'success'`: Sets the status.
        *   `attempts`, `lastAttemptAt`, `responseStatus`, `responseBody`: Stores details about the successful attempt.
        *   `errorMessage: null`: Clears any previous error message.
        *   The `where` clause (`and(eq(..., deliveryId), eq(..., 'in_progress'))`) ensures only the *currently in-progress* delivery is marked as success, preventing race conditions if another process somehow intervened.
    *   **`logger.info(...)`**: Logs the successful delivery.
    *   **`return { success: true }`**: Indicates the task completed successfully.

```typescript
        const isRetryable = response.status >= 500 || response.status === 429

        if (!isRetryable || attempts >= MAX_ATTEMPTS) {
          await db
            .update(workflowLogWebhookDelivery)
            .set({
              status: 'failed',
              attempts,
              lastAttemptAt: new Date(),
              responseStatus: response.status,
              responseBody: truncatedBody,
              errorMessage: `HTTP ${response.status}`,
              updatedAt: new Date(),
            })
            .where(
              and(
                eq(workflowLogWebhookDelivery.id, deliveryId),
                eq(workflowLogWebhookDelivery.status, 'in_progress')
              )
            )

          logger.warn(`Webhook delivery ${deliveryId} failed permanently`, {
            status: response.status,
            attempts,
            executionId: log.executionId,
          })

          return { success: false }
        }
```

*   **`const isRetryable = response.status >= 500 || response.status === 429`**: Defines what constitutes a "retryable" HTTP error. Typically, 5xx server errors and 429 Too Many Requests are considered transient and worth retrying.
*   **`if (!isRetryable || attempts >= MAX_ATTEMPTS)`**: Checks if the failure is *not* retryable (e.g., 4xx client errors like 400 Bad Request, 403 Forbidden) OR if the maximum number of retry `attempts` has been reached.
    *   **`await db.update(...)`**: Updates the delivery record to `'failed'` because no more retries will be attempted.
    *   **`logger.warn(...)`**: Logs a warning about the permanent failure.
    *   **`return { success: false }`**: Indicates the task completed with a permanent failure.

```typescript
        const baseDelay = RETRY_DELAYS[Math.min(attempts - 1, RETRY_DELAYS.length - 1)]
        const delayWithJitter = getRetryDelayWithJitter(baseDelay)
        const nextAttemptAt = new Date(Date.now() + delayWithJitter)

        await db
          .update(workflowLogWebhookDelivery)
          .set({
            status: 'pending',
            attempts,
            lastAttemptAt: new Date(),
            nextAttemptAt,
            responseStatus: response.status,
            responseBody: truncatedBody,
            errorMessage: `HTTP ${response.status} - will retry`,
            updatedAt: new Date(),
          })
          .where(
            and(
              eq(workflowLogWebhookDelivery.id, deliveryId),
              eq(workflowLogWebhookDelivery.status, 'in_progress')
            )
          )

        // Schedule the next retry
        await wait.for({ seconds: delayWithJitter / 1000 })

        // Recursively call the task for retry
        await logsWebhookDelivery.trigger({
          deliveryId,
          subscriptionId,
          log,
        })

        return { success: false, retrying: true }
```

This block handles a *retryable* failure where `MAX_ATTEMPTS` has not been reached.

*   **`const baseDelay = RETRY_DELAYS[Math.min(attempts - 1, RETRY_DELAYS.length - 1)]`**: Calculates the base delay for the *next* retry from the `RETRY_DELAYS` array. `Math.min` ensures we don't go out of bounds of the array if `attempts` exceeds the array's length (it will use the last, longest delay). `attempts - 1` is used because `attempts` is 1-indexed (e.g., first retry is `attempts = 1`, index `0`).
*   **`const delayWithJitter = getRetryDelayWithJitter(baseDelay)`**: Applies jitter to the base delay.
*   **`const nextAttemptAt = new Date(Date.now() + delayWithJitter)`**: Calculates the exact timestamp for the next retry.
*   **`await db.update(...)`**: Updates the `workflowLogWebhookDelivery` record to `pending` again, but this time setting `nextAttemptAt` to schedule the next attempt. It also records the `responseStatus`, `responseBody`, and a descriptive `errorMessage`.
*   **`await wait.for({ seconds: delayWithJitter / 1000 })`**: This is a Trigger.dev specific function that pauses the execution of *this specific task instance* for the calculated delay. Critically, Trigger.dev ensures this pause is resilient to server restarts.
*   **`await logsWebhookDelivery.trigger({ ... })`**: This is the core of the *manual retry* mechanism. Instead of letting Trigger.dev automatically retry, we explicitly trigger *another instance* of this `logsWebhookDelivery` task with the *same parameters*. This new task instance will pick up the `pending` delivery after `nextAttemptAt` has passed. This effectively re-queues the delivery with all its context.
*   **`return { success: false, retrying: true }`**: Indicates the task is not yet successful and is scheduled for a retry.

```typescript
      } catch (error: any) {
        clearTimeout(timeoutId)

        if (error.name === 'AbortError') {
          logger.error(`Webhook delivery ${deliveryId} timed out`, {
            executionId: log.executionId,
            attempts,
          })
          error.message = 'Request timeout after 30 seconds'
        }

        const baseDelay = RETRY_DELAYS[Math.min(attempts - 1, RETRY_DELAYS.length - 1)]
        const delayWithJitter = getRetryDelayWithJitter(baseDelay)
        const nextAttemptAt = new Date(Date.now() + delayWithJitter)

        await db
          .update(workflowLogWebhookDelivery)
          .set({
            status: attempts >= MAX_ATTEMPTS ? 'failed' : 'pending',
            attempts,
            lastAttemptAt: new Date(),
            nextAttemptAt: attempts >= MAX_ATTEMPTS ? null : nextAttemptAt,
            errorMessage: error.message,
            updatedAt: new Date(),
          })
          .where(
            and(
              eq(workflowLogWebhookDelivery.id, deliveryId),
              eq(workflowLogWebhookDelivery.status, 'in_progress')
            )
          )

        if (attempts >= MAX_ATTEMPTS) {
          logger.error(`Webhook delivery ${deliveryId} failed after ${attempts} attempts`, {
            error: error.message,
            executionId: log.executionId,
          })
          return { success: false }
        }

        // Schedule the next retry
        await wait.for({ seconds: delayWithJitter / 1000 })

        // Recursively call the task for retry
        await logsWebhookDelivery.trigger({
          deliveryId,
          subscriptionId,
          log,
        })

        return { success: false, retrying: true }
      }
    } catch (error: any) {
      logger.error(`Webhook delivery ${deliveryId} encountered unexpected error`, {
        error: error.message,
        stack: error.stack,
      })

      // Mark as failed for unexpected errors
      await db
        .update(workflowLogWebhookDelivery)
        .set({
          status: 'failed',
          errorMessage: `Unexpected error: ${error.message}`,
          updatedAt: new Date(),
        })
        .where(eq(workflowLogWebhookDelivery.id, deliveryId))

      return { success: false, error: error.message }
    }
  },
})
```

*   **`catch (error: any)` (Inner `try...catch` for `fetch`):** This block handles errors that occur during the `fetch` request itself (e.g., network issues, DNS resolution failure, or the `AbortError` from the timeout).
    *   **`clearTimeout(timeoutId)`**: Ensures the timeout is cleared.
    *   **`if (error.name === 'AbortError') { ... }`**: Specifically checks for a timeout error and updates its message for clarity in logs.
    *   The rest of this block (calculating `baseDelay`, `delayWithJitter`, `nextAttemptAt`, updating the database, `wait.for`, and `logsWebhookDelivery.trigger`) is almost identical to the HTTP error retry logic. It determines if the delivery should be marked `failed` (if `MAX_ATTEMPTS` is reached) or `pending` for a retry.
*   **`} catch (error: any)` (Outer `try...catch` for `run` function):** This is the ultimate fallback error handler. It catches any unexpected error that might have occurred *anywhere* within the `run` function, outside the specific `fetch` block.
    *   **`logger.error(...)`**: Logs the unexpected error, including the stack trace for debugging.
    *   **`await db.update(...)`**: Marks the delivery as `'failed'` in the database with an "Unexpected error" message. Since this error is outside the controlled retry flow, it's treated as a permanent failure.
    *   **`return { success: false, error: error.message }`**: Returns failure.

---

### Simplifying Complex Logic

1.  **Atomic Claiming of Delivery Rows:**
    *   **Complexity:** In a distributed system, multiple workers might try to process the same webhook delivery simultaneously. Without careful handling, this could lead to duplicate webhook sends or inconsistent state.
    *   **Simplification:** The code uses a clever database `UPDATE ... WHERE` clause that acts like a "lock." It tries to update the delivery status from `'pending'` to `'in_progress'` *and* increments the `attempts` count *only if* it's currently `pending` and its scheduled `nextAttemptAt` time has passed. If this update successfully changes a row (meaning `claimed.length > 0`), that worker "owns" the delivery. If no row is updated, another worker beat it, or it wasn't ready, so it safely exits. This ensures only one worker processes a delivery at a time.

2.  **Custom Recursive Retry Mechanism:**
    *   **Complexity:** Implementing robust retries with exponential backoff and jitter, distinguishing between retryable and non-retryable errors, and managing delivery state in the database can be intricate.
    *   **Simplification:** The code leverages Trigger.dev's `wait.for` and `trigger` capabilities to create an elegant, resilient custom retry loop.
        *   When a retryable error occurs, the task updates the database to `pending` with a `nextAttemptAt` timestamp.
        *   It then uses `wait.for` to pause its *current* execution for the calculated delay.
        *   Finally, it calls `logsWebhookDelivery.trigger()` to effectively re-queue *itself* as a *new* Trigger.dev task with the same parameters.
        *   This new task will eventually be picked up by a worker *after* the `nextAttemptAt` time has passed (due to the atomic claim logic), continuing the retry sequence. This pattern offloads the long-term waiting to Trigger.dev's robust infrastructure while maintaining fine-grained control over the retry logic.

3.  **Dynamic Payload Construction:**
    *   **Complexity:** Webhook payloads can vary based on user preferences (e.g., include final output, trace spans, rate limits).
    *   **Simplification:** The code clearly separates the core payload structure from optional fields. It uses `if (subscription.someOption && log.someData)` blocks to conditionally add data, and even dynamically imports modules (`await import(...)`) for fetching more complex data like rate limits only when needed. This keeps the core payload construction clean and ensures resources are only used when necessary.

---

This file demonstrates a professional and resilient approach to webhook delivery, combining security best practices, robust error handling, and a sophisticated retry strategy within a modern background job framework.