```typescript
import { randomUUID } from 'crypto'
import { db } from '@sim/db'
import { idempotencyKey } from '@sim/db/schema'
import { and, eq } from 'drizzle-orm'
import { createLogger } from '@/lib/logs/console/logger'
import { getRedisClient } from '@/lib/redis'

// Purpose:
// This file defines a robust `IdempotencyService` class that provides a mechanism to prevent duplicate processing of operations, such as webhook events or API calls. It uses both Redis (for fast, temporary storage) and a database (for persistent storage and fallback) to ensure that an operation is executed only once, even in the face of network issues, retries, or other failures.  The service supports configurable TTL (time-to-live) and namespaces for idempotency keys, allowing for flexible management of idempotency across different parts of an application. It also offers a helper function to generate idempotency keys from webhook data, following RFC conventions.

// Detailed Explanation:

// Imports:
// - `randomUUID` from 'crypto':  Used to generate a unique ID if a suitable idempotency key cannot be derived from the request (e.g., a webhook without a specific ID).
// - `db` from '@sim/db':  Represents the database connection instance, presumably using Drizzle ORM (based on other imports).
// - `idempotencyKey` from '@sim/db/schema': Defines the schema for the `idempotencyKey` table in the database, used to store idempotency keys and results.
// - `and, eq` from 'drizzle-orm': Functions from Drizzle ORM used to build SQL queries for filtering data. `eq` creates an equality condition (e.g., `idempotencyKey.key = normalizedKey`), and `and` combines multiple conditions.
// - `createLogger` from '@/lib/logs/console/logger':  A function to create a logger instance for logging events within the `IdempotencyService`.
// - `getRedisClient` from '@/lib/redis': A function to retrieve a Redis client instance for caching and distributed locking.

const logger = createLogger('IdempotencyService')

// Interfaces:
// - `IdempotencyConfig`:  Defines the configuration options for the `IdempotencyService`.
export interface IdempotencyConfig {
  /**
   * Time-to-live for the idempotency key in seconds
   * Default: 7 days (604800 seconds)
   */
  ttlSeconds?: number // Optional TTL (time-to-live) for the idempotency key in seconds. Defaults to 7 days.

  /**
   * Namespace for the idempotency key (e.g., 'gmail', 'webhook', 'trigger')
   * Default: 'default'
   */
  namespace?: string // Optional namespace for the idempotency key.  Helps to isolate idempotency keys for different parts of the application. Defaults to 'default'.
}

// - `IdempotencyResult`:  Defines the result of checking for an existing idempotency key.
export interface IdempotencyResult {
  /**
   * Whether this is the first time processing this key
   */
  isFirstTime: boolean // Indicates whether this is the first time the key has been seen.

  /**
   * The normalized idempotency key used for storage
   */
  normalizedKey: string // The normalized (standardized) idempotency key.

  /**
   * Previous result if this key was already processed
   */
  previousResult?: any // The result of the previous execution, if any.

  /**
   * Storage method used ('redis', 'database')
   */
  storageMethod: 'redis' | 'database' // Indicates whether the idempotency information was retrieved from Redis or the database.
}

// - `ProcessingResult`:  Defines the structure for storing the result of an operation.
export interface ProcessingResult {
  success: boolean // Indicates whether the operation was successful.
  result?: any // The result of the operation (if successful).
  error?: string // An error message if the operation failed.
  status?: 'in-progress' | 'completed' | 'failed' // The status of the operation.
  startedAt?: number // Timestamp indicating when the processing started
}

// - `AtomicClaimResult`: Defines the result of attempting to atomically claim an idempotency key.
export interface AtomicClaimResult {
  claimed: boolean // Indicates whether the claim was successful.
  existingResult?: ProcessingResult // If the claim failed, this contains the existing result.
  normalizedKey: string // The normalized idempotency key.
  storageMethod: 'redis' | 'database' // Indicates where the existing result was found.
}

// Constants:
const DEFAULT_TTL = 60 * 60 * 24 * 7 // 7 days  // Default TTL for idempotency keys (7 days in seconds).
const REDIS_KEY_PREFIX = 'idempotency:' // Prefix for Redis keys to avoid collisions.
const MAX_WAIT_TIME_MS = 300000 // 5 minutes max wait for in-progress operations // Maximum time to wait (in milliseconds) for an in-progress operation to complete.
const POLL_INTERVAL_MS = 1000 // Check every 1 second for completion  // Interval (in milliseconds) at which to poll for the completion of an in-progress operation.

/**
 * Universal idempotency service for webhooks, triggers, and any other operations
 * that need duplicate prevention.
 */
export class IdempotencyService {
  private config: Required<IdempotencyConfig>

  // Constructor:
  constructor(config: IdempotencyConfig = {}) {
    // Merges the provided configuration with default values.  The `??` operator is the nullish coalescing operator, which returns the right-hand side operand when the left-hand side operand is `null` or `undefined`.
    this.config = {
      ttlSeconds: config.ttlSeconds ?? DEFAULT_TTL,
      namespace: config.namespace ?? 'default',
    }
  }

  /**
   * Generate a normalized idempotency key from various sources
   */
  private normalizeKey(
    provider: string,
    identifier: string,
    additionalContext?: Record<string, any>
  ): string {
    // Creates a consistent idempotency key based on the provider, identifier, and optional context.
    const base = `${this.config.namespace}:${provider}:${identifier}`

    if (additionalContext && Object.keys(additionalContext).length > 0) {
      // Sort keys for consistent hashing
      const sortedKeys = Object.keys(additionalContext).sort()
      const contextStr = sortedKeys.map((key) => `${key}=${additionalContext[key]}`).join('&')
      return `${base}:${contextStr}`
    }

    return base
  }

  /**
   * Check if an operation has already been processed
   */
  async checkIdempotency(
    provider: string,
    identifier: string,
    additionalContext?: Record<string, any>
  ): Promise<IdempotencyResult> {
    // Checks if an operation has already been processed, first in Redis and then in the database.
    const normalizedKey = this.normalizeKey(provider, identifier, additionalContext)
    const redisKey = `${REDIS_KEY_PREFIX}${normalizedKey}`

    // Try Redis first
    try {
      const redis = getRedisClient()
      // If Redis is available...
      if (redis) {
        const cachedResult = await redis.get(redisKey)
        // If the key exists in Redis...
        if (cachedResult) {
          logger.debug(`Idempotency hit in Redis: ${normalizedKey}`)
          return {
            isFirstTime: false,
            normalizedKey,
            previousResult: JSON.parse(cachedResult), // Parse the JSON string back into an object.
            storageMethod: 'redis',
          }
        }

        logger.debug(`Idempotency miss in Redis: ${normalizedKey}`)
        return {
          isFirstTime: true,
          normalizedKey,
          storageMethod: 'redis',
        }
      }
    } catch (error) {
      logger.warn(`Redis idempotency check failed for ${normalizedKey}:`, error)
    }

    // Always fallback to database when Redis is not available
    try {
      // Query the database for the idempotency key.
      const existing = await db
        .select({ result: idempotencyKey.result, createdAt: idempotencyKey.createdAt })
        .from(idempotencyKey)
        .where(
          and(
            eq(idempotencyKey.key, normalizedKey),
            eq(idempotencyKey.namespace, this.config.namespace)
          )
        )
        .limit(1)

      // If the key exists in the database...
      if (existing.length > 0) {
        const item = existing[0]
        // Check if the key has expired based on TTL.
        const isExpired = Date.now() - item.createdAt.getTime() > this.config.ttlSeconds * 1000

        // If the key is not expired...
        if (!isExpired) {
          logger.debug(`Idempotency hit in database: ${normalizedKey}`)
          return {
            isFirstTime: false,
            normalizedKey,
            previousResult: item.result,
            storageMethod: 'database',
          }
        }
        // If the key is expired, delete it from the database.
        await db
          .delete(idempotencyKey)
          .where(eq(idempotencyKey.key, normalizedKey))
          .catch((err) => logger.warn(`Failed to clean up expired key ${normalizedKey}:`, err))
      }

      logger.debug(`Idempotency miss in database: ${normalizedKey}`)
      return {
        isFirstTime: true,
        normalizedKey,
        storageMethod: 'database',
      }
    } catch (error) {
      logger.error(`Database idempotency check failed for ${normalizedKey}:`, error)
      throw new Error(`Failed to check idempotency: database unavailable`)
    }
  }

  /**
   * Atomically claim an idempotency key for processing
   * Returns true if successfully claimed, false if already exists
   */
  async atomicallyClaim(
    provider: string,
    identifier: string,
    additionalContext?: Record<string, any>
  ): Promise<AtomicClaimResult> {
    // Attempts to atomically claim an idempotency key, preventing race conditions.  Uses Redis for speed and the database as a fallback.
    const normalizedKey = this.normalizeKey(provider, identifier, additionalContext)
    const redisKey = `${REDIS_KEY_PREFIX}${normalizedKey}`
    // Create a ProcessingResult object to store the in-progress status
    const inProgressResult: ProcessingResult = {
      success: false,
      status: 'in-progress',
      startedAt: Date.now(),
    }

    // Try Redis first
    try {
      const redis = getRedisClient()
      // If Redis is available...
      if (redis) {
        // Atomically set the key in Redis if it doesn't already exist. 'NX' option ensures that the key is only set if it doesn't exist. 'EX' sets the expiration time.
        const claimed = await redis.set(
          redisKey,
          JSON.stringify(inProgressResult), // Store the in-progress result as a JSON string
          'EX',
          this.config.ttlSeconds,
          'NX'
        )

        // If the claim was successful...
        if (claimed === 'OK') {
          logger.debug(`Atomically claimed idempotency key in Redis: ${normalizedKey}`)
          return {
            claimed: true,
            normalizedKey,
            storageMethod: 'redis',
          }
        }
        // If the claim failed (key already exists)...
        const existingData = await redis.get(redisKey) // Retrieve the existing data from Redis.
        const existingResult = existingData ? JSON.parse(existingData) : null // Parse the existing data as a JSON string or set it to null
        logger.debug(`Idempotency key already claimed in Redis: ${normalizedKey}`)
        return {
          claimed: false,
          existingResult,
          normalizedKey,
          storageMethod: 'redis',
        }
      }
    } catch (error) {
      logger.warn(`Redis atomic claim failed for ${normalizedKey}:`, error)
    }

    // Always fallback to database when Redis is not available
    try {
      // Atomically insert the idempotency key into the database. `onConflictDoNothing` prevents the insert if the key already exists.
      const insertResult = await db
        .insert(idempotencyKey)
        .values({
          key: normalizedKey,
          namespace: this.config.namespace,
          result: inProgressResult, // Store the in-progress ProcessingResult
          createdAt: new Date(),
        })
        .onConflictDoNothing() // Prevents insertion if the key already exists.
        .returning({ key: idempotencyKey.key })

      // If the insert was successful...
      if (insertResult.length > 0) {
        logger.debug(`Atomically claimed idempotency key in database: ${normalizedKey}`)
        return {
          claimed: true,
          normalizedKey,
          storageMethod: 'database',
        }
      }
      // If the insert failed (key already exists)...
      const existing = await db
        .select({ result: idempotencyKey.result })
        .from(idempotencyKey)
        .where(
          and(
            eq(idempotencyKey.key, normalizedKey),
            eq(idempotencyKey.namespace, this.config.namespace)
          )
        )
        .limit(1)

      const existingResult =
        existing.length > 0 ? (existing[0].result as ProcessingResult) : undefined
      logger.debug(`Idempotency key already claimed in database: ${normalizedKey}`)
      return {
        claimed: false,
        existingResult,
        normalizedKey,
        storageMethod: 'database',
      }
    } catch (error) {
      logger.error(`Database atomic claim failed for ${normalizedKey}:`, error)
      throw new Error(`Failed to claim idempotency key: database unavailable`)
    }
  }

  /**
   * Wait for an in-progress operation to complete and return its result
   */
  async waitForResult<T>(normalizedKey: string, storageMethod: 'redis' | 'database'): Promise<T> {
    // Waits for an in-progress operation to complete, polling either Redis or the database.

    const startTime = Date.now()
    const redisKey = `${REDIS_KEY_PREFIX}${normalizedKey}`

    // Poll for the result until the maximum wait time is reached.
    while (Date.now() - startTime < MAX_WAIT_TIME_MS) {
      try {
        let currentResult: ProcessingResult | null = null

        // Check Redis if that's where the key was originally claimed.
        if (storageMethod === 'redis') {
          const redis = getRedisClient()
          if (redis) {
            const data = await redis.get(redisKey)
            currentResult = data ? JSON.parse(data) : null
          }
        } else if (storageMethod === 'database') {
          // Check the database if that's where the key was originally claimed.
          const existing = await db
            .select({ result: idempotencyKey.result })
            .from(idempotencyKey)
            .where(
              and(
                eq(idempotencyKey.key, normalizedKey),
                eq(idempotencyKey.namespace, this.config.namespace)
              )
            )
            .limit(1)
          currentResult = existing.length > 0 ? (existing[0].result as ProcessingResult) : null
        }

        // If the operation has completed...
        if (currentResult?.status === 'completed') {
          logger.debug(`Operation completed, returning result: ${normalizedKey}`)
          if (currentResult.success === false) {
            // If the operation failed, throw an error.
            throw new Error(currentResult.error || 'Previous operation failed')
          }
          return currentResult.result as T // Return the result.
        }

        // If the operation has failed...
        if (currentResult?.status === 'failed') {
          logger.debug(`Operation failed, throwing error: ${normalizedKey}`)
          throw new Error(currentResult.error || 'Previous operation failed') // Throw an error.
        }

        // Wait for the poll interval before checking again.
        await new Promise((resolve) => setTimeout(resolve, POLL_INTERVAL_MS))
      } catch (error) {
        // If the error is due to a failed operation, re-throw it.
        if (error instanceof Error && error.message.includes('operation failed')) {
          throw error
        }
        logger.warn(`Error while waiting for result ${normalizedKey}:`, error)
        // Wait for the poll interval before checking again, even in case of error.
        await new Promise((resolve) => setTimeout(resolve, POLL_INTERVAL_MS))
      }
    }

    // If the operation has not completed within the maximum wait time, throw a timeout error.
    throw new Error(`Timeout waiting for idempotency operation to complete: ${normalizedKey}`)
  }

  /**
   * Store the result of processing for future idempotency checks
   */
  async storeResult(
    normalizedKey: string,
    result: ProcessingResult,
    storageMethod: 'redis' | 'database'
  ): Promise<void> {
    // Stores the result of an operation for future idempotency checks.  Uses Redis for speed and the database as a fallback.
    const serializedResult = JSON.stringify(result)

    // Try Redis first
    try {
      if (storageMethod === 'redis') {
        const redis = getRedisClient()
        if (redis) {
          // Store the result in Redis with the specified TTL. `setex` sets the key with an expiration time.
          await redis.setex(
            `${REDIS_KEY_PREFIX}${normalizedKey}`,
            this.config.ttlSeconds,
            serializedResult
          )
          logger.debug(`Stored idempotency result in Redis: ${normalizedKey}`)
          return
        }
      }
    } catch (error) {
      logger.warn(`Failed to store result in Redis for ${normalizedKey}:`, error)
    }

    // Always fallback to database when Redis is not available
    try {
      // Store the result in the database. `onConflictDoUpdate` updates the existing row if the key already exists.
      await db
        .insert(idempotencyKey)
        .values({
          key: normalizedKey,
          namespace: this.config.namespace,
          result: result,
          createdAt: new Date(),
        })
        .onConflictDoUpdate({
          target: [idempotencyKey.key, idempotencyKey.namespace],
          set: {
            result: result,
            createdAt: new Date(),
          },
        })

      logger.debug(`Stored idempotency result in database: ${normalizedKey}`)
    } catch (error) {
      logger.error(`Failed to store result in database for ${normalizedKey}:`, error)
      throw new Error(`Failed to store idempotency result: database unavailable`)
    }
  }

  /**
   * Execute an operation with idempotency protection using atomic claims
   * Eliminates race conditions by claiming the key before execution
   */
  async executeWithIdempotency<T>(
    provider: string,
    identifier: string,
    operation: () => Promise<T>,
    additionalContext?: Record<string, any>
  ): Promise<T> {
    // Executes an operation with idempotency protection.  This is the main entry point for using the `IdempotencyService`.
    const claimResult = await this.atomicallyClaim(provider, identifier, additionalContext)

    // If the key was not claimed (already exists)...
    if (!claimResult.claimed) {
      const existingResult = claimResult.existingResult

      // If the existing operation has completed...
      if (existingResult?.status === 'completed') {
        logger.info(`Returning cached result for: ${claimResult.normalizedKey}`)
        if (existingResult.success === false) {
          // If the previous operation failed, throw an error.
          throw new Error(existingResult.error || 'Previous operation failed')
        }
        return existingResult.result as T // Return the cached result.
      }

      // If the existing operation has failed...
      if (existingResult?.status === 'failed') {
        logger.info(`Previous operation failed for: ${claimResult.normalizedKey}`)
        throw new Error(existingResult.error || 'Previous operation failed') // Throw an error.
      }

      // If the existing operation is in progress...
      if (existingResult?.status === 'in-progress') {
        logger.info(`Waiting for in-progress operation: ${claimResult.normalizedKey}`)
        return await this.waitForResult<T>(claimResult.normalizedKey, claimResult.storageMethod) // Wait for the result.
      }

      if (existingResult) {
        return existingResult.result as T
      }

      throw new Error(`Unexpected state: key claimed but no existing result found`)
    }

    // If the key was successfully claimed (first time)...
    try {
      logger.info(`Executing new operation: ${claimResult.normalizedKey}`)
      const result = await operation() // Execute the operation.

      // Store the successful result.
      await this.storeResult(
        claimResult.normalizedKey,
        { success: true, result, status: 'completed' },
        claimResult.storageMethod
      )

      logger.debug(`Successfully completed operation: ${claimResult.normalizedKey}`)
      return result // Return the result.
    } catch (error) {
      // If the operation failed...
      const errorMessage = error instanceof Error ? error.message : 'Unknown error'

      // Store the error result.
      await this.storeResult(
        claimResult.normalizedKey,
        { success: false, error: errorMessage, status: 'failed' },
        claimResult.storageMethod
      )

      logger.warn(`Operation failed: ${claimResult.normalizedKey} - ${errorMessage}`)
      throw error // Re-throw the error.
    }
  }

  /**
   * Create an idempotency key from a webhook payload following RFC best practices
   * Standard webhook headers (webhook-id, x-webhook-id, etc.)
   */
  static createWebhookIdempotencyKey(webhookId: string, headers?: Record<string, string>): string {
    // Creates an idempotency key from a webhook ID and headers, following RFC best practices.
    const normalizedHeaders = headers
      ? Object.fromEntries(Object.entries(headers).map(([k, v]) => [k.toLowerCase(), v]))
      : undefined

    // Standardize header names to lowercase for case-insensitive lookup.
    const webhookIdHeader =
      normalizedHeaders?.['webhook-id'] ||
      normalizedHeaders?.['x-webhook-id'] ||
      normalizedHeaders?.['x-shopify-webhook-id'] ||
      normalizedHeaders?.['x-github-delivery'] ||
      normalizedHeaders?.['x-event-id'] ||
      normalizedHeaders?.['x-teams-notification-id']

    // Prioritize existing webhook ID headers.
    if (webhookIdHeader) {
      return `${webhookId}:${webhookIdHeader}`
    }

    // If no header exists, generate a unique ID for the given webhook ID.
    const uniqueId = randomUUID()
    return `${webhookId}:${uniqueId}`
  }
}

// Create instances of the IdempotencyService with specific configurations.
export const webhookIdempotency = new IdempotencyService({
  namespace: 'webhook',
  ttlSeconds: 60 * 60 * 24 * 7, // 7 days
})

export const pollingIdempotency = new IdempotencyService({
  namespace: 'polling',
  ttlSeconds: 60 * 60 * 24 * 3, // 3 days
})
```
