As a TypeScript expert and technical writer, I'll walk you through this Next.js API route file, explaining its purpose, simplifying its logic, and detailing each significant line of code.

---

## Explanation of `app/api/webhooks/[id]/route.ts`

This TypeScript file defines a Next.js API route that handles HTTP requests for a **single webhook resource**, identified by its unique ID. It acts as a RESTful endpoint, allowing clients to:

1.  **Retrieve** details of a specific webhook (`GET` request).
2.  **Update** existing properties of a webhook (`PATCH` request).
3.  **Delete** a webhook (`DELETE` request).

It integrates with a database (using Drizzle ORM), manages user authentication and authorization, and includes specific logic for interacting with external services (like Airtable, Microsoft Teams, and Telegram) when a webhook linked to them is deleted.

---

### Key Concepts Used in This File

Before diving into the line-by-line explanation, let's understand some recurring patterns:

*   **Next.js API Routes:** This file is a "Route Handler" in Next.js. Functions like `GET`, `PATCH`, `DELETE` are automatically invoked when an HTTP request matching the route (`/api/webhooks/[id]`) comes in. The `[id]` part means `id` is a dynamic segment, accessible via `params`.
*   **Drizzle ORM:** An object-relational mapper (ORM) for TypeScript that allows you to interact with your database using TypeScript syntax instead of raw SQL. `db`, `webhook`, `workflow`, `eq` are all part of Drizzle.
*   **Authentication (`getSession`):** Verifies the identity of the user making the request.
*   **Authorization (`getUserEntityPermissions`):** Determines if the authenticated user has the necessary rights to perform the requested action on the specific resource (webhook). This often involves checking if the user owns the associated workflow or has permissions within a workspace.
*   **Logging (`createLogger`):** For tracking the flow of operations and debugging potential issues. Each request is assigned a `requestId` for easy tracing.
*   **External Service Integrations:** The `DELETE` endpoint contains specific logic to clean up corresponding webhooks in third-party services (Airtable, Microsoft Teams, Telegram) when they are removed from the internal database. This is crucial for data consistency and avoiding orphaned webhooks.

---

### Line-by-Line Explanation

```typescript
import { db } from '@sim/db'
import { webhook, workflow } from '@sim/db/schema'
import { eq } from 'drizzle-orm'
import { type NextRequest, NextResponse } from 'next/server'
import { getSession } from '@/lib/auth'
import { createLogger } from '@/lib/logs/console/logger'
import { getUserEntityPermissions } from '@/lib/permissions/utils'
import { getBaseUrl } from '@/lib/urls/utils'
import { generateRequestId } from '@/lib/utils'
import { getOAuthToken } from '@/app/api/auth/oauth/utils'
```

*   **`import { db } from '@sim/db'`**: Imports the database client instance, likely configured to connect to your PostgreSQL or other SQL database. `db` is the main object used to interact with the database via Drizzle ORM.
*   **`import { webhook, workflow } from '@sim/db/schema'`**: Imports the Drizzle ORM schema definitions for the `webhook` and `workflow` tables. These objects represent your database tables in TypeScript, allowing you to build queries against them.
*   **`import { eq } from 'drizzle-orm'`**: Imports the `eq` (equals) function from Drizzle ORM. This is used in `where` clauses to specify equality conditions (e.g., `WHERE id = 'some_id'`).
*   **`import { type NextRequest, NextResponse } from 'next/server'`**: Imports types and classes from Next.js server utilities.
    *   `NextRequest`: Represents the incoming HTTP request, providing access to headers, body, query parameters, etc.
    *   `NextResponse`: Used to construct and send the HTTP response back to the client.
*   **`import { getSession } from '@/lib/auth'`**: Imports a utility function `getSession`. This function is responsible for retrieving the current user's authentication session, which typically contains user ID and other session-related data.
*   **`import { createLogger } from '@/lib/logs/console/logger'`**: Imports a function to create a logger instance. This is part of a custom logging system.
*   **`import { getUserEntityPermissions } from '@/lib/permissions/utils'`**: Imports a utility function to check a user's permissions on a specific entity (like a workspace). This is central to the authorization logic.
*   **`import { getBaseUrl } from '@/lib/urls/utils'`**: Imports a utility to get the base URL of the application. This is often used when constructing callback URLs for external services.
*   **`import { generateRequestId } from '@/lib/utils'`**: Imports a utility function to generate a unique request ID. This ID is used for tracing logs related to a specific incoming request, making debugging easier.
*   **`import { getOAuthToken } from '@/app/api/auth/oauth/utils'`**: Imports a utility function to retrieve an OAuth access token for a specific user and provider. This is essential for making authenticated calls to third-party APIs (like Airtable).

---

