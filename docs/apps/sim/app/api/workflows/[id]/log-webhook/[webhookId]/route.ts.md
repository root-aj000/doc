This TypeScript file defines two API route handlers, `PUT` and `DELETE`, for managing workflow log webhooks in a Next.js application. These handlers allow users to update an existing webhook's configuration or delete it entirely, ensuring proper authentication, authorization, and data validation.

Let's break down the file section by section.

---

### **Purpose of this File and What it Does**

This file acts as the backend API endpoint for managing specific workflow log webhooks. It exposes two primary functions:

1.  **`PUT` (Update)**: This function handles `HTTP PUT` requests to update the details of an existing workflow log webhook. It performs:
    *   User authentication.
    *   Authorization check to ensure the user has access to the parent workflow.
    *   Validation of the incoming request payload using Zod.
    *   Checks for the existence of the webhook and prevents duplicate URLs within the same workflow.
    *   Updates the webhook's configuration in the database.
    *   Encrypts sensitive data like the webhook secret if provided.
2.  **`DELETE` (Delete)**: This function handles `HTTP DELETE` requests to remove a workflow log webhook. It performs:
    *   User authentication.
    *   Authorization check to ensure the user has access to the parent workflow.
    *   Deletes the specified webhook from the database.

In essence, this file provides the necessary logic for users to securely modify or remove webhooks associated with their workflows, ensuring data integrity and access control.

---

### **Detailed Line-by-Line Explanation**

#### **1. Imports and Setup**

```typescript
import { db } from '@sim/db'
import { permissions, workflow, workflowLogWebhook } from '@sim/db/schema'
import { and, eq } from 'drizzle-orm'
import { type NextRequest, NextResponse } from 'next/server'
import { z } from 'zod'
import { getSession } from '@/lib/auth'
import { createLogger } from '@/lib/logs/console/logger'
import { encryptSecret } from '@/lib/utils'
```

*   **`import { db } from '@sim/db'`**:
    *   Imports the database client instance, typically configured for Drizzle ORM, from a shared `db` module within the `@sim` monorepo or project. This `db` object is essential for all database interactions (queries, updates, deletes).
*   **`import { permissions, workflow, workflowLogWebhook } from '@sim/db/schema'`**:
    *   Imports the Drizzle ORM schema definitions for three database tables:
        *   `permissions`: Likely stores user permissions related to various entities (e.g., workspaces).
        *   `workflow`: Represents individual workflows in the system.
        *   `workflowLogWebhook`: Represents webhooks configured to send workflow log data.
    *   These schema objects are used to construct database queries and define the structure of the data.
*   **`import { and, eq } from 'drizzle-orm'`**:
    *   Imports specific functions from the `drizzle-orm` library:
        *   `and`: Used to combine multiple conditions in a SQL `WHERE` clause with a logical AND operator.
        *   `eq`: Used to create an equality condition (e.g., `column = value`).
    *   These are fundamental for building complex Drizzle queries.
*   **`import { type NextRequest, NextResponse } from 'next/server'`**:
    *   Imports types and classes specific to Next.js API route handlers:
        *   `NextRequest`: A type representing the incoming HTTP request object, extending standard `Request` with Next.js specific features.
        *   `NextResponse`: A class used to construct and send HTTP responses from API routes.
*   **`import { z } from 'zod'`**:
    *   Imports the Zod library, a TypeScript-first schema declaration and validation library. It's used here to define and validate the structure and types of incoming request bodies.
*   **`import { getSession } from '@/lib/auth'`**:
    *   Imports a utility function `getSession` from `@/lib/auth`. This function is responsible for retrieving the current user's session information, typically used for authentication and identifying the logged-in user.
*   **`import { createLogger } from '@/lib/logs/console/logger'`**:
    *   Imports a `createLogger` function from a custom logging utility. This allows for creating a logger instance with a specific name, helping to categorize and filter log messages.
