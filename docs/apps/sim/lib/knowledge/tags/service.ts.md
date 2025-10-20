This file, `tags.service.ts`, acts as a central **Tags Service** within a larger application that manages "knowledge bases," "documents," and "embeddings" (which are typically chunks or parts of documents).

Its core purpose is to provide a comprehensive set of functions for:
1.  **Defining and managing tags:** Creating, updating, and deleting what a tag *is* (e.g., "Category", "Author", "Topic") and what type of data it holds (e.g., text, number). These are called "tag definitions."
2.  **Assigning and managing tags on actual content:** Applying these defined tags to `document` and `embedding` records in the database.
3.  **Handling "tag slots":** Managing the limited, predefined database columns (like `tag1`, `tag2`) where tag values are stored.
4.  **Providing usage statistics:** Reporting how many documents or embeddings use a particular tag.

In essence, this file is responsible for the entire lifecycle and management of tags within the knowledge base system, from their definition to their application and eventual cleanup.

---

## Detailed Code Explanation

Let's break down the code section by section, explaining its purpose and each line.

### 1. Imports

```typescript
import { randomUUID } from 'crypto'
import { db } from '@sim/db'
import { document, embedding, knowledgeBaseTagDefinitions } from '@sim/db/schema'
import { and, eq, isNotNull, isNull, sql } from 'drizzle-orm'
import {
  getSlotsForFieldType,
  SUPPORTED_FIELD_TYPES,
  type TAG_SLOT_CONFIG,
} from '@/lib/knowledge/consts'
import type { BulkTagDefinitionsData, DocumentTagDefinition } from '@/lib/knowledge/tags/types'
import type {
  CreateTagDefinitionData,
  TagDefinition,
  UpdateTagDefinitionData,
} from '@/lib/knowledge/types'
import { createLogger } from '@/lib/logs/console/logger'
```

*   **`import { randomUUID } from 'crypto'`**: Imports the `randomUUID` function from Node.js's built-in `crypto` module. This function generates universally unique identifiers (UUIDs), which are often used as primary keys for database records.
*   **`import { db } from '@sim/db'`**: Imports the Drizzle ORM database client instance, named `db`. This object is used to interact with the database, performing queries like `SELECT`, `INSERT`, `UPDATE`, and `DELETE`.
*   **`import { document, embedding, knowledgeBaseTagDefinitions } from '@sim/db/schema'`**: Imports the schema definitions for three database tables:
    *   `document`: Represents documents stored in the system.
    *   `embedding`: Represents chunks or segments of documents (often used in AI/ML contexts for vector embeddings).
    *   `knowledgeBaseTagDefinitions`: Stores the definitions of tags (e.g., "Category" tag, "Author" tag) for different knowledge bases.
    These schema objects allow Drizzle to build type-safe queries.
*   **`import { and, eq, isNotNull, isNull, sql } from 'drizzle-orm'`**: Imports various helper functions from Drizzle ORM to construct database queries:
    *   `and`: Combines multiple conditions with a logical AND.
    *   `eq`: Checks for equality (e.g., `column = value`).
    *   `isNotNull`: Checks if a column's value is not `NULL`.
    *   `isNull`: Checks if a column's value is `NULL`.
    *   `sql`: Allows embedding raw SQL expressions into Drizzle queries. This is particularly important when dealing with dynamic column names, as seen later.
*   **`import { getSlotsForFieldType, SUPPORTED_FIELD_TYPES, type TAG_SLOT_CONFIG, } from '@/lib/knowledge/consts'`**: Imports constants and utility functions related to how tags are structured:
    *   `getSlotsForFieldType`: A function that determines which specific `tagSlot` (e.g., `tag1`, `tag2`) are available for a given `fieldType` (e.g., "text", "number").
    *   `SUPPORTED_FIELD_TYPES`: An array listing all data types that tags can represent (e.g., `['text', 'number', 'date']`).
    *   `TAG_SLOT_CONFIG`: A type definition that likely describes the configuration of tag slots.
*   **`import type { BulkTagDefinitionsData, DocumentTagDefinition } from '@/lib/knowledge/tags/types'`**: Imports TypeScript types specific to tag operations:
    *   `BulkTagDefinitionsData`: The structure of data expected when performing bulk operations on tag definitions.
    *   `DocumentTagDefinition`: The shape of a tag definition as it relates to documents.
*   **`import type { CreateTagDefinitionData, TagDefinition, UpdateTagDefinitionData, } from '@/lib/knowledge/types'`**: Imports more general TypeScript types for tag definitions:
    *   `CreateTagDefinitionData`: The data needed to create a new tag definition.
    *   `TagDefinition`: The general shape of a tag definition object.
    *   `UpdateTagDefinitionData`: The data needed to update an existing tag definition.
*   **`import { createLogger } from '@/lib/logs/console/logger'`**: Imports a utility function to create a logger instance, which helps in debugging and monitoring the service's operations.

### 2. Logger Initialization

```typescript
const logger = createLogger('TagsService')
```

*   **`const logger = createLogger('TagsService')`**: Initializes a logger instance specifically for this `TagsService`. This allows all logs originating from this file to be easily identifiable by the `TagsService` prefix, making it easier to filter and understand log output.

### 3. Tag Slot Validation

```typescript
const VALID_TAG_SLOTS = ['tag1', 'tag2', 'tag3', 'tag4', 'tag5', 'tag6', 'tag7'] as const

function validateTagSlot(tagSlot: string): asserts tagSlot is (typeof VALID_TAG_SLOTS)[number] {
  if (!VALID_TAG_SLOTS.includes(tagSlot as (typeof VALID_TAG_SLOTS)[number])) {
    throw new Error(`Invalid tag slot: ${tagSlot}. Must be one of: ${VALID_TAG_SLOTS.join(', ')}`)
  }
}
```

*   **`const VALID_TAG_SLOTS = [...] as const`**: Defines a constant array `VALID_TAG_SLOTS`. These represent the actual column names in the `document` and `embedding` tables that are reserved for storing tag values (e.g., `tag1`, `tag2`). The `as const` assertion tells TypeScript that this array's values are immutable and provides a literal type for its elements, which is useful for type safety.
*   **`function validateTagSlot(tagSlot: string): asserts tagSlot is (typeof VALID_TAG_SLOTS)[number]`**: This is a **type assertion function**.
    *   It takes a `tagSlot` string as input.
    *   The `asserts tagSlot is (typeof VALID_TAG_SLOTS)[number]` return type signature is crucial: it tells TypeScript that *if this function completes without throwing an error*, then `tagSlot` is guaranteed to be one of the literal string types defined in `VALID_TAG_SLOTS` (e.g., `'tag1'`, `'tag2'`). This is important for ensuring that dynamic column names used in `sql.raw` expressions are always valid and secure.
    *   **`if (!VALID_TAG_SLOTS.includes(tagSlot as (typeof VALID_TAG_SLOTS)[number]))`**: Checks if the provided `tagSlot` string exists within the `VALID_TAG_SLOTS` array. The type assertion `as (typeof VALID_TAG_SLOTS)[number]` is used here to satisfy TypeScript that `tagSlot` is potentially one of the expected literal types for the `includes` method.
    *   **`throw new Error(...)`**: If `tagSlot` is not found in `VALID_TAG_SLOTS`, an `Error` is thrown, preventing the use of an invalid (and potentially dangerous) column name in a database query.

