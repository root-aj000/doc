This TypeScript file defines a `TextChunker` class designed to break down large blocks of text into smaller, manageable pieces (chunks). This process, known as "chunking," is crucial for applications like Retrieval Augmented Generation (RAG), where Large Language Models (LLMs) need to process relevant snippets of information rather than entire documents.

Let's dive into the details.

---

## Purpose of This File

The primary purpose of `TextChunker` is to:

1.  **Split Text Hierarchically**: It intelligently breaks down text using a predefined list of separators, starting with larger semantic units (like document sections) and progressively moving to smaller ones (like paragraphs, sentences, or even words). This aims to preserve the semantic integrity of the chunks.
2.  **Estimate Token Count**: It provides a simple mechanism to estimate the "token" count of text. LLMs typically operate on tokens, and chunking needs to respect these token limits.
3.  **Respect Chunk Size Limits**: It ensures that generated chunks do not exceed a specified maximum token count (`chunkSize`) and meet a minimum length (`minChunkSize`).
4.  **Add Overlap**: Optionally, it can introduce overlap between consecutive chunks. This helps prevent loss of context at chunk boundaries, which is beneficial for RAG applications.
5.  **Normalize Text**: It cleans up common text inconsistencies (like different line endings, excessive newlines, or multiple spaces) before chunking.
6.  **Provide Metadata**: Each output chunk includes not just the text but also its estimated token count and original `startIndex` and `endIndex` in the cleaned input text, aiding in context tracking.

In essence, it's a lightweight, configurable tool to prepare text efficiently for RAG systems by creating semantically relevant and size-optimized chunks.

---

## Simplified Complex Logic

The core complexity lies in the `splitRecursively` method. Here's a simplified explanation:

Imagine you have a very long story and you want to break it into chapters, then sections, then paragraphs, and so on, but each piece must be less than 500 words.

1.  **Start Big**: The chunker tries to split your story using the biggest separator first (e.g., "document sections" like `\n\n\n`).
2.  **Check Size**: For each resulting piece, it asks: "Is this piece small enough (under 500 words)?"
    *   **Yes, Small Enough**: If it is, great! That piece becomes a chunk (or part of one).
    *   **No, Too Big**: If a piece is still too big, it takes that piece and *restarts* the process, but now using the *next smaller* separator (e.g., "paragraphs" like `\n\n`). This is the "recursive" part.
3.  **Combine & Refine**: As it goes, it tries to combine smaller pieces back together to form chunks, as long as the combined chunk doesn't exceed the 500-word limit. If adding a new piece makes the current chunk too large, it finalizes the current chunk, starts a new one with the new piece, and potentially recursively splits the new piece if it's still too large.
4.  **Last Resort**: If it runs out of all intelligent separators and a piece is *still* too big, it will just cut the text into fixed-length segments, even if it breaks words or sentences, to meet the size constraint.
5.  **Overlap (Post-Processing)**: After all the chunks are created, if you've asked for overlap, it goes back and adds a few words from the end of the *previous* chunk to the beginning of the *current* chunk. This helps ensure that if an important context spans two chunks, both chunks contain a bit of that bridging information.

This hierarchical approach helps keep related text together, which is vital for maintaining meaning when feeding text to an LLM.

---

## Line-by-Line Explanation

Let's break down the code in detail.

```typescript
import type { Chunk, ChunkerOptions } from './types'
```

*   **`import type { Chunk, ChunkerOptions } from './types'`**: This line imports two type definitions from a local file named `types.ts` (or `types.d.ts`).
    *   `Chunk`: Represents the structure of a single text chunk, likely including the text content itself, token count, and metadata.
    *   `ChunkerOptions`: Defines the structure of the configuration object passed to the `TextChunker` constructor.
    *   `type` keyword: This is a TypeScript-specific import that only brings in type declarations, not actual runnable code. This means these imports are completely removed during compilation to JavaScript.

```typescript
/**
 * Lightweight text chunker optimized for RAG applications
 * Uses hierarchical splitting with simple character-based token estimation
 */
export class TextChunker {
```

*   **`/** ... */`**: This is a JSDoc comment providing a high-level description of the `TextChunker` class. It highlights its purpose (RAG optimization), core methodology (hierarchical splitting), and token estimation approach (character-based).
*   **`export class TextChunker {`**: This declares a new class named `TextChunker`. The `export` keyword makes this class accessible from other TypeScript or JavaScript files that import it.

