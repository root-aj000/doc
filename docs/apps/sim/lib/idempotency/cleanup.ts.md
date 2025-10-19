```typescript
import { db } from '@sim/db'
import { idempotencyKey } from '@sim/db/schema'
import { and, eq, lt } from 'drizzle-orm'
import { createLogger } from '@/lib/logs/console/logger'

const logger = createLogger('IdempotencyCleanup')

export interface CleanupOptions {
  /**
   * Maximum age of idempotency keys in seconds before they're considered expired
   * Default: 7 days (604800 seconds)
   */
  maxAgeSeconds?: number

  /**
   * Maximum number of keys to delete in a single batch
   * Default: 1000
   */
  batchSize?: number

  /**
   * Specific namespace to clean up, or undefined to clean all namespaces
   */
  namespace?: string
}

/**
 * Clean up expired idempotency keys from the database
 */
export async function cleanupExpiredIdempotencyKeys(
  options: CleanupOptions = {}
): Promise<{ deleted: number; errors: string[] }> {
  const {
    maxAgeSeconds = 7 * 24 * 60 * 60, // 7 days
    batchSize = 1000,
    namespace,
  } = options

  const errors: string[] = []
  let totalDeleted = 0

  try {
    const cutoffDate = new Date(Date.now() - maxAgeSeconds * 1000)

    logger.info('Starting idempotency key cleanup', {
      cutoffDate: cutoffDate.toISOString(),
      namespace: namespace || 'all',
      batchSize,
    })

    let hasMore = true
    let batchCount = 0

    while (hasMore) {
      try {
        const whereCondition = namespace
          ? and(lt(idempotencyKey.createdAt, cutoffDate), eq(idempotencyKey.namespace, namespace))
          : lt(idempotencyKey.createdAt, cutoffDate)

        // First, find IDs to delete with limit
        const toDelete = await db
          .select({ key: idempotencyKey.key, namespace: idempotencyKey.namespace })
          .from(idempotencyKey)
          .where(whereCondition)
          .limit(batchSize)

        if (toDelete.length === 0) {
          break
        }

        // Delete the found records
        const deleteResult = await db
          .delete(idempotencyKey)
          .where(
            and(
              ...toDelete.map((item) =>
                and(eq(idempotencyKey.key, item.key), eq(idempotencyKey.namespace, item.namespace))
              )
            )
          )
          .returning({ key: idempotencyKey.key })

        const deletedCount = deleteResult.length
        totalDeleted += deletedCount
        batchCount++

        if (deletedCount === 0) {
          hasMore = false
          logger.info('No more expired idempotency keys found')
        } else if (deletedCount < batchSize) {
          hasMore = false
          logger.info(`Deleted final batch of ${deletedCount} expired idempotency keys`)
        } else {
          logger.info(`Deleted batch ${batchCount}: ${deletedCount} expired idempotency keys`)

          await new Promise((resolve) => setTimeout(resolve, 100))
        }
      } catch (batchError) {
        const errorMessage =
          batchError instanceof Error ? batchError.message : 'Unknown batch error'
        logger.error(`Error deleting batch ${batchCount + 1}:`, batchError)
        errors.push(`Batch ${batchCount + 1}: ${errorMessage}`)

        batchCount++

        if (errors.length > 5) {
          logger.error('Too many batch errors, stopping cleanup')
          break
        }
      }
    }

    logger.info('Idempotency key cleanup completed', {
      totalDeleted,
      batchCount,
      errors: errors.length,
    })
  } catch (error) {
    const errorMessage = error instanceof Error ? error.message : 'Unknown error'
    logger.error('Failed to cleanup expired idempotency keys:', error)
    errors.push(`General error: ${errorMessage}`)
  }

  return { deleted: totalDeleted, errors }
}

/**
 * Get statistics about idempotency key usage
 */
export async function getIdempotencyKeyStats(): Promise<{
  totalKeys: number
  keysByNamespace: Record<string, number>
  oldestKey: Date | null
  newestKey: Date | null
}> {
  try {
    const allKeys = await db
      .select({
        namespace: idempotencyKey.namespace,
        createdAt: idempotencyKey.createdAt,
      })
      .from(idempotencyKey)

    const totalKeys = allKeys.length
    const keysByNamespace: Record<string, number> = {}
    let oldestKey: Date | null = null
    let newestKey: Date | null = null

    for (const key of allKeys) {
      keysByNamespace[key.namespace] = (keysByNamespace[key.namespace] || 0) + 1

      if (!oldestKey || key.createdAt < oldestKey) {
        oldestKey = key.createdAt
      }
      if (!newestKey || key.createdAt > newestKey) {
        newestKey = key.createdAt
      }
    }

    return {
      totalKeys,
      keysByNamespace,
      oldestKey,
      newestKey,
    }
  } catch (error) {
    logger.error('Failed to get idempotency key stats:', error)
    return {
      totalKeys: 0,
      keysByNamespace: {},
      oldestKey: null,
      newestKey: null,
    }
  }
}
```

