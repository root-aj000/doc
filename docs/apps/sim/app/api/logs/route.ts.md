This file is a **Next.js API route handler** (`pages/api/logs.ts` or `app/api/logs/route.ts` if using the `app` directory). It acts as a backend endpoint for fetching and filtering workflow execution logs.

It allows clients to query logs with various criteria, such as specific workflows, folders, date ranges, and provides different levels of detail (basic vs. full) to optimize data transfer and performance. It also handles user authentication and authorization, ensuring that only users with appropriate permissions can access the logs.

---

### Core Functionality Summary

1.  **Authentication:** Verifies if the request comes from an authenticated user.
2.  **Request Validation:** Uses Zod to parse and validate incoming query parameters for filtering, pagination, and detail level.
3.  **Dynamic SQL Query Construction:** Builds a Drizzle ORM query based on the validated parameters, dynamically applying filters for workflow IDs, folder IDs, date ranges, search terms, and more.
4.  **Database Joins:** Joins `workflowExecutionLogs` with `workflow` and `permissions` tables to retrieve workflow metadata and enforce access control (user must have permission to the workspace).
5.  **Conditional Data Retrieval:** Selects specific columns based on the requested detail level (`basic` or `full`), optimizing performance by excluding large data fields in `basic` mode.
6.  **Pagination:** Implements `limit` and `offset` for fetching paginated results and performs a separate count query to provide total record count.
7.  **Data Transformation:** Processes the raw database results, enriching them with structured workflow information, trace spans, and detailed cost summaries (especially in `full` mode).
8.  **Error Handling:** Catches and responds to validation errors (Zod) and general server errors, providing appropriate HTTP status codes and messages.

---

### Detailed Line-by-Line Explanation

Let's break down the code section by section.

#### Imports

```typescript
import { db } from '@sim/db'
import { permissions, workflow, workflowExecutionLogs } from '@sim/db/schema'
import { and, desc, eq, gte, inArray, lte, type SQL, sql } from 'drizzle-orm'
import { type NextRequest, NextResponse } from 'next/server'
import { z } from 'zod'
import { getSession } from '@/lib/auth'
import { createLogger } from '@/lib/logs/console/logger'
import { generateRequestId } from '@/lib/utils'
```

These lines import necessary modules and utilities:

*   `import { db } from '@sim/db'`: Imports the Drizzle ORM database client instance. `db` is the primary interface for interacting with the database.
*   `import { permissions, workflow, workflowExecutionLogs } from '@sim/db/schema'`: Imports table schemas (definitions) from the Drizzle ORM.
    *   `permissions`: Represents user permissions (likely for workspaces or workflows).
    *   `workflow`: Represents the definitions of workflows.
    *   `workflowExecutionLogs`: The main table storing logs for each workflow execution.
*   `import { and, desc, eq, gte, inArray, lte, type SQL, sql } from 'drizzle-orm'`: Imports Drizzle ORM's utility functions for building SQL queries:
    *   `and`: Combines multiple conditions with a logical `AND`.
    *   `desc`: Specifies descending order for sorting.
    *   `eq`: Checks for equality (`=`).
    *   `gte`: Checks for "greater than or equal to" (`>=`).
    *   `inArray`: Checks if a column's value is present in a given array of values (`IN (...)`).
    *   `lte`: Checks for "less than or equal to" (`<=`).
    *   `type SQL`: A Drizzle type representing a raw SQL expression, useful for building dynamic conditions.
    *   `sql`: A Drizzle template literal tag for executing raw SQL fragments or functions.
*   `import { type NextRequest, NextResponse } from 'next/server'`: Imports types and classes from Next.js for handling API requests and responses.
    *   `NextRequest`: The type for the incoming HTTP request object.
    *   `NextResponse`: A utility class for creating HTTP responses.
*   `import { z } from 'zod'`: Imports the Zod library, a powerful TypeScript-first schema declaration and validation library.
*   `import { getSession } from '@/lib/auth'`: Imports a utility function to retrieve the user's session, typically used for authentication.
*   `import { createLogger } from '@/lib/logs/console/logger'`: Imports a custom logger utility for structured logging within the application.
*   `import { generateRequestId } from '@/lib/utils'`: Imports a utility to generate a unique request ID, useful for tracing individual requests through logs.

#### Configuration and Logger

```typescript
const logger = createLogger('LogsAPI')

export const revalidate = 0
```

