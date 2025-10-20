This TypeScript file defines a Next.js API route that handles CRUD (Create, Read, Update, Delete) operations for "custom tools." These tools are likely user-defined functions or integrations that an application can use, stored in a database.

The file serves as the backend logic for managing these custom tools, providing endpoints for:
1.  **Fetching** all custom tools belonging to a user (`GET`).
2.  **Creating new** custom tools or **updating existing** ones (`POST`).
3.  **Deleting** a specific custom tool (`DELETE`).

It incorporates robust authentication, data validation using Zod, and transactional database operations to ensure data integrity.

---

### Key Concepts and Libraries Used

Before diving into the code, let's understand the main components:

*   **Next.js API Routes:** This file exports `GET`, `POST`, and `DELETE` asynchronous functions, which Next.js automatically maps to `/api/custom-tools` (assuming this file is at `app/api/custom-tools/route.ts` or similar). These functions receive a `NextRequest` object and return a `NextResponse` object.
*   **Drizzle ORM:** A type-safe ORM (Object-Relational Mapper) for TypeScript. It's used to interact with the database (`db`), define database schemas (`customTools`), and build queries (`eq`, `select`, `insert`, `update`, `delete`, `transaction`).
*   **Zod:** A TypeScript-first schema declaration and validation library. It's used to define the expected structure and types of incoming data for API requests, ensuring that the data conforms to the application's requirements.
*   **Authentication (`getSession`, `getUserId`):** Custom utilities for managing user sessions and retrieving user IDs, ensuring that only authenticated and authorized users can access or modify their tools.
*   **Logging (`createLogger`, `generateRequestId`):** Custom utilities for structured logging, including a unique request ID for tracing operations.

---

### Detailed Code Explanation

Let's break down the file line by line.

#### 1. Imports

```typescript
import { db } from '@sim/db'
import { customTools } from '@sim/db/schema'
import { eq } from 'drizzle-orm'
import { type NextRequest, NextResponse } from 'next/server'
import { z } from 'zod'
import { getSession } from '@/lib/auth'
import { createLogger } from '@/lib/logs/console/logger'
import { generateRequestId } from '@/lib/utils'
import { getUserId } from '@/app/api/auth/oauth/utils'
```

*   `import { db } from '@sim/db'`: Imports the Drizzle ORM database client instance. This `db` object is how the application interacts with the underlying database (e.g., PostgreSQL, MySQL).
*   `import { customTools } from '@sim/db/schema'`: Imports the Drizzle schema definition for the `customTools` table. This defines the structure of the `custom_tools` table in the database, including its columns and their types.
*   `import { eq } from 'drizzle-orm'`: Imports the `eq` (equals) function from Drizzle ORM. This is a helper for building `WHERE` clauses in database queries, specifically for checking if a column's value is equal to a given value.
*   `import { type NextRequest, NextResponse } from 'next/server'`: Imports `NextRequest` and `NextResponse` from Next.js.
    *   `NextRequest`: Represents the incoming HTTP request object, providing access to headers, body, URL, etc.
    *   `NextResponse`: Used to construct and send HTTP responses, allowing control over status codes, headers, and body.
*   `import { z } from 'zod'`: Imports the `z` object from the Zod library, which is the primary interface for defining schemas.
*   `import { getSession } from '@/lib/auth'`: Imports a custom utility function `getSession`. This function is responsible for retrieving the current user's session information, typically from a cookie or authentication token.
*   `import { createLogger } from '@/lib/logs/console/logger'`: Imports a custom utility to create a logger instance. This allows for structured and configurable logging throughout the API route.
*   `import { generateRequestId } from '@/lib/utils'`: Imports a custom utility to generate a unique request ID. This ID is used for tracing individual requests through the logs, making debugging easier.
*   `import { getUserId } from '@/app/api/auth/oauth/utils'`: Imports another custom utility function `getUserId`. This function is used to retrieve a user's ID, specifically in scenarios where a `workflowId` (e.g., from an OAuth context or an automated workflow) is provided instead of a direct user session.

#### 2. Logger Initialization

```typescript
const logger = createLogger('CustomToolsAPI')
```

*   `const logger = createLogger('CustomToolsAPI')`: Initializes a logger instance named `CustomToolsAPI`. This logger will be used to record information, warnings, and errors related to operations within this API route.

#### 3. Zod Schema Definition for Custom Tools

