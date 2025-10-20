This TypeScript file is a crucial part of a system that manages access and permissions for knowledge bases, documents, and their embedded "chunks" (also referred to as embeddings). It acts as a **centralized access control layer** for these entities.

## Overall Purpose of This File

The primary purpose of this file is to provide a set of asynchronous functions that determine whether a specific user has the necessary permissions to perform actions (read or write) on a given **Knowledge Base**, a **Document** within a knowledge base, or a specific **Chunk (Embedding)** within a document.

It achieves this by:
1.  **Defining Data Structures:** Establishing clear TypeScript interfaces for `KnowledgeBase`, `Document`, and `Embedding` (chunk) data, as well as the standardized return types for access checks.
2.  **Database Interaction:** Querying a Drizzle ORM-backed database to retrieve entity information and verify their existence and status (e.g., not deleted, processing completed).
3.  **Permission Logic:** Implementing rules that check if a user is the owner, or if they have workspace-level permissions that grant access to the entity.
4.  **Cascading Permissions:** Ensuring that access to child entities (documents, chunks) is contingent upon access to their parent entities (knowledge bases, documents).

In essence, this file answers the question: "Can `userX` access/modify `entityY`?" and provides structured information about *why* or *why not*.

---

## Simplified Complex Logic

The core logic can be simplified into these principles:

1.  **Ownership First:** If a user directly owns a Knowledge Base, they generally have full access to it and its contents.
2.  **Workspace Hierarchy:** If a Knowledge Base belongs to a Workspace, users with appropriate permissions (e.g., 'read', 'write', 'admin') on that Workspace gain access to the Knowledge Base and its contents.
3.  **Soft Deletion:** Entities are rarely truly deleted from the database; instead, a `deletedAt` timestamp is set. All access checks filter out entities where `deletedAt` is not null.
4.  **Access Cascades Down:** You cannot access a Document if you don't have access to its parent Knowledge Base. Similarly, you cannot access a Chunk if you don't have access to its parent Document and Knowledge Base.
5.  **Write vs. Read Access:** Write access usually implies stricter permissions (owner, or 'write'/'admin' workspace roles), while read access might be broader.
6.  **Processing Status:** For granular data like "chunks" (embeddings), the system checks if the parent Document's processing is `completed`. If not, the chunk data might be incomplete or unavailable.
7.  **Standardized Responses:** All access check functions return a union type (`AccessCheck`) which clearly indicates `hasAccess: true` along with the relevant data, or `hasAccess: false` with optional `notFound` and `reason` fields. This makes handling access results consistent and type-safe.

---

## Detailed Line-by-Line Explanation

Let's break down the code:

### Imports

```typescript
import { db } from '@sim/db'
import { document, embedding, knowledgeBase } from '@sim/db/schema'
import { and, eq, isNull } from 'drizzle-orm'
import { getUserEntityPermissions } from '@/lib/permissions/utils'
```

*   `import { db } from '@sim/db'`: This line imports the `db` object, which is an instance of a database client (likely configured with Drizzle ORM). This `db` object is used to perform all database queries. The `@sim/db` alias suggests it's a module within the `@sim` (likely "Simulate" or "Simulator") project structure.
*   `import { document, embedding, knowledgeBase } from '@sim/db/schema'`: These imports bring in the Drizzle ORM schema definitions for the `document`, `embedding`, and `knowledgeBase` tables. These definitions allow Drizzle to construct type-safe SQL queries.
*   `import { and, eq, isNull } from 'drizzle-orm'`: These are helper functions from Drizzle ORM used to build query conditions:
    *   `and`: Combines multiple conditions with a logical AND operator.
    *   `eq`: Checks for equality (e.g., `column = value`).
    *   `isNull`: Checks if a column's value is NULL (often used for soft deletion checks).
*   `import { getUserEntityPermissions } from '@/lib/permissions/utils'`: This imports a utility function that's external to this file. It's responsible for checking a user's permissions on a specific "entity" (like a workspace). The `@/lib/permissions/utils` path suggests it's located in the `lib/permissions` directory of the project.

