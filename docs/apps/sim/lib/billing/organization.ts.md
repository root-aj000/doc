This TypeScript file is a core component of a billing and organization management system. It provides functionalities related to creating and managing organizations, particularly when users sign up for team or enterprise plans, and syncing usage limits based on their subscription status.

---

### **Purpose of this File**

This file serves two primary purposes:

1.  **Organization Management for Team Plans**: It handles the logic for creating and associating organizations with users, especially when a user upgrades to a team or enterprise-level subscription. It ensures that a user only owns one organization for such plans, or creates a new one if necessary.
2.  **Subscription Usage Limit Synchronization**: It orchestrates the synchronization of usage limits for users based on their active subscriptions. It intelligently distinguishes between individual user subscriptions and organization-wide subscriptions, applying limits to either a single user or all members of an organization, respectively.

In essence, this file links billing events (like subscription upgrades) with core application concepts (organizations and user usage limits).

---

### **Detailed Explanation**

Let's break down the code section by section.

#### **Imports**

```typescript
import { db } from '@sim/db'
import * as schema from '@sim/db/schema'
import { and, eq } from 'drizzle-orm'
import { syncUsageLimitsFromSubscription } from '@/lib/billing/core/usage'
import { createLogger } from '@/lib/logs/console/logger'
```

*   `import { db } from '@sim/db'`: Imports the database client instance, likely configured using Drizzle ORM. This `db` object is the primary interface for interacting with the database.
*   `import * as schema from '@sim/db/schema'`: Imports all database schema definitions (tables, relations, etc.) as a single `schema` object. This allows us to reference tables like `schema.member` or `schema.organization` in our queries.
*   `import { and, eq } from 'drizzle-orm'`: Imports helper functions from Drizzle ORM.
    *   `and`: Used to combine multiple conditions in a `WHERE` clause with a logical "AND".
    *   `eq`: Used to check for equality (`=`) in a `WHERE` clause.
*   `import { syncUsageLimitsFromSubscription } from '@/lib/billing/core/usage'`: Imports a utility function from another part of the codebase. This function is responsible for the actual logic of applying usage limits to a specific user based on their subscription details.
*   `import { createLogger } from '@/lib/logs/console/logger'`: Imports a function to create a logger instance, used for structured logging of events and errors within this file.

#### **Logger Initialization**

```typescript
const logger = createLogger('BillingOrganization')
```

*   `const logger = createLogger('BillingOrganization')`: Initializes a logger instance with the name `BillingOrganization`. This helps in tracing logs back to their source file or module, making debugging easier.

#### **`SubscriptionData` Type Definition**

```typescript
type SubscriptionData = {
  id: string
  plan: string
  referenceId: string
  status: string
  seats?: number
}
```

*   `type SubscriptionData = { ... }`: Defines a TypeScript type alias named `SubscriptionData`. This type specifies the expected structure of an object representing subscription information.
    *   `id: string`: The unique identifier of the subscription itself.
    *   `plan: string`: The name or ID of the subscription plan (e.g., "basic", "premium", "team").
    *   `referenceId: string`: A crucial field that links the subscription to either a `userId` (for individual plans) or an `organizationId` (for team/enterprise plans).
    *   `status: string`: The current status of the subscription (e.g., "active", "cancelled", "trialing").
    *   `seats?: number`: An optional field indicating the number of seats included in the subscription, typically for team plans. The `?` makes it optional.

#### **`getUserOwnedOrganization` Function**

This asynchronous function checks if a given user already owns an organization.

```typescript
/**
 * Check if a user already owns an organization
 */
async function getUserOwnedOrganization(userId: string): Promise<string | null> {
  // 1. Find existing memberships where the user is an 'owner'
  const existingMemberships = await db
    .select({ organizationId: schema.member.organizationId })
    .from(schema.member)
    .where(and(eq(schema.member.userId, userId), eq(schema.member.role, 'owner')))
    .limit(1)

  // 2. If an ownership record is found, retrieve the organization ID
  if (existingMemberships.length > 0) {
    const [existingOrg] = await db
      .select({ id: schema.organization.id })
      .from(schema.organization)
      .where(eq(schema.organization.id, existingMemberships[0].organizationId))
      .limit(1)

    return existingOrg?.id || null
  }

  // 3. If no ownership record is found, return null
  return null
}
```

