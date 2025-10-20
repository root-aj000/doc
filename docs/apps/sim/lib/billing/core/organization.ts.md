This TypeScript file is a central component for managing and retrieving billing and usage-related data for organizations within the application. It interacts directly with the database to fetch and update information about organizations, their members, subscriptions, and individual user statistics.

It provides functions to:
1.  **Retrieve comprehensive billing and usage details** for a given organization, including member-level usage.
2.  **Update an organization's custom usage limit**, with specific rules for different subscription plans.
3.  **Generate a summarized billing overview** suitable for an administrative dashboard.
4.  **Check a user's administrative privileges** within an organization.

In essence, this file acts as an **API layer for organization-specific billing and usage data**, encapsulating complex database queries and business logic related to subscriptions, seat management, and usage tracking.

---

## Detailed Explanation

Let's break down the code section by section.

### Imports and Logger Setup

```typescript
import { db } from '@sim/db'
import { member, organization, subscription, user, userStats } from '@sim/db/schema'
import { and, eq } from 'drizzle-orm'
import { getPlanPricing } from '@/lib/billing/core/billing'
import { getFreeTierLimit } from '@/lib/billing/subscriptions/utils'
import { createLogger } from '@/lib/logs/console/logger'

const logger = createLogger('OrganizationBilling')
```

*   **`import { db } from '@sim/db'`**: This line imports the Drizzle ORM database client instance, typically configured to connect to your PostgreSQL database. All database operations will use this `db` object.
*   **`import { member, organization, subscription, user, userStats } from '@sim/db/schema'`**: This imports the Drizzle ORM schema definitions for various tables:
    *   `member`: Represents the relationship between a user and an organization (who is a member of which organization, and their role).
    *   `organization`: Contains details about an organization (name, ID, custom usage limits, etc.).
    *   `subscription`: Stores information about an organization's billing subscription (plan, status, seats, billing periods, etc.).
    *   `user`: Contains basic user information (ID, name, email).
    *   `userStats`: Holds usage statistics for individual users, like their current period cost or usage limit.
    These schema objects are used to build type-safe and declarative database queries.
*   **`import { and, eq } from 'drizzle-orm'`**: These are helper functions from Drizzle ORM:
    *   `and`: Used to combine multiple conditions in a `WHERE` clause with a logical AND.
    *   `eq`: Used to create an equality comparison (e.g., `column = value`).
*   **`import { getPlanPricing } from '@/lib/billing/core/billing'`**: Imports a utility function that likely retrieves the pricing details (e.g., base price per seat) for a given subscription plan. This decouples pricing logic from this file.
*   **`import { getFreeTierLimit } from '@/lib/billing/subscriptions/utils'`**: Imports a utility function that provides the default usage limit for users on a free tier or without specific limits.
*   **`import { createLogger } from '@/lib/logs/console/logger'`**: Imports a function to create a logger instance for structured logging.
*   **`const logger = createLogger('OrganizationBilling')`**: Initializes a logger specific to this file (`OrganizationBilling`), making it easier to trace logs originating from these functions.

### Helper Function: `getOrganizationSubscription`