*   `const logger = createLogger('LogsAPI')`: Initializes a logger instance named 'LogsAPI'. This logger will be used throughout the file to log informational messages, warnings, and errors, often including the `requestId` for context.
*   `export const revalidate = 0`: This is a Next.js specific export. Setting `revalidate = 0` (or `false` for older versions) indicates that this API route should **not be cached** by Next.js's data cache. This is appropriate for dynamic API endpoints that return fresh data on every request.

#### Query Parameter Validation Schema

```typescript
const QueryParamsSchema = z.object({
  details: z.enum(['basic', 'full']).optional().default('basic'),
  limit: z.coerce.number().optional().default(100),
  offset: z.coerce.number().optional().default(0),
  level: z.string().optional(),
  workflowIds: z.string().optional(), // Comma-separated list of workflow IDs
  folderIds: z.string().optional(), // Comma-separated list of folder IDs
  triggers: z.string().optional(), // Comma-separated list of trigger types
  startDate: z.string().optional(),
  endDate: z.string().optional(),
  search: z.string().optional(),
  workflowName: z.string().optional(),
  folderName: z.string().optional(),
  workspaceId: z.string(),
})
```

This Zod schema defines the expected structure and types of the URL query parameters that the API route accepts. It ensures that incoming parameters are valid before they are used in the database query.

*   `z.object({...})`: Defines an object schema.
*   `details: z.enum(['basic', 'full']).optional().default('basic')`:
    *   `z.enum(['basic', 'full'])`: Expects the `details` parameter to be either 'basic' or 'full'.
    *   `.optional()`: The parameter is not mandatory.
    *   `.default('basic')`: If not provided, it defaults to 'basic'. This allows clients to request either a basic set of log data (for lists) or a full, detailed set (for individual log views).
*   `limit: z.coerce.number().optional().default(100)`:
    *   `z.coerce.number()`: Attempts to convert the parameter to a number. If `limit` is passed as a string (e.g., `?limit=50`), Zod will coerce it to the number `50`.
    *   `.optional().default(100)`: Defaults to 100 if not provided, allowing clients to specify how many logs to retrieve per page (pagination).
*   `offset: z.coerce.number().optional().default(0)`: Similar to `limit`, but specifies how many records to skip from the beginning, defaulting to 0. Used for pagination.
*   `level: z.string().optional()`: An optional string to filter logs by their severity level (e.g., 'info', 'warn', 'error').
*   `workflowIds: z.string().optional()`: An optional string expected to be a comma-separated list of workflow IDs. This allows filtering logs for specific workflows.
*   `folderIds: z.string().optional()`: An optional string, comma-separated list of folder IDs to filter logs by the folder their associated workflow belongs to.
*   `triggers: z.string().optional()`: An optional string, comma-separated list of trigger types (e.g., 'manual', 'schedule', 'webhook') to filter logs.
*   `startDate: z.string().optional()`: An optional string representing the start date for filtering logs (e.g., `2023-01-01`).
*   `endDate: z.string().optional()`: An optional string representing the end date for filtering logs.
*   `search: z.string().optional()`: An optional string for a general search query, likely matching against `executionId` or other log fields.
*   `workflowName: z.string().optional()`: An optional string for searching workflow logs by the workflow's name.
*   `folderName: z.string().optional()`: An optional string for searching workflow logs by the folder's name (associated with the workflow).
*   `workspaceId: z.string()`: A **required** string representing the workspace ID. This is critical for scoping queries to a specific workspace and for authorization.

#### The `GET` Request Handler

```typescript
export async function GET(request: NextRequest) {
  const requestId = generateRequestId()

  try {
    const session = await getSession()
    if (!session?.user?.id) {
      logger.warn(`[${requestId}] Unauthorized logs access attempt`)
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
    }

    const userId = session.user.id

    try {
      const { searchParams } = new URL(request.url)
      const params = QueryParamsSchema.parse(Object.fromEntries(searchParams.entries()))

      // ... (rest of the logic inside the inner try block)

    } catch (validationError) {
      if (validationError instanceof z.ZodError) {
        logger.warn(`[${requestId}] Invalid logs request parameters`, {
          errors: validationError.errors,
        })
        return NextResponse.json(
          {
            error: 'Invalid request parameters',
            details: validationError.errors,
          },
          { status: 400 }
        )
      }
      throw validationError
    }
  } catch (error: any) {
    logger.error(`[${requestId}] logs fetch error`, error)
    return NextResponse.json({ error: error.message }, { status: 500 })
  }
}
```

This is the main function that handles incoming `GET` requests to the API route.

*   `export async function GET(request: NextRequest)`: Defines an asynchronous function `GET` that receives a `NextRequest` object. This is the standard signature for a Next.js API route handler.
*   `const requestId = generateRequestId()`: Generates a unique ID for the current request. This `requestId` is then used in log messages for easy tracing of request flow and debugging.

