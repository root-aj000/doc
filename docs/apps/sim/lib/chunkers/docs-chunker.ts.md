This TypeScript file defines the `DocsChunker` class, a specialized tool for processing documentation written in MDX (`.mdx`) format. Its primary goal is to take a collection of MDX files, break them down into smaller, meaningful "chunks," extract relevant metadata (like headers and frontmatter), generate unique URLs for these chunks, and then create numerical representations (embeddings) for each chunk.

The output of this process is an array of `DocChunk` objects, which are highly structured pieces of information ready to be used in applications like semantic search, intelligent Q&A systems, or Retrieval-Augmented Generation (RAG) pipelines.

Essentially, this file acts as a pre-processing pipeline to transform raw documentation into an easily queryable and contextually rich format.

### Simplified Complex Logic: How it Chunks Docs Smartly

The most complex part of this file is how it intelligently splits a document into chunks while preserving context, especially around headings and tables. Here's a simplified breakdown:

1.  **Read and Prepare:** It first reads the MDX file, separates any frontmatter (YAML metadata like title) from the actual markdown content, and then "cleans" the markdown by removing MDX-specific JSX components, import statements, and excessive whitespace.

2.  **Identify Structure:** Before breaking the text, it identifies all headings and their positions within the document. This is crucial for later associating chunks with their nearest header for better context. It also detects the precise boundaries of any markdown tables, which are critical to avoid splitting.

3.  **Initial Chunking (Generic):** It uses a general-purpose `TextChunker` to perform an initial split of the cleaned markdown content into smaller pieces based on a desired size (e.g., 300 tokens).

4.  **Table-Aware Merging:** This is where the "smart" part comes in. After the initial generic chunking, it checks if any of these chunks accidentally split a table. If a chunk contains only part of a table, it intelligently merges it with adjacent chunks to ensure the entire table remains within a single chunk. This prevents crucial tabular data from being fragmented and losing meaning.

5.  **Size Re-Enforcement:** Because merging chunks for tables might make some chunks exceed the target size (e.g., 300 tokens), it then performs another pass. Any chunks that are now too large are re-split, but this time, it tries to split them along natural line breaks (like paragraphs) to maintain readability and context as much as possible, while still respecting the size limit.

6.  **Contextual Annotation & Embedding:** Finally, for each resulting chunk:
    *   It determines the most relevant header that appears before the chunk, providing crucial hierarchical context.
    *   It estimates the token count.
    *   It generates a unique URL that links directly to the document and, if possible, to the specific header relevant to the chunk.
    *   It sends all the chunks in a batch to an external service to generate their numerical "embeddings." These embeddings are vectors that capture the semantic meaning of the text, enabling efficient similarity searches.

7.  **Final Output:** All this information is packaged into `DocChunk` objects, which contain the chunk's text, its source document, header context, URL, embedding, and other metadata.

---

### Detailed Line-by-Line Explanation

```typescript
import fs from 'fs/promises'
import path from 'path'
import { generateEmbeddings } from '@/lib/embeddings/utils'
import { createLogger } from '@/lib/logs/console/logger'
import { TextChunker } from './text-chunker'
import type { DocChunk, DocsChunkerOptions } from './types'
```

*   **`import fs from 'fs/promises'`**: Imports Node.js's file system module, specifically its promise-based API, for asynchronous file operations (like reading directories and files).
*   **`import path from 'path'`**: Imports Node.js's `path` module, which provides utilities for working with file and directory paths.
*   **`import { generateEmbeddings } from '@/lib/embeddings/utils'`**: Imports a function `generateEmbeddings` from a custom utility file. This function is responsible for creating numerical vector representations (embeddings) of text, likely by interacting with an external AI model.
*   **`import { createLogger } from '@/lib/logs/console/logger'`**: Imports a `createLogger` function from a custom logging utility. This is used to create a logger instance for reporting progress and errors.
*   **`import { TextChunker } from './text-chunker'`**: Imports the `TextChunker` class from a local file. This is a generic utility for splitting raw text into smaller pieces based on size and overlap, and it's leveraged by `DocsChunker`.
*   **`import type { DocChunk, DocsChunkerOptions } from './types'`**: Imports TypeScript `type` definitions for `DocChunk` (the structure of the final output chunks) and `DocsChunkerOptions` (options for the `DocsChunker` constructor).

```typescript
interface HeaderInfo {
  level: number
  text: string
  slug?: string
  anchor?: string
  position?: number
}
```

*   **`interface HeaderInfo`**: Defines the structure for information extracted from a markdown header.
    *   **`level: number`**: The heading level (e.g., 1 for `# Heading`, 2 for `## Subheading`).
    *   **`text: string`**: The actual text content of the header.
    *   **`slug?: string`**: An optional URL-friendly version of the header text, useful for generating unique IDs.
    *   **`anchor?: string`**: An optional, URL-safe anchor ID derived from the header text, used for deep linking within a document.
    *   **`position?: number`**: An optional character index in the original markdown content where the header starts, useful for relating headers to chunks.

```typescript
interface Frontmatter {
  title?: string
  description?: string
  [key: string]: any
}
```

*   **`interface Frontmatter`**: Defines the structure for document metadata typically found at the top of markdown/MDX files (YAML frontmatter).
    *   **`title?: string`**: An optional title for the document.
    *   **`description?: string`**: An optional description for the document.
    *   **`[key: string]: any`**: An index signature allowing for any other arbitrary key-value pairs to be present in the frontmatter.

```typescript
const logger = createLogger('DocsChunker')
```

*   **`const logger = createLogger('DocsChunker')`**: Creates a logger instance specifically for the `DocsChunker` class, making it easy to identify log messages originating from this component.

