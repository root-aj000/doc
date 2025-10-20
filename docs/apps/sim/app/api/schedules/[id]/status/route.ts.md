This TypeScript file defines an API endpoint for a Next.js application. Its primary goal is to allow authenticated and authorized users to retrieve the current status of a specific "workflow schedule."

---

### Purpose of this File and What it Does

This file creates a **Next.js API Route** that handles `GET` requests to a path like `/api/schedules/[id]/status`.

In essence, it acts as a secure gateway to check the live status of a scheduled automated task (a "workflow schedule").

Here's a breakdown of its key functions:

1.  **Retrieve Schedule Information:** It queries a database to find a specific `workflowSchedule` record identified by an ID provided in the request URL.
2.  **Authentication:** It first ensures that the person making the request is logged in and has an active user session.
3.  **Authorization:** This is a critical part. It performs two levels of checks to ensure the logged-in user *has the right* to view this particular schedule:
    *   **Ownership:** Is the user the creator or owner of the workflow that this schedule belongs to?
    *   **Workspace Permissions:** If not the owner, does the workflow belong to a shared workspace, and does the user have any permissions within that workspace?
4.  **Respond with Status:** If all checks pass, it returns a JSON object containing key status details of the workflow schedule (e.g., current status, last run time, next run time, etc.).
5.  **Robust Error Handling:** It gracefully handles various error scenarios, such as the schedule not being found, the user not being authenticated, or the user not being authorized, returning appropriate HTTP status codes and messages.
6.  **Logging:** It uses a structured logging system to record important events and errors, making it easier to monitor and debug the API.

---

### Simplifying Complex Logic

Let's break down the more complex parts into simpler terms:

1.  **Database Interaction (Drizzle ORM):**
    *   Instead of writing raw SQL, this code uses `drizzle-orm`. Think of Drizzle as a translator. You write JavaScript code like `db.select().from().where()`, and Drizzle translates it into the correct SQL query for the database.
    *   `db`: This is our connected database client.
    *   `workflow`, `workflowSchedule`: These are like blueprints for our database tables. `workflowSchedule` stores details about *when* a task should run, and `workflow` stores details about the *actual task*.
    *   `eq`: This is a Drizzle function that means "equals to." So, `eq(workflowSchedule.id, scheduleId)` translates to `WHERE workflowSchedule.id = :scheduleId` in SQL.
    *   `select({...})`: This specifies *which columns* we want to retrieve from the database.
    *   `from(...)`: This specifies *which table* we are querying.
    *   `where(...)`: This specifies the *conditions* for selecting rows.
    *   `limit(1)`: This ensures we only get at most one matching record.

2.  **Authorization Logic:**
    The authorization check is designed to be flexible:
    *   **First Attempt (Ownership):** The system first checks if the logged-in user is the direct owner of the workflow. This is the simplest and most direct form of authorization.
    *   **Second Attempt (Workspace Sharing):** If the user is *not* the owner, the system then checks if the workflow is associated with a "workspace." If it is, it calls a special function (`getUserEntityPermissions`) to see if the logged-in user has *any* kind of access or permission within that specific workspace. If they do, they are considered authorized to view the schedule, even if they didn't create it directly.
    *   This ensures that workflows shared within teams (via workspaces) are accessible to team members.

---

### Line-by-Line Explanation

