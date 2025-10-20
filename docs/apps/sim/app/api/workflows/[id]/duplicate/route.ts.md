This file defines an API endpoint in a Next.js application responsible for duplicating an existing workflow, including all its associated components like blocks, edges, and subflows. It's a `POST` endpoint, meaning it expects data in the request body to specify details for the new, duplicated workflow.

---

### Purpose of this file and what it does

This TypeScript file (`src/app/api/workflows/[id]/duplicate/route.ts`) creates a **backend API endpoint** for a Next.js application. Specifically, it handles `POST` requests to `/api/workflows/[id]/duplicate`.

Its primary function is to:

1.  **Receive a request** to duplicate a workflow, identified by `[id]` in the URL path.
2.  **Authenticate and authorize** the user making the request. Only authenticated users with appropriate permissions can duplicate workflows.
3.  **Validate the incoming data** for the new workflow (e.g., its name, description).
4.  **Perform a deep copy** of the specified source workflow. This isn't just duplicating the workflow's main entry in the database; it also copies all related data:
    *   The workflow's basic properties (name, description, color, etc.).
    *   All its **workflow blocks** (individual steps or nodes in the workflow).
    *   All its **workflow edges** (connections between blocks).
    *   All its **workflow subflows** (complex, nested logic within certain blocks, like loops or parallel executions).
5.  **Generate new unique IDs** for all duplicated items (the workflow itself, its blocks, edges, and subflows) to prevent conflicts with the original.
6.  **Maintain relationships** between the newly generated blocks, edges, and subflows, ensuring the duplicated workflow is fully functional and consistent.
7.  **Persist all duplicated data** to the database within a single, atomic transaction to ensure data integrity.
8.  **Return a success or error response** to the client.

In essence, it provides a robust mechanism for users to create a new workflow that is an exact copy of an existing one, allowing for easy iteration, template creation, or branching of workflow designs.

---

### Simplify complex logic

Imagine you have a complex flowchart (a "workflow") with many steps ( "blocks"), connections between steps ("edges"), and some special groups of steps ("subflows"). You want to make an exact copy of this flowchart so you can modify the copy without affecting the original.

Here's how this code simplifies that process:

