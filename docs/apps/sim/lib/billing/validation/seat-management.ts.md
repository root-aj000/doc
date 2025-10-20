This file, `seat-management.ts`, is a core part of a system that manages "seats" within an organization. In many software services, a "seat" refers to a license or slot that a user occupies, often tied to a subscription plan. This file provides a comprehensive set of functions to:

*   **Validate Seat Availability:** Check if an organization has enough available seats to invite new members.
*   **Retrieve Seat Information:** Get detailed data about an organization's current seat usage, maximum allowed seats, and subscription plan.
*   **Validate Bulk Invitations:** Prepare a list of emails for bulk invitations, filtering out duplicates, existing members, and pending invites, and then check seat availability for the remaining valid new invites.
*   **Update Organization Seats:** Allow modification of the total number of seats an organization is allocated, with checks to prevent reducing seats below the current member count.
*   **Validate Member Removal:** Determine if a specific user can be removed from an organization, enforcing rules about roles (e.g., owners cannot be removed by admins, owners cannot remove themselves).
*   **Get Seat Usage Analytics:** Provide detailed analytics on seat utilization, including active/inactive members and individual member activity.

Essentially, this file acts as the central hub for all business logic related to managing an organization's member capacity and ensuring that membership changes adhere to subscription limits and role-based permissions.

---

### **Imports**

Let's start by looking at the `import` statements at the top of the file. These lines bring in necessary tools and data structures from other parts of the application or external libraries.

```typescript
import { db } from '@sim/db'
import { invitation, member, organization, subscription, user, userStats } from '@sim/db/schema'
import { and, count, eq } from 'drizzle-orm'
import { getOrganizationSubscription } from '@/lib/billing/core/billing'
import { quickValidateEmail } from '@/lib/email/validation'
import { createLogger } from '@/lib/logs/console/logger'
```

*   `import { db } from '@sim/db'`: This imports the Drizzle ORM database client instance, named `db`, from the application's database module. This `db` object is what we'll use to execute all database queries.
*   `import { invitation, member, organization, subscription, user, userStats } from '@sim/db/schema'`: These imports bring in the Drizzle ORM "schema" definitions for various tables in the database. Each imported name (e.g., `invitation`, `member`) represents a table, allowing us to interact with its columns and relationships in a type-safe manner.
    *   `invitation`: Represents pending invitations to join an organization.
    *   `member`: Represents a user's membership within an organization.
    *   `organization`: Represents an organization entity.
    *   `subscription`: Represents an organization's subscription details.
    *   `user`: Represents a user account.
    *   `userStats`: Represents statistics related to a user, like last active time.
*   `import { and, count, eq } from 'drizzle-orm'`: These are utility functions from the Drizzle ORM library itself, used to construct database queries:
    *   `and`: Combines multiple conditions with a logical "AND".
    *   `count`: An aggregate function to count rows.
    *   `eq`: Checks if a column is "equal to" a specific value.
*   `import { getOrganizationSubscription } from '@/lib/billing/core/billing'`: Imports a function from the billing module. This function is likely responsible for fetching an organization's active subscription details, possibly interacting with an external billing provider like Stripe.
*   `import { quickValidateEmail } from '@/lib/email/validation'`: Imports a utility function for quickly validating the format of an email address.
*   `import { createLogger } from '@/lib/logs/console/logger'`: Imports a function to create a logger instance, used for logging messages (e.g., debug, info, error) to the console or other logging systems.

---

### **Logger Initialization**

```typescript
const logger = createLogger('SeatManagement')
```

*   `const logger = createLogger('SeatManagement')`: This line creates a specific logger instance for this file, naming it `SeatManagement`. This helps in tracing logs back to their source, making debugging easier. All `logger.debug`, `logger.error`, etc., calls in this file will use this instance.

---

### **Interfaces**

Interfaces in TypeScript define the shapes of objects. They are crucial for type safety and code readability, making it clear what data to expect.

```typescript
interface SeatValidationResult {
  canInvite: boolean
  reason?: string
  currentSeats: number
  maxSeats: number
  availableSeats: number
}
```

*   `interface SeatValidationResult`: This interface defines the structure of the object returned by functions that validate seat availability.
    *   `canInvite: boolean`: A boolean indicating whether new members can be invited. `true` if seats are available, `false` otherwise.
    *   `reason?: string`: An optional string providing a human-readable explanation if `canInvite` is `false` (e.g., "No active subscription," "Not enough seats").
    *   `currentSeats: number`: The total number of seats currently occupied by members in the organization.
    *   `maxSeats: number`: The maximum number of seats allowed by the organization's subscription plan.
    *   `availableSeats: number`: The number of seats remaining that can be filled.

```typescript
interface OrganizationSeatInfo {
  organizationId: string
  organizationName: string
  currentSeats: number
  maxSeats: number
  availableSeats: number
  subscriptionPlan: string
  canAddSeats: boolean
}
```

