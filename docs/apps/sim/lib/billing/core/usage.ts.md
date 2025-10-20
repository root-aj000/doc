This TypeScript file, `usage-management.ts`, is a core part of the application's billing and usage tracking system. It provides a suite of functions designed to manage user and organization resource consumption limits, retrieve detailed usage data, project future costs, and send automated notifications when usage thresholds are met.

It acts as the central hub for:
1.  **Initializing and updating user usage statistics**: Setting up default limits for new users and allowing updates to these limits.
2.  **Retrieving usage data**: Providing a comprehensive view of a user's current consumption, their assigned limit, and related billing period information.
3.  **Determining effective limits**: Calculating the correct usage limit based on a user's subscription plan (free, pro, team, enterprise), taking into account individual user settings, organization-wide limits, and minimums based on billing seats.
4.  **Billing projections**: Estimating future costs based on current consumption trends.
5.  **Usage status checks**: Determining if a user is nearing or has exceeded their usage limits.
6.  **Sending notifications**: Alerting users or organization administrators when usage approaches a critical threshold (e.g., 80% of limit).
7.  **Synchronization**: Adjusting usage limits automatically when subscription plans change.

Essentially, this file ensures that resource usage is tracked, limits are enforced, and users are informed about their consumption.

---

### Detailed Code Explanation

Let's break down each part of the file, explaining its purpose and the logic behind each line.

#### Imports

```typescript
import { db } from '@sim/db'
import { member, organization, settings, user, userStats } from '@sim/db/schema'
import { eq, inArray } from 'drizzle-orm'
import { getEmailSubject, renderUsageThresholdEmail } from '@/components/emails/render-email'
import { getHighestPrioritySubscription } from '@/lib/billing/core/subscription'
import {
  canEditUsageLimit,
  getFreeTierLimit,
  getPerUserMinimumLimit,
} from '@/lib/billing/subscriptions/utils'
import type { BillingData, UsageData, UsageLimitInfo } from '@/lib/billing/types'
import { sendEmail } from '@/lib/email/mailer'
import { getEmailPreferences } from '@/lib/email/unsubscribe'
import { isBillingEnabled } from '@/lib/environment'
import { createLogger } from '@/lib/logs/console/logger'
import { getBaseUrl } from '@/lib/urls/utils'
```

*   `import { db } from '@sim/db'`: Imports the database client instance, allowing interaction with the database.
*   `import { member, organization, settings, user, userStats } from '@sim/db/schema'`: Imports Drizzle ORM schema definitions for various database tables. These objects (e.g., `userStats`, `organization`) represent the structure of your database tables and are used to build database queries.
    *   `member`: Represents the relationship between users and organizations.
    *   `organization`: Stores data about organizations, including their custom usage limits.
    *   `settings`: User-specific settings, including notification preferences.
    *   `user`: Core user information.
    *   `userStats`: Stores individual user usage data and their specific usage limit.
*   `import { eq, inArray } from 'drizzle-orm'`: Imports specific functions from the Drizzle ORM library.
    *   `eq`: Used to create an "equals" condition in a WHERE clause (e.g., `WHERE id = 'someId'`).
    *   `inArray`: Used to create an "IN" condition in a WHERE clause (e.g., `WHERE userId IN ('id1', 'id2')`).
*   `import { getEmailSubject, renderUsageThresholdEmail } from '@/components/emails/render-email'`: Imports functions for rendering email templates and defining email subjects for usage notifications.
*   `import { getHighestPrioritySubscription } from '@/lib/billing/core/subscription'`: Imports a utility function to determine a user's most relevant subscription, especially if they have multiple.
*   `import { canEditUsageLimit, getFreeTierLimit, getPerUserMinimumLimit, } from '@/lib/billing/subscriptions/utils'`: Imports billing utility functions:
    *   `canEditUsageLimit`: Checks if a user's current plan allows them to modify their usage limit.
    *   `getFreeTierLimit`: Returns the default usage limit for users on the free plan.
    *   `getPerUserMinimumLimit`: Returns the minimum allowed usage limit for a user based on their subscription plan.
*   `import type { BillingData, UsageData, UsageLimitInfo } from '@/lib/billing/types'`: Imports TypeScript type definitions for various data structures returned by functions in this file, ensuring type safety and clarity.
*   `import { sendEmail } from '@/lib/email/mailer'`: Imports the email sending service.
*   `import { getEmailPreferences } from '@/lib/email/unsubscribe'`: Imports a function to retrieve user email notification preferences, respecting unsubscribe choices.
*   `import { isBillingEnabled } from '@/lib/environment'`: Imports a flag to check if the billing system is active in the current environment.
*   `import { createLogger } from '@/lib/logs/console/logger'`: Imports a utility to create a logger instance for structured logging.
*   `import { getBaseUrl } from '@/lib/urls/utils'`: Imports a function to get the base URL of the application, used for constructing links in emails.

#### Logger Initialization

```typescript
const logger = createLogger('UsageManagement')
```

*   `const logger = createLogger('UsageManagement')`: Initializes a logger specifically for this file, tagged with `'UsageManagement'`, making it easier to filter logs related to usage tracking.

---

### `handleNewUser`

```typescript
/**
 * Handle new user setup when they join the platform
 * Creates userStats record with default free credits
 */
export async function handleNewUser(userId: string): Promise<void> {
  try {
    await db.insert(userStats).values({
      id: crypto.randomUUID(),
      userId: userId,
      currentUsageLimit: getFreeTierLimit().toString(),
      usageLimitUpdatedAt: new Date(),
    })

    logger.info('User stats record created for new user', { userId })
  } catch (error) {
    logger.error('Failed to create user stats record for new user', {
      userId,
      error,
    })
    throw error
  }
}
```

**Purpose**: This function is called when a new user joins the platform. Its responsibility is to create an initial `userStats` record for them in the database, setting their default usage limit to the free tier's limit.

*   `export async function handleNewUser(userId: string): Promise<void>`: Defines an asynchronous function named `handleNewUser` that takes a `userId` (string) and returns nothing (`void`).
*   `try { ... } catch (error) { ... }`: Standard error handling block to catch and log any issues during database operations.
*   `await db.insert(userStats).values({ ... })`: This is a Drizzle ORM call to insert a new row into the `userStats` table.
    *   `id: crypto.randomUUID()`: Generates a unique identifier for the new `userStats` record. `crypto.randomUUID()` is a standard Web Crypto API function for generating UUIDs.
    *   `userId: userId`: Associates this `userStats` record with the specific user identified by `userId`.
    *   `currentUsageLimit: getFreeTierLimit().toString()`: Sets the initial usage limit for the new user to the value returned by `getFreeTierLimit()`. The limit is stored as a string in the database, so `.toString()` converts the number.
    *   `usageLimitUpdatedAt: new Date()`: Records the timestamp when this limit was last updated.