```typescript
const logger = createLogger('WebhookAPI')

export const dynamic = 'force-dynamic'
```

*   **`const logger = createLogger('WebhookAPI')`**: Initializes a logger instance specifically named 'WebhookAPI'. This helps categorize logs originating from this file.
*   **`export const dynamic = 'force-dynamic'`**: This is a Next.js specific export. It instructs Next.js to treat this route as a "dynamic" route, meaning it should not be cached statically. Each request will execute the code, ensuring fresh data and behavior. This is crucial for API routes that handle dynamic data and user-specific actions.

---

### `GET` Endpoint: Retrieve a Specific Webhook

This function handles `GET` requests to `/api/webhooks/[id]`, allowing a client to fetch the details of a single webhook by its ID. It includes robust authentication, authorization, and error handling.

```typescript
// Get a specific webhook
export async function GET(request: NextRequest, { params }: { params: Promise<{ id: string }> }) {
  const requestId = generateRequestId()

  try {
    const { id } = await params
    logger.debug(`[${requestId}] Fetching webhook with ID: ${id}`)

    const session = await getSession()
    if (!session?.user?.id) {
      logger.warn(`[${requestId}] Unauthorized webhook access attempt`)
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
    }

    const webhooks = await db
      .select({
        webhook: webhook,
        workflow: {
          id: workflow.id,
          name: workflow.name,
          userId: workflow.userId,
          workspaceId: workflow.workspaceId,
        },
      })
      .from(webhook)
      .innerJoin(workflow, eq(webhook.workflowId, workflow.id))
      .where(eq(webhook.id, id))
      .limit(1)

    if (webhooks.length === 0) {
      logger.warn(`[${requestId}] Webhook not found: ${id}`)
      return NextResponse.json({ error: 'Webhook not found' }, { status: 404 })
    }

    const webhookData = webhooks[0]

    // Check if user has permission to access this webhook
    let hasAccess = false

    // Case 1: User owns the workflow
    if (webhookData.workflow.userId === session.user.id) {
      hasAccess = true
    }

    // Case 2: Workflow belongs to a workspace and user has any permission
    if (!hasAccess && webhookData.workflow.workspaceId) {
      const userPermission = await getUserEntityPermissions(
        session.user.id,
        'workspace',
        webhookData.workflow.workspaceId
      )
      if (userPermission !== null) {
        hasAccess = true
      }
    }

    if (!hasAccess) {
      logger.warn(`[${requestId}] User ${session.user.id} denied access to webhook: ${id}`)
      return NextResponse.json({ error: 'Access denied' }, { status: 403 })
    }

    logger.info(`[${requestId}] Successfully retrieved webhook: ${id}`)
    return NextResponse.json({ webhook: webhooks[0] }, { status: 200 })
  } catch (error) {
    logger.error(`[${requestId}] Error fetching webhook`, error)
    return NextResponse.json({ error: 'Internal server error' }, { status: 500 })
  }
}
```

*   **`export async function GET(request: NextRequest, { params }: { params: Promise<{ id: string }> })`**: Defines an asynchronous function `GET` that Next.js will call for `GET` requests.
    *   `request: NextRequest`: The incoming HTTP request object.
    *   `{ params }: { params: Promise<{ id: string }> }`: Extracts the `params` object, which contains dynamic route segments. For this route, `[id]` means `params` will have an `id` property (e.g., if the URL is `/api/webhooks/123`, `id` will be `'123'`). The `Promise` indicates that Next.js might resolve these parameters asynchronously.
*   **`const requestId = generateRequestId()`**: Generates a unique ID for this specific request, useful for tracing logs.
*   **`try { ... } catch (error) { ... }`**: A standard `try-catch` block to gracefully handle any errors that occur during the execution of the function.
*   **`const { id } = await params`**: Awaits the `params` promise to resolve and then destructures the `id` property from it. This `id` is the webhook ID from the URL.
*   **`logger.debug(...)`**: Logs a debug message indicating the start of the webhook fetching process with the `requestId` and webhook ID.
*   **`const session = await getSession()`**: Retrieves the current user's session information.
*   **`if (!session?.user?.id)`**: Checks if a session exists and if the user is authenticated (i.e., has a user ID).
    *   `session?.user?.id`: Uses optional chaining (`?.`) to safely access nested properties, preventing errors if `session` or `session.user` is null/undefined.
