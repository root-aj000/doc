This TypeScript file defines a Next.js API route that handles operations related to organization invitations. It acts as an endpoint for retrieving details about an invitation (`GET` request) and for managing its status (accepting, rejecting, or canceling via `PUT` request).

The core functionality revolves around a user being invited to an organization, and then deciding to accept or reject that invitation. It also includes complex logic for handling existing subscriptions and linked workspace invitations when an invitation is accepted.

---

### **File Purpose and What it Does**

This file, likely residing at a path like `/api/organizations/[id]/invitations/[invitationId]`, serves as a backend API for managing a specific organization invitation.

1.  **`GET` Request Handler:**
    *   Retrieves the details of a particular organization invitation and the associated organization.
    *   Requires the user to be authenticated.
    *   Returns the invitation and organization data if found, or appropriate error messages if not.

2.  **`PUT` Request Handler:**
    *   Allows a user to update the status of an invitation (e.g., 'accepted', 'rejected', 'cancelled').
    *   **Acceptance Flow:**
        *   If the invitation is accepted, it adds the user as a member to the organization.
        *   It includes sophisticated logic to manage a user's *personal Pro subscription* if they are joining a *paid organization team*: it "snapshots" their personal Pro usage and marks their personal Pro subscription for cancellation at the end of the current billing period to avoid double billing or redundant subscriptions.
        *   It also automatically accepts any linked workspace invitations and grants the user permissions to those workspaces.
    *   **Cancellation Flow:**
        *   Allows an organization administrator to cancel a pending invitation.
        *   Also cancels any linked pending workspace invitations.
    *   **Rejection Flow:**
        *   Updates the invitation status to 'rejected'.
    *   Includes numerous validation steps (e.g., user authentication, invitation status, user email matching, admin permissions for cancellation, preventing a user from joining multiple organizations).
    *   Leverages database transactions to ensure data consistency during complex updates.

---

### **Detailed Line-by-Line Explanation**

#### **Imports**

```typescript
import { randomUUID } from 'crypto'
import { db } from '@sim/db'
import {
  invitation,
  member,
  organization,
  permissions,
  subscription as subscriptionTable,
  user,
  userStats,
  type WorkspaceInvitationStatus,
  workspaceInvitation,
} from '@sim/db/schema'
import { and, eq } from 'drizzle-orm'
import { type NextRequest, NextResponse } from 'next/server'
import { getSession } from '@/lib/auth'
import { requireStripeClient } from '@/lib/billing/stripe-client'
import { createLogger } from '@/lib/logs/console/logger'
```

*   `import { randomUUID } from 'crypto'`: Imports the `randomUUID` function from Node.js's built-in `crypto` module. This is used to generate universally unique identifiers (UUIDs) for new database records (e.g., for `member` or `permissions`).
*   `import { db } from '@sim/db'`: Imports the Drizzle ORM database client instance. `@sim/db` is likely an alias pointing to the project's database configuration file. This `db` object is used to interact with the database.
*   `import { ... } from '@sim/db/schema'`: Imports various Drizzle ORM schema definitions. These define the structure of your database tables and allow you to interact with them in a type-safe way:
    *   `invitation`: Represents the table for organization invitations.
    *   `member`: Represents the table for organization members.
    *   `organization`: Represents the table for organizations.
    *   `permissions`: Represents the table for user permissions on specific entities (like workspaces).
    *   `subscription as subscriptionTable`: Imports the `subscription` table schema, aliased as `subscriptionTable` to avoid naming conflicts if another `subscription` variable is used. This table stores details about user or organization subscriptions.
    *   `user`: Represents the table for user accounts.
    *   `userStats`: Represents the table for user-specific statistics or usage data, often related to billing.
    *   `type WorkspaceInvitationStatus`: Imports a TypeScript type definition for the status of a workspace invitation (e.g., 'pending', 'accepted', 'rejected').
    *   `workspaceInvitation`: Represents the table for invitations to specific workspaces (which are often nested within organizations).
*   `import { and, eq } from 'drizzle-orm'`: Imports Drizzle ORM helper functions:
    *   `and`: Used to combine multiple conditions with a logical AND operator in `WHERE` clauses.
    *   `eq`: Used to create an equality condition (e.g., `column = value`).
