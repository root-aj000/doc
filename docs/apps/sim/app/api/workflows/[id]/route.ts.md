This TypeScript file defines a Next.js API route responsible for handling operations related to a single workflow identified by its ID. It provides endpoints for `GET` (fetching), `DELETE` (removing), and `PUT` (updating metadata) a specific workflow.

---

### **Purpose of this file and what it does**

This file `api/workflows/[id]/route.ts` acts as a backend service for managing individual workflows in a system. It's a Next.js Route Handler, meaning functions exported from this file (like `GET`, `DELETE`, `PUT`) automatically become API endpoints accessible at `/api/workflows/[id]`, where `[id]` is a dynamic segment representing the workflow's unique identifier.

**In essence, this file: **
1.  **Retrieves a Workflow (`GET`):** Allows clients to fetch the complete data for a specific workflow, including its structure (blocks, edges, loops, parallels). It employs robust authentication (user session, API keys, or internal tokens) and authorization checks to ensure only authorized users or services can access the data. It also demonstrates a "hybrid" data loading approach, preferring a normalized database structure for workflow state over a single JSON blob.
2.  **Deletes a Workflow (`DELETE`):** Enables authorized users to remove a workflow from the system. It includes critical checks for ownership/permissions and handles associated "templates" (which might be derived from workflows) by either deleting them or unlinking them based on user-provided parameters. It also notifies a separate Socket.IO service about the deletion to manage real-time client connections.
3.  **Updates Workflow Metadata (`PUT`):** Permits authorized users to modify basic metadata of a workflow, such as its name, description, color, or the folder it belongs to. It uses `Zod` for strong input validation and enforces permissions before applying changes.

This file is crucial for enabling the core functionality of a workflow management application, ensuring data integrity, security, and a flexible data model.

---

### **Simplify complex logic**

The core logic revolves around three main pillars for each operation:

1.  **Authentication:** Who is making the request? Is it a logged-in user, a service using an API key, or an internal system call? The system first tries to identify the caller.
2.  **Authorization:** Once identified, does this specific caller have the necessary permissions to perform the requested action (read, delete, update) on *this specific workflow*? This involves checking if they own the workflow, or if they have appropriate permissions within a workspace that the workflow belongs to.
3.  **Data Handling:** After verifying permissions, the code interacts with the database (using Drizzle ORM) to either fetch, delete, or update the workflow data. The `GET` request has an interesting twist: it tries to load the workflow's complex *state* (like its visual layout) from multiple, structured database tables rather than a single JSON field, creating a more robust and scalable data model.

Throughout, logging is used to track requests, and comprehensive error handling ensures the API responds gracefully to issues like unauthorized access, not-found resources, or invalid input.

---

### **Explain each line of code deep**

Let's break down the file section by section.

#### **Imports and Setup**

```typescript
import { db } from '@sim/db'
import { templates, workflow } from '@sim/db/schema'
import { eq } from 'drizzle-orm'
import { type NextRequest, NextResponse } from 'next/server'
import { z } from 'zod'
import { authenticateApiKeyFromHeader, updateApiKeyLastUsed } from '@/lib/api-key/service'
import { getSession } from '@/lib/auth'
import { verifyInternalToken } from '@/lib/auth/internal'
import { env } from '@/lib/env'
import { createLogger } from '@/lib/logs/console/logger'
import { generateRequestId } from '@/lib/utils'
import { loadWorkflowFromNormalizedTables } from '@/lib/workflows/db-helpers'
import { getWorkflowAccessContext, getWorkflowById } from '@/lib/workflows/utils'

const logger = createLogger('WorkflowByIdAPI')
```

*   **`import { db } from '@sim/db'`**: Imports the Drizzle ORM database client instance, which is configured to connect to your database. This `db` object is used for all database interactions.
*   **`import { templates, workflow } from '@sim/db/schema'`**: Imports the Drizzle ORM schema definitions for the `templates` and `workflow` tables. These are objects that represent the tables in your database and are used to construct queries.
*   **`import { eq } from 'drizzle-orm'`**: Imports the `eq` (equals) function from Drizzle ORM. This is a common utility for building `WHERE` clauses in database queries (e.g., `WHERE id = 'someId'`).
*   **`import { type NextRequest, NextResponse } from 'next/server'`**: Imports types and classes from Next.js for handling server-side HTTP requests and responses.
    *   `NextRequest`: Represents an incoming HTTP request, providing access to headers, body, URL, etc.
    *   `NextResponse`: Used to create and send HTTP responses, allowing control over status codes, headers, and body content.
*   **`import { z } from 'zod'`**: Imports the `z` object from the Zod library. Zod is a TypeScript-first schema declaration and validation library, used here to validate incoming request bodies.
*   **`import { authenticateApiKeyFromHeader, updateApiKeyLastUsed } from '@/lib/api-key/service'`**: Imports functions related to API key management:
    *   `authenticateApiKeyFromHeader`: Validates an API key found in request headers and returns the associated user ID.
    *   `updateApiKeyLastUsed`: Updates the `lastUsed` timestamp for an API key in the database.