```typescript
/**
 * Get organization subscription directly by organization ID
 * This is for our new pattern where referenceId = organizationId
 */
async function getOrganizationSubscription(organizationId: string) {
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

This asynchronous helper function is responsible for fetching an organization's active subscription details from the database.

*   **`async function getOrganizationSubscription(organizationId: string)`**: Declares an asynchronous function named `getOrganizationSubscription` that takes one argument: `organizationId` (a string). It's `async` because it performs database operations.
*   **`try { ... } catch (error) { ... }`**: This block handles potential errors during the database operation. If an error occurs, it will be caught, logged, and the function will return `null`.
*   **`const orgSubs = await db ...`**: This is a Drizzle ORM query to retrieve subscription data:
    *   **`.select()`**: Specifies that all columns from the selected table should be retrieved.
    *   **`.from(subscription)`**: Indicates that the query should be performed on the `subscription` table.
    *   **`.where(and(eq(subscription.referenceId, organizationId), eq(subscription.status, 'active')))`**: This is the filtering condition.
        *   `and(...)`: Combines two conditions.
        *   `eq(subscription.referenceId, organizationId)`: Matches subscriptions where the `referenceId` column (which stores the organization ID in this new pattern) equals the provided `organizationId`.
        *   `eq(subscription.status, 'active')`: Further filters to ensure only active subscriptions are considered.
    *   **`.limit(1)`**: Restricts the query to return at most one record, as an organization is expected to have only one active subscription at a time.
*   **`return orgSubs.length > 0 ? orgSubs[0] : null`**: Checks if any subscription record was found. If `orgSubs` array has elements, it returns the first (and only) element; otherwise, it returns `null`.
*   **`logger.error(...)`**: If an error occurs in the `try` block, this line logs the error message along with the `error` object and the `organizationId` for context.
*   **`return null`**: In case of an error, the function explicitly returns `null`.

### Utility Function: `roundCurrency`

```typescript
function roundCurrency(value: number): number {
  return Math.round(value * 100) / 100
}
```

*   **`function roundCurrency(value: number): number`**: Declares a synchronous function `roundCurrency` that takes a `number` (`value`) and returns a `number`.
*   **`return Math.round(value * 100) / 100`**: This common pattern rounds a number to two decimal places, suitable for currency values. It multiplies the value by 100 to shift the decimal two places to the right, rounds it to the nearest whole number, and then divides by 100 to shift the decimal back.

### Data Interfaces

```typescript
interface OrganizationUsageData {
  organizationId: string
  organizationName: string
  subscriptionPlan: string
  subscriptionStatus: string
  totalSeats: number
  usedSeats: number
  seatsCount: number
  totalCurrentUsage: number
  totalUsageLimit: number
  minimumBillingAmount: number
  averageUsagePerMember: number
  billingPeriodStart: Date | null
  billingPeriodEnd: Date | null
  members: MemberUsageData[]
}

interface MemberUsageData {
  userId: string
  userName: string
  userEmail: string
  currentUsage: number
  usageLimit: number
  percentUsed: number
  isOverLimit: boolean
  role: string
  joinedAt: Date
  lastActive: Date | null
}
```

These TypeScript interfaces define the structure of the data that `getOrganizationBillingData` will return. This is crucial for type safety and understanding the output.

*   **`interface OrganizationUsageData`**: Defines the shape of the comprehensive data object for an organization.
    *   `organizationId`, `organizationName`: Basic identification.
    *   `subscriptionPlan`, `subscriptionStatus`: Details about the active subscription.
    *   `totalSeats`, `usedSeats`, `seatsCount`: Information related to licensed and utilized seats. `totalSeats` and `seatsCount` appear to be redundant here, both referring to licensed seats.
    *   `totalCurrentUsage`, `totalUsageLimit`: Aggregated usage and the overall limit for the organization.
    *   `minimumBillingAmount`: The base amount the organization will be billed for its licensed seats/plan.
    *   `averageUsagePerMember`: The average usage across all members.
    *   `billingPeriodStart`, `billingPeriodEnd`: The current billing cycle dates.
    *   `members: MemberUsageData[]`: An array containing detailed usage data for each individual member, conforming to the `MemberUsageData` interface.
*   **`interface MemberUsageData`**: Defines the shape of individual member usage data.
    *   `userId`, `userName`, `userEmail`: Member identification.
    *   `currentUsage`: The member's current usage cost/metric.
    *   `usageLimit`: The individual limit for this member.
    *   `percentUsed`: How much of their limit they have consumed (as a percentage).
    *   `isOverLimit`: A boolean indicating if the member has exceeded their limit.
    *   `role`: The member's role within the organization (e.g., 'owner', 'admin', 'member').
    *   `joinedAt`: The date the member joined the organization.
    *   `lastActive`: The last known activity date for the member, or `null` if not available.

### Main Function: `getOrganizationBillingData`

```typescript
/**
 * Get comprehensive organization billing and usage data
 */
