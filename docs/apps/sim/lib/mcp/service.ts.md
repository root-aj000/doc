Okay, here's a comprehensive explanation of the provided TypeScript code, broken down into sections for clarity.

**Purpose of this File**

This TypeScript file defines the `McpService` class, which provides a stateless service for interacting with MCP (Micro Control Plane) servers.  It encapsulates the logic for:

1.  **Discovering Tools:**  Retrieving the list of available tools from one or more MCP servers.
2.  **Executing Tools:**  Calling a specific tool on a specific MCP server and retrieving the result.
3.  **Managing Server Configurations:**  Fetching and processing MCP server configurations from a database, including resolving environment variables.
4.  **Caching:** Implementing a caching mechanism to improve performance by storing the results of tool discovery.
5.  **Error Handling:** Providing robust error handling and logging for MCP operations.
6.  **Summarizing Server Status:**  Getting a summarized status of each MCP server, including connection status and the number of tools available.

**Overall Structure**

The code is structured as a class (`McpService`) with several private helper methods and a few public methods that expose the core functionality. It uses a `Map` for caching, along with periodic cleanup to manage the cache size and remove expired entries.  It also uses Drizzle ORM for database interactions.

**Detailed Code Explanation**

```typescript
/**
 * MCP Service - Clean stateless service for MCP operations
 */
```

*   **Docblock:**  A brief description of the service's purpose.

```typescript
import { db } from '@sim/db'
import { mcpServers } from '@sim/db/schema'
import { and, eq, isNull } from 'drizzle-orm'
import { isTest } from '@/lib/environment'
import { getEffectiveDecryptedEnv } from '@/lib/environment/utils'
import { createLogger } from '@/lib/logs/console/logger'
import { McpClient } from '@/lib/mcp/client'
import type {
  McpServerConfig,
  McpServerSummary,
  McpTool,
  McpToolCall,
  McpToolResult,
  McpTransport,
} from '@/lib/mcp/types'
import { MCP_CONSTANTS } from '@/lib/mcp/utils'
import { generateRequestId } from '@/lib/utils'
```

*   **Imports:** This section imports necessary modules and types:
    *   `db`:  An instance of the Drizzle ORM database client (likely configured elsewhere).
    *   `mcpServers`:  The Drizzle schema definition for the `mcpServers` table in the database.  This table likely stores MCP server configurations.
    *   `and, eq, isNull`: Drizzle ORM functions for building SQL `WHERE` clause conditions. `and` combines multiple conditions, `eq` checks for equality, and `isNull` checks for null values.
    *   `isTest`:  A boolean flag indicating whether the code is running in a test environment.  This is used to disable cleanup during testing.
    *   `getEffectiveDecryptedEnv`: A utility function to retrieve decrypted environment variables, potentially specific to a user and workspace.
    *   `createLogger`:  A function to create a logger instance for logging messages with a specific prefix.
    *   `McpClient`:  A class (likely defined in `@/lib/mcp/client`) that handles communication with an MCP server.
    *   `McpServerConfig, McpServerSummary, McpTool, McpToolCall, McpToolResult, McpTransport`: TypeScript type definitions for MCP-related data structures.  These define the shape of the objects used throughout the service.
    *   `MCP_CONSTANTS`:  A collection of constant values related to MCP, such as cache timeouts.
    *   `generateRequestId`:  A utility function to generate a unique request ID for logging purposes.

```typescript
const logger = createLogger('McpService')
```

*   **Logger Initialization:** Creates a logger instance named "McpService" for logging messages from this service.

```typescript
interface ToolCache {
  tools: McpTool[]
  expiry: Date
  lastAccessed: Date
}

interface CacheStats {
  totalEntries: number
  activeEntries: number
  expiredEntries: number
  maxCacheSize: number
  cacheHitRate: number
  memoryUsage: {
    approximateBytes: number
    entriesEvicted: number
  }
}
```

*   **Interface Declarations:**  Defines two interfaces:
    *   `ToolCache`: Represents a cached list of MCP tools. It includes the `tools` themselves, an `expiry` date (when the cache entry should be considered invalid), and a `lastAccessed` date (used for LRU eviction).
    *   `CacheStats`: Represents statistics about the tool cache, used for monitoring and debugging.

