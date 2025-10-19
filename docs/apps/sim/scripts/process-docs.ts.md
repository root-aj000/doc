This TypeScript file is a utility script designed to process documentation files (likely Markdown or MDX) within a project. Its primary goal is to **convert these human-readable documents into "chunks" of text, generate vector embeddings for each chunk, and then store these chunks and their embeddings in a database.** This process is crucial for implementing features like semantic search or AI-powered question-answering systems over your documentation.

It also provides command-line interface (CLI) options for customization, such as clearing existing data, specifying input paths, controlling chunking parameters, and performing "dry runs" without saving to the database.

Let's break down the code section by section.

---

## Detailed Explanation

### Shebang and Imports

```typescript
#!/usr/bin/env bun

import path from 'path'
import { db } from '@sim/db'
import { docsEmbeddings } from '@sim/db/schema'
import { sql } from 'drizzle-orm'
import { type DocChunk, DocsChunker } from '@/lib/chunkers'
import { isDev } from '@/lib/environment'
import { createLogger } from '@/lib/logs/console/logger'
```

*   **`#!/usr/bin/env bun`**: This is a "shebang" line. It tells the operating system to execute this script using the `bun` runtime. Bun is a fast JavaScript runtime, similar to Node.js.
*   **`import path from 'path'`**: Imports Node.js's built-in `path` module. This module provides utilities for working with file and directory paths, like joining path segments.
*   **`import { db } from '@sim/db'`**: Imports a `db` object, which is likely an instance of a database client (e.g., Drizzle ORM) configured to connect to your application's database. The `@sim/db` alias suggests it's coming from an internal library or module within the `@sim` project.
*   **`import { docsEmbeddings } from '@sim/db/schema'`**: Imports the `docsEmbeddings` object, which represents the schema definition for the `docsEmbeddings` table in your database. This is used by the ORM (likely Drizzle) to interact with that specific table.
*   **`import { sql } from 'drizzle-orm'`**: Imports the `sql` helper from the `drizzle-orm` library. This is used to write raw SQL expressions within Drizzle queries, particularly for aggregate functions like `count(*)`.
*   **`import { type DocChunk, DocsChunker } from '@/lib/chunkers'`**: Imports two things from a custom module:
    *   `DocChunk` (type): An interface or type definition describing the structure of a single documentation chunk (e.g., its text, source, embedding, etc.).
    *   `DocsChunker`: A class responsible for taking raw documentation files, splitting them into smaller, meaningful chunks, and generating embeddings for each chunk.
*   **`import { isDev } from '@/lib/environment'`**: Imports a boolean variable `isDev` from a custom environment module. This variable likely indicates whether the application is running in a development environment (`true`) or production (`false`). It's used to determine default URLs.
*   **`import { createLogger } from '@/lib/logs/console/logger'`**: Imports a function to create a logger instance. This allows for structured and informative console output.

### Logger Initialization

```typescript
const logger = createLogger('ProcessDocs')
```

*   **`const logger = createLogger('ProcessDocs')`**: Creates a logger instance specifically for this `ProcessDocs` script. The string `'ProcessDocs'` is often used as a label or namespace for log messages, making it easier to identify where a log originated.

### `ProcessingOptions` Interface

```typescript
interface ProcessingOptions {
  /** Clear existing docs embeddings before processing */
  clearExisting?: boolean
  /** Path to docs directory */
  docsPath?: string
  /** Base URL for generating links */
  baseUrl?: string
  /** Chunk size in tokens */
  chunkSize?: number
  /** Minimum chunk size */
  minChunkSize?: number
  /** Overlap between chunks */
  overlap?: number
  /** Dry run - only display results, don't save to DB */
  dryRun?: boolean
  /** Verbose output */
  verbose?: boolean
}
```

This interface defines the various configuration options that can be passed to the `processDocs` function. All properties are optional (`?`) as the function provides default values.