### 4. `getNextAvailableSlot` Function

```typescript
/**
 * Get the next available slot for a knowledge base and field type
 */
export async function getNextAvailableSlot(
  knowledgeBaseId: string,
  fieldType: string,
  existingBySlot?: Map<string, any>
): Promise<string | null> {
  const availableSlots = getSlotsForFieldType(fieldType)
  let usedSlots: Set<string>

  if (existingBySlot) {
    usedSlots = new Set(
      Array.from(existingBySlot.entries())
        .filter(([_, def]) => def.fieldType === fieldType)
        .map(([slot, _]) => slot)
    )
  } else {
    const existingDefinitions = await db
      .select({ tagSlot: knowledgeBaseTagDefinitions.tagSlot })
      .from(knowledgeBaseTagDefinitions)
      .where(
        and(
          eq(knowledgeBaseTagDefinitions.knowledgeBaseId, knowledgeBaseId),
          eq(knowledgeBaseTagDefinitions.fieldType, fieldType)
        )
      )

    usedSlots = new Set(existingDefinitions.map((def) => def.tagSlot))
  }

  for (const slot of availableSlots) {
    if (!usedSlots.has(slot)) {
      return slot
    }
  }

  return null // All slots for this field type are used
}
```

*   **Purpose**: This function finds the first unused "tag slot" (e.g., `tag1`, `tag2`) for a specific knowledge base and a given field type. This is crucial because there's a limited number of `tagX` columns available in the database.
*   **`export async function getNextAvailableSlot(...)`**: Defines an asynchronous function that can be called from other modules.
    *   `knowledgeBaseId: string`: The ID of the knowledge base to check.
    *   `fieldType: string`: The data type of the tag (e.g., 'text', 'number'). Different field types might have different sets of available slots.
    *   `existingBySlot?: Map<string, any>`: An optional `Map` containing already known tag definitions, keyed by their `tagSlot`. This is an optimization, particularly for bulk operations, to avoid extra database queries.
    *   `Promise<string | null>`: The function will return a `string` (the available slot) or `null` if no slot is found.
*   **`const availableSlots = getSlotsForFieldType(fieldType)`**: Calls a utility function to get all potential tag slots that are suitable for the specified `fieldType`. For example, `getSlotsForFieldType('text')` might return `['tag1', 'tag2', 'tag3']`.
*   **`let usedSlots: Set<string>`**: Declares a `Set` to store the tag slots that are currently in use. A `Set` is used for efficient checking of membership (i.e., `usedSlots.has(slot)`).
*   **`if (existingBySlot)`**: Checks if the optional `existingBySlot` map was provided.
    *   **`usedSlots = new Set(...)`**: If `existingBySlot` is present, it constructs the `usedSlots` set by:
        *   `Array.from(existingBySlot.entries())`: Converts the map's key-value pairs into an array of `[key, value]` arrays.
        *   `.filter(([_, def]) => def.fieldType === fieldType)`: Filters these entries to only include definitions that match the current `fieldType`.
        *   `.map(([slot, _]) => slot)`: Extracts only the `tagSlot` (the key) from the filtered entries.
        This path avoids a database call if the information is already available in memory.
*   **`else { ... }`**: If `existingBySlot` was *not* provided, a database query is made:
    *   **`const existingDefinitions = await db .select({ tagSlot: knowledgeBaseTagDefinitions.tagSlot }) .from(knowledgeBaseTagDefinitions) .where( and( eq(knowledgeBaseTagDefinitions.knowledgeBaseId, knowledgeBaseId), eq(knowledgeBaseTagDefinitions.fieldType, fieldType) ) )`**:
        *   `await db.select(...)`: Initiates a Drizzle `SELECT` query.
        *   `{ tagSlot: knowledgeBaseTagDefinitions.tagSlot }`: Selects only the `tagSlot` column from the `knowledgeBaseTagDefinitions` table.
        *   `.from(knowledgeBaseTagDefinitions)`: Specifies the table to query.
        *   `.where(and(...))`: Filters the results using an `AND` condition.
            *   `eq(knowledgeBaseTagDefinitions.knowledgeBaseId, knowledgeBaseId)`: Matches definitions belonging to the given `knowledgeBaseId`.
            *   `eq(knowledgeBaseTagDefinitions.fieldType, fieldType)`: Matches definitions that also have the specified `fieldType`.
    *   **`usedSlots = new Set(existingDefinitions.map((def) => def.tagSlot))`**: Creates the `usedSlots` set from the `tagSlot` values returned by the database query.