export async function getOrganizationBillingData(
  organizationId: string
): Promise<OrganizationUsageData | null> {
  try {
    // Get organization info
    const orgRecord = await db
      .select()
      .from(organization)
      .where(eq(organization.id, organizationId))
      .limit(1)

    if (orgRecord.length === 0) {
      logger.warn('Organization not found', { organizationId })
      return null
    }

    const organizationData = orgRecord[0]

    // Get organization subscription directly (referenceId = organizationId)
    const subscription = await getOrganizationSubscription(organizationId)

    if (!subscription) {
      logger.warn('No subscription found for organization', { organizationId })
      return null
    }

    // Get all organization members with their usage data
    const membersWithUsage = await db
      .select({
        userId: member.userId,
        userName: user.name,
        userEmail: user.email,
        role: member.role,
        joinedAt: member.createdAt,
        // User stats fields
        currentPeriodCost: userStats.currentPeriodCost,
        currentUsageLimit: userStats.currentUsageLimit,
        lastActive: userStats.lastActive,
      })
      .from(member)
      .innerJoin(user, eq(member.userId, user.id))
      .leftJoin(userStats, eq(member.userId, userStats.userId))
      .where(eq(member.organizationId, organizationId))

    // Process member data
    const members: MemberUsageData[] = membersWithUsage.map((memberRecord) => {
      const currentUsage = Number(memberRecord.currentPeriodCost || 0)
      const usageLimit = Number(memberRecord.currentUsageLimit || getFreeTierLimit())
      const percentUsed = usageLimit > 0 ? (currentUsage / usageLimit) * 100 : 0

      return {
        userId: memberRecord.userId,
        userName: memberRecord.userName,
        userEmail: memberRecord.userEmail,
        currentUsage,
        usageLimit,
        percentUsed: Math.round(percentUsed * 100) / 100,
        isOverLimit: currentUsage > usageLimit,
        role: memberRecord.role,
        joinedAt: memberRecord.joinedAt,
        lastActive: memberRecord.lastActive,
      }
    })

    // Calculate aggregated statistics
    const totalCurrentUsage = members.reduce((sum, member) => sum + member.currentUsage, 0)

    // Get per-seat pricing for the plan
    const { basePrice: pricePerSeat } = getPlanPricing(subscription.plan)

    // Use Stripe subscription seats as source of truth
    // Ensure we always have at least 1 seat (protect against 0 or falsy values)
    const licensedSeats = Math.max(subscription.seats || 1, 1)

    // Calculate minimum billing amount
    let minimumBillingAmount: number
    let totalUsageLimit: number

    if (subscription.plan === 'enterprise') {
      // Enterprise has fixed pricing set through custom Stripe product
      // Their usage limit is configured to match their monthly cost
      const configuredLimit = organizationData.orgUsageLimit
        ? Number.parseFloat(organizationData.orgUsageLimit)
        : 0
      minimumBillingAmount = configuredLimit // For enterprise, this equals their fixed monthly cost
      totalUsageLimit = configuredLimit // Same as their monthly cost
    } else {
      // Team plan: Billing is based on licensed seats from Stripe
      minimumBillingAmount = licensedSeats * pricePerSeat

      // Total usage limit: never below the minimum based on licensed seats
      const configuredLimit = organizationData.orgUsageLimit
        ? Number.parseFloat(organizationData.orgUsageLimit)
        : null
      totalUsageLimit =
        configuredLimit !== null
          ? Math.max(configuredLimit, minimumBillingAmount)
          : minimumBillingAmount
    }

    const averageUsagePerMember = members.length > 0 ? totalCurrentUsage / members.length : 0

    // Billing period comes from the organization's subscription
    const billingPeriodStart = subscription.periodStart || null
    const billingPeriodEnd = subscription.periodEnd || null

    return {
      organizationId,
      organizationName: organizationData.name || '',
      subscriptionPlan: subscription.plan,
      subscriptionStatus: subscription.status || 'inactive',
      totalSeats: Math.max(subscription.seats || 1, 1),
      usedSeats: members.length,
      seatsCount: licensedSeats,
      totalCurrentUsage: roundCurrency(totalCurrentUsage),
      totalUsageLimit: roundCurrency(totalUsageLimit),
      minimumBillingAmount: roundCurrency(minimumBillingAmount),
      averageUsagePerMember: roundCurrency(averageUsagePerMember),
      billingPeriodStart,
      billingPeriodEnd,
      members: members.sort((a, b) => b.currentUsage - a.currentUsage), // Sort by usage desc
    }
  } catch (error) {
    logger.error('Failed to get organization billing data', { organizationId, error })
    throw error
  }
}
```

This is the most complex function, responsible for gathering and processing all billing and usage-related data for an organization.

*   **`export async function getOrganizationBillingData(...)`**: Declares an exported asynchronous function that takes an `organizationId` and promises to return `OrganizationUsageData` or `null`.
*   **`try { ... } catch (error) { ... }`**: The entire operation is wrapped in a `try...catch` block. If any error occurs, it's logged, and then re-thrown (`throw error`) to allow calling functions to handle it.
*   **`// Get organization info`**:
    *   **`const orgRecord = await db.select().from(organization).where(eq(organization.id, organizationId)).limit(1)`**: Fetches the organization's record from the `organization` table using its ID.
    *   **`if (orgRecord.length === 0)`**: Checks if the organization was found. If not, it logs a warning and returns `null`.
    *   **`const organizationData = orgRecord[0]`**: Extracts the single organization record.
