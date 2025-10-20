As a TypeScript expert and technical writer, I'll walk you through this code step-by-step, breaking down its purpose, simplifying complex parts, and explaining each line in detail.

---

## TypeScript Code Explanation: Usage Monitoring and Billing Enforcement

This TypeScript file is a crucial component for managing and enforcing user and organization-level usage limits within a software application. It acts as a central hub for checking how much a user (and their team) has consumed against their allocated budget or plan, determining if they are approaching or have exceeded these limits, and facilitating appropriate responses, such as notifications or blocking actions.

It handles various scenarios, including:
1.  **Individual User Limits:** Monitoring a single user's consumption.
2.  **Organization/Team Limits (Pooled Usage):** Aggregating usage across an entire team within an organization and checking it against a collective limit.
3.  **Billing Status:** Adapting behavior based on whether the billing system is enabled or disabled.
4.  **Notifications:** Triggering client-side events for warnings or limit exceedances.
5.  **Server-Side Enforcement:** Providing a robust check for API routes and background processes to prevent actions when limits are exceeded.

---

### File Structure and Imports

```typescript
import { db } from '@sim/db'
import { member, organization, userStats } from '@sim/db/schema'
import { eq, inArray } from 'drizzle-orm'
import { getOrganizationSubscription, getPlanPricing } from '@/lib/billing/core/billing'
import { getUserUsageLimit } from '@/lib/billing/core/usage'
import { isBillingEnabled } from '@/lib/environment'
import { createLogger } from '@/lib/logs/console/logger'

const logger = createLogger('UsageMonitor')

const WARNING_THRESHOLD = 80
```

#### Explanation of Imports:

*   `import { db } from '@sim/db'`: This line imports the Drizzle ORM database instance, typically configured to connect to your PostgreSQL (or other) database. `db` is your primary interface for database operations like selecting, inserting, updating, and deleting records.
*   `import { member, organization, userStats } from '@sim/db/schema'`: These imports bring in the schema definitions for three database tables:
    *   `member`: Likely stores information about users' memberships in organizations (e.g., `userId`, `organizationId`).
    *   `organization`: Contains details about organizations, including their own usage limits (`orgUsageLimit`).
    *   `userStats`: Holds individual user statistics, particularly their current usage costs (`currentPeriodCost`, `totalCost`) and billing status (`billingBlocked`).
*   `import { eq, inArray } from 'drizzle-orm'`: These are utility functions from the Drizzle ORM library:
    *   `eq`: Used to create an "equals" condition in database queries (e.g., `WHERE userId = 'abc'`).
    *   `inArray`: Used to create an "in array" condition (e.g., `WHERE userId IN ('abc', 'def')`), highly useful for efficient queries involving multiple IDs.
*   `import { getOrganizationSubscription, getPlanPricing } from '@/lib/billing/core/billing'`: These imports bring in functions related to your billing system (likely Stripe or a similar provider):
    *   `getOrganizationSubscription`: Fetches details about an organization's active subscription.
    *   `getPlanPricing`: Retrieves pricing information for different billing plans (e.g., 'free', 'team', 'enterprise').
*   `import { getUserUsageLimit } from '@/lib/billing/core/usage'`: Imports a function specifically designed to calculate or retrieve an individual user's usage limit. This might involve looking at their plan or other user-specific settings.
*   `import { isBillingEnabled } from '@/lib/environment'`: A simple boolean flag imported from an environment configuration file. It controls whether the billing and usage limit checks are active or bypassed.
*   `import { createLogger } from '@/lib/logs/console/logger'`: Imports a factory function for creating a logger instance, used for outputting informational, warning, and error messages to the console or a logging service.

#### Global Declarations:

*   `const logger = createLogger('UsageMonitor')`: Initializes a logger instance specifically for this module, named 'UsageMonitor'. This helps in tracing logs back to their source.
*   `const WARNING_THRESHOLD = 80`: Defines a constant percentage. If a user's usage reaches or exceeds this percentage (e.g., 80%), they will trigger a "warning" status, indicating they are approaching their limit.

---

### `UsageData` Interface

```typescript
interface UsageData {
  percentUsed: number
  isWarning: boolean
  isExceeded: boolean
  currentUsage: number
  limit: number
}
```

#### Explanation of `UsageData` Interface:

This interface defines the structure of the object returned by the `checkUsageStatus` function. It standardizes the usage information, making it easy to understand and use throughout the application.

