This TypeScript file defines an API route for managing **workflow log webhooks** within a Next.js application. It acts as the backend for creating, retrieving, and deleting webhooks that can be configured to send notifications about workflow execution logs to external services.

It leverages Drizzle ORM for database interactions, Zod for robust request body validation, and Next.js's API route capabilities for handling HTTP requests. Crucially, it incorporates strong authentication and authorization checks to ensure that only authorized users can manage webhooks for workflows they have access to.

---

## Detailed Explanation

### File Purpose and What It Does

This file is a Next.js API route handler, typically located at a path like `src/app/api/workflows/[id]/log-webhooks/route.ts` (or similar, given the `[id]` dynamic segment). It exposes three main functionalities:

1.  **`GET` Request:** Retrieves a list of all log webhooks associated with a specific workflow, provided the authenticated user has access to that workflow.
2.  **`POST` Request:** Creates a new log webhook for a specific workflow. It validates the incoming data, checks for duplicates, encrypts sensitive information (like secrets), and then stores the webhook configuration in the database.
3.  **`DELETE` Request:** Removes an existing log webhook from a specific workflow, again after verifying user authentication and authorization.

In essence, this file provides the CRUD (Create, Read, Delete) operations for log webhooks linked to a particular workflow ID.

### Imports

Let's break down each import statement:

```typescript
import { db } from '@sim/db'
```

*   **`db`**: This imports the Drizzle ORM database instance. `db` is your connection to the database (e.g., PostgreSQL, MySQL) and allows you to perform database operations like selecting, inserting, updating, and deleting data using a type-safe query builder.

```typescript
import { permissions, workflow, workflowLogWebhook } from '@sim/db/schema'
```

*   **`permissions`, `workflow`, `workflowLogWebhook`**: These are Drizzle ORM schema definitions. They represent the structure of tables in your database:
    *   `permissions`: Likely stores information about what users have access to what resources (e.g., workspaces).
    *   `workflow`: Defines the structure of the `workflows` table, which represents individual workflows in your application.
    *   `workflowLogWebhook`: Defines the structure of the `workflow_log_webhooks` table, which stores the configuration for each log webhook. These schemas are used to build type-safe queries.

```typescript
import { and, eq } from 'drizzle-orm'
```

*   **`and`, `eq`**: These are Drizzle ORM utility functions used to construct complex query conditions:
    *   `and()`: Combines multiple conditions with a logical `AND` operator (all conditions must be true).
    *   `eq()`: Checks if a column's value is equal to a specified value.

```typescript
import { type NextRequest, NextResponse } from 'next/server'
```

*   **`NextRequest`, `NextResponse`**: These are Next.js-specific types and classes for handling HTTP requests and responses in API routes:
    *   `NextRequest`: Represents the incoming HTTP request, providing access to its URL, headers, body, etc.
    *   `NextResponse`: Used to construct and send the HTTP response back to the client, allowing you to set status codes, JSON data, etc.

```typescript
import { v4 as uuidv4 } from 'uuid'
```

*   **`uuidv4`**: This imports the `v4` function from the `uuid` library. `uuidv4()` generates a universally unique identifier (UUID) of version 4, which is a random string useful for creating unique primary keys for database records.

```typescript
import { z } from 'zod'
```

*   **`z`**: This imports the main object from the `zod` library. Zod is a TypeScript-first schema declaration and validation library, used here to define and validate the structure and types of incoming request bodies.

```typescript
import { getSession } from '@/lib/auth'
```

*   **`getSession`**: This function is imported from a local utility file (`@/lib/auth`). It's responsible for retrieving the current user's authentication session, which typically contains the user's ID and other relevant information.

```typescript
import { createLogger } from '@/lib/logs/console/logger'
```

*   **`createLogger`**: This function is imported from a local logging utility. It's used to create a logger instance, enabling structured logging of events and errors within the API route.

```typescript
import { encryptSecret } from '@/lib/utils'
```

*   **`encryptSecret`**: This function is imported from a local utility file (`@/lib/utils`). It's used to securely encrypt sensitive data, specifically the webhook secret, before storing it in the database.

### Logger Setup

```typescript
const logger = createLogger('WorkflowLogWebhookAPI')
```