```typescript
const CustomToolSchema = z.object({
  tools: z.array(
    z.object({
      id: z.string().optional(),
      title: z.string().min(1, 'Tool title is required'),
      schema: z.object({
        type: z.literal('function'),
        function: z.object({
          name: z.string().min(1, 'Function name is required'),
          description: z.string().optional(),
          parameters: z.object({
            type: z.string(),
            properties: z.record(z.any()),
            required: z.array(z.string()).optional(),
          }),
        }),
      }),
      code: z.string(),
    })
  ),
})
```

This `CustomToolSchema` defines the expected structure of the data that will be sent to the `POST` endpoint. It uses Zod to enforce types and validation rules.

*   `const CustomToolSchema = z.object({ ... })`: Defines the top-level structure as an object.
*   `tools: z.array(...)`: Specifies that the `body` of the request must contain a `tools` property, which is an array.
*   `z.object({ ... })` (inner): Defines the structure of each item within the `tools` array (i.e., each individual custom tool).
    *   `id: z.string().optional()`: An optional string field for the tool's ID. It's optional because new tools won't have an ID yet, but existing ones will.
    *   `title: z.string().min(1, 'Tool title is required')`: A required string field for the tool's title. It must have at least one character.
    *   `schema: z.object({ ... })`: A required object field representing the tool's schema, often used for defining how the tool can be called or its inputs/outputs.
        *   `type: z.literal('function')`: The `type` property within the schema must literally be the string `'function'`. This suggests these custom tools are designed to represent callable functions.
        *   `function: z.object({ ... })`: A nested object containing details about the function.
            *   `name: z.string().min(1, 'Function name is required')`: The name of the function, required and must have at least one character.
            *   `description: z.string().optional()`: An optional description for the function.
            *   `parameters: z.object({ ... })`: An object describing the function's parameters, likely following a JSON Schema-like structure.
                *   `type: z.string()`: The overall type of the parameters object (e.g., `"object"`).
                *   `properties: z.record(z.any())`: An object where keys are parameter names (strings) and values are their definitions (`z.any()` means any type, allowing for flexible schema definitions for parameters).
                *   `required: z.array(z.string()).optional()`: An optional array of strings, listing the names of parameters that are required.
    *   `code: z.string()`: A required string field that will hold the actual code implementation for the custom tool.

#### 4. `GET` Endpoint - Fetch Custom Tools

This function handles `GET` requests to retrieve all custom tools for a specific user.

```typescript
export async function GET(request: NextRequest) {
  const requestId = generateRequestId()
  const searchParams = request.nextUrl.searchParams
  const workflowId = searchParams.get('workflowId')

  try {
    let userId: string | undefined

    // If workflowId is provided, get userId from the workflow
    if (workflowId) {
      userId = await getUserId(requestId, workflowId)

      if (!userId) {
        logger.warn(`[${requestId}] No valid user found for workflow: ${workflowId}`)
        return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
      }
    } else {
      // Otherwise use session-based auth (for client-side)
      const session = await getSession()
      if (!session?.user?.id) {
        logger.warn(`[${requestId}] Unauthorized custom tools access attempt`)
        return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
      }
      userId = session.user.id
    }

    const result = await db.select().from(customTools).where(eq(customTools.userId, userId))

    return NextResponse.json({ data: result }, { status: 200 })
  } catch (error) {
    logger.error(`[${requestId}] Error fetching custom tools:`, error)
    return NextResponse.json({ error: 'Failed to fetch custom tools' }, { status: 500 })
  }
}
```

*   `export async function GET(request: NextRequest)`: Declares an asynchronous function named `GET`, which is the entry point for HTTP GET requests. It receives the `NextRequest` object.
*   `const requestId = generateRequestId()`: Generates a unique ID for this specific request for logging purposes.
*   `const searchParams = request.nextUrl.searchParams`: Extracts the URL's query parameters (e.g., `?key=value`) from the request.
*   `const workflowId = searchParams.get('workflowId')`: Tries to retrieve a `workflowId` from the query parameters. This is part of an alternative authentication strategy.
*   `try { ... } catch (error) { ... }`: A `try...catch` block to gracefully handle any errors that occur during the request processing.
*   `let userId: string | undefined`: Declares a variable `userId` to store the ID of the user whose tools are being fetched. It can be a string or `undefined`.

