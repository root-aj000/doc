This file is a **Next.js API route** (`/api/webhooks`) that provides an interface for managing webhooks within the application. It exposes two main HTTP methods:

1.  **`GET`**: To retrieve a list of webhooks for the authenticated user, potentially filtered by a specific workflow and block.
2.  **`POST`**: To create a new webhook or update an existing one. This function also handles the interaction with various external third-party services (like Airtable, Microsoft Teams, Telegram, Gmail, Outlook) to set up or configure the actual webhook subscriptions with those services.

In essence, this file acts as a central hub for users to define and manage their integrations, allowing the application to receive real-time data or events from external platforms.

---

### Key Technologies Used:

*   **Next.js API Routes**: For building server-side API endpoints in a React application.
*   **Drizzle ORM**: An object-relational mapper for interacting with the database in a type-safe manner.
*   **PostgreSQL (via `@sim/db`)**: The underlying database.
*   **`nanoid`**: For generating short, unique, URL-friendly IDs.
*   **NextAuth.js (via `getSession`)**: For user authentication and session management.
*   **Custom Logging (`createLogger`)**: For structured logging throughout the API.
*   **Permissions System (`getUserEntityPermissions`)**: For granular access control, especially in collaborative scenarios.
*   **OAuth Integration (`getOAuthToken`)**: For obtaining access tokens to interact with third-party services on behalf of the user.

---

### Detailed Code Explanation

Let's break down the code section by section.

#### Imports

```typescript
import { db } from '@sim/db'
import { webhook, workflow } from '@sim/db/schema'
import { and, desc, eq } from 'drizzle-orm'
import { nanoid } from 'nanoid'
import { type NextRequest, NextResponse } from 'next/server'
import { getSession } from '@/lib/auth'
import { createLogger } from '@/lib/logs/console/logger'
import { getUserEntityPermissions } from '@/lib/permissions/utils'
import { getBaseUrl } from '@/lib/urls/utils'
import { generateRequestId } from '@/lib/utils'
import { getOAuthToken } from '@/app/api/auth/oauth/utils'
```

These lines import necessary modules and utilities:

*   **`db`**: The Drizzle ORM database client instance, configured for PostgreSQL.
*   **`webhook`, `workflow`**: Drizzle schema definitions for the `webhook` and `workflow` tables in the database. These define the structure of the data.
*   **`and`, `desc`, `eq`**: Drizzle ORM helper functions for building database queries:
    *   `and`: Combines multiple conditions with a logical AND.
    *   `desc`: Specifies descending order for a column.
    *   `eq`: Checks for equality (`=`).
*   **`nanoid`**: A library for generating small, secure, URL-friendly unique IDs. Used for new webhook IDs.
*   **`NextRequest`, `NextResponse`**: Types and classes from Next.js for handling incoming HTTP requests and crafting responses in API routes.
*   **`getSession`**: A utility to retrieve the authenticated user's session from NextAuth.js. Essential for authorization.
*   **`createLogger`**: A custom logging utility to log events and errors.
*   **`getUserEntityPermissions`**: A utility to check a user's permissions on specific entities (like workspaces). Critical for collaborative features.
*   **`getBaseUrl`**: A utility to get the base URL of the application. Used for constructing webhook notification URLs.
*   **`generateRequestId`**: A utility to generate a unique request ID for logging, aiding in tracing requests through the system.
*   **`getOAuthToken`**: A utility to retrieve OAuth access tokens for third-party services, used when interacting with those services.

#### Logger and Dynamic Export

```typescript
const logger = createLogger('WebhooksAPI')

export const dynamic = 'force-dynamic'
```

*   **`const logger = createLogger('WebhooksAPI')`**: Initializes a logger instance specifically for this API route, making it easier to filter logs by origin.
*   **`export const dynamic = 'force-dynamic'`**: This is a Next.js App Router specific export. Setting it to `'force-dynamic'` ensures that this API route is always executed dynamically on the server at request time, rather than being statically optimized or cached. This is crucial for routes that handle user authentication and dynamic data, as caching could lead to stale data or security issues.

---

### `GET` Function: Get all webhooks for the current user