```typescript
/**
 * Docs-specific chunker that processes .mdx files and tracks header context
 */
export class DocsChunker {
  private readonly textChunker: TextChunker
  private readonly baseUrl: string

  constructor(options: DocsChunkerOptions = {}) {
    // Use the existing TextChunker for chunking logic
    this.textChunker = new TextChunker({
      chunkSize: options.chunkSize ?? 300, // Max 300 tokens per chunk
      minChunkSize: options.minChunkSize ?? 1,
      overlap: options.overlap ?? 50,
    })
    // Use localhost docs in development, production docs otherwise
    this.baseUrl = options.baseUrl ?? 'https://docs.sim.ai'
  }
```

*   **`export class DocsChunker { ... }`**: Defines the main `DocsChunker` class, which is exported for use in other parts of the application.
*   **`private readonly textChunker: TextChunker`**: Declares a private, read-only property `textChunker`. This will hold an instance of the generic `TextChunker` class, used for the initial text splitting.
*   **`private readonly baseUrl: string`**: Declares a private, read-only property `baseUrl`. This stores the base URL for the documentation, used to construct full URLs to documents.
*   **`constructor(options: DocsChunkerOptions = {})`**: The class constructor, which takes an optional `DocsChunkerOptions` object.
    *   **`this.textChunker = new TextChunker(...)`**: Initializes the `textChunker` instance.
        *   **`chunkSize: options.chunkSize ?? 300`**: Sets the maximum desired size for each text chunk (in an estimated token count). Defaults to 300 tokens if not provided in `options`.
        *   **`minChunkSize: options.minChunkSize ?? 1`**: Sets the minimum desired size for a text chunk. Defaults to 1.
        *   **`overlap: options.overlap ?? 50`**: Sets the number of characters/tokens that chunks should overlap with each other, helping to preserve context across chunk boundaries. Defaults to 50.
    *   **`this.baseUrl = options.baseUrl ?? 'https://docs.sim.ai'`**: Sets the `baseUrl` for the documentation. If no `baseUrl` is provided in the `options`, it defaults to `https://docs.sim.ai`. This implies it might use a different base URL for local development.

```typescript
  /**
   * Process all .mdx files in the docs directory
   */
  async chunkAllDocs(docsPath: string): Promise<DocChunk[]> {
    const allChunks: DocChunk[] = []

    try {
      const mdxFiles = await this.findMdxFiles(docsPath)
      logger.info(`Found ${mdxFiles.length} .mdx files to process`)

      for (const filePath of mdxFiles) {
        try {
          const chunks = await this.chunkMdxFile(filePath, docsPath)
          allChunks.push(...chunks)
          logger.info(`Processed ${filePath}: ${chunks.length} chunks`)
        } catch (error) {
          logger.error(`Error processing ${filePath}:`, error)
        }
      }

      logger.info(`Total chunks generated: ${allChunks.length}`)
      return allChunks
    } catch (error) {
      logger.error('Error processing docs:', error)
      throw error
    }
  }
```

*   **`async chunkAllDocs(docsPath: string): Promise<DocChunk[]>`**: This asynchronous public method is the entry point for processing an entire directory of documentation. It takes the `docsPath` (the root directory of the docs) and returns a Promise that resolves to an array of `DocChunk` objects.
    *   **`const allChunks: DocChunk[] = []`**: Initializes an empty array to store all the processed chunks from all files.
    *   **`try { ... } catch (error) { ... }`**: A try-catch block to handle potential errors during the overall documentation processing.
    *   **`const mdxFiles = await this.findMdxFiles(docsPath)`**: Calls a private helper method `findMdxFiles` to recursively locate all `.mdx` files within the `docsPath`.
    *   **`logger.info(...)`**: Logs the number of MDX files found.
    *   **`for (const filePath of mdxFiles) { ... }`**: Iterates through each found MDX file.
        *   **`try { ... } catch (error) { ... }`**: Another try-catch block to handle errors specific to processing a single file, allowing the overall process to continue even if one file fails.
        *   **`const chunks = await this.chunkMdxFile(filePath, docsPath)`**: Calls a private asynchronous method `chunkMdxFile` to process the current `filePath`. `docsPath` is passed to help determine the relative path.
        *   **`allChunks.push(...chunks)`**: Appends all the `DocChunk` objects returned from processing the current file to the `allChunks` array using the spread operator.
        *   **`logger.info(...)`**: Logs the number of chunks generated for the current file.
        *   **`logger.error(...)`**: Logs any errors encountered while processing a specific file.
    *   **`logger.info(...)`**: Logs the total number of chunks generated after all files have been processed.
    *   **`return allChunks`**: Returns the array of all generated `DocChunk` objects.
    *   **`logger.error(...)`**: Logs any errors from the outer try-catch block (e.g., if `findMdxFiles` fails).
    *   **`throw error`**: Re-throws the error to the caller, indicating a major failure in processing the docs.

