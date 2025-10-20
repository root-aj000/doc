This TypeScript file defines an **API route handler** for managing workspace invitations within a Next.js application. It specifically handles requests to `/api/workspaces/invitations/[invitationId]`, acting as a central point for fetching invitation details, accepting invitations, deleting pending invitations, and resending invitations.

It leverages Drizzle ORM for database interactions, Next.js for API routing, and a custom email service to facilitate the invitation process.

---

## Detailed Explanation

### 1. File Purpose and What It Does

This file creates a Next.js Route Handler (API endpoint) that exposes three different operations for a specific workspace invitation, identified by `invitationId`:

*   **`GET /api/workspaces/invitations/[invitationId]`**:
    *   **Retrieves invitation details**: If accessed by a logged-in user without a `token` in the URL, it provides information about a specific invitation.
    *   **Accepts an invitation**: If accessed by a logged-in user *with* a `token` in the URL, it attempts to accept the invitation, adds the user to the workspace, and redirects them. It handles various validation checks (expiry, status, email match).
*   **`DELETE /api/workspaces/invitations/[invitationId]`**:
    *   Allows a workspace administrator to delete a *pending* invitation.
*   **`POST /api/workspaces/invitations/[invitationId]`**:
    *   Allows a workspace administrator to resend a *pending* invitation, generating a new token and expiry, and sending a new email.

In essence, this file is the backend logic for managing the lifecycle of a workspace invitation from viewing to acceptance, deletion, or re-issuance.

### 2. Imports (Setup & Dependencies)

This section imports all necessary modules, functions, and types from various libraries and local files.

```typescript
import { randomUUID } from 'crypto' // For generating unique IDs
import { render } from '@react-email/render' // To render React components into HTML for emails
import { db } from '@sim/db' // Our Drizzle ORM database instance
import {
  permissions, // Drizzle schema for user permissions
  user, // Drizzle schema for user accounts
  type WorkspaceInvitationStatus, // Type definition for invitation status (e.g., 'pending', 'accepted')
  workspace, // Drizzle schema for workspaces
  workspaceInvitation, // Drizzle schema for workspace invitations
} from '@sim/db/schema' // Central database schema definitions
import { and, eq } from 'drizzle-orm' // Drizzle ORM functions for building queries (AND, EQUAL)
import { type NextRequest, NextResponse } from 'next/server' // Next.js server utilities for handling requests and responses
import { WorkspaceInvitationEmail } from '@/components/emails/workspace-invitation' // React Email component for invitation emails
import { getSession } from '@/lib/auth' // Utility to retrieve the current user's session
import { sendEmail } from '@/lib/email/mailer' // Utility to send emails
import { getFromEmailAddress } from '@/lib/email/utils' // Utility to get the sender's email address
import { hasWorkspaceAdminAccess } from '@/lib/permissions/utils' // Utility to check if a user has admin access to a workspace
import { getBaseUrl } from '@/lib/urls/utils' // Utility to get the base URL of the application
```

*   `randomUUID`: Generates universally unique identifiers. Used for new permission entries and updating invitation tokens.
*   `render` (from `@react-email/render`): A utility that takes a React component (like `WorkspaceInvitationEmail`) and renders it into a static HTML string, which can then be sent as an email body.
*   `db` (from `@sim/db`): This is the configured Drizzle ORM client, providing an interface to interact with the database.
*   Schema imports (`permissions`, `user`, `workspace`, `workspaceInvitation`): These are Drizzle ORM schema objects representing the database tables. They are used to construct queries.
*   `WorkspaceInvitationStatus`: A TypeScript type representing the possible states of a workspace invitation (e.g., 'pending', 'accepted', 'declined', 'expired').
*   `and`, `eq` (from `drizzle-orm`): Helper functions from Drizzle ORM used to build `WHERE` clauses in SQL queries. `eq` checks for equality, `and` combines multiple conditions.
*   `NextRequest`, `NextResponse` (from `next/server`): Core Next.js types and classes for handling incoming HTTP requests and crafting HTTP responses in Route Handlers.
*   `WorkspaceInvitationEmail`: A React component designed specifically for generating the HTML content of the invitation email.
*   `getSession`: A local utility function to retrieve the authenticated user's session data, including their ID and email.
*   `sendEmail`: A local utility function that abstracts away the actual email sending mechanism (e.g., using a service like Resend, SendGrid, etc.).
*   `getFromEmailAddress`: A local utility to determine the sender's email address for transactional emails.
*   `hasWorkspaceAdminAccess`: A local utility that checks if a given user has administrative privileges for a specific workspace. This is crucial for security.
*   `getBaseUrl`: A local utility to get the root URL of the application (e.g., `https://app.example.com`), used for constructing invitation links and redirects.

