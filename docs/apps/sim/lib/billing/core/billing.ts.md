This TypeScript file, `billing.ts`, serves as the central hub for handling **billing logic, subscription management, and usage calculation** within the application. Its primary goal is to provide a clear and comprehensive understanding of a user's or organization's current billing status, including their plan, usage, and any potential overages.

It interacts with the database to fetch subscription and user data, calculates usage against plan limits, and determines projected costs. This information is crucial for displaying billing summaries in the user interface, generating invoices, and processing billing events (like subscription updates or deletions).

---

### **Core Billing Model Explained**

Before diving into the code, it's essential to understand the billing model implemented here, as described in the comments:

1.  **Base Price (Upfront Payment):** Users (or organizations) pay a fixed "base price" upfront, typically via a Stripe subscription, at the start of each billing period (e.g., $20 for a Pro plan).
2.  **Usage Coverage:** This base price covers a certain amount of usage. If a user's usage during the month is *less than or equal to* the base price equivalent, they pay nothing extra.
3.  **Overage Charges:** If a user's usage *exceeds* the amount covered by their base price, they are charged the difference as an "overage" at the end of the billing cycle.
4.  **Monthly Cycle:** Usage resets each month, and the base price is paid again, along with any accumulated overages from the previous period.

In essence, it's a "pay-as-you-go" model with a prepaid credit.

---

### **File Structure and Imports**

```typescript
import { db } from '@sim/db'
import { member, subscription, user, userStats } from '@sim/db/schema'
import { and, eq } from 'drizzle-orm'
import { getHighestPrioritySubscription } from '@/lib/billing/core/subscription'
import { getUserUsageData } from '@/lib/billing/core/usage'
import {
  getFreeTierLimit,
  getProTierLimit,
  getTeamTierLimitPerSeat,
} from '@/lib/billing/subscriptions/utils'
import { createLogger } from '@/lib/logs/console/logger'

const logger = createLogger('Billing')
```

This section imports all necessary modules and sets up a logger.

*   **`import { db } from '@sim/db'`**:
    *   **Purpose**: Imports the database connection instance. This `db` object is configured with Drizzle ORM and allows us to perform database queries.
    *   **Deep Explanation**: `db` is likely an initialized Drizzle ORM client, providing methods like `select()`, `insert()`, `update()`, etc., to interact with the underlying database (e.g., PostgreSQL).

*   **`import { member, subscription, user, userStats } from '@sim/db/schema'`**:
    *   **Purpose**: Imports the Drizzle ORM schema definitions for various database tables. These are essentially TypeScript representations of your database tables, allowing type-safe queries.
    *   **Deep Explanation**:
        *   `member`: Represents the table storing information about users who are members of an organization.
        *   `subscription`: Represents the table storing details about user or organization subscriptions (plan, status, period, etc.).
        *   `user`: Represents the main user table.
        *   `userStats`: Represents a table likely holding various statistics or historical data for users, potentially including billing-related snapshots.

*   **`import { and, eq } from 'drizzle-orm'`**:
    *   **Purpose**: Imports helper functions from Drizzle ORM to build complex query conditions.
    *   **Deep Explanation**:
        *   `and`: Used to combine multiple conditions with a logical AND operator in a `WHERE` clause (e.g., `condition1 AND condition2`).
        *   `eq`: Used to check for equality between a column and a value (e.g., `column = 'value'`).

*   **`import { getHighestPrioritySubscription } from '@/lib/billing/core/subscription'`**:
    *   **Purpose**: Imports a utility function that fetches the most relevant active subscription for a given user.
    *   **Deep Explanation**: This function likely handles scenarios where a user might have multiple subscriptions (though typically one active primary) or ensures that a valid, active subscription is prioritized if found. It abstracts away the complexity of determining the "primary" subscription.

*   **`import { getUserUsageData } from '@/lib/billing/core/usage'`**:
    *   **Purpose**: Imports a function to retrieve a user's current usage metrics for the ongoing billing period.
    *   **Deep Explanation**: This function is critical as it provides `currentUsage`, `limit`, `billingPeriodStart`, `billingPeriodEnd`, and `lastPeriodCost` for a user. It's the source of truth for how much a user has consumed.

*   **`import { getFreeTierLimit, getProTierLimit, getTeamTierLimitPerSeat } from '@/lib/billing/subscriptions/utils'`**:
    *   **Purpose**: Imports functions that provide specific configuration values related to plan limits and per-seat pricing.
    *   **Deep Explanation**: These functions likely encapsulate business logic for how much usage each tier covers or costs:
        *   `getFreeTierLimit()`: Returns the maximum usage allowed for the free plan.
        *   `getProTierLimit()`: Returns the monetary equivalent or usage limit associated with the Pro tier's base price.
        *   `getTeamTierLimitPerSeat()`: Returns the monetary equivalent or usage limit associated with each seat in a Team tier.

*   **`import { createLogger } from '@/lib/logs/console/logger'`**:
    *   **Purpose**: Imports a custom logging utility.
    *   **Deep Explanation**: `createLogger` likely configures a logger instance (e.g., Winston, Pino, or a custom console logger) to emit structured logs with specific context, like the module name (`'Billing'` in this case), which aids in debugging and monitoring.