*   `logger.info('User stats record created for new user', { userId })`: Logs a successful creation of the user stats record, including the `userId` for context.
*   `logger.error(...)`: Logs any errors that occur during the insertion.
*   `throw error`: Re-throws the error after logging, allowing upstream callers to handle it.

---

### `getUserUsageData`

```typescript
/**
 * Get comprehensive usage data for a user
 */
export async function getUserUsageData(userId: string): Promise<UsageData> {
  try {
    const [userStatsData, subscription] = await Promise.all([
      db.select().from(userStats).where(eq(userStats.userId, userId)).limit(1),
      getHighestPrioritySubscription(userId),
    ])

    if (userStatsData.length === 0) {
      logger.error('User stats not found for userId', { userId })
      throw new Error(`User stats not found for userId: ${userId}`)
    }

    const stats = userStatsData[0]
    let currentUsage = Number.parseFloat(stats.currentPeriodCost?.toString() ?? '0')

    // For Pro users, include any snapshotted usage (from when they joined a team)
    // This ensures they see their total Pro usage in the UI
    if (subscription && subscription.plan === 'pro' && subscription.referenceId === userId) {
      const snapshotUsage = Number.parseFloat(stats.proPeriodCostSnapshot?.toString() ?? '0')
      if (snapshotUsage > 0) {
        currentUsage += snapshotUsage
        logger.info('Including Pro snapshot in usage display', {
          userId,
          currentPeriodCost: stats.currentPeriodCost,
          proPeriodCostSnapshot: snapshotUsage,
          totalUsage: currentUsage,
        })
      }
    }

    // Determine usage limit based on plan type
    let limit: number

    if (!subscription || subscription.plan === 'free' || subscription.plan === 'pro') {
      // Free/Pro: Use individual user limit from userStats
      limit = stats.currentUsageLimit
        ? Number.parseFloat(stats.currentUsageLimit)
        : getFreeTierLimit()
    } else {
      // Team/Enterprise: Use organization limit but never below minimum (seats × cost per seat)
      const orgData = await db
        .select({ orgUsageLimit: organization.orgUsageLimit })
        .from(organization)
        .where(eq(organization.id, subscription.referenceId))
        .limit(1)

      const { getPlanPricing } = await import('@/lib/billing/core/billing')
      const { basePrice } = getPlanPricing(subscription.plan)
      const minimum = (subscription.seats || 1) * basePrice

      if (orgData.length > 0 && orgData[0].orgUsageLimit) {
        const configured = Number.parseFloat(orgData[0].orgUsageLimit)
        limit = Math.max(configured, minimum)
      } else {
        limit = minimum
      }
    }

    const percentUsed = limit > 0 ? Math.min((currentUsage / limit) * 100, 100) : 0
    const isWarning = percentUsed >= 80
    const isExceeded = currentUsage >= limit

    // Derive billing period dates from subscription (source of truth).
    // For free users or missing dates, expose nulls.
    const billingPeriodStart = subscription?.periodStart ?? null
    const billingPeriodEnd = subscription?.periodEnd ?? null

    return {
      currentUsage,
      limit,
      percentUsed,
      isWarning,
      isExceeded,
      billingPeriodStart,
      billingPeriodEnd,
      lastPeriodCost: Number.parseFloat(stats.lastPeriodCost?.toString() || '0'),
    }
  } catch (error) {
    logger.error('Failed to get user usage data', { userId, error })
    throw error
  }
}
```

**Purpose**: This is a crucial function that fetches and calculates a comprehensive set of usage-related data for a given user, including their current consumption, effective limit, percentage used, and billing period.

*   `export async function getUserUsageData(userId: string): Promise<UsageData>`: Defines an asynchronous function `getUserUsageData` taking a `userId` and returning a `Promise` resolving to an `UsageData` object.
*   `const [userStatsData, subscription] = await Promise.all([...])`: Concurrently fetches two pieces of data to improve performance:
    *   `db.select().from(userStats).where(eq(userStats.userId, userId)).limit(1)`: Retrieves the `userStats` record for the specified user. `eq(userStats.userId, userId)` is the Drizzle condition for `WHERE userStats.userId = userId`. `limit(1)` ensures only one record is fetched.
    *   `getHighestPrioritySubscription(userId)`: Fetches the user's primary subscription details.
*   `if (userStatsData.length === 0) { ... }`: Checks if the `userStats` record was found. If not, it logs an error and throws an exception, as this record is essential.
*   `const stats = userStatsData[0]`: Extracts the actual `userStats` object from the single-element array.
*   `let currentUsage = Number.parseFloat(stats.currentPeriodCost?.toString() ?? '0')`: Initializes `currentUsage` by parsing the `currentPeriodCost` from `stats`. The `?.toString() ?? '0'` handles cases where `currentPeriodCost` might be `null` or `undefined`.
*   **Pro User Snapshot Logic**:
    *   `if (subscription && subscription.plan === 'pro' && subscription.referenceId === userId)`: This condition applies only to 'Pro' users where their subscription is directly linked to their `userId` (meaning they are an individual Pro subscriber, not part of a team).
    *   `const snapshotUsage = Number.parseFloat(stats.proPeriodCostSnapshot?.toString() ?? '0')`: Retrieves a `proPeriodCostSnapshot`. This is likely a historical usage amount recorded when a user transitioned to Pro or joined a team, ensuring their total Pro usage is displayed correctly.
    *   `if (snapshotUsage > 0) { currentUsage += snapshotUsage; logger.info(...) }`: If a snapshot exists and is positive, it's added to `currentUsage` to reflect the total usage for display purposes, and this action is logged.
*   **Determine Usage Limit**:
    *   `let limit: number`: Declares a variable `limit` to store the calculated usage limit.
    *   `if (!subscription || subscription.plan === 'free' || subscription.plan === 'pro')`:
        *   **Free/Pro plans**: If there's no subscription (implies free) or the plan is 'free' or 'pro', the user's individual `currentUsageLimit` from their `userStats` is used.
        *   `limit = stats.currentUsageLimit ? Number.parseFloat(stats.currentUsageLimit) : getFreeTierLimit()`: If `currentUsageLimit` exists, it's parsed; otherwise, the `getFreeTierLimit()` is used as a fallback.
    *   `else { ... }`:
        *   **Team/Enterprise plans**: For organizational plans, the limit is determined by the organization's settings.
        *   `const orgData = await db.select({ orgUsageLimit: organization.orgUsageLimit }).from(organization).where(eq(organization.id, subscription.referenceId)).limit(1)`: Fetches the `orgUsageLimit` from the `organization` table, using the `subscription.referenceId` (which points to the organization ID for team/enterprise plans).
        *   `const { getPlanPricing } = await import('@/lib/billing/core/billing')`: Dynamically imports `getPlanPricing` function for potentially deferred loading.
        *   `const { basePrice } = getPlanPricing(subscription.plan)`: Gets the base price per user for the current plan.
        *   `const minimum = (subscription.seats || 1) * basePrice`: Calculates a minimum organizational limit based on the number of seats in the subscription multiplied by the base price per seat. If `seats` is not defined, it defaults to 1.
        *   `if (orgData.length > 0 && orgData[0].orgUsageLimit) { ... }`: If an organizational limit is explicitly configured (`orgUsageLimit` exists).
            *   `const configured = Number.parseFloat(orgData[0].orgUsageLimit)`: Parses the configured organization limit.
            *   `limit = Math.max(configured, minimum)`: The effective limit is the greater of the configured organization limit or the calculated minimum based on seats. This ensures the limit never falls below the minimum required for the plan.
        *   `else { limit = minimum }`: If no custom organizational limit is set, the limit defaults to the calculated minimum.