*   `async function getUserOwnedOrganization(userId: string): Promise<string | null>`: Declares an asynchronous function that takes a `userId` (string) and returns a `Promise` that resolves to either the `organizationId` (string) if found, or `null` otherwise.
*   `const existingMemberships = await db ... .limit(1)`:
    *   `db.select({ organizationId: schema.member.organizationId })`: Selects only the `organizationId` column from the `member` table.
    *   `.from(schema.member)`: Specifies that the query is targeting the `member` table.
    *   `.where(and(eq(schema.member.userId, userId), eq(schema.member.role, 'owner')))`: Filters the members where both `userId` matches the input `userId` AND the `role` is `'owner'`. This identifies if the user is an owner of any organization.
    *   `.limit(1)`: Limits the result set to just one record, as we only need to know *if* they own an organization, not how many.
*   `if (existingMemberships.length > 0)`: Checks if any ownership records were found.
*   `const [existingOrg] = await db ... .limit(1)`:
    *   If an ownership record exists, this query fetches the actual `id` of that organization from the `organization` table.
    *   `db.select({ id: schema.organization.id })`: Selects the `id` column from the `organization` table.
    *   `.from(schema.organization)`: Specifies the `organization` table.
    *   `.where(eq(schema.organization.id, existingMemberships[0].organizationId))`: Filters for the organization whose `id` matches the `organizationId` found in the `existingMemberships` result.
    *   `.limit(1)`: Again, limits to one result.
