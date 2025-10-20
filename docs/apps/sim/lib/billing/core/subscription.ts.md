This TypeScript file, `subscription-core.ts`, acts as the **central nervous system for managing user subscriptions and billing-related logic** within the application. Its primary goal is to provide a single, reliable source of truth for determining a user's subscription status, available features, and billing limits. It consolidates and simplifies various subscription-related checks from different parts of the codebase into a cohesive API.

In essence, it helps answer critical questions like:
*   What is the highest-tier subscription a user has?
*   Is a user on a Pro, Team, or Enterprise plan?
*   Has a user exceeded their billing usage limits?
*   What is a user's complete subscription state (plan name, limits, etc.)?
*   Should a welcome email be sent for a new subscription?

It leverages database queries (using Drizzle ORM) to fetch subscription and user data, combined with utility functions to interpret plan details and calculate limits.

---

### Deep Dive into the Code

Let's break down each part of the file, explaining its purpose and the logic behind it.

#### Imports

The file starts by importing necessary modules and types from various parts of the application and external libraries.

```typescript
import { db } from '@sim/db' // Imports the Drizzle ORM database client instance. This is how the application interacts with its database.
import { member, subscription, user, userStats } from '@sim/db/schema' // Imports the Drizzle ORM schema definitions for specific database tables: `member`, `subscription`, `user`, and `userStats`. These objects represent the tables in a type-safe way for queries.
import { and, eq, inArray } from 'drizzle-orm' // Imports Drizzle ORM's utility functions for building complex SQL `WHERE` clauses:
                                              // `and`: Combines multiple conditions with a logical AND.
                                              // `eq`: Checks if a column is equal to a value.
                                              // `inArray`: Checks if a column's value is present in an array of values.
import {
  checkEnterprisePlan,
  checkProPlan,
  checkTeamPlan,
  getFreeTierLimit,
  getPerUserMinimumLimit,
} from '@/lib/billing/subscriptions/utils' // Imports utility functions specifically designed to check subscription plan types (e.g., `checkProPlan` returns true if a subscription object represents a Pro plan) and get billing limits.
import type { UserSubscriptionState } from '@/lib/billing/types' // Imports a TypeScript type definition for `UserSubscriptionState`. This ensures type safety when defining the structure of the comprehensive subscription state object.
import { isProd } from '@/lib/environment' // Imports a boolean flag `isProd` which is `true` if the application is running in a production environment, and `false` otherwise. This is often used for feature toggles or debugging.
import { createLogger } from '@/lib/logs/console/logger' // Imports a utility function to create a logger instance. This allows the file to output informational, warning, and error messages to the console or a logging system.
import { getBaseUrl } from '@/lib/urls/utils' // Imports a utility function to retrieve the base URL of the application. This is typically used when constructing links in emails or redirects.
```

#### Logger Initialization

```typescript
const logger = createLogger('SubscriptionCore') // Creates a logger instance specifically for this file, named 'SubscriptionCore'. This helps categorize log messages originating from this module.
```

#### File Description Comments

```typescript
/**
 * Core subscription management - single source of truth
 * Consolidates logic from both lib/subscription.ts and lib/subscription/subscription.ts
 */
// This comment explains the high-level purpose of the file: to be the authoritative module for subscription management, centralizing logic previously scattered across multiple files.
```

---

### `getHighestPrioritySubscription(userId: string)`

This asynchronous function is crucial. Its purpose is to **find the most valuable (highest priority) active subscription for a given user.** Subscriptions can be personal (direct to the user) or organizational (through a team the user is a member of). The priority order is defined as Enterprise > Team > Pro > Free.

#### Simplified Logic

1.  **Find personal subscriptions:** Query the database for any active subscriptions directly associated with the `userId`.
2.  **Find organization IDs:** Query the database to find all organizations the `userId` is a member of.
3.  **Find organization subscriptions:** If the user is part of any organizations, query the database for active subscriptions associated with those organization IDs.
4.  **Combine and filter:** Put all personal and organizational active subscriptions together.
5.  **Prioritize:** Iterate through the combined subscriptions, checking for Enterprise, then Team, then Pro plans. The first one found according to this priority is returned.
6.  **No subscriptions:** If no active subscriptions are found (personal or organizational), return `null`.
7.  **Error Handling:** Log any errors and return `null` if something goes wrong during the process.

