This TypeScript file defines a `SnapshotService` responsible for managing and storing "snapshots" of workflow states. Imagine you have a complex visual workflow editor (like a diagram builder) where users connect various blocks and configure them. This service ensures that the exact configuration of a workflow at any given point can be saved, retrieved, and, crucially, efficiently stored by avoiding duplicate saves of identical states.

Let's break down its functionality in detail.

---

### Purpose of this file and what it does

This `SnapshotService` is designed to:

1.  **Persist Workflow States:** Save the complete, current configuration of a workflow (`WorkflowState`) into a database. This allows users to revert to previous versions, track changes, or simply resume work exactly where they left off.
2.  **Deduplicate Snapshots:** Prevent storing redundant copies of a workflow state if its functional configuration hasn't changed. For example, if a user moves a block on the canvas but doesn't change any of its settings or connections, the service intelligently recognizes this as the "same" functional state and reuses an existing snapshot. This saves database space and improves performance.
3.  **Retrieve Snapshots:** Allow fetching a specific snapshot either by its unique ID or by its workflow ID and a computed hash.
4.  **Cleanup Old Snapshots:** Provide a mechanism to automatically delete outdated snapshots, preventing the database from growing indefinitely with historical data that is no longer needed.

In essence, it acts as a version control and storage layer for workflow configurations, prioritizing efficiency and data integrity.

---

### Simplify Complex Logic

The most intricate part of this service is the **deduplication mechanism**, which relies on **state hashing**. Here's a simplified explanation:

1.  **What is a "Workflow State"?**
    Think of a `WorkflowState` as a big JSON object that describes *everything* about your workflow: which blocks are present, how they're connected (edges), what values are configured within each block, their positions on the screen, etc.

2.  **The Deduplication Problem:**
    If a user opens a workflow, moves a block slightly (changes its `position`), and then saves, should that be considered a *new* snapshot? Visually, yes. Functionally, no, because the workflow will behave exactly the same way. Storing a new snapshot for every slight visual tweak is inefficient.

3.  **The Solution: Functional Hashing:**
    *   The service needs a way to determine if two workflow states are *functionally* identical, even if their visual properties (like `position`) differ.
    *   It achieves this by creating a "normalized" version of the `WorkflowState`. This normalization process **intentionally ignores cosmetic details** like block positions. It focuses only on the structural and configurational elements that affect how the workflow *runs*.
    *   Once the state is normalized into a consistent, canonical form, it's converted into a string in a very specific, ordered way (`normalizedStringify`). This ensures that the same functional state always produces the exact same string.
    *   Finally, a **SHA256 hash** is computed from this string. This hash is a short, unique fingerprint for that specific functional state.

4.  **How Deduplication Works with the Hash:**
    *   When a new snapshot needs to be created, the service first computes this "functional hash" for the incoming state.
    *   It then checks the database: "Do I already have a snapshot for this `workflowId` with this exact `stateHash`?"
    *   If yes, great! It reuses the existing snapshot (returns it and marks it as `isNew: false`), avoiding a new database write.
    *   If no, then it truly is a new functional state. The service stores the *full* original state (including positions, because you might want to recreate the exact visual layout later), along with the new `stateHash`, and marks it as `isNew: true`.

This intelligent hashing ensures that while the database always stores the *complete* visual and functional state for accurate recreation, it only creates *new records* when there's a meaningful, functional change to the workflow.

---

### Explain Each Line of Code Deep

Let's go through the code line by line.

```typescript
import { createHash } from 'crypto'
```

*   **`import { createHash } from 'crypto'`**: Imports the `createHash` function from Node.js's built-in `crypto` module. This function is essential for generating cryptographic hash values (like SHA256) from data, which is crucial for the deduplication logic.

```typescript
import { db } from '@sim/db'
import { workflowExecutionSnapshots } from '@sim/db/schema'
import { and, eq, lt } from 'drizzle-orm'
```