*   **`for (const slot of availableSlots)`**: Iterates through each `slot` that is theoretically available for the `fieldType`.
    *   **`if (!usedSlots.has(slot))`**: Checks if the current `slot` is *not* present in the `usedSlots` set (meaning it's not currently assigned to a tag definition).
    *   **`return slot`**: If an unused slot is found, it's immediately returned.
*   **`return null`**: If the loop completes without finding an available slot, it means all suitable slots for that `fieldType` are already in use, so `null` is returned.

### 5. `getDocumentTagDefinitions` Function

```typescript
/**
 * Get all tag definitions for a knowledge base
 */
export async function getDocumentTagDefinitions(
  knowledgeBaseId: string
): Promise<DocumentTagDefinition[]> {
  const definitions = await db
    .select({
      id: knowledgeBaseTagDefinitions.id,
      knowledgeBaseId: knowledgeBaseTagDefinitions.knowledgeBaseId,
      tagSlot: knowledgeBaseTagDefinitions.tagSlot,
      displayName: knowledgeBaseTagDefinitions.displayName,
      fieldType: knowledgeBaseTagDefinitions.fieldType,
      createdAt: knowledgeBaseTagDefinitions.createdAt,
      updatedAt: knowledgeBaseTagDefinitions.updatedAt,
    })
    .from(knowledgeBaseTagDefinitions)
    .where(eq(knowledgeBaseTagDefinitions.knowledgeBaseId, knowledgeBaseId))
    .orderBy(knowledgeBaseTagDefinitions.tagSlot)

  return definitions.map((def) => ({
    ...def,
    tagSlot: def.tagSlot as string,
  }))
}
```

*   **Purpose**: Retrieves all tag definitions configured for a specific `knowledgeBaseId`.
*   **`export async function getDocumentTagDefinitions(...)`**: An asynchronous function.
    *   `knowledgeBaseId: string`: The ID of the knowledge base.
    *   `Promise<DocumentTagDefinition[]>`: Returns an array of tag definition objects.
*   **`const definitions = await db .select({ ... }) .from(knowledgeBaseTagDefinitions) .where(eq(knowledgeBaseTagDefinitions.knowledgeBaseId, knowledgeBaseId)) .orderBy(knowledgeBaseTagDefinitions.tagSlot)`**:
    *   `await db.select(...)`: Performs a `SELECT` query.
    *   `{ id: ..., knowledgeBaseId: ..., tagSlot: ..., displayName: ..., fieldType: ..., createdAt: ..., updatedAt: ... }`: Specifies all the columns to retrieve from the `knowledgeBaseTagDefinitions` table.
    *   `.from(knowledgeBaseTagDefinitions)`: Targets the tag definitions table.
    *   `.where(eq(knowledgeBaseTagDefinitions.knowledgeBaseId, knowledgeBaseId))`: Filters definitions to only those belonging to the given `knowledgeBaseId`.
    *   `.orderBy(knowledgeBaseTagDefinitions.tagSlot)`: Sorts the results by `tagSlot` for consistent ordering.
*   **`return definitions.map((def) => ({ ...def, tagSlot: def.tagSlot as string, }))`**:
    *   Maps over the retrieved `definitions`.
    *   `{ ...def }`: Spreads all existing properties of each definition.
    *   `tagSlot: def.tagSlot as string`: Casts the `tagSlot` property to `string`. This might be necessary if Drizzle's inferred type for `tagSlot` is more specific (e.g., a union of literal strings from `VALID_TAG_SLOTS`), but the `DocumentTagDefinition` type expects a general `string`.

### 6. `getTagDefinitions` Function

```typescript
/**
 * Get all tag definitions for a knowledge base (alias for compatibility)
 */
export async function getTagDefinitions(knowledgeBaseId: string): Promise<TagDefinition[]> {
  const tagDefinitions = await db
    .select({
      id: knowledgeBaseTagDefinitions.id,
      tagSlot: knowledgeBaseTagDefinitions.tagSlot,
      displayName: knowledgeBaseTagDefinitions.displayName,
      fieldType: knowledgeBaseTagDefinitions.fieldType,
      createdAt: knowledgeBaseTagDefinitions.createdAt,
      updatedAt: knowledgeBaseTagDefinitions.updatedAt,
    })
    .from(knowledgeBaseTagDefinitions)
    .where(eq(knowledgeBaseTagDefinitions.knowledgeBaseId, knowledgeBaseId))
    .orderBy(knowledgeBaseTagDefinitions.tagSlot)

  return tagDefinitions.map((def) => ({
    ...def,
    tagSlot: def.tagSlot as string,
  }))
}
```

*   **Purpose**: This function is essentially an alias for `getDocumentTagDefinitions`. It serves the same purpose but returns the definitions as `TagDefinition[]`, likely for backward compatibility or different type expectations in certain parts of the application.
*   **Logic**: It performs the exact same database query as `getDocumentTagDefinitions` and the same mapping operation, only the return type specified is `TagDefinition[]`.

### 7. `createOrUpdateTagDefinitionsBulk` Function

```typescript
/**
 * Create or update tag definitions in bulk
 */
export async function createOrUpdateTagDefinitionsBulk(
  knowledgeBaseId: string,
  bulkData: BulkTagDefinitionsData,
  requestId: string
): Promise<{
  created: DocumentTagDefinition[]
  updated: DocumentTagDefinition[]
  errors: string[]
}> {
  const { definitions } = bulkData
  const created: DocumentTagDefinition[] = []
  const updated: DocumentTagDefinition[] = []
  const errors: string[] = []

  // Get existing definitions to check for conflicts and determine operations
  const existingDefinitions = await getDocumentTagDefinitions(knowledgeBaseId)
  const existingBySlot = new Map(existingDefinitions.map((def) => [def.tagSlot, def]))
  const existingByDisplayName = new Map(existingDefinitions.map((def) => [def.displayName, def]))

  // Process each definition
  for (const defData of definitions) {
    try {
      const { tagSlot, displayName, fieldType, originalDisplayName } = defData

      // Validate field type
      if (!SUPPORTED_FIELD_TYPES.includes(fieldType as (typeof SUPPORTED_FIELD_TYPES)[number])) {
        errors.push(`Invalid field type: ${fieldType}`)
        continue
      }

      // Check if this is an update (has originalDisplayName) or create
      const isUpdate = !!originalDisplayName

      if (isUpdate) {
        // Update existing definition
        const existingDef = existingByDisplayName.get(originalDisplayName!)
        if (!existingDef) {
          errors.push(`Tag definition with display name "${originalDisplayName}" not found`)
          continue
        }

        // Check if new display name conflicts with another definition
        if (displayName !== originalDisplayName && existingByDisplayName.has(displayName)) {
          errors.push(`Display name "${displayName}" already exists`)
          continue
        }

        const now = new Date()
        await db
          .update(knowledgeBaseTagDefinitions)
          .set({
            displayName,
            fieldType,
            updatedAt: now,
          })
          .where(eq(knowledgeBaseTagDefinitions.id, existingDef.id))

        updated.push({
          id: existingDef.id,
          knowledgeBaseId,
          tagSlot: existingDef.tagSlot,
          displayName,
          fieldType,
          createdAt: existingDef.createdAt,
          updatedAt: now,
        })
      } else {
        // Create new definition
        let finalTagSlot = tagSlot

        // If no slot provided or slot is taken, find next available
        if (!finalTagSlot || existingBySlot.has(finalTagSlot)) {
          const nextSlot = await getNextAvailableSlot(knowledgeBaseId, fieldType, existingBySlot)
          if (!nextSlot) {
            errors.push(`No available slots for field type "${fieldType}"`)
            continue
          }
          finalTagSlot = nextSlot
        }

        // Check slot conflicts
        if (existingBySlot.has(finalTagSlot)) {
          errors.push(`Tag slot "${finalTagSlot}" is already in use`)
          continue
        }

        // Check display name conflicts
        if (existingByDisplayName.has(displayName)) {
          errors.push(`Display name "${displayName}" already exists`)
          continue
        }

        const id = randomUUID()
        const now = new Date()

        const newDefinition = {
          id,
          knowledgeBaseId,
          tagSlot: finalTagSlot as (typeof TAG_SLOT_CONFIG.text.slots)[number],
          displayName,
          fieldType,
          createdAt: now,
          updatedAt: now,
        }

        await db.insert(knowledgeBaseTagDefinitions).values(newDefinition)

        // Add to maps to track for subsequent definitions in this batch
        existingBySlot.set(finalTagSlot, newDefinition)
        existingByDisplayName.set(displayName, newDefinition)

        created.push(newDefinition as DocumentTagDefinition)
      }
    } catch (error) {
      errors.push(`Error processing definition "${defData.displayName}": ${error}`)
    }
  }

  logger.info(
    `[${requestId}] Bulk tag definitions processed: ${created.length} created, ${updated.length} updated, ${errors.length} errors`
  )

  return { created, updated, errors }
}
```

*   **Purpose**: This function handles the creation or update of multiple tag definitions in a single, atomic operation (though it's not a database transaction for the entire loop, individual DB operations are separate). It's designed to be robust, handling conflicts and finding available slots.
*   **`export async function createOrUpdateTagDefinitionsBulk(...)`**: An asynchronous function.
    *   `knowledgeBaseId: string`: The ID of the knowledge base.
    *   `bulkData: BulkTagDefinitionsData`: An object containing the array of `definitions` to be processed.
    *   `requestId: string`: A unique ID for the request, used for logging and traceability.
    *   `Promise<{ created: ..., updated: ..., errors: ... }>`: Returns an object summarizing the operation, including arrays of successfully created/updated definitions and any errors encountered.
*   **`const { definitions } = bulkData`**: Destructures the `definitions` array from `bulkData`.
*   **`const created: DocumentTagDefinition[] = []`, `const updated: DocumentTagDefinition[] = []`, `const errors: string[] = []`**: Initializes empty arrays to collect the results of the bulk operation.
*   **`const existingDefinitions = await getDocumentTagDefinitions(knowledgeBaseId)`**: Retrieves all *current* tag definitions for the knowledge base. This is crucial for detecting conflicts (e.g., trying to create a tag with an already used display name or slot) and determining if an operation is an update or a create.
*   **`const existingBySlot = new Map(...)`, `const existingByDisplayName = new Map(...)`**: Creates two `Map` objects from the `existingDefinitions`.
    *   `existingBySlot`: Maps `tagSlot` (e.g., `'tag1'`) to its corresponding definition object.
    *   `existingByDisplayName`: Maps `displayName` (e.g., `'Category'`) to its corresponding definition object.
    These maps provide efficient `O(1)` average time complexity for checking if a slot or display name is already in use, avoiding repeated database queries within the loop.
*   **`for (const defData of definitions)`**: Iterates through each `defData` (individual tag definition data) provided in the `bulkData.definitions` array.
*   **`try { ... } catch (error) { ... }`**: A `try...catch` block wraps the processing of each individual definition. If an error occurs for one definition, it's caught and added to the `errors` array, allowing the bulk operation to continue processing other definitions.
*   **`const { tagSlot, displayName, fieldType, originalDisplayName } = defData`**: Destructures properties from the current definition data. `originalDisplayName` is key for distinguishing updates.
*   **`if (!SUPPORTED_FIELD_TYPES.includes(...))`**: Validates that the `fieldType` provided in `defData` is one of the `SUPPORTED_FIELD_TYPES`. If not, an error is recorded, and the loop continues to the next definition.
*   **`const isUpdate = !!originalDisplayName`**: Determines if the current `defData` represents an update operation. If `originalDisplayName` is present (and truthy), it's an update; otherwise, it's a creation.

    *   **`if (isUpdate)` (Update Logic)**:
        *   **`const existingDef = existingByDisplayName.get(originalDisplayName!)`**: Tries to find the existing definition using its `originalDisplayName`. The `!` is a non-null assertion because `isUpdate` guarantees `originalDisplayName` exists.
        *   **`if (!existingDef)`**: If the definition to be updated isn't found, an error is recorded.
        *   **`if (displayName !== originalDisplayName && existingByDisplayName.has(displayName))`**: Checks for a conflict if the `displayName` is being changed. If the new `displayName` is different from the old one *and* it's already used by another existing definition, an error is recorded.
        *   **`await db.update(knowledgeBaseTagDefinitions).set({ displayName, fieldType, updatedAt: now, }).where(eq(knowledgeBaseTagDefinitions.id, existingDef.id))`**:
            *   `db.update(...)`: Performs an `UPDATE` query on `knowledgeBaseTagDefinitions`.
            *   `.set(...)`: Sets the new `displayName`, `fieldType`, and `updatedAt`.
            *   `.where(eq(knowledgeBaseTagDefinitions.id, existingDef.id))`: Updates only the definition matching the `id` of the `existingDef`.
        *   `updated.push(...)`: Adds the details of the successfully updated definition to the `updated` array.
*   **`else` (Create Logic)**:
    *   **`let finalTagSlot = tagSlot`**: Initializes `finalTagSlot` with the `tagSlot` provided in `defData` (if any).
    *   **`if (!finalTagSlot || existingBySlot.has(finalTagSlot))`**: Checks if a `tagSlot` was *not* provided or if the provided `tagSlot` is *already taken* by an existing definition (checked against `existingBySlot`).
        *   **`const nextSlot = await getNextAvailableSlot(knowledgeBaseId, fieldType, existingBySlot)`**: If a slot needs to be found, calls `getNextAvailableSlot`. Crucially, it passes `existingBySlot` as an optimization, allowing `getNextAvailableSlot` to consider definitions already known in this batch without a new DB query.
        *   **`if (!nextSlot)`**: If `getNextAvailableSlot` returns `null` (no slots available), an error is recorded.
        *   **`finalTagSlot = nextSlot`**: Assigns the found `nextSlot`.
    *   **`if (existingBySlot.has(finalTagSlot))`**: Even after trying to find a `nextSlot`, this re-checks for a conflict. This is important to catch cases where a `finalTagSlot` might have been selected or proposed that somehow became occupied *within the same bulk batch* (e.g., if two create operations request the same `tagSlot` or if `getNextAvailableSlot` returned a slot that was then chosen by a prior entry in the `definitions` array for a different `fieldType`).
    *   **`if (existingByDisplayName.has(displayName))`**: Checks if the proposed `displayName` is already in use.
    *   **`const id = randomUUID()`**: Generates a new UUID for the new definition.
    *   **`const now = new Date()`**: Gets the current timestamp.
    *   **`const newDefinition = { ... }`**: Creates the object representing the new tag definition. `tagSlot` is cast for type compatibility.
    *   **`await db.insert(knowledgeBaseTagDefinitions).values(newDefinition)`**: Inserts the new definition into the database.
    *   **`existingBySlot.set(finalTagSlot, newDefinition)`**: **Crucial for bulk processing**: Adds the newly created definition to the `existingBySlot` map. This ensures that subsequent definitions in the *same bulk batch* can detect conflicts with this newly created definition without requiring another database read.
    *   **`existingByDisplayName.set(displayName, newDefinition)`**: Similarly, adds the new definition to `existingByDisplayName`.
    *   `created.push(newDefinition as DocumentTagDefinition)`: Adds the newly created definition to the `created` array.
*   **`logger.info(...)`**: Logs a summary of the bulk operation's outcome.
*   **`return { created, updated, errors }`**: Returns the collected results.

### 8. `getTagDefinitionById` Function

```typescript
/**
 * Get a single tag definition by ID
 */
export async function getTagDefinitionById(
  tagDefinitionId: string
): Promise<DocumentTagDefinition | null> {
  const result = await db
    .select({
      id: knowledgeBaseTagDefinitions.id,
      knowledgeBaseId: knowledgeBaseTagDefinitions.knowledgeBaseId,
      tagSlot: knowledgeBaseTagDefinitions.tagSlot,
      displayName: knowledgeBaseTagDefinitions.displayName,
      fieldType: knowledgeBaseTagDefinitions.fieldType,
      createdAt: knowledgeBaseTagDefinitions.createdAt,
      updatedAt: knowledgeBaseTagDefinitions.updatedAt,
    })
    .from(knowledgeBaseTagDefinitions)
    .where(eq(knowledgeBaseTagDefinitions.id, tagDefinitionId))
    .limit(1)

  if (result.length === 0) {
    return null
  }

  const def = result[0]
  return {
    ...def,
    tagSlot: def.tagSlot as string,
  }
}
```

*   **Purpose**: Retrieves a single tag definition using its unique ID.
*   **`export async function getTagDefinitionById(...)`**: Asynchronous function.
    *   `tagDefinitionId: string`: The ID of the tag definition to retrieve.
    *   `Promise<DocumentTagDefinition | null>`: Returns the definition object or `null` if not found.
*   **`const result = await db.select({ ... }).from(...).where(...).limit(1)`**:
    *   Performs a `SELECT` query, retrieving all columns for a tag definition.
    *   `.where(eq(knowledgeBaseTagDefinitions.id, tagDefinitionId))`: Filters by the given `tagDefinitionId`.
    *   `.limit(1)`: Ensures that only at most one record is returned, as IDs are unique.
*   **`if (result.length === 0) { return null }`**: If no record was found, returns `null`.
*   **`const def = result[0]`**: If a record was found, it's the first (and only) element in the `result` array.
*   **`return { ...def, tagSlot: def.tagSlot as string, }`**: Returns the definition, again casting `tagSlot` to `string`.

### 9. `updateTagValuesInDocumentsAndChunks` Function

```typescript
/**
 * Update tags on all documents and chunks when a tag value is changed
 */
export async function updateTagValuesInDocumentsAndChunks(
  knowledgeBaseId: string,
  tagSlot: string,
  oldValue: string | null,
  newValue: string | null,
  requestId: string
): Promise<{ documentsUpdated: number; chunksUpdated: number }> {
  validateTagSlot(tagSlot)

  let documentsUpdated = 0
  let chunksUpdated = 0

  await db.transaction(async (tx) => {
    if (oldValue) {
      await tx
        .update(document)
        .set({
          [tagSlot]: newValue,
        })
        .where(
          and(
            eq(document.knowledgeBaseId, knowledgeBaseId),
            eq(sql.raw(`${document}.${tagSlot}`), oldValue)
          )
        )
      documentsUpdated = 1
    }

    if (oldValue) {
      await tx
        .update(embedding)
        .set({
          [tagSlot]: newValue,
        })
        .where(
          and(
            eq(embedding.knowledgeBaseId, knowledgeBaseId),
            eq(sql.raw(`${embedding}.${tagSlot}`), oldValue)
          )
        )
      chunksUpdated = 1
    }
  })

  logger.info(
    `[${requestId}] Updated tag values: ${documentsUpdated} documents, ${chunksUpdated} chunks`
  )

  return { documentsUpdated, chunksUpdated }
}
```

*   **Purpose**: This function is called when a *specific tag value* (e.g., changing all instances of "Category: Science" to "Category: STEM") needs to be updated across all documents and embeddings within a knowledge base.
*   **`export async function updateTagValuesInDocumentsAndChunks(...)`**: Asynchronous function.
    *   `knowledgeBaseId: string`: The ID of the knowledge base.
    *   `tagSlot: string`: The specific tag column (e.g., `'tag1'`) where the value is stored.
    *   `oldValue: string | null`: The old value to find and replace.
    *   `newValue: string | null`: The new value to set.
    *   `requestId: string`: For logging.
    *   `Promise<{ documentsUpdated: number; chunksUpdated: number }>`: Returns counts of updated documents and chunks.
*   **`validateTagSlot(tagSlot)`**: Calls the helper function to ensure `tagSlot` is a valid, predefined column name. This is critical for security as `tagSlot` is used dynamically in raw SQL.
*   **`let documentsUpdated = 0`, `let chunksUpdated = 0`**: Initializes counters for updated records.
*   **`await db.transaction(async (tx) => { ... })`**:
    *   This wraps the update operations in a **database transaction**. This means that *both* the document update and the embedding update must succeed for the entire operation to commit. If one fails, the other is automatically rolled back, ensuring data consistency. `tx` is the transaction-specific database client.
*   **`if (oldValue)`**: The update logic only proceeds if an `oldValue` is provided. If `oldValue` is `null`, it implies we are changing `NULL` to a `newValue` (or vice-versa), which might be handled differently or not supported by this specific function.
*   **Document Update**:
    *   **`await tx.update(document).set({ [tagSlot]: newValue, }).where( and( eq(document.knowledgeBaseId, knowledgeBaseId), eq(sql.raw(`${document}.${tagSlot}`), oldValue) ) )`**:
        *   `tx.update(document)`: Initiates an `UPDATE` on the `document` table within the transaction.
        *   `.set({ [tagSlot]: newValue, })`: Dynamically sets the value of the column specified by `tagSlot` to `newValue`. `[tagSlot]` is JavaScript's computed property name syntax.
        *   `.where(and(...))`: Filters documents to update:
            *   `eq(document.knowledgeBaseId, knowledgeBaseId)`: Ensures updates are only for the specified knowledge base.
            *   `eq(sql.raw(`${document}.${tagSlot}`), oldValue)`: **This is important.** It compares the value in the *dynamically determined* `tagSlot` column (e.g., `document.tag1`) to the `oldValue`. `sql.raw()` is necessary because Drizzle doesn't statically know `tagSlot` as a column name; it's injected dynamically.
        *   `documentsUpdated = 1`: *Note: This currently assumes only one document is updated. In a real-world scenario, you'd typically retrieve the count of affected rows from the Drizzle update operation (`.returning()`) to get the actual number.*
*   **Embedding Update**: The logic is identical to the document update, but it applies to the `embedding` table.
*   **`logger.info(...)`**: Logs the outcome of the update operation.
*   **`return { documentsUpdated, chunksUpdated }`**: Returns the counts.

### 10. `cleanupUnusedTagDefinitions` Function

```typescript
/**
 * Cleanup unused tag definitions for a knowledge base
 */
export async function cleanupUnusedTagDefinitions(
  knowledgeBaseId: string,
  requestId: string
): Promise<number> {
  const definitions = await getDocumentTagDefinitions(knowledgeBaseId)
  let cleanedUp = 0

  for (const def of definitions) {
    const tagSlot = def.tagSlot
    validateTagSlot(tagSlot)

    const docCountResult = await db
      .select({ count: sql<number>`count(*)` })
      .from(document)
      .where(
        and(
          eq(document.knowledgeBaseId, knowledgeBaseId),
          isNull(document.deletedAt),
          sql`${sql.raw(tagSlot)} IS NOT NULL`
        )
      )

    const chunkCountResult = await db
      .select({ count: sql<number>`count(*)` })
      .from(embedding)
      .where(
        and(eq(embedding.knowledgeBaseId, knowledgeBaseId), sql`${sql.raw(tagSlot)} IS NOT NULL`)
      )

    const docCount = Number(docCountResult[0]?.count || 0)
    const chunkCount = Number(chunkCountResult[0]?.count || 0)

    if (docCount === 0 && chunkCount === 0) {
      await db.delete(knowledgeBaseTagDefinitions).where(eq(knowledgeBaseTagDefinitions.id, def.id))

      cleanedUp++
      logger.info(
        `[${requestId}] Cleaned up unused tag definition: ${def.displayName} (${def.tagSlot})`
      )
    }
  }

  logger.info(`[${requestId}] Cleanup completed: ${cleanedUp} unused tag definitions removed`)
  return cleanedUp
}
```

*   **Purpose**: This function identifies and removes tag definitions that are no longer actively used (i.e., no document or embedding currently has a non-NULL value for that tag slot).
*   **`export async function cleanupUnusedTagDefinitions(...)`**: Asynchronous function.
    *   `knowledgeBaseId: string`: The ID of the knowledge base.
    *   `requestId: string`: For logging.
    *   `Promise<number>`: Returns the count of definitions that were cleaned up.
*   **`const definitions = await getDocumentTagDefinitions(knowledgeBaseId)`**: Fetches all tag definitions for the knowledge base.
*   **`let cleanedUp = 0`**: Initializes a counter for removed definitions.
*   **`for (const def of definitions)`**: Iterates through each tag definition.
    *   **`const tagSlot = def.tagSlot`**: Extracts the `tagSlot` from the current definition.
    *   **`validateTagSlot(tagSlot)`**: Ensures `tagSlot` is valid.
    *   **`const docCountResult = await db.select({ count: sql<number>`count(*)` }).from(document).where(and(..., isNull(document.deletedAt), sql`${sql.raw(tagSlot)} IS NOT NULL`))`**:
        *   Performs a `SELECT count(*)` query on the `document` table.
        *   `eq(document.knowledgeBaseId, knowledgeBaseId)`: Filters by knowledge base.
        *   `isNull(document.deletedAt)`: Excludes soft-deleted documents.
        *   `sql`${sql.raw(tagSlot)} IS NOT NULL`**: Checks if the dynamic `tagSlot` column has a non-`NULL` value. This means a document is actively using this tag.
    *   **`const chunkCountResult = await db.select({ count: sql<number>`count(*)` }).from(embedding).where(and(eq(embedding.knowledgeBaseId, knowledgeBaseId), sql`${sql.raw(tagSlot)} IS NOT NULL`))`**:
        *   Similar `SELECT count(*)` query, but on the `embedding` table.
    *   **`const docCount = Number(docCountResult[0]?.count || 0)`**: Extracts the document count, defaulting to 0 if no result or count is found.
    *   **`const chunkCount = Number(chunkCountResult[0]?.count || 0)`**: Extracts the chunk count.
    *   **`if (docCount === 0 && chunkCount === 0)`**: If both counts are zero, the tag definition is considered unused.
        *   **`await db.delete(knowledgeBaseTagDefinitions).where(eq(knowledgeBaseTagDefinitions.id, def.id))`**: Deletes the unused tag definition from the `knowledgeBaseTagDefinitions` table.
        *   `cleanedUp++`: Increments the counter.
        *   `logger.info(...)`: Logs the cleanup of this specific definition.
*   **`logger.info(...)`**: Logs a summary of the total cleanup.
*   **`return cleanedUp`**: Returns the total number of cleaned-up definitions.

### 11. `deleteAllTagDefinitions` Function

```typescript
/**
 * Delete all tag definitions for a knowledge base
 */
export async function deleteAllTagDefinitions(
  knowledgeBaseId: string,
  requestId: string
): Promise<number> {
  const result = await db
    .delete(knowledgeBaseTagDefinitions)
    .where(eq(knowledgeBaseTagDefinitions.knowledgeBaseId, knowledgeBaseId))
    .returning({ id: knowledgeBaseTagDefinitions.id })

  const deletedCount = result.length
  logger.info(`[${requestId}] Deleted ${deletedCount} tag definitions for KB: ${knowledgeBaseId}`)

  return deletedCount
}
```

*   **Purpose**: Deletes all tag definitions belonging to a specific `knowledgeBaseId`.
*   **`export async function deleteAllTagDefinitions(...)`**: Asynchronous function.
    *   `knowledgeBaseId: string`: The ID of the knowledge base.
    *   `requestId: string`: For logging.
    *   `Promise<number>`: Returns the count of deleted definitions.
*   **`const result = await db.delete(knowledgeBaseTagDefinitions).where(eq(knowledgeBaseTagDefinitions.knowledgeBaseId, knowledgeBaseId)).returning({ id: knowledgeBaseTagDefinitions.id })`**:
    *   `db.delete(knowledgeBaseTagDefinitions)`: Initiates a `DELETE` query on the `knowledgeBaseTagDefinitions` table.
    *   `.where(eq(knowledgeBaseTagDefinitions.knowledgeBaseId, knowledgeBaseId))`: Deletes only definitions associated with the given `knowledgeBaseId`.
    *   `.returning({ id: knowledgeBaseTagDefinitions.id })`: Returns the `id` of each deleted record. This is how Drizzle can provide a count of deleted rows.
*   **`const deletedCount = result.length`**: The number of deleted definitions is simply the length of the `result` array.
*   **`logger.info(...)`**: Logs the total count of deleted definitions.
*   **`return deletedCount`**: Returns the count.
*   **Important Note**: This function *only* deletes the tag *definitions*. It does **not** clear the tag values from the `document` or `embedding` tables. If that's desired, `deleteTagDefinition` (for a single tag) or a custom bulk cleanup would be needed.

### 12. `deleteTagDefinition` Function

```typescript
/**
 * Delete a tag definition with comprehensive cleanup
 * This removes the definition and clears all document/chunk references
 */
export async function deleteTagDefinition(
  tagDefinitionId: string,
  requestId: string
): Promise<{ tagSlot: string; displayName: string }> {
  const tagDef = await db
    .select({
      id: knowledgeBaseTagDefinitions.id,
      knowledgeBaseId: knowledgeBaseTagDefinitions.knowledgeBaseId,
      tagSlot: knowledgeBaseTagDefinitions.tagSlot,
      displayName: knowledgeBaseTagDefinitions.displayName,
    })
    .from(knowledgeBaseTagDefinitions)
    .where(eq(knowledgeBaseTagDefinitions.id, tagDefinitionId))
    .limit(1)

  if (tagDef.length === 0) {
    throw new Error(`Tag definition ${tagDefinitionId} not found`)
  }

  const definition = tagDef[0]
  const knowledgeBaseId = definition.knowledgeBaseId
  const tagSlot = definition.tagSlot as string

  validateTagSlot(tagSlot)

  await db.transaction(async (tx) => {
    await tx
      .update(document)
      .set({ [tagSlot]: null })
      .where(
        and(eq(document.knowledgeBaseId, knowledgeBaseId), isNotNull(sql`${sql.raw(tagSlot)}`))
      )

    await tx
      .update(embedding)
      .set({ [tagSlot]: null })
      .where(
        and(eq(embedding.knowledgeBaseId, knowledgeBaseId), isNotNull(sql`${sql.raw(tagSlot)}`))
      )

    await tx
      .delete(knowledgeBaseTagDefinitions)
      .where(eq(knowledgeBaseTagDefinitions.id, tagDefinitionId))
  })

  logger.info(
    `[${requestId}] Deleted tag definition with cleanup: ${definition.displayName} (${tagSlot})`
  )

  return {
    tagSlot,
    displayName: definition.displayName,
  }
}
```

*   **Purpose**: This function deletes a single tag definition and, crucially, performs a **comprehensive cleanup**: it clears (sets to `NULL`) the corresponding tag values from all documents and embeddings within that knowledge base before deleting the definition itself.
*   **`export async function deleteTagDefinition(...)`**: Asynchronous function.
    *   `tagDefinitionId: string`: The ID of the tag definition to delete.
    *   `requestId: string`: For logging.
    *   `Promise<{ tagSlot: string; displayName: string }>`: Returns the `tagSlot` and `displayName` of the deleted definition.
*   **`const tagDef = await db.select({ ... }).from(...).where(...).limit(1)`**: Retrieves the tag definition to be deleted, selecting its `id`, `knowledgeBaseId`, `tagSlot`, and `displayName`.
*   **`if (tagDef.length === 0) { throw new Error(...) }`**: If the definition is not found, an error is thrown.
*   **`const definition = tagDef[0]`, `const knowledgeBaseId = definition.knowledgeBaseId`, `const tagSlot = definition.tagSlot as string`**: Extracts necessary information from the retrieved definition.
*   **`validateTagSlot(tagSlot)`**: Validates the `tagSlot` for security.
*   **`await db.transaction(async (tx) => { ... })`**: Wraps all operations in a database transaction to ensure atomicity. If any step (clearing documents, clearing embeddings, deleting definition) fails, the entire transaction is rolled back.
*   **Document Cleanup**:
    *   **`await tx.update(document).set({ [tagSlot]: null }).where(and(eq(document.knowledgeBaseId, knowledgeBaseId), isNotNull(sql`${sql.raw(tagSlot)}`)))`**:
        *   `tx.update(document).set({ [tagSlot]: null })`: Sets the value of the dynamic `tagSlot` column to `NULL` for relevant documents.
        *   `.where(and(eq(document.knowledgeBaseId, knowledgeBaseId), isNotNull(sql`${sql.raw(tagSlot)}`)))`: Filters to documents in the specific knowledge base that *currently have a non-NULL value* in that `tagSlot` column. This efficiently targets only documents that need their tag cleared.
*   **Embedding Cleanup**: The logic is identical to document cleanup, but applied to the `embedding` table.
*   **Definition Deletion**:
    *   **`await tx.delete(knowledgeBaseTagDefinitions).where(eq(knowledgeBaseTagDefinitions.id, tagDefinitionId))`**: Finally, deletes the tag definition record itself from the `knowledgeBaseTagDefinitions` table.
*   **`logger.info(...)`**: Logs the comprehensive deletion.
*   **`return { tagSlot, displayName: definition.displayName }`**: Returns details of the deleted tag.

### 13. `createTagDefinition` Function

```typescript
/**
 * Create a new tag definition
 */
export async function createTagDefinition(
  data: CreateTagDefinitionData,
  requestId: string
): Promise<TagDefinition> {
  const tagDefinitionId = randomUUID()
  const now = new Date()

  const newDefinition = {
    id: tagDefinitionId,
    knowledgeBaseId: data.knowledgeBaseId,
    tagSlot: data.tagSlot as (typeof TAG_SLOT_CONFIG.text.slots)[number],
    displayName: data.displayName,
    fieldType: data.fieldType,
    createdAt: now,
    updatedAt: now,
  }

  await db.insert(knowledgeBaseTagDefinitions).values(newDefinition)

  logger.info(
    `[${requestId}] Created tag definition: ${data.displayName} -> ${data.tagSlot} in KB ${data.knowledgeBaseId}`
  )

  return {
    id: tagDefinitionId,
    tagSlot: data.tagSlot,
    displayName: data.displayName,
    fieldType: data.fieldType,
    createdAt: now,
    updatedAt: now,
  }
}
```

*   **Purpose**: Creates a single new tag definition.
*   **`export async function createTagDefinition(...)`**: Asynchronous function.
    *   `data: CreateTagDefinitionData`: The data for the new tag definition (including `knowledgeBaseId`, `tagSlot`, `displayName`, `fieldType`).
    *   `requestId: string`: For logging.
    *   `Promise<TagDefinition>`: Returns the newly created tag definition object.
*   **`const tagDefinitionId = randomUUID()`**: Generates a unique ID for the new definition.
*   **`const now = new Date()`**: Gets the current timestamp.
*   **`const newDefinition = { ... }`**: Constructs the `newDefinition` object with all required properties. `tagSlot` is cast to match the expected type from `TAG_SLOT_CONFIG`.
*   **`await db.insert(knowledgeBaseTagDefinitions).values(newDefinition)`**: Inserts the new definition into the `knowledgeBaseTagDefinitions` table.
*   **`logger.info(...)`**: Logs the creation.
*   **`return { ... }`**: Returns the newly created `TagDefinition` object.

### 14. `updateTagDefinition` Function

```typescript
/**
 * Update an existing tag definition
 */
export async function updateTagDefinition(
  tagDefinitionId: string,
  data: UpdateTagDefinitionData,
  requestId: string
): Promise<TagDefinition> {
  const now = new Date()

  const updateData: {
    updatedAt: Date
    displayName?: string
    fieldType?: string
  } = {
    updatedAt: now,
  }

  if (data.displayName !== undefined) {
    updateData.displayName = data.displayName
  }

  if (data.fieldType !== undefined) {
    updateData.fieldType = data.fieldType
  }

  const updatedRows = await db
    .update(knowledgeBaseTagDefinitions)
    .set(updateData)
    .where(eq(knowledgeBaseTagDefinitions.id, tagDefinitionId))
    .returning({
      id: knowledgeBaseTagDefinitions.id,
      tagSlot: knowledgeBaseTagDefinitions.tagSlot,
      displayName: knowledgeBaseTagDefinitions.displayName,
      fieldType: knowledgeBaseTagDefinitions.fieldType,
      createdAt: knowledgeBaseTagDefinitions.createdAt,
      updatedAt: knowledgeBaseTagDefinitions.updatedAt,
    })

  if (updatedRows.length === 0) {
    throw new Error(`Tag definition ${tagDefinitionId} not found`)
  }

  const updated = updatedRows[0]
  logger.info(`[${requestId}] Updated tag definition: ${tagDefinitionId}`)

  return {
    ...updated,
    tagSlot: updated.tagSlot as string,
  }
}
```

*   **Purpose**: Updates an existing tag definition's `displayName` and/or `fieldType`.
*   **`export async function updateTagDefinition(...)`**: Asynchronous function.
    *   `tagDefinitionId: string`: The ID of the definition to update.
    *   `data: UpdateTagDefinitionData`: An object potentially containing new `displayName` and `fieldType`.
    *   `requestId: string`: For logging.
    *   `Promise<TagDefinition>`: Returns the updated tag definition object.
*   **`const now = new Date()`**: Gets the current timestamp.
*   **`const updateData: { ... } = { updatedAt: now }`**: Initializes an object `updateData` that will hold the fields to be updated. `updatedAt` is always updated.
*   **`if (data.displayName !== undefined) { ... }`, `if (data.fieldType !== undefined) { ... }`**: Conditionally adds `displayName` and `fieldType` to `updateData` *only if they are provided* in the `data` object. This allows for partial updates.
*   **`const updatedRows = await db.update(knowledgeBaseTagDefinitions).set(updateData).where(eq(knowledgeBaseTagDefinitions.id, tagDefinitionId)).returning({ ... })`**:
    *   `db.update(knowledgeBaseTagDefinitions).set(updateData)`: Performs an `UPDATE` query with the `updateData`.
    *   `.where(eq(knowledgeBaseTagDefinitions.id, tagDefinitionId))`: Filters to the specific definition.
    *   `.returning({ ... })`: Returns the full updated record (all specified columns). This is efficient as it avoids a subsequent `SELECT` query.
*   **`if (updatedRows.length === 0) { throw new Error(...) }`**: If no rows were updated, it means the `tagDefinitionId` was not found, so an error is thrown.
*   **`const updated = updatedRows[0]`**: Gets the first (and only) updated record.
*   **`logger.info(...)`**: Logs the update.
*   **`return { ...updated, tagSlot: updated.tagSlot as string, }`**: Returns the updated definition, casting `tagSlot` to `string`.

### 15. `getTagUsage` Function

```typescript
/**
 * Get tag usage with detailed document information (original format)
 */
export async function getTagUsage(
  knowledgeBaseId: string,
  requestId = 'api'
): Promise<
  Array<{
    tagName: string
    tagSlot: string
    documentCount: number
    documents: Array<{ id: string; name: string; tagValue: string }>
  }>
> {
  const definitions = await getDocumentTagDefinitions(knowledgeBaseId)
  const usage = []

  for (const def of definitions) {
    const tagSlot = def.tagSlot
    validateTagSlot(tagSlot)

    const documentsWithTag = await db
      .select({
        id: document.id,
        filename: document.filename,
        tagValue: sql<string>`${sql.raw(tagSlot)}`,
      })
      .from(document)
      .where(
        and(
          eq(document.knowledgeBaseId, knowledgeBaseId),
          isNull(document.deletedAt),
          isNotNull(sql`${sql.raw(tagSlot)}`)
        )
      )

    usage.push({
      tagName: def.displayName,
      tagSlot: def.tagSlot,
      documentCount: documentsWithTag.length,
      documents: documentsWithTag.map((doc) => ({
        id: doc.id,
        name: doc.filename,
        tagValue: doc.tagValue || '',
      })),
    })
  }

  logger.info(`[${requestId}] Retrieved detailed tag usage for ${usage.length} definitions`)

  return usage
}
```

*   **Purpose**: Provides detailed information about which tags are used, including a list of specific documents that have each tag and their respective tag values.
*   **`export async function getTagUsage(...)`**: Asynchronous function.
    *   `knowledgeBaseId: string`: The ID of the knowledge base.
    *   `requestId = 'api'`: An optional request ID for logging, defaulting to 'api'.
    *   `Promise<Array<{ ... }>>`: Returns an array of objects, each describing a tag's usage with document details.
*   **`const definitions = await getDocumentTagDefinitions(knowledgeBaseId)`**: Retrieves all tag definitions for the knowledge base.
*   **`const usage = []`**: Initializes an empty array to store the usage data.
*   **`for (const def of definitions)`**: Iterates through each tag definition.
    *   **`const tagSlot = def.tagSlot`**: Extracts the `tagSlot`.
    *   **`validateTagSlot(tagSlot)`**: Validates the `tagSlot`.
    *   **`const documentsWithTag = await db.select({ id: document.id, filename: document.filename, tagValue: sql<string>`${sql.raw(tagSlot)}`, }).from(document).where(and(eq(document.knowledgeBaseId, knowledgeBaseId), isNull(document.deletedAt), isNotNull(sql`${sql.raw(tagSlot)}`)))`**:
        *   Performs a `SELECT` query on the `document` table.
        *   `id: document.id`, `filename: document.filename`: Selects the document's ID and filename.
        *   `tagValue: sql<string>`${sql.raw(tagSlot)}``: **Dynamically selects the value of the `tagSlot` column.** The `sql<string>` assertion helps TypeScript understand the expected type of the raw SQL output.
        *   `.where(and(eq(document.knowledgeBaseId, knowledgeBaseId), isNull(document.deletedAt), isNotNull(sql`${sql.raw(tagSlot)}`)))`: Filters to non-deleted documents in the knowledge base that have a non-NULL value in the current `tagSlot`.
    *   **`usage.push({ ... })`**: Adds an object to the `usage` array for the current tag definition, containing:
        *   `tagName`: The display name of the tag.
        *   `tagSlot`: The actual slot (e.g., `'tag1'`).
        *   `documentCount`: The number of documents found.
        *   `documents`: An array of simplified document objects (id, name, tagValue).
*   **`logger.info(...)`**: Logs the retrieval of usage data.
*   **`return usage`**: Returns the collected usage data.

### 16. `getTagUsageStats` Function

```typescript
/**
 * Get tag usage statistics
 */
export async function getTagUsageStats(
  knowledgeBaseId: string,
  requestId: string
): Promise<
  Array<{
    tagSlot: string
    displayName: string
    fieldType: string
    documentCount: number
    chunkCount: number
  }>
> {
  const definitions = await getDocumentTagDefinitions(knowledgeBaseId)
  const stats = []

  for (const def of definitions) {
    const tagSlot = def.tagSlot
    validateTagSlot(tagSlot)

    const docCountResult = await db
      .select({ count: sql<number>`count(*)` })
      .from(document)
      .where(
        and(
          eq(document.knowledgeBaseId, knowledgeBaseId),
          isNull(document.deletedAt),
          sql`${sql.raw(tagSlot)} IS NOT NULL`
        )
      )

    const chunkCountResult = await db
      .select({ count: sql<number>`count(*)` })
      .from(embedding)
      .where(
        and(eq(embedding.knowledgeBaseId, knowledgeBaseId), sql`${sql.raw(tagSlot)} IS NOT NULL`)
      )

    stats.push({
      tagSlot: def.tagSlot,
      displayName: def.displayName,
      fieldType: def.fieldType,
      documentCount: Number(docCountResult[0]?.count || 0),
      chunkCount: Number(chunkCountResult[0]?.count || 0),
    })
  }

  logger.info(`[${requestId}] Retrieved tag usage stats for ${stats.length} definitions`)

  return stats
}
```

*   **Purpose**: Provides aggregated usage statistics for each tag definition, showing how many documents and embeddings currently have a value for that tag. This is less detailed than `getTagUsage` but provides quick counts.
*   **`export async function getTagUsageStats(...)`**: Asynchronous function.
    *   `knowledgeBaseId: string`: The ID of the knowledge base.
    *   `requestId: string`: For logging.
    *   `Promise<Array<{ ... }>>`: Returns an array of objects, each with tag details and counts.
*   **`const definitions = await getDocumentTagDefinitions(knowledgeBaseId)`**: Retrieves all tag definitions.
*   **`const stats = []`**: Initializes an empty array for statistics.
*   **`for (const def of definitions)`**: Iterates through each tag definition.
    *   **`const tagSlot = def.tagSlot`**: Extracts the `tagSlot`.
    *   **`validateTagSlot(tagSlot)`**: Validates the `tagSlot`.
    *   **`const docCountResult = await db.select({ count: sql<number>`count(*)` }).from(document).where(and(..., isNull(document.deletedAt), sql`${sql.raw(tagSlot)} IS NOT NULL`))`**:
        *   Performs a `SELECT count(*)` query on `document`, filtered by knowledge base, non-deleted status, and where the dynamic `tagSlot` column is `NOT NULL`.
    *   **`const chunkCountResult = await db.select({ count: sql<number>`count(*)` }).from(embedding).where(and(eq(embedding.knowledgeBaseId, knowledgeBaseId), sql`${sql.raw(tagSlot)} IS NOT NULL`))`**:
        *   Similar `SELECT count(*)` query, but on `embedding`.
    *   **`stats.push({ ... })`**: Adds an object to the `stats` array for the current definition, including:
        *   `tagSlot`, `displayName`, `fieldType`: Details from the definition.
        *   `documentCount`: The number of documents using this tag.
        *   `chunkCount`: The number of embeddings using this tag.
*   **`logger.info(...)`**: Logs the retrieval of usage statistics.
*   **`return stats`**: Returns the collected statistics.

---

This detailed breakdown covers the purpose, complex logic simplification, and line-by-line explanation of the provided TypeScript code, highlighting key concepts like Drizzle ORM usage, transactions, dynamic SQL, and type safety with assertion functions.