```typescript
  /**
   * Process a single .mdx file
   */
  async chunkMdxFile(filePath: string, basePath: string): Promise<DocChunk[]> {
    const content = await fs.readFile(filePath, 'utf-8')
    const relativePath = path.relative(basePath, filePath)

    // Parse frontmatter and content
    const { data: frontmatter, content: markdownContent } = this.parseFrontmatter(content)

    // Extract headers from the content
    const headers = this.extractHeaders(markdownContent)

    // Generate document URL
    const documentUrl = this.generateDocumentUrl(relativePath)

    // Split content into chunks
    const textChunks = await this.splitContent(markdownContent)

    // Generate embeddings for all chunks at once (batch processing)
    logger.info(`Generating embeddings for ${textChunks.length} chunks in ${relativePath}`)
    const embeddings = textChunks.length > 0 ? await generateEmbeddings(textChunks) : []
    const embeddingModel = 'text-embedding-3-small'

    // Convert to DocChunk objects with header context and embeddings
    const chunks: DocChunk[] = []
    let currentPosition = 0

    for (let i = 0; i < textChunks.length; i++) {
      const chunkText = textChunks[i]
      const chunkStart = currentPosition
      const chunkEnd = currentPosition + chunkText.length

      // Find the most relevant header for this chunk
      const relevantHeader = this.findRelevantHeader(headers, chunkStart)

      const chunk: DocChunk = {
        text: chunkText,
        tokenCount: Math.ceil(chunkText.length / 4), // Simple token estimation
        sourceDocument: relativePath,
        headerLink: relevantHeader ? `${documentUrl}#${relevantHeader.anchor}` : documentUrl,
        headerText: relevantHeader?.text || frontmatter.title || 'Document Root',
        headerLevel: relevantHeader?.level || 1,
        embedding: embeddings[i] || [],
        embeddingModel,
        metadata: {
          startIndex: chunkStart,
          endIndex: chunkEnd,
          title: frontmatter.title,
        },
      }

      chunks.push(chunk)
      currentPosition = chunkEnd
    }

    return chunks
  }
```

*   **`async chunkMdxFile(filePath: string, basePath: string): Promise<DocChunk[]>`**: This asynchronous private method processes a single MDX file. It takes the `filePath` of the MDX file and its `basePath` (the root of the docs directory) and returns an array of `DocChunk` objects.
    *   **`const content = await fs.readFile(filePath, 'utf-8')`**: Reads the entire content of the MDX file into a string.
    *   **`const relativePath = path.relative(basePath, filePath)`**: Calculates the path of the file relative to the `basePath`. E.g., if `basePath` is `/docs` and `filePath` is `/docs/features/example.mdx`, `relativePath` will be `features/example.mdx`.
    *   **`const { data: frontmatter, content: markdownContent } = this.parseFrontmatter(content)`**: Calls `parseFrontmatter` to extract YAML frontmatter (metadata) and the remaining markdown content from the file. The extracted data is destructured into `frontmatter` and `markdownContent`.
    *   **`const headers = this.extractHeaders(markdownContent)`**: Calls `extractHeaders` to find all markdown headers (`#`, `##`, etc.) within the `markdownContent` and their positions.
    *   **`const documentUrl = this.generateDocumentUrl(relativePath)`**: Calls `generateDocumentUrl` to create the full web URL for the document based on its relative path.
    *   **`const textChunks = await this.splitContent(markdownContent)`**: This is a critical step. It calls `splitContent` to break the `markdownContent` into an array of strings (the actual text chunks), applying all the specific chunking, cleaning, table-handling, and size enforcement logic.
    *   **`logger.info(...)`**: Logs that embeddings are being generated.
    *   **`const embeddings = textChunks.length > 0 ? await generateEmbeddings(textChunks) : []`**: Calls the external `generateEmbeddings` function, passing all the `textChunks` to process them in a batch. If there are no chunks, it returns an empty array.
    *   **`const embeddingModel = 'text-embedding-3-small'`**: Defines the name of the embedding model used, which is included in the `DocChunk` metadata.
    *   **`const chunks: DocChunk[] = []`**: Initializes an empty array to store the final `DocChunk` objects.
    *   **`let currentPosition = 0`**: A variable to keep track of the current character position in the original `markdownContent` as chunks are processed, helping to determine `startIndex` and `endIndex` for each chunk.
    *   **`for (let i = 0; i < textChunks.length; i++) { ... }`**: Loops through each `textChunk` generated by `splitContent`.
        *   **`const chunkText = textChunks[i]`**: Gets the current text content of the chunk.
        *   **`const chunkStart = currentPosition`**: Records the starting character position of the current chunk.
        *   **`const chunkEnd = currentPosition + chunkText.length`**: Records the ending character position.
        *   **`const relevantHeader = this.findRelevantHeader(headers, chunkStart)`**: Calls `findRelevantHeader` to find the header that immediately precedes or contains the start of the current `chunkText`. This provides contextual information.
        *   **`const chunk: DocChunk = { ... }`**: Creates a `DocChunk` object for the current text chunk.
            *   **`text: chunkText`**: The actual text content of the chunk.
            *   **`tokenCount: Math.ceil(chunkText.length / 4)`**: A simple estimation of the token count (roughly 4 characters per token).
            *   **`sourceDocument: relativePath`**: The relative path to the original MDX file.
            *   **`headerLink: relevantHeader ? \`${documentUrl}#${relevantHeader.anchor}\` : documentUrl`**: Constructs a deep link. If a `relevantHeader` is found, it appends its anchor to the `documentUrl`; otherwise, it just links to the document root.
            *   **`headerText: relevantHeader?.text || frontmatter.title || 'Document Root'`**: The text of the `relevantHeader`. If no header is relevant, it falls back to the document's `frontmatter.title`, or finally to 'Document Root'.
            *   **`headerLevel: relevantHeader?.level || 1`**: The level of the `relevantHeader`. Defaults to 1 (root level) if no header is relevant.
            *   **`embedding: embeddings[i] || []`**: The numerical embedding vector for this specific chunk, retrieved from the `embeddings` array generated earlier. Defaults to an empty array if no embedding is found.
            *   **`embeddingModel`**: The name of the embedding model used.
            *   **`metadata: { ... }`**: Additional metadata.
                *   **`startIndex: chunkStart`**: The starting character index of the chunk in the original document.
                *   **`endIndex: chunkEnd`**: The ending character index.
                *   **`title: frontmatter.title`**: The document's title from the frontmatter.
        *   **`chunks.push(chunk)`**: Adds the newly created `DocChunk` to the `chunks` array.
        *   **`currentPosition = chunkEnd`**: Updates `currentPosition` to the end of the current chunk for the next iteration.
    *   **`return chunks`**: Returns the array of fully constructed `DocChunk` objects for the file.

