This TypeScript file defines a Next.js API Route Handler that manages members within an organization. It provides three distinct functionalities: fetching details of an organization member (`GET`), updating a member's role (`PUT`), and removing a member from an organization (`DELETE`).

The file leverages Drizzle ORM for database interactions, Next.js for API routing, and integrates with an authentication system, a billing system, and a logging utility.

---

## Overall Purpose and What This File Does

This file acts as the backend API endpoint for managing individual organization members. It's accessible at a URL pattern like `/api/organizations/[organizationId]/members/[memberId]`.

Here's a breakdown of its primary functions:

1.  **`GET` Request (Fetch Member Details):**
    *   Allows an authenticated user to retrieve detailed information about a specific member of an organization.
    *   Includes robust permission checks: the requesting user must be a member of the organization, and either an admin/owner or the target member themselves, to view details.
    *   Optionally fetches and combines billing usage data if the requesting user has administrative privileges and explicitly asks for it.

2.  **`PUT` Request (Update Member Role):**
    *   Enables an authenticated and authorized user (owner or admin) to change the role of another member within the organization (e.g., from 'member' to 'admin', or vice-versa).
    *   Implements strict role-based access control (RBAC) rules:
        *   Only owners can promote members to 'admin'.
        *   Only owners can modify the roles of existing 'admin' members.
        *   The 'owner' role itself cannot be changed.

3.  **`DELETE` Request (Remove Member):**
    *   Allows an authenticated and authorized user to remove a member from an organization. This can be done by an owner/admin removing any member, or a regular member removing themselves.
    *   Prevents the organization's 'owner' from being removed.
    *   Includes a crucial and complex post-removal billing logic: if the removed user was on a personal 'Pro' plan that was set to cancel (because they were previously on a paid team plan), and by leaving *this* organization they are no longer part of *any* paid team, their personal 'Pro' plan's cancellation is reverted, and their usage statistics are restored.

In essence, this file is a secure and feature-rich API for granular management of organization members, with careful consideration for permissions, roles, and billing implications.

---

## Core Concepts and Tools Used

Before diving into the line-by-line explanation, let's briefly touch upon the main technologies and patterns:

*   **Next.js Route Handlers:** The `export async function GET`, `PUT`, `DELETE` functions are Next.js 13+ Route Handlers. They act as API endpoints, directly responding to HTTP requests.
*   **Drizzle ORM:** A type-safe ORM for TypeScript that allows interaction with databases using a syntax similar to SQL, but with full TypeScript inference. It's used for all database queries (`db.select`, `db.update`, `db.delete`).
*   **Authentication (`getSession`):** A custom utility (`@/lib/auth`) to retrieve the authenticated user's session, which typically includes their `userId`.
*   **Role-Based Access Control (RBAC):** Permissions are often determined by a user's `role` within an organization (e.g., 'owner', 'admin', 'member').
*   **Logging (`createLogger`):** A custom logging utility (`@/lib/logs/console/logger`) to record events and errors, aiding in debugging and monitoring.
*   **Stripe Integration (`requireStripeClient`):** Used in the `DELETE` method to interact with the Stripe API for managing subscriptions.

---

## Detailed Line-by-Line Explanation

```typescript
import { db } from '@sim/db'
```

*   **`import { db } from '@sim/db'`**: This line imports the `db` object, which is an instance of the Drizzle ORM client configured to connect to your database. This `db` object is the primary interface for all database operations in this file.

```typescript
import { member, subscription as subscriptionTable, user, userStats } from '@sim/db/schema'
```

*   **`import { member, subscription as subscriptionTable, user, userStats } from '@sim/db/schema'`**: This line imports table schema definitions from your Drizzle database schema.
    *   `member`: Represents the `member` table, which links users to organizations and defines their role within that organization.
    *   `subscription as subscriptionTable`: Imports the `subscription` table. It's aliased to `subscriptionTable` to avoid naming conflicts if another `subscription` variable is used in the file. This table likely stores details about user or organization subscriptions (e.g., plan, status).
    *   `user`: Represents the `user` table, containing general user information like name and email.
    *   `userStats`: Represents the `userStats` table, which might store billing-related statistics or usage data for individual users.

```typescript
import { and, eq } from 'drizzle-orm'
```

*   **`import { and, eq } from 'drizzle-orm'`**: These are helper functions from Drizzle ORM used for constructing database queries.
    *   `and`: Combines multiple conditions with a logical AND operator.
    *   `eq`: Checks if two values are equal (used for `WHERE` clauses).

```typescript
import { type NextRequest, NextResponse } from 'next/server'
```