### Purpose of this file

This TypeScript file provides functions for managing idempotency keys within a database.  Idempotency keys are used to ensure that an operation is executed only once, even if the request is received multiple times.  The file includes:

1.  **`cleanupExpiredIdempotencyKeys`**: A function to delete expired idempotency keys, preventing the database from growing indefinitely.
2.  **`getIdempotencyKeyStats`**: A function to retrieve statistics about the usage of idempotency keys, such as the total number of keys, keys per namespace, and the oldest and newest keys.

### Detailed Explanation

**Imports:**

```typescript
import { db } from '@sim/db'
import { idempotencyKey } from '@sim/db/schema'
import { and, eq, lt } from 'drizzle-orm'
import { createLogger } from '@/lib/logs/console/logger'
```

*   `db` from `@sim/db`:  This imports the database connection object.  It's assumed that this object is already configured to connect to the database.
*   `idempotencyKey` from `@sim/db/schema`:  This imports the database schema definition for the `idempotencyKey` table.  This likely defines the columns and data types in the table.
*   `and, eq, lt` from `drizzle-orm`: These are functions from the Drizzle ORM (Object-Relational Mapper) used for building database queries.
    *   `and`:  Used to combine multiple conditions in a `WHERE` clause with an "AND" operator.
    *   `eq`:  Used to create an "equals" condition in a `WHERE` clause.
    *   `lt`:  Used to create a "less than" condition in a `WHERE` clause.
*   `createLogger` from `'@/lib/logs/console/logger'`: Imports a function to create a logger instance. This is used for logging information, warnings, and errors within the module.

**Logger Initialization:**

```typescript
const logger = createLogger('IdempotencyCleanup')
```

*   Creates a logger instance named 'IdempotencyCleanup'.  This logger is used throughout the file to log events related to idempotency key management.  The logger likely provides structured logging, making it easier to filter and analyze logs.

**`CleanupOptions` Interface:**

```typescript
export interface CleanupOptions {
  /**
   * Maximum age of idempotency keys in seconds before they're considered expired
   * Default: 7 days (604800 seconds)
   */
  maxAgeSeconds?: number

  /**
   * Maximum number of keys to delete in a single batch
   * Default: 1000
   */
  batchSize?: number

  /**
   * Specific namespace to clean up, or undefined to clean all namespaces
   */
  namespace?: string
}
```

*   Defines an interface for specifying options when cleaning up expired idempotency keys.  This allows customization of the cleanup process.
    *   `maxAgeSeconds`:  The maximum age (in seconds) of an idempotency key before it's considered expired.  Defaults to 7 days.
    *   `batchSize`:  The maximum number of keys to delete in a single database operation.  Defaults to 1000.  This is important for performance and to avoid locking the database for too long.
    *   `namespace`:  An optional namespace to filter the cleanup.  If provided, only keys within that namespace will be deleted. If not provided, all namespaces are cleaned.

**`cleanupExpiredIdempotencyKeys` Function:**