```typescript
class McpService {
  private toolCache = new Map<string, ToolCache>()
  private readonly cacheTimeout = MCP_CONSTANTS.CACHE_TIMEOUT
  private readonly maxCacheSize = 1000
  private cleanupInterval: NodeJS.Timeout | null = null
  private cacheHits = 0
  private cacheMisses = 0
  private entriesEvicted = 0

  constructor() {
    this.startPeriodicCleanup()
  }
```

*   **`McpService` Class:** This is the main class that encapsulates the MCP service logic.
    *   `toolCache`:  A `Map` that stores cached lists of `McpTool` objects. The key is likely a string representing the workspace ID.
    *   `cacheTimeout`:  The duration (in milliseconds) that a cache entry is considered valid.  It's initialized from `MCP_CONSTANTS.CACHE_TIMEOUT`.
    *   `maxCacheSize`: The maximum number of entries allowed in the `toolCache`.
    *   `cleanupInterval`:  A `NodeJS.Timeout` object that stores the interval ID for the periodic cache cleanup. It is initially `null`.
    *   `cacheHits`, `cacheMisses`, `entriesEvicted`: Counters to track cache performance.
    *   `constructor()`: Initializes the service and starts the periodic cache cleanup by calling `this.startPeriodicCleanup()`.

```typescript
  /**
   * Start periodic cleanup of expired cache entries
   */
  private startPeriodicCleanup(): void {
    this.cleanupInterval = setInterval(
      () => {
        this.cleanupExpiredEntries()
      },
      5 * 60 * 1000
    )
  }

  /**
   * Stop periodic cleanup
   */
  private stopPeriodicCleanup(): void {
    if (this.cleanupInterval) {
      clearInterval(this.cleanupInterval)
      this.cleanupInterval = null
    }
  }

  /**
   * Cleanup expired cache entries
   */
  private cleanupExpiredEntries(): void {
    const now = new Date()
    const expiredKeys: string[] = []

    this.toolCache.forEach((cache, key) => {
      if (cache.expiry <= now) {
        expiredKeys.push(key)
      }
    })

    expiredKeys.forEach((key) => this.toolCache.delete(key))

    if (expiredKeys.length > 0) {
      logger.debug(`Cleaned up ${expiredKeys.length} expired cache entries`)
    }
  }
```

*   **Cache Cleanup Methods:**
    *   `startPeriodicCleanup()`: Starts a timer that calls `cleanupExpiredEntries()` every 5 minutes (5 \* 60 \* 1000 milliseconds).  It stores the interval ID in `this.cleanupInterval`.
    *   `stopPeriodicCleanup()`: Stops the periodic cache cleanup by clearing the interval. This is important to prevent memory leaks when the service is no longer needed.
    *   `cleanupExpiredEntries()`: Iterates through the `toolCache`, checks if each entry's `expiry` date is in the past, and deletes expired entries.

```typescript
  /**
   * Evict least recently used entries when cache exceeds max size
   */
  private evictLRUEntries(): void {
    if (this.toolCache.size <= this.maxCacheSize) {
      return
    }

    const entries: { key: string; cache: ToolCache }[] = []
    this.toolCache.forEach((cache, key) => {
      entries.push({ key, cache })
    })
    entries.sort((a, b) => a.cache.lastAccessed.getTime() - b.cache.lastAccessed.getTime())

    const entriesToRemove = this.toolCache.size - this.maxCacheSize + 1
    for (let i = 0; i < entriesToRemove && i < entries.length; i++) {
      this.toolCache.delete(entries[i].key)
      this.entriesEvicted++
    }

    logger.debug(`Evicted ${entriesToRemove} LRU cache entries to maintain size limit`)
  }

  /**
   * Get cache entry and update last accessed time
   */
  private getCacheEntry(key: string): ToolCache | undefined {
    const entry = this.toolCache.get(key)
    if (entry) {
      entry.lastAccessed = new Date()
      this.cacheHits++
      return entry
    }
    this.cacheMisses++
    return undefined
  }

  /**
   * Set cache entry with LRU eviction
   */
  private setCacheEntry(key: string, tools: McpTool[]): void {
    const now = new Date()
    const cache: ToolCache = {
      tools,
      expiry: new Date(now.getTime() + this.cacheTimeout),
      lastAccessed: now,
    }

    this.toolCache.set(key, cache)

    this.evictLRUEntries()
  }

  /**
   * Calculate approximate memory usage of cache
   */
  private calculateMemoryUsage(): number {
    let totalBytes = 0

    this.toolCache.forEach((cache, key) => {
      totalBytes += key.length * 2 // UTF-16 encoding
      totalBytes += JSON.stringify(cache.tools).length * 2
      totalBytes += 64
    })

    return totalBytes
  }
```