*   **`import { getSession } from '@/lib/auth'`**: Imports a function to retrieve the user's session information, typically from a cookie. This is used for authenticating logged-in users.
*   **`import { verifyInternalToken } from '@/lib/auth/internal'`**: Imports a function to verify an internal bearer token, used for authenticating requests originating from other internal services.
*   **`import { env } from '@/lib/env'`**: Imports environment variables, allowing the application to access configurations like API URLs.
*   **`import { createLogger } from '@/lib/logs/console/logger'`**: Imports a utility to create a logger instance, used for structured logging of events and errors within this API route.
*   **`import { generateRequestId } from '@/lib/utils'`**: Imports a utility function to generate a unique request ID for each incoming request, aiding in tracing and debugging.
*   **`import { loadWorkflowFromNormalizedTables } from '@/lib/workflows/db-helpers'`**: Imports a specific helper function to load complex workflow data (blocks, edges, etc.) from multiple normalized database tables. This is part of the "hybrid approach" mentioned earlier.
*   **`import { getWorkflowAccessContext, getWorkflowById } from '@/lib/workflows/utils'`**: Imports utility functions for workflow-specific operations:
    *   `getWorkflowAccessContext`: Determines a user's access level (owner, workspace permission) to a specific workflow.
    *   `getWorkflowById`: Fetches basic metadata of a workflow by its ID.
*   **`const logger = createLogger('WorkflowByIdAPI')`**: Initializes a logger instance with the context name 'WorkflowByIdAPI'. This helps identify logs originating from this specific file.

#### **Zod Schema for Workflow Updates**

```typescript
const UpdateWorkflowSchema = z.object({
  name: z.string().min(1, 'Name is required').optional(),
  description: z.string().optional(),
  color: z.string().optional(),
  folderId: z.string().nullable().optional(),
})
```

*   **`const UpdateWorkflowSchema = z.object(...)`**: Defines a Zod schema named `UpdateWorkflowSchema`. This schema specifies the expected structure and validation rules for the data sent in a `PUT` request to update a workflow.
*   **`name: z.string().min(1, 'Name is required').optional()`**: Defines a `name` property. It must be a string, have a minimum length of 1 character (and an error message if not), and is `optional()`, meaning it doesn't have to be present in the request body to be valid.
*   **`description: z.string().optional()`**: Defines an optional `description` property, which must be a string.
*   **`color: z.string().optional()`**: Defines an optional `color` property, which must be a string.
*   **`folderId: z.string().nullable().optional()`**: Defines an optional `folderId` property. It can be a string representing a folder ID or `null` (if the workflow is moved out of a folder), and it's also optional.

#### **`GET /api/workflows/[id]` - Fetch a single workflow**

```typescript
/**
 * GET /api/workflows/[id]
 * Fetch a single workflow by ID
 * Uses hybrid approach: try normalized tables first, fallback to JSON blob
 */
export async function GET(request: NextRequest, { params }: { params: Promise<{ id: string }> }) {
  const requestId = generateRequestId()
  const startTime = Date.now()
  const { id: workflowId } = await params
```

*   **`export async function GET(...)`**: Defines the GET handler for this API route. Next.js automatically routes GET requests to this function.
    *   `request: NextRequest`: The incoming request object.
    *   `{ params }: { params: Promise<{ id: string }> }`: Destructures `params` from the second argument. `params` will contain route parameters, in this case, `id` (e.g., if the URL is `/api/workflows/wf123`, `id` will be `wf123`). It's wrapped in `Promise` because Next.js route handlers can pass dynamic parameters as promises in certain scenarios.
*   **`const requestId = generateRequestId()`**: Generates a unique ID for the current request, used for consistent logging.
*   **`const startTime = Date.now()`**: Records the timestamp when the request started, used to calculate the total execution time for logging.
*   **`const { id: workflowId } = await params`**: Extracts the `id` property from the `params` object (waiting for the promise to resolve if necessary) and renames it to `workflowId` for clarity.

```typescript
  try {
    const authHeader = request.headers.get('authorization')
    let isInternalCall = false

    if (authHeader?.startsWith('Bearer ')) {
      const token = authHeader.split(' ')[1]
      isInternalCall = await verifyInternalToken(token)
    }

    let userId: string | null = null
```