*   **`import { type NextRequest, NextResponse } from 'next/server'`**: These are Next.js specific imports for handling API routes.
    *   `NextRequest`: Represents the incoming HTTP request, providing access to headers, body, URL, etc. (`type` indicates it's only imported for type checking, not for runtime value).
    *   `NextResponse`: Used to construct and send the HTTP response back to the client, including JSON data, status codes, and headers.

```typescript
import { getSession } from '@/lib/auth'
```

*   **`import { getSession } from '@/lib/auth'`**: Imports a custom utility function `getSession` from `@/lib/auth`. This function is responsible for retrieving the authenticated user's session data, which typically contains information like the user's ID.

```typescript
import { getUserUsageData } from '@/lib/billing/core/usage'
```

*   **`import { getUserUsageData } from '@/lib/billing/core/usage'`**: Imports a function `getUserUsageData` from your billing core library. This function likely calculates or fetches more dynamic user usage data, potentially including billing period dates.

```typescript
import { requireStripeClient } from '@/lib/billing/stripe-client'
```

*   **`import { requireStripeClient } from '@/lib/billing/stripe-client'`**: Imports a function `requireStripeClient` from your Stripe integration library. This function returns a configured Stripe client instance, allowing you to interact with the Stripe API.

```typescript
import { createLogger } from '@/lib/logs/console/logger'
```

*   **`import { createLogger } from '@/lib/logs/console/logger'`**: Imports a function `createLogger` from your logging utility. This allows you to create a specific logger instance for this file, making log messages easier to trace back to their source.

```typescript
const logger = createLogger('OrganizationMemberAPI')
```

*   **`const logger = createLogger('OrganizationMemberAPI')`**: Initializes a logger instance named 'OrganizationMemberAPI'. All log messages from this file will be associated with this name, which is helpful for filtering and understanding logs.

---

### `GET` Endpoint: Fetching Organization Member Details

```typescript
/**
 * GET /api/organizations/[id]/members/[memberId]
 * Get individual organization member details
 */
export async function GET(
  request: NextRequest,
  { params }: { params: Promise<{ id: string; memberId: string }> }
) {
  try {
    // ... (rest of the GET method)
  } catch (error) {
    // ... (error handling)
  }
}
```

*   **`/** ... */`**: A JSDoc comment explaining the purpose of this function and the API endpoint it handles.
*   **`export async function GET(...)`**: Defines an asynchronous function named `GET`. In Next.js Route Handlers, exported functions matching HTTP methods (GET, POST, PUT, DELETE, etc.) are automatically invoked when a request for that method hits the route.
    *   `request: NextRequest`: The first argument is the `NextRequest` object, containing details about the incoming request.
    *   `{ params }: { params: Promise<{ id: string; memberId: string }> }`: The second argument is an object containing `params`. `params` itself is a `Promise` that resolves to an object with `id` (the organization ID) and `memberId` (the specific member's user ID) extracted from the URL path. Next.js automatically parses dynamic route segments like `[id]` and `[memberId]` into this `params` object.

#### `GET` Method - Core Logic

```typescript
  try {
    const session = await getSession()
```

*   **`try {`**: Starts a `try` block. This is crucial for error handling; any exceptions thrown within this block will be caught by the `catch` block below.
*   **`const session = await getSession()`**: Calls the `getSession()` utility to retrieve the current user's authenticated session. This function is `await`ed because fetching session data is typically an asynchronous operation (e.g., checking a cookie or token against a database).

```typescript
    if (!session?.user?.id) {
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
    }
```

*   **`if (!session?.user?.id)`**: Checks if the session object exists and contains a `user` object with an `id`. If not, it means the user is not authenticated.
*   **`return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })`**: If the user is not authenticated, it returns a JSON response with an `error` message and an HTTP `401 Unauthorized` status code. This immediately stops further execution.

```typescript
    const { id: organizationId, memberId } = await params
```

*   **`const { id: organizationId, memberId } = await params`**: Destructures the `params` object (which was awaited) to extract the `id` (renaming it to `organizationId` for clarity) and `memberId` from the URL.

```typescript
    const url = new URL(request.url)
    const includeUsage = url.searchParams.get('include') === 'usage'
```

*   **`const url = new URL(request.url)`**: Creates a `URL` object from the `request.url`. This object provides convenient methods to work with URL components.
*   **`const includeUsage = url.searchParams.get('include') === 'usage'`**: Checks if the URL contains a query parameter `include=usage`. If present, the `includeUsage` flag will be `true`, indicating that the client wants to receive usage data for the member.

```typescript
    // Verify user has access to this organization
    const userMember = await db
      .select()
      .from(member)
      .where(and(eq(member.organizationId, organizationId), eq(member.userId, session.user.id)))
      .limit(1)
```

*   **`// Verify user has access to this organization`**: A comment indicating the purpose of the following code.
*   **`const userMember = await db .select() .from(member) .where(and(eq(member.organizationId, organizationId), eq(member.userId, session.user.id))) .limit(1)`**: This is a Drizzle ORM query to check if the *requesting* user (`session.user.id`) is actually a member of the organization (`organizationId`).
    *   `db.select()`: Starts a select query, by default selecting all columns.
    *   `.from(member)`: Specifies the `member` table.
    *   `.where(and(eq(member.organizationId, organizationId), eq(member.userId, session.user.id)))`: Filters the results.
        *   `and(...)`: Combines two conditions.
        *   `eq(member.organizationId, organizationId)`: Ensures the member record belongs to the specified `organizationId`.
        *   `eq(member.userId, session.user.id)`: Ensures the member record belongs to the currently authenticated user.
    *   `.limit(1)`: Optimizes the query by asking for only one matching record since we only need to know if *any* record exists.

```typescript
    if (userMember.length === 0) {
      return NextResponse.json(
        { error: 'Forbidden - Not a member of this organization' },
        { status: 403 }
      )
    }
```

*   **`if (userMember.length === 0)`**: If no `member` record was found for the requesting user in the specified organization, it means they are not a member.
*   **`return NextResponse.json(...)`**: Returns a JSON response with a `403 Forbidden` status, indicating that the user does not have permission to access resources within this organization.

```typescript
    const userRole = userMember[0].role
    const hasAdminAccess = ['owner', 'admin'].includes(userRole)
```

*   **`const userRole = userMember[0].role`**: Retrieves the role of the *requesting* user from the `userMember` array (since `limit(1)` was used, there will be at most one entry at index 0).
*   **`const hasAdminAccess = ['owner', 'admin'].includes(userRole)`**: A boolean flag indicating whether the requesting user has administrative privileges (i.e., their `userRole` is either 'owner' or 'admin').

```typescript
    // Get target member details
    const memberQuery = db
      .select({
        id: member.id,
        userId: member.userId,
        organizationId: member.organizationId,
        role: member.role,
        createdAt: member.createdAt,
        userName: user.name,
        userEmail: user.email,
      })
      .from(member)
      .innerJoin(user, eq(member.userId, user.id))
      .where(and(eq(member.organizationId, organizationId), eq(member.userId, memberId)))
      .limit(1)
```

*   **`// Get target member details`**: Comment for the next code block.
*   **`const memberQuery = db .select({...}) .from(member) .innerJoin(user, eq(member.userId, user.id)) .where(and(eq(member.organizationId, organizationId), eq(member.userId, memberId))) .limit(1)`**: This Drizzle query fetches the details of the *target* member (`memberId`).
    *   `db.select({...})`: Selects specific columns to include in the result.
        *   It explicitly selects columns from the `member` table (`id`, `userId`, `organizationId`, `role`, `createdAt`).
        *   It also selects `name` and `email` from the `user` table, aliasing them to `userName` and `userEmail` for clarity in the final output.
    *   `.from(member)`: Starts the query from the `member` table.
    *   `.innerJoin(user, eq(member.userId, user.id))`: Performs an inner join with the `user` table. The join condition `eq(member.userId, user.id)` links member records to their corresponding user records using the `userId`.
    *   `.where(and(eq(member.organizationId, organizationId), eq(member.userId, memberId)))`: Filters the results to find the specific target member within the given organization.
        *   `eq(member.organizationId, organizationId)`: Ensures the member belongs to the correct organization.
        *   `eq(member.userId, memberId)`: Matches the `userId` of the target member.
    *   `.limit(1)`: Again, optimizes the query by expecting only one result.

```typescript
    const memberEntry = await memberQuery
```

*   **`const memberEntry = await memberQuery`**: Executes the constructed Drizzle query and awaits its result. `memberEntry` will be an array of objects.

```typescript
    if (memberEntry.length === 0) {
      return NextResponse.json({ error: 'Member not found' }, { status: 404 })
    }
```

*   **`if (memberEntry.length === 0)`**: If no record is found for the `memberId` in the `organizationId`, it means the target member does not exist.
*   **`return NextResponse.json(...)`**: Returns a JSON response with a `404 Not Found` status.

```typescript
    // Check if user can view this member's details
    const canViewDetails = hasAdminAccess || session.user.id === memberId
```

*   **`// Check if user can view this member's details`**: Comment for permission check.
*   **`const canViewDetails = hasAdminAccess || session.user.id === memberId`**: This line determines if the requesting user has permission to view the target member's details.
    *   `hasAdminAccess`: True if the requesting user is an 'owner' or 'admin'.
    *   `session.user.id === memberId`: True if the requesting user is trying to view *their own* details.
    *   The `||` (OR) operator means either condition is sufficient for viewing access.

```typescript
    if (!canViewDetails) {
      return NextResponse.json({ error: 'Forbidden - Insufficient permissions' }, { status: 403 })
    }
```

*   **`if (!canViewDetails)`**: If the requesting user doesn't meet the `canViewDetails` criteria.
*   **`return NextResponse.json(...)`**: Returns a JSON response with a `403 Forbidden` status.

```typescript
    let memberData = memberEntry[0]
```

*   **`let memberData = memberEntry[0]`**: Initializes a mutable variable `memberData` with the details of the found member. This uses `let` because `memberData` might be updated later if usage data is included.

```typescript
    // Include usage data if requested and user has permission
    if (includeUsage && hasAdminAccess) {
      const usageData = await db
        .select({
          currentPeriodCost: userStats.currentPeriodCost,
          currentUsageLimit: userStats.currentUsageLimit,
          usageLimitUpdatedAt: userStats.usageLimitUpdatedAt,
          lastPeriodCost: userStats.lastPeriodCost,
        })
        .from(userStats)
        .where(eq(userStats.userId, memberId))
        .limit(1)
```

*   **`// Include usage data if requested and user has permission`**: Comment for conditional usage data inclusion.
*   **`if (includeUsage && hasAdminAccess)`**: This block only executes if the client requested usage data (`includeUsage` is true) AND the requesting user has admin access (`hasAdminAccess` is true). Regular members cannot view other members' usage data.
*   **`const usageData = await db .select({...}) .from(userStats) .where(eq(userStats.userId, memberId)) .limit(1)`**: This Drizzle query fetches specific usage statistics for the target `memberId` from the `userStats` table. It selects `currentPeriodCost`, `currentUsageLimit`, `usageLimitUpdatedAt`, and `lastPeriodCost`.

```typescript
      const computed = await getUserUsageData(memberId)
```

*   **`const computed = await getUserUsageData(memberId)`**: Calls the `getUserUsageData` helper function, passing the `memberId`. This function likely performs calculations or fetches additional dynamic billing information (e.g., the exact start and end dates of the current billing period) not directly stored in `userStats`.

```typescript
      if (usageData.length > 0) {
        memberData = {
          ...memberData,
          usage: {
            ...usageData[0],
            billingPeriodStart: computed.billingPeriodStart,
            billingPeriodEnd: computed.billingPeriodEnd,
          },
        } as typeof memberData & {
          usage: (typeof usageData)[0] & {
            billingPeriodStart: Date | null
            billingPeriodEnd: Date | null
          }
        }
      }
    }
```

*   **`if (usageData.length > 0)`**: Checks if any usage data was found in `userStats`.
*   **`memberData = { ...memberData, usage: { ...usageData[0], billingPeriodStart: computed.billingPeriodStart, billingPeriodEnd: computed.billingPeriodEnd, }, }`**: This line updates the `memberData` object by adding a new `usage` property.
    *   `...memberData`: Spreads all existing properties of `memberData` into the new object.
    *   `usage: { ... }`: Creates a new `usage` object.
    *   `...usageData[0]`: Spreads all properties from the fetched `usageData` (e.g., `currentPeriodCost`) into the `usage` object.
    *   `billingPeriodStart: computed.billingPeriodStart, billingPeriodEnd: computed.billingPeriodEnd`: Adds the computed billing period dates to the `usage` object.
*   **`as typeof memberData & { usage: (typeof usageData)[0] & { billingPeriodStart: Date | null; billingPeriodEnd: Date | null } }`**: This is a TypeScript type assertion. It explicitly tells TypeScript that the `memberData` variable, after this assignment, now has the original `memberData` type *plus* a new `usage` property with the specified structure. This is necessary because TypeScript's type inference might not automatically know that `memberData` has gained a new property.

```typescript
    return NextResponse.json({
      success: true,
      data: memberData,
      userRole,
      hasAdminAccess,
    })
```

*   **`return NextResponse.json({...})`**: If everything is successful, it returns a JSON response.
    *   `success: true`: A boolean indicating success.
    *   `data: memberData`: The collected member details (potentially including usage data).
    *   `userRole`: The role of the *requesting* user (useful for frontends to adjust UI based on permissions).
    *   `hasAdminAccess`: A boolean indicating if the *requesting* user has admin access.

```typescript
  } catch (error) {
    logger.error('Failed to get organization member', {
      organizationId: (await params).id,
      memberId: (await params).memberId,
      error,
    })

    return NextResponse.json({ error: 'Internal server error' }, { status: 500 })
  }
```

*   **`} catch (error) {`**: Catches any errors that occurred in the `try` block.
*   **`logger.error('Failed to get organization member', { ... })`**: Logs the error using the `logger` instance. It includes contextual information like `organizationId`, `memberId`, and the `error` object itself, which is vital for debugging. `(await params).id` and `(await params).memberId` are awaited here because `params` is a `Promise`.
*   **`return NextResponse.json({ error: 'Internal server error' }, { status: 500 })`**: Returns a generic `500 Internal Server Error` response to the client, preventing sensitive error details from being exposed.

---

### `PUT` Endpoint: Updating Organization Member Role

```typescript
/**
 * PUT /api/organizations/[id]/members/[memberId]
 * Update organization member role
 */
export async function PUT(
  request: NextRequest,
  { params }: { params: Promise<{ id: string; memberId: string }> }
) {
  try {
    // ... (rest of the PUT method)
  } catch (error) {
    // ... (error handling)
  }
}
```

*   **`/** ... */`**: JSDoc comment for the `PUT` endpoint.
*   **`export async function PUT(...)`**: Defines the asynchronous `PUT` Route Handler, with the same `NextRequest` and `params` arguments as `GET`. This function is designed to handle requests to update a member's role.

#### `PUT` Method - Core Logic

```typescript
  try {
    const session = await getSession()

    if (!session?.user?.id) {
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
    }
```

*   **`try { ... }`**: Standard error handling block.
*   **`const session = await getSession()`**: Fetches the authenticated user's session.
*   **`if (!session?.user?.id)`**: Unauthorized check, same as in `GET`.

```typescript
    const { id: organizationId, memberId } = await params
    const { role } = await request.json()
```

*   **`const { id: organizationId, memberId } = await params`**: Extracts `organizationId` and `memberId` from the URL parameters.
*   **`const { role } = await request.json()`**: Parses the JSON body of the incoming request. It expects an object containing a `role` property (e.g., `{ "role": "admin" }`). `request.json()` is awaited as reading the body is an asynchronous operation.

```typescript
    // Validate input
    if (!role || !['admin', 'member'].includes(role)) {
      return NextResponse.json({ error: 'Invalid role' }, { status: 400 })
    }
```

*   **`// Validate input`**: Comment for input validation.
*   **`if (!role || !['admin', 'member'].includes(role))`**: Validates the `role` received from the request body.
    *   `!role`: Checks if `role` is undefined, null, or an empty string.
    *   `!['admin', 'member'].includes(role)`: Checks if the `role` is not one of the allowed values ('admin' or 'member'). The 'owner' role is typically set during organization creation and isn't changeable via this API to prevent accidental lockouts.
*   **`return NextResponse.json(...)`**: If the role is invalid, returns a `400 Bad Request` error.

```typescript
    // Verify user has admin access
    const userMember = await db
      .select()
      .from(member)
      .where(and(eq(member.organizationId, organizationId), eq(member.userId, session.user.id)))
      .limit(1)

    if (userMember.length === 0) {
      return NextResponse.json(
        { error: 'Forbidden - Not a member of this organization' },
        { status: 403 }
      )
    }

    if (!['owner', 'admin'].includes(userMember[0].role)) {
      return NextResponse.json({ error: 'Forbidden - Admin access required' }, { status: 403 })
    }
```

*   **`// Verify user has admin access`**: Comment for permission check.
*   **`const userMember = await db.select()...limit(1)`**: Same query as in `GET` to verify the *requesting* user is a member of the organization.
*   **`if (userMember.length === 0)`**: If the requesting user isn't a member, returns `403 Forbidden`.
*   **`if (!['owner', 'admin'].includes(userMember[0].role))`**: This is the core permission check for `PUT`. Only users with 'owner' or 'admin' roles can update other members' roles. If not, returns `403 Forbidden`.

```typescript
    // Check if target member exists
    const targetMember = await db
      .select()
      .from(member)
      .where(and(eq(member.organizationId, organizationId), eq(member.userId, memberId)))
      .limit(1)

    if (targetMember.length === 0) {
      return NextResponse.json({ error: 'Member not found' }, { status: 404 })
    }
```

*   **`// Check if target member exists`**: Comment for target member existence check.
*   **`const targetMember = await db.select()...limit(1)`**: Queries the database to ensure the `memberId` (the member whose role is being updated) actually exists within the `organizationId`.
*   **`if (targetMember.length === 0)`**: If the target member is not found, returns `404 Not Found`.

```typescript
    // Prevent changing owner role
    if (targetMember[0].role === 'owner') {
      return NextResponse.json({ error: 'Cannot change owner role' }, { status: 400 })
    }
```

*   **`// Prevent changing owner role`**: Comment for owner role protection.
*   **`if (targetMember[0].role === 'owner')`**: This is a critical rule: the 'owner' role cannot be changed via this API. This prevents an organization from losing its owner or an owner from accidentally demoting themselves.
*   **`return NextResponse.json(...)`**: Returns `400 Bad Request` if an attempt is made to change the owner's role.

```typescript
    // Prevent non-owners from promoting to admin
    if (role === 'admin' && userMember[0].role !== 'owner') {
      return NextResponse.json(
        { error: 'Only owners can promote members to admin' },
        { status: 403 }
      )
    }
```

*   **`// Prevent non-owners from promoting to admin`**: Comment for promotion rule.
*   **`if (role === 'admin' && userMember[0].role !== 'owner')`**: This rule dictates that only an 'owner' can promote another member to 'admin'. An existing 'admin' cannot promote someone else to 'admin'.
*   **`return NextResponse.json(...)`**: Returns `403 Forbidden` if this rule is violated.

```typescript
    // Prevent admins from changing other admins' roles - only owners can modify admin roles
    if (targetMember[0].role === 'admin' && userMember[0].role !== 'owner') {
      return NextResponse.json({ error: 'Only owners can change admin roles' }, { status: 403 })
    }
```

*   **`// Prevent admins from changing other admins' roles - only owners can modify admin roles`**: Comment for admin role modification rule.
*   **`if (targetMember[0].role === 'admin' && userMember[0].role !== 'owner')`**: This rule prevents an 'admin' from changing the role of another 'admin'. Only an 'owner' has the authority to demote or modify the role of an 'admin'.
*   **`return NextResponse.json(...)`**: Returns `403 Forbidden` if this rule is violated.

```typescript
    // Update member role
    const updatedMember = await db
      .update(member)
      .set({ role })
      .where(and(eq(member.organizationId, organizationId), eq(member.userId, memberId)))
      .returning()
```

*   **`// Update member role`**: Comment for the update operation.
*   **`const updatedMember = await db .update(member) .set({ role }) .where(and(eq(member.organizationId, organizationId), eq(member.userId, memberId))) .returning()`**: This is a Drizzle ORM update query.
    *   `db.update(member)`: Specifies that we want to update the `member` table.
    *   `.set({ role })`: Sets the `role` column to the new `role` value received from the request body.
    *   `.where(and(eq(member.organizationId, organizationId), eq(member.userId, memberId)))`: Ensures that only the specific target member within the specified organization is updated.
    *   `.returning()`: Tells Drizzle to return the updated rows. Without this, it might only return the number of affected rows.

```typescript
    if (updatedMember.length === 0) {
      return NextResponse.json({ error: 'Failed to update member role' }, { status: 500 })
    }
```

*   **`if (updatedMember.length === 0)`**: If for some reason the update operation didn't return any updated rows (which shouldn't happen if the `targetMember` check passed), it indicates a failure.
*   **`return NextResponse.json(...)`**: Returns `500 Internal Server Error`.

```typescript
    logger.info('Organization member role updated', {
      organizationId,
      memberId,
      newRole: role,
      updatedBy: session.user.id,
    })
```

*   **`logger.info(...)`**: Logs a successful role update, including relevant context like the organization ID, the member whose role was changed, their new role, and who performed the update.

```typescript
    return NextResponse.json({
      success: true,
      message: 'Member role updated successfully',
      data: {
        id: updatedMember[0].id,
        userId: updatedMember[0].userId,
        role: updatedMember[0].role,
        updatedBy: session.user.id,
      },
    })
  } catch (error) {
    logger.error('Failed to update organization member role', {
      organizationId: (await params).id,
      memberId: (await params).memberId,
      error,
    })

    return NextResponse.json({ error: 'Internal server error' }, { status: 500 })
  }
}
```

*   **`return NextResponse.json({...})`**: Returns a success JSON response, confirming the update and including basic details of the updated member.
*   **`catch (error) { ... }`**: Standard error handling, similar to the `GET` method, logging the error and returning a generic `500` status.

---

### `DELETE` Endpoint: Removing Member from Organization

```typescript
/**
 * DELETE /api/organizations/[id]/members/[memberId]
 * Remove member from organization
 */
export async function DELETE(
  request: NextRequest,
  { params }: { params: Promise<{ id: string; memberId: string }> }
) {
  try {
    // ... (rest of the DELETE method)
  } catch (error) {
    // ... (error handling)
  }
}
```

*   **`/** ... */`**: JSDoc comment for the `DELETE` endpoint.
*   **`export async function DELETE(...)`**: Defines the asynchronous `DELETE` Route Handler, with the same `NextRequest` and `params` arguments. This function is designed to handle requests to remove a member.

#### `DELETE` Method - Core Logic

```typescript
  try {
    const session = await getSession()

    if (!session?.user?.id) {
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
    }
```

*   **`try { ... }`**: Standard error handling block.
*   **`const session = await getSession()`**: Fetches the authenticated user's session.
*   **`if (!session?.user?.id)`**: Unauthorized check, same as in `GET` and `PUT`.

```typescript
    const { id: organizationId, memberId } = await params
```

*   **`const { id: organizationId, memberId } = await params`**: Extracts `organizationId` and `memberId` from the URL parameters. `memberId` here is the ID of the user to be removed.

```typescript
    // Verify user has admin access
    const userMember = await db
      .select()
      .from(member)
      .where(and(eq(member.organizationId, organizationId), eq(member.userId, session.user.id)))
      .limit(1)

    if (userMember.length === 0) {
      return NextResponse.json(
        { error: 'Forbidden - Not a member of this organization' },
        { status: 403 }
      )
    }
```

*   **`// Verify user has admin access`**: Comment for permission check.
*   **`const userMember = await db.select()...limit(1)`**: Same query to verify the *requesting* user is a member of the organization.
*   **`if (userMember.length === 0)`**: If the requesting user isn't a member, returns `403 Forbidden`.

```typescript
    const canRemoveMembers =
      ['owner', 'admin'].includes(userMember[0].role) || session.user.id === memberId

    if (!canRemoveMembers) {
      return NextResponse.json({ error: 'Forbidden - Insufficient permissions' }, { status: 403 })
    }
```

*   **`const canRemoveMembers = ['owner', 'admin'].includes(userMember[0].role) || session.user.id === memberId`**: This determines if the requesting user has permission to remove a member.
    *   `['owner', 'admin'].includes(userMember[0].role)`: Owners or Admins can remove *any* member.
    *   `session.user.id === memberId`: A user can always remove *themselves* from an organization.
    *   The `||` operator means either condition grants permission.
*   **`if (!canRemoveMembers)`**: If the requesting user lacks permission, returns `403 Forbidden`.

```typescript
    // Check if target member exists
    const targetMember = await db
      .select()
      .from(member)
      .where(and(eq(member.organizationId, organizationId), eq(member.userId, memberId)))
      .limit(1)

    if (targetMember.length === 0) {
      return NextResponse.json({ error: 'Member not found' }, { status: 404 })
    }
```

*   **`// Check if target member exists`**: Comment.
*   **`const targetMember = await db.select()...limit(1)`**: Queries to ensure the `memberId` (the user to be removed) actually exists within the `organizationId`.
*   **`if (targetMember.length === 0)`**: If not found, returns `404 Not Found`.

```typescript
    // Prevent removing the owner
    if (targetMember[0].role === 'owner') {
      return NextResponse.json({ error: 'Cannot remove organization owner' }, { status: 400 })
    }
```

*   **`// Prevent removing the owner`**: Comment for owner protection.
*   **`if (targetMember[0].role === 'owner')`**: Critical rule: the 'owner' of an organization cannot be removed via this endpoint. This prevents accidental deletion of the organization's primary administrator.
*   **`return NextResponse.json(...)`**: Returns `400 Bad Request` if an attempt is made to remove the owner.

```typescript
    // Remove member
    const removedMember = await db
      .delete(member)
      .where(and(eq(member.organizationId, organizationId), eq(member.userId, memberId)))
      .returning()

    if (removedMember.length === 0) {
      return NextResponse.json({ error: 'Failed to remove member' }, { status: 500 })
    }
```

*   **`// Remove member`**: Comment for the delete operation.
*   **`const removedMember = await db .delete(member) .where(and(eq(member.organizationId, organizationId), eq(member.userId, memberId))) .returning()`**: This Drizzle query deletes the specified member record.
    *   `db.delete(member)`: Specifies deletion from the `member` table.
    *   `.where(...)`: Ensures only the correct record is deleted.
    *   `.returning()`: Returns the deleted record(s), which allows verifying the operation.
*   **`if (removedMember.length === 0)`**: If no records were returned (meaning no record was deleted), returns `500 Internal Server Error`.

```typescript
    logger.info('Organization member removed', {
      organizationId,
      removedMemberId: memberId,
      removedBy: session.user.id,
      wasSelfRemoval: session.user.id === memberId,
    })
```

*   **`logger.info(...)`**: Logs the successful member removal with contextual information.

#### `DELETE` Method - Post-Removal Billing Logic (Complex Part)

This section handles a specific billing edge case: if a user had a personal 'Pro' subscription set to cancel (because they were on a paid team plan) and they now leave their *last* paid team, their personal 'Pro' plan's cancellation should be reverted.

```typescript
    // If the removed user left their last paid team and has a personal Pro set to cancel_at_period_end, restore it
    try {
      const remainingPaidTeams = await db
        .select({ orgId: member.organizationId })
        .from(member)
        .where(eq(member.userId, memberId))
```

*   **`// If the removed user left their last paid team...`**: Explanatory comment for this complex block.
*   **`try { ... }`**: This entire post-removal logic is wrapped in its own `try...catch` block. This is a good practice, as failures in this secondary, non-critical operation should not prevent the primary member removal from succeeding (though they should be logged).
*   **`const remainingPaidTeams = await db .select({ orgId: member.organizationId }) .from(member) .where(eq(member.userId, memberId))`**: This query finds all other organizations that the *removed user* (`memberId`) is still a part of. It selects just the `organizationId` for these.

```typescript
      let hasAnyPaidTeam = false
      if (remainingPaidTeams.length > 0) {
        const orgIds = remainingPaidTeams.map((m) => m.orgId)
        const orgPaidSubs = await db
          .select()
          .from(subscriptionTable)
          .where(and(eq(subscriptionTable.status, 'active'), eq(subscriptionTable.plan, 'team')))

        hasAnyPaidTeam = orgPaidSubs.some((s) => orgIds.includes(s.referenceId))
      }
```

*   **`let hasAnyPaidTeam = false`**: Initializes a flag.
*   **`if (remainingPaidTeams.length > 0)`**: Only proceeds if the removed user is still a member of other organizations.
*   **`const orgIds = remainingPaidTeams.map((m) => m.orgId)`**: Extracts all `organizationId`s from the `remainingPaidTeams` results into an array.
*   **`const orgPaidSubs = await db .select() .from(subscriptionTable) .where(and(eq(subscriptionTable.status, 'active'), eq(subscriptionTable.plan, 'team')))`**: This query fetches *all* active 'team' plan subscriptions from the `subscriptionTable`.
*   **`hasAnyPaidTeam = orgPaidSubs.some((s) => orgIds.includes(s.referenceId))`**: This checks if *any* of the `organizationId`s the user is still a member of (`orgIds`) match the `referenceId` of an `active` 'team' subscription (`orgPaidSubs`). In simpler terms: is the user still part of *any* organization that has an active 'team' subscription?

```typescript
      if (!hasAnyPaidTeam) {
        const personalProRows = await db
          .select()
          .from(subscriptionTable)
          .where(
            and(
              eq(subscriptionTable.referenceId, memberId),
              eq(subscriptionTable.status, 'active'),
              eq(subscriptionTable.plan, 'pro')
            )
          )
          .limit(1)

        const personalPro = personalProRows[0]
        if (
          personalPro &&
          personalPro.cancelAtPeriodEnd === true &&
          personalPro.stripeSubscriptionId
        ) {
          // ... (Stripe and DB updates for personal Pro)
        }
      }
```

*   **`if (!hasAnyPaidTeam)`**: This block executes only if the user has indeed left their *last* paid team (or was never on one). This is the trigger for checking for personal Pro restoration.
*   **`const personalProRows = await db .select() .from(subscriptionTable) .where(and(eq(subscriptionTable.referenceId, memberId), eq(subscriptionTable.status, 'active'), eq(subscriptionTable.plan, 'pro'))) .limit(1)`**: This query looks for an active personal 'Pro' subscription for the `memberId`. `referenceId` likely refers to the `userId` for personal subscriptions.
*   **`const personalPro = personalProRows[0]`**: Gets the first (and only) personal Pro subscription if found.
*   **`if (personalPro && personalPro.cancelAtPeriodEnd === true && personalPro.stripeSubscriptionId)`**: Checks if:
    *   A personal Pro subscription exists.
    *   It's currently set to `cancelAtPeriodEnd: true` (meaning it's scheduled for cancellation).
    *   It has a `stripeSubscriptionId` (necessary for Stripe API interaction).