### 3. `GET` Request Handler

This function handles `GET` requests to `/api/workspaces/invitations/[invitationId]`. It has two main purposes: fetching invitation details or accepting an invitation via a token.

```typescript
// GET /api/workspaces/invitations/[invitationId] - Get invitation details OR accept via token
export async function GET(
  req: NextRequest, // The incoming Next.js request object
  { params }: { params: Promise<{ invitationId: string }> } // Dynamic route parameters, invitationId is extracted from the URL
) {
  // Extract the invitationId from the URL parameters
  const { invitationId } = await params
  // Retrieve the current user's session information
  const session = await getSession()
  // Check for a 'token' query parameter in the URL (e.g., ?token=xyz)
  const token = req.nextUrl.searchParams.get('token')
  // Determine if this is an "accept invitation" flow based on the presence of a token
  const isAcceptFlow = !!token

  // --- 1. Authorization Check (No Session) ---
  if (!session?.user?.id) {
    // If no user is logged in:
    if (isAcceptFlow) {
      // If it's an accept flow, redirect the user to a login/signup page with the invite details
      // so they can log in and then accept the invite.
      return NextResponse.redirect(new URL(`/invite/${invitationId}?token=${token}`, getBaseUrl()))
    }
    // If it's just trying to view invitation details without a token, return an Unauthorized error
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
  }

  try {
    // --- 2. Determine Database Query Condition ---
    // If a token is present, find the invitation by token; otherwise, find by ID.
    const whereClause = token
      ? eq(workspaceInvitation.token, token) // Look up by token for acceptance flow
      : eq(workspaceInvitation.id, invitationId) // Look up by ID for viewing details

    // --- 3. Fetch Invitation from Database ---
    const invitation = await db
      .select() // Select all columns
      .from(workspaceInvitation) // From the workspaceInvitation table
      .where(whereClause) // Using the determined condition (by token or ID)
      .then((rows) => rows[0]) // Get the first result (or undefined if not found)

    // --- 4. Handle Invitation Not Found ---
    if (!invitation) {
      if (isAcceptFlow) {
        // If it's an accept flow and the invitation isn't found (or token invalid/expired),
        // redirect to the invite page with an error.
        return NextResponse.redirect(
          new URL(`/invite/${invitationId}?error=invalid-token`, getBaseUrl())
        )
      }
      // If just viewing details, return a 404 Not Found error.
      return NextResponse.json({ error: 'Invitation not found or has expired' }, { status: 404 })
    }

    // --- 5. Check Invitation Expiry ---
    if (new Date() > new Date(invitation.expiresAt)) {
      if (isAcceptFlow) {
        // If expired during an accept flow, redirect with an expired error.
        return NextResponse.redirect(
          new URL(`/invite/${invitation.id}?error=expired`, getBaseUrl())
        )
      }
      // If expired during viewing, return a 400 Bad Request error.
      return NextResponse.json({ error: 'Invitation has expired' }, { status: 400 })
    }

    // --- 6. Fetch Workspace Details ---
    const workspaceDetails = await db
      .select()
      .from(workspace)
      .where(eq(workspace.id, invitation.workspaceId)) // Find the workspace associated with the invitation
      .then((rows) => rows[0])

    // --- 7. Handle Workspace Not Found ---
    if (!workspaceDetails) {
      if (isAcceptFlow) {
        // If workspace not found during accept flow, redirect with error.
        return NextResponse.redirect(
          new URL(`/invite/${invitation.id}?error=workspace-not-found`, getBaseUrl())
        )
      }
      // If workspace not found during viewing, return a 404 Not Found.
      return NextResponse.json({ error: 'Workspace not found' }, { status: 404 })
    }

    // --- 8. Logic for Accepting an Invitation (`isAcceptFlow` is true) ---
    if (isAcceptFlow) {
      // Ensure the invitation is still pending.
      if (invitation.status !== ('pending' as WorkspaceInvitationStatus)) {
        // If not pending (already accepted/declined), redirect with an error.
        return NextResponse.redirect(
          new URL(`/invite/${invitation.id}?error=already-processed`, getBaseUrl())
        )
      }

      // Convert emails to lowercase for case-insensitive comparison.
      const userEmail = session.user.email.toLowerCase()
      const invitationEmail = invitation.email.toLowerCase()

      // Fetch the current user's full data from the database.
      const userData = await db
        .select()
        .from(user)
        .where(eq(user.id, session.user.id))
        .then((rows) => rows[0])

      // If user data can't be found (shouldn't happen if session is valid, but good to check).
      if (!userData) {
        return NextResponse.redirect(
          new URL(`/invite/${invitation.id}?error=user-not-found`, getBaseUrl())
        )
      }

      // Crucial security check: Ensure the logged-in user's email matches the invitation email.
      const isValidMatch = userEmail === invitationEmail

      if (!isValidMatch) {
        // If emails don't match, prevent acceptance and redirect with an error.
        return NextResponse.redirect(
          new URL(`/invite/${invitation.id}?error=email-mismatch`, getBaseUrl())
        )
      }

      // Check if the user already has permissions for this workspace.
      const existingPermission = await db
        .select()
        .from(permissions)
        .where(
          and(
            eq(permissions.entityId, invitation.workspaceId),
            eq(permissions.entityType, 'workspace'),
            eq(permissions.userId, session.user.id)
          )
        )
        .then((rows) => rows[0])

      if (existingPermission) {
        // If user is already a member, just update the invitation status to accepted.
        // This prevents duplicate permission entries.
        await db
          .update(workspaceInvitation)
          .set({
            status: 'accepted' as WorkspaceInvitationStatus,
            updatedAt: new Date(),
          })
          .where(eq(workspaceInvitation.id, invitation.id))

        // Redirect the user directly to the workspace dashboard.
        return NextResponse.redirect(
          new URL(`/workspace/${invitation.workspaceId}/w`, getBaseUrl())
        )
      }

      // If the user is not already a member, proceed to add them.
      // Use a database transaction to ensure both operations (add permission, update invitation) succeed or fail together.
      await db.transaction(async (tx) => {
        // Insert a new permission record for the user in the workspace.
        await tx.insert(permissions).values({
          id: randomUUID(), // Generate a unique ID for the new permission
          entityType: 'workspace' as const, // Specify permission type
          entityId: invitation.workspaceId, // Link to the workspace
          userId: session.user.id, // Link to the user
          permissionType: invitation.permissions || 'read', // Assign permissions from the invitation, default to 'read'
          createdAt: new Date(),
          updatedAt: new Date(),
        })

        // Update the invitation status to 'accepted'.
        await tx
          .update(workspaceInvitation)
          .set({
            status: 'accepted' as WorkspaceInvitationStatus,
            updatedAt: new Date(),
          })
          .where(eq(workspaceInvitation.id, invitation.id))
      })

      // After successful acceptance, redirect the user to the workspace dashboard.
      return NextResponse.redirect(new URL(`/workspace/${invitation.workspaceId}/w`, getBaseUrl()))
    }

    // --- 9. Logic for Viewing Invitation Details (Non-Accept Flow) ---
    // If not an accept flow (i.e., just viewing details), return the invitation and workspace name.
    return NextResponse.json({
      ...invitation, // Spread all properties of the invitation object
      workspaceName: workspaceDetails.name, // Add the workspace name for context
    })
  } catch (error) {
    // --- 10. Error Handling ---
    console.error('Error fetching workspace invitation:', error)
    return NextResponse.json({ error: 'Failed to fetch invitation details' }, { status: 500 })
  }
}
```