*   `interface OrganizationSeatInfo`: This interface defines the structure of the object returned by functions that provide comprehensive seat information for an organization.
    *   `organizationId: string`: The unique identifier of the organization.
    *   `organizationName: string`: The name of the organization.
    *   `currentSeats: number`: The count of members currently occupying seats.
    *   `maxSeats: number`: The maximum allowed seats based on the subscription.
    *   `availableSeats: number`: The number of remaining seats.
    *   `subscriptionPlan: string`: The name of the organization's current subscription plan (e.g., 'team', 'enterprise').
    *   `canAddSeats: boolean`: A boolean indicating if the organization's plan allows for adding more seats (e.g., 'enterprise' plans might have fixed seats, while 'team' plans might be adjustable).

---

### **`validateSeatAvailability` Function**

This function checks if an organization has enough available seats based on its subscription plan to invite a specified number of additional members.

```typescript
/**
 * Validate if an organization can invite new members based on seat limits
 */
export async function validateSeatAvailability(
  organizationId: string,
  additionalSeats = 1
): Promise<SeatValidationResult> {
  try {
    // Get organization subscription directly (referenceId = organizationId)
    const subscription = await getOrganizationSubscription(organizationId)

    if (!subscription) {
      return {
        canInvite: false,
        reason: 'No active subscription found',
        currentSeats: 0,
        maxSeats: 0,
        availableSeats: 0,
      }
    }

    // Free and Pro plans don't support organizations
    if (['free', 'pro'].includes(subscription.plan)) {
      return {
        canInvite: false,
        reason: 'Organization features require Team or Enterprise plan',
        currentSeats: 0,
        maxSeats: 0,
        availableSeats: 0,
      }
    }

    // Get current member count
    const memberCount = await db
      .select({ count: count() })
      .from(member)
      .where(eq(member.organizationId, organizationId))

    const currentSeats = memberCount[0]?.count || 0

    // Determine seat limits based on subscription
    // Team: seats from Stripe subscription quantity
    // Enterprise: seats from metadata (stored in subscription.seats)
    const maxSeats = subscription.seats || 1

    const availableSeats = Math.max(0, maxSeats - currentSeats)
    const canInvite = availableSeats >= additionalSeats

    const result: SeatValidationResult = {
      canInvite,
      currentSeats,
      maxSeats,
      availableSeats,
    }

    if (!canInvite) {
      if (additionalSeats === 1) {
        result.reason = `No available seats. Currently using ${currentSeats} of ${maxSeats} seats.`
      } else {
        result.reason = `Not enough available seats. Need ${additionalSeats} seats, but only ${availableSeats} available.`
      }
    }

    logger.debug('Seat validation result', {
      organizationId,
      additionalSeats,
      result,
    })

    return result
  } catch (error) {
    logger.error('Failed to validate seat availability', { organizationId, additionalSeats, error })
    return {
      canInvite: false,
      reason: 'Failed to check seat availability',
      currentSeats: 0,
      maxSeats: 0,
      availableSeats: 0,
    }
  }
}
```

**Purpose:** To determine if an organization has enough seats available for a certain number of new members, considering its subscription plan.

**Simplified Logic:**
1.  Fetch the organization's subscription details. If none, invitations are blocked.
2.  Check if the subscription plan supports organization features (e.g., 'free' and 'pro' plans might not). If not, invitations are blocked.
3.  Count the number of members already in the organization.
4.  Determine the maximum allowed seats from the subscription.
5.  Calculate available seats (`maxSeats - currentMembers`).
6.  Compare available seats to the `additionalSeats` requested. If enough, `canInvite` is `true`.
7.  Return a `SeatValidationResult` object with all these details, including a reason if invitations are blocked.

**Line-by-Line Explanation:**

*   `export async function validateSeatAvailability(...)`: Defines an asynchronous function that can be imported and used elsewhere. It takes an `organizationId` (string) and an optional `additionalSeats` (number, defaulting to 1) and returns a `Promise` that resolves to a `SeatValidationResult` object.
*   `try { ... } catch (error) { ... }`: A `try-catch` block is used to handle any potential errors during the execution of the function. If an error occurs, it's caught, logged, and a default `SeatValidationResult` indicating `canInvite: false` is returned.
*   `const subscription = await getOrganizationSubscription(organizationId)`: Calls another asynchronous function to fetch the organization's subscription details using its ID. The `await` keyword pauses execution until the `Promise` returned by `getOrganizationSubscription` is resolved.
*   `if (!subscription) { ... }`: Checks if a subscription record was found.
*   `return { canInvite: false, reason: 'No active subscription found', ... }`: If no subscription, returns a result indicating that inviting is not possible, with a specific reason. `currentSeats`, `maxSeats`, and `availableSeats` are set to `0` as they cannot be determined without a subscription.
*   `if (['free', 'pro'].includes(subscription.plan)) { ... }`: Checks if the organization's subscription plan is 'free' or 'pro'. These plans are explicitly stated to not support organization features.
*   `return { canInvite: false, reason: 'Organization features require Team or Enterprise plan', ... }`: If the plan is 'free' or 'pro', returns a result indicating `canInvite: false` with the corresponding reason.
*   `const memberCount = await db.select({ count: count() }).from(member).where(eq(member.organizationId, organizationId))`: This is a Drizzle ORM query:
    *   `db.select({ count: count() })`: Selects an aggregated count of rows. The `count()` function from Drizzle is used here. The result is aliased as `count`.
    *   `.from(member)`: Specifies that we are querying the `member` table.
    *   `.where(eq(member.organizationId, organizationId))`: Filters the `member` table to only include members belonging to the specified `organizationId`. `eq` ensures `member.organizationId` is equal to the provided `organizationId`.