*   `return existingOrg?.id || null`: Returns the `id` of the found organization. If `existingOrg` is `undefined` or `null` (which shouldn't happen here if `existingMemberships` had a record, but acts as a safeguard), it defaults to `null`.
*   `return null`: If no existing memberships with an 'owner' role were found in the first query, the function returns `null`.

#### **`createOrganizationWithOwner` Function**

This asynchronous function creates a new organization and automatically adds the specified user as an owner.

```typescript
/**
 * Create a new organization and add user as owner
 */
async function createOrganizationWithOwner(
  userId: string,
  organizationName: string,
  organizationSlug: string,
  metadata: Record<string, any> = {}
): Promise<string> {
  // 1. Generate a unique ID for the new organization
  const orgId = `org_${crypto.randomUUID()}`

  // 2. Insert the new organization into the database
  const [newOrg] = await db
    .insert(schema.organization)
    .values({
      id: orgId,
      name: organizationName,
      slug: organizationSlug,
      metadata,
    })
    .returning({ id: schema.organization.id })

  // 3. Add the user as an owner/admin of the newly created organization
  await db.insert(schema.member).values({
    id: crypto.randomUUID(), // Generate a unique ID for the membership record
    userId: userId,
    organizationId: newOrg.id,
    role: 'owner',
  })

  // 4. Log the successful creation
  logger.info('Created organization with owner', {
    userId,
    organizationId: newOrg.id,
    organizationName,
  })

  // 5. Return the ID of the new organization
  return newOrg.id
}
```

*   `async function createOrganizationWithOwner(...)`: Declares an asynchronous function that takes `userId`, `organizationName`, `organizationSlug`, and an optional `metadata` object. It returns a `Promise` that resolves to the new organization's `id` (string).
*   `const orgId = \`org_\${crypto.randomUUID()}\``: Generates a unique ID for the organization, prefixed with `org_`. `crypto.randomUUID()` is a standard Web Crypto API function for generating universally unique identifiers.
*   `const [newOrg] = await db ... .returning({ id: schema.organization.id })`:
    *   `db.insert(schema.organization)`: Starts an insert query on the `organization` table.
    *   `.values({ id: orgId, name: organizationName, slug: organizationSlug, metadata, })`: Specifies the values for the new organization record.
    *   `.returning({ id: schema.organization.id })`: After insertion, it returns the `id` of the newly created organization. The `[newOrg]` destructuring assigns the first (and only) returned record to `newOrg`.
*   `await db.insert(schema.member).values({ ... })`:
    *   Inserts a new record into the `member` table.
    *   `id: crypto.randomUUID()`: Generates a unique ID for this membership record.
    *   `userId: userId`: Associates the membership with the provided `userId`.
    *   `organizationId: newOrg.id`: Associates the membership with the newly created organization.
    *   `role: 'owner'`: Assigns the `owner` role to this user for the new organization.
*   `logger.info('Created organization with owner', { ... })`: Logs an informational message indicating the successful creation of the organization and its owner, including relevant IDs and names.
*   `return newOrg.id`: Returns the ID of the newly created organization.

#### **`createOrganizationForTeamPlan` Function**

This is an **exported** function that orchestrates the creation of an organization specifically for users upgrading to a team or enterprise plan. It first checks if the user already owns an organization.

```typescript
/**
 * Create organization for team/enterprise plan upgrade
 */
export async function createOrganizationForTeamPlan(
  userId: string,
  userName?: string,
  userEmail?: string,
  organizationSlug?: string
): Promise<string> {
  try {
    // 1. Check if user already owns an organization
    const existingOrgId = await getUserOwnedOrganization(userId)
    if (existingOrgId) {
      return existingOrgId // If so, return the existing organization's ID
    }

    // 2. If no existing organization, determine default name and slug
    const organizationName = userName || `${userEmail || 'User'}'s Team`
    const slug = organizationSlug || `${userId}-team-${Date.now()}`

    // 3. Create the new organization with the user as owner
    const orgId = await createOrganizationWithOwner(userId, organizationName, slug, {
      createdForTeamPlan: true,
      originalUserId: userId,
    })

    // 4. Log the successful creation
    logger.info('Created organization for team/enterprise plan', {
      userId,
      organizationId: orgId,
      organizationName,
    })

    // 5. Return the ID of the new organization
    return orgId
  } catch (error) {
    // 6. Handle and log any errors during the process
    logger.error('Failed to create organization for team/enterprise plan', {
      userId,
      error,
    })
    throw error // Re-throw the error to propagate it up the call stack
  }
}
```

*   `export async function createOrganizationForTeamPlan(...)`: Declares an asynchronous, **exported** function, making it accessible from other files. It takes `userId` and optional `userName`, `userEmail`, `organizationSlug`. It returns a `Promise` resolving to the organization's `id`.
*   `try { ... } catch (error) { ... }`: A `try...catch` block is used to gracefully handle any errors that occur during the organization creation process.
*   `const existingOrgId = await getUserOwnedOrganization(userId)`: Calls the previously defined `getUserOwnedOrganization` function to check if the user already owns an organization.
*   `if (existingOrgId) { return existingOrgId }`: If an existing organization is found, its ID is immediately returned, preventing the creation of a duplicate organization.
*   `const organizationName = userName || \`${userEmail || 'User'}'s Team\``: Constructs the organization name. It prioritizes `userName` if provided, otherwise uses `userEmail` (e.g., "john@example.com's Team"), or falls back to "User's Team" if neither is available.
*   `const slug = organizationSlug || \`${userId}-team-\${Date.now()}\``: Constructs the organization slug (a URL-friendly identifier). It prioritizes `organizationSlug` if provided, otherwise creates a default slug using the `userId`, "team", and a timestamp (`Date.now()`) to ensure uniqueness.
*   `const orgId = await createOrganizationWithOwner(...)`: Calls the `createOrganizationWithOwner` function with the determined details to create the new organization.
    *   `metadata: { createdForTeamPlan: true, originalUserId: userId }`: Adds specific metadata to the organization record, indicating that it was created for a team plan and storing the original user's ID.
*   `logger.info('Created organization for team/enterprise plan', { ... })`: Logs the successful creation specifically for a team/enterprise plan.
*   `return orgId`: Returns the ID of the newly created organization.
*   `catch (error) { ... }`: If an error occurs:
    *   `logger.error('Failed to create organization for team/enterprise plan', { ... })`: Logs an error message with relevant context.
    *   `throw error`: Re-throws the error so that the calling function can handle it further.

#### **`syncSubscriptionUsageLimits` Function**

This is an **exported** function that synchronizes usage limits for users based on their subscription data. It differentiates between individual and organization-wide subscriptions.