*   **`import { db } from '@sim/db'`**: Imports the Drizzle ORM database client instance (`db`) from a local module. This `db` object is the primary way the service interacts with the database to perform queries (insert, select, delete).
*   **`import { workflowExecutionSnapshots } from '@sim/db/schema'`**: Imports the Drizzle schema definition for the `workflowExecutionSnapshots` table. This provides a type-safe way to reference the table and its columns in database queries.
*   **`import { and, eq, lt } from 'drizzle-orm'`**: Imports specific functions from the `drizzle-orm` library:
    *   `and`: Used to combine multiple conditions in a `WHERE` clause with a logical AND.
    *   `eq`: Stands for "equals," used to create an equality comparison in a `WHERE` clause.
    *   `lt`: Stands for "less than," used to create a "less than" comparison (e.g., for dates) in a `WHERE` clause.

```typescript
import { v4 as uuidv4 } from 'uuid'
```

*   **`import { v4 as uuidv4 } from 'uuid'`**: Imports the `v4` function from the `uuid` library, aliasing it as `uuidv4`. This function generates universally unique identifiers (UUIDs) in version 4 format, which are used to give each snapshot a unique `id`.

```typescript
import { createLogger } from '@/lib/logs/console/logger'
```

*   **`import { createLogger } from '@/lib/logs/console/logger'`**: Imports a custom `createLogger` function from an internal logging utility. This allows the service to produce structured log messages (debug, info, error) for monitoring and debugging.

```typescript
import type {
  SnapshotService as ISnapshotService,
  SnapshotCreationResult,
  WorkflowExecutionSnapshot,
  WorkflowExecutionSnapshotInsert,
  WorkflowState,
} from '@/lib/logs/types'
```

*   **`import type { ... } from '@/lib/logs/types'`**: Imports various TypeScript type definitions from an internal types file. Using `import type` ensures these imports are stripped out at compile time, reducing bundle size if this were for a client-side application (though less critical for a server-side module like this).
    *   `SnapshotService as ISnapshotService`: An interface defining the contract that the `SnapshotService` class must fulfill. This promotes consistency and ensures all required methods are implemented.
    *   `SnapshotCreationResult`: A type describing the object returned after attempting to create a snapshot, indicating if it was new or reused.
    *   `WorkflowExecutionSnapshot`: The type representing a *read* snapshot record from the database, including database-managed fields like `createdAt`.
    *   `WorkflowExecutionSnapshotInsert`: The type representing the data structure required to *insert* a new snapshot record into the database. This usually omits database-generated fields like `createdAt`.
    *   `WorkflowState`: The core type representing the complete configuration and layout of a workflow (blocks, edges, etc.).

```typescript
const logger = createLogger('SnapshotService')
```

*   **`const logger = createLogger('SnapshotService')`**: Initializes a logger instance specifically for the `SnapshotService`. Log messages generated by this service will be tagged with `'SnapshotService'` for easier identification in logs.

---

#### `SnapshotService` Class Definition

```typescript
export class SnapshotService implements ISnapshotService {
```

*   **`export class SnapshotService implements ISnapshotService {`**: Declares a class named `SnapshotService` and exports it, making it available for other modules to use. The `implements ISnapshotService` clause ensures that this class adheres to the `ISnapshotService` interface, guaranteeing it implements all required methods with their specified signatures.

---

#### `createSnapshot` Method

```typescript
  async createSnapshot(
    workflowId: string,
    state: WorkflowState
  ): Promise<WorkflowExecutionSnapshot> {
    const result = await this.createSnapshotWithDeduplication(workflowId, state)
    return result.snapshot
  }
```

*   **`async createSnapshot(...)`**: An asynchronous method that takes a `workflowId` (string) and the `WorkflowState` object as input, and returns a `Promise` that resolves to a `WorkflowExecutionSnapshot`.
*   **`const result = await this.createSnapshotWithDeduplication(workflowId, state)`**: This method is a simple wrapper. It delegates the actual work (including deduplication logic) to `createSnapshotWithDeduplication`. The `await` keyword pauses execution until that asynchronous call completes.
*   **`return result.snapshot`**: Returns the `snapshot` property from the `SnapshotCreationResult` returned by the deduplication method. This `createSnapshot` method effectively hides the `isNew` flag from its callers, providing a simpler API that just returns the snapshot.