### Data Interfaces

These interfaces define the structure of data for Knowledge Bases, Documents, and Embeddings (chunks) as they are typically retrieved from or represented in the system.

#### `KnowledgeBaseData`

```typescript
export interface KnowledgeBaseData {
  id: string
  userId: string
  workspaceId?: string | null
  name: string
  description?: string | null
  tokenCount: number
  embeddingModel: string
  embeddingDimension: number
  chunkingConfig: unknown
  deletedAt?: Date | null
  createdAt: Date
  updatedAt: Date
}
```

*   `export interface KnowledgeBaseData`: Defines the shape of a Knowledge Base entity.
*   `id: string`: Unique identifier for the knowledge base.
*   `userId: string`: The ID of the user who owns this knowledge base.
*   `workspaceId?: string | null`: Optional ID of the workspace this knowledge base belongs to. `?` makes it optional, `| null` indicates it can explicitly be `null`.
*   `name: string`: The name of the knowledge base.
*   `description?: string | null`: Optional description.
*   `tokenCount: number`: The total number of tokens across all documents in this knowledge base.
*   `embeddingModel: string`: The name of the embedding model used for this knowledge base (e.g., "text-embedding-ada-002").
*   `embeddingDimension: number`: The dimension of the embeddings generated by the model.
*   `chunkingConfig: unknown`: Configuration details for how documents are split into chunks. `unknown` is used as the specific structure isn't critical for this file.
*   `deletedAt?: Date | null`: Optional timestamp indicating when the knowledge base was soft-deleted. If null, it's not deleted.
*   `createdAt: Date`: Timestamp when the knowledge base was created.
*   `updatedAt: Date`: Timestamp when the knowledge base was last updated.

#### `DocumentData`

```typescript
export interface DocumentData {
  id: string
  knowledgeBaseId: string
  filename: string
  fileUrl: string
  fileSize: number
  mimeType: string
  chunkCount: number
  tokenCount: number
  characterCount: number
  processingStatus: string
  processingStartedAt?: Date | null
  processingCompletedAt?: Date | null
  processingError?: string | null
  enabled: boolean
  deletedAt?: Date | null
  uploadedAt: Date
  // Document tags
  tag1?: string | null
  tag2?: string | null
  tag3?: string | null
  tag4?: string | null
  tag5?: string | null
  tag6?: string | null
  tag7?: string | null
}
```

*   `export interface DocumentData`: Defines the shape of a Document entity.
*   `id: string`: Unique identifier for the document.
*   `knowledgeBaseId: string`: The ID of the knowledge base this document belongs to.
*   `filename: string`: The original filename.
*   `fileUrl: string`: The URL where the actual file content is stored.
*   `fileSize: number`: Size of the file in bytes.
*   `mimeType: string`: MIME type of the file (e.g., "application/pdf", "text/plain").
*   `chunkCount: number`: Number of chunks (embeddings) generated from this document.
*   `tokenCount: number`: Total number of tokens in this document.
*   `characterCount: number`: Total number of characters in this document.
*   `processingStatus: string`: Current status of document processing (e.g., 'uploaded', 'processing', 'completed', 'failed').
*   `processingStartedAt?: Date | null`: Timestamp when processing began.
*   `processingCompletedAt?: Date | null`: Timestamp when processing finished.
*   `processingError?: string | null`: Any error message if processing failed.
*   `enabled: boolean`: Whether the document is active/enabled.
*   `deletedAt?: Date | null`: Soft deletion timestamp.
*   `uploadedAt: Date`: Timestamp when the document was uploaded.
*   `tag1` to `tag7`: Optional string fields for custom tags or metadata associated with the document.

#### `EmbeddingData`

