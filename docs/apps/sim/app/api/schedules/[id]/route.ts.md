This TypeScript file defines API routes for managing workflow schedules within a Next.js application. Specifically, it handles two HTTP methods: `DELETE` for removing an existing schedule and `PUT` for updating a schedule's status (e.g., reactivating or disabling it).

It integrates with a Drizzle ORM database, uses Next.js server utilities, and includes robust authentication, authorization, and logging mechanisms.

---

## Detailed Explanation

### Imports

The first section of the file brings in various modules and utilities needed for its functionality.

*   `import { db } from '@sim/db'`
    *   **Purpose:** Imports the initialized Drizzle ORM database client. This `db` object is the primary interface for interacting with the database.
    *   **Line-by-Line:** `db` is an object representing our database connection, allowing us to perform queries like `select`, `insert`, `update`, and `delete`.
*   `import { workflow, workflowSchedule } from '@sim/db/schema'`
    *   **Purpose:** Imports the Drizzle ORM schema definitions for the `workflow` and `workflowSchedule` tables. These are essentially TypeScript representations of our database tables.
    *   **Line-by-Line:** `workflow` and `workflowSchedule` are objects that Drizzle uses to understand the structure and columns of the respective database tables. They allow for type-safe queries.
*   `import { eq } from 'drizzle-orm'`
    *   **Purpose:** Imports the `eq` (equals) function from Drizzle ORM. This is a common utility for constructing `WHERE` clauses in database queries.
    *   **Line-by-Line:** `eq` is a comparison operator used in Drizzle queries. For example, `eq(column, value)` translates to `column = value` in SQL.
*   `import { type NextRequest, NextResponse } from 'next/server'`
    *   **Purpose:** Imports type definitions and a class from Next.js for handling server-side HTTP requests and responses.
    *   **Line-by-Line:**
        *   `NextRequest` is a type representing the incoming HTTP request object, providing access to headers, body, URL, etc.
        *   `NextResponse` is a class used to create and send HTTP responses, allowing us to set status codes, JSON bodies, and headers.
*   `import { getSession } from '@/lib/auth'`
    *   **Purpose:** Imports a utility function `getSession` from a local authentication library. This function is used to retrieve the current user's session information.
    *   **Line-by-Line:** `getSession()` will typically return an object containing user details (like `id`, `email`, `name`) if the user is authenticated, or `null` otherwise.
*   `import { createLogger } from '@/lib/logs/console/logger'`
    *   **Purpose:** Imports a utility for creating a structured logger instance.
    *   **Line-by-Line:** `createLogger` is a factory function that, when called, returns a logger object (e.g., with methods like `info`, `warn`, `error`, `debug`) to output messages to the console or other logging destinations.
*   `import { getUserEntityPermissions } from '@/lib/permissions/utils'`
    *   **Purpose:** Imports a utility function for checking user permissions on specific entities (like workspaces).
    *   **Line-by-Line:** `getUserEntityPermissions` takes a user ID, an entity type (e.g., `'workspace'`), and an entity ID, and returns the user's permission level for that entity (e.g., `'read'`, `'write'`, `'admin'`, or `null`).
*   `import { generateRequestId } from '@/lib/utils'`
    *   **Purpose:** Imports a utility function to generate a unique request ID.
    *   **Line-by-Line:** `generateRequestId()` creates a unique string identifier for each incoming request, which is extremely useful for tracing requests through logs, especially in distributed systems.

### Global Configuration and Logger Initialization

*   `const logger = createLogger('ScheduleAPI')`
    *   **Purpose:** Initializes a logger instance specifically for this API file, tagged with `'ScheduleAPI'`. This helps in filtering and understanding logs related to schedule operations.
    *   **Line-by-Line:** Calls `createLogger` with a context string, creating a `logger` object ready to be used throughout the file.
*   `export const dynamic = 'force-dynamic'`
    *   **Purpose:** This is a Next.js specific configuration. It forces the route segment (this API route) to be dynamic, meaning it will be rendered on demand at request time and not statically cached during build.
    *   **Line-by-Line:** By setting `dynamic` to `'force-dynamic'`, we ensure that every request to this API route executes the latest code and fetches fresh data, preventing potential caching issues with sensitive or frequently changing data.

