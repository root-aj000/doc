This TypeScript file defines a Next.js API route that manages organization invitations. It provides endpoints for fetching existing invitations, creating new ones (both single and in batches, potentially including workspace-specific invites), and canceling pending invitations.

It integrates with a Drizzle ORM database (`@sim/db`), handles user authentication and authorization, performs various validation checks (email format, seat availability, permissions), and sends invitation emails.

---

## Detailed Explanation

### 1. File Purpose and What It Does

This file (`/api/organizations/[id]/invitations.ts`) acts as a central API endpoint for managing invitations within a specific organization. It's designed to be part of a backend system that allows administrators of an organization to invite new users.

Here's a breakdown of its core functionalities:

*   **GET `/api/organizations/[id]/invitations`**: Retrieves a list of all pending and processed invitations for a given organization. It also includes details about the inviter.
*   **POST `/api/organizations/[id]/invitations`**: Creates one or more new invitations for an organization. This is the most complex endpoint, handling:
    *   Validation of email addresses and roles.
    *   Checks for existing members or pending invitations to avoid duplicates.
    *   Seat availability checks (a billing/subscription-related feature).
    *   Optional "batch" invitations, where multiple users can be invited at once.
    *   Optional "workspace invitations" within the batch, where invited users are also granted access to specific workspaces upon joining.
    *   Sending out email notifications to the invited users.
    *   An optional "validation-only" mode to test invites without sending them.
*   **DELETE `/api/organizations/[id]/invitations?invitationId=...`**: Cancels a specific pending invitation for an organization, also canceling any linked workspace invitations.

All endpoints require user authentication and specific authorization (the user must be an `owner` or `admin` of the organization) to perform these actions.

---

### 2. Code Breakdown

Let's go through the code line by line and simplify complex logic.

#### Imports

```typescript
import { randomUUID } from 'crypto'
import { db } from '@sim/db'
import {
  invitation,
  member,
  organization,
  user,
  type WorkspaceInvitationStatus,
  workspace,
  workspaceInvitation,
} from '@sim/db/schema'
import { and, eq, inArray, isNull, or } from 'drizzle-orm'
import { type NextRequest, NextResponse } from 'next/server'
import {
  getEmailSubject,
  renderBatchInvitationEmail,
  renderInvitationEmail,
}
import { getSession } from '@/lib/auth'
import {
  validateBulkInvitations,
  validateSeatAvailability,
} from '@/lib/billing/validation/seat-management'
import { sendEmail } from '@/lib/email/mailer'
import { quickValidateEmail } from '@/lib/email/validation'
import { createLogger } from '@/lib/logs/console/logger'
import { hasWorkspaceAdminAccess } from '@/lib/permissions/utils'
import { getBaseUrl } from '@/lib/urls/utils'
```

*   `import { randomUUID } from 'crypto'`: Imports the `randomUUID` function from Node.js's built-in `crypto` module. This is used to generate universally unique identifiers (UUIDs) for new invitations.
*   `import { db } from '@sim/db'`: Imports the database client instance, likely configured for Drizzle ORM. This `db` object is the primary way to interact with the database.
*   `import { invitation, member, organization, user, type WorkspaceInvitationStatus, workspace, workspaceInvitation } from '@sim/db/schema'`: Imports schema definitions from the application's Drizzle ORM schema file. These are TypeScript objects representing the database tables:
    *   `invitation`: Stores details about organization invitations.
    *   `member`: Stores details about users who are members of an organization and their roles.
    *   `organization`: Stores details about organizations.
    *   `user`: Stores details about platform users.
    *   `WorkspaceInvitationStatus`: A TypeScript type for the status of a workspace invitation (e.g., 'pending', 'accepted', 'cancelled').
    *   `workspace`: Stores details about workspaces (sub-units within an organization).
    *   `workspaceInvitation`: Stores details about invitations specifically for workspaces.
*   `import { and, eq, inArray, isNull, or } from 'drizzle-orm'`: Imports helper functions from Drizzle ORM for constructing database queries:
    *   `and`: Combines multiple conditions with a logical AND.
    *   `eq`: Checks for equality (e.g., `column = value`).
    *   `inArray`: Checks if a column's value is present in an array of values.
    *   `isNull`: Checks if a column's value is NULL.
    *   `or`: Combines multiple conditions with a logical OR.