*   **`// Get organization subscription directly`**:
    *   **`const subscription = await getOrganizationSubscription(organizationId)`**: Calls the helper function defined earlier to get the active subscription.
    *   **`if (!subscription)`**: Checks if an active subscription was found. If not, it logs a warning and returns `null`.
*   **`// Get all organization members with their usage data`**: This is a multi-table join query.
    *   **`const membersWithUsage = await db.select({ ... }).from(member)`**: Starts a query selecting specific columns (aliased for clarity) from the `member` table.
        *   It selects user identification (`userId`, `userName`, `userEmail`), member role and join date (`role`, `joinedAt`), and user statistics (`currentPeriodCost`, `currentUsageLimit`, `lastActive`).
    *   **`.innerJoin(user, eq(member.userId, user.id))`**: Joins `member` with the `user` table. An `INNER JOIN` means only members who have a corresponding user record will be included. This links member records to actual user details.
    *   **`.leftJoin(userStats, eq(member.userId, userStats.userId))`**: Joins with `userStats`. A `LEFT JOIN` means all members will be included, even if they don't have an entry in `userStats`. For members without `userStats`, the `userStats` fields will be `null`.
    *   **`.where(eq(member.organizationId, organizationId))`**: Filters the results to include only members belonging to the specified `organizationId`.
*   **`// Process member data`**:
    *   **`const members: MemberUsageData[] = membersWithUsage.map((memberRecord) => { ... })`**: Iterates through each `memberRecord` fetched from the database query and transforms it into the `MemberUsageData` interface format.
        *   **`const currentUsage = Number(memberRecord.currentPeriodCost || 0)`**: Converts the `currentPeriodCost` to a number. `|| 0` handles cases where `currentPeriodCost` might be `null` (due to `LEFT JOIN` with `userStats` or if the field is empty), defaulting it to 0.
        *   **`const usageLimit = Number(memberRecord.currentUsageLimit || getFreeTierLimit())`**: Similar conversion for `currentUsageLimit`. If `currentUsageLimit` is `null` or 0, it falls back to a default free tier limit using `getFreeTierLimit()`.
        *   **`const percentUsed = usageLimit > 0 ? (currentUsage / usageLimit) * 100 : 0`**: Calculates the percentage of usage. It checks `usageLimit > 0` to prevent division by zero.
        *   **`percentUsed: Math.round(percentUsed * 100) / 100`**: Rounds the `percentUsed` to two decimal places for cleaner display.
        *   **`isOverLimit: currentUsage > usageLimit`**: A boolean flag to easily identify if a member has exceeded their limit.
*   **`// Calculate aggregated statistics`**:
    *   **`const totalCurrentUsage = members.reduce((sum, member) => sum + member.currentUsage, 0)`**: Calculates the sum of `currentUsage` for all members using the `reduce` array method.
*   **`// Get per-seat pricing for the plan`**:
    *   **`const { basePrice: pricePerSeat } = getPlanPricing(subscription.plan)`**: Calls `getPlanPricing` with the subscription plan to get the `basePrice` for each seat, aliased as `pricePerSeat`.