```typescript
  private readonly chunkSize: number
  private readonly minChunkSize: number
  private readonly overlap: number
```

*   **`private readonly chunkSize: number`**: Declares a private, read-only property named `chunkSize` which will store the maximum desired token count for each chunk. `private` means it can only be accessed within the `TextChunker` class. `readonly` means its value can only be set during initialization (in the constructor) and cannot be changed afterward.
*   **`private readonly minChunkSize: number`**: Declares a private, read-only property `minChunkSize` for the minimum character length (not tokens, for consistency with `trim()` and `length`) a chunk must have to be considered valid. Chunks shorter than this will be discarded or merged.
*   **`private readonly overlap: number`**: Declares a private, read-only property `overlap`. This will store the number of words to overlap between consecutive chunks. If 0 or less, no overlap will be added.

```typescript
  // Hierarchical separators ordered from largest to smallest semantic units
  private readonly separators = [
    '\n\n\n', // Document sections
    '\n---\n', // Markdown horizontal rules
    '\n***\n', // Markdown horizontal rules (alternative)
    '\n___\n', // Markdown horizontal rules (alternative)
    '\n# ', // Markdown H1 headings
    '\n## ', // Markdown H2 headings
    '\n### ', // Markdown H3 headings
    '\n#### ', // Markdown H4 headings
    '\n##### ', // Markdown H5 headings
    '\n###### ', // Markdown H6 headings
    '\n\n', // Paragraphs
    '\n', // Lines
    '. ', // Sentences (with trailing space)
    '! ', // Exclamations (with trailing space)
    '? ', // Questions (with trailing space)
    '; ', // Semicolons (with trailing space)
    ', ', // Commas (with trailing space)
    ' ', // Words (single space)
  ]
```

*   **`private readonly separators = [`**: Declares a private, read-only array named `separators`. This array holds the strings that will be used to split the text.
*   **`'\n\n\n', // Document sections`**: The array is ordered hierarchically from the largest semantic units (like triple newlines indicating a major section break) down to the smallest (a single space for word splitting). This order is crucial for the `splitRecursively` method to work effectively, attempting to preserve larger semantic structures first.
*   The comments explain what each separator typically represents (e.g., Markdown headings, paragraphs, sentences, words).

```typescript
  constructor(options: ChunkerOptions = {}) {
    this.chunkSize = options.chunkSize ?? 512
    this.minChunkSize = options.minChunkSize ?? 1
    this.overlap = options.overlap ?? 0
  }
```

*   **`constructor(options: ChunkerOptions = {}) {`**: This is the class constructor. It's called when a new `TextChunker` instance is created. It accepts an `options` object of type `ChunkerOptions`. The `= {}` provides a default empty object, allowing the constructor to be called without arguments.
*   **`this.chunkSize = options.chunkSize ?? 512`**: Assigns the `chunkSize` property. It tries to use `options.chunkSize` if it's provided and not `null` or `undefined`. Otherwise, it defaults to `512`. The `??` operator is the nullish coalescing operator.
*   **`this.minChunkSize = options.minChunkSize ?? 1`**: Assigns `minChunkSize`. Defaults to `1`.
*   **`this.overlap = options.overlap ?? 0`**: Assigns `overlap`. Defaults to `0`.

```typescript
  /**
   * Simple token estimation using character count
   */
  private estimateTokens(text: string): number {
    // Handle empty or whitespace-only text
    if (!text?.trim()) return 0

    // Simple estimation: ~4 characters per token
    return Math.ceil(text.length / 4)
  }
```

*   **`private estimateTokens(text: string): number {`**: Declares a private method `estimateTokens` that takes a `string` and returns a `number`. This method provides a very basic way to estimate token count.
*   **`if (!text?.trim()) return 0`**: Checks if the input `text` is `null`, `undefined`, an empty string, or a string consisting only of whitespace characters (after `trim()`). If so, it returns `0` tokens. The `?.` (optional chaining) ensures `trim()` is only called if `text` is not `null` or `undefined`.
*   **`return Math.ceil(text.length / 4)`**: This is the estimation logic. It assumes, as a general rule of thumb for many LLMs, that roughly 4 characters correspond to 1 token. `Math.ceil()` rounds up to the nearest whole number, ensuring that even a partial token counts as one.

```typescript
  /**
   * Split text recursively using hierarchical separators
   */
  private async splitRecursively(text: string, separatorIndex = 0): Promise<string[]> {
    const tokenCount = this.estimateTokens(text)
```