*   `const currentSeats = memberCount[0]?.count || 0`: Extracts the actual count from the query result. Drizzle queries often return an array, so `memberCount[0]` accesses the first (and in this case, only) result. The `?.` (optional chaining) handles cases where `memberCount[0]` might be undefined (e.g., no members, though `count()` usually returns `0` in such cases). `|| 0` provides a fallback of `0` if the count is falsy.
*   `const maxSeats = subscription.seats || 1`: Determines the maximum number of seats. It takes the `seats` property from the `subscription` object. If `subscription.seats` is `null` or `undefined` (or `0`), it defaults to `1` to ensure a minimum seat count. This comment clarifies how `maxSeats` are determined for different plans.
*   `const availableSeats = Math.max(0, maxSeats - currentSeats)`: Calculates the number of seats still available. `Math.max(0, ...)` ensures that `availableSeats` never goes below `0`, even if `currentSeats` somehow exceeds `maxSeats`.
*   `const canInvite = availableSeats >= additionalSeats`: A boolean indicating if there are enough `availableSeats` to accommodate the `additionalSeats` requested.
*   `const result: SeatValidationResult = { ... }`: Creates the `SeatValidationResult` object with the calculated values.
*   `if (!canInvite) { ... }`: If `canInvite` is `false`, a specific reason is added to the `result` object.
*   `if (additionalSeats === 1) { ... } else { ... }`: Provides a tailored reason message depending on whether a single seat or multiple seats were requested.
*   `logger.debug('Seat validation result', { ... })`: Logs a debug message with relevant context (organization ID, requested additional seats, and the full validation result). This is helpful for monitoring and debugging.
*   `return result`: Returns the final `SeatValidationResult` object.
*   `catch (error) { ... }`: If an error occurs during the `try` block, this `catch` block executes.
*   `logger.error('Failed to validate seat availability', { ... })`: Logs the error with context for debugging.
*   `return { canInvite: false, reason: 'Failed to check seat availability', ... }`: Returns a generic failure `SeatValidationResult`.

---

### **`getOrganizationSeatInfo` Function**

This function provides a comprehensive overview of an organization's seat usage, max limits, and subscription details.

```typescript
/**
 * Get comprehensive seat information for an organization
 */
export async function getOrganizationSeatInfo(
  organizationId: string
): Promise<OrganizationSeatInfo | null> {
  try {
    const organizationData = await db
      .select({
        id: organization.id,
        name: organization.name,
      })
      .from(organization)
      .where(eq(organization.id, organizationId))
      .limit(1)

    if (organizationData.length === 0) {
      return null
    }

    const subscription = await getOrganizationSubscription(organizationId)

    if (!subscription) {
      return null
    }

    const memberCount = await db
      .select({ count: count() })
      .from(member)
      .where(eq(member.organizationId, organizationId))

    const currentSeats = memberCount[0]?.count || 0

    const maxSeats = subscription.seats || 1

    const canAddSeats = subscription.plan !== 'enterprise'

    const availableSeats = Math.max(0, maxSeats - currentSeats)

    return {
      organizationId,
      organizationName: organizationData[0].name,
      currentSeats,
      maxSeats,
      availableSeats,
      subscriptionPlan: subscription.plan,
      canAddSeats,
    }
  } catch (error) {
    logger.error('Failed to get organization seat info', { organizationId, error })
    return null
  }
}
```

**Purpose:** To retrieve all relevant information about an organization's seat management status, including current usage, limits, and subscription plan.

**Simplified Logic:**
1.  Fetch organization's basic data (like name). If not found, return `null`.
2.  Fetch the organization's subscription. If none, return `null`.
3.  Count current members.
4.  Determine `maxSeats` from the subscription.
5.  Calculate `availableSeats`.
6.  Determine if more seats can be added based on the subscription plan (e.g., 'enterprise' plans might be fixed).
7.  Combine all this information into an `OrganizationSeatInfo` object.

**Line-by-Line Explanation:**