```typescript
// Get all webhooks for the current user
export async function GET(request: NextRequest) {
  const requestId = generateRequestId()

  try {
    const session = await getSession()
    if (!session?.user?.id) {
      logger.warn(`[${requestId}] Unauthorized webhooks access attempt`)
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
    }

    // ... (rest of the GET logic)
  } catch (error) {
    logger.error(`[${requestId}] Error fetching webhooks`, error)
    return NextResponse.json({ error: 'Internal server error' }, { status: 500 })
  }
}
```

This asynchronous function handles `GET` requests to the `/api/webhooks` endpoint. Its primary role is to fetch webhooks associated with the authenticated user, potentially filtering them based on query parameters.

#### Simplified Logic:

1.  **Authentication**: Checks if the user is logged in. If not, returns 401 Unauthorized.
2.  **Query Parameters**: Extracts `workflowId` and `blockId` from the URL.
3.  **Conditional Logic**:
    *   **If `workflowId` and `blockId` are provided**: This indicates a request for webhooks within a specific workflow block. It performs a rigorous permission check (owner or workspace collaborator with read/write/admin access) before fetching the webhooks.
    *   **If `workflowId` is provided but `blockId` is not**: Currently returns an empty array, suggesting this path is not fully supported or is a placeholder.
    *   **Default (no `workflowId` or `blockId`)**: Fetches all webhooks owned directly by the current user.
4.  **Database Query**: Uses Drizzle ORM to query the `webhook` and `workflow` tables.
5.  **Response**: Returns the found webhooks (or an empty array) with a 200 status, or an appropriate error status.

#### Line-by-Line Explanation of `GET`:

```typescript
export async function GET(request: NextRequest) {
  const requestId = generateRequestId() // Generates a unique ID for this request for logging purposes.

  try {
    const session = await getSession() // Retrieves the user's session using NextAuth.js.
    if (!session?.user?.id) { // Checks if a session exists and if the user ID is present.
      logger.warn(`[${requestId}] Unauthorized webhooks access attempt`) // Logs a warning if no user ID is found.
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 }) // Returns a 401 Unauthorized response.
    }

    // Get query parameters
    const { searchParams } = new URL(request.url) // Creates a URL object from the request URL to easily access query parameters.
    const workflowId = searchParams.get('workflowId') // Extracts the 'workflowId' query parameter.
    const blockId = searchParams.get('blockId') // Extracts the 'blockId' query parameter.

    if (workflowId && blockId) {
      // Collaborative-aware path: allow collaborators with read access to view webhooks
      // Fetch workflow to verify access
      const wf = await db
        .select({ id: workflow.id, userId: workflow.userId, workspaceId: workflow.workspaceId }) // Selects specific fields from the workflow table.
        .from(workflow) // Specifies the workflow table.
        .where(eq(workflow.id, workflowId)) // Filters workflows by the provided workflowId.
        .limit(1) // Limits the result to 1 workflow.

      if (!wf.length) { // Checks if a workflow with the given ID was found.
        logger.warn(`[${requestId}] Workflow not found: ${workflowId}`) // Logs a warning if not found.
        return NextResponse.json({ error: 'Workflow not found' }, { status: 404 }) // Returns a 404 Not Found response.
      }

      const wfRecord = wf[0] // Gets the first (and only) workflow record.
      let canRead = wfRecord.userId === session.user.id // Initially checks if the current user is the owner of the workflow.
      if (!canRead && wfRecord.workspaceId) { // If not the owner and the workflow belongs to a workspace...
        const permission = await getUserEntityPermissions( // ...check the user's permissions on that workspace.
          session.user.id,
          'workspace',
          wfRecord.workspaceId
        )
        // User can read if they have 'read', 'write', or 'admin' permission on the workspace.
        canRead = permission === 'read' || permission === 'write' || permission === 'admin'
      }

      if (!canRead) { // If the user doesn't have read access to the workflow...
        logger.warn(
          `[${requestId}] User ${session.user.id} denied permission to read webhooks for workflow ${workflowId}`
        ) // Logs a warning.
        return NextResponse.json({ webhooks: [] }, { status: 200 }) // Returns an empty array of webhooks with a 200 status, indicating no readable webhooks.
      }

      const webhooks = await db
        .select({ // Selects the entire webhook record and specific fields from the joined workflow.
          webhook: webhook,
          workflow: {
            id: workflow.id,
            name: workflow.name,
          },
        })
        .from(webhook) // Starts the query from the webhook table.
        .innerJoin(workflow, eq(webhook.workflowId, workflow.id)) // Joins with the workflow table where workflowId matches.
        .where(and(eq(webhook.workflowId, workflowId), eq(webhook.blockId, blockId))) // Filters by both workflowId and blockId.
        .orderBy(desc(webhook.updatedAt)) // Orders results by updatedAt in descending order.

      logger.info(
        `[${requestId}] Retrieved ${webhooks.length} webhooks for workflow ${workflowId} block ${blockId}`
      ) // Logs the number of webhooks retrieved.
      return NextResponse.json({ webhooks }, { status: 200 }) // Returns the webhooks.
    }

    if (workflowId && !blockId) {
      // For now, allow the call but return empty results to avoid breaking the UI
      logger.debug(`[${requestId}] WorkflowId provided without blockId. Returning empty array as a placeholder.`)
      return NextResponse.json({ webhooks: [] }, { status: 200 }) // If workflowId is provided but not blockId, returns an empty array. This might be a placeholder for future functionality.
    }

    // Default: list webhooks owned by the session user
    logger.debug(`[${requestId}] Fetching user-owned webhooks for ${session.user.id}`) // Logs that user-owned webhooks are being fetched.
    const webhooks = await db
      .select({ // Selects the entire webhook record and specific fields from the joined workflow.
        webhook: webhook,
        workflow: {
          id: workflow.id,
          name: workflow.name,
        },
      })
      .from(webhook) // Starts the query from the webhook table.
      .innerJoin(workflow, eq(webhook.workflowId, workflow.id)) // Joins with the workflow table.
      .where(eq(workflow.userId, session.user.id)) // Filters webhooks where the associated workflow is owned by the current user.

    logger.info(`[${requestId}] Retrieved ${webhooks.length} user-owned webhooks`) // Logs the number of user-owned webhooks retrieved.
    return NextResponse.json({ webhooks }, { status: 200 }) // Returns the user-owned webhooks.
  } catch (error) {
    logger.error(`[${requestId}] Error fetching webhooks`, error) // Logs any unexpected errors.
    return NextResponse.json({ error: 'Internal server error' }, { status: 500 }) // Returns a 500 Internal Server Error response.
  }
}
```