---

#### `createSnapshotWithDeduplication` Method (Core Logic)

```typescript
  async createSnapshotWithDeduplication(
    workflowId: string,
    state: WorkflowState
  ): Promise<SnapshotCreationResult> {
```

*   **`async createSnapshotWithDeduplication(...)`**: An asynchronous method that takes `workflowId` and `state`, similar to `createSnapshot`. However, it returns a `Promise` that resolves to a `SnapshotCreationResult`, which includes both the `snapshot` itself and a boolean `isNew` indicating if a new record was created.

```typescript
    // Hash the position-less state for deduplication (functional equivalence)
    const stateHash = this.computeStateHash(state)
```

*   **`// Hash the position-less state for deduplication (functional equivalence)`**: A comment explaining the purpose of the next line: generating a hash that represents the *functional* state, ignoring visual details.
*   **`const stateHash = this.computeStateHash(state)`**: Calls a private helper method `computeStateHash` with the `WorkflowState`. This method performs the normalization and hashing discussed earlier. The resulting SHA256 hash string is stored in `stateHash`.

```typescript
    const existingSnapshot = await this.getSnapshotByHash(workflowId, stateHash)
    if (existingSnapshot) {
      logger.debug(`Reusing existing snapshot for workflow ${workflowId} with hash ${stateHash}`)
      return {
        snapshot: existingSnapshot,
        isNew: false,
      }
    }
```

*   **`const existingSnapshot = await this.getSnapshotByHash(workflowId, stateHash)`**: Calls another asynchronous helper method `getSnapshotByHash` to check if a snapshot with this specific `workflowId` and `stateHash` already exists in the database.
*   **`if (existingSnapshot)`**: If `getSnapshotByHash` returns a non-null value (meaning an existing snapshot was found):
    *   **`logger.debug(...)`**: Logs a debug message indicating that an existing snapshot is being reused.
    *   **`return { snapshot: existingSnapshot, isNew: false }`**: Returns an object conforming to `SnapshotCreationResult`. It includes the `existingSnapshot` and sets `isNew` to `false`, signaling that no new record was created. This completes the deduplication logic.

```typescript
    // Store the FULL state (including positions) so we can recreate the exact workflow
    // Even though we hash without positions, we want to preserve the complete state
    const snapshotData: WorkflowExecutionSnapshotInsert = {
      id: uuidv4(),
      workflowId,
      stateHash,
      stateData: state, // Full state with positions, subblock values, etc.
    }
```

*   **`// Store the FULL state ...`**: Comments explaining that even though the hash ignores positions, the actual stored data preserves the *full* `WorkflowState` for exact recreation.
*   **`const snapshotData: WorkflowExecutionSnapshotInsert = { ... }`**: If no existing snapshot was found (the `if (existingSnapshot)` block was skipped), this block prepares the data for a new database insertion.
    *   **`id: uuidv4()`**: Generates a new, unique ID for this snapshot using `uuidv4()`.
    *   **`workflowId`**: Uses the `workflowId` passed to the method.
    *   **`stateHash`**: Uses the `stateHash` computed earlier.
    *   **`stateData: state`**: Crucially, stores the *entire* original `state` object (including positions and all other details) directly into the `stateData` column. The type `WorkflowExecutionSnapshotInsert` ensures type safety for this database insertion object.

```typescript
    const [newSnapshot] = await db
      .insert(workflowExecutionSnapshots)
      .values(snapshotData)
      .returning()
```

*   **`const [newSnapshot] = await db.insert(...)`**: This is a Drizzle ORM query to insert the `snapshotData` into the `workflowExecutionSnapshots` table.
    *   **`db.insert(workflowExecutionSnapshots)`**: Starts an insert query for the `workflowExecutionSnapshots` table.
    *   **`.values(snapshotData)`**: Specifies the values to be inserted, using the `snapshotData` object.
    *   **`.returning()`**: Configures the Drizzle query to return the *inserted row*. Since we're inserting a single row, Drizzle returns an array containing that single row. The `[newSnapshot]` syntax uses array destructuring to extract that single returned row into the `newSnapshot` variable.