*   **`clearExisting?: boolean`**: If `true`, all existing documentation embeddings in the database will be deleted before new ones are saved.
*   **`docsPath?: string`**: The file path to the directory containing the documentation files (e.g., `markdown` or `mdx` files).
*   **`baseUrl?: string`**: The base URL used to construct full, browsable links to the processed documentation chunks. This is important for linking from search results back to the original document.
*   **`chunkSize?: number`**: The target maximum size (in tokens) for each documentation chunk. Tokens are small units of text, often words or sub-words.
*   **`minChunkSize?: number`**: The minimum size (in tokens) for a chunk. Chunks smaller than this might be merged or handled differently by the chunker.
*   **`overlap?: number`**: The number of tokens that adjacent chunks should share. Overlapping helps provide context when reading chunks sequentially and ensures no information is lost at chunk boundaries.
*   **`dryRun?: boolean`**: If `true`, the script will perform all processing (chunking, embedding generation) but will *not* save any data to the database. It's useful for testing or previewing the output.
*   **`verbose?: boolean`**: If `true`, the script will output more detailed information, such as previews of the chunk text.

### `processDocs` Function

```typescript
/**
 * Process documentation files and optionally save embeddings to database
 */
async function processDocs(options: ProcessingOptions = {}) {
  const config = {
    docsPath: options.docsPath || path.join(process.cwd(), '../../apps/docs/content/docs'),
    baseUrl: options.baseUrl || (isDev ? 'http://localhost:4000' : 'https://docs.sim.ai'),
    chunkSize: options.chunkSize || 1024,
    minChunkSize: options.minChunkSize || 100,
    overlap: options.overlap || 200,
    clearExisting: options.clearExisting ?? false,
    dryRun: options.dryRun ?? false,
    verbose: options.verbose ?? false,
  }

  let processedChunks = 0
  let failedChunks = 0

  try {
    logger.info('🚀 Starting docs processing with config:', {
      docsPath: config.docsPath,
      baseUrl: config.baseUrl,
      chunkSize: config.chunkSize,
      clearExisting: config.clearExisting,
      dryRun: config.dryRun,
    })

    // Initialize the chunker
    const chunker = new DocsChunker({
      chunkSize: config.chunkSize,
      minChunkSize: config.minChunkSize,
      overlap: config.overlap,
      baseUrl: config.baseUrl,
    })

    // Process all .mdx files
    logger.info(`📚 Processing docs from: ${config.docsPath}`)
    const chunks = await chunker.chunkAllDocs(config.docsPath)

    if (chunks.length === 0) {
      logger.warn('⚠️ No chunks generated from docs')
      return { success: false, processedChunks: 0, failedChunks: 0 }
    }

    logger.info(`📊 Generated ${chunks.length} chunks with embeddings`)

    // Group chunks by document for summary
    const chunksByDoc = chunks.reduce<Record<string, DocChunk[]>>((acc, chunk) => {
      if (!acc[chunk.sourceDocument]) {
        acc[chunk.sourceDocument] = []
      }
      acc[chunk.sourceDocument].push(chunk)
      return acc
    }, {})

    // Display summary
    logger.info(`\n=== DOCUMENT SUMMARY ===`)
    for (const [doc, docChunks] of Object.entries(chunksByDoc)) {
      logger.info(`${doc}: ${docChunks.length} chunks`)
    }

    // Display sample chunks in verbose or dry-run mode
    if (config.verbose || config.dryRun) {
      logger.info(`\n=== SAMPLE CHUNKS ===`)
      chunks.slice(0, 3).forEach((chunk, index) => {
        logger.info(`\nChunk ${index + 1}:`)
        logger.info(`  Source: ${chunk.sourceDocument}`)
        logger.info(`  Header: ${chunk.headerText} (Level ${chunk.headerLevel})`)
        logger.info(`  Link: ${chunk.headerLink}`)
        logger.info(`  Tokens: ${chunk.tokenCount}`)
        logger.info(`  Embedding: ${chunk.embedding.length} dimensions (${chunk.embeddingModel})`)
        if (config.verbose) {
          logger.info(`  Text Preview: ${chunk.text.substring(0, 200)}...`)
        }
      })
    }

    // If dry run, stop here
    if (config.dryRun) {
      logger.info('\n✅ Dry run complete - no data saved to database')
      return { success: true, processedChunks: chunks.length, failedChunks: 0 }
    }

    // Clear existing embeddings if requested
    if (config.clearExisting) {
      logger.info('🗑️ Clearing existing docs embeddings...')
      try {
        await db.delete(docsEmbeddings)
        logger.info(`✅ Successfully deleted existing embeddings`)
      } catch (error) {
        logger.error('❌ Failed to delete existing embeddings:', error)
        throw new Error('Failed to clear existing embeddings')
      }
    }

    // Save chunks to database in batches
    const batchSize = 10
    logger.info(`💾 Saving chunks to database (batch size: ${batchSize})...`)

    for (let i = 0; i < chunks.length; i += batchSize) {
      const batch = chunks.slice(i, i + batchSize)

      try {
        const batchData = batch.map((chunk) => ({
          chunkText: chunk.text,
          sourceDocument: chunk.sourceDocument,
          sourceLink: chunk.headerLink,
          headerText: chunk.headerText,
          headerLevel: chunk.headerLevel,
          tokenCount: chunk.tokenCount,
          embedding: chunk.embedding,
          embeddingModel: chunk.embeddingModel,
          metadata: chunk.metadata,
        }))

        await db.insert(docsEmbeddings).values(batchData)
        processedChunks += batch.length

        if (i % (batchSize * 5) === 0 || i + batchSize >= chunks.length) {
          logger.info(
            `  💾 Saved ${Math.min(i + batchSize, chunks.length)}/${chunks.length} chunks`
          )
        }
      } catch (error) {
        logger.error(`❌ Failed to save batch ${Math.floor(i / batchSize) + 1}:`, error)
        failedChunks += batch.length
      }
    }

    // Verify final count
    const savedCount = await db
      .select({ count: sql<number>`count(*)` })
      .from(docsEmbeddings)
      .then((res) => res[0]?.count || 0)

    logger.info(
      `\n✅ Processing complete!\n` +
        `   📊 Total chunks: ${chunks.length}\n` +
        `   ✅ Processed: ${processedChunks}\n` +
        `   ❌ Failed: ${failedChunks}\n` +
        `   💾 Total in DB: ${savedCount}`
    )

    return { success: failedChunks === 0, processedChunks, failedChunks }
  } catch (error) {
    logger.error('❌ Fatal error during processing:', error)
    return { success: false, processedChunks, failedChunks }
  }
}
```