```typescript
/**
 * Sync usage limits for subscription members
 * Updates usage limits for all users associated with the subscription
 */
export async function syncSubscriptionUsageLimits(subscription: SubscriptionData) {
  try {
    // 1. Log the initiation of the sync process
    logger.info('Syncing subscription usage limits', {
      subscriptionId: subscription.id,
      referenceId: subscription.referenceId,
      plan: subscription.plan,
    })

    // 2. Determine if the subscription's referenceId points to a user or an organization
    const users = await db
      .select({ id: schema.user.id })
      .from(schema.user)
      .where(eq(schema.user.id, subscription.referenceId))
      .limit(1)

    if (users.length > 0) {
      // Case A: Individual user subscription
      await syncUsageLimitsFromSubscription(subscription.referenceId) // Sync limits for that specific user

      logger.info('Synced usage limits for individual user subscription', {
        userId: subscription.referenceId,
        subscriptionId: subscription.id,
        plan: subscription.plan,
      })
    } else {
      // Case B: Organization subscription
      const members = await db
        .select({ userId: schema.member.userId })
        .from(schema.member)
        .where(eq(schema.member.organizationId, subscription.referenceId))

      if (members.length > 0) {
        // Iterate through all members of the organization and sync their limits
        for (const member of members) {
          try {
            await syncUsageLimitsFromSubscription(member.userId)
          } catch (memberError) {
            // Log errors for individual members, but continue with others
            logger.error('Failed to sync usage limits for organization member', {
              userId: member.userId,
              organizationId: subscription.referenceId,
              subscriptionId: subscription.id,
              error: memberError,
            })
          }
        }

        logger.info('Synced usage limits for organization members', {
          organizationId: subscription.referenceId,
          memberCount: members.length,
          subscriptionId: subscription.id,
          plan: subscription.plan,
        })
      }
    }
  } catch (error) {
    // 3. Handle and log any general errors during the sync process
    logger.error('Failed to sync subscription usage limits', {
      subscriptionId: subscription.id,
      referenceId: subscription.referenceId,
      error,
    })
    throw error // Re-throw the error
  }
}
```

*   `export async function syncSubscriptionUsageLimits(subscription: SubscriptionData)`: Declares an asynchronous, **exported** function that takes a `subscription` object (of type `SubscriptionData`) and returns a `Promise<void>` (it doesn't return a specific value).
*   `try { ... } catch (error) { ... }`: A `try...catch` block to handle potential errors during the entire synchronization process.
*   `logger.info('Syncing subscription usage limits', { ... })`: Logs the start of the synchronization, including details about the subscription.
*   `const users = await db ... .limit(1)`:
    *   This query attempts to find a user whose `id` matches `subscription.referenceId`.
    *   This is the key step to determine if `referenceId` refers to a `user` or an `organization`. If a user is found, it's an individual subscription.
*   `if (users.length > 0)`:
    *   **Individual User Subscription**: If a user matching `subscription.referenceId` is found:
        *   `await syncUsageLimitsFromSubscription(subscription.referenceId)`: Calls the imported `syncUsageLimitsFromSubscription` function to apply limits directly to this `userId`.
        *   `logger.info('Synced usage limits for individual user subscription', { ... })`: Logs success for the individual user.
*   `else { ... }`:
    *   **Organization Subscription**: If `subscription.referenceId` doesn't match any `userId`, it's assumed to be an `organizationId`.
    *   `const members = await db ...`:
        *   `db.select({ userId: schema.member.userId })`: Selects the `userId` of all members.
        *   `.from(schema.member)`: From the `member` table.
        *   `.where(eq(schema.member.organizationId, subscription.referenceId))`: Where `organizationId` matches the `subscription.referenceId`. This fetches all users belonging to that organization.
    *   `if (members.length > 0)`: If members are found for the organization:
        *   `for (const member of members) { ... }`: Iterates through each member of the organization.
        *   `try { await syncUsageLimitsFromSubscription(member.userId) } catch (memberError) { ... }`: For each member, it attempts to sync their usage limits.
            *   Crucially, this `try...catch` block is *inside* the loop. This means if syncing fails for one member, it logs the error but continues attempting to sync for other members, preventing a single failure from stopping the entire process for the organization.
            *   `logger.error('Failed to sync usage limits for organization member', { ... })`: Logs an error if a member's usage limit sync fails.
        *   `logger.info('Synced usage limits for organization members', { ... })`: Logs success after attempting to sync limits for all organization members.
*   `catch (error) { ... }`: If any error occurs in the outer `try` block (e.g., database connection issues, or issues with the initial `syncUsageLimitsFromSubscription` call for an individual user), it logs the general failure and re-throws the error.

---

This file demonstrates robust practices in a TypeScript application, including:
*   Clear function responsibilities.
*   Effective use of Drizzle ORM for database interactions.
*   Structured logging for observability.
*   Comprehensive error handling to ensure application stability.
*   Type safety with TypeScript interfaces.
*   Idempotent logic (checking for existing organizations before creating new ones).