```typescript
    logger.debug(`Created new snapshot for workflow ${workflowId} with hash ${stateHash}`)
    logger.debug(`Stored full state with ${Object.keys(state.blocks || {}).length} blocks`)
    return {
      snapshot: {
        ...newSnapshot,
        stateData: newSnapshot.stateData as WorkflowState,
        createdAt: newSnapshot.createdAt.toISOString(),
      },
      isNew: true,
    }
  }
```

*   **`logger.debug(...)`**: Logs debug messages confirming a new snapshot was created and providing some basic stats (number of blocks).
*   **`return { snapshot: { ... }, isNew: true }`**: Returns the `SnapshotCreationResult` object.
    *   **`snapshot: { ...newSnapshot, ... }`**: The `newSnapshot` object returned by Drizzle might have `stateData` as `any` or `JSONB` type, and `createdAt` as a `Date` object.
        *   **`...newSnapshot`**: Spreads all properties from the `newSnapshot` object returned by the database.
        *   **`stateData: newSnapshot.stateData as WorkflowState`**: Explicitly casts `stateData` to `WorkflowState`. This is a type assertion, telling TypeScript that even though the database might return it as a generic JSON type, we know it's actually a `WorkflowState`.
        *   **`createdAt: newSnapshot.createdAt.toISOString()`**: Converts the `createdAt` `Date` object (returned by Drizzle) into an ISO 8601 string. This ensures consistency in how dates are exposed by the service.
    *   **`isNew: true`**: Indicates that a new snapshot record was indeed created in the database.

---

#### `getSnapshot` Method

```typescript
  async getSnapshot(id: string): Promise<WorkflowExecutionSnapshot | null> {
    const [snapshot] = await db
      .select()
      .from(workflowExecutionSnapshots)
      .where(eq(workflowExecutionSnapshots.id, id))
      .limit(1)

    if (!snapshot) return null

    return {
      ...snapshot,
      stateData: snapshot.stateData as WorkflowState,
      createdAt: snapshot.createdAt.toISOString(),
    }
  }
```

*   **`async getSnapshot(id: string): Promise<WorkflowExecutionSnapshot | null>`**: An asynchronous method to retrieve a snapshot by its unique `id`. It returns a `Promise` resolving to a `WorkflowExecutionSnapshot` or `null` if not found.
*   **`const [snapshot] = await db.select()...`**: This is a Drizzle ORM query to select a snapshot.
    *   **`db.select()`**: Starts a select query. By default, it selects all columns.
    *   **`.from(workflowExecutionSnapshots)`**: Specifies the table to select from.
    *   **`.where(eq(workflowExecutionSnapshots.id, id))`**: Filters the results to find a row where the `id` column matches the provided `id`. `eq` ensures type safety for the comparison.
    *   **`.limit(1)`**: Ensures that only one result (at most) is returned, as `id` is expected to be unique.
    *   Again, array destructuring `[snapshot]` extracts the single result if found.
*   **`if (!snapshot) return null`**: If no snapshot was found (the array was empty), `snapshot` will be `undefined`, so `null` is returned.
*   **`return { ...snapshot, ... }`**: If a snapshot is found, it's processed similarly to the `createSnapshotWithDeduplication` method:
    *   Spreads the `snapshot` properties.
    *   Type asserts `stateData` to `WorkflowState`.
    *   Converts `createdAt` to an ISO string.

---

#### `getSnapshotByHash` Method

```typescript
  async getSnapshotByHash(
    workflowId: string,
    hash: string
  ): Promise<WorkflowExecutionSnapshot | null> {
    const [snapshot] = await db
      .select()
      .from(workflowExecutionSnapshots)
      .where(
        and(
          eq(workflowExecutionSnapshots.workflowId, workflowId),
          eq(workflowExecutionSnapshots.stateHash, hash)
        )
      )
      .limit(1)

    if (!snapshot) return null

    return {
      ...snapshot,
      stateData: snapshot.stateData as WorkflowState,
      createdAt: snapshot.createdAt.toISOString(),
    }
  }
```