*   **`// Use Stripe subscription seats as source of truth`**:
    *   **`const licensedSeats = Math.max(subscription.seats || 1, 1)`**: Determines the number of licensed seats. It uses `subscription.seats` (from Stripe), but defaults to `1` if `seats` is `null` or `0`, ensuring a minimum of `1` seat.
*   **`// Calculate minimum billing amount`**: This section contains conditional logic based on the subscription plan.
    *   **`if (subscription.plan === 'enterprise')`**:
        *   **`const configuredLimit = organizationData.orgUsageLimit ? Number.parseFloat(organizationData.orgUsageLimit) : 0`**: For Enterprise plans, the `orgUsageLimit` stored in the `organization` table is treated as both the fixed monthly cost and the usage limit. `Number.parseFloat` converts the string value from the database to a number, with a fallback to `0`.
        *   **`minimumBillingAmount = configuredLimit`**: The minimum billing is set to this configured limit.
        *   **`totalUsageLimit = configuredLimit`**: The total usage limit is also set to this configured limit.
    *   **`else { // Team plan: ... }`**: For "team" plans (or any other non-enterprise plan).
        *   **`minimumBillingAmount = licensedSeats * pricePerSeat`**: The minimum billing is calculated by multiplying the `licensedSeats` by the `pricePerSeat`.
        *   **`const configuredLimit = organizationData.orgUsageLimit ? Number.parseFloat(organizationData.orgUsageLimit) : null`**: Fetches the custom `orgUsageLimit` from the organization data, if it exists.
        *   **`totalUsageLimit = configuredLimit !== null ? Math.max(configuredLimit, minimumBillingAmount) : minimumBillingAmount`**: The total usage limit is determined: if a `configuredLimit` exists, the actual limit is the *maximum* of the `configuredLimit` and the `minimumBillingAmount` (ensuring the limit is never below the base cost). If no `configuredLimit` is set, it defaults to the `minimumBillingAmount`.
*   **`const averageUsagePerMember = members.length > 0 ? totalCurrentUsage / members.length : 0`**: Calculates the average usage per member, preventing division by zero if there are no members.
*   **`const billingPeriodStart = subscription.periodStart || null`**: Retrieves the billing period start date from the subscription, defaulting to `null` if not available.
*   **`const billingPeriodEnd = subscription.periodEnd || null`**: Retrieves the billing period end date from the subscription, defaulting to `null` if not available.
*   **`return { ... }`**: Constructs and returns the final `OrganizationUsageData` object, applying `roundCurrency` to all relevant numerical values and sorting members by `currentUsage` in descending order.

### Function: `updateOrganizationUsageLimit`

```typescript
/**
 * Update organization usage limit (cap)
 */
export async function updateOrganizationUsageLimit(
  organizationId: string,
  newLimit: number
): Promise<{ success: boolean; error?: string }> {
  try {
    // Validate the organization exists
    const orgRecord = await db
      .select()
      .from(organization)
      .where(eq(organization.id, organizationId))
      .limit(1)

    if (orgRecord.length === 0) {
      return { success: false, error: 'Organization not found' }
    }

    // Get subscription to validate minimum
    const subscription = await getOrganizationSubscription(organizationId)
    if (!subscription) {
      return { success: false, error: 'No active subscription found' }
    }

    // Enterprise plans have fixed usage limits that cannot be changed
    if (subscription.plan === 'enterprise') {
      return {
        success: false,
        error: 'Enterprise plans have fixed usage limits that cannot be changed',
      }
    }

    // Only team plans can update their usage limits
    if (subscription.plan !== 'team') {
      return {
        success: false,
        error: 'Only team organizations can update usage limits',
      }
    }

    // Team plans have minimum based on seats
    const { basePrice } = getPlanPricing(subscription.plan)
    const minimumLimit = Math.max(subscription.seats || 1, 1) * basePrice

    // Validate new limit is not below minimum
    if (newLimit < minimumLimit) {
      return {
        success: false,
        error: `Usage limit cannot be less than minimum billing amount of $${roundCurrency(minimumLimit).toFixed(2)}`,
      }
    }

    // Update the organization usage limit
    // Convert number to string for decimal column
    await db
      .update(organization)
      .set({
        orgUsageLimit: roundCurrency(newLimit).toFixed(2),
        updatedAt: new Date(),
      })
      .where(eq(organization.id, organizationId))

    logger.info('Organization usage limit updated', {
      organizationId,
      newLimit,
      minimumLimit,
    })

    return { success: true }
  } catch (error) {
    logger.error('Failed to update organization usage limit', {
      organizationId,
      newLimit,
      error,
    })
    return {
      success: false,
      error: 'Failed to update usage limit',
    }
  }
}
```