```typescript
export interface EmbeddingData {
  id: string
  knowledgeBaseId: string
  documentId: string
  chunkIndex: number
  chunkHash: string
  content: string
  contentLength: number
  tokenCount: number
  embedding?: number[] | null
  embeddingModel: string
  startOffset: number
  endOffset: number
  // Tag fields for filtering
  tag1?: string | null
  tag2?: string | null
  tag3?: string | null
  tag4?: string | null
  tag5?: string | null
  tag6?: string | null
  tag7?: string | null
  enabled: boolean
  createdAt: Date
  updatedAt: Date
}
```

*   `export interface EmbeddingData`: Defines the shape of an Embedding entity, often referred to as a "chunk" of a document.
*   `id: string`: Unique identifier for the embedding/chunk.
*   `knowledgeBaseId: string`: The ID of the knowledge base this chunk's parent document belongs to.
*   `documentId: string`: The ID of the document this chunk originated from.
*   `chunkIndex: number`: The numerical index of this chunk within its document.
*   `chunkHash: string`: A hash of the chunk's content, useful for identifying changes.
*   `content: string`: The actual text content of this chunk.
*   `contentLength: number`: Length of the `content` string.
*   `tokenCount: number`: Number of tokens in this chunk's content.
*   `embedding?: number[] | null`: The actual numerical vector (array of numbers) representing the embedding. It's optional as it might be generated later.
*   `embeddingModel: string`: The embedding model used to generate this specific embedding.
*   `startOffset: number`: The starting character index of this chunk within the original document.
*   `endOffset: number`: The ending character index of this chunk within the original document.
*   `tag1` to `tag7`: Optional string fields for custom tags, inherited or derived from the document or specific to the chunk.
*   `enabled: boolean`: Whether this chunk is active/enabled.
*   `createdAt: Date`: Timestamp when the embedding was created.
*   `updatedAt: Date`: Timestamp when the embedding was last updated.

### Access Result Interfaces

These interfaces define the standardized structure for the return values of the access check functions, allowing for clear and type-safe handling of success and failure states. They use a **discriminated union** pattern with the `hasAccess` property.

#### `KnowledgeBaseAccessResult` / `KnowledgeBaseAccessDenied` / `KnowledgeBaseAccessCheck`

```typescript
export interface KnowledgeBaseAccessResult {
  hasAccess: true
  knowledgeBase: Pick<KnowledgeBaseData, 'id' | 'userId'>
}

export interface KnowledgeBaseAccessDenied {
  hasAccess: false
  notFound?: boolean
  reason?: string
}

export type KnowledgeBaseAccessCheck = KnowledgeBaseAccessResult | KnowledgeBaseAccessDenied
```

*   `KnowledgeBaseAccessResult`: Represents a successful access check.
    *   `hasAccess: true`: A boolean literal indicating success.
    *   `knowledgeBase: Pick<KnowledgeBaseData, 'id' | 'userId'>`: Provides a subset of `KnowledgeBaseData` (only `id` and `userId`) when access is granted. `Pick` is a TypeScript utility type that creates a new type by picking specified properties from an existing type. This is efficient as only the necessary data is returned.
*   `KnowledgeBaseAccessDenied`: Represents a failed access check.
    *   `hasAccess: false`: A boolean literal indicating failure.
    *   `notFound?: boolean`: Optional flag, `true` if the entity was not found at all.
    *   `reason?: string`: Optional message explaining why access was denied (e.g., "Unauthorized", "Not found").
*   `KnowledgeBaseAccessCheck`: A **union type** that combines the `KnowledgeBaseAccessResult` and `KnowledgeBaseAccessDenied` interfaces. This allows functions to return either a success or a denial object, and TypeScript can infer the exact shape based on the `hasAccess` property (the discriminator).

The `DocumentAccessResult`/`DocumentAccessDenied`/`DocumentAccessCheck` and `ChunkAccessResult`/`ChunkAccessDenied`/`ChunkAccessCheck` interfaces follow the exact same pattern, providing the full `DocumentData` or `EmbeddingData` along with parent `KnowledgeBaseData` when access is granted.

### Core Access Check Functions

These are the main functions that implement the access control logic.