*   **`async getSnapshotByHash(...)`**: An asynchronous method to retrieve a snapshot by `workflowId` and `stateHash`. This is used by the deduplication logic.
*   **`const [snapshot] = await db.select()...`**: Similar Drizzle select query.
    *   **`.where(and(eq(...), eq(...)))`**: The key difference here is the `where` clause. It uses `and` to combine two conditions:
        *   `eq(workflowExecutionSnapshots.workflowId, workflowId)`: `workflowId` must match.
        *   `eq(workflowExecutionSnapshots.stateHash, hash)`: `stateHash` must match.
    *   **`.limit(1)`**: Again, limits to one result.
*   The rest of the method (`if (!snapshot) return null` and the return object transformation) is identical to `getSnapshot`.

---

#### `computeStateHash` Method

```typescript
  computeStateHash(state: WorkflowState): string {
    const normalizedState = this.normalizeStateForHashing(state)
    const stateString = this.normalizedStringify(normalizedState)
    return createHash('sha256').update(stateString).digest('hex')
  }
```

*   **`computeStateHash(state: WorkflowState): string`**: A synchronous method that takes a `WorkflowState` and returns its SHA256 hash as a string. This is the orchestrator for the hashing process.
*   **`const normalizedState = this.normalizeStateForHashing(state)`**: Calls the private `normalizeStateForHashing` method to transform the raw `state` into a canonical, position-agnostic representation.
*   **`const stateString = this.normalizedStringify(normalizedState)`**: Takes the `normalizedState` object and converts it into a consistent, sorted JSON-like string using the private `normalizedStringify` method. This is crucial for ensuring identical objects always produce the same string.
*   **`return createHash('sha256').update(stateString).digest('hex')`**:
    *   **`createHash('sha256')`**: Creates a new SHA256 hash object.
    *   **`.update(stateString)`**: Feeds the `stateString` into the hash algorithm.
    *   **`.digest('hex')`**: Computes the final hash and returns it as a hexadecimal string.

---

#### `cleanupOrphanedSnapshots` Method

```typescript
  async cleanupOrphanedSnapshots(olderThanDays: number): Promise<number> {
    const cutoffDate = new Date()
    cutoffDate.setDate(cutoffDate.getDate() - olderThanDays)

    const deletedSnapshots = await db
      .delete(workflowExecutionSnapshots)
      .where(lt(workflowExecutionSnapshots.createdAt, cutoffDate))
      .returning({ id: workflowExecutionSnapshots.id })

    const deletedCount = deletedSnapshots.length
    logger.info(`Cleaned up ${deletedCount} orphaned snapshots older than ${olderThanDays} days`)
    return deletedCount
  }
```

*   **`async cleanupOrphanedSnapshots(olderThanDays: number): Promise<number>`**: An asynchronous method to delete old snapshots. It takes a number of days (`olderThanDays`) and returns a `Promise` resolving to the count of deleted snapshots.
*   **`const cutoffDate = new Date()`**: Creates a `Date` object representing the current date and time.
*   **`cutoffDate.setDate(cutoffDate.getDate() - olderThanDays)`**: Modifies `cutoffDate` to be `olderThanDays` days in the past. This sets the threshold for deletion.
*   **`const deletedSnapshots = await db.delete(...).returning(...)`**: A Drizzle ORM query to delete snapshots.
    *   **`db.delete(workflowExecutionSnapshots)`**: Starts a delete query for the `workflowExecutionSnapshots` table.
    *   **`.where(lt(workflowExecutionSnapshots.createdAt, cutoffDate))`**: Filters the rows to be deleted. `lt` (less than) ensures that only snapshots whose `createdAt` timestamp is *before* the `cutoffDate` are targeted.
    *   **`.returning({ id: workflowExecutionSnapshots.id })`**: Configures Drizzle to return the `id` of each deleted row.