#### Line-by-Line Explanation

```typescript
export async function getHighestPrioritySubscription(userId: string) {
  try {
    // Start of a try-catch block for robust error handling.
    // Any error during database operations or logic will be caught.

    // Get direct subscriptions
    const personalSubs = await db
      .select() // Selects all columns from the subscription table.
      .from(subscription) // Specifies the `subscription` table as the source.
      .where(and(eq(subscription.referenceId, userId), eq(subscription.status, 'active')))
      // Filters the subscriptions:
      // `eq(subscription.referenceId, userId)`: Matches subscriptions where the `referenceId` (which is the user ID for personal subscriptions) equals the provided `userId`.
      // `eq(subscription.status, 'active')`: Ensures only subscriptions with an 'active' status are considered.
      // `and(...)`: Combines these two conditions, meaning *both* must be true.

    // Get organization memberships
    const memberships = await db
      .select({ organizationId: member.organizationId }) // Selects only the `organizationId` column from the `member` table.
                                                         // The `{ organizationId: member.organizationId }` syntax renames the column in the result if needed, though here it just explicitly states the column.
      .from(member) // Specifies the `member` table.
      .where(eq(member.userId, userId)) // Filters to find all memberships where the `userId` matches the input `userId`.
                                        // This gives us a list of organizations the user belongs to.

    const orgIds = memberships.map((m: { organizationId: string }) => m.organizationId)
    // Transforms the array of membership objects (e.g., `[{ organizationId: 'org1' }]`) into a plain array of organization IDs (e.g., `['org1']`).
    // This is necessary for the `inArray` Drizzle clause later.

    // Get organization subscriptions
    let orgSubs: any[] = [] // Initializes an empty array to hold subscriptions belonging to organizations.
    if (orgIds.length > 0) {
      // Only proceed if the user is a member of at least one organization.
      orgSubs = await db
        .select() // Selects all columns from the subscription table.
        .from(subscription) // Specifies the `subscription` table.
        .where(and(inArray(subscription.referenceId, orgIds), eq(subscription.status, 'active')))
        // Filters organization subscriptions:
        // `inArray(subscription.referenceId, orgIds)`: Matches subscriptions where the `referenceId` (which is the organization ID for organizational subscriptions) is one of the `orgIds` found earlier.
        // `eq(subscription.status, 'active')`: Ensures only active organization subscriptions are considered.
        // `and(...)`: Combines these conditions.
    }

    const allSubs = [...personalSubs, ...orgSubs] // Combines the personal and organizational subscriptions into a single array.

    if (allSubs.length === 0) return null // If no active personal or organizational subscriptions are found, return `null`.

    // Return highest priority subscription
    // The following blocks implement the priority logic: Enterprise > Team > Pro.
    // `find()` method iterates through the array and returns the *first* element that satisfies the provided testing function.

    const enterpriseSub = allSubs.find((s) => checkEnterprisePlan(s))
    // Tries to find an Enterprise plan subscription among all active subscriptions.
    // `checkEnterprisePlan(s)` is an external utility function that determines if a given subscription object `s` is an Enterprise plan.
    if (enterpriseSub) return enterpriseSub // If an Enterprise plan is found, it's the highest priority, so return it immediately.

    const teamSub = allSubs.find((s) => checkTeamPlan(s))
    // If no Enterprise plan, try to find a Team plan subscription.
    if (teamSub) return teamSub // If a Team plan is found, return it.

    const proSub = allSubs.find((s) => checkProPlan(s))
    // If no Enterprise or Team plan, try to find a Pro plan subscription.
    if (proSub) return proSub // If a Pro plan is found, return it.

    return null // If no Enterprise, Team, or Pro plan is found, return `null` (implying a 'Free' tier or no specific plan).
  } catch (error) {
    logger.error('Error getting highest priority subscription', { error, userId })
    // Catches any errors that occurred within the try block.
    // Logs the error with context (`error` object and `userId`).
    return null // Returns `null` on error to indicate failure to retrieve a subscription.
  }
}
```