*   **`const logger = createLogger('Billing')`**:
    *   **Purpose**: Initializes a logger instance specifically for this billing module.
    *   **Deep Explanation**: Any log messages from this file will be tagged with `'Billing'`, making it easy to filter and identify billing-related events in application logs.

---

### **`getOrganizationSubscription` Function**

```typescript
/**
 * Get organization subscription directly by organization ID
 */
export async function getOrganizationSubscription(organizationId: string) {
  try {
    const orgSubs = await db
      .select()
      .from(subscription)
      .where(and(eq(subscription.referenceId, organizationId), eq(subscription.status, 'active')))
      .limit(1)

    return orgSubs.length > 0 ? orgSubs[0] : null
  } catch (error) {
    logger.error('Error getting organization subscription', { error, organizationId })
    return null
  }
}
```

*   **Purpose**: This asynchronous function is responsible for retrieving a *single, active* subscription record from the database specifically for a given organization. It's designed to fetch the organization's primary billing subscription.

*   **Simplified Logic**: It queries the `subscription` table, looking for an entry where the `referenceId` matches the provided `organizationId` and the `status` is `'active'`. It expects only one such subscription and returns it.

*   **Deep Line-by-Line Explanation**:
    *   `export async function getOrganizationSubscription(organizationId: string)`:
        *   `export`: Makes the function accessible from other files.
        *   `async`: Declares an asynchronous function, meaning it will perform operations that might take time (like database queries) and will return a `Promise`.
        *   `getOrganizationSubscription`: The name of the function.
        *   `(organizationId: string)`: It accepts one argument, `organizationId`, which is a string.
    *   `try { ... } catch (error) { ... }`: This is a standard `try-catch` block for error handling. If any error occurs within the `try` block, the `catch` block will execute.
    *   `const orgSubs = await db.select().from(subscription) ...`:
        *   `await`: Pauses the execution of the function until the database query completes.
        *   `db.select()`: Starts a Drizzle ORM query to select all columns (`*`) from a table.
        *   `.from(subscription)`: Specifies that the query should target the `subscription` table.
        *   `.where(...)`: Applies a filtering condition to the query.
        *   `and(eq(subscription.referenceId, organizationId), eq(subscription.status, 'active'))`: This is the core of the filtering:
            *   `and(...)`: Combines two conditions with a logical AND. Both conditions must be true for a row to be returned.
            *   `eq(subscription.referenceId, organizationId)`: Checks if the `referenceId` column in the `subscription` table is equal to the provided `organizationId`. The `referenceId` column is likely used to link subscriptions to either users or organizations.
            *   `eq(subscription.status, 'active')`: Checks if the `status` column is `'active'`.
        *   `.limit(1)`: Instructs the database to return at most one matching record. An organization should typically have only one active subscription.
    *   `return orgSubs.length > 0 ? orgSubs[0] : null`:
        *   This is a ternary operator checking the result of the database query.
        *   `orgSubs.length > 0`: If the `orgSubs` array contains one or more elements (meaning an active subscription was found).
        *   `? orgSubs[0]`: Return the first element of the `orgSubs` array (the found subscription).
        *   `: null`: Otherwise (if no subscription was found), return `null`.
    *   `catch (error) { ... }`:
        *   `logger.error('Error getting organization subscription', { error, organizationId })`: If an error occurs during the database operation, it logs the error message along with the `error` object and the `organizationId` for context.
        *   `return null`: In case of an error, the function gracefully returns `null`.

---

### **`getPlanPricing` Function**

```typescript
/**
 * Get plan pricing information
 */
export function getPlanPricing(plan: string): {
  basePrice: number // What they pay upfront via Stripe subscription
} {
  switch (plan) {
    case 'free':
      return { basePrice: 0 } // Free plan has no charges
    case 'pro':
      return { basePrice: getProTierLimit() }
    case 'team':
      return { basePrice: getTeamTierLimitPerSeat() } // Per-seat pricing
    default:
      return { basePrice: 0 }
  }
}
```

*   **Purpose**: This function determines the "base price" (the upfront payment amount) for a given subscription plan. This is the amount that is paid initially via Stripe and acts as credit against usage.

*   **Simplified Logic**: It uses a `switch` statement to return a predefined `basePrice` based on the plan name.