*   `import { type NextRequest, NextResponse } from 'next/server'`: Imports types and classes from Next.js for handling API requests and responses:
    *   `NextRequest`: Represents the incoming HTTP request.
    *   `NextResponse`: Used to send HTTP responses back to the client.
*   `import { getSession } from '@/lib/auth'`: Imports a utility function to retrieve the user's session information. This is crucial for authentication and authorization. `@/lib/auth` is likely an alias to a local library path.
*   `import { requireStripeClient } from '@/lib/billing/stripe-client'`: Imports a function to get a Stripe client instance. This is used for interacting with the Stripe API for billing operations (e.g., canceling subscriptions).
*   `import { createLogger } from '@/lib/logs/console/logger'`: Imports a function to create a logger instance for structured logging.

#### **Logger Initialization**

```typescript
const logger = createLogger('OrganizationInvitation')
```

*   `const logger = createLogger('OrganizationInvitation')`: Initializes a logger instance specifically for this module, naming it 'OrganizationInvitation'. This helps in identifying the source of log messages.

---

#### **GET Invitation Details Handler**

This function handles HTTP GET requests to retrieve invitation information.

```typescript
export async function GET(
  _req: NextRequest,
  { params }: { params: Promise<{ id: string; invitationId: string }> }
) {
  const { id: organizationId, invitationId } = await params
  const session = await getSession()

  if (!session?.user?.id) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
  }

  try {
    const orgInvitation = await db
      .select()
      .from(invitation)
      .where(and(eq(invitation.id, invitationId), eq(invitation.organizationId, organizationId)))
      .then((rows) => rows[0])

    if (!orgInvitation) {
      return NextResponse.json({ error: 'Invitation not found' }, { status: 404 })
    }

    const org = await db
      .select()
      .from(organization)
      .where(eq(organization.id, organizationId))
      .then((rows) => rows[0])

    if (!org) {
      return NextResponse.json({ error: 'Organization not found' }, { status: 404 })
    }

    return NextResponse.json({
      invitation: orgInvitation,
      organization: org,
    })
  } catch (error) {
    logger.error('Error fetching organization invitation:', error)
    return NextResponse.json({ error: 'Failed to fetch invitation' }, { status: 500 })
  }
}
```

*   `export async function GET(...)`: Defines an asynchronous function `GET` which will be automatically invoked by Next.js for incoming GET requests to this route.
    *   `_req: NextRequest`: The incoming request object. It's prefixed with `_` to indicate it's not directly used in this function's logic.
    *   `{ params }: { params: Promise<{ id: string; invitationId: string }> }`: This object contains the dynamic route parameters. `id` refers to the `organizationId` and `invitationId` refers to the `invitationId` from the URL path (e.g., `/api/organizations/org123/invitations/inv456`). Next.js provides these parameters as a `Promise`, so we `await` them.
*   `const { id: organizationId, invitationId } = await params`: Destructures the `params` object, extracting `id` and aliasing it to `organizationId`, and extracting `invitationId`.
*   `const session = await getSession()`: Retrieves the current user's session.
*   `if (!session?.user?.id)`: Checks if there is an active session and if a user ID is present within that session.
*   `return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })`: If no authenticated user, returns an "Unauthorized" error with a 401 HTTP status code.
*   `try { ... } catch (error) { ... }`: A `try...catch` block to gracefully handle any errors that occur during database operations or other logic.
*   `const orgInvitation = await db.select().from(invitation).where(and(eq(invitation.id, invitationId), eq(invitation.organizationId, organizationId))).then((rows) => rows[0])`:
    *   This is a Drizzle ORM query to fetch a specific invitation.
    *   `db.select()`: Starts a select query. By default, it selects all columns.
    *   `.from(invitation)`: Specifies the `invitation` table.
    *   `.where(and(eq(invitation.id, invitationId), eq(invitation.organizationId, organizationId)))`: Filters the invitations where the `id` matches `invitationId` AND the `organizationId` matches the one from the URL.
    *   `.then((rows) => rows[0])`: Drizzle returns an array of rows. We expect at most one, so we take the first element (`rows[0]`).