---

### `isProPlan(userId: string): Promise<boolean>`

This function checks if a user is currently on a "Pro-level" plan. This includes Pro, Team, and Enterprise plans, as they typically offer all features available in the Pro plan and more.

#### Simplified Logic

1.  **Non-production bypass:** If not in production, always return `true` (likely for easier development/testing).
2.  **Get highest subscription:** Call `getHighestPrioritySubscription` to find the user's best plan.
3.  **Check plan type:** If a subscription is found, check if it's a Pro, Team, or Enterprise plan using utility functions.
4.  **Log and return:** Log the plan type if it's pro-level, then return `true` or `false`.
5.  **Error Handling:** Log errors and return `false`.

#### Line-by-Line Explanation

```typescript
export async function isProPlan(userId: string): Promise<boolean> {
  try {
    if (!isProd) {
      // If the application is NOT running in a production environment (`isProd` is false),
      return true // always return `true`. This is a common pattern for local development or staging to bypass billing restrictions.
    }

    const subscription = await getHighestPrioritySubscription(userId)
    // Calls the previously defined function to get the highest priority active subscription for the user.

    const isPro =
      subscription && // First, check if a subscription object actually exists (it's not `null`).
      (checkProPlan(subscription) || // If it exists, check if it's a Pro plan.
        checkTeamPlan(subscription) || // OR check if it's a Team plan.
        checkEnterprisePlan(subscription)) // OR check if it's an Enterprise plan.
    // If any of these conditions are true, `isPro` will be true. This captures all plans that are "at least Pro-level".

    if (isPro) {
      logger.info('User has pro-level plan', { userId, plan: subscription.plan })
      // If the user has a pro-level plan, log an informational message with the user ID and the specific plan name.
      // `subscription.plan` would be 'pro', 'team', or 'enterprise'.
    }

    return !!isPro // Converts `isPro` to a strict boolean value (e.g., `true` or `false`).
  } catch (error) {
    logger.error('Error checking pro plan status', { error, userId })
    // Catches errors, logs them, and
    return false // returns `false` to indicate that the user is not on a Pro plan (or at least, the check failed).
  }
}
```

---

### `isTeamPlan(userId: string): Promise<boolean>`

Similar to `isProPlan`, this function specifically checks if a user is on a "Team-level" plan, which includes Team and Enterprise plans.

#### Simplified Logic

Similar to `isProPlan`, but only checks for Team or Enterprise plans.

#### Line-by-Line Explanation

```typescript
export async function isTeamPlan(userId: string): Promise<boolean> {
  try {
    if (!isProd) {
      return true // Bypass for non-production environments.
    }

    const subscription = await getHighestPrioritySubscription(userId)
    // Gets the user's highest priority subscription.

    const isTeam =
      subscription && // Check if a subscription exists.
      (checkTeamPlan(subscription) || // If it exists, check if it's a Team plan.
        checkEnterprisePlan(subscription)) // OR check if it's an Enterprise plan.
    // This captures plans that are "at least Team-level".

    if (isTeam) {
      logger.info('User has team-level plan', { userId, plan: subscription.plan })
      // Log if a team-level plan is found.
    }

    return !!isTeam // Return the boolean result.
  } catch (error) {
    logger.error('Error checking team plan status', { error, userId })
    return false // Log error and return `false`.
  }
}
```

---

### `isEnterprisePlan(userId: string): Promise<boolean>`

This function checks if a user is specifically on an Enterprise plan.

#### Simplified Logic

Similar to `isProPlan`, but only checks for Enterprise plans.

#### Line-by-Line Explanation

```typescript
export async function isEnterprisePlan(userId: string): Promise<boolean> {
  try {
    if (!isProd) {
      return true // Bypass for non-production environments.
    }

    const subscription = await getHighestPrioritySubscription(userId)
    // Gets the user's highest priority subscription.

    const isEnterprise = subscription && checkEnterprisePlan(subscription)
    // Check if a subscription exists AND if it's specifically an Enterprise plan.

    if (isEnterprise) {
      logger.info('User has enterprise plan', { userId, plan: subscription.plan })
      // Log if an Enterprise plan is found.
    }

    return !!isEnterprise // Return the boolean result.
  } catch (error) {
    logger.error('Error checking enterprise plan status', { error, userId })
    return false // Log error and return `false`.
  }
}
```