```typescript
          try {
            const stripe = requireStripeClient()
            await stripe.subscriptions.update(personalPro.stripeSubscriptionId, {
              cancel_at_period_end: false,
            })
          } catch (stripeError) {
            logger.error('Stripe restore cancel_at_period_end failed for personal Pro', {
              userId: memberId,
              stripeSubscriptionId: personalPro.stripeSubscriptionId,
              error: stripeError,
            })
          }
```

*   **`try { ... } catch (stripeError) { ... }`**: Another nested `try...catch` specifically for the Stripe API call, ensuring that a Stripe error doesn't halt the entire process.
*   **`const stripe = requireStripeClient()`**: Obtains the Stripe client.
*   **`await stripe.subscriptions.update(...)`**: Calls the Stripe API to update the subscription. `cancel_at_period_end: false` reverts the cancellation.
*   **`logger.error(...)`**: Logs any error from the Stripe API call.

```typescript
          try {
            await db
              .update(subscriptionTable)
              .set({ cancelAtPeriodEnd: false })
              .where(eq(subscriptionTable.id, personalPro.id))

            logger.info('Restored personal Pro after leaving last paid team', {
              userId: memberId,
              personalSubscriptionId: personalPro.id,
            })
          } catch (dbError) {
            logger.error('DB update failed when restoring personal Pro', {
              userId: memberId,
              subscriptionId: personalPro.id,
              error: dbError,
            })
          }
```