*   **Deep Line-by-Line Explanation**:
    *   `export function getPlanPricing(plan: string): { basePrice: number }`:
        *   `export`: Makes the function available externally.
        *   `function getPlanPricing`: The function name.
        *   `(plan: string)`: It takes a `plan` name as a string (e.g., 'free', 'pro', 'team').
        *   `:{ basePrice: number }`: Specifies that the function returns an object with a single property `basePrice`, which is a number. The comment `// What they pay upfront via Stripe subscription` clarifies its meaning.
    *   `switch (plan) { ... }`: This control flow statement executes different blocks of code based on the value of the `plan` variable.
    *   `case 'free': return { basePrice: 0 }`:
        *   If the `plan` is `'free'`, it returns an object `{ basePrice: 0 }`. The free plan has no upfront charge.
    *   `case 'pro': return { basePrice: getProTierLimit() }`:
        *   If the `plan` is `'pro'`, it returns an object where `basePrice` is determined by calling `getProTierLimit()`. This function (`getProTierLimit()`) is expected to return the monetary value (e.g., $20) that the Pro plan covers, effectively defining its base price.
    *   `case 'team': return { basePrice: getTeamTierLimitPerSeat() }`:
        *   If the `plan` is `'team'`, it returns an object where `basePrice` is determined by `getTeamTierLimitPerSeat()`. This indicates that the Team plan's base price is calculated *per seat*, and this function provides the cost for a single seat.
    *   `default: return { basePrice: 0 }`:
        *   If the `plan` does not match any of the specified `case`s (e.g., an unknown plan), it defaults to a `basePrice` of `0`. This is a safe fallback.

---

### **`calculateUserOverage` Function**

```typescript
/**
 * Calculate overage billing for a user
 * Returns only the amount that exceeds their subscription base price
 */
export async function calculateUserOverage(userId: string): Promise<{
  basePrice: number
  actualUsage: number
  overageAmount: number
  plan: string
} | null> {
  try {
    // Get user's subscription and usage data
    const [subscription, usageData, userRecord] = await Promise.all([
      getHighestPrioritySubscription(userId),
      getUserUsageData(userId),
      db.select().from(user).where(eq(user.id, userId)).limit(1),
    ])

    if (userRecord.length === 0) {
      logger.warn('User not found for overage calculation', { userId })
      return null
    }

    const plan = subscription?.plan || 'free'
    const { basePrice } = getPlanPricing(plan)
    const actualUsage = usageData.currentUsage

    // Calculate overage: any usage beyond what they already paid for
    const overageAmount = Math.max(0, actualUsage - basePrice)

    return {
      basePrice,
      actualUsage,
      overageAmount,
      plan,
    }
  } catch (error) {
    logger.error('Failed to calculate user overage', { userId, error })
    return null
  }
}
```

*   **Purpose**: This asynchronous function calculates the billing overage for a *single user*. It determines how much their actual usage has exceeded the amount covered by their subscription's base price.

*   **Simplified Logic**: It fetches the user's subscription, their current usage, and their user record. It then uses the subscription plan to find the base price, compares it to the actual usage, and returns the difference (the overage), ensuring the overage is never negative.

*   **Deep Line-by-Line Explanation**:
    *   `export async function calculateUserOverage(userId: string): Promise<{ ... } | null>`:
        *   An `async` function that takes a `userId` (string).
        *   It returns a `Promise` that resolves to an object containing `basePrice`, `actualUsage`, `overageAmount`, and `plan`, or `null` if an error occurs or the user isn't found.
    *   `try { ... } catch (error) { ... }`: Standard error handling.
    *   `const [subscription, usageData, userRecord] = await Promise.all([ ... ])`:
        *   This uses `Promise.all` to concurrently fetch three pieces of data, which is more efficient than fetching them sequentially. The `await` keyword waits for all three promises to resolve.
            *   `getHighestPrioritySubscription(userId)`: Fetches the user's primary subscription details.
            *   `getUserUsageData(userId)`: Fetches the user's current usage data for the billing period.
            *   `db.select().from(user).where(eq(user.id, userId)).limit(1)`: Directly queries the `user` table to get the user's record itself, primarily for validation.
    *   `if (userRecord.length === 0) { ... }`:
        *   Checks if the `userRecord` array is empty. If so, the user was not found in the database.
        *   `logger.warn(...)`: Logs a warning message.
        *   `return null`: Returns `null` as overage calculation isn't possible without a user.
    *   `const plan = subscription?.plan || 'free'`:
        *   Determines the user's plan. If `subscription` is `null` (no active subscription found), it defaults to `'free'`. The `?.` (optional chaining) safely accesses `plan` if `subscription` exists.
    *   `const { basePrice } = getPlanPricing(plan)`:
        *   Calls `getPlanPricing` with the determined `plan` to get the upfront base price for that plan. This value represents the amount of usage "covered" by the subscription.
    *   `const actualUsage = usageData.currentUsage`:
        *   Retrieves the user's `currentUsage` value from the `usageData` object.
    *   `const overageAmount = Math.max(0, actualUsage - basePrice)`:
        *   This is the core overage calculation:
            *   `actualUsage - basePrice`: Calculates the raw difference between usage and the covered amount.
            *   `Math.max(0, ...)`: Ensures that the `overageAmount` is never negative. If `actualUsage` is less than or equal to `basePrice`, the result will be 0, meaning no overage.
    *   `return { basePrice, actualUsage, overageAmount, plan }`:
        *   Returns an object containing all the calculated and relevant billing details.
    *   `catch (error) { ... }`:
        *   `logger.error(...)`: Logs any errors that occurred during the calculation.
        *   `return null`: Returns `null` on error.

---

### **`calculateSubscriptionOverage` Function**