*   **Calculate Metrics**:
    *   `const percentUsed = limit > 0 ? Math.min((currentUsage / limit) * 100, 100) : 0`: Calculates the percentage of the limit used. It avoids division by zero if `limit` is 0 and caps the percentage at 100%.
    *   `const isWarning = percentUsed >= 80`: Sets `isWarning` to `true` if usage is 80% or more.
    *   `const isExceeded = currentUsage >= limit`: Sets `isExceeded` to `true` if current usage meets or exceeds the limit.
*   **Billing Period Dates**:
    *   `const billingPeriodStart = subscription?.periodStart ?? null`: Retrieves the billing period start date from the subscription, defaulting to `null` if not available.
    *   `const billingPeriodEnd = subscription?.periodEnd ?? null`: Retrieves the billing period end date from the subscription, defaulting to `null` if not available.
*   `return { ... }`: Returns an `UsageData` object containing all the calculated and retrieved usage metrics.
    *   `lastPeriodCost: Number.parseFloat(stats.lastPeriodCost?.toString() || '0')`: Includes the cost from the previous billing period.
*   `catch (error) { ... }`: Logs and re-throws any errors.

---

### `getUserUsageLimitInfo`

```typescript
/**
 * Get usage limit information for a user
 */
export async function getUserUsageLimitInfo(userId: string): Promise<UsageLimitInfo> {
  try {
    const [subscription, userStatsRecord] = await Promise.all([
      getHighestPrioritySubscription(userId),
      db.select().from(userStats).where(eq(userStats.userId, userId)).limit(1),
    ])

    if (userStatsRecord.length === 0) {
      throw new Error(`User stats not found for userId: ${userId}`)
    }

    const stats = userStatsRecord[0]

    // Determine limits based on plan type
    let currentLimit: number
    let minimumLimit: number
    let canEdit: boolean

    if (!subscription || subscription.plan === 'free' || subscription.plan === 'pro') {
      // Free/Pro: Use individual limits
      currentLimit = stats.currentUsageLimit
        ? Number.parseFloat(stats.currentUsageLimit)
        : getFreeTierLimit()
      minimumLimit = getPerUserMinimumLimit(subscription)
      canEdit = canEditUsageLimit(subscription)
    } else {
      // Team/Enterprise: Use organization limits (users cannot edit)
      const orgData = await db
        .select({ orgUsageLimit: organization.orgUsageLimit })
        .from(organization)
        .where(eq(organization.id, subscription.referenceId))
        .limit(1)

      const { getPlanPricing } = await import('@/lib/billing/core/billing')
      const { basePrice } = getPlanPricing(subscription.plan)
      const minimum = (subscription.seats || 1) * basePrice

      if (orgData.length > 0 && orgData[0].orgUsageLimit) {
        const configured = Number.parseFloat(orgData[0].orgUsageLimit)
        currentLimit = Math.max(configured, minimum)
      } else {
        currentLimit = minimum
      }
      minimumLimit = minimum
      canEdit = false // Team/enterprise members cannot edit limits
    }

    return {
      currentLimit,
      canEdit,
      minimumLimit,
      plan: subscription?.plan || 'free',
      updatedAt: stats.usageLimitUpdatedAt,
    }
  } catch (error) {
    logger.error('Failed to get usage limit info', { userId, error })
    throw error
  }
}
```

**Purpose**: This function provides specific information about a user's usage limit, including their current limit, the minimum allowed limit for their plan, and whether they have permission to edit their limit. This is often used for UI displays on billing pages.

*   `export async function getUserUsageLimitInfo(userId: string): Promise<UsageLimitInfo>`: Defines an asynchronous function returning a `Promise` resolving to an `UsageLimitInfo` object.
*   `const [subscription, userStatsRecord] = await Promise.all([...])`: Similar to `getUserUsageData`, it fetches the user's subscription and `userStats` concurrently.
*   `if (userStatsRecord.length === 0) { ... }`: Throws an error if `userStats` is not found.
*   `const stats = userStatsRecord[0]`: Extracts the `userStats` object.
*   `let currentLimit: number; let minimumLimit: number; let canEdit: boolean;`: Declares variables to store the calculated limit information.
*   **Limit Determination Logic (similar to `getUserUsageData` but focused only on limits):**
    *   `if (!subscription || subscription.plan === 'free' || subscription.plan === 'pro')`:
        *   **Free/Pro plans**: `currentLimit` is taken from `userStats` or defaults to free tier. `minimumLimit` is obtained via `getPerUserMinimumLimit`. `canEdit` is determined by `canEditUsageLimit`.
    *   `else { ... }`:
        *   **Team/Enterprise plans**:
            *   Fetches `orgUsageLimit` from the organization table.
            *   Calculates `minimum` based on plan pricing and seats.
            *   `currentLimit` is the greater of the configured `orgUsageLimit` or the `minimum`.
            *   `minimumLimit` is set to the calculated `minimum`.
            *   `canEdit` is explicitly set to `false` because team/enterprise members cannot edit individual limits; limits are managed at the organization level.
*   `return { ... }`: Returns an `UsageLimitInfo` object containing the `currentLimit`, `canEdit` status, `minimumLimit`, `plan` name, and `updatedAt` timestamp from `userStats`.
*   `catch (error) { ... }`: Logs and re-throws errors.

---

### `initializeUserUsageLimit`

```typescript
/**
 * Initialize usage limits for a new user
 */
export async function initializeUserUsageLimit(userId: string): Promise<void> {
  // Check if user already has usage stats
  const existingStats = await db
    .select()
    .from(userStats)
    .where(eq(userStats.userId, userId))
    .limit(1)

  if (existingStats.length > 0) {
    return // User already has usage stats
  }

  // Check user's subscription to determine initial limit
  const subscription = await getHighestPrioritySubscription(userId)
  const isTeamOrEnterprise =
    subscription && (subscription.plan === 'team' || subscription.plan === 'enterprise')

  // Create initial usage stats
  await db.insert(userStats).values({
    id: crypto.randomUUID(),
    userId,
    // Team/enterprise: null (use org limit), Free/Pro: individual limit
    currentUsageLimit: isTeamOrEnterprise ? null : getFreeTierLimit().toString(),
    usageLimitUpdatedAt: new Date(),
  })

  logger.info('Initialized user stats', {
    userId,
    plan: subscription?.plan || 'free',
    hasIndividualLimit: !isTeamOrEnterprise,
  })
}
```