*   `percentUsed: number`: The percentage of the allocated limit that has been consumed. For example, `50` means 50% of the limit is used.
*   `isWarning: boolean`: A flag indicating `true` if the user is approaching their limit (i.e., `percentUsed` is above `WARNING_THRESHOLD` but below 100%).
*   `isExceeded: boolean`: A flag indicating `true` if the user has consumed 100% or more of their allocated limit.
*   `currentUsage: number`: The actual amount of usage (e.g., cost in dollars) incurred by the user or team in the current billing period.
*   `limit: number`: The total allocated limit (e.g., cost in dollars) for the user or organization.

---

### `checkUsageStatus` Function (Core Logic)

```typescript
/**
 * Checks a user's cost usage against their subscription plan limit
 * and returns usage information including whether they're approaching the limit
 */
export async function checkUsageStatus(userId: string): Promise<UsageData> {
  try {
    // If billing is disabled, always return permissive limits
    if (!isBillingEnabled) {
      // Get actual usage from the database for display purposes
      const statsRecords = await db.select().from(userStats).where(eq(userStats.userId, userId))
      const currentUsage =
        statsRecords.length > 0
          ? Number.parseFloat(statsRecords[0].currentPeriodCost?.toString())
          : 0

      return {
        percentUsed: Math.min((currentUsage / 1000) * 100, 100),
        isWarning: false,
        isExceeded: false,
        currentUsage,
        limit: 1000,
      }
    }

    // ... (rest of the function) ...

  } catch (error) {
    logger.error('Error checking usage status', {
      error: error instanceof Error ? { message: error.message, stack: error.stack } : error,
      userId,
    })

    // Block execution if we can't determine usage status
    logger.error('Cannot determine usage status - blocking execution', {
      userId,
      error: error instanceof Error ? error.message : String(error),
    })

    return {
      percentUsed: 100,
      isWarning: false,
      isExceeded: true, // Block execution when we can't determine status
      currentUsage: 0,
      limit: 0, // Zero limit forces blocking
    }
  }
}
```

#### Purpose of `checkUsageStatus`:

This is the primary function responsible for calculating and returning the usage status for a given user. It takes a `userId` as input and returns a `Promise` that resolves to a `UsageData` object. It's designed to be robust, handling cases where billing is disabled, where user stats don't exist, and calculating both individual and organizational pooled usage limits.

#### Line-by-Line Explanation of `checkUsageStatus`:

*   `export async function checkUsageStatus(userId: string): Promise<UsageData> {`: Defines an asynchronous function `checkUsageStatus` that accepts a `userId` string and promises to return a `UsageData` object. The `export` keyword makes it available for other modules to import and use.
*   `try { ... } catch (error) { ... }`: This is a `try-catch` block for comprehensive error handling. If any error occurs within the `try` block, execution jumps to the `catch` block.
    *   **Error Handling in `catch`:**
        *   `logger.error('Error checking usage status', { ... })`: Logs a detailed error message, including the `userId` and the error object itself. It smartly extracts `message` and `stack` from `Error` instances for better debugging.
        *   `logger.error('Cannot determine usage status - blocking execution', { ... })`: This is a critical safety measure. If the system *cannot reliably determine* the user's usage status due to an error (e.g., database connection issues, misconfiguration), it assumes a "worst-case" scenario and blocks execution.
        *   `return { percentUsed: 100, isWarning: false, isExceeded: true, currentUsage: 0, limit: 0, }`: In a critical error scenario, it returns `isExceeded: true` and `limit: 0`. A zero limit ensures that any subsequent check against this data will interpret the user as having exceeded their limit, thus effectively blocking further actions until the issue is resolved.

---

#### `checkUsageStatus` - Billing Disabled Logic:

```typescript
    // If billing is disabled, always return permissive limits
    if (!isBillingEnabled) {
      // Get actual usage from the database for display purposes
      const statsRecords = await db.select().from(userStats).where(eq(userStats.userId, userId))
      const currentUsage =
        statsRecords.length > 0
          ? Number.parseFloat(statsRecords[0].currentPeriodCost?.toString())
          : 0

      return {
        percentUsed: Math.min((currentUsage / 1000) * 100, 100),
        isWarning: false,
        isExceeded: false,
        currentUsage,
        limit: 1000,
      }
    }
```

