This TypeScript file is a utility for constructing dynamic database queries, specifically for filtering and ordering workflow execution logs. It's designed to make it easy to build complex `WHERE` and `ORDER BY` clauses for a Drizzle ORM query based on a set of user-provided filters.

### Core Purpose of this File

Imagine you have a dashboard that displays "workflow execution logs" to users. Users want to be able to search, filter, and sort these logs by various criteria like:
*   Which workspace they belong to.
*   Specific workflows or folders.
*   When they started.
*   Their duration or cost.
*   Their status (info/error).
*   Even paginate through results using a "cursor" for efficient loading of the next page.

This file provides two key functions to handle these requirements:

1.  **`buildLogFilters`**: Takes a `LogFilters` object (containing all the user's filter choices) and generates a Drizzle ORM `SQL` expression that represents the `WHERE` clause of your database query. This `WHERE` clause will dynamically include only the conditions for the filters the user has actually applied.
2.  **`getOrderBy`**: Takes an `order` parameter (descending or ascending) and generates a Drizzle ORM `SQL` expression for the `ORDER BY` clause, typically sorting by the `startedAt` timestamp.

In essence, this file acts as a flexible, type-safe query builder for the filtering and sorting aspects of retrieving workflow execution logs, abstracting away the complexities of constructing SQL conditions.

---

### Code Explanation

Let's break down the code section by section.

#### Imports

```typescript
import { workflow, workflowExecutionLogs } from '@sim/db/schema'
import { and, desc, eq, gte, inArray, lte, type SQL, sql } from 'drizzle-orm'
```

*   **`workflow`, `workflowExecutionLogs`**: These are Drizzle ORM schema objects imported from `@sim/db/schema`.
    *   `workflow`: Represents the database table definition for individual workflows.
    *   `workflowExecutionLogs`: Represents the database table definition for the logs of each workflow execution.
    *   These objects allow Drizzle to understand the structure of your database tables and construct queries against them in a type-safe manner.

*   **`and`, `desc`, `eq`, `gte`, `inArray`, `lte`, `type SQL`, `sql`**: These are utility functions and types imported from `drizzle-orm`. They are the building blocks for constructing SQL expressions using Drizzle's query builder.
    *   `and`: Combines multiple conditions with a logical `AND`.
    *   `desc`: Specifies descending order for an `ORDER BY` clause.
    *   `eq`: Checks for equality (`=`).
    *   `gte`: Checks for "greater than or equal to" (`>=`).
    *   `inArray`: Checks if a column's value is present in a given array (`IN (...)`).
    *   `lte`: Checks for "less than or equal to" (`<=`).
    *   `type SQL`: A Drizzle type representing a raw SQL expression. Used for type-hinting.
    *   `sql`: A template literal tag function that allows you to write raw SQL snippets directly within Drizzle queries. This is particularly useful for advanced operations not directly supported by Drizzle's fluent API (like JSONB operations or composite row comparisons).

---

#### `LogFilters` Interface

```typescript
export interface LogFilters {
  workspaceId: string
  workflowIds?: string[]
  folderIds?: string[]
  triggers?: string[]
  level?: 'info' | 'error'
  startDate?: Date
  endDate?: Date
  executionId?: string
  minDurationMs?: number
  maxDurationMs?: number
  minCost?: number
  maxCost?: number
  model?: string
  cursor?: {
    startedAt: string
    id: string
  }
  order?: 'desc' | 'asc'
}
```

This interface defines the shape of the object that will hold all possible filtering and pagination criteria. It acts as the input contract for the `buildLogFilters` function.

*   **`workspaceId: string`**: **Required**. The unique identifier of the workspace to which the logs belong. This is crucial for multi-tenant applications to ensure users only see their own data.
*   **`workflowIds?: string[]`**: An optional array of workflow IDs. If provided, only logs from these specific workflows will be returned. `?` denotes it's optional.
*   **`folderIds?: string[]`**: An optional array of folder IDs. If provided, only logs from workflows within these specific folders will be returned.
*   **`triggers?: string[]`**: An optional array of trigger types (e.g., 'manual', 'schedule'). If provided, logs will be filtered by these trigger types.
*   **`level?: 'info' | 'error'`**: An optional filter to show only 'info' level logs or 'error' level logs.
*   **`startDate?: Date`**: An optional `Date` object. If provided, only logs that started *on or after* this date will be included.
*   **`endDate?: Date`**: An optional `Date` object. If provided, only logs that started *on or before* this date will be included.
*   **`executionId?: string`**: An optional specific execution ID. If provided, only the log for this exact execution ID will be returned.
*   **`minDurationMs?: number`**: An optional minimum duration in milliseconds. Logs with `totalDurationMs` less than this value will be excluded.
*   **`maxDurationMs?: number`**: An optional maximum duration in milliseconds. Logs with `totalDurationMs` greater than this value will be excluded.
*   **`minCost?: number`**: An optional minimum cost. This filters based on a `total` cost value that is likely stored within a JSONB column.
*   **`maxCost?: number`**: An optional maximum cost. Similar to `minCost`, but for the upper bound.
*   **`model?: string`**: An optional filter for specific models. This likely checks if a given model identifier exists within a `models` array or object inside a JSONB `cost` column.
*   **`cursor?: { startedAt: string; id: string }`**: This is for **cursor-based pagination**. Instead of traditional `OFFSET` pagination (which can be slow for large datasets), cursor pagination uses the values of the last item from the previous page to determine where the next page should start.
    *   `startedAt`: The `startedAt` timestamp of the last item seen on the previous page.
    *   `id`: The unique ID of the last item seen on the previous page (used as a tie-breaker if `startedAt` values are identical).
*   **`order?: 'desc' | 'asc'`**: The desired sort order for the results, either descending (newest first) or ascending (oldest first).

---

#### `buildLogFilters` Function

```typescript
export function buildLogFilters(filters: LogFilters): SQL<unknown> {
  const conditions: SQL<unknown>[] = []

  // Required: workspace and permissions check
  conditions.push(eq(workflow.workspaceId, filters.workspaceId))

  // Cursor-based pagination
  if (filters.cursor) {
    const cursorDate = new Date(filters.cursor.startedAt)
    if (filters.order === 'desc') {
      conditions.push(
        sql`(${workflowExecutionLogs.startedAt}, ${workflowExecutionLogs.id}) < (${cursorDate}, ${filters.cursor.id})`
      )
    } else {
      conditions.push(
        sql`(${workflowExecutionLogs.startedAt}, ${workflowExecutionLogs.id}) > (${cursorDate}, ${filters.cursor.id})`
      )
    }
  }

  // Workflow IDs filter
  if (filters.workflowIds && filters.workflowIds.length > 0) {
    conditions.push(inArray(workflow.id, filters.workflowIds))
  }

  // Folder IDs filter
  if (filters.folderIds && filters.folderIds.length > 0) {
    conditions.push(inArray(workflow.folderId, filters.folderIds))
  }

  // Triggers filter
  if (filters.triggers && filters.triggers.length > 0 && !filters.triggers.includes('all')) {
    conditions.push(inArray(workflowExecutionLogs.trigger, filters.triggers))
  }

  // Level filter
  if (filters.level) {
    conditions.push(eq(workflowExecutionLogs.level, filters.level))
  }

  // Date range filters
  if (filters.startDate) {
    conditions.push(gte(workflowExecutionLogs.startedAt, filters.startDate))
  }

  if (filters.endDate) {
    conditions.push(lte(workflowExecutionLogs.startedAt, filters.endDate))
  }

  // Search filter (execution ID)
  if (filters.executionId) {
    conditions.push(eq(workflowExecutionLogs.executionId, filters.executionId))
  }

  // Duration filters
  if (filters.minDurationMs !== undefined) {
    conditions.push(gte(workflowExecutionLogs.totalDurationMs, filters.minDurationMs))
  }

  if (filters.maxDurationMs !== undefined) {
    conditions.push(lte(workflowExecutionLogs.totalDurationMs, filters.maxDurationMs))
  }

  // Cost filters
  if (filters.minCost !== undefined) {
    conditions.push(sql`(${workflowExecutionLogs.cost}->>'total')::numeric >= ${filters.minCost}`)
  }

  if (filters.maxCost !== undefined) {
    conditions.push(sql`(${workflowExecutionLogs.cost}->>'total')::numeric <= ${filters.maxCost}`)
  }

  // Model filter
  if (filters.model) {
    conditions.push(sql`${workflowExecutionLogs.cost}->>'models' ? ${filters.model}`)
  }

  // Combine all conditions with AND
  return conditions.length > 0 ? and(...conditions)! : sql`true`
}
```

This function takes a `LogFilters` object and returns a `SQL<unknown>` object, which is a Drizzle-compatible representation of a SQL `WHERE` clause.

1.  **`const conditions: SQL<unknown>[] = []`**:
    *   An empty array `conditions` is initialized. This array will store individual Drizzle SQL expressions, each representing a single filter condition.

2.  **`conditions.push(eq(workflow.workspaceId, filters.workspaceId))`**:
    *   This is the **first and mandatory condition**. It ensures that all retrieved logs belong to the specified `workspaceId`. This is a fundamental security and data isolation measure in multi-tenant applications.
    *   `eq(workflow.workspaceId, filters.workspaceId)`: Creates an equality comparison (`workflow.workspaceId = '...'`).

3.  **Cursor-based pagination**:
    ```typescript
    if (filters.cursor) {
      const cursorDate = new Date(filters.cursor.startedAt)
      if (filters.order === 'desc') {
        conditions.push(
          sql`(${workflowExecutionLogs.startedAt}, ${workflowExecutionLogs.id}) < (${cursorDate}, ${filters.cursor.id})`
        )
      } else {
        conditions.push(
          sql`(${workflowExecutionLogs.startedAt}, ${workflowExecutionLogs.id}) > (${cursorDate}, ${filters.cursor.id})`
        )
      }
    }
    ```
    *   This block handles cursor pagination. If `filters.cursor` is provided, it means we're fetching a subsequent page.
    *   `cursorDate = new Date(filters.cursor.startedAt)`: Converts the `startedAt` string from the cursor into a `Date` object for comparison.
    *   `if (filters.order === 'desc')`: If sorting in descending order (newest first), we want records *older* than the cursor.
        *   `sql`...`<...` (composite row comparison): This is a PostgreSQL feature that compares two rows element by element.
            *   It checks `(startedAt, id) < (cursorDate, cursorId)`. This means "give me all rows where `startedAt` is earlier than `cursorDate`, OR `startedAt` is the same as `cursorDate` AND `id` is less than `cursorId`." This handles tie-breaking with `id` to ensure consistent and unique pagination.
    *   `else` (ascending order): If sorting in ascending order (oldest first), we want records *newer* than the cursor.
        *   `sql`...`>...`: Similar composite comparison, but checks for `(startedAt, id) > (cursorDate, cursorId)`.

4.  **Workflow IDs filter**:
    ```typescript
    if (filters.workflowIds && filters.workflowIds.length > 0) {
      conditions.push(inArray(workflow.id, filters.workflowIds))
    }
    ```
    *   Checks if `workflowIds` are provided and the array is not empty.
    *   `inArray(workflow.id, filters.workflowIds)`: Generates a SQL `IN` clause (e.g., `workflow.id IN ('id1', 'id2')`).

5.  **Folder IDs filter**:
    ```typescript
    if (filters.folderIds && filters.folderIds.length > 0) {
      conditions.push(inArray(workflow.folderId, filters.folderIds))
    }
    ```
    *   Similar to `workflowIds`, but filters based on the `folderId` associated with the workflow.

6.  **Triggers filter**:
    ```typescript
    if (filters.triggers && filters.triggers.length > 0 && !filters.triggers.includes('all')) {
      conditions.push(inArray(workflowExecutionLogs.trigger, filters.triggers))
    }
    ```
    *   Filters `workflowExecutionLogs.trigger` by a list of trigger types.
    *   `!filters.triggers.includes('all')`: A special condition where if the list of triggers explicitly includes `'all'`, it means no trigger filtering should be applied. This is a common UI pattern where 'all' means "don't filter by this criterion."

7.  **Level filter**:
    ```typescript
    if (filters.level) {
      conditions.push(eq(workflowExecutionLogs.level, filters.level))
    }
    ```
    *   If `filters.level` is provided (e.g., 'info' or 'error'), it adds an equality condition (`workflowExecutionLogs.level = 'info'`).

8.  **Date range filters**:
    ```typescript
    if (filters.startDate) {
      conditions.push(gte(workflowExecutionLogs.startedAt, filters.startDate))
    }

    if (filters.endDate) {
      conditions.push(lte(workflowExecutionLogs.startedAt, filters.endDate))
    }
    ```
    *   `gte(workflowExecutionLogs.startedAt, filters.startDate)`: Adds `workflowExecutionLogs.startedAt >= filters.startDate`.
    *   `lte(workflowExecutionLogs.startedAt, filters.endDate)`: Adds `workflowExecutionLogs.startedAt <= filters.endDate`.

9.  **Search filter (execution ID)**:
    ```typescript
    if (filters.executionId) {
      conditions.push(eq(workflowExecutionLogs.executionId, filters.executionId))
    }
    ```
    *   Filters for an exact `executionId`.

10. **Duration filters**:
    ```typescript
    if (filters.minDurationMs !== undefined) {
      conditions.push(gte(workflowExecutionLogs.totalDurationMs, filters.minDurationMs))
    }

    if (filters.maxDurationMs !== undefined) {
      conditions.push(lte(workflowExecutionLogs.totalDurationMs, filters.maxDurationMs))
    }
    ```
    *   Filters logs based on their `totalDurationMs`.
    *   `!== undefined` is used instead of just `if (filters.minDurationMs)` because `0` is a valid minimum/maximum duration, and `0` is a falsy value in JavaScript. Using `!== undefined` ensures that `0` is still considered a valid filter.

11. **Cost filters**:
    ```typescript
    if (filters.minCost !== undefined) {
      conditions.push(sql`(${workflowExecutionLogs.cost}->>'total')::numeric >= ${filters.minCost}`)
    }

    if (filters.maxCost !== undefined) {
      conditions.push(sql`(${workflowExecutionLogs.cost}->>'total')::numeric <= ${filters.maxCost}`)
    }
    ```
    *   These filters operate on the `cost` column, which is likely a PostgreSQL `JSONB` type.
    *   `workflowExecutionLogs.cost->>'total'`: This is PostgreSQL syntax to extract the value associated with the key `'total'` from the `cost` JSONB column as *text*.
    *   `::numeric`: Casts the extracted text value to a `numeric` type so it can be compared with `filters.minCost` or `filters.maxCost` (which are numbers).
    *   `sql` template literal: Used here because Drizzle's fluent API doesn't directly support all PostgreSQL JSONB operators for type-safe comparisons.

12. **Model filter**:
    ```typescript
    if (filters.model) {
      conditions.push(sql`${workflowExecutionLogs.cost}->>'models' ? ${filters.model}`)
    }
    ```
    *   Another JSONB filter.
    *   `workflowExecutionLogs.cost->>'models' ? ${filters.model}`: This PostgreSQL operator (`?`) checks if a specific *key* or *string* (`filters.model`) exists within a JSONB object or array at the path specified (`cost->>'models'`). This implies that `models` within the `cost` JSONB column likely stores a list of model identifiers.

13. **Combine all conditions with `AND`**:
    ```typescript
    return conditions.length > 0 ? and(...conditions)! : sql`true`
    ```
    *   After collecting all relevant conditions, this line combines them:
        *   `conditions.length > 0`: If there are any conditions (which there will be, at least the `workspaceId` one).
        *   `and(...conditions)`: The `and` function from Drizzle takes multiple SQL expressions and combines them into a single `AND` clause (e.g., `condition1 AND condition2 AND ...`).
        *   `!`: The non-null assertion operator. Since `conditions` will always have at least `workspaceId` condition, `and(...conditions)` will never return null or undefined, so we assert it.
        *   `sql`true``: If for some reason `conditions` is empty (e.g., if the `workspaceId` condition was removed and no other filters were applied), `sql`true`` is returned. This effectively means "no WHERE clause" or "the WHERE clause is always true," allowing the query to return all records (subject to other joins/limits).

---

#### `getOrderBy` Function

```typescript
export function getOrderBy(order: 'desc' | 'asc' = 'desc') {
  return order === 'desc'
    ? desc(workflowExecutionLogs.startedAt)
    : sql`${workflowExecutionLogs.startedAt} ASC`
}
```

This function generates the `ORDER BY` clause for the database query.

*   **`order: 'desc' | 'asc' = 'desc'`**:
    *   Takes an `order` parameter, which can only be `'desc'` or `'asc'`.
    *   It has a default value of `'desc'`, meaning if no order is specified, results will be sorted in descending order (newest logs first).

*   **`order === 'desc' ? desc(workflowExecutionLogs.startedAt) : sql`${workflowExecutionLogs.startedAt} ASC``**:
    *   This is a ternary operator that returns a different Drizzle SQL expression based on the `order` parameter:
        *   If `order` is `'desc'`: It returns `desc(workflowExecutionLogs.startedAt)`, which translates to `ORDER BY "workflow_execution_logs"."started_at" DESC`.
        *   If `order` is `'asc'`: It returns `sql`${workflowExecutionLogs.startedAt} ASC``, which translates to `ORDER BY "workflow_execution_logs"."started_at" ASC`. While Drizzle also has an `asc` function, using `sql` here demonstrates another way to achieve the same result.

---

### How These Functions Are Used Together

These functions would typically be used within a Drizzle query like this:

```typescript
import { db } from '@sim/db'; // Your Drizzle database instance
import { workflow, workflowExecutionLogs } from '@sim/db/schema';
import { eq } from 'drizzle-orm';
import { buildLogFilters, getOrderBy, LogFilters } from './workflow-filters'; // This file

async function fetchWorkflowLogs(filters: LogFilters, limit: number = 20) {
  const whereClause = buildLogFilters(filters);
  const orderByClause = getOrderBy(filters.order);

  const query = db
    .select()
    .from(workflowExecutionLogs)
    // You might need a join here if your filters depend on the 'workflow' table
    // For example, if 'workflow.folderId' is used in filters, you need to join
    .leftJoin(workflow, eq(workflow.id, workflowExecutionLogs.workflowId))
    .where(whereClause)
    .orderBy(orderByClause)
    .limit(limit);

  return query;
}

// Example usage:
const myFilters: LogFilters = {
  workspaceId: 'abc-123',
  level: 'error',
  startDate: new Date('2023-01-01'),
  order: 'desc',
  // ... other filters
};

// To get the next page:
const previousPageLastItem = { startedAt: '2023-11-15T10:00:00Z', id: 'some-last-id' };
const nextPageFilters: LogFilters = {
  ...myFilters,
  cursor: previousPageLastItem,
};

// In an API endpoint:
// const logs = await fetchWorkflowLogs(myFilters, 20);
// const nextPageLogs = await fetchWorkflowLogs(nextPageFilters, 20);
```

This setup provides a robust, type-safe, and highly flexible way to query workflow execution logs in a dynamic and efficient manner, especially with the inclusion of cursor-based pagination for large datasets.