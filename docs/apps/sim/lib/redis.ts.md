This TypeScript file provides a robust and flexible solution for managing state, preventing duplicate processing (message IDs), and implementing distributed locks using **Redis**. It's specifically designed with **serverless environments** in mind, offering connection pooling and an **in-memory fallback cache** for scenarios where Redis might not be available or configured.

### Purpose of this File

The primary goals of this file are:

1.  **Centralized Cache & State Management**: To store temporary data (like processed message IDs or lock statuses) that needs to be shared across multiple instances of your application, which is crucial in distributed systems like serverless functions.
2.  **Preventing Duplicate Processing**: By storing identifiers of messages that have already been processed, the system can avoid re-processing the same message if it's received multiple times.
3.  **Distributed Locking**: To ensure that certain operations are executed by only one instance of an application at a time, preventing race conditions and data corruption in a distributed environment.
4.  **Resilience**: It includes an in-memory fallback mechanism, meaning that if Redis is unavailable, the application can still function (albeit with local, non-shared caching/locking) instead of completely failing.
5.  **Serverless Optimization**: It configures the Redis client (`ioredis`) with settings optimal for serverless functions, such as aggressive connection timeouts and retry strategies, to ensure efficient resource usage and faster cold starts.

---

### Detailed Explanation

Let's break down the code section by section.

#### 1. Imports and Initial Setup

```typescript
import Redis from 'ioredis'
import { env } from '@/lib/env'
import { createLogger } from '@/lib/logs/console/logger'

const logger = createLogger('Redis')
```

*   `import Redis from 'ioredis'`: This line imports the `ioredis` library, which is a high-performance Redis client for Node.js. It's used to interact with a Redis server.
*   `import { env } from '@/lib/env'`: This imports an `env` object, presumably from a local utility file (`@/lib/env`). This object is used to access environment variables, such as `REDIS_URL`.
*   `import { createLogger } from '@/lib/logs/console/logger'`: This imports a function to create a logger instance, likely a custom wrapper around a logging library, for consistent logging throughout the application.
*   `const logger = createLogger('Redis')`: An instance of the logger is created specifically for Redis-related messages, helping to categorize logs.

```typescript
// Only use Redis if explicitly configured
const redisUrl = env.REDIS_URL
```

*   `const redisUrl = env.REDIS_URL`: This line retrieves the Redis connection URL from environment variables. The `REDIS_URL` environment variable is essential for Redis to be used. If it's not set, Redis functionality will be disabled, and the system will rely on the in-memory fallback.

```typescript
// Global Redis client for connection pooling
let globalRedisClient: Redis | null = null
```

*   `let globalRedisClient: Redis | null = null`: This declares a variable to hold the single, shared Redis client instance. It's initialized to `null`. Using a global variable for the client is a common pattern for **connection pooling** in Node.js applications. This prevents creating a new connection for every request, which is inefficient and can exhaust server resources. The `| null` type annotation indicates that it can either be a `Redis` client object or `null`.

```typescript
// Fallback in-memory cache for when Redis is not available
const inMemoryCache = new Map<string, { value: string; expiry: number | null }>()
const MAX_CACHE_SIZE = 1000
```

*   `const inMemoryCache = new Map<string, { value: string; expiry: number | null }>()`: This creates a `Map` object to serve as a simple, in-memory fallback cache.
    *   It stores keys as `string`.
    *   Each value is an object `{ value: string; expiry: number | null }`, where `value` is the cached data (e.g., '1' to indicate presence) and `expiry` is a timestamp (in milliseconds since epoch) indicating when the entry should expire, or `null` if it never expires.
*   `const MAX_CACHE_SIZE = 1000`: This constant defines the maximum number of entries allowed in the `inMemoryCache`. This is a safeguard to prevent the in-memory cache from consuming too much memory in long-running processes.

#### 2. `getRedisClient()` Function

This function is responsible for initializing and returning a *single, shared* Redis client instance.

```typescript
/**
 * Get a Redis client instance
 * Uses connection pooling to avoid creating a new connection for each request
 */
export function getRedisClient(): Redis | null {
  // For server-side only
  if (typeof window !== 'undefined') return null
```