*   **Cache Management Methods:**
    *   `evictLRUEntries()`: Implements a Least Recently Used (LRU) eviction policy. If the cache size exceeds `maxCacheSize`, it sorts the cache entries by their `lastAccessed` timestamp and removes the least recently used entries until the cache size is within the limit.
    *   `getCacheEntry(key: string)`: Retrieves a cache entry by its key. If the entry is found, it updates the `lastAccessed` timestamp, increments the `cacheHits` counter, and returns the entry. If the entry is not found, it increments the `cacheMisses` counter and returns `undefined`.
    *   `setCacheEntry(key: string, tools: McpTool[])`: Adds or updates a cache entry.  It creates a new `ToolCache` object with the given tools, an expiry date based on `cacheTimeout`, and the current timestamp as the `lastAccessed` time.  Then, it calls `evictLRUEntries()` to ensure the cache size remains within the limit.
    *   `calculateMemoryUsage()`: Calculates the approximate memory usage of the cache in bytes. This is useful for monitoring and optimizing cache performance. It accounts for the size of the keys and the serialized tool data.

```typescript
  /**
   * Dispose of the service and cleanup resources
   */
  dispose(): void {
    this.stopPeriodicCleanup()
    this.toolCache.clear()
    logger.info('MCP Service disposed and cleanup stopped')
  }

  /**
   * Resolve environment variables in strings
   */
  private resolveEnvVars(value: string, envVars: Record<string, string>): string {
    const envMatches = value.match(/\{\{([^}]+)\}\}/g)
    if (!envMatches) return value

    let resolvedValue = value
    const missingVars: string[] = []

    for (const match of envMatches) {
      const envKey = match.slice(2, -2).trim()
      const envValue = envVars[envKey]

      if (envValue === undefined) {
        missingVars.push(envKey)
        continue
      }

      resolvedValue = resolvedValue.replace(match, envValue)
    }

    if (missingVars.length > 0) {
      throw new Error(
        `Missing required environment variable${missingVars.length > 1 ? 's' : ''}: ${missingVars.join(', ')}. ` +
          `Please set ${missingVars.length > 1 ? 'these variables' : 'this variable'} in your workspace or personal environment settings.`
      )
    }

    return resolvedValue
  }

  /**
   * Resolve environment variables in server config
   */
  private async resolveConfigEnvVars(
    config: McpServerConfig,
    userId: string,
    workspaceId?: string
  ): Promise<McpServerConfig> {
    try {
      const envVars = await getEffectiveDecryptedEnv(userId, workspaceId)

      const resolvedConfig = { ...config }

      if (resolvedConfig.url) {
        resolvedConfig.url = this.resolveEnvVars(resolvedConfig.url, envVars)
      }

      if (resolvedConfig.headers) {
        const resolvedHeaders: Record<string, string> = {}
        for (const [key, value] of Object.entries(resolvedConfig.headers)) {
          resolvedHeaders[key] = this.resolveEnvVars(value, envVars)
        }
        resolvedConfig.headers = resolvedHeaders
      }

      return resolvedConfig
    } catch (error) {
      logger.error('Failed to resolve environment variables for MCP server config:', error)
      return config
    }
  }
```

