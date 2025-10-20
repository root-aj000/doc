This TypeScript file acts as a **service layer** for managing "chunks" of information within a knowledge base system. It provides a set of asynchronous functions to interact with a database (using Drizzle ORM) to create, query, update, and delete these chunks.

Each "chunk" represents a small, semantically meaningful piece of content derived from a larger `document`. These chunks are often used in AI/ML applications, particularly for Retrieval-Augmented Generation (RAG) systems, where they are accompanied by vector embeddings for similarity search.

### Key Responsibilities:
*   **Querying Chunks:** Retrieving chunks associated with a specific document, with support for filtering, searching, and pagination.
*   **Creating Chunks:** Adding new chunks to a document, which involves generating vector embeddings, estimating token counts, and updating parent document statistics.
*   **Updating Chunks:** Modifying existing chunks, including their content or enabled status. Updating content triggers re-embedding and token recalculation.
*   **Deleting Chunks:** Removing chunks, either individually or in batches, and adjusting parent document statistics accordingly.
*   **Batch Operations:** Performing mass enable/disable/delete actions on multiple chunks.
*   **Maintaining Data Consistency:** Many operations use database transactions to ensure that multiple related database updates (e.g., updating a chunk and then updating its parent document's statistics) are treated as a single, atomic unit.
*   **Logging:** Using a custom logger to record significant actions and aid in debugging.

---

## Detailed Code Explanation

Let's break down each part of the file, starting with imports and then each function.

### Imports

```typescript
import { createHash, randomUUID } from 'crypto'
import { db } from '@sim/db'
import { document, embedding } from '@sim/db/schema'
import { and, asc, eq, ilike, inArray, sql } from 'drizzle-orm'
import { generateEmbeddings } from '@/lib/embeddings/utils'
import type {
  BatchOperationResult,
  ChunkData,
  ChunkFilters,
  ChunkQueryResult,
  CreateChunkData,
} from '@/lib/knowledge/chunks/types'
import { createLogger } from '@/lib/logs/console/logger'
import { estimateTokenCount } from '@/lib/tokenization/estimators'
```

*   **`import { createHash, randomUUID } from 'crypto'`**:
    *   `createHash`: A Node.js built-in function used to create cryptographic hash objects. Here, it's used with `sha256` to generate a unique hash for chunk content, which can help detect if content has changed.
    *   `randomUUID`: Generates a cryptographically strong, version 4 UUID (Universally Unique Identifier). This is used to create unique IDs for new chunks.
*   **`import { db } from '@sim/db'`**:
    *   Imports the Drizzle ORM database client instance. This `db` object is the primary interface for all database interactions. The `@sim/db` likely refers to an alias or module path within the project.
*   **`import { document, embedding } from '@sim/db/schema'`**:
    *   Imports schema definitions for the `document` and `embedding` tables from the Drizzle ORM schema file. These objects represent the tables and their columns, allowing for type-safe database queries.
    *   `embedding` table likely stores the actual chunk data, including its content, metadata, and the vector embedding.
    *   `document` table stores metadata about the parent document, like total chunk count, token count, etc.
*   **`import { and, asc, eq, ilike, inArray, sql } from 'drizzle-orm'`**:
    *   These are utility functions and types from the Drizzle ORM library, used to construct SQL queries in a type-safe and programmatic way.
        *   `and`: Combines multiple conditions with a logical `AND`.
        *   `asc`: Specifies ascending order for `orderBy` clauses.
        *   `eq`: Checks for equality (e.g., `WHERE column = value`).
        *   `ilike`: Performs a case-insensitive `LIKE` comparison (e.g., `WHERE column ILIKE '%search%'`).
        *   `inArray`: Checks if a column's value is present in a given array (e.g., `WHERE column IN (value1, value2)`).
        *   `sql`: Allows embedding raw SQL expressions into Drizzle queries. Useful for database-specific functions or when Drizzle doesn't have a direct helper.
*   **`import { generateEmbeddings } from '@/lib/embeddings/utils'`**:
    *   Imports a utility function responsible for generating vector embeddings (numerical representations of text) from text content. This likely calls an external AI service (e.g., OpenAI, Cohere).
*   **`import type {...} from '@/lib/knowledge/chunks/types'`**:
    *   Imports various TypeScript types defining the structure of data related to chunks, such as:
        *   `BatchOperationResult`: The expected return type for batch operations.
        *   `ChunkData`: The structure of a chunk object returned from a query.
        *   `ChunkFilters`: The structure for filter parameters when querying chunks.
        *   `ChunkQueryResult`: The full result structure for a chunk query, including pagination.
        *   `CreateChunkData`: The input data required to create a new chunk.
    *   Using `type` ensures these imports are stripped during compilation, having no runtime impact.
*   **`import { createLogger } from '@/lib/logs/console/logger'`**:
    *   Imports a function to create a logger instance, used for outputting informational or debug messages to the console.
*   **`import { estimateTokenCount } from '@/lib/tokenization/estimators'`**:
    *   Imports a function to estimate the number of "tokens" in a given string. Tokens are pieces of words or characters used by large language models, and their count is crucial for cost estimation and model limits.

---

### Logger Initialization

```typescript
const logger = createLogger('ChunksService')
```

*   **`const logger = createLogger('ChunksService')`**: Initializes a logger instance specifically named 'ChunksService'. This helps in identifying logs originating from this particular service file, which is useful for debugging and monitoring.

---

### `queryChunks` Function

```typescript
/**
 * Query chunks for a document with filtering and pagination
 */
export async function queryChunks(
  documentId: string,
  filters: ChunkFilters,
  requestId: string
): Promise<ChunkQueryResult> {
  const { search, enabled = 'all', limit = 50, offset = 0 } = filters

  // Build query conditions
  const conditions = [eq(embedding.documentId, documentId)]

  // Add enabled filter
  if (enabled === 'true') {
    conditions.push(eq(embedding.enabled, true))
  } else if (enabled === 'false') {
    conditions.push(eq(embedding.enabled, false))
  }

  // Add search filter
  if (search) {
    conditions.push(ilike(embedding.content, `%${search}%`))
  }

  // Fetch chunks
  const chunks = await db
    .select({
      id: embedding.id,
      chunkIndex: embedding.chunkIndex,
      content: embedding.content,
      contentLength: embedding.contentLength,
      tokenCount: embedding.tokenCount,
      enabled: embedding.enabled,
      startOffset: embedding.startOffset,
      endOffset: embedding.endOffset,
      tag1: embedding.tag1,
      tag2: embedding.tag2,
      tag3: embedding.tag3,
      tag4: embedding.tag4,
      tag5: embedding.tag5,
      tag6: embedding.tag6,
      tag7: embedding.tag7,
      createdAt: embedding.createdAt,
      updatedAt: embedding.updatedAt,
    })
    .from(embedding)
    .where(and(...conditions))
    .orderBy(asc(embedding.chunkIndex))
    .limit(limit)
    .offset(offset)

  // Get total count for pagination
  const totalCount = await db
    .select({ count: sql`count(*)` })
    .from(embedding)
    .where(and(...conditions))

  logger.info(`[${requestId}] Retrieved ${chunks.length} chunks for document ${documentId}`)

  return {
    chunks: chunks as ChunkData[],
    pagination: {
      total: Number(totalCount[0]?.count || 0),
      limit,
      offset,
      hasMore: chunks.length === limit,
    },
  }
}
```

*   **Purpose:** This function retrieves a list of chunks belonging to a specific document, allowing for filtering, searching, and pagination.
*   **Simplified Logic:** It constructs database query conditions based on the provided filters, fetches a limited set of matching chunks, and separately counts all matching chunks to provide pagination metadata.
*   **Line-by-Line Explanation:**
    *   `export async function queryChunks(...)`: Declares an asynchronous function `queryChunks` that is exported for use by other modules.
        *   `documentId: string`: The ID of the parent document whose chunks are to be queried.
        *   `filters: ChunkFilters`: An object containing optional filtering, searching, and pagination parameters.
        *   `requestId: string`: A unique ID for the current request, used for logging and tracing.
        *   `Promise<ChunkQueryResult>`: Specifies that the function will return a Promise that resolves to a `ChunkQueryResult` object.
    *   `const { search, enabled = 'all', limit = 50, offset = 0 } = filters`: Destructures the `filters` object, providing default values for `enabled`, `limit`, and `offset` if they are not specified.
    *   `const conditions = [eq(embedding.documentId, documentId)]`: Initializes an array of Drizzle query conditions. The first condition ensures that only chunks belonging to the specified `documentId` are retrieved.
        *   `eq(embedding.documentId, documentId)`: Drizzle's `eq` (equals) function, creating a `WHERE documentId = '...'` clause. `embedding.documentId` refers to the `documentId` column in the `embedding` table.
    *   `if (enabled === 'true') { conditions.push(eq(embedding.enabled, true)) } else if (enabled === 'false') { conditions.push(eq(embedding.enabled, false)) }`: Adds conditions to filter chunks based on their `enabled` status. If `enabled` is 'all' (the default), no additional condition is added.
    *   `if (search) { conditions.push(ilike(embedding.content, `%${search}%`)) }`: If a `search` string is provided, a condition is added to search for that string within the `content` of the chunks (case-insensitive).
        *   `ilike(embedding.content, `%${search}%`)`: Drizzle's `ilike` function, creating a `WHERE content ILIKE '%search_term%'` clause for partial, case-insensitive matching.
    *   `const chunks = await db.select({...}).from(embedding).where(and(...conditions)).orderBy(asc(embedding.chunkIndex)).limit(limit).offset(offset)`: This is the main Drizzle ORM query to fetch the actual chunk data.
        *   `db.select({...})`: Specifies which columns to retrieve from the `embedding` table. It's a selective retrieval to get only the necessary fields for `ChunkData`.
        *   `.from(embedding)`: Specifies the table to query from, which is the `embedding` table.
        *   `.where(and(...conditions))`: Applies all the accumulated `conditions` using Drizzle's `and` function, which logically combines them (e.g., `WHERE condition1 AND condition2`). The spread operator `...conditions` unpacks the array of conditions.
        *   `.orderBy(asc(embedding.chunkIndex))`: Sorts the results by `chunkIndex` in ascending order.
        *   `.limit(limit)`: Restricts the number of rows returned to the `limit` specified (for pagination).
        *   `.offset(offset)`: Skips a specified number of rows before returning results (for pagination).
    *   `const totalCount = await db.select({ count: sql`count(*)` }).from(embedding).where(and(...conditions))`: This separate query is performed to get the *total number* of chunks matching the conditions, ignoring `limit` and `offset`. This is essential for calculating pagination metadata.
        *   `select({ count: sql`count(*)` })`: Selects a single column named `count`, whose value is the result of the raw SQL `count(*)` aggregate function. This `sql` tag template literal allows embedding raw SQL.
    *   `logger.info(...)`: Logs a message indicating how many chunks were retrieved for the given document and request.
    *   `return { chunks: chunks as ChunkData[], pagination: {...} }`: Returns an object containing the fetched `chunks` (type-casted to `ChunkData[]`) and `pagination` metadata.
        *   `total: Number(totalCount[0]?.count || 0)`: Extracts the total count from the `totalCount` result. Drizzle returns an array of objects for `select`, so `totalCount[0]?.count` accesses the count. `Number(...)` ensures it's a number, and `|| 0` handles cases where no count is returned.
        *   `hasMore: chunks.length === limit`: A boolean indicating if there might be more chunks to fetch. If the number of chunks returned is equal to the `limit`, it implies there could be more data beyond the current page.

---

### `createChunk` Function

```typescript
/**
 * Create a new chunk for a document
 */
export async function createChunk(
  knowledgeBaseId: string,
  documentId: string,
  docTags: Record<string, string | null>,
  chunkData: CreateChunkData,
  requestId: string
): Promise<ChunkData> {
  // Generate embedding for the content first (outside transaction for performance)
  logger.info(`[${requestId}] Generating embedding for manual chunk`)
  const embeddings = await generateEmbeddings([chunkData.content])

  // Calculate accurate token count
  const tokenCount = estimateTokenCount(chunkData.content, 'openai')

  const chunkId = randomUUID()
  const now = new Date()

  // Use transaction to atomically get next index and insert chunk
  const newChunk = await db.transaction(async (tx) => {
    // Get the next chunk index atomically within the transaction
    const lastChunk = await tx
      .select({ chunkIndex: embedding.chunkIndex })
      .from(embedding)
      .where(eq(embedding.documentId, documentId))
      .orderBy(sql`${embedding.chunkIndex} DESC`)
      .limit(1)

    const nextChunkIndex = lastChunk.length > 0 ? lastChunk[0].chunkIndex + 1 : 0

    const chunkDBData = {
      id: chunkId,
      knowledgeBaseId,
      documentId,
      chunkIndex: nextChunkIndex,
      chunkHash: createHash('sha256').update(chunkData.content).digest('hex'),
      content: chunkData.content,
      contentLength: chunkData.content.length,
      tokenCount: tokenCount.count,
      embedding: embeddings[0],
      embeddingModel: 'text-embedding-3-small',
      startOffset: 0, // Manual chunks don't have document offsets
      endOffset: chunkData.content.length,
      // Inherit tags from parent document
      tag1: docTags.tag1,
      tag2: docTags.tag2,
      tag3: docTags.tag3,
      tag4: docTags.tag4,
      tag5: docTags.tag5,
      tag6: docTags.tag6,
      tag7: docTags.tag7,
      enabled: chunkData.enabled ?? true,
      createdAt: now,
      updatedAt: now,
    }

    await tx.insert(embedding).values(chunkDBData)

    // Update document statistics
    await tx
      .update(document)
      .set({
        chunkCount: sql`${document.chunkCount} + 1`,
        tokenCount: sql`${document.tokenCount} + ${tokenCount.count}`,
        characterCount: sql`${document.characterCount} + ${chunkData.content.length}`,
      })
      .where(eq(document.id, documentId))

    return {
      id: chunkId,
      chunkIndex: nextChunkIndex,
      content: chunkData.content,
      contentLength: chunkData.content.length,
      tokenCount: tokenCount.count,
      enabled: chunkData.enabled ?? true,
      startOffset: 0,
      endOffset: chunkData.content.length,
      tag1: docTags.tag1,
      tag2: docTags.tag2,
      tag3: docTags.tag3,
      tag4: docTags.tag4,
      tag5: docTags.tag5,
      tag6: docTags.tag6,
      tag7: docTags.tag7,
      createdAt: now,
      updatedAt: now,
    } as ChunkData
  })

  logger.info(`[${requestId}] Created chunk ${chunkId} in document ${documentId}`)

  return newChunk
}
```

*   **Purpose:** This function creates a brand new chunk of content, generates its embedding, calculates its token count, assigns it an index, and stores it in the database while also updating the parent document's statistics.
*   **Simplified Logic:** It first generates the AI embedding and token count for the new chunk's content. Then, inside a database transaction, it determines the next available `chunkIndex` for the document, inserts the new chunk data, and finally updates the parent document's aggregate statistics (chunk count, token count, character count).
*   **Line-by-Line Explanation:**
    *   `export async function createChunk(...)`: Declares an asynchronous function `createChunk` that is exported.
        *   `knowledgeBaseId: string`, `documentId: string`: IDs for the associated knowledge base and document.
        *   `docTags: Record<string, string | null>`: An object containing tags inherited from the parent document.
        *   `chunkData: CreateChunkData`: The content and other initial data for the new chunk.
        *   `requestId: string`: Request ID for logging.
        *   `Promise<ChunkData>`: Returns the created chunk's data.
    *   `logger.info(...)`: Logs that embedding generation is starting.
    *   `const embeddings = await generateEmbeddings([chunkData.content])`: Calls the `generateEmbeddings` utility to get the vector embedding for the chunk's content. This is done *outside* the transaction because it's a potentially long-running external call and doesn't need to be part of the atomic database operation.
    *   `const tokenCount = estimateTokenCount(chunkData.content, 'openai')`: Estimates the number of tokens in the chunk's content using the `estimateTokenCount` utility. The `'openai'` argument likely specifies the tokenization model.
    *   `const chunkId = randomUUID()`: Generates a unique UUID for the new chunk's primary key.
    *   `const now = new Date()`: Gets the current timestamp, used for `createdAt` and `updatedAt`.
    *   `const newChunk = await db.transaction(async (tx) => {...})`: This is a crucial part. It executes a series of database operations within a Drizzle transaction.
        *   **Why a transaction?** To ensure atomicity. If any step within the transaction fails (e.g., getting the next index, inserting the chunk, or updating the document), the entire transaction is rolled back, leaving the database in its original state. This prevents inconsistent data.
        *   `async (tx) => {...}`: The function passed to `db.transaction` receives a `tx` object, which is a transaction-specific Drizzle client. All database operations within this block *must* use `tx` instead of `db` to be part of the transaction.
    *   `const lastChunk = await tx.select({...}).from(embedding).where(eq(embedding.documentId, documentId)).orderBy(sql`${embedding.chunkIndex} DESC`).limit(1)`: Inside the transaction, this query finds the chunk with the highest `chunkIndex` for the given document.
        *   `orderBy(sql`${embedding.chunkIndex} DESC`)`: Orders results by `chunkIndex` in descending order. `sql` is used here to ensure the order by clause is correctly interpreted.
        *   `limit(1)`: Fetches only the top result (the one with the highest index).
    *   `const nextChunkIndex = lastChunk.length > 0 ? lastChunk[0].chunkIndex + 1 : 0`: Calculates the next `chunkIndex`. If no previous chunks exist for the document, it starts at 0; otherwise, it's one more than the highest existing index.
    *   `const chunkDBData = {...}`: Creates an object containing all the data to be inserted into the `embedding` table.
        *   `chunkHash: createHash('sha256').update(chunkData.content).digest('hex')`: Generates a SHA256 hash of the chunk's content. This can be used for integrity checks or to detect duplicate content.
        *   `embedding: embeddings[0]`: Stores the generated vector embedding. `embeddings` is an array because `generateEmbeddings` can handle multiple texts, but here we only pass one.
        *   `embeddingModel: 'text-embedding-3-small'`: Records the name of the model used to generate the embedding.
        *   `startOffset: 0, endOffset: chunkData.content.length`: For manually created chunks, offsets are typically 0 and the content length, indicating the entire chunk is the content. For automatically split documents, these would indicate the chunk's position within the original document.
        *   `tag1: docTags.tag1, ...`: Inherits tags from the parent document, which is a common practice for context or filtering.
        *   `enabled: chunkData.enabled ?? true`: Sets the `enabled` status, defaulting to `true` if not provided.
    *   `await tx.insert(embedding).values(chunkDBData)`: Inserts the `chunkDBData` into the `embedding` table within the transaction.
    *   `await tx.update(document).set({...}).where(eq(document.id, documentId))`: Updates the parent `document`'s statistics.
        *   `chunkCount: sql`${document.chunkCount} + 1``: Increments the `chunkCount` by 1. The `sql` template literal is used to perform arithmetic directly in the database.
        *   `tokenCount: sql`${document.tokenCount} + ${tokenCount.count}``: Adds the new chunk's `tokenCount` to the document's total.
        *   `characterCount: sql`${document.characterCount} + ${chunkData.content.length}``: Adds the new chunk's content length to the document's total.
    *   `return {...} as ChunkData`: Returns an object representing the newly created chunk (similar to `chunkDBData` but conforming to `ChunkData` type) from within the transaction.
    *   `logger.info(...)`: Logs the successful creation of the chunk.
    *   `return newChunk`: Returns the chunk data that was committed by the transaction.

---

### `batchChunkOperation` Function

```typescript
/**
 * Perform batch operations on chunks
 */
export async function batchChunkOperation(
  documentId: string,
  operation: 'enable' | 'disable' | 'delete',
  chunkIds: string[],
  requestId: string
): Promise<BatchOperationResult> {
  logger.info(
    `[${requestId}] Starting batch ${operation} operation on ${chunkIds.length} chunks for document ${documentId}`
  )

  const errors: string[] = []
  let successCount = 0

  if (operation === 'delete') {
    // Handle batch delete with transaction for consistency
    await db.transaction(async (tx) => {
      // Get chunks to delete for statistics update
      const chunksToDelete = await tx
        .select({
          id: embedding.id,
          tokenCount: embedding.tokenCount,
          contentLength: embedding.contentLength,
        })
        .from(embedding)
        .where(and(eq(embedding.documentId, documentId), inArray(embedding.id, chunkIds)))

      if (chunksToDelete.length === 0) {
        errors.push('No matching chunks found to delete')
        return
      }

      const totalTokensToRemove = chunksToDelete.reduce((sum, chunk) => sum + chunk.tokenCount, 0)
      const totalCharsToRemove = chunksToDelete.reduce((sum, chunk) => sum + chunk.contentLength, 0)

      // Delete chunks
      const deleteResult = await tx
        .delete(embedding)
        .where(and(eq(embedding.documentId, documentId), inArray(embedding.id, chunkIds)))

      // Update document statistics
      await tx
        .update(document)
        .set({
          chunkCount: sql`${document.chunkCount} - ${chunksToDelete.length}`,
          tokenCount: sql`${document.tokenCount} - ${totalTokensToRemove}`,
          characterCount: sql`${document.characterCount} - ${totalCharsToRemove}`,
        })
        .where(eq(document.id, documentId))

      successCount = chunksToDelete.length
    })
  } else {
    // Handle enable/disable operations
    const enabled = operation === 'enable'

    await db
      .update(embedding)
      .set({
        enabled,
        updatedAt: new Date(),
      })
      .where(and(eq(embedding.documentId, documentId), inArray(embedding.id, chunkIds)))

    // For enable/disable, we assume all chunks were processed successfully
    successCount = chunkIds.length
  }

  logger.info(
    `[${requestId}] Batch ${operation} completed: ${successCount} chunks processed, ${errors.length} errors`
  )

  return {
    success: errors.length === 0,
    processed: successCount,
    errors,
  }
}
```

*   **Purpose:** This function allows performing `enable`, `disable`, or `delete` operations on multiple chunks simultaneously for a given document.
*   **Simplified Logic:** It branches based on the `operation`. For `delete`, it uses a transaction to first fetch the chunks to be deleted (to get their statistics), then deletes them, and finally updates the parent document's aggregate statistics. For `enable`/`disable`, it performs a direct batch update on the `enabled` status of the specified chunks.
*   **Line-by-Line Explanation:**
    *   `export async function batchChunkOperation(...)`: Declares an asynchronous function `batchChunkOperation` that is exported.
        *   `documentId: string`: The ID of the parent document.
        *   `operation: 'enable' | 'disable' | 'delete'`: The type of batch operation to perform.
        *   `chunkIds: string[]`: An array of chunk IDs to apply the operation to.
        *   `requestId: string`: Request ID for logging.
        *   `Promise<BatchOperationResult>`: Returns an object indicating success, processed count, and any errors.
    *   `logger.info(...)`: Logs the start of the batch operation.
    *   `const errors: string[] = []` and `let successCount = 0`: Initializes variables to track operation results.
    *   `if (operation === 'delete') { ... }`: This block handles batch deletion.
        *   `await db.transaction(async (tx) => { ... })`: Deletion involves updating the parent document's statistics, so it's wrapped in a transaction to ensure atomicity.
        *   `const chunksToDelete = await tx.select({...}).from(embedding).where(and(eq(embedding.documentId, documentId), inArray(embedding.id, chunkIds)))`: Before deleting, it queries for the chunks that *would* be deleted to retrieve their `tokenCount` and `contentLength`. This is crucial for correctly updating the parent document's aggregate statistics.
            *   `inArray(embedding.id, chunkIds)`: Drizzle's `inArray` function creates a `WHERE id IN ('id1', 'id2', ...)` clause.
        *   `if (chunksToDelete.length === 0) { ... }`: If no chunks match the provided `chunkIds` for the document, an error is recorded, and the transaction is exited.
        *   `const totalTokensToRemove = chunksToDelete.reduce(...)` and `const totalCharsToRemove = chunksToDelete.reduce(...)`: Calculates the sum of token counts and content lengths for all chunks to be deleted.
        *   `await tx.delete(embedding).where(...)`: Performs the actual batch deletion of chunks.
        *   `await tx.update(document).set({...}).where(eq(document.id, documentId))`: Updates the parent document's statistics, subtracting the counts and lengths of the deleted chunks.
        *   `successCount = chunksToDelete.length`: Sets the success count to the number of chunks actually found and deleted.
    *   `else { ... }`: This block handles 'enable' and 'disable' operations.
        *   `const enabled = operation === 'enable'`: Determines the boolean value for the `enabled` field.
        *   `await db.update(embedding).set({ enabled, updatedAt: new Date() }).where(and(eq(embedding.documentId, documentId), inArray(embedding.id, chunkIds)))`: Performs a batch update to set the `enabled` status and `updatedAt` timestamp for all specified chunks. This doesn't require a transaction because it's a single table update and doesn't affect document-level aggregates.
        *   `successCount = chunkIds.length`: Assumes all chunks specified in `chunkIds` were successfully updated.
    *   `logger.info(...)`: Logs the completion status of the batch operation.
    *   `return { success: errors.length === 0, processed: successCount, errors }`: Returns the result object.

---

### `updateChunk` Function

```typescript
/**
 * Update a single chunk
 */
export async function updateChunk(
  chunkId: string,
  updateData: {
    content?: string
    enabled?: boolean
  },
  requestId: string
): Promise<ChunkData> {
  const dbUpdateData: {
    updatedAt: Date
    content?: string
    contentLength?: number
    tokenCount?: number
    chunkHash?: string
    embedding?: number[]
    enabled?: boolean
  } = {
    updatedAt: new Date(),
  }

  // Use transaction if content is being updated to ensure consistent document statistics
  if (updateData.content !== undefined && typeof updateData.content === 'string') {
    return await db.transaction(async (tx) => {
      // Get current chunk data for character count calculation and content comparison
      const currentChunk = await tx
        .select({
          documentId: embedding.documentId,
          content: embedding.content,
          contentLength: embedding.contentLength,
          tokenCount: embedding.tokenCount,
        })
        .from(embedding)
        .where(eq(embedding.id, chunkId))
        .limit(1)

      if (currentChunk.length === 0) {
        throw new Error(`Chunk ${chunkId} not found`)
      }

      const oldContentLength = currentChunk[0].contentLength
      const oldTokenCount = currentChunk[0].tokenCount
      const content = updateData.content! // We know it's defined from the if check above
      const newContentLength = content.length

      // Only regenerate embedding if content actually changed
      if (content !== currentChunk[0].content) {
        logger.info(`[${requestId}] Content changed, regenerating embedding for chunk ${chunkId}`)

        // Generate new embedding for the updated content
        const embeddings = await generateEmbeddings([content])

        // Calculate accurate token count
        const tokenCount = estimateTokenCount(content, 'openai')

        dbUpdateData.content = content
        dbUpdateData.contentLength = newContentLength
        dbUpdateData.tokenCount = tokenCount.count
        dbUpdateData.chunkHash = createHash('sha256').update(content).digest('hex')
        // Add the embedding field to the update data
        dbUpdateData.embedding = embeddings[0]
      } else {
        // Content hasn't changed, just update other fields if needed
        dbUpdateData.content = content
        dbUpdateData.contentLength = newContentLength
        dbUpdateData.tokenCount = oldTokenCount // Keep the same token count if content is identical
        dbUpdateData.chunkHash = createHash('sha256').update(content).digest('hex')
      }

      if (updateData.enabled !== undefined) {
        dbUpdateData.enabled = updateData.enabled
      }

      // Update the chunk
      await tx.update(embedding).set(dbUpdateData).where(eq(embedding.id, chunkId))

      // Update document statistics for the character and token count changes
      const charDiff = newContentLength - oldContentLength
      const tokenDiff = dbUpdateData.tokenCount! - oldTokenCount

      await tx
        .update(document)
        .set({
          characterCount: sql`${document.characterCount} + ${charDiff}`,
          tokenCount: sql`${document.tokenCount} + ${tokenDiff}`,
        })
        .where(eq(document.id, currentChunk[0].documentId))

      // Fetch and return the updated chunk
      const updatedChunk = await tx
        .select({
          id: embedding.id,
          chunkIndex: embedding.chunkIndex,
          content: embedding.content,
          contentLength: embedding.contentLength,
          tokenCount: embedding.tokenCount,
          enabled: embedding.enabled,
          startOffset: embedding.startOffset,
          endOffset: embedding.endOffset,
          tag1: embedding.tag1,
          tag2: embedding.tag2,
          tag3: embedding.tag3,
          tag4: embedding.tag4,
          tag5: embedding.tag5,
          tag6: embedding.tag6,
          tag7: embedding.tag7,
          createdAt: embedding.createdAt,
          updatedAt: embedding.updatedAt,
        })
        .from(embedding)
        .where(eq(embedding.id, chunkId))
        .limit(1)

      logger.info(
        `[${requestId}] Updated chunk: ${chunkId}${updateData.content !== currentChunk[0].content ? ' (regenerated embedding)' : ''}`
      )

      return updatedChunk[0] as ChunkData
    })
  }

  // If only enabled status is being updated, no need for transaction
  if (updateData.enabled !== undefined) {
    dbUpdateData.enabled = updateData.enabled
  }

  await db.update(embedding).set(dbUpdateData).where(eq(embedding.id, chunkId))

  // Fetch the updated chunk
  const updatedChunk = await db
    .select({
      id: embedding.id,
      chunkIndex: embedding.chunkIndex,
      content: embedding.content,
      contentLength: embedding.contentLength,
      tokenCount: embedding.tokenCount,
      enabled: embedding.enabled,
      startOffset: embedding.startOffset,
      endOffset: embedding.endOffset,
      tag1: embedding.tag1,
      tag2: embedding.tag2,
      tag3: embedding.tag3,
      tag4: embedding.tag4,
      tag5: embedding.tag5,
      tag6: embedding.tag6,
      tag7: embedding.tag7,
      createdAt: embedding.createdAt,
      updatedAt: embedding.updatedAt,
    })
    .from(embedding)
    .where(eq(embedding.id, chunkId))
    .limit(1)

  if (updatedChunk.length === 0) {
    throw new Error(`Chunk ${chunkId} not found`)
  }

  logger.info(`[${requestId}] Updated chunk: ${chunkId}`)

  return updatedChunk[0] as ChunkData
}
```

*   **Purpose:** This function updates an existing chunk's content or its enabled status. If the content is updated, it triggers re-embedding and recalculation of token/character counts, also updating the parent document's statistics.
*   **Simplified Logic:** It checks if `content` is being updated.
    *   **If content is updated:** It runs within a transaction. It fetches the current chunk data to compare content and get old statistics. If content *actually* changed, it regenerates embeddings and token counts. It then updates the chunk and adjusts the parent document's token and character counts based on the *difference*. Finally, it fetches and returns the updated chunk.
    *   **If only `enabled` status is updated:** It performs a direct update on the chunk without a transaction, as no document statistics are affected.
*   **Line-by-Line Explanation:**
    *   `export async function updateChunk(...)`: Declares an asynchronous function `updateChunk` that is exported.
        *   `chunkId: string`: The ID of the chunk to update.
        *   `updateData: { content?: string; enabled?: boolean }`: An object containing the fields to update. `?` indicates optional fields.
        *   `requestId: string`: Request ID for logging.
        *   `Promise<ChunkData>`: Returns the updated chunk's data.
    *   `const dbUpdateData: { ... } = { updatedAt: new Date(), }`: Initializes an object to hold the database update values, setting `updatedAt` to the current time by default.
    *   `if (updateData.content !== undefined && typeof updateData.content === 'string') { ... }`: This block handles updates that include content modification. Since content changes affect token/character counts and embeddings, and thus parent document statistics, it requires a transaction.
        *   `return await db.transaction(async (tx) => { ... })`: Encloses the content update logic in a Drizzle transaction.
        *   `const currentChunk = await tx.select({...}).from(embedding).where(eq(embedding.id, chunkId)).limit(1)`: Fetches the current state of the chunk *before* updating it. This is crucial for:
            *   Comparing the `content` to see if it actually changed.
            *   Getting the `oldContentLength` and `oldTokenCount` to calculate the difference for document statistics.
            *   Getting the `documentId` to update the parent document.
        *   `if (currentChunk.length === 0) { throw new Error(...) }`: Throws an error if the chunk is not found.
        *   `const oldContentLength = currentChunk[0].contentLength; const oldTokenCount = currentChunk[0].tokenCount;`: Stores the original content length and token count.
        *   `const content = updateData.content!`: Safely asserts that `updateData.content` is defined (because of the `if` condition).
        *   `const newContentLength = content.length;`: Calculates the new content length.
        *   `if (content !== currentChunk[0].content) { ... }`: This condition is critical for performance. It only regenerates embeddings and recalculates tokens if the content has genuinely changed.
            *   `logger.info(...)`: Logs that content changed and embedding generation is occurring.
            *   `const embeddings = await generateEmbeddings([content])`: Generates a new vector embedding for the updated content.
            *   `const tokenCount = estimateTokenCount(content, 'openai')`: Estimates the new token count.
            *   `dbUpdateData.content = content; ... dbUpdateData.embedding = embeddings[0];`: Populates `dbUpdateData` with all new content-related values, including the new embedding and hash.
        *   `else { ... }`: If the content hasn't changed (e.g., the update call might have included the same content string), it still updates `contentLength` and `chunkHash` (for consistency, even if technically same) but reuses `oldTokenCount` and importantly *does not regenerate the embedding*.
        *   `if (updateData.enabled !== undefined) { dbUpdateData.enabled = updateData.enabled; }`: If `enabled` status is also part of the update, it's added to `dbUpdateData`.
        *   `await tx.update(embedding).set(dbUpdateData).where(eq(embedding.id, chunkId))`: Updates the chunk record in the database with `dbUpdateData`.
        *   `const charDiff = newContentLength - oldContentLength; const tokenDiff = dbUpdateData.tokenCount! - oldTokenCount;`: Calculates the difference in character and token counts. These differences are then used to update the parent document.
        *   `await tx.update(document).set({...}).where(eq(document.id, currentChunk[0].documentId))`: Updates the parent document's `characterCount` and `tokenCount` by adding the calculated `charDiff` and `tokenDiff`.
        *   `const updatedChunk = await tx.select({...}).from(embedding).where(eq(embedding.id, chunkId)).limit(1)`: Fetches the *fully updated* chunk from the database to ensure all fields, including `updatedAt`, are current for the return value.
        *   `logger.info(...)`: Logs the chunk update, indicating if embedding was regenerated.
        *   `return updatedChunk[0] as ChunkData`: Returns the first (and only) updated chunk record.
    *   `// If only enabled status is being updated, no need for transaction`: This comment introduces the alternative path if only `enabled` status is changed.
    *   `if (updateData.enabled !== undefined) { dbUpdateData.enabled = updateData.enabled; }`: If `enabled` is part of the update (and `content` was not), it's added to `dbUpdateData`.
    *   `await db.update(embedding).set(dbUpdateData).where(eq(embedding.id, chunkId))`: Performs the update on the `embedding` table directly, outside a transaction, as no document statistics are affected.
    *   `const updatedChunk = await db.select({...}).from(embedding).where(eq(embedding.id, chunkId)).limit(1)`: Fetches the updated chunk to return its current state.
    *   `if (updatedChunk.length === 0) { throw new Error(...) }`: Throws an error if the chunk is not found.
    *   `logger.info(...)`: Logs the chunk update.
    *   `return updatedChunk[0] as ChunkData`: Returns the updated chunk data.

---

### `deleteChunk` Function

```typescript
/**
 * Delete a single chunk with document statistics updates
 */
export async function deleteChunk(
  chunkId: string,
  documentId: string,
  requestId: string
): Promise<void> {
  await db.transaction(async (tx) => {
    // Get chunk data before deletion for statistics update
    const chunkToDelete = await tx
      .select({
        tokenCount: embedding.tokenCount,
        contentLength: embedding.contentLength,
      })
      .from(embedding)
      .where(eq(embedding.id, chunkId))
      .limit(1)

    if (chunkToDelete.length === 0) {
      throw new Error('Chunk not found')
    }

    const chunk = chunkToDelete[0]

    // Delete the chunk
    await tx.delete(embedding).where(eq(embedding.id, chunkId))

    // Update document statistics
    await tx
      .update(document)
      .set({
        chunkCount: sql`${document.chunkCount} - 1`,
        tokenCount: sql`${document.tokenCount} - ${chunk.tokenCount}`,
        characterCount: sql`${document.characterCount} - ${chunk.contentLength}`,
      })
      .where(eq(document.id, documentId))
  })

  logger.info(`[${requestId}] Deleted chunk: ${chunkId}`)
}
```

*   **Purpose:** This function deletes a single chunk and subsequently updates the parent document's aggregate statistics.
*   **Simplified Logic:** It operates within a transaction. It first retrieves the chunk's token count and content length (necessary for updating parent document statistics), then deletes the chunk, and finally subtracts the chunk's statistics from the parent document.
*   **Line-by-Line Explanation:**
    *   `export async function deleteChunk(...)`: Declares an asynchronous function `deleteChunk` that is exported.
        *   `chunkId: string`: The ID of the chunk to delete.
        *   `documentId: string`: The ID of the parent document.
        *   `requestId: string`: Request ID for logging.
        *   `Promise<void>`: Returns nothing upon successful deletion.
    *   `await db.transaction(async (tx) => { ... })`: Wraps the deletion and statistic updates in a transaction for atomicity.
    *   `const chunkToDelete = await tx.select({...}).from(embedding).where(eq(embedding.id, chunkId)).limit(1)`: Fetches the `tokenCount` and `contentLength` of the chunk *before* it's deleted. This data is needed to correctly decrement the parent document's statistics.
    *   `if (chunkToDelete.length === 0) { throw new Error('Chunk not found') }`: Throws an error if the chunk doesn't exist.
    *   `const chunk = chunkToDelete[0]`: Extracts the data of the chunk to be deleted.
    *   `await tx.delete(embedding).where(eq(embedding.id, chunkId))`: Performs the actual deletion of the chunk from the `embedding` table.
    *   `await tx.update(document).set({...}).where(eq(document.id, documentId))`: Updates the parent `document`'s statistics.
        *   `chunkCount: sql`${document.chunkCount} - 1``: Decrements the `chunkCount` by 1.
        *   `tokenCount: sql`${document.tokenCount} - ${chunk.tokenCount}``: Subtracts the deleted chunk's token count.
        *   `characterCount: sql`${document.characterCount} - ${chunk.contentLength}``: Subtracts the deleted chunk's content length.
    *   `logger.info(...)`: Logs the successful deletion of the chunk.

---

This file demonstrates robust and well-structured database interactions, prioritizing data consistency through transactions, efficient querying with Drizzle ORM, and integration with external services like embedding generators and token estimators, all while providing clear logging for operational insights.