This function allows updating an organization's `orgUsageLimit` (custom cap), with several validation steps.

*   **`export async function updateOrganizationUsageLimit(...)`**: Declares an exported async function that takes `organizationId` and `newLimit`, returning an object indicating `success` and potentially an `error` message.
*   **`try { ... } catch (error) { ... }`**: Handles errors during the update process.
*   **`// Validate the organization exists`**:
    *   Fetches the `organization` record to ensure it exists. If not, returns `success: false` with an error.
*   **`// Get subscription to validate minimum`**:
    *   Fetches the organization's `subscription` using `getOrganizationSubscription`. If no active subscription is found, returns an error.
*   **`// Enterprise plans have fixed usage limits that cannot be changed`**:
    *   Checks if the plan is 'enterprise'. If so, it returns an error because enterprise limits are not customizable through this function.
*   **`// Only team plans can update their usage limits`**:
    *   Further validates that the plan is specifically 'team'. If not, it returns an error. This implies only 'team' plans have flexible usage limits.
*   **`// Team plans have minimum based on seats`**:
    *   **`const { basePrice } = getPlanPricing(subscription.plan)`**: Gets the base price for the 'team' plan.
    *   **`const minimumLimit = Math.max(subscription.seats || 1, 1) * basePrice`**: Calculates the `minimumLimit` based on licensed seats and the base price, ensuring it's never below 1 seat's cost.
*   **`// Validate new limit is not below minimum`**:
    *   **`if (newLimit < minimumLimit)`**: Compares the `newLimit` with the calculated `minimumLimit`. If `newLimit` is too low, it returns an error with a formatted message showing the minimum allowed.
*   **`// Update the organization usage limit`**:
    *   **`await db.update(organization).set({ ... }).where(...)`**: Performs the database update.
        *   **`.set({ orgUsageLimit: roundCurrency(newLimit).toFixed(2), updatedAt: new Date() })`**: Sets the `orgUsageLimit` to the `newLimit` (rounded and converted to a string for the database decimal column) and updates the `updatedAt` timestamp.
        *   **`.where(eq(organization.id, organizationId))`**: Ensures only the specified organization's record is updated.
*   **`logger.info(...)`**: Logs a success message with relevant details.
*   **`return { success: true }`**: Indicates a successful update.
*   **`catch (error)`**: Logs any unexpected errors and returns `success: false` with a generic error message.

### Function: `getOrganizationBillingSummary`

```typescript
/**
 * Get organization billing summary for admin dashboard
 */
export async function getOrganizationBillingSummary(organizationId: string) {
  try {
    const billingData = await getOrganizationBillingData(organizationId)

    if (!billingData) {
      return null
    }

    // Calculate additional metrics
    const membersOverLimit = billingData.members.filter((m) => m.isOverLimit).length
    const membersNearLimit = billingData.members.filter(
      (m) => !m.isOverLimit && m.percentUsed >= 80
    ).length

    const topUsers = billingData.members.slice(0, 5).map((m) => ({
      name: m.userName,
      usage: m.currentUsage,
      limit: m.usageLimit,
      percentUsed: m.percentUsed,
    }))

    return {
      organization: {
        id: billingData.organizationId,
        name: billingData.organizationName,
        plan: billingData.subscriptionPlan,
        status: billingData.subscriptionStatus,
      },
      usage: {
        total: billingData.totalCurrentUsage,
        limit: billingData.totalUsageLimit,
        average: billingData.averageUsagePerMember,
        percentUsed:
          billingData.totalUsageLimit > 0
            ? (billingData.totalCurrentUsage / billingData.totalUsageLimit) * 100
            : 0,
      },
      seats: {
        total: billingData.totalSeats,
        used: billingData.usedSeats,
        available: billingData.totalSeats - billingData.usedSeats,
      },
      alerts: {
        membersOverLimit,
        membersNearLimit,
      },
      billingPeriod: {
        start: billingData.billingPeriodStart,
        end: billingData.billingPeriodEnd,
      },
      topUsers,
    }
  } catch (error) {
    logger.error('Failed to get organization billing summary', { organizationId, error })
    throw error
  }
}
```