*   `if (!orgInvitation)`: Checks if an invitation was found.
*   `return NextResponse.json({ error: 'Invitation not found' }, { status: 404 })`: If no invitation is found with the given IDs, returns a "Not found" error with a 404 status.
*   `const org = await db.select().from(organization).where(eq(organization.id, organizationId)).then((rows) => rows[0])`:
    *   Similar to the invitation query, this fetches the `organization` details using its `organizationId`.
*   `if (!org)`: Checks if the organization was found.
*   `return NextResponse.json({ error: 'Organization not found' }, { status: 404 })`: If the organization is not found, returns a 404 error.
*   `return NextResponse.json({ invitation: orgInvitation, organization: org, })`: If both invitation and organization are found, returns them as a JSON object with a 200 (OK) status.
*   `logger.error('Error fetching organization invitation:', error)`: Logs any unexpected errors caught during the process.
*   `return NextResponse.json({ error: 'Failed to fetch invitation' }, { status: 500 })`: Returns a generic "Failed to fetch invitation" error with a 500 (Internal Server Error) status for any unhandled exceptions.

---

#### **PUT Invitation Status Handler**

This function handles HTTP PUT requests to update an invitation's status.

```typescript
export async function PUT(
  req: NextRequest,
  { params }: { params: Promise<{ id: string; invitationId: string }> }
) {
  const { id: organizationId, invitationId } = await params

  logger.info(
    '[PUT /api/organizations/[id]/invitations/[invitationId]] Invitation acceptance request',
    {
      organizationId,
      invitationId,
      path: req.url,
    }
  )

  const session = await getSession()

  if (!session?.user?.id) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
  }

  try {
    const { status } = await req.json()

    if (!status || !['accepted', 'rejected', 'cancelled'].includes(status)) {
      return NextResponse.json(
        { error: 'Invalid status. Must be "accepted", "rejected", or "cancelled"' },
        { status: 400 }
      )
    }

    const orgInvitation = await db
      .select()
      .from(invitation)
      .where(and(eq(invitation.id, invitationId), eq(invitation.organizationId, organizationId)))
      .then((rows) => rows[0])

    if (!orgInvitation) {
      return NextResponse.json({ error: 'Invitation not found' }, { status: 404 })
    }

    if (orgInvitation.status !== 'pending') {
      return NextResponse.json({ error: 'Invitation already processed' }, { status: 400 })
    }

    if (status === 'accepted') {
      const userData = await db
        .select()
        .from(user)
        .where(eq(user.id, session.user.id))
        .then((rows) => rows[0])

      if (!userData || userData.email.toLowerCase() !== orgInvitation.email.toLowerCase()) {
        return NextResponse.json(
          { error: 'Email mismatch. You can only accept invitations sent to your email address.' },
          { status: 403 }
        )
      }
    }

    if (status === 'cancelled') {
      const isAdmin = await db
        .select()
        .from(member)
        .where(
          and(
            eq(member.organizationId, organizationId),
            eq(member.userId, session.user.id),
            eq(member.role, 'admin')
          )
        )
        .then((rows) => rows.length > 0)

      if (!isAdmin) {
        return NextResponse.json(
          { error: 'Only organization admins can cancel invitations' },
          { status: 403 }
        )
      }
    }

    // Enforce: user can only be part of a single organization
    if (status === 'accepted') {
      // Check if user is already a member of ANY organization
      const existingOrgMemberships = await db
        .select({ organizationId: member.organizationId })
        .from(member)
        .where(eq(member.userId, session.user.id))

      if (existingOrgMemberships.length > 0) {
        // Check if already a member of THIS specific organization
        const alreadyMemberOfThisOrg = existingOrgMemberships.some(
          (m) => m.organizationId === organizationId
        )

        if (alreadyMemberOfThisOrg) {
          return NextResponse.json(
            { error: 'You are already a member of this organization' },
            { status: 400 }
          )
        }

        // Member of a different organization
        // Mark the invitation as rejected since they can't accept it
        await db
          .update(invitation)
          .set({
            status: 'rejected',
          })
          .where(eq(invitation.id, invitationId))

        return NextResponse.json(
          {
            error:
              'You are already a member of an organization. Leave your current organization before accepting a new invitation.',
          },
          { status: 409 }
        )
      }
    }

    let personalProToCancel: any = null

    await db.transaction(async (tx) => {
      await tx.update(invitation).set({ status }).where(eq(invitation.id, invitationId))

      if (status === 'accepted') {
        await tx.insert(member).values({
          id: randomUUID(),
          userId: session.user.id,
          organizationId,
          role: orgInvitation.role,
          createdAt: new Date(),
        })

        // Snapshot Pro usage and cancel Pro subscription when joining a paid team
        try {
          const orgSubs = await tx
            .select()
            .from(subscriptionTable)
            .where(
              and(
                eq(subscriptionTable.referenceId, organizationId),
                eq(subscriptionTable.status, 'active')
              )
            )
            .limit(1)

          const orgSub = orgSubs[0]
          const orgIsPaid = orgSub && (orgSub.plan === 'team' || orgSub.plan === 'enterprise')

          if (orgIsPaid) {
            const userId = session.user.id

            // Find user's active personal Pro subscription
            const personalSubs = await tx
              .select()
              .from(subscriptionTable)
              .where(
                and(
                  eq(subscriptionTable.referenceId, userId),
                  eq(subscriptionTable.status, 'active'),
                  eq(subscriptionTable.plan, 'pro')
                )
              )
              .limit(1)

            const personalPro = personalSubs[0]
            if (personalPro) {
              // Snapshot the current Pro usage before resetting
              const userStatsRows = await tx
                .select({
                  currentPeriodCost: userStats.currentPeriodCost,
                })
                .from(userStats)
                .where(eq(userStats.userId, userId))
                .limit(1)

              if (userStatsRows.length > 0) {
                const currentProUsage = userStatsRows[0].currentPeriodCost || '0'

                // Snapshot Pro usage and reset currentPeriodCost so new usage goes to team
                await tx
                  .update(userStats)
                  .set({
                    proPeriodCostSnapshot: currentProUsage,
                    currentPeriodCost: '0', // Reset so new usage is attributed to team
                  })
                  .where(eq(userStats.userId, userId))

                logger.info('Snapshotted Pro usage when joining team', {
                  userId,
                  proUsageSnapshot: currentProUsage,
                  organizationId,
                })
              }

              // Mark for cancellation after transaction
              if (personalPro.cancelAtPeriodEnd !== true) {
                personalProToCancel = personalPro
              }
            }
          }
        } catch (error) {
          logger.error('Failed to handle Pro user joining team', {
            userId: session.user.id,
            organizationId,
            error,
          })
          // Don't fail the whole invitation acceptance due to this
        }

        const linkedWorkspaceInvitations = await tx
          .select()
          .from(workspaceInvitation)
          .where(
            and(
              eq(workspaceInvitation.orgInvitationId, invitationId),
              eq(workspaceInvitation.status, 'pending' as WorkspaceInvitationStatus)
            )
          )

        for (const wsInvitation of linkedWorkspaceInvitations) {
          await tx
            .update(workspaceInvitation)
            .set({
              status: 'accepted' as WorkspaceInvitationStatus,
              updatedAt: new Date(),
            })
            .where(eq(workspaceInvitation.id, wsInvitation.id))

          await tx.insert(permissions).values({
            id: randomUUID(),
            entityType: 'workspace',
            entityId: wsInvitation.workspaceId,
            userId: session.user.id,
            permissionType: wsInvitation.permissions || 'read',
            createdAt: new Date(),
            updatedAt: new Date(),
          })
        }
      } else if (status === 'cancelled') {
        await tx
          .update(workspaceInvitation)
          .set({ status: 'cancelled' as WorkspaceInvitationStatus })
          .where(eq(workspaceInvitation.orgInvitationId, invitationId))
      }
    })

    // Handle Pro subscription cancellation after transaction commits
    if (personalProToCancel) {
      try {
        const stripe = requireStripeClient()
        if (personalProToCancel.stripeSubscriptionId) {
          try {
            await stripe.subscriptions.update(personalProToCancel.stripeSubscriptionId, {
              cancel_at_period_end: true,
            })
          } catch (stripeError) {
            logger.error('Failed to set cancel_at_period_end on Stripe for personal Pro', {
              userId: session.user.id,
              subscriptionId: personalProToCancel.id,
              stripeSubscriptionId: personalProToCancel.stripeSubscriptionId,
              error: stripeError,
            })
          }
        }

        await db
          .update(subscriptionTable)
          .set({ cancelAtPeriodEnd: true })
          .where(eq(subscriptionTable.id, personalProToCancel.id))

        logger.info('Auto-cancelled personal Pro at period end after joining paid team', {
          userId: session.user.id,
          personalSubscriptionId: personalProToCancel.id,
          organizationId,
        })
      } catch (dbError) {
        logger.error('Failed to update DB cancelAtPeriodEnd for personal Pro', {
          userId: session.user.id,
          subscriptionId: personalProToCancel.id,
          error: dbError,
        })
      }
    }

    logger.info(`Organization invitation ${status}`, {
      organizationId,
      invitationId,
      userId: session.user.id,
      email: orgInvitation.email,
    })

    return NextResponse.json({
      success: true,
      message: `Invitation ${status} successfully`,
      invitation: { ...orgInvitation, status },
    })
  } catch (error) {
    logger.error(`Error updating organization invitation:`, error)
    return NextResponse.json({ error: 'Failed to update invitation' }, { status: 500 })
  }
}
```

