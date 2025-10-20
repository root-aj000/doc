This TypeScript file, `document.service.ts`, acts as the central **Document Service** for a "Knowledge Base" system. Its primary role is to manage the lifecycle of documents, from their initial creation and structured tagging to their complex background processing (which involves parsing, chunking, and generating embeddings for AI applications), and finally, to providing methods for retrieving, updating, and deleting them.

Essentially, this file orchestrates how raw files become queryable "knowledge" for an AI.

### Core Responsibilities

1.  **Document Creation & Tagging:** Handles the initial upload of documents and manages dynamic tags associated with them, creating tag definitions as needed.
2.  **Intelligent Document Processing:** Takes raw documents, breaks them into smaller, meaningful chunks, and generates vector embeddings for each chunk. This is crucial for Retrieval Augmented Generation (RAG) and semantic search. It intelligently dispatches this heavy lifting to available background processing systems (Trigger.dev, Redis queue, or in-memory concurrency).
3.  **Data Persistence:** Interacts with a database (using Drizzle ORM) to store document metadata, chunk information, embedding vectors, and tag definitions.
4.  **Lifecycle Management:** Provides functions to fetch, update, enable/disable, retry, and soft-delete documents.
5.  **Concurrency & Resilience:** Implements mechanisms for concurrent processing, batching, timeouts, and fallback strategies to ensure robust handling of documents, especially large ones.

---

### Imports and Their Purpose

Let's break down each import statement and what it brings to this file:

*   `import crypto, { randomUUID } from 'crypto'`:
    *   `crypto`: Node.js built-in module for cryptographic functionalities. Here, it's used for `createHash` to generate unique hashes for document chunks and `randomUUID` to create unique identifiers (UUIDs) for documents and embeddings.
    *   `randomUUID`: Specifically imported for generating universally unique identifiers.
*   `import { db } from '@sim/db'`:
    *   Imports the database client instance, likely configured with Drizzle ORM, allowing interactions with the PostgreSQL (or other) database.
*   `import { document, embedding, knowledgeBaseTagDefinitions } from '@sim/db/schema'`:
    *   Imports the Drizzle ORM schema definitions for three key tables:
        *   `document`: Stores metadata about each uploaded document (e.g., filename, URL, processing status, tags).
        *   `embedding`: Stores the individual chunks of documents along with their generated vector embeddings and associated metadata.
        *   `knowledgeBaseTagDefinitions`: Stores definitions for the custom tags used within a knowledge base (e.g., `tag1` maps to "Author").
*   `import { tasks } from '@trigger.dev/sdk'`:
    *   Imports the client for Trigger.dev, a platform for running background jobs and workflows. This allows the service to offload heavy document processing to external workers.
*   `import { and, asc, desc, eq, inArray, isNull, sql } from 'drizzle-orm'`:
    *   Imports various utility functions from Drizzle ORM to build database queries:
        *   `and`: Combines multiple conditions with a logical AND.
        *   `asc`, `desc`: Specifies ascending or descending order for sorting results.
        *   `eq`: Checks for equality (e.g., `WHERE id = '...'`).
        *   `inArray`: Checks if a column's value is present in a given array (e.g., `WHERE id IN ('id1', 'id2')`).
        *   `isNull`: Checks if a column's value is NULL.
        *   `sql`: Allows embedding raw SQL snippets into Drizzle queries (useful for things like `COUNT(*)` or `LOWER(column) LIKE '%'`).
*   `import { generateEmbeddings } from '@/lib/embeddings/utils'`:
    *   Imports a function responsible for calling an external API (like OpenAI, Cohere, etc.) to generate vector embeddings for text.
*   `import { env } from '@/lib/env'`:
    *   Imports environment variables, which are used for configuration settings (e.g., API keys, timeouts, concurrency limits).
*   `import { getSlotsForFieldType, type TAG_SLOT_CONFIG } from '@/lib/knowledge/consts'`:
    *   Imports utilities related to how tags are managed. `getSlotsForFieldType` likely provides available slots (`tag1`, `tag2`, etc.) for different data types, and `TAG_SLOT_CONFIG` defines these slots.
*   `import { processDocument } from '@/lib/knowledge/documents/document-processor'`:
    *   Imports the core document parsing function. This function takes a file (URL, filename, mimeType) and returns structured chunks of text, ready for embedding.
*   `import { getNextAvailableSlot } from '@/lib/knowledge/tags/service'`:
    *   Imports a service function that helps dynamically assign a unique database slot (`tag1` through `tag7`) for new user-defined tags.
*   `import { createLogger } from '@/lib/logs/console/logger'`:
    *   Imports a utility to create a logger instance, used for structured logging throughout the service.
*   `import { getRedisClient } from '@/lib/redis'`:
    *   Imports a function to retrieve a Redis client instance, used for implementing a Redis-backed job queue.
*   `import type { DocumentProcessingPayload } from '@/background/knowledge-processing'`:
    *   Imports a TypeScript type definition for the data structure expected by the background document processing workflow (e.g., Trigger.dev payload).
*   `import { DocumentProcessingQueue } from './queue'`:
    *   Imports the local `DocumentProcessingQueue` class, which manages in-memory or Redis-backed processing queues.
*   `import type { DocumentSortField, SortOrder } from './types'`:
    *   Imports TypeScript type definitions for sorting options when retrieving documents.

---

### Global Constants and Configuration

These define crucial operational parameters for the service.

*   `const logger = createLogger('DocumentService')`:
    *   **Purpose:** Initializes a logger instance specifically named "DocumentService" to provide contextual logging output, making it easier to trace operations related to documents.
*   `const TIMEOUTS = {...} as const`:
    *   **Purpose:** Defines various timeout durations for different asynchronous operations, preventing processes from hanging indefinitely. `as const` ensures these values are treated as literal types, providing stronger type safety.
    *   `OVERALL_PROCESSING`: Sets the maximum allowed time (default 10 minutes) for a document to complete its entire parsing, embedding, and database insertion process. If it exceeds this, the operation is forcefully terminated.
    *   `EMBEDDINGS_API`: Sets a timeout for calls to the external embedding generation API (default 10 seconds, multiplied by 18 for some reason, perhaps allowing for multiple retries or long-running calls for very large batches).
*   `const LARGE_DOC_CONFIG = {...}`:
    *   **Purpose:** Configures how the service handles large documents to optimize performance and prevent resource exhaustion.
    *   `MAX_CHUNKS_PER_BATCH`: When inserting generated embeddings into the database, this specifies the maximum number of embedding records to insert in a single database batch (500).
    *   `MAX_EMBEDDING_BATCH`: When calling the external embedding generation API, this specifies the maximum number of text chunks to send in a single API request (500), helping to manage API rate limits and payload sizes.
    *   `MAX_FILE_SIZE`: Sets an absolute limit on the size of an individual file that can be processed (100 MB).
    *   `MAX_CHUNKS_PER_DOCUMENT`: Defines the maximum number of chunks a single document can be split into (100,000). Documents exceeding this are considered too large or complex and will fail processing.