*   A `logger` instance is created with the name `'WorkflowLogWebhookAPI'`. This allows for organized logging, making it easier to identify logs originating from this specific API endpoint.

### `CreateWebhookSchema` (Zod Schema)

```typescript
const CreateWebhookSchema = z.object({
  url: z.string().url(),
  secret: z.string().optional(),
  includeFinalOutput: z.boolean().optional().default(false),
  includeTraceSpans: z.boolean().optional().default(false),
  includeRateLimits: z.boolean().optional().default(false),
  includeUsageData: z.boolean().optional().default(false),
  levelFilter: z
    .array(z.enum(['info', 'error']))
    .optional()
    .default(['info', 'error']),
  triggerFilter: z
    .array(z.enum(['api', 'webhook', 'schedule', 'manual', 'chat']))
    .optional()
    .default(['api', 'webhook', 'schedule', 'manual', 'chat']),
  active: z.boolean().optional().default(true),
})
```

This Zod schema defines the expected structure and validation rules for the data submitted when creating a new webhook.

*   `z.object({...})`: Defines that the expected input is a JavaScript object.
*   **`url: z.string().url()`**:
    *   `z.string()`: The `url` field must be a string.
    *   `.url()`: It must also be a valid URL format (e.g., `https://example.com/webhook`). This ensures the webhook can actually be called.
*   **`secret: z.string().optional()`**:
    *   `z.string()`: The `secret` field, if provided, must be a string.
    *   `.optional()`: This field is not required; it can be omitted from the request body. Webhooks often use secrets for signature verification to ensure requests are legitimate.
*   **`includeFinalOutput: z.boolean().optional().default(false)`**:
    *   `z.boolean()`: Must be a boolean (`true` or `false`).
    *   `.optional()`: This field is not required.
    *   `.default(false)`: If not provided, it will default to `false`. This likely controls whether the final output of the workflow execution should be included in the webhook payload.
*   **`includeTraceSpans: z.boolean().optional().default(false)`**: Similar to `includeFinalOutput`, defaults to `false`. Controls whether detailed trace spans (information about individual steps/operations within the workflow) should be included.
*   **`includeRateLimits: z.boolean().optional().default(false)`**: Similar, defaults to `false`. Controls whether rate limit information should be included.
*   **`includeUsageData: z.boolean().optional().default(false)`**: Similar, defaults to `false`. Controls whether resource usage data should be included.
*   **`levelFilter: z.array(z.enum(['info', 'error'])).optional().default(['info', 'error'])`**:
    *   `z.array(...)`: This field expects an array.
    *   `z.enum(['info', 'error'])`: Each item in the array must be one of the specified literal strings: `'info'` or `'error'`. This acts as a filter for the types of log levels the webhook should receive.
    *   `.optional()`: The field is not required.
    *   `.default(['info', 'error'])`: If not provided, it defaults to an array containing both `'info'` and `'error'`, meaning it will receive all log levels by default.
*   **`triggerFilter: z.array(z.enum(['api', 'webhook', 'schedule', 'manual', 'chat'])).optional().default(['api', 'webhook', 'schedule', 'manual', 'chat'])`**:
    *   Similar to `levelFilter`, but for workflow trigger types.
    *   It expects an array where each item is one of `'api'`, `'webhook'`, `'schedule'`, `'manual'`, or `'chat'`. This filters webhooks based on how the workflow was initiated.
    *   `.optional().default([...])`: Also optional and defaults to including all trigger types.
*   **`active: z.boolean().optional().default(true)`**:
    *   `z.boolean()`: Must be a boolean.
    *   `.optional().default(true)`: Optional, and defaults to `true`, meaning the webhook is active by default upon creation.

### `GET` Request Handler

This function handles HTTP `GET` requests to retrieve webhooks for a given workflow.

