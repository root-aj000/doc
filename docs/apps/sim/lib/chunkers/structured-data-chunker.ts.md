This TypeScript file defines a powerful and intelligent **`StructuredDataChunker`** class. Its primary purpose is to take large blocks of structured text data (like the content of a CSV file, a tab-separated value file, or data extracted from a spreadsheet like XLSX) and break it down into smaller, more manageable "chunks."

These chunks are specifically designed to be highly effective when used with Large Language Models (LLMs) or for generating embeddings. The core idea is to preserve the *semantic meaning* of the data within each chunk, ensure chunks are within a desired token count range, and provide useful context.

---

### **1. Configuration for Structured Data Chunking (`STRUCTURED_CHUNKING_CONFIG`)**

This section sets up important parameters that guide how the data will be split. Think of these as the "rules" the chunker follows.

```typescript
const STRUCTURED_CHUNKING_CONFIG = {
  // Target 2000-3000 tokens per chunk for better semantic meaning
  TARGET_CHUNK_SIZE: 2500, // The ideal number of tokens we aim for in each chunk. This is a sweet spot for many LLMs.
  MIN_CHUNK_SIZE: 500,     // The smallest acceptable token count for a chunk. Prevents very tiny, uninformative chunks.
  MAX_CHUNK_SIZE: 4000,    // The absolute maximum token count a chunk should have. Essential to avoid exceeding LLM context windows.

  // For spreadsheets, group rows together
  ROWS_PER_CHUNK: 100,      // A default starting point for the number of rows to include in a chunk.
  MIN_ROWS_PER_CHUNK: 20,   // The minimum number of rows required before a chunk can be considered "full" and cut.
  MAX_ROWS_PER_CHUNK: 500,  // The maximum number of rows to include in a single chunk, regardless of token count.

  // For better embeddings quality
  INCLUDE_HEADERS_IN_EACH_CHUNK: true, // If true, the header row will be prepended to every chunk's content for context. This greatly improves semantic understanding.
  MAX_HEADER_SIZE: 200,                // A token limit for the header itself. If headers are too long, they might be truncated (though not implemented in this snippet, it's a common consideration).
}
```

---

### **2. `StructuredDataChunker` Class**

This is the main class that encapsulates all the logic for chunking structured data. It's designed with static methods, meaning you don't need to create an instance of the class (`new StructuredDataChunker()`) to use its functions; you can call them directly (e.g., `StructuredDataChunker.chunkStructuredData(...)`).

```typescript
export class StructuredDataChunker {
  // ... methods inside ...
}
```

---

### **3. Core Logic: `static async chunkStructuredData(...)`**

This is the central method that performs the intelligent chunking. It takes raw structured data as a string and returns an array of `Chunk` objects.

