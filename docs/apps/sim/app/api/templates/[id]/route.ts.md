This file defines a Next.js API route that handles **CRUD (Create, Read, Update, Delete)** operations for individual template resources. Specifically, it allows clients to **GET** (retrieve), **PUT** (update), and **DELETE** (remove) a template based on its unique ID.

It acts as a backend endpoint, exposing functionality for managing "templates" within the application. It includes robust features like authentication, authorization (checking user permissions), data validation, logging, and proper error handling.

---

## **File Purpose and What It Does**

This TypeScript file (`/api/templates/[id]/route.ts` - indicated by the `[id]` dynamic segment in the path) creates a RESTful API endpoint in a Next.js application. It is designed to manage specific template entries in a database, identified by their unique `id`.

Here's a breakdown of its core responsibilities:

1.  **Retrieve a Template (`GET`):** When a `GET` request is made to `/api/templates/{id}`, it fetches the template data from the database. It also increments a "views" counter for that template and returns the template's details.
2.  **Update a Template (`PUT`):** When a `PUT` request is made to `/api/templates/{id}` with new data, it validates the incoming data, checks if the user has permission to update the template (either as the owner or an administrator), and then updates the corresponding template record in the database.
3.  **Delete a Template (`DELETE`):** When a `DELETE` request is made to `/api/templates/{id}`, it verifies user permissions (owner or administrator) and then removes the template record from the database.

Throughout these operations, it performs:
*   **Authentication:** Ensures the user is logged in.
*   **Authorization:** Checks if the logged-in user has the necessary permissions (e.g., owner of the template or admin of the associated workspace) to perform the requested action.
*   **Data Validation:** For `PUT` requests, it uses `zod` to validate the structure and content of the incoming data.
*   **Logging:** Uses a custom logger to record events, warnings, and errors for debugging and monitoring.
*   **Error Handling:** Catches potential errors during database operations or validation and returns appropriate HTTP status codes and messages.

---

## **Detailed Line-by-Line Explanation**

### **Imports**

```typescript
import { db } from '@sim/db'
import { templates, workflow } from '@sim/db/schema'
import { eq, sql } from 'drizzle-orm'
import { type NextRequest, NextResponse } from 'next/server'
import { z } from 'zod'
import { getSession } from '@/lib/auth'
import { createLogger } from '@/lib/logs/console/logger'
import { hasAdminPermission } from '@/lib/permissions/utils'
import { generateRequestId } from '@/lib/utils'
```

*   **`import { db } from '@sim/db'`**: Imports the database client instance, likely configured with Drizzle ORM, allowing interaction with the application's database. `@sim/db` suggests an internal alias for the database configuration.
*   **`import { templates, workflow } from '@sim/db/schema'`**: Imports the Drizzle ORM schema definitions for the `templates` and `workflow` tables. These objects represent the database tables and their columns, providing a type-safe way to build queries.
*   **`import { eq, sql } from 'drizzle-orm'`**: Imports specific utility functions from Drizzle ORM:
    *   `eq`: Used to create an equality condition (`WHERE column = value`) in database queries.
    *   `sql`: Allows embedding raw SQL expressions directly into Drizzle queries. This is particularly useful for operations like incrementing a counter atomically or using database-specific functions.
*   **`import { type NextRequest, NextResponse } from 'next/server'`**: Imports types and classes from Next.js for handling server-side API routes:
    *   `NextRequest`: Represents the incoming HTTP request, providing access to headers, body, URL, etc.
    *   `NextResponse`: Used to construct and send the HTTP response, allowing setting status codes, headers, and JSON bodies.
*   **`import { z } from 'zod'`**: Imports the `zod` library, a TypeScript-first schema declaration and validation library. It's used here to define and validate the shape of incoming JSON data for `PUT` requests.
*   **`import { getSession } from '@/lib/auth'`**: Imports a utility function `getSession` from an internal authentication library. This function is responsible for retrieving the current user's session data, typically including their ID and other authenticated information.
*   **`import { createLogger } from '@/lib/logs/console/logger'`**: Imports a custom logger utility. This helps in structured logging of events, warnings, and errors to the console or other logging destinations.
*   **`import { hasAdminPermission } from '@/lib/permissions/utils'`**: Imports a utility function to check if a given user has administrative permissions within a specific workspace. This is crucial for authorization checks.
*   **`import { generateRequestId } from '@/lib/utils'`**: Imports a utility function to generate a unique request ID. This ID is used for tracing individual requests through logs, making debugging easier.