*   **`dispose()`:**  Cleans up resources when the service is no longer needed. It stops the periodic cleanup, clears the cache, and logs a message.  This is essential for preventing memory leaks and ensuring proper shutdown.
*   **`resolveEnvVars(value: string, envVars: Record<string, string>)`:**  Resolves environment variables within a string. It searches for patterns like `{{ENV_VAR}}` and replaces them with the corresponding values from the `envVars` object. If any environment variables are missing, it throws an error to indicate that the required variables need to be set.
*   **`resolveConfigEnvVars(config: McpServerConfig, userId: string, workspaceId?: string)`:**  Resolves environment variables within an `McpServerConfig` object. It uses `getEffectiveDecryptedEnv` to retrieve the relevant environment variables for the given user and workspace. Then, it calls `resolveEnvVars` to replace any environment variable placeholders in the `url` and `headers` properties of the config.  It handles potential errors during environment variable resolution and logs them.

```typescript
  /**
   * Get server configuration from database
   */
  private async getServerConfig(
    serverId: string,
    workspaceId: string
  ): Promise<McpServerConfig | null> {
    const [server] = await db
      .select()
      .from(mcpServers)
      .where(
        and(
          eq(mcpServers.id, serverId),
          eq(mcpServers.workspaceId, workspaceId),
          eq(mcpServers.enabled, true),
          isNull(mcpServers.deletedAt)
        )
      )
      .limit(1)

    if (!server) {
      return null
    }

    return {
      id: server.id,
      name: server.name,
      description: server.description || undefined,
      transport: server.transport as 'http' | 'sse',
      url: server.url || undefined,
      headers: (server.headers as Record<string, string>) || {},
      timeout: server.timeout || 30000,
      retries: server.retries || 3,
      enabled: server.enabled,
      createdAt: server.createdAt.toISOString(),
      updatedAt: server.updatedAt.toISOString(),
    }
  }

  /**
   * Get all enabled servers for a workspace
   */
  private async getWorkspaceServers(workspaceId: string): Promise<McpServerConfig[]> {
    const whereConditions = [
      eq(mcpServers.workspaceId, workspaceId),
      eq(mcpServers.enabled, true),
      isNull(mcpServers.deletedAt),
    ]

    const servers = await db
      .select()
      .from(mcpServers)
      .where(and(...whereConditions))

    return servers.map((server) => ({
      id: server.id,
      name: server.name,
      description: server.description || undefined,
      transport: server.transport as McpTransport,
      url: server.url || undefined,
      headers: (server.headers as Record<string, string>) || {},
      timeout: server.timeout || 30000,
      retries: server.retries || 3,
      enabled: server.enabled,
      createdAt: server.createdAt.toISOString(),
      updatedAt: server.updatedAt.toISOString(),
    }))
  }
```

*   **Database Interaction Methods:**
    *   `getServerConfig(serverId: string, workspaceId: string)`:  Fetches a single MCP server configuration from the database based on the `serverId` and `workspaceId`.  It uses Drizzle ORM to construct the SQL query with the following conditions:
        *   `mcpServers.id = serverId`
        *   `mcpServers.workspaceId = workspaceId`
        *   `mcpServers.enabled = true` (only enabled servers are retrieved)
        *   `mcpServers.deletedAt is null` (only non-deleted servers are retrieved)
        It limits the result to 1 to retrieve only one server. It transforms the database record into an `McpServerConfig` object before returning it.  The `createdAt` and `updatedAt` fields are converted to ISO strings.
    *   `getWorkspaceServers(workspaceId: string)`:  Retrieves all enabled MCP server configurations for a given `workspaceId` from the database. It uses similar Drizzle ORM logic as `getServerConfig`, but it doesn't filter by `serverId` and doesn't limit the result. It maps the results into an array of `McpServerConfig` objects.

```typescript
  /**
   * Create and connect to an MCP client with security policy
   */
  private async createClient(config: McpServerConfig): Promise<McpClient> {
    const securityPolicy = {
      requireConsent: true,
      auditLevel: 'basic' as const,
      maxToolExecutionsPerHour: 1000,
      allowedOrigins: config.url ? [new URL(config.url).origin] : undefined,
    }

    const client = new McpClient(config, securityPolicy)
    await client.connect()
    return client
  }
```

