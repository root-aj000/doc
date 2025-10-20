This TypeScript file, `knowledge-search-utils.ts`, serves as a comprehensive utility for performing sophisticated searches within a knowledge base system. It enables retrieving relevant information (embeddings) based on various criteria, including **metadata tags** and **vector similarity**.

At its core, the file interacts with a database containing `document` (representing entire files) and `embedding` (representing smaller, searchable chunks of documents, each with a vector representation and metadata tags). It provides functions to query these embeddings, supporting different search strategies and performance optimizations.

---

## Simplified Overview

Imagine you have a vast library of books (documents), and each book is broken down into small, meaningful paragraphs or sections (embeddings). Each section might have labels like "Chapter: Introduction," "Topic: AI," "Author: John Doe" (these are like tags), and also a special numerical code that represents its meaning (a vector embedding).

This file allows you to:

1.  **Find document names**: Given a list of document IDs, tell you their original filenames.
2.  **Search by tags**: "Show me all sections labeled 'Topic: AI' AND 'Author: John Doe'."
3.  **Search by meaning (vector similarity)**: "Given this query text, show me the sections whose meaning is most similar to it." (This is done by converting the query text into a numerical vector and comparing it to the sections' vectors).
4.  **Search by tags AND meaning**: "Show me all sections labeled 'Topic: AI' that are *also* most similar in meaning to this query text."
5.  **Optimize performance**: If you're searching across many "knowledge bases" (sub-libraries), it intelligently decides whether to run searches in parallel to get results faster.

The file uses Drizzle ORM to build safe and efficient database queries, and it's designed to be flexible and performant for different search scenarios.

---

## Detailed Explanation

Let's break down the code section by section.

### 1. Imports and Logger Setup

```typescript
import { db } from '@sim/db'
import { document, embedding } from '@sim/db/schema'
import { and, eq, inArray, isNull, sql } from 'drizzle-orm'
import { createLogger } from '@/lib/logs/console/logger'

const logger = createLogger('KnowledgeSearchUtils')
```

*   **`import { db } from '@sim/db'`**: This imports the Drizzle ORM database client instance. This `db` object is how all database operations (select, insert, update, delete) are performed.
*   **`import { document, embedding } from '@sim/db/schema'`**: These imports bring in the Drizzle ORM table definitions for `document` and `embedding`. These are essentially JavaScript objects that represent the structure of your database tables, allowing Drizzle to build type-safe queries.
    *   `document`: Likely stores metadata about the original files (e.g., `id`, `filename`, `deletedAt`).
    *   `embedding`: Stores actual content chunks, their vector representations (`embedding` column), associated `documentId`, `chunkIndex`, and up to seven `tag` columns for categorization.
*   **`import { and, eq, inArray, isNull, sql } from 'drizzle-orm'`**: These are helper functions from Drizzle ORM used to construct `WHERE` clauses in database queries:
    *   `and`: Combines multiple conditions with a logical `AND`.
    *   `eq`: Checks for equality (e.g., `column = value`).
    *   `inArray`: Checks if a column's value is present in a given array (e.g., `column IN (value1, value2)`).
    *   `isNull`: Checks if a column's value is `NULL`.
    *   `sql`: Allows embedding raw SQL snippets directly into Drizzle queries. This is particularly useful for database-specific functions or operators not directly supported by Drizzle's helpers, like vector similarity operators.
*   **`import { createLogger } from '@/lib/logs/console/logger'`**: Imports a utility to create a logger instance, which is used for outputting debugging information and messages.
*   **`const logger = createLogger('KnowledgeSearchUtils')`**: Initializes a logger specifically for this file, making it easy to trace logs originating from these search utilities.

### 2. `getDocumentNamesByIds` Function

```typescript
export async function getDocumentNamesByIds(
  documentIds: string[]
): Promise<Record<string, string>> {
  if (documentIds.length === 0) {
    return {}
  }

  const uniqueIds = [...new Set(documentIds)]
  const documents = await db
    .select({
      id: document.id,
      filename: document.filename,
    })
    .from(document)
    .where(and(inArray(document.id, uniqueIds), isNull(document.deletedAt)))

  const documentNameMap: Record<string, string> = {}
  documents.forEach((doc) => {
    documentNameMap[doc.id] = doc.filename
  })

  return documentNameMap
}
```

*   **Purpose**: This function fetches the filenames for a given list of document IDs. It's useful for displaying the source files of search results.
*   **`export async function getDocumentNamesByIds(...)`**: Defines an asynchronous function that takes an array of `documentIds` (strings) and returns a `Promise` that resolves to a `Record<string, string>` (an object mapping document IDs to their filenames).
*   **`if (documentIds.length === 0) { return {} }`**: An early exit condition. If no IDs are provided, an empty object is returned immediately, preventing unnecessary database queries.
*   **`const uniqueIds = [...new Set(documentIds)]`**: Creates a new array containing only unique document IDs. This is a good practice to avoid redundant checks in the `inArray` clause and potentially optimize the database query.
*   **`const documents = await db.select(...).from(...).where(...)`**: This is a Drizzle ORM query:
    *   **`.select({ id: document.id, filename: document.filename })`**: Specifies which columns to retrieve: the `id` and `filename` from the `document` table.
    *   **`.from(document)`**: Indicates that the query is targeting the `document` table.
    *   **`.where(and(inArray(document.id, uniqueIds), isNull(document.deletedAt)))`**: Filters the results:
        *   `and(...)`: Combines the following two conditions.
        *   `inArray(document.id, uniqueIds)`: Selects only documents whose `id` is present in the `uniqueIds` array.
        *   `isNull(document.deletedAt)`: Ensures that only documents *not marked as deleted* are retrieved. This is a common soft-delete pattern.
*   **`const documentNameMap: Record<string, string> = {}`**: Initializes an empty object to store the ID-to-filename mapping.
*   **`documents.forEach((doc) => { documentNameMap[doc.id] = doc.filename })`**: Iterates through the fetched `documents` and populates the `documentNameMap` with `id: filename` pairs.
*   **`return documentNameMap`**: Returns the constructed map.

### 3. Interfaces for Search Results and Parameters

```typescript
export interface SearchResult {
  id: string
  content: string
  documentId: string
  chunkIndex: number
  tag1: string | null
  tag2: string | null
  tag3: string | null
  tag4: string | null
  tag5: string | null
  tag6: string | null
  tag7: string | null
  distance: number
  knowledgeBaseId: string
}

export interface SearchParams {
  knowledgeBaseIds: string[]
  topK: number
  filters?: Record<string, string>
  queryVector?: string
  distanceThreshold?: number
}
```

*   **`export interface SearchResult`**: Defines the structure of a single search result object.
    *   `id`: The unique identifier of the embedding (chunk).
    *   `content`: The textual content of the embedding.
    *   `documentId`: The ID of the original document this embedding belongs to.
    *   `chunkIndex`: The index or order of this chunk within its document.
    *   `tag1` through `tag7`: Optional metadata tags associated with the embedding. Can be `null`.
    *   `distance`: A numerical value indicating the similarity or dissimilarity to the search query. Lower values typically mean more similar. This is primarily for vector searches. For tag-only searches, it's often `0`.
    *   `knowledgeBaseId`: The ID of the knowledge base this embedding belongs to.
*   **`export interface SearchParams`**: Defines the parameters required for any search operation.
    *   `knowledgeBaseIds`: An array of IDs for the knowledge bases to search within.
    *   `topK`: The maximum number of search results to return.
    *   `filters?`: An optional `Record<string, string>` where keys are tag names (e.g., 'tag1') and values are the desired tag values. This is used for tag-based filtering. The `?` makes it optional.
    *   `queryVector?`: An optional string representation of the vector embedding for the search query. This is required for vector similarity searches.
    *   `distanceThreshold?`: An optional number representing the maximum allowed distance for a result to be considered relevant in a vector search. Results with a distance greater than this threshold will be excluded.

### 4. Shared Embedding Utility Export

```typescript
// Use shared embedding utility
export { generateSearchEmbedding } from '@/lib/embeddings/utils'
```

*   This line simply re-exports a function `generateSearchEmbedding` from another module. This function is likely responsible for taking a plain text query and converting it into a numerical vector (`queryVector`) that can be used for vector similarity searches. This promotes code reuse and separation of concerns.

### 5. `getTagFilters` Function

```typescript
function getTagFilters(filters: Record<string, string>, embedding: any) {
  return Object.entries(filters).map(([key, value]) => {
    // Handle OR logic within same tag
    const values = value.includes('|OR|') ? value.split('|OR|') : [value]
    logger.debug(`[getTagFilters] Processing ${key}="${value}" -> values:`, values)

    const getColumnForKey = (key: string) => {
      switch (key) {
        case 'tag1': return embedding.tag1
        case 'tag2': return embedding.tag2
        case 'tag3': return embedding.tag3
        case 'tag4': return embedding.tag4
        case 'tag5': return embedding.tag5
        case 'tag6': return embedding.tag6
        case 'tag7': return embedding.tag7
        default: return null
      }
    }

    const column = getColumnForKey(key)
    if (!column) return sql`1=1` // No-op for unknown keys

    if (values.length === 1) {
      // Single value - simple equality
      logger.debug(`[getTagFilters] Single value filter: ${key} = ${values[0]}`)
      return sql`LOWER(${column}) = LOWER(${values[0]})`
    }
    // Multiple values - OR logic
    logger.debug(`[getTagFilters] OR filter: ${key} IN (${values.join(', ')})`)
    const orConditions = values.map((v) => sql`LOWER(${column}) = LOWER(${v})`)
    return sql`(${sql.join(orConditions, sql` OR `)})`
  })
}
```

*   **Purpose**: This helper function dynamically generates an array of Drizzle `SQL` conditions based on the provided `filters` object. It's designed to build complex tag filtering logic for database queries.
*   **`function getTagFilters(filters: Record<string, string>, embedding: any)`**: Takes the `filters` object (e.g., `{ tag1: "AI", tag2: "Science|OR|Technology" }`) and the Drizzle `embedding` table schema object.
*   **`return Object.entries(filters).map(([key, value]) => { ... })`**: Iterates over each key-value pair in the `filters` object (e.g., `['tag1', 'AI']`, `['tag2', 'Science|OR|Technology']`). For each pair, it creates a SQL condition.
*   **`const values = value.includes('|OR|') ? value.split('|OR|') : [value]`**: This is a crucial part for handling "OR" logic within a single tag. If a filter value contains `|OR|`, it's split into multiple values. For example, `'Science|OR|Technology'` becomes `['Science', 'Technology']`. Otherwise, it's treated as a single value.
*   **`getColumnForKey(key: string)`**: An inner helper function that maps a string tag name (e.g., `'tag1'`) to the corresponding Drizzle column object (e.g., `embedding.tag1`). This makes the code flexible to work with different tag columns.
*   **`if (!column) return sql`1=1``**: If an unknown tag key is provided in the `filters`, it returns `sql`1=1``, which is a SQL no-operation (always true), effectively ignoring that invalid filter without breaking the query.
*   **`if (values.length === 1)`**:
    *   If there's only one value for a tag (no `|OR|`), it creates a simple equality condition: `sql`LOWER(${column}) = LOWER(${values[0]})``. `LOWER()` is used to perform case-insensitive comparison in the database.
*   **`else { /* Multiple values - OR logic */ }`**:
    *   If there are multiple values (due to `|OR|`), it generates an `OR` condition.
    *   `const orConditions = values.map((v) => sql`LOWER(${column}) = LOWER(${v})`)`: Creates an array of individual `LOWER(column) = LOWER(value)` conditions.
    *   `return sql`(${sql.join(orConditions, sql` OR `)})``: Joins these individual conditions with ` OR ` and wraps them in parentheses to ensure correct SQL precedence. For example, `(LOWER(tag2) = LOWER('Science') OR LOWER(tag2) = LOWER('Technology'))`.

### 6. `getQueryStrategy` Function

```typescript
export function getQueryStrategy(kbCount: number, topK: number) {
  const useParallel = kbCount > 4 || (kbCount > 2 && topK > 50)
  const distanceThreshold = kbCount > 3 ? 0.8 : 1.0
  const parallelLimit = Math.ceil(topK / kbCount) + 5

  return {
    useParallel,
    distanceThreshold,
    parallelLimit,
    singleQueryOptimized: kbCount <= 2,
  }
}
```

*   **Purpose**: This function determines an optimal strategy for executing search queries based on the number of knowledge bases (`kbCount`) and the desired number of results (`topK`). This is a performance optimization to avoid overwhelming a single database query or to leverage parallel processing.
*   **`useParallel`**: A boolean flag. It becomes `true` (suggesting parallel queries) if:
    *   There are more than 4 knowledge bases (`kbCount > 4`).
    *   OR if there are more than 2 knowledge bases (`kbCount > 2`) AND `topK` (results) is greater than 50. This heuristic aims to balance the overhead of parallel queries against the potential performance gain for larger searches.
*   **`distanceThreshold`**: A numerical value used in vector searches. It adjusts the default maximum acceptable distance for results based on `kbCount`. If `kbCount > 3`, it's `0.8`; otherwise, it's `1.0`. A lower threshold means stricter similarity, potentially returning fewer but more precise results. This heuristic might be designed to return more relevant results when searching across many KBs.
*   **`parallelLimit`**: If `useParallel` is `true`, this calculates the `LIMIT` to apply to each individual parallel query. It's `Math.ceil(topK / kbCount) + 5`. This ensures each parallel query fetches slightly more than its "fair share" of results, allowing for better overall selection after combining and sorting. The `+ 5` provides a buffer.
*   **`singleQueryOptimized`**: A boolean flag, `true` if `kbCount` is 2 or less. This indicates that for a small number of KBs, a single database query (using `inArray` for `knowledgeBaseId`) is likely more efficient than parallel queries due to less overhead.
*   **Return Value**: An object containing these strategic parameters.

### 7. `executeTagFilterQuery` Function

```typescript
async function executeTagFilterQuery(
  knowledgeBaseIds: string[],
  filters: Record<string, string>
): Promise<{ id: string }[]> {
  if (knowledgeBaseIds.length === 1) {
    return await db
      .select({ id: embedding.id })
      .from(embedding)
      .innerJoin(document, eq(embedding.documentId, document.id))
      .where(
        and(
          eq(embedding.knowledgeBaseId, knowledgeBaseIds[0]),
          eq(embedding.enabled, true),
          isNull(document.deletedAt),
          ...getTagFilters(filters, embedding)
        )
      )
  }
  return await db
    .select({ id: embedding.id })
    .from(embedding)
    .innerJoin(document, eq(embedding.documentId, document.id))
    .where(
      and(
        inArray(embedding.knowledgeBaseId, knowledgeBaseIds),
        eq(embedding.enabled, true),
        isNull(document.deletedAt),
        ...getTagFilters(filters, embedding)
      )
    )
}
```

*   **Purpose**: This internal helper function executes a database query to find the IDs of embeddings that match a given set of `filters` (tags) and belong to specified `knowledgeBaseIds`. It does *not* perform vector search, only tag filtering.
*   **`if (knowledgeBaseIds.length === 1)`**:
    *   **Optimization**: If only one knowledge base ID is provided, it uses `eq(embedding.knowledgeBaseId, knowledgeBaseIds[0])`. This is slightly more efficient than `inArray` for a single value.
*   **`return await db.select({ id: embedding.id })...`**:
    *   **`.select({ id: embedding.id })`**: Only retrieves the `id` of the matching embeddings. This is often all that's needed for an intermediate filtering step.
    *   **`.from(embedding)`**: Starts the query from the `embedding` table.
    *   **`.innerJoin(document, eq(embedding.documentId, document.id))`**: Joins the `embedding` table with the `document` table. This is necessary because some conditions (like `document.deletedAt`) are on the `document` table, and `embedding` records are linked to `document` records. `eq(embedding.documentId, document.id)` specifies the join condition.
    *   **`.where(and(...))`**: Applies multiple conditions using `and`:
        *   `eq(embedding.knowledgeBaseId, knowledgeBaseIds[0])` or `inArray(embedding.knowledgeBaseId, knowledgeBaseIds)`: Filters by the specified knowledge base(s).
        *   `eq(embedding.enabled, true)`: Ensures only enabled embeddings are considered (an active/inactive flag).
        *   `isNull(document.deletedAt)`: Excludes embeddings whose parent document has been soft-deleted.
        *   `...getTagFilters(filters, embedding)`: This is where the dynamic tag filter conditions generated by `getTagFilters` are spread into the `and` clause.

### 8. `executeVectorSearchOnIds` Function

```typescript
async function executeVectorSearchOnIds(
  embeddingIds: string[],
  queryVector: string,
  topK: number,
  distanceThreshold: number
): Promise<SearchResult[]> {
  if (embeddingIds.length === 0) {
    return []
  }

  return await db
    .select({
      id: embedding.id,
      content: embedding.content,
      documentId: embedding.documentId,
      chunkIndex: embedding.chunkIndex,
      tag1: embedding.tag1,
      tag2: embedding.tag2,
      tag3: embedding.tag3,
      tag4: embedding.tag4,
      tag5: embedding.tag5,
      tag6: embedding.tag6,
      tag7: embedding.tag7,
      distance: sql<number>`${embedding.embedding} <=> ${queryVector}::vector`.as('distance'),
      knowledgeBaseId: embedding.knowledgeBaseId,
    })
    .from(embedding)
    .innerJoin(document, eq(embedding.documentId, document.id))
    .where(
      and(
        inArray(embedding.id, embeddingIds),
        isNull(document.deletedAt),
        sql`${embedding.embedding} <=> ${queryVector}::vector < ${distanceThreshold}`
      )
    )
    .orderBy(sql`${embedding.embedding} <=> ${queryVector}::vector`)
    .limit(topK)
}
```

*   **Purpose**: This internal helper function performs a vector similarity search *only on a pre-filtered list of `embeddingIds`*. This is a critical optimization when combining tag and vector searches, as it reduces the number of embeddings the vector comparison needs to check.
*   **`if (embeddingIds.length === 0) { return [] }`**: Early exit if no embedding IDs are provided for the vector search.
*   **`return await db.select(...).from(...).where(...).orderBy(...).limit(...)`**:
    *   **`.select(...)`**: Selects all the fields required for the `SearchResult` interface, including the crucial `distance`.
    *   **`distance: sql<number>`${embedding.embedding} <=> ${queryVector}::vector`.as('distance')`**: This is the core of the vector similarity search:
        *   `embedding.embedding`: The vector column from the database.
        *   `<=>`: This is the **`pgvector` distance operator**. It calculates the "distance" (usually cosine distance) between two vector embeddings. A smaller distance means higher similarity.
        *   `${queryVector}::vector`: The input `queryVector` (string representation of a vector) is explicitly cast to a `vector` type for the database.
        *   `.as('distance')`: Aliases the calculated distance value as `distance` in the result set, matching the `SearchResult` interface.
    *   **`.innerJoin(document, eq(embedding.documentId, document.id))`**: Joins with the `document` table to check for deleted documents.
    *   **`.where(and(...))`**:
        *   `inArray(embedding.id, embeddingIds)`: Crucially limits the vector search to only those embedding IDs that were passed into the function (e.g., from a prior tag filter).
        *   `isNull(document.deletedAt)`: Excludes embeddings from deleted documents.
        *   `sql`${embedding.embedding} <=> ${queryVector}::vector < ${distanceThreshold}``: Filters out results whose distance to the `queryVector` is *greater than or equal to* the `distanceThreshold`. Only "close enough" embeddings are kept.
    *   **`.orderBy(sql`${embedding.embedding} <=> ${queryVector}::vector`)`**: Sorts the results by their calculated distance in ascending order (most similar first).
    *   **`.limit(topK)`**: Restricts the number of returned results to `topK`.

### 9. `handleTagOnlySearch` Function

```typescript
export async function handleTagOnlySearch(params: SearchParams): Promise<SearchResult[]> {
  const { knowledgeBaseIds, topK, filters } = params

  if (!filters || Object.keys(filters).length === 0) {
    throw new Error('Tag filters are required for tag-only search')
  }

  logger.debug(`[handleTagOnlySearch] Executing tag-only search with filters:`, filters)

  const strategy = getQueryStrategy(knowledgeBaseIds.length, topK)

  if (strategy.useParallel) {
    // Parallel approach for many KBs
    const parallelLimit = Math.ceil(topK / knowledgeBaseIds.length) + 5

    const queryPromises = knowledgeBaseIds.map(async (kbId) => {
      return await db
        .select({
          id: embedding.id,
          content: embedding.content,
          documentId: embedding.documentId,
          chunkIndex: embedding.chunkIndex,
          tag1: embedding.tag1,
          tag2: embedding.tag2,
          tag3: embedding.tag3,
          tag4: embedding.tag4,
          tag5: embedding.tag5,
          tag6: embedding.tag6,
          tag7: embedding.tag7,
          distance: sql<number>`0`.as('distance'), // No distance for tag-only searches
          knowledgeBaseId: embedding.knowledgeBaseId,
        })
        .from(embedding)
        .innerJoin(document, eq(embedding.documentId, document.id))
        .where(
          and(
            eq(embedding.knowledgeBaseId, kbId),
            eq(embedding.enabled, true),
            isNull(document.deletedAt),
            ...getTagFilters(filters, embedding)
          )
        )
        .limit(parallelLimit)
    })

    const parallelResults = await Promise.all(queryPromises)
    return parallelResults.flat().slice(0, topK)
  }
  // Single query for fewer KBs
  return await db
    .select({
      id: embedding.id,
      content: embedding.content,
      documentId: embedding.documentId,
      chunkIndex: embedding.chunkIndex,
      tag1: embedding.tag1,
      tag2: embedding.tag2,
      tag3: embedding.tag3,
      tag4: embedding.tag4,
      tag5: embedding.tag5,
      tag6: embedding.tag6,
      tag7: embedding.tag7,
      distance: sql<number>`0`.as('distance'), // No distance for tag-only searches
      knowledgeBaseId: embedding.knowledgeBaseId,
    })
    .from(embedding)
    .innerJoin(document, eq(embedding.documentId, document.id))
    .where(
      and(
        inArray(embedding.knowledgeBaseId, knowledgeBaseIds),
        eq(embedding.enabled, true),
        isNull(document.deletedAt),
        ...getTagFilters(filters, embedding)
      )
    )
    .limit(topK)
}
```

*   **Purpose**: This is the main entry point for performing searches based *only* on metadata tags.
*   **`if (!filters || Object.keys(filters).length === 0)`**: Input validation: Tag filters are mandatory for a tag-only search.
*   **`const strategy = getQueryStrategy(knowledgeBaseIds.length, topK)`**: Determines the query execution strategy (parallel vs. single query).
*   **`if (strategy.useParallel)`**:
    *   **`knowledgeBaseIds.map(async (kbId) => { ... })`**: For each knowledge base, a separate Drizzle query is constructed.
    *   **`.where(and(eq(embedding.knowledgeBaseId, kbId), ...))`**: Each parallel query targets a *single* `kbId`.
    *   **`.limit(parallelLimit)`**: Each parallel query fetches up to `parallelLimit` results (slightly more than its proportional share of `topK`).
    *   **`distance: sql<number>`0`.as('distance')`**: Since this is a tag-only search, there's no vector distance. The `distance` field is set to `0` to satisfy the `SearchResult` interface.
    *   **`const parallelResults = await Promise.all(queryPromises)`**: Executes all queries concurrently.
    *   **`return parallelResults.flat().slice(0, topK)`**: Combines all results from the parallel queries into a single array (`flat()`) and then takes only the `topK` results from the beginning. Since there's no ordering by distance, the order might be somewhat arbitrary after combining.
*   **`else { /* Single query for fewer KBs */ }`**:
    *   **`.where(and(inArray(embedding.knowledgeBaseId, knowledgeBaseIds), ...))`**: For a smaller number of KBs, a single query using `inArray` is used, which is generally more efficient than the overhead of multiple parallel queries.
    *   The `select`, `from`, `innerJoin`, `where`, and `limit` logic is similar to the parallel version but targets all `knowledgeBaseIds` in one go.

### 10. `handleVectorOnlySearch` Function

```typescript
export async function handleVectorOnlySearch(params: SearchParams): Promise<SearchResult[]> {
  const { knowledgeBaseIds, topK, queryVector, distanceThreshold } = params

  if (!queryVector || !distanceThreshold) {
    throw new Error('Query vector and distance threshold are required for vector-only search')
  }

  logger.debug(`[handleVectorOnlySearch] Executing vector-only search`)

  const strategy = getQueryStrategy(knowledgeBaseIds.length, topK)

  if (strategy.useParallel) {
    // Parallel approach for many KBs
    const parallelLimit = Math.ceil(topK / knowledgeBaseIds.length) + 5

    const queryPromises = knowledgeBaseIds.map(async (kbId) => {
      return await db
        .select({
          id: embedding.id,
          content: embedding.content,
          documentId: embedding.documentId,
          chunkIndex: embedding.chunkIndex,
          tag1: embedding.tag1,
          tag2: embedding.tag2,
          tag3: embedding.tag3,
          tag4: embedding.tag4,
          tag5: embedding.tag5,
          tag6: embedding.tag6,
          tag7: embedding.tag7,
          distance: sql<number>`${embedding.embedding} <=> ${queryVector}::vector`.as('distance'),
          knowledgeBaseId: embedding.knowledgeBaseId,
        })
        .from(embedding)
        .innerJoin(document, eq(embedding.documentId, document.id))
        .where(
          and(
            eq(embedding.knowledgeBaseId, kbId),
            eq(embedding.enabled, true),
            isNull(document.deletedAt),
            sql`${embedding.embedding} <=> ${queryVector}::vector < ${distanceThreshold}`
          )
        )
        .orderBy(sql`${embedding.embedding} <=> ${queryVector}::vector`)
        .limit(parallelLimit)
    })

    const parallelResults = await Promise.all(queryPromises)
    const allResults = parallelResults.flat()
    return allResults.sort((a, b) => a.distance - b.distance).slice(0, topK)
  }
  // Single query for fewer KBs
  return await db
    .select({
      id: embedding.id,
      content: embedding.content,
      documentId: embedding.documentId,
      chunkIndex: embedding.chunkIndex,
      tag1: embedding.tag1,
      tag2: embedding.tag2,
      tag3: embedding.tag3,
      tag4: embedding.tag4,
      tag5: embedding.tag5,
      tag6: embedding.tag6,
      tag7: embedding.tag7,
      distance: sql<number>`${embedding.embedding} <=> ${queryVector}::vector`.as('distance'),
      knowledgeBaseId: embedding.knowledgeBaseId,
    })
    .from(embedding)
    .innerJoin(document, eq(embedding.documentId, document.id))
    .where(
      and(
        inArray(embedding.knowledgeBaseId, knowledgeBaseIds),
        eq(embedding.enabled, true),
        isNull(document.deletedAt),
        sql`${embedding.embedding} <=> ${queryVector}::vector < ${distanceThreshold}`
      )
    )
    .orderBy(sql`${embedding.embedding} <=> ${queryVector}::vector`)
    .limit(topK)
}
```

*   **Purpose**: This is the main entry point for performing searches based *only* on vector similarity.
*   **`if (!queryVector || !distanceThreshold)`**: Input validation: `queryVector` and `distanceThreshold` are mandatory for a vector-only search.
*   **`const strategy = getQueryStrategy(knowledgeBaseIds.length, topK)`**: Determines the query execution strategy.
*   The `if (strategy.useParallel)` and `else` blocks follow a similar structure to `handleTagOnlySearch`, but with key differences for vector search:
    *   **`distance: sql<number>`${embedding.embedding} <=> ${queryVector}::vector`.as('distance')`**: The actual vector distance is calculated and included in the selection.
    *   **`sql`${embedding.embedding} <=> ${queryVector}::vector < ${distanceThreshold}``**: This condition is added to the `WHERE` clause to filter by distance.
    *   **`.orderBy(sql`${embedding.embedding} <=> ${queryVector}::vector`)`**: Results are always ordered by distance (most similar first).
    *   **`return allResults.sort((a, b) => a.distance - b.distance).slice(0, topK)` (for parallel)**: After combining results from parallel queries, they *must* be re-sorted by `distance` because the order from individual queries might be lost or inconsistent when `flat()`ing, and then `slice()`ed to `topK`.

### 11. `handleTagAndVectorSearch` Function

```typescript
export async function handleTagAndVectorSearch(params: SearchParams): Promise<SearchResult[]> {
  const { knowledgeBaseIds, topK, filters, queryVector, distanceThreshold } = params

  if (!filters || Object.keys(filters).length === 0) {
    throw new Error('Tag filters are required for tag and vector search')
  }
  if (!queryVector || !distanceThreshold) {
    throw new Error('Query vector and distance threshold are required for tag and vector search')
  }

  logger.debug(`[handleTagAndVectorSearch] Executing tag + vector search with filters:`, filters)

  // Step 1: Filter by tags first
  const tagFilteredIds = await executeTagFilterQuery(knowledgeBaseIds, filters)

  if (tagFilteredIds.length === 0) {
    logger.debug(`[handleTagAndVectorSearch] No results found after tag filtering`)
    return []
  }

  logger.debug(
    `[handleTagAndVectorSearch] Found ${tagFilteredIds.length} results after tag filtering`
  )

  // Step 2: Perform vector search only on tag-filtered results
  return await executeVectorSearchOnIds(
    tagFilteredIds.map((r) => r.id),
    queryVector,
    topK,
    distanceThreshold
  )
}
```

*   **Purpose**: This is the main entry point for performing searches that combine *both* metadata tag filtering and vector similarity.
*   **Input Validation**: Checks that both `filters` and `queryVector`/`distanceThreshold` are provided.
*   **`Step 1: Filter by tags first`**:
    *   **`const tagFilteredIds = await executeTagFilterQuery(knowledgeBaseIds, filters)`**: This is a crucial optimization. Instead of trying to apply both tag and vector filters in a single, potentially slow query, it first calls `executeTagFilterQuery`. This efficiently narrows down the candidate embeddings to only those that match the tag criteria.
*   **`if (tagFilteredIds.length === 0) { return [] }`**: If no embeddings match the tag filters, there's no need to proceed with vector search, so an empty array is returned.
*   **`Step 2: Perform vector search only on tag-filtered results`**:
    *   **`return await executeVectorSearchOnIds(tagFilteredIds.map((r) => r.id), queryVector, topK, distanceThreshold)`**: The `executeVectorSearchOnIds` function is then called. By passing `tagFilteredIds.map((r) => r.id)`, the vector search is constrained to a much smaller, pre-filtered subset of embeddings. This significantly improves performance compared to running a vector search across the entire dataset and then filtering by tags.

---

This detailed breakdown covers the purpose, logic, and line-by-line explanation of the `knowledge-search-utils.ts` file, highlighting its design for flexible and performant knowledge base searching.