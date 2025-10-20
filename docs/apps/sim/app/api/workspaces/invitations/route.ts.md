This TypeScript file defines an API route for handling workspace invitations in a Next.js application. It allows authenticated users to:

1.  **Retrieve** a list of all pending workspace invitations associated with the workspaces they are a member of (via `GET` request).
2.  **Create** new workspace invitations, sending an email to the invited user (via `POST` request).

It integrates with a database (using Drizzle ORM), an authentication system, and an email sending service.

---

### File Purpose and What it Does

This file acts as the backend logic for managing workspace invitations. It exposes two main endpoints:

*   **`GET /api/invitations` (hypothetical path):** Fetches all workspace invitations relevant to the current user. This means it will list invitations for workspaces where the user has existing permissions, allowing them to see who else has been invited or is awaiting acceptance.
*   **`POST /api/invitations` (hypothetical path):** Creates a new invitation. An authenticated user (who must have admin permissions for the target workspace) can invite another user by their email address. This process involves:
    *   Validating input and permissions.
    *   Checking for existing users or pending invitations to prevent duplicates.
    *   Storing the invitation in the database.
    *   Generating a unique invitation token.
    *   Sending an email to the invitee with a link to accept the invitation.

**Simplified Complex Logic:**

*   **`GET`:** Instead of thinking about complex SQL joins, imagine this as: "Show me all the invitations that exist for any of the teams (workspaces) I'm already a part of."
*   **`POST`:** "I want to invite someone to my team. First, the system checks if I'm allowed to invite people to *this specific team* (am I an admin?). Then it checks if that person is *already on the team* or *already has a pending invite*. If all clear, it creates a new record for the invite and sends them an email with a special link to join."

---

### Detailed Line-by-Line Explanation

#### Imports

```typescript
import { randomUUID } from 'crypto'
```
*   **Purpose:** Imports the `randomUUID` function from Node.js's built-in `crypto` module.
*   **Explanation:** This function is used to generate cryptographically strong, universally unique identifiers (UUIDs). In this file, it's crucial for creating unique IDs for new invitation records and for generating secure, non-guessable invitation tokens.

```typescript
import { render } from '@react-email/render'
```
*   **Purpose:** Imports the `render` function from the `@react-email/render` library.
*   **Explanation:** `React Email` is a framework for building emails with React components. The `render` function takes a React email component (like `WorkspaceInvitationEmail` later in the file) and converts it into a static HTML string that can be sent in an email.

```typescript
import { db } from '@sim/db'
```
*   **Purpose:** Imports the initialized Drizzle ORM database client.
*   **Explanation:** `@sim/db` likely refers to a project-specific module that sets up and exports a configured Drizzle ORM instance. This `db` object is the primary interface for interacting with the database throughout this file.

```typescript
import {
  permissions,
  type permissionTypeEnum,
  user,
  type WorkspaceInvitationStatus,
  workspace,
  workspaceInvitation,
} from '@sim/db/schema'
```
*   **Purpose:** Imports Drizzle ORM schema definitions and related types.
*   **Explanation:**
    *   `permissions`: Represents the database table where user permissions for entities (like workspaces) are stored.
    *   `type permissionTypeEnum`: A TypeScript type representing the allowed values for permission types (e.g., 'admin', 'read', 'write'). This enhances type safety.
    *   `user`: Represents the database table storing user information.
    *   `type WorkspaceInvitationStatus`: A TypeScript type defining the possible statuses for a workspace invitation (e.g., 'pending', 'accepted', 'declined').
    *   `workspace`: Represents the database table storing workspace details.
    *   `workspaceInvitation`: Represents the database table storing invitation records.
    *   These imports allow Drizzle ORM to construct type-safe queries against these specific tables.

```typescript
import { and, eq, inArray } from 'drizzle-orm'
```
*   **Purpose:** Imports Drizzle ORM's utility functions for building query conditions.
*   **Explanation:**
    *   `and`: Combines multiple conditions with a logical AND. Used when a query needs to satisfy several criteria simultaneously.
    *   `eq`: Checks for equality between a column and a value. (e.g., `column === value`).
    *   `inArray`: Checks if a column's value is present within a given array of values. (e.g., `column IN (value1, value2)`).
    *   These functions are essential for creating flexible and precise database `WHERE` clauses.