```typescript
  /**
   * Find all .mdx files recursively
   */
  private async findMdxFiles(dirPath: string): Promise<string[]> {
    const files: string[] = []

    const entries = await fs.readdir(dirPath, { withFileTypes: true })

    for (const entry of entries) {
      const fullPath = path.join(dirPath, entry.name)

      if (entry.isDirectory()) {
        const subFiles = await this.findMdxFiles(fullPath)
        files.push(...subFiles)
      } else if (entry.isFile() && entry.name.endsWith('.mdx')) {
        files.push(fullPath)
      }
    }

    return files
  }
```

*   **`private async findMdxFiles(dirPath: string): Promise<string[]>`**: An asynchronous private helper method that recursively searches a directory for `.mdx` files. It takes a `dirPath` and returns a Promise resolving to an array of full file paths.
    *   **`const files: string[] = []`**: Initializes an empty array to store the paths of found MDX files.
    *   **`const entries = await fs.readdir(dirPath, { withFileTypes: true })`**: Reads the contents of the `dirPath`. `withFileTypes: true` makes `entries` an array of `Dirent` objects, which include `isDirectory()` and `isFile()` methods.
    *   **`for (const entry of entries) { ... }`**: Loops through each entry (file or directory) found in the current directory.
        *   **`const fullPath = path.join(dirPath, entry.name)`**: Constructs the full path for the current entry.
        *   **`if (entry.isDirectory()) { ... }`**: If the entry is a directory:
            *   **`const subFiles = await this.findMdxFiles(fullPath)`**: Recursively calls `findMdxFiles` for the subdirectory.
            *   **`files.push(...subFiles)`**: Adds all `.mdx` files found in the subdirectory to the `files` array.
        *   **`else if (entry.isFile() && entry.name.endsWith('.mdx')) { ... }`**: If the entry is a file and its name ends with `.mdx`:
            *   **`files.push(fullPath)`**: Adds its full path to the `files` array.
    *   **`return files`**: Returns the array of all found MDX file paths.

```typescript
  /**
   * Extract headers and their positions from markdown content
   */
  private extractHeaders(content: string): HeaderInfo[] {
    const headers: HeaderInfo[] = []
    const headerRegex = /^(#{1,6})\s+(.+)$/gm
    let match

    while ((match = headerRegex.exec(content)) !== null) {
      const level = match[1].length
      const text = match[2].trim()
      const anchor = this.generateAnchor(text)

      headers.push({
        text,
        level,
        anchor,
        position: match.index,
      })
    }

    return headers
  }
```

*   **`private extractHeaders(content: string): HeaderInfo[]`**: A private helper method that extracts markdown headers from a given `content` string. It returns an array of `HeaderInfo` objects.
    *   **`const headers: HeaderInfo[] = []`**: Initializes an empty array to store extracted header information.
    *   **`const headerRegex = /^(#{1,6})\s+(.+)$/gm`**: Defines a regular expression to match markdown headers:
        *   **`^`**: Matches the start of a line.
        *   **`(#{1,6})`**: Captures one to six `#` characters (the header level).
        *   **`\s+`**: Matches one or more whitespace characters.
        *   **`(.+)`**: Captures the rest of the line (the header text).
        *   **`$`**: Matches the end of a line.
        *   **`g`**: Global flag, to find all matches, not just the first.
        *   **`m`**: Multiline flag, to make `^` and `$` match start/end of lines.
    *   **`while ((match = headerRegex.exec(content)) !== null) { ... }`**: Loops as long as the `headerRegex` finds matches in the `content`.
        *   **`const level = match[1].length`**: The number of `#` characters determines the header level.
        *   **`const text = match[2].trim()`**: The captured text of the header, with leading/trailing whitespace removed.
        *   **`const anchor = this.generateAnchor(text)`**: Calls `generateAnchor` to create a URL-friendly anchor string from the header text.
        *   **`headers.push({ ... })`**: Adds a new `HeaderInfo` object to the `headers` array, including the extracted text, level, generated anchor, and the `match.index` (the character position where the header was found).
    *   **`return headers`**: Returns the array of `HeaderInfo` objects.

```typescript
  /**
   * Generate URL-safe anchor from header text
   */
  private generateAnchor(headerText: string): string {
    return headerText
      .toLowerCase()
      .replace(/[^\w\s-]/g, '') // Remove special characters except hyphens
      .replace(/\s+/g, '-') // Replace spaces with hyphens
      .replace(/-+/g, '-') // Replace multiple hyphens with single
      .replace(/^-|-$/g, '') // Remove leading/trailing hyphens
  }
```

*   **`private generateAnchor(headerText: string): string`**: A private helper method to convert a header's text into a URL-safe anchor (slug).
    *   **`.toLowerCase()`**: Converts the header text to lowercase.
    *   **`.replace(/[^\w\s-]/g, '')`**: Removes any characters that are not a word character (`\w`), whitespace (`\s`), or a hyphen (`-`).
    *   **`.replace(/\s+/g, '-')`**: Replaces one or more whitespace characters with a single hyphen.
    *   **`.replace(/-+/g, '-')`**: Replaces multiple consecutive hyphens with a single hyphen.
    *   **`.replace(/^-|-$/g, '')`**: Removes any leading or trailing hyphens.

```typescript
  /**
   * Generate document URL from relative path
   */
  private generateDocumentUrl(relativePath: string): string {
    // Convert file path to URL path
    // e.g., "tools/knowledge.mdx" -> "/tools/knowledge"
    const urlPath = relativePath.replace(/\.mdx$/, '').replace(/\\/g, '/') // Handle Windows paths

    return `${this.baseUrl}/${urlPath}`
  }
```