---

### `DELETE` Function: Delete a Schedule

This function handles HTTP `DELETE` requests to `/api/schedule/[id]`, allowing for the removal of a specific workflow schedule.

*   `export async function DELETE(`
    *   `request: NextRequest,`
    *   `{ params }: { params: Promise<{ id: string }> }`
    *   `) {`
    *   **Purpose:** Defines an asynchronous function that will respond to `DELETE` requests. Next.js automatically maps HTTP methods (like `DELETE`, `GET`, `POST`, `PUT`) to exported functions of the same name within API route files.
    *   **Line-by-Line:**
        *   `request: NextRequest`: The incoming HTTP request object.
        *   `{ params }: { params: Promise<{ id: string }> }`: This destructures the second argument, which contains route parameters. `params` is expected to be a `Promise` that resolves to an object with an `id` property (the schedule ID from the URL, e.g., `/api/schedule/123` would have `id: '123'`).

*   `const requestId = generateRequestId()`
    *   **Purpose:** Generates a unique ID for this specific request, used for consistent logging.
    *   **Line-by-Line:** Calls the utility function to get a new `requestId` string.

*   `try { ... } catch (error) { ... }`
    *   **Purpose:** This block encloses the main logic to gracefully handle any errors that might occur during execution, preventing the server from crashing and providing a standardized error response.

*   `const { id } = await params`
    *   **Purpose:** Extracts the schedule ID from the route parameters.
    *   **Line-by-Line:** Awaits the `params` promise to resolve and then destructures the `id` property from the resulting object.

*   `logger.debug(`[${requestId}] Deleting schedule with ID: ${id}`)`
    *   **Purpose:** Logs a debug message indicating that a schedule deletion attempt has begun for a specific ID, including the `requestId` for traceability.

*   `const session = await getSession()`
    *   **Purpose:** Retrieves the current user's session information.
    *   **Line-by-Line:** Awaits the `getSession()` utility, which fetches the authenticated user's details.

*   `if (!session?.user?.id) { ... }`
    *   **Purpose:** Checks if the user is authenticated. If there's no active session or no user ID in the session, the request is unauthorized.
    *   **Line-by-Line:**
        *   `!session?.user?.id`: Uses optional chaining (`?.`) to safely check if `session` exists, then `user` exists, and finally `id` exists. If any of these are missing, it evaluates to `true`.
        *   `logger.warn(...)`: Logs a warning about the unauthorized attempt.
        *   `return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })`: Sends a JSON response with an "Unauthorized" error message and an HTTP status code of `401`.

*   `// Find the schedule and check ownership`
    *   `const schedules = await db`
        *   `.select({`
            *   `schedule: workflowSchedule,`
            *   `workflow: {`
                *   `id: workflow.id,`
                *   `userId: workflow.userId,`
                *   `workspaceId: workflow.workspaceId,`
            *   `},`
        *   `})`
        *   `.from(workflowSchedule)`
        *   `.innerJoin(workflow, eq(workflowSchedule.workflowId, workflow.id))`
        *   `.where(eq(workflowSchedule.id, id))`
        *   `.limit(1)`
    *   **Purpose:** This is a complex Drizzle ORM query designed to retrieve the specific `workflowSchedule` record and its associated `workflow` record. It's crucial for checking both the schedule's existence and the ownership/permissions for deletion.
    *   **Line-by-Line:**
        *   `db.select(...)`: Starts a `SELECT` query.
        *   `{ schedule: workflowSchedule, workflow: { ... } }`: Specifies what columns to select. It selects all columns from `workflowSchedule` (aliased as `schedule`) and specific columns (`id`, `userId`, `workspaceId`) from the joined `workflow` table (aliased as `workflow`). This structure makes the returned data easy to work with.
        *   `.from(workflowSchedule)`: Specifies that the primary table for the query is `workflowSchedule`.
        *   `.innerJoin(workflow, eq(workflowSchedule.workflowId, workflow.id))`: Performs an `INNER JOIN` with the `workflow` table. The join condition is `workflowSchedule.workflowId = workflow.id`, linking a schedule to its parent workflow.
        *   `.where(eq(workflowSchedule.id, id))`: Filters the results to find the schedule with the ID matching the one provided in the URL (`id`).
        *   `.limit(1)`: Ensures that the query returns at most one record, as we are looking for a unique schedule by its ID.