```typescript
  /**
   * Chunk structured data intelligently based on rows and semantic boundaries
   * @param content The raw structured data string (e.g., CSV content)
   * @param options Configuration options, like explicit headers or sheet name
   * @returns A Promise resolving to an array of Chunk objects
   */
  static async chunkStructuredData(
    content: string,
    options: StructuredDataOptions = {} // `options` is an object to customize chunking, defaulting to an empty object if not provided.
  ): Promise<Chunk[]> {
    const chunks: Chunk[] = [] // Initialize an empty array to store the resulting chunks.

    // Split the entire content string into individual lines (rows).
    // `.filter((line) => line.trim())` removes any completely empty lines to ensure only meaningful data is processed.
    const lines = content.split('\n').filter((line) => line.trim())

    // If there are no lines after filtering (empty content), return an empty array of chunks immediately.
    if (lines.length === 0) {
      return chunks
    }

    // --- Header Detection ---
    // Determine the header line.
    // If `options.headers` is provided (an array of strings), it's joined with a tab character to form the header string.
    // Otherwise, the very first line (`lines[0]`) from the input content is assumed to be the header.
    const headerLine = options.headers?.join('\t') || lines[0]

    // Determine where the actual data rows begin.
    // If `options.headers` were provided, it means the input `content` itself *only* contains data, so data starts at index 0.
    // If headers were *not* provided in options (meaning `lines[0]` was used as header), then data starts from the second line (index 1).
    const dataStartIndex = options.headers ? 0 : 1

    // --- Calculate Optimal Rows Per Chunk (Dynamic Sizing) ---
    // This makes the chunker adaptive to different data densities.
    // It estimates the average number of tokens in a row by sampling the first few data rows (up to 10 or `lines.length`).
    const estimatedTokensPerRow = StructuredDataChunker.estimateTokensPerRow(
      lines.slice(dataStartIndex, Math.min(10, lines.length))
    )

    // Based on the `estimatedTokensPerRow`, calculate an `optimalRowsPerChunk` to hit the `TARGET_CHUNK_SIZE`.
    // This allows the chunker to create chunks that are closer to the target token size while preserving row integrity.
    const optimalRowsPerChunk =
      StructuredDataChunker.calculateOptimalRowsPerChunk(estimatedTokensPerRow)

    // Log the chunking strategy being used for transparency.
    console.log(
      `Structured data chunking: ${lines.length} rows, ~${estimatedTokensPerRow} tokens/row, ${optimalRowsPerChunk} rows/chunk`
    )

    // --- Iterative Chunk Creation Loop ---
    let currentChunkRows: string[] = []       // Stores rows for the chunk currently being built.
    let currentTokenEstimate = 0              // Tracks the estimated token count for `currentChunkRows`.
    // Pre-calculate header tokens if they will be included in each chunk. This avoids recalculating in every loop iteration.
    const headerTokens = StructuredDataChunker.estimateTokens(headerLine)
    let chunkStartRow = dataStartIndex        // Keeps track of the original line number where the current chunk started.

    // Loop through all data lines, starting from `dataStartIndex`.
    for (let i = dataStartIndex; i < lines.length; i++) {
      const row = lines[i]                      // Get the current row.
      const rowTokens = StructuredDataChunker.estimateTokens(row) // Estimate tokens for *this specific* row.

      // Calculate the projected token count if the current `row` were added to `currentChunkRows`.
      // It also accounts for the header tokens if `INCLUDE_HEADERS_IN_EACH_CHUNK` is true.
      const projectedTokens =
        currentTokenEstimate +
        rowTokens +
        (STRUCTURED_CHUNKING_CONFIG.INCLUDE_HEADERS_IN_EACH_CHUNK ? headerTokens : 0)

      // --- Complex Logic: Deciding When to Create a New Chunk ---
      // This condition determines if the accumulated `currentChunkRows` should be finalized into a chunk.
      const shouldCreateChunk =
        // Condition 1: If adding the current row would exceed the target chunk size significantly,
        // AND we already have at least the minimum number of rows required in the current chunk.
        (projectedTokens > STRUCTURED_CHUNKING_CONFIG.TARGET_CHUNK_SIZE &&
          currentChunkRows.length >= STRUCTURED_CHUNKING_CONFIG.MIN_ROWS_PER_CHUNK) ||
        // Condition 2: If we've simply reached the dynamically calculated `optimalRowsPerChunk` (prioritizing semantic grouping).
        currentChunkRows.length >= optimalRowsPerChunk

      // If a chunk should be created (based on `shouldCreateChunk`) AND there are actual rows accumulated:
      if (shouldCreateChunk && currentChunkRows.length > 0) {
        // Format the collected rows and header into a single string content for the chunk.
        const chunkContent = StructuredDataChunker.formatChunk(
          headerLine,
          currentChunkRows,
          options.sheetName
        )
        // Create the final `Chunk` object and add it to the `chunks` array.
        // `i - 1` is used as `endRow` because `i` is the *next* row that caused the chunk to be cut.
        chunks.push(StructuredDataChunker.createChunk(chunkContent, chunkStartRow, i - 1))

        // Reset for the next chunk:
        currentChunkRows = []          // Clear rows.
        currentTokenEstimate = 0       // Reset token count.
        chunkStartRow = i              // The new chunk starts with the current row `i`.
      }

      // Add the current row to the `currentChunkRows` for the chunk being built.
      currentChunkRows.push(row)
      // Update the token estimate for the `currentChunkRows`.
      currentTokenEstimate += rowTokens
    }

    // --- Handle Remaining Rows (After Loop) ---
    // After the loop finishes, there might be some rows left that didn't quite form a full chunk.
    // These are collected into a final chunk.
    if (currentChunkRows.length > 0) {
      const chunkContent = StructuredDataChunker.formatChunk(
        headerLine,
        currentChunkRows,
        options.sheetName
      )
      // The `endRow` for the last chunk is the last line of the original content.
      chunks.push(StructuredDataChunker.createChunk(chunkContent, chunkStartRow, lines.length - 1))
    }

    // Log the total number of chunks created.
    console.log(`Created ${chunks.length} chunks from ${lines.length} rows of structured data`)

    return chunks // Return the array of all created chunks.
  }
```

---

### **4. Helper Methods within `StructuredDataChunker`**

These `private static` methods assist the main `chunkStructuredData` logic by handling specific tasks. Being `private`, they are only meant for internal use within the class.

#### **`private static formatChunk(...)`**

This method assembles the final string content for a chunk, adding context like sheet names, headers, and row counts.