### **Logger Initialization and Caching Control**

```typescript
const logger = createLogger('TemplateByIdAPI')

export const revalidate = 0
```

*   **`const logger = createLogger('TemplateByIdAPI')`**: Initializes a logger instance with the name `'TemplateByIdAPI'`. This allows log messages originating from this file to be easily identified (e.g., `[TemplateByIdAPI] ...`).
*   **`export const revalidate = 0`**: This is a Next.js-specific configuration for API routes. Setting `revalidate` to `0` explicitly disables caching for this API route. This means every request will execute the handler function directly, ensuring fresh data and consistent authorization checks.

### **`GET` Method: Retrieve a Template**

```typescript
// GET /api/templates/[id] - Retrieve a single template by ID
export async function GET(request: NextRequest, { params }: { params: Promise<{ id: string }> }) {
  const requestId = generateRequestId()
  const { id } = await params

  try {
    const session = await getSession()
    if (!session?.user?.id) {
      logger.warn(`[${requestId}] Unauthorized template access attempt for ID: ${id}`)
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
    }

    logger.debug(`[${requestId}] Fetching template: ${id}`)

    // Fetch the template by ID
    const result = await db.select().from(templates).where(eq(templates.id, id)).limit(1)

    if (result.length === 0) {
      logger.warn(`[${requestId}] Template not found: ${id}`)
      return NextResponse.json({ error: 'Template not found' }, { status: 404 })
    }

    const template = result[0]

    // Increment the view count
    try {
      await db
        .update(templates)
        .set({
          views: sql`${templates.views} + 1`,
          updatedAt: new Date(),
        })
        .where(eq(templates.id, id))

      logger.debug(`[${requestId}] Incremented view count for template: ${id}`)
    } catch (viewError) {
      // Log the error but don't fail the request
      logger.warn(`[${requestId}] Failed to increment view count for template: ${id}`, viewError)
    }

    logger.info(`[${requestId}] Successfully retrieved template: ${id}`)

    return NextResponse.json({
      data: {
        ...template,
        views: template.views + 1, // Return the incremented view count
      },
    })
  } catch (error: any) {
    logger.error(`[${requestId}] Error fetching template: ${id}`, error)
    return NextResponse.json({ error: 'Internal server error' }, { status: 500 })
  }
}
```

*   **`export async function GET(...)`**: Defines the `GET` handler function for the API route. Next.js automatically calls this function for incoming `GET` requests.
    *   `request: NextRequest`: The incoming HTTP request object.
    *   `{ params }: { params: Promise<{ id: string }> }`: Destructures the `params` object, which contains dynamic route segments. `[id]` means `params` will have an `id` property. It's wrapped in a `Promise` by Next.js, so it needs to be awaited.
*   **`const requestId = generateRequestId()`**: Generates a unique ID for this specific request.
*   **`const { id } = await params`**: Extracts the `id` from the awaited `params` object. This `id` is the template's unique identifier from the URL (e.g., `/api/templates/123` would give `id: "123"`).
*   **`try { ... } catch (error: any) { ... }`**: A `try...catch` block wraps the entire operation to gracefully handle any unexpected errors, preventing the server from crashing and returning a generic 500 error.
*   **`const session = await getSession()`**: Retrieves the current user's session.
*   **`if (!session?.user?.id) { ... }`**: **Authentication Check**. If no session exists or the user ID is missing, it means the user is not authenticated.
    *   `logger.warn(...)`: Logs a warning about unauthorized access.
    *   `return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })`: Returns a `401 Unauthorized` HTTP status code and a JSON error message.
*   **`logger.debug(`[${requestId}] Fetching template: ${id}`) `**: Logs a debug message indicating that a template fetch operation is starting.
*   **`const result = await db.select().from(templates).where(eq(templates.id, id)).limit(1)`**: This is the Drizzle ORM query to fetch a template:
    *   `db.select()`: Starts a select query. By default, selects all columns.
    *   `.from(templates)`: Specifies the `templates` table.
    *   `.where(eq(templates.id, id))`: Filters the results where the `id` column of the `templates` table is equal to the `id` extracted from the request parameters.
    *   `.limit(1)`: Ensures only one record is returned, as `id` is expected to be unique.
*   **`if (result.length === 0) { ... }`**: **Template Not Found Check**. If the query returns no results, the template with the given ID doesn't exist.
    *   `logger.warn(...)`: Logs a warning.
    *   `return NextResponse.json({ error: 'Template not found' }, { status: 404 })`: Returns a `404 Not Found` HTTP status code.