#### `checkKnowledgeBaseAccess`

```typescript
/**
 * Check if a user has access to a knowledge base
 */
export async function checkKnowledgeBaseAccess(
  knowledgeBaseId: string,
  userId: string
): Promise<KnowledgeBaseAccessCheck> {
  // Line 1: Query the database for the knowledge base
  const kb = await db
    .select({
      id: knowledgeBase.id,
      userId: knowledgeBase.userId,
      workspaceId: knowledgeBase.workspaceId,
    })
    .from(knowledgeBase)
    .where(and(eq(knowledgeBase.id, knowledgeBaseId), isNull(knowledgeBase.deletedAt)))
    .limit(1)

  // Line 2: Check if the knowledge base was found
  if (kb.length === 0) {
    return { hasAccess: false, notFound: true }
  }

  // Line 3: Extract the first (and only) result
  const kbData = kb[0]

  // Line 4: Case 1: User owns the knowledge base directly
  if (kbData.userId === userId) {
    return { hasAccess: true, knowledgeBase: kbData }
  }

  // Line 5: Case 2: Knowledge base belongs to a workspace the user has permissions for
  if (kbData.workspaceId) { // Check if the knowledge base is associated with a workspace
    const userPermission = await getUserEntityPermissions(userId, 'workspace', kbData.workspaceId)
    // If user has any permission (not null), access is granted
    if (userPermission !== null) {
      return { hasAccess: true, knowledgeBase: kbData }
    }
  }

  // Line 6: If neither of the above conditions is met, access is denied
  return { hasAccess: false }
}
```

**Purpose:** Determines if a user has *read* access to a specific knowledge base.

*   **Line 1 (Database Query):**
    *   `const kb = await db.select({...}).from(knowledgeBase).where(and(...)).limit(1)`: This performs an asynchronous database query using Drizzle ORM.
    *   `.select({ id: knowledgeBase.id, userId: knowledgeBase.userId, workspaceId: knowledgeBase.workspaceId })`: Specifies that only the `id`, `userId`, and `workspaceId` columns should be retrieved from the `knowledgeBase` table. These are the only columns needed for this access check.
    *   `.from(knowledgeBase)`: Specifies the table to query.
    *   `.where(and(eq(knowledgeBase.id, knowledgeBaseId), isNull(knowledgeBase.deletedAt)))`: This is the filtering condition.
        *   `and(...)`: Ensures both conditions inside must be true.
        *   `eq(knowledgeBase.id, knowledgeBaseId)`: Matches the knowledge base by its provided `id`.
        *   `isNull(knowledgeBase.deletedAt)`: Crucially, ensures that the knowledge base has *not* been soft-deleted.
    *   `.limit(1)`: Optimizes the query by telling the database to stop after finding the first matching record, as we only expect one.
*   **Line 2 (Knowledge Base Not Found):**
    *   `if (kb.length === 0)`: Checks if the database query returned any results. If `kb` is an empty array, no knowledge base matching the ID and not deleted was found.
    *   `return { hasAccess: false, notFound: true }`: Returns a denial object, specifically indicating that the knowledge base itself was not found.
*   **Line 3 (Extract Data):**
    *   `const kbData = kb[0]`: Since `limit(1)` was used, if a result exists, it will be the first element of the `kb` array. This extracts that data.
*   **Line 4 (Direct Ownership Check):**
    *   `if (kbData.userId === userId)`: Checks if the `userId` passed to the function matches the `userId` (owner) of the retrieved knowledge base.
    *   `return { hasAccess: true, knowledgeBase: kbData }`: If they match, access is granted, and the relevant knowledge base data is returned.
*   **Line 5 (Workspace Permission Check):**
    *   `if (kbData.workspaceId)`: Checks if the knowledge base is associated with a `workspaceId`. If `workspaceId` is null, this condition is skipped.
    *   `const userPermission = await getUserEntityPermissions(userId, 'workspace', kbData.workspaceId)`: Calls the external utility function to get the user's permissions for the *workspace* associated with this knowledge base.
    *   `if (userPermission !== null)`: If `getUserEntityPermissions` returns anything other than `null` (meaning the user has *any* permission on that workspace), access is granted.
    *   `return { hasAccess: true, knowledgeBase: kbData }`: Returns a success object.