1.  **Identify the Original:** You tell the system which flowchart you want to copy (using its ID).
2.  **Who Are You?** The system first checks if you're a logged-in user and if you're allowed to even *see* (let alone copy) the original flowchart. If not, it stops you.
3.  **New Flowchart Details:** You can optionally provide a new name, description, or other basic details for the copy.
4.  **Database Transaction (All or Nothing):** This is crucial! All the copying happens inside a special "transaction." Think of it like a single, unbreakable operation. If any part of the copying fails (e.g., a connection can't be made), the *entire* duplication is rolled back, and nothing is changed in the database. This prevents half-copied, broken flowcharts.
5.  **The Copying Process:**
    *   **Workflow Itself:** A new entry is created for the main flowchart, with a brand-new ID.
    *   **Mapping Old to New:** As each step ("block") from the original flowchart is copied, the system creates a "lookup table." It remembers the original step's ID and pairs it with the *new* ID assigned to its copy. This is like saying, "Original Step A now corresponds to New Step A'."
    *   **Blocks:** All individual steps are copied. If a step had a "parent" step (e.g., it was nested inside another), its new parent ID is updated using the lookup table.
    *   **Edges (Connections):** All connections between steps are copied. The `from` and `to` IDs of these connections are updated to point to the *new* IDs of the copied steps, using our lookup table.
    *   **Subflows (Special Groups):** Any special groups of steps are also copied. Their internal references to steps are updated to point to the *new* IDs of the copied steps. Their own ID is also updated to match the ID of the block they are associated with.
6.  **Finish Up:** Once everything is copied and all connections are correctly re-established with new IDs, the transaction is completed, and the new, fully functional copy is saved.
7.  **Report Back:** The system tells you if the copy was successful and provides details about the new flowchart. If something went wrong, it explains why.

The "complex logic" primarily revolves around managing these new IDs and ensuring all internal references within the copied data correctly point to other *newly generated* data, rather than still pointing to the original items. This is handled by the `blockIdMapping` and careful updates to the `data` and `config` fields of blocks and subflows.

---

### Explain each line of code deep

Let's break down the code line by line.

```typescript
// --- IMPORTS ---
import { db } from '@sim/db'
import { workflow, workflowBlocks, workflowEdges, workflowSubflows } from '@sim/db/schema'
import { eq } from 'drizzle-orm'
import { type NextRequest, NextResponse } from 'next/server'
import { z } from 'zod'
import { getSession } from '@/lib/auth'
import { createLogger } from '@/lib/logs/console/logger'
import { getUserEntityPermissions } from '@/lib/permissions/utils'
import { generateRequestId } from '@/lib/utils'
import type { Variable } from '@/stores/panel/variables/types'
import type { LoopConfig, ParallelConfig } from '@/stores/workflows/workflow/types'
```

*   **`import { db } from '@sim/db'`**: Imports the database connection instance, likely configured with Drizzle ORM, which is used to interact with the database. `@sim/db` is an alias for a specific path in the project.
*   **`import { workflow, workflowBlocks, workflowEdges, workflowSubflows } from '@sim/db/schema'`**: Imports the schema definitions (table objects) for the `workflow`, `workflowBlocks`, `workflowEdges`, and `workflowSubflows` tables. These are used by Drizzle ORM to build queries.
*   **`import { eq } from 'drizzle-orm'`**: Imports the `eq` (equals) function from Drizzle ORM. This is a helper function used in `WHERE` clauses to specify equality conditions.
*   **`import { type NextRequest, NextResponse } from 'next/server'`**: Imports types and classes specific to Next.js API Routes. `NextRequest` represents the incoming HTTP request, and `NextResponse` is used to construct the HTTP response.
*   **`import { z } from 'zod'`**: Imports the `z` object from the Zod library. Zod is a TypeScript-first schema declaration and validation library, used here to validate incoming request body data.
*   **`import { getSession } from '@/lib/auth'`**: Imports a utility function `getSession` responsible for retrieving the current user's session information (e.g., user ID, authentication status). ` '@/lib/auth'` is an alias for a local path.
*   **`import { createLogger } from '@/lib/logs/console/logger'`**: Imports a function to create a logger instance, used for structured logging of events and errors.
*   **`import { getUserEntityPermissions } from '@/lib/permissions/utils'`**: Imports a utility function `getUserEntityPermissions` to check a user's permissions on a specific entity (like a workspace).
*   **`import { generateRequestId } from '@/lib/utils'`**: Imports a utility function to generate a unique request ID, useful for tracing logs related to a single request.
*   **`import type { Variable } from '@/stores/panel/variables/types'`**: Imports a TypeScript `type` definition for `Variable`. This is used for type-checking and doesn't directly become part of the runtime JavaScript. It helps ensure the `variables` field in the workflow schema is correctly typed.
*   **`import type { LoopConfig, ParallelConfig } from '@/stores/workflows/workflow/types'`**: Imports TypeScript `type` definitions for `LoopConfig` and `ParallelConfig`. These are used for type-checking the `config` field within `workflowSubflows`, indicating that subflows can have different configuration types.

```typescript
const logger = createLogger('WorkflowDuplicateAPI')
```

*   **`const logger = createLogger('WorkflowDuplicateAPI')`**: Initializes a logger instance specifically for this API, giving it the name `'WorkflowDuplicateAPI'`. This helps in filtering and identifying logs originating from this particular part of the application.

```typescript
const DuplicateRequestSchema = z.object({
  name: z.string().min(1, 'Name is required'),
  description: z.string().optional(),
  color: z.string().optional(),
  workspaceId: z.string().optional(),
  folderId: z.string().nullable().optional(),
})
```

*   **`const DuplicateRequestSchema = z.object({...})`**: Defines a Zod schema named `DuplicateRequestSchema`. This schema specifies the expected structure and validation rules for the JSON data sent in the request body when duplicating a workflow.
    *   **`name: z.string().min(1, 'Name is required')`**: The `name` field must be a string and have a minimum length of 1 character. If it's empty, an error message 'Name is required' will be returned.
    *   **`description: z.string().optional()`**: The `description` field must be a string if provided, but it's optional (can be omitted).
    *   **`color: z.string().optional()`**: Similar to description, `color` must be a string if provided, but is optional.
    *   **`workspaceId: z.string().optional()`**: The `workspaceId` field, if provided, must be a string. It's optional, meaning the new workflow can inherit the workspace of the source workflow if not specified.
    *   **`folderId: z.string().nullable().optional()`**: The `folderId` field, if provided, must be a string. It's `nullable()` meaning it can explicitly be `null`, and `optional()` meaning it can also be entirely omitted. This allows for placing the duplicated workflow in a specific folder, no folder (`null`), or inheriting the source workflow's folder.

```typescript
// POST /api/workflows/[id]/duplicate - Duplicate a workflow with all its blocks, edges, and subflows
export async function POST(req: NextRequest, { params }: { params: Promise<{ id: string }> }) {
```

*   **`export async function POST(...)`**: Defines the main asynchronous function that handles `POST` requests for this API route. In Next.js App Router, functions named `GET`, `POST`, `PUT`, `DELETE`, etc., are special and automatically exposed as API endpoints.
    *   **`req: NextRequest`**: The first parameter is the incoming request object, typed as `NextRequest`, which provides access to the request body, headers, URL, etc.
    *   **`{ params }: { params: Promise<{ id: string }> }`**: The second parameter is an object that destructures `params`. `params` is an object containing route parameters. In this case, `[id]` in the file path translates to `id` in the `params` object. The `Promise` indicates that `params` might be an asynchronous value, common in Next.js. `{ id: string }` specifies that `params` will have an `id` property of type `string`.

```typescript
  const { id: sourceWorkflowId } = await params
  const requestId = generateRequestId()
  const startTime = Date.now()
```

*   **`const { id: sourceWorkflowId } = await params`**: Destructures the `id` property from the `params` object and renames it to `sourceWorkflowId` for clarity. `await params` is used because `params` is typed as a `Promise`.
*   **`const requestId = generateRequestId()`**: Generates a unique ID for the current request using the `generateRequestId` utility function. This ID will be logged with messages to help track the flow of a single request.
*   **`const startTime = Date.now()`**: Records the current timestamp when the request processing begins, used later to calculate the total execution time.

```typescript
  const session = await getSession()
  if (!session?.user?.id) {
    logger.warn(`[${requestId}] Unauthorized workflow duplication attempt for ${sourceWorkflowId}`)
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
  }
```

*   **`const session = await getSession()`**: Calls the `getSession` utility to retrieve the user's session information. This is an asynchronous operation as it might involve database lookups or external authentication services.
*   **`if (!session?.user?.id)`**: Checks if a user is logged in. `session?.user?.id` uses optional chaining (`?.`) to safely access nested properties, returning `undefined` if any part of the path is `null` or `undefined`. If there's no `user.id` in the session, the user is not authenticated.
    *   **`logger.warn(...)`**: Logs a warning message indicating an unauthorized attempt, including the `requestId` and `sourceWorkflowId` for debugging.
    *   **`return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })`**: Returns an HTTP 401 (Unauthorized) status code with a JSON error message, indicating that the user needs to be authenticated to perform this action.

```typescript
  try {
    const body = await req.json()
    const { name, description, color, workspaceId, folderId } = DuplicateRequestSchema.parse(body)
```

*   **`try { ... } catch (error) { ... }`**: This `try...catch` block wraps the core logic of the API route to handle potential errors gracefully.
*   **`const body = await req.json()`**: Parses the incoming request body as JSON. This is an asynchronous operation.
*   **`const { name, description, color, workspaceId, folderId } = DuplicateRequestSchema.parse(body)`**: Validates the parsed JSON `body` against the `DuplicateRequestSchema` defined earlier using Zod's `parse` method.
    *   If the `body` matches the schema, the validated properties are destructured into individual constants.
    *   If validation fails (e.g., `name` is missing or empty), Zod throws a `ZodError`, which will be caught by the `catch` block.

```typescript
    logger.info(
      `[${requestId}] Duplicating workflow ${sourceWorkflowId} for user ${session.user.id}`
    )
```

*   **`logger.info(...)`**: Logs an informational message indicating that the duplication process has started, including the `requestId`, `sourceWorkflowId`, and the `session.user.id`.

```typescript
    // Generate new workflow ID
    const newWorkflowId = crypto.randomUUID()
    const now = new Date()
```

*   **`const newWorkflowId = crypto.randomUUID()`**: Generates a universally unique identifier (UUID) for the new workflow using the `crypto.randomUUID()` API. This ensures the new workflow has a unique ID, distinct from the source workflow.
*   **`const now = new Date()`**: Creates a new `Date` object representing the current date and time. This will be used for `createdAt`, `updatedAt`, and `lastSynced` timestamps for all new database entries.

```typescript
    // Duplicate workflow and all related data in a transaction
    const result = await db.transaction(async (tx) => {
```

*   **`const result = await db.transaction(async (tx) => { ... })`**: This is a critical part of the logic. It initiates a **database transaction** using Drizzle ORM's `db.transaction` method.
    *   A transaction ensures **atomicity**: either all database operations within the transaction succeed and are committed, or if any operation fails, the entire transaction is rolled back, leaving the database unchanged. This prevents partial, inconsistent data in case of an error during the duplication process.
    *   The callback function receives a `tx` object (transaction client), which is used for all database operations *within* this transaction instead of the global `db` object.

```typescript
      // First verify the source workflow exists
      const sourceWorkflowRow = await tx
        .select()
        .from(workflow)
        .where(eq(workflow.id, sourceWorkflowId))
        .limit(1)

      if (sourceWorkflowRow.length === 0) {
        throw new Error('Source workflow not found')
      }

      const source = sourceWorkflowRow[0]
```

*   **`const sourceWorkflowRow = await tx.select().from(workflow).where(eq(workflow.id, sourceWorkflowId)).limit(1)`**: Queries the `workflow` table to retrieve the source workflow based on `sourceWorkflowId`.
    *   `tx.select()`: Starts a select query.
    *   `.from(workflow)`: Specifies the `workflow` table.
    *   `.where(eq(workflow.id, sourceWorkflowId))`: Filters rows where the `id` column of the `workflow` table equals `sourceWorkflowId`. `eq` is the Drizzle equality operator.
    *   `.limit(1)`: Limits the result to one row, as `id` is a primary key and should be unique.
*   **`if (sourceWorkflowRow.length === 0)`**: Checks if any workflow was found. If the array is empty, the source workflow does not exist.
    *   **`throw new Error('Source workflow not found')`**: Throws an error, which will be caught by the outer `try...catch` block, rolling back the transaction.
*   **`const source = sourceWorkflowRow[0]`**: If found, extracts the first (and only) row from the result array, representing the source workflow object.

```typescript
      // Check if user has permission to access the source workflow
      let canAccessSource = false

      // Case 1: User owns the workflow
      if (source.userId === session.user.id) {
        canAccessSource = true
      }

      // Case 2: User has admin or write permission in the source workspace
      if (!canAccessSource && source.workspaceId) {
        const userPermission = await getUserEntityPermissions(
          session.user.id,
          'workspace',
          source.workspaceId
        )
        if (userPermission === 'admin' || userPermission === 'write') {
          canAccessSource = true
        }
      }

      if (!canAccessSource) {
        throw new Error('Source workflow not found or access denied')
      }
```

*   **`let canAccessSource = false`**: Initializes a boolean flag to track if the user has permission to access the source workflow.
*   **`if (source.userId === session.user.id)`**: **Permission Check 1**: Checks if the current authenticated user (`session.user.id`) is the owner (`userId`) of the source workflow. If so, `canAccessSource` is set to `true`.
*   **`if (!canAccessSource && source.workspaceId)`**: **Permission Check 2**: If the user is *not* the owner, and the source workflow is associated with a `workspaceId`, then it proceeds to check workspace permissions.
    *   **`const userPermission = await getUserEntityPermissions(session.user.id, 'workspace', source.workspaceId)`**: Calls a utility function to get the user's permission level (`'admin'`, `'write'`, `'read'`, or `null`) for the specific `workspaceId`.
    *   **`if (userPermission === 'admin' || userPermission === 'write')`**: If the user has either 'admin' or 'write' permissions in that workspace, they are granted access.
*   **`if (!canAccessSource)`**: After both checks, if `canAccessSource` is still `false`, the user does not have sufficient permissions.
    *   **`throw new Error('Source workflow not found or access denied')`**: Throws an error. The wording is intentionally vague for security reasons (don't reveal if the workflow simply doesn't exist or if access is denied to an existing one). This error will be caught by the outer `try...catch`.

```typescript
      // Create the new workflow first (required for foreign key constraints)
      await tx.insert(workflow).values({
        id: newWorkflowId,
        userId: session.user.id,
        workspaceId: workspaceId || source.workspaceId,
        folderId: folderId || source.folderId,
        name,
        description: description || source.description,
        color: color || source.color,
        lastSynced: now,
        createdAt: now,
        updatedAt: now,
        isDeployed: false,
        collaborators: [],
        runCount: 0,
        // Duplicate variables with new IDs and new workflowId
        variables: (() => {
          const sourceVars = (source.variables as Record<string, Variable>) || {}
          const remapped: Record<string, Variable> = {}
          for (const [, variable] of Object.entries(sourceVars) as [string, Variable][]) {
            const newVarId = crypto.randomUUID()
            remapped[newVarId] = {
              ...variable,
              id: newVarId,
              workflowId: newWorkflowId,
            }
          }
          return remapped
        })(),
        isPublished: false,
        marketplaceData: null,
      })
```

*   **`await tx.insert(workflow).values({...})`**: Inserts a new row into the `workflow` table, representing the duplicated workflow. This is done first because `workflowBlocks`, `workflowEdges`, and `workflowSubflows` will have foreign key relationships to this new workflow.
    *   **`id: newWorkflowId`**: Assigns the newly generated UUID to the new workflow.
    *   **`userId: session.user.id`**: Sets the owner of the new workflow to the currently authenticated user.
    *   **`workspaceId: workspaceId || source.workspaceId`**: If a `workspaceId` was provided in the request body, use it; otherwise, inherit the `workspaceId` from the `source` workflow.
    *   **`folderId: folderId || source.folderId`**: Similar logic for `folderId`.
    *   **`name`**: Uses the `name` from the request body (which is required by the schema).
    *   **`description: description || source.description`**: If a `description` was provided, use it; otherwise, inherit from the `source` workflow.
    *   **`color: color || source.color`**: Similar logic for `color`.
    *   **`lastSynced: now`, `createdAt: now`, `updatedAt: now`**: Sets the timestamp fields to the current time (`now`).
    *   **`isDeployed: false`, `collaborators: []`, `runCount: 0`, `isPublished: false`, `marketplaceData: null`**: Sets default or empty values for fields that shouldn't be directly copied or are reset for a new workflow.
    *   **`variables: (() => { ... })()`**: This is an immediately invoked function expression (IIFE) that handles duplicating the workflow's variables.
        *   **`const sourceVars = (source.variables as Record<string, Variable>) || {}`**: Retrieves the `variables` from the `source` workflow. It's cast to `Record<string, Variable>` to ensure type safety and defaults to an empty object if `source.variables` is null/undefined.
        *   **`const remapped: Record<string, Variable> = {}`**: Initializes an empty object to store the duplicated and re-ID'd variables.
        *   **`for (const [, variable] of Object.entries(sourceVars) as [string, Variable][])`**: Iterates over each variable in the `sourceVars` object. `Object.entries` returns `[key, value]` pairs, but we only need the `variable` (value).
        *   **`const newVarId = crypto.randomUUID()`**: Generates a new unique ID for each duplicated variable.
        *   **`remapped[newVarId] = { ...variable, id: newVarId, workflowId: newWorkflowId, }`**: Creates a new variable object. It spreads all existing properties of the `variable` (`...variable`), then *overrides* its `id` with `newVarId` and its `workflowId` with `newWorkflowId`. This ensures variables are unique and linked to the new workflow.
        *   **`return remapped`**: The IIFE returns the `remapped` object, which becomes the value for the `variables` field of the new workflow.

```typescript
      // Copy all blocks from source workflow with new IDs
      const sourceBlocks = await tx
        .select()
        .from(workflowBlocks)
        .where(eq(workflowBlocks.workflowId, sourceWorkflowId))

      // Create a mapping from old block IDs to new block IDs
      const blockIdMapping = new Map<string, string>()

      if (sourceBlocks.length > 0) {
        // First pass: Create all block ID mappings
        sourceBlocks.forEach((block) => {
          const newBlockId = crypto.randomUUID()
          blockIdMapping.set(block.id, newBlockId)
        })

        // Second pass: Create blocks with updated parent relationships
        const newBlocks = sourceBlocks.map((block) => {
          const newBlockId = blockIdMapping.get(block.id)!

          // Update parent ID to point to the new parent block ID if it exists
          const blockData =
            block.data && typeof block.data === 'object' && !Array.isArray(block.data)
              ? (block.data as any)
              : {}
          let newParentId = blockData.parentId
          if (blockData.parentId && blockIdMapping.has(blockData.parentId)) {
            newParentId = blockIdMapping.get(blockData.parentId)!
          }

          // Update data.parentId and extent if they exist in the data object
          let updatedData = block.data
          let newExtent = blockData.extent
          if (block.data && typeof block.data === 'object' && !Array.isArray(block.data)) {
            const dataObj = block.data as any
            if (dataObj.parentId && typeof dataObj.parentId === 'string') {
              updatedData = { ...dataObj }
              if (blockIdMapping.has(dataObj.parentId)) {
                ;(updatedData as any).parentId = blockIdMapping.get(dataObj.parentId)!
                // Ensure extent is set to 'parent' for child blocks
                ;(updatedData as any).extent = 'parent'
                newExtent = 'parent' // Update local newExtent too
              }
            }
          }

          return {
            ...block,
            id: newBlockId,
            workflowId: newWorkflowId,
            parentId: newParentId,
            extent: newExtent,
            data: updatedData,
            createdAt: now,
            updatedAt: now,
          }
        })

        await tx.insert(workflowBlocks).values(newBlocks)
        logger.info(
          `[${requestId}] Copied ${sourceBlocks.length} blocks with updated parent relationships`
        )
      }
```

*   **`const sourceBlocks = await tx.select().from(workflowBlocks).where(eq(workflowBlocks.workflowId, sourceWorkflowId))`**: Fetches all `workflowBlocks` associated with the `sourceWorkflowId`.
*   **`const blockIdMapping = new Map<string, string>()`**: Initializes an empty `Map` object. This map will store the relationship between the *original* block IDs and their *newly generated* IDs. This is crucial for updating references later (edges, subflows, and parent blocks).
*   **`if (sourceBlocks.length > 0)`**: Proceeds only if there are blocks to copy.
    *   **`sourceBlocks.forEach((block) => { ... })`**: **First pass:** Iterates through each `sourceBlock`.
        *   **`const newBlockId = crypto.randomUUID()`**: Generates a unique ID for the *new* copy of the current block.
        *   **`blockIdMapping.set(block.id, newBlockId)`**: Stores the mapping: `originalBlockId -> newBlockId`.
    *   **`const newBlocks = sourceBlocks.map((block) => { ... })`**: **Second pass:** Transforms each `sourceBlock` into a `newBlock` for insertion.
        *   **`const newBlockId = blockIdMapping.get(block.id)!`**: Retrieves the `newBlockId` from the mapping created in the first pass. The `!` (non-null assertion operator) is used here because we are sure the ID exists in the map.
        *   **`const blockData = ... ? (block.data as any) : {}`**: Safely extracts the `data` field from the block, which is often a JSON object, and casts it to `any` for easier property access. Defaults to an empty object if `data` is not an object.
        *   **`let newParentId = blockData.parentId`**: Initializes `newParentId` with the `parentId` from the `blockData`.
        *   **`if (blockData.parentId && blockIdMapping.has(blockData.parentId))`**: Checks if the block has a `parentId` in its data *and* if that `parentId` exists in our `blockIdMapping` (meaning the parent block itself was also duplicated).
        *   **`newParentId = blockIdMapping.get(blockData.parentId)!`**: If the parent was duplicated, update `newParentId` to the *new* ID of the parent block.
        *   **`let updatedData = block.data; let newExtent = blockData.extent; if (block.data && ...)`**: This block handles more complex updates within the `block.data` JSON itself. Some blocks (like 'group' blocks) store children's parent IDs and 'extent' (whether it's contained by a parent) within their `data` payload.
            *   It checks for `dataObj.parentId` and updates it to the new `parentId` using `blockIdMapping`.
            *   It also ensures that if a block is now a child of a new parent, its `extent` property within its `data` is explicitly set to `'parent'`, and updates `newExtent` accordingly. This is crucial for correct rendering and behavior of nested blocks in the UI.
        *   **`return { ...block, id: newBlockId, workflowId: newWorkflowId, parentId: newParentId, extent: newExtent, data: updatedData, createdAt: now, updatedAt: now, }`**: Constructs the `newBlock` object. It spreads the original block's properties, then overrides `id`, `workflowId`, `parentId`, `extent`, `data`, and timestamps with their new values.
    *   **`await tx.insert(workflowBlocks).values(newBlocks)`**: Inserts all the `newBlocks` into the `workflowBlocks` table in a single batch operation.
    *   **`logger.info(...)`**: Logs the number of blocks copied.

