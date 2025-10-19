```typescript
import { randomUUID } from 'crypto'
import { db } from '@sim/db'
import { document, knowledgeBase, permissions } from '@sim/db/schema'
import { and, count, eq, isNotNull, isNull, or } from 'drizzle-orm'
import type {
  ChunkingConfig,
  CreateKnowledgeBaseData,
  KnowledgeBaseWithCounts,
} from '@/lib/knowledge/types'
import { createLogger } from '@/lib/logs/console/logger'
import { getUserEntityPermissions } from '@/lib/permissions/utils'

const logger = createLogger('KnowledgeBaseService')

/**
 * Get knowledge bases that a user can access
 */
export async function getKnowledgeBases(
  userId: string,
  workspaceId?: string | null
): Promise<KnowledgeBaseWithCounts[]> {
  const knowledgeBasesWithCounts = await db
    .select({
      id: knowledgeBase.id,
      name: knowledgeBase.name,
      description: knowledgeBase.description,
      tokenCount: knowledgeBase.tokenCount,
      embeddingModel: knowledgeBase.embeddingModel,
      embeddingDimension: knowledgeBase.embeddingDimension,
      chunkingConfig: knowledgeBase.chunkingConfig,
      createdAt: knowledgeBase.createdAt,
      updatedAt: knowledgeBase.updatedAt,
      workspaceId: knowledgeBase.workspaceId,
      docCount: count(document.id),
    })
    .from(knowledgeBase)
    .leftJoin(
      document,
      and(eq(document.knowledgeBaseId, knowledgeBase.id), isNull(document.deletedAt))
    )
    .leftJoin(
      permissions,
      and(
        eq(permissions.entityType, 'workspace'),
        eq(permissions.entityId, knowledgeBase.workspaceId),
        eq(permissions.userId, userId)
      )
    )
    .where(
      and(
        isNull(knowledgeBase.deletedAt),
        workspaceId
          ? // When filtering by workspace
            or(
              // Knowledge bases belonging to the specified workspace (user must have workspace permissions)
              and(eq(knowledgeBase.workspaceId, workspaceId), isNotNull(permissions.userId)),
              // Fallback: User-owned knowledge bases without workspace (legacy)
              and(eq(knowledgeBase.userId, userId), isNull(knowledgeBase.workspaceId))
            )
          : // When not filtering by workspace, use original logic
            or(
              // User owns the knowledge base directly
              eq(knowledgeBase.userId, userId),
              // User has permissions on the knowledge base's workspace
              isNotNull(permissions.userId)
            )
      )
    )
    .groupBy(knowledgeBase.id)
    .orderBy(knowledgeBase.createdAt)

  return knowledgeBasesWithCounts.map((kb) => ({
    ...kb,
    chunkingConfig: kb.chunkingConfig as ChunkingConfig,
    docCount: Number(kb.docCount),
  }))
}

/**
 * Create a new knowledge base
 */
export async function createKnowledgeBase(
  data: CreateKnowledgeBaseData,
  requestId: string
): Promise<KnowledgeBaseWithCounts> {
  const kbId = randomUUID()
  const now = new Date()

  if (data.workspaceId) {
    const hasPermission = await getUserEntityPermissions(data.userId, 'workspace', data.workspaceId)
    if (hasPermission === null) {
      throw new Error('User does not have permission to create knowledge bases in this workspace')
    }
  }

  const newKnowledgeBase = {
    id: kbId,
    name: data.name,
    description: data.description ?? null,
    workspaceId: data.workspaceId ?? null,
    userId: data.userId,
    tokenCount: 0,
    embeddingModel: data.embeddingModel,
    embeddingDimension: data.embeddingDimension,
    chunkingConfig: data.chunkingConfig,
    createdAt: now,
    updatedAt: now,
    deletedAt: null,
  }

  await db.insert(knowledgeBase).values(newKnowledgeBase)

  logger.info(`[${requestId}] Created knowledge base: ${data.name} (${kbId})`)

  return {
    id: kbId,
    name: data.name,
    description: data.description ?? null,
    tokenCount: 0,
    embeddingModel: data.embeddingModel,
    embeddingDimension: data.embeddingDimension,
    chunkingConfig: data.chunkingConfig,
    createdAt: now,
    updatedAt: now,
    workspaceId: data.workspaceId ?? null,
    docCount: 0,
  }
}

/**
 * Update a knowledge base
 */
export async function updateKnowledgeBase(
  knowledgeBaseId: string,
  updates: {
    name?: string
    description?: string
    workspaceId?: string | null
    chunkingConfig?: {
      maxSize: number
      minSize: number
      overlap: number
    }
  },
  requestId: string
): Promise<KnowledgeBaseWithCounts> {
  const now = new Date()
  const updateData: {
    updatedAt: Date
    name?: string
    description?: string | null
    workspaceId?: string | null
    chunkingConfig?: {
      maxSize: number
      minSize: number
      overlap: number
    }
    embeddingModel?: string
    embeddingDimension?: number
  } = {
    updatedAt: now,
  }

  if (updates.name !== undefined) updateData.name = updates.name
  if (updates.description !== undefined) updateData.description = updates.description
  if (updates.workspaceId !== undefined) updateData.workspaceId = updates.workspaceId
  if (updates.chunkingConfig !== undefined) {
    updateData.chunkingConfig = updates.chunkingConfig
    updateData.embeddingModel = 'text-embedding-3-small'
    updateData.embeddingDimension = 1536
  }

  await db.update(knowledgeBase).set(updateData).where(eq(knowledgeBase.id, knowledgeBaseId))

  const updatedKb = await db
    .select({
      id: knowledgeBase.id,
      name: knowledgeBase.name,
      description: knowledgeBase.description,
      tokenCount: knowledgeBase.tokenCount,
      embeddingModel: knowledgeBase.embeddingModel,
      embeddingDimension: knowledgeBase.embeddingDimension,
      chunkingConfig: knowledgeBase.chunkingConfig,
      createdAt: knowledgeBase.createdAt,
      updatedAt: knowledgeBase.updatedAt,
      workspaceId: knowledgeBase.workspaceId,
      docCount: count(document.id),
    })
    .from(knowledgeBase)
    .leftJoin(
      document,
      and(eq(document.knowledgeBaseId, knowledgeBase.id), isNull(document.deletedAt))
    )
    .where(eq(knowledgeBase.id, knowledgeBaseId))
    .groupBy(knowledgeBase.id)
    .limit(1)

  if (updatedKb.length === 0) {
    throw new Error(`Knowledge base ${knowledgeBaseId} not found`)
  }

  logger.info(`[${requestId}] Updated knowledge base: ${knowledgeBaseId}`)

  return {
    ...updatedKb[0],
    chunkingConfig: updatedKb[0].chunkingConfig as ChunkingConfig,
    docCount: Number(updatedKb[0].docCount),
  }
}

/**
 * Get a single knowledge base by ID
 */
export async function getKnowledgeBaseById(
  knowledgeBaseId: string
): Promise<KnowledgeBaseWithCounts | null> {
  const result = await db
    .select({
      id: knowledgeBase.id,
      name: knowledgeBase.name,
      description: knowledgeBase.description,
      tokenCount: knowledgeBase.tokenCount,
      embeddingModel: knowledgeBase.embeddingModel,
      embeddingDimension: knowledgeBase.embeddingDimension,
      chunkingConfig: knowledgeBase.chunkingConfig,
      createdAt: knowledgeBase.createdAt,
      updatedAt: knowledgeBase.updatedAt,
      workspaceId: knowledgeBase.workspaceId,
      docCount: count(document.id),
    })
    .from(knowledgeBase)
    .leftJoin(
      document,
      and(eq(document.knowledgeBaseId, knowledgeBase.id), isNull(document.deletedAt))
    )
    .where(and(eq(knowledgeBase.id, knowledgeBaseId), isNull(knowledgeBase.deletedAt)))
    .groupBy(knowledgeBase.id)
    .limit(1)

  if (result.length === 0) {
    return null
  }

  return {
    ...result[0],
    chunkingConfig: result[0].chunkingConfig as ChunkingConfig,
    docCount: Number(result[0].docCount),
  }
}

/**
 * Delete a knowledge base (soft delete)
 */
export async function deleteKnowledgeBase(
  knowledgeBaseId: string,
  requestId: string
): Promise<void> {
  const now = new Date()

  await db
    .update(knowledgeBase)
    .set({
      deletedAt: now,
      updatedAt: now,
    })
    .where(eq(knowledgeBase.id, knowledgeBaseId))

  logger.info(`[${requestId}] Soft deleted knowledge base: ${knowledgeBaseId}`)
}
```