*   **Line 6 (Default Deny):**
    *   `return { hasAccess: false }`: If none of the above conditions (owner, workspace permissions) grant access, then access is denied by default.

#### `checkKnowledgeBaseWriteAccess`

```typescript
/**
 * Check if a user has write access to a knowledge base
 * Write access is granted if:
 * 1. User owns the knowledge base directly, OR
 * 2. User has write or admin permissions on the knowledge base's workspace
 */
export async function checkKnowledgeBaseWriteAccess(
  knowledgeBaseId: string,
  userId: string
): Promise<KnowledgeBaseAccessCheck> {
  // Lines 1-3: Same as checkKnowledgeBaseAccess – fetch KB and handle not found
  const kb = await db
    .select({
      id: knowledgeBase.id,
      userId: knowledgeBase.userId,
      workspaceId: knowledgeBase.workspaceId,
    })
    .from(knowledgeBase)
    .where(and(eq(knowledgeBase.id, knowledgeBaseId), isNull(knowledgeBase.deletedAt)))
    .limit(1)

  if (kb.length === 0) {
    return { hasAccess: false, notFound: true }
  }

  const kbData = kb[0]

  // Line 4: Case 1: User owns the knowledge base directly (same as read access)
  if (kbData.userId === userId) {
    return { hasAccess: true, knowledgeBase: kbData }
  }

  // Line 5: Case 2: Knowledge base belongs to a workspace and user has write/admin permissions
  if (kbData.workspaceId) { // Check if the knowledge base is associated with a workspace
    const userPermission = await getUserEntityPermissions(userId, 'workspace', kbData.workspaceId)
    // This is the key difference: user must have 'write' or 'admin' permission
    if (userPermission === 'write' || userPermission === 'admin') {
      return { hasAccess: true, knowledgeBase: kbData }
    }
  }

  // Line 6: Default deny if no write access is explicitly granted
  return { hasAccess: false }
}
```

**Purpose:** Determines if a user has *write* access to a specific knowledge base. The logic is very similar to `checkKnowledgeBaseAccess`, with a stricter requirement for workspace permissions.

*   **Lines 1-3:** Identical to `checkKnowledgeBaseAccess`. It fetches the knowledge base and handles the "not found" scenario.
*   **Line 4 (Direct Ownership Check):** Identical to `checkKnowledgeBaseAccess`. If the user owns the KB, they have write access.
*   **Line 5 (Workspace Permission Check - *Key Difference*):**
    *   `if (kbData.workspaceId)`: Checks for workspace association.
    *   `const userPermission = await getUserEntityPermissions(userId, 'workspace', kbData.workspaceId)`: Fetches user's permissions for the workspace.
    *   `if (userPermission === 'write' || userPermission === 'admin')`: **This is the critical distinction for write access.** Instead of `userPermission !== null`, it specifically checks if the permission level is 'write' or 'admin'. Other permissions (like 'read') would not grant write access.
    *   `return { hasAccess: true, knowledgeBase: kbData }`: Returns success if the user has sufficient workspace permissions.
*   **Line 6 (Default Deny):** Identical to `checkKnowledgeBaseAccess`.

#### `checkDocumentWriteAccess`