```typescript
      // Copy all edges from source workflow with updated block references
      const sourceEdges = await tx
        .select()
        .from(workflowEdges)
        .where(eq(workflowEdges.workflowId, sourceWorkflowId))

      if (sourceEdges.length > 0) {
        const newEdges = sourceEdges.map((edge) => ({
          ...edge,
          id: crypto.randomUUID(), // Generate new edge ID
          workflowId: newWorkflowId,
          sourceBlockId: blockIdMapping.get(edge.sourceBlockId) || edge.sourceBlockId,
          targetBlockId: blockIdMapping.get(edge.targetBlockId) || edge.targetBlockId,
          createdAt: now,
          updatedAt: now,
        }))

        await tx.insert(workflowEdges).values(newEdges)
        logger.info(
          `[${requestId}] Copied ${sourceEdges.length} edges with updated block references`
        )
      }
```

*   **`const sourceEdges = await tx.select().from(workflowEdges).where(eq(workflowEdges.workflowId, sourceWorkflowId))`**: Fetches all `workflowEdges` associated with the `sourceWorkflowId`.
*   **`if (sourceEdges.length > 0)`**: Proceeds only if there are edges to copy.
    *   **`const newEdges = sourceEdges.map((edge) => ({ ... }))`**: Transforms each `sourceEdge` into a `newEdge`.
        *   **`id: crypto.randomUUID()`**: Assigns a new unique ID to each duplicated edge.
        *   **`workflowId: newWorkflowId`**: Links the edge to the `newWorkflowId`.
        *   **`sourceBlockId: blockIdMapping.get(edge.sourceBlockId) || edge.sourceBlockId`**: Uses the `blockIdMapping` to get the new ID for the source block of the edge. If for some reason the source block wasn't in the mapping (e.g., it was filtered out or an error occurred), it falls back to the original ID (though in a healthy system, this fallback shouldn't be hit for copied blocks).
        *   **`targetBlockId: blockIdMapping.get(edge.targetBlockId) || edge.targetBlockId`**: Same logic for the target block ID.
        *   **`createdAt: now`, `updatedAt: now`**: Sets the timestamps.
    *   **`await tx.insert(workflowEdges).values(newEdges)`**: Inserts all `newEdges` into the `workflowEdges` table.
    *   **`logger.info(...)`**: Logs the number of edges copied.