*   `import { type NextRequest, NextResponse } from 'next/server'`: Imports types and classes from Next.js for handling API requests and responses.
    *   `NextRequest`: Represents an incoming HTTP request.
    *   `NextResponse`: Used to construct and send HTTP responses.
*   `import { getEmailSubject, renderBatchInvitationEmail, renderInvitationEmail } from '@/components/emails/render-email'`: Imports functions responsible for generating the HTML content of invitation emails.
    *   `getEmailSubject`: Returns the appropriate subject line for an email.
    *   `renderBatchInvitationEmail`: Renders a specific email template for multiple invitations or invitations that include workspace details.
    *   `renderInvitationEmail`: Renders a standard email template for a single organization invitation.
*   `import { getSession } from '@/lib/auth'`: Imports a utility function to retrieve the current user's session information (e.g., logged-in status, user ID).
*   `import { validateBulkInvitations, validateSeatAvailability } from '@/lib/billing/validation/seat-management'`: Imports functions related to billing and seat management:
    *   `validateBulkInvitations`: Performs a comprehensive validation check for multiple invitation emails.
    *   `validateSeatAvailability`: Checks if the organization has enough available seats in its subscription plan to accommodate new invitations.
*   `import { sendEmail } from '@/lib/email/mailer'`: Imports a utility function to send emails.
*   `import { quickValidateEmail } from '@/lib/email/validation'`: Imports a lightweight utility for basic email format validation.
*   `import { createLogger } from '@/lib/logs/console/logger'`: Imports a function to create a logger instance for structured logging.
*   `import { hasWorkspaceAdminAccess } from '@/lib/permissions/utils'`: Imports a utility function to check if a user has administrative access to a specific workspace.
*   `import { getBaseUrl } from '@/lib/urls/utils'`: Imports a utility function to get the base URL of the application, used for constructing invitation links.

---

#### Logger and Interface Definition

```typescript
const logger = createLogger('OrganizationInvitations')

interface WorkspaceInvitation {
  workspaceId: string
  permission: 'admin' | 'write' | 'read'
}
```

*   `const logger = createLogger('OrganizationInvitations')`: Initializes a logger instance named `OrganizationInvitations`. This logger will be used to record important events and errors within this file.
*   `interface WorkspaceInvitation { ... }`: Defines a TypeScript interface `WorkspaceInvitation`. This describes the expected structure of data when inviting users to specific workspaces in a batch.
    *   `workspaceId: string`: The unique identifier of the workspace.
    *   `permission: 'admin' | 'write' | 'read'`: The level of access the invited user will have in that workspace.

---

#### GET `/api/organizations/[id]/invitations`

This function handles `GET` requests to retrieve all pending and processed invitations for a given organization.