*   **`const template = result[0]`**: If a template is found, it's the first (and only) element in the `result` array.
*   **`try { ... } catch (viewError) { ... }` (Increment View Count)**: This nested `try...catch` block handles incrementing the template's view count. It's nested because failure to increment views should not prevent the template from being returned.
    *   **`await db.update(templates).set({ views: sql`${templates.views} + 1`, updatedAt: new Date(), }).where(eq(templates.id, id))`**: Updates the `templates` table:
        *   `db.update(templates)`: Specifies the table to update.
        *   `.set(...)`: Defines the columns to update.
            *   `views: sql`${templates.views} + 1``: **Crucial for atomic updates.** Uses Drizzle's `sql` function to execute a raw SQL expression directly on the database server. This `views = views + 1` operation is atomic, meaning it safely increments the counter even if multiple requests try to do so simultaneously, preventing race conditions.
            *   `updatedAt: new Date()`: Updates the `updatedAt` timestamp to the current time.
        *   `.where(eq(templates.id, id))`: Ensures only the specific template is updated.
    *   `logger.debug(...)`: Logs a message upon successful view count increment.
    *   `catch (viewError) { logger.warn(...) }`: If the view count update fails (e.g., database error), it logs a warning but allows the main request to continue, returning the template data.
*   **`logger.info(`[${requestId}] Successfully retrieved template: ${id}`) `**: Logs an informational message about successful retrieval.
*   **`return NextResponse.json({ data: { ...template, views: template.views + 1, }, })`**: Returns a `200 OK` HTTP status code with the template data.
    *   `...template`: Spreads all properties of the fetched `template` object.
    *   `views: template.views + 1`: Explicitly sets the `views` property in the response to the *new, incremented* value, even though the database update might be slightly asynchronous or might have failed without impacting the main response.
*   **`catch (error: any) { ... }`**: The outer `catch` block.
    *   `logger.error(...)`: Logs the general error.
    *   `return NextResponse.json({ error: 'Internal server error' }, { status: 500 })`: Returns a `500 Internal Server Error` for any unhandled exceptions.

### **`updateTemplateSchema` (Zod Schema)**

```typescript
const updateTemplateSchema = z.object({
  name: z.string().min(1).max(100),
  description: z.string().min(1).max(500),
  author: z.string().min(1).max(100),
  category: z.string().min(1),
  icon: z.string().min(1),
  color: z.string().regex(/^#[0-9A-F]{6}$/i),
  state: z.any().optional(), // Workflow state
})
```

*   **`const updateTemplateSchema = z.object({ ... })`**: Defines a Zod schema named `updateTemplateSchema`. This schema specifies the expected structure and validation rules for the data sent in the `PUT` request body when updating a template.
    *   `z.object({ ... })`: Indicates that the incoming data should be a JavaScript object.
    *   `name: z.string().min(1).max(100)`: The `name` property must be a string, with a minimum length of 1 character and a maximum of 100 characters.
    *   `description: z.string().min(1).max(500)`: Similar validation for `description`.
    *   `author: z.string().min(1).max(100)`: Similar validation for `author`.
    *   `category: z.string().min(1)`: Similar validation for `category`.
    *   `icon: z.string().min(1)`: Similar validation for `icon`.
    *   `color: z.string().regex(/^#[0-9A-F]{6}$/i)`: The `color` property must be a string that matches a hexadecimal color code pattern (e.g., `#RRGGBB`). `i` at the end makes the regex case-insensitive.
    *   `state: z.any().optional()`: The `state` property can be any type (`z.any()`) and is optional (`.optional()`), meaning it doesn't have to be present in the request body. This likely represents some flexible workflow state.

### **`PUT` Method: Update a Template**