```typescript
export async function cleanupExpiredIdempotencyKeys(
  options: CleanupOptions = {}
): Promise<{ deleted: number; errors: string[] }> {
  const {
    maxAgeSeconds = 7 * 24 * 60 * 60, // 7 days
    batchSize = 1000,
    namespace,
  } = options

  const errors: string[] = []
  let totalDeleted = 0

  try {
    const cutoffDate = new Date(Date.now() - maxAgeSeconds * 1000)

    logger.info('Starting idempotency key cleanup', {
      cutoffDate: cutoffDate.toISOString(),
      namespace: namespace || 'all',
      batchSize,
    })

    let hasMore = true
    let batchCount = 0

    while (hasMore) {
      try {
        const whereCondition = namespace
          ? and(lt(idempotencyKey.createdAt, cutoffDate), eq(idempotencyKey.namespace, namespace))
          : lt(idempotencyKey.createdAt, cutoffDate)

        // First, find IDs to delete with limit
        const toDelete = await db
          .select({ key: idempotencyKey.key, namespace: idempotencyKey.namespace })
          .from(idempotencyKey)
          .where(whereCondition)
          .limit(batchSize)

        if (toDelete.length === 0) {
          break
        }

        // Delete the found records
        const deleteResult = await db
          .delete(idempotencyKey)
          .where(
            and(
              ...toDelete.map((item) =>
                and(eq(idempotencyKey.key, item.key), eq(idempotencyKey.namespace, item.namespace))
              )
            )
          )
          .returning({ key: idempotencyKey.key })

        const deletedCount = deleteResult.length
        totalDeleted += deletedCount
        batchCount++

        if (deletedCount === 0) {
          hasMore = false
          logger.info('No more expired idempotency keys found')
        } else if (deletedCount < batchSize) {
          hasMore = false
          logger.info(`Deleted final batch of ${deletedCount} expired idempotency keys`)
        } else {
          logger.info(`Deleted batch ${batchCount}: ${deletedCount} expired idempotency keys`)

          await new Promise((resolve) => setTimeout(resolve, 100))
        }
      } catch (batchError) {
        const errorMessage =
          batchError instanceof Error ? batchError.message : 'Unknown batch error'
        logger.error(`Error deleting batch ${batchCount + 1}:`, batchError)
        errors.push(`Batch ${batchCount + 1}: ${errorMessage}`)

        batchCount++

        if (errors.length > 5) {
          logger.error('Too many batch errors, stopping cleanup')
          break
        }
      }
    }

    logger.info('Idempotency key cleanup completed', {
      totalDeleted,
      batchCount,
      errors: errors.length,
    })
  } catch (error) {
    const errorMessage = error instanceof Error ? error.message : 'Unknown error'
    logger.error('Failed to cleanup expired idempotency keys:', error)
    errors.push(`General error: ${errorMessage}`)
  }

  return { deleted: totalDeleted, errors }
}
```

*   **Purpose:** This function removes expired idempotency keys from the database in batches to prevent performance issues.

*   **Parameters:**
    *   `options`: An optional `CleanupOptions` object to configure the cleanup process.  Defaults to an empty object, which uses the default options.

*   **Return Value:**
    *   A `Promise` that resolves to an object containing:
        *   `deleted`: The total number of keys deleted.
        *   `errors`: An array of error messages that occurred during the cleanup process.