```typescript
/**
 * GET /api/organizations/[id]/invitations
 * Get all pending invitations for an organization
 */
export async function GET(request: NextRequest, { params }: { params: Promise<{ id: string }> }) {
  try {
    // 1. Authenticate User
    const session = await getSession() // Retrieves the current user's authentication session.

    if (!session?.user?.id) { // Checks if a user is logged in (session exists and has a user ID).
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 }) // If not logged in, returns a 401 Unauthorized response.
    }

    // 2. Extract Organization ID
    const { id: organizationId } = await params // Extracts the 'id' (organization ID) from the URL parameters.

    // 3. Authorize User (Membership Check)
    const memberEntry = await db // Starts a database query using the Drizzle client.
      .select() // Selects all columns from the table.
      .from(member) // Specifies the 'member' table.
      .where(and(eq(member.organizationId, organizationId), eq(member.userId, session.user.id))) // Filters members where organization ID matches and user ID matches the current user.
      .limit(1) // Limits the result to one entry, as a user can only be a member once per organization.

    if (memberEntry.length === 0) { // If no member entry is found, the user is not part of this organization.
      return NextResponse.json(
        { error: 'Forbidden - Not a member of this organization' },
        { status: 403 } // Returns a 403 Forbidden response.
      )
    }

    // 4. Authorize User (Role Check)
    const userRole = memberEntry[0].role // Gets the role of the current user in the organization.
    const hasAdminAccess = ['owner', 'admin'].includes(userRole) // Checks if the user's role is 'owner' or 'admin'.

    if (!hasAdminAccess) { // If the user doesn't have owner or admin access.
      return NextResponse.json({ error: 'Forbidden - Admin access required' }, { status: 403 }) // Returns a 403 Forbidden response.
    }

    // 5. Fetch Invitations
    const invitations = await db // Starts another database query.
      .select({ // Selects specific columns to return for each invitation.
        id: invitation.id,
        email: invitation.email,
        role: invitation.role,
        status: invitation.status,
        expiresAt: invitation.expiresAt,
        createdAt: invitation.createdAt,
        inviterName: user.name, // Selects the inviter's name from the 'user' table.
        inviterEmail: user.email, // Selects the inviter's email from the 'user' table.
      })
      .from(invitation) // Specifies the 'invitation' table.
      .leftJoin(user, eq(invitation.inviterId, user.id)) // Joins with the 'user' table on 'inviterId' to get inviter details. A left join ensures invitations without a matching user (unlikely but safe) are still returned.
      .where(eq(invitation.organizationId, organizationId)) // Filters invitations belonging to the specified organization.
      .orderBy(invitation.createdAt) // Orders the results by creation date.

    // 6. Return Success Response
    return NextResponse.json({ // Returns a JSON response indicating success.
      success: true,
      data: { // Contains the fetched invitations and the user's role.
        invitations,
        userRole,
      },
    })
  } catch (error) {
    // 7. Handle Errors
    logger.error('Failed to get organization invitations', { // Logs the error with context.
      organizationId: (await params).id, // Logs the organization ID.
      error, // Logs the error object itself.
    })

    return NextResponse.json({ error: 'Internal server error' }, { status: 500 }) // Returns a generic 500 Internal Server Error response for unexpected issues.
  }
}
```

---

#### POST `/api/organizations/[id]/invitations`

This function handles `POST` requests to create new invitations. It includes comprehensive validation, seat management, and email sending.