*   **`import { encryptSecret } from '@/lib/utils'`**:
    *   Imports a utility function `encryptSecret` from `@/lib/utils`. This function is likely used to securely encrypt sensitive data, such as webhook secrets, before storing them in the database.

---

#### **2. Logger Initialization**

```typescript
const logger = createLogger('WorkflowLogWebhookUpdate')
```

*   **`const logger = createLogger('WorkflowLogWebhookUpdate')`**:
    *   Initializes a logger instance. All log messages originating from this file will be associated with the name `'WorkflowLogWebhookUpdate'`, making it easier to trace logs related to these API operations.

---

#### **3. Type Definitions and Validation Schema**

```typescript
type WebhookUpdatePayload = Pick<
  typeof workflowLogWebhook.$inferInsert,
  | 'url'
  | 'includeFinalOutput'
  | 'includeTraceSpans'
  | 'includeRateLimits'
  | 'includeUsageData'
  | 'levelFilter'
  | 'triggerFilter'
  | 'secret'
  | 'updatedAt'
>

const UpdateWebhookSchema = z.object({
  url: z.string().url('Invalid webhook URL'),
  secret: z.string().optional(),
  includeFinalOutput: z.boolean(),
  includeTraceSpans: z.boolean(),
  includeRateLimits: z.boolean(),
  includeUsageData: z.boolean(),
  levelFilter: z.array(z.enum(['info', 'error'])),
  triggerFilter: z.array(z.enum(['api', 'webhook', 'schedule', 'manual', 'chat'])),
})
```

*   **`type WebhookUpdatePayload = Pick<typeof workflowLogWebhook.$inferInsert, ...>`**:
    *   Defines a TypeScript type `WebhookUpdatePayload`.
    *   **`typeof workflowLogWebhook.$inferInsert`**: Drizzle ORM schemas (`workflowLogWebhook` in this case) provide helper types. `$inferInsert` extracts the TypeScript type for the data shape required when inserting a new record into the `workflowLogWebhook` table.
    *   **`Pick<..., ...>`**: This is a TypeScript utility type that constructs a new type by picking a set of properties from an existing type.
    *   In this line, `WebhookUpdatePayload` is created by picking specific fields (`url`, `includeFinalOutput`, etc.) from the `$inferInsert` type of `workflowLogWebhook`. This ensures that the data object used for updating the webhook precisely matches the expected database schema fields, preventing type errors. `updatedAt` is included because it's set on update.
*   **`const UpdateWebhookSchema = z.object({ ... })`**:
    *   Defines a Zod schema named `UpdateWebhookSchema`. This schema specifies the expected structure and types of the data that will be received in the `PUT` request body.
    *   **`url: z.string().url('Invalid webhook URL')`**: The `url` field must be a string and a valid URL. If not, it will return the specified error message.
    *   **`secret: z.string().optional()`**: The `secret` field must be a string, but it's optional, meaning it might not be present in the request body.
    *   **`includeFinalOutput: z.boolean()`**, **`includeTraceSpans: z.boolean()`**, **`includeRateLimits: z.boolean()`**, **`includeUsageData: z.boolean()`**: These fields must all be boolean values.
    *   **`levelFilter: z.array(z.enum(['info', 'error']))`**: The `levelFilter` field must be an array. Each item in the array must be one of the specified literal strings: `'info'` or `'error'`. `z.enum` ensures strict type checking for these values.
    *   **`triggerFilter: z.array(z.enum(['api', 'webhook', 'schedule', 'manual', 'chat']))`**: Similar to `levelFilter`, this field must be an array, and each item must be one of the specified literal trigger types.
    *   This schema is crucial for validating incoming API requests and providing clear error messages if the data doesn't conform.

---

#### **4. `PUT` Function (Update Workflow Log Webhook)**

This function handles `PUT` requests to `/api/workflows/[id]/webhooks/[webhookId]`, allowing an authenticated and authorized user to update an existing webhook.