*   **`logger.warn(...)`**: Logs a warning if the user is not authenticated.
*   **`return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })`**: Sends a JSON response with an "Unauthorized" error and an HTTP status code `401`.
*   **`const webhooks = await db ... .limit(1)`**: This is a Drizzle ORM query to fetch the webhook and its associated workflow information.
    *   **`.select({ webhook: webhook, workflow: { ... } })`**: Specifies which columns/data to retrieve. It selects the entire `webhook` object and *specific* fields from the `workflow` table (ID, name, user ID, workspace ID). This is efficient as it only fetches necessary `workflow` data.
    *   **`.from(webhook)`**: Specifies that the primary table for the query is `webhook`.
    *   **`.innerJoin(workflow, eq(webhook.workflowId, workflow.id))`**: Joins the `webhook` table with the `workflow` table. The join condition is where `webhook.workflowId` matches `workflow.id`. This links a webhook to its parent workflow.
    *   **`.where(eq(webhook.id, id))`**: Filters the results to only include the webhook whose `id` column matches the `id` extracted from the URL.
    *   **`.limit(1)`**: Ensures that the query returns at most one result, as we're expecting a single webhook for a given ID.
*   **`if (webhooks.length === 0)`**: Checks if no webhook was found with the given ID.
*   **`logger.warn(...)`**: Logs a warning if the webhook is not found.
*   **`return NextResponse.json({ error: 'Webhook not found' }, { status: 404 })`**: Sends a "Webhook not found" error with an HTTP status code `404`.
*   **`const webhookData = webhooks[0]`**: Extracts the first (and only) webhook record found from the query result.
*   **`let hasAccess = false`**: Initializes a boolean flag to track user access permission.
*   **`if (webhookData.workflow.userId === session.user.id)`**: **Authorization Check (Case 1: Workflow Owner)**. If the user ID from the session matches the `userId` of the workflow associated with the webhook, the user is considered to have access.
*   **`if (!hasAccess && webhookData.workflow.workspaceId)`**: **Authorization Check (Case 2: Workspace Permissions)**. This block runs *only if* the user isn't the owner and *if* the workflow belongs to a workspace (`workspaceId` is present).
    *   **`const userPermission = await getUserEntityPermissions(session.user.id, 'workspace', webhookData.workflow.workspaceId)`**: Calls a utility function to get the user's permissions for the specific `workspace`.
    *   **`if (userPermission !== null)`**: If `getUserEntityPermissions` returns anything other than `null` (meaning the user has *some* permission – read, write, or admin), then `hasAccess` is set to `true`.
*   **`if (!hasAccess)`**: If, after both checks, `hasAccess` is still `false`.
*   **`logger.warn(...)`**: Logs a warning indicating an access denial.
*   **`return NextResponse.json({ error: 'Access denied' }, { status: 403 })`**: Sends an "Access denied" error with an HTTP status code `403` (Forbidden).
*   **`logger.info(...)`**: Logs an info message upon successful retrieval of the webhook.
*   **`return NextResponse.json({ webhook: webhooks[0] }, { status: 200 })`**: Sends the retrieved webhook data in a JSON response with an HTTP status code `200` (OK).
*   **`catch (error)`**: Catches any unexpected errors.
*   **`logger.error(...)`**: Logs the error with the `requestId`.
*   **`return NextResponse.json({ error: 'Internal server error' }, { status: 500 })`**: Sends a generic "Internal server error" with an HTTP status code `500`.

---

### `PATCH` Endpoint: Update a Webhook

This function handles `PATCH` requests to `/api/webhooks/[id]`, allowing a client to update one or more properties of an existing webhook. It also includes authentication, stricter authorization (requiring write/admin permissions), and robust error handling.