*   `export async function getOrganizationSeatInfo(...)`: Defines an asynchronous function that takes an `organizationId` and returns a `Promise` that resolves to an `OrganizationSeatInfo` object or `null` if the organization or its subscription isn't found.
*   `try { ... } catch (error) { ... }`: Standard error handling.
*   `const organizationData = await db.select(...).from(organization).where(...).limit(1)`: Drizzle query to fetch the `id` and `name` of the organization.
    *   `.select({ id: organization.id, name: organization.name })`: Selects the `id` and `name` columns from the `organization` table.
    *   `.from(organization)`: Specifies the `organization` table.
    *   `.where(eq(organization.id, organizationId))`: Filters for the specific organization.
    *   `.limit(1)`: Ensures only one record is returned, as `organizationId` should be unique.
*   `if (organizationData.length === 0) { return null }`: If no organization is found with the given ID, returns `null`.
*   `const subscription = await getOrganizationSubscription(organizationId)`: Fetches the organization's subscription details, reusing the `getOrganizationSubscription` function.
*   `if (!subscription) { return null }`: If no subscription is found, returns `null`.
*   `const memberCount = await db.select({ count: count() }).from(member).where(eq(member.organizationId, organizationId))`: Same Drizzle query as in `validateSeatAvailability` to count current members.
*   `const currentSeats = memberCount[0]?.count || 0`: Extracts the member count.
*   `const maxSeats = subscription.seats || 1`: Determines the maximum seats, defaulting to 1 if not specified.
*   `const canAddSeats = subscription.plan !== 'enterprise'`: Sets a boolean indicating whether seats can be added. This assumes that 'enterprise' plans have fixed seat counts that cannot be adjusted directly, while other plans (like 'team') might allow it.
*   `const availableSeats = Math.max(0, maxSeats - currentSeats)`: Calculates available seats.
*   `return { ... }`: Returns the constructed `OrganizationSeatInfo` object.
*   `catch (error) { ... }`: Catches and logs errors.
*   `return null`: Returns `null` on error, as comprehensive info couldn't be retrieved.

---

### **`validateBulkInvitations` Function**

This function processes a list of emails for bulk invitations, cleaning the list and performing seat validation for the final set of new, valid, and uninvited emails.

```typescript
/**
 * Validate and reserve seats for bulk invitations
 */
export async function validateBulkInvitations(
  organizationId: string,
  emailList: string[]
): Promise<{
  canInviteAll: boolean
  validEmails: string[]
  duplicateEmails: string[]
  existingMembers: string[]
  seatsNeeded: number
  seatsAvailable: number
  validationResult: SeatValidationResult
}> {
  try {
    const uniqueEmails = [...new Set(emailList)]
    const validEmails = uniqueEmails.filter(
      (email) => quickValidateEmail(email.trim().toLowerCase()).isValid
    )
    const duplicateEmails = emailList.filter((email, index) => emailList.indexOf(email) !== index)

    const existingMembers = await db
      .select({ userEmail: user.email })
      .from(member)
      .innerJoin(user, eq(member.userId, user.id))
      .where(eq(member.organizationId, organizationId))

    const existingEmails = existingMembers.map((m) => m.userEmail)
    const newEmails = validEmails.filter((email) => !existingEmails.includes(email))

    const pendingInvitations = await db
      .select({ email: invitation.email })
      .from(invitation)
      .where(and(eq(invitation.organizationId, organizationId), eq(invitation.status, 'pending')))

    const pendingEmails = pendingInvitations.map((i) => i.email)
    const finalEmailsToInvite = newEmails.filter((email) => !pendingEmails.includes(email))

    const seatsNeeded = finalEmailsToInvite.length
    const validationResult = await validateSeatAvailability(organizationId, seatsNeeded)

    return {
      canInviteAll: validationResult.canInvite && finalEmailsToInvite.length > 0,
      validEmails: finalEmailsToInvite,
      duplicateEmails,
      existingMembers: validEmails.filter((email) => existingEmails.includes(email)),
      seatsNeeded,
      seatsAvailable: validationResult.availableSeats,
      validationResult,
    }
  } catch (error) {
    logger.error('Failed to validate bulk invitations', {
      organizationId,
      emailCount: emailList.length,
      error,
    })

    const validationResult: SeatValidationResult = {
      canInvite: false,
      reason: 'Validation failed',
      currentSeats: 0,
      maxSeats: 0,
      availableSeats: 0,
    }

    return {
      canInviteAll: false,
      validEmails: [],
      duplicateEmails: [],
      existingMembers: [],
      seatsNeeded: 0,
      seatsAvailable: 0,
      validationResult,
    }
  }
}
```

**Purpose:** To take a raw list of emails and determine which ones can actually be invited to an organization, checking for duplicates, existing members, pending invitations, and overall seat availability.

**Simplified Logic:**
1.  Clean the input `emailList`: remove duplicates and validate email formats. Identify any original duplicates.
2.  Query the database for emails of existing members in the organization.
3.  Filter out existing members from the valid email list to get truly `newEmails`.
4.  Query the database for emails of users who already have pending invitations.
5.  Filter out those with pending invitations from `newEmails` to get `finalEmailsToInvite`.
6.  Call `validateSeatAvailability` using the count of `finalEmailsToInvite` as `additionalSeats`.
7.  Return a comprehensive result object, including lists of `validEmails` (to invite), `duplicateEmails`, `existingMembers`, and the `SeatValidationResult`.