```typescript
/**
 * Check if a user has write access to a specific document
 * Write access is granted if user has write access to the knowledge base
 */
export async function checkDocumentWriteAccess(
  knowledgeBaseId: string,
  documentId: string,
  userId: string
): Promise<DocumentAccessCheck> {
  // Line 1: First check if user has write access to the knowledge base (cascading permission)
  const kbAccess = await checkKnowledgeBaseWriteAccess(knowledgeBaseId, userId)

  // Line 2: If KB write access is denied, document write access is also denied
  if (!kbAccess.hasAccess) {
    return {
      hasAccess: false,
      notFound: kbAccess.notFound, // Pass through if KB was not found
      reason: kbAccess.notFound ? 'Knowledge base not found' : 'Unauthorized knowledge base access',
    }
  }

  // Line 3: Check if the document exists and belongs to the specified knowledge base
  const doc = await db
    .select({
      id: document.id,
      filename: document.filename,
      fileUrl: document.fileUrl,
      fileSize: document.fileSize,
      mimeType: document.mimeType,
      chunkCount: document.chunkCount,
      tokenCount: document.tokenCount,
      characterCount: document.characterCount,
      enabled: document.enabled,
      processingStatus: document.processingStatus,
      processingError: document.processingError,
      uploadedAt: document.uploadedAt,
      processingStartedAt: document.processingStartedAt,
      processingCompletedAt: document.processingCompletedAt,
      knowledgeBaseId: document.knowledgeBaseId, // Include knowledgeBaseId for verification
    })
    .from(document)
    .where(
      and(
        eq(document.id, documentId),
        eq(document.knowledgeBaseId, knowledgeBaseId), // Ensure document belongs to the KB
        isNull(document.deletedAt)
      )
    )
    .limit(1)

  // Line 4: If the document was not found or not associated with the KB
  if (doc.length === 0) {
    return { hasAccess: false, notFound: true, reason: 'Document not found' }
  }

  // Line 5: If both KB write access and document existence checks pass, grant access
  return {
    hasAccess: true,
    document: doc[0] as DocumentData, // Cast the result to the full DocumentData interface
    knowledgeBase: kbAccess.knowledgeBase!, // Use the knowledge base info from the prior check
  }
}
```

**Purpose:** Determines if a user has *write* access to a specific document. This function demonstrates **cascading permissions**, where access to a child entity (document) depends on access to its parent (knowledge base).

*   **Line 1 (Knowledge Base Write Access Check):**
    *   `const kbAccess = await checkKnowledgeBaseWriteAccess(knowledgeBaseId, userId)`: Calls the `checkKnowledgeBaseWriteAccess` function first. This is crucial: you can't have write access to a document if you don't have write access to its parent knowledge base.
*   **Line 2 (Handle KB Access Denial):**
    *   `if (!kbAccess.hasAccess)`: If the `kbAccess` check failed, immediately deny access to the document.
    *   `return { hasAccess: false, notFound: kbAccess.notFound, reason: ... }`: Returns a denial object, reusing the `notFound` status and providing a reason based on the KB access outcome.
*   **Line 3 (Document Database Query):**
    *   `const doc = await db.select({...}).from(document).where(and(...)).limit(1)`: Queries the `document` table.
    *   `.select({...})`: Selects all relevant document fields.
    *   `.where(and(eq(document.id, documentId), eq(document.knowledgeBaseId, knowledgeBaseId), isNull(document.deletedAt)))`: Filters by:
        *   `eq(document.id, documentId)`: Matches the document by its ID.
        *   `eq(document.knowledgeBaseId, knowledgeBaseId)`: **Crucially**, ensures that the document actually belongs to the specified `knowledgeBaseId`. This prevents a user with KB access from accessing *any* document by just guessing a `documentId`.
        *   `isNull(document.deletedAt)`: Ensures the document is not soft-deleted.