### Purpose of this file

This TypeScript file defines a service for managing knowledge bases within an application.  It provides functions for creating, reading, updating, and deleting (soft delete) knowledge bases. It handles database interactions using Drizzle ORM, manages user permissions, and logs activity.

### General Structure and Simplification

The code is structured into several asynchronous functions, each responsible for a specific operation on knowledge bases:

*   `getKnowledgeBases`: Retrieves knowledge bases accessible to a given user, considering workspace and direct ownership permissions.
*   `createKnowledgeBase`: Creates a new knowledge base, assigning it a unique ID and checking user permissions for workspace creation.
*   `updateKnowledgeBase`: Updates an existing knowledge base with new information.
*   `getKnowledgeBaseById`: Retrieves a single knowledge base by its ID.
*   `deleteKnowledgeBase`: Soft deletes a knowledge base by setting its `deletedAt` timestamp.

The code uses the following libraries/modules:

*   `crypto`:  Used for generating unique IDs for new knowledge bases.
*   `@sim/db`:  Presumably a custom module for database access using Drizzle ORM.
*   `@sim/db/schema`: Defines the database schema for tables like `document`, `knowledgeBase`, and `permissions`.
*   `drizzle-orm`: A TypeScript ORM used for interacting with the database.  It provides functions for constructing SQL queries.
*   `@/lib/knowledge/types`:  Defines TypeScript types related to knowledge bases, such as `ChunkingConfig`, `CreateKnowledgeBaseData`, and `KnowledgeBaseWithCounts`.
*   `@/lib/logs/console/logger`: A custom logger for recording application events.
*   `@/lib/permissions/utils`: A utility function for checking user permissions on entities.