```typescript
/**
 * Calculate overage amount for a subscription
 * Shared logic between invoice.finalized and customer.subscription.deleted handlers
 */
export async function calculateSubscriptionOverage(sub: {
  id: string
  plan: string | null
  referenceId: string
  seats?: number | null
}): Promise<number> {
  // Enterprise plans have no overages
  if (sub.plan === 'enterprise') {
    logger.info('Enterprise plan has no overages', {
      subscriptionId: sub.id,
      plan: sub.plan,
    })
    return 0
  }

  let totalOverage = 0

  if (sub.plan === 'team') {
    // Team plan: sum all member usage
    const members = await db
      .select({ userId: member.userId })
      .from(member)
      .where(eq(member.organizationId, sub.referenceId))

    let totalTeamUsage = 0
    for (const m of members) {
      const usage = await getUserUsageData(m.userId)
      totalTeamUsage += usage.currentUsage
    }

    const { basePrice } = getPlanPricing(sub.plan)
    const baseSubscriptionAmount = (sub.seats || 1) * basePrice
    totalOverage = Math.max(0, totalTeamUsage - baseSubscriptionAmount)

    logger.info('Calculated team overage', {
      subscriptionId: sub.id,
      totalTeamUsage,
      baseSubscriptionAmount,
      totalOverage,
    })
  } else if (sub.plan === 'pro') {
    // Pro plan: include snapshot if user joined a team
    const usage = await getUserUsageData(sub.referenceId)
    let totalProUsage = usage.currentUsage

    // Add any snapshotted Pro usage (from when they joined a team)
    const userStatsRows = await db
      .select({ proPeriodCostSnapshot: userStats.proPeriodCostSnapshot })
      .from(userStats)
      .where(eq(userStats.userId, sub.referenceId))
      .limit(1)

    if (userStatsRows.length > 0 && userStatsRows[0].proPeriodCostSnapshot) {
      const snapshotUsage = Number.parseFloat(userStatsRows[0].proPeriodCostSnapshot.toString())
      totalProUsage += snapshotUsage
      logger.info('Including snapshotted Pro usage in overage calculation', {
        userId: sub.referenceId,
        currentUsage: usage.currentUsage,
        snapshotUsage,
        totalProUsage,
      })
    }

    const { basePrice } = getPlanPricing(sub.plan)
    totalOverage = Math.max(0, totalProUsage - basePrice)

    logger.info('Calculated pro overage', {
      subscriptionId: sub.id,
      totalProUsage,
      basePrice,
      totalOverage,
    })
  } else {
    // Free plan or unknown plan type
    const usage = await getUserUsageData(sub.referenceId)
    const { basePrice } = getPlanPricing(sub.plan || 'free')
    totalOverage = Math.max(0, usage.currentUsage - basePrice)

    logger.info('Calculated overage for plan', {
      subscriptionId: sub.id,
      plan: sub.plan || 'free',
      usage: usage.currentUsage,
      basePrice,
      totalOverage,
    })
  }

  return totalOverage
}
```

*   **Purpose**: This crucial asynchronous function calculates the *total* overage amount for a specific subscription. Unlike `calculateUserOverage` which focuses on a single user, this function considers the nature of the subscription (e.g., team vs. individual) and aggregates usage accordingly. It's explicitly designed for backend processes like finalizing invoices or handling subscription cancellations.

*   **Simplified Logic**:
    1.  Immediately returns 0 for 'enterprise' plans (no overages).
    2.  For 'team' plans, it sums up the usage of *all members* in the associated organization and compares it against the total base price (seats * per-seat price).
    3.  For 'pro' plans, it calculates the individual user's usage and *adds any "snapshotted" usage* (historical usage from specific events, like joining a team) before comparing it to the Pro plan's base price.
    4.  For 'free' or other plans, it simply compares the individual user's usage against the plan's base price.
    5.  It always returns 0 if usage is below the covered amount.