```typescript
      // Copy all subflows from source workflow with new IDs and updated block references
      const sourceSubflows = await tx
        .select()
        .from(workflowSubflows)
        .where(eq(workflowSubflows.workflowId, sourceWorkflowId))

      if (sourceSubflows.length > 0) {
        const newSubflows = sourceSubflows
          .map((subflow) => {
            // The subflow ID should match the corresponding block ID
            const newSubflowId = blockIdMapping.get(subflow.id)

            if (!newSubflowId) {
              logger.warn(
                `[${requestId}] Subflow ${subflow.id} (${subflow.type}) has no corresponding block, skipping`
              )
              return null
            }

            logger.info(`[${requestId}] Mapping subflow ${subflow.id} → ${newSubflowId}`, {
              subflowType: subflow.type,
            })

            // Update block references in subflow config
            let updatedConfig: LoopConfig | ParallelConfig = subflow.config as
              | LoopConfig
              | ParallelConfig
            if (subflow.config && typeof subflow.config === 'object') {
              updatedConfig = JSON.parse(JSON.stringify(subflow.config)) as
                | LoopConfig
                | ParallelConfig

              // Update the config ID to match the new subflow ID
              ;(updatedConfig as any).id = newSubflowId

              // Update node references in config if they exist
              if ('nodes' in updatedConfig && Array.isArray(updatedConfig.nodes)) {
                updatedConfig.nodes = updatedConfig.nodes.map(
                  (nodeId: string) => blockIdMapping.get(nodeId) || nodeId
                )
              }
            }

            return {
              ...subflow,
              id: newSubflowId, // Use the same ID as the corresponding block
              workflowId: newWorkflowId,
              config: updatedConfig,
              createdAt: now,
              updatedAt: now,
            }
          })
          .filter((subflow): subflow is NonNullable<typeof subflow> => subflow !== null)

        if (newSubflows.length > 0) {
          await tx.insert(workflowSubflows).values(newSubflows)
        }

        logger.info(
          `[${requestId}] Copied ${newSubflows.length}/${sourceSubflows.length} subflows with updated block references and matching IDs`,
          {
            subflowMappings: newSubflows.map((sf) => ({
              oldId: sourceSubflows.find((s) => blockIdMapping.get(s.id) === sf.id)?.id,
              newId: sf.id,
              type: sf.type,
              config: sf.config,
            })),
            blockIdMappings: Array.from(blockIdMapping.entries()).map(([oldId, newId]) => ({
              oldId,
              newId,
            })),
          }
        )
      }
```