*   `if (!isBillingEnabled)`: Checks if the global `isBillingEnabled` flag is `false`. If billing is turned off (e.g., for development, testing, or a free-tier app), the system should not enforce limits.
*   `const statsRecords = await db.select().from(userStats).where(eq(userStats.userId, userId))`: Even when billing is disabled, it fetches the user's statistics from the `userStats` table. This is done to retrieve their *actual current usage* so it can still be displayed to the user, even if limits aren't enforced.
*   `const currentUsage = statsRecords.length > 0 ? Number.parseFloat(statsRecords[0].currentPeriodCost?.toString()) : 0`:
    *   Checks if any `statsRecords` were found.
    *   If yes, it attempts to parse the `currentPeriodCost` from the first record. `statsRecords[0].currentPeriodCost?.toString()` safely converts the cost (which might be a `number` or `Decimal` type from the DB) to a string before `Number.parseFloat` converts it to a float. The `?.` (optional chaining) handles cases where `currentPeriodCost` might be `null` or `undefined`.
    *   If no records or `currentPeriodCost` is missing, `currentUsage` defaults to `0`.
*   `return { ... }`: Returns a `UsageData` object with permissive settings:
    *   `percentUsed: Math.min((currentUsage / 1000) * 100, 100)`: Calculates percentage based on a fixed, large limit (1000). `Math.min(..., 100)` ensures the percentage doesn't go over 100% in display, even if `currentUsage` exceeds 1000 (though `isExceeded` will still be `false` here).
    *   `isWarning: false`, `isExceeded: false`: No warnings or exceedance when billing is disabled.
    *   `currentUsage`: The actual usage fetched from the DB.
    *   `limit: 1000`: A fixed, generous limit when billing is off.

---

#### `checkUsageStatus` - Billing Enabled Logic (Individual User):

```typescript
    // Get usage limit from user_stats (per-user cap)
    const limit = await getUserUsageLimit(userId)
    logger.info('Using stored usage limit', { userId, limit })

    // Get actual usage from the database
    const statsRecords = await db.select().from(userStats).where(eq(userStats.userId, userId))

    // If no stats record exists, create a default one
    if (statsRecords.length === 0) {
      logger.info('No usage stats found for user', { userId, limit })

      return {
        percentUsed: 0,
        isWarning: false,
        isExceeded: false,
        currentUsage: 0,
        limit,
      }
    }

    // Get the current period cost from the user stats (use currentPeriodCost if available, fallback to totalCost)
    const currentUsage = Number.parseFloat(
      statsRecords[0].currentPeriodCost?.toString() || statsRecords[0].totalCost.toString()
    )

    // Calculate percentage used
    const percentUsed = Math.min((currentUsage / limit) * 100, 100)

    // Check org-level cap for team/enterprise pooled usage
    let isExceeded = currentUsage >= limit
    let isWarning = percentUsed >= WARNING_THRESHOLD && percentUsed < 100
```

*   `const limit = await getUserUsageLimit(userId)`: Calls an external helper function to determine the individual user's usage limit. This limit is specific to the user, potentially based on their individual plan.
*   `logger.info('Using stored usage limit', { userId, limit })`: Logs the retrieved individual limit for debugging.
*   `const statsRecords = await db.select().from(userStats).where(eq(userStats.userId, userId))`: Fetches the user's usage statistics from the database.
*   `if (statsRecords.length === 0) { ... }`: If no `userStats` record is found for the given `userId`:
    *   `logger.info('No usage stats found for user', { userId, limit })`: Logs this event, indicating a new user or a data inconsistency.
    *   `return { ... }`: Returns a `UsageData` object with `0` usage, `false` for warning/exceeded. This assumes a new user starts with no usage and no immediate limit issues.
*   `const currentUsage = Number.parseFloat(statsRecords[0].currentPeriodCost?.toString() || statsRecords[0].totalCost.toString())`:
    *   This is a robust way to get the current period's cost.
    *   It first tries `currentPeriodCost`. If that's `null` or `undefined` (using `?.toString()`), it falls back to `totalCost.toString()`. This ensures a value is always obtained if a record exists.
    *   `Number.parseFloat()` converts the string representation of the cost into a floating-point number.