This is the main asynchronous function that orchestrates the documentation processing.

*   **`async function processDocs(options: ProcessingOptions = {})`**: Defines an asynchronous function `processDocs` that accepts an `options` object of type `ProcessingOptions`. If no options are provided, it defaults to an empty object `{}`.
*   **`const config = { ... }`**: This block sets up the final configuration for the processing run. It uses the provided `options` if they exist, otherwise it falls back to sensible default values.
    *   `docsPath`: Defaults to a relative path `../../apps/docs/content/docs` from the current working directory if not specified. `process.cwd()` gets the current working directory.
    *   `baseUrl`: Defaults to `http://localhost:4000` in development (`isDev` is true) and `https://docs.sim.ai` in production.
    *   `chunkSize`, `minChunkSize`, `overlap`: Default token values for chunking parameters.
    *   `clearExisting`, `dryRun`, `verbose`: Use the provided option or default to `false`. The `??` (nullish coalescing) operator ensures that `false` is used only if the option is `null` or `undefined`, preserving an explicit `true` if provided.
*   **`let processedChunks = 0; let failedChunks = 0;`**: Initializes counters to track the number of successfully processed and failed chunks during database saving.
*   **`try { ... } catch (error) { ... }`**: This `try-catch` block wraps the entire processing logic to gracefully handle any unexpected errors that might occur. If a fatal error occurs, it's logged, and the function returns `success: false`.
*   **`logger.info('🚀 Starting docs processing with config:', { ... })`**: Logs the initial configuration parameters, providing a clear overview of how the script is running.
*   **`const chunker = new DocsChunker({ ... })`**: Creates a new instance of the `DocsChunker` class, passing in the configured `chunkSize`, `minChunkSize`, `overlap`, and `baseUrl`.
*   **`logger.info(`📚 Processing docs from: ${config.docsPath}`)`**: Logs the directory from which documents are being processed.
*   **`const chunks = await chunker.chunkAllDocs(config.docsPath)`**: This is a critical step. It calls the `chunkAllDocs` method on the `chunker` instance, passing the `docsPath`. This method is responsible for:
    1.  Finding all relevant document files (e.g., `.mdx`) in the specified directory.
    2.  Reading their content.
    3.  Splitting them into `DocChunk` objects according to the configured `chunkSize`, `minChunkSize`, and `overlap`.
    4.  Generating vector embeddings for each `DocChunk`.
    This operation is asynchronous (`await`) because it might involve file I/O and external API calls for embedding generation.