*   **`try { ... } catch (error: any) { ... }`**: Encapsulates the main logic within a `try-catch` block to handle any synchronous or asynchronous errors that occur during request processing.
*   **`const authHeader = request.headers.get('authorization')`**: Retrieves the `Authorization` header from the incoming request.
*   **`let isInternalCall = false`**: Initializes a flag to track if the request is an internal system call.
*   **`if (authHeader?.startsWith('Bearer ')) { ... }`**: Checks if an `Authorization` header exists and starts with "Bearer ". This is the common format for JWT or API token authentication.
*   **`const token = authHeader.split(' ')[1]`**: Extracts the actual token string by splitting the header value (e.g., "Bearer YOUR_TOKEN" -> "YOUR_TOKEN").
*   **`isInternalCall = await verifyInternalToken(token)`**: Calls a helper function to verify if the extracted token is a valid internal system token. If valid, `isInternalCall` becomes `true`.
*   **`let userId: string | null = null`**: Initializes a variable to store the authenticated user's ID. It will be `null` if no user is authenticated or if it's an internal call.

```typescript
    if (isInternalCall) {
      logger.info(`[${requestId}] Internal API call for workflow ${workflowId}`)
    } else {
      const session = await getSession()
      let authenticatedUserId: string | null = session?.user?.id || null

      if (!authenticatedUserId) {
        const apiKeyHeader = request.headers.get('x-api-key')
        if (apiKeyHeader) {
          const authResult = await authenticateApiKeyFromHeader(apiKeyHeader)
          if (authResult.success && authResult.userId) {
            authenticatedUserId = authResult.userId
            if (authResult.keyId) {
              await updateApiKeyLastUsed(authResult.keyId).catch((error) => {
                logger.warn(`[${requestId}] Failed to update API key last used timestamp:`, {
                  keyId: authResult.keyId,
                  error,
                })
              })
            }
          }
        }
      }

      if (!authenticatedUserId) {
        logger.warn(`[${requestId}] Unauthorized access attempt for workflow ${workflowId}`)
        return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
      }

      userId = authenticatedUserId
    }
```

*   **`if (isInternalCall) { ... } else { ... }`**: Conditional block: if it's an internal call, no user authentication is needed. Otherwise, proceed with user/API key authentication.
    *   **`logger.info(...)`**: Logs that an internal API call is being made.
*   **`const session = await getSession()`**: Tries to retrieve the user's session.
*   **`let authenticatedUserId: string | null = session?.user?.id || null`**: Extracts the user ID from the session if it exists. If not, `authenticatedUserId` remains `null`.
*   **`if (!authenticatedUserId) { ... }`**: If a user ID couldn't be obtained from the session, it attempts to authenticate via an API key.
    *   **`const apiKeyHeader = request.headers.get('x-api-key')`**: Checks for an `x-api-key` header.
    *   **`if (apiKeyHeader) { ... }`**: If an API key is present:
        *   **`const authResult = await authenticateApiKeyFromHeader(apiKeyHeader)`**: Validates the API key.
        *   **`if (authResult.success && authResult.userId) { ... }`**: If API key authentication is successful and a `userId` is returned:
            *   **`authenticatedUserId = authResult.userId`**: Sets the `authenticatedUserId`.
            *   **`if (authResult.keyId) { ... }`**: If a `keyId` was also returned (meaning a specific API key was found):
                *   **`await updateApiKeyLastUsed(authResult.keyId).catch(...)`**: Asynchronously updates the `lastUsed` timestamp for the API key. `.catch()` ensures that failure to update the timestamp doesn't break the main request flow, merely logging a warning.
*   **`if (!authenticatedUserId) { ... }`**: After trying both session and API key authentication, if `authenticatedUserId` is still `null`, it means no valid user or API key was found.
    *   **`logger.warn(...)`**: Logs an unauthorized access attempt.
    *   **`return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })`**: Returns an HTTP 401 Unauthorized response.
*   **`userId = authenticatedUserId`**: If a user was successfully authenticated, assigns their ID to the `userId` variable.

```typescript
    let accessContext = null
    let workflowData = await getWorkflowById(workflowId)

    if (!workflowData) {
      logger.warn(`[${requestId}] Workflow ${workflowId} not found`)
      return NextResponse.json({ error: 'Workflow not found' }, { status: 404 })
    }

    // Check if user has access to this workflow
    let hasAccess = false

    if (isInternalCall) {
      // Internal calls have full access
      hasAccess = true
    } else {
      // Case 1: User owns the workflow
      if (workflowData) { // This check is redundant due to the early return above, but harmless.
        accessContext = await getWorkflowAccessContext(workflowId, userId ?? undefined)

        if (!accessContext) {
          logger.warn(`[${requestId}] Workflow ${workflowId} not found`)
          return NextResponse.json({ error: 'Workflow not found' }, { status: 404 })
        }

        workflowData = accessContext.workflow // Update workflowData with potentially richer info from context.

        if (accessContext.isOwner) {
          hasAccess = true
        }

        if (!hasAccess && workflowData.workspaceId && accessContext.workspacePermission) {
          hasAccess = true
        }
      }

      if (!hasAccess) {
        logger.warn(`[${requestId}] User ${userId} denied access to workflow ${workflowId}`)
        return NextResponse.json({ error: 'Access denied' }, { status: 403 })
      }
    }
```