*   **Authentication Logic (Simplified):** This block determines the `userId` using one of two methods:
    *   `if (workflowId) { ... }`: If a `workflowId` is present in the URL query, it attempts to fetch the `userId` associated with that workflow using `getUserId()`. This is useful for backend processes or integrations where a traditional user session might not be available.
        *   `userId = await getUserId(requestId, workflowId)`: Calls the utility to get `userId`.
        *   `if (!userId)`: If no user is found for the given workflow ID, it logs a warning and returns an "Unauthorized" (401) response.
    *   `else { ... }`: If no `workflowId` is provided, it falls back to standard session-based authentication. This is typical for requests originating from a client-side user interface.
        *   `const session = await getSession()`: Retrieves the current user session.
        *   `if (!session?.user?.id)`: If no active session or user ID in the session is found, it logs a warning and returns an "Unauthorized" (401) response.
        *   `userId = session.user.id`: Assigns the authenticated user's ID to `userId`.

*   `const result = await db.select().from(customTools).where(eq(customTools.userId, userId))`: This is the core database query.
    *   `db.select()`: Starts a Drizzle query to select data. By default, it selects all columns.
    *   `.from(customTools)`: Specifies that the query is targeting the `customTools` table.
    *   `.where(eq(customTools.userId, userId))`: Filters the results to include only those custom tools where the `userId` column matches the `userId` obtained from authentication.
*   `return NextResponse.json({ data: result }, { status: 200 })`: If the query is successful, it returns a JSON response containing the fetched custom tools in a `data` property, with an HTTP status code of 200 (OK).
*   `catch (error) { ... }`: If any error occurs within the `try` block:
    *   `logger.error(...)`: Logs the error with the `requestId`.
    *   `return NextResponse.json({ error: 'Failed to fetch custom tools' }, { status: 500 })`: Returns a JSON response with an error message and an HTTP status code of 500 (Internal Server Error).

#### 5. `POST` Endpoint - Create or Update Custom Tools

This function handles `POST` requests to create new custom tools or update existing ones for the authenticated user. It supports batch operations.

```typescript
export async function POST(req: NextRequest) {
  const requestId = generateRequestId()

  try {
    const session = await getSession()
    if (!session?.user?.id) {
      logger.warn(`[${requestId}] Unauthorized custom tools update attempt`)
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
    }

    const body = await req.json()

    try {
      // Validate the request body
      const { tools } = CustomToolSchema.parse(body)

      // Use a transaction for multi-step database operations
      return await db.transaction(async (tx) => {
        // Process each tool: either update existing or create new
        for (const tool of tools) {
          const nowTime = new Date()

          if (tool.id) {
            // First check if this tool belongs to the user
            const existingTool = await tx
              .select()
              .from(customTools)
              .where(eq(customTools.id, tool.id))
              .limit(1)

            if (existingTool.length === 0) {
              // Tool doesn't exist, create it (this case allows creating a tool with a pre-defined ID)
              await tx.insert(customTools).values({
                id: tool.id,
                userId: session.user.id,
                title: tool.title,
                schema: tool.schema,
                code: tool.code,
                createdAt: nowTime,
                updatedAt: nowTime,
              })
            } else if (existingTool[0].userId === session.user.id) {
              // Tool exists and belongs to user, update it
              await tx
                .update(customTools)
                .set({
                  title: tool.title,
                  schema: tool.schema,
                  code: tool.code,
                  updatedAt: nowTime,
                })
                .where(eq(customTools.id, tool.id))
            } else {
              // Log and silently continue if user attempts to update a tool they don't own
              logger.warn(
                `[${requestId}] Silent continuation on unauthorized tool update attempt: ${tool.id}`
              )
            }
          } else {
            // No ID provided, create a new tool with a generated ID
            await tx.insert(customTools).values({
              id: crypto.randomUUID(), // Generates a new unique ID
              userId: session.user.id,
              title: tool.title,
              schema: tool.schema,
              code: tool.code,
              createdAt: nowTime,
              updatedAt: nowTime,
            })
          }
        }

        return NextResponse.json({ success: true })
      })
    } catch (validationError) {
      if (validationError instanceof z.ZodError) {
        logger.warn(`[${requestId}] Invalid custom tools data`, {
          errors: validationError.errors,
        })
        return NextResponse.json(
          { error: 'Invalid request data', details: validationError.errors },
          { status: 400 }
        )
      }
      throw validationError
    }
  } catch (error) {
    logger.error(`[${requestId}] Error updating custom tools`, error)
    return NextResponse.json({ error: 'Failed to update custom tools' }, { status: 500 })
  }
}
```

*   `export async function POST(req: NextRequest)`: Declares an asynchronous function named `POST`, the entry point for HTTP POST requests.
*   `const requestId = generateRequestId()`: Generates a request ID.
*   `try { ... } catch (error) { ... }`: The outer `try...catch` handles general server errors (e.g., database connection issues).
*   **Authentication:**
    *   `const session = await getSession()`: Retrieves the user session.
    *   `if (!session?.user?.id)`: Checks for an active session and user ID. If not found, logs a warning and returns "Unauthorized" (401). Unlike `GET`, there's no `workflowId` alternative here; `POST` operations typically require an actively logged-in user.