*   **Deep Line-by-Line Explanation**:
    *   `export async function calculateSubscriptionOverage(sub: { ... }): Promise<number>`:
        *   An `async` function taking a `sub` object (representing a subscription with `id`, `plan`, `referenceId`, and optional `seats`).
        *   It returns a `Promise` that resolves to a `number` representing the total overage.
    *   `if (sub.plan === 'enterprise') { ... }`:
        *   **Enterprise Exemption**: Checks if the subscription plan is `'enterprise'`. Enterprise plans are explicitly defined as having no overages, so it logs this information and immediately returns `0`.
    *   `let totalOverage = 0`: Initializes a variable to store the calculated overage.
    *   `if (sub.plan === 'team') { ... }`:
        *   **Team Plan Logic**: This block executes if the subscription is a `'team'` plan.
        *   `const members = await db.select({ userId: member.userId }).from(member).where(eq(member.organizationId, sub.referenceId))`:
            *   Queries the `member` table to retrieve all `userId`s that belong to the organization identified by `sub.referenceId`. (The `referenceId` for an organization subscription points to the organization's ID).
        *   `let totalTeamUsage = 0`: Initializes a variable to sum up all team members' usage.
        *   `for (const m of members) { ... }`:
            *   Iterates through each `member` (represented by `m`).
            *   `const usage = await getUserUsageData(m.userId)`: For each member, it fetches their individual usage data.
            *   `totalTeamUsage += usage.currentUsage`: Adds the member's `currentUsage` to the `totalTeamUsage`.
        *   `const { basePrice } = getPlanPricing(sub.plan)`: Gets the *per-seat* base price for the 'team' plan.
        *   `const baseSubscriptionAmount = (sub.seats || 1) * basePrice`:
            *   Calculates the total amount covered by the team's subscription. This is `licensed seats` (from `sub.seats`, defaulting to 1 if not specified) multiplied by the `basePrice` per seat. This represents the total upfront payment for the team.
        *   `totalOverage = Math.max(0, totalTeamUsage - baseSubscriptionAmount)`:
            *   Calculates the overage: the difference between the entire team's `totalTeamUsage` and the `baseSubscriptionAmount`. `Math.max(0, ...)` ensures no negative overage.
        *   `logger.info('Calculated team overage', { ... })`: Logs the details of the team overage calculation.
    *   `else if (sub.plan === 'pro') { ... }`:
        *   **Pro Plan Logic**: This block executes if the subscription is a `'pro'` plan.
        *   `const usage = await getUserUsageData(sub.referenceId)`: Fetches usage data for the individual user associated with the Pro subscription (here, `sub.referenceId` would be the `userId`).
        *   `let totalProUsage = usage.currentUsage`: Initializes `totalProUsage` with the user's current usage.
        *   `const userStatsRows = await db.select({ proPeriodCostSnapshot: userStats.proPeriodCostSnapshot }).from(userStats).where(eq(userStats.userId, sub.referenceId)).limit(1)`:
            *   Queries the `userStats` table to find a `proPeriodCostSnapshot` for the user. This snapshot likely captures historical Pro usage when a user's billing context changed (e.g., they moved from an individual Pro plan to a team, and their current period's Pro usage needed to be accounted for).
        *   `if (userStatsRows.length > 0 && userStatsRows[0].proPeriodCostSnapshot) { ... }`:
            *   If a snapshot exists:
                *   `const snapshotUsage = Number.parseFloat(...)`: Parses the snapshot value to a number.
                *   `totalProUsage += snapshotUsage`: Adds the snapshotted usage to the current usage.
                *   `logger.info(...)`: Logs that snapshot usage is being included.
        *   `const { basePrice } = getPlanPricing(sub.plan)`: Gets the base price for the 'pro' plan.
        *   `totalOverage = Math.max(0, totalProUsage - basePrice)`: Calculates overage for the Pro plan.
        *   `logger.info('Calculated pro overage', { ... })`: Logs the details of the Pro overage calculation.
    *   `else { ... }`:
        *   **Default/Free Plan Logic**: This block handles 'free' plans or any other unhandled plan types.
        *   `const usage = await getUserUsageData(sub.referenceId)`: Fetches usage for the individual user (again, `sub.referenceId` is likely the `userId`).
        *   `const { basePrice } = getPlanPricing(sub.plan || 'free')`: Gets the base price for the plan, defaulting to 'free' if `sub.plan` is `null`.
        *   `totalOverage = Math.max(0, usage.currentUsage - basePrice)`: Calculates overage based on individual usage.
        *   `logger.info('Calculated overage for plan', { ... })`: Logs the details for this default calculation.
    *   `return totalOverage`: Returns the final calculated overage amount.

---

### **`getSimplifiedBillingSummary` Function**