```typescript
export async function PUT(
  request: NextRequest,
  { params }: { params: Promise<{ id: string; webhookId: string }> }
) {
  try {
    const session = await getSession()
    if (!session?.user?.id) {
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
    }

    const { id: workflowId, webhookId } = await params
    const userId = session.user.id

    // Check if user has access to the workflow
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

    // Check if webhook exists and belongs to this workflow
    const existingWebhook = await db
      .select()
      .from(workflowLogWebhook)
      .where(
        and(eq(workflowLogWebhook.id, webhookId), eq(workflowLogWebhook.workflowId, workflowId))
      )
      .limit(1)

    if (existingWebhook.length === 0) {
      return NextResponse.json({ error: 'Webhook not found' }, { status: 404 })
    }

    const body = await request.json()
    const validationResult = UpdateWebhookSchema.safeParse(body)

    if (!validationResult.success) {
      return NextResponse.json(
        { error: 'Invalid request', details: validationResult.error.errors },
        { status: 400 }
      )
    }

    const data = validationResult.data

    // Check for duplicate URL (excluding current webhook)
    const duplicateWebhook = await db
      .select({ id: workflowLogWebhook.id })
      .from(workflowLogWebhook)
      .where(
        and(eq(workflowLogWebhook.workflowId, workflowId), eq(workflowLogWebhook.url, data.url))
      )
      .limit(1)

    if (duplicateWebhook.length > 0 && duplicateWebhook[0].id !== webhookId) {
      return NextResponse.json(
        { error: 'A webhook with this URL already exists for this workflow' },
        { status: 409 }
      )
    }

    // Prepare update data
    const updateData: WebhookUpdatePayload = {
      url: data.url,
      includeFinalOutput: data.includeFinalOutput,
      includeTraceSpans: data.includeTraceSpans,
      includeRateLimits: data.includeRateLimits,
      includeUsageData: data.includeUsageData,
      levelFilter: data.levelFilter,
      triggerFilter: data.triggerFilter,
      updatedAt: new Date(),
    }

    // Only update secret if provided
    if (data.secret) {
      const { encrypted } = await encryptSecret(data.secret)
      updateData.secret = encrypted
    }

    const updatedWebhooks = await db
      .update(workflowLogWebhook)
      .set(updateData)
      .where(eq(workflowLogWebhook.id, webhookId))
      .returning()

    if (updatedWebhooks.length === 0) {
      return NextResponse.json({ error: 'Webhook not found' }, { status: 404 })
    }

    const updatedWebhook = updatedWebhooks[0]

    logger.info('Webhook updated', {
      webhookId,
      workflowId,
      userId,
    })

    return NextResponse.json({
      data: {
        id: updatedWebhook.id,
        url: updatedWebhook.url,
        includeFinalOutput: updatedWebhook.includeFinalOutput,
        includeTraceSpans: updatedWebhook.includeTraceSpans,
        includeRateLimits: updatedWebhook.includeRateLimits,
        includeUsageData: updatedWebhook.includeUsageData,
        levelFilter: updatedWebhook.levelFilter,
        triggerFilter: updatedWebhook.triggerFilter,
        active: updatedWebhook.active,
        createdAt: updatedWebhook.createdAt.toISOString(),
        updatedAt: updatedWebhook.updatedAt.toISOString(),
      },
    })
  } catch (error) {
    logger.error('Failed to update webhook', { error })
    return NextResponse.json({ error: 'Failed to update webhook' }, { status: 500 })
  }
}
```

*   **`export async function PUT(...)`**:
    *   Defines an asynchronous function named `PUT`. Next.js automatically maps `HTTP PUT` requests to this function when the route matches the file's path (e.g., `app/api/workflows/[id]/webhooks/[webhookId]/route.ts`).
    *   **`request: NextRequest`**: The incoming HTTP request object provided by Next.js.
    *   **`{ params }: { params: Promise<{ id: string; webhookId: string }> }`**: This object contains route parameters.
        *   `params`: An object that holds dynamic segments from the URL.
        *   `id`: Corresponds to `[id]` in the route, representing the `workflowId`.
        *   `webhookId`: Corresponds to `[webhookId]` in the route.
        *   `Promise<{ ... }>`: Next.js often resolves route params for you, but technically, they can be awaited.