```typescript
import { db } from '@sim/db'
import { workflow, workflowSchedule } from '@sim/db/schema'
import { eq } from 'drizzle-orm'
import { type NextRequest, NextResponse } from 'next/server'
import { getSession } from '@/lib/auth'
import { createLogger } from '@/lib/logs/console/logger'
import { getUserEntityPermissions } from '@/lib/permissions/utils'
import { generateRequestId } from '@/lib/utils'
```
*   **`import { db } from '@sim/db'`**: Imports the Drizzle ORM database connection instance, which is used to interact with the database.
*   **`import { workflow, workflowSchedule } from '@sim/db/schema'`**: Imports schema definitions for the `workflow` and `workflowSchedule` tables (or collections) from the application's database schema. These are Drizzle "table objects" used to build queries.
*   **`import { eq } from 'drizzle-orm'`**: Imports the `eq` (equals) function from Drizzle ORM, used to create equality conditions in database queries (e.g., `WHERE id = value`).
*   **`import { type NextRequest, NextResponse } from 'next/server'`**: Imports `NextRequest` (a type for the incoming HTTP request) and `NextResponse` (a class for creating HTTP responses) from Next.js, specific to its Edge Runtime or API routes.
*   **`import { getSession } from '@/lib/auth'`**: Imports a utility function `getSession` from the application's authentication library. This function is responsible for retrieving the current user's session information.
*   **`import { createLogger } from '@/lib/logs/console/logger'`**: Imports a utility function `createLogger` to initialize a structured logger instance for console logging.
*   **`import { getUserEntityPermissions } from '@/lib/permissions/utils'`**: Imports a utility function `getUserEntityPermissions` used to check if a user has specific permissions related to a given entity (like a workspace).
*   **`import { generateRequestId } from '@/lib/utils'`**: Imports a utility function `generateRequestId` that creates a unique ID for each incoming request, helpful for tracing and debugging.

```typescript
const logger = createLogger('ScheduleStatusAPI')
```
*   **`const logger = createLogger('ScheduleStatusAPI')`**: Initializes a logger specifically for this API route, naming it 'ScheduleStatusAPI'. This helps categorize and filter log messages originating from this file.

```typescript
export async function GET(req: NextRequest, { params }: { params: Promise<{ id: string }> }) {
```
*   **`export async function GET(...)`**: This defines an asynchronous function named `GET`. In Next.js API routes, exported functions named after HTTP methods (`GET`, `POST`, `PUT`, `DELETE`) are automatically invoked when a request matching that method comes to this route.
*   **`req: NextRequest`**: The first parameter, `req`, is the incoming HTTP request object, typed as `NextRequest`, providing access to headers, body, URL, etc.
*   **`{ params }: { params: Promise<{ id: string }> }`**: The second parameter is an object that contains route parameters. Here, `params` is destructured from this object. It's typed as a `Promise<{ id: string }>`, indicating that the `id` (from the dynamic route segment `[id]`) will be available, possibly after an asynchronous resolution.

```typescript
  const requestId = generateRequestId()
  const { id } = await params
  const scheduleId = id
```
*   **`const requestId = generateRequestId()`**: Generates a unique identifier for the current request. This ID is often logged with messages to track a request's lifecycle.
*   **`const { id } = await params`**: Awaits the `params` promise to resolve and then destructures the `id` property from it. This `id` comes from the URL, e.g., if the URL is `/api/schedules/123/status`, then `id` will be `"123"`.
*   **`const scheduleId = id`**: Assigns the extracted `id` to a more descriptive variable named `scheduleId` for clarity.

```typescript
  try {
    const session = await getSession()
    if (!session?.user?.id) {
      logger.warn(`[${requestId}] Unauthorized schedule status request`)
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
    }
```
*   **`try { ... } catch (error) { ... }`**: This `try...catch` block wraps the main logic to gracefully handle any unexpected errors that might occur during execution.
*   **`const session = await getSession()`**: Calls the `getSession` utility to retrieve the current user's authentication session details. This is an asynchronous operation.
*   **`if (!session?.user?.id)`**: Checks if a `session` exists and, within that session, if a `user` object with an `id` is present. If any of these are missing, it means the user is not authenticated.
*   **`logger.warn(...)`**: Logs a warning message using the `requestId` if the user is not authenticated.
*   **`return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })`**: Returns a JSON response with an `error` message and an HTTP status code of `401 Unauthorized`, indicating that authentication is required.