**Simplified Logic for GET:**

1.  **Identify Intent**: Is the user trying to *view* an invitation or *accept* one? This is determined by the presence of a `token` in the URL.
2.  **Authentication**: Ensure a user is logged in. If an acceptance flow needs login, redirect them to log in first.
3.  **Find Invitation**: Search the database for the invitation, either by its unique ID (for viewing) or by the provided token (for acceptance).
4.  **Validate Invitation**: Check if the invitation exists, hasn't expired, and its associated workspace still exists. Handle these as errors or redirects if they fail.
5.  **Acceptance Flow (`isAcceptFlow`):**
    *   **Status Check**: Ensure the invitation is still `pending`.
    *   **Email Match**: Crucially, verify that the logged-in user's email *exactly matches* the email the invitation was sent to. This prevents unauthorized users from accepting invites.
    *   **Existing Member Check**: See if the user is already a member of the workspace.
    *   **Add Member & Update**: If they're not a member, add them to the workspace (by creating a `permission` record) and then mark the invitation as `accepted`. This is done in a database *transaction* to ensure both steps succeed or fail together. If they are already a member, just update the invitation status.
    *   **Redirect**: Send the user to the workspace dashboard.
6.  **Viewing Flow (else block)**: If it's just a request to view invitation details, return the invitation data along with the workspace name.
7.  **Error Handling**: Catch any unexpected errors and return a generic server error.