```typescript
/**
 * POST /api/organizations/[id]/invitations
 * Create organization invitations with optional validation and batch workspace invitations
 * Query parameters:
 * - ?validate=true - Only validate, don't send invitations
 * - ?batch=true - Include workspace invitations
 */
export async function POST(request: NextRequest, { params }: { params: Promise<{ id: string }> }) {
  try {
    // 1. Authenticate User
    const session = await getSession() // Retrieves the current user's authentication session.

    if (!session?.user?.id) { // Checks if a user is logged in.
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 }) // Returns a 401 Unauthorized response.
    }

    // 2. Extract Organization ID and Query Parameters
    const { id: organizationId } = await params // Extracts the 'id' (organization ID) from the URL parameters.
    const url = new URL(request.url) // Creates a URL object from the request URL to parse query parameters.
    const validateOnly = url.searchParams.get('validate') === 'true' // Checks if the 'validate' query parameter is 'true'. If so, only validation occurs, no invites are sent.
    const isBatch = url.searchParams.get('batch') === 'true' // Checks if the 'batch' query parameter is 'true'. If so, it implies handling multiple invites, potentially with workspace details.

    // 3. Parse Request Body
    const body = await request.json() // Parses the JSON body of the request.
    const { email, emails, role = 'member', workspaceInvitations } = body // Destructures relevant properties from the request body.
    // 'email' for a single invite, 'emails' for multiple, 'role' (defaults to 'member'), 'workspaceInvitations' for batch workspace invites.

    const invitationEmails = email ? [email] : emails // Consolidates 'email' or 'emails' into an array called 'invitationEmails'.

    // 4. Validate Basic Input
    if (!invitationEmails || !Array.isArray(invitationEmails) || invitationEmails.length === 0) {
      return NextResponse.json({ error: 'Email or emails array is required' }, { status: 400 }) // Returns 400 if no valid emails are provided.
    }

    if (!['member', 'admin'].includes(role)) { // Checks if the requested 'role' is valid.
      return NextResponse.json({ error: 'Invalid role' }, { status: 400 }) // Returns 400 if the role is invalid.
    }

    // 5. Authorize User (Membership and Role Check)
    const memberEntry = await db // Checks if the current user is a member of the organization.
      .select()
      .from(member)
      .where(and(eq(member.organizationId, organizationId), eq(member.userId, session.user.id)))
      .limit(1)

    if (memberEntry.length === 0) { // If user is not a member.
      return NextResponse.json(
        { error: 'Forbidden - Not a member of this organization' },
        { status: 403 }
      )
    }

    if (!['owner', 'admin'].includes(memberEntry[0].role)) { // If user is not an owner or admin.
      return NextResponse.json({ error: 'Forbidden - Admin access required' }, { status: 403 })
    }

    // 6. Handle 'Validate Only' Mode
    if (validateOnly) {
      const validationResult = await validateBulkInvitations(organizationId, invitationEmails) // Calls an external function to perform detailed validation without sending invites.

      logger.info('Invitation validation completed', { // Logs the validation event.
        organizationId,
        userId: session.user.id,
        emailCount: invitationEmails.length,
        result: validationResult,
      })

      return NextResponse.json({ // Returns the validation results.
        success: true,
        data: validationResult,
        validatedBy: session.user.id,
        validatedAt: new Date().toISOString(),
      })
    }

    // 7. Validate Seat Availability (if not validateOnly)
    const seatValidation = await validateSeatAvailability(organizationId, invitationEmails.length) // Checks if the organization has enough subscription seats for the requested number of invitations.

    if (!seatValidation.canInvite) { // If there aren't enough seats.
      return NextResponse.json(
        { // Returns a 400 error with details about seat availability.
          error: seatValidation.reason,
          seatInfo: {
            currentSeats: seatValidation.currentSeats,
            maxSeats: seatValidation.maxSeats,
            availableSeats: seatValidation.availableSeats,
            seatsRequested: invitationEmails.length,
          },
        },
        { status: 400 }
      )
    }

    // 8. Fetch Organization Details
    const organizationEntry = await db // Fetches the organization's name for use in email templates.
      .select({ name: organization.name })
      .from(organization)
      .where(eq(organization.id, organizationId))
      .limit(1)

    if (organizationEntry.length === 0) { // If the organization is not found (shouldn't happen if previous checks pass, but a safeguard).
      return NextResponse.json({ error: 'Organization not found' }, { status: 404 })
    }

    // 9. Process and Validate Emails
    const processedEmails = invitationEmails // Takes the raw invitation emails.
      .map((email: string) => {
        const normalized = email.trim().toLowerCase() // Cleans up each email (trims whitespace, converts to lowercase).
        const validation = quickValidateEmail(normalized) // Performs a quick format validation.
        return validation.isValid ? normalized : null // Returns the normalized email if valid, otherwise null.
      })
      .filter(Boolean) as string[] // Removes any nulls, resulting in an array of only valid, processed email strings.

    if (processedEmails.length === 0) { // If, after processing, no valid emails remain.
      return NextResponse.json({ error: 'No valid emails provided' }, { status: 400 })
    }

    // 10. Validate Workspace Invitations (if batch and workspaces are specified)
    const validWorkspaceInvitations: WorkspaceInvitation[] = [] // Initializes an array to store validated workspace invitations.
    if (isBatch && workspaceInvitations && workspaceInvitations.length > 0) { // Checks if batch mode is on and workspace invitations are provided.
      for (const wsInvitation of workspaceInvitations) { // Loops through each potential workspace invitation.
        // Checks if the current user has admin access to the target workspace.
        const canInvite = await hasWorkspaceAdminAccess(session.user.id, wsInvitation.workspaceId)

        if (!canInvite) { // If the user lacks permission for a specific workspace.
          return NextResponse.json(
            {
              error: `You don't have permission to invite users to workspace ${wsInvitation.workspaceId}`,
            },
            { status: 403 } // Returns a 403 Forbidden error.
          )
        }

        validWorkspaceInvitations.push(wsInvitation) // If valid, adds the workspace invitation to the list.
      }
    }

    // 11. Identify Existing Members and Pending Invitations
    const existingMembers = await db // Fetches emails of all current members of the organization.
      .select({ userEmail: user.email })
      .from(member)
      .innerJoin(user, eq(member.userId, user.id)) // Joins with 'user' table to get emails.
      .where(eq(member.organizationId, organizationId))

    const existingEmails = existingMembers.map((m) => m.userEmail) // Extracts only the email strings.
    // Filters 'processedEmails' to find those that are *not* already existing members.
    const newEmails = processedEmails.filter((email: string) => !existingEmails.includes(email))

    const existingInvitations = await db // Fetches emails of all users who already have pending invitations for this organization.
      .select({ email: invitation.email })
      .from(invitation)
      .where(and(eq(invitation.organizationId, organizationId), eq(invitation.status, 'pending')))

    const pendingEmails = existingInvitations.map((i) => i.email) // Extracts only the email strings.
    // Filters 'newEmails' to find those that *do not* have a pending invitation. These are the emails that will actually receive new invites.
    const emailsToInvite = newEmails.filter((email: string) => !pendingEmails.includes(email))

    if (emailsToInvite.length === 0) { // If, after all filtering, there are no new emails to invite.
      return NextResponse.json(
        { // Returns a 400 error explaining why no invitations were sent.
          error: 'All emails are already members or have pending invitations',
          details: {
            existingMembers: processedEmails.filter((email: string) =>
              existingEmails.includes(email)
            ), // Lists emails that were already members.
            pendingInvitations: processedEmails.filter((email: string) =>
              pendingEmails.includes(email)
            ), // Lists emails that already had pending invites.
          },
        },
        { status: 400 }
      )
    }

    // 12. Create Organization Invitations in DB
    const expiresAt = new Date(Date.now() + 7 * 24 * 60 * 60 * 1000) // Calculates an expiration date 7 days from now.
    const invitationsToCreate = emailsToInvite.map((email: string) => ({ // Maps each email to invite into an 'invitation' object for the database.
      id: randomUUID(), // Generates a unique ID for the invitation.
      email,
      inviterId: session.user.id,
      organizationId,
      role,
      status: 'pending' as const, // Sets the initial status to 'pending'.
      expiresAt,
      createdAt: new Date(),
    }))

    await db.insert(invitation).values(invitationsToCreate) // Inserts all new organization invitations into the database in a single batch.

    // 13. Create Workspace Invitations in DB (if applicable)
    const workspaceInvitationIds: string[] = [] // Initializes an array to store IDs of created workspace invitations.
    if (isBatch && validWorkspaceInvitations.length > 0) { // Checks if batch mode is on and there are valid workspace invites.
      for (const email of emailsToInvite) { // Loops through each email that is receiving an organization invitation.
        // Finds the organization invitation record for the current email to link workspace invites to it.
        const orgInviteForEmail = invitationsToCreate.find((inv) => inv.email === email)
        for (const wsInvitation of validWorkspaceInvitations) { // Loops through each valid workspace invitation detail.
          const wsInvitationId = randomUUID() // Generates a unique ID for the workspace invitation.
          const token = randomUUID() // Generates a unique token for the workspace invitation (for direct acceptance).

          await db.insert(workspaceInvitation).values({ // Inserts the workspace invitation into the database.
            id: wsInvitationId,
            workspaceId: wsInvitation.workspaceId,
            email,
            inviterId: session.user.id,
            role: 'member', // Default role for workspace invitations (often member).
            status: 'pending',
            token,
            permissions: wsInvitation.permission, // Sets the specific permissions for this workspace.
            orgInvitationId: orgInviteForEmail?.id, // Links this workspace invitation to the corresponding organization invitation.
            expiresAt,
            createdAt: new Date(),
            updatedAt: new Date(),
          })

          workspaceInvitationIds.push(wsInvitationId) // Adds the new workspace invitation ID to the list.
        }
      }
    }

    // 14. Send Invitation Emails
    const inviter = await db // Fetches the name of the inviter for use in email templates.
      .select({ name: user.name })
      .from(user)
      .where(eq(user.id, session.user.id))
      .limit(1)

    for (const email of emailsToInvite) { // Loops through each email that received an invitation.
      const orgInvitation = invitationsToCreate.find((inv) => inv.email === email) // Finds the created organization invitation for the current email.
      if (!orgInvitation) continue // Skips if, for some reason, the organization invitation isn't found (shouldn't happen).

      let emailResult // Variable to store the result of sending the email.
      if (isBatch && validWorkspaceInvitations.length > 0) { // If it's a batch invite with workspace details.
        const workspaceDetails = await db // Fetches details (like name) for all involved workspaces.
          .select({
            id: workspace.id,
            name: workspace.name,
          })
          .from(workspace)
          .where(
            inArray( // Filters workspaces whose IDs are in the list of valid workspace invitations.
              workspace.id,
              validWorkspaceInvitations.map((w) => w.workspaceId)
            )
          )

        const workspaceInvitationsWithNames = validWorkspaceInvitations.map((wsInv) => ({
          workspaceId: wsInv.workspaceId,
          // Finds the workspace name, defaults to 'Unknown Workspace' if not found.
          workspaceName:
            workspaceDetails.find((w) => w.id === wsInv.workspaceId)?.name || 'Unknown Workspace',
          permission: wsInv.permission,
        }))

        // Renders the batch invitation email HTML.
        const emailHtml = await renderBatchInvitationEmail(
          inviter[0]?.name || 'Someone', // Inviter's name or a fallback.
          organizationEntry[0]?.name || 'organization', // Organization name or a fallback.
          role, // The role the user is invited with.
          workspaceInvitationsWithNames, // Details of the workspace invitations.
          `${getBaseUrl()}/invite/${orgInvitation.id}` // The link for the user to accept the invitation.
        )

        emailResult = await sendEmail({ // Sends the batch email.
          to: email,
          subject: getEmailSubject('batch-invitation'), // Gets the subject line for batch invites.
          html: emailHtml,
          emailType: 'transactional', // Specifies the email type.
        })
      } else { // If it's a single (non-batch or no workspace) invitation.
        // Renders the standard invitation email HTML.
        const emailHtml = await renderInvitationEmail(
          inviter[0]?.name || 'Someone',
          organizationEntry[0]?.name || 'organization',
          `${getBaseUrl()}/invite/${orgInvitation.id}`,
          email
        )

        emailResult = await sendEmail({ // Sends the standard email.
          to: email,
          subject: getEmailSubject('invitation'), // Gets the subject line for standard invites.
          html: emailHtml,
          emailType: 'transactional',
        })
      }

      if (!emailResult.success) { // If email sending failed for a specific invite.
        logger.error('Failed to send invitation email', { // Logs the email sending error.
          email,
          error: emailResult.message,
        })
      }
    }

    // 15. Log Success
    logger.info('Organization invitations created', { // Logs a successful invitation creation event.
      organizationId,
      invitedBy: session.user.id,
      invitationCount: invitationsToCreate.length,
      emails: emailsToInvite,
      role,
      isBatch,
      workspaceInvitationCount: workspaceInvitationIds.length,
    })

    // 16. Return Success Response
    return NextResponse.json({ // Returns a detailed success response.
      success: true,
      message: `${invitationsToCreate.length} invitation(s) sent successfully`,
      data: {
        invitationsSent: invitationsToCreate.length, // Number of new invitations sent.
        invitedEmails: emailsToInvite, // List of emails that received new invitations.
        // Lists emails that were provided but already existed as members or pending invites.
        existingMembers: processedEmails.filter((email: string) => existingEmails.includes(email)),
        pendingInvitations: processedEmails.filter((email: string) =>
          pendingEmails.includes(email)
        ),
        // Lists emails that were provided but were invalid.
        invalidEmails: invitationEmails.filter(
          (email: string) => !quickValidateEmail(email.trim().toLowerCase()).isValid
        ),
        workspaceInvitations: isBatch ? validWorkspaceInvitations.length : 0, // Number of workspace invitations created.
        seatInfo: { // Updated seat information after invitations.
          seatsUsed: seatValidation.currentSeats + invitationsToCreate.length,
          maxSeats: seatValidation.maxSeats,
          availableSeats: seatValidation.availableSeats - invitationsToCreate.length,
        },
      },
    })
  } catch (error) {
    // 17. Handle Errors
    logger.error('Failed to create organization invitations', { // Logs the error.
      organizationId: (await params).id,
      error,
    })

    return NextResponse.json({ error: 'Internal server error' }, { status: 500 }) // Returns a generic 500 Internal Server Error.
  }
}
```

---

#### DELETE `/api/organizations/[id]/invitations`

This function handles `DELETE` requests to cancel a pending organization invitation.

```typescript
/**
 * DELETE /api/organizations/[id]/invitations?invitationId=...
 * Cancel a pending invitation
 */