*   **`const sourceSubflows = await tx.select().from(workflowSubflows).where(eq(workflowSubflows.workflowId, sourceWorkflowId))`**: Fetches all `workflowSubflows` associated with the `sourceWorkflowId`.
*   **`if (sourceSubflows.length > 0)`**: Proceeds only if there are subflows to copy.
    *   **`const newSubflows = sourceSubflows.map((subflow) => { ... }).filter(...)`**: This transforms `sourceSubflows` into `newSubflows`, and also filters out any subflows that couldn't be correctly mapped.
        *   **`const newSubflowId = blockIdMapping.get(subflow.id)`**: Crucially, subflows often share the *same ID* as their corresponding `workflowBlock` (e.g., a Loop block has a subflow with the same ID). So, we use the `blockIdMapping` to find the `newSubflowId`.
        *   **`if (!newSubflowId) { ... return null; }`**: If a subflow's original ID doesn't have a corresponding new block ID in the mapping, it means the block itself might have been skipped or an inconsistency exists. In such a case, a warning is logged, and `null` is returned, effectively skipping this subflow.
        *   **`logger.info(...)`**: Logs the mapping of subflow IDs.
        *   **`let updatedConfig: LoopConfig | ParallelConfig = subflow.config as ...`**: Handles updating the `config` object of the subflow. `config` holds the specific logic for loop or parallel subflows (e.g., which nodes belong to them).
        *   **`updatedConfig = JSON.parse(JSON.stringify(subflow.config))`**: This is a common way to deep clone a JSON-serializable object. It creates a completely new object for `updatedConfig` to avoid modifying the original `sourceSubflow.config`.
        *   **`(updatedConfig as any).id = newSubflowId`**: Updates the `id` *within* the `config` object itself to match the `newSubflowId`.
        *   **`if ('nodes' in updatedConfig && Array.isArray(updatedConfig.nodes))`**: Checks if the `config` object has a `nodes` property (which `LoopConfig` or `ParallelConfig` might have).
        *   **`updatedConfig.nodes = updatedConfig.nodes.map(...)`**: If `nodes` exist, it iterates through them and updates each `nodeId` to its *new* ID using `blockIdMapping`.
        *   **`return { ...subflow, id: newSubflowId, workflowId: newWorkflowId, config: updatedConfig, createdAt: now, updatedAt: now, }`**: Constructs the `newSubflow` object, overriding `id`, `workflowId`, `config`, and timestamps.
    *   **`.filter((subflow): subflow is NonNullable<typeof subflow> => subflow !== null)`**: Filters out any `null` values that might have been returned by the `map` function (for skipped subflows). `NonNullable` is a TypeScript utility type.
    *   **`if (newSubflows.length > 0) { await tx.insert(workflowSubflows).values(newSubflows); }`**: If there are any valid `newSubflows` left after filtering, they are inserted into the database.
    *   **`logger.info(...)`**: Logs detailed information about the subflow copying, including mappings for auditing and debugging.