*   `const percentUsed = Math.min((currentUsage / limit) * 100, 100)`: Calculates the percentage of the `limit` used. `Math.min(..., 100)` ensures the displayed percentage never exceeds 100%, even if `currentUsage` is greater than `limit`.
*   `let isExceeded = currentUsage >= limit`: Initializes `isExceeded` to `true` if the `currentUsage` meets or exceeds the *individual* `limit`. This `let` declaration indicates it might be updated later (e.g., by organization-level checks).
*   `let isWarning = percentUsed >= WARNING_THRESHOLD && percentUsed < 100`: Initializes `isWarning` to `true` if the `percentUsed` is above the `WARNING_THRESHOLD` but still below 100%. Like `isExceeded`, this can be updated later.

---

#### `checkUsageStatus` - Organization-Level Cap (Pooled Usage):

This is the most complex part, where the system checks if a user's *organization* (or team) has exceeded its collective usage limit.

```typescript
    try {
      const memberships = await db
        .select({ organizationId: member.organizationId })
        .from(member)
        .where(eq(member.userId, userId))
      if (memberships.length > 0) {
        for (const m of memberships) {
          const orgRows = await db
            .select({ id: organization.id, orgUsageLimit: organization.orgUsageLimit })
            .from(organization)
            .where(eq(organization.id, m.organizationId))
            .limit(1)
          if (orgRows.length) {
            const org = orgRows[0]
            // Sum pooled usage
            const teamMembers = await db
              .select({ userId: member.userId })
              .from(member)
              .where(eq(member.organizationId, org.id))

            // Get all team member usage in a single query to avoid N+1
            let pooledUsage = 0
            if (teamMembers.length > 0) {
              const memberIds = teamMembers.map((tm) => tm.userId)
              const allMemberStats = await db
                .select({ current: userStats.currentPeriodCost, total: userStats.totalCost })
                .from(userStats)
                .where(inArray(userStats.userId, memberIds))

              for (const stats of allMemberStats) {
                pooledUsage += Number.parseFloat(
                  stats.current?.toString() || stats.total.toString()
                )
              }
            }
            // Determine org cap
            let orgCap = org.orgUsageLimit ? Number.parseFloat(String(org.orgUsageLimit)) : 0
            if (!orgCap || Number.isNaN(orgCap)) {
              // Fall back to minimum billing amount from Stripe subscription
              const orgSub = await getOrganizationSubscription(org.id)
              if (orgSub?.seats) {
                const { basePrice } = getPlanPricing(orgSub.plan)
                orgCap = (orgSub.seats || 1) * basePrice
              } else {
                // If no subscription, use team default
                const { basePrice } = getPlanPricing('team')
                orgCap = basePrice // Default to 1 seat minimum
              }
            }
            if (pooledUsage >= orgCap) {
              isExceeded = true
              isWarning = false
              break // If org limit is exceeded, no need to check other orgs
            }
          }
        }
      }
    } catch (error) {
      logger.warn('Error checking organization usage limits', { error, userId })
    }
```