---

### `hasExceededCostLimit(userId: string): Promise<boolean>`

This function determines if a user has gone over their allowed billing cost limit for the current period. The limit can vary based on their subscription plan.

#### Simplified Logic

1.  **Non-production bypass:** If not in production, always return `false` (no limits apply).
2.  **Get highest subscription:** Find the user's highest priority plan.
3.  **Determine limit:**
    *   Start with the default free tier limit.
    *   If a subscription exists:
        *   For Team/Enterprise plans, dynamically import and use an organization-specific usage limit function.
        *   For Pro/Free (individual) plans, use a per-user minimum limit utility.
4.  **Fetch user stats:** Get the user's current usage statistics from the database.
5.  **Calculate cost:** Extract the `currentPeriodCost` (or `totalCost` as a fallback) from the user stats.
6.  **Compare:** Check if the `currentCost` is greater than or equal to the determined `limit`.
7.  **Log and return:** Log the check and return the comparison result.
8.  **Error Handling:** Log errors and return `false` (conservative in case of error).

#### Line-by-Line Explanation

```typescript
export async function hasExceededCostLimit(userId: string): Promise<boolean> {
  try {
    if (!isProd) {
      return false // If not in production, return `false` (no cost limits apply).
    }

    const subscription = await getHighestPrioritySubscription(userId)
    // Get the user's highest priority subscription.

    let limit = getFreeTierLimit() // Initialize `limit` with the default free tier limit.

    if (subscription) {
      // If the user has an active subscription:
      if (subscription.plan === 'team' || subscription.plan === 'enterprise') {
        // If it's a Team or Enterprise plan, the limit is often organization-wide or more dynamic.
        // We dynamically `import` the `getUserUsageLimit` function because it might be a larger module
        // and only needed for these specific plan types, helping with bundle size or reducing initial load.
        const { getUserUsageLimit } = await import('@/lib/billing/core/usage')
        limit = await getUserUsageLimit(userId) // Get the specific usage limit for the user (potentially organization-based).
        logger.info('Using organization limit', {
          userId,
          plan: subscription.plan,
          limit,
        })
      } else {
        // For Pro/Free individual plans, use a simpler per-user minimum limit.
        limit = getPerUserMinimumLimit(subscription) // Get the limit based on the individual subscription.
        logger.info('Using subscription-based limit', {
          userId,
          plan: subscription.plan,
          limit,
        })
      }
    } else {
      // If no subscription is found, the user is on the free tier.
      logger.info('Using free tier limit', { userId, limit })
    }

    // Get user stats to check current period usage
    const statsRecords = await db
      .select() // Select all columns.
      .from(userStats) // From the `userStats` table.
      .where(eq(userStats.userId, userId)) // Where the `userId` matches.

    if (statsRecords.length === 0) {
      // If no user stats are found for the user, it means they haven't incurred any costs yet,
      return false // so they cannot have exceeded a limit.
    }

    // Use current period cost instead of total cost for accurate billing period tracking
    const currentCost = Number.parseFloat(
      statsRecords[0].currentPeriodCost?.toString() || statsRecords[0].totalCost.toString()
    )
    // Extracts the cost from the first (and likely only) stats record.
    // It prioritizes `currentPeriodCost` (for accurate billing cycle tracking).
    // If `currentPeriodCost` is `null` or `undefined`, it falls back to `totalCost`.
    // `.toString()` ensures it's a string before `Number.parseFloat()` handles potential decimal values.

    logger.info('Checking cost limit', { userId, currentCost, limit })
    // Log the current cost and the determined limit for debugging and monitoring.

    return currentCost >= limit // Returns `true` if the current cost has met or exceeded the limit, `false` otherwise.
  } catch (error) {
    logger.error('Error checking cost limit', { error, userId })
    // Catches errors, logs them.
    return false // Returns `false` as a conservative default in case of an error (better to allow usage than block mistakenly).
  }
}
```