*   **`private async splitRecursively(text: string, separatorIndex = 0): Promise<string[]> {`**: Declares a private, asynchronous method `splitRecursively`. It takes the `text` to split and an optional `separatorIndex` (defaulting to `0`, meaning it starts with the first separator in the `separators` array). It returns a `Promise` that resolves to an array of `string`s (the chunks).
*   **`const tokenCount = this.estimateTokens(text)`**: Estimates the token count of the current `text` using the `estimateTokens` method.

```typescript
    // If chunk is small enough, return it
    if (tokenCount <= this.chunkSize) {
      return text.length >= this.minChunkSize ? [text] : []
    }
```

*   **`if (tokenCount <= this.chunkSize)`**: This is the primary base case for the recursion. If the current `text` is already within the `chunkSize` limit (in tokens), it's considered a valid chunk or part of one.
*   **`return text.length >= this.minChunkSize ? [text] : []`**: Before returning, it checks if the `text` also meets the `minChunkSize` (in characters). If it does, it returns an array containing just this `text`. Otherwise, it returns an empty array, effectively discarding this small piece.

```typescript
    // If we've run out of separators, force split by character count
    if (separatorIndex >= this.separators.length) {
      const chunks: string[] = []
      // Calculate target length based on character count proportional to token limit
      const targetLength = Math.ceil((text.length * this.chunkSize) / tokenCount)

      for (let i = 0; i < text.length; i += targetLength) {
        const chunk = text.slice(i, i + targetLength).trim()
        if (chunk.length >= this.minChunkSize) {
          chunks.push(chunk)
        }
      }
      return chunks
    }
```

*   **`if (separatorIndex >= this.separators.length)`**: This is another base case: if we've exhausted all defined `separators` (meaning `separatorIndex` has gone past the last valid index).
*   **`const chunks: string[] = []`**: Initializes an empty array to store the resulting forced-split chunks.
*   **`const targetLength = Math.ceil((text.length * this.chunkSize) / tokenCount)`**: Calculates an approximate character length for each sub-chunk. It tries to divide the `text` into segments that, on average, would correspond to `chunkSize` tokens, proportional to the total token count of the `text`.
*   **`for (let i = 0; i < text.length; i += targetLength)`**: A loop to iterate through the `text` and slice it into segments of `targetLength`.
*   **`const chunk = text.slice(i, i + targetLength).trim()`**: Extracts a slice of the `text` and trims any leading/trailing whitespace.
*   **`if (chunk.length >= this.minChunkSize) { chunks.push(chunk) }`**: If the extracted `chunk` meets the minimum length requirement, it's added to the `chunks` array.
*   **`return chunks`**: Returns the array of forcibly split chunks.

```typescript
    const separator = this.separators[separatorIndex]
    const parts = text.split(separator).filter((part) => part.trim())
```

*   **`const separator = this.separators[separatorIndex]`**: Gets the current separator string from the `separators` array based on `separatorIndex`.
*   **`const parts = text.split(separator).filter((part) => part.trim())`**: Splits the `text` using the `separator`. The `.filter((part) => part.trim())` removes any empty strings or strings containing only whitespace that might result from the split (e.g., if a separator appears at the beginning or end of the text, or if there are multiple consecutive separators).

```typescript
    // If no split occurred, try next separator
    if (parts.length <= 1) {
      return await this.splitRecursively(text, separatorIndex + 1)
    }
```

*   **`if (parts.length <= 1)`**: If `text.split(separator)` results in `0` or `1` part, it means the current `separator` was not found in the `text`, or it only occurred in a way that produced empty parts (which were filtered out). In this case, the current separator isn't effective for splitting this `text`.
*   **`return await this.splitRecursively(text, separatorIndex + 1)`**: The method then recursively calls itself with the *same* `text` but with the `separatorIndex` incremented, effectively trying the *next smaller* separator.

```typescript
    const chunks: string[] = []
    let currentChunk = ''

    for (const part of parts) {
      const testChunk = currentChunk + (currentChunk ? separator : '') + part
```

*   **`const chunks: string[] = []`**: Initializes an empty array to collect the final chunks generated at this level of recursion.
*   **`let currentChunk = ''`**: Initializes an empty string `currentChunk`. This variable will accumulate `parts` until it becomes too large, at which point it's finalized and pushed to `chunks`.
*   **`for (const part of parts) { ... }`**: Loops through each `part` obtained from splitting the `text` with the current `separator`.
*   **`const testChunk = currentChunk + (currentChunk ? separator : '') + part`**: Constructs a `testChunk`. If `currentChunk` already has content, it adds the `separator` back between `currentChunk` and the `part` to preserve the original structure. If `currentChunk` is empty (for the first `part`), it just uses the `part` itself.