### Line-by-line Explanation

**Imports:**

```typescript
import { randomUUID } from 'crypto'
import { db } from '@sim/db'
import { document, knowledgeBase, permissions } from '@sim/db/schema'
import { and, count, eq, isNotNull, isNull, or } from 'drizzle-orm'
import type {
  ChunkingConfig,
  CreateKnowledgeBaseData,
  KnowledgeBaseWithCounts,
} from '@/lib/knowledge/types'
import { createLogger } from '@/lib/logs/console/logger'
import { getUserEntityPermissions } from '@/lib/permissions/utils'
```

*   `import { randomUUID } from 'crypto'`: Imports the `randomUUID` function from the `crypto` module.  This function is used to generate unique IDs for new knowledge bases.
*   `import { db } from '@sim/db'`: Imports the database client instance (`db`) from a custom module `@sim/db`.  This client is used to execute database queries.
*   `import { document, knowledgeBase, permissions } from '@sim/db/schema'`: Imports the database table schema definitions (`document`, `knowledgeBase`, `permissions`) from a custom module `@sim/db/schema`. These are used to construct queries with Drizzle ORM.  These likely represent the database tables for documents, knowledge bases, and permissions, respectively.
*   `import { and, count, eq, isNotNull, isNull, or } from 'drizzle-orm'`: Imports functions from the `drizzle-orm` library for building SQL queries.
    *   `and`, `or`:  Logical operators for combining conditions in `WHERE` clauses.
    *   `count`: An aggregate function for counting rows.
    *   `eq`:  An equality operator for comparing columns to values.
    *   `isNotNull`, `isNull`: Operators for checking if a column is (or is not) `NULL`.