```typescript
import { type NextRequest, NextResponse } from 'next/server'
```
*   **Purpose:** Imports types and classes specific to Next.js API routes.
*   **Explanation:**
    *   `NextRequest`: A type that represents the incoming HTTP request object in a Next.js API route. It provides access to request methods, headers, body, query parameters, etc.
    *   `NextResponse`: A class used to construct and send HTTP responses from a Next.js API route, allowing control over status codes, headers, and JSON body.

```typescript
import { WorkspaceInvitationEmail } from '@/components/emails/workspace-invitation'
```
*   **Purpose:** Imports a React component designed for rendering the workspace invitation email.
*   **Explanation:** This component (likely built with `React Email`) defines the structure and content of the email that will be sent to invited users. It takes props like `workspaceName`, `inviterName`, and `invitationLink` to personalize the email.

```typescript
import { getSession } from '@/lib/auth'
```
*   **Purpose:** Imports a utility function for retrieving the current user's authentication session.
*   **Explanation:** `@/lib/auth` likely contains authentication-related logic. `getSession` is used to determine if a user is logged in and to access their user ID and other session data, which is crucial for authorization checks.

```typescript
import { sendEmail } from '@/lib/email/mailer'
```
*   **Purpose:** Imports a custom utility function for sending emails.
*   **Explanation:** `@/lib/email/mailer` probably wraps an email sending service (e.g., Resend, SendGrid). The `sendEmail` function provides a standardized interface for sending emails from the application, abstracting away the specifics of the chosen email provider.

```typescript
import { getFromEmailAddress } from '@/lib/email/utils'
```
*   **Purpose:** Imports a utility function to get the sender's email address.
*   **Explanation:** This function likely retrieves the configured "from" email address (e.g., `no-reply@sim.com`) from environment variables or a configuration file, ensuring consistency for outgoing emails.

```typescript
import { createLogger } from '@/lib/logs/console/logger'
```
*   **Purpose:** Imports a function to create a logger instance.
*   **Explanation:** This helps with structured logging. `createLogger` likely configures a logger that outputs messages to the console (or a logging service) with specific prefixes or formatting, making it easier to trace logs originating from this particular file.

```typescript
import { getBaseUrl } from '@/lib/urls/utils'
```
*   **Purpose:** Imports a utility function to get the base URL of the application.
*   **Explanation:** This is essential for constructing absolute URLs, particularly for the invitation link that will be included in the email. It ensures the link points to the correct application domain.

#### Configuration and Initialization

```typescript
export const dynamic = 'force-dynamic'
```
*   **Purpose:** Next.js App Router configuration to force dynamic rendering.
*   **Explanation:** By default, Next.js API routes can be statically optimized or dynamically rendered. `force-dynamic` explicitly tells Next.js to always execute this API route at runtime on the server, ensuring fresh data for every request and preventing static caching of its responses. This is important for routes dealing with user-specific data or real-time operations.

```typescript
const logger = createLogger('WorkspaceInvitationsAPI')
```
*   **Purpose:** Initializes a logger instance for this specific module.
*   **Explanation:** All log messages generated within this file will be associated with the name 'WorkspaceInvitationsAPI'. This helps in filtering and understanding logs when debugging or monitoring the application.

```typescript
type PermissionType = (typeof permissionTypeEnum.enumValues)[number]
```
*   **Purpose:** Creates a TypeScript type alias `PermissionType`.
*   **Explanation:** `permissionTypeEnum` is likely a Drizzle ORM enum definition, which has an `enumValues` array containing all valid permission strings (e.g., `['admin', 'write', 'read']`). This line extracts these string literal types into `PermissionType`, providing strong type checking for permission strings throughout the file. For example, a variable of type `PermissionType` can only be assigned 'admin', 'write', or 'read', not any arbitrary string.

#### `GET` Request Handler

```typescript
// Get all invitations for the user's workspaces
export async function GET(req: NextRequest) {
```
*   **Purpose:** Defines the `GET` HTTP request handler for this API route.
*   **Explanation:** This function will be executed when an HTTP `GET` request is made to this API endpoint. It's an `async` function because it performs asynchronous operations like database queries and session retrieval. `req: NextRequest` indicates that it receives the Next.js request object.

```typescript
  const session = await getSession()
```
*   **Purpose:** Retrieves the current user's session information.
*   **Explanation:** This calls the `getSession` utility function (imported earlier) to check if a user is logged in and to get their authenticated session details, including their `user.id`.