**Purpose**: This function sets up the initial `userStats` record for a user, similar to `handleNewUser`, but with more nuanced logic for setting the `currentUsageLimit` based on whether the user immediately belongs to a team/enterprise plan.

*   `export async function initializeUserUsageLimit(userId: string): Promise<void>`: Defines an asynchronous function taking a `userId`.
*   `const existingStats = await db.select().from(userStats).where(eq(userStats.userId, userId)).limit(1)`: Checks if a `userStats` record already exists for this `userId`.
*   `if (existingStats.length > 0) { return }`: If a record already exists, the function exits early to avoid re-initializing.
*   `const subscription = await getHighestPrioritySubscription(userId)`: Fetches the user's subscription.
*   `const isTeamOrEnterprise = subscription && (subscription.plan === 'team' || subscription.plan === 'enterprise')`: Determines if the user is on a team or enterprise plan.
*   `await db.insert(userStats).values({ ... })`: Inserts a new `userStats` record.
    *   `id: crypto.randomUUID(), userId`: Standard ID generation and user association.
    *   `currentUsageLimit: isTeamOrEnterprise ? null : getFreeTierLimit().toString()`: This is the key difference from `handleNewUser`.
        *   If the user is on a Team or Enterprise plan, their `currentUsageLimit` is set to `null`. This signifies that their usage limit is managed at the *organization* level, not individually.
        *   Otherwise (Free or Pro plan), it defaults to the `getFreeTierLimit()`.
    *   `usageLimitUpdatedAt: new Date()`: Records the update timestamp.
*   `logger.info(...)`: Logs the initialization, including the plan type and whether an individual limit was set.

---

### `updateUserUsageLimit`

```typescript
/**
 * Update a user's custom usage limit
 */
export async function updateUserUsageLimit(
  userId: string,
  newLimit: number,
  setBy?: string // For team admin tracking
): Promise<{ success: boolean; error?: string }> {
  try {
    const subscription = await getHighestPrioritySubscription(userId)

    // Team/enterprise users don't have individual limits
    if (subscription && (subscription.plan === 'team' || subscription.plan === 'enterprise')) {
      return {
        success: false,
        error: 'Team and enterprise members use organization limits',
      }
    }

    // Only pro users can edit limits (free users cannot)
    if (!subscription || subscription.plan === 'free') {
      return { success: false, error: 'Free plan users cannot edit usage limits' }
    }

    const minimumLimit = getPerUserMinimumLimit(subscription)

    logger.info('Applying plan-based validation', {
      userId,
      newLimit,
      minimumLimit,
      plan: subscription?.plan,
    })

    // Validate new limit is not below minimum
    if (newLimit < minimumLimit) {
      return {
        success: false,
        error: `Usage limit cannot be below plan minimum of $${minimumLimit}`,
      }
    }

    // Get current usage to validate against
    const userStatsRecord = await db
      .select()
      .from(userStats)
      .where(eq(userStats.userId, userId))
      .limit(1)

    if (userStatsRecord.length > 0) {
      const currentUsage = Number.parseFloat(
        userStatsRecord[0].currentPeriodCost?.toString() || userStatsRecord[0].totalCost.toString()
      )

      // Validate new limit is not below current usage
      if (newLimit < currentUsage) {
        return {
          success: false,
          error: `Usage limit cannot be below current usage of $${currentUsage.toFixed(2)}`,
        }
      }
    }

    // Update the usage limit
    await db
      .update(userStats)
      .set({
        currentUsageLimit: newLimit.toString(),
        usageLimitUpdatedAt: new Date(),
      })
      .where(eq(userStats.userId, userId))

    logger.info('Updated user usage limit', {
      userId,
      newLimit,
      setBy: setBy || userId,
      planMinimum: minimumLimit,
      plan: subscription?.plan,
    })

    return { success: true }
  } catch (error) {
    logger.error('Failed to update usage limit', { userId, newLimit, error })
    return { success: false, error: 'Failed to update usage limit' }
  }
}
```

**Purpose**: Allows an authorized user (typically a Pro user for themselves, or an admin for a Team/Enterprise organization, though the latter is handled elsewhere) to set a custom usage limit for a specific user. It includes validation to ensure the new limit is valid.

*   `export async function updateUserUsageLimit( userId: string, newLimit: number, setBy?: string): Promise<{ success: boolean; error?: string }>`: Defines an asynchronous function taking `userId`, `newLimit` (number), and an optional `setBy` (string, for auditing who made the change). It returns an object indicating `success` and potentially an `error` message.
*   `const subscription = await getHighestPrioritySubscription(userId)`: Fetches the user's subscription.
*   **Early Exit Validations**:
    *   `if (subscription && (subscription.plan === 'team' || subscription.plan === 'enterprise')) { ... }`: Users on Team/Enterprise plans cannot set individual limits, so it returns an error.
    *   `if (!subscription || subscription.plan === 'free') { ... }`: Free plan users cannot edit their limits, so it returns an error. Only Pro users (or those with an active, non-free, non-team/enterprise subscription) can edit their individual limit.
*   `const minimumLimit = getPerUserMinimumLimit(subscription)`: Gets the minimum allowed limit for the user's current plan.
*   `logger.info('Applying plan-based validation', { ... })`: Logs validation details.
*   `if (newLimit < minimumLimit) { ... }`: Validates that the `newLimit` is not below the plan's minimum.
*   **Current Usage Validation**:
    *   `const userStatsRecord = await db.select().from(userStats).where(eq(userStats.userId, userId)).limit(1)`: Fetches the user's current `userStats`.
    *   `if (userStatsRecord.length > 0) { ... }`: If `userStats` is found:
        *   `const currentUsage = Number.parseFloat(userStatsRecord[0].currentPeriodCost?.toString() || userStatsRecord[0].totalCost.toString())`: Gets the current usage, preferring `currentPeriodCost` but falling back to `totalCost`.
        *   `if (newLimit < currentUsage) { ... }`: Validates that the `newLimit` is not below the user's *current actual usage*. This prevents users from setting a limit they have already exceeded.
*   **Update Database**:
    *   `await db.update(userStats).set({ currentUsageLimit: newLimit.toString(), usageLimitUpdatedAt: new Date(), }).where(eq(userStats.userId, userId))`: If all validations pass, updates the `currentUsageLimit` and `usageLimitUpdatedAt` in the `userStats` table for the specified `userId`.
*   `logger.info('Updated user usage limit', { ... })`: Logs the successful update.
*   `return { success: true }`: Returns success.
*   `catch (error) { ... }`: Catches, logs, and returns a generic error message for any other failures.

---

### `getUserUsageLimit`