*   `const PROCESSING_CONFIG = {...}`:
    *   **Purpose:** Defines concurrency and delay settings for **in-memory** document processing. These are default values, potentially scaled down from environment variables to be conservative for in-memory work.
    *   `maxConcurrentDocuments`: Maximum number of documents to process concurrently in memory (default 4).
    *   `batchSize`: Number of documents grouped into a single batch for processing (default 10).
    *   `delayBetweenBatches`: Pause in milliseconds between processing successive batches (default 200ms).
    *   `delayBetweenDocuments`: Pause in milliseconds between initiating processing for individual documents within a batch (default 100ms).
*   `const REDIS_PROCESSING_CONFIG = {...}`:
    *   **Purpose:** Defines concurrency and delay settings when **Redis** is used as the job queue. These are typically higher than `PROCESSING_CONFIG` because Redis queues enable distributed processing, allowing for more aggressive parallelism.
    *   `maxConcurrentDocuments`: Maximum concurrent documents allowed when using Redis (default 20).
    *   `batchSize`: Number of documents processed in a batch (default 20).
    *   `delayBetweenBatches`: Delay between batches (default 100ms).
    *   `delayBetweenDocuments`: Delay between documents within a batch (default 50ms).

---

### Helper Functions

*   `function withTimeout<T>(promise: Promise<T>, timeoutMs: number, operation = 'Operation'): Promise<T>`:
    *   **Purpose:** A generic utility function to enforce a timeout on any `Promise`. It's a "race" between your promise completing and a timer expiring.
    *   **Logic:**
        1.  `Promise.race([promise, new Promise<never>(...)])`: Creates a race condition. The first promise to settle (resolve or reject) wins.
        2.  The first participant is the `promise` you want to run.
        3.  The second participant is a `new Promise<never>` that will `reject` with a `TimeoutError` after `timeoutMs` milliseconds.
        4.  If your `promise` resolves or rejects before the `setTimeout` triggers, its result is returned. If the `setTimeout` triggers first, an error indicating a timeout is thrown.
*   `let documentQueue: DocumentProcessingQueue | null = null`:
    *   **Purpose:** Declares a variable to hold the singleton instance of `DocumentProcessingQueue`. It's initialized to `null` and will be lazily created.
*   `export function getDocumentQueue(): DocumentProcessingQueue`:
    *   **Purpose:** Provides a way to get the (singleton) `DocumentProcessingQueue` instance. This ensures only one queue object is created and used across the application.
    *   **Logic:**
        1.  Checks if `documentQueue` is already initialized (`if (!documentQueue)`).
        2.  If not, it calls `getRedisClient()` to check if Redis is available.
        3.  Based on Redis availability, it selects either `REDIS_PROCESSING_CONFIG` or `PROCESSING_CONFIG`.
        4.  It then creates a new `DocumentProcessingQueue` instance with the selected configuration (max concurrency, retry delay, max retries) and assigns it to `documentQueue`.
        5.  Returns the `documentQueue` instance.
*   `export function getProcessingConfig()`:
    *   **Purpose:** Returns the appropriate processing configuration (either Redis-based or in-memory) based on Redis availability.
    *   **Logic:**
        1.  Calls `getRedisClient()` to check for Redis.
        2.  Returns `REDIS_PROCESSING_CONFIG` if Redis is available, otherwise `PROCESSING_CONFIG`.

---

### Interface Definitions

These TypeScript interfaces define the expected shapes of data structures used throughout the service, ensuring type safety and clarity.

*   `interface DocumentData`:
    *   **Purpose:** Defines the essential information returned after a document has been initially created (before full processing).
    *   `documentId`: The unique ID of the document.
    *   `filename`: The original name of the file.
    *   `fileUrl`: The URL where the file can be accessed.
    *   `fileSize`: The size of the file in bytes.
    *   `mimeType`: The MIME type of the file (e.g., `application/pdf`, `text/plain`).
*   `interface ProcessingOptions`:
    *   **Purpose:** Specifies parameters that control how a document should be processed (e.g., how it's broken into chunks).
    *   `chunkSize`: The target maximum size (in characters or tokens) for each document chunk.
    *   `minCharactersPerChunk`: The minimum characters required for a chunk to be considered valid.
    *   `recipe`: A name indicating a specific processing strategy or pipeline.
    *   `lang`: The language of the document, which might influence parsing or embedding models.
    *   `chunkOverlap`: The number of characters that should overlap between consecutive chunks to maintain context.
*   `interface DocumentJobData`:
    *   **Purpose:** Defines the payload structure for a background processing job, containing all necessary information to process a single document. This is what's passed to Trigger.dev or the Redis queue.
    *   `knowledgeBaseId`: The ID of the knowledge base this document belongs to.
    *   `documentId`: The unique ID of the document.
    *   `docData`: An object containing `filename`, `fileUrl`, `fileSize`, `mimeType`.
    *   `processingOptions`: An object containing `chunkSize`, `minCharactersPerChunk`, `recipe`, `lang`, `chunkOverlap`.
    *   `requestId`: A unique identifier for the overall request, used for logging and tracing.
*   `interface DocumentTagData`:
    *   **Purpose:** Represents a single structured tag provided for a document, used for dynamic tag processing.
    *   `tagName`: The user-friendly name of the tag (e.g., "Author", "Category").
    *   `fieldType`: The data type of the tag value (e.g., "text", "number").
    *   `value`: The actual value of the tag (e.g., "John Doe", "Research Papers").

---

### Core Function Explanations

#### `export async function processDocumentTags(...)`

*   **Purpose:** This function is responsible for processing structured document tags. It ensures that custom tags provided by the user (like "Author") are mapped to predefined database "slots" (`tag1`, `tag2`, etc.) and that these mappings are stored as "tag definitions". If a new tag is encountered, it creates a new definition and assigns it an available slot.
*   **Simplified Logic:**
    1.  Initialize an object (`result`) to hold the final `tag1` to `tag7` values, initially all `null`.
    2.  If no tags are provided, return early.
    3.  Fetch all existing tag definitions for the `knowledgeBaseId` from the database. This includes knowing which `displayName` maps to which `tagSlot`.
    4.  Iterate through each `tag` in the `tagData` array:
        *   If the tag's `tagName` already has a definition (e.g., "Author" is already mapped to `tag1`), use its existing slot.
        *   If it's a *new* `tagName`, find the `next available slot` (e.g., `tag1` if it's free, then `tag2`, etc.).
        *   If a new slot is found, create a `new tag definition` in the `knowledgeBaseTagDefinitions` table, mapping the `tagName` to the `tagSlot`.
        *   Assign the `tag.value` to the `targetSlot` (e.g., `result['tag1'] = 'John Doe'`).
    5.  Return the `result` object containing the `tag1` through `tag7` values.