---

### `getUserSubscriptionState(userId: string): Promise<UserSubscriptionState>`

This is a comprehensive function designed to provide a complete picture of a user's subscription status in a single object. It gathers all relevant information (plan type, limits, highest subscription object) efficiently.

#### Simplified Logic

1.  **Parallel Data Fetching:** Fetch the highest priority subscription and user stats simultaneously using `Promise.all` to optimize performance.
2.  **Determine Plan Types:** Based on the fetched subscription, determine `isPro`, `isTeam`, `isEnterprise`, and `isFree` flags. Apply non-production bypass for these checks.
3.  **Determine Plan Name:** Assign a human-readable plan name ('free', 'pro', 'team', 'enterprise') based on the determined plan types.
4.  **Check Cost Limit (reusing stats):** Re-evaluate `hasExceededLimit` using the already fetched `statsRecords` and the same limit determination logic as `hasExceededCostLimit`.
5.  **Return State Object:** Compile all this information into a `UserSubscriptionState` object.
6.  **Error Handling:** Log errors and return a default "free tier" state.

#### Line-by-Line Explanation

```typescript
export async function getUserSubscriptionState(userId: string): Promise<UserSubscriptionState> {
  try {
    // Get subscription and user stats in parallel to minimize DB calls
    const [subscription, statsRecords] = await Promise.all([
      getHighestPrioritySubscription(userId), // Calls to get the highest priority subscription.
      db.select().from(userStats).where(eq(userStats.userId, userId)).limit(1),
      // Queries for user stats, similar to `hasExceededCostLimit`, but uses `limit(1)` as only one record is expected/needed.
    ])
    // `Promise.all` executes both promises concurrently, waiting for both to resolve before continuing.
    // This is an optimization to reduce the total time spent waiting for database queries.

    // Determine plan types based on subscription (avoid redundant DB calls)
    // The `!isProd` condition acts as a bypass, making these true in development/staging.
    const isPro =
      !isProd ||
      (subscription &&
        (checkProPlan(subscription) ||
          checkTeamPlan(subscription) ||
          checkEnterprisePlan(subscription)))
    // `isPro` is true if not in production, OR if a subscription exists and it's Pro, Team, or Enterprise.

    const isTeam =
      !isProd ||
      (subscription && (checkTeamPlan(subscription) || checkEnterprisePlan(subscription)))
    // `isTeam` is true if not in production, OR if a subscription exists and it's Team or Enterprise.

    const isEnterprise = !isProd || (subscription && checkEnterprisePlan(subscription))
    // `isEnterprise` is true if not in production, OR if a subscription exists and it's Enterprise.

    const isFree = !isPro && !isTeam && !isEnterprise
    // `isFree` is true if none of the Pro, Team, or Enterprise checks passed.

    // Determine plan name
    let planName = 'free' // Default to 'free'.
    if (isEnterprise) planName = 'enterprise' // If Enterprise, set planName.
    else if (isTeam) planName = 'team' // Else if Team, set planName.
    else if (isPro) planName = 'pro' // Else if Pro, set planName.
    // This order ensures the highest priority plan name is assigned.

    // Check cost limit using already-fetched user stats
    let hasExceededLimit = false // Default to false.
    if (isProd && statsRecords.length > 0) {
      // Only check limits in production AND if user stats records exist.
      let limit = getFreeTierLimit() // Initialize limit with free tier.
      if (subscription) {
        // Same logic as `hasExceededCostLimit` to determine the specific limit.
        if (subscription.plan === 'team' || subscription.plan === 'enterprise') {
          const { getUserUsageLimit } = await import('@/lib/billing/core/usage')
          limit = await getUserUsageLimit(userId)
        } else {
          limit = getPerUserMinimumLimit(subscription)
        }
      }

      const currentCost = Number.parseFloat(
        statsRecords[0].currentPeriodCost?.toString() || statsRecords[0].totalCost.toString()
      )
      // Extracts current cost using the *already fetched* `statsRecords`.
      hasExceededLimit = currentCost >= limit // Determine if limit is exceeded.
    }

    return {
      isPro, // Boolean indicating Pro-level plan.
      isTeam, // Boolean indicating Team-level plan.
      isEnterprise, // Boolean indicating Enterprise plan.
      isFree, // Boolean indicating Free plan.
      highestPrioritySubscription: subscription, // The actual subscription object (or null).
      hasExceededLimit, // Boolean indicating if cost limit is exceeded.
      planName, // Human-readable plan name.
    }
  } catch (error) {
    logger.error('Error getting user subscription state', { error, userId })
    // Catches errors.

    // Return safe defaults in case of error
    return {
      isPro: false,
      isTeam: false,
      isEnterprise: false,
      isFree: true, // Default to free if there's an error.
      highestPrioritySubscription: null,
      hasExceededLimit: false,
      planName: 'free', // Default plan name.
    }
  }
}
```