*   `try { ... } catch (error) { ... }`: This inner `try-catch` block specifically wraps the organization-level checks. This means that if there's an error calculating organization limits (e.g., issues with the billing service), it won't block the entire `checkUsageStatus` function; it will just log a warning and proceed with the individual user's status.
*   `const memberships = await db.select({ organizationId: member.organizationId }).from(member).where(eq(member.userId, userId))`: Queries the `member` table to find all organizations the current `userId` belongs to. It only selects the `organizationId`.
*   `if (memberships.length > 0) { for (const m of memberships) { ... } }`: If the user is part of one or more organizations, it iterates through each `membership`.
*   `const orgRows = await db.select({ id: organization.id, orgUsageLimit: organization.orgUsageLimit }).from(organization).where(eq(organization.id, m.organizationId)).limit(1)`: For each `organizationId` the user belongs to, it fetches details about that organization from the `organization` table, specifically its `id` and `orgUsageLimit`. `limit(1)` is used because we only expect one record for a given `organizationId`.
*   `if (orgRows.length) { const org = orgRows[0] ... }`: If an organization record is found:
    *   `const org = orgRows[0]`: Extracts the organization object.
    *   `const teamMembers = await db.select({ userId: member.userId }).from(member).where(eq(member.organizationId, org.id))`: Retrieves all user IDs (team members) that belong to the current `org.id`.
    *   `let pooledUsage = 0`: Initializes a variable to accumulate the total usage of all team members.
    *   `if (teamMembers.length > 0) { ... }`: If there are team members:
        *   `const memberIds = teamMembers.map((tm) => tm.userId)`: Creates an array of `userId`s for all team members.
        *   `const allMemberStats = await db.select({ current: userStats.currentPeriodCost, total: userStats.totalCost }).from(userStats).where(inArray(userStats.userId, memberIds))`: **This is an N+1 query optimization.** Instead of querying the database for each `teamMember`, it fetches all their `userStats` records in a single efficient query using `inArray`. This is much better for performance than running a separate query for each member.
        *   `for (const stats of allMemberStats) { pooledUsage += Number.parseFloat(stats.current?.toString() || stats.total.toString()) }`: Iterates through the fetched `allMemberStats` and sums up their `currentPeriodCost` (or `totalCost` as fallback) to calculate the `pooledUsage` for the entire organization.
    *   `let orgCap = org.orgUsageLimit ? Number.parseFloat(String(org.orgUsageLimit)) : 0`: Determines the organization's usage cap. It first checks `org.orgUsageLimit` from the database. It converts this to a number, defaulting to `0` if it's missing.
    *   `if (!orgCap || Number.isNaN(orgCap)) { ... }`: If the `orgCap` from the database is not set or is an invalid number, it falls back to determining the cap based on the organization's subscription plan.
        *   `const orgSub = await getOrganizationSubscription(org.id)`: Fetches the organization's subscription details (e.g., from Stripe).
        *   `if (orgSub?.seats) { const { basePrice } = getPlanPricing(orgSub.plan); orgCap = (orgSub.seats || 1) * basePrice }`: If a subscription exists and has `seats` information, it calculates the `orgCap` by multiplying the number of seats by the `basePrice` of their plan. It defaults to `1` seat if `orgSub.seats` is `null`/`undefined`.
        *   `else { const { basePrice } = getPlanPricing('team'); orgCap = basePrice }`: If there's no active subscription for the organization, it falls back to a default "team" plan's base price as the `orgCap`, ensuring a minimum limit.
    *   `if (pooledUsage >= orgCap) { isExceeded = true; isWarning = false; break }`: If the `pooledUsage` for the organization meets or exceeds the `orgCap`:
        *   `isExceeded = true`: Sets the `isExceeded` flag to `true`.
        *   `isWarning = false`: An exceedance takes precedence over a warning, so `isWarning` is set to `false`.
        *   `break`: Exits the loop over memberships. Once *any* organization the user belongs to has exceeded its limit, the user is considered blocked, and there's no need to check other organizations.
*   `logger.warn('Error checking organization usage limits', { error, userId })`: Logs a warning if an error occurred during the organization checks. This warning is less severe than the outer `catch` because it allows the function to still return *some* usage data based on the individual limit, rather than blocking entirely.

---

#### `checkUsageStatus` - Final Return:

```typescript
    logger.info('Final usage statistics', {
      userId,
      currentUsage,
      limit,
      percentUsed,
      isWarning,
      isExceeded,
    })

    return {
      percentUsed,
      isWarning,
      isExceeded,
      currentUsage,
      limit,
    }
```

*   `logger.info('Final usage statistics', { ... })`: Logs the final calculated usage statistics, which is helpful for auditing and debugging.
*   `return { ... }`: Returns the final `UsageData` object, which now incorporates both individual and organization-level limit checks.

---

### `checkAndNotifyUsage` Function (Client-Side Notifications)

```typescript
/**
 * Displays a notification to the user when they're approaching their usage limit
 * Can be called on app startup or before executing actions that might incur costs
 */
export async function checkAndNotifyUsage(userId: string): Promise<void> {
  try {
    // Skip usage notifications if billing is disabled
    if (!isBillingEnabled) {
      return
    }

    const usageData = await checkUsageStatus(userId)

    if (usageData.isExceeded) {
      // User has exceeded their limit
      logger.warn('User has exceeded usage limits', {
        userId,
        usage: usageData.currentUsage,
        limit: usageData.limit,
      })

      // Dispatch event to show a UI notification
      if (typeof window !== 'undefined') {
        window.dispatchEvent(
          new CustomEvent('usage-exceeded', {
            detail: { usageData },
          })
        )
      }
    } else if (usageData.isWarning) {
      // User is approaching their limit
      logger.info('User approaching usage limits', {
        userId,
        usage: usageData.currentUsage,
        limit: usageData.limit,
        percent: usageData.percentUsed,
      })

      // Dispatch event to show a UI notification
      if (typeof window !== 'undefined') {
        window.dispatchEvent(
          new CustomEvent('usage-warning', {
            detail: { usageData },
          })
        )

        // Optionally open the subscription tab in settings
        window.dispatchEvent(
          new CustomEvent('open-settings', {
            detail: { tab: 'subscription' },
          })
        )
      }
    }
  } catch (error) {
    logger.error('Error in usage notification system', { error, userId })
  }
}
```