```typescript
// Update a webhook
export async function PATCH(request: NextRequest, { params }: { params: Promise<{ id: string }> }) {
  const requestId = generateRequestId()

  try {
    const { id } = await params
    logger.debug(`[${requestId}] Updating webhook with ID: ${id}`)

    const session = await getSession()
    if (!session?.user?.id) {
      logger.warn(`[${requestId}] Unauthorized webhook update attempt`)
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
    }

    const body = await request.json()
    const { path, provider, providerConfig, isActive } = body

    // Find the webhook and check permissions
    const webhooks = await db
      .select({
        webhook: webhook,
        workflow: {
          id: workflow.id,
          userId: workflow.userId,
          workspaceId: workflow.workspaceId,
        },
      })
      .from(webhook)
      .innerJoin(workflow, eq(webhook.workflowId, workflow.id))
      .where(eq(webhook.id, id))
      .limit(1)

    if (webhooks.length === 0) {
      logger.warn(`[${requestId}] Webhook not found: ${id}`)
      return NextResponse.json({ error: 'Webhook not found' }, { status: 404 })
    }

    const webhookData = webhooks[0]

    // Check if user has permission to modify this webhook
    let canModify = false

    // Case 1: User owns the workflow
    if (webhookData.workflow.userId === session.user.id) {
      canModify = true
    }

    // Case 2: Workflow belongs to a workspace and user has write or admin permission
    if (!canModify && webhookData.workflow.workspaceId) {
      const userPermission = await getUserEntityPermissions(
        session.user.id,
        'workspace',
        webhookData.workflow.workspaceId
      )
      if (userPermission === 'write' || userPermission === 'admin') {
        canModify = true
      }
    }

    if (!canModify) {
      logger.warn(
        `[${requestId}] User ${session.user.id} denied permission to modify webhook: ${id}`
      )
      return NextResponse.json({ error: 'Access denied' }, { status: 403 })
    }

    logger.debug(`[${requestId}] Updating webhook properties`, {
      hasPathUpdate: path !== undefined,
      hasProviderUpdate: provider !== undefined,
      hasConfigUpdate: providerConfig !== undefined,
      hasActiveUpdate: isActive !== undefined,
    })

    // Update the webhook
    const updatedWebhook = await db
      .update(webhook)
      .set({
        path: path !== undefined ? path : webhooks[0].webhook.path,
        provider: provider !== undefined ? provider : webhooks[0].webhook.provider,
        providerConfig:
          providerConfig !== undefined ? providerConfig : webhooks[0].webhook.providerConfig,
        isActive: isActive !== undefined ? isActive : webhooks[0].webhook.isActive,
        updatedAt: new Date(),
      })
      .where(eq(webhook.id, id))
      .returning()

    logger.info(`[${requestId}] Successfully updated webhook: ${id}`)
    return NextResponse.json({ webhook: updatedWebhook[0] }, { status: 200 })
  } catch (error) {
    logger.error(`[${requestId}] Error updating webhook`, error)
    return NextResponse.json({ error: 'Internal server error' }, { status: 500 })
  }
}
```

*   **`export async function PATCH(...)`**: Defines the asynchronous function for `PATCH` requests. The signature is identical to `GET`.
*   **`const requestId = generateRequestId()`**: Same as `GET`, generates a request ID.
*   **`try { ... } catch (error) { ... }`**: Standard error handling.
*   **`const { id } = await params`**: Extracts the webhook ID from the URL.
*   **`logger.debug(...)`**: Logs the start of the update operation.
*   **`const session = await getSession()` / `if (!session?.user?.id)`**: Performs the same authentication check as the `GET` endpoint.
*   **`const body = await request.json()`**: Parses the JSON body of the incoming `PATCH` request. This body contains the data to update.
*   **`const { path, provider, providerConfig, isActive } = body`**: Destructures the `body` to extract potential update fields. These fields might be optional, as `PATCH` typically supports partial updates.
*   **`const webhooks = await db ... .limit(1)`**: Performs the exact same database query as in the `GET` endpoint to retrieve the webhook and its associated workflow, crucial for permission checks.
*   **`if (webhooks.length === 0)`**: Handles the "webhook not found" scenario, similar to `GET`.
*   **`const webhookData = webhooks[0]`**: Extracts the webhook data.
*   **`let canModify = false`**: Initializes a flag for modification permissions.
*   **`if (webhookData.workflow.userId === session.user.id)`**: **Authorization Check (Case 1: Workflow Owner)**. If the user owns the workflow, they can modify its webhooks.
*   **`if (!canModify && webhookData.workflow.workspaceId)`**: **Authorization Check (Case 2: Workspace Permissions)**. This check is performed only if the user is not the owner and the workflow belongs to a workspace.
    *   **`const userPermission = await getUserEntityPermissions(...)`**: Retrieves the user's permissions for the workspace.
    *   **`if (userPermission === 'write' || userPermission === 'admin')`**: For `PATCH` (modification), the user needs explicit `'write'` or `'admin'` permission in the workspace, which is stricter than the `GET` endpoint's "any permission" rule.
*   **`if (!canModify)`**: If the user lacks modification permissions.
*   **`logger.warn(...)`**: Logs an access denial warning.
*   **`return NextResponse.json({ error: 'Access denied' }, { status: 403 })`**: Sends an "Access denied" error with `403` status.
*   **`logger.debug(...)`**: Logs which specific properties are being updated, based on whether they were provided in the request body.
*   **`const updatedWebhook = await db.update(webhook).set({ ... }).where(eq(webhook.id, id)).returning()`**: This is the Drizzle ORM query to update the webhook record.
    *   **`.update(webhook)`**: Specifies the `webhook` table for the update operation.
    *   **`.set({ ... })`**: Defines the columns to update. For each property (e.g., `path`, `provider`), it uses a ternary operator:
        *   `path !== undefined ? path : webhooks[0].webhook.path`: If `path` was provided in the request body (`!== undefined`), use the new `path` value. Otherwise, retain the `path` from the existing webhook record (`webhooks[0].webhook.path`). This allows for partial updates where only specified fields are changed.
    *   **`updatedAt: new Date()`**: Automatically sets the `updatedAt` timestamp to the current time, common practice for record modification.
    *   **`.where(eq(webhook.id, id))`**: Ensures only the specific webhook identified by `id` is updated.
    *   **`.returning()`**: Instructs Drizzle to return the *updated* record(s).