```typescript
/**
 * Get comprehensive billing and subscription summary
 */
export async function getSimplifiedBillingSummary(
  userId: string,
  organizationId?: string
): Promise<{
  type: 'individual' | 'organization'
  plan: string
  basePrice: number
  currentUsage: number
  overageAmount: number
  totalProjected: number
  usageLimit: number
  percentUsed: number
  isWarning: boolean
  isExceeded: boolean
  daysRemaining: number
  // Subscription details
  isPaid: boolean
  isPro: boolean
  isTeam: boolean
  isEnterprise: boolean
  status: string | null
  seats: number | null
  metadata: any
  stripeSubscriptionId: string | null
  periodEnd: Date | string | null
  // Usage details
  usage: {
    current: number
    limit: number
    percentUsed: number
    isWarning: boolean
    isExceeded: boolean
    billingPeriodStart: Date | null
    billingPeriodEnd: Date | null
    lastPeriodCost: number
    daysRemaining: number
  }
  organizationData?: {
    seatCount: number
    memberCount: number
    totalBasePrice: number
    totalCurrentUsage: number
    totalOverage: number
  }
}> {
  try {
    // Get subscription and usage data upfront
    const [subscription, usageData] = await Promise.all([
      organizationId
        ? getOrganizationSubscription(organizationId)
        : getHighestPrioritySubscription(userId),
      getUserUsageData(userId),
    ])

    // Determine subscription type flags
    const plan = subscription?.plan || 'free'
    const isPaid = plan !== 'free'
    const isPro = plan === 'pro'
    const isTeam = plan === 'team'
    const isEnterprise = plan === 'enterprise'

    if (organizationId) {
      // Organization billing summary
      if (!subscription) {
        return getDefaultBillingSummary('organization')
      }

      // Get all organization members
      const members = await db
        .select({ userId: member.userId })
        .from(member)
        .where(eq(member.organizationId, organizationId))

      const { basePrice: basePricePerSeat } = getPlanPricing(subscription.plan)
      // Use licensed seats from Stripe as source of truth
      const licensedSeats = subscription.seats || 1
      const totalBasePrice = basePricePerSeat * licensedSeats // Based on Stripe subscription

      let totalCurrentUsage = 0

      // Calculate total team usage across all members
      for (const memberInfo of members) {
        const memberUsageData = await getUserUsageData(memberInfo.userId)
        totalCurrentUsage += memberUsageData.currentUsage
      }

      // Calculate team-level overage: total usage beyond what was already paid to Stripe
      const totalOverage = Math.max(0, totalCurrentUsage - totalBasePrice)

      // Get user's personal limits for warnings
      const percentUsed =
        usageData.limit > 0 ? Math.round((usageData.currentUsage / usageData.limit) * 100) : 0

      // Calculate days remaining in billing period
      const daysRemaining = usageData.billingPeriodEnd
        ? Math.max(
            0,
            Math.ceil((usageData.billingPeriodEnd.getTime() - Date.now()) / (1000 * 60 * 60 * 24))
          )
        : 0

      return {
        type: 'organization',
        plan: subscription.plan,
        basePrice: totalBasePrice,
        currentUsage: totalCurrentUsage,
        overageAmount: totalOverage,
        totalProjected: totalBasePrice + totalOverage,
        usageLimit: usageData.limit, // NOTE: This is the individual user's limit, not team limit.
        percentUsed,
        isWarning: percentUsed >= 80 && percentUsed < 100,
        isExceeded: usageData.currentUsage >= usageData.limit, // NOTE: This is the individual user's usage
        daysRemaining,
        // Subscription details
        isPaid,
        isPro,
        isTeam,
        isEnterprise,
        status: subscription.status || null,
        seats: subscription.seats || null,
        metadata: subscription.metadata || null,
        stripeSubscriptionId: subscription.stripeSubscriptionId || null,
        periodEnd: subscription.periodEnd || null,
        // Usage details
        usage: {
          current: usageData.currentUsage, // NOTE: This is the individual user's usage
          limit: usageData.limit,
          percentUsed,
          isWarning: percentUsed >= 80 && percentUsed < 100,
          isExceeded: usageData.currentUsage >= usageData.limit,
          billingPeriodStart: usageData.billingPeriodStart,
          billingPeriodEnd: usageData.billingPeriodEnd,
          lastPeriodCost: usageData.lastPeriodCost,
          daysRemaining,
        },
        organizationData: {
          seatCount: licensedSeats,
          memberCount: members.length,
          totalBasePrice,
          totalCurrentUsage,
          totalOverage,
        },
      }
    }

    // Individual billing summary
    const { basePrice } = getPlanPricing(plan)

    // For team and enterprise plans, calculate total team usage instead of individual usage
    let currentUsage = usageData.currentUsage
    if ((isTeam || isEnterprise) && subscription?.referenceId) {
      // Get all team members and sum their usage
      const teamMembers = await db
        .select({ userId: member.userId })
        .from(member)
        .where(eq(member.organizationId, subscription.referenceId))

      let totalTeamUsage = 0
      for (const teamMember of teamMembers) {
        const memberUsageData = await getUserUsageData(teamMember.userId)
        totalTeamUsage += memberUsageData.currentUsage
      }
      currentUsage = totalTeamUsage
    }

    const overageAmount = Math.max(0, currentUsage - basePrice)
    const percentUsed = usageData.limit > 0 ? (currentUsage / usageData.limit) * 100 : 0

    // Calculate days remaining in billing period
    const daysRemaining = usageData.billingPeriodEnd
      ? Math.max(
          0,
          Math.ceil((usageData.billingPeriodEnd.getTime() - Date.now()) / (1000 * 60 * 60 * 24))
        )
      : 0

    return {
      type: 'individual',
      plan,
      basePrice,
      currentUsage: currentUsage,
      overageAmount,
      totalProjected: basePrice + overageAmount,
      usageLimit: usageData.limit,
      percentUsed,
      isWarning: percentUsed >= 80 && percentUsed < 100,
      isExceeded: currentUsage >= usageData.limit,
      daysRemaining,
      // Subscription details
      isPaid,
      isPro,
      isTeam,
      isEnterprise,
      status: subscription?.status || null,
      seats: subscription?.seats || null,
      metadata: subscription?.metadata || null,
      stripeSubscriptionId: subscription?.stripeSubscriptionId || null,
      periodEnd: subscription?.periodEnd || null,
      // Usage details
      usage: {
        current: currentUsage,
        limit: usageData.limit,
        percentUsed,
        isWarning: percentUsed >= 80 && percentUsed < 100,
        isExceeded: currentUsage >= usageData.limit,
        billingPeriodStart: usageData.billingPeriodStart,
        billingPeriodEnd: usageData.billingPeriodEnd,
        lastPeriodCost: usageData.lastPeriodCost,
        daysRemaining,
      },
    }
  } catch (error) {
    logger.error('Failed to get simplified billing summary', { userId, organizationId, error })
    return getDefaultBillingSummary(organizationId ? 'organization' : 'individual')
  }
}
```