*   **Line-by-Line Deep Dive:**
    *   `const result: Record<string, string | null> = {}`: Initializes an empty object to store the final resolved tag values, where keys will be `tag1`...`tag7` and values will be strings or `null`.
    *   `const textSlots = getSlotsForFieldType('text')`: Retrieves an array of all available tag slot names (e.g., `['tag1', 'tag2', 'tag3']`) that are designated for "text" field types from `TAG_SLOT_CONFIG`.
    *   `textSlots.forEach((slot) => { result[slot] = null })`: Pre-populates the `result` object with all `textSlots` set to `null`. This ensures that if a tag isn't explicitly provided, its slot defaults to `null`.
    *   `if (!Array.isArray(tagData) || tagData.length === 0) { return result }`: Handles edge case where no tags are provided or `tagData` is not an array, returning the `null`-filled `result`.
    *   `try { ... } catch (error) { ... }`: Standard error handling block.
    *   `const existingDefinitions = await db.select().from(knowledgeBaseTagDefinitions).where(eq(knowledgeBaseTagDefinitions.knowledgeBaseId, knowledgeBaseId))`: Fetches all existing tag definitions for the given `knowledgeBaseId` from the `knowledgeBaseTagDefinitions` table.
    *   `const existingByName = new Map(...)` and `const existingBySlot = new Map(...)`: Creates two `Map` objects for efficient lookups: one to quickly find a tag definition by its `displayName` (e.g., "Author") and another by its `tagSlot` (e.g., `tag1`).
    *   `for (const tag of tagData) { ... }`: Loops through each tag object provided in the `tagData` array.
    *   `if (!tag.tagName?.trim() || !tag.value?.trim()) continue`: Skips any tag that has an empty or missing `tagName` or `value`.
    *   `const tagName = tag.tagName.trim()`, `const fieldType = tag.fieldType`, `const value = tag.value.trim()`: Extracts and trims the tag's name, type, and value.
    *   `let targetSlot: string | null = null`: Declares a variable to hold the database slot (`tag1` to `tag7`) that this tag will occupy.
    *   `const existingDef = existingByName.get(tagName)`: Checks if a definition for this `tagName` already exists.
    *   `if (existingDef) { targetSlot = existingDef.tagSlot }`: If an existing definition is found, use its `tagSlot`.
    *   `else { targetSlot = await getNextAvailableSlot(knowledgeBaseId, fieldType, existingBySlot) ... }`: If it's a new tag name:
        *   `getNextAvailableSlot`: Calls a service function to find the next available `tagSlot` (e.g., `tag1`, `tag2`) for this `knowledgeBaseId` and `fieldType`, considering already used slots in `existingBySlot`.
        *   `if (targetSlot) { ... await db.insert(...) }`: If a `targetSlot` is found, a `newDefinition` object is created with a `randomUUID`, `knowledgeBaseId`, `targetSlot`, `displayName`, `fieldType`, and timestamps. This new definition is then `insert`ed into `knowledgeBaseTagDefinitions`.
        *   `existingBySlot.set(targetSlot, newDefinition)`: Updates the `existingBySlot` map to include the newly created definition for subsequent iterations.
        *   `logger.info(...)`: Logs the creation of the new tag definition.
    *   `if (targetSlot) { result[targetSlot] = value }`: If a `targetSlot` was determined (either existing or newly created), the tag's `value` is assigned to that slot in the `result` object.
    *   `logger.error(...)`: Logs any errors during tag processing.

#### `export async function processDocumentsWithQueue(...)`

*   **Purpose:** This function serves as the central dispatcher for document processing. It tries to use the most robust and scalable background processing method available, falling back to simpler methods if higher-priority ones are not configured or fail. The priority order is: Trigger.dev > Redis Queue > In-Memory Concurrency.
*   **Simplified Logic:**
    1.  **Attempt Trigger.dev:** If Trigger.dev is configured, format the documents into its required payload and send them as background tasks. If successful, we're done. If it fails, log a warning and fall back.
    2.  **Attempt Redis Queue:** If Trigger.dev isn't used or fails, check if a Redis client is available. If so, add each document as a job to the Redis-backed queue. Then, *start the queue processor* (which will pick up jobs and call `processDocumentAsync`). If this fails, log a warning and fall back.
    3.  **Fallback to In-Memory:** If neither Trigger.dev nor Redis is available or successful, process the documents directly in the current process, using a local concurrency control mechanism (`processDocumentsWithConcurrencyControl`).
*   **Line-by-Line Deep Dive:**
    *   `if (isTriggerAvailable()) { ... }`: Checks if Trigger.dev is enabled (based on `env` variables).
    *   `logger.info(...)`: Logs that Trigger.dev is being used.
    *   `const triggerPayloads = createdDocuments.map(...)`: Transforms the `createdDocuments` array into `DocumentProcessingPayload` format, ensuring all necessary data and default `processingOptions` are included for Trigger.dev.
    *   `const result = await processDocumentsWithTrigger(triggerPayloads, requestId)`: Calls the function that dispatches jobs to Trigger.dev.
    *   `if (result.success) { ... return }`: If Trigger.dev successfully queued the jobs, log success and exit the function.
    *   `logger.warn(...)`: If Trigger.dev fails, log a warning and proceed to the next fallback.
    *   `catch (error) { logger.warn(...) }`: Catches any exceptions thrown by Trigger.dev integration and logs a warning, then falls back.
    *   `const queue = getDocumentQueue()` and `const redisClient = getRedisClient()`: Gets instances of the processing queue and Redis client.
    *   `if (redisClient) { ... }`: Checks if Redis is actually configured and available.
    *   `logger.info(...)`: Logs that Redis queue is being used.
    *   `const jobPromises = createdDocuments.map((doc) => queue.addJob<DocumentJobData>(...))`: For each document, adds a job to the `queue` (which could be Redis-backed if `redisClient` exists). `addJob` returns a promise that resolves when the job is added.
    *   `await Promise.all(jobPromises)`: Waits for all documents to be successfully added to the queue.
    *   `queue.processJobs(async (job) => { ... }).catch(...)`: This is crucial. It starts the queue's internal worker that will *consume* jobs. For each `job` from the queue, it extracts the `data` and then calls `processDocumentAsync` to actually do the heavy lifting. The `.catch` handles errors that occur during the processing of individual jobs.
    *   `logger.info(...)`: Logs that all documents have been queued for Redis processing.
    *   `return`: Exits the function since Redis processing is initiated.
    *   `catch (error) { logger.warn(...) }`: Catches any exceptions during Redis queueing and falls back.
    *   `logger.info(...)`: If neither Trigger.dev nor Redis worked, logs that in-memory processing is being used.
    *   `await processDocumentsWithConcurrencyControl(...)`: Calls the in-memory fallback processing function.