---

### `POST` Function: Create or Update a webhook

```typescript
// Create or Update a webhook
export async function POST(request: NextRequest) {
  const requestId = generateRequestId()
  const userId = (await getSession())?.user?.id

  if (!userId) {
    logger.warn(`[${requestId}] Unauthorized webhook creation attempt`)
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
  }

  try {
    // ... (rest of the POST logic)
  } catch (error: any) {
    logger.error(`[${requestId}] Error creating/updating webhook`, {
      message: error.message,
      stack: error.stack,
    })
    return NextResponse.json({ error: 'Internal server error' }, { status: 500 })
  }
}
```

This asynchronous function handles `POST` requests. It's responsible for creating a new webhook entry in the database or updating an existing one. Crucially, it also handles the setup of the webhook with external third-party services based on the `provider`.

#### Simplified Logic:

1.  **Authentication**: Checks if the user is logged in. If not, returns 401 Unauthorized.
2.  **Input Validation**: Parses the request body and validates essential fields like `workflowId`.
3.  **Path Generation/Reuse**:
    *   For "credential-based" providers (like Gmail, Outlook) and Microsoft Teams chat subscriptions, it attempts to reuse an existing webhook path for a given workflow and block. This prevents generating new external webhook URLs on every save, which is common for services that require a stable endpoint tied to a user's credential.
    *   If no path is provided and it's not a credential-based provider, it returns a 400 error.
    *   If no path exists and it's a credential-based provider, it generates a new unique path.