*   **`private generateDocumentUrl(relativePath: string): string`**: A private helper method to construct the full web URL for a document.
    *   **`const urlPath = relativePath.replace(/\.mdx$/, '').replace(/\\/g, '/')`**:
        *   **`.replace(/\.mdx$/, '')`**: Removes the `.mdx` file extension from the end of the path.
        *   **`.replace(/\\/g, '/')`**: Replaces any backslashes (`\`, common in Windows paths) with forward slashes (`/`) to ensure a consistent URL format.
    *   **`return \`${this.baseUrl}/${urlPath}\``**: Combines the `baseUrl` (from the constructor) with the `urlPath` to form the complete document URL.

```typescript
  /**
   * Find the most relevant header for a given position
   */
  private findRelevantHeader(headers: HeaderInfo[], position: number): HeaderInfo | null {
    if (headers.length === 0) return null

    // Find the last header that comes before this position
    let relevantHeader: HeaderInfo | null = null

    for (const header of headers) {
      if (header.position !== undefined && header.position <= position) {
        relevantHeader = header
      } else {
        // Headers are typically sorted by position, so we can break early
        break
      }
    }

    return relevantHeader
  }
```

*   **`private findRelevantHeader(headers: HeaderInfo[], position: number): HeaderInfo | null`**: A private helper method to find the most contextually relevant header for a given character `position` within the document. It returns a `HeaderInfo` object or `null` if no header is found.
    *   **`if (headers.length === 0) return null`**: If there are no headers in the document, return `null`.
    *   **`let relevantHeader: HeaderInfo | null = null`**: Initializes a variable to store the found relevant header.
    *   **`for (const header of headers) { ... }`**: Iterates through the `headers` array (assumed to be sorted by `position`).
        *   **`if (header.position !== undefined && header.position <= position)`**: If the header has a defined position and that position is less than or equal to the target `position` (meaning the header appears before or at the target position):
            *   **`relevantHeader = header`**: This header becomes the `relevantHeader`.
        *   **`else { break }`**: If a header's position is *after* the target `position`, it means all subsequent headers will also be after, so we can stop searching.
    *   **`return relevantHeader`**: Returns the last header found that was before or at the given `position`.

```typescript
  /**
   * Split content into chunks using the existing TextChunker with table awareness
   */
  private async splitContent(content: string): Promise<string[]> {
    // Clean the content first
    const cleanedContent = this.cleanContent(content)

    // Detect table boundaries to avoid splitting them
    const tableBoundaries = this.detectTableBoundaries(cleanedContent)

    // Use the existing TextChunker
    const chunks = await this.textChunker.chunk(cleanedContent)

    // Post-process chunks to ensure tables aren't split
    const processedChunks = this.mergeTableChunks(
      chunks.map((chunk) => chunk.text), // Extract text from TextChunker's output
      tableBoundaries,
      cleanedContent
    )

    // Ensure no chunk exceeds 300 tokens
    const finalChunks = this.enforceSizeLimit(processedChunks)

    return finalChunks
  }
```

*   **`private async splitContent(content: string): Promise<string[]>`**: An asynchronous private method that orchestrates the chunking process for markdown content, incorporating table awareness. It returns a Promise resolving to an array of cleaned and chunked strings.
    *   **`const cleanedContent = this.cleanContent(content)`**: Calls `cleanContent` to remove MDX-specific syntax and excessive whitespace from the raw markdown.
    *   **`const tableBoundaries = this.detectTableBoundaries(cleanedContent)`**: Calls `detectTableBoundaries` to identify the start and end character positions of all markdown tables within the `cleanedContent`.
    *   **`const chunks = await this.textChunker.chunk(cleanedContent)`**: Uses the generic `TextChunker` (initialized in the constructor) to perform an initial chunking of the `cleanedContent`. Note that `textChunker.chunk` returns objects with a `text` property, so `chunks.map((chunk) => chunk.text)` is used later.
    *   **`const processedChunks = this.mergeTableChunks(...)`**: Calls `mergeTableChunks` to post-process the `chunks`. This step ensures that any tables that were accidentally split by the initial chunking are merged back into single chunks.
        *   **`chunks.map((chunk) => chunk.text)`**: Extracts just the `text` string from each chunk object returned by `textChunker.chunk`.
        *   **`tableBoundaries`**: The detected table locations are passed to guide the merging.
        *   **`cleanedContent`**: The original cleaned content is passed to allow precise slicing for merging.
    *   **`const finalChunks = this.enforceSizeLimit(processedChunks)`**: Calls `enforceSizeLimit` to ensure that even after merging (which might create larger chunks), no chunk exceeds the maximum desired token count (e.g., 300 tokens). If a chunk is too big, it's intelligently re-split.
    *   **`return finalChunks`**: Returns the final array of processed and sized chunks.

```typescript
  /**
   * Clean content by removing MDX-specific elements and excessive whitespace
   */
  private cleanContent(content: string): string {
    return (
      content
        // Remove import statements
        .replace(/^import\s+.*$/gm, '')
        // Remove JSX components and React-style comments
        .replace(/<[^>]+>/g, ' ')
        .replace(/\{\/\*[\s\S]*?\*\/\}/g, ' ')
        // Remove excessive whitespace
        .replace(/\n{3,}/g, '\n\n')
        .replace(/[ \t]{2,}/g, ' ')
        .trim()
    )
  }
```

*   **`private cleanContent(content: string): string`**: A private helper method to clean the markdown content by removing MDX-specific syntax and normalizing whitespace.
    *   **`.replace(/^import\s+.*$/gm, '')`**: Removes entire lines that start with `import` (MDX import statements). `^` and `$` combined with `m` (multiline) flag match start/end of lines.
    *   **`.replace(/<[^>]+>/g, ' ')`**: Removes JSX tags (like `<Component />` or `<div>`). It replaces them with a space to avoid merging words.
    *   **`.replace(/\{\/\*[\s\S]*?\*\/\}/g, ' ')`**: Removes React-style comments (`{/* comment */}`). `[\s\S]*?` matches any character (including newlines) non-greedily.
    *   **`.replace(/\n{3,}/g, '\n\n')`**: Replaces three or more consecutive newline characters with exactly two newlines, normalizing paragraph spacing.
    *   **`.replace(/[ \t]{2,}/g, ' ')`**: Replaces two or more consecutive spaces or tabs with a single space, normalizing horizontal spacing.
    *   **`.trim()`**: Removes leading and trailing whitespace from the entire cleaned content.

```typescript
  /**
   * Parse frontmatter from MDX content
   */
  private parseFrontmatter(content: string): { data: Frontmatter; content: string } {
    const frontmatterRegex = /^---\r?\n([\s\S]*?)\r?\n---\r?\n([\s\S]*)$/
    const match = content.match(frontmatterRegex)

    if (!match) {
      return { data: {}, content }
    }

    const [, frontmatterText, markdownContent] = match
    const data: Frontmatter = {}

    // Simple YAML parsing for title and description
    const lines = frontmatterText.split('\n')
    for (const line of lines) {
      const colonIndex = line.indexOf(':')
      if (colonIndex > 0) {
        const key = line.slice(0, colonIndex).trim()
        const value = line
          .slice(colonIndex + 1)
          .trim()
          .replace(/^['"]|['"]$/g, '')
        data[key] = value
      }
    }

    return { data, content: markdownContent }
  }
```

*   **`private parseFrontmatter(content: string): { data: Frontmatter; content: string }`**: A private helper method to parse YAML frontmatter from the beginning of a document. It returns an object containing the parsed `Frontmatter` data and the remaining markdown `content`.
    *   **`const frontmatterRegex = /^---\r?\n([\s\S]*?)\r?\n---\r?\n([\s\S]*)$/`**: Defines a regular expression to find frontmatter:
        *   **`^---\r?\n`**: Matches `---` at the very beginning of the string, followed by an optional carriage return and a newline.
        *   **`([\s\S]*?)`**: Captures any characters (including newlines) non-greedily, which is the frontmatter YAML content.
        *   **`\r?\n---\r?\n`**: Matches the closing `---` block.
        *   **`([\s\S]*)`**: Captures all remaining content, which is the actual markdown.
    *   **`const match = content.match(frontmatterRegex)`**: Attempts to match the regex against the `content`.
    *   **`if (!match) { return { data: {}, content } }`**: If no frontmatter is found, return an empty `data` object and the original `content`.
    *   **`const [, frontmatterText, markdownContent] = match`**: Destructures the `match` array. `match[0]` is the full match, `match[1]` is the captured frontmatter text, and `match[2]` is the captured markdown content.
    *   **`const data: Frontmatter = {}`**: Initializes an empty object to store the parsed frontmatter key-value pairs.
    *   **`const lines = frontmatterText.split('\n')`**: Splits the frontmatter text into individual lines.
    *   **`for (const line of lines) { ... }`**: Iterates through each line of the frontmatter.
        *   **`const colonIndex = line.indexOf(':')`**: Finds the first colon, which separates the key from the value.
        *   **`if (colonIndex > 0) { ... }`**: If a colon is found (and not at the beginning of the line):
            *   **`const key = line.slice(0, colonIndex).trim()`**: Extracts the key before the colon and trims whitespace.
            *   **`const value = line.slice(colonIndex + 1).trim().replace(/^['"]|['"]$/g, '')`**: Extracts the value after the colon, trims whitespace, and removes any surrounding single or double quotes.
            *   **`data[key] = value`**: Adds the key-value pair to the `data` object.
    *   **`return { data, content: markdownContent }`**: Returns the parsed frontmatter `data` and the remaining `markdownContent`.

```typescript
  /**
   * Estimate token count (rough approximation)
   */
  private estimateTokens(text: string): number {
    // Rough approximation: 1 token ≈ 4 characters
    return Math.ceil(text.length / 4)
  }
```

*   **`private estimateTokens(text: string): number`**: A private helper method that provides a rough estimation of the token count for a given string.
    *   **`return Math.ceil(text.length / 4)`**: Divides the character length of the text by 4 and rounds up. This is a common heuristic for English text (e.g., in OpenAI's tokenizers).

```typescript
  /**
   * Detect table boundaries in markdown content to avoid splitting them
   */
  private detectTableBoundaries(content: string): { start: number; end: number }[] {
    const tables: { start: number; end: number }[] = []
    const lines = content.split('\n')

    let inTable = false
    let tableStart = -1

    for (let i = 0; i < lines.length; i++) {
      const line = lines[i].trim()

      // Detect table start (markdown table row with pipes)
      if (line.includes('|') && line.split('|').length >= 3 && !inTable) {
        // Check if next line is table separator (contains dashes and pipes)
        const nextLine = lines[i + 1]?.trim()
        if (nextLine?.includes('|') && nextLine.includes('-')) {
          inTable = true
          tableStart = i
        }
      }
      // Detect table end (empty line or non-table content)
      else if (inTable && (!line.includes('|') || line === '' || line.startsWith('#'))) {
        tables.push({
          start: this.getCharacterPosition(lines, tableStart),
          end: this.getCharacterPosition(lines, i - 1) + lines[i - 1]?.length || 0,
        })
        inTable = false
      }
    }

    // Handle table at end of content
    if (inTable && tableStart >= 0) {
      tables.push({
        start: this.getCharacterPosition(lines, tableStart),
        end: content.length,
      })
    }

    return tables
  }
```

*   **`private detectTableBoundaries(content: string): { start: number; end: number }[]`**: A private helper method to identify the character start and end positions of markdown tables within the `content`.
    *   **`const tables: { start: number; end: number }[] = []`**: Initializes an array to store the boundaries of detected tables.
    *   **`const lines = content.split('\n')`**: Splits the content into an array of lines for easier processing.
    *   **`let inTable = false`**: A boolean flag to track whether the parser is currently inside a table block.
    *   **`let tableStart = -1`**: Stores the line index where the current table began.
    *   **`for (let i = 0; i < lines.length; i++) { ... }`**: Loops through each line of the content.
        *   **`const line = lines[i].trim()`**: Gets the current line, trimmed of whitespace.
        *   **`if (line.includes('|') && line.split('|').length >= 3 && !inTable)`**: This condition tries to detect the *start* of a table:
            *   `line.includes('|')`: The line contains pipes (typical for markdown tables).
            *   `line.split('|').length >= 3`: There are at least two pipe-separated columns (e.g., `| Col1 | Col2 |`).
            *   `!inTable`: Ensures we are not already inside a table.
            *   **`const nextLine = lines[i + 1]?.trim()`**: Looks at the next line (if it exists).
            *   **`if (nextLine?.includes('|') && nextLine.includes('-')) { ... }`**: If the next line is a table separator (e.g., `|---|---|`):
                *   **`inTable = true`**: Sets the flag to `true`, indicating we are now inside a table.
                *   **`tableStart = i`**: Records the current line index as the start of the table.
        *   **`else if (inTable && (!line.includes('|') || line === '' || line.startsWith('#')))`**: This condition tries to detect the *end* of a table:
            *   `inTable`: We are currently inside a table.
            *   `!line.includes('|') || line === '' || line.startsWith('#')`: The current line no longer looks like a table row (no pipes, or it's empty, or it's a heading, indicating the end of the table).
            *   **`tables.push(...)`**: Records the boundaries of the just-ended table:
                *   **`start: this.getCharacterPosition(lines, tableStart)`**: Uses `getCharacterPosition` to convert the starting line index to a character position.
                *   **`end: this.getCharacterPosition(lines, i - 1) + lines[i - 1]?.length || 0`**: Calculates the end character position by taking the character position of the *previous* line (the last table line) and adding its length.
            *   **`inTable = false`**: Resets the `inTable` flag.
    *   **`if (inTable && tableStart >= 0) { ... }`**: Handles the case where a table is the very last content in the document and doesn't end with a non-table line.
        *   **`tables.push(...)`**: Records its boundaries, with the end being the full `content.length`.
    *   **`return tables`**: Returns the array of detected table boundaries.

```typescript
  /**
   * Get character position from line number
   */
  private getCharacterPosition(lines: string[], lineIndex: number): number {
    return lines.slice(0, lineIndex).reduce((acc, line) => acc + line.length + 1, 0)
  }
```

*   **`private getCharacterPosition(lines: string[], lineIndex: number): number`**: A private helper method to calculate the starting character index of a given `lineIndex` within an array of `lines`.
    *   **`lines.slice(0, lineIndex)`**: Gets all lines *before* the target `lineIndex`.
    *   **`.reduce((acc, line) => acc + line.length + 1, 0)`**: Iterates through these preceding lines:
        *   `acc`: The accumulated character count.
        *   `line.length`: The length of the current line.
        *   `+ 1`: Accounts for the newline character (`\n`) that separates each line.
        *   `0`: The initial value of `acc`.
    *   The result is the total character count up to the start of the specified `lineIndex`.

```typescript
  /**
   * Merge chunks that would split tables
   */
  private mergeTableChunks(
    chunks: string[],
    tableBoundaries: { start: number; end: number }[],
    originalContent: string
  ): string[] {
    if (tableBoundaries.length === 0) {
      return chunks
    }

    const mergedChunks: string[] = []
    let currentPosition = 0 // Tracks position in originalContent to find chunk locations

    for (const chunk of chunks) {
      const chunkStart = originalContent.indexOf(chunk, currentPosition)
      const chunkEnd = chunkStart + chunk.length

      // Check if this chunk intersects with any table
      const intersectsTable = tableBoundaries.some(
        (table) =>
          (chunkStart >= table.start && chunkStart <= table.end) || // Chunk starts within table
          (chunkEnd >= table.start && chunkEnd <= table.end) || // Chunk ends within table
          (chunkStart <= table.start && chunkEnd >= table.end) // Chunk entirely spans table
      )

      if (intersectsTable) {
        // Find which table(s) this chunk intersects with
        const affectedTables = tableBoundaries.filter(
          (table) =>
            (chunkStart >= table.start && chunkStart <= table.end) ||
            (chunkEnd >= table.start && chunkEnd <= table.end) ||
            (chunkStart <= table.start && chunkEnd >= table.end)
        )

        // Create a chunk that includes the complete table(s)
        const minStart = Math.min(chunkStart, ...affectedTables.map((t) => t.start))
        const maxEnd = Math.max(chunkEnd, ...affectedTables.map((t) => t.end))
        const completeChunk = originalContent.slice(minStart, maxEnd)

        // Only add if we haven't already included this content
        if (!mergedChunks.some((existing) => existing.includes(completeChunk.trim()))) {
          mergedChunks.push(completeChunk.trim())
        }
      } else {
        mergedChunks.push(chunk)
      }

      currentPosition = chunkEnd
    }

    return mergedChunks.filter((chunk) => chunk.length > 50) // Filter out tiny chunks
  }
```

*   **`private mergeTableChunks(...)`**: A private helper method responsible for merging text chunks that might have inadvertently split markdown tables. It takes the initial `chunks` (as strings), `tableBoundaries`, and the `originalContent` (cleaned markdown) to find exact positions.
    *   **`if (tableBoundaries.length === 0) { return chunks }`**: If there are no tables, no merging is needed, so return the original chunks.
    *   **`const mergedChunks: string[] = []`**: Initializes an array to store the chunks after merging.
    *   **`let currentPosition = 0`**: Keeps track of where the last chunk was found in `originalContent` to assist `indexOf`.
    *   **`for (const chunk of chunks) { ... }`**: Iterates through each of the initially generated chunks.
        *   **`const chunkStart = originalContent.indexOf(chunk, currentPosition)`**: Finds the starting character position of the current `chunk` within the `originalContent`, starting the search from `currentPosition`.
        *   **`const chunkEnd = chunkStart + chunk.length`**: Calculates the ending character position of the chunk.
        *   **`const intersectsTable = tableBoundaries.some(...)`**: Checks if the current chunk's boundaries (`chunkStart`, `chunkEnd`) overlap with *any* of the `tableBoundaries`. The `some` method returns `true` if at least one table boundary overlaps based on three conditions: chunk starts in table, chunk ends in table, or chunk completely spans a table.
        *   **`if (intersectsTable) { ... }`**: If the chunk intersects with a table:
            *   **`const affectedTables = tableBoundaries.filter(...)`**: Filters the `tableBoundaries` to find all tables that specifically intersect with this `chunk`.
            *   **`const minStart = Math.min(chunkStart, ...affectedTables.map((t) => t.start))`**: Finds the earliest starting point among the current chunk and all affected tables.
            *   **`const maxEnd = Math.max(chunkEnd, ...affectedTables.map((t) => t.end))`**: Finds the latest ending point among the current chunk and all affected tables.
            *   **`const completeChunk = originalContent.slice(minStart, maxEnd)`**: Creates a new chunk by slicing the `originalContent` from `minStart` to `maxEnd`, ensuring it contains the full table(s) and the surrounding text.
            *   **`if (!mergedChunks.some((existing) => existing.includes(completeChunk.trim()))) { ... }`**: This check prevents adding duplicate content if a previous merge already included this `completeChunk`.
            *   **`mergedChunks.push(completeChunk.trim())`**: Adds the newly formed, table-aware chunk to `mergedChunks`.
        *   **`else { mergedChunks.push(chunk) }`**: If the chunk does *not* intersect with any table, it's added directly to `mergedChunks`.
        *   **`currentPosition = chunkEnd`**: Updates `currentPosition` for the next iteration.
    *   **`return mergedChunks.filter((chunk) => chunk.length > 50)`**: Returns the final array of merged chunks, filtering out any chunks that are very small (less than 50 characters) as they are unlikely to be meaningful.

```typescript
  /**
   * Enforce 300 token size limit on chunks
   */
  private enforceSizeLimit(chunks: string[]): string[] {
    const finalChunks: string[] = []

    for (const chunk of chunks) {
      const tokens = this.estimateTokens(chunk)

      if (tokens <= 300) {
        // Chunk is within limit
        finalChunks.push(chunk)
      } else {
        // Chunk is too large - split it
        const lines = chunk.split('\n')
        let currentChunk = ''

        for (const line of lines) {
          const testChunk = currentChunk ? `${currentChunk}\n${line}` : line

          if (this.estimateTokens(testChunk) <= 300) {
            currentChunk = testChunk
          } else {
            // Adding this line would exceed limit
            if (currentChunk.trim()) {
              finalChunks.push(currentChunk.trim())
            }
            currentChunk = line
          }
        }

        // Add final chunk if it has content
        if (currentChunk.trim()) {
          finalChunks.push(currentChunk.trim())
        }
      }
    }

    return finalChunks.filter((chunk) => chunk.trim().length > 100)
  }
}
```

*   **`private enforceSizeLimit(chunks: string[]): string[]`**: A private helper method to ensure that all chunks adhere to a maximum token count (e.g., 300 tokens). This is particularly important after `mergeTableChunks` might have created oversized chunks.
    *   **`const finalChunks: string[] = []`**: Initializes an array to store the size-compliant chunks.
    *   **`for (const chunk of chunks) { ... }`**: Iterates through each `chunk` (which might be oversized from table merging).
        *   **`const tokens = this.estimateTokens(chunk)`**: Estimates the token count for the current chunk.
        *   **`if (tokens <= 300) { ... }`**: If the chunk is already within the limit:
            *   **`finalChunks.push(chunk)`**: Add it directly to `finalChunks`.
        *   **`else { ... }`**: If the chunk is too large:
            *   **`const lines = chunk.split('\n')`**: Splits the oversized chunk into individual lines.
            *   **`let currentChunk = ''`**: Initializes a temporary string to build new, smaller chunks line by line.
            *   **`for (const line of lines) { ... }`**: Iterates through each `line` of the oversized chunk.
                *   **`const testChunk = currentChunk ? \`${currentChunk}\n${line}\` : line`**: Constructs a potential next chunk by adding the current `line` to `currentChunk`.
                *   **`if (this.estimateTokens(testChunk) <= 300) { ... }`**: If adding this `line` keeps the `testChunk` within the token limit:
                    *   **`currentChunk = testChunk`**: Update `currentChunk` to include the `line`.
                *   **`else { ... }`**: If adding this `line` would exceed the limit:
                    *   **`if (currentChunk.trim()) { finalChunks.push(currentChunk.trim()) }`**: If `currentChunk` has content, push it to `finalChunks` as a complete chunk.
                    *   **`currentChunk = line`**: Start a new `currentChunk` with the `line` that caused the overflow.
            *   **`if (currentChunk.trim()) { finalChunks.push(currentChunk.trim()) }`**: After the loop, if `currentChunk` still holds content, add it as the last chunk for this oversized original chunk.
    *   **`return finalChunks.filter((chunk) => chunk.trim().length > 100)`**: Returns the final array of chunks, applying a final filter to remove any chunks that are too small (less than 100 characters) after all splitting and merging, ensuring meaningful chunks.