#### `async function processDocumentsWithConcurrencyControl(...)`

*   **Purpose:** This function handles processing a list of documents directly within the current Node.js process. It manages concurrency and introduces delays to prevent overwhelming resources or external APIs, especially when not using a dedicated background job system like Redis or Trigger.dev.
*   **Simplified Logic:**
    1.  Divide the `createdDocuments` into smaller `batches`.
    2.  Iterate through each `batch`:
        *   Process all documents within that batch concurrently, but respect a maximum concurrency limit and introduce small delays between individual documents (`processBatchWithConcurrency`).
        *   After a batch is complete, pause for a configured `delayBetweenBatches` before starting the next batch.
    3.  Log progress throughout.
*   **Line-by-Line Deep Dive:**
    *   `const totalDocuments = createdDocuments.length`: Gets the total number of documents.
    *   `const batches = []`: Initializes an array to store document batches.
    *   `for (let i = 0; i < totalDocuments; i += PROCESSING_CONFIG.batchSize) { batches.push(createdDocuments.slice(i, i + PROCESSING_CONFIG.batchSize)) }`: Loops through `createdDocuments`, slicing them into `batchSize` chunks and pushing each chunk into the `batches` array.
    *   `logger.info(...)`: Logs the number of documents and batches.
    *   `for (const [batchIndex, batch] of batches.entries()) { ... }`: Iterates through each `batch` with its `batchIndex`.
    *   `logger.info(...)`: Logs the start of each batch.
    *   `await processBatchWithConcurrency(batch, knowledgeBaseId, processingOptions, requestId)`: Calls the function to process the current batch of documents. `await` ensures that one batch finishes before the next begins.
    *   `if (batchIndex < batches.length - 1) { ... }`: Checks if this is not the last batch.
    *   `const config = getProcessingConfig()`: Gets the current processing configuration (could be `PROCESSING_CONFIG` or `REDIS_PROCESSING_CONFIG`, though for this in-memory fallback, it should always be `PROCESSING_CONFIG`).
    *   `if (config.delayBetweenBatches > 0) { await new Promise((resolve) => setTimeout(resolve, config.delayBetweenBatches)) }`: If `delayBetweenBatches` is configured, pauses execution for that duration before starting the next batch.
    *   `logger.info(...)`: Logs completion of processing initiation.

#### `async function processBatchWithConcurrency(...)`

*   **Purpose:** Processes a single batch of documents concurrently, but not exceeding a specified `maxConcurrentDocuments` limit. It uses a "semaphore" pattern to control how many tasks run at once.
*   **Simplified Logic (Semaphore):**
    1.  Create a "semaphore" (an array) with a size equal to `maxConcurrentDocuments`, filled with `0`s (representing available slots).
    2.  For each document in the batch:
        *   **Wait for a slot:** Continuously check the semaphore array for an available slot (`0`). When found, "claim" it by setting it to `1`. This is done asynchronously, waiting if no slots are free.
        *   **Process:** Once a slot is claimed, call `processDocumentAsync` for that document.
        *   **Release slot:** After `processDocumentAsync` finishes (or fails), "release" the slot by setting it back to `0`.
        *   Optionally, introduce a small delay between starting documents within the batch.
    3.  Wait for all documents in the batch to complete (or fail).
    4.  If a document processing fails, update its status in the database.
*   **Line-by-Line Deep Dive:**
    *   `const config = getProcessingConfig()`: Gets the current processing configuration.
    *   `const semaphore = new Array(config.maxConcurrentDocuments).fill(0)`: Initializes the semaphore array. `0` means a slot is free, `1` means it's occupied.
    *   `const processingPromises = batch.map(async (doc, index) => { ... })`: Maps each `doc` in the `batch` to an `async` function, creating an array of promises.
    *   `if (index > 0 && config.delayBetweenDocuments > 0) { await new Promise((resolve) => setTimeout(resolve, index * config.delayBetweenDocuments)) }`: For all but the first document in the batch, introduces a `delayBetweenDocuments` multiplied by its `index`. This creates a staggered start, preventing a "thundering herd" problem where many documents try to start simultaneously.
    *   `await new Promise<void>((resolve) => { ... checkSlot() })`: This block implements the semaphore's "wait for slot" logic:
        *   `const checkSlot = () => { ... }`: An inner function that tries to find a free slot.
        *   `const availableIndex = semaphore.findIndex((slot) => slot === 0)`: Finds the index of the first available slot (`0`).
        *   `if (availableIndex !== -1) { semaphore[availableIndex] = 1; resolve() }`: If a slot is found, claim it (`1`) and resolve this inner promise, allowing processing to continue.
        *   `else { setTimeout(checkSlot, 100) }`: If no slot is available, wait 100ms and try `checkSlot` again. This creates a busy-wait loop that pauses until a slot is free.
    *   `try { ... await processDocumentAsync(...) } catch (error: unknown) { ... }`: The actual document processing call within a `try-catch` block.
    *   `logger.info(...)`: Logs the start and success of document processing.
    *   `logger.error(...)`: Logs any errors during `processDocumentAsync`.
    *   `try { await db.update(document).set({ ... }).where(...) } catch (dbError: unknown) { ... }`: If `processDocumentAsync` fails, attempts to update the `document` record's `processingStatus` to 'failed' and store the error message in the database. Includes nested error handling for the DB update itself.
    *   `finally { ... semaphore[slotIndex] = 0 }`: Ensures that the semaphore slot is `release`d (`0`) whether `processDocumentAsync` succeeded or failed.
    *   `await Promise.allSettled(processingPromises)`: Waits for all promises in the `processingPromises` array to settle (either resolve or reject). `allSettled` is used instead of `all` so that if one document fails, it doesn't immediately stop the processing of other documents in the batch.

#### `export async function processDocumentAsync(...)`