*   `if (schedules.length === 0) { ... }`
    *   **Purpose:** Checks if a schedule with the given ID was found in the database.
    *   **Line-by-Line:** If the `schedules` array is empty, it means no matching schedule was found.
        *   `logger.warn(...)`: Logs a warning indicating the schedule was not found.
        *   `return NextResponse.json({ error: 'Schedule not found' }, { status: 404 })`: Sends a `404 Not Found` response.

*   `const workflowRecord = schedules[0].workflow`
    *   **Purpose:** Extracts the associated workflow details from the query result.
    *   **Line-by-Line:** Since `limit(1)` was used, `schedules[0]` will contain the single result. We then access its `workflow` property, which holds the selected `workflow` details.

*   `// Check authorization - either the user owns the workflow or has write/admin workspace permissions`
    *   `let isAuthorized = workflowRecord.userId === session.user.id`
    *   **Purpose:** Initializes an authorization flag. The simplest form of authorization is checking if the current user owns the workflow associated with the schedule.
    *   **Line-by-Line:** Sets `isAuthorized` to `true` if the `userId` on the `workflowRecord` matches the `id` of the authenticated `session.user`.

*   `// If not authorized by ownership and the workflow belongs to a workspace, check workspace permissions`
    *   `if (!isAuthorized && workflowRecord.workspaceId) { ... }`
    *   **Purpose:** This is a more complex authorization check. If the user isn't the direct owner of the workflow, and the workflow is part of a workspace, it then checks if the user has `write` or `admin` permissions on that workspace. This allows team members with appropriate permissions to manage schedules.
    *   **Line-by-Line:**
        *   `!isAuthorized`: Checks if the user is *not* already authorized by direct ownership.
        *   `workflowRecord.workspaceId`: Checks if the workflow is associated with a `workspaceId`.
        *   `const userPermission = await getUserEntityPermissions(...)`: Calls the permission utility with the user's ID, entity type `'workspace'`, and the `workspaceId`.
        *   `isAuthorized = userPermission === 'write' || userPermission === 'admin'`: Updates `isAuthorized` to `true` if the `userPermission` is either `'write'` or `'admin'`.

*   `if (!isAuthorized) { ... }`
    *   **Purpose:** If, after all checks, the user is still not authorized, prevents deletion.
    *   **Line-by-Line:**
        *   `logger.warn(...)`: Logs a warning about the unauthorized deletion attempt.
        *   `return NextResponse.json({ error: 'Unauthorized' }, { status: 403 })`: Sends a `403 Forbidden` response. This is different from `401 Unauthorized` because here the user is authenticated, but they lack the *permission* to perform the action.

*   `// Delete the schedule`
    *   `await db.delete(workflowSchedule).where(eq(workflowSchedule.id, id))`
    *   **Purpose:** Executes the actual deletion of the schedule from the database.
    *   **Line-by-Line:**
        *   `db.delete(workflowSchedule)`: Initiates a `DELETE` operation on the `workflowSchedule` table.
        *   `.where(eq(workflowSchedule.id, id))`: Specifies the condition for deletion: delete the record where the `id` column matches the provided `id`.

*   `logger.info(`[${requestId}] Successfully deleted schedule: ${id}`)`
    *   **Purpose:** Logs a success message after the schedule is deleted.

*   `return NextResponse.json({ success: true }, { status: 200 })`
    *   **Purpose:** Sends a successful JSON response.
    *   **Line-by-Line:** Returns a `200 OK` response with a JSON body indicating success.