*   **`try { ... } catch (error) { ... }`**:
    *   A standard `try...catch` block to gracefully handle any errors that occur during the execution of the `PUT` request. If an error occurs, it's logged, and a `500 Internal Server Error` response is sent.
*   **`const session = await getSession()`**:
    *   Calls the `getSession` utility to retrieve the current user's session.
*   **`if (!session?.user?.id) { ... }`**:
    *   Checks if a valid user session exists and if the user ID is present. If not, the request is `Unauthorized` (HTTP `401`).
*   **`const { id: workflowId, webhookId } = await params`**:
    *   Destructures the `params` object to extract `id` (renamed to `workflowId` for clarity) and `webhookId`.
*   **`const userId = session.user.id`**:
    *   Retrieves the ID of the authenticated user from the session.
*   **`// Check if user has access to the workflow`**:
    *   This section performs an authorization check.
    *   **`const hasAccess = await db.select({ id: workflow.id }).from(workflow)`**: Starts a Drizzle query to select the `id` from the `workflow` table.
    *   **`.innerJoin(permissions, ...)`**: Joins the `workflow` table with the `permissions` table.
    *   **`and( ... )`**: Specifies multiple conditions for the join:
        *   `eq(permissions.entityType, 'workspace')`: The permission must be for a 'workspace' entity.
        *   `eq(permissions.entityId, workflow.workspaceId)`: The permission's `entityId` must match the `workspaceId` of the current `workflow`. This links the workflow to its parent workspace and checks permissions at the workspace level.
        *   `eq(permissions.userId, userId)`: The permission must belong to the current `userId`.
    *   **`.where(eq(workflow.id, workflowId))`**: Filters the results to specifically find the workflow with the ID provided in the URL `workflowId`.
    *   **`.limit(1)`**: Optimizes the query by asking for only one result, as we only need to know if *any* matching workflow exists.
*   **`if (hasAccess.length === 0) { ... }`**:
    *   If no matching workflow is found (meaning the user doesn't have access or the workflow doesn't exist), returns a `404 Not Found` response.
*   **`// Check if webhook exists and belongs to this workflow`**:
    *   This section verifies that the `webhookId` provided in the URL actually corresponds to an existing webhook and that it belongs to the `workflowId`.
    *   **`const existingWebhook = await db.select().from(workflowLogWebhook)`**: Selects all columns from the `workflowLogWebhook` table.
    *   **`.where(and(eq(workflowLogWebhook.id, webhookId), eq(workflowLogWebhook.workflowId, workflowId)))`**: Filters by both the `webhookId` and `workflowId` to ensure the webhook is associated with the correct workflow.
    *   **`.limit(1)`**: Again, limits to one result for efficiency.
*   **`if (existingWebhook.length === 0) { ... }`**:
    *   If no such webhook is found, returns a `404 Not Found` response.
*   **`const body = await request.json()`**:
    *   Parses the incoming request body as JSON.
*   **`const validationResult = UpdateWebhookSchema.safeParse(body)`**:
    *   Uses the Zod schema `UpdateWebhookSchema` to validate the parsed request body. `safeParse` returns an object indicating success or failure without throwing an error, making it suitable for API responses.
*   **`if (!validationResult.success) { ... }`**:
    *   If validation fails, returns a `400 Bad Request` response, including details about the validation errors.
*   **`const data = validationResult.data`**:
    *   If validation is successful, `validationResult.data` contains the parsed and validated data.