#### Purpose of `checkAndNotifyUsage`:

This function is designed to be called on the client-side (e.g., in a web browser) to check a user's usage status and, if necessary, dispatch custom browser events that can trigger UI notifications or other client-side actions. It is typically invoked on application startup or before operations that might incur significant costs.

#### Line-by-Line Explanation of `checkAndNotifyUsage`:

*   `export async function checkAndNotifyUsage(userId: string): Promise<void> {`: Defines an asynchronous function that takes a `userId` and doesn't return any specific value (`void`).
*   `try { ... } catch (error) { ... }`: Standard error handling for the notification system. If an error occurs, it's logged, but the application continues.
*   `if (!isBillingEnabled) { return }`: Skips all notification logic if billing is disabled, as there are no limits to warn about.
*   `const usageData = await checkUsageStatus(userId)`: Calls the core `checkUsageStatus` function to get the detailed usage information.
*   `if (usageData.isExceeded) { ... }`: If the user has exceeded their limits:
    *   `logger.warn('User has exceeded usage limits', { ... })`: Logs a warning message.
    *   `if (typeof window !== 'undefined') { ... }`: This is a crucial client-side check. `window` is a global object available in browser environments. This `if` condition ensures that the code trying to dispatch browser events only runs when it's actually in a browser, preventing errors if this code is accidentally run on the server (e.g., during server-side rendering).
    *   `window.dispatchEvent(new CustomEvent('usage-exceeded', { detail: { usageData }, }))`: Dispatches a `CustomEvent` named `usage-exceeded`. This event can be listened to by other parts of the client-side application (e.g., a UI component) to display a banner, modal, or toast notification to the user, informing them they've exceeded their limits. The `detail` property carries the `usageData` for context.
*   `else if (usageData.isWarning) { ... }`: If the user is *approaching* their limits (warning threshold met):
    *   `logger.info('User approaching usage limits', { ... })`: Logs an informational message.
    *   `if (typeof window !== 'undefined') { ... }`: Again, the client-side check.
    *   `window.dispatchEvent(new CustomEvent('usage-warning', { detail: { usageData }, }))`: Dispatches a `CustomEvent` named `usage-warning` for client-side UI notification.
    *   `window.dispatchEvent(new CustomEvent('open-settings', { detail: { tab: 'subscription' }, }))`: Optionally dispatches another `CustomEvent` to prompt the UI to open the settings page, specifically to the "subscription" tab, encouraging the user to upgrade.
*   `logger.error('Error in usage notification system', { error, userId })`: Logs any errors encountered within the notification logic.

---

### `checkServerSideUsageLimits` Function (Server-Side Enforcement)

```typescript
/**
 * Server-side function to check if a user has exceeded their usage limits
 * For use in API routes, webhooks, and scheduled executions
 *
 * @param userId The ID of the user to check
 * @returns An object containing the exceeded status and usage details
 */
export async function checkServerSideUsageLimits(userId: string): Promise<{
  isExceeded: boolean
  currentUsage: number
  limit: number
  message?: string
}> {
  try {
    if (!isBillingEnabled) {
      return {
        isExceeded: false,
        currentUsage: 0,
        limit: 99999,
      }
    }

    logger.info('Server-side checking usage limits for user', { userId })

    const stats = await db
      .select({
        blocked: userStats.billingBlocked,
        current: userStats.currentPeriodCost,
        total: userStats.totalCost,
      })
      .from(userStats)
      .where(eq(userStats.userId, userId))
      .limit(1)
    if (stats.length > 0 && stats[0].blocked) {
      const currentUsage = Number.parseFloat(
        stats[0].current?.toString() || stats[0].total.toString()
      )
      return {
        isExceeded: true,
        currentUsage,
        limit: 0,
        message: 'Billing issue detected. Please update your payment method to continue.',
      }
    }

    const usageData = await checkUsageStatus(userId)

    return {
      isExceeded: usageData.isExceeded,
      currentUsage: usageData.currentUsage,
      limit: usageData.limit,
      message: usageData.isExceeded
        ? `Usage limit exceeded: ${usageData.currentUsage?.toFixed(2) || 0}$ used of ${usageData.limit?.toFixed(2) || 0}$ limit. Please upgrade your plan to continue.`
        : undefined,
    }
  } catch (error) {
    logger.error('Error in server-side usage limit check', {
      error: error instanceof Error ? { message: error.message, stack: error.stack } : error,
      userId,
    })

    logger.error('Cannot determine usage limits - blocking execution', {
      userId,
      error: error instanceof Error ? error.message : String(error),
    })

    return {
      isExceeded: true, // Block execution when we can't determine limits
      currentUsage: 0,
      limit: 0, // Zero limit forces blocking
      message:
        error instanceof Error && error.message.includes('No user stats record found')
          ? 'User account not properly initialized. Please contact support.'
          : 'Unable to determine usage limits. Execution blocked for security. Please contact support.',
    }
  }
}
```