```typescript
  if (!session?.user?.id) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
  }
```
*   **Purpose:** Implements an authorization check.
*   **Explanation:** If `session` is `null` or `undefined`, or if `session.user` or `session.user.id` is missing, it means no authenticated user is making the request. In this case, the server immediately responds with a JSON error message `'Unauthorized'` and an HTTP status code `401` (Unauthorized), preventing unauthenticated access.

```typescript
  try {
```
*   **Purpose:** Starts a `try` block to handle potential errors during the execution of the `GET` request logic.

```typescript
    // Get all workspaces where the user has permissions
    const userWorkspaces = await db
      .select({ id: workspace.id })
      .from(workspace)
      .innerJoin(
        permissions,
        and(
          eq(permissions.entityId, workspace.id),
          eq(permissions.entityType, 'workspace'),
          eq(permissions.userId, session.user.id)
        )
      )
```
*   **Purpose:** Queries the database to find all workspaces where the currently authenticated user has any kind of permission.
*   **Explanation:**
    *   `db.select({ id: workspace.id })`: Specifies that only the `id` column from the `workspace` table should be returned.
    *   `.from(workspace)`: Indicates that the primary table for this query is `workspace`.
    *   `.innerJoin(...)`: Performs an inner join with the `permissions` table. An `innerJoin` means only rows that have a match in both tables based on the join conditions will be included.
    *   `permissions`: The table being joined.
    *   `and(...)`: Combines multiple conditions for the join:
        *   `eq(permissions.entityId, workspace.id)`: Links permissions to workspaces by matching the `entityId` in `permissions` with the `id` in `workspace`.
        *   `eq(permissions.entityType, 'workspace')`: Ensures that the permission entry specifically relates to a workspace (as permissions might apply to other entities).
        *   `eq(permissions.userId, session.user.id)`: Filters permissions to only those belonging to the currently logged-in user.
    *   The result `userWorkspaces` will be an array of objects, each containing the `id` of a workspace the user is associated with.

```typescript
    if (userWorkspaces.length === 0) {
      return NextResponse.json({ invitations: [] })
    }
```
*   **Purpose:** Handles the case where the user is not a member of any workspace.
*   **Explanation:** If the `userWorkspaces` array is empty, it means the user has no permissions in any workspace. In this scenario, there would be no invitations relevant to them, so the function returns an empty `invitations` array with a `200 OK` status.

```typescript
    // Get all workspaceIds where the user is a member
    const workspaceIds = userWorkspaces.map((w) => w.id)
```
*   **Purpose:** Extracts just the workspace IDs from the `userWorkspaces` array.
*   **Explanation:** The previous query returned objects like `{ id: 'some-uuid' }`. This line uses the `map` array method to transform that array into a simpler array containing only the UUIDs, which is more convenient for the next database query.