**Line-by-Line Explanation:**

*   `export async function validateBulkInvitations(...)`: Defines the asynchronous function. It takes an `organizationId` and an `emailList` (array of strings) and returns a `Promise` resolving to a complex object containing various lists and a `SeatValidationResult`.
*   `try { ... } catch (error) { ... }`: Standard error handling.
*   `const uniqueEmails = [...new Set(emailList)]`: Creates a new array `uniqueEmails` containing only the unique email addresses from the original `emailList`. `Set` objects only store unique values, and the spread operator (`...`) converts the Set back into an array.
*   `const validEmails = uniqueEmails.filter(...)`: Filters the `uniqueEmails` array.
    *   `email.trim().toLowerCase()`: Each email is trimmed (whitespace removed) and converted to lowercase for consistent validation.
    *   `quickValidateEmail(...).isValid`: Calls the imported email validation function. Only emails that return `isValid: true` are kept.
*   `const duplicateEmails = emailList.filter(...)`: Identifies emails that appeared more than once in the *original* `emailList`.
    *   `emailList.indexOf(email) !== index`: This condition is true if the first occurrence of `email` is not at the current `index`, meaning it's a duplicate.
*   `const existingMembers = await db.select({ userEmail: user.email }).from(member).innerJoin(user, eq(member.userId, user.id)).where(eq(member.organizationId, organizationId))`: This Drizzle query fetches the email addresses of all existing members in the organization.
    *   `.select({ userEmail: user.email })`: Selects the `email` column from the `user` table and aliases it as `userEmail`.
    *   `.from(member)`: Starts the query from the `member` table.
    *   `.innerJoin(user, eq(member.userId, user.id))`: Joins the `member` table with the `user` table where `member.userId` matches `user.id`. This allows us to get user details (like email) for members.
    *   `.where(eq(member.organizationId, organizationId))`: Filters members by the provided `organizationId`.
*   `const existingEmails = existingMembers.map((m) => m.userEmail)`: Extracts just the email strings from the `existingMembers` query result into a flat array.
*   `const newEmails = validEmails.filter((email) => !existingEmails.includes(email))`: Filters the `validEmails` list, keeping only those emails that are *not* already present in the `existingEmails` list. These are emails for potential new members.
*   `const pendingInvitations = await db.select({ email: invitation.email }).from(invitation).where(and(eq(invitation.organizationId, organizationId), eq(invitation.status, 'pending')))`: Drizzle query to find email addresses of users who already have a pending invitation for this organization.
    *   `.select({ email: invitation.email })`: Selects the `email` column from the `invitation` table.
    *   `.from(invitation)`: Specifies the `invitation` table.
    *   `.where(and(eq(invitation.organizationId, organizationId), eq(invitation.status, 'pending')))`: Filters invitations by `organizationId` AND `status` being 'pending'. The `and` function from Drizzle combines the two conditions.
*   `const pendingEmails = pendingInvitations.map((i) => i.email)`: Extracts the email strings from the `pendingInvitations` query result.
*   `const finalEmailsToInvite = newEmails.filter((email) => !pendingEmails.includes(email))`: Filters `newEmails`, keeping only those that *do not* already have a pending invitation. This is the final, clean list of emails that can actually be invited.
*   `const seatsNeeded = finalEmailsToInvite.length`: The number of seats required for these final emails.
*   `const validationResult = await validateSeatAvailability(organizationId, seatsNeeded)`: Calls the `validateSeatAvailability` function (defined earlier in this file) to check if there are enough seats for all `finalEmailsToInvite`.
*   `return { ... }`: Returns the comprehensive result object:
    *   `canInviteAll`: True if `validateSeatAvailability` indicates `canInvite` AND there are actually emails to invite.
    *   `validEmails`: The `finalEmailsToInvite` list.
    *   `duplicateEmails`: The list of duplicates found in the original input.
    *   `existingMembers`: Valid emails from the input that are *already members*.
    *   `seatsNeeded`: Number of `finalEmailsToInvite`.
    *   `seatsAvailable`: Directly from `validationResult`.
    *   `validationResult`: The full result from the seat availability check.
*   `catch (error) { ... }`: Catches and logs errors.
*   `const validationResult: SeatValidationResult = { ... }`: Creates a default `SeatValidationResult` for error scenarios.
*   `return { canInviteAll: false, ... }`: Returns a default failure object if an error occurs during validation.

---

### **`updateOrganizationSeats` Function**

This function allows an administrator to manually adjust the maximum number of seats an organization can have, with a crucial check to prevent reducing seats below the current member count.