```typescript
      // Update the workflow timestamp
      await tx
        .update(workflow)
        .set({
          updatedAt: now,
        })
        .where(eq(workflow.id, newWorkflowId))
```

*   **`await tx.update(workflow).set({ updatedAt: now }).where(eq(workflow.id, newWorkflowId))`**: Updates the `updatedAt` timestamp of the *newly created* workflow to `now`. This ensures the new workflow's `updatedAt` field reflects the time it was duplicated.

```typescript
      return {
        id: newWorkflowId,
        name,
        description: description || source.description,
        color: color || source.color,
        workspaceId: workspaceId || source.workspaceId,
        folderId: folderId || source.folderId,
        blocksCount: sourceBlocks.length,
        edgesCount: sourceEdges.length,
        subflowsCount: sourceSubflows.length,
      }
    })
```

*   **`return { ... }`**: This object is the return value of the `db.transaction` callback. If the transaction is successful, this object will be assigned to the `result` constant. It contains key information about the newly created workflow, including its ID and counts of duplicated components.

```typescript
    const elapsed = Date.now() - startTime
    logger.info(
      `[${requestId}] Successfully duplicated workflow ${sourceWorkflowId} to ${newWorkflowId} in ${elapsed}ms`
    )

    return NextResponse.json(result, { status: 201 })
  } catch (error) {
```