*   `} catch (error) { ... }` (Error Handling for `DELETE`)
    *   **Purpose:** Catches any unexpected errors during the deletion process.
    *   **Line-by-Line:**
        *   `logger.error(...)`: Logs the error with the `requestId` and the error object itself for debugging.
        *   `return NextResponse.json({ error: 'Internal server error' }, { status: 500 })`: Sends a generic `500 Internal Server Error` response, avoiding exposing sensitive error details to the client.

---

### `PUT` Function: Update a Schedule

This function handles HTTP `PUT` requests to `/api/schedule/[id]`, primarily used to change a schedule's status (reactivate or disable).

*   `export async function PUT(`
    *   `request: NextRequest,`
    *   `{ params }: { params: Promise<{ id: string }> }`
    *   `) {`
    *   **Purpose:** Defines an asynchronous function to handle `PUT` requests.
    *   **Line-by-Line:** Similar to the `DELETE` function, it receives the `NextRequest` and route `params` containing the schedule `id`.

*   `const requestId = generateRequestId()`
    *   **Purpose:** Generates a unique request ID for logging.

*   `try { ... } catch (error) { ... }`
    *   **Purpose:** Error handling block for the `PUT` operation.

*   `const { id } = await params`
    *   `const scheduleId = id`
    *   **Purpose:** Extracts and renames the schedule ID for clarity.
    *   **Line-by-Line:** Awaits `params` and assigns its `id` property to a new constant `scheduleId`.

*   `logger.debug(`[${requestId}] Updating schedule with ID: ${scheduleId}`)`
    *   **Purpose:** Logs a debug message for the update attempt.

*   `const session = await getSession()`
    *   `if (!session?.user?.id) { ... }`
    *   **Purpose:** Authenticates the user, similar to the `DELETE` function.
    *   **Line-by-Line:** Checks for an active session and user ID; returns `401 Unauthorized` if not found.

*   `const body = await request.json()`
    *   `const { action } = body`
    *   **Purpose:** Parses the JSON body of the incoming request to get the desired `action`.
    *   **Line-by-Line:** `request.json()` asynchronously reads the request body and parses it as JSON. Then, `action` is extracted from the parsed `body` object.

*   `const [schedule] = await db`
    *   `.select({`
        *   `id: workflowSchedule.id,`
        *   `workflowId: workflowSchedule.workflowId,`
        *   `status: workflowSchedule.status,`
    *   `})`
    *   `.from(workflowSchedule)`
    *   `.where(eq(workflowSchedule.id, scheduleId))`
    *   `.limit(1)`
    *   **Purpose:** Fetches essential details of the target `workflowSchedule` (ID, associated `workflowId`, and current `status`) to inform subsequent logic.
    *   **Line-by-Line:**
        *   `db.select(...)`: Selects specific columns.
        *   `[schedule]`: Uses array destructuring because Drizzle's `select` returns an array of results, and `limit(1)` guarantees at most one result. This effectively assigns the first (and only) result to `schedule`.
        *   `.from(workflowSchedule)`: Queries the `workflowSchedule` table.
        *   `.where(eq(workflowSchedule.id, scheduleId))`: Filters by the provided `scheduleId`.
        *   `.limit(1)`: Limits to a single result.

*   `if (!schedule) { ... }`
    *   **Purpose:** Checks if the schedule was found.
    *   **Line-by-Line:** If `schedule` is `null` or `undefined`, it means no schedule matched the `scheduleId`. Logs a warning and returns `404 Not Found`.

*   `const [workflowRecord] = await db`
    *   `.select({ userId: workflow.userId })`
    *   `.from(workflow)`
    *   `.where(eq(workflow.id, schedule.workflowId))`
    *   `.limit(1)`
    *   **Purpose:** Fetches the `userId` of the workflow associated with the schedule. This is needed for the ownership check.
    *   **Line-by-Line:** Similar `select` and `where` pattern to retrieve the `userId` from the `workflow` table, using the `workflowId` obtained from the `schedule`.