```typescript
    const [schedule] = await db
      .select({
        id: workflowSchedule.id,
        workflowId: workflowSchedule.workflowId,
        status: workflowSchedule.status,
        failedCount: workflowSchedule.failedCount,
        lastRanAt: workflowSchedule.lastRanAt,
        lastFailedAt: workflowSchedule.lastFailedAt,
        nextRunAt: workflowSchedule.nextRunAt,
      })
      .from(workflowSchedule)
      .where(eq(workflowSchedule.id, scheduleId))
      .limit(1)
```
*   **`const [schedule] = await db.select(...)`**: This is a Drizzle ORM query to fetch a single `workflowSchedule` record from the database.
    *   **`db.select(...)`**: Initiates a selection query.
    *   **`{ id: workflowSchedule.id, ... }`**: Specifies the exact columns (fields) we want to retrieve from the `workflowSchedule` table. We are selecting `id`, `workflowId`, `status`, `failedCount`, `lastRanAt`, `lastFailedAt`, and `nextRunAt`.
    *   **`.from(workflowSchedule)`**: Specifies that we are querying the `workflowSchedule` table.
    *   **`.where(eq(workflowSchedule.id, scheduleId))`**: Filters the results. It selects only the row where the `id` column of the `workflowSchedule` table matches the `scheduleId` obtained from the request URL. `eq` stands for "equals."
    *   **`.limit(1)`**: Ensures that the query returns at most one record.
    *   **`const [schedule]`**: Drizzle's `select` method always returns an array, even if `limit(1)` is used. This array destructuring extracts the first (and only) element into the `schedule` variable. If no record is found, `schedule` will be `undefined`.

```typescript
    if (!schedule) {
      logger.warn(`[${requestId}] Schedule not found: ${scheduleId}`)
      return NextResponse.json({ error: 'Schedule not found' }, { status: 404 })
    }
```
*   **`if (!schedule)`**: Checks if a `workflowSchedule` record was found by the previous database query. If `schedule` is `undefined` (or null), it means the schedule doesn't exist.
*   **`logger.warn(...)`**: Logs a warning message indicating that the requested schedule ID was not found.
*   **`return NextResponse.json({ error: 'Schedule not found' }, { status: 404 })`**: Returns a JSON response with an error message and an HTTP status code of `404 Not Found`.

```typescript
    const [workflowRecord] = await db
      .select({ userId: workflow.userId, workspaceId: workflow.workspaceId })
      .from(workflow)
      .where(eq(workflow.id, schedule.workflowId))
      .limit(1)
```
*   **`const [workflowRecord] = await db.select(...)`**: Another Drizzle query, this time to fetch information about the *workflow* associated with the found schedule.
    *   **`db.select(...)`**: Initiates a selection query.
    *   **`{ userId: workflow.userId, workspaceId: workflow.workspaceId }`**: Selects only the `userId` (the owner of the workflow) and `workspaceId` (if it belongs to a workspace) from the `workflow` table.
    *   **`.from(workflow)`**: Specifies that we are querying the `workflow` table.
    *   **`.where(eq(workflow.id, schedule.workflowId))`**: Filters the results to find the workflow whose `id` matches the `workflowId` obtained from the `schedule` record. This links the schedule back to its parent workflow.
    *   **`.limit(1)`**: Ensures at most one workflow record is returned.
    *   **`const [workflowRecord]`**: Destructures the first element of the result array into `workflowRecord`.

```typescript
    if (!workflowRecord) {
      logger.warn(`[${requestId}] Workflow not found for schedule: ${scheduleId}`)
      return NextResponse.json({ error: 'Workflow not found' }, { status: 404 })
    }
```
*   **`if (!workflowRecord)`**: Checks if the associated `workflow` record was found. If not, it could indicate a data inconsistency (a schedule pointing to a non-existent workflow).
*   **`logger.warn(...)`**: Logs a warning if the parent workflow is not found.
*   **`return NextResponse.json({ error: 'Workflow not found' }, { status: 404 })`**: Returns a JSON response with an error message and an HTTP status code of `404 Not Found`.