*   **`const elapsed = Date.now() - startTime`**: Calculates the total time taken for the duplication process.
*   **`logger.info(...)`**: Logs a success message, indicating the source and new workflow IDs, and the execution time.
*   **`return NextResponse.json(result, { status: 201 })`**: Returns a successful HTTP 201 (Created) status code with the `result` object (containing details of the new workflow) as JSON to the client.

```typescript
    if (error instanceof Error) {
      if (error.message === 'Source workflow not found') {
        logger.warn(`[${requestId}] Source workflow ${sourceWorkflowId} not found`)
        return NextResponse.json({ error: 'Source workflow not found' }, { status: 404 })
      }

      if (error.message === 'Source workflow not found or access denied') {
        logger.warn(
          `[${requestId}] User ${session.user.id} denied access to source workflow ${sourceWorkflowId}`
        )
        return NextResponse.json({ error: 'Access denied' }, { status: 403 })
      }
    }
```

*   **`catch (error)`**: This block handles any errors thrown during the `try` block.
*   **`if (error instanceof Error)`**: Checks if the caught error is a standard JavaScript `Error` object.
    *   **`if (error.message === 'Source workflow not found')`**: Catches the specific error thrown if the source workflow wasn't found.
        *   **`logger.warn(...)`**: Logs a warning.
        *   **`return NextResponse.json({ error: 'Source workflow not found' }, { status: 404 })`**: Returns an HTTP 404 (Not Found) response.
    *   **`if (error.message === 'Source workflow not found or access denied')`**: Catches the specific error related to access denial.
        *   **`logger.warn(...)`**: Logs a warning.
        *   **`return NextResponse.json({ error: 'Access denied' }, { status: 403 })`**: Returns an HTTP 403 (Forbidden) response.