*   **`let accessContext = null`**: Initializes a variable to store information about the user's access level to the workflow.
*   **`let workflowData = await getWorkflowById(workflowId)`**: Fetches the basic metadata of the workflow from the database.
*   **`if (!workflowData) { ... }`**: If no workflow is found with the given `workflowId`:
    *   **`logger.warn(...)`**: Logs that the workflow was not found.
    *   **`return NextResponse.json({ error: 'Workflow not found' }, { status: 404 })`**: Returns an HTTP 404 Not Found response.
*   **`let hasAccess = false`**: Initializes a flag to track if the authenticated user (or internal call) has access.
*   **`if (isInternalCall) { ... }`**: If it's an internal call, `hasAccess` is automatically set to `true` as internal services typically have full access.
*   **`else { ... }`**: If it's not an internal call, proceed with user-specific access checks:
    *   **`accessContext = await getWorkflowAccessContext(workflowId, userId ?? undefined)`**: Determines the user's access context for this workflow. `userId ?? undefined` is used because `getWorkflowAccessContext` expects `userId` to be `string | undefined`, not `string | null`.
    *   **`if (!accessContext) { ... }`**: This check re-confirms workflow existence *within the context helper*, potentially catching race conditions or stricter access requirements. If no context, implies workflow not found.
    *   **`workflowData = accessContext.workflow`**: Updates `workflowData` with the potentially more complete workflow object returned by `getWorkflowAccessContext`.
    *   **`if (accessContext.isOwner) { hasAccess = true }`**: If the user is the owner of the workflow, they have access.
    *   **`if (!hasAccess && workflowData.workspaceId && accessContext.workspacePermission) { hasAccess = true }`**: If the user is not the owner, but the workflow belongs to a workspace (`workflowData.workspaceId`) AND the user has *any* permission within that workspace (`accessContext.workspacePermission` is not null/undefined), they also have access for `GET` (read) operations.
*   **`if (!hasAccess) { ... }`**: If, after all checks, `hasAccess` is still `false`:
    *   **`logger.warn(...)`**: Logs an access denied attempt.
    *   **`return NextResponse.json({ error: 'Access denied' }, { status: 403 })`**: Returns an HTTP 403 Forbidden response.

```typescript
    logger.debug(`[${requestId}] Attempting to load workflow ${workflowId} from normalized tables`)
    const normalizedData = await loadWorkflowFromNormalizedTables(workflowId)

    if (normalizedData) {
      logger.debug(`[${requestId}] Found normalized data for workflow ${workflowId}:`, {
        blocksCount: Object.keys(normalizedData.blocks).length,
        edgesCount: normalizedData.edges.length,
        loopsCount: Object.keys(normalizedData.loops).length,
        parallelsCount: Object.keys(normalizedData.parallels).length,
        loops: normalizedData.loops,
      })

      const finalWorkflowData = {
        ...workflowData,
        state: {
          // Default values for expected properties
          deploymentStatuses: {},
          // Data from normalized tables
          blocks: normalizedData.blocks,
          edges: normalizedData.edges,
          loops: normalizedData.loops,
          parallels: normalizedData.parallels,
          lastSaved: Date.now(),
          isDeployed: workflowData.isDeployed || false,
          deployedAt: workflowData.deployedAt,
        },
      }

      logger.info(`[${requestId}] Loaded workflow ${workflowId} from normalized tables`)
      const elapsed = Date.now() - startTime
      logger.info(`[${requestId}] Successfully fetched workflow ${workflowId} in ${elapsed}ms`)

      return NextResponse.json({ data: finalWorkflowData }, { status: 200 })
    }
    return NextResponse.json({ error: 'Workflow has no normalized data' }, { status: 400 })
  } catch (error: any) {
    const elapsed = Date.now() - startTime
    logger.error(`[${requestId}] Error fetching workflow ${workflowId} after ${elapsed}ms`, error)
    return NextResponse.json({ error: 'Internal server error' }, { status: 500 })
  }
}
```

*   **`logger.debug(...)`**: Logs an attempt to load data from normalized tables.
*   **`const normalizedData = await loadWorkflowFromNormalizedTables(workflowId)`**: This is the core data loading step. It calls a helper to retrieve the detailed, structured representation of the workflow (its blocks, edges, etc.) from potentially several related database tables.
*   **`if (normalizedData) { ... }`**: If the `normalizedData` is successfully retrieved:
    *   **`logger.debug(...)`**: Logs the counts of different workflow components (blocks, edges, loops, parallels) found in the normalized data.
    *   **`const finalWorkflowData = { ...workflowData, state: { ... } }`**: Constructs the final response object. It takes the basic `workflowData` (metadata) and merges it with a `state` property.
        *   `state`: This object contains the complex structural data of the workflow, retrieved from `normalizedData`.
        *   `deploymentStatuses: {}` : A default empty object for deployment statuses.
        *   `blocks`, `edges`, `loops`, `parallels`: The actual workflow graph data.
        *   `lastSaved: Date.now()`: Records the current time as when the workflow state was last "saved" (or fetched).
        *   `isDeployed`, `deployedAt`: Inherits these properties from the `workflowData` (metadata).
    *   **`logger.info(...)`**: Logs that the workflow was successfully loaded from normalized tables.
    *   **`const elapsed = Date.now() - startTime`**: Calculates the total time taken for the request.
    *   **`logger.info(...)`**: Logs the successful fetch and its duration.
    *   **`return NextResponse.json({ data: finalWorkflowData }, { status: 200 })`**: Returns an HTTP 200 OK response with the `finalWorkflowData`.