*   `export async function PUT(...)`: Defines the asynchronous `PUT` handler. It takes the `NextRequest` and dynamic `params` similar to the `GET` request.
*   `const { id: organizationId, invitationId } = await params`: Extracts `organizationId` and `invitationId` from the URL parameters.
*   `logger.info(...)`: Logs the incoming PUT request with relevant details for debugging and monitoring.
*   `const session = await getSession()`: Retrieves the user session.
*   `if (!session?.user?.id)`: Checks for user authentication; returns 401 Unauthorized if not authenticated.
*   `try { ... } catch (error) { ... }`: Main error handling block for the `PUT` request.
*   `const { status } = await req.json()`: Parses the request body as JSON and extracts the `status` property. This `status` is what the invitation will be updated to (e.g., 'accepted', 'rejected', 'cancelled').
*   `if (!status || !['accepted', 'rejected', 'cancelled'].includes(status))`: Validates the `status` provided in the request body. If it's missing or not one of the allowed values, returns a 400 Bad Request error.
*   `const orgInvitation = await db.select().from(invitation).where(and(eq(invitation.id, invitationId), eq(invitation.organizationId, organizationId))).then((rows) => rows[0])`: Fetches the invitation record from the database, similar to the `GET` request.
*   `if (!orgInvitation)`: Returns 404 Not Found if the invitation doesn't exist.
*   `if (orgInvitation.status !== 'pending')`: Ensures that only invitations with a 'pending' status can be modified. If it's already accepted, rejected, or cancelled, it returns a 400 Bad Request error to prevent reprocessing.
*   `if (status === 'accepted') { ... }`: **Logic specific to accepting an invitation.**
    *   `const userData = await db.select().from(user).where(eq(user.id, session.user.id)).then((rows) => rows[0])`: Fetches the current user's data.
    *   `if (!userData || userData.email.toLowerCase() !== orgInvitation.email.toLowerCase())`: **Email Mismatch Check**: Verifies that the email of the currently logged-in user matches the email address to which the invitation was sent (case-insensitive). This prevents users from accepting invitations intended for others. Returns a 403 Forbidden error if there's a mismatch.