*   **`const deletedCount = deletedSnapshots.length`**: Counts how many snapshots were deleted based on the length of the `deletedSnapshots` array.
*   **`logger.info(...)`**: Logs an informational message about the cleanup operation and the number of items deleted.
*   **`return deletedCount`**: Returns the total count of deleted snapshots.

---

#### `private normalizeStateForHashing` Method (Deep Normalization)

```typescript
  private normalizeStateForHashing(state: WorkflowState): any {
    // Use the same normalization logic as hasWorkflowChanged for consistency

    // 1. Normalize edges (same as hasWorkflowChanged)
    const normalizedEdges = (state.edges || [])
      .map((edge) => ({
        source: edge.source,
        sourceHandle: edge.sourceHandle,
        target: edge.target,
        targetHandle: edge.targetHandle,
      }))
      .sort((a, b) =>
        `${a.source}-${a.sourceHandle}-${a.target}-${a.targetHandle}`.localeCompare(
          `${b.source}-${b.sourceHandle}-${b.target}-${b.targetHandle}`
        )
      )
```

*   **`private normalizeStateForHashing(state: WorkflowState): any`**: A private helper method used by `computeStateHash`. It takes the raw `WorkflowState` and returns a deeply normalized version of it. The `any` return type is used because the precise structure might vary, and the goal is a hashable representation, not a strict type.
*   **`// Use the same normalization logic...`**: A comment highlighting consistency with other parts of the application.
*   **`const normalizedEdges = (state.edges || []) ...`**: Normalizes the array of edges.
    *   **`(state.edges || [])`**: Ensures that `edges` is an array, even if `state.edges` is `null` or `undefined`.
    *   **`.map((edge) => ({ ... }))`**: Creates a new array where each edge object is transformed. It explicitly selects only the `source`, `sourceHandle`, `target`, and `targetHandle` properties. This *removes any other properties* that might exist on an edge (like `id` or visual metadata) that are not relevant to its functional connection.
    *   **`.sort((a, b) => ...)`**: Sorts the normalized edges. This is **critically important** for hashing. If the order of edges in the array changes but the actual connections remain the same, a simple JSON stringify would produce a different string, leading to a different hash. By sorting them based on a combined string of their functional properties, the order becomes canonical.
        *   The sort key `\`${a.source}-${a.sourceHandle}-${a.target}-${a.targetHandle}\``.localeCompare(...)` creates a unique string identifier for each edge and compares them lexicographically to ensure a consistent sort order.

```typescript
    // 2. Normalize blocks (same as hasWorkflowChanged)
    const normalizedBlocks: Record<string, any> = {}

    for (const [blockId, block] of Object.entries(state.blocks || {})) {
      // Skip position as it doesn't affect functionality
      const { position, ...blockWithoutPosition } = block

      // Handle subBlocks with detailed comparison (same as hasWorkflowChanged)
      const subBlocks = blockWithoutPosition.subBlocks || {}
      const normalizedSubBlocks: Record<string, any> = {}

      for (const [subBlockId, subBlock] of Object.entries(subBlocks)) {
        // Normalize value with special handling for null/undefined
        const value = subBlock.value ?? null

        normalizedSubBlocks[subBlockId] = {
          type: subBlock.type,
          value: this.normalizeValue(value),
          // Include other properties except value and type
          ...Object.fromEntries(
            Object.entries(subBlock).filter(([key]) => key !== 'value' && key !== 'type')
          ),
        }
      }

      normalizedBlocks[blockId] = {
        ...blockWithoutPosition,
        subBlocks: normalizedSubBlocks,
      }
    }