```typescript
    // Find all invitations for those workspaces
    const invitations = await db
      .select()
      .from(workspaceInvitation)
      .where(inArray(workspaceInvitation.workspaceId, workspaceIds))
```
*   **Purpose:** Queries the database to find all workspace invitations that belong to the workspaces identified in the previous step.
*   **Explanation:**
    *   `db.select()`: Selects all columns from the target table.
    *   `.from(workspaceInvitation)`: Specifies the `workspaceInvitation` table as the source.
    *   `.where(inArray(workspaceInvitation.workspaceId, workspaceIds))`: Filters the `workspaceInvitation` records. It selects only those invitations where the `workspaceId` column matches any of the IDs present in the `workspaceIds` array (i.e., invitations for the user's workspaces).

```typescript
    return NextResponse.json({ invitations })
```
*   **Purpose:** Returns the fetched invitations as a JSON response.
*   **Explanation:** If the queries are successful, this line sends a `200 OK` response containing a JSON object with a key `invitations` whose value is the array of invitation records found.

```typescript
  } catch (error) {
    logger.error('Error fetching workspace invitations:', error)
    return NextResponse.json({ error: 'Failed to fetch invitations' }, { status: 500 })
  }
}
```
*   **Purpose:** Catches and handles any exceptions that occur during the `GET` request.
*   **Explanation:** If any error (e.g., database connection issue, query error) happens within the `try` block, execution jumps here.
    *   `logger.error(...)`: The error is logged using the `logger` instance for debugging.
    *   `return NextResponse.json(...)`: A generic error message `'Failed to fetch invitations'` is returned to the client with an HTTP `500` status code (Internal Server Error) to avoid exposing sensitive internal error details.

#### `POST` Request Handler

```typescript
// Create a new invitation
export async function POST(req: NextRequest) {
```
*   **Purpose:** Defines the `POST` HTTP request handler for this API route.
*   **Explanation:** This function will be executed when an HTTP `POST` request is made to this API endpoint. It's an `async` function, accepting the `NextRequest` object.

```typescript
  const session = await getSession()

  if (!session?.user?.id) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
  }
```
*   **Purpose:** Performs the same authorization check as the `GET` handler.
*   **Explanation:** Ensures that only authenticated users can attempt to create new invitations, responding with `401 Unauthorized` if not.

```typescript
  try {
```
*   **Purpose:** Starts a `try` block to handle potential errors during the execution of the `POST` request logic.

```typescript
    const { workspaceId, email, role = 'member', permission = 'read' } = await req.json()
```
*   **Purpose:** Parses the JSON body of the incoming `POST` request.
*   **Explanation:** `req.json()` asynchronously reads and parses the request body as JSON. It then destructures the required fields: `workspaceId`, `email`, and optionally `role` (defaulting to `'member'`) and `permission` (defaulting to `'read'`).

```typescript
    if (!workspaceId || !email) {
      return NextResponse.json({ error: 'Workspace ID and email are required' }, { status: 400 })
    }
```
*   **Purpose:** Basic input validation.
*   **Explanation:** Checks if the essential `workspaceId` and `email` fields were provided in the request body. If either is missing, it returns a `400 Bad Request` error.

```typescript
    // Validate permission type
    const validPermissions: PermissionType[] = ['admin', 'write', 'read']
    if (!validPermissions.includes(permission)) {
      return NextResponse.json(
        { error: `Invalid permission: must be one of ${validPermissions.join(', ')}` },
        { status: 400 }
      )
    }
```
*   **Purpose:** Validates that the provided `permission` string is one of the allowed types.
*   **Explanation:**
    *   `validPermissions`: An array explicitly listing the acceptable permission strings.
    *   `!validPermissions.includes(permission)`: Checks if the `permission` received from the request body is *not* found in the `validPermissions` array.
    *   If invalid, it returns a `400 Bad Request` error, informing the client of the valid options.

```typescript
    // Check if user has admin permissions for this workspace
    const userPermission = await db
      .select()
      .from(permissions)
      .where(
        and(
          eq(permissions.entityId, workspaceId),
          eq(permissions.entityType, 'workspace'),
          eq(permissions.userId, session.user.id),
          eq(permissions.permissionType, 'admin')
        )
      )
      .then((rows) => rows[0])
```
*   **Purpose:** Verifies that the authenticated user attempting to create the invitation has 'admin' permissions for the specified `workspaceId`.
*   **Explanation:**
    *   `db.select().from(permissions)`: Queries the `permissions` table.
    *   `where(and(...))`: Filters the permissions based on multiple conditions:
        *   `eq(permissions.entityId, workspaceId)`: The permission must be for the target workspace.
        *   `eq(permissions.entityType, 'workspace')`: The permission must be for a workspace entity.
        *   `eq(permissions.userId, session.user.id)`: The permission must belong to the current user.
        *   `eq(permissions.permissionType, 'admin')`: **Crucially**, the user must specifically have `admin` permission.
    *   `.then((rows) => rows[0])`: Drizzle queries return an array of results. This takes the first element of that array (which will be `undefined` if no matching permission is found).

```typescript
    if (!userPermission) {
      return NextResponse.json(
        { error: 'You need admin permissions to invite users' },
        { status: 403 }
      )
    }
```
*   **Purpose:** Denies the request if the inviting user is not an admin.
*   **Explanation:** If `userPermission` is `undefined` (meaning the user doesn't have admin rights for that workspace), it returns a `403 Forbidden` error, indicating the user does not have the necessary permissions to perform this action.

```typescript
    // Get the workspace details for the email
    const workspaceDetails = await db
      .select()
      .from(workspace)
      .where(eq(workspace.id, workspaceId))
      .then((rows) => rows[0])
```
*   **Purpose:** Fetches the details of the target workspace, primarily its `name`, to be used in the invitation email.
*   **Explanation:**
    *   `db.select().from(workspace)`: Queries the `workspace` table.
    *   `where(eq(workspace.id, workspaceId))`: Filters to find the specific workspace by its ID.
    *   `.then((rows) => rows[0])`: Retrieves the first (and only expected) matching workspace record.

```typescript
    if (!workspaceDetails) {
      return NextResponse.json({ error: 'Workspace not found' }, { status: 404 })
    }
```
*   **Purpose:** Handles the case where the specified `workspaceId` does not correspond to an existing workspace.
*   **Explanation:** If `workspaceDetails` is `undefined`, it means the `workspaceId` provided in the request body is invalid or refers to a non-existent workspace. A `404 Not Found` error is returned.

```typescript
    // Check if the user is already a member
    // First find if a user with this email exists
    const existingUser = await db
      .select()
      .from(user)
      .where(eq(user.email, email))
      .then((rows) => rows[0])
```
*   **Purpose:** Checks if a user account already exists in the system with the invited email address.
*   **Explanation:**
    *   `db.select().from(user)`: Queries the `user` table.
    *   `where(eq(user.email, email))`: Filters to find a user record whose `email` matches the `email` from the request.
    *   `.then((rows) => rows[0])`: Retrieves the first matching user record, if any.

```typescript
    if (existingUser) {
      // Check if the user already has permissions for this workspace
      const existingPermission = await db
        .select()
        .from(permissions)
        .where(
          and(
            eq(permissions.entityId, workspaceId),
            eq(permissions.entityType, 'workspace'),
            eq(permissions.userId, existingUser.id)
          )
        )
        .then((rows) => rows[0])

      if (existingPermission) {
        return NextResponse.json(
          {
            error: `${email} already has access to this workspace`,
            email,
          },
          { status: 400 }
        )
      }
    }
```
*   **Purpose:** If a user with the invited email *already exists*, this block checks if they are *already a member* of the target workspace.
*   **Explanation:**
    *   `if (existingUser)`: This block only executes if `existingUser` was found in the previous step.
    *   `db.select().from(permissions).where(and(...))`: Queries the `permissions` table again, but this time:
        *   `eq(permissions.userId, existingUser.id)`: It checks for permissions belonging to the `existingUser` (not the inviter).
        *   It still checks `entityId` and `entityType` for the target `workspaceId`.
    *   `.then((rows) => rows[0])`: Retrieves the permission record if the `existingUser` already has permissions for this workspace.
    *   `if (existingPermission)`: If a permission record is found, it means the invited user is already a member. A `400 Bad Request` error is returned to prevent inviting someone who already has access.

```typescript
    // Check if there's already a pending invitation
    const existingInvitation = await db
      .select()
      .from(workspaceInvitation)
      .where(
        and(
          eq(workspaceInvitation.workspaceId, workspaceId),
          eq(workspaceInvitation.email, email),
          eq(workspaceInvitation.status, 'pending' as WorkspaceInvitationStatus)
        )
      )
      .then((rows) => rows[0])
```
*   **Purpose:** Prevents sending duplicate invitations to the same email for the same workspace if an invitation is already pending.
*   **Explanation:**
    *   `db.select().from(workspaceInvitation)`: Queries the `workspaceInvitation` table.
    *   `where(and(...))`: Filters invitations by:
        *   `eq(workspaceInvitation.workspaceId, workspaceId)`: For the specific target workspace.
        *   `eq(workspaceInvitation.email, email)`: For the specific invited email address.
        *   `eq(workspaceInvitation.status, 'pending' as WorkspaceInvitationStatus)`: **Crucially**, only considers invitations that are still in a `pending` state. The type assertion `as WorkspaceInvitationStatus` is used here because Drizzle might infer a broader string type, and we want to enforce the specific enum value.
    *   `.then((rows) => rows[0])`: Retrieves the first matching pending invitation, if any.

```typescript
    if (existingInvitation) {
      return NextResponse.json(
        {
          error: `${email} has already been invited to this workspace`,
          email,
        },
        { status: 400 }
      )
    }
```
*   **Purpose:** Responds with an error if a pending invitation already exists.
*   **Explanation:** If `existingInvitation` is found, it means an invitation has already been sent and is awaiting acceptance. A `400 Bad Request` error is returned to prevent redundant invitations.

```typescript
    // Generate a unique token and set expiry date (1 week from now)
    const token = randomUUID()
```
*   **Purpose:** Generates a secure, unique token for the new invitation.
*   **Explanation:** This `token` will be part of the invitation link sent in the email. It's crucial for verifying the invitation's authenticity when the user clicks the link.

```typescript
    const expiresAt = new Date()
    expiresAt.setDate(expiresAt.getDate() + 7) // 7 days expiry
```
*   **Purpose:** Calculates the expiration date for the invitation.
*   **Explanation:**
    *   `new Date()`: Creates a `Date` object representing the current time.
    *   `expiresAt.setDate(expiresAt.getDate() + 7)`: Modifies the `expiresAt` date to be exactly 7 days (one week) from the current date. After this date, the invitation token should no longer be valid.

```typescript
    // Create the invitation
    const invitationData = {
      id: randomUUID(),
      workspaceId,
      email,
      inviterId: session.user.id,
      role,
      status: 'pending' as WorkspaceInvitationStatus,
      token,
      permissions: permission,
      expiresAt,
      createdAt: new Date(),
      updatedAt: new Date(),
    }
```
*   **Purpose:** Assembles the data for the new `workspaceInvitation` record.
*   **Explanation:** This object contains all the necessary fields for a new invitation:
    *   `id: randomUUID()`: A unique ID for the invitation *record itself* in the database.
    *   `workspaceId`: The ID of the workspace the user is invited to.
    *   `email`: The email of the person being invited.
    *   `inviterId: session.user.id`: The ID of the user who sent the invitation.
    *   `role`: The role assigned to the invitee (e.g., 'member').
    *   `status: 'pending' as WorkspaceInvitationStatus`: The initial status of the invitation. The type assertion `as WorkspaceInvitationStatus` ensures type safety with the Drizzle schema.
    *   `token`: The unique token generated earlier, used for the invitation link.
    *   `permissions`: The specific permission level (e.g., 'read', 'write', 'admin') granted to the invitee.
    *   `expiresAt`: The calculated expiration date.
    *   `createdAt`, `updatedAt`: Timestamps for when the record was created and last updated.

```typescript
    // Create invitation
    await db.insert(workspaceInvitation).values(invitationData)
```
*   **Purpose:** Inserts the new invitation record into the database.
*   **Explanation:**
    *   `db.insert(workspaceInvitation)`: Prepares an insert operation on the `workspaceInvitation` table.
    *   `.values(invitationData)`: Specifies the data (`invitationData` object) to be inserted into the table.

```typescript
    // Send the invitation email
    await sendInvitationEmail({
      to: email,
      inviterName: session.user.name || session.user.email || 'A user',
      workspaceName: workspaceDetails.name,
      invitationId: invitationData.id,
      token: token,
    })
```
*   **Purpose:** Calls a helper function to send the invitation email to the invited person.
*   **Explanation:** This line invokes the `sendInvitationEmail` function (defined later in the file) with all the necessary details to construct and send a personalized email:
    *   `to`: The email address of the invitee.
    *   `inviterName`: The name of the person who sent the invite, falling back to their email or a generic 'A user' if the name isn't available.
    *   `workspaceName`: The name of the workspace they are invited to.
    *   `invitationId`: The unique ID of the invitation record itself (used in the link).
    *   `token`: The secure, unique token for validating the invitation (used in the link).

```typescript
    return NextResponse.json({ success: true, invitation: invitationData })
```
*   **Purpose:** Returns a success response to the client.
*   **Explanation:** If all steps (validation, database insertion, email sending) are successful, this sends a `200 OK` response with a JSON object indicating `success: true` and includes the full `invitationData` that was just created.

```typescript
  } catch (error) {
    logger.error('Error creating workspace invitation:', error)
    return NextResponse.json({ error: 'Failed to create invitation' }, { status: 500 })
  }
}
```
*   **Purpose:** Catches and handles any exceptions that occur during the `POST` request.
*   **Explanation:** Similar to the `GET` handler's `catch` block, this logs any errors and returns a generic `500 Internal Server Error` to the client.

#### `sendInvitationEmail` Helper Function

```typescript
// Helper function to send invitation email using the Resend API
async function sendInvitationEmail({
  to,
  inviterName,
  workspaceName,
  invitationId,
  token,
}: {
  to: string
  inviterName: string
  workspaceName: string
  invitationId: string
  token: string
}) {
```
*   **Purpose:** Defines an asynchronous helper function to encapsulate the logic for sending an invitation email.
*   **Explanation:** This function is `async` because email sending is an asynchronous operation. It accepts a single object parameter with type annotations, ensuring all required email details are provided: `to` (recipient email), `inviterName`, `workspaceName`, `invitationId`, and `token`.

```typescript
  try {
```
*   **Purpose:** Starts a `try` block to handle potential errors specifically during the email sending process.

```typescript
    const baseUrl = getBaseUrl()
    // Use invitation ID in path, token in query parameter for security
    const invitationLink = `${baseUrl}/invite/${invitationId}?token=${token}`
```
*   **Purpose:** Constructs the full, unique invitation URL.
*   **Explanation:**
    *   `const baseUrl = getBaseUrl()`: Retrieves the application's root URL (e.g., `https://app.example.com`).
    *   `const invitationLink = ...`: Concatenates the base URL, a fixed path segment (`/invite/`), the `invitationId` (to identify the specific invitation), and the `token` as a URL query parameter (`?token=...`). This structure is important for security: the `invitationId` can be public, but the `token` must be kept secret and validated upon acceptance.

```typescript
    const emailHtml = await render(
      WorkspaceInvitationEmail({
        workspaceName,
        inviterName,
        invitationLink,
      })
    )
```
*   **Purpose:** Renders the React email component into an HTML string.
*   **Explanation:**
    *   `WorkspaceInvitationEmail({...})`: Creates an instance of the React email component, passing in the necessary data (workspace name, inviter name, and the generated invitation link) as props.
    *   `await render(...)`: Uses the `render` function from `@react-email/render` to convert this React component into its final HTML representation, which is stored in `emailHtml`.

```typescript
    const fromAddress = getFromEmailAddress()
```
*   **Purpose:** Retrieves the sender's email address.
*   **Explanation:** Calls the utility function to get the configured "from" email address for sending emails.

```typescript
    logger.info(`Attempting to send email from ${fromAddress} to ${to}`)
```
*   **Purpose:** Logs the attempt to send an email.
*   **Explanation:** This informational log helps track email sending activities and provides context for potential issues.

```typescript
    const result = await sendEmail({
      to,
      subject: `You've been invited to join "${workspaceName}" on Sim`,
      html: emailHtml,
      from: fromAddress,
      emailType: 'transactional',
    })