*   **Outer `try...catch` Block (General Error Handling):**
    *   `try { ... } catch (error: any) { ... }`: This outer block catches any unexpected errors that occur during the entire request processing.
    *   `logger.error(`[${requestId}] logs fetch error`, error)`: Logs the error using the `logger` instance, including the `requestId` for context.
    *   `return NextResponse.json({ error: error.message }, { status: 500 })`: Returns a JSON response with an error message and a `500 Internal Server Error` status code.

*   **Authentication:**
    *   `const session = await getSession()`: Calls the `getSession` utility to retrieve the current user's session information. This function typically interacts with a session management system (like NextAuth.js).
    *   `if (!session?.user?.id)`: Checks if a session exists and if the user ID is present in the session.
    *   `logger.warn(`[${requestId}] Unauthorized logs access attempt`)`: If no valid user ID is found, a warning is logged.
    *   `return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })`: Returns a JSON response indicating "Unauthorized" with a `401 Unauthorized` HTTP status code.
    *   `const userId = session.user.id`: If authentication is successful, extracts the `userId` from the session for subsequent authorization checks.

*   **Inner `try...catch` Block (Parameter Validation Error Handling):**
    *   `try { ... } catch (validationError) { ... }`: This inner block specifically handles errors related to parsing and validating the query parameters.
    *   `if (validationError instanceof z.ZodError)`: Checks if the error is an instance of `ZodError`, which means the input parameters did not conform to `QueryParamsSchema`.
    *   `logger.warn(`[${requestId}] Invalid logs request parameters`, { errors: validationError.errors })`: Logs a warning with details of the validation errors.
    *   `return NextResponse.json({ error: 'Invalid request parameters', details: validationError.errors }, { status: 400 })`: Returns a JSON response explaining the invalid parameters and a `400 Bad Request` status code.
    *   `throw validationError`: If the error is not a ZodError, it's re-thrown to be caught by the outer `try...catch` block.

#### Parsing Query Parameters

```typescript
      const { searchParams } = new URL(request.url)
      const params = QueryParamsSchema.parse(Object.fromEntries(searchParams.entries()))
```

*   `const { searchParams } = new URL(request.url)`: Creates a `URL` object from the incoming request's URL. This object provides easy access to parts of the URL, including `searchParams` (the query string).
*   `Object.fromEntries(searchParams.entries())`: Converts the `URLSearchParams` object (which is an iterable of key-value pairs) into a plain JavaScript object. For example, `?limit=10&offset=0` becomes `{ limit: "10", offset: "0" }`.
*   `const params = QueryParamsSchema.parse(...)`: Uses the previously defined Zod schema to validate and parse the extracted query parameters. If validation fails, it throws a `ZodError` which is caught by the inner `try...catch` block. The `params` object now contains the validated and typed query parameters.

#### Conditional Column Selection (`selectColumns`)

```typescript
      const selectColumns =
        params.details === 'full'
          ? {
              id: workflowExecutionLogs.id,
              // ... (many columns)
              executionData: workflowExecutionLogs.executionData, // Large field - only in full mode
              files: workflowExecutionLogs.files, // Large field - only in full mode
              // ... (more columns from workflow table)
            }
          : {
              id: workflowExecutionLogs.id,
              // ... (same basic columns)
              executionData: sql<null>`NULL`, // Exclude large execution data in basic mode
              files: sql<null>`NULL`, // Exclude files in basic mode
              // ... (same workflow columns)
            }
```

This section dynamically determines which columns to select from the database based on the `details` parameter. This is a crucial performance optimization.

*   `params.details === 'full' ? { ... } : { ... }`: A ternary operator that conditionally defines the `selectColumns` object.
*   **If `params.details` is 'full'**:
    *   An object is created containing all relevant columns from `workflowExecutionLogs` and `workflow` tables.
    *   Notably, `workflowExecutionLogs.executionData` and `workflowExecutionLogs.files` are included. These fields can contain large JSON or text blobs, which are expensive to retrieve and transfer over the network.
*   **If `params.details` is 'basic' (or defaults to it)**:
    *   A similar object is created, but `executionData` and `files` are explicitly set to `sql<null>`NULL``.
    *   `sql<null>`NULL``: This Drizzle ORM syntax inserts the raw SQL keyword `NULL` into the `SELECT` statement for these columns. This ensures that the columns are still part of the result shape (preventing TypeScript errors) but their values are `null` directly from the database, without actually fetching potentially large data. This significantly reduces the database load and network payload for basic list views.
    *   All `workflow` related fields are included in both `basic` and `full` modes, as they provide essential context about the workflow itself.