```typescript
export async function GET(request: NextRequest, { params }: { params: Promise<{ id: string }> }) {
  try {
    // 1. Authenticate User
    const session = await getSession()
    if (!session?.user?.id) {
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
    }

    // 2. Extract Workflow ID and User ID
    const { id: workflowId } = await params
    const userId = session.user.id

    // 3. Authorize User: Check if user has access to the workflow
    const hasAccess = await db
      .select({ id: workflow.id })
      .from(workflow)
      .innerJoin(
        permissions,
        and(
          eq(permissions.entityType, 'workspace'),
          eq(permissions.entityId, workflow.workspaceId),
          eq(permissions.userId, userId)
        )
      )
      .where(eq(workflow.id, workflowId))
      .limit(1)

    if (hasAccess.length === 0) {
      return NextResponse.json({ error: 'Workflow not found' }, { status: 404 })
    }

    // 4. Fetch Webhooks
    const webhooks = await db
      .select({
        id: workflowLogWebhook.id,
        url: workflowLogWebhook.url,
        includeFinalOutput: workflowLogWebhook.includeFinalOutput,
        includeTraceSpans: workflowLogWebhook.includeTraceSpans,
        includeRateLimits: workflowLogWebhook.includeRateLimits,
        includeUsageData: workflowLogWebhook.includeUsageData,
        levelFilter: workflowLogWebhook.levelFilter,
        triggerFilter: workflowLogWebhook.triggerFilter,
        active: workflowLogWebhook.active,
        createdAt: workflowLogWebhook.createdAt,
        updatedAt: workflowLogWebhook.updatedAt,
      })
      .from(workflowLogWebhook)
      .where(eq(workflowLogWebhook.workflowId, workflowId))

    // 5. Return Webhooks
    return NextResponse.json({ data: webhooks })
  } catch (error) {
    // 6. Handle Errors
    logger.error('Error fetching log webhooks', { error })
    return NextResponse.json({ error: 'Internal server error' }, { status: 500 })
  }
}
```

#### Line-by-Line Explanation (`GET`):

*   `export async function GET(request: NextRequest, { params }: { params: Promise<{ id: string }> }) { ... }`: This defines an asynchronous function that will handle `GET` requests.
    *   `request: NextRequest`: The incoming request object.
    *   `{ params }: { params: Promise<{ id: string }> }`: This destructures `params` from the second argument. `params` will contain dynamic route segments, in this case, `id` (e.g., if the route is `/api/workflows/[id]/log-webhooks`, `id` will be the workflow ID). It's wrapped in `Promise` because Next.js might resolve dynamic params asynchronously.
*   `try { ... } catch (error) { ... }`: A standard `try...catch` block to gracefully handle any errors that occur during the request processing.
*   **Authentication:**
    *   `const session = await getSession()`: Calls the `getSession` utility to retrieve the current user's session.
    *   `if (!session?.user?.id) { ... }`: Checks if a session exists and if a user ID is present in the session. If not, the user is not authenticated.
    *   `return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })`: Returns a `401 Unauthorized` HTTP response.
*   **Extract IDs:**
    *   `const { id: workflowId } = await params`: Extracts the `id` from the `params` object and renames it to `workflowId` for clarity.
    *   `const userId = session.user.id`: Gets the authenticated user's ID from the session.
*   **Authorization (Complex Logic Simplified):**
    *   `const hasAccess = await db ...`: This is a Drizzle ORM query designed to check if the `userId` has permissions to access the `workflowId`.
    *   `.select({ id: workflow.id })`: We only need to select one column (`id` from the `workflow` table) to confirm if a record exists. We don't need all workflow details here.
    *   `.from(workflow)`: Start querying the `workflow` table.
    *   `.innerJoin(permissions, ...)`: This is the core of the authorization check. It joins the `workflow` table with the `permissions` table. An `INNER JOIN` means only rows where there's a match in *both* tables will be included.
    *   `and(...)`: This combines multiple conditions for the join:
        *   `eq(permissions.entityType, 'workspace')`: The permission must be specifically for a 'workspace' type of entity. This implies permissions are managed at the workspace level, not directly on individual workflows.
        *   `eq(permissions.entityId, workflow.workspaceId)`: The `entityId` in the `permissions` table must match the `workspaceId` of the `workflow`. This links the permission to the specific workspace that owns the workflow.
        *   `eq(permissions.userId, userId)`: The permission must belong to the currently authenticated user (`userId`).
    *   `.where(eq(workflow.id, workflowId))`: Further filters the results to ensure we're looking at the specific `workflowId` requested in the URL.
    *   `.limit(1)`: We only need to find *one* matching record to confirm access, so we limit the result set for efficiency.
    *   `if (hasAccess.length === 0) { ... }`: If the query returns an empty array, it means no permission record was found for that user, workspace, and workflow combination.
    *   `return NextResponse.json({ error: 'Workflow not found' }, { status: 404 })`: Returns a `404 Not Found` response. While it's an authorization failure, returning 'Workflow not found' is often a security best practice to avoid leaking information about existing workflows.