*   **`return NextResponse.json({ error: 'Workflow has no normalized data' }, { status: 400 })`**: If `normalizedData` is `null` or `undefined` (meaning the workflow's state couldn't be loaded from normalized tables), it returns an HTTP 400 Bad Request indicating the absence of this structured data. *Note: The comment "fallback to JSON blob" implies there might be an alternative way to load older workflows that stored state as a JSON blob, but the current code only handles the normalized data path.*
*   **`catch (error: any) { ... }`**: Catches any errors that occurred within the `try` block.
    *   **`const elapsed = Date.now() - startTime`**: Calculates time to error.
    *   **`logger.error(...)`**: Logs the error with the request ID, workflow ID, and duration.
    *   **`return NextResponse.json({ error: 'Internal server error' }, { status: 500 })`**: Returns an HTTP 500 Internal Server Error response.

#### **`DELETE /api/workflows/[id]` - Delete a workflow**

```typescript
/**
 * DELETE /api/workflows/[id]
 * Delete a workflow by ID
 */
export async function DELETE(
  request: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  const requestId = generateRequestId()
  const startTime = Date.now()
  const { id: workflowId } = await params
```

*   **`export async function DELETE(...)`**: Defines the DELETE handler. Similar to GET, it takes `NextRequest` and `params`.
*   **`requestId`, `startTime`, `workflowId`**: Same initialization as in the GET handler.

```typescript
  try {
    const session = await getSession()
    if (!session?.user?.id) {
      logger.warn(`[${requestId}] Unauthorized deletion attempt for workflow ${workflowId}`)
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
    }

    const userId = session.user.id
```

*   **`try { ... } catch (error: any) { ... }`**: Main error handling block.
*   **`const session = await getSession()`**: Retrieves the user's session.
*   **`if (!session?.user?.id) { ... }`**: Checks if a valid user session exists. If not, the user is unauthorized to delete.
    *   **`logger.warn(...)`**: Logs the unauthorized attempt.
    *   **`return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })`**: Returns an HTTP 401 Unauthorized response.
*   **`const userId = session.user.id`**: Stores the authenticated user's ID.

```typescript
    const accessContext = await getWorkflowAccessContext(workflowId, userId)
    const workflowData = accessContext?.workflow || (await getWorkflowById(workflowId))

    if (!workflowData) {
      logger.warn(`[${requestId}] Workflow ${workflowId} not found for deletion`)
      return NextResponse.json({ error: 'Workflow not found' }, { status: 404 })
    }

    // Check if user has permission to delete this workflow
    let canDelete = false

    // Case 1: User owns the workflow
    if (workflowData.userId === userId) {
      canDelete = true
    }

    // Case 2: Workflow belongs to a workspace and user has admin permission
    if (!canDelete && workflowData.workspaceId) {
      const context = accessContext || (await getWorkflowAccessContext(workflowId, userId)) // Re-fetch context if it wasn't available before.
      if (context?.workspacePermission === 'admin') {
        canDelete = true
      }
    }

    if (!canDelete) {
      logger.warn(
        `[${requestId}] User ${userId} denied permission to delete workflow ${workflowId}`
      )
      return NextResponse.json({ error: 'Access denied' }, { status: 403 })
    }
```

*   **`const accessContext = await getWorkflowAccessContext(workflowId, userId)`**: Gets the user's access context for the workflow.
*   **`const workflowData = accessContext?.workflow || (await getWorkflowById(workflowId))`**: Retrieves the workflow data. It tries to get it from the `accessContext` first (which might be richer) or falls back to a direct `getWorkflowById` call.
*   **`if (!workflowData) { ... }`**: If the workflow is not found:
    *   **`logger.warn(...)`**: Logs the issue.
    *   **`return NextResponse.json({ error: 'Workflow not found' }, { status: 404 })`**: Returns an HTTP 404 Not Found response.