#### Building the Base Database Query (`baseQuery`)

```typescript
      const baseQuery = db
        .select(selectColumns)
        .from(workflowExecutionLogs)
        .innerJoin(
          workflow,
          and(
            eq(workflowExecutionLogs.workflowId, workflow.id),
            eq(workflow.workspaceId, params.workspaceId)
          )
        )
        .innerJoin(
          permissions,
          and(
            eq(permissions.entityType, 'workspace'),
            eq(permissions.entityId, workflow.workspaceId),
            eq(permissions.userId, userId)
          )
        )
```

This constructs the initial part of the SQL query, including the tables to select from and the necessary joins.

*   `db.select(selectColumns)`: Starts a Drizzle query to select the columns defined in `selectColumns`.
*   `.from(workflowExecutionLogs)`: Specifies that the primary table for the query is `workflowExecutionLogs`.
*   `.innerJoin(...)`: Performs an `INNER JOIN` operation, combining rows from two or more tables based on a related column between them.
    *   **First `innerJoin` (with `workflow`)**:
        *   `workflow`: The table to join with.
        *   `and(eq(workflowExecutionLogs.workflowId, workflow.id), eq(workflow.workspaceId, params.workspaceId))`: The join condition. It ensures that:
            1.  The `workflowId` in `workflowExecutionLogs` matches the `id` in the `workflow` table.
            2.  The `workspaceId` of the `workflow` matches the `params.workspaceId` provided in the query. This ensures we only fetch logs for workflows within the specified workspace.
    *   **Second `innerJoin` (with `permissions`)**:
        *   `permissions`: The table to join with for authorization.
        *   `and(eq(permissions.entityType, 'workspace'), eq(permissions.entityId, workflow.workspaceId), eq(permissions.userId, userId))`: The join condition for authorization. It ensures that:
            1.  The permission `entityType` is 'workspace'.
            2.  The permission `entityId` matches the `workspaceId` of the workflow (from the previous join).
            3.  The `userId` in the `permissions` table matches the authenticated `userId`.
            This join effectively filters out logs for any workspace the authenticated user does not have permission to access.

#### Building Dynamic `WHERE` Conditions

```typescript
      let conditions: SQL | undefined

      // Filter by level
      if (params.level && params.level !== 'all') {
        conditions = and(conditions, eq(workflowExecutionLogs.level, params.level))
      }

      // Filter by specific workflow IDs
      if (params.workflowIds) {
        const workflowIds = params.workflowIds.split(',').filter(Boolean)
        if (workflowIds.length > 0) {
          conditions = and(conditions, inArray(workflow.id, workflowIds))
        }
      }

      // Filter by folder IDs
      if (params.folderIds) {
        const folderIds = params.folderIds.split(',').filter(Boolean)
        if (folderIds.length > 0) {
          conditions = and(conditions, inArray(workflow.folderId, folderIds))
        }
      }

      // Filter by triggers
      if (params.triggers) {
        const triggers = params.triggers.split(',').filter(Boolean)
        if (triggers.length > 0 && !triggers.includes('all')) {
          conditions = and(conditions, inArray(workflowExecutionLogs.trigger, triggers))
        }
      }

      // Filter by date range
      if (params.startDate) {
        conditions = and(
          conditions,
          gte(workflowExecutionLogs.startedAt, new Date(params.startDate))
        )
      }
      if (params.endDate) {
        conditions = and(conditions, lte(workflowExecutionLogs.startedAt, new Date(params.endDate)))
      }

      // Filter by search query
      if (params.search) {
        const searchTerm = `%${params.search}%`
        // With message removed, restrict search to executionId only
        conditions = and(conditions, sql`${workflowExecutionLogs.executionId} ILIKE ${searchTerm}`)
      }

      // Filter by workflow name (from advanced search input)
      if (params.workflowName) {
        const nameTerm = `%${params.workflowName}%`
        conditions = and(conditions, sql`${workflow.name} ILIKE ${nameTerm}`)
      }

      // Filter by folder name (best-effort text match when present on workflows)
      if (params.folderName) {
        const folderTerm = `%${params.folderName}%`
        conditions = and(conditions, sql`${workflow.name} ILIKE ${folderTerm}`)
      }
```

This section dynamically builds a `WHERE` clause based on the provided query parameters.