```typescript
// PUT /api/templates/[id] - Update a template
export async function PUT(request: NextRequest, { params }: { params: Promise<{ id: string }> }) {
  const requestId = generateRequestId()
  const { id } = await params

  try {
    const session = await getSession()
    if (!session?.user?.id) {
      logger.warn(`[${requestId}] Unauthorized template update attempt for ID: ${id}`)
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
    }

    const body = await request.json()
    const validationResult = updateTemplateSchema.safeParse(body)

    if (!validationResult.success) {
      logger.warn(`[${requestId}] Invalid template data for update: ${id}`, validationResult.error)
      return NextResponse.json(
        { error: 'Invalid template data', details: validationResult.error.errors },
        { status: 400 }
      )
    }

    const { name, description, author, category, icon, color, state } = validationResult.data

    // Check if template exists
    const existingTemplate = await db.select().from(templates).where(eq(templates.id, id)).limit(1)

    if (existingTemplate.length === 0) {
      logger.warn(`[${requestId}] Template not found for update: ${id}`)
      return NextResponse.json({ error: 'Template not found' }, { status: 404 })
    }

    // Permission: template owner OR admin of the workflow's workspace (if any)
    let canUpdate = existingTemplate[0].userId === session.user.id

    if (!canUpdate && existingTemplate[0].workflowId) {
      const wfRows = await db
        .select({ workspaceId: workflow.workspaceId })
        .from(workflow)
        .where(eq(workflow.id, existingTemplate[0].workflowId))
        .limit(1)

      const workspaceId = wfRows[0]?.workspaceId as string | null | undefined
      if (workspaceId) {
        const hasAdmin = await hasAdminPermission(session.user.id, workspaceId)
        if (hasAdmin) canUpdate = true
      }
    }

    if (!canUpdate) {
      logger.warn(`[${requestId}] User denied permission to update template ${id}`)
      return NextResponse.json({ error: 'Access denied' }, { status: 403 })
    }

    // Update the template
    const updatedTemplate = await db
      .update(templates)
      .set({
        name,
        description,
        author,
        category,
        icon,
        color,
        ...(state && { state }), // Conditionally add state if it exists
        updatedAt: new Date(),
      })
      .where(eq(templates.id, id))
      .returning()

    logger.info(`[${requestId}] Successfully updated template: ${id}`)

    return NextResponse.json({
      data: updatedTemplate[0],
      message: 'Template updated successfully',
    })
  } catch (error: any) {
    logger.error(`[${requestId}] Error updating template: ${id}`, error)
    return NextResponse.json({ error: 'Internal server error' }, { status: 500 })
  }
}
```

*   **`export async function PUT(...)`**: Defines the `PUT` handler, similar to `GET`.
*   **`const requestId = generateRequestId()` & `const { id } = await params`**: Same as `GET` for request ID and template ID extraction.
*   **Authentication Check**: Identical to the `GET` method.
*   **`const body = await request.json()`**: Parses the incoming JSON request body.
*   **`const validationResult = updateTemplateSchema.safeParse(body)`**: **Data Validation**. Attempts to validate the `body` against the `updateTemplateSchema` defined earlier. `safeParse` is non-throwing, returning an object indicating success or failure.
*   **`if (!validationResult.success) { ... }`**: If validation fails:
    *   `logger.warn(...)`: Logs a warning with validation errors.
    *   `return NextResponse.json({ error: 'Invalid template data', details: validationResult.error.errors }, { status: 400 })`: Returns a `400 Bad Request` with details about the validation errors.
*   **`const { name, description, author, category, icon, color, state } = validationResult.data`**: If validation is successful, extracts the validated data.
*   **Template Existence Check**: Identical to `GET`, ensuring the template to be updated actually exists. Returns `404 Not Found` if not.
*   **`let canUpdate = existingTemplate[0].userId === session.user.id`**: **Authorization Check (Owner)**. Initializes `canUpdate` to `true` if the current user (`session.user.id`) is the owner of the template (`existingTemplate[0].userId`).
*   **`if (!canUpdate && existingTemplate[0].workflowId) { ... }`**: **Authorization Check (Admin)**. If the user is not the owner (`!canUpdate`) AND the template is associated with a workflow (`existingTemplate[0].workflowId` exists):
    *   `const wfRows = await db.select({ workspaceId: workflow.workspaceId }).from(workflow).where(eq(workflow.id, existingTemplate[0].workflowId)).limit(1)`: Queries the `workflow` table to find the `workspaceId` associated with the template's workflow.
    *   `const workspaceId = wfRows[0]?.workspaceId as string | null | undefined`: Extracts the `workspaceId`.
    *   `if (workspaceId) { const hasAdmin = await hasAdminPermission(session.user.id, workspaceId); if (hasAdmin) canUpdate = true }`: If a `workspaceId` is found, calls `hasAdminPermission` to check if the current user is an admin of that workspace. If they are, `canUpdate` is set to `true`.