*   `import type { ChunkingConfig, CreateKnowledgeBaseData, KnowledgeBaseWithCounts } from '@/lib/knowledge/types'`: Imports TypeScript types related to knowledge bases from a custom module `@/lib/knowledge/types`.  The `type` keyword ensures these are only used for type checking and are not included in the compiled JavaScript.
    *   `ChunkingConfig`: Defines the structure for chunking configuration (max size, min size, overlap).
    *   `CreateKnowledgeBaseData`: Defines the expected data structure for creating a new knowledge base.
    *   `KnowledgeBaseWithCounts`: Defines the data structure for a knowledge base, including a count of associated documents.
*   `import { createLogger } from '@/lib/logs/console/logger'`: Imports a function to create a logger instance from a custom module `@/lib/logs/console/logger`.
*   `import { getUserEntityPermissions } from '@/lib/permissions/utils'`: Imports a utility function to check user permissions on a specific entity (e.g., workspace) from a custom module `@/lib/permissions/utils`.

**Logger Initialization:**

```typescript
const logger = createLogger('KnowledgeBaseService')
```

*   Creates a logger instance named `KnowledgeBaseService`.  This allows for logging messages specifically related to this service, aiding in debugging and monitoring.

**`getKnowledgeBases` function:**

```typescript
/**
 * Get knowledge bases that a user can access
 */
export async function getKnowledgeBases(
  userId: string,
  workspaceId?: string | null
): Promise<KnowledgeBaseWithCounts[]> {
  const knowledgeBasesWithCounts = await db
    .select({
      id: knowledgeBase.id,
      name: knowledgeBase.name,
      description: knowledgeBase.description,
      tokenCount: knowledgeBase.tokenCount,
      embeddingModel: knowledgeBase.embeddingModel,
      embeddingDimension: knowledgeBase.embeddingDimension,
      chunkingConfig: knowledgeBase.chunkingConfig,
      createdAt: knowledgeBase.createdAt,
      updatedAt: knowledgeBase.updatedAt,
      workspaceId: knowledgeBase.workspaceId,
      docCount: count(document.id),
    })
    .from(knowledgeBase)
    .leftJoin(
      document,
      and(eq(document.knowledgeBaseId, knowledgeBase.id), isNull(document.deletedAt))
    )
    .leftJoin(
      permissions,
      and(
        eq(permissions.entityType, 'workspace'),
        eq(permissions.entityId, knowledgeBase.workspaceId),
        eq(permissions.userId, userId)
      )
    )
    .where(
      and(
        isNull(knowledgeBase.deletedAt),
        workspaceId
          ? // When filtering by workspace
            or(
              // Knowledge bases belonging to the specified workspace (user must have workspace permissions)
              and(eq(knowledgeBase.workspaceId, workspaceId), isNotNull(permissions.userId)),
              // Fallback: User-owned knowledge bases without workspace (legacy)
              and(eq(knowledgeBase.userId, userId), isNull(knowledgeBase.workspaceId))
            )
          : // When not filtering by workspace, use original logic
            or(
              // User owns the knowledge base directly
              eq(knowledgeBase.userId, userId),
              // User has permissions on the knowledge base's workspace
              isNotNull(permissions.userId)
            )
      )
    )
    .groupBy(knowledgeBase.id)
    .orderBy(knowledgeBase.createdAt)

  return knowledgeBasesWithCounts.map((kb) => ({
    ...kb,
    chunkingConfig: kb.chunkingConfig as ChunkingConfig,
    docCount: Number(kb.docCount),
  }))
}
```

*   **Function Signature:**
    *   `export async function getKnowledgeBases(userId: string, workspaceId?: string | null): Promise<KnowledgeBaseWithCounts[]>`:  Defines an asynchronous function named `getKnowledgeBases` that takes a `userId` (string) and an optional `workspaceId` (string or null) as input and returns a promise that resolves to an array of `KnowledgeBaseWithCounts` objects.