*   **`try { ... } catch (dbError) { ... }`**: Another nested `try...catch` for the database update, logging any errors.
*   **`await db .update(subscriptionTable) .set({ cancelAtPeriodEnd: false }) .where(eq(subscriptionTable.id, personalPro.id))`**: Updates the local database to reflect the change made in Stripe, setting `cancelAtPeriodEnd` to `false` for the personal Pro subscription.
*   **`logger.info(...)`**: Logs the successful database update.
*   **`logger.error(...)`**: Logs any error from the database update.

```typescript
          // Also restore the snapshotted Pro usage back to currentPeriodCost
          try {
            const userStatsRows = await db
              .select({
                currentPeriodCost: userStats.currentPeriodCost,
                proPeriodCostSnapshot: userStats.proPeriodCostSnapshot,
              })
              .from(userStats)
              .where(eq(userStats.userId, memberId))
              .limit(1)

            if (userStatsRows.length > 0) {
              const currentUsage = userStatsRows[0].currentPeriodCost || '0'
              const snapshotUsage = userStatsRows[0].proPeriodCostSnapshot || '0'

              const currentNum = Number.parseFloat(currentUsage)
              const snapshotNum = Number.parseFloat(snapshotUsage)
              const restoredUsage = (currentNum + snapshotNum).toString()

              await db
                .update(userStats)
                .set({
                  currentPeriodCost: restoredUsage,
                  proPeriodCostSnapshot: '0', // Clear the snapshot
                })
                .where(eq(userStats.userId, memberId))

              logger.info('Restored Pro usage after leaving team', {
                userId: memberId,
                previousUsage: currentUsage,
                snapshotUsage: snapshotUsage,
                restoredUsage: restoredUsage,
              })
            }
          } catch (usageRestoreError) {
            logger.error('Failed to restore Pro usage after leaving team', {
              userId: memberId,
              error: usageRestoreError,
            })
          }
        }
      }
    } catch (postRemoveError) {
      logger.error('Post-removal personal Pro restore check failed', {
        organizationId,
        memberId,
        error: postRemoveError,
      })
    }
```