#### Purpose of `checkServerSideUsageLimits`:

This function provides a dedicated server-side check for usage limits. Unlike `checkAndNotifyUsage` (which focuses on client-side alerts), this function is intended for critical decision points on the backend, such as:
*   Before processing an API request that incurs cost.
*   Before executing a background job.
*   As part of a webhook handler.

Its primary goal is to return a clear `isExceeded` status along with a user-friendly message, allowing server-side logic to either proceed or block an action.

#### Line-by-Line Explanation of `checkServerSideUsageLimits`:

*   `export async function checkServerSideUsageLimits(userId: string): Promise<{ ... }> {`: Defines an asynchronous function that takes a `userId` and returns a `Promise` resolving to an object containing `isExceeded`, `currentUsage`, `limit`, and an optional `message`.
*   `try { ... } catch (error) { ... }`: Standard `try-catch` for robust error handling.
    *   **Error Handling in `catch`:** Similar to `checkUsageStatus`, but with more specific messages. It logs the error and crucially returns `isExceeded: true` with a `limit: 0` (blocking behavior) if it cannot reliably determine the usage status. It also provides specific error messages based on common scenarios like "No user stats record found."
*   `if (!isBillingEnabled) { return { isExceeded: false, currentUsage: 0, limit: 99999, } }`: If billing is disabled, it immediately returns a permissive status: `isExceeded: false`, with a very high `limit`, ensuring no blocking occurs.
*   `logger.info('Server-side checking usage limits for user', { userId })`: Logs that a server-side check is being performed.
*   `const stats = await db.select({ blocked: userStats.billingBlocked, current: userStats.currentPeriodCost, total: userStats.totalCost, }).from(userStats).where(eq(userStats.userId, userId)).limit(1)`: Directly queries `userStats` for the `billingBlocked` flag, `currentPeriodCost`, and `totalCost`. This is a quick check for explicit billing issues.
*   `if (stats.length > 0 && stats[0].blocked) { ... }`: If a `userStats` record exists and its `billingBlocked` flag is `true`:
    *   `const currentUsage = Number.parseFloat(stats[0].current?.toString() || stats[0].total.toString())`: Gets the user's current usage for context.
    *   `return { isExceeded: true, currentUsage, limit: 0, message: 'Billing issue detected. Please update your payment method to continue.', }`: Returns `isExceeded: true` immediately, regardless of actual usage figures, as the user is explicitly blocked due to a billing issue. `limit: 0` enforces this. A specific message is provided.
*   `const usageData = await checkUsageStatus(userId)`: If not explicitly blocked by `billingBlocked`, it calls the comprehensive `checkUsageStatus` function to get the full usage calculation.
*   `return { ... }`: Constructs and returns the final result:
    *   `isExceeded: usageData.isExceeded`: Directly uses the result from `checkUsageStatus`.
    *   `currentUsage: usageData.currentUsage`, `limit: usageData.limit`: Provides the usage figures.
    *   `message: usageData.isExceeded ? ... : undefined`: If `isExceeded` is `true`, it generates a detailed message indicating the usage amount and limit, suggesting an upgrade. Otherwise, no message is provided.

---

### Conclusion

This file provides a robust and layered approach to usage monitoring and billing enforcement. By separating concerns into different functions for core calculation (`checkUsageStatus`), client-side notifications (`checkAndNotifyUsage`), and server-side blocking (`checkServerSideUsageLimits`), it offers flexibility and safety. The detailed error handling, N+1 query optimization, and fallbacks for subscription data demonstrate a well-engineered solution for a critical part of any SaaS application.