*   **Logic:**

    1.  **Options Processing:**
        *   Extracts the `maxAgeSeconds`, `batchSize`, and `namespace` from the `options` object, using default values if they are not provided.

    2.  **Initialization:**
        *   Initializes an empty `errors` array to store any errors that occur during the cleanup process.
        *   Initializes `totalDeleted` to 0 to keep track of the number of deleted keys.

    3.  **Error Handling (Try/Catch):**
        *   Wraps the entire cleanup process in a `try...catch` block to handle any unexpected errors.

    4.  **Calculate Cutoff Date:**
        *   Calculates the `cutoffDate` by subtracting `maxAgeSeconds * 1000` milliseconds from the current time.  Keys older than this date will be considered expired.

    5.  **Logging:**
        *   Logs the start of the cleanup process, including the `cutoffDate`, `namespace`, and `batchSize`.

    6.  **Batch Processing (While Loop):**
        *   Enters a `while` loop that continues as long as `hasMore` is true.  This loop processes the cleanup in batches.
        *   `hasMore` is initialized to true and set to false when no more keys are found or an error occurs.
        *   `batchCount` keeps track of the number of batches processed.

    7.  **Batch Error Handling (Nested Try/Catch):**
        *   Each batch is processed within a nested `try...catch` block to handle errors specific to that batch.  This allows the cleanup to continue even if some batches fail.

    8.  **Construct WHERE Condition:**
        *   Constructs the `whereCondition` for the database query based on whether a `namespace` is provided.
            *   If `namespace` is provided, the condition includes both the `createdAt` being less than the `cutoffDate` and the `namespace` matching the provided value.
            *   If `namespace` is not provided, the condition only includes the `createdAt` being less than the `cutoffDate`.

    9.  **Select Keys to Delete:**
        *   Executes a database query to select the keys to delete.
            *   `db.select(...)`:  Selects the `key` and `namespace` columns from the `idempotencyKey` table.
            *   `from(idempotencyKey)`: Specifies the table to select from.
            *   `where(whereCondition)`:  Filters the results based on the constructed `whereCondition`.
            *   `limit(batchSize)`:  Limits the number of results to the `batchSize`. This is crucial for batch processing.
            *   `await`:  Waits for the database query to complete. The result is stored in `toDelete`.

    10. **Check for More Keys:**
        *   If `toDelete.length === 0`, it means no more expired keys were found, so the `hasMore` flag is set to `false`, and the loop breaks.

    11. **Delete Keys:**
        *   Executes a database query to delete the selected keys.
            *   `db.delete(idempotencyKey)`:  Specifies the table to delete from.
            *   `where(...)`:  This is where the complex logic comes in.  It constructs a `WHERE` clause that matches the keys found in the `toDelete` array.
                *   `toDelete.map((item) => and(eq(idempotencyKey.key, item.key), eq(idempotencyKey.namespace, item.namespace)))`: this line maps over the `toDelete` array and builds an array of conditions.  Each element in the array is an "and" condition, that ensures both the `key` and `namespace` match.
                *   `and(...)`: wraps the array of conditions with another and, to combine them.
            *   `returning({ key: idempotencyKey.key })`:  Specifies that the deleted keys should be returned. This is useful for verification and auditing.
            *   `await`:  Waits for the delete operation to complete. The result is stored in `deleteResult`.

    12. **Update Counts and Logging:**
        *   `deletedCount`: Gets the number of deleted keys from the `deleteResult`.
        *   `totalDeleted`:  Adds the `deletedCount` to the `totalDeleted` counter.
        *   `batchCount`: Increments the `batchCount`.
        *   Logs the result of the batch deletion.
        *   If `deletedCount < batchSize`, it means that there are less keys to delete than the batch size. In this case, `hasMore` is set to false.

    13. **Delay (Optional):**

        *   `await new Promise((resolve) => setTimeout(resolve, 100))`:  Introduces a 100ms delay after each batch.  This can help to avoid overwhelming the database and improve performance.

    14. **Batch Error Handling:**

        *   If an error occurs during the batch processing (in the nested `catch` block):
            *   Logs the error message.
            *   Pushes the error message to the `errors` array.
            *   Increments the `batchCount`.
            *   If the number of errors exceeds 5, the loop breaks to prevent further errors.

    15. **Cleanup Completion Logging:**
        *   After the `while` loop completes, logs the completion of the cleanup process, including the `totalDeleted`, `batchCount`, and the number of errors.

    16. **General Error Handling:**
        *   If an error occurs during the entire cleanup process (in the outer `catch` block):
            *   Logs the error message.
            *   Pushes the error message to the `errors` array.

    17. **Return Results:**
        *   Returns an object containing the `totalDeleted` count and the `errors` array.

**`getIdempotencyKeyStats` Function:**

```typescript
/**
 * Get statistics about idempotency key usage
 */
export async function getIdempotencyKeyStats(): Promise<{
  totalKeys: number
  keysByNamespace: Record<string, number>
  oldestKey: Date | null
  newestKey: Date | null
}> {
  try {
    const allKeys = await db
      .select({
        namespace: idempotencyKey.namespace,
        createdAt: idempotencyKey.createdAt,
      })
      .from(idempotencyKey)

    const totalKeys = allKeys.length
    const keysByNamespace: Record<string, number> = {}
    let oldestKey: Date | null = null
    let newestKey: Date | null = null

    for (const key of allKeys) {
      keysByNamespace[key.namespace] = (keysByNamespace[key.namespace] || 0) + 1

      if (!oldestKey || key.createdAt < oldestKey) {
        oldestKey = key.createdAt
      }
      if (!newestKey || key.createdAt > newestKey) {
        newestKey = key.createdAt
      }
    }

    return {
      totalKeys,
      keysByNamespace,
      oldestKey,
      newestKey,
    }
  } catch (error) {
    logger.error('Failed to get idempotency key stats:', error)
    return {
      totalKeys: 0,
      keysByNamespace: {},
      oldestKey: null,
      newestKey: null,
    }
  }
}
```