```

*   **`const normalizedBlocks: Record<string, any> = {}`**: Initializes an empty object to store the normalized blocks.
*   **`for (const [blockId, block] of Object.entries(state.blocks || {}))`**: Iterates over each block in the `state.blocks` object (or an empty object if `blocks` is `null`/`undefined`).
*   **`const { position, ...blockWithoutPosition } = block`**: Uses object destructuring to extract the `position` property and create a new `blockWithoutPosition` object that contains all other properties of the block *except* `position`. This is the core step for ignoring visual layout.
*   **`const subBlocks = blockWithoutPosition.subBlocks || {}`**: Gets the `subBlocks` for the current block, or an empty object if none exist.
*   **`const normalizedSubBlocks: Record<string, any> = {}`**: Initializes an empty object for normalized sub-blocks.
*   **`for (const [subBlockId, subBlock] of Object.entries(subBlocks))`**: Iterates over each sub-block.
*   **`const value = subBlock.value ?? null`**: Retrieves the `value` of the sub-block, defaulting to `null` if it's `undefined`. This ensures consistent representation of missing values.
*   **`normalizedSubBlocks[subBlockId] = { ... }`**: Constructs the normalized representation for the current sub-block:
    *   **`type: subBlock.type`**: Includes the `type`.
    *   **`value: this.normalizeValue(value)`**: Calls the private `normalizeValue` helper (explained next) to recursively normalize the `value` property itself. This handles nested objects/arrays within values.
    *   **`...Object.fromEntries(Object.entries(subBlock).filter(...))`**: Spreads all *other* properties of the sub-block *except* `value` and `type`. This ensures that only relevant properties are included in the hash, and they are processed correctly.
*   **`normalizedBlocks[blockId] = { ...blockWithoutPosition, subBlocks: normalizedSubBlocks, }`**: Assigns the normalized block data to `normalizedBlocks`, merging the `blockWithoutPosition` with the newly normalized `subBlocks`.

```typescript
    // 3. Normalize loops and parallels
    const normalizedLoops: Record<string, any> = {}
    for (const [loopId, loop] of Object.entries(state.loops || {})) {
      normalizedLoops[loopId] = this.normalizeValue(loop)
    }

    const normalizedParallels: Record<string, any> = {}
    for (const [parallelId, parallel] of Object.entries(state.parallels || {})) {
      normalizedParallels[parallelId] = this.normalizeValue(parallel)
    }

    return {
      blocks: normalizedBlocks,
      edges: normalizedEdges,
      loops: normalizedLoops,
      parallels: normalizedParallels,
    }
  }
```

*   **`const normalizedLoops: Record<string, any> = {}`** and **`const normalizedParallels: Record<string, any> = {}`**: Initializes empty objects for normalized loops and parallels.
*   **`for (const [loopId, loop] of Object.entries(state.loops || {}))`** and **`for (const [parallelId, parallel] of Object.entries(state.parallels || {}))`**: Iterates through `loops` and `parallels` (if they exist).
*   **`normalizedLoops[loopId] = this.normalizeValue(loop)`** and **`normalizedParallels[parallelId] = this.normalizeValue(parallel)`**: For each loop/parallel, its entire value is passed to `this.normalizeValue` for recursive normalization.
*   **`return { blocks: normalizedBlocks, edges: normalizedEdges, loops: normalizedLoops, parallels: normalizedParallels, }`**: Returns the final, top-level normalized state object, containing the normalized `blocks`, `edges`, `loops`, and `parallels`. This object is then passed to `normalizedStringify`.

---

#### `private normalizeValue` Method (Recursive Value Normalization)

```typescript
  private normalizeValue(value: any): any {
    // Handle null/undefined consistently
    if (value === null || value === undefined) return null

    // Handle arrays
    if (Array.isArray(value)) {
      return value.map((item) => this.normalizeValue(item))
    }

    // Handle objects
    if (typeof value === 'object') {
      const normalized: Record<string, any> = {}
      for (const [key, val] of Object.entries(value)) {
        normalized[key] = this.normalizeValue(val)
      }
      return normalized
    }

    // Handle primitives
    return value
  }