*   **`// Check for duplicate URL (excluding current webhook)`**:
    *   This section prevents creating a duplicate webhook *URL* within the same workflow, but allows the current webhook to keep its URL.
    *   **`const duplicateWebhook = await db.select({ id: workflowLogWebhook.id }).from(workflowLogWebhook)`**: Selects the ID of any existing webhooks.
    *   **`.where(and(eq(workflowLogWebhook.workflowId, workflowId), eq(workflowLogWebhook.url, data.url)))`**: Finds webhooks within the *same workflow* that have the *same URL* as the one provided in the request.
    *   **`if (duplicateWebhook.length > 0 && duplicateWebhook[0].id !== webhookId) { ... }`**:
        *   If a duplicate URL is found (`duplicateWebhook.length > 0`) AND it's *not* the current webhook being updated (`duplicateWebhook[0].id !== webhookId`), then it's an actual duplicate.
        *   Returns a `409 Conflict` response, indicating that the resource could not be updated due to a conflict (duplicate URL).
*   **`// Prepare update data`**:
    *   **`const updateData: WebhookUpdatePayload = { ... }`**: Creates an object `updateData` that will be used to update the database record. Its type is `WebhookUpdatePayload` to ensure it matches the schema.
    *   All fields from the validated `data` (except `secret`, which is handled conditionally) are copied into `updateData`.
    *   **`updatedAt: new Date()`**: The `updatedAt` timestamp is set to the current date and time.
*   **`// Only update secret if provided`**:
    *   **`if (data.secret) { ... }`**: Checks if a `secret` was provided in the request body.
    *   **`const { encrypted } = await encryptSecret(data.secret)`**: If a secret is provided, it's encrypted using the `encryptSecret` utility function.
    *   **`updateData.secret = encrypted`**: The encrypted secret is then added to the `updateData` object.
*   **`const updatedWebhooks = await db.update(workflowLogWebhook).set(updateData).where(eq(workflowLogWebhook.id, webhookId)).returning()`**:
    *   This is the Drizzle ORM command to perform the database update.
        *   **`db.update(workflowLogWebhook)`**: Specifies the table to update.
        *   **`.set(updateData)`**: Provides the data to update (the `updateData` object).
        *   **`.where(eq(workflowLogWebhook.id, webhookId))`**: Specifies the condition for *which* record to update – only the webhook with the matching `webhookId`.
        *   **`.returning()`**: Crucially, this tells Drizzle to return the *updated* rows. This is important for getting the final state of the webhook after the database operation.
*   **`if (updatedWebhooks.length === 0) { ... }`**:
    *   If `returning()` yields no results, it means the webhook wasn't found or updated (e.g., if it was deleted concurrently). Returns a `404 Not Found`.
*   **`const updatedWebhook = updatedWebhooks[0]`**:
    *   Takes the first (and only) updated webhook from the array.
*   **`logger.info('Webhook updated', { ... })`**:
    *   Logs a success message with relevant context (webhook ID, workflow ID, user ID).
*   **`return NextResponse.json({ data: { ... } })`**:
    *   Returns a successful `200 OK` response. The response body contains the updated webhook's details, including its ID, URL, configuration settings, and formatted `createdAt`/`updatedAt` timestamps.

---

#### **5. `DELETE` Function (Delete Workflow Log Webhook)**

This function handles `DELETE` requests to `/api/workflows/[id]/webhooks/[webhookId]`, allowing an authenticated and authorized user to delete an existing webhook.

```typescript
export async function DELETE(
  request: NextRequest,
  { params }: { params: Promise<{ id: string; webhookId: string }> }
) {
  try {
    const session = await getSession()
    if (!session?.user?.id) {
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
    }

    const { id: workflowId, webhookId } = await params
    const userId = session.user.id

    // Check if user has access to the workflow
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

    // Delete the webhook (will cascade delete deliveries)
    const deletedWebhook = await db
      .delete(workflowLogWebhook)
      .where(
        and(eq(workflowLogWebhook.id, webhookId), eq(workflowLogWebhook.workflowId, workflowId))
      )
      .returning()

    if (deletedWebhook.length === 0) {
      return NextResponse.json({ error: 'Webhook not found' }, { status: 404 })
    }

    logger.info('Webhook deleted', {
      webhookId,
      workflowId,
      userId,
    })

    return NextResponse.json({ success: true })
  } catch (error) {
    logger.error('Failed to delete webhook', { error })
    return NextResponse.json({ error: 'Failed to delete webhook' }, { status: 500 })
  }
}
```

