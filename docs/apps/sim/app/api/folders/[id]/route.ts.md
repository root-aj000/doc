This file defines API endpoints for managing `workflowFolder` resources within a Next.js application. Specifically, it handles updating an existing folder (`PUT` request) and deleting a folder along with its entire contents (`DELETE` request). It ensures proper user authentication, authorization, and data integrity checks (like preventing circular folder references).

## Detailed Explanation

Let's break down the code section by section:

### 1. Imports

This section brings in necessary functions, types, and database schema definitions from various parts of the application and external libraries.

```typescript
import { db } from '@sim/db'
import { workflow, workflowFolder } from '@sim/db/schema'
import { and, eq } from 'drizzle-orm'
import { type NextRequest, NextResponse } from 'next/server'
import { getSession } from '@/lib/auth'
import { createLogger } from '@/lib/logs/console/logger'
import { getUserEntityPermissions } from '@/lib/permissions/utils'
```

*   **`import { db } from '@sim/db'`**: Imports the Drizzle ORM database client instance. This `db` object is used to interact with your PostgreSQL (or other supported) database.
*   **`import { workflow, workflowFolder } from '@sim/db/schema'`**: Imports table schema definitions for `workflow` and `workflowFolder` from your Drizzle schema. These objects represent your database tables and are used to construct queries.
*   **`import { and, eq } from 'drizzle-orm'`**: Imports helper functions from Drizzle ORM for building complex `WHERE` clauses in database queries:
    *   `eq`: Checks if two values are equal (`column = 'value'`).
    *   `and`: Combines multiple conditions with a logical AND (`condition1 AND condition2`).
*   **`import { type NextRequest, NextResponse } from 'next/server'`**: Imports types and classes from Next.js server-side utilities:
    *   `NextRequest`: Represents the incoming HTTP request object.
    *   `NextResponse`: Used to construct the HTTP response object.
*   **`import { getSession } from '@/lib/auth'`**: Imports a utility function to retrieve the current user's session information. This is crucial for authentication and authorization.
*   **`import { createLogger } from '@/lib/logs/console/logger'`**: Imports a function to create a logger instance for structured logging within the application.
*   **`import { getUserEntityPermissions } from '@/lib/permissions/utils'`**: Imports a utility function to check a user's permissions for a specific entity (like a workspace).

### 2. Logger Initialization

```typescript
const logger = createLogger('FoldersIDAPI')
```

*   **`const logger = createLogger('FoldersIDAPI')`**: Creates a new logger instance specifically named `'FoldersIDAPI'`. This helps in tracing logs back to the specific part of the application that generated them.

### 3. PUT Endpoint: Update a Folder

This `PUT` function handles `PUT` requests to `/api/folders/[id]`, allowing clients to update properties of a specific `workflowFolder`.

```typescript
// PUT - Update a folder
export async function PUT(request: NextRequest, { params }: { params: Promise<{ id: string }> }) {
  try {
    // ... (rest of the function)
  } catch (error) {
    logger.error('Error updating folder:', { error })
    return NextResponse.json({ error: 'Internal server error' }, { status: 500 })
  }
}
```

*   **`export async function PUT(...)`**: Defines an asynchronous function named `PUT`. In Next.js App Router, an `async` function exported with an HTTP method name (`GET`, `POST`, `PUT`, `DELETE`, etc.) in a `route.ts` file automatically becomes an API endpoint handler for that method.
*   **`request: NextRequest`**: The first argument is the incoming request object, containing details like headers, body, and URL.
*   **`{ params }: { params: Promise<{ id: string }> }`**: The second argument is an object containing route parameters.
    *   `params` is an object where keys correspond to dynamic segments in the route path (e.g., `[id]` in `/folders/[id]`).
    *   The `Promise<{ id: string }>` type indicates that the `params` object, specifically `id`, will be resolved asynchronously by Next.js.

#### Inside the `try` block:

```typescript
    const session = await getSession()
    if (!session?.user?.id) {
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
    }
```

*   **`const session = await getSession()`**: Attempts to retrieve the current user's session.
*   **`if (!session?.user?.id)`**: Checks if a valid user session exists. If not (no session or no user ID in the session), it means the user is not authenticated.
*   **`return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })`**: Sends a 401 Unauthorized response, indicating the user must log in.

```typescript
    const { id } = await params
    const body = await request.json()
    const { name, color, isExpanded, parentId } = body
```