4.  **Workflow Permissions**: Verifies that the authenticated user has permission to modify the associated workflow (either by owning it or having 'write'/'admin' access in its workspace).
5.  **Identify Target Webhook**: Determines if the request is for an update or a new creation by checking for an existing webhook by `workflowId` + `blockId` (for credential-based providers) or by the generated/provided `path`. It also checks for path conflicts (same path used for a different workflow).
6.  **Database Operation**: Performs an `UPDATE` on an existing webhook or an `INSERT` for a new one in the `webhook` table.
7.  **External Service Integration**: If the webhook is successfully saved, it then triggers provider-specific logic to set up the webhook on the external service (e.g., Airtable, Microsoft Teams, Telegram, Gmail, Outlook). Each of these integrations involves specific API calls to the respective third-party platform.
8.  **Response**: Returns the saved webhook data with a 200 (updated) or 201 (created) status, or an appropriate error status.

#### Line-by-Line Explanation of `POST`:

```typescript
export async function POST(request: NextRequest) {
  const requestId = generateRequestId() // Generates a unique request ID for logging.
  const userId = (await getSession())?.user?.id // Retrieves the user ID from the session.

  if (!userId) { // Checks if the user is authenticated.
    logger.warn(`[${requestId}] Unauthorized webhook creation attempt`) // Logs a warning.
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 }) // Returns 401 Unauthorized.
  }

  try {
    const body = await request.json() // Parses the JSON body of the request.
    const { workflowId, path, provider, providerConfig, blockId } = body // Destructures relevant fields from the request body.

    // Validate input
    if (!workflowId) { // Ensures workflowId is provided.
      logger.warn(`[${requestId}] Missing required fields for webhook creation`, {
        hasWorkflowId: !!workflowId,
        hasPath: !!path,
      }) // Logs a warning with details on missing fields.
      return NextResponse.json({ error: 'Missing required fields' }, { status: 400 }) // Returns 400 Bad Request.
    }

    // Determine final path with special handling for credential-based providers
    let finalPath = path // Initialize finalPath with the provided path.
    const credentialBasedProviders = ['gmail', 'outlook'] // Defines providers that typically use a stable path tied to credentials.
    const isCredentialBased = credentialBasedProviders.includes(provider) // Checks if the current provider is credential-based.
    // Special case for Microsoft Teams chat subscription, treated similarly for path generation.
    const isMicrosoftTeamsChatSubscription =
      provider === 'microsoftteams' &&
      typeof providerConfig === 'object' &&
      providerConfig?.triggerId === 'microsoftteams_chat_subscription'

    // If path is missing or empty
    if (!finalPath || finalPath.trim() === '') {
      if (isCredentialBased || isMicrosoftTeamsChatSubscription) {
        // For credential-based/Teams chat: try to reuse existing path for this workflow+block
        if (blockId) { // If a blockId is provided...
          const existingForBlock = await db
            .select({ id: webhook.id, path: webhook.path }) // Selects ID and path of existing webhooks.
            .from(webhook)
            .where(and(eq(webhook.workflowId, workflowId), eq(webhook.blockId, blockId))) // Filters by workflowId and blockId.
            .limit(1)

          if (existingForBlock.length > 0) { // If an existing webhook for this block is found...
            finalPath = existingForBlock[0].path // ...reuse its path.
            logger.info(
              `[${requestId}] Reusing existing generated path for ${provider} trigger: ${finalPath}`
            ) // Logs that a path was reused.
          }
        }

        // If still no path after attempting reuse, generate a new dummy path (first-time save)
        if (!finalPath || finalPath.trim() === '') { // If finalPath is still empty...
          finalPath = `${provider}-${crypto.randomUUID()}` // ...generate a new unique path.
          logger.info(`[${requestId}] Generated webhook path for ${provider} trigger: ${finalPath}`) // Logs the generated path.
        }
      } else {
        // If not credential-based and path is missing, it's an error.
        logger.warn(`[${requestId}] Missing path for webhook creation`, {
          hasWorkflowId: !!workflowId,
          hasPath: !!path,
        })
        return NextResponse.json({ error: 'Missing required path' }, { status: 400 }) // Returns 400 Bad Request.
      }
    }

    // Check if the workflow exists and user has permission to modify it
    const workflowData = await db
      .select({
        id: workflow.id,
        userId: workflow.userId,
        workspaceId: workflow.workspaceId,
      })
      .from(workflow)
      .where(eq(workflow.id, workflowId))
      .limit(1)

    if (workflowData.length === 0) { // Checks if the workflow exists.
      logger.warn(`[${requestId}] Workflow not found: ${workflowId}`) // Logs a warning.
      return NextResponse.json({ error: 'Workflow not found' }, { status: 404 }) // Returns 404 Not Found.
    }

    const workflowRecord = workflowData[0] // Gets the workflow record.

    // Check if user has permission to modify this workflow
    let canModify = false // Initialize permission flag.

    // Case 1: User owns the workflow
    if (workflowRecord.userId === userId) { // If the current user is the workflow owner...
      canModify = true // ...they have modify permission.
    }

    // Case 2: Workflow belongs to a workspace and user has write or admin permission
    if (!canModify && workflowRecord.workspaceId) { // If not owner and in a workspace...
      const userPermission = await getUserEntityPermissions( // ...check workspace permissions.
        userId,
        'workspace',
        workflowRecord.workspaceId
      )
      if (userPermission === 'write' || userPermission === 'admin') { // If user has 'write' or 'admin' access...
        canModify = true // ...they have modify permission.
      }
    }

    if (!canModify) { // If the user doesn't have modify permission...
      logger.warn(
        `[${requestId}] User ${userId} denied permission to modify webhook for workflow ${workflowId}`
      ) // Logs a warning.
      return NextResponse.json({ error: 'Access denied' }, { status: 403 }) // Returns 403 Forbidden.
    }

    // Determine existing webhook to update (prefer by workflow+block for credential-based providers)
    let targetWebhookId: string | null = null // Variable to store the ID of an existing webhook to update.
    if (isCredentialBased && blockId) { // For credential-based providers, prioritize finding by blockId.
      const existingForBlock = await db
        .select({ id: webhook.id })
        .from(webhook)
        .where(and(eq(webhook.workflowId, workflowId), eq(webhook.blockId, blockId)))
        .limit(1)
      if (existingForBlock.length > 0) {
        targetWebhookId = existingForBlock[0].id // Found an existing webhook by blockId.
      }
    }
    if (!targetWebhookId) { // If no webhook found by blockId or not credential-based...
      const existingByPath = await db // ...try finding by the finalPath.
        .select({ id: webhook.id, workflowId: webhook.workflowId })
        .from(webhook)
        .where(eq(webhook.path, finalPath))
        .limit(1)
      if (existingByPath.length > 0) {
        // If a webhook with the same path exists but belongs to a different workflow, return an error
        if (existingByPath[0].workflowId !== workflowId) { // Path conflict: path exists for a different workflow.
          logger.warn(`[${requestId}] Webhook path conflict: ${finalPath}`) // Logs a warning.
          return NextResponse.json(
            { error: 'Webhook path already exists.', code: 'PATH_EXISTS' },
            { status: 409 } // Returns 409 Conflict.
          )
        }
        targetWebhookId = existingByPath[0].id // Found an existing webhook by path for the same workflow.
      }
    }

    let savedWebhook: any = null // Variable to hold the result of the database operation.

    // Use the original provider config - Gmail/Outlook configuration functions will inject userId automatically
    const finalProviderConfig = providerConfig // The provider configuration from the request body.

    if (targetWebhookId) {
      logger.info(`[${requestId}] Updating existing webhook for path: ${finalPath}`, {
        webhookId: targetWebhookId,
        provider,
        hasCredentialId: !!(finalProviderConfig as any)?.credentialId,
        credentialId: (finalProviderConfig as any)?.credentialId,
      }) // Logs the update operation.
      const updatedResult = await db // Performs a Drizzle update operation.
        .update(webhook)
        .set({ // Sets the new values for the webhook.
          blockId,
          provider,
          providerConfig: finalProviderConfig,
          isActive: true,
          updatedAt: new Date(),
        })
        .where(eq(webhook.id, targetWebhookId)) // Filters by the targetWebhookId.
        .returning() // Returns the updated record.
      savedWebhook = updatedResult[0] // Stores the updated webhook.
      logger.info(`[${requestId}] Webhook updated successfully`, {
        webhookId: savedWebhook.id,
        savedProviderConfig: savedWebhook.providerConfig,
      }) // Logs successful update.
    } else {
      // Create a new webhook
      const webhookId = nanoid() // Generates a new unique ID for the webhook.
      logger.info(`[${requestId}] Creating new webhook with ID: ${webhookId}`) // Logs the creation.
      const newResult = await db // Performs a Drizzle insert operation.
        .insert(webhook)
        .values({ // Sets the values for the new webhook.
          id: webhookId,
          workflowId,
          blockId,
          path: finalPath,
          provider,
          providerConfig: finalProviderConfig,
          isActive: true,
          createdAt: new Date(),
          updatedAt: new Date(),
        })
        .returning() // Returns the newly created record.
      savedWebhook = newResult[0] // Stores the newly created webhook.
    }

    // --- Attempt to create webhook in Airtable if provider is 'airtable' ---
    if (savedWebhook && provider === 'airtable') { // If webhook saved and provider is Airtable...
      logger.info(
        `[${requestId}] Airtable provider detected. Attempting to create webhook in Airtable.`
      ) // Logs the intention.
      try {
        await createAirtableWebhookSubscription(request, userId, savedWebhook, requestId) // Calls the helper function for Airtable.
      } catch (err) {
        logger.error(`[${requestId}] Error creating Airtable webhook`, err) // Logs any errors during Airtable setup.
        return NextResponse.json(
          {
            error: 'Failed to create webhook in Airtable',
            details: err instanceof Error ? err.message : 'Unknown error',
          },
          { status: 500 } // Returns 500 if Airtable setup fails.
        )
      }
    }
    // --- End Airtable specific logic ---

    // --- Microsoft Teams subscription setup ---
    if (savedWebhook && provider === 'microsoftteams') { // If webhook saved and provider is Microsoft Teams...
      const { createTeamsSubscription } = await import('@/lib/webhooks/webhook-helpers') // Dynamically imports the Teams helper.
      logger.info(`[${requestId}] Creating Teams subscription for webhook ${savedWebhook.id}`) // Logs the intention.

      const success = await createTeamsSubscription( // Calls the helper function for Teams.
        request,
        savedWebhook,
        workflowRecord,
        requestId
      )

      if (!success) { // If Teams setup fails.
        return NextResponse.json(
          {
            error: 'Failed to create Teams subscription',
            details: 'Could not create subscription with Microsoft Graph API',
          },
          { status: 500 } // Returns 500.
        )
      }
    }
    // --- End Teams subscription setup ---

    // --- Telegram webhook setup ---
    if (savedWebhook && provider === 'telegram') { // If webhook saved and provider is Telegram...
      const { createTelegramWebhook } = await import('@/lib/webhooks/webhook-helpers') // Dynamically imports the Telegram helper.
      logger.info(`[${requestId}] Creating Telegram webhook for webhook ${savedWebhook.id}`) // Logs the intention.

      const success = await createTelegramWebhook(request, savedWebhook, requestId) // Calls the helper function for Telegram.

      if (!success) { // If Telegram setup fails.
        return NextResponse.json(
          {
            error: 'Failed to create Telegram webhook',
          },
          { status: 500 } // Returns 500.
        )
      }
    }
    // --- End Telegram webhook setup ---

    // --- Gmail webhook setup ---
    if (savedWebhook && provider === 'gmail') { // If webhook saved and provider is Gmail...
      logger.info(`[${requestId}] Gmail provider detected. Setting up Gmail webhook configuration.`) // Logs the intention.
      try {
        const { configureGmailPolling } = await import('@/lib/webhooks/utils') // Dynamically imports the Gmail helper.
        const success = await configureGmailPolling(savedWebhook, requestId) // Calls the helper function for Gmail.

        if (!success) { // If Gmail setup fails.
          logger.error(`[${requestId}] Failed to configure Gmail polling`)
          return NextResponse.json(
            {
              error: 'Failed to configure Gmail polling',
              details: 'Please check your Gmail account permissions and try again',
            },
            { status: 500 } // Returns 500.
          )
        }

        logger.info(`[${requestId}] Successfully configured Gmail polling`) // Logs success.
      } catch (err) {
        logger.error(`[${requestId}] Error setting up Gmail webhook configuration`, err) // Logs any errors during Gmail setup.
        return NextResponse.json(
          {
            error: 'Failed to configure Gmail webhook',
            details: err instanceof Error ? err.message : 'Unknown error',
          },
          { status: 500 } // Returns 500.
        )
      }
    }
    // --- End Gmail specific logic ---

    // --- Outlook webhook setup ---
    if (savedWebhook && provider === 'outlook') { // If webhook saved and provider is Outlook...
      logger.info(
        `[${requestId}] Outlook provider detected. Setting up Outlook webhook configuration.`
      ) // Logs the intention.
      try {
        const { configureOutlookPolling } = await import('@/lib/webhooks/utils') // Dynamically imports the Outlook helper.
        const success = await configureOutlookPolling(savedWebhook, requestId) // Calls the helper function for Outlook.

        if (!success) { // If Outlook setup fails.
          logger.error(`[${requestId}] Failed to configure Outlook polling`)
          return NextResponse.json(
            {
              error: 'Failed to configure Outlook polling',
              details: 'Please check your Outlook account permissions and try again',
            },
            { status: 500 } // Returns 500.
          )
        }

        logger.info(`[${requestId}] Successfully configured Outlook polling`) // Logs success.
      } catch (err) {
        logger.error(`[${requestId}] Error setting up Outlook webhook configuration`, err) // Logs any errors during Outlook setup.
        return NextResponse.json(
          {
            error: 'Failed to configure Outlook webhook',
            details: err instanceof Error ? err.message : 'Unknown error',
          },
          { status: 500 } // Returns 500.
        )
      }
    }
    // --- End Outlook specific logic ---

    const status = targetWebhookId ? 200 : 201 // Determines response status: 200 for update, 201 for creation.
    return NextResponse.json({ webhook: savedWebhook }, { status }) // Returns the saved webhook with the appropriate status.
  } catch (error: any) {
    logger.error(`[${requestId}] Error creating/updating webhook`, {
      message: error.message,
      stack: error.stack,
    }) // Logs any unexpected errors.
    return NextResponse.json({ error: 'Internal server error' }, { status: 500 }) // Returns 500 Internal Server Error.
  }
}
```