This function leverages `getOrganizationBillingData` to produce a simplified, aggregated summary, suitable for display in an administrative dashboard.

*   **`export async function getOrganizationBillingSummary(organizationId: string)`**: Declares an exported async function.
*   **`try { ... } catch (error) { ... }`**: Handles errors, logging and re-throwing them.
*   **`const billingData = await getOrganizationBillingData(organizationId)`**: Calls the comprehensive data retrieval function.
*   **`if (!billingData)`**: If `getOrganizationBillingData` returns `null` (e.g., organization or subscription not found), this function also returns `null`.
*   **`// Calculate additional metrics`**:
    *   **`const membersOverLimit = billingData.members.filter((m) => m.isOverLimit).length`**: Counts members whose `isOverLimit` flag is true.
    *   **`const membersNearLimit = billingData.members.filter((m) => !m.isOverLimit && m.percentUsed >= 80).length`**: Counts members who are not yet over limit but have used 80% or more of their usage.
    *   **`const topUsers = billingData.members.slice(0, 5).map(...)`**: Selects the top 5 members (already sorted by usage in `getOrganizationBillingData`) and maps them to a simplified object with `name`, `usage`, `limit`, and `percentUsed`.
*   **`return { ... }`**: Constructs a structured summary object with various categories:
    *   `organization`: Basic org and plan details.
    *   `usage`: Total, limit, average, and overall percentage of usage.
    *   `seats`: Total licensed seats, used seats, and calculated available seats.
    *   `alerts`: Counts of members over or near their limits.
    *   `billingPeriod`: Start and end dates of the current billing cycle.
    *   `topUsers`: The list of top 5 users by usage.

### Function: `isOrganizationOwnerOrAdmin`

```typescript
/**
 * Check if a user is an owner or admin of a specific organization
 *
 * @param userId - The ID of the user to check
 * @param organizationId - The ID of the organization
 * @returns Promise<boolean> - True if the user is an owner or admin of the organization
 */
export async function isOrganizationOwnerOrAdmin(
  userId: string,
  organizationId: string
): Promise<boolean> {
  try {
    const memberRecord = await db
      .select({ role: member.role })
      .from(member)
      .where(and(eq(member.userId, userId), eq(member.organizationId, organizationId)))
      .limit(1)

    if (memberRecord.length === 0) {
      return false
    }

    const userRole = memberRecord[0].role
    return ['owner', 'admin'].includes(userRole)
  } catch (error) {
    logger.error('Error checking organization ownership/admin status:', error)
    return false
  }
}
```

This function determines if a given user holds an 'owner' or 'admin' role within a specific organization.

*   **`export async function isOrganizationOwnerOrAdmin(...)`**: Declares an exported async function that takes `userId` and `organizationId`, returning a `Promise<boolean>`.
*   **`try { ... } catch (error) { ... }`**: Handles errors.
*   **`const memberRecord = await db.select({ role: member.role }).from(member).where(and(eq(member.userId, userId), eq(member.organizationId, organizationId))).limit(1)`**: Queries the `member` table to find the role of the specified `userId` within the `organizationId`. It only selects the `role` column for efficiency.
*   **`if (memberRecord.length === 0)`**: If no matching member record is found, the user is not part of the organization, so it returns `false`.
*   **`const userRole = memberRecord[0].role`**: Extracts the role from the found record.
*   **`return ['owner', 'admin'].includes(userRole)`**: Checks if the `userRole` is either 'owner' or 'admin' using the `includes` array method. If it is one of these, it returns `true`; otherwise, `false`.
*   **`catch (error)`**: Logs any errors and returns `false` to indicate that the check failed.

---

This file encapsulates significant business logic and database interactions, providing a robust and centralized way to handle organization billing and usage data. The use of Drizzle ORM ensures type safety and a clear way to construct database queries. The separation of concerns into helper functions and the main data retrieval function makes the codebase modular and easier to maintain.