*   **`if (!canUpdate) { ... }`**: If, after both checks, `canUpdate` is still `false`:
    *   `logger.warn(...)`: Logs a warning about permission denial.
    *   `return NextResponse.json({ error: 'Access denied' }, { status: 403 })`: Returns a `403 Forbidden` HTTP status code.
*   **`const updatedTemplate = await db.update(templates).set({ ... }).where(eq(templates.id, id)).returning()`**: **Update Operation**.
    *   `db.update(templates)`: Specifies the table to update.
    *   `.set({ ... })`: Sets the new values for the template's properties using the validated data.
        *   `...(state && { state })`: This is a conditional spread syntax. It means "if `state` exists (is truthy), then add the `{ state: state }` property to the update object; otherwise, do nothing." This ensures optional fields are only updated if provided.
        *   `updatedAt: new Date()`: Updates the timestamp.
    *   `.where(eq(templates.id, id))`: Updates only the specific template.
    *   `.returning()`: **Important for Drizzle.** This makes the `update` query return the *updated* rows. In this case, it will return the single updated template object.
*   **`logger.info(...)`**: Logs a success message.
*   **`return NextResponse.json({ data: updatedTemplate[0], message: 'Template updated successfully', })`**: Returns a `200 OK` with the first updated template object and a success message.
*   **`catch (error: any) { ... }`**: Generic error handling, similar to `GET`.

### **`DELETE` Method: Delete a Template**

```typescript
// DELETE /api/templates/[id] - Delete a template
export async function DELETE(
  request: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  const requestId = generateRequestId()
  const { id } = await params

  try {
    const session = await getSession()
    if (!session?.user?.id) {
      logger.warn(`[${requestId}] Unauthorized template delete attempt for ID: ${id}`)
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
    }

    // Fetch template
    const existing = await db.select().from(templates).where(eq(templates.id, id)).limit(1)
    if (existing.length === 0) {
      logger.warn(`[${requestId}] Template not found for delete: ${id}`)
      return NextResponse.json({ error: 'Template not found' }, { status: 404 })
    }

    const template = existing[0]

    // Permission: owner or admin of the workflow's workspace (if any)
    let canDelete = template.userId === session.user.id

    if (!canDelete && template.workflowId) {
      // Look up workflow to get workspaceId
      const wfRows = await db
        .select({ workspaceId: workflow.workspaceId })
        .from(workflow)
        .where(eq(workflow.id, template.workflowId))
        .limit(1)

      const workspaceId = wfRows[0]?.workspaceId as string | null | undefined
      if (workspaceId) {
        const hasAdmin = await hasAdminPermission(session.user.id, workspaceId)
        if (hasAdmin) canDelete = true
      }
    }

    if (!canDelete) {
      logger.warn(`[${requestId}] User denied permission to delete template ${id}`)
      return NextResponse.json({ error: 'Access denied' }, { status: 403 })
    }

    await db.delete(templates).where(eq(templates.id, id))

    logger.info(`[${requestId}] Deleted template: ${id}`)
    return NextResponse.json({ success: true })
  } catch (error: any) {
    logger.error(`[${requestId}] Error deleting template: ${id}`, error)
    return NextResponse.json({ error: 'Internal server error' }, { status: 500 })
  }
}
```

*   **`export async function DELETE(...)`**: Defines the `DELETE` handler.
*   **`const requestId = generateRequestId()` & `const { id } = await params`**: Same as `GET` and `PUT`.
*   **Authentication Check**: Identical to previous methods.
*   **Template Existence Check**: Fetches the template first to ensure it exists before attempting to delete it. Returns `404 Not Found` if not.
*   **Authorization Check**: This logic is **identical to the `PUT` method**. It verifies that the current user is either the owner of the template or an administrator of the associated workflow's workspace. Returns `403 Forbidden` if permission is denied.
*   **`await db.delete(templates).where(eq(templates.id, id))`**: **Delete Operation**.
    *   `db.delete(templates)`: Specifies the `templates` table for deletion.
    *   `.where(eq(templates.id, id))`: Deletes only the specific template matching the `id`.
*   **`logger.info(...)`**: Logs a success message.
*   **`return NextResponse.json({ success: true })`**: Returns a `200 OK` with a success indicator.
*   **`catch (error: any) { ... }`**: Generic error handling, similar to `GET` and `PUT`.

---

This file demonstrates a well-structured and secure approach to building a RESTful API endpoint in Next.js, incorporating essential features like authentication, granular authorization, data validation, and comprehensive logging.