```typescript
/**
 * Get usage limit for a user (used by checkUsageStatus for server-side checks)
 * Free/Pro: Individual user limit from userStats
 * Team/Enterprise: Organization limit
 */
export async function getUserUsageLimit(userId: string): Promise<number> {
  const subscription = await getHighestPrioritySubscription(userId)

  if (!subscription || subscription.plan === 'free' || subscription.plan === 'pro') {
    // Free/Pro: Use individual limit from userStats
    const userStatsQuery = await db
      .select({ currentUsageLimit: userStats.currentUsageLimit })
      .from(userStats)
      .where(eq(userStats.userId, userId))
      .limit(1)

    if (userStatsQuery.length === 0) {
      throw new Error(
        `No user stats record found for userId: ${userId}. User must be properly initialized before execution.`
      )
    }

    // Individual limits should never be null for free/pro users
    if (!userStatsQuery[0].currentUsageLimit) {
      throw new Error(
        `Invalid null usage limit for ${subscription?.plan || 'free'} user: ${userId}. User stats must be properly initialized.`
      )
    }

    return Number.parseFloat(userStatsQuery[0].currentUsageLimit)
  }
  // Team/Enterprise: Use organization limit but never below minimum
  const orgData = await db
    .select({ orgUsageLimit: organization.orgUsageLimit })
    .from(organization)
    .where(eq(organization.id, subscription.referenceId))
    .limit(1)

  if (orgData.length === 0) {
    throw new Error(`Organization not found: ${subscription.referenceId} for user: ${userId}`)
  }

  if (orgData[0].orgUsageLimit) {
    const configured = Number.parseFloat(orgData[0].orgUsageLimit)
    const { getPlanPricing } = await import('@/lib/billing/core/billing')
    const { basePrice } = getPlanPricing(subscription.plan)
    const minimum = (subscription.seats || 1) * basePrice
    return Math.max(configured, minimum)
  }

  // If org hasn't set a custom limit, use minimum (seats × cost per seat)
  const { getPlanPricing } = await import('@/lib/billing/core/billing')
  const { basePrice } = getPlanPricing(subscription.plan)
  return (subscription.seats || 1) * basePrice
}
```

**Purpose**: This function specifically retrieves *only* the effective usage limit for a user, handling the different logic for individual vs. organizational plans. It's designed for server-side checks where only the limit value is needed.

*   `export async function getUserUsageLimit(userId: string): Promise<number>`: Defines an asynchronous function taking `userId` and returning a `Promise` resolving to a `number` (the usage limit).
*   `const subscription = await getHighestPrioritySubscription(userId)`: Fetches the user's subscription.
*   **Free/Pro Plan Logic**:
    *   `if (!subscription || subscription.plan === 'free' || subscription.plan === 'pro')`: If the user is on a free or pro plan (or no active subscription).
    *   `const userStatsQuery = await db.select({ currentUsageLimit: userStats.currentUsageLimit }).from(userStats).where(eq(userStats.userId, userId)).limit(1)`: Fetches just the `currentUsageLimit` from `userStats`.
    *   `if (userStatsQuery.length === 0) { ... }`: Throws an error if `userStats` record is missing (indicates initialization failure).
    *   `if (!userStatsQuery[0].currentUsageLimit) { ... }`: Throws an error if the individual limit for a free/pro user is unexpectedly `null`.
    *   `return Number.parseFloat(userStatsQuery[0].currentUsageLimit)`: Parses and returns the individual limit.
*   **Team/Enterprise Plan Logic**:
    *   `const orgData = await db.select({ orgUsageLimit: organization.orgUsageLimit }).from(organization).where(eq(organization.id, subscription.referenceId)).limit(1)`: Fetches the `orgUsageLimit` for the organization.
    *   `if (orgData.length === 0) { ... }`: Throws an error if the organization linked by the subscription isn't found.
    *   `if (orgData[0].orgUsageLimit) { ... }`: If a custom organization limit is set:
        *   Calculates the `minimum` limit based on seats and base price.
        *   `return Math.max(configured, minimum)`: Returns the greater of the configured organization limit or the calculated minimum.
    *   `else { ... }`: If no custom organization limit is set, calculates and returns the minimum limit based on seats and base price.

---

### `checkUsageStatus`

```typescript
/**
 * Check usage status with warning thresholds
 */
export async function checkUsageStatus(userId: string): Promise<{
  status: 'ok' | 'warning' | 'exceeded'
  usageData: UsageData
}> {
  try {
    const usageData = await getUserUsageData(userId)

    let status: 'ok' | 'warning' | 'exceeded' = 'ok'
    if (usageData.isExceeded) {
      status = 'exceeded'
    } else if (usageData.isWarning) {
      status = 'warning'
    }

    return {
      status,
      usageData,
    }
  } catch (error) {
    logger.error('Failed to check usage status', { userId, error })
    throw error
  }
}
```

**Purpose**: This function provides a quick and clear status of a user's usage relative to their limit, categorizing it as 'ok', 'warning', or 'exceeded'.

*   `export async function checkUsageStatus(userId: string): Promise<{ status: 'ok' | 'warning' | 'exceeded'; usageData: UsageData }>`: Defines an asynchronous function returning an object with a `status` (a literal type indicating 'ok', 'warning', or 'exceeded') and the full `usageData`.
*   `const usageData = await getUserUsageData(userId)`: Reuses `getUserUsageData` to get all necessary usage details.
*   `let status: 'ok' | 'warning' | 'exceeded' = 'ok'`: Initializes the status as 'ok'.
*   `if (usageData.isExceeded) { status = 'exceeded' } else if (usageData.isWarning) { status = 'warning' }`: Logic to determine the status: `exceeded` takes precedence over `warning`.
*   `return { status, usageData }`: Returns the determined status and the comprehensive usage data.
*   `catch (error) { ... }`: Catches, logs, and re-throws errors.

---

### `syncUsageLimitsFromSubscription`

```typescript
/**
 * Sync usage limits based on subscription changes
 */
export async function syncUsageLimitsFromSubscription(userId: string): Promise<void> {
  const [subscription, currentUserStats] = await Promise.all([
    getHighestPrioritySubscription(userId),
    db.select().from(userStats).where(eq(userStats.userId, userId)).limit(1),
  ])

  if (currentUserStats.length === 0) {
    throw new Error(`User stats not found for userId: ${userId}`)
  }

  const currentStats = currentUserStats[0]

  // Team/enterprise: Should have null individual limits
  if (subscription && (subscription.plan === 'team' || subscription.plan === 'enterprise')) {
    if (currentStats.currentUsageLimit !== null) {
      await db
        .update(userStats)
        .set({
          currentUsageLimit: null,
          usageLimitUpdatedAt: new Date(),
        })
        .where(eq(userStats.userId, userId))

      logger.info('Cleared individual limit for team/enterprise member', {
        userId,
        plan: subscription.plan,
      })
    }
    return
  }

  // Free/Pro: Handle individual limits
  const defaultLimit = getPerUserMinimumLimit(subscription)
  const currentLimit = currentStats.currentUsageLimit
    ? Number.parseFloat(currentStats.currentUsageLimit)
    : 0

  if (!subscription || subscription.status !== 'active') {
    // Downgraded to free
    await db
      .update(userStats)
      .set({
        currentUsageLimit: getFreeTierLimit().toString(),
        usageLimitUpdatedAt: new Date(),
      })
      .where(eq(userStats.userId, userId))

    logger.info('Set limit to free tier', { userId })
  } else if (currentLimit < defaultLimit) {
    await db
      .update(userStats)
      .set({
        currentUsageLimit: defaultLimit.toString(),
        usageLimitUpdatedAt: new Date(),
      })
      .where(eq(userStats.userId, userId))

    logger.info('Raised limit to plan minimum', {
      userId,
      newLimit: defaultLimit,
    })
  }
  // Keep higher custom limits unchanged
}
```