```
*   **Purpose:** Sends the actual email using the configured mailer utility.
*   **Explanation:**
    *   `await sendEmail(...)`: Calls the `sendEmail` utility function with all the required email parameters:
        *   `to`: Recipient's email address.
        *   `subject`: The email subject line, dynamically including the `workspaceName`.
        *   `html`: The HTML content generated by `render`.
        *   `from`: The sender's email address.
        *   `emailType: 'transactional'`: A label for the email type, often used by email services for categorization, tracking, or specific sending policies (e.g., to distinguish from marketing emails).

```typescript
    if (result.success) {
      logger.info(`Invitation email sent successfully to ${to}`, { result })
    } else {
      logger.error(`Failed to send invitation email to ${to}`, { error: result.message })
    }
```
*   **Purpose:** Logs the outcome of the email sending attempt.
*   **Explanation:** Checks the `success` property of the `result` object returned by `sendEmail`.
    *   If `true`, it logs a success message with additional details from the `result`.
    *   If `false`, it logs an error message, including the error message from the `result` object.

```typescript
  } catch (error) {
    logger.error('Error sending invitation email:', error)
    // Continue even if email fails - the invitation is still created
  }
}
```
*   **Purpose:** Catches and logs any errors that occur during the email sending process (e.g., network issues, mailer misconfiguration).
*   **Explanation:**
    *   `logger.error(...)`: Logs the error.
    *   **Important Comment:** The comment `// Continue even if email fails - the invitation is still created` highlights a design decision. Even if the email fails to send, the invitation record has already been successfully created in the database. This means the invitation is technically valid, and the user could be resent the email later or manually accept. The system prioritizes database consistency over immediate email delivery.

---

### Summary

This file provides a robust API for managing workspace invitations. It enforces authentication and authorization, validates inputs, prevents duplicate entries, and integrates securely with a database and email service. The `GET` endpoint facilitates listing relevant invitations, while the `POST` endpoint handles the entire invitation creation workflow, from validation to database persistence and email delivery.