```typescript
  /**
   * Format a chunk with headers and context
   * @param headerLine The header string to include.
   * @param rows An array of data rows for this specific chunk.
   * @param sheetName (Optional) The name of the sheet, for additional context.
   * @returns The fully formatted string content for the chunk.
   */
  private static formatChunk(headerLine: string, rows: string[], sheetName?: string): string {
    let content = '' // Initialize an empty string to build the chunk content.

    // Add sheet name context if available (e.g., "=== Sheet1 ===").
    if (sheetName) {
      content += `=== ${sheetName} ===\n\n`
    }

    // Add headers for context, if configured to do so.
    if (STRUCTURED_CHUNKING_CONFIG.INCLUDE_HEADERS_IN_EACH_CHUNK) {
      content += `Headers: ${headerLine}\n` // Prepend "Headers: " to the header line.
      // Add a separator line for readability, up to 80 characters long or header length.
      content += `${'-'.repeat(Math.min(80, headerLine.length))}\n`
    }

    // Join all the data rows for this chunk with newline characters.
    content += rows.join('\n')

    // Add a footer indicating the number of data rows in this chunk for context.
    content += `\n\n[Rows ${rows.length} of data]`

    return content // Return the complete string.
  }
```

#### **`private static createChunk(...)`**

This method constructs the `Chunk` object, which includes the chunk's text, its estimated token count, and metadata about its original row indices.

```typescript
  /**
   * Create a chunk object with actual row indices
   * @param content The formatted string content of the chunk.
   * @param startRow The original starting row index of this chunk in the full dataset.
   * @param endRow The original ending row index of this chunk in the full dataset.
   * @returns A `Chunk` object.
   */
  private static createChunk(content: string, startRow: number, endRow: number): Chunk {
    // Estimate the token count for the chunk's final content.
    const tokenCount = StructuredDataChunker.estimateTokens(content)

    // Return a `Chunk` object conforming to the `Chunk` type definition.
    return {
      text: content,
      tokenCount,
      metadata: {
        startIndex: startRow, // The starting row index in the original data.
        endIndex: endRow,     // The ending row index in the original data.
      },
    }
  }
```

#### **`private static estimateTokens(...)`**

A simple utility method to get a rough estimate of the number of tokens in a given text string. This is crucial for managing chunk sizes.

```typescript
  /**
   * Estimate tokens in text (rough approximation)
   * @param text The string for which to estimate tokens.
   * @returns An estimated number of tokens.
   */
  private static estimateTokens(text: string): number {
    // A common rule of thumb for English text is 1 token per 4 characters.
    // For structured data, which often contains many numbers and short strings,
    // a slightly more generous estimate of 1 token per 3 characters might be more accurate.
    return Math.ceil(text.length / 3) // `Math.ceil` ensures even partial tokens are counted up.
  }
```

#### **`private static estimateTokensPerRow(...)`**

This method calculates the average token count per row by sampling a few initial rows. This average helps the chunker adapt to different data densities.

```typescript
  /**
   * Estimate average tokens per row from a sample of rows.
   * @param sampleRows An array of string rows to use for estimation.
   * @returns The estimated average number of tokens per row.
   */
  private static estimateTokensPerRow(sampleRows: string[]): number {
    if (sampleRows.length === 0) return 50 // If no sample rows, return a default estimate of 50 tokens per row.

    // Calculate the total tokens in all sample rows.
    const totalTokens = sampleRows.reduce(
      (sum, row) => sum + StructuredDataChunker.estimateTokens(row),
      0
    )
    // Divide total tokens by the number of sample rows to get the average, rounded up.
    return Math.ceil(totalTokens / sampleRows.length)
  }
```

#### **`private static calculateOptimalRowsPerChunk(...)`**

This method dynamically determines how many rows should ideally go into a chunk based on the estimated tokens per row and the target chunk size. It also ensures this value stays within the configured minimum and maximum row limits.

```typescript
  /**
   * Calculate optimal rows per chunk based on token estimates
   * @param tokensPerRow The estimated average number of tokens per row.
   * @returns The calculated optimal number of rows per chunk.
   */
  private static calculateOptimalRowsPerChunk(tokensPerRow: number): number {
    // Calculate an initial optimal number of rows by dividing the target chunk size by tokens per row.
    const optimal = Math.floor(STRUCTURED_CHUNKING_CONFIG.TARGET_CHUNK_SIZE / tokensPerRow)

    // Ensure the `optimal` value is clamped between the `MIN_ROWS_PER_CHUNK` and `MAX_ROWS_PER_CHUNK` from config.
    // `Math.max` ensures it's at least the minimum. `Math.min` ensures it's no more than the maximum.
    return Math.min(
      Math.max(optimal, STRUCTURED_CHUNKING_CONFIG.MIN_ROWS_PER_CHUNK),
      STRUCTURED_CHUNKING_CONFIG.MAX_ROWS_PER_CHUNK
    )
  }
```