*   **`const { id } = await params`**: Extracts the `id` of the folder to be updated from the route parameters.
*   **`const body = await request.json()`**: Parses the JSON body of the incoming request, which contains the update data.
*   **`const { name, color, isExpanded, parentId } = body`**: Destructures specific fields (`name`, `color`, `isExpanded`, `parentId`) from the request body. These are the fields a user can update.

```typescript
    // Verify the folder exists
    const existingFolder = await db
      .select()
      .from(workflowFolder)
      .where(eq(workflowFolder.id, id))
      .then((rows) => rows[0])

    if (!existingFolder) {
      return NextResponse.json({ error: 'Folder not found' }, { status: 404 })
    }
```

*   **`const existingFolder = await db.select()...`**: Queries the database to find the `workflowFolder` with the given `id`.
    *   `db.select()`: Starts a select query. Without specific columns, it selects all columns (`*`).
    *   `.from(workflowFolder)`: Specifies the table to query.
    *   `.where(eq(workflowFolder.id, id))`: Filters the results to only include the row where the `id` column matches the `id` from the request.
    *   `.then((rows) => rows[0])`: Drizzle queries return an array of results. This common pattern extracts the first element if it exists, assuming `id` is a primary key and will return at most one row.
*   **`if (!existingFolder)`**: If no folder is found with the provided `id`, it means the folder doesn't exist.
*   **`return NextResponse.json({ error: 'Folder not found' }, { status: 404 })`**: Sends a 404 Not Found response.

```typescript
    // Check if user has write permissions for the workspace
    const workspacePermission = await getUserEntityPermissions(
      session.user.id,
      'workspace',
      existingFolder.workspaceId
    )

    if (!workspacePermission || workspacePermission === 'read') {
      return NextResponse.json(
        { error: 'Write access required to update folders' },
        { status: 403 }
      )
    }
```

*   **`const workspacePermission = await getUserEntityPermissions(...)`**: Calls the permission utility to check the authenticated user's permissions for the workspace that the `existingFolder` belongs to.
    *   `session.user.id`: The ID of the authenticated user.
    *   `'workspace'`: The type of entity being checked.
    *   `existingFolder.workspaceId`: The ID of the specific workspace.
*   **`if (!workspacePermission || workspacePermission === 'read')`**: Checks if the user has no permissions or only 'read' access to the workspace. To update a folder, 'write' or 'admin' access is typically required.
*   **`return NextResponse.json({ error: 'Write access required...' }, { status: 403 })`**: Sends a 403 Forbidden response, indicating the user does not have sufficient permissions.

```typescript
    // Prevent setting a folder as its own parent or creating circular references
    if (parentId && parentId === id) {
      return NextResponse.json({ error: 'Folder cannot be its own parent' }, { status: 400 })
    }
```

*   **`if (parentId && parentId === id)`**: Basic validation to prevent a folder from being set as its own parent. This is a common logical error and would break folder hierarchies.
*   **`return NextResponse.json({ error: 'Folder cannot be its own parent' }, { status: 400 })`**: Sends a 400 Bad Request response.

```typescript
    // Check for circular references if parentId is provided
    if (parentId) {
      const wouldCreateCycle = await checkForCircularReference(id, parentId)
      if (wouldCreateCycle) {
        return NextResponse.json(
          { error: 'Cannot create circular folder reference' },
          { status: 400 }
        )
      }
    }
```

*   **`if (parentId)`**: This check is only relevant if the `parentId` is being updated.
*   **`const wouldCreateCycle = await checkForCircularReference(id, parentId)`**: Calls a helper function (explained later) to determine if setting the current folder (`id`) under the proposed `parentId` would create a circular reference in the folder hierarchy (e.g., Folder A -> Folder B -> Folder C -> Folder A).
*   **`if (wouldCreateCycle)`**: If a circular reference is detected.
*   **`return NextResponse.json({ error: 'Cannot create circular...' }, { status: 400 })`**: Sends a 400 Bad Request response.

```typescript
    // Update the folder
    const updates: any = { updatedAt: new Date() }
    if (name !== undefined) updates.name = name.trim()
    if (color !== undefined) updates.color = color
    if (isExpanded !== undefined) updates.isExpanded = isExpanded
    if (parentId !== undefined) updates.parentId = parentId || null
```