*   `if (status === 'cancelled') { ... }`: **Logic specific to canceling an invitation.**
    *   `const isAdmin = await db.select().from(member).where(and(eq(member.organizationId, organizationId), eq(member.userId, session.user.id), eq(member.role, 'admin'))).then((rows) => rows.length > 0)`: Checks if the currently logged-in user is an 'admin' within the specified organization.
    *   `if (!isAdmin)`: If the user is not an admin, returns a 403 Forbidden error, as only admins can cancel invitations.
*   `// Enforce: user can only be part of a single organization`: This comment highlights a crucial business rule.
*   `if (status === 'accepted') { ... }`: **Single Organization Membership Enforcement (if accepting):**
    *   `const existingOrgMemberships = await db.select({ organizationId: member.organizationId }).from(member).where(eq(member.userId, session.user.id))`: Queries for any existing organization memberships for the current user.
    *   `if (existingOrgMemberships.length > 0)`: If the user is already a member of *any* organization:
        *   `const alreadyMemberOfThisOrg = existingOrgMemberships.some((m) => m.organizationId === organizationId)`: Checks if the user is *already a member of this specific organization*.
        *   `if (alreadyMemberOfThisOrg)`: If they are already a member of this organization, returns a 400 Bad Request.
        *   `await db.update(invitation).set({ status: 'rejected', }).where(eq(invitation.id, invitationId))`: If the user is a member of a *different* organization, the current invitation is automatically marked as 'rejected'. This is a design choice to prevent users from accepting an invitation they cannot fulfill due to the single-organization policy.
        *   `return NextResponse.json({ error: 'You are already a member of an organization...' }, { status: 409 })`: Returns a 409 Conflict error, explaining that the user must leave their current organization first.