*   `export function getRedisClient(): Redis | null`: This declares an exported function `getRedisClient` that returns either a `Redis` client instance or `null`.
*   `if (typeof window !== 'undefined') return null`: This is a common pattern to ensure code only runs on the server-side (e.g., in Node.js) and not in a web browser. If `window` is defined (meaning it's running in a browser environment), it immediately returns `null` because Redis connections should not be made directly from a client-side browser.

```typescript
  // Return null immediately if no Redis URL is configured
  if (!redisUrl) {
    return null
  }
```

*   `if (!redisUrl) { return null }`: If the `REDIS_URL` environment variable was not set, this condition will be true, and the function will immediately return `null`, indicating that Redis is not available for use.

```typescript
  if (globalRedisClient) return globalRedisClient
```

*   `if (globalRedisClient) return globalRedisClient`: This is the core of the **connection pooling** logic. If `globalRedisClient` already holds an active Redis client instance (meaning it's already been initialized), that existing instance is immediately returned. This prevents creating redundant connections.

```typescript
  try {
    // Create a new Redis client with optimized settings for serverless
    globalRedisClient = new Redis(redisUrl, {
      // Keep alive is critical for serverless to reuse connections
      keepAlive: 1000,
      // Faster connection timeout for serverless
      connectTimeout: 5000,
      // Disable reconnection attempts in serverless
      maxRetriesPerRequest: 3,
      // Retry strategy with exponential backoff
      retryStrategy: (times) => {
        if (times > 5) {
          logger.warn('Redis connection failed after 5 attempts, using fallback')
          return null // Stop retrying
        }
        return Math.min(times * 200, 2000) // Exponential backoff
      },
    })
```

*   `globalRedisClient = new Redis(redisUrl, { ... })`: If `globalRedisClient` is `null`, a new `Redis` client instance is created using the `redisUrl` and a set of configuration options specifically tailored for serverless environments:
    *   `keepAlive: 1000`: This option tells `ioredis` to send `KEEP-ALIVE` pings every 1000ms (1 second). This is crucial in serverless to prevent idle connections from being terminated by load balancers or firewalls, allowing functions to reuse existing connections across invocations.
    *   `connectTimeout: 5000`: Sets a relatively fast connection timeout of 5 seconds. In serverless, you want to fail fast if a connection can't be established, rather than waiting indefinitely.
    *   `maxRetriesPerRequest: 3`: Limits the number of times `ioredis` will retry a command if the connection is lost. This prevents individual requests from hanging too long if Redis is experiencing issues.
    *   `retryStrategy: (times) => { ... }`: This defines a custom strategy for how `ioredis` should attempt to reconnect if the connection drops.
        *   `if (times > 5) { ... return null }`: If the client has attempted to reconnect more than 5 times, it logs a warning and returns `null`, stopping further reconnection attempts. This is a pragmatic approach for serverless; if Redis is down for an extended period, it's better to log and let the fallback handle it.
        *   `return Math.min(times * 200, 2000)`: Implements an **exponential backoff** strategy. The delay between retries increases (e.g., 200ms, then 400ms, 600ms, etc.), but it's capped at 2000ms (2 seconds) to avoid excessively long delays.

```typescript
    // Handle connection events
    globalRedisClient.on('error', (err: any) => {
      logger.error('Redis connection error:', { err })
      if (err.code === 'ECONNREFUSED' || err.code === 'ETIMEDOUT') {
        globalRedisClient = null
      }
    })

    globalRedisClient.on('connect', () => {})

    return globalRedisClient
  } catch (error) {
    logger.error('Failed to initialize Redis client:', { error })
    return null
  }
}
```

*   `globalRedisClient.on('error', (err: any) => { ... })`: This registers an event listener for `error` events on the Redis client.
    *   `logger.error('Redis connection error:', { err })`: It logs any connection errors.
    *   `if (err.code === 'ECONNREFUSED' || err.code === 'ETIMEDOUT') { globalRedisClient = null }`: If the error is a connection refusal (`ECONNREFUSED`) or a timeout (`ETIMEDOUT`), it explicitly sets `globalRedisClient` back to `null`. This ensures that the next call to `getRedisClient` will attempt to create a *new* connection, rather than trying to reuse a client that is known to be in a bad state.
*   `globalRedisClient.on('connect', () => {})`: This registers an event listener for the `connect` event. Currently, it does nothing, but it could be used for logging successful connections or other setup tasks.
*   `return globalRedisClient`: If the client is successfully created or already exists, it's returned.
*   `catch (error) { ... return null }`: This `try-catch` block handles any errors that might occur during the *initialization* of the Redis client (e.g., invalid URL format). It logs the error and returns `null`.

#### 3. Message ID Cache Functions

These functions provide a way to check if a "message" (or any identifier) has been processed and to mark it as such. This is crucial for idempotency – ensuring that an operation has the same result whether it's performed once or many times.

```typescript
const MESSAGE_ID_PREFIX = 'processed:' // Generic prefix
const MESSAGE_ID_EXPIRY = 60 * 60 * 24 * 7 // 7 days in seconds
```

*   `const MESSAGE_ID_PREFIX = 'processed:'`: A prefix is defined for keys used in the message ID cache. This helps organize keys in Redis and prevent conflicts with other data.
*   `const MESSAGE_ID_EXPIRY = 60 * 60 * 24 * 7`: Defines a default expiry time for processed message IDs: 7 days, in seconds. This prevents the cache from growing indefinitely and storing old, irrelevant message IDs.

##### `hasProcessedMessage()` Function

```typescript
/**
 * Check if a key exists in Redis or fallback cache.
 * @param key The key to check (e.g., messageId, lockKey).
 * @returns True if the key exists and hasn't expired, false otherwise.
 */
export async function hasProcessedMessage(key: string): Promise<boolean> {
  try {
    const redis = getRedisClient()
    const fullKey = `${MESSAGE_ID_PREFIX}${key}` // Use generic prefix
```

*   `export async function hasProcessedMessage(key: string): Promise<boolean>`: An `async` function that takes a `key` (e.g., a message ID) and returns a Promise that resolves to `true` if the key exists (is processed), or `false` otherwise.
*   `const redis = getRedisClient()`: Attempts to get a Redis client instance.
*   `const fullKey = `${MESSAGE_ID_PREFIX}${key}``: Constructs the full key by prepending the `MESSAGE_ID_PREFIX`.

```typescript
    if (redis) {
      // Use Redis if available
      const result = await redis.exists(fullKey)
      return result === 1
    }
    // Fallback to in-memory cache
    const cacheEntry = inMemoryCache.get(fullKey)
    if (!cacheEntry) return false

    // Check if the entry has expired
    if (cacheEntry.expiry && cacheEntry.expiry < Date.now()) {
      inMemoryCache.delete(fullKey)
      return false
    }

    return true
  } catch (error) {
    logger.error(`Error checking key ${key}:`, { error })
    // Fallback to in-memory cache on error
    const fullKey = `${MESSAGE_ID_PREFIX}${key}`
    const cacheEntry = inMemoryCache.get(fullKey)
    return !!cacheEntry && (!cacheEntry.expiry || cacheEntry.expiry > Date.now())
  }
}
```

*   `if (redis) { ... }`: If a Redis client is available:
    *   `const result = await redis.exists(fullKey)`: It uses the Redis `EXISTS` command to check if the `fullKey` exists. `exists` returns `1` if the key exists, `0` otherwise.
    *   `return result === 1`: Returns `true` if the key exists in Redis, `false` otherwise.
*   `// Fallback to in-memory cache`: If Redis is *not* available:
    *   `const cacheEntry = inMemoryCache.get(fullKey)`: It tries to retrieve the entry from the `inMemoryCache`.
    *   `if (!cacheEntry) return false`: If no entry is found, it means the key is not processed.
    *   `if (cacheEntry.expiry && cacheEntry.expiry < Date.now()) { ... return false }`: If an entry is found, it checks if it has expired by comparing its `expiry` timestamp with the current time (`Date.now()`). If expired, it's removed from the cache (`inMemoryCache.delete`) and `false` is returned.
    *   `return true`: If the entry exists and hasn't expired, `true` is returned.
*   `catch (error) { ... }`: This `try-catch` block handles errors that might occur during Redis operations (e.g., network issues *after* connection).
    *   `logger.error(...)`: Logs the error.
    *   `// Fallback to in-memory cache on error`: Even if a Redis error occurs, it attempts to use the in-memory cache as a last resort. This improves resilience.
    *   `return !!cacheEntry && (!cacheEntry.expiry || cacheEntry.expiry > Date.now())`: Returns `true` if the in-memory entry exists and is not expired, `false` otherwise.

##### `markMessageAsProcessed()` Function

```typescript
/**
 * Mark a key as processed/present in Redis or fallback cache.
 * @param key The key to mark (e.g., messageId, lockKey).
 * @param expirySeconds Optional expiry time in seconds (defaults to 7 days).
 */
export async function markMessageAsProcessed(
  key: string,
  expirySeconds: number = MESSAGE_ID_EXPIRY
): Promise<void> {
  try {
    const redis = getRedisClient()
    const fullKey = `${MESSAGE_ID_PREFIX}${key}` // Use generic prefix
```

*   `export async function markMessageAsProcessed(...)`: An `async` function to mark a `key` as processed. It takes an optional `expirySeconds` parameter, defaulting to `MESSAGE_ID_EXPIRY` (7 days). It returns `void` as it performs an action.
*   `const redis = getRedisClient()`: Gets the Redis client.
*   `const fullKey = `${MESSAGE_ID_PREFIX}${key}``: Constructs the full key.

```typescript
    if (redis) {
      // Use Redis if available - use pipelining for efficiency
      await redis.set(fullKey, '1', 'EX', expirySeconds)
    } else {
      // Fallback to in-memory cache
      const expiry = expirySeconds ? Date.now() + expirySeconds * 1000 : null
      inMemoryCache.set(fullKey, { value: '1', expiry })
```

*   `if (redis) { ... }`: If a Redis client is available:
    *   `await redis.set(fullKey, '1', 'EX', expirySeconds)`: Sets the `fullKey` in Redis with a value of `'1'` (simply indicating presence) and an expiration time using the `EX` option, specified in `expirySeconds`. This is an atomic operation.
*   `else { ... }`: If Redis is *not* available (or `getRedisClient` returned `null`):
    *   `const expiry = expirySeconds ? Date.now() + expirySeconds * 1000 : null`: Calculates the expiry timestamp for the in-memory cache. `Date.now()` returns milliseconds, so `expirySeconds` is converted to milliseconds. If `expirySeconds` is 0 or not provided, `expiry` is `null` (no expiration).
    *   `inMemoryCache.set(fullKey, { value: '1', expiry })`: Stores the key in the `inMemoryCache` with its value and calculated expiry.

```typescript
      // Clean up old message IDs if cache gets too large
      if (inMemoryCache.size > MAX_CACHE_SIZE) {
        const now = Date.now()

        // First try to remove expired entries
        for (const [cacheKey, entry] of inMemoryCache.entries()) {
          if (entry.expiry && entry.expiry < now) {
            inMemoryCache.delete(cacheKey)
          }
        }

        // If still too large, remove oldest entries (FIFO based on insertion order)
        if (inMemoryCache.size > MAX_CACHE_SIZE) {
          const keysToDelete = Array.from(inMemoryCache.keys()).slice(
            0,
            inMemoryCache.size - MAX_CACHE_SIZE
          )

          for (const keyToDelete of keysToDelete) {
            inMemoryCache.delete(keyToDelete)
          }
        }
      }
    }
  } catch (error) {
    logger.error(`Error marking key ${key} as processed:`, { error })
    // Fallback to in-memory cache on error
    const fullKey = `${MESSAGE_ID_PREFIX}${key}`
    const expiry = expirySeconds ? Date.now() + expirySeconds * 1000 : null
    inMemoryCache.set(fullKey, { value: '1', expiry })
  }
}
```

*   `if (inMemoryCache.size > MAX_CACHE_SIZE) { ... }`: This block implements a cache eviction strategy for the in-memory fallback. If the `inMemoryCache` exceeds its `MAX_CACHE_SIZE`:
    *   `for (const [cacheKey, entry] of inMemoryCache.entries()) { ... }`: It first iterates through all entries and removes any that have already expired.
    *   `if (inMemoryCache.size > MAX_CACHE_SIZE) { ... }`: If, *after* removing expired entries, the cache is *still* too large:
        *   `const keysToDelete = Array.from(inMemoryCache.keys()).slice(0, inMemoryCache.size - MAX_CACHE_SIZE)`: It converts the `Map` keys to an array and selects the `MAX_CACHE_SIZE` oldest keys (Maps iterate in insertion order, so the beginning of the `keys()` array holds the oldest entries).
        *   `for (const keyToDelete of keysToDelete) { inMemoryCache.delete(keyToDelete) }`: These oldest entries are then deleted to bring the cache size back down. This effectively implements a basic **FIFO (First-In, First-Out)** eviction for non-expired entries.
*   `catch (error) { ... }`: This `try-catch` block handles errors during Redis operations for `markMessageAsProcessed`.
    *   It logs the error.
    *   `// Fallback to in-memory cache on error`: Crucially, even if Redis fails *while trying to mark a message*, it falls back to marking it in the in-memory cache, ensuring local idempotency even under Redis failures.

#### 4. Distributed Locking Functions

These functions implement a basic distributed locking mechanism using Redis. A distributed lock ensures that only one process across multiple servers or serverless functions can execute a critical section of code at a time.

##### `acquireLock()` Function

```typescript
/**
 * Attempts to acquire a lock using Redis SET NX command.
 * @param lockKey The key to use for the lock.
 * @param value The value to set (e.g., a unique identifier for the process holding the lock).
 * @param expirySeconds The lock's time-to-live in seconds.
 * @returns True if the lock was acquired successfully, false otherwise.
 */
export async function acquireLock(
  lockKey: string,
  value: string,
  expirySeconds: number
): Promise<boolean> {
  try {
    const redis = getRedisClient()
    if (!redis) {
      logger.warn('Redis client not available, cannot acquire lock.')
      // Fallback behavior: maybe allow processing but log a warning?
      // Or treat as lock acquired if no Redis? Depends on desired behavior.
      return true // Or false, depending on safety requirements
    }
```

*   `export async function acquireLock(...)`: An `async` function to acquire a lock. It takes `lockKey`, a `value` (often a unique ID of the locker), and `expirySeconds`. It returns `true` if the lock was acquired, `false` otherwise.
*   `const redis = getRedisClient()`: Gets the Redis client.
*   `if (!redis) { ... return true }`: If Redis is not available, it logs a warning. The `return true` here is a **design decision**. In this specific implementation, it means "if Redis is not present, assume the lock is acquired, and proceed (but be aware there's no *distributed* locking)." This might be acceptable for non-critical operations, but for highly critical sections, you'd likely `return false` to prevent execution if a robust distributed lock isn't possible.

```typescript
    // Use SET key value EX expirySeconds NX
    // Returns "OK" if successful, null if key already exists (lock held)
    const result = await redis.set(lockKey, value, 'EX', expirySeconds, 'NX')

    return result === 'OK'
  } catch (error) {
    logger.error(`Error acquiring lock for key ${lockKey}:`, { error })
    // Treat errors as failure to acquire lock for safety
    return false
  }
}
```

*   `const result = await redis.set(lockKey, value, 'EX', expirySeconds, 'NX')`: This is the core of atomic distributed locking in Redis.
    *   `SET lockKey value`: Sets the `lockKey` to the `value`.
    *   `EX expirySeconds`: Sets an expiration time for the lock in seconds. This is crucial as it prevents "deadlocks" if the process holding the lock crashes before releasing it.
    *   `NX`: This option stands for "**N**ot E**X**ist." It tells Redis to *only* set the key if it **does not already exist**.
    *   If the key is successfully set (meaning the lock was acquired), Redis returns `"OK"`. If the key already existed (meaning another process holds the lock), Redis returns `null`.
*   `return result === 'OK'`: Returns `true` if the lock was acquired, `false` otherwise.
*   `catch (error) { ... return false }`: Handles errors during the Redis `SET` operation. It logs the error and, for safety, returns `false` (assuming the lock couldn't be acquired).

##### `getLockValue()` Function

```typescript
/**
 * Retrieves the value of a key from Redis.
 * @param key The key to retrieve.
 * @returns The value of the key, or null if the key doesn't exist or an error occurs.
 */
export async function getLockValue(key: string): Promise<string | null> {
  try {
    const redis = getRedisClient()
    if (!redis) {
      logger.warn('Redis client not available, cannot get lock value.')
      return null // Cannot determine lock value
    }
    return await redis.get(key)
  } catch (error) {
    logger.error(`Error getting value for key ${key}:`, { error })
    return null
  }
}
```

*   `export async function getLockValue(...)`: An `async` function to retrieve the value associated with a `key` (e.g., to check who holds a lock). It returns a `string` (the value) or `null`.
*   `const redis = getRedisClient()`: Gets the Redis client.
*   `if (!redis) { ... return null }`: If Redis is not available, it logs a warning and returns `null`.
*   `return await redis.get(key)`: If Redis is available, it uses the `GET` command to retrieve the value of the `key`. Redis returns the value if the key exists, or `null` if it doesn't.
*   `catch (error) { ... return null }`: Handles errors during the `GET` operation, logs them, and returns `null`.

##### `releaseLock()` Function

```typescript
/**
 * Releases a lock by deleting the key.
 * Ideally, use Lua script for safe release (check value before deleting),
 * but simple DEL is often sufficient if lock expiry is handled well.
 * @param lockKey The key of the lock to release.
 */
export async function releaseLock(lockKey: string): Promise<void> {
  try {
    const redis = getRedisClient()
    if (redis) {
      await redis.del(lockKey)
    } else {
      logger.warn('Redis client not available, cannot release lock.')
      // No fallback needed for releasing if using in-memory cache for locking wasn't implemented
    }
  } catch (error) {
    logger.error(`Error releasing lock for key ${lockKey}:`, { error })
  }
}
```

*   `export async function releaseLock(...)`: An `async` function to release a lock. It takes the `lockKey` and returns `void`.
*   `const redis = getRedisClient()`: Gets the Redis client.
*   `if (redis) { await redis.del(lockKey) }`: If Redis is available, it uses the `DEL` command to delete the `lockKey`, thereby releasing the lock.
*   `else { logger.warn(...) }`: If Redis is not available, it logs a warning. There's no in-memory fallback for releasing a lock here, as the `acquireLock` function's fallback behavior (`return true`) implies that no *actual* distributed lock was established anyway.
*   `catch (error) { ... }`: Handles errors during the `DEL` operation and logs them.

#### 5. `closeRedisConnection()` Function

```typescript
/**
 * Close the Redis connection
 * Important for proper cleanup in serverless environments
 */
export async function closeRedisConnection(): Promise<void> {
  if (globalRedisClient) {
    try {
      await globalRedisClient.quit()
    } catch (error) {
      logger.error('Error closing Redis connection:', { error })
    } finally {
      globalRedisClient = null
    }
  }
}
```

*   `export async function closeRedisConnection(): Promise<void>`: An `async` function to gracefully close the Redis connection. This is particularly important in serverless functions (like AWS Lambda) that might keep "warm" instances alive. Explicitly closing connections ensures resources are released properly.
*   `if (globalRedisClient) { ... }`: Checks if a Redis client instance actually exists.
*   `try { await globalRedisClient.quit() }`: Calls the `quit()` method on the `ioredis` client. This sends a `QUIT` command to the Redis server and then closes the connection.
*   `catch (error) { logger.error(...) }`: Logs any errors that occur during the closing process.
*   `finally { globalRedisClient = null }`: Regardless of whether `quit()` succeeded or failed, `globalRedisClient` is reset to `null`. This prevents attempts to reuse a closed connection and ensures that the next call to `getRedisClient` will create a new one if needed.

---

### Simplified Complex Logic

1.  **Connection Pooling (`getRedisClient`)**: Instead of opening and closing a new connection every time you need Redis, this file ensures that once a connection is established, it's reused for all subsequent operations. Think of it like a carpool: everyone shares the same car to reduce traffic and fuel consumption, rather than everyone driving their own.
2.  **Serverless Optimizations**:
    *   **Keep-Alive**: Like periodically poking someone to make sure they're still awake. This stops cloud providers from cutting off idle connections, allowing reuse.
    *   **Fast Timeouts & Limited Retries**: If Redis isn't responding quickly, the system gives up quickly (5 seconds) and doesn't try too many times (3 attempts per request), freeing up the serverless function to do other work or report an error faster.
    *   **Exponential Backoff**: When retrying connections, it waits a little longer each time, like trying to call someone back after they didn't answer: you don't call them immediately again, you wait a bit longer the second time.
3.  **In-Memory Fallback Cache**: If Redis isn't configured or is completely unavailable, the system doesn't crash. It seamlessly switches to using a local cache (a `Map` in memory) to keep track of processed messages. This is like having a backup notepad when your main computer is down.
4.  **Cache Eviction (`markMessageAsProcessed` for in-memory)**: The in-memory cache has a maximum size. When it gets too full, it first discards entries that have already expired. If it's still too big, it starts removing the oldest non-expired entries, making sure it doesn't consume too much memory.
5.  **Distributed Locking (`acquireLock`)**: This uses a special Redis command (`SET ... NX EX`) that ensures that only *one* process can "claim" a lock at a time. If someone else already has the lock, your attempt fails. The `EX` (expire) part is critical to prevent a process from holding a lock forever if it crashes. This is like a single "Do Not Disturb" sign for a shared resource – only one person can put it up at a time, and it automatically falls down after a certain period if no one takes it down.

This file creates a highly resilient and performant way to manage state across distributed systems, leveraging Redis where available, and falling back gracefully when it's not.