*   **Purpose:** This is the heart of the document processing logic for a *single* document. It orchestrates parsing the document, generating embeddings for its chunks, and saving everything to the database. It includes robust error handling and timeout management.
*   **Simplified Logic:**
    1.  Mark the document as "processing" in the database.
    2.  Use `withTimeout` to wrap the entire process, ensuring it doesn't run longer than `OVERALL_PROCESSING` timeout.
    3.  **Parse Document:** Call `processDocument` (an external utility) to break the file content into text chunks and extract metadata.
    4.  **Check Chunk Limit:** Ensure the document hasn't produced too many chunks (per `LARGE_DOC_CONFIG`).
    5.  **Generate Embeddings:** For each text chunk, call `generateEmbeddings` (an external AI API utility) to get its vector representation. This is done in batches for efficiency.
    6.  **Fetch Document Tags:** Retrieve the `tag1` to `tag7` values from the main `document` record so they can be associated with each `embedding` record.
    7.  **Prepare Embedding Records:** Create database records for each chunk, including its text, embedding vector, and inherited tags.
    8.  **Database Transaction:**
        *   **Insert Embeddings:** Insert all `embedding` records into the database in batches.
        *   **Update Document:** Update the main `document` record with final statistics (chunk count, token count, character count) and set its `processingStatus` to 'completed'.
    9.  If any error occurs at any stage, update the document's status to 'failed' in the database and log the details.
*   **Line-by-Line Deep Dive:**
    *   `const startTime = Date.now()`: Records the start time for performance logging.
    *   `try { ... } catch (error) { ... }`: Outer error handling block for the entire process.
    *   `await db.update(document).set({ processingStatus: 'processing', processingStartedAt: new Date(), processingError: null }).where(eq(document.id, documentId))`: Updates the document's status in the `document` table to 'processing', sets the start timestamp, and clears any previous error.
    *   `await withTimeout((async () => { ... })(), TIMEOUTS.OVERALL_PROCESSING, 'Document processing')`: Wraps the core logic in the `withTimeout` helper function using the `OVERALL_PROCESSING` timeout. The `(async () => { ... })()` is an immediately invoked async function expression.
    *   `const processed = await processDocument(...)`: Calls the external `document-processor` to parse the `fileUrl` and chunk the document based on `processingOptions`.
    *   `if (processed.chunks.length > LARGE_DOC_CONFIG.MAX_CHUNKS_PER_DOCUMENT) { throw new Error(...) }`: Checks if the parsed document created too many chunks, throwing an error if it exceeds the configured limit.
    *   `const now = new Date()`: Gets the current timestamp.
    *   `const chunkTexts = processed.chunks.map((chunk) => chunk.text)`: Extracts just the text content from all processed chunks.
    *   `const embeddings: number[][] = []`: Initializes an array to hold the generated embedding vectors.
    *   `if (chunkTexts.length > 0) { ... }`: Only proceed with embedding generation if there are chunks.
        *   `const batchSize = LARGE_DOC_CONFIG.MAX_EMBEDDING_BATCH`: Gets the batch size for embedding API calls.
        *   `for (let i = 0; i < chunkTexts.length; i += batchSize) { ... }`: Loops through `chunkTexts` in batches.
        *   `const batch = chunkTexts.slice(i, i + batchSize)`: Extracts the current batch of texts.
        *   `const batchEmbeddings = await generateEmbeddings(batch)`: Calls the external `generateEmbeddings` utility with the current batch of texts.
        *   `embeddings.push(...batchEmbeddings)`: Appends the generated embeddings for the batch to the main `embeddings` array.
    *   `const documentRecord = await db.select({ ... }).from(document).where(eq(document.id, documentId)).limit(1)`: Fetches the specific `tag` columns (tag1 to tag7) from the `document` record. This is important because these tags need to be copied to the individual embedding records.
    *   `const documentTags = documentRecord[0] || {}`: Extracts the tag values, defaulting to an empty object if no record is found.
    *   `const embeddingRecords = processed.chunks.map((chunk, chunkIndex) => ({ ... }))`: Maps each parsed `chunk` to a new `embedding` database record object.
        *   `id: crypto.randomUUID()`: Generates a unique ID for each embedding record.
        *   `knowledgeBaseId`, `documentId`, `chunkIndex`: Links the embedding to its parent document and knowledge base.
        *   `chunkHash`: A SHA256 hash of the chunk's text for content integrity or deduplication.
        *   `content`, `contentLength`, `tokenCount`: Stores the chunk text and its size metrics.
        *   `embedding`: The generated vector embedding for the chunk.
        *   `embeddingModel`: The name of the model used to generate the embedding.
        *   `startOffset`, `endOffset`: Metadata about the chunk's position in the original document.
        *   `tag1: documentTags.tag1, ... tag7: documentTags.tag7`: **Crucially, copies the tags from the main document record to each embedding record.** This allows filtering/retrieval of specific chunks based on document-level tags.
    *   `await db.transaction(async (tx) => { ... })`: Initiates a Drizzle database transaction. This ensures that either all database operations (inserting embeddings and updating the document) succeed, or none of them do, maintaining data integrity.
        *   `if (embeddingRecords.length > 0) { ... }`: If there are embedding records to insert:
            *   `const batchSize = LARGE_DOC_CONFIG.MAX_CHUNKS_PER_BATCH`: Uses the configured batch size for database inserts.
            *   `for (let i = 0; i < embeddingRecords.length; i += batchSize) { ... }`: Loops through `embeddingRecords` in batches for insertion.
            *   `await tx.insert(embedding).values(batch)`: Inserts the current batch of embedding records into the `embedding` table within the transaction.
        *   `await tx.update(document).set({ ... }).where(eq(document.id, documentId))`: Updates the main `document` record within the transaction:
            *   Sets `chunkCount`, `tokenCount`, `characterCount` based on the processed metadata.
            *   Sets `processingStatus` to 'completed'.
            *   Sets `processingCompletedAt` to `now`.
            *   Clears `processingError`.
    *   `const processingTime = Date.now() - startTime`: Calculates the total processing time.
    *   `logger.info(...)`: Logs successful completion and processing time.
    *   `catch (error) { ... }`: The outer catch block for `processDocumentAsync`.
        *   `logger.error(...)`: Logs detailed error information.
        *   `await db.update(document).set({ processingStatus: 'failed', processingError: error instanceof Error ? error.message : 'Unknown error', processingCompletedAt: new Date(), }).where(eq(document.id, documentId))`: Updates the document's status to 'failed' in the database, records the error message, and sets the completion timestamp.

#### `export function isTriggerAvailable(): boolean`

*   **Purpose:** Simple utility function to check if Trigger.dev is configured and enabled in the environment.
*   **Logic:** Returns `true` if `env.TRIGGER_SECRET_KEY` is present (indicating configuration) AND `env.TRIGGER_DEV_ENABLED` is not explicitly `false`.

#### `export async function processDocumentsWithTrigger(...)`