*   **Line 4 (Document Not Found):**
    *   `if (doc.length === 0)`: If the document was not found (or didn't match the `knowledgeBaseId` or was deleted).
    *   `return { hasAccess: false, notFound: true, reason: 'Document not found' }`: Returns a denial.
*   **Line 5 (Grant Access):**
    *   `return { hasAccess: true, document: doc[0] as DocumentData, knowledgeBase: kbAccess.knowledgeBase! }`: If both the knowledge base write access and the document existence/ownership checks pass, grant access.
        *   `doc[0] as DocumentData`: Casts the Drizzle result to the `DocumentData` interface.
        *   `kbAccess.knowledgeBase!`: Uses the knowledge base data obtained from the initial `kbAccess` check. The `!` (non-null assertion operator) is used here because TypeScript knows `kbAccess.hasAccess` was true, so `kbAccess.knowledgeBase` must be present.

#### `checkDocumentAccess`

```typescript
/**
 * Check if a user has access to a document within a knowledge base
 */
export async function checkDocumentAccess(
  knowledgeBaseId: string,
  documentId: string,
  userId: string
): Promise<DocumentAccessCheck> {
  // Line 1: First check if user has read access to the knowledge base
  const kbAccess = await checkKnowledgeBaseAccess(knowledgeBaseId, userId)

  // Line 2: If KB read access is denied, document access is also denied
  if (!kbAccess.hasAccess) {
    return {
      hasAccess: false,
      notFound: kbAccess.notFound,
      reason: kbAccess.notFound ? 'Knowledge base not found' : 'Unauthorized knowledge base access',
    }
  }

  // Line 3: Query for the document, ensuring it belongs to the KB and is not deleted
  const doc = await db
    .select() // Select all columns for the document
    .from(document)
    .where(
      and(
        eq(document.id, documentId),
        eq(document.knowledgeBaseId, knowledgeBaseId),
        isNull(document.deletedAt)
      )
    )
    .limit(1)

  // Line 4: If document not found
  if (doc.length === 0) {
    return { hasAccess: false, notFound: true, reason: 'Document not found' }
  }

  // Line 5: Grant access
  return {
    hasAccess: true,
    document: doc[0] as DocumentData,
    knowledgeBase: kbAccess.knowledgeBase!,
  }
}
```

**Purpose:** Determines if a user has *read* access to a specific document. This is similar to `checkDocumentWriteAccess`, but it uses `checkKnowledgeBaseAccess` (for read permissions) as its initial check.

*   **Line 1 (Knowledge Base Read Access Check):**
    *   `const kbAccess = await checkKnowledgeBaseAccess(knowledgeBaseId, userId)`: Calls the `checkKnowledgeBaseAccess` function (for read permissions).
*   **Line 2 (Handle KB Access Denial):** Identical to `checkDocumentWriteAccess`.
*   **Line 3 (Document Database Query):**
    *   `const doc = await db.select().from(document).where(and(...)).limit(1)`: Queries the `document` table.
    *   `.select()`: Selects *all* columns of the document. This is different from `checkDocumentWriteAccess` where specific columns were picked, but both are valid.
    *   `.where(and(eq(document.id, documentId), eq(document.knowledgeBaseId, knowledgeBaseId), isNull(document.deletedAt)))`: Filters identically to `checkDocumentWriteAccess` to ensure correct document ownership and non-deletion.
*   **Line 4 (Document Not Found):** Identical to `checkDocumentWriteAccess`.
*   **Line 5 (Grant Access):** Identical to `checkDocumentWriteAccess`.

#### `checkChunkAccess`

```typescript
/**
 * Check if a user has access to a chunk within a document and knowledge base
 */
export async function checkChunkAccess(
  knowledgeBaseId: string,
  documentId: string,
  chunkId: string,
  userId: string
): Promise<ChunkAccessCheck> {
  // Line 1: First check if user has read access to the knowledge base (cascading permission)
  const kbAccess = await checkKnowledgeBaseAccess(knowledgeBaseId, userId)

  // Line 2: If KB access is denied, chunk access is also denied
  if (!kbAccess.hasAccess) {
    return {
      hasAccess: false,
      notFound: kbAccess.notFound,
      reason: kbAccess.notFound ? 'Knowledge base not found' : 'Unauthorized knowledge base access',
    }
  }

  // Line 3: Query for the parent document, ensuring it belongs to the KB and is not deleted
  const doc = await db
    .select() // Select all columns for the document
    .from(document)
    .where(
      and(
        eq(document.id, documentId),
        eq(document.knowledgeBaseId, knowledgeBaseId),
        isNull(document.deletedAt)
      )
    )
    .limit(1)

  // Line 4: If document not found
  if (doc.length === 0) {
    return { hasAccess: false, notFound: true, reason: 'Document not found' }
  }

  // Line 5: Extract document data and check processing status (crucial for chunks)
  const docData = doc[0] as DocumentData

  // If document processing is not completed, chunks are not ready for access
  if (docData.processingStatus !== 'completed') {
    return {
      hasAccess: false,
      reason: `Document is not ready for access (status: ${docData.processingStatus})`,
    }
  }

  // Line 6: Query for the specific embedding (chunk), ensuring it belongs to the document
  const chunk = await db
    .select() // Select all columns for the embedding/chunk
    .from(embedding)
    .where(and(eq(embedding.id, chunkId), eq(embedding.documentId, documentId)))
    .limit(1)

  // Line 7: If chunk not found
  if (chunk.length === 0) {
    return { hasAccess: false, notFound: true, reason: 'Chunk not found' }
  }

  // Line 8: Grant access if all checks pass
  return {
    hasAccess: true,
    chunk: chunk[0] as EmbeddingData, // Cast the result to EmbeddingData
    document: docData, // Provide the parent document data
    knowledgeBase: kbAccess.knowledgeBase!, // Provide the parent knowledge base data
  }
}
```

**Purpose:** Determines if a user has *read* access to a specific embedding (chunk) within a document. This function has the deepest cascading permission checks and an additional check for document processing status.

*   **Line 1 (Knowledge Base Read Access Check):**
    *   `const kbAccess = await checkKnowledgeBaseAccess(knowledgeBaseId, userId)`: First, checks if the user has read access to the parent knowledge base.
*   **Line 2 (Handle KB Access Denial):** Identical to previous document checks.
*   **Line 3 (Document Database Query):**
    *   `const doc = await db.select().from(document).where(and(...)).limit(1)`: Queries for the parent document, ensuring it belongs to the `knowledgeBaseId` and is not deleted. This is very similar to `checkDocumentAccess`.
*   **Line 4 (Document Not Found):** Identical to previous document checks.
*   **Line 5 (Document Processing Status Check - *New Critical Step*):**
    *   `const docData = doc[0] as DocumentData`: Extracts the document data.
    *   `if (docData.processingStatus !== 'completed')`: **This is a unique and critical check for chunk access.** If the document's processing status is not `completed` (meaning chunks might not be fully generated or indexed), access to individual chunks is denied.
    *   `return { hasAccess: false, reason: ... }`: Returns a denial with a specific reason.
*   **Line 6 (Chunk Database Query):**
    *   `const chunk = await db.select().from(embedding).where(and(eq(embedding.id, chunkId), eq(embedding.documentId, documentId))).limit(1)`: Queries the `embedding` (chunk) table.
    *   `.where(and(eq(embedding.id, chunkId), eq(embedding.documentId, documentId)))`: Filters by:
        *   `eq(embedding.id, chunkId)`: Matches the embedding by its ID.
        *   `eq(embedding.documentId, documentId)`: **Crucially**, ensures the chunk belongs to the specified `documentId`. This prevents access to chunks from other documents by guessing IDs. (Note: `isNull(embedding.deletedAt)` is *not* present here, implying embeddings might not have a soft-delete mechanism, or it's handled at the document level).
*   **Line 7 (Chunk Not Found):**
    *   `if (chunk.length === 0)`: If the chunk was not found or didn't belong to the specified document.
    *   `return { hasAccess: false, notFound: true, reason: 'Chunk not found' }`: Returns a denial.
*   **Line 8 (Grant Access):**
    *   `return { hasAccess: true, chunk: chunk[0] as EmbeddingData, document: docData, knowledgeBase: kbAccess.knowledgeBase! }`: If all checks (KB access, document existence, document processing status, and chunk existence/ownership) pass, grant access and return all relevant data.

This file provides a robust and layered approach to access control, leveraging database queries, external permission utilities, and a clear, type-safe return structure.