*   **`let canDelete = false`**: Initializes a flag for delete permissions.
*   **`if (workflowData.userId === userId) { canDelete = true }`**: The user can delete if they are the owner of the workflow.
*   **`if (!canDelete && workflowData.workspaceId) { ... }`**: If not the owner, check workspace permissions. This only applies if the workflow belongs to a `workspaceId`.
    *   **`const context = accessContext || (await getWorkflowAccessContext(workflowId, userId))`**: Ensures `context` is available, fetching it again if it was `null` initially (though it should be available if `workflowData` was found).
    *   **`if (context?.workspacePermission === 'admin') { canDelete = true }`**: The user can delete if they have 'admin' permission within the workflow's workspace.
*   **`if (!canDelete) { ... }`**: If the user lacks delete permissions:
    *   **`logger.warn(...)`**: Logs the denied attempt.
    *   **`return NextResponse.json({ error: 'Access denied' }, { status: 403 })`**: Returns an HTTP 403 Forbidden response.

```typescript
    // Check if workflow has published templates before deletion
    const { searchParams } = new URL(request.url)
    const checkTemplates = searchParams.get('check-templates') === 'true'
    const deleteTemplatesParam = searchParams.get('deleteTemplates')

    if (checkTemplates) {
      // Return template information for frontend to handle
      const publishedTemplates = await db
        .select()
        .from(templates)
        .where(eq(templates.workflowId, workflowId))

      return NextResponse.json({
        hasPublishedTemplates: publishedTemplates.length > 0,
        count: publishedTemplates.length,
        publishedTemplates: publishedTemplates.map((t) => ({
          id: t.id,
          name: t.name,
          views: t.views,
          stars: t.stars,
        })),
      })
    }

    // Handle template deletion based on user choice
    if (deleteTemplatesParam !== null) {
      const deleteTemplates = deleteTemplatesParam === 'delete'

      if (deleteTemplates) {
        // Delete all templates associated with this workflow
        await db.delete(templates).where(eq(templates.workflowId, workflowId))
        logger.info(`[${requestId}] Deleted templates for workflow ${workflowId}`)
      } else {
        // Orphan the templates (set workflowId to null)
        await db
          .update(templates)
          .set({ workflowId: null })
          .where(eq(templates.workflowId, workflowId))
        logger.info(`[${requestId}] Orphaned templates for workflow ${workflowId}`)
      }
    }
```

*   **`const { searchParams } = new URL(request.url)`**: Parses the request URL to access its query parameters (e.g., `?check-templates=true`).
*   **`const checkTemplates = searchParams.get('check-templates') === 'true'`**: Checks if the `check-templates` query parameter is set to `true`. This is likely a frontend-driven check before a final delete action.
*   **`const deleteTemplatesParam = searchParams.get('deleteTemplates')`**: Retrieves the `deleteTemplates` query parameter, which dictates how associated templates should be handled.
*   **`if (checkTemplates) { ... }`**: If `checkTemplates` is `true`, the API acts as an informational endpoint:
    *   **`const publishedTemplates = await db.select().from(templates).where(eq(templates.workflowId, workflowId))`**: Queries the `templates` table to find all templates linked to the workflow being deleted.
    *   **`return NextResponse.json({ ... })`**: Returns a JSON response containing whether any templates exist, their count, and a mapped list of their IDs, names, views, and stars. This allows the frontend to prompt the user about what to do with associated templates.
*   **`if (deleteTemplatesParam !== null) { ... }`**: If the `deleteTemplates` parameter is provided (meaning the user has made a choice on templates):
    *   **`const deleteTemplates = deleteTemplatesParam === 'delete'`**: Interprets the parameter: `true` if "delete", `false` if anything else (e.g., "orphan").
    *   **`if (deleteTemplates) { ... }`**: If `deleteTemplates` is `true`:
        *   **`await db.delete(templates).where(eq(templates.workflowId, workflowId))`**: Deletes all templates associated with this workflow from the database.
        *   **`logger.info(...)`**: Logs the deletion of templates.
    *   **`else { ... }`**: If `deleteTemplates` is `false` (meaning orphan):
        *   **`await db.update(templates).set({ workflowId: null }).where(eq(templates.workflowId, workflowId))`**: Updates the associated templates, setting their `workflowId` to `null`, effectively unlinking them from the workflow but keeping the template records.
        *   **`logger.info(...)`**: Logs the templated being orphaned.

```typescript
    await db.delete(workflow).where(eq(workflow.id, workflowId))

    const elapsed = Date.now() - startTime
    logger.info(`[${requestId}] Successfully deleted workflow ${workflowId} in ${elapsed}ms`)

    // Notify Socket.IO system to disconnect users from this workflow's room
    // This prevents "Block not found" errors when collaborative updates try to process
    // after the workflow has been deleted
    try {
      const socketUrl = env.SOCKET_SERVER_URL || 'http://localhost:3002'
      const socketResponse = await fetch(`${socketUrl}/api/workflow-deleted`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ workflowId }),
      })

      if (socketResponse.ok) {
        logger.info(
          `[${requestId}] Notified Socket.IO server about workflow ${workflowId} deletion`
        )
      } else {
        logger.warn(
          `[${requestId}] Failed to notify Socket.IO server about workflow ${workflowId} deletion`
        )
      }
    } catch (error) {
      logger.warn(
        `[${requestId}] Error notifying Socket.IO server about workflow ${workflowId} deletion:`,
        error
      )
      // Don't fail the deletion if Socket.IO notification fails
    }

    return NextResponse.json({ success: true }, { status: 200 })
  } catch (error: any) {
    const elapsed = Date.now() - startTime
    logger.error(`[${requestId}] Error deleting workflow ${workflowId} after ${elapsed}ms`, error)
    return NextResponse.json({ error: 'Internal server error' }, { status: 500 })
  }
}
```