*   `if (!workflowRecord) { ... }`
    *   **Purpose:** Checks if the associated workflow was found. Although less likely if a schedule exists, it's a good defensive check.
    *   **Line-by-Line:** If the `workflowRecord` is not found, logs a warning and returns `404 Not Found`.

*   `if (workflowRecord.userId !== session.user.id) { ... }`
    *   **Purpose:** Performs an authorization check. For updating a schedule, the user must be the direct owner of the associated workflow. (Note: Unlike `DELETE`, `PUT` here doesn't include workspace permissions).
    *   **Line-by-Line:** If the workflow's `userId` does not match the authenticated user's `id`, it means the user doesn't own the workflow. Logs a warning and returns `403 Forbidden`.

*   `if (action === 'reactivate' || (body.status && body.status === 'active')) { ... }`
    *   **Purpose:** Handles the logic for reactivating a schedule. It checks if the `action` is 'reactivate' or if `body.status` is 'active'.
    *   **Line-by-Line:**
        *   `if (schedule.status === 'active')`: If the schedule is already active, returns a `200 OK` with a message that no change is needed.
        *   `const now = new Date()`: Gets the current date and time.
        *   `const nextRunAt = new Date(now.getTime() + 60 * 1000)`: Calculates `nextRunAt` to be 1 minute from now. This is a common pattern for reactivating schedules: give them a near-future trigger time.
        *   `await db.update(workflowSchedule).set({ ... }).where(...)`: Updates the `workflowSchedule` record:
            *   `status: 'active'`: Sets the status to active.
            *   `failedCount: 0`: Resets any previous failure count.
            *   `updatedAt: now`: Updates the modification timestamp.
            *   `nextRunAt`: Sets the calculated `nextRunAt` time.
            *   `.where(eq(workflowSchedule.id, scheduleId))`: Ensures only the target schedule is updated.
        *   `logger.info(...)`: Logs a success message.
        *   `return NextResponse.json({ ... })`: Returns a `200 OK` response confirming activation and including the `nextRunAt` time.

*   `if (action === 'disable' || (body.status && body.status === 'disabled')) { ... }`
    *   **Purpose:** Handles the logic for disabling a schedule. It checks if the `action` is 'disable' or if `body.status` is 'disabled'.
    *   **Line-by-Line:**
        *   `if (schedule.status === 'disabled')`: If the schedule is already disabled, returns a `200 OK` with a message.
        *   `const now = new Date()`: Gets the current date and time.
        *   `await db.update(workflowSchedule).set({ ... }).where(...)`: Updates the `workflowSchedule` record:
            *   `status: 'disabled'`: Sets the status to disabled.
            *   `updatedAt: now`: Updates the modification timestamp.
            *   `nextRunAt: null`: Crucially, clears `nextRunAt` so the scheduler knows not to run this schedule until it's reactivated.
            *   `.where(eq(workflowSchedule.id, scheduleId))`: Ensures only the target schedule is updated.
        *   `logger.info(...)`: Logs a success message.
        *   `return NextResponse.json({ ... })`: Returns a `200 OK` response confirming disablement.

*   `logger.warn(`[${requestId}] Unsupported update action for schedule: ${scheduleId}`)`
    *   `return NextResponse.json({ error: 'Unsupported update action' }, { status: 400 })`
    *   **Purpose:** If the `action` or `status` in the request body does not match 'reactivate', 'active', 'disable', or 'disabled', it means an unsupported update was requested.
    *   **Line-by-Line:** Logs a warning about the unsupported action and returns a `400 Bad Request` error.

*   `} catch (error) { ... }` (Error Handling for `PUT`)
    *   **Purpose:** Catches any unexpected errors during the update process.
    *   **Line-by-Line:**
        *   `logger.error(...)`: Logs the error with the `requestId`.
        *   `return NextResponse.json({ error: 'Failed to update schedule' }, { status: 500 })`: Sends a `500 Internal Server Error` response.

---

This file provides a well-structured and secure way to manage workflow schedules via a RESTful API, demonstrating best practices in authentication, authorization, logging, and database interaction.