*   **`logger.info(...)`**: Logs a success message for the webhook update.
*   **`return NextResponse.json({ webhook: updatedWebhook[0] }, { status: 200 })`**: Sends the first (and only) updated webhook record in the response.
*   **`catch (error)`**: Catches unexpected errors.
*   **`logger.error(...)`**: Logs the error.
*   **`return NextResponse.json({ error: 'Internal server error' }, { status: 500 })`**: Sends a generic `500` error response.

---

### `DELETE` Endpoint: Delete a Webhook

This function handles `DELETE` requests to `/api/webhooks/[id]`. It's the most complex endpoint because, in addition to database deletion and permission checks, it includes logic to delete associated webhooks from external services like Airtable, Microsoft Teams, and Telegram to maintain data consistency.

```typescript
// Delete a webhook
export async function DELETE(
  request: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  const requestId = generateRequestId()

  try {
    const { id } = await params
    logger.debug(`[${requestId}] Deleting webhook with ID: ${id}`)

    const session = await getSession()
    if (!session?.user?.id) {
      logger.warn(`[${requestId}] Unauthorized webhook deletion attempt`)
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
    }

    // Find the webhook and check permissions
    const webhooks = await db
      .select({
        webhook: webhook,
        workflow: {
          id: workflow.id,
          userId: workflow.userId,
          workspaceId: workflow.workspaceId,
        },
      })
      .from(webhook)
      .innerJoin(workflow, eq(webhook.workflowId, workflow.id))
      .where(eq(webhook.id, id))
      .limit(1)

    if (webhooks.length === 0) {
      logger.warn(`[${requestId}] Webhook not found: ${id}`)
      return NextResponse.json({ error: 'Webhook not found' }, { status: 404 })
    }

    const webhookData = webhooks[0]

    // Check if user has permission to delete this webhook
    let canDelete = false

    // Case 1: User owns the workflow
    if (webhookData.workflow.userId === session.user.id) {
      canDelete = true
    }

    // Case 2: Workflow belongs to a workspace and user has write or admin permission
    if (!canDelete && webhookData.workflow.workspaceId) {
      const userPermission = await getUserEntityPermissions(
        session.user.id,
        'workspace',
        webhookData.workflow.workspaceId
      )
      if (userPermission === 'write' || userPermission === 'admin') {
        canDelete = true
      }
    }

    if (!canDelete) {
      logger.warn(
        `[${requestId}] User ${session.user.id} denied permission to delete webhook: ${id}`
      )
      return NextResponse.json({ error: 'Access denied' }, { status: 403 })
    }

    const foundWebhook = webhookData.webhook

    // If it's an Airtable webhook, delete it from Airtable first
    if (foundWebhook.provider === 'airtable') {
      try {
        const { baseId, externalId } = (foundWebhook.providerConfig || {}) as {
          baseId?: string
          externalId?: string
        }

        if (!baseId) {
          logger.warn(`[${requestId}] Missing baseId for Airtable webhook deletion.`, {
            webhookId: id,
          })
          return NextResponse.json(
            { error: 'Missing baseId for Airtable webhook deletion' },
            { status: 400 }
          )
        }

        // Get access token for the workflow owner
        const userIdForToken = webhookData.workflow.userId
        const accessToken = await getOAuthToken(userIdForToken, 'airtable')
        if (!accessToken) {
          logger.warn(
            `[${requestId}] Could not retrieve Airtable access token for user ${userIdForToken}. Cannot delete webhook in Airtable.`,
            { webhookId: id }
          )
          return NextResponse.json(
            { error: 'Airtable access token not found for webhook deletion' },
            { status: 401 }
          )
        }

        // Resolve externalId if missing by listing webhooks and matching our notificationUrl
        let resolvedExternalId: string | undefined = externalId

        if (!resolvedExternalId) {
          try {
            const expectedNotificationUrl = `${getBaseUrl()}/api/webhooks/trigger/${foundWebhook.path}`

            const listUrl = `https://api.airtable.com/v0/bases/${baseId}/webhooks`
            const listResp = await fetch(listUrl, {
              headers: {
                Authorization: `Bearer ${accessToken}`,
              },
            })
            const listBody = await listResp.json().catch(() => null)

            if (listResp.ok && listBody && Array.isArray(listBody.webhooks)) {
              const match = listBody.webhooks.find((w: any) => {
                const url: string | undefined = w?.notificationUrl
                if (!url) return false
                // Prefer exact match; fallback to suffix match to handle origin/host remaps
                return (
                  url === expectedNotificationUrl ||
                  url.endsWith(`/api/webhooks/trigger/${foundWebhook.path}`)
                )
              })
              if (match?.id) {
                resolvedExternalId = match.id as string
                // Persist resolved externalId for future operations
                try {
                  await db
                    .update(webhook)
                    .set({
                      providerConfig: {
                        ...(foundWebhook.providerConfig || {}),
                        externalId: resolvedExternalId,
                      },
                      updatedAt: new Date(),
                    })
                    .where(eq(webhook.id, id))
                } catch {
                  // non-fatal persistence error
                }
                logger.info(`[${requestId}] Resolved Airtable externalId by listing webhooks`, {
                  baseId,
                  externalId: resolvedExternalId,
                })
              } else {
                logger.warn(`[${requestId}] Could not resolve Airtable externalId from list`, {
                  baseId,
                  expectedNotificationUrl,
                })
              }
            } else {
              logger.warn(`[${requestId}] Failed to list Airtable webhooks to resolve externalId`, {
                baseId,
                status: listResp.status,
                body: listBody,
              })
            }
          } catch (e: any) {
            logger.warn(`[${requestId}] Error attempting to resolve Airtable externalId`, {
              error: e?.message,
            })
          }
        }

        // If still not resolvable, skip remote deletion but proceed with local delete
        if (!resolvedExternalId) {
          logger.info(
            `[${requestId}] Airtable externalId not found; skipping remote deletion and proceeding to remove local record`,
            { baseId }
          )
        }

        if (resolvedExternalId) {
          const airtableDeleteUrl = `https://api.airtable.com/v0/bases/${baseId}/webhooks/${resolvedExternalId}`
          const airtableResponse = await fetch(airtableDeleteUrl, {
            method: 'DELETE',
            headers: {
              Authorization: `Bearer ${accessToken}`,
            },
          })

          // Attempt to parse error body for better diagnostics
          if (!airtableResponse.ok) {
            let responseBody: any = null
            try {
              responseBody = await airtableResponse.json()
            } catch {
              // ignore parse errors
            }

            logger.error(
              `[${requestId}] Failed to delete Airtable webhook in Airtable. Status: ${airtableResponse.status}`,
              { baseId, externalId: resolvedExternalId, response: responseBody }
            )
            return NextResponse.json(
              {
                error: 'Failed to delete webhook from Airtable',
                details:
                  (responseBody && (responseBody.error?.message || responseBody.error)) ||
                  `Status ${airtableResponse.status}`,
              },
              { status: 500 }
            )
          }

          logger.info(`[${requestId}] Successfully deleted Airtable webhook in Airtable`, {
            baseId,
            externalId: resolvedExternalId,
          })
        }
      } catch (error: any) {
        logger.error(`[${requestId}] Error deleting Airtable webhook`, {
          webhookId: id,
          error: error.message,
          stack: error.stack,
        })
        return NextResponse.json(
          { error: 'Failed to delete webhook from Airtable', details: error.message },
          { status: 500 }
        )
      }
    }

    // Delete Microsoft Teams subscription if applicable
    if (foundWebhook.provider === 'microsoftteams') {
      const { deleteTeamsSubscription } = await import('@/lib/webhooks/webhook-helpers')
      logger.info(`[${requestId}] Deleting Teams subscription for webhook ${id}`)
      await deleteTeamsSubscription(foundWebhook, webhookData.workflow, requestId)
      // Don't fail webhook deletion if subscription cleanup fails
    }

    // Delete Telegram webhook if applicable
    if (foundWebhook.provider === 'telegram') {
      try {
        const { botToken } = (foundWebhook.providerConfig || {}) as { botToken?: string }

        if (!botToken) {
          logger.warn(`[${requestId}] Missing botToken for Telegram webhook deletion.`, {
            webhookId: id,
          })
          return NextResponse.json(
            { error: 'Missing botToken for Telegram webhook deletion' },
            { status: 400 }
          )
        }

        const telegramApiUrl = `https://api.telegram.org/bot${botToken}/deleteWebhook`
        const telegramResponse = await fetch(telegramApiUrl, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
        })

        const responseBody = await telegramResponse.json()
        if (!telegramResponse.ok || !responseBody.ok) {
          const errorMessage =
            responseBody.description ||
            `Failed to delete Telegram webhook. Status: ${telegramResponse.status}`
          logger.error(`[${requestId}] ${errorMessage}`, { response: responseBody })
          return NextResponse.json(
            { error: 'Failed to delete webhook from Telegram', details: errorMessage },
            { status: 500 }
          )
        }

        logger.info(`[${requestId}] Successfully deleted Telegram webhook for webhook ${id}`)
      } catch (error: any) {
        logger.error(`[${requestId}] Error deleting Telegram webhook`, {
          webhookId: id,
          error: error.message,
          stack: error.stack,
        })
        return NextResponse.json(
          { error: 'Failed to delete webhook from Telegram', details: error.message },
          { status: 500 }
        )
      }
    }

    // Delete the webhook from the database
    await db.delete(webhook).where(eq(webhook.id, id))

    logger.info(`[${requestId}] Successfully deleted webhook: ${id}`)
    return NextResponse.json({ success: true }, { status: 200 })
  } catch (error: any) {
    logger.error(`[${requestId}] Error deleting webhook`, {
      error: error.message,
      stack: error.stack,
    })
    return NextResponse.json({ error: 'Internal server error' }, { status: 500 })
  }
}
```

*   **`export async function DELETE(...)`**: Defines the asynchronous function for `DELETE` requests. The signature is identical to `GET` and `PATCH`.
*   **`const requestId = generateRequestId()`**: Generates a request ID.
*   **`try { ... } catch (error) { ... }`**: Standard error handling.
*   **`const { id } = await params`**: Extracts the webhook ID from the URL.
*   **`logger.debug(...)`**: Logs the start of the deletion operation.
*   **`const session = await getSession()` / `if (!session?.user?.id)`**: Performs the same authentication check.
*   **`const webhooks = await db ... .limit(1)`**: Performs the same database query to retrieve the webhook and its associated workflow for permission checks.
*   **`if (webhooks.length === 0)`**: Handles the "webhook not found" scenario.
*   **`const webhookData = webhooks[0]`**: Extracts the webhook data.
*   **`let canDelete = false`**: Initializes a flag for deletion permissions.
*   **`if (webhookData.workflow.userId === session.user.id)`**: **Authorization Check (Case 1: Workflow Owner)**. If the user owns the workflow, they can delete its webhooks.
*   **`if (!canDelete && webhookData.workflow.workspaceId)`**: **Authorization Check (Case 2: Workspace Permissions)**. This check is similar to `PATCH`: the user needs `'write'` or `'admin'` permission in the workspace to delete a webhook.
*   **`if (!canDelete)`**: If the user lacks deletion permissions.
*   **`logger.warn(...)`**: Logs an access denial warning.
*   **`return NextResponse.json({ error: 'Access denied' }, { status: 403 })`**: Sends an "Access denied" error with `403` status.
*   **`const foundWebhook = webhookData.webhook`**: Stores the webhook details for easy access in subsequent checks.

    ---
    ### **External Service Integration Logic (Airtable, Microsoft Teams, Telegram)**

    This is the most complex part of the `DELETE` function, ensuring that when a webhook is deleted from the application's database, any corresponding subscription or webhook in a third-party service is also removed. This prevents stale or defunct webhooks from lingering in external systems.

*   **`if (foundWebhook.provider === 'airtable') { ... }`**: This block executes if the webhook's `provider` is `'airtable'`.
    *   **`const { baseId, externalId } = (foundWebhook.providerConfig || {}) as { ... }`**: Extracts `baseId` (the Airtable base ID) and `externalId` (the ID of the webhook in Airtable) from the `providerConfig` JSON field.
    *   **`if (!baseId)`**: Checks if `baseId` is missing, which is critical for interacting with Airtable. Returns a `400` error if missing.
    *   **`const userIdForToken = webhookData.workflow.userId`**: Identifies the user who owns the workflow (and thus likely authorized the Airtable integration).
    *   **`const accessToken = await getOAuthToken(userIdForToken, 'airtable')`**: Retrieves the OAuth access token for the workflow owner to authenticate with the Airtable API. Returns `401` if no token is found.
    *   **`let resolvedExternalId: string | undefined = externalId`**: Initializes a variable to hold the Airtable webhook ID.
    *   **`if (!resolvedExternalId) { ... }`**: **Crucial robust logic:** If `externalId` is *not* found in the local database (e.g., it wasn't stored during creation or was lost), this block attempts to find it by listing all webhooks in Airtable and matching the `notificationUrl`.
        *   **`const expectedNotificationUrl = `${getBaseUrl()}/api/webhooks/trigger/${foundWebhook.path}``**: Constructs the URL that this application would have registered with Airtable for the webhook.
        *   **`const listUrl = ... ; const listResp = await fetch(listUrl, ...)`**: Makes an API call to Airtable to list all webhooks for the `baseId`.
        *   **`const match = listBody.webhooks.find(...)`**: Iterates through the listed Airtable webhooks to find one whose `notificationUrl` matches `expectedNotificationUrl`. It includes a fallback `endsWith` check for robustness.
        *   **`if (match?.id) { ... }`**: If a match is found, `resolvedExternalId` is updated.
            *   **`try { await db.update(webhook).set({ ... }).where(eq(webhook.id, id)) } catch { ... }`**: Attempts to persist the `resolvedExternalId` back to the database. This is wrapped in a `try-catch` and marked as "non-fatal" because failing to update the local DB should not prevent the deletion process if the ID was successfully resolved.
        *   **`if (!resolvedExternalId) { ... }`**: If `externalId` still cannot be resolved, logs a message and proceeds, but skips the remote deletion step.
    *   **`if (resolvedExternalId) { ... }`**: If an `externalId` (either from the database or resolved) is available, proceed with remote deletion.
        *   **`const airtableDeleteUrl = ... ; const airtableResponse = await fetch(airtableDeleteUrl, { method: 'DELETE', ... })`**: Makes an `HTTP DELETE` request to the Airtable API to remove the specific webhook.
        *   **`if (!airtableResponse.ok) { ... }`**: Handles errors from the Airtable API. Attempts to parse the response body for more detailed error messages. Returns a `500` error if Airtable deletion fails.
        *   **`logger.info(...)`**: Logs successful deletion from Airtable.
    *   **`catch (error: any)`**: Catches any errors during the Airtable-specific deletion process. Logs the error and returns a `500` response.

*   **`if (foundWebhook.provider === 'microsoftteams') { ... }`**: This block executes if the webhook is for `'microsoftteams'`.
    *   **`const { deleteTeamsSubscription } = await import('@/lib/webhooks/webhook-helpers')`**: Dynamically imports a helper function. Dynamic imports help keep the initial bundle size smaller by only loading code when it's actually needed.
    *   **`await deleteTeamsSubscription(foundWebhook, webhookData.workflow, requestId)`**: Calls the helper function to delete the Microsoft Teams subscription.
    *   **`// Don't fail webhook deletion if subscription cleanup fails`**: A comment indicating a design decision: if the Teams cleanup fails, the overall webhook deletion process in the database should still proceed. This implies that the Teams subscription might become an orphaned resource if this helper fails.