*   **Fetch Webhooks:**
    *   `const webhooks = await db.select({ ... }).from(workflowLogWebhook).where(eq(workflowLogWebhook.workflowId, workflowId))`: After successful authorization, this query fetches all log webhooks (`workflowLogWebhook`) that are associated with the `workflowId`.
    *   `.select({...})`: Specifies which columns from the `workflowLogWebhook` table should be returned.
*   **Return Response:**
    *   `return NextResponse.json({ data: webhooks })`: Sends a `200 OK` response with the fetched `webhooks` data as a JSON object.
*   **Error Handling:**
    *   `logger.error('Error fetching log webhooks', { error })`: Logs the error details for debugging purposes.
    *   `return NextResponse.json({ error: 'Internal server error' }, { status: 500 })`: Returns a `500 Internal Server Error` response if any unexpected error occurs.

### `POST` Request Handler

This function handles HTTP `POST` requests to create a new webhook for a given workflow.

```typescript
export async function POST(request: NextRequest, { params }: { params: Promise<{ id: string }> }) {
  try {
    // 1. Authenticate User (Same as GET)
    const session = await getSession()
    if (!session?.user?.id) {
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
    }

    // 2. Extract Workflow ID and User ID (Same as GET)
    const { id: workflowId } = await params
    const userId = session.user.id

    // 3. Authorize User: Check if user has access to the workflow (Same as GET)
    const hasAccess = await db
      .select({ id: workflow.id })
      .from(workflow)
      .innerJoin(
        permissions,
        and(
          eq(permissions.entityType, 'workspace'),
          eq(permissions.entityId, workflow.workspaceId),
          eq(permissions.userId, userId)
        )
      )
      .where(eq(workflow.id, workflowId))
      .limit(1)

    if (hasAccess.length === 0) {
      return NextResponse.json({ error: 'Workflow not found' }, { status: 404 })
    }

    // 4. Validate Request Body
    const body = await request.json()
    const validationResult = CreateWebhookSchema.safeParse(body)

    if (!validationResult.success) {
      return NextResponse.json(
        { error: 'Invalid request', details: validationResult.error.errors },
        { status: 400 }
      )
    }
    const data = validationResult.data

    // 5. Check for Duplicate URL
    const existingWebhook = await db
      .select({ id: workflowLogWebhook.id })
      .from(workflowLogWebhook)
      .where(
        and(eq(workflowLogWebhook.workflowId, workflowId), eq(workflowLogWebhook.url, data.url))
      )
      .limit(1)

    if (existingWebhook.length > 0) {
      return NextResponse.json(
        { error: 'A webhook with this URL already exists for this workflow' },
        { status: 409 }
      )
    }

    // 6. Encrypt Secret (if provided)
    let encryptedSecret: string | null = null
    if (data.secret) {
      const { encrypted } = await encryptSecret(data.secret)
      encryptedSecret = encrypted
    }

    // 7. Insert New Webhook into Database
    const [webhook] = await db
      .insert(workflowLogWebhook)
      .values({
        id: uuidv4(),
        workflowId,
        url: data.url,
        secret: encryptedSecret,
        includeFinalOutput: data.includeFinalOutput,
        includeTraceSpans: data.includeTraceSpans,
        includeRateLimits: data.includeRateLimits,
        includeUsageData: data.includeUsageData,
        levelFilter: data.levelFilter,
        triggerFilter: data.triggerFilter,
        active: data.active,
      })
      .returning()

    // 8. Log Success
    logger.info('Created log webhook', {
      workflowId,
      webhookId: webhook.id,
      url: data.url,
    })

    // 9. Return Created Webhook Details
    return NextResponse.json({
      data: {
        id: webhook.id,
        url: webhook.url,
        includeFinalOutput: webhook.includeFinalOutput,
        includeTraceSpans: webhook.includeTraceSpans,
        includeRateLimits: webhook.includeRateLimits,
        includeUsageData: webhook.includeUsageData,
        levelFilter: webhook.levelFilter,
        triggerFilter: webhook.triggerFilter,
        active: webhook.active,
        createdAt: webhook.createdAt,
        updatedAt: webhook.updatedAt,
      },
    })
  } catch (error) {
    // 10. Handle Errors (Same as GET)
    logger.error('Error creating log webhook', { error })
    return NextResponse.json({ error: 'Internal server error' }, { status: 500 })
  }
}
```