*   **`await db.delete(workflow).where(eq(workflow.id, workflowId))`**: Performs the actual deletion of the workflow record from the `workflow` table.
*   **`const elapsed = Date.now() - startTime`**: Calculates the total time for the deletion.
*   **`logger.info(...)`**: Logs the successful deletion and its duration.
*   **`try { ... } catch (error) { ... }`**: A nested `try-catch` specifically for the Socket.IO notification. This is important: if the notification fails, the workflow should still be considered deleted, so the main deletion process doesn't revert.
    *   **`const socketUrl = env.SOCKET_SERVER_URL || 'http://localhost:3002'`**: Determines the URL of the Socket.IO server, using an environment variable or a default.
    *   **`const socketResponse = await fetch(`${socketUrl}/api/workflow-deleted`, { ... })`**: Sends an HTTP POST request to the Socket.IO server's `/api/workflow-deleted` endpoint, informing it that a workflow has been deleted. This is crucial for real-time collaboration: clients connected to this workflow's room should be disconnected or updated to prevent errors.
    *   **`if (socketResponse.ok) { ... } else { ... }`**: Logs whether the Socket.IO notification was successful or not.
    *   **`catch (error) { logger.warn(...) }`**: Catches errors during the `fetch` call to the Socket.IO server, logging a warning but allowing the overall deletion to succeed.
*   **`return NextResponse.json({ success: true }, { status: 200 })`**: Returns an HTTP 200 OK response indicating successful deletion.
*   **`catch (error: any) { ... }`**: Catches any errors in the main `DELETE` function.
    *   **`elapsed`, `logger.error(...)`**: Logs the error details and duration.
    *   **`return NextResponse.json({ error: 'Internal server error' }, { status: 500 })`**: Returns an HTTP 500 Internal Server Error response.

#### **`PUT /api/workflows/[id]` - Update workflow metadata**

```typescript
/**
 * PUT /api/workflows/[id]
 * Update workflow metadata (name, description, color, folderId)
 */
export async function PUT(request: NextRequest, { params }: { params: Promise<{ id: string }> }) {
  const requestId = generateRequestId()
  const startTime = Date.now()
  const { id: workflowId } = await params
```

*   **`export async function PUT(...)`**: Defines the PUT handler. Similar setup to GET and DELETE.
*   **`requestId`, `startTime`, `workflowId`**: Same initialization.

```typescript
  try {
    // Get the session
    const session = await getSession()
    if (!session?.user?.id) {
      logger.warn(`[${requestId}] Unauthorized update attempt for workflow ${workflowId}`)
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
    }

    const userId = session.user.id

    // Parse and validate request body
    const body = await request.json()
    const updates = UpdateWorkflowSchema.parse(body)
```

*   **`try { ... } catch (error: any) { ... }`**: Main error handling block.
*   **`const session = await getSession()`**: Retrieves the user session.
*   **`if (!session?.user?.id) { ... }`**: Checks for an authenticated user. If not found:
    *   **`logger.warn(...)`**: Logs unauthorized attempt.
    *   **`return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })`**: Returns HTTP 401.
*   **`const userId = session.user.id`**: Stores the authenticated user's ID.
*   **`const body = await request.json()`**: Parses the incoming request body as JSON.
*   **`const updates = UpdateWorkflowSchema.parse(body)`**: Validates the parsed `body` against the `UpdateWorkflowSchema` defined earlier using Zod. If the body doesn't match the schema, Zod will throw a `ZodError`.

```typescript
    // Fetch the workflow to check ownership/access
    const accessContext = await getWorkflowAccessContext(workflowId, userId)
    const workflowData = accessContext?.workflow || (await getWorkflowById(workflowId))

    if (!workflowData) {
      logger.warn(`[${requestId}] Workflow ${workflowId} not found for update`)
      return NextResponse.json({ error: 'Workflow not found' }, { status: 404 })
    }

    // Check if user has permission to update this workflow
    let canUpdate = false

    // Case 1: User owns the workflow
    if (workflowData.userId === userId) {
      canUpdate = true
    }

    // Case 2: Workflow belongs to a workspace and user has write or admin permission
    if (!canUpdate && workflowData.workspaceId) {
      const context = accessContext || (await getWorkflowAccessContext(workflowId, userId))
      if (context?.workspacePermission === 'write' || context?.workspacePermission === 'admin') {
        canUpdate = true
      }
    }

    if (!canUpdate) {
      logger.warn(
        `[${requestId}] User ${userId} denied permission to update workflow ${workflowId}`
      )
      return NextResponse.json({ error: 'Access denied' }, { status: 403 })
    }
```