*   **Database Query:**
    *   `const knowledgeBasesWithCounts = await db.select({...}).from(knowledgeBase)...`:  This is the core database query using Drizzle ORM.  It selects data from the `knowledgeBase` table and joins it with the `document` and `permissions` tables.
    *   `.select({...})`: Specifies the columns to select from the tables.  It includes all fields of the `knowledgeBase` table and a count of associated documents using `count(document.id)`.
    *   `.from(knowledgeBase)`: Specifies the primary table to query, which is `knowledgeBase`.
    *   `.leftJoin(document, and(eq(document.knowledgeBaseId, knowledgeBase.id), isNull(document.deletedAt)))`: Performs a left join with the `document` table based on `document.knowledgeBaseId` matching `knowledgeBase.id`.  It also filters for documents that haven't been soft-deleted (`isNull(document.deletedAt)`).  This join is used to count the number of active documents associated with each knowledge base.
    *   `.leftJoin(permissions, and(eq(permissions.entityType, 'workspace'), eq(permissions.entityId, knowledgeBase.workspaceId), eq(permissions.userId, userId)))`: Performs a left join with the `permissions` table. The join conditions ensure that the user (`userId`) has permissions to the `workspace` that the knowledge base belongs to.
    *   `.where(and(isNull(knowledgeBase.deletedAt), ...))`: Filters the results based on several conditions combined with the `and` operator.
        *   `isNull(knowledgeBase.deletedAt)`: Ensures that only knowledge bases that haven't been soft-deleted are returned.
        *   `workspaceId ? ... : ...`: A conditional check: if `workspaceId` is provided, it applies one set of `or` conditions; otherwise, it applies a different set. This allows for filtering by workspace.
        *   **If `workspaceId` is provided:**
            *   `or(and(eq(knowledgeBase.workspaceId, workspaceId), isNotNull(permissions.userId)), and(eq(knowledgeBase.userId, userId), isNull(knowledgeBase.workspaceId)))`:  Allows users to see knowledge bases in a workspace if they either have explicit workspace permissions via the `permissions` table (`isNotNull(permissions.userId)`) or if the knowledge base is directly owned by the user (`eq(knowledgeBase.userId, userId)`) and doesn't have a workspace associated with it (`isNull(knowledgeBase.workspaceId)`).  The latter is likely for handling legacy data.
        *   **If `workspaceId` is NOT provided:**
            *   `or(eq(knowledgeBase.userId, userId), isNotNull(permissions.userId))`: Allows users to see knowledge bases they own directly (`eq(knowledgeBase.userId, userId)`) or have permissions to via workspace permissions (`isNotNull(permissions.userId)`).
    *   `.groupBy(knowledgeBase.id)`: Groups the results by `knowledgeBase.id` so that the `count(document.id)` function works correctly, providing a count per knowledge base.
    *   `.orderBy(knowledgeBase.createdAt)`: Orders the results by the `createdAt` timestamp.
*   **Post-processing:**
    *   `.map((kb) => ({ ...kb, chunkingConfig: kb.chunkingConfig as ChunkingConfig, docCount: Number(kb.docCount) }))`:  Transforms the result array.
        *   It casts the `chunkingConfig` field to the `ChunkingConfig` type for better type safety.
        *   It converts the `docCount` (which is likely returned as a bigint or string by the database) to a Number.

**`createKnowledgeBase` function:**

```typescript
/**
 * Create a new knowledge base
 */
export async function createKnowledgeBase(
  data: CreateKnowledgeBaseData,
  requestId: string
): Promise<KnowledgeBaseWithCounts> {
  const kbId = randomUUID()
  const now = new Date()

  if (data.workspaceId) {
    const hasPermission = await getUserEntityPermissions(data.userId, 'workspace', data.workspaceId)
    if (hasPermission === null) {
      throw new Error('User does not have permission to create knowledge bases in this workspace')
    }
  }

  const newKnowledgeBase = {
    id: kbId,
    name: data.name,
    description: data.description ?? null,
    workspaceId: data.workspaceId ?? null,
    userId: data.userId,
    tokenCount: 0,
    embeddingModel: data.embeddingModel,
    embeddingDimension: data.embeddingDimension,
    chunkingConfig: data.chunkingConfig,
    createdAt: now,
    updatedAt: now,
    deletedAt: null,
  }

  await db.insert(knowledgeBase).values(newKnowledgeBase)

  logger.info(`[${requestId}] Created knowledge base: ${data.name} (${kbId})`)

  return {
    id: kbId,
    name: data.name,
    description: data.description ?? null,
    tokenCount: 0,
    embeddingModel: data.embeddingModel,
    embeddingDimension: data.embeddingDimension,
    chunkingConfig: data.chunkingConfig,
    createdAt: now,
    updatedAt: now,
    workspaceId: data.workspaceId ?? null,
    docCount: 0,
  }
}
```