*   **`createClient(config: McpServerConfig)`:**  Creates and connects to an `McpClient` instance. It defines a `securityPolicy` object that specifies security constraints for the client. The `allowedOrigins` is derived from the server's `url`. The `client.connect()` method is called to establish a connection to the MCP server.

```typescript
  /**
   * Execute a tool on a specific server
   */
  async executeTool(
    userId: string,
    serverId: string,
    toolCall: McpToolCall,
    workspaceId: string
  ): Promise<McpToolResult> {
    const requestId = generateRequestId()

    try {
      logger.info(
        `[${requestId}] Executing MCP tool ${toolCall.name} on server ${serverId} for user ${userId}`
      )

      const config = await this.getServerConfig(serverId, workspaceId)
      if (!config) {
        throw new Error(`Server ${serverId} not found or not accessible`)
      }

      const resolvedConfig = await this.resolveConfigEnvVars(config, userId, workspaceId)

      const client = await this.createClient(resolvedConfig)

      try {
        const result = await client.callTool(toolCall)
        logger.info(`[${requestId}] Successfully executed tool ${toolCall.name}`)
        return result
      } finally {
        await client.disconnect()
      }
    } catch (error) {
      logger.error(
        `[${requestId}] Failed to execute tool ${toolCall.name} on server ${serverId}:`,
        error
      )
      throw error
    }
  }

  /**
   * Discover tools from all workspace servers
   */
  async discoverTools(
    userId: string,
    workspaceId: string,
    forceRefresh = false
  ): Promise<McpTool[]> {
    const requestId = generateRequestId()

    const cacheKey = `workspace:${workspaceId}`

    try {
      if (!forceRefresh) {
        const cached = this.getCacheEntry(cacheKey)
        if (cached && cached.expiry > new Date()) {
          logger.debug(`[${requestId}] Using cached tools for user ${userId}`)
          return cached.tools
        }
      }

      logger.info(`[${requestId}] Discovering MCP tools for workspace ${workspaceId}`)

      const servers = await this.getWorkspaceServers(workspaceId)

      if (servers.length === 0) {
        logger.info(`[${requestId}] No servers found for workspace ${workspaceId}`)
        return []
      }

      const allTools: McpTool[] = []
      const results = await Promise.allSettled(
        servers.map(async (config) => {
          const resolvedConfig = await this.resolveConfigEnvVars(config, userId, workspaceId)
          const client = await this.createClient(resolvedConfig)
          try {
            const tools = await client.listTools()
            logger.debug(
              `[${requestId}] Discovered ${tools.length} tools from server ${config.name}`
            )
            return tools
          } finally {
            await client.disconnect()
          }
        })
      )

      results.forEach((result, index) => {
        if (result.status === 'fulfilled') {
          allTools.push(...result.value)
        } else {
          logger.warn(
            `[${requestId}] Failed to discover tools from server ${servers[index].name}:`,
            result.reason
          )
        }
      })

      this.setCacheEntry(cacheKey, allTools)

      logger.info(
        `[${requestId}] Discovered ${allTools.length} tools from ${servers.length} servers`
      )
      return allTools
    } catch (error) {
      logger.error(`[${requestId}] Failed to discover MCP tools for user ${userId}:`, error)
      throw error
    }
  }

  /**
   * Discover tools from a specific server
   */
  async discoverServerTools(
    userId: string,
    serverId: string,
    workspaceId: string
  ): Promise<McpTool[]> {
    const requestId = generateRequestId()

    try {
      logger.info(`[${requestId}] Discovering tools from server ${serverId} for user ${userId}`)

      const config = await this.getServerConfig(serverId, workspaceId)
      if (!config) {
        throw new Error(`Server ${serverId} not found or not accessible`)
      }

      const resolvedConfig = await this.resolveConfigEnvVars(config, userId, workspaceId)

      const client = await this.createClient(resolvedConfig)

      try {
        const tools = await client.listTools()
        logger.info(`[${requestId}] Discovered ${tools.length} tools from server ${config.name}`)
        return tools
      } finally {
        await client.disconnect()
      }
    } catch (error) {
      logger.error(`[${requestId}] Failed to discover tools from server ${serverId}:`, error)
      throw error
    }
  }
```