```typescript
      if (this.estimateTokens(testChunk) <= this.chunkSize) {
        currentChunk = testChunk
      } else {
        // Save current chunk if it meets minimum size
        if (currentChunk.trim() && currentChunk.length >= this.minChunkSize) {
          chunks.push(currentChunk.trim())
        }

        // Start new chunk with current part
        // If part itself is too large, split it further
        if (this.estimateTokens(part) > this.chunkSize) {
          chunks.push(...(await this.splitRecursively(part, separatorIndex + 1)))
          currentChunk = ''
        } else {
          currentChunk = part
        }
      }
    }
```

*   **`if (this.estimateTokens(testChunk) <= this.chunkSize)`**: Checks if `testChunk` (current accumulated content + the new `part`) is still within the `chunkSize` limit.
    *   **`currentChunk = testChunk`**: If it fits, the `part` is appended to `currentChunk`.
*   **`else { ... }`**: If `testChunk` is too large (meaning adding the current `part` made it exceed `chunkSize`):
    *   **`if (currentChunk.trim() && currentChunk.length >= this.minChunkSize)`**: The `currentChunk` (before adding the too-large `part`) is finalized. If it has content and meets the minimum size, it's added to the `chunks` array.
    *   **`if (this.estimateTokens(part) > this.chunkSize)`**: Now, consider the `part` that just caused the overflow. If *this `part` itself* is too large (exceeds `chunkSize`), it needs further splitting.
        *   **`chunks.push(...(await this.splitRecursively(part, separatorIndex + 1)))`**: Recursively calls `splitRecursively` on this `part` with the `next smaller` separator (`separatorIndex + 1`). The spread operator (`...`) adds all the resulting sub-chunks from that recursive call directly to the `chunks` array.
        *   **`currentChunk = ''`**: `currentChunk` is reset because the `part` was handled separately.
    *   **`else { currentChunk = part }`**: If the `part` itself is not too large, it becomes the start of a new `currentChunk`.

```typescript
    // Add final chunk if it exists and meets minimum size
    if (currentChunk.trim() && currentChunk.length >= this.minChunkSize) {
      chunks.push(currentChunk.trim())
    }

    return chunks
  }
```

*   **`if (currentChunk.trim() && currentChunk.length >= this.minChunkSize)`**: After the loop finishes, there might be some content left in `currentChunk` that hasn't been pushed to the `chunks` array. This check ensures that any remaining `currentChunk` is added if it's not empty and meets the minimum size.
*   **`return chunks`**: Returns the array of all chunks generated at this level of recursion.

```typescript
  /**
   * Add overlap between chunks if specified
   */
  private addOverlap(chunks: string[]): string[] {
    if (this.overlap <= 0 || chunks.length <= 1) {
      return chunks
    }
```

*   **`private addOverlap(chunks: string[]): string[] {`**: Declares a private method `addOverlap` that takes an array of string `chunks` and returns a new array of string `chunks` with overlap applied.
*   **`if (this.overlap <= 0 || chunks.length <= 1)`**: This is an early exit condition. If `overlap` is not configured (0 or less) or there's only one chunk (so no previous chunk to overlap with), it returns the original `chunks` array without modification.

```typescript
    const overlappedChunks: string[] = []

    for (let i = 0; i < chunks.length; i++) {
      let chunk = chunks[i]

      // Add overlap from previous chunk
      if (i > 0) {
        const prevChunk = chunks[i - 1]
        const words = prevChunk.split(/\s+/) // Split previous chunk into words
        // Get the last 'overlap' number of words from the previous chunk
        const overlapWords = words.slice(-Math.min(this.overlap, words.length))

        if (overlapWords.length > 0) {
          chunk = `${overlapWords.join(' ')} ${chunk}` // Prepend overlap words, joined by space
        }
      }

      overlappedChunks.push(chunk)
    }

    return overlappedChunks
  }
```