*   **Function Signature:**
    *   `export async function createKnowledgeBase(data: CreateKnowledgeBaseData, requestId: string): Promise<KnowledgeBaseWithCounts>`:  Defines an asynchronous function named `createKnowledgeBase` that takes `data` (of type `CreateKnowledgeBaseData`) and `requestId` (string) as input and returns a promise that resolves to a `KnowledgeBaseWithCounts` object.
*   **ID and Timestamp Generation:**
    *   `const kbId = randomUUID()`: Generates a unique ID for the new knowledge base using the `randomUUID` function.
    *   `const now = new Date()`: Gets the current date and time.
*   **Permission Check:**
    *   `if (data.workspaceId) { ... }`: Checks if a `workspaceId` is provided in the input data.  If so, it proceeds to check user permissions.
    *   `const hasPermission = await getUserEntityPermissions(data.userId, 'workspace', data.workspaceId)`: Calls the `getUserEntityPermissions` function to check if the user has permission to create knowledge bases in the specified workspace.
    *   `if (hasPermission === null) { throw new Error('User does not have permission to create knowledge bases in this workspace') }`: If the user doesn't have permission, it throws an error, preventing the creation of the knowledge base.
*   **Knowledge Base Object Creation:**
    *   `const newKnowledgeBase = { ... }`: Creates a new object `newKnowledgeBase` with the data from the input `data` and the generated `kbId` and `now`.  It also sets default values for fields like `tokenCount` (0) and `deletedAt` (null).
*   **Database Insertion:**
    *   `await db.insert(knowledgeBase).values(newKnowledgeBase)`:  Inserts the new knowledge base data into the `knowledgeBase` table using Drizzle ORM.
*   **Logging:**
    *   `logger.info(`[${requestId}] Created knowledge base: ${data.name} (${kbId})`)`: Logs the creation of the knowledge base, including the request ID, name, and ID of the knowledge base.
*   **Return Value:**
    *   Returns a `KnowledgeBaseWithCounts` object representing the newly created knowledge base.

**`updateKnowledgeBase` function:**

```typescript
/**
 * Update a knowledge base
 */
export async function updateKnowledgeBase(
  knowledgeBaseId: string,
  updates: {
    name?: string
    description?: string
    workspaceId?: string | null
    chunkingConfig?: {
      maxSize: number
      minSize: number
      overlap: number
    }
  },
  requestId: string
): Promise<KnowledgeBaseWithCounts> {
  const now = new Date()
  const updateData: {
    updatedAt: Date
    name?: string
    description?: string | null
    workspaceId?: string | null
    chunkingConfig?: {
      maxSize: number
      minSize: number
      overlap: number
    }
    embeddingModel?: string
    embeddingDimension?: number
  } = {
    updatedAt: now,
  }

  if (updates.name !== undefined) updateData.name = updates.name
  if (updates.description !== undefined) updateData.description = updates.description
  if (updates.workspaceId !== undefined) updateData.workspaceId = updates.workspaceId
  if (updates.chunkingConfig !== undefined) {
    updateData.chunkingConfig = updates.chunkingConfig
    updateData.embeddingModel = 'text-embedding-3-small'
    updateData.embeddingDimension = 1536
  }

  await db.update(knowledgeBase).set(updateData).where(eq(knowledgeBase.id, knowledgeBaseId))

  const updatedKb = await db
    .select({
      id: knowledgeBase.id,
      name: knowledgeBase.name,
      description: knowledgeBase.description,
      tokenCount: knowledgeBase.tokenCount,
      embeddingModel: knowledgeBase.embeddingModel,
      embeddingDimension: knowledgeBase.embeddingDimension,
      chunkingConfig: knowledgeBase.chunkingConfig,
      createdAt: knowledgeBase.createdAt,
      updatedAt: knowledgeBase.updatedAt,
      workspaceId: knowledgeBase.workspaceId,
      docCount: count(document.id),
    })
    .from(knowledgeBase)
    .leftJoin(
      document,
      and(eq(document.knowledgeBaseId, knowledgeBase.id), isNull(document.deletedAt))
    )
    .where(eq(knowledgeBase.id, knowledgeBaseId))
    .groupBy(knowledgeBase.id)
    .limit(1)

  if (updatedKb.length === 0) {
    throw new Error(`Knowledge base ${knowledgeBaseId} not found`)
  }

  logger.info(`[${requestId}] Updated knowledge base: ${knowledgeBaseId}`)

  return {
    ...updatedKb[0],
    chunkingConfig: updatedKb[0].chunkingConfig as ChunkingConfig,
    docCount: Number(updatedKb[0].docCount),
  }
}
```

