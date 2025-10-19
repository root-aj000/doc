```typescript
import { db } from '@sim/db'
import { workflowLogWebhook, workflowLogWebhookDelivery } from '@sim/db/schema'
import { and, eq } from 'drizzle-orm'
import { v4 as uuidv4 } from 'uuid'
import { createLogger } from '@/lib/logs/console/logger'
import type { WorkflowExecutionLog } from '@/lib/logs/types'
import { logsWebhookDelivery } from '@/background/logs-webhook-delivery'

const logger = createLogger('LogsEventEmitter')

/**
 * Emits a workflow execution completed event, triggering webhook deliveries based on subscriptions.
 * @param log The workflow execution log containing details of the completed execution.
 * @returns A promise that resolves when the event emission process is complete.
 */
export async function emitWorkflowExecutionCompleted(log: WorkflowExecutionLog): Promise<void> {
  try {
    // 1. Retrieve active webhook subscriptions for the given workflow.
    const subscriptions = await db
      .select()
      .from(workflowLogWebhook)
      .where(
        and(eq(workflowLogWebhook.workflowId, log.workflowId), eq(workflowLogWebhook.active, true))
      )

    // 2. If no active subscriptions are found, exit the function.
    if (subscriptions.length === 0) {
      return
    }

    logger.debug(
      `Found ${subscriptions.length} active webhook subscriptions for workflow ${log.workflowId}`
    )

    // 3. Iterate through each active subscription.
    for (const subscription of subscriptions) {
      // 4. Check if the log's level and trigger match the subscription's filters.
      const levelMatches = subscription.levelFilter?.includes(log.level) ?? true
      const triggerMatches = subscription.triggerFilter?.includes(log.trigger) ?? true

      // 5. If either the level or trigger doesn't match, skip to the next subscription.
      if (!levelMatches || !triggerMatches) {
        logger.debug(`Skipping subscription ${subscription.id} due to filter mismatch`, {
          level: log.level,
          trigger: log.trigger,
          levelFilter: subscription.levelFilter,
          triggerFilter: subscription.triggerFilter,
        })
        continue
      }

      // 6. Generate a unique ID for the webhook delivery.
      const deliveryId = uuidv4()

      // 7. Insert a new record into the workflowLogWebhookDelivery table to track the delivery attempt.
      await db.insert(workflowLogWebhookDelivery).values({
        id: deliveryId,
        subscriptionId: subscription.id,
        workflowId: log.workflowId,
        executionId: log.executionId,
        status: 'pending',
        attempts: 0,
        nextAttemptAt: new Date(),
      })

      // 8. Prepare the log data to be sent in the webhook.
      // Start with a copy of the original log, but clear the `executionData` initially.
      const webhookLog = {
        ...log,
        executionData: {},
      }

      // 9. Conditionally include specific fields from `executionData` based on the subscription settings.
      if (log.executionData) {
        const data = log.executionData as any // Cast to `any` to allow flexible access to properties.
        const webhookData: any = {} // Object to hold the data that will be included in the webhook.

        // Check if the subscription wants to include the final output.
        if (subscription.includeFinalOutput && data.finalOutput) {
          webhookData.finalOutput = data.finalOutput
        }

        // Check if the subscription wants to include trace spans.
        if (subscription.includeTraceSpans && data.traceSpans) {
          webhookData.traceSpans = data.traceSpans
        }

        // Indicate that rate limits should be fetched and included in the delivery.
        // The actual fetching will be done in the webhook delivery process.
        if (subscription.includeRateLimits) {
          webhookData.includeRateLimits = true
        }

        // Indicate that usage data should be fetched and included in the delivery.
        // The actual fetching will be done in the webhook delivery process.
        if (subscription.includeUsageData) {
          webhookData.includeUsageData = true
        }

        // Assign the prepared `webhookData` to the `executionData` field of the log that will be sent.
        webhookLog.executionData = webhookData
      }

      // 10. Trigger the webhook delivery process in the background.
      await logsWebhookDelivery.trigger({
        deliveryId,
        subscriptionId: subscription.id,
        log: webhookLog,
      })

      logger.info(`Enqueued webhook delivery ${deliveryId} for subscription ${subscription.id}`)
    }
  } catch (error) {
    // 11. Catch any errors that occur during the process and log them.
    logger.error('Failed to emit workflow execution completed event', {
      error,
      workflowId: log.workflowId,
      executionId: log.executionId,
    })
  }
}
```

### Purpose of this file:

This file contains the logic for emitting a `workflow execution completed` event.  When a workflow finishes, this function is called to:

1.  Find relevant webhook subscriptions.
2.  Filter subscriptions based on configured level and trigger filters.
3.  Create a delivery record in the database.
4.  Prepare the log data (selectively including execution data based on subscription settings).
5.  Trigger a background process (`logsWebhookDelivery`) to actually send the webhook.

In essence, it orchestrates the process of notifying subscribers (via webhooks) about completed workflow executions, applying filters and managing the delivery process.

### Simplification and Explanation:

The code is already reasonably well-structured, but here's a breakdown with simplification and more explicit explanations:

**Imports:**

*   `db`: An object representing the database connection (presumably using Drizzle ORM, as indicated by the other imports).
*   `workflowLogWebhook`, `workflowLogWebhookDelivery`: Database schema definitions for the `workflow_log_webhook` and `workflow_log_webhook_delivery` tables, respectively.  These likely define the structure and columns of those tables.
*   `and`, `eq`: Functions from Drizzle ORM for building SQL `AND` and `EQUALS` conditions in database queries.
*   `uuidv4`: A function from the `uuid` library for generating version 4 UUIDs (Universally Unique Identifiers). Used to uniquely identify each webhook delivery.
*   `createLogger`: A function for creating a logger instance (likely using a custom logging library). This provides a consistent way to log messages throughout the application.
*   `WorkflowExecutionLog`: A type definition for the `log` object passed to the `emitWorkflowExecutionCompleted` function.  This interface describes the structure of the workflow execution log data.
*   `logsWebhookDelivery`: An object representing a background task or queue for handling the actual delivery of webhooks.  This likely uses a queueing system (like Redis or RabbitMQ) to ensure reliable delivery, even under heavy load.

**Logger:**

*   `const logger = createLogger('LogsEventEmitter')`: Creates a logger instance specifically for this module, making it easier to identify log messages originating from this file.

**`emitWorkflowExecutionCompleted` function:**

This is the core of the file. Let's break it down step by step:

1.  **`export async function emitWorkflowExecutionCompleted(log: WorkflowExecutionLog): Promise<void> {`**:
    *   This defines an asynchronous function named `emitWorkflowExecutionCompleted` that takes a `WorkflowExecutionLog` object as input and returns a `Promise` that resolves to `void` (meaning it doesn't return a value).  The `async` keyword allows the use of `await` inside the function.
    *   The `WorkflowExecutionLog` type enforces the structure of the `log` object, ensuring that it contains the necessary information about the completed workflow execution (e.g., `workflowId`, `executionId`, `level`, `trigger`, `executionData`).

2.  **`try { ... } catch (error) { ... }`**:
    *   This is a standard `try...catch` block for error handling. Any code within the `try` block that throws an error will be caught by the `catch` block.

3.  **`const subscriptions = await db ...`**:
    *   This is the database query to find active webhook subscriptions that match the workflow ID.
    *   `db.select().from(workflowLogWebhook)`:  This initiates a `SELECT` query on the `workflowLogWebhook` table.  It's selecting all columns (`select()`).
    *   `.where(and(eq(workflowLogWebhook.workflowId, log.workflowId), eq(workflowLogWebhook.active, true)))`: This adds a `WHERE` clause to the query to filter the subscriptions.
        *   `and(...)`:  Ensures that *both* conditions within the parentheses must be true.
        *   `eq(workflowLogWebhook.workflowId, log.workflowId)`: Checks if the `workflowId` column in the `workflowLogWebhook` table is equal to the `workflowId` property of the `log` object.
        *   `eq(workflowLogWebhook.active, true)`: Checks if the `active` column in the `workflowLogWebhook` table is equal to `true`. This ensures that only active subscriptions are retrieved.
    *   `await`: The `await` keyword pauses the execution of the function until the database query completes and returns the results.  The `subscriptions` variable will then contain an array of `workflowLogWebhook` objects that match the criteria.

4.  **`if (subscriptions.length === 0) { return }`**:
    *   This checks if any active subscriptions were found for the workflow. If not, the function returns early, as there's nothing to do.

5.  **`logger.debug(...)`**:
    *   This logs a debug message indicating the number of active subscriptions found for the workflow.  Debug messages are typically used for development and troubleshooting and are not usually enabled in production environments.

6.  **`for (const subscription of subscriptions) { ... }`**:
    *   This loop iterates through each of the active webhook subscriptions that were found.

7.  **`const levelMatches = subscription.levelFilter?.includes(log.level) ?? true`**:
    *   This line determines whether the log's level matches the subscription's configured level filter.  Let's break it down:
        *   `subscription.levelFilter?`:  This uses the optional chaining operator (`?.`).  If `subscription.levelFilter` is `null` or `undefined`, the entire expression will evaluate to `undefined`.
        *   `.includes(log.level)`: If `subscription.levelFilter` is an array (presumably an array of allowed log levels), this checks if the `log.level` is present in that array.
        *   `?? true`:  This is the nullish coalescing operator (`??`).  If the expression on the left-hand side (`subscription.levelFilter?.includes(log.level)`) evaluates to `null` or `undefined` (which would happen if `subscription.levelFilter` is `null` or `undefined`), then the expression on the right-hand side (`true`) is used.  This means that if no level filter is specified in the subscription, *all* levels are considered to match.

8.  **`const triggerMatches = subscription.triggerFilter?.includes(log.trigger) ?? true`**:
    *   This line works the same way as the `levelMatches` line, but it checks if the log's trigger matches the subscription's configured trigger filter.

9.  **`if (!levelMatches || !triggerMatches) { ... }`**:
    *   This checks if either the level or the trigger *doesn't* match the subscription's filters. If either one doesn't match, the code inside the `if` block is executed, which skips to the next subscription.

10. **`logger.debug(...)`**:
    *   This logs a debug message indicating that a subscription is being skipped due to a filter mismatch.

11. **`continue`**:
    *   This keyword skips the rest of the current iteration of the `for` loop and moves on to the next subscription.

12. **`const deliveryId = uuidv4()`**:
    *   This generates a unique ID for the webhook delivery using the `uuidv4` function.

13. **`await db.insert(workflowLogWebhookDelivery).values({ ... })`**:
    *   This inserts a new record into the `workflowLogWebhookDelivery` table to track the delivery attempt.
    *   `db.insert(workflowLogWebhookDelivery)`:  This initiates an `INSERT` query on the `workflowLogWebhookDelivery` table.
    *   `.values({ ... })`: This specifies the values to be inserted into the new row.
        *   `id: deliveryId`: The unique ID generated for this delivery.
        *   `subscriptionId: subscription.id`: The ID of the subscription that triggered this delivery.
        *   `workflowId: log.workflowId`: The ID of the workflow that generated the log.
        *   `executionId: log.executionId`: The ID of the specific workflow execution.
        *   `status: 'pending'`: The initial status of the delivery (pending, as it hasn't been sent yet).
        *   `attempts: 0`: The number of delivery attempts so far (0 for the initial attempt).
        *   `nextAttemptAt: new Date()`: The timestamp for the next delivery attempt.  Initially set to the current time, meaning the delivery should be attempted immediately.
    *   `await`: The `await` keyword pauses the execution of the function until the database insertion completes.

14. **`const webhookLog = { ...log, executionData: {} }`**:
    *   This creates a new object `webhookLog` that is a shallow copy of the original `log` object, but with the `executionData` property explicitly set to an empty object.  This is done to avoid accidentally including sensitive or unnecessary data in the webhook payload if the subscription doesn't request it.
    *   The spread operator (`...log`) copies all properties from the original `log` object.

15. **`if (log.executionData) { ... }`**:
    *   This checks if the original `log` object contains any `executionData`. If not, the code inside the `if` block is skipped.

16. **`const data = log.executionData as any`**:
    *  This line casts log.executionData to 'any', which allows the code to access properties without strict type checking. While this offers flexibility, it also reduces type safety.  It's done here to handle potentially varying structures of `executionData`.

17. **`const webhookData: any = {}`**:
    *  This creates an empty object `webhookData` that will hold the data that is included in the webhook payload.

18. **Conditionally adding data based on `subscription` options:**
    * The `if (subscription.includeFinalOutput && data.finalOutput) { ... }` block, `if (subscription.includeTraceSpans && data.traceSpans) { ... }` block, `if (subscription.includeRateLimits) { ... }` block, and `if (subscription.includeUsageData) { ... }` block are the meat of the operation.
    *   They conditionally add properties to the `webhookData` object based on the settings configured in the `subscription` object.
    *   For example, `if (subscription.includeFinalOutput && data.finalOutput)` checks if the subscription wants to include the final output of the workflow execution *and* if the `executionData` actually contains a `finalOutput` property. If both conditions are true, the `finalOutput` property is copied from `data` to `webhookData`.
    *   The same logic applies to `traceSpans`, `includeRateLimits`, and `includeUsageData`.

19. **`webhookLog.executionData = webhookData`**:
    *   After selectively adding data to the `webhookData` object, this line assigns the `webhookData` object to the `executionData` property of the `webhookLog` object.  This is the data that will be sent in the webhook.

20. **`await logsWebhookDelivery.trigger({ ... })`**:
    *   This triggers the background task or queue to actually deliver the webhook.
    *   `logsWebhookDelivery.trigger(...)`:  This calls a method (presumably named `trigger`) on the `logsWebhookDelivery` object to initiate the webhook delivery process.  The argument to `trigger` is an object containing the necessary information for the delivery.
        *   `deliveryId`: The unique ID of the delivery.
        *   `subscriptionId: subscription.id`: The ID of the subscription.
        *   `log: webhookLog`: The prepared log data (including the selectively included `executionData`).
    *   `await`: The `await` keyword pauses the execution of the function until the `logsWebhookDelivery.trigger` method completes. This doesn't necessarily mean that the webhook has been successfully delivered; it simply means that the delivery process has been initiated.

21. **`logger.info(...)`**:
    *   This logs an informational message indicating that a webhook delivery has been enqueued.

22. **`catch (error) { ... }`**:
    *   If any error occurs within the `try` block, this `catch` block will be executed.
    *   `logger.error(...)`: This logs an error message, including the error object itself, the workflow ID, and the execution ID. This is crucial for debugging and identifying issues with the webhook emission process.

**In summary,** this function intelligently and selectively distributes workflow execution logs to subscribers via webhooks, respecting their configured filters and data preferences, while also ensuring robust error handling and logging. The use of a background task (`logsWebhookDelivery`) ensures that the webhook delivery process doesn't block the main application thread and that deliveries are retried in case of failure. The detailed logging throughout the function makes it easier to troubleshoot any issues that may arise.