### 4. `DELETE` Request Handler

This function handles `DELETE` requests to `/api/workspaces/invitations/[invitationId]`, allowing administrators to revoke pending invitations.

```typescript
// DELETE /api/workspaces/invitations/[invitationId] - Delete a workspace invitation
export async function DELETE(
  _req: NextRequest, // The request object (unused, hence _req)
  { params }: { params: Promise<{ invitationId: string }> } // Dynamic route parameters
) {
  // Extract the invitationId from the URL parameters
  const { invitationId } = await params
  // Retrieve the current user's session information
  const session = await getSession()

  // --- 1. Authorization Check (No Session) ---
  if (!session?.user?.id) {
    // If no user is logged in, return an Unauthorized error.
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
  }

  try {
    // --- 2. Fetch Invitation Details ---
    const invitation = await db
      .select({
        // Select only necessary fields for efficiency and security
        id: workspaceInvitation.id,
        workspaceId: workspaceInvitation.workspaceId,
        email: workspaceInvitation.email,
        inviterId: workspaceInvitation.inviterId,
        status: workspaceInvitation.status,
      })
      .from(workspaceInvitation)
      .where(eq(workspaceInvitation.id, invitationId)) // Find the invitation by its ID
      .then((rows) => rows[0])

    // --- 3. Handle Invitation Not Found ---
    if (!invitation) {
      return NextResponse.json({ error: 'Invitation not found' }, { status: 404 })
    }

    // --- 4. Permission Check (Admin Access) ---
    // Check if the logged-in user has admin access to the workspace associated with the invitation.
    const hasAdminAccess = await hasWorkspaceAdminAccess(session.user.id, invitation.workspaceId)

    if (!hasAdminAccess) {
      // If not an admin, return a Forbidden error.
      return NextResponse.json({ error: 'Insufficient permissions' }, { status: 403 })
    }

    // --- 5. Status Check (Must be Pending) ---
    if (invitation.status !== ('pending' as WorkspaceInvitationStatus)) {
      // Only pending invitations can be deleted. Once accepted/declined, they are historical.
      return NextResponse.json({ error: 'Can only delete pending invitations' }, { status: 400 })
    }

    // --- 6. Delete Invitation from Database ---
    await db.delete(workspaceInvitation).where(eq(workspaceInvitation.id, invitationId))

    // --- 7. Success Response ---
    return NextResponse.json({ success: true })
  } catch (error) {
    // --- 8. Error Handling ---
    console.error('Error deleting workspace invitation:', error)
    return NextResponse.json({ error: 'Failed to delete invitation' }, { status: 500 })
  }
}
```

**Simplified Logic for DELETE:**

1.  **Authentication**: Ensure a user is logged in.
2.  **Find Invitation**: Retrieve the invitation to be deleted.
3.  **Validate Existence**: If not found, return an error.
4.  **Admin Check**: Verify that the logged-in user has administrative rights for the workspace this invitation belongs to. This is a critical security step.
5.  **Status Check**: Ensure the invitation is still in a `pending` state. Accepted or declined invitations cannot be deleted; they become part of the historical record.
6.  **Delete**: Remove the invitation from the database.
7.  **Success**: Return a success message.
8.  **Error Handling**: Catch any unexpected errors.

### 5. `POST` Request Handler

This function handles `POST` requests to `/api/workspaces/invitations/[invitationId]`, enabling administrators to resend an invitation.