*   **`if (chunks.length === 0) { ... }`**: Checks if any chunks were generated. If not, it logs a warning and returns early, indicating failure.
*   **`logger.info(`📊 Generated ${chunks.length} chunks with embeddings`)`**: Logs the total number of chunks successfully generated.
*   **`const chunksByDoc = chunks.reduce<Record<string, DocChunk[]>>((acc, chunk) => { ... }, {})`**: This line groups the generated `chunks` by their `sourceDocument`.
    *   `chunks.reduce(...)`: The `reduce` method iterates over the `chunks` array.
    *   `<Record<string, DocChunk[]>>`: Specifies the type of the accumulator (`acc`) as an object where keys are document paths (strings) and values are arrays of `DocChunk` objects.
    *   The callback function checks if an entry for `chunk.sourceDocument` already exists in `acc`. If not, it creates an empty array. Then, it pushes the current `chunk` into that document's array.
    *   `{}`: The initial value of the accumulator.
*   **`logger.info(`\n=== DOCUMENT SUMMARY ===`); for (const [doc, docChunks] of Object.entries(chunksByDoc)) { ... }`**: This block iterates through the `chunksByDoc` object (using `Object.entries` to get key-value pairs) and logs a summary, showing how many chunks were generated for each original document.
*   **`if (config.verbose || config.dryRun) { ... }`**: This conditional block displays sample chunks for debugging or previewing.
    *   `chunks.slice(0, 3)`: Takes the first three chunks from the `chunks` array.
    *   `.forEach((chunk, index) => { ... })`: Iterates over these sample chunks.
    *   It logs detailed information about each sample chunk, including its source, header, link, token count, and embedding dimensions.
    *   **`if (config.verbose) { ... }`**: If `verbose` mode is enabled, it also logs a short preview of the chunk's actual text content.
*   **`if (config.dryRun) { ... }`**: If `dryRun` mode is enabled, the script stops here after processing and displaying results, logs a message, and returns `success: true` without interacting with the database.
*   **`if (config.clearExisting) { ... }`**: If the `clearExisting` option is `true`:
    *   `await db.delete(docsEmbeddings)`: Uses the Drizzle ORM `db` instance to delete all records from the `docsEmbeddings` table. This clears out old data.
    *   It includes a `try-catch` block specifically for this delete operation, re-throwing an error if deletion fails to halt further processing.