*   **Purpose:** Retrieves statistics about the idempotency keys stored in the database.

*   **Parameters:**
    *   None

*   **Return Value:**
    *   A `Promise` that resolves to an object containing:
        *   `totalKeys`: The total number of idempotency keys in the database.
        *   `keysByNamespace`: An object where the keys are namespaces and the values are the number of keys in that namespace.
        *   `oldestKey`: The creation date of the oldest idempotency key.
        *   `newestKey`: The creation date of the newest idempotency key.

*   **Logic:**

    1.  **Error Handling (Try/Catch):**
        *   Wraps the entire process in a `try...catch` block to handle any unexpected errors.

    2.  **Select All Keys:**
        *   Executes a database query to select all `namespace` and `createdAt` values from the `idempotencyKey` table.
            *   `db.select(...)`: Selects the `namespace` and `createdAt` columns.
            *   `from(idempotencyKey)`: Specifies the table to select from.
            *   `await`: Waits for the database query to complete. The result is stored in `allKeys`.

    3.  **Initialize Variables:**
        *   `totalKeys`: Initialized to the length of the `allKeys` array.
        *   `keysByNamespace`: Initialized as an empty object to store the counts of keys per namespace.
        *   `oldestKey`: Initialized to `null`.
        *   `newestKey`: Initialized to `null`.

    4.  **Iterate Through Keys:**
        *   Iterates through the `allKeys` array.
        *   For each `key`:
            *   Updates the `keysByNamespace` object by incrementing the count for the key's namespace.  If the namespace doesn't exist in the object, it's initialized to 0 before incrementing.
            *   Updates `oldestKey` if the current key's `createdAt` is older than the current `oldestKey`.
            *   Updates `newestKey` if the current key's `createdAt` is newer than the current `newestKey`.

    5.  **Return Statistics:**
        *   Returns an object containing the calculated `totalKeys`, `keysByNamespace`, `oldestKey`, and `newestKey`.

    6.  **Error Handling:**
        *   If an error occurs during the process (in the `catch` block):
            *   Logs the error message.
            *   Returns an object with default values (0 for `totalKeys`, empty object for `keysByNamespace`, and `null` for `oldestKey` and `newestKey`). This prevents the application from crashing if it cannot retrieve the stats.

### Simplification of Complex Logic

The most complex logic in this file lies in the `cleanupExpiredIdempotencyKeys` function, specifically the construction of the `WHERE` clause for the `delete` operation within the batch processing loop.

Here's the original code:

```typescript
.where(
  and(
    ...toDelete.map((item) =>
      and(eq(idempotencyKey.key, item.key), eq(idempotencyKey.namespace, item.namespace))
    )
  )
)
```

This code does the following:

1.  **`toDelete.map(...)`**:  Iterates over each item in the `toDelete` array, which contains the `key` and `namespace` of the keys to be deleted.
2.  **`and(eq(idempotencyKey.key, item.key), eq(idempotencyKey.namespace, item.namespace))`**:  For each item, it creates an `and` condition that checks if the `key` and `namespace` in the `idempotencyKey` table match the `key` and `namespace` of the current item.  This ensures that only keys with the exact matching `key` and `namespace` are deleted.
3.  **`...toDelete.map(...)`**: uses the spread operator to expand the array of AND conditions into the `and()` function.

A more efficient approach can be to directly use `IN` operator, it may perform better depending on the database system:

```typescript
const keysToDelete = toDelete.map(item => item.key);
const namespacesToDelete = toDelete.map(item => item.namespace);

.where(and(
    idempotencyKey.key.in(keysToDelete),
    idempotencyKey.namespace.in(namespacesToDelete)
))
```

This will require the `in` operator to be imported from `drizzle-orm`. This is a more direct and potentially more performant way to express the same logic, especially when dealing with a large number of keys to delete.

### Summary

This file provides essential functionality for managing idempotency keys in a database.  The `cleanupExpiredIdempotencyKeys` function ensures that the database doesn't become cluttered with expired keys, while the `getIdempotencyKeyStats` function provides valuable insights into the usage of idempotency keys. The file uses best practices such as batch processing, logging, and error handling to ensure robustness and maintainability.