*   **Core Functionality Methods:**
    *   `executeTool(userId: string, serverId: string, toolCall: McpToolCall, workspaceId: string)`:  Executes a specific tool on a specific MCP server.
        1.  It generates a unique `requestId` for logging.
        2.  It retrieves the server configuration using `getServerConfig`.
        3.  It resolves environment variables in the configuration using `resolveConfigEnvVars`.
        4.  It creates an `McpClient` using `createClient`.
        5.  It calls the `client.callTool(toolCall)` method to execute the tool.
        6.  It disconnects the client in a `finally` block to ensure that the connection is closed even if an error occurs.
        7.  It logs success or failure messages.
    *   `discoverTools(userId: string, workspaceId: string, forceRefresh = false)`:  Discovers all tools from all enabled MCP servers for a given workspace.
        1.  It generates a unique `requestId` for logging.
        2.  It uses a cache key based on the `workspaceId`.
        3.  It checks the cache for existing tools unless `forceRefresh` is true.
        4.  If the cache is empty or needs to be refreshed, it retrieves all servers for the workspace using `getWorkspaceServers`.
        5.  It iterates through the servers, resolves environment variables, creates an `McpClient`, and calls `client.listTools()` to discover the tools on each server.  `Promise.allSettled` is used to ensure that all server discovery operations complete, even if some fail.
        6.  It aggregates the tools from all servers into a single array.
        7.  It updates the cache with the discovered tools.
    *   `discoverServerTools(userId: string, serverId: string, workspaceId: string)`: Discovers tools from a specific MCP server.  It is similar to `discoverTools`, but it only retrieves the configuration for the specified `serverId` and calls `client.listTools()` on that server.

```typescript
  /**
   * Get server summaries for a user
   */
  async getServerSummaries(userId: string, workspaceId: string): Promise<McpServerSummary[]> {
    const requestId = generateRequestId()

    try {
      logger.info(`[${requestId}] Getting server summaries for workspace ${workspaceId}`)

      const servers = await this.getWorkspaceServers(workspaceId)
      const summaries: McpServerSummary[] = []

      for (const config of servers) {
        try {
          const resolvedConfig = await this.resolveConfigEnvVars(config, userId, workspaceId)
          const client = await this.createClient(resolvedConfig)
          const tools = await client.listTools()
          await client.disconnect()

          summaries.push({
            id: config.id,
            name: config.name,
            url: config.url,
            transport: config.transport,
            status: 'connected',
            toolCount: tools.length,
            lastSeen: new Date(),
            error: undefined,
          })
        } catch (error) {
          summaries.push({
            id: config.id,
            name: config.name,
            url: config.url,
            transport: config.transport,
            status: 'error',
            toolCount: 0,
            lastSeen: undefined,
            error: error instanceof Error ? error.message : 'Connection failed',
          })
        }
      }

      return summaries
    } catch (error) {
      logger.error(`[${requestId}] Failed to get server summaries for user ${userId}:`, error)
      throw error
    }
  }

  /**
   * Clear tool cache for a workspace or all workspaces
   */
  clearCache(workspaceId?: string): void {
    if (workspaceId) {
      const workspaceCacheKey = `workspace:${workspaceId}`
      this.toolCache.delete(workspaceCacheKey)
      logger.debug(`Cleared MCP tool cache for workspace ${workspaceId}`)
    } else {
      this.toolCache.clear()
      this.cacheHits = 0
      this.cacheMisses = 0
      this.entriesEvicted = 0
      logger.debug('Cleared all MCP tool cache and reset statistics')
    }
  }

  /**
   * Get comprehensive cache statistics
   */
  getCacheStats(): CacheStats {
    const entries: { key: string; cache: ToolCache }[] = []
    this.toolCache.forEach((cache, key) => {
      entries.push({ key, cache })
    })

    const now = new Date()
    const activeEntries = entries.filter(({ cache }) => cache.expiry > now)
    const totalRequests = this.cacheHits + this.cacheMisses
    const hitRate = totalRequests > 0 ? this.cacheHits / totalRequests : 0

    return {
      totalEntries: entries.length,
      activeEntries: activeEntries.length,
      expiredEntries: entries.length - activeEntries.length,
      maxCacheSize: this.maxCacheSize,
      cacheHitRate: Math.round(hitRate * 100) / 100,
      memoryUsage: {
        approximateBytes: this.calculateMemoryUsage(),
        entriesEvicted: this.entriesEvicted,
      },
    }
  }
}
```