```typescript
/**
 * Update organization seat count in subscription
 */
export async function updateOrganizationSeats(
  organizationId: string,
  newSeatCount: number,
  updatedBy: string
): Promise<{ success: boolean; error?: string }> {
  try {
    const subscriptionRecord = await getOrganizationSubscription(organizationId)

    if (!subscriptionRecord) {
      return { success: false, error: 'No active subscription found' }
    }

    const memberCount = await db
      .select({ count: count() })
      .from(member)
      .where(eq(member.organizationId, organizationId))

    const currentMembers = memberCount[0]?.count || 0

    if (newSeatCount < currentMembers) {
      return {
        success: false,
        error: `Cannot reduce seats below current member count (${currentMembers})`,
      }
    }

    await db
      .update(subscription)
      .set({
        seats: newSeatCount,
      })
      .where(eq(subscription.id, subscriptionRecord.id))

    logger.info('Organization seat count updated', {
      organizationId,
      oldSeatCount: subscriptionRecord.seats,
      newSeatCount,
      updatedBy,
    })

    return { success: true }
  } catch (error) {
    logger.error('Failed to update organization seats', {
      organizationId,
      newSeatCount,
      updatedBy,
      error,
    })

    return {
      success: false,
      error: error instanceof Error ? error.message : 'Unknown error',
    }
  }
}
```

**Purpose:** To change the allocated `maxSeats` for an organization, typically managed via its subscription details.

**Simplified Logic:**
1.  Fetch the organization's subscription. If not found, report an error.
2.  Count the current number of members.
3.  Crucially, check if the `newSeatCount` is less than the `currentMembers`. If so, prevent the update and return an error, as you cannot remove seats that are already occupied.
4.  If valid, update the `seats` column in the `subscription` record in the database.
5.  Log the change and return success.

**Line-by-Line Explanation:**

*   `export async function updateOrganizationSeats(...)`: Defines the asynchronous function. It takes `organizationId`, `newSeatCount` (the desired max seats), and `updatedBy` (who is performing the action) and returns a `Promise` resolving to an object indicating `success` and an optional `error` message.
*   `try { ... } catch (error) { ... }`: Standard error handling.
*   `const subscriptionRecord = await getOrganizationSubscription(organizationId)`: Fetches the current subscription record.
*   `if (!subscriptionRecord) { ... }`: If no subscription is found, return `false` with an error.
*   `const memberCount = await db.select({ count: count() }).from(member).where(eq(member.organizationId, organizationId))`: Counts the current number of members in the organization.
*   `const currentMembers = memberCount[0]?.count || 0`: Extracts the current member count.
*   `if (newSeatCount < currentMembers) { ... }`: **Important Business Logic:** This check prevents reducing the `maxSeats` below the number of `currentMembers`. It ensures that an organization cannot accidentally or intentionally have more members than allowed by its seat count.
*   `return { success: false, error: ... }`: If the `newSeatCount` is too low, returns `false` with a descriptive error message.
*   `await db.update(subscription).set({ seats: newSeatCount }).where(eq(subscription.id, subscriptionRecord.id))`: This is a Drizzle ORM update query:
    *   `db.update(subscription)`: Specifies that we are updating the `subscription` table.
    *   `.set({ seats: newSeatCount })`: Sets the `seats` column to the `newSeatCount` value.
    *   `.where(eq(subscription.id, subscriptionRecord.id))`: Filters the update to target the specific subscription record identified by `subscriptionRecord.id`.
*   `logger.info('Organization seat count updated', { ... })`: Logs an informational message about the successful update, including old and new seat counts and who updated it.
*   `return { success: true }`: Returns a success object.
*   `catch (error) { ... }`: Catches and logs errors.
*   `return { success: false, error: ... }`: Returns a failure object, including the error message if it's an `Error` instance.

---

### **`validateMemberRemoval` Function**

This function verifies if a user can be removed from an organization, applying role-based access control rules.

```typescript
/**
 * Check if a user can be removed from an organization
 */
export async function validateMemberRemoval(
  organizationId: string,
  userIdToRemove: string,
  removedBy: string
): Promise<{ canRemove: boolean; reason?: string }> {
  try {
    const memberRecord = await db
      .select({ role: member.role })
      .from(member)
      .where(and(eq(member.organizationId, organizationId), eq(member.userId, userIdToRemove)))
      .limit(1)

    if (memberRecord.length === 0) {
      return { canRemove: false, reason: 'Member not found in organization' }
    }

    if (memberRecord[0].role === 'owner') {
      return { canRemove: false, reason: 'Cannot remove organization owner' }
    }

    const removerMemberRecord = await db
      .select({ role: member.role })
      .from(member)
      .where(and(eq(member.organizationId, organizationId), eq(member.userId, removedBy)))
      .limit(1)

    if (removerMemberRecord.length === 0) {
      return { canRemove: false, reason: 'You are not a member of this organization' }
    }

    const removerRole = removerMemberRecord[0].role
    const targetRole = memberRecord[0].role

    if (removerRole === 'owner') {
      return userIdToRemove === removedBy
        ? { canRemove: false, reason: 'Cannot remove yourself as owner' }
        : { canRemove: true }
    }

    if (removerRole === 'admin') {
      return targetRole === 'member'
        ? { canRemove: true }
        : { canRemove: false, reason: 'Insufficient permissions to remove this member' }
    }

    return { canRemove: false, reason: 'Insufficient permissions' }
  } catch (error) {
    logger.error('Failed to validate member removal', {
      organizationId,
      userIdToRemove,
      removedBy,
      error,
    })

    return { canRemove: false, reason: 'Validation failed' }
  }
}
```