#### Line-by-Line Explanation (`POST`):

*   `export async function POST(...) { ... }`: Defines the asynchronous function for `POST` requests.
*   **Authentication and Authorization:** The first few steps (getting session, extracting IDs, and checking `hasAccess`) are identical to the `GET` handler, ensuring that only authenticated and authorized users can create webhooks.
*   **Request Body Validation:**
    *   `const body = await request.json()`: Parses the incoming request body as JSON.
    *   `const validationResult = CreateWebhookSchema.safeParse(body)`: Uses the Zod schema defined earlier to validate the `body`. `safeParse` is non-throwing; it returns an object indicating success or failure.
    *   `if (!validationResult.success) { ... }`: If validation fails (`success` is `false`).
    *   `return NextResponse.json({ error: 'Invalid request', details: validationResult.error.errors }, { status: 400 })`: Returns a `400 Bad Request` with details about what failed validation.
    *   `const data = validationResult.data`: If validation is successful, `data` contains the validated and type-inferred object.
*   **Check for Duplicate URL:**
    *   `const existingWebhook = await db.select({ id: workflowLogWebhook.id }).from(workflowLogWebhook).where(and(eq(workflowLogWebhook.workflowId, workflowId), eq(workflowLogWebhook.url, data.url))).limit(1)`: This query checks if a webhook with the same `url` already exists for *this specific workflow*.
    *   `if (existingWebhook.length > 0) { ... }`: If a duplicate is found.
    *   `return NextResponse.json({ error: 'A webhook with this URL already exists for this workflow' }, { status: 409 })`: Returns a `409 Conflict` status, indicating that the request conflicts with the current state of the resource.
*   **Encrypt Secret:**
    *   `let encryptedSecret: string | null = null`: Initializes a variable to store the encrypted secret.
    *   `if (data.secret) { ... }`: If a `secret` was provided in the request body.
    *   `const { encrypted } = await encryptSecret(data.secret)`: Calls the `encryptSecret` utility to encrypt the plain-text secret. This is crucial for security, as secrets should never be stored in plain text.
    *   `encryptedSecret = encrypted`: Stores the encrypted value.
*   **Insert New Webhook:**
    *   `const [webhook] = await db.insert(workflowLogWebhook).values({ ... }).returning()`: This Drizzle ORM query inserts a new row into the `workflowLogWebhook` table.
    *   `id: uuidv4()`: Generates a unique ID for the new webhook.
    *   `workflowId`: Links the webhook to the current workflow.
    *   `url: data.url`, `secret: encryptedSecret`, etc.: Populates the remaining fields from the validated `data` object and the encrypted secret.
    *   `.returning()`: This Drizzle-specific method causes the `INSERT` query to return the inserted row(s). We destructure `[webhook]` because `returning()` always returns an array, even for a single insert.
*   **Log Success:**
    *   `logger.info('Created log webhook', { ... })`: Logs a successful creation event, including relevant IDs and the URL for traceability.
*   **Return Created Webhook:**
    *   `return NextResponse.json({ data: { ... } })`: Returns a `200 OK` response with selected details of the newly created webhook. The `secret` is intentionally *not* returned in plain text.
*   **Error Handling:**
    *   Similar to the `GET` handler, catches and logs errors, returning a `500 Internal Server Error`.

### `DELETE` Request Handler

This function handles HTTP `DELETE` requests to remove an existing webhook.