*   **Utility and Management Methods:**
    *   `getServerSummaries(userId: string, workspaceId: string)`:  Retrieves summaries of all MCP servers for a given workspace. It gets the list of servers, then attempts to connect to each server and retrieve the list of tools.  It constructs an `McpServerSummary` object for each server, indicating its status (connected or error), the number of tools, and any error message.
    *   `clearCache(workspaceId?: string)`: Clears the tool cache.  If a `workspaceId` is provided, it only clears the cache for that workspace.  Otherwise, it clears the entire cache.
    *   `getCacheStats()`:  Returns statistics about the cache, including the total number of entries, the number of active entries, the cache hit rate, and the approximate memory usage. This is useful for monitoring and tuning the cache.

```typescript
export const mcpService = new McpService()

/**
 * Setup process signal handlers for graceful shutdown
 */
export function setupMcpServiceCleanup() {
  if (isTest) {
    return
  }

  const cleanup = () => {
    mcpService.dispose()
  }

  process.on('SIGTERM', cleanup)
  process.on('SIGINT', cleanup)

  return () => {
    process.removeListener('SIGTERM', cleanup)
    process.removeListener('SIGINT', cleanup)
  }
}
```

*   **Service Instance and Shutdown Handling:**
    *   `export const mcpService = new McpService()`: Creates a singleton instance of the `McpService` and exports it. This makes the service available for use in other modules.
    *   `setupMcpServiceCleanup()`: Sets up signal handlers to gracefully shut down the `McpService` when the process receives a `SIGTERM` or `SIGINT` signal (e.g., when the application is being stopped). It calls the `mcpService.dispose()` method to clean up resources.  The `isTest` check prevents this cleanup from running during tests. The function returns a cleanup function that removes the signal listeners, this is good practice to avoid memory leaks if the function is called multiple times.

**Simplifying Complex Logic**

Here are some ways the code simplifies complex logic:

*   **Encapsulation:**  The `McpService` class encapsulates all the logic for interacting with MCP servers, hiding the complexity from the calling code.
*   **Abstraction:**  The `McpClient` class provides an abstraction over the underlying MCP communication protocol.
*   **Caching:**  The caching mechanism avoids redundant calls to MCP servers, improving performance.
*   **Error Handling:**  The code includes comprehensive error handling and logging, making it easier to debug and troubleshoot problems.
*   **Configuration Management:**  The code handles the retrieval and processing of MCP server configurations, including resolving environment variables.

**Key Improvements and Considerations**

*   **Caching Strategy:** The current cache invalidation strategy is based solely on time. Consider more sophisticated invalidation strategies, such as invalidating the cache when the MCP server configuration changes.
*   **Error Handling:** The code includes error handling, but consider adding more specific error handling for different types of errors. For example, distinguish between network errors, authentication errors, and MCP server errors.  Using custom error classes could improve clarity.
*   **Security:** The `securityPolicy` in `createClient` is a good start, but it might need further refinement based on the specific security requirements of the MCP environment.  Consider more granular control over allowed tools and actions.
*   **Asynchronous Operations:**  The code makes extensive use of asynchronous operations (async/await), which is important for performance and responsiveness.  Ensure that all asynchronous operations are properly handled and that errors are caught.
*   **Logging:** The code includes logging, which is essential for debugging and monitoring. Consider using a more structured logging format (e.g., JSON) for easier analysis.
*   **Testing:**  Write unit tests and integration tests to ensure the correctness and reliability of the `McpService`.  Pay particular attention to testing error handling and caching.

I hope this detailed explanation is helpful! Let me know if you have any further questions.