*   **Purpose:** Dispatches an array of document processing tasks to the Trigger.dev background job system.
*   **Simplified Logic:**
    1.  Verify Trigger.dev is available.
    2.  For each document payload, call `tasks.trigger` to schedule a `knowledge-process-document` job.
    3.  Collect the job IDs from Trigger.dev.
    4.  Return success or failure.
*   **Line-by-Line Deep Dive:**
    *   `if (!isTriggerAvailable()) { throw new Error(...) }`: Pre-check to ensure Trigger.dev is configured.
    *   `try { ... } catch (error) { ... }`: Error handling for Trigger.dev interaction.
    *   `logger.info(...)`: Logs the intention to trigger background jobs.
    *   `const jobPromises = documents.map(async (document) => { ... })`: Maps each `document` payload to an `async` function that triggers a job.
    *   `const job = await tasks.trigger('knowledge-process-document', document)`: Calls the Trigger.dev SDK to schedule a job with the specified `workflow` name and `payload`.
    *   `return job.id`: Returns the ID of the triggered job.
    *   `const jobIds = await Promise.all(jobPromises)`: Waits for all job triggering calls to complete and collects their IDs.
    *   `logger.info(...)`: Logs the number of jobs successfully triggered.
    *   `return { success: true, ... }`: Returns a success object with job IDs.
    *   `logger.error(...)`: Logs any errors during job triggering.
    *   `return { success: false, ... }`: Returns a failure object with an error message.

#### `export async function createDocumentRecords(...)`

*   **Purpose:** Creates multiple new document records in the database as part of a bulk upload operation. It also integrates with the tag processing logic.
*   **Simplified Logic:**
    1.  Run all operations within a database transaction.
    2.  For each document in the input array:
        *   Generate a new UUID for the document.
        *   If `documentTagsData` is provided (as a JSON string), parse it and use `processDocumentTags` to map user-defined tags to `tag1`-`tag7` slots and create tag definitions if new. These processed tags take precedence over explicitly provided `tagX` fields.
        *   Construct a new `document` record with a 'pending' processing status.
        *   Collect these new document records.
    3.  Insert all collected new document records into the database in a single batch.
    4.  Return a simplified list of the created documents.
*   **Line-by-Line Deep Dive:**
    *   `return await db.transaction(async (tx) => { ... })`: Wraps the entire operation in a database transaction (`tx`), ensuring atomicity.
    *   `const now = new Date()`: Gets the current timestamp.
    *   `const documentRecords = []`, `const returnData: DocumentData[] = []`: Initializes arrays to hold the full DB records and the simplified return data, respectively.
    *   `for (const docData of documents) { ... }`: Loops through each document's data in the input array.
    *   `const documentId = randomUUID()`: Generates a new unique ID for the document.
    *   `let processedTags: Record<string, string | null> = { ... }`: Initializes `processedTags` with `null` for all tag slots, or with values from `docData.tagX` if provided.
    *   `if (docData.documentTagsData) { ... }`: Checks if structured tag data (JSON string) is provided.
        *   `try { const tagData = JSON.parse(docData.documentTagsData) ... } catch (error) { ... }`: Attempts to parse the `documentTagsData` string as JSON. If successful and it's an array, it calls `processDocumentTags` (which handles definition creation and slot mapping) to get the resolved `tag1`-`tag7` values. Handles parsing errors gracefully.
    *   `const newDocument = { ... }`: Creates the `newDocument` object, including:
        *   The generated `documentId`, `knowledgeBaseId`, file metadata.
        *   Initial `chunkCount`, `tokenCount`, `characterCount` as `0`.
        *   `processingStatus` as 'pending'.
        *   `uploadedAt` as `now`.
        *   `tag1` through `tag7`: Prioritizes `processedTags` values, falling back to `docData.tagX` if `processedTags` don't have a value for a slot.
    *   `documentRecords.push(newDocument)`: Adds the constructed document record to the `documentRecords` array for bulk insertion.
    *   `returnData.push({ ... })`: Adds simplified data for the created document to the `returnData` array.
    *   `if (documentRecords.length > 0) { await tx.insert(document).values(documentRecords) ... }`: If there are documents to insert, performs a bulk insert into the `document` table using the transaction (`tx`).
    *   `return returnData`: Returns the simplified list of created documents.

#### `export async function getDocuments(...)`

*   **Purpose:** Fetches a list of documents for a given knowledge base, supporting filtering, searching, sorting, and pagination.
*   **Simplified Logic:**
    1.  Extract and apply default values for options like `limit`, `offset`, `sortBy`, `sortOrder`.
    2.  Build a list of `WHERE` conditions:
        *   Always filter by `knowledgeBaseId` and `isNull(deletedAt)`.
        *   Optionally filter out disabled documents.
        *   Optionally add a `LIKE` search condition for `filename`.
    3.  Query the database to get the total count of documents matching the `WHERE` conditions (for pagination).
    4.  Determine if there are more documents (`hasMore`).
    5.  Construct the `ORDER BY` clause dynamically based on `sortBy` and `sortOrder`, including a secondary sort for stability.
    6.  Execute the main query to select the desired document fields, applying `WHERE`, `ORDER BY`, `LIMIT`, and `OFFSET`.
    7.  Log the retrieval information.
    8.  Return the fetched documents and pagination metadata.
*   **Line-by-Line Deep Dive:**
    *   `const { includeDisabled = false, search, limit = 50, offset = 0, sortBy = 'filename', sortOrder = 'asc' } = options`: Destructures `options` with default values.
    *   `const whereConditions = [ ... ]`: Initializes an array to build dynamic `WHERE` clauses.
    *   `eq(document.knowledgeBaseId, knowledgeBaseId)`: Adds the primary filter condition.
    *   `isNull(document.deletedAt)`: Ensures only non-deleted documents are retrieved.
    *   `if (!includeDisabled) { whereConditions.push(eq(document.enabled, true)) }`: Adds condition to filter by `enabled` status if `includeDisabled` is `false`.
    *   `if (search) { whereConditions.push(sql`LOWER(${document.filename}) LIKE LOWER(${`%${search}%`})`) }`: If `search` term is provided, adds a case-insensitive `LIKE` condition to search the `filename`. `sql` is used for raw SQL.
    *   `const totalResult = await db.select({ count: sql<number>`COUNT(*)` }).from(document).where(and(...whereConditions))`: Executes a `COUNT(*)` query using the `whereConditions` to get the total number of matching documents.
    *   `const total = totalResult[0]?.count || 0`: Extracts the total count, defaulting to 0.
    *   `const hasMore = offset + limit < total`: Calculates if there are more pages of results.
    *   `const getOrderByColumn = () => { ... }`: A helper function to map the `sortBy` string (`'filename'`, `'fileSize'`, etc.) to the corresponding Drizzle ORM schema column.
    *   `const primaryOrderBy = sortOrder === 'asc' ? asc(getOrderByColumn()) : desc(getOrderByColumn())`: Constructs the primary `ORDER BY` clause.
    *   `const secondaryOrderBy = sortBy === 'filename' ? desc(document.uploadedAt) : asc(document.filename)`: Adds a secondary sort condition for stability. If the primary sort values are identical, the secondary sort provides a consistent tie-breaker (e.g., if sorting by filename, use upload date as a tie-breaker).
    *   `const documents = await db.select({ ... }).from(document).where(and(...whereConditions)).orderBy(primaryOrderBy, secondaryOrderBy).limit(limit).offset(offset)`: Executes the main query: selects specific fields (including tags), applies `whereConditions` (combined with `and`), `orderBy` clauses, `limit`, and `offset` for pagination.
    *   `logger.info(...)`: Logs how many documents were retrieved.
    *   `return { documents: documents.map(...), pagination: { ... } }`: Returns an object containing the mapped documents (casting `processingStatus` for type safety) and pagination metadata.