export async function DELETE(
  request: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  try {
    // 1. Authenticate User
    const session = await getSession() // Retrieves the current user's authentication session.

    if (!session?.user?.id) { // Checks if a user is logged in.
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 }) // Returns a 401 Unauthorized response.
    }

    // 2. Extract Organization ID and Invitation ID
    const { id: organizationId } = await params // Extracts the 'id' (organization ID) from the URL parameters.
    const url = new URL(request.url) // Creates a URL object to parse query parameters.
    const invitationId = url.searchParams.get('invitationId') // Extracts the 'invitationId' from the query parameters.

    if (!invitationId) { // Checks if 'invitationId' was provided in the query.
      return NextResponse.json(
        { error: 'Invitation ID is required as query parameter' },
        { status: 400 } // Returns a 400 Bad Request if missing.
      )
    }

    // 3. Authorize User (Membership and Role Check)
    const memberEntry = await db // Checks if the current user is a member of the organization.
      .select()
      .from(member)
      .where(and(eq(member.organizationId, organizationId), eq(member.userId, session.user.id)))
      .limit(1)

    if (memberEntry.length === 0) { // If user is not a member.
      return NextResponse.json(
        { error: 'Forbidden - Not a member of this organization' },
        { status: 403 }
      )
    }

    if (!['owner', 'admin'].includes(memberEntry[0].role)) { // If user is not an owner or admin.
      return NextResponse.json({ error: 'Forbidden - Admin access required' }, { status: 403 })
    }

    // 4. Cancel Organization Invitation
    const result = await db // Updates the 'invitation' table.
      .update(invitation)
      .set({ status: 'cancelled' }) // Sets the invitation status to 'cancelled'.
      .where( // Defines conditions for the update:
        and( // All these conditions must be true:
          eq(invitation.id, invitationId), // Matches the specific invitation ID.
          eq(invitation.organizationId, organizationId), // Ensures it belongs to the correct organization.
          or( // The invitation status must be 'pending' OR 'rejected' to be cancelled.
            eq(invitation.status, 'pending'),
            eq(invitation.status, 'rejected')
          )
        )
      )
      .returning() // Returns the updated invitation record.

    if (result.length === 0) { // If no invitation was found or updated (e.g., already accepted, or incorrect ID/organization).
      return NextResponse.json(
        { error: 'Invitation not found or already processed' },
        { status: 404 } // Returns a 404 Not Found error.
      )
    }

    // 5. Cancel Related Workspace Invitations
    // Cancels workspace invitations that were linked to the cancelled organization invitation.
    await db
      .update(workspaceInvitation)
      .set({ status: 'cancelled' as WorkspaceInvitationStatus }) // Sets status to 'cancelled'.
      .where(eq(workspaceInvitation.orgInvitationId, invitationId)) // Filters by the 'orgInvitationId' matching the cancelled organization invite.

    // Cancels standalone workspace invitations (not linked to an org invite) that match the email, status, and inviter.
    await db
      .update(workspaceInvitation)
      .set({ status: 'cancelled' as WorkspaceInvitationStatus })
      .where(
        and(
          isNull(workspaceInvitation.orgInvitationId), // Specifically targets workspace invites that are NOT linked to an org invite.
          eq(workspaceInvitation.email, result[0].email), // Matches the email of the cancelled org invitation.
          eq(workspaceInvitation.status, 'pending' as WorkspaceInvitationStatus), // Must be a pending workspace invite.
          eq(workspaceInvitation.inviterId, session.user.id) // Must have been invited by the current user.
        )
      )

    // 6. Log Success
    logger.info('Organization invitation cancelled', { // Logs the successful cancellation.
      organizationId,
      invitationId,
      cancelledBy: session.user.id,
      email: result[0].email, // Logs the email of the cancelled invitation.
    })

    // 7. Return Success Response
    return NextResponse.json({
      success: true,
      message: 'Invitation cancelled successfully',
    })
  } catch (error) {
    // 8. Handle Errors
    logger.error('Failed to cancel organization invitation', { // Logs the error.
      organizationId: (await params).id,
      error,
    })

    return NextResponse.json({ error: 'Internal server error' }, { status: 500 }) // Returns a generic 500 Internal Server Error.
  }
}
```