**Purpose**: This function is critical for maintaining consistency between a user's subscription plan and their stored `userStats.currentUsageLimit`. It's typically called when a user's subscription changes (e.g., upgrades, downgrades, or joins/leaves a team).

*   `export async function syncUsageLimitsFromSubscription(userId: string): Promise<void>`: Defines an asynchronous function taking `userId`.
*   `const [subscription, currentUserStats] = await Promise.all([...])`: Concurrently fetches the user's subscription and current `userStats`.
*   `if (currentUserStats.length === 0) { ... }`: Throws an error if `userStats` not found.
*   `const currentStats = currentUserStats[0]`: Extracts the `userStats` object.
*   **Team/Enterprise Plan Synchronization**:
    *   `if (subscription && (subscription.plan === 'team' || subscription.plan === 'enterprise'))`: If the user is on a team or enterprise plan.
    *   `if (currentStats.currentUsageLimit !== null) { ... }`: If the user *currently* has an individual limit set (which they shouldn't for team/enterprise plans).
        *   `await db.update(userStats).set({ currentUsageLimit: null, usageLimitUpdatedAt: new Date(), }).where(eq(userStats.userId, userId))`: Updates their `currentUsageLimit` to `null` to reflect that their limit is now organization-managed.
        *   Logs the action.
    *   `return`: Exits after handling team/enterprise specific logic.
*   **Free/Pro Plan Synchronization**: This block runs if the user is *not* on a team/enterprise plan (i.e., Free or Pro).
    *   `const defaultLimit = getPerUserMinimumLimit(subscription)`: Gets the minimum limit applicable to their current Free/Pro plan.
    *   `const currentLimit = currentStats.currentUsageLimit ? Number.parseFloat(currentStats.currentUsageLimit) : 0`: Gets the user's existing individual limit, or 0 if `null`.
    *   `if (!subscription || subscription.status !== 'active')`:
        *   **Downgraded to Free**: If there's no active subscription (implies free plan).
        *   `await db.update(userStats).set({ currentUsageLimit: getFreeTierLimit().toString(), usageLimitUpdatedAt: new Date(), }).where(eq(userStats.userId, userId))`: Sets their limit back to the `freeTierLimit`.
        *   Logs the action.
    *   `else if (currentLimit < defaultLimit) { ... }`:
        *   **Upgraded or Plan Minimum Increased**: If their current individual limit (`currentLimit`) is *below* the `defaultLimit` for their *active* subscription (e.g., they upgraded from free to pro, or their plan's minimum increased).
        *   `await db.update(userStats).set({ currentUsageLimit: defaultLimit.toString(), usageLimitUpdatedAt: new Date(), }).where(eq(userStats.userId, userId))`: Updates their limit to the `defaultLimit`. This *raises* their limit if it was previously lower.
        *   Logs the action.
    *   `// Keep higher custom limits unchanged`: This important comment indicates that if a user has a custom limit that is *higher* than the `defaultLimit` (e.g., a Pro user increased their limit), this sync function will *not* lower it. It only ensures the limit meets the plan's minimum.

---

### `getTeamUsageLimits`

```typescript
/**
 * Get usage limit information for team members (for admin dashboard)
 */
export async function getTeamUsageLimits(organizationId: string): Promise<
  Array<{
    userId: string
    userName: string
    userEmail: string
    currentLimit: number
    currentUsage: number
    totalCost: number
    lastActive: Date | null
  }>
> {
  try {
    const teamMembers = await db
      .select({
        userId: member.userId,
        userName: user.name,
        userEmail: user.email,
        currentLimit: userStats.currentUsageLimit,
        currentPeriodCost: userStats.currentPeriodCost,
        totalCost: userStats.totalCost,
        lastActive: userStats.lastActive,
      })
      .from(member)
      .innerJoin(user, eq(member.userId, user.id))
      .leftJoin(userStats, eq(member.userId, userStats.userId))
      .where(eq(member.organizationId, organizationId))

    return teamMembers.map((memberData) => ({
      userId: memberData.userId,
      userName: memberData.userName,
      userEmail: memberData.userEmail,
      currentLimit: Number.parseFloat(memberData.currentLimit || getFreeTierLimit().toString()),
      currentUsage: Number.parseFloat(memberData.currentPeriodCost || '0'),
      totalCost: Number.parseFloat(memberData.totalCost || '0'),
      lastActive: memberData.lastActive,
    }))
  } catch (error) {
    logger.error('Failed to get team usage limits', { organizationId, error })
    return []
  }
}
```

**Purpose**: This function is designed for an organization's administrative dashboard. It fetches usage and limit information for all members within a specific organization.

*   `export async function getTeamUsageLimits(organizationId: string): Promise<Array<{ ... }>>`: Defines an asynchronous function taking an `organizationId` and returning a `Promise` resolving to an array of objects, each representing a team member with their usage details.
*   `const teamMembers = await db.select({ ... }).from(member).innerJoin(user, ...).leftJoin(userStats, ...).where(eq(member.organizationId, organizationId))`: This is a complex Drizzle ORM query:
    *   `db.select({ ... })`: Specifies which columns to retrieve from the joined tables: `userId`, `userName`, `userEmail` (from `user`), `currentLimit`, `currentPeriodCost`, `totalCost`, `lastActive` (from `userStats`).
    *   `from(member)`: Starts the query from the `member` table (which links users to organizations).
    *   `innerJoin(user, eq(member.userId, user.id))`: Joins the `member` table with the `user` table on `userId` to get user details. An `innerJoin` means only members with a corresponding user record will be included.
    *   `leftJoin(userStats, eq(member.userId, userStats.userId))`: Joins with `userStats` to get usage data. A `leftJoin` ensures that all team members are included, even if they don't yet have a `userStats` record (though they should, if properly initialized).
    *   `where(eq(member.organizationId, organizationId))`: Filters the results to only include members of the specified `organizationId`.
*   `return teamMembers.map((memberData) => ({ ... }))`: Maps the raw database results into a more structured array of objects.
    *   `currentLimit: Number.parseFloat(memberData.currentLimit || getFreeTierLimit().toString())`: Parses the `currentLimit`. If `memberData.currentLimit` is `null` (e.g., for a team member whose limit is organization-managed, or if `userStats` was missing), it defaults to the free tier limit for display purposes, though the actual limit might be higher (the organization's limit).
    *   `currentUsage: Number.parseFloat(memberData.currentPeriodCost || '0')`: Parses current usage, defaulting to '0'.
    *   `totalCost: Number.parseFloat(memberData.totalCost || '0')`: Parses total cost, defaulting to '0'.
*   `catch (error) { ... }`: Catches, logs, and returns an empty array on error.

---

### `getEffectiveCurrentPeriodCost`

```typescript
/**
 * Returns the effective current period usage cost for a user.
 * - Free/Pro: user's own currentPeriodCost (fallback to totalCost)
 * - Team/Enterprise: pooled sum of all members' currentPeriodCost within the organization
 */
export async function getEffectiveCurrentPeriodCost(userId: string): Promise<number> {
  const subscription = await getHighestPrioritySubscription(userId)

  // If no team/org subscription, return the user's own usage
  if (!subscription || subscription.plan === 'free' || subscription.plan === 'pro') {
    const rows = await db
      .select({ current: userStats.currentPeriodCost })
      .from(userStats)
      .where(eq(userStats.userId, userId))
      .limit(1)

    if (rows.length === 0) return 0
    return rows[0].current ? Number.parseFloat(rows[0].current.toString()) : 0
  }

  // Team/Enterprise: pooled usage across org members
  const teamMembers = await db
    .select({ userId: member.userId })
    .from(member)
    .where(eq(member.organizationId, subscription.referenceId))

  if (teamMembers.length === 0) return 0

  const memberIds = teamMembers.map((m) => m.userId)
  const rows = await db
    .select({ current: userStats.currentPeriodCost })
    .from(userStats)
    .where(inArray(userStats.userId, memberIds))

  let pooled = 0
  for (const r of rows) {
    pooled += r.current ? Number.parseFloat(r.current.toString()) : 0
  }
  return pooled
}
```

**Purpose**: This function calculates the *actual* current period cost that applies to a given user. This differs for individual plans (where it's the user's own cost) versus organizational plans (where it's the sum of all team members' costs).

*   `export async function getEffectiveCurrentPeriodCost(userId: string): Promise<number>`: Defines an asynchronous function returning the total cost as a number.
*   `const subscription = await getHighestPrioritySubscription(userId)`: Fetches the user's subscription.
*   **Free/Pro Plan Logic**:
    *   `if (!subscription || subscription.plan === 'free' || subscription.plan === 'pro')`: If the user is on a free/pro plan.
    *   `const rows = await db.select({ current: userStats.currentPeriodCost }).from(userStats).where(eq(userStats.userId, userId)).limit(1)`: Fetches the `currentPeriodCost` for the individual user.
    *   `if (rows.length === 0) return 0`: If no `userStats` record, returns 0.
    *   `return rows[0].current ? Number.parseFloat(rows[0].current.toString()) : 0`: Parses and returns the individual user's current cost.
*   **Team/Enterprise Plan Logic (Pooled Usage)**:
    *   `const teamMembers = await db.select({ userId: member.userId }).from(member).where(eq(member.organizationId, subscription.referenceId))`: Fetches all `userId`s of members belonging to the organization specified by `subscription.referenceId`.
    *   `if (teamMembers.length === 0) return 0`: If no team members, returns 0.
    *   `const memberIds = teamMembers.map((m) => m.userId)`: Extracts an array of all team member `userId`s.
    *   `const rows = await db.select({ current: userStats.currentPeriodCost }).from(userStats).where(inArray(userStats.userId, memberIds))`: Fetches `currentPeriodCost` for all members in the `memberIds` array using `inArray` Drizzle condition.
    *   `let pooled = 0; for (const r of rows) { pooled += r.current ? Number.parseFloat(r.current.toString()) : 0 }`: Iterates through the results and sums up all `currentPeriodCost`s to get the total pooled usage.
    *   `return pooled`: Returns the total pooled cost.

---

### `calculateBillingProjection`

```typescript
/**
 * Calculate billing projection based on current usage
 */
export async function calculateBillingProjection(userId: string): Promise<BillingData> {
  try {
    const usageData = await getUserUsageData(userId)

    if (!usageData.billingPeriodStart || !usageData.billingPeriodEnd) {
      return {
        currentPeriodCost: usageData.currentUsage,
        projectedCost: usageData.currentUsage,
        limit: usageData.limit,
        billingPeriodStart: null,
        billingPeriodEnd: null,
        daysRemaining: 0,
      }
    }

    const now = new Date()
    const periodStart = new Date(usageData.billingPeriodStart)
    const periodEnd = new Date(usageData.billingPeriodEnd)

    const totalDays = Math.ceil(
      (periodEnd.getTime() - periodStart.getTime()) / (1000 * 60 * 60 * 24)
    )
    const daysElapsed = Math.ceil((now.getTime() - periodStart.getTime()) / (1000 * 60 * 60 * 24))
    const daysRemaining = Math.max(0, totalDays - daysElapsed)

    // Project cost based on daily usage rate
    const dailyRate = daysElapsed > 0 ? usageData.currentUsage / daysElapsed : 0
    const projectedCost = dailyRate * totalDays

    return {
      currentPeriodCost: usageData.currentUsage,
      projectedCost: Math.min(projectedCost, usageData.limit), // Cap at limit
      limit: usageData.limit,
      billingPeriodStart: usageData.billingPeriodStart,
      billingPeriodEnd: usageData.billingPeriodEnd,
      daysRemaining,
    }
  } catch (error) {
    logger.error('Failed to calculate billing projection', { userId, error })
    throw error
  }
}
```

**Purpose**: This function estimates a user's total cost for the current billing period based on their current usage and how much of the period has elapsed.

*   `export async function calculateBillingProjection(userId: string): Promise<BillingData>`: Defines an asynchronous function returning a `Promise` resolving to a `BillingData` object.
*   `const usageData = await getUserUsageData(userId)`: Fetches comprehensive usage data using the existing helper function.
*   `if (!usageData.billingPeriodStart || !usageData.billingPeriodEnd) { ... }`: If billing period dates are missing (e.g., for free users without an active subscription), it returns a simplified `BillingData` where projected cost is just the current usage and no remaining days.
*   `const now = new Date(); const periodStart = new Date(usageData.billingPeriodStart); const periodEnd = new Date(usageData.billingPeriodEnd);`: Converts the date strings to `Date` objects for calculations.
*   **Date Calculations**:
    *   `totalDays`: Calculates the total number of days in the billing period. `(periodEnd.getTime() - periodStart.getTime())` gives milliseconds, divided by `(1000 * 60 * 60 * 24)` converts to days. `Math.ceil` ensures partial days are rounded up.
    *   `daysElapsed`: Calculates how many days have passed since the start of the billing period until `now`.
    *   `daysRemaining`: Calculates the remaining days, ensuring it doesn't go below zero.
*   `const dailyRate = daysElapsed > 0 ? usageData.currentUsage / daysElapsed : 0`: Calculates the average daily usage rate. If no days have elapsed, the rate is 0.
*   `const projectedCost = dailyRate * totalDays`: Projects the total cost by multiplying the daily rate by the total days in the period.
*   `return { ... }`: Returns a `BillingData` object.
    *   `projectedCost: Math.min(projectedCost, usageData.limit)`: The projected cost is capped at the user's `usageData.limit`. This means if the projection exceeds the limit, it assumes they stop using resources once the limit is hit.
*   `catch (error) { ... }`: Catches, logs, and re-throws errors.

---

### `maybeSendUsageThresholdEmail`

```typescript
/**
 * Send usage threshold notification when crossing from <80% to ≥80%.
 * - Skips when billing is disabled.
 * - Respects user-level notifications toggle and unsubscribe preferences.
 * - For organization plans, emails owners/admins who have notifications enabled.
 */
export async function maybeSendUsageThresholdEmail(params: {
  scope: 'user' | 'organization'
  planName: string
  percentBefore: number
  percentAfter: number
  userId?: string
  userEmail?: string
  userName?: string
  organizationId?: string
  currentUsageAfter: number
  limit: number
}): Promise<void> {
  try {
    if (!isBillingEnabled) return
    // Only on upward crossing to >= 80%
    if (!(params.percentBefore < 80 && params.percentAfter >= 80)) return
    if (params.limit <= 0 || params.currentUsageAfter <= 0) return

    const baseUrl = getBaseUrl()
    const ctaLink = `${baseUrl}/workspace?billing=usage`
    const sendTo = async (email: string, name?: string) => {
      const prefs = await getEmailPreferences(email)
      if (prefs?.unsubscribeAll || prefs?.unsubscribeNotifications) return

      const html = await renderUsageThresholdEmail({
        userName: name,
        planName: params.planName,
        percentUsed: Math.min(100, Math.round(params.percentAfter)),
        currentUsage: params.currentUsageAfter,
        limit: params.limit,
        ctaLink,
      })

      await sendEmail({
        to: email,
        subject: getEmailSubject('usage-threshold'),
        html,
        emailType: 'notifications',
      })
    }

    if (params.scope === 'user' && params.userId && params.userEmail) {
      const rows = await db
        .select({ enabled: settings.billingUsageNotificationsEnabled })
        .from(settings)
        .where(eq(settings.userId, params.userId))
        .limit(1)
      if (rows.length > 0 && rows[0].enabled === false) return
      await sendTo(params.userEmail, params.userName)
    } else if (params.scope === 'organization' && params.organizationId) {
      const admins = await db
        .select({
          email: user.email,
          name: user.name,
          enabled: settings.billingUsageNotificationsEnabled,
          role: member.role,
        })
        .from(member)
        .innerJoin(user, eq(member.userId, user.id))
        .leftJoin(settings, eq(settings.userId, member.userId))
        .where(eq(member.organizationId, params.organizationId))

      for (const a of admins) {
        const isAdmin = a.role === 'owner' || a.role === 'admin'
        if (!isAdmin) continue
        if (a.enabled === false) continue
        if (!a.email) continue
        await sendTo(a.email, a.name || undefined)
      }
    }
  } catch (error) {
    logger.error('Failed to send usage threshold email', {
      scope: params.scope,
      userId: params.userId,
      organizationId: params.organizationId,
      error,
    })
  }
}
```

**Purpose**: This function is triggered when usage changes and checks if a usage threshold email needs to be sent. It only sends an email if usage crosses the 80% mark *upwards* and respects user/organization notification preferences.

*   `export async function maybeSendUsageThresholdEmail(params: { ... }): Promise<void>`: Defines an asynchronous function taking a `params` object containing all necessary information for sending the email, including `scope` ('user' or 'organization'), usage percentages, and identifiers.
*   `if (!isBillingEnabled) return`: Early exit if billing is disabled in the environment.
*   `if (!(params.percentBefore < 80 && params.percentAfter >= 80)) return`: This is the core condition for sending the email. It *only* sends if the percentage *before* the change was less than 80% and the percentage *after* is 80% or greater. This prevents repeated emails if usage hovers around 80% or decreases.
*   `if (params.limit <= 0 || params.currentUsageAfter <= 0) return`: Skips sending if the limit or current usage is not a positive value (e.g., no meaningful usage or limit).
*   `const baseUrl = getBaseUrl(); const ctaLink = `${baseUrl}/workspace?billing=usage``: Constructs a call-to-action link for the email, directing users to their billing/usage workspace.
*   **`sendTo` Helper Function**:
    *   `const sendTo = async (email: string, name?: string) => { ... }`: This nested asynchronous function encapsulates the logic for sending a single email.
    *   `const prefs = await getEmailPreferences(email)`: Retrieves the user's email preferences for the given `email`.
    *   `if (prefs?.unsubscribeAll || prefs?.unsubscribeNotifications) return`: If the user has globally unsubscribed or unsubscribed from notifications specifically, the email is not sent.
    *   `const html = await renderUsageThresholdEmail({ ... })`: Renders the HTML content of the usage threshold email using the provided parameters.
    *   `await sendEmail({ to: email, subject: getEmailSubject('usage-threshold'), html, emailType: 'notifications', })`: Sends the email using the `sendEmail` service.
*   **User vs. Organization Scope**:
    *   `if (params.scope === 'user' && params.userId && params.userEmail) { ... }`:
        *   **User scope**: For individual users.
        *   `const rows = await db.select({ enabled: settings.billingUsageNotificationsEnabled }).from(settings).where(eq(settings.userId, params.userId)).limit(1)`: Checks the user's specific setting for `billingUsageNotificationsEnabled`.
        *   `if (rows.length > 0 && rows[0].enabled === false) return`: If the setting exists and is explicitly `false`, the email is skipped.
        *   `await sendTo(params.userEmail, params.userName)`: Calls the `sendTo` helper to send the email to the individual user.
    *   `else if (params.scope === 'organization' && params.organizationId) { ... }`:
        *   **Organization scope**: For team/enterprise plans, emails are sent to relevant administrators.
        *   `const admins = await db.select({ ... }).from(member).innerJoin(user, ...).leftJoin(settings, ...).where(eq(member.organizationId, params.organizationId))`: This query fetches all members of the organization, their user details, and their notification settings.
        *   `for (const a of admins) { ... }`: Iterates through each fetched member.
            *   `const isAdmin = a.role === 'owner' || a.role === 'admin'`: Checks if the member has an 'owner' or 'admin' role.
            *   `if (!isAdmin) continue`: Skips if not an admin.
            *   `if (a.enabled === false) continue`: Skips if the admin has disabled billing usage notifications.
            *   `if (!a.email) continue`: Skips if the admin's email is missing.
            *   `await sendTo(a.email, a.name || undefined)`: Sends the email to the eligible admin.
*   `catch (error) { ... }`: Catches and logs any errors during the email sending process, but does not re-throw, as email notifications are often considered non-critical for the primary operation.