#### `export async function createSingleDocument(...)`

*   **Purpose:** Creates a single document record in the database, similar to `createDocumentRecords` but specifically for one document and returning its full new record.
*   **Simplified Logic:**
    1.  Generate a new UUID and timestamp.
    2.  Initialize processed tags, possibly from `documentTagsData` (JSON string) or individual `tagX` fields. If `documentTagsData` is present, parse it and use `processDocumentTags` to resolve tags and create definitions.
    3.  Construct the `newDocument` object with 'pending' status.
    4.  Insert the new document into the `document` table.
    5.  Log and return the full created document object.
*   **Line-by-Line Deep Dive:**
    *   `const documentId = randomUUID()` and `const now = new Date()`: Generates ID and timestamp.
    *   `let processedTags: Record<string, string | null> = { ... }`: Initializes `processedTags` from direct `tagX` fields provided in `documentData`.
    *   `if (documentData.documentTagsData) { ... }`: If structured `documentTagsData` is present:
        *   `try { ... processedTags = await processDocumentTags(...) } catch (error) { ... }`: Parses the JSON string and calls `processDocumentTags` to resolve dynamic tags to slots, overriding any direct `tagX` values that might have been initialized. Handles parsing errors.
    *   `const newDocument = { ...documentData, id: documentId, knowledgeBaseId, ...processedTags, ... }`: Constructs the `newDocument` object. It uses a spread operator for `documentData` and `processedTags` to merge them, ensuring `id` and `knowledgeBaseId` are explicitly set, and `processedTags` take precedence for tag values.
    *   `await db.insert(document).values(newDocument)`: Inserts the new document into the `document` table.
    *   `logger.info(...)`: Logs the document creation.
    *   `return newDocument as ...`: Returns the newly created document object, casting it to the full document type.

#### `export async function bulkDocumentOperation(...)`

*   **Purpose:** Allows performing bulk 'enable', 'disable', or 'soft delete' operations on multiple documents within a knowledge base.
*   **Simplified Logic:**
    1.  Verify that all `documentIds` provided actually belong to the given `knowledgeBaseId` and are not already soft-deleted.
    2.  If the `operation` is 'delete', update the `deletedAt` field for all valid documents to the current timestamp.
    3.  If the `operation` is 'enable' or 'disable', update the `enabled` field accordingly for all valid documents.
    4.  Return the count of successfully updated documents and their IDs/updated fields.
*   **Line-by-Line Deep Dive:**
    *   `logger.info(...)`: Logs the start of the bulk operation.
    *   `const documentsToUpdate = await db.select({ id: document.id, enabled: document.enabled }).from(document).where(and(eq(document.knowledgeBaseId, knowledgeBaseId), inArray(document.id, documentIds), isNull(document.deletedAt)))`: Selects the `id` and `enabled` status of documents that match the `knowledgeBaseId`, are in the provided `documentIds` list, and are *not* already deleted. This step is crucial for security and data integrity, ensuring only authorized and valid documents are affected.
    *   `if (documentsToUpdate.length === 0) { throw new Error(...) }`: If no valid documents are found, throws an error.
    *   `if (documentsToUpdate.length !== documentIds.length) { logger.warn(...) }`: Logs a warning if some requested `documentIds` were not found or didn't belong to the knowledge base.
    *   `let updateResult: Array<{ id: string; ... }> `: Declares a variable to hold the results of the update query (which rows were affected).
    *   `if (operation === 'delete') { ... }`: If the operation is 'delete':
        *   `updateResult = await db.update(document).set({ deletedAt: new Date() }).where(and(eq(document.knowledgeBaseId, knowledgeBaseId), inArray(document.id, documentIds), isNull(document.deletedAt))).returning({ id: document.id, deletedAt: document.deletedAt })`: Updates the `deletedAt` field to the current date for valid documents. `returning` captures the `id` and the new `deletedAt` value of the updated rows.
    *   `else { ... }`: If the operation is 'enable' or 'disable':
        *   `const enabled = operation === 'enable'`: Sets the `enabled` flag based on the operation.
        *   `updateResult = await db.update(document).set({ enabled }).where(and(eq(document.knowledgeBaseId, knowledgeBaseId), inArray(document.id, documentIds), isNull(document.deletedAt))).returning({ id: document.id, enabled: document.enabled })`: Updates the `enabled` field for valid documents, returning `id` and `enabled`.
    *   `const successCount = updateResult.length`: Counts how many documents were successfully updated.
    *   `logger.info(...)`: Logs the completion of the bulk operation.
    *   `return { success: true, successCount, updatedDocuments: updateResult }`: Returns the result.

#### `export async function markDocumentAsFailedTimeout(...)`

*   **Purpose:** Marks a document as having failed processing due to a timeout. This is typically used by external monitoring systems (e.g., a "dead man's switch" in a background worker) to detect documents stuck in an incomplete state.
*   **Simplified Logic:**
    1.  Calculate how long the document has been processing.
    2.  Throw an error if the processing duration is below a `DEAD_PROCESS_THRESHOLD_MS` to prevent prematurely marking active documents as failed.
    3.  Update the document's `processingStatus` to 'failed', set an appropriate error message, and record the `processingCompletedAt` timestamp.
    4.  Log the action.