```typescript
    // Check authorization - either the user owns the workflow or has workspace permissions
    let isAuthorized = workflowRecord.userId === session.user.id

    // If not authorized by ownership and the workflow belongs to a workspace, check workspace permissions
    if (!isAuthorized && workflowRecord.workspaceId) {
      const userPermission = await getUserEntityPermissions(
        session.user.id,
        'workspace',
        workflowRecord.workspaceId
      )
      isAuthorized = userPermission !== null
    }
```
*   **`let isAuthorized = workflowRecord.userId === session.user.id`**: This is the first authorization check. It sets `isAuthorized` to `true` if the `userId` of the workflow record matches the `id` of the currently logged-in user (meaning the user owns the workflow). Otherwise, `isAuthorized` is `false`.
*   **`if (!isAuthorized && workflowRecord.workspaceId)`**: This block executes if the user is *not* the owner (`!isAuthorized`) AND the workflow is associated with a `workspaceId` (i.e., it's part of a shared workspace).
    *   **`const userPermission = await getUserEntityPermissions(...)`**: Calls a utility function to check if the logged-in user (`session.user.id`) has *any* permissions (`'workspace'`) within the specific `workflowRecord.workspaceId`.
    *   **`isAuthorized = userPermission !== null`**: If `getUserEntityPermissions` returns something (i.e., not `null`), it means the user has some level of permission in that workspace, so `isAuthorized` is set to `true`.

```typescript
    if (!isAuthorized) {
      logger.warn(`[${requestId}] User not authorized to view this schedule: ${scheduleId}`)
      return NextResponse.json({ error: 'Not authorized to view this schedule' }, { status: 403 })
    }
```
*   **`if (!isAuthorized)`**: After both ownership and workspace permission checks, if `isAuthorized` is still `false`, it means the user does not have permission to view this schedule.
*   **`logger.warn(...)`**: Logs a warning indicating that the user is not authorized.
*   **`return NextResponse.json({ error: 'Not authorized to view this schedule' }, { status: 403 })`**: Returns a JSON response with an error message and an HTTP status code of `403 Forbidden`. This differs from `401 Unauthorized` because `403` means the user *is* authenticated but simply doesn't have access to *this specific resource*.

```typescript
    return NextResponse.json({
      status: schedule.status,
      failedCount: schedule.failedCount,
      lastRanAt: schedule.lastRanAt,
      lastFailedAt: schedule.lastFailedAt,
      nextRunAt: schedule.nextRanAt,
      isDisabled: schedule.status === 'disabled',
    })
```
*   **`return NextResponse.json(...)`**: If all authentication and authorization checks pass, this line returns a successful JSON response.
*   The JSON object contains the key status details extracted from the `schedule` record: `status`, `failedCount`, `lastRanAt`, `lastFailedAt`, and `nextRunAt`.
*   **`isDisabled: schedule.status === 'disabled'`**: A derived property `isDisabled` is added for convenience, which is `true` if the schedule's `status` is `'disabled'`, and `false` otherwise.

```typescript
  } catch (error) {
    logger.error(`[${requestId}] Error retrieving schedule status: ${scheduleId}`, error)
    return NextResponse.json({ error: 'Failed to retrieve schedule status' }, { status: 500 })
  }
}
```
*   **`} catch (error)`**: This block catches any unexpected errors that occurred within the `try` block (e.g., database connection issues, unhandled runtime errors).
*   **`logger.error(...)`**: Logs the error with the `requestId` and the `scheduleId`, including the actual `error` object for detailed debugging.
*   **`return NextResponse.json({ error: 'Failed to retrieve schedule status' }, { status: 500 })`**: Returns a generic JSON error message with an HTTP status code of `500 Internal Server Error`. This is good practice to avoid exposing sensitive internal error details to the client.