This TypeScript file defines a Next.js API route (`GET` handler) designed to perform a crucial maintenance task: **cleaning up old execution logs for users on the free plan**. It operates as a scheduled cron job, identifying and archiving these logs to AWS S3 before deleting them from the primary database, along with any associated files and orphaned snapshots.

## Detailed Explanation

### Purpose of this file and what it does

This file (`src/app/api/internal/logs-cleanup/route.ts`) acts as a backend endpoint, specifically for internal use (likely triggered by a cron service like AWS EventBridge, Vercel Cron Jobs, or similar). Its primary responsibilities are:

1.  **Identify Free Users:** It first determines which users are on the free plan by checking their subscription status.
2.  **Identify Old Logs:** For these free users, it then finds `workflowExecutionLogs` (detailed records of workflow runs) that are older than a specified retention period (default 7 days).
3.  **Archive to S3:** Before deleting, it archives the full log data to an AWS S3 bucket. This serves as a long-term, cost-effective storage for historical data, especially for enhanced logs that might contain more detail than a standard log.
4.  **Delete Associated Files:** If a log has associated files (e.g., input/output data) stored in cloud storage, it attempts to delete these files from the storage service.
5.  **Delete from Database:** After successful archiving, it deletes the log entries from the `workflowExecutionLogs` table in the Drizzle database.
6.  **Cleanup Orphaned Snapshots:** It also cleans up any `stateSnapshots` (records of a workflow's state at a point in time) that are no longer referenced or are older than a specific retention period, preventing storage bloat.
7.  **Batch Processing:** To handle potentially large volumes of logs efficiently and prevent timeouts, it processes logs in batches.
8.  **Security:** It includes an authentication mechanism (`verifyCronAuth`) to ensure only authorized internal services can trigger this cleanup.

In essence, this script ensures that free users' old, detailed logs are moved to cheaper storage (S3) and then removed from the operational database to manage database size, improve performance, and adhere to data retention policies, while still maintaining a historical record.

### Simplify complex logic

The core logic can be broken down into these steps:

1.  **Preparation:**
    *   Verify the request is authorized (it's a cron job, not a public API).
    *   Check if S3 configuration (bucket, region) is valid.
    *   Calculate the "retention date": any log created *before* this date is considered old and eligible for cleanup. This date is usually "today minus X days".

2.  **Target Identification:**
    *   Find all `userId`s that do *not* have an active 'pro', 'team', or 'enterprise' subscription. These are our "free users."
    *   Find all `workflowId`s belonging to these identified free users. Logs for these workflows are the target.

3.  **Iterative Log Cleanup (Batch by Batch):**
    *   Initialize a `results` object to keep track of how many logs were archived, deleted, failed, etc.
    *   Start a loop that continues as long as there are more logs to process and a maximum number of batches hasn't been reached.
    *   **Inside the loop:**
        *   **Fetch a batch:** Query the database for `BATCH_SIZE` (e.g., 2000) old logs belonging to the target `workflowIds` and created before the `retentionDate`.
        *   **Process each log in the batch:**
            *   **Archive:** Take the log's data, convert it to JSON, and upload it as a file to a specific folder in the S3 bucket. This folder structure is `archived-enhanced-logs/YYYY-MM-DD/logId.json`.
            *   **Delete Files:** If the log contains references to other files (like images, documents, etc.) stored in cloud storage, delete those files.
            *   **Delete from DB:** Once successfully archived and files are deleted, remove the log entry from the `workflowExecutionLogs` table.
            *   **Track Outcomes:** Update the `results` object for each step (archived, deleted, failed).
        *   **Check for more:** If the fetched batch was full (`BATCH_SIZE`), it means there might be more logs, so the loop continues. Otherwise, if a partial batch was fetched, it means all eligible logs have been processed.

4.  **Snapshot Cleanup:**
    *   After the main log cleanup, a separate service (`snapshotService`) is called to clean up any orphaned or expired workflow state snapshots.

5.  **Reporting:**
    *   Finally, return a JSON response summarizing the cleanup operation, including total logs processed, files deleted, time taken, and whether all eligible logs were processed or if a batch limit was reached.

### Explain each line of code deep

```typescript
import { PutObjectCommand } from '@aws-sdk/client-s3'
// Dynamic import for S3 client to avoid client-side bundling
import { db } from '@sim/db'
import { subscription, user, workflow, workflowExecutionLogs } from '@sim/db/schema'
import { and, eq, inArray, lt, sql } from 'drizzle-orm'
import { type NextRequest, NextResponse } from 'next/server'
import { verifyCronAuth } from '@/lib/auth/internal'
import { env } from '@/lib/env'
import { createLogger } from '@/lib/logs/console/logger'
import { snapshotService } from '@/lib/logs/execution/snapshot/service'
import { deleteFile, isUsingCloudStorage } from '@/lib/uploads'
```
These lines import necessary modules and functions from various libraries and internal utilities:
-   `PutObjectCommand` from `@aws-sdk/client-s3`: This is a specific command used to upload an object (a file or string data) to an Amazon S3 bucket.
-   `db` from `@sim/db`: This imports the Drizzle ORM database client instance, configured to interact with the application's database.
-   `subscription`, `user`, `workflow`, `workflowExecutionLogs` from `@sim/db/schema`: These are Drizzle ORM schema definitions for the corresponding database tables. They represent the structure of these tables and allow for type-safe queries.
-   `and`, `eq`, `inArray`, `lt`, `sql` from `drizzle-orm`: These are Drizzle ORM utility functions used to construct SQL queries:
    -   `and`: Combines multiple conditions with a logical AND.
    -   `eq`: Checks for equality (e.g., `column = value`).
    -   `inArray`: Checks if a column's value is present in a given array (e.g., `column IN (value1, value2)`).
    -   `lt`: Checks for "less than" (e.g., `column < value`).
    -   `sql`: Allows embedding raw SQL expressions directly into Drizzle queries.
-   `type NextRequest, NextResponse` from `next/server`: These are Next.js types and classes for handling incoming HTTP requests and sending HTTP responses in API routes.
-   `verifyCronAuth` from `@/lib/auth/internal`: An internal utility function used to authenticate requests specifically from cron jobs or other internal services, ensuring only authorized callers can trigger this endpoint.
-   `env` from `@/lib/env`: An object containing environment variables, securely loaded and typed. Used here for S3 bucket name, region, and log retention days.
-   `createLogger` from `@/lib/logs/console/logger`: An internal utility to create a structured logger instance for this specific module, helping with debugging and monitoring.
-   `snapshotService` from `@/lib/logs/execution/snapshot/service`: An internal service responsible for managing workflow state snapshots, specifically for cleaning up old or orphaned ones.
-   `deleteFile`, `isUsingCloudStorage` from `@/lib/uploads`: Internal utilities related to file management:
    -   `deleteFile`: Deletes a file from the configured cloud storage.
    -   `isUsingCloudStorage`: Checks if the application is currently configured to use cloud storage for uploads (e.g., S3) rather than local storage.

```typescript
export const dynamic = 'force-dynamic'
```
This line is a Next.js specific configuration. `export const dynamic = 'force-dynamic'` forces this API route to be rendered dynamically on every request, rather than being statically optimized or cached at build time. This is essential for a cron job that needs to execute fresh logic and database queries each time it runs.

```typescript
const logger = createLogger('LogsCleanupAPI')
```
Initializes a logger instance named `LogsCleanupAPI`. This logger will be used throughout the file to output informational messages, warnings, and errors to the console (or configured logging destination), aiding in monitoring and debugging.

```typescript
const BATCH_SIZE = 2000
const S3_CONFIG = {
  bucket: env.S3_LOGS_BUCKET_NAME || '',
  region: env.AWS_REGION || '',
}
```
-   `BATCH_SIZE = 2000`: Defines a constant for the number of logs to process in a single database query and iteration. This helps manage memory usage and execution time, preventing potential timeouts when dealing with large datasets.
-   `S3_CONFIG`: An object that holds the AWS S3 configuration details.
    -   `bucket: env.S3_LOGS_BUCKET_NAME || ''`: Retrieves the S3 bucket name for storing logs from environment variables. If `S3_LOGS_BUCKET_NAME` is not set, it defaults to an empty string.
    -   `region: env.AWS_REGION || ''`: Retrieves the AWS region from environment variables. If `AWS_REGION` is not set, it defaults to an empty string.

```typescript
export async function GET(request: NextRequest) {
  try {
    const authError = verifyCronAuth(request, 'logs cleanup')
    if (authError) {
      return authError
    }
```
-   `export async function GET(request: NextRequest)`: This defines the main asynchronous GET request handler for this Next.js API route. `request: NextRequest` provides access to the incoming HTTP request details.
-   `try { ... } catch (error) { ... }`: A standard try-catch block to handle any unexpected errors that might occur during the execution of the cleanup process, preventing the server from crashing and providing a graceful error response.
-   `const authError = verifyCronAuth(request, 'logs cleanup')`: Calls the internal authentication utility to verify if the incoming request is authorized to trigger this cron job. The second argument `'logs cleanup'` is a descriptive name for the operation.
-   `if (authError) { return authError }`: If `verifyCronAuth` returns an `authError` (which is typically a `NextResponse` indicating an authentication failure), the function immediately returns that error response, preventing unauthorized access.

```typescript
    if (!S3_CONFIG.bucket || !S3_CONFIG.region) {
      return new NextResponse('Configuration error: S3 bucket or region not set', { status: 500 })
    }
```
Checks if the required S3 configuration values (bucket name and region) were successfully loaded from environment variables. If either is missing, it returns a 500 Internal Server Error response, as the archiving process cannot proceed without them.

```typescript
    const retentionDate = new Date()
    retentionDate.setDate(retentionDate.getDate() - Number(env.FREE_PLAN_LOG_RETENTION_DAYS || '7'))
```
-   `const retentionDate = new Date()`: Creates a `Date` object representing the current date and time.
-   `retentionDate.setDate(retentionDate.getDate() - Number(env.FREE_PLAN_LOG_RETENTION_DAYS || '7'))`: Modifies the `retentionDate` to be `X` days in the past. `env.FREE_PLAN_LOG_RETENTION_DAYS` is an environment variable specifying the retention period for free plan logs (e.g., 7 days). `Number(...)` converts the string env var to a number, defaulting to `7` if the variable is not set. Logs created *before* this calculated `retentionDate` will be targeted for cleanup.

```typescript
    const freeUsers = await db
      .select({ userId: user.id })
      .from(user)
      .leftJoin(
        subscription,
        sql`${user.id} = ${subscription.referenceId} AND ${subscription.status} = 'active' AND ${subscription.plan} IN ('pro', 'team', 'enterprise')`
      )
      .where(sql`${subscription.id} IS NULL`)
```
This is a Drizzle ORM query to identify users on the free plan:
-   `await db.select({ userId: user.id })`: Selects only the `id` column from the `user` table, aliasing it as `userId`.
-   `.from(user)`: Specifies that the query starts from the `user` table.
-   `.leftJoin(...)`: Performs a `LEFT JOIN` with the `subscription` table. A `LEFT JOIN` returns all rows from the `user` table, and the matching rows from `subscription`. If no match, the `subscription` columns will be `NULL`.
    -   `sql`${user.id} = ${subscription.referenceId} AND ${subscription.status} = 'active' AND ${subscription.plan} IN ('pro', 'team', 'enterprise')``: This is the join condition. It links `user.id` to `subscription.referenceId` (assuming `referenceId` links to `userId`), *but only for active subscriptions on specific paid plans*. The `sql` template literal allows embedding raw SQL for complex conditions.
-   `.where(sql`${subscription.id} IS NULL`)`: This is the crucial part for identifying free users. Because it's a `LEFT JOIN`, if a user has *no* active 'pro', 'team', or 'enterprise' subscription matching the join condition, then `subscription.id` for that user will be `NULL`. Thus, this `WHERE` clause filters for users who did not have a matching paid subscription.

```typescript
    if (freeUsers.length === 0) {
      logger.info('No free users found for log cleanup')
      return NextResponse.json({ message: 'No free users found for cleanup' })
    }

    const freeUserIds = freeUsers.map((u) => u.userId)
```
-   Checks if any free users were found. If `freeUsers` array is empty, it logs an informational message and returns an empty success response, as there's nothing to clean up.
-   `const freeUserIds = freeUsers.map((u) => u.userId)`: If free users are found, this line extracts just their `userId`s into a new array. This array will be used to filter workflows.

```typescript
    const workflowsQuery = await db
      .select({ id: workflow.id })
      .from(workflow)
      .where(inArray(workflow.userId, freeUserIds))

    if (workflowsQuery.length === 0) {
      logger.info('No workflows found for free users')
      return NextResponse.json({ message: 'No workflows found for cleanup' })
    }

    const workflowIds = workflowsQuery.map((w) => w.id)
```
-   `const workflowsQuery = await db.select({ id: workflow.id }).from(workflow).where(inArray(workflow.userId, freeUserIds))`: Queries the `workflow` table to find all workflow IDs (`workflow.id`) that belong to the `freeUserIds` identified earlier. `inArray` is a Drizzle helper for SQL `IN` clause.
-   Checks if any workflows were found for these free users. If not, it logs an informational message and returns an empty success response.
-   `const workflowIds = workflowsQuery.map((w) => w.id)`: Extracts just the `id`s of the found workflows into a new array. This array will be used to filter `workflowExecutionLogs`.

```typescript
    const results = {
      enhancedLogs: {
        total: 0,
        archived: 0,
        archiveFailed: 0,
        deleted: 0,
        deleteFailed: 0,
      },
      files: {
        total: 0,
        deleted: 0,
        deleteFailed: 0,
      },
      snapshots: {
        cleaned: 0,
        cleanupFailed: 0,
      },
    }
```
Initializes a `results` object. This object will accumulate statistics about the cleanup operation across different categories (enhanced logs, associated files, snapshots). It's used to provide a summary in the final response.

```typescript
    const startTime = Date.now()
    const MAX_BATCHES = 10

    // Process enhanced logging cleanup
    let batchesProcessed = 0
    let hasMoreLogs = true

    logger.info(`Starting enhanced logs cleanup for ${workflowIds.length} workflows`)
```
-   `const startTime = Date.now()`: Records the start time to calculate the total execution duration later.
-   `const MAX_BATCHES = 10`: Sets a limit on the number of batches to process in a single cron job run. This prevents a single long-running cleanup from monopolizing resources or exceeding time limits, especially for very large datasets. The cron job can then be scheduled to run periodically to pick up remaining batches.
-   `batchesProcessed = 0`: Counter for the number of batches processed so far.
-   `hasMoreLogs = true`: A flag that controls the main cleanup loop. It's set to `true` initially to ensure the loop runs at least once.
-   `logger.info(...)`: Logs a message indicating the start of the enhanced logs cleanup.

```typescript
    while (hasMoreLogs && batchesProcessed < MAX_BATCHES) {
      // Query enhanced execution logs that need cleanup
      const oldEnhancedLogs = await db
        .select({
          id: workflowExecutionLogs.id,
          workflowId: workflowExecutionLogs.workflowId,
          executionId: workflowExecutionLogs.executionId,
          stateSnapshotId: workflowExecutionLogs.stateSnapshotId,
          level: workflowExecutionLogs.level,
          trigger: workflowExecutionLogs.trigger,
          startedAt: workflowExecutionLogs.startedAt,
          endedAt: workflowExecutionLogs.endedAt,
          totalDurationMs: workflowExecutionLogs.totalDurationMs,
          executionData: workflowExecutionLogs.executionData,
          cost: workflowExecutionLogs.cost,
          files: workflowExecutionLogs.files,
          createdAt: workflowExecutionLogs.createdAt,
        })
        .from(workflowExecutionLogs)
        .where(
          and(
            inArray(workflowExecutionLogs.workflowId, workflowIds),
            lt(workflowExecutionLogs.createdAt, retentionDate)
          )
        )
        .limit(BATCH_SIZE)

      results.enhancedLogs.total += oldEnhancedLogs.length
```
This is the core `while` loop for batch processing:
-   `while (hasMoreLogs && batchesProcessed < MAX_BATCHES)`: The loop continues as long as the `hasMoreLogs` flag is true (meaning previous batches were full, implying more logs exist) AND the `MAX_BATCHES` limit has not been reached.
-   `const oldEnhancedLogs = await db.select({ ... }).from(workflowExecutionLogs).where(and(...)).limit(BATCH_SIZE)`: This Drizzle query fetches a batch of old enhanced logs:
    -   `.select({ ... })`: Specifies all the columns to retrieve from `workflowExecutionLogs`. These columns represent the complete data of an execution log, which will be archived.
    -   `.from(workflowExecutionLogs)`: Targets the `workflowExecutionLogs` table.
    -   `.where(and(inArray(workflowExecutionLogs.workflowId, workflowIds), lt(workflowExecutionLogs.createdAt, retentionDate)))`: Filters the logs:
        -   `inArray(workflowExecutionLogs.workflowId, workflowIds)`: Ensures only logs belonging to the previously identified `workflowIds` (of free users) are selected.
        -   `lt(workflowExecutionLogs.createdAt, retentionDate)`: Ensures only logs created *before* the calculated `retentionDate` are selected.
        -   `and(...)`: Combines these two conditions using a logical AND.
    -   `.limit(BATCH_SIZE)`: Restricts the number of returned logs to `BATCH_SIZE` (e.g., 2000), ensuring batch processing.
-   `results.enhancedLogs.total += oldEnhancedLogs.length`: Adds the number of logs fetched in the current batch to the `total` count in the `results` object.

```typescript
      for (const log of oldEnhancedLogs) {
        const today = new Date().toISOString().split('T')[0]

        // Archive enhanced log with more detailed structure
        const enhancedLogKey = `archived-enhanced-logs/${today}/${log.id}.json`
        const enhancedLogData = JSON.stringify({
          ...log,
          archivedAt: new Date().toISOString(),
          logType: 'enhanced',
        })
```
This `for...of` loop iterates through each log retrieved in the current batch:
-   `const today = new Date().toISOString().split('T')[0]`: Gets the current date in `YYYY-MM-DD` format (e.g., `2023-10-27`). This is used to create a date-based folder structure in S3 for easier organization of archived logs.
-   `const enhancedLogKey = `archived-enhanced-logs/${today}/${log.id}.json``: Constructs the S3 object key (path and filename) for the archived log. It puts logs into a folder `archived-enhanced-logs/`, then a subfolder for the current `today`'s date, and finally names the file `log.id.json`.
-   `const enhancedLogData = JSON.stringify({ ...log, archivedAt: new Date().toISOString(), logType: 'enhanced' })`: Converts the entire `log` object (all selected columns) into a JSON string. It also adds two extra fields: `archivedAt` (the timestamp when it was archived) and `logType: 'enhanced'` for better context.

```typescript
        try {
          const { getS3Client } = await import('@/lib/uploads/s3/s3-client')
          await getS3Client().send(
            new PutObjectCommand({
              Bucket: S3_CONFIG.bucket,
              Key: enhancedLogKey,
              Body: enhancedLogData,
              ContentType: 'application/json',
              Metadata: {
                logId: String(log.id),
                workflowId: String(log.workflowId),
                executionId: String(log.executionId),
                logType: 'enhanced',
                archivedAt: new Date().toISOString(),
              },
            })
          )

          results.enhancedLogs.archived++
```
This block attempts to archive the log to S3:
-   `try { ... } catch (archiveError) { ... }`: A nested try-catch block specifically for handling errors during the S3 archiving process for *each individual log*.
-   `const { getS3Client } = await import('@/lib/uploads/s3/s3-client')`: **Dynamic Import**. This is a crucial optimization. Instead of importing the `@aws-sdk/client-s3` (or the underlying `s3-client`) at the top of the file, it's imported *only when needed* inside the loop. This prevents the S3 client from being bundled if the code were ever run client-side (though it's a server-side route here, it's a good defensive practice) and can potentially slightly reduce initial startup time by loading it lazily. `getS3Client()` returns an initialized AWS S3 client.
-   `await getS3Client().send(new PutObjectCommand({ ... }))`: Uses the S3 client to send a `PutObjectCommand`.
    -   `Bucket: S3_CONFIG.bucket`: Specifies the target S3 bucket name.
    -   `Key: enhancedLogKey`: Specifies the S3 object key (path and filename) determined earlier.
    -   `Body: enhancedLogData`: The JSON string content of the log to be uploaded.
    -   `ContentType: 'application/json'`: Tells S3 that the object is a JSON file.
    -   `Metadata: { ... }`: Custom metadata key-value pairs associated with the S3 object. These can be useful for searching, filtering, or providing additional context directly on the S3 object without needing to download its content. `String(log.id)` is used to ensure the values are strings as required by S3 metadata.
-   `results.enhancedLogs.archived++`: If archiving is successful, increments the `archived` counter in the `results` object.

```typescript
          // Clean up associated files if using cloud storage
          if (isUsingCloudStorage() && log.files && Array.isArray(log.files)) {
            for (const file of log.files) {
              if (file && typeof file === 'object' && file.key) {
                results.files.total++
                try {
                  await deleteFile(file.key)
                  results.files.deleted++
                  logger.info(`Deleted file: ${file.key}`)
                } catch (fileError) {
                  results.files.deleteFailed++
                  logger.error(`Failed to delete file ${file.key}:`, { fileError })
                }
              }
            }
          }
```
This block handles the deletion of files associated with the log:
-   `if (isUsingCloudStorage() && log.files && Array.isArray(log.files))`: Checks three conditions:
    1.  `isUsingCloudStorage()`: Confirms that the application is configured to use cloud storage (like S3) for files. If not, there's no need to delete files from there.
    2.  `log.files`: Checks if the log record has a `files` property.
    3.  `Array.isArray(log.files)`: Ensures that `log.files` is actually an array (as expected, where each element represents an associated file).
-   `for (const file of log.files)`: Iterates through each file entry in the `log.files` array.
-   `if (file && typeof file === 'object' && file.key)`: Validates that each `file` entry is a valid object and has a `key` property (which would be the identifier/path in cloud storage).
-   `results.files.total++`: Increments the total count of files identified for deletion.
-   `try { ... } catch (fileError) { ... }`: A nested try-catch block for individual file deletions.
-   `await deleteFile(file.key)`: Calls the internal `deleteFile` utility to delete the file using its `key` (e.g., S3 key).
-   `results.files.deleted++`: If deletion is successful, increments the `deleted` counter for files.
-   `logger.info(...)`: Logs a success message for file deletion.
-   `results.files.deleteFailed++`, `logger.error(...)`: If file deletion fails, increments `deleteFailed` and logs an error.

```typescript
          try {
            // Delete enhanced log
            const deleteResult = await db
              .delete(workflowExecutionLogs)
              .where(eq(workflowExecutionLogs.id, log.id))
              .returning({ id: workflowExecutionLogs.id })

            if (deleteResult.length > 0) {
              results.enhancedLogs.deleted++
            } else {
              results.enhancedLogs.deleteFailed++
              logger.warn(
                `Failed to delete enhanced log ${log.id} after archiving: No rows deleted`
              )
            }
          } catch (deleteError) {
            results.enhancedLogs.deleteFailed++
            logger.error(`Error deleting enhanced log ${log.id} after archiving:`, { deleteError })
          }
        } catch (archiveError) {
          results.enhancedLogs.archiveFailed++
          logger.error(`Failed to archive enhanced log ${log.id}:`, { archiveError })
        }
      } // End of for loop for each log in batch
```
This block attempts to delete the log from the database *after* successful archiving and file cleanup:
-   `try { ... } catch (deleteError) { ... }`: A nested try-catch block for database deletion.
-   `const deleteResult = await db.delete(workflowExecutionLogs).where(eq(workflowExecutionLogs.id, log.id)).returning({ id: workflowExecutionLogs.id })`:
    -   `db.delete(workflowExecutionLogs)`: Initiates a delete operation on the `workflowExecutionLogs` table.
    -   `where(eq(workflowExecutionLogs.id, log.id))`: Specifies that only the log with the current `log.id` should be deleted.
    -   `returning({ id: workflowExecutionLogs.id })`: Instructs Drizzle to return the `id` of the deleted row. This is useful to confirm that a row was actually deleted.
-   `if (deleteResult.length > 0)`: Checks if `returning` returned any IDs, indicating successful deletion.
    -   `results.enhancedLogs.deleted++`: Increments the `deleted` count if successful.
-   `else { ... }`: If no rows were deleted (e.g., the log somehow disappeared between fetching and deleting), it's considered a failure to delete.
    -   `results.enhancedLogs.deleteFailed++`: Increments the `deleteFailed` count.
    -   `logger.warn(...)`: Logs a warning message.
-   `catch (deleteError)`: Catches any database errors during deletion.
    -   `results.enhancedLogs.deleteFailed++`: Increments `deleteFailed`.
    -   `logger.error(...)`: Logs an error message.
-   `catch (archiveError)`: This is the catch block for the *outer* `try` block that encompasses S3 archiving and file/DB deletion for a single log. If S3 archiving fails, none of the subsequent steps (file deletion, DB deletion) are attempted for that specific log.
    -   `results.enhancedLogs.archiveFailed++`: Increments `archiveFailed`.
    -   `logger.error(...)`: Logs an error message for archiving failure.

```typescript
      batchesProcessed++
      hasMoreLogs = oldEnhancedLogs.length === BATCH_SIZE

      logger.info(
        `Processed enhanced logs batch ${batchesProcessed}: ${oldEnhancedLogs.length} logs`
      )
    } // End of while loop for batches
```
After processing all logs in the current batch:
-   `batchesProcessed++`: Increments the counter for batches processed.
-   `hasMoreLogs = oldEnhancedLogs.length === BATCH_SIZE`: Updates the `hasMoreLogs` flag. If the number of logs fetched in this batch was exactly `BATCH_SIZE`, it means there might be more logs remaining, so `hasMoreLogs` remains `true`. If fewer than `BATCH_SIZE` logs were fetched, it implies the end of the available logs, so `hasMoreLogs` becomes `false`, and the `while` loop will terminate.
-   `logger.info(...)`: Logs an informational message about the completion of the current batch.

```typescript
    // Cleanup orphaned snapshots
    try {
      const snapshotRetentionDays = Number(env.FREE_PLAN_LOG_RETENTION_DAYS || '7') + 1 // Keep snapshots 1 day longer
      const cleanedSnapshots = await snapshotService.cleanupOrphanedSnapshots(snapshotRetentionDays)
      results.snapshots.cleaned = cleanedSnapshots
      logger.info(`Cleaned up ${cleanedSnapshots} orphaned snapshots`)
    } catch (snapshotError) {
      results.snapshots.cleanupFailed = 1
      logger.error('Error cleaning up orphaned snapshots:', { snapshotError })
    }
```
This block performs cleanup for orphaned snapshots:
-   `try { ... } catch (snapshotError) { ... }`: A try-catch block for snapshot cleanup.
-   `const snapshotRetentionDays = Number(env.FREE_PLAN_LOG_RETENTION_DAYS || '7') + 1`: Calculates the retention period for snapshots. It uses the same log retention days but adds 1 day, likely to ensure that logs are archived *before* their associated snapshots might be considered for deletion, preventing data integrity issues.
-   `const cleanedSnapshots = await snapshotService.cleanupOrphanedSnapshots(snapshotRetentionDays)`: Calls the `cleanupOrphanedSnapshots` method of the `snapshotService`. This method is responsible for identifying and deleting snapshots that are no longer referenced by active workflow executions or logs, or are older than the specified `snapshotRetentionDays`. It returns the count of cleaned snapshots.
-   `results.snapshots.cleaned = cleanedSnapshots`: Stores the count of cleaned snapshots in the `results` object.
-   `logger.info(...)`: Logs a success message.
-   `results.snapshots.cleanupFailed = 1`, `logger.error(...)`: If snapshot cleanup fails, sets `cleanupFailed` to 1 and logs an error.

```typescript
    const timeElapsed = (Date.now() - startTime) / 1000
    const reachedLimit = batchesProcessed >= MAX_BATCHES && hasMoreLogs

    return NextResponse.json({
      message: `Processed ${batchesProcessed} enhanced log batches (${results.enhancedLogs.total} logs, ${results.files.total} files) in ${timeElapsed.toFixed(2)}s${reachedLimit ? ' (batch limit reached)' : ''}`,
      results,
      complete: !hasMoreLogs,
      batchLimitReached: reachedLimit,
    })
  } catch (error) {
    logger.error('Error in log cleanup process:', { error })
    return NextResponse.json({ error: 'Failed to process log cleanup' }, { status: 500 })
  }
}
```
This is the final part of the `GET` handler:
-   `const timeElapsed = (Date.now() - startTime) / 1000`: Calculates the total time taken for the cleanup process in seconds.
-   `const reachedLimit = batchesProcessed >= MAX_BATCHES && hasMoreLogs`: Determines if the cleanup process stopped because it hit the `MAX_BATCHES` limit, even though `hasMoreLogs` was still true (meaning there were more logs to process).
-   `return NextResponse.json({ ... })`: Returns a JSON response summarizing the entire operation:
    -   `message`: A human-readable summary string including batch count, total logs, total files, and time taken, indicating if the batch limit was reached.
    -   `results`: The detailed `results` object containing all the counts (archived, deleted, failed, etc.).
    -   `complete`: A boolean indicating if all eligible logs were processed (`!hasMoreLogs`).
    -   `batchLimitReached`: A boolean indicating if the operation stopped due to hitting the `MAX_BATCHES` limit.
-   `catch (error)`: This is the catch block for the *outermost* `try` block, handling any unhandled errors in the overall cleanup process.
    -   `logger.error(...)`: Logs the general error.
    -   `return NextResponse.json({ error: 'Failed to process log cleanup' }, { status: 500 })`: Returns a generic 500 error response to the caller.