*   **`const accessContext = await getWorkflowAccessContext(workflowId, userId)`**: Gets access context.
*   **`const workflowData = accessContext?.workflow || (await getWorkflowById(workflowId))`**: Retrieves workflow data.
*   **`if (!workflowData) { ... }`**: If workflow not found: logs and returns HTTP 404.
*   **`let canUpdate = false`**: Initializes flag for update permissions.
*   **`if (workflowData.userId === userId) { canUpdate = true }`**: User can update if they own the workflow.
*   **`if (!canUpdate && workflowData.workspaceId) { ... }`**: If not owner and workflow is in a workspace:
    *   **`const context = accessContext || (await getWorkflowAccessContext(workflowId, userId))`**: Ensures context is available.
    *   **`if (context?.workspacePermission === 'write' || context?.workspacePermission === 'admin') { canUpdate = true }`**: User can update if they have 'write' or 'admin' permission in the workspace.
*   **`if (!canUpdate) { ... }`**: If user lacks update permissions: logs and returns HTTP 403.

```typescript
    // Build update object
    const updateData: any = { updatedAt: new Date() }
    if (updates.name !== undefined) updateData.name = updates.name
    if (updates.description !== undefined) updateData.description = updates.description
    if (updates.color !== undefined) updateData.color = updates.color
    if (updates.folderId !== undefined) updateData.folderId = updates.folderId

    // Update the workflow
    const [updatedWorkflow] = await db
      .update(workflow)
      .set(updateData)
      .where(eq(workflow.id, workflowId))
      .returning()

    const elapsed = Date.now() - startTime
    logger.info(`[${requestId}] Successfully updated workflow ${workflowId} in ${elapsed}ms`, {
      updates: updateData,
    })

    return NextResponse.json({ workflow: updatedWorkflow }, { status: 200 })
  } catch (error: any) {
    const elapsed = Date.now() - startTime
    if (error instanceof z.ZodError) {
      logger.warn(`[${requestId}] Invalid workflow update data for ${workflowId}`, {
        errors: error.errors,
      })
      return NextResponse.json(
        { error: 'Invalid request data', details: error.errors },
        { status: 400 }
      )
    }

    logger.error(`[${requestId}] Error updating workflow ${workflowId} after ${elapsed}ms`, error)
    return NextResponse.json({ error: 'Internal server error' }, { status: 500 })
  }
}
```

*   **`const updateData: any = { updatedAt: new Date() }`**: Initializes an object `updateData` to hold the fields to be updated. `updatedAt` is always set to the current time. `any` is used here because the properties are added conditionally.
*   **`if (updates.name !== undefined) updateData.name = updates.name`**: Conditionally adds properties from the validated `updates` object to `updateData`. It checks for `!== undefined` to allow `null` values (e.g., `folderId: null`) to be passed through correctly if Zod allows it.
*   **`const [updatedWorkflow] = await db.update(workflow).set(updateData).where(eq(workflow.id, workflowId)).returning()`**: Performs the database update using Drizzle ORM.
    *   `db.update(workflow)`: Targets the `workflow` table.
    *   `.set(updateData)`: Specifies the fields and values to update.
    *   `.where(eq(workflow.id, workflowId))`: Filters the update to apply only to the workflow with the matching ID.
    *   `.returning()`: Tells Drizzle to return the updated record(s). Since we're updating a single record by ID, it returns an array with one element, which is destructured into `updatedWorkflow`.
*   **`const elapsed = Date.now() - startTime`**: Calculates the duration.
*   **`logger.info(...)`**: Logs the successful update and its duration, including the `updateData` object for context.
*   **`return NextResponse.json({ workflow: updatedWorkflow }, { status: 200 })`**: Returns an HTTP 200 OK response with the `updatedWorkflow` object.
*   **`catch (error: any) { ... }`**: Catches any errors during the `PUT` operation.
    *   **`const elapsed = Date.now() - startTime`**: Calculates the time to error.
    *   **`if (error instanceof z.ZodError) { ... }`**: Specifically checks if the error is a `ZodError`. This means the input validation failed.
        *   **`logger.warn(...)`**: Logs the validation error.
        *   **`return NextResponse.json({ error: 'Invalid request data', details: error.errors }, { status: 400 })`**: Returns an HTTP 400 Bad Request response, including details about the validation errors, which is very helpful for clients.
    *   **`logger.error(...)`**: If it's not a `ZodError`, it logs a general error.
    *   **`return NextResponse.json({ error: 'Internal server error' }, { status: 500 })`**: Returns an HTTP 500 Internal Server Error.