*   **`const batchSize = 10; logger.info(`💾 Saving chunks to database (batch size: ${batchSize})...`);`**: Defines a `batchSize` for saving chunks to the database. This is a common optimization to avoid overwhelming the database with too many individual insert statements by grouping them into larger transactions.
*   **`for (let i = 0; i < chunks.length; i += batchSize) { ... }`**: This loop iterates through the `chunks` array, taking `batchSize` chunks at a time.
    *   **`const batch = chunks.slice(i, i + batchSize)`**: Extracts a subset of chunks for the current batch.
    *   **`try { ... } catch (error) { ... }`**: A `try-catch` block specifically for saving each batch, allowing individual batch failures without stopping the entire process.
    *   **`const batchData = batch.map((chunk) => ({ ... }))`**: Transforms each `DocChunk` object into a plain JavaScript object that matches the structure expected by the `docsEmbeddings` database table. It maps properties like `text` to `chunkText`, `headerLink` to `sourceLink`, etc.
    *   **`await db.insert(docsEmbeddings).values(batchData)`**: Uses Drizzle ORM to insert all objects in `batchData` into the `docsEmbeddings` table. This is an efficient bulk insert.
    *   **`processedChunks += batch.length`**: Increments the `processedChunks` counter by the number of chunks successfully saved in the current batch.
    *   **`if (i % (batchSize * 5) === 0 || i + batchSize >= chunks.length) { ... }`**: This condition periodically logs progress to the console (every 5 batches or at the very end).
    *   **`logger.error(`❌ Failed to save batch ...`, error); failedChunks += batch.length;`**: If a batch fails to save, an error is logged, and `failedChunks` is incremented for the entire batch.
*   **`const savedCount = await db .select({ count: sql<number>`count(*)` }) .from(docsEmbeddings) .then((res) => res[0]?.count || 0)`**: After all batches are processed, this queries the database to get the actual total count of records in the `docsEmbeddings` table.
    *   `db.select({ count: sql<number>`count(*)` })`: Drizzle syntax to select an aggregate `count(*)` and cast it as a number.
    *   `.from(docsEmbeddings)`: Specifies the table to count from.
    *   `.then((res) => res[0]?.count || 0)`: Extracts the `count` value from the query result, handling cases where the result might be empty.
*   **`logger.info(`\n✅ Processing complete!\\n` + `   ...` )`**: Logs a final summary of the entire operation, showing total chunks, successfully processed chunks, failed chunks, and the final count in the database.
*   **`return { success: failedChunks === 0, processedChunks, failedChunks }`**: Returns an object indicating the overall success (true if no chunks failed to save), the number of processed chunks, and the number of failed chunks.
*   **`catch (error)` (outer block)**: If any error occurs *outside* the specific batch saving or clearing logic, it's caught here, logged as a fatal error, and the function returns `success: false`.

### `main` Function

```typescript
/**
 * Main entry point with CLI argument parsing
 */
async function main() {
  const args = process.argv.slice(2)

  const options: ProcessingOptions = {
    clearExisting: args.includes('--clear'),
    dryRun: args.includes('--dry-run'),
    verbose: args.includes('--verbose'),
  }

  // Parse custom path if provided
  const pathIndex = args.indexOf('--path')
  if (pathIndex !== -1 && args[pathIndex + 1]) {
    options.docsPath = args[pathIndex + 1]
  }

  // Parse custom base URL if provided
  const urlIndex = args.indexOf('--url')
  if (urlIndex !== -1 && args[urlIndex + 1]) {
    options.baseUrl = args[urlIndex + 1]
  }

  // Parse chunk size if provided
  const chunkSizeIndex = args.indexOf('--chunk-size')
  if (chunkSizeIndex !== -1 && args[chunkSizeIndex + 1]) {
    options.chunkSize = Number.parseInt(args[chunkSizeIndex + 1], 10)
  }

  // Show help if requested
  if (args.includes('--help') || args.includes('-h')) {
    console.log(`
📚 Process Documentation Script

Usage: bun run process-docs.ts [options]