*   **Line-by-Line Deep Dive:**
    *   `const now = new Date()`: Gets the current timestamp.
    *   `const processingDuration = now.getTime() - processingStartedAt.getTime()`: Calculates the duration in milliseconds.
    *   `const DEAD_PROCESS_THRESHOLD_MS = 150 * 1000`: Defines a threshold (150 seconds / 2.5 minutes).
    *   `if (processingDuration <= DEAD_PROCESS_THRESHOLD_MS) { throw new Error(...) }`: Ensures that a document has been "stuck" for a sufficiently long time before marking it as failed, preventing race conditions or false positives.
    *   `await db.update(document).set({ processingStatus: 'failed', processingError: 'Processing timed out - background process may have been terminated', processingCompletedAt: now }).where(eq(document.id, documentId))`: Updates the document record in the database with the 'failed' status, error message, and completion timestamp.
    *   `logger.info(...)`: Logs the action with the duration.
    *   `return { success: true, processingDuration }`: Returns success and the calculated duration.

#### `export async function retryDocumentProcessing(...)`

*   **Purpose:** Allows an administrator or system to retry processing a document that previously failed. This involves cleaning up old data and restarting the processing pipeline.
*   **Simplified Logic:**
    1.  **Transaction:**
        *   Delete all existing `embedding` records associated with this document.
        *   Reset the main `document` record's processing-related fields (status to 'pending', clear timestamps, errors, chunk/token counts).
    2.  Define default `processingOptions`.
    3.  Start `processDocumentAsync` for this document *in the background* (without `await`ing it directly), so the API call returns quickly. Handle any potential errors from this background process.
    4.  Log the initiation of the retry.
*   **Line-by-Line Deep Dive:**
    *   `await db.transaction(async (tx) => { ... })`: Wraps the cleanup and reset operations in a transaction for atomicity.
    *   `await tx.delete(embedding).where(eq(embedding.documentId, documentId))`: Deletes all embedding records linked to this `documentId`. This ensures a clean slate for new embeddings.
    *   `await tx.update(document).set({ ... }).where(eq(document.id, documentId))`: Resets the document's `processingStatus` to 'pending', clears `processingStartedAt`, `processingCompletedAt`, `processingError`, `chunkCount`, `tokenCount`, and `characterCount`.
    *   `const processingOptions = { ... }`: Defines a set of default `processingOptions` for the retry.
    *   `processDocumentAsync(knowledgeBaseId, documentId, docData, processingOptions).catch((error: unknown) => { logger.error(...) })`: **Crucially, this calls `processDocumentAsync` without `await`**. This makes the retry operation *asynchronous* from the perspective of the caller, meaning the `retryDocumentProcessing` function returns immediately while the actual processing happens in the background. The `.catch()` ensures any errors from this background process are logged.
    *   `logger.info(...)`: Logs the initiation of the retry.
    *   `return { success: true, status: 'pending', message: 'Document retry processing started' }`: Returns a success message, indicating that the retry has been initiated.

#### `export async function updateDocument(...)`

*   **Purpose:** Updates specific fields of an existing document, including a special consideration for propagating tag updates to associated embedding records.
*   **Simplified Logic:**
    1.  Build an object (`dbUpdateData`) containing only the fields that are actually provided in `updateData`.
    2.  Specifically check for updates to `tag1` through `tag7`.
    3.  **Transaction:**
        *   Update the main `document` record with the `dbUpdateData`.
        *   If any tags (`tag1`-`tag7`) were updated in the `document` record, *also* update the corresponding tag fields in *all* `embedding` records linked to this document. This keeps document-level tags consistent across all its chunks.
    4.  Fetch and return the fully updated document record.
*   **Line-by-Line Deep Dive:**
    *   `const dbUpdateData: Partial<{ ... }> = {}`: Initializes an empty object to build the update payload for the `document` table. `Partial` allows it to only contain some of the fields.
    *   `const TAG_SLOTS = ['tag1', ..., 'tag7'] as const`: Defines an array of all tag slot names.
    *   `if (updateData.filename !== undefined) dbUpdateData.filename = updateData.filename`: This pattern is repeated for all standard updatable fields. It checks if `undefined` (meaning the field was not provided in `updateData`) and only adds it to `dbUpdateData` if it *was* provided.
    *   `TAG_SLOTS.forEach((slot: TagSlot) => { ... })`: Loops through each tag slot.
        *   `const updateValue = (updateData as any)[slot]`: Gets the value for the current tag slot from `updateData`. `as any` is used because TypeScript doesn't natively understand that `slot` will match a key on `updateData`.
        *   `if (updateValue !== undefined) { (dbUpdateData as any)[slot] = updateValue }`: If a tag slot was provided in `updateData`, add it to `dbUpdateData`.
    *   `await db.transaction(async (tx) => { ... })`: Wraps the updates in a database transaction.
    *   `await tx.update(document).set(dbUpdateData).where(eq(document.id, documentId))`: Updates the `document` record with the constructed `dbUpdateData`.
    *   `const hasTagUpdates = TAG_SLOTS.some((field) => (updateData as any)[field] !== undefined)`: Checks if *any* of the `tag1`-`tag7` fields were present in `updateData`.
    *   `if (hasTagUpdates) { ... await tx.update(embedding).set(embeddingUpdateData).where(eq(embedding.documentId, documentId)) }`: If tags were updated:
        *   `const embeddingUpdateData: Record<string, string | null> = {}`: Builds a specific update object for embedding tags.
        *   It iterates `TAG_SLOTS` again, adding only the provided tag updates (or `null` if explicitly set to `null`) to `embeddingUpdateData`.
        *   `await tx.update(embedding).set(embeddingUpdateData).where(eq(embedding.documentId, documentId))`: Updates *all* `embedding` records associated with this `documentId` to reflect the new tag values. This ensures that when chunks are retrieved, their associated tags are consistent with the document's current tags.
    *   `const updatedDocument = await db.select().from(document).where(eq(document.id, documentId)).limit(1)`: After the transaction, fetches the full (and now updated) document record from the database.
    *   `if (updatedDocument.length === 0) { throw new Error(...) }`: Throws an error if the document isn't found (shouldn't happen if the initial update succeeded).
    *   `logger.info(...)`: Logs the document update.
    *   `const doc = updatedDocument[0]; return { ... }`: Returns the first (and only) updated document, casting its `processingStatus` for type safety.

#### `export async function deleteDocument(...)`

*   **Purpose:** Performs a "soft delete" of a single document, meaning it marks the document as deleted without actually removing it from the database. This allows for potential recovery or auditing.
*   **Simplified Logic:**
    1.  Update the `deletedAt` field of the document to the current timestamp.
    2.  Log the action.
*   **Line-by-Line Deep Dive:**
    *   `await db.update(document).set({ deletedAt: new Date() }).where(eq(document.id, documentId))`: Updates the `deletedAt` column of the specified document to the current timestamp.
    *   `logger.info(...)`: Logs the document deletion.
    *   `return { success: true, message: 'Document deleted successfully' }`: Returns a success message.