*   **`export async function DELETE(...)`**:
    *   Defines an asynchronous function named `DELETE` for handling `HTTP DELETE` requests.
*   **`request: NextRequest, { params }: { params: Promise<{ id: string; webhookId: string }> }`**:
    *   Same request and parameters as the `PUT` function.
*   **`try { ... } catch (error) { ... }`**:
    *   Same error handling pattern as `PUT`.
*   **Authentication and Authorization (`getSession`, `hasAccess` check)**:
    *   The authentication and authorization logic (`getSession`, checking `session?.user?.id`, and querying `permissions` for workflow access) is identical to the `PUT` function, ensuring only authorized users can delete webhooks associated with workflows they have access to. If the user doesn't have access or the workflow isn't found, it returns a `401` or `404` respectively.
*   **`// Delete the webhook (will cascade delete deliveries)`**:
    *   **`const deletedWebhook = await db.delete(workflowLogWebhook)`**: Starts a Drizzle query to delete records from the `workflowLogWebhook` table.
    *   **`.where(and(eq(workflowLogWebhook.id, webhookId), eq(workflowLogWebhook.workflowId, workflowId)))`**: Specifies the condition for deletion: delete only the webhook matching both the `webhookId` and `workflowId`. This ensures a webhook from one workflow cannot be deleted via another workflow's ID.
    *   **`.returning()`**: Returns the deleted rows. This is used to confirm if a webhook was actually deleted.
*   **`if (deletedWebhook.length === 0) { ... }`**:
    *   If no webhook was deleted (meaning it didn't exist or didn't match the workflow ID), returns a `404 Not Found` response.
*   **`logger.info('Webhook deleted', { ... })`**:
    *   Logs a success message with context after a successful deletion.
*   **`return NextResponse.json({ success: true })`**:
    *   Returns a successful `200 OK` response with a simple `success: true` payload, indicating the webhook was deleted.

---

### **Simplified Complex Logic**

1.  **Access Control (`hasAccess` query)**: Instead of just checking if a user exists, the code performs a robust check: "Does this authenticated user have `workspace` level permission for the `workspace` that *owns* this specific `workflow`?" This prevents users from manipulating workflows they don't have rights to, even if they guess a valid workflow ID.
2.  **Data Validation (`UpdateWebhookSchema` with Zod)**: Instead of manually checking each field from the request body, Zod schema (`UpdateWebhookSchema`) is used to define the expected shape, types, and constraints (like `url` format or enum values for `levelFilter`). This centralizes validation logic and makes it clear what data is expected. `safeParse` ensures that validation errors are caught gracefully and returned to the client as a `400 Bad Request`.
3.  **Duplicate URL Check**: The logic for duplicate URLs is nuanced. It needs to check for other webhooks within the *same workflow* that use the same URL. Crucially, it must *exclude* the webhook currently being updated (`duplicateWebhook[0].id !== webhookId`) to avoid false positives, allowing a webhook to retain its current URL during an update.
4.  **Database Operations with Drizzle ORM**: Drizzle queries (`select`, `from`, `innerJoin`, `where`, `and`, `eq`, `update`, `delete`, `set`, `returning`) abstract away raw SQL, making database interactions more type-safe and readable.
    *   `returning()` is key as it provides the actual rows that were affected by an `UPDATE` or `DELETE` operation, allowing the API to confirm the operation and send back the latest state to the client.
5.  **Sensitive Data Handling (`encryptSecret`)**: The `secret` field for a webhook is conditionally encrypted using a dedicated utility (`encryptSecret`). This ensures that sensitive information is not stored in plain text in the database, enhancing security.

In summary, this file provides a secure and well-structured API for managing workflow log webhooks, incorporating best practices for authentication, authorization, data validation, and database interaction in a Next.js environment.