Options:
  --clear          Clear existing embeddings before processing
  --dry-run        Process and display results without saving to DB
  --verbose        Show detailed output including text previews
  --path <path>    Custom path to docs directory
  --url <url>      Custom base URL for links
  --chunk-size <n> Custom chunk size in tokens (default: 1024)
  --help, -h       Show this help message

Examples:
  # Dry run to test chunking
  bun run process-docs.ts --dry-run

  # Process and save to database
  bun run process-docs.ts

  # Clear existing and reprocess
  bun run process-docs.ts --clear

  # Custom path with verbose output
  bun run process-docs.ts --path ./my-docs --verbose
    `)
    process.exit(0)
  }

  const result = await processDocs(options)
  process.exit(result.success ? 0 : 1)
}
```

The `main` function serves as the command-line interface (CLI) entry point for the script.

*   **`async function main() { ... }`**: Defines an asynchronous `main` function.
*   **`const args = process.argv.slice(2)`**: `process.argv` is an array containing command-line arguments.
    *   `process.argv[0]` is the path to the Bun executable.
    *   `process.argv[1]` is the path to the script being executed (`process-docs.ts`).
    *   `.slice(2)` extracts all arguments provided by the user, skipping the first two elements.
*   **`const options: ProcessingOptions = { ... }`**: Initializes an `options` object for `processDocs`.
    *   It checks for simple boolean flags like `--clear`, `--dry-run`, and `--verbose` using `args.includes()`.
*   **Parsing Custom Options (`--path`, `--url`, `--chunk-size`)**: These blocks handle arguments that take a value (e.g., `--path ./my-docs`).
    *   `args.indexOf('--path')`: Finds the index of the `--path` argument.
    *   `pathIndex !== -1 && args[pathIndex + 1]`: Checks if `--path` was found and if there's a value immediately following it.
    *   `options.docsPath = args[pathIndex + 1]`: Assigns the value to the `docsPath` option.
    *   `Number.parseInt(args[chunkSizeIndex + 1], 10)`: Converts the string value of `--chunk-size` to an integer.
*   **`if (args.includes('--help') || args.includes('-h')) { ... }`**: Checks if the `--help` or `-h` flag is present. If so, it prints a helpful usage message to the console and exits the script with a status code of `0` (indicating success).
*   **`const result = await processDocs(options)`**: Calls the main `processDocs` function with the parsed CLI options and awaits its completion.
*   **`process.exit(result.success ? 0 : 1)`**: Exits the Bun process.
    *   If `result.success` is `true`, it exits with status code `0` (success).
    *   Otherwise, it exits with status code `1` (failure). This is important for scripting and automation to check if the script ran correctly.

### Entry Point Guard

```typescript
// Run if executed directly
if (import.meta.url === `file://${process.argv[1]}`) {
  main().catch((error) => {
    logger.error('Fatal error:', error)
    process.exit(1)
  })
}
```

This is a common pattern in modern JavaScript/TypeScript modules:

*   **`if (import.meta.url === `file://${process.argv[1]}`)`**: This condition checks if the current module is being executed directly as a script (e.g., `bun run process-docs.ts`) rather than being imported as a module into another file.
    *   `import.meta.url`: The URL of the current module.
    *   ``file://${process.argv[1]}``: The file URL constructed from the path of the script being executed.
*   **`main().catch((error) => { ... })`**: If the script is run directly, it calls the `main` function. The `.catch()` block ensures that any unhandled errors *within* the `main` function (or `processDocs` called by `main`) are caught, logged, and cause the process to exit with a non-zero status code (`1`) to signal an error.

### Export

```typescript
export { processDocs }
```

*   **`export { processDocs }`**: This line makes the `processDocs` function available for other modules to import and use. This means the core logic can be reused programmatically without having to invoke the CLI `main` function.

---

In summary, this script is a robust tool for maintaining up-to-date, semantically searchable documentation by automating the complex process of chunking, embedding, and storing content in a database. Its CLI interface makes it easy to integrate into development workflows and CI/CD pipelines.