*   **`// Also restore the snapshotted Pro usage...`**: Comment for usage restoration.
*   **`try { ... } catch (usageRestoreError) { ... }`**: Another nested `try...catch` for the usage restoration logic.
*   **`const userStatsRows = await db .select({...}) .from(userStats) .where(eq(userStats.userId, memberId)) .limit(1)`**: Fetches the `currentPeriodCost` (current usage) and `proPeriodCostSnapshot` (usage snapshot, likely taken when the user joined a paid team and their personal Pro was paused) from `userStats`.
*   **`if (userStatsRows.length > 0)`**: Checks if user stats exist.
*   **`const currentUsage = ...`, `const snapshotUsage = ...`**: Retrieves the usage values, defaulting to '0' if null/undefined.
*   **`const currentNum = Number.parseFloat(currentUsage)`, `const snapshotNum = Number.parseFloat(snapshotUsage)`**: Converts the string usage values to numbers for calculation.
*   **`const restoredUsage = (currentNum + snapshotNum).toString()`**: Calculates the `restoredUsage` by adding the current usage and the snapshotted usage, then converts it back to a string (as it's likely stored as a string in the DB). This implies the `proPeriodCostSnapshot` was usage *that would have been billed* to the personal Pro plan but was paused when the user was on a team.
*   **`await db .update(userStats) .set({ currentPeriodCost: restoredUsage, proPeriodCostSnapshot: '0', // Clear the snapshot }) .where(eq(userStats.userId, memberId))`**: Updates the `userStats` table:
    *   Sets `currentPeriodCost` to the `restoredUsage`.
    *   Clears `proPeriodCostSnapshot` by setting it back to '0'.
*   **`logger.info(...)`**: Logs the successful usage restoration.
*   **`logger.error(...)`**: Logs any error from the usage restoration.
*   **`} catch (postRemoveError) { logger.error(...) }`**: This is the outer `catch` block for the entire post-removal billing logic, catching any errors that weren't caught by the more specific nested `try...catch` blocks within it.

```typescript
    return NextResponse.json({
      success: true,
      message:
        session.user.id === memberId
          ? 'You have left the organization'
          : 'Member removed successfully',
      data: {
        removedMemberId: memberId,
        removedBy: session.user.id,
        removedAt: new Date().toISOString(),
      },
    })
  } catch (error) {
    logger.error('Failed to remove organization member', {
      organizationId: (await params).id,
      memberId: (await params).memberId,
      error,
    })

    return NextResponse.json({ error: 'Internal server error' }, { status: 500 })
  }
}
```

*   **`return NextResponse.json({...})`**: Returns a success JSON response.
    *   `message`: A dynamic message; it tells the user they have left the organization if they removed themselves, otherwise it confirms a member was removed.
    *   `data`: Includes details about the removed member.
*   **`catch (error) { ... }`**: Standard error handling for the main `DELETE` operation, logging the error and returning a generic `500` status.

---

## Simplification of Complex Logic

The most complex part of this file is the post-removal billing logic in the `DELETE` method. Here's a simplified explanation:

**Scenario:** Imagine a user (`User A`) has a personal "Pro" subscription that's paused or set to cancel. This usually happens because they joined an organization that has a "Team" subscription, which covers their Pro features.

**When `User A` leaves an organization:**

1.  **Check if `User A` is still part of any *other* paid organizations:** The system first looks at all other organizations `User A` is still a member of. Then, for each of those organizations, it checks if they have an active "Team" subscription.
2.  **If `User A` is *not* part of any paid organization anymore:** This means they've now left their *last* paid team.
3.  **Find `User A`'s personal "Pro" plan:** The system then searches for `User A`'s own personal "Pro" subscription.
4.  **If `User A` has a personal "Pro" plan that was set to cancel:** This is the trigger condition.
    *   **Un-cancel the Stripe subscription:** The code reaches out to Stripe (the payment processor) and tells it to *stop* the scheduled cancellation of `User A`'s personal Pro plan.
    *   **Update the local database:** It then updates the `subscription` table in its own database to reflect that the personal Pro plan is no longer set to cancel.
    *   **Restore Usage Data:** If `User A`'s personal Pro usage was "snapshotted" (meaning, their individual usage tracking was paused while on a team plan, and any usage was recorded separately), that snapshotted usage is added back to their current usage. The snapshot is then cleared. This ensures their personal billing accurately reflects their usage now that they're no longer covered by a team plan.

**Why this complexity?**
This logic ensures a smooth billing experience. A user shouldn't lose access to their personal Pro features or have double-billing if they move between paid teams or decide to go back to a personal plan after leaving all teams. It gracefully handles the transition of a user's subscription status as their organization memberships change.

---

This detailed explanation covers the purpose, functionality, and line-by-line breakdown of the provided TypeScript code, aiming to be both comprehensive and easy to understand for a technical audience.