**Purpose:** To ensure that a user attempting to remove another user from an organization has the necessary permissions and that the removal action is valid according to organizational rules (e.g., preventing owners from being removed by anyone other than themselves, or preventing admins from removing other admins/owners).

**Simplified Logic:**
1.  Verify that the `userIdToRemove` exists in the organization. If not, can't remove.
2.  If the target is an 'owner', immediately block removal (owners have special protections).
3.  Verify that the `removedBy` user exists in the organization. If not, they can't perform the action.
4.  Determine the roles of both the `removedBy` user and the `userIdToRemove`.
5.  Apply role-based rules:
    *   **Owner Removing:** An owner can remove any member, but cannot remove themselves.
    *   **Admin Removing:** An admin can only remove 'member' roles. They cannot remove other admins or owners.
6.  If no specific role allows the removal, block it due to insufficient permissions.

**Line-by-Line Explanation:**

*   `export async function validateMemberRemoval(...)`: Defines the asynchronous function. It takes `organizationId`, `userIdToRemove`, and `removedBy` and returns a `Promise` resolving to an object with `canRemove` (boolean) and an optional `reason`.
*   `try { ... } catch (error) { ... }`: Standard error handling.
*   `const memberRecord = await db.select({ role: member.role }).from(member).where(and(eq(member.organizationId, organizationId), eq(member.userId, userIdToRemove))).limit(1)`: Drizzle query to fetch the `role` of the user being targeted for removal.
    *   `and(...)`: Combines conditions for `organizationId` and `userIdToRemove`.
*   `if (memberRecord.length === 0) { ... }`: If the `userIdToRemove` is not found as a member of the organization, return `canRemove: false`.
*   `if (memberRecord[0].role === 'owner') { ... }`: **Important Business Logic:** If the target user is an 'owner', they cannot be removed by anyone except specific owner self-removal (handled later). This is a strong protection.
*   `const removerMemberRecord = await db.select({ role: member.role }).from(member).where(and(eq(member.organizationId, organizationId), eq(member.userId, removedBy))).limit(1)`: Drizzle query to fetch the `role` of the user performing the removal (`removedBy`).
*   `if (removerMemberRecord.length === 0) { ... }`: If the `removedBy` user is not a member of the organization, they cannot perform the removal.
*   `const removerRole = removerMemberRecord[0].role`: Extracts the role of the user initiating the removal.
*   `const targetRole = memberRecord[0].role`: Extracts the role of the user being removed.
*   `if (removerRole === 'owner') { ... }`: Checks if the `removedBy` user is an 'owner'.
    *   `return userIdToRemove === removedBy ? { canRemove: false, reason: 'Cannot remove yourself as owner' } : { canRemove: true }`: **Important Business Logic:** An owner can remove anyone *except* themselves. This prevents an organization from losing its sole owner.
*   `if (removerRole === 'admin') { ... }`: Checks if the `removedBy` user is an 'admin'.
    *   `return targetRole === 'member' ? { canRemove: true } : { canRemove: false, reason: 'Insufficient permissions to remove this member' }`: **Important Business Logic:** An admin can only remove users with the 'member' role. They cannot remove other 'admin's or 'owner's.
*   `return { canRemove: false, reason: 'Insufficient permissions' }`: If none of the above conditions (owner or admin with specific permissions) are met, then the `removedBy` user has insufficient permissions for any other role.
*   `catch (error) { ... }`: Catches and logs errors.
*   `return { canRemove: false, reason: 'Validation failed' }`: Returns a generic failure on error.

---

### **`getOrganizationSeatAnalytics` Function**

This function compiles detailed analytics about an organization's seat usage and member activity.