```typescript
// POST /api/workspaces/invitations/[invitationId] - Resend a workspace invitation
export async function POST(
  _req: NextRequest, // The request object (unused, hence _req)
  { params }: { params: Promise<{ invitationId: string }> } // Dynamic route parameters
) {
  // Extract the invitationId from the URL parameters
  const { invitationId } = await params
  // Retrieve the current user's session information
  const session = await getSession()

  // --- 1. Authorization Check (No Session) ---
  if (!session?.user?.id) {
    // If no user is logged in, return an Unauthorized error.
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
  }

  try {
    // --- 2. Fetch Invitation Details ---
    const invitation = await db
      .select()
      .from(workspaceInvitation)
      .where(eq(workspaceInvitation.id, invitationId)) // Find the invitation by its ID
      .then((rows) => rows[0])

    // --- 3. Handle Invitation Not Found ---
    if (!invitation) {
      return NextResponse.json({ error: 'Invitation not found' }, { status: 404 })
    }

    // --- 4. Permission Check (Admin Access) ---
    // Verify that the logged-in user has admin access to the workspace.
    const hasAdminAccess = await hasWorkspaceAdminAccess(session.user.id, invitation.workspaceId)
    if (!hasAdminAccess) {
      return NextResponse.json({ error: 'Insufficient permissions' }, { status: 403 })
    }

    // --- 5. Status Check (Must be Pending) ---
    if (invitation.status !== ('pending' as WorkspaceInvitationStatus)) {
      // Only pending invitations can be resent.
      return NextResponse.json({ error: 'Can only resend pending invitations' }, { status: 400 })
    }

    // --- 6. Fetch Workspace Details ---
    const ws = await db
      .select()
      .from(workspace)
      .where(eq(workspace.id, invitation.workspaceId)) // Get the associated workspace details
      .then((rows) => rows[0])

    // --- 7. Handle Workspace Not Found ---
    if (!ws) {
      return NextResponse.json({ error: 'Workspace not found' }, { status: 404 })
    }

    // --- 8. Generate New Token and Expiry ---
    const newToken = randomUUID() // Create a brand new unique token for the invitation
    const newExpiresAt = new Date()
    newExpiresAt.setDate(newExpiresAt.getDate() + 7) // Set expiry to 7 days from now

    // --- 9. Update Invitation in Database ---
    await db
      .update(workspaceInvitation)
      .set({ token: newToken, expiresAt: newExpiresAt, updatedAt: new Date() }) // Update token, expiry, and updated timestamp
      .where(eq(workspaceInvitation.id, invitationId))

    // --- 10. Prepare and Send Email ---
    const baseUrl = getBaseUrl()
    // Construct the new invitation link with the updated token
    const invitationLink = `${baseUrl}/invite/${invitationId}?token=${newToken}`

    // Render the React email component into HTML
    const emailHtml = await render(
      WorkspaceInvitationEmail({
        workspaceName: ws.name,
        inviterName: session.user.name || session.user.email || 'A user', // Use inviter's name or email, or a generic fallback
        invitationLink,
      })
    )

    // Send the email using the mailer utility
    const result = await sendEmail({
      to: invitation.email,
      subject: `You've been invited to join "${ws.name}" on Sim`,
      html: emailHtml,
      from: getFromEmailAddress(),
      emailType: 'transactional',
    })

    // --- 11. Handle Email Sending Failure ---
    if (!result.success) {
      // If the email failed to send, return an error.
      return NextResponse.json(
        { error: 'Failed to send invitation email. Please try again.' },
        { status: 500 }
      )
    }

    // --- 12. Success Response ---
    return NextResponse.json({ success: true })
  } catch (error) {
    // --- 13. Error Handling ---
    console.error('Error resending workspace invitation:', error)
    return NextResponse.json({ error: 'Failed to resend invitation' }, { status: 500 })
  }
}
```

**Simplified Logic for POST:**

1.  **Authentication**: Ensure a user is logged in.
2.  **Find Invitation**: Retrieve the invitation to be resent.
3.  **Validate Existence**: If not found, return an error.
4.  **Admin Check**: Verify the logged-in user has admin rights for the workspace.
5.  **Status Check**: Only `pending` invitations can be resent.
6.  **Find Workspace**: Retrieve details of the workspace the invitation belongs to.
7.  **Generate New Credentials**: Create a new, unique token and update the expiry date for the invitation (typically 7 days from now).
8.  **Update Database**: Persist the new token and expiry in the database.
9.  **Prepare Email**: Construct the new invitation link using the updated token. Render the `WorkspaceInvitationEmail` React component into an HTML string, populating it with relevant details like workspace name and inviter's name.
10. **Send Email**: Use the `sendEmail` utility to dispatch the new invitation email to the original invitee.
11. **Email Sending Status**: Check if the email was sent successfully. If not, return an error.
12. **Success**: Return a success message.
13. **Error Handling**: Catch any unexpected errors.

---

### Conclusion

This file is a robust and secure API endpoint that provides comprehensive management for workspace invitations. It carefully handles authentication, authorization, data validation, database transactions, and integrates an email service for notifications. The distinct `GET`, `DELETE`, and `POST` methods allow for distinct actions on an invitation resource, adhering to RESTful principles where appropriate. The inclusion of `try...catch` blocks and specific error responses for various scenarios ensures a resilient and user-friendly experience, while security checks like `hasWorkspaceAdminAccess` and email matching in the acceptance flow protect against unauthorized access.