*   `let personalProToCancel: any = null`: Initializes a variable to store personal Pro subscription data if it needs to be cancelled later. This is `any` type to allow flexibility as it will store a Drizzle row result.
*   `await db.transaction(async (tx) => { ... })`: **Database Transaction Block**: This is critical for ensuring data consistency. All database operations within this block are treated as a single, atomic unit. If any operation fails, all changes within the transaction are rolled back. `tx` is the transaction-specific database client.
    *   `await tx.update(invitation).set({ status }).where(eq(invitation.id, invitationId))`: Updates the main `invitation` status to the requested `status` ('accepted', 'rejected', or 'cancelled').
    *   `if (status === 'accepted') { ... }`: **Actions upon invitation acceptance within the transaction:**
        *   `await tx.insert(member).values(...)`: Inserts a new record into the `member` table, officially making the user a member of the organization.
            *   `id: randomUUID()`: Generates a unique ID for the new member record.
            *   `userId: session.user.id`: Links to the accepting user.
            *   `organizationId`: Links to the organization.
            *   `role: orgInvitation.role`: Assigns the role specified in the original invitation.
            *   `createdAt: new Date()`: Sets the creation timestamp.
        *   `// Snapshot Pro usage and cancel Pro subscription when joining a paid team`: Comment explaining the complex Pro subscription logic.
        *   `try { ... } catch (error) { ... }`: Inner try-catch specifically for the Pro subscription logic, to ensure this complex part doesn't fail the entire transaction if it encounters an issue.
            *   `const orgSubs = await tx.select().from(subscriptionTable).where(and(eq(subscriptionTable.referenceId, organizationId), eq(subscriptionTable.status, 'active'))).limit(1)`: Checks if the organization itself has an active subscription.
            *   `const orgSub = orgSubs[0]`: Gets the first active organization subscription.
            *   `const orgIsPaid = orgSub && (orgSub.plan === 'team' || orgSub.plan === 'enterprise')`: Determines if the organization's subscription is a paid "team" or "enterprise" plan.
            *   `if (orgIsPaid) { ... }`: If the organization is paid:
                *   `const userId = session.user.id`: Gets the current user's ID.
                *   `const personalSubs = await tx.select().from(subscriptionTable).where(and(eq(subscriptionTable.referenceId, userId), eq(subscriptionTable.status, 'active'), eq(subscriptionTable.plan, 'pro'))).limit(1)`: Checks if the *current user* has an active 'pro' personal subscription.
                *   `const personalPro = personalSubs[0]`: Gets the user's personal Pro subscription if found.
                *   `if (personalPro) { ... }`: If the user has a personal Pro subscription:
                    *   `const userStatsRows = await tx.select({ currentPeriodCost: userStats.currentPeriodCost }).from(userStats).where(eq(userStats.userId, userId)).limit(1)`: Fetches the user's current usage statistics.
                    *   `if (userStatsRows.length > 0) { ... }`: If user stats exist:
                        *   `const currentProUsage = userStatsRows[0].currentPeriodCost || '0'`: Gets the `currentPeriodCost` (usage for the current billing period).
                        *   `await tx.update(userStats).set({ proPeriodCostSnapshot: currentProUsage, currentPeriodCost: '0', }).where(eq(userStats.userId, userId))`: **Usage Snapshot**: Updates `userStats` to store the `currentPeriodCost` in `proPeriodCostSnapshot` (to record personal usage before joining the team) and resets `currentPeriodCost` to '0' (so new usage is attributed to the team's subscription).
                        *   `logger.info('Snapshotted Pro usage when joining team', ...)`: Logs this important event.
                    *   `if (personalPro.cancelAtPeriodEnd !== true) { personalProToCancel = personalPro }`: If the personal Pro subscription is not *already* set to cancel at the period end, store its data in `personalProToCancel` for processing *after* the transaction. This prevents immediate cancellation and ensures the user gets their full current period.
            *   `logger.error('Failed to handle Pro user joining team', ...)`: Logs errors specific to the Pro user logic, but allows the main transaction to continue.
        *   `const linkedWorkspaceInvitations = await tx.select().from(workspaceInvitation).where(and(eq(workspaceInvitation.orgInvitationId, invitationId), eq(workspaceInvitation.status, 'pending' as WorkspaceInvitationStatus)))`: Finds any workspace invitations that were linked to this organization invitation and are still pending.
        *   `for (const wsInvitation of linkedWorkspaceInvitations) { ... }`: Iterates through each linked pending workspace invitation:
            *   `await tx.update(workspaceInvitation).set({ status: 'accepted' as WorkspaceInvitationStatus, updatedAt: new Date(), }).where(eq(workspaceInvitation.id, wsInvitation.id))`: Updates the workspace invitation status to 'accepted'.
            *   `await tx.insert(permissions).values(...)`: Inserts a new `permissions` record for the user, granting them the specified access to the workspace.
                *   `permissionType: wsInvitation.permissions || 'read'`: Uses the permission type specified in the workspace invitation, defaulting to 'read' if not set.
    *   `else if (status === 'cancelled') { ... }`: **Actions upon invitation cancellation within the transaction:**
        *   `await tx.update(workspaceInvitation).set({ status: 'cancelled' as WorkspaceInvitationStatus }).where(eq(workspaceInvitation.orgInvitationId, invitationId))`: Updates all linked workspace invitations to 'cancelled'.
*   `// Handle Pro subscription cancellation after transaction commits`: Comment indicating the next step.
*   `if (personalProToCancel) { ... }`: **Post-Transaction Pro Subscription Cancellation**: This block runs *only after* the database transaction has successfully committed. This is important because Stripe operations are external and shouldn't block the database transaction from completing.
    *   `try { ... } catch (dbError) { ... }`: Outer try-catch for Stripe/DB updates related to Pro cancellation.
    *   `const stripe = requireStripeClient()`: Gets the Stripe API client.
    *   `if (personalProToCancel.stripeSubscriptionId) { ... }`: If the personal Pro subscription has a Stripe ID:
        *   `try { await stripe.subscriptions.update(personalProToCancel.stripeSubscriptionId, { cancel_at_period_end: true, }) } catch (stripeError) { logger.error('Failed to set cancel_at_period_end on Stripe for personal Pro', ...) }`: Calls the Stripe API to set `cancel_at_period_end` to `true`. This means the subscription will run until the end of its current billing period and then not renew. Errors here are logged but don't prevent the local DB update.
    *   `await db.update(subscriptionTable).set({ cancelAtPeriodEnd: true }).where(eq(subscriptionTable.id, personalProToCancel.id))`: Updates the local `subscriptionTable` record to reflect that cancellation at period end is pending.
    *   `logger.info('Auto-cancelled personal Pro at period end after joining paid team', ...)`: Logs the successful auto-cancellation.
    *   `logger.error('Failed to update DB cancelAtPeriodEnd for personal Pro', ...)`: Logs errors if the local DB update fails.
*   `logger.info(...)`: Logs the final outcome of the invitation update.
*   `return NextResponse.json(...)`: Returns a success response, including the updated invitation data.
*   `logger.error(`Error updating organization invitation:`, error)`: Logs any unexpected errors caught by the main `PUT` request's `try...catch` block.
*   `return NextResponse.json({ error: 'Failed to update invitation' }, { status: 500 })`: Returns a generic 500 Internal Server Error for unhandled exceptions.

---

This file demonstrates robust API design principles, including:
*   **Authentication and Authorization:** Checking user session and roles.
*   **Input Validation:** Ensuring request data is valid.
*   **Idempotency:** Preventing reprocessing of already handled invitations.
*   **Data Integrity:** Using database transactions for complex, multi-step operations.
*   **Error Handling:** Gracefully managing various error scenarios and providing informative responses.
*   **Logging:** Detailed logging for monitoring and debugging.
*   **External Service Integration:** Handling interactions with third-party APIs (Stripe).
*   **Business Logic Complexity:** Managing nuanced scenarios like subscription changes and single-organization membership.