*   **`if (foundWebhook.provider === 'telegram') { ... }`**: This block executes if the webhook is for `'telegram'`.
    *   **`const { botToken } = (foundWebhook.providerConfig || {}) as { botToken?: string }`**: Extracts the `botToken` from `providerConfig`.
    *   **`if (!botToken)`**: Checks for a missing `botToken`. Returns a `400` error if missing.
    *   **`const telegramApiUrl = ... ; const telegramResponse = await fetch(telegramApiUrl, { method: 'POST', ... })`**: Makes an `HTTP POST` request to the Telegram Bot API's `deleteWebhook` endpoint.
    *   **`if (!telegramResponse.ok || !responseBody.ok) { ... }`**: Checks both the HTTP response status and Telegram's internal `ok` flag in the response body. Returns a `500` error if deletion fails.
    *   **`logger.info(...)`**: Logs successful deletion from Telegram.
    *   **`catch (error: any)`**: Catches any errors during the Telegram-specific deletion process. Logs the error and returns a `500` response.

    ---

*   **`await db.delete(webhook).where(eq(webhook.id, id))`**: After attempting (and ideally succeeding) to delete the webhook from any external services, this line performs the final deletion from the application's database using Drizzle ORM.
    *   **`.delete(webhook)`**: Specifies the `webhook` table for deletion.
    *   **`.where(eq(webhook.id, id))`**: Ensures only the specific webhook identified by `id` is deleted.
*   **`logger.info(...)`**: Logs a success message for the overall webhook deletion.
*   **`return NextResponse.json({ success: true }, { status: 200 })`**: Sends a success response with `200` status.
*   **`catch (error: any)`**: Catches any unexpected errors in the `DELETE` function.
*   **`logger.error(...)`**: Logs the error with detailed information (`error.message`, `error.stack`).
*   **`return NextResponse.json({ error: 'Internal server error' }, { status: 500 })`**: Sends a generic `500` error response.

---

### Summary

This file serves as a comprehensive and secure API for managing webhooks within the application. It demonstrates best practices for:
*   **Next.js API Route Handlers**: Using `GET`, `PATCH`, `DELETE` functions.
*   **Database Interaction**: Leveraging Drizzle ORM for querying and modifying data.
*   **Authentication & Authorization**: Securing endpoints based on user sessions and granular permissions.
*   **Logging & Error Handling**: Providing traceability and robust error responses.
*   **External Service Integration**: Crucially handling the lifecycle of webhooks in third-party systems, especially during deletion, to maintain data integrity and prevent issues. The `DELETE` endpoint's logic for Airtable is particularly robust in handling cases where external IDs might be missing.