#### **`static isStructuredData(...)`**

This method is a utility to check if a given content string *appears* to be structured data, either by its MIME type or by analyzing its internal structure.

```typescript
  /**
   * Check if content appears to be structured data (e.g., CSV, XLSX data)
   * @param content The string content to check.
   * @param mimeType (Optional) The MIME type of the content, if known.
   * @returns `true` if it looks like structured data, `false` otherwise.
   */
  static isStructuredData(content: string, mimeType?: string): boolean {
    // --- 1. Check MIME type first ---
    if (mimeType) {
      const structuredMimeTypes = [
        'text/csv',                                             // Comma-separated values
        'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet', // XLSX (Excel)
        'application/vnd.ms-excel',                             // Older XLS (Excel)
        'text/tab-separated-values',                            // Tab-separated values
      ]
      // If the provided mimeType matches any known structured data type, return true.
      if (structuredMimeTypes.includes(mimeType)) {
        return true
      }
    }

    // --- 2. If no MIME type or it didn't match, check content structure ---
    // Take the first 10 lines to analyze for consistency.
    const lines = content.split('\n').slice(0, 10)
    // If there are fewer than 2 lines, it's unlikely to be structured data.
    if (lines.length < 2) return false

    // Define common delimiters found in structured data.
    const delimiters = [',', '\t', '|'] // Comma, Tab, Pipe

    // Iterate through each possible delimiter.
    for (const delimiter of delimiters) {
      // For each line, count how many times the current delimiter appears.
      const counts = lines.map(
        (line) => (line.match(new RegExp(`\\${delimiter}`, 'g')) || []).length
      )
      // Calculate the average count of the delimiter across the sample lines.
      const avgCount = counts.reduce((a, b) => a + b, 0) / counts.length

      // If the average delimiter count is greater than 2 (meaning there are multiple columns)
      // AND the delimiter count is consistent across most lines (each line's count is within +/- 2 of the average),
      // then it's highly likely structured data.
      if (avgCount > 2 && counts.every((c) => Math.abs(c - avgCount) <= 2)) {
        return true
      }
    }

    // If no consistent delimiter pattern was found, it's not considered structured data by this heuristic.
    return false
  }
```

---

### **5. Type Definitions (`Chunk`, `StructuredDataOptions`)**

At the very top, you see an import statement:

```typescript
import type { Chunk, StructuredDataOptions } from './types'
```

This line brings in TypeScript `type` definitions from a separate file named `types.ts` (or `types/index.ts`). These types are crucial for ensuring type safety and clarity in the code:

*   **`Chunk`**: This type likely defines the structure of the objects returned by `chunkStructuredData`. It probably includes properties like `text` (the chunk's content), `tokenCount`, and `metadata` (containing information like `startIndex` and `endIndex`).
*   **`StructuredDataOptions`**: This type defines the expected structure of the `options` object that can be passed to `chunkStructuredData`. It would specify properties like `headers` (an array of header strings) and `sheetName` (a string).

---

### **Summary of Complex Logic Simplification**

The most intricate part of this code is how `chunkStructuredData` decides when to cut a new chunk. It balances two main goals:

1.  **Token Limit Adherence:** Don't create chunks that are too large for an LLM's context window.
2.  **Semantic Cohesion:** Keep related rows together, even if it means cutting a chunk slightly before a strict token limit, or slightly after.

It achieves this by:
*   **Dynamically estimating tokens per row:** This makes the chunker adaptive to different datasets.
*   **Calculating an `optimalRowsPerChunk`:** This proactively tries to gather enough rows to hit the `TARGET_CHUNK_SIZE` without splitting a logical block.
*   **Employing a `shouldCreateChunk` condition with two parts:**
    *   **"Hard limit" check:** If adding the next row pushes the *projected* token count way over the `TARGET_CHUNK_SIZE` and we already have a `MIN_ROWS_PER_CHUNK` collected, it's time to cut. This prevents creating overly large chunks.
    *   **"Semantic break" check:** If we've gathered the `optimalRowsPerChunk` (even if we're not exactly at the `TARGET_CHUNK_SIZE`), it's considered a good point to make a new chunk, prioritizing keeping roughly similar amounts of rows together.

This combination ensures that chunks are generally well-sized for LLM processing while trying its best to maintain the natural grouping of structured data rows.