---

### `createAirtableWebhookSubscription` Helper Function

```typescript
async function createAirtableWebhookSubscription(
  request: NextRequest,
  userId: string,
  webhookData: any,
  requestId: string
) {
  try {
    // ... (logic)
  } catch (error: any) {
    logger.error(
      `[${requestId}] Exception during Airtable webhook creation for webhook ${webhookData.id}.`,
      {
        message: error.message,
        stack: error.stack,
      }
    )
  }
}
```

This is a private helper function (not exported) specifically designed to interact with the Airtable API to create a new webhook subscription. It's called by the `POST` function when the `provider` is `'airtable'`.

#### Simplified Logic:

1.  **Retrieve Configuration**: Extracts necessary Airtable-specific configuration (`baseId`, `tableId`, `includeCellValuesInFieldIds`) from the `webhookData`.
2.  **OAuth Token**: Fetches the Airtable OAuth access token for the `userId`. Without it, it cannot authenticate with Airtable.
3.  **Construct URL**: Generates the notification URL that Airtable should send events to (pointing back to this application's webhook trigger endpoint).
4.  **Airtable API Call**: Makes a `POST` request to the Airtable API to create a new webhook, specifying what data types to watch and for which table.
5.  **Process Response**: Checks the Airtable API's response for success or errors.
6.  **Store External ID**: If successful, it updates the `providerConfig` of the internal `webhook` record with the `id` provided by Airtable (referred to as `externalId`). This links the internal webhook to its external counterpart.
7.  **Error Handling**: Logs and handles various errors, including missing configuration, failed token retrieval, and Airtable API errors.

#### Line-by-Line Explanation of `createAirtableWebhookSubscription`:

```typescript
async function createAirtableWebhookSubscription(
  request: NextRequest, // The original NextRequest object (though not directly used inside, good for context).
  userId: string, // The ID of the authenticated user.
  webhookData: any, // The newly created/updated webhook record from the database.
  requestId: string // The unique request ID for logging.
) {
  try {
    const { path, providerConfig } = webhookData // Destructures path and providerConfig from the webhook data.
    const { baseId, tableId, includeCellValuesInFieldIds } = providerConfig || {} // Extracts Airtable-specific config.

    if (!baseId || !tableId) { // Validates if required Airtable configuration is present.
      logger.warn(`[${requestId}] Missing baseId or tableId for Airtable webhook creation.`, {
        webhookId: webhookData.id,
      }) // Logs a warning.
      return // Exits the function if critical config is missing.
    }

    const accessToken = await getOAuthToken(userId, 'airtable') // Retrieves the Airtable OAuth access token for the user.
    if (!accessToken) { // Checks if a token was retrieved.
      logger.warn(
        `[${requestId}] Could not retrieve Airtable access token for user ${userId}. Cannot create webhook in Airtable.`
      ) // Logs a warning.
      // Throws an error that will be caught by the calling POST function's try/catch.
      throw new Error(
        'Airtable account connection required. Please connect your Airtable account in the trigger configuration and try again.'
      )
    }

    const notificationUrl = `${getBaseUrl()}/api/webhooks/trigger/${path}` // Constructs the full URL where Airtable should send notifications.

    const airtableApiUrl = `https://api.airtable.com/v0/bases/${baseId}/webhooks` // The Airtable API endpoint for creating webhooks.

    const specification: any = { // Defines the webhook's specification for Airtable.
      options: {
        filters: {
          dataTypes: ['tableData'], // Specifies to watch for changes in table data.
          recordChangeScope: tableId, // Specifies to watch changes only within the given table ID.
        },
      },
    }

    // Conditionally add the 'includes' field based on the config
    if (includeCellValuesInFieldIds === 'all') { // If configured to include all cell values...
      specification.options.includes = {
        includeCellValuesInFieldIds: 'all', // ...add this to the specification.
      }
    }

    const requestBody: any = { // Constructs the request body for the Airtable API call.
      notificationUrl: notificationUrl,
      specification: specification,
    }

    const airtableResponse = await fetch(airtableApiUrl, { // Makes the HTTP POST request to Airtable.
      method: 'POST',
      headers: {
        Authorization: `Bearer ${accessToken}`, // Uses the OAuth access token for authorization.
        'Content-Type': 'application/json',
      },
      body: JSON.stringify(requestBody), // Sends the request body as JSON.
    })

    // Airtable often returns 200 OK even for errors in the body, check payload
    const responseBody = await airtableResponse.json() // Parses the response body from Airtable.

    if (!airtableResponse.ok || responseBody.error) { // Checks if the HTTP response was not OK or if the response body contains an error.
      const errorMessage =
        responseBody.error?.message || responseBody.error || 'Unknown Airtable API error' // Extracts the error message.
      const errorType = responseBody.error?.type
      logger.error(
        `[${requestId}] Failed to create webhook in Airtable for webhook ${webhookData.id}. Status: ${airtableResponse.status}`,
        { type: errorType, message: errorMessage, response: responseBody }
      ) // Logs the Airtable API error.
      // If there's an error from Airtable, throw an error to propagate it up.
      throw new Error(`Airtable API error: ${errorMessage}`)
    } else {
      logger.info(
        `[${requestId}] Successfully created webhook in Airtable for webhook ${webhookData.id}.`,
        {
          airtableWebhookId: responseBody.id, // Logs the Airtable-provided webhook ID.
        }
      )
      // Store the airtableWebhookId (responseBody.id) within the providerConfig
      try {
        const currentConfig = (webhookData.providerConfig as Record<string, any>) || {} // Gets the current provider config.
        const updatedConfig = {
          ...currentConfig,
          externalId: responseBody.id, // Adds/updates the 'externalId' with Airtable's webhook ID.
        }
        await db // Updates the internal webhook record in the database.
          .update(webhook)
          .set({ providerConfig: updatedConfig, updatedAt: new Date() }) // Sets the updated providerConfig and updatedAt timestamp.
          .where(eq(webhook.id, webhookData.id)) // Filters by the internal webhook ID.
      } catch (dbError: any) {
        logger.error(
          `[${requestId}] Failed to store externalId in providerConfig for webhook ${webhookData.id}.`,
          dbError
        )
        // Even if saving fails, the webhook exists in Airtable. Log and continue.
      }
    }
  } catch (error: any) {
    logger.error(
      `[${requestId}] Exception during Airtable webhook creation for webhook ${webhookData.id}.`,
      {
        message: error.message,
        stack: error.stack,
      }
    ) // Catches and logs any exceptions that occur during this process.
    throw error; // Re-throw the error so the POST function can handle it.
  }
}
```