*   **`const overlappedChunks: string[] = []`**: Initializes an empty array to store the chunks with overlap.
*   **`for (let i = 0; i < chunks.length; i++) { ... }`**: Loops through each `chunk` in the input `chunks` array.
*   **`let chunk = chunks[i]`**: Gets the current chunk.
*   **`if (i > 0)`**: This condition ensures overlap is only added for chunks *after* the very first one.
*   **`const prevChunk = chunks[i - 1]`**: Gets the preceding chunk.
*   **`const words = prevChunk.split(/\s+/)`**: Splits the `prevChunk` into an array of words, using one or more whitespace characters as a delimiter.
*   **`const overlapWords = words.slice(-Math.min(this.overlap, words.length))`**: Extracts words for overlap.
    *   `Math.min(this.overlap, words.length)`: Ensures we don't try to get more words than actually exist in `prevChunk`.
    *   `words.slice(-N)`: Returns the last `N` elements of the `words` array.
*   **`if (overlapWords.length > 0)`**: If there are actual words to overlap.
*   **`chunk = `${overlapWords.join(' ')} ${chunk}``**: Prepends the `overlapWords` (joined back by single spaces) to the current `chunk`, separated by a space.
*   **`overlappedChunks.push(chunk)`**: Adds the (potentially) overlapped `chunk` to the `overlappedChunks` array.
*   **`return overlappedChunks`**: Returns the array of chunks, now with overlap.

```typescript
  /**
   * Clean and normalize text
   */
  private cleanText(text: string): string {
    return text
      .replace(/\r\n/g, '\n') // Normalize Windows line endings
      .replace(/\r/g, '\n') // Normalize old Mac line endings
      .replace(/\n{3,}/g, '\n\n') // Limit consecutive newlines to two
      .replace(/\t/g, ' ') // Convert tabs to spaces
      .replace(/ {2,}/g, ' ') // Collapse multiple spaces into a single space
      .trim() // Remove leading/trailing whitespace
  }
```

*   **`private cleanText(text: string): string {`**: Declares a private method `cleanText` that takes a `string` and returns a cleaned `string`.
*   **`.replace(/\r\n/g, '\n')`**: Replaces all occurrences (`g` flag for global) of Windows-style line endings (`\r\n`) with Unix-style (`\n`).
*   **`.replace(/\r/g, '\n')`**: Replaces old Mac-style line endings (`\r`) with Unix-style (`\n`).
*   **`.replace(/\n{3,}/g, '\n\n')`**: Replaces three or more consecutive newlines with exactly two newlines, limiting excessive blank lines.
*   **`.replace(/\t/g, ' ')`**: Replaces all tab characters (`\t`) with a single space.
*   **`.replace(/ {2,}/g, ' ')`**: Replaces two or more consecutive spaces with a single space, collapsing whitespace.
*   **`.trim()`**: Removes any leading or trailing whitespace (spaces, newlines, tabs) from the entire string.

```typescript
  /**
   * Main chunking method
   */
  async chunk(text: string): Promise<Chunk[]> {
    if (!text?.trim()) {
      return []
    }
```

*   **`async chunk(text: string): Promise<Chunk[]> {`**: This is the main public method to initiate the chunking process. It's asynchronous, takes the input `text`, and returns a `Promise` resolving to an array of `Chunk` objects.
*   **`if (!text?.trim()) { return [] }`**: Similar to `estimateTokens`, this is an initial guard clause. If the input `text` is empty or only whitespace, it immediately returns an empty array, as there's nothing to chunk.

```typescript
    // Clean the text
    const cleanedText = this.cleanText(text)

    // Split into chunks
    let chunks = await this.splitRecursively(cleanedText)

    // Add overlap if configured
    chunks = this.addOverlap(chunks)
```

*   **`const cleanedText = this.cleanText(text)`**: First, the raw input `text` is passed through the `cleanText` method to normalize it.
*   **`let chunks = await this.splitRecursively(cleanedText)`**: The cleaned text is then passed to the core `splitRecursively` method to generate an initial array of string chunks. The `await` keyword pauses execution until the asynchronous `splitRecursively` call completes.
*   **`chunks = this.addOverlap(chunks)`**: If overlap is configured, the `addOverlap` method is called on the string `chunks` array. The result (either modified or original) updates the `chunks` variable.