*   **`const updates: any = { updatedAt: new Date() }`**: Initializes an object `updates`. The `updatedAt` timestamp is always updated. The `any` type is used here for convenience, as properties are conditionally added. In a stricter setup, you might build this object more explicitly.
*   **`if (name !== undefined) updates.name = name.trim()`**: If `name` was provided in the request body, it's added to the `updates` object after `trim()`ming any leading/trailing whitespace.
*   **`if (color !== undefined) updates.color = color`**: If `color` was provided, it's added.
*   **`if (isExpanded !== undefined) updates.isExpanded = isExpanded`**: If `isExpanded` was provided, it's added.
*   **`if (parentId !== undefined) updates.parentId = parentId || null`**: If `parentId` was provided, it's added. The `|| null` ensures that if `parentId` is explicitly sent as `null` (e.g., to unassign a parent), it's stored as `null` in the database, which is typically how nullable foreign keys are handled.

```typescript
    const [updatedFolder] = await db
      .update(workflowFolder)
      .set(updates)
      .where(eq(workflowFolder.id, id))
      .returning()
```

*   **`await db.update(workflowFolder)`**: Initiates an update query on the `workflowFolder` table.
*   **`.set(updates)`**: Sets the columns to be updated with the values from the `updates` object.
*   **`.where(eq(workflowFolder.id, id))`**: Specifies that only the folder matching the `id` should be updated.
*   **`.returning()`**: This Drizzle method causes the `UPDATE` query to return the updated rows.
*   **`const [updatedFolder]`**: Drizzle's `returning()` method returns an array of updated rows. This syntax destructures the first (and in this case, only) updated row into `updatedFolder`.

```typescript
    logger.info('Updated folder:', { id, updates })
    return NextResponse.json({ folder: updatedFolder })
```

*   **`logger.info('Updated folder:', { id, updates })`**: Logs a success message with the ID of the updated folder and the changes made.
*   **`return NextResponse.json({ folder: updatedFolder })`**: Sends a 200 OK response with the `updatedFolder` object in the JSON body.

#### Inside the `catch` block:

```typescript
  } catch (error) {
    logger.error('Error updating folder:', { error })
    return NextResponse.json({ error: 'Internal server error' }, { status: 500 })
  }
```

*   **`catch (error)`**: Catches any unexpected errors that occur during the execution of the `try` block.
*   **`logger.error('Error updating folder:', { error })`**: Logs the error details for debugging.
*   **`return NextResponse.json({ error: 'Internal server error' }, { status: 500 })`**: Sends a generic 500 Internal Server Error response to the client, hiding sensitive error details.

### 4. DELETE Endpoint: Delete a Folder and Its Contents

This `DELETE` function handles `DELETE` requests to `/api/folders/[id]`, allowing clients to delete a specific `workflowFolder` and all associated data.

```typescript
// DELETE - Delete a folder and all its contents
export async function DELETE(
  request: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  try {
    // ... (rest of the function)
  } catch (error) {
    logger.error('Error deleting folder:', { error })
    return NextResponse.json({ error: 'Internal server error' }, { status: 500 })
  }
}
```

*   **`export async function DELETE(...)`**: Similar to `PUT`, this defines the handler for `DELETE` requests.
*   Parameters (`request`, `params`) are identical to the `PUT` function.

#### Inside the `try` block:

```typescript
    const session = await getSession()
    if (!session?.user?.id) {
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
    }
```

*   **Authentication check**: Identical to the `PUT` endpoint, ensures the user is logged in.

```typescript
    const { id } = await params

    // Verify the folder exists
    const existingFolder = await db
      .select()
      .from(workflowFolder)
      .where(eq(workflowFolder.id, id))
      .then((rows) => rows[0])

    if (!existingFolder) {
      return NextResponse.json({ error: 'Folder not found' }, { status: 404 })
    }
```

*   **Extract `id` and Folder Existence Check**: Identical to the `PUT` endpoint, extracts the folder ID and verifies that the folder exists.

```typescript
    // Check if user has admin permissions for the workspace (admin-only for deletions)
    const workspacePermission = await getUserEntityPermissions(
      session.user.id,
      'workspace',
      existingFolder.workspaceId
    )

    if (workspacePermission !== 'admin') {
      return NextResponse.json(
        { error: 'Admin access required to delete folders' },
        { status: 403 }
      )
    }
```

*   **Permission Check for Deletion**: This is stricter than the `PUT` endpoint.
    *   `if (workspacePermission !== 'admin')`: For deletions, only users with 'admin' access to the workspace are allowed. This prevents accidental data loss from users with lesser permissions.
*   **`return NextResponse.json({ error: 'Admin access required...' }, { status: 403 })`**: Sends a 403 Forbidden response if the user is not an admin.