```typescript
    if (error instanceof z.ZodError) {
      logger.warn(`[${requestId}] Invalid duplication request data`, { errors: error.errors })
      return NextResponse.json(
        { error: 'Invalid request data', details: error.errors },
        { status: 400 }
      )
    }
```

*   **`if (error instanceof z.ZodError)`**: Catches errors specifically thrown by Zod during input validation (when `DuplicateRequestSchema.parse(body)` fails).
    *   **`logger.warn(...)`**: Logs a warning about invalid input, including the specific Zod validation errors.
    *   **`return NextResponse.json({ error: 'Invalid request data', details: error.errors }, { status: 400 })`**: Returns an HTTP 400 (Bad Request) response, providing details of the validation errors to the client.

```typescript
    const elapsed = Date.now() - startTime
    logger.error(
      `[${requestId}] Error duplicating workflow ${sourceWorkflowId} after ${elapsed}ms:`,
      error
    )
    return NextResponse.json({ error: 'Failed to duplicate workflow' }, { status: 500 })
  }
}
```

*   **`const elapsed = Date.now() - startTime`**: Recalculates elapsed time for the error case.
*   **`logger.error(...)`**: If an unhandled error occurs (not one of the specific ones above), an error message is logged, including the `requestId`, source workflow ID, execution time, and the full error object for debugging.
*   **`return NextResponse.json({ error: 'Failed to duplicate workflow' }, { status: 500 })`**: Returns a generic HTTP 500 (Internal Server Error) response, informing the client that the operation failed without exposing internal server details.