```typescript
/**
 * Get seat usage analytics for an organization
 */
export async function getOrganizationSeatAnalytics(organizationId: string) {
  try {
    const seatInfo = await getOrganizationSeatInfo(organizationId)

    if (!seatInfo) {
      return null
    }

    const memberActivity = await db
      .select({
        userId: member.userId,
        userName: user.name,
        userEmail: user.email,
        role: member.role,
        joinedAt: member.createdAt,
        lastActive: userStats.lastActive,
      })
      .from(member)
      .innerJoin(user, eq(member.userId, user.id))
      .leftJoin(userStats, eq(member.userId, userStats.userId))
      .where(eq(member.organizationId, organizationId))

    const utilizationRate =
      seatInfo.maxSeats > 0 ? (seatInfo.currentSeats / seatInfo.maxSeats) * 100 : 0

    const recentlyActive = memberActivity.filter((memberData) => {
      if (!memberData.lastActive) return false
      const daysSinceActive = (Date.now() - memberData.lastActive.getTime()) / (1000 * 60 * 60 * 24)
      return daysSinceActive <= 30 // Active in last 30 days
    }).length

    return {
      ...seatInfo,
      utilizationRate: Math.round(utilizationRate * 100) / 100,
      activeMembers: recentlyActive,
      inactiveMembers: seatInfo.currentSeats - recentlyActive,
      memberActivity,
    }
  } catch (error) {
    logger.error('Failed to get organization seat analytics', { organizationId, error })
    return null
  }
}
```

**Purpose:** To provide a comprehensive report on seat usage, including the overall utilization rate and a breakdown of member activity (e.g., how many members were recently active).

**Simplified Logic:**
1.  Fetch basic seat information using `getOrganizationSeatInfo`. If not found, return `null`.
2.  Query the database to get detailed information about each member, including their name, email, role, join date, and `lastActive` timestamp (from `userStats`).
3.  Calculate the `utilizationRate` (current seats / max seats).
4.  Determine how many members were "recently active" (e.g., in the last 30 days) based on their `lastActive` timestamp.
5.  Combine all this data into a single analytics object, including the `seatInfo` and the new calculated metrics.

**Line-by-Line Explanation:**

*   `export async function getOrganizationSeatAnalytics(organizationId: string)`: Defines the asynchronous function. It takes an `organizationId` and returns a `Promise` resolving to an analytics object or `null`.
*   `try { ... } catch (error) { ... }`: Standard error handling.
*   `const seatInfo = await getOrganizationSeatInfo(organizationId)`: Reuses the `getOrganizationSeatInfo` function to get the foundational seat data.
*   `if (!seatInfo) { return null }`: If basic seat info cannot be retrieved, returns `null`.
*   `const memberActivity = await db.select(...).from(member).innerJoin(user, ...).leftJoin(userStats, ...).where(...)`: This is a complex Drizzle query to get detailed member information:
    *   `.select({ userId: member.userId, userName: user.name, userEmail: user.email, role: member.role, joinedAt: member.createdAt, lastActive: userStats.lastActive })`: Selects various fields from `member`, `user`, and `userStats` tables.
    *   `.from(member)`: Starts with the `member` table.
    *   `.innerJoin(user, eq(member.userId, user.id))`: Joins `member` with `user` to get user name and email. `innerJoin` means only members who have a corresponding user record will be included.
    *   `.leftJoin(userStats, eq(member.userId, userStats.userId))`: Joins `member` with `userStats` to get `lastActive` data. `leftJoin` means all members will be included, even if they don't have a `userStats` record (in which case `userStats.lastActive` would be `null`).
    *   `.where(eq(member.organizationId, organizationId))`: Filters for members of the specific organization.
*   `const utilizationRate = seatInfo.maxSeats > 0 ? (seatInfo.currentSeats / seatInfo.maxSeats) * 100 : 0`: Calculates the percentage of seats currently used. It prevents division by zero by checking `seatInfo.maxSeats > 0`.
*   `const recentlyActive = memberActivity.filter((memberData) => { ... }).length`: Filters the `memberActivity` array to find members who have been active in the last 30 days.
    *   `if (!memberData.lastActive) return false`: If a member has no `lastActive` timestamp (e.g., from `leftJoin` or if it's never set), they are not considered recently active.
    *   `const daysSinceActive = (Date.now() - memberData.lastActive.getTime()) / (1000 * 60 * 60 * 24)`: Calculates the difference between the current time and `lastActive` time in milliseconds, then converts it to days.
    *   `return daysSinceActive <= 30`: Keeps only members whose last activity was within the last 30 days.
    *   `.length`: Counts the number of such active members.
*   `return { ...seatInfo, utilizationRate: Math.round(utilizationRate * 100) / 100, activeMembers: recentlyActive, inactiveMembers: seatInfo.currentSeats - recentlyActive, memberActivity, }`: Returns the final analytics object:
    *   `...seatInfo`: Spreads all properties from the `seatInfo` object, effectively merging it into the new object.
    *   `utilizationRate: Math.round(utilizationRate * 100) / 100`: The calculated utilization rate, rounded to two decimal places.
    *   `activeMembers`: The count of recently active members.
    *   `inactiveMembers`: Calculated by subtracting `activeMembers` from `currentSeats`.
    *   `memberActivity`: The detailed list of all members and their activity data.
*   `catch (error) { ... }`: Catches and logs errors.
*   `return null`: Returns `null` on error.

---

This detailed breakdown covers the purpose, simplified logic, and line-by-line explanation of the entire `seat-management.ts` file, highlighting key functionalities, Drizzle ORM queries, and important business logic checks.