*   **Function Signature:**
    *   `export async function updateKnowledgeBase(knowledgeBaseId: string, updates: { ... }, requestId: string): Promise<KnowledgeBaseWithCounts>`:  Defines an asynchronous function named `updateKnowledgeBase` that takes the `knowledgeBaseId` (string), `updates` (an object containing the fields to update), and `requestId` (string) as input, and returns a promise that resolves to a `KnowledgeBaseWithCounts` object.
*   **Timestamp and Update Data Initialization:**
    *   `const now = new Date()`: Gets the current date and time.
    *   `const updateData: { ... } = { updatedAt: now }`: Initializes an object `updateData` that will contain the fields to be updated in the database.  It starts with the `updatedAt` field set to the current timestamp.
*   **Conditional Updates:**
    *   `if (updates.name !== undefined) updateData.name = updates.name`:  Checks if the `name` field is present in the `updates` object.  If it is, it adds it to the `updateData` object. This pattern is repeated for `description`, `workspaceId` and `chunkingConfig`.
    *   `if (updates.chunkingConfig !== undefined) { ... }`: This block handles updates to the `chunkingConfig`. When the `chunkingConfig` is updated, the `embeddingModel` and `embeddingDimension` are also updated.
*   **Database Update:**
    *   `await db.update(knowledgeBase).set(updateData).where(eq(knowledgeBase.id, knowledgeBaseId))`: Updates the `knowledgeBase` table using Drizzle ORM.
        *   `.set(updateData)`: Specifies the fields to update with the values from the `updateData` object.
        *   `.where(eq(knowledgeBase.id, knowledgeBaseId))`: Specifies the condition for the update, ensuring that only the knowledge base with the matching `knowledgeBaseId` is updated.
*   **Retrieve Updated Knowledge Base:**
    *   The code then retrieves the updated knowledge base from the database using a `select` query similar to the `getKnowledgeBases` function, but it filters by `knowledgeBase.id` and limits the result to 1.
*   **Error Handling:**
    *   `if (updatedKb.length === 0) { throw new Error(`Knowledge base ${knowledgeBaseId} not found`) }`:  Checks if the updated knowledge base was found in the database. If not, it throws an error.
*   **Logging:**
    *   `logger.info(`[${requestId}] Updated knowledge base: ${knowledgeBaseId}`)`: Logs the update of the knowledge base.
*   **Return Value:**
    *   Returns a `KnowledgeBaseWithCounts` object representing the updated knowledge base.

**`getKnowledgeBaseById` function:**