*   `const body = await req.json()`: Parses the JSON body of the incoming request.
*   `try { ... } catch (validationError) { ... }`: An inner `try...catch` specifically to handle validation errors from Zod.
    *   `const { tools } = CustomToolSchema.parse(body)`: **Crucial validation step.** It attempts to parse and validate the `body` against the `CustomToolSchema` defined earlier.
        *   If the `body` matches the schema, the validated `tools` array is extracted.
        *   If validation fails, a `z.ZodError` is thrown, which is caught by the inner `catch` block.
    *   `return await db.transaction(async (tx) => { ... })`: **Simplified Complex Logic: Database Transaction.** This block ensures that a series of database operations (creating or updating multiple tools) are treated as a single, atomic unit.
        *   `tx` is a transaction object that allows performing operations within the transaction.
        *   **Purpose of Transaction:** If any database operation inside this `async` function fails (e.g., due to a constraint violation or a server error), the entire transaction is rolled back, meaning none of the changes are saved to the database. This prevents partial updates and maintains data consistency.
    *   `for (const tool of tools) { ... }`: The code iterates through each `tool` object in the validated `tools` array.
        *   `const nowTime = new Date()`: Gets the current timestamp, used for `createdAt` and `updatedAt` fields.
        *   `if (tool.id) { ... }`: **Logic for existing tools (update or re-create if ID supplied but not found/owned).**
            *   `const existingTool = await tx.select().from(customTools).where(eq(customTools.id, tool.id)).limit(1)`: Queries the database (within the transaction `tx`) to see if a tool with the provided `tool.id` already exists.
            *   `if (existingTool.length === 0)`: If no tool with that `id` is found, it means the client provided an ID for a tool that doesn't exist. The code proceeds to `insert` it.
                *   `await tx.insert(customTools).values(...)`: Inserts a new record into the `customTools` table using the provided `tool.id`, the authenticated `userId`, and other validated data.
            *   `else if (existingTool[0].userId === session.user.id)`: If the tool *does* exist, it further checks if the `userId` of the existing tool matches the authenticated `session.user.id`. This is a crucial ownership check.
                *   `await tx.update(customTools).set(...).where(eq(customTools.id, tool.id))`: If the user owns the tool, it updates the `title`, `schema`, `code`, and `updatedAt` fields for that specific tool.
            *   `else { ... }`: If the tool exists but its `userId` does *not* match the session `userId` (meaning the user is trying to modify a tool they don't own).
                *   `logger.warn(...)`: It logs a warning about an unauthorized update attempt.
                *   **"Silent continuation":** Instead of returning an error, the code `continues` to the next tool in the loop. This design choice means that a malicious user attempting to update other users' tools won't cause the entire batch operation to fail for their *own* legitimate tools, but their unauthorized attempt is logged.
        *   `else { ... }`: **Logic for new tools (no ID provided).**
            *   `await tx.insert(customTools).values(...)`: Inserts a new record into the `customTools` table.
            *   `id: crypto.randomUUID()`: Generates a new, globally unique ID for this new tool, as none was provided.
            *   The other fields are populated from the validated `tool` object and `session.user.id`.
    *   `return NextResponse.json({ success: true })`: If all tool creations/updates within the transaction are successful, it returns a success response.
*   `catch (validationError) { ... }`: This block handles errors thrown by `CustomToolSchema.parse(body)`.
    *   `if (validationError instanceof z.ZodError)`: Checks if the error is specifically a Zod validation error.
        *   `logger.warn(...)`: Logs the validation error details.
        *   `return NextResponse.json({ error: 'Invalid request data', details: validationError.errors }, { status: 400 })`: Returns a JSON response indicating "Invalid request data" with detailed Zod errors and an HTTP status code of 400 (Bad Request).
    *   `throw validationError`: If it's not a Zod error, re-throws it to be caught by the outer `try...catch` (general server error).
*   `catch (error) { ... }`: The outer `catch` block for general errors during the POST request.
    *   `logger.error(...)`: Logs the error.
    *   `return NextResponse.json({ error: 'Failed to update custom tools' }, { status: 500 })`: Returns a 500 status with a generic error message.

#### 6. `DELETE` Endpoint - Delete a Custom Tool

This function handles `DELETE` requests to remove a specific custom tool by its ID.

```typescript
export async function DELETE(request: NextRequest) {
  const requestId = generateRequestId()
  const searchParams = request.nextUrl.searchParams
  const toolId = searchParams.get('id')

  if (!toolId) {
    logger.warn(`[${requestId}] Missing tool ID for deletion`)
    return NextResponse.json({ error: 'Tool ID is required' }, { status: 400 })
  }

  try {
    const session = await getSession()
    if (!session?.user?.id) {
      logger.warn(`[${requestId}] Unauthorized custom tool deletion attempt`)
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
    }

    // Check if the tool exists and belongs to the user
    const existingTool = await db
      .select()
      .from(customTools)
      .where(eq(customTools.id, toolId))
      .limit(1)

    if (existingTool.length === 0) {
      logger.warn(`[${requestId}] Tool not found: ${toolId}`)
      return NextResponse.json({ error: 'Tool not found' }, { status: 404 })
    }

    if (existingTool[0].userId !== session.user.id) {
      logger.warn(`[${requestId}] User attempted to delete a tool they don't own: ${toolId}`)
      return NextResponse.json({ error: 'Unauthorized' }, { status: 403 })
    }

    // Delete the tool
    await db.delete(customTools).where(eq(customTools.id, toolId))

    logger.info(`[${requestId}] Deleted tool: ${toolId}`)
    return NextResponse.json({ success: true })
  } catch (error) {
    logger.error(`[${requestId}] Error deleting custom tool:`, error)
    return NextResponse.json({ error: 'Failed to delete custom tool' }, { status: 500 })
  }
}
```

*   `export async function DELETE(request: NextRequest)`: Declares an asynchronous function named `DELETE` for HTTP DELETE requests.
*   `const requestId = generateRequestId()`: Generates a request ID.
*   `const searchParams = request.nextUrl.searchParams`: Gets URL query parameters.
*   `const toolId = searchParams.get('id')`: Extracts the `id` of the tool to be deleted from the query parameters.
*   `if (!toolId) { ... }`: Basic validation: if `toolId` is missing, logs a warning and returns a 400 (Bad Request) error.
*   `try { ... } catch (error) { ... }`: `try...catch` block for error handling.
*   **Authentication:**
    *   `const session = await getSession()`: Retrieves the user session.
    *   `if (!session?.user?.id)`: Checks for an active session and user ID. If not found, logs a warning and returns "Unauthorized" (401).
*   **Ownership Check (Simplified):** This is a two-step process to ensure the user can only delete *their own* tools.
    *   `const existingTool = await db.select().from(customTools).where(eq(customTools.id, toolId)).limit(1)`: Queries the database to find the tool with the given `toolId`. `limit(1)` optimizes the query since we only need to know if it exists.
    *   `if (existingTool.length === 0)`: If no tool is found with that ID, it logs a warning and returns a 404 (Not Found) response.
    *   `if (existingTool[0].userId !== session.user.id)`: If the tool *is* found, it then checks if the `userId` of that tool matches the authenticated user's `userId`. If they don't match, it means the user is trying to delete someone else's tool. It logs a warning and returns a 403 (Forbidden) response.
*   `await db.delete(customTools).where(eq(customTools.id, toolId))`: If all checks pass, it proceeds to delete the tool from the `customTools` table where the `id` matches `toolId`.
*   `logger.info(...)`: Logs a successful deletion.
*   `return NextResponse.json({ success: true })`: Returns a success response.
*   `catch (error) { ... }`: Handles any general errors during deletion.
    *   `logger.error(...)`: Logs the error.
    *   `return NextResponse.json({ error: 'Failed to delete custom tool' }, { status: 500 })`: Returns a 500 status with a generic error message.

---

### Summary

This file effectively creates a robust API for managing custom tools. It meticulously handles:

*   **Authentication and Authorization:** Ensures only legitimate and authorized users can interact with their own tools, supporting both session-based and workflow-based authentication for reading tools.
*   **Data Validation:** Uses Zod to strictly validate incoming data for `POST` requests, preventing malformed or incomplete data from reaching the database.
*   **Database Interactions:** Leverages Drizzle ORM for type-safe and efficient database operations, including transactions for atomic updates.
*   **Error Handling and Logging:** Provides comprehensive error handling with appropriate HTTP status codes and detailed logging with request IDs for easier debugging and monitoring.
*   **Idempotency and Consistency:** The `POST` endpoint, through its transaction and logic, handles both new tool creation and existing tool updates gracefully, maintaining data consistency. The "silent continuation" on unauthorized updates is a specific design choice to prevent a single unauthorized tool from failing a batch operation.