---

### `sendPlanWelcomeEmail(subscription: any): Promise<void>`

This function is responsible for sending a welcome email to users who subscribe to Pro or Team plans.

#### Simplified Logic

1.  **Check plan type:** Only proceed if the `subscription.plan` is 'pro' or 'team'.
2.  **Fetch user details:** Get the user's email and name using the `referenceId` from the subscription.
3.  **Dynamic imports:** Dynamically import email rendering and sending utilities.
4.  **Render email:** Generate the HTML content of the welcome email, customizing it with the plan name, user name, and a login link.
5.  **Send email:** Use the mailer utility to send the email to the user.
6.  **Log success/error:** Log the outcome of the email sending attempt.
7.  **Error Handling:** Log errors and re-throw them for upstream handling.

#### Line-by-Line Explanation

```typescript
export async function sendPlanWelcomeEmail(subscription: any): Promise<void> {
  // `subscription: any` means the input subscription object can be of any type,
  // indicating it might not have a strict type definition available here, or for flexibility.
  try {
    const subPlan = subscription.plan // Extract the plan name from the subscription object.
    if (subPlan === 'pro' || subPlan === 'team') {
      // Only send emails for 'pro' or 'team' plans.
      const userId = subscription.referenceId // The `referenceId` in this context is the user ID.
      const users = await db
        .select({ email: user.email, name: user.name }) // Selects the `email` and `name` from the `user` table.
        .from(user)
        .where(eq(user.id, userId)) // Filters for the specific user by ID.
        .limit(1) // Limits the result to 1, as only one user is expected for a given ID.

      if (users.length > 0 && users[0].email) {
        // If a user record is found AND they have an email address:

        // Dynamically import email-related utilities.
        // This is often done to delay loading large modules until they are actually needed,
        // which can improve initial application load time.
        const { getEmailSubject, renderPlanWelcomeEmail } = await import(
          '@/components/emails/render-email'
        )
        const { sendEmail } = await import('@/lib/email/mailer')

        const baseUrl = getBaseUrl() // Get the application's base URL.
        const html = await renderPlanWelcomeEmail({
          planName: subPlan === 'pro' ? 'Pro' : 'Team', // Capitalize plan name for display in email.
          userName: users[0].name || undefined, // Use user's name if available, otherwise `undefined`.
          loginLink: `${baseUrl}/login`, // Construct the login link.
        })
        // `renderPlanWelcomeEmail` is an external function that takes template data and produces HTML for the email.

        await sendEmail({
          to: users[0].email, // Recipient's email address.
          subject: getEmailSubject(subPlan === 'pro' ? 'plan-welcome-pro' : 'plan-welcome-team'),
          // Get the appropriate subject line based on the plan type using a utility function.
          html, // The rendered HTML content of the email.
          emailType: 'updates', // Categorize the email type.
        })
        // `sendEmail` is an external function that sends the email using a configured mailer service.

        logger.info('Plan welcome email sent successfully', {
          userId,
          email: users[0].email,
          plan: subPlan,
        })
        // Log success.
      }
    }
  } catch (error) {
    logger.error('Failed to send plan welcome email', {
      error,
      subscriptionId: subscription.id,
      plan: subscription.plan,
    })
    // Log any errors that occur during the email sending process.
    throw error // Re-throw the error to ensure upstream processes are aware of the failure.
  }
}
```