```typescript
    // Convert to Chunk objects with metadata
    let previousEndIndex = 0
    const chunkPromises = chunks.map(async (chunkText, index) => {
      let startIndex: number
      let actualContentLength: number

      if (index === 0 || this.overlap <= 0) {
        // First chunk or no overlap - start from previous end
        startIndex = previousEndIndex
        actualContentLength = chunkText.length
      } else {
        // Calculate overlap length in characters
        const prevChunk = chunks[index - 1]
        const prevWords = prevChunk.split(/\s+/)
        const overlapWords = prevWords.slice(-Math.min(this.overlap, prevWords.length))
        const overlapLength = Math.min(
          chunkText.length,
          overlapWords.length > 0 ? overlapWords.join(' ').length + 1 : 0 // +1 for space
        )

        startIndex = previousEndIndex - overlapLength
        actualContentLength = chunkText.length - overlapLength
      }

      const safeStart = Math.max(0, startIndex)
      const endIndexSafe = safeStart + actualContentLength

      const chunk: Chunk = {
        text: chunkText,
        tokenCount: this.estimateTokens(chunkText),
        metadata: {
          startIndex: safeStart,
          endIndex: endIndexSafe,
        },
      }

      previousEndIndex = endIndexSafe
      return chunk
    })

    return await Promise.all(chunkPromises)
  }
}
```

*   **`let previousEndIndex = 0`**: Initializes a variable to keep track of the end index of the *original content* portion of the previous chunk within the `cleanedText`. This is crucial for calculating accurate `startIndex` and `endIndex` for each chunk's *original content*, especially with overlap.
*   **`const chunkPromises = chunks.map(async (chunkText, index) => { ... })`**: This uses the `map` method to transform the array of `string` chunks into an array of `Promise<Chunk>` objects. Each chunk is processed asynchronously (though the `previousEndIndex` makes the order of calculation important, `map` creates promises for each).
*   **`let startIndex: number; let actualContentLength: number;`**: Declares variables to store the calculated start index and the length of the *non-overlapped* content.
*   **`if (index === 0 || this.overlap <= 0)`**:
    *   For the first chunk, or if no `overlap` is configured, the `startIndex` is simply where the previous chunk ended (`previousEndIndex`).
    *   `actualContentLength` is the full `chunkText.length`.
*   **`else { ... }`**: If it's not the first chunk and `overlap` is active:
    *   **`const prevChunk = chunks[index - 1]`**: Gets the preceding chunk (which might also have overlap).
    *   **`const prevWords = prevChunk.split(/\s+/)`**: Splits it into words.
    *   **`const overlapWords = prevWords.slice(-Math.min(this.overlap, prevWords.length))`**: Extracts the words that were supposed to overlap.
    *   **`const overlapLength = Math.min(chunkText.length, overlapWords.length > 0 ? overlapWords.join(' ').length + 1 : 0)`**: Calculates the *character length* of the overlap.
        *   `overlapWords.join(' ').length + 1`: This is the length of the overlapping words plus one for the space that was added *between* the overlap and the original chunk text.
        *   `Math.min(chunkText.length, ...)`: Ensures the `overlapLength` doesn't exceed the `chunkText` itself (e.g., if the chunk is very short).
    *   **`startIndex = previousEndIndex - overlapLength`**: The `startIndex` for the *actual new content* of this chunk is calculated by taking the `previousEndIndex` and subtracting the `overlapLength`. This effectively "backs up" the index to where the *unique* content of the current chunk begins in the original `cleanedText`.
    *   **`actualContentLength = chunkText.length - overlapLength`**: The length of the *unique* content is the total `chunkText.length` minus the `overlapLength`.
*   **`const safeStart = Math.max(0, startIndex)`**: Ensures `startIndex` never goes below 0, which could happen if `overlapLength` is very large for a short initial `cleanedText`.
*   **`const endIndexSafe = safeStart + actualContentLength`**: Calculates the safe end index of the *unique* content portion of the chunk in the original `cleanedText`.
*   **`const chunk: Chunk = { ... }`**: Creates a `Chunk` object.
    *   `text: chunkText`: The full text of the chunk (including overlap if any).
    *   `tokenCount: this.estimateTokens(chunkText)`: The estimated token count of this full chunk.
    *   `metadata: { startIndex: safeStart, endIndex: endIndexSafe }`: Metadata indicating the range of the *unique content* of this chunk within the original `cleanedText`.
*   **`previousEndIndex = endIndexSafe`**: Updates `previousEndIndex` for the next iteration of the `map` loop, so the next chunk can correctly calculate its metadata.
*   **`return chunk`**: Returns the created `Chunk` object.
*   **`return await Promise.all(chunkPromises)`**: After `map` has created an array of `Promise<Chunk>` objects, `Promise.all` waits for all these promises to resolve concurrently. It then returns a single `Promise` that resolves to an array of `Chunk` objects, which is the final output of the `chunk` method.