*   **Purpose**: This is the most comprehensive function in the file. Its purpose is to generate a detailed, user-friendly billing summary, suitable for display in a dashboard or settings page. It handles both individual user billing and organization-level billing, providing a holistic view of subscription status, usage, and projected costs.

*   **Simplified Logic**:
    1.  It fetches the relevant subscription (either for an individual or an organization) and the user's individual usage data.
    2.  It then branches its logic depending on whether an `organizationId` is provided:
        *   **Organization Summary**: Aggregates usage across all members of the organization, calculates total base price based on licensed seats, and computes team-level overage. It also includes the *individual user's* specific usage limits and warnings for context.
        *   **Individual Summary**: Calculates usage and overage for the individual. Crucially, if the individual is part of a Team or Enterprise plan, their `currentUsage` in the summary will reflect the *total team usage*, not just their own.
    3.  It bundles all this information into a structured object, including flags for warning/exceeded usage, days remaining, and detailed subscription information.

*   **Deep Line-by-Line Explanation**:
    *   `export async function getSimplifiedBillingSummary(userId: string, organizationId?: string): Promise<{ ... }> { ... }`:
        *   An `async` function.
        *   Takes `userId` (string) and an optional `organizationId` (string). The presence of `organizationId` dictates whether it calculates an individual or organizational summary.
        *   Returns a `Promise` resolving to a large, detailed billing summary object.
    *   `try { ... } catch (error) { ... }`: Standard error handling; returns a default summary on error.
    *   `const [subscription, usageData] = await Promise.all([ ... ])`:
        *   Concurrently fetches the subscription and the *individual user's* `usageData`.
        *   `organizationId ? getOrganizationSubscription(organizationId) : getHighestPrioritySubscription(userId)`: Dynamically fetches either the organization's subscription (if `organizationId` is provided) or the user's highest priority subscription.
    *   `const plan = subscription?.plan || 'free'; const isPaid = plan !== 'free'; ...`:
        *   Determines the active plan and sets various boolean flags (`isPaid`, `isPro`, `isTeam`, `isEnterprise`) for easy access and conditional rendering in the UI.
    *   `if (organizationId) { ... }`: **START of Organization Billing Summary Logic**
        *   `if (!subscription) { return getDefaultBillingSummary('organization') }`: If no organization subscription is found, it returns a default "free" summary for the organization context.
        *   `const members = await db.select({ userId: member.userId }).from(member).where(eq(member.organizationId, organizationId))`: Retrieves a list of all `userId`s that are members of the specified organization.
        *   `const { basePrice: basePricePerSeat } = getPlanPricing(subscription.plan)`: Gets the `basePrice` per seat for the organization's plan.
        *   `const licensedSeats = subscription.seats || 1`: Fetches the number of seats licensed under the subscription. Defaults to 1 if `seats` is `null`.
        *   `const totalBasePrice = basePricePerSeat * licensedSeats`: Calculates the total base cost for the organization based on its licensed seats.
        *   `let totalCurrentUsage = 0`: Initializes a variable to accumulate total team usage.
        *   `for (const memberInfo of members) { ... }`: Loops through each member of the organization.
            *   `const memberUsageData = await getUserUsageData(memberInfo.userId)`: Fetches the *individual usage* for each team member.
            *   `totalCurrentUsage += memberUsageData.currentUsage`: Adds each member's individual usage to `totalCurrentUsage`.
        *   `const totalOverage = Math.max(0, totalCurrentUsage - totalBasePrice)`: Calculates the organization-level overage.
        *   `const percentUsed = usageData.limit > 0 ? Math.round((usageData.currentUsage / usageData.limit) * 100) : 0`:
            *   **Important Nuance**: This `percentUsed` calculation, `isWarning`, and `isExceeded` check use the *individual user's* `usageData` (fetched at the beginning). This means that even in an organization summary, the user viewing it will see their *personal* usage status against *their own individual limit*, not necessarily the team's combined usage against a team limit. This provides individual feedback within a team context.
        *   `const daysRemaining = usageData.billingPeriodEnd ? ... : 0`: Calculates the days remaining in the *individual user's* current billing period.
        *   `return { type: 'organization', ... }`: Returns the complete organization billing summary object. It includes:
            *   General summary fields (`plan`, `basePrice`, `currentUsage`, `overageAmount`, etc.).
            *   Detailed subscription info (`isPaid`, `status`, `seats`, etc.).
            *   Detailed individual `usage` info (again, reflecting the individual user's `usageData`).
            *   An `organizationData` object with specific team metrics (`seatCount`, `memberCount`, `totalBasePrice`, `totalCurrentUsage`, `totalOverage`).
    *   `// Individual billing summary`: **START of Individual Billing Summary Logic**
        *   `const { basePrice } = getPlanPricing(plan)`: Gets the base price for the user's plan.
        *   `let currentUsage = usageData.currentUsage`: Initializes `currentUsage` with the *individual user's* current usage.
        *   `if ((isTeam || isEnterprise) && subscription?.referenceId) { ... }`:
            *   **Important Nuance**: If the user is on a 'team' or 'enterprise' plan *and* their subscription is linked to an organization (`subscription.referenceId` exists and points to an organization ID), the `currentUsage` for *this individual summary* will be overridden to reflect the *total usage of their entire team*.
            *   `const teamMembers = await db.select({ userId: member.userId }).from(member).where(eq(member.organizationId, subscription.referenceId))`: Fetches all members of the team.
            *   `let totalTeamUsage = 0; for (const teamMember of teamMembers) { ... }`: Sums up usage for all team members.
            *   `currentUsage = totalTeamUsage`: Assigns the aggregated team usage to `currentUsage`. This means an individual on a team plan will see their team's combined usage as "their" usage in their individual summary.
        *   `const overageAmount = Math.max(0, currentUsage - basePrice)`: Calculates overage based on the (potentially team-aggregated) `currentUsage`.
        *   `const percentUsed = usageData.limit > 0 ? (currentUsage / usageData.limit) * 100 : 0`: Calculates percentage used. This `percentUsed` now reflects the (potentially team-aggregated) `currentUsage` against the *individual's* `usageData.limit`.
        *   `const daysRemaining = usageData.billingPeriodEnd ? ... : 0`: Calculates days remaining in the individual's billing period.
        *   `return { type: 'individual', ... }`: Returns the complete individual billing summary object, similar in structure to the organization one but without `organizationData`, and with `currentUsage` potentially reflecting team usage as described above.
    *   `catch (error) { ... }`:
        *   `logger.error(...)`: Logs any errors.
        *   `return getDefaultBillingSummary(organizationId ? 'organization' : 'individual')`: Returns a default summary if an error occurs.

---

### **`getDefaultBillingSummary` Function**

```typescript
/**
 * Get default billing summary for error cases
 */
function getDefaultBillingSummary(type: 'individual' | 'organization') {
  return {
    type,
    plan: 'free',
    basePrice: 0,
    currentUsage: 0,
    overageAmount: 0,
    totalProjected: 0,
    usageLimit: getFreeTierLimit(),
    percentUsed: 0,
    isWarning: false,
    isExceeded: false,
    daysRemaining: 0,
    // Subscription details
    isPaid: false,
    isPro: false,
    isTeam: false,
    isEnterprise: false,
    status: null,
    seats: null,
    metadata: null,
    stripeSubscriptionId: null,
    periodEnd: null,
    // Usage details
    usage: {
      current: 0,
      limit: getFreeTierLimit(),
      percentUsed: 0,
      isWarning: false,
      isExceeded: false,
      billingPeriodStart: null,
      billingPeriodEnd: null,
      lastPeriodCost: 0,
      daysRemaining: 0,
    },
  }
}
```

*   **Purpose**: This helper function provides a standard, "default" billing summary. It's used in error scenarios (e.g., if a subscription or user cannot be found) or as an initial state, ensuring that the calling code always receives a valid, structured object, typically reflecting a "free tier" status.

*   **Simplified Logic**: It simply returns a hardcoded object with all billing metrics set to their default "free" or zero values.

*   **Deep Line-by-Line Explanation**:
    *   `function getDefaultBillingSummary(type: 'individual' | 'organization')`:
        *   A regular (non-exported, non-async) function.
        *   It takes a `type` argument, which can be either `'individual'` or `'organization'`, allowing the default summary to reflect the correct context.
    *   `return { ... }`: Returns a large object with predefined default values for all properties of the billing summary.
        *   `type`: Set to the `type` argument passed in.
        *   `plan: 'free'`, `basePrice: 0`, `currentUsage: 0`, `overageAmount: 0`, `totalProjected: 0`: All core billing metrics are set to zero or 'free'.
        *   `usageLimit: getFreeTierLimit()`: The usage limit defaults to the limit for the free tier.
        *   `percentUsed: 0`, `isWarning: false`, `isExceeded: false`, `daysRemaining: 0`: Usage-related flags are reset.
        *   `isPaid: false`, `isPro: false`, `isTeam: false`, `isEnterprise: false`: All plan type flags are set to `false`.
        *   `status: null`, `seats: null`, `metadata: null`, `stripeSubscriptionId: null`, `periodEnd: null`: Subscription details are set to `null` as there's no active subscription being summarized.
        *   `usage: { ... }`: A nested `usage` object also mirrors the default "free" state, using `getFreeTierLimit()` for its `limit`.

---

### **Summary**

This `billing.ts` file is a cornerstone of the application's monetization strategy. It meticulously manages the complexities of usage-based billing, accounting for:
*   Different subscription plans (Free, Pro, Team, Enterprise).
*   Individual user usage and organizational team usage.
*   Upfront payments (base price) and post-period overage charges.
*   Historical usage snapshots for specific plan transitions.
*   Provides clear, structured billing summaries for user interfaces and backend processes.

By centralizing these calculations and interactions with the database, it ensures consistency and accuracy in billing, which is critical for any subscription-based service. The use of Drizzle ORM provides type safety and a robust way to interact with the database, while the detailed logging helps in monitoring and debugging billing events.