```typescript
    // Recursively delete folder and all its contents
    const deletionStats = await deleteFolderRecursively(id, existingFolder.workspaceId)

    logger.info('Deleted folder and all contents:', {
      id,
      deletionStats,
    })

    return NextResponse.json({
      success: true,
      deletedItems: deletionStats,
    })
```

*   **`const deletionStats = await deleteFolderRecursively(id, existingFolder.workspaceId)`**: Calls a helper function (explained next) to handle the complex logic of deleting the folder, all its child folders, and all workflows within them. It returns statistics on how many items were deleted.
*   **`logger.info(...)`**: Logs the successful deletion and the statistics.
*   **`return NextResponse.json({ success: true, deletedItems: deletionStats })`**: Sends a 200 OK response, indicating success and providing details on what was deleted.

#### Inside the `catch` block:

```typescript
  } catch (error) {
    logger.error('Error deleting folder:', { error })
    return NextResponse.json({ error: 'Internal server error' }, { status: 500 })
  }
```

*   **Error Handling**: Identical to the `PUT` endpoint, catches and logs errors, then sends a generic 500 response.

### 5. Helper Function: `deleteFolderRecursively`

This function is responsible for the complex task of deleting a folder and everything nested inside it, including subfolders and workflows.

```typescript
// Helper function to recursively delete a folder and all its contents
async function deleteFolderRecursively(
  folderId: string,
  workspaceId: string
): Promise<{ folders: number; workflows: number }> {
  const stats = { folders: 0, workflows: 0 }

  // Get all child folders first (workspace-scoped, not user-scoped)
  const childFolders = await db
    .select({ id: workflowFolder.id })
    .from(workflowFolder)
    .where(and(eq(workflowFolder.parentId, folderId), eq(workflowFolder.workspaceId, workspaceId)))

  // Recursively delete child folders
  for (const childFolder of childFolders) {
    const childStats = await deleteFolderRecursively(childFolder.id, workspaceId)
    stats.folders += childStats.folders
    stats.workflows += childStats.workflows
  }

  // Delete all workflows in this folder (workspace-scoped, not user-scoped)
  // The database cascade will handle deleting related workflow_blocks, workflow_edges, workflow_subflows
  const workflowsInFolder = await db
    .select({ id: workflow.id })
    .from(workflow)
    .where(and(eq(workflow.folderId, folderId), eq(workflow.workspaceId, workspaceId)))

  if (workflowsInFolder.length > 0) {
    await db
      .delete(workflow)
      .where(and(eq(workflow.folderId, folderId), eq(workflow.workspaceId, workspaceId)))

    stats.workflows += workflowsInFolder.length
  }

  // Delete this folder
  await db.delete(workflowFolder).where(eq(workflowFolder.id, folderId))

  stats.folders += 1

  return stats
}
```

*   **`async function deleteFolderRecursively(folderId: string, workspaceId: string): Promise<{ folders: number; workflows: number }>`**:
    *   An asynchronous function that takes the `folderId` to delete and the `workspaceId` (to ensure operations are confined to the correct workspace).
    *   It returns a `Promise` that resolves to an object containing the counts of deleted folders and workflows.
*   **`const stats = { folders: 0, workflows: 0 }`**: Initializes an object to keep track of the total number of folders and workflows deleted during this recursive call.
*   **`const childFolders = await db.select({ id: workflowFolder.id })...`**:
    *   Fetches the IDs of all child folders directly nested under the `folderId` being deleted.
    *   `and(eq(workflowFolder.parentId, folderId), eq(workflowFolder.workspaceId, workspaceId))`: Crucially, it filters by `parentId` AND `workspaceId` to ensure data integrity and prevent accidental cross-workspace deletions.
*   **`for (const childFolder of childFolders) { ... }`**:
    *   Iterates through each child folder found.
    *   **`const childStats = await deleteFolderRecursively(childFolder.id, workspaceId)`**: Recursively calls `deleteFolderRecursively` for each child, effectively traversing down the folder tree.
    *   **`stats.folders += childStats.folders`** and **`stats.workflows += childStats.workflows`**: Aggregates the deletion counts from the child recursive calls into the current `stats` object.
*   **`const workflowsInFolder = await db.select({ id: workflow.id })...`**:
    *   After handling all child folders, this selects all `workflow` IDs that belong directly to the current `folderId`.
    *   Again, filtered by `folderId` and `workspaceId`.