```

*   **`private normalizeValue(value: any): any`**: A recursive private helper method that ensures consistent representation of various data types, especially within complex nested structures. This is called from `normalizeStateForHashing` for `value` properties within sub-blocks, and for `loops` and `parallels` themselves.
*   **`if (value === null || value === undefined) return null`**: Standardizes `null` and `undefined` to `null`. This prevents minor differences in how "empty" values are represented from causing different hashes.
*   **`if (Array.isArray(value))`**: If the value is an array:
    *   **`return value.map((item) => this.normalizeValue(item))`**: Recursively calls `normalizeValue` on each item in the array. This handles arrays of objects, arrays of arrays, etc.
*   **`if (typeof value === 'object')`**: If the value is an object (but not an array, as that's handled above):
    *   **`const normalized: Record<string, any> = {}`**: Initializes an empty object for the normalized version.
    *   **`for (const [key, val] of Object.entries(value))`**: Iterates over each key-value pair in the object.
    *   **`normalized[key] = this.normalizeValue(val)`**: Recursively calls `normalizeValue` on the value associated with each key. This ensures deep normalization of nested objects.
    *   **`return normalized`**: Returns the deeply normalized object.
*   **`return value`**: If none of the above conditions are met (i.e., it's a primitive like string, number, boolean), the value is returned as is.

---

#### `private normalizedStringify` Method (Consistent Stringification)

```typescript
  private normalizedStringify(obj: any): string {
    if (obj === null || obj === undefined) return 'null'
    if (typeof obj === 'string') return `"${obj}"`
    if (typeof obj === 'number' || typeof obj === 'boolean') return String(obj)

    if (Array.isArray(obj)) {
      return `[${obj.map((item) => this.normalizedStringify(item)).join(',')}]`
    }

    if (typeof obj === 'object') {
      const keys = Object.keys(obj).sort()
      const pairs = keys.map((key) => `"${key}":${this.normalizedStringify(obj[key])}`)
      return `{${pairs.join(',')}}`
    }

    return String(obj)
  }
```

*   **`private normalizedStringify(obj: any): string`**: A recursive private helper method that converts a JavaScript object into a JSON-like string in a completely deterministic way. This is essential because standard `JSON.stringify()` doesn't guarantee a consistent key order for objects, which would lead to different hashes for functionally identical objects.
*   **`if (obj === null || obj === undefined) return 'null'`**: Handles `null` and `undefined` by returning the string `'null'`.
*   **`if (typeof obj === 'string') return \`"${obj}"\``**: For strings, it returns them wrapped in double quotes, just like JSON.
*   **`if (typeof obj === 'number' || typeof obj === 'boolean') return String(obj)`**: For numbers and booleans, it converts them directly to their string representation.
*   **`if (Array.isArray(obj))`**: If it's an array:
    *   **`return \`[${obj.map((item) => this.normalizedStringify(item)).join(',')}]\``**: Recursively stringifies each item in the array, joins them with commas, and wraps the whole thing in square brackets. The order of items in arrays is preserved.
*   **`if (typeof obj === 'object')`**: If it's an object (and not an array):
    *   **`const keys = Object.keys(obj).sort()`**: This is the **most crucial part** for objects. It gets all keys of the object and **sorts them alphabetically**. This ensures that `{ a: 1, b: 2 }` and `{ b: 2, a: 1 }` (which are functionally identical but might have different internal iteration orders) will always produce the same string representation.
    *   **`const pairs = keys.map((key) => \`"${key}":${this.normalizedStringify(obj[key])}\`)`**: For each sorted key, it recursively stringifies the key itself (wrapped in quotes) and its corresponding value, joining them with a colon.
    *   **`return \`{${pairs.join(',')}}\``**: Joins all the key-value pairs with commas and wraps the result in curly braces.
*   **`return String(obj)`**: A fallback for any other unexpected types, converting them to a string.

---

#### Service Instantiation

```typescript
export const snapshotService = new SnapshotService()
```

*   **`export const snapshotService = new SnapshotService()`**: Creates a single instance of the `SnapshotService` class and exports it. This is a common pattern for "singleton" services in Node.js applications, meaning there's only one instance of the service available throughout the application, which manages its own state (like the logger) and provides its functionalities.

---

This detailed breakdown covers every aspect of the `SnapshotService`, from its overarching purpose to the intricate logic behind its state normalization and hashing.