```typescript
export async function DELETE(
  request: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  try {
    // 1. Authenticate User (Same as GET/POST)
    const session = await getSession()
    if (!session?.user?.id) {
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
    }

    // 2. Extract Workflow ID and User ID
    const { id: workflowId } = await params
    const userId = session.user.id

    // 3. Get Webhook ID from Query Parameters
    const { searchParams } = new URL(request.url)
    const webhookId = searchParams.get('webhookId')

    if (!webhookId) {
      return NextResponse.json({ error: 'webhookId is required' }, { status: 400 })
    }

    // 4. Authorize User: Check if user has access to the workflow (Same as GET/POST)
    const hasAccess = await db
      .select({ id: workflow.id })
      .from(workflow)
      .innerJoin(
        permissions,
        and(
          eq(permissions.entityType, 'workspace'),
          eq(permissions.entityId, workflow.workspaceId),
          eq(permissions.userId, userId)
        )
      )
      .where(eq(workflow.id, workflowId))
      .limit(1)

    if (hasAccess.length === 0) {
      return NextResponse.json({ error: 'Workflow not found' }, { status: 404 })
    }

    // 5. Delete Webhook from Database
    const deleted = await db
      .delete(workflowLogWebhook)
      .where(
        and(eq(workflowLogWebhook.id, webhookId), eq(workflowLogWebhook.workflowId, workflowId))
      )
      .returning({ id: workflowLogWebhook.id })

    // 6. Check if deletion was successful
    if (deleted.length === 0) {
      return NextResponse.json({ error: 'Webhook not found' }, { status: 404 })
    }

    // 7. Log Success
    logger.info('Deleted log webhook', {
      workflowId,
      webhookId,
    })

    // 8. Return Success
    return NextResponse.json({ success: true })
  } catch (error) {
    // 9. Handle Errors
    logger.error('Error deleting log webhook', { error })
    return NextResponse.json({ error: 'Internal server error' }, { status: 500 })
  }
}
```

#### Line-by-Line Explanation (`DELETE`):

*   `export async function DELETE(...) { ... }`: Defines the asynchronous function for `DELETE` requests.
*   **Authentication and Authorization:** The initial steps are again identical, ensuring only authorized users can delete webhooks.
*   **Extract Webhook ID:**
    *   `const { searchParams } = new URL(request.url)`: Creates a `URL` object from the `request.url` to easily parse query parameters.
    *   `const webhookId = searchParams.get('webhookId')`: Retrieves the value of the `webhookId` query parameter (e.g., from a URL like `/api/workflows/123/log-webhooks?webhookId=abc-def`).
    *   `if (!webhookId) { ... }`: Checks if `webhookId` was provided.
    *   `return NextResponse.json({ error: 'webhookId is required' }, { status: 400 })`: Returns a `400 Bad Request` if `webhookId` is missing.
*   **Delete Webhook:**
    *   `const deleted = await db.delete(workflowLogWebhook).where(and(eq(workflowLogWebhook.id, webhookId), eq(workflowLogWebhook.workflowId, workflowId))).returning({ id: workflowLogWebhook.id })`: This Drizzle ORM query performs the deletion.
    *   `.delete(workflowLogWebhook)`: Specifies that we are deleting from the `workflowLogWebhook` table.
    *   `.where(and(eq(workflowLogWebhook.id, webhookId), eq(workflowLogWebhook.workflowId, workflowId)))`: This is a crucial safety measure. It ensures that the webhook to be deleted not only matches the `webhookId` but also belongs to the `workflowId` specified in the path parameters. This prevents a user with access to workflow A from deleting a webhook belonging to workflow B by just guessing its ID.
    *   `.returning({ id: workflowLogWebhook.id })`: Returns the ID of the deleted webhook, which can be used to confirm that a record was indeed deleted.
*   **Check Deletion Success:**
    *   `if (deleted.length === 0) { ... }`: If `deleted` is an empty array, it means no webhook matching the criteria was found and deleted.
    *   `return NextResponse.json({ error: 'Webhook not found' }, { status: 404 })`: Returns a `404 Not Found`.
*   **Log Success:**
    *   `logger.info('Deleted log webhook', { ... })`: Logs the successful deletion.
*   **Return Success:**
    *   `return NextResponse.json({ success: true })`: Returns a `200 OK` response with a simple success message.
*   **Error Handling:**
    *   Catches and logs errors, returning a `500 Internal Server Error`.

---

This file demonstrates a robust and well-structured approach to building API endpoints for resource management in a Next.js application, integrating authentication, authorization, data validation, secure storage practices, and comprehensive error handling.