*   **`if (workflowsInFolder.length > 0) { ... }`**:
    *   If there are workflows in this folder, it proceeds to delete them.
    *   **`await db.delete(workflow).where(and(eq(workflow.folderId, folderId), eq(workflow.workspaceId, workspaceId)))`**: Deletes all workflows associated with this `folderId` and `workspaceId`.
    *   **`// The database cascade will handle deleting related workflow_blocks, workflow_edges, workflow_subflows`**: This is an important comment. It indicates that your database schema (e.g., PostgreSQL with foreign key constraints and `ON DELETE CASCADE` rules) is configured to automatically delete related records (like workflow blocks, edges, subflows) when a `workflow` record is deleted. This simplifies the application code significantly.
    *   **`stats.workflows += workflowsInFolder.length`**: Adds the count of deleted workflows to the `stats`.
*   **`await db.delete(workflowFolder).where(eq(workflowFolder.id, folderId))`**: Finally, after all its children and contents are deleted, the current `folderId` itself is deleted from the `workflowFolder` table.
*   **`stats.folders += 1`**: Increments the folder count for the currently deleted folder.
*   **`return stats`**: Returns the accumulated deletion statistics.

### 6. Helper Function: `checkForCircularReference`

This function prevents creating a folder hierarchy where a folder indirectly becomes its own parent (e.g., A -> B -> C -> A).

```typescript
// Helper function to check for circular references
async function checkForCircularReference(folderId: string, parentId: string): Promise<boolean> {
  let currentParentId: string | null = parentId
  const visited = new Set<string>()

  while (currentParentId) {
    if (visited.has(currentParentId)) {
      return true // Circular reference detected (we've seen this parent before in the current path)
    }

    if (currentParentId === folderId) {
      return true // Would create a cycle (the folder being moved is an ancestor of the target parent)
    }

    visited.add(currentParentId)

    // Get the parent of the current parent
    const parent: { parentId: string | null } | undefined = await db
      .select({ parentId: workflowFolder.parentId })
      .from(workflowFolder)
      .where(eq(workflowFolder.id, currentParentId))
      .then((rows) => rows[0])

    currentParentId = parent?.parentId || null
  }

  return false
}
```

*   **`async function checkForCircularReference(folderId: string, parentId: string): Promise<boolean>`**:
    *   An asynchronous function that takes the `folderId` of the folder being moved and the `parentId` of its proposed new parent.
    *   It returns a `Promise` that resolves to `true` if a circular reference would be created, `false` otherwise.
*   **`let currentParentId: string | null = parentId`**: Initializes `currentParentId` to the proposed new `parentId`. This variable will walk up the hierarchy from the proposed parent.
*   **`const visited = new Set<string>()`**: Creates a `Set` to keep track of all folder IDs encountered during the traversal. Sets are efficient for checking if an item has been visited.
*   **`while (currentParentId)`**: The loop continues as long as `currentParentId` is not `null` (meaning we haven't reached the root of the hierarchy).
*   **`if (visited.has(currentParentId))`**: If the `currentParentId` is already in the `visited` set, it means we've encountered this folder ID before during this particular traversal. This indicates an *existing* circular reference in the database, which should ideally not happen but is a good safeguard.
*   **`if (currentParentId === folderId)`**: This is the primary check for *creating* a new circular reference. If, while traversing up from the `parentId`, we encounter the `folderId` of the folder *being moved*, it means that `folderId` is an ancestor of the `parentId`. Placing `folderId` under `parentId` would then complete a cycle (e.g., if A is moving to be a child of B, but B is already a child of A, then A -> B -> A is a cycle).
*   **`visited.add(currentParentId)`**: Adds the current parent's ID to the `visited` set to prevent infinite loops and detect existing cycles.
*   **`const parent: { parentId: string | null } | undefined = await db.select({ parentId: workflowFolder.parentId })...`**:
    *   Queries the database to get the `parentId` of the `currentParentId` (i.e., the grandparent of the folder being moved, or the parent of the current node in the traversal).
    *   `select({ parentId: workflowFolder.parentId })`: Only selects the `parentId` column.
    *   `.then((rows) => rows[0])`: Extracts the first result.
*   **`currentParentId = parent?.parentId || null`**: Updates `currentParentId` to the `parentId` of the fetched `parent`. If `parent` is `undefined` (no such folder) or `parent.parentId` is `null` (it's a root folder), `currentParentId` becomes `null`, terminating the loop.
*   **`return false`**: If the loop completes without detecting any circular references, it means the proposed move is valid.

This comprehensive explanation covers the purpose, logic, and line-by-line breakdown of the provided TypeScript code, highlighting key patterns, safeguards, and helper functions.