*   `let conditions: SQL | undefined`: Initializes a variable to store the combined SQL conditions. It's `undefined` initially because there might be no filters.
*   **Logic for each filter parameter**: Each `if` block checks if a specific parameter is provided and then appends a new condition to the `conditions` variable.
    *   `and(conditions, newCondition)`: This is key. The `and` helper function from Drizzle allows you to chain conditions. If `conditions` is `undefined`, `and(undefined, newCondition)` effectively just returns `newCondition`. If `conditions` already has a value, it logically `AND`s the `newCondition` with the existing `conditions`.
    *   **Level, Workflow IDs, Folder IDs, Triggers**:
        *   `split(',')`: Splits the comma-separated string into an array.
        *   `.filter(Boolean)`: Removes any empty strings from the array (e.g., if there's a trailing comma).
        *   `inArray()`: Used to filter a column by multiple values (e.g., `workflow.id IN ('id1', 'id2')`).
    *   **Date Range (`startDate`, `endDate`)**:
        *   `new Date(params.startDate)`: Converts the date string to a `Date` object, which Drizzle can compare against `startedAt` (assumed to be a `Date` type in the database).
        *   `gte()` (greater than or equal to) and `lte()` (less than or equal to) are used for date comparisons.
    *   **Search, Workflow Name, Folder Name**:
        *   `const searchTerm = `%${params.search}%``: Creates a wildcard search term. The `%` is a SQL wildcard character matching any sequence of characters.
        *   `sql`${workflowExecutionLogs.executionId} ILIKE ${searchTerm}``: Uses a Drizzle `sql` template literal to perform a case-insensitive `ILIKE` comparison. This is often used for fuzzy text matching in PostgreSQL. The comment notes that the search is restricted to `executionId` now that `message` is removed. The `folderName` filter also uses `workflow.name`, which might imply folder names are embedded or derived from workflow names in this specific schema.

#### Executing the Query and Getting Count

```typescript
      // Execute the query using the optimized join
      const logs = await baseQuery
        .where(conditions)
        .orderBy(desc(workflowExecutionLogs.startedAt))
        .limit(params.limit)
        .offset(params.offset)

      // Get total count for pagination using the same join structure
      const countQuery = db
        .select({ count: sql<number>`count(*)` })
        .from(workflowExecutionLogs)
        .innerJoin(
          workflow,
          and(
            eq(workflowExecutionLogs.workflowId, workflow.id),
            eq(workflow.workspaceId, params.workspaceId)
          )
        )
        .innerJoin(
          permissions,
          and(
            eq(permissions.entityType, 'workspace'),
            eq(permissions.entityId, workflow.workspaceId),
            eq(permissions.userId, userId)
          )
        )
        .where(conditions)

      const countResult = await countQuery

      const count = countResult[0]?.count || 0
```

This part executes the main log retrieval query and a separate query to get the total count for pagination.

*   **`logs` Query Execution**:
    *   `await baseQuery`: Starts the execution of the query built earlier.
    *   `.where(conditions)`: Applies all the dynamically built `WHERE` conditions to the query. If `conditions` is `undefined`, no `WHERE` clause is added.
    *   `.orderBy(desc(workflowExecutionLogs.startedAt))`: Sorts the results by the `startedAt` timestamp in descending order (newest first).
    *   `.limit(params.limit)`: Limits the number of results returned (for pagination).
    *   `.offset(params.offset)`: Skips a certain number of results (for pagination).
    *   The result, `logs`, is an array of objects, where each object represents a log entry with the selected columns.

*   **`countQuery` (for Pagination)**:
    *   `db.select({ count: sql<number>`count(*)` })`: Starts a new query to select only the count of records. `sql<number>`count(*)` is Drizzle's way to express a raw `COUNT(*)` aggregate function, ensuring its type is `number`.
    *   `.from(...)`, `.innerJoin(...)`, `.where(conditions)`: Crucially, the `countQuery` uses **the exact same `FROM`, `JOIN`, and `WHERE` clauses** as the main `logs` query. This ensures that the total count accurately reflects the number of records that match all applied filters and permissions, even if only a subset of those records is returned by the `logs` query due to `limit` and `offset`.
    *   `const countResult = await countQuery`: Executes the count query. It will return an array, typically with one object like `[{ count: 123 }]`.
    *   `const count = countResult[0]?.count || 0`: Extracts the `count` value from the first (and usually only) element of the `countResult` array. It defaults to `0` if no results are found.

#### Post-Processing: Block Executions, Trace Spans, and Cost Summaries

This section contains functions and logic to further process `executionData` into more structured `traceSpans` and `cost` summaries. The comments indicate a transition where "Block executions are now extracted from trace spans instead of separate table." This suggests that the raw block execution data might be embedded within the `executionData` JSON blob or derived from a trace span structure.

```typescript
      // Block executions are now extracted from trace spans instead of separate table
      const blockExecutionsByExecution: Record<string, any[]> = {}

      // Create clean trace spans from block executions
      const createTraceSpans = (blockExecutions: any[]) => {
        return blockExecutions.map((block, index) => {
          // For error blocks, include error information in the output
          let output = block.outputData
          if (block.status === 'error' && block.errorMessage) {
            output = {
              ...output,
              error: block.errorMessage,
              stackTrace: block.errorStackTrace,
            }
          }

          return {
            id: block.id,
            name: `Block ${block.blockName || block.blockType} (${block.blockType})`,
            type: block.blockType,
            duration: block.durationMs,
            startTime: block.startedAt,
            endTime: block.endedAt,
            status: block.status === 'success' ? 'success' : 'error',
            blockId: block.blockId,
            input: block.inputData,
            output,
            tokens: block.cost?.tokens?.total || 0,
            relativeStartMs: index * 100,
            children: [],
            toolCalls: [],
          }
        })
      }

      // Extract cost information from block executions
      const extractCostSummary = (blockExecutions: any[]) => {
        let totalCost = 0
        let totalInputCost = 0
        let totalOutputCost = 0
        let totalTokens = 0
        let totalPromptTokens = 0
        let totalCompletionTokens = 0
        const models = new Map()

        blockExecutions.forEach((block) => {
          if (block.cost) {
            totalCost += Number(block.cost.total) || 0
            totalInputCost += Number(block.cost.input) || 0
            totalOutputCost += Number(block.cost.output) || 0
            totalTokens += block.cost.tokens?.total || 0
            totalPromptTokens += block.cost.tokens?.prompt || 0
            totalCompletionTokens += block.cost.tokens?.completion || 0

            // Track per-model costs
            if (block.cost.model) {
              if (!models.has(block.cost.model)) {
                models.set(block.cost.model, {
                  input: 0,
                  output: 0,
                  total: 0,
                  tokens: { prompt: 0, completion: 0, total: 0 },
                })
              }
              const modelCost = models.get(block.cost.model)
              modelCost.input += Number(block.cost.input) || 0
              modelCost.output += Number(block.cost.output) || 0
              modelCost.total += Number(block.cost.total) || 0
              modelCost.tokens.prompt += block.cost.tokens?.prompt || 0
              modelCost.tokens.completion += block.cost.tokens?.completion || 0
              modelCost.tokens.total += block.cost.tokens?.total || 0
            }
          }
        })

        return {
          total: totalCost,
          input: totalInputCost,
          output: totalOutputCost,
          tokens: {
            total: totalTokens,
            prompt: totalPromptTokens,
            completion: totalCompletionTokens,
          },
          models: Object.fromEntries(models), // Convert Map to object for JSON serialization
        }
      }
```

*   `const blockExecutionsByExecution: Record<string, any[]> = {}`: This object is declared, but in the current code flow, it remains an empty object. The comment "Block executions are now extracted from trace spans instead of separate table" implies that the source of this data has changed. If `blockExecutions` were fetched separately, they would populate this map. However, the subsequent `enhancedLogs` logic prioritizes `log.executionData.traceSpans` and `log.cost`, suggesting `blockExecutions` might be a fallback or an older approach, or is expected to be present within `executionData`.
*   `createTraceSpans = (blockExecutions: any[]) => { ... }`: A helper function that takes an array of "block execution" objects and transforms them into a standardized "trace span" format.
    *   It iterates over `blockExecutions` and maps each `block` to a new object representing a trace span.
    *   It standardizes fields like `name`, `type`, `duration`, `status`.
    *   If a block has an `error` status, it includes `errorMessage` and `errorStackTrace` in the `output` field.
    *   `relativeStartMs`: Provides a simple offset for visualization, implying a sequence of blocks.
    *   `children: [], toolCalls: []`: Placeholder for nested spans or tool calls, suggesting a hierarchical trace structure.
*   `extractCostSummary = (blockExecutions: any[]) => { ... }`: A helper function to aggregate cost information from an array of block executions.
    *   It initializes various cost and token counters.
    *   It iterates through `blockExecutions`, summing `total`, `input`, `output` costs, and `prompt`, `completion`, `total` tokens.
    *   `models = new Map()`: Uses a `Map` to track costs broken down by `model` (e.g., different LLM models), aggregating input/output costs and token counts for each model.
    *   `Object.fromEntries(models)`: Converts the `Map` into a plain object for easy JSON serialization.

#### Transforming Logs to Enhanced Format (`enhancedLogs`)

```typescript
      // Transform to clean log format with workflow data included
      const enhancedLogs = logs.map((log) => {
        const blockExecutions = blockExecutionsByExecution[log.executionId] || []

        // Only process trace spans and detailed cost in full mode
        let traceSpans = []
        let finalOutput: any
        let costSummary = (log.cost as any) || { total: 0 }

        if (params.details === 'full' && log.executionData) {
          // Use stored trace spans if available, otherwise create from block executions
          const storedTraceSpans = (log.executionData as any)?.traceSpans
          traceSpans =
            storedTraceSpans && Array.isArray(storedTraceSpans) && storedTraceSpans.length > 0
              ? storedTraceSpans
              : createTraceSpans(blockExecutions) // blockExecutions will be empty here if not populated elsewhere

          // Prefer stored cost JSON; otherwise synthesize from blocks
          costSummary =
            log.cost && Object.keys(log.cost as any).length > 0
              ? (log.cost as any)
              : extractCostSummary(blockExecutions) // blockExecutions will be empty here if not populated elsewhere

          // Include finalOutput if present on executionData
          try {
            const fo = (log.executionData as any)?.finalOutput
            if (fo !== undefined) finalOutput = fo
          } catch {}
        }

        const workflowSummary = {
          id: log.workflowId,
          name: log.workflowName,
          description: log.workflowDescription,
          color: log.workflowColor,
          folderId: log.workflowFolderId,
          userId: log.workflowUserId,
          workspaceId: log.workflowWorkspaceId,
          createdAt: log.workflowCreatedAt,
          updatedAt: log.workflowUpdatedAt,
        }

        return {
          id: log.id,
          workflowId: log.workflowId,
          executionId: params.details === 'full' ? log.executionId : undefined,
          level: log.level,
          duration: log.totalDurationMs ? `${log.totalDurationMs}ms` : null,
          trigger: log.trigger,
          createdAt: log.startedAt.toISOString(),
          files: params.details === 'full' ? log.files || undefined : undefined,
          workflow: workflowSummary,
          executionData:
            params.details === 'full'
              ? {
                  totalDuration: log.totalDurationMs,
                  traceSpans,
                  blockExecutions,
                  finalOutput,
                  enhanced: true,
                }
              : undefined,
          cost:
            params.details === 'full'
              ? (costSummary as any)
              : { total: (costSummary as any)?.total || 0 },
        }
      })
```

This section iterates through the `logs` fetched from the database and transforms them into a client-friendly, enhanced format, adding nested workflow details, trace spans, and cost summaries conditionally.

*   `logs.map((log) => { ... })`: Iterates over each raw `log` object retrieved from the database.
*   `const blockExecutions = blockExecutionsByExecution[log.executionId] || []`: Attempts to get block executions for the current log's `executionId`. As noted before, `blockExecutionsByExecution` is currently empty, so this will always be an empty array unless it's populated from another source (e.g., if `executionData` contains it, or if block executions were fetched in an earlier step, which is not shown here).
*   `let traceSpans = []; let finalOutput: any; let costSummary = (log.cost as any) || { total: 0 };`: Initializes variables for trace spans, final output, and cost. `costSummary` defaults to the existing `log.cost` or `{ total: 0 }`.
*   `if (params.details === 'full' && log.executionData)`: This is the core conditional logic for detailed information. Trace spans, detailed costs, and `finalOutput` are only processed and included if `details` is 'full' AND `executionData` exists on the log.
    *   `const storedTraceSpans = (log.executionData as any)?.traceSpans`: Tries to extract `traceSpans` directly from `log.executionData` (which is typically a JSONB field). This is the preferred source.
    *   `traceSpans = storedTraceSpans && ... ? storedTraceSpans : createTraceSpans(blockExecutions)`: If `storedTraceSpans` are found and valid, they are used. Otherwise, `createTraceSpans` is called (which, given `blockExecutions` is empty, would result in an empty array for `traceSpans` here).
    *   `costSummary = log.cost && ... ? (log.cost as any) : extractCostSummary(blockExecutions)`: Similarly, if `log.cost` (from the database) is present, it's used. Otherwise, `extractCostSummary` is called (which again would operate on an empty `blockExecutions` array here). This indicates a preference for pre-calculated/stored cost data in `log.cost`.
    *   `finalOutput`: Attempts to extract `finalOutput` from `log.executionData`.
*   `const workflowSummary = { ... }`: Creates a nested `workflow` object with relevant details extracted from the joined `workflow` table fields. This provides context about the workflow directly within each log entry.
*   **Returned Log Object**: Constructs the final log object for the API response.
    *   `id`, `workflowId`, `level`, `duration`, `trigger`, `createdAt`: Basic log properties.
    *   `executionId`: Only included if `details` is 'full'.
    *   `duration`: Formatted as `"${totalDurationMs}ms"`.
    *   `files`: Only included if `details` is 'full'.
    *   `workflow`: The `workflowSummary` object is nested here.
    *   `executionData`: Only included if `details` is 'full'. Contains `totalDuration`, `traceSpans`, `blockExecutions` (likely empty here), `finalOutput`, and an `enhanced: true` flag.
    *   `cost`: If `details` is 'full', the detailed `costSummary` is included. Otherwise, only the `total` cost is provided.

#### Final Response

```typescript
      return NextResponse.json(
        {
          data: enhancedLogs,
          total: Number(count),
          page: Math.floor(params.offset / params.limit) + 1,
          pageSize: params.limit,
          totalPages: Math.ceil(Number(count) / params.limit),
        },
        { status: 200 }
      )
```

This section constructs the final JSON response sent back to the client.

*   `return NextResponse.json(...)`: Creates and returns a `NextResponse` object with JSON body.
*   **JSON Body**:
    *   `data: enhancedLogs`: The array of transformed log objects.
    *   `total: Number(count)`: The total number of logs matching the filters, converted to a number.
    *   `page: Math.floor(params.offset / params.limit) + 1`: Calculates the current page number based on `offset` and `limit`.
    *   `pageSize: params.limit`: The number of items per page.
    *   `totalPages: Math.ceil(Number(count) / params.limit)`: Calculates the total number of pages available.
*   `{ status: 200 }`: Sets the HTTP status code to `200 OK`, indicating a successful request.

---

### Simplified Complex Logic

1.  **Conditional Data Retrieval (`details: 'basic' | 'full'`)**:
    *   **Problem:** Some log fields (like `executionData` and `files`) can be very large. Fetching them always would slow down the database and make API responses huge, especially for list views.
    *   **Solution:** The code explicitly checks the `details` query parameter.
        *   If `details=basic`, it tells the database to return `NULL` for these large fields. This means the database doesn't even *try* to read or send that data, saving significant resources for simple listing views.
        *   If `details=full`, it fetches all the data, as the client likely needs it for a detailed view of a single log.

2.  **Dynamic Query Building for Filters (`conditions`)**:
    *   **Problem:** Users can apply many different filters (by level, workflow, date, search, etc.). Hardcoding every combination in SQL would be impossible.
    *   **Solution:** The code starts with an empty set of conditions. For each filter parameter provided by the user, it adds a new SQL condition (e.g., `level = 'error'`, `workflowId IN ('x', 'y')`). These conditions are then chained together using `AND` operators. This creates a highly flexible query that adapts to exactly what the user wants to filter.

3.  **Database Joins for Authorization and Context**:
    *   **Problem:** Log entries (`workflowExecutionLogs`) don't contain all the necessary information directly, like workflow names or user permissions.
    *   **Solution:** The code uses `INNER JOIN` operations:
        *   It joins `workflowExecutionLogs` with the `workflow` table to get details about the workflow (name, description, folder, etc.) associated with each log.
        *   It then joins the result with a `permissions` table. This `permissions` table holds information about which users have access to which workspaces. By joining on the user's ID and the workspace ID, the database itself filters out any logs from workspaces the user isn't allowed to see, ensuring strong access control directly at the query level.

4.  **Post-Processing Execution Data (Trace Spans & Cost)**:
    *   **Problem:** Raw `executionData` or `blockExecutions` might be complex or in a raw format. Clients often need a standardized, hierarchical view of "trace spans" (steps of execution) and aggregated cost summaries.
    *   **Solution:**
        *   The `enhancedLogs` mapping iterates over each log.
        *   If `details=full`, it first tries to find pre-formatted `traceSpans` and `cost` directly within the `executionData` field or `log.cost` (prioritizing stored, potentially pre-computed data).
        *   If those aren't available, it falls back to helper functions (`createTraceSpans`, `extractCostSummary`). These functions take raw "block execution" data (which in this particular code is likely expected to be part of `executionData` or derived from it) and transform it into a more consumable format for UI display, including structured errors and per-model cost breakdowns.

In essence, this API route is a robust and optimized solution for querying complex workflow execution data, balancing detailed information with performance and strict access control.