```typescript
/**
 * Get a single knowledge base by ID
 */
export async function getKnowledgeBaseById(
  knowledgeBaseId: string
): Promise<KnowledgeBaseWithCounts | null> {
  const result = await db
    .select({
      id: knowledgeBase.id,
      name: knowledgeBase.name,
      description: knowledgeBase.description,
      tokenCount: knowledgeBase.tokenCount,
      embeddingModel: knowledgeBase.embeddingModel,
      embeddingDimension: knowledgeBase.embeddingDimension,
      chunkingConfig: knowledgeBase.chunkingConfig,
      createdAt: knowledgeBase.createdAt,
      updatedAt: knowledgeBase.updatedAt,
      workspaceId: knowledgeBase.workspaceId,
      docCount: count(document.id),
    })
    .from(knowledgeBase)
    .leftJoin(
      document,
      and(eq(document.knowledgeBaseId, knowledgeBase.id), isNull(document.deletedAt))
    )
    .where(and(eq(knowledgeBase.id, knowledgeBaseId), isNull(knowledgeBase.deletedAt)))
    .groupBy(knowledgeBase.id)
    .limit(1)

  if (result.length === 0) {
    return null
  }

  return {
    ...result[0],
    chunkingConfig: result[0].chunkingConfig as ChunkingConfig,
    docCount: Number(result[0].docCount),
  }
}
```

*   **Function Signature:**
    *   `export async function getKnowledgeBaseById(knowledgeBaseId: string): Promise<KnowledgeBaseWithCounts | null>`:  Defines an asynchronous function named `getKnowledgeBaseById` that takes the `knowledgeBaseId` (string) as input and returns a promise that resolves to a `KnowledgeBaseWithCounts` object or `null` if the knowledge base is not found.
*   **Database Query:**
    *   The code retrieves a single knowledge base from the database using a `select` query similar to the `getKnowledgeBases` and `updateKnowledgeBase` functions, but it filters by `knowledgeBase.id` and `isNull(knowledgeBase.deletedAt)` and limits the result to 1.
*   **Null Check:**
    *   `if (result.length === 0) { return null }`:  Checks if the knowledge base was found in the database. If not, it returns `null`.
*   **Return Value:**
    *   Returns a `KnowledgeBaseWithCounts` object representing the retrieved knowledge base, or null if not found.

**`deleteKnowledgeBase` function:**

```typescript
/**
 * Delete a knowledge base (soft delete)
 */
export async function deleteKnowledgeBase(
  knowledgeBaseId: string,
  requestId: string
): Promise<void> {
  const now = new Date()

  await db
    .update(knowledgeBase)
    .set({
      deletedAt: now,
      updatedAt: now,
    })
    .where(eq(knowledgeBase.id, knowledgeBaseId))

  logger.info(`[${requestId}] Soft deleted knowledge base: ${knowledgeBaseId}`)
}
```

*   **Function Signature:**
    *   `export async function deleteKnowledgeBase(knowledgeBaseId: string, requestId: string): Promise<void>`:  Defines an asynchronous function named `deleteKnowledgeBase` that takes the `knowledgeBaseId` (string) and `requestId` (string) as input and returns a promise that resolves to `void` (meaning it doesn't return any value).
*   **Timestamp:**
    *   `const now = new Date()`: Gets the current date and time.
*   **Database Update (Soft Delete):**
    *   `await db.update(knowledgeBase).set({ deletedAt: now, updatedAt: now }).where(eq(knowledgeBase.id, knowledgeBaseId))`: Updates the `knowledgeBase` table using Drizzle ORM to perform a soft delete.
        *   `.set({ deletedAt: now, updatedAt: now })`: Sets the `deletedAt` field to the current timestamp, indicating that the knowledge base has been deleted.  It also updates the `updatedAt` field.
        *   `.where(eq(knowledgeBase.id, knowledgeBaseId))`: Specifies the condition for the update, ensuring that only the knowledge base with the matching `knowledgeBaseId` is deleted.
*   **Logging:**
    *   `logger.info(`[${requestId}] Soft deleted knowledge base: ${knowledgeBaseId}`)`: Logs the soft deletion of the knowledge base.

### Summary

This file provides a complete service for managing knowledge bases, handling database interactions, permissions, and logging. The code is well-structured and uses TypeScript types effectively.  The use of Drizzle ORM provides a type-safe way to interact with the database. The code includes logging for auditing and debugging purposes.
