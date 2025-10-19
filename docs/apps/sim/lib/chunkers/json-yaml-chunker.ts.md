This TypeScript file defines a robust utility for splitting large JSON or YAML strings into smaller, manageable "chunks." This process, known as **chunking**, is crucial when dealing with AI models that have input size limitations (measured in "tokens"). The `JsonYamlChunker` aims to break down structured data intelligently, preserving its context and relationships as much as possible, rather than just cutting text arbitrarily.

---

### **Purpose of this File: Intelligent Structured Data Chunking**

At its core, this file provides a `JsonYamlChunker` class designed to:

1.  **Identify Structured Data:** Determine if an input string is valid JSON or YAML.
2.  **Parse and Structure:** Convert the string into its corresponding JavaScript object or array.
3.  **Intelligent Chunking:** Recursively traverse the parsed data (objects and arrays) and break it down into smaller, self-contained "chunks." Each chunk will be a valid JSON string representing a part of the original data.
4.  **Token Counting:** Estimate or accurately count the number of tokens in each chunk, ensuring they adhere to predefined size limits.
5.  **Context Preservation:** Prioritize keeping related pieces of data together within a chunk, rather than splitting an object or array in the middle, to maximize semantic coherence for AI models.
6.  **Fallback Mechanism:** If the input string isn't valid JSON or YAML, it falls back to a simpler line-by-line text chunking strategy.

This is particularly useful for tasks like preparing documents or datasets for large language models (LLMs) or embedding models, where you need to process large inputs but are constrained by token limits.

---

### **Detailed Code Explanation**

Let's break down the file section by section.

#### **1. Imports and Setup**

```typescript
import * as yaml from 'js-yaml'
import { createLogger } from '@/lib/logs/console/logger'
import { getAccurateTokenCount } from '@/lib/tokenization'
import { estimateTokenCount } from '@/lib/tokenization/estimators'
import type { Chunk, ChunkerOptions } from './types'

const logger = createLogger('JsonYamlChunker')
```

*   **`import * as yaml from 'js-yaml'`**: Imports the `js-yaml` library, which provides functionality to parse (load) and serialize (dump) YAML strings. The `* as yaml` syntax means all exports from `js-yaml` will be available under the `yaml` namespace.
*   **`import { createLogger } from '@/lib/logs/console/logger'`**: Imports a `createLogger` function from an internal logging utility. This allows the file to create a dedicated logger instance for its operations, making it easier to track and debug.
*   **`import { getAccurateTokenCount } from '@/lib/tokenization'`**: Imports `getAccurateTokenCount`, likely a function that uses a robust tokenization library (like `tiktoken` for OpenAI models) to precisely count tokens for a given text and model.
*   **`import { estimateTokenCount } from '@/lib/tokenization/estimators'`**: Imports `estimateTokenCount`, a fallback function that provides a less precise but faster token count estimation if the accurate method fails or is unavailable.
*   **`import type { Chunk, ChunkerOptions } from './types'`**: Imports TypeScript `type` definitions for `Chunk` and `ChunkerOptions` from a local `types.ts` file. These define the structure of the output chunks and the configuration options for the chunker.
    *   A `Chunk` type typically includes `text` (the chunked content), `tokenCount`, and `metadata` (like `startIndex`, `endIndex`).
    *   `ChunkerOptions` likely defines optional parameters for the `JsonYamlChunker` constructor.
*   **`const logger = createLogger('JsonYamlChunker')`**: Initializes a logger instance specifically for this `JsonYamlChunker` module. Log messages from this file will be tagged with `'JsonYamlChunker'`, aiding in debugging.

#### **2. Token Counting Helper Function**

```typescript
function getTokenCount(text: string): number {
  try {
    return getAccurateTokenCount(text, 'text-embedding-3-small')
  } catch (error) {
    logger.warn('Tiktoken failed, falling back to estimation')
    const estimate = estimateTokenCount(text)
    return estimate.count
  }
}
```

*   **`function getTokenCount(text: string): number`**: This is a private helper function used throughout the file to determine the token count of a given string `text`.
*   **`try { return getAccurateTokenCount(text, 'text-embedding-3-small') }`**: It first attempts to use the `getAccurateTokenCount` function, specifying `'text-embedding-3-small'` as the model. This model name is important because tokenization rules can vary between different AI models. This method provides a precise token count.
*   **`catch (error) { ... }`**: If `getAccurateTokenCount` fails (e.g., due to an unsupported model, an underlying library issue, or invalid input), the `catch` block executes.
    *   **`logger.warn('Tiktoken failed, falling back to estimation')`**: A warning message is logged, indicating that the accurate token counting method (likely `tiktoken`) failed.
    *   **`const estimate = estimateTokenCount(text)`**: It then falls back to `estimateTokenCount`, which provides an approximate token count.
    *   **`return estimate.count`**: The estimated token count is returned.

#### **3. Chunking Configuration**

```typescript
/**
 * Configuration for JSON/YAML chunking
 * Reduced limits to ensure we stay well under OpenAI's 8,191 token limit per embedding request
 */
const JSON_YAML_CHUNKING_CONFIG = {
  TARGET_CHUNK_SIZE: 1000, // Target tokens per chunk
  MIN_CHUNK_SIZE: 100, // Minimum tokens per chunk
  MAX_CHUNK_SIZE: 1500, // Maximum tokens per chunk
  MAX_DEPTH_FOR_SPLITTING: 5, // Maximum depth to traverse for splitting
}
```

*   **`JSON_YAML_CHUNKING_CONFIG`**: This constant object defines the default parameters for the chunking process.
    *   The JSDoc comment explains that these limits are set to ensure chunks remain well within typical AI model input limits (e.g., OpenAI's 8,191 tokens for embedding models).
    *   **`TARGET_CHUNK_SIZE: 1000`**: The ideal number of tokens for a chunk. The chunker tries to create chunks around this size.
    *   **`MIN_CHUNK_SIZE: 100`**: Chunks smaller than this might be considered too small or insignificant. The chunker tries to avoid creating chunks below this size unless absolutely necessary.
    *   **`MAX_CHUNK_SIZE: 1500`**: The absolute upper limit for a chunk's token count. No chunk should exceed this size.
    *   **`MAX_DEPTH_FOR_SPLITTING: 5`**: This parameter (though not explicitly used in the provided code snippet's logic) typically defines how deep into nested data structures the chunker will attempt to split content. It's a safeguard against excessively deep recursion, which could lead to performance issues or very tiny, uncontextual chunks. If an object/array at a depth greater than this is encountered, it might be chunked as a whole without further splitting its internal structure.

#### **4. `JsonYamlChunker` Class**

This is the main class that orchestrates the chunking logic.

```typescript
export class JsonYamlChunker {
  private chunkSize: number
  private minChunkSize: number

  constructor(options: ChunkerOptions = {}) {
    this.chunkSize = options.chunkSize || JSON_YAML_CHUNKING_CONFIG.TARGET_CHUNK_SIZE
    this.minChunkSize = options.minChunkSize || JSON_YAML_CHUNKING_CONFIG.MIN_CHUNK_SIZE
  }

  /**
   * Check if content is structured JSON/YAML data
   */
  static isStructuredData(content: string): boolean {
    try {
      JSON.parse(content)
      return true
    } catch {
      try {
        yaml.load(content)
        return true
      } catch {
        return false
      }
    }
  }
  // ... rest of the class methods
}
```

*   **`export class JsonYamlChunker`**: Defines the main class, making it available for use in other modules.
*   **`private chunkSize: number`**: A private instance property to store the target chunk size for this specific `JsonYamlChunker` instance.
*   **`private minChunkSize: number`**: A private instance property to store the minimum chunk size for this specific `JsonYamlChunker` instance.
*   **`constructor(options: ChunkerOptions = {})`**: The constructor for the class.
    *   It accepts an optional `options` object of type `ChunkerOptions`. If no options are provided, an empty object `{}` is used as a default.
    *   **`this.chunkSize = options.chunkSize || JSON_YAML_CHUNKING_CONFIG.TARGET_CHUNK_SIZE`**: Initializes `this.chunkSize`. If `options.chunkSize` is provided, it uses that; otherwise, it defaults to `JSON_YAML_CHUNKING_CONFIG.TARGET_CHUNK_SIZE`.
    *   **`this.minChunkSize = options.minChunkSize || JSON_YAML_CHUNKING_CONFIG.MIN_CHUNK_SIZE`**: Initializes `this.minChunkSize`. If `options.minChunkSize` is provided, it uses that; otherwise, it defaults to `JSON_YAML_CHUNKING_CONFIG.MIN_CHUNK_SIZE`.
*   **`static isStructuredData(content: string): boolean`**: A static method, meaning it can be called directly on the class (`JsonYamlChunker.isStructuredData(...)`) without creating an instance.
    *   **Purpose**: Determines if the input `content` string is valid JSON or YAML.
    *   **`try { JSON.parse(content); return true; } catch { ... }`**: It first attempts to parse the `content` as JSON using `JSON.parse()`. If successful, it's JSON, and `true` is returned. If `JSON.parse()` throws an error (meaning it's not valid JSON), the `catch` block is executed.
    *   **`try { yaml.load(content); return true; } catch { return false; }`**: Inside the first `catch` block, it then attempts to parse the `content` as YAML using `yaml.load()`. If successful, it's YAML, and `true` is returned. If `yaml.load()` also throws an error, then the content is neither valid JSON nor YAML, and `false` is returned.

#### **5. Main Chunking Method (`chunk`)**

```typescript
  /**
   * Chunk JSON/YAML content intelligently based on structure
   */
  async chunk(content: string): Promise<Chunk[]> {
    try {
      let data: any
      try {
        data = JSON.parse(content)
      } catch {
        data = yaml.load(content)
      }
      const chunks = this.chunkStructuredData(data)

      const tokenCounts = chunks.map((c) => c.tokenCount)
      const totalTokens = tokenCounts.reduce((a, b) => a + b, 0)
      const maxTokens = Math.max(...tokenCounts)
      const avgTokens = Math.round(totalTokens / chunks.length)

      logger.info(
        `JSON chunking complete: ${chunks.length} chunks, ${totalTokens} total tokens (avg: ${avgTokens}, max: ${maxTokens})`
      )

      return chunks
    } catch (error) {
      logger.info('JSON parsing failed, falling back to text chunking')
      return this.chunkAsText(content)
    }
  }
```

*   **`async chunk(content: string): Promise<Chunk[]>`**: The primary public method to initiate the chunking process. It takes the `content` string and returns a `Promise` that resolves to an array of `Chunk` objects.
*   **`try { ... } catch (error) { ... }`**: This outer `try...catch` block handles errors during the initial parsing of the structured data.
    *   **`let data: any; try { data = JSON.parse(content) } catch { data = yaml.load(content) }`**: It first tries to parse the `content` as JSON. If that fails, it tries to parse it as YAML. The parsed JavaScript object or array is stored in the `data` variable.
    *   **`const chunks = this.chunkStructuredData(data)`**: If parsing is successful, it calls the private `chunkStructuredData` method with the parsed `data`. This method handles the intelligent, recursive splitting of the structured data.
    *   **`const tokenCounts = chunks.map((c) => c.tokenCount)`**: Extracts an array of `tokenCount` values from all generated chunks.
    *   **`const totalTokens = tokenCounts.reduce((a, b) => a + b, 0)`**: Calculates the sum of tokens across all chunks.
    *   **`const maxTokens = Math.max(...tokenCounts)`**: Finds the maximum token count among all chunks.
    *   **`const avgTokens = Math.round(totalTokens / chunks.length)`**: Calculates the average token count per chunk.
    *   **`logger.info(...)`**: Logs summary statistics about the chunking operation (number of chunks, total tokens, average, and maximum chunk tokens).
    *   **`return chunks`**: Returns the array of generated `Chunk` objects.
*   **`catch (error) { ... }`**: If the initial parsing of the `content` (either as JSON or YAML) fails, this `catch` block executes.
    *   **`logger.info('JSON parsing failed, falling back to text chunking')`**: Logs a message indicating that structured parsing failed.
    *   **`return this.chunkAsText(content)`**: Calls the `chunkAsText` method, which handles content as plain text, splitting it line by line.

#### **6. Recursive Structured Chunking (`chunkStructuredData`)**

```typescript
  /**
   * Chunk structured data based on its structure
   */
  private chunkStructuredData(data: any, path: string[] = []): Chunk[] {
    const chunks: Chunk[] = []

    if (Array.isArray(data)) {
      return this.chunkArray(data, path)
    }

    if (typeof data === 'object' && data !== null) {
      return this.chunkObject(data, path)
    }

    // Handle primitive types (string, number, boolean, null)
    const content = JSON.stringify(data, null, 2)
    const tokenCount = getTokenCount(content)

    if (tokenCount >= this.minChunkSize) {
      chunks.push({
        text: content,
        tokenCount,
        metadata: {
          startIndex: 0,
          endIndex: content.length,
        },
      })
    }

    return chunks
  }
```

*   **`private chunkStructuredData(data: any, path: string[] = []): Chunk[]`**: This is a private, recursive method that intelligently chunks data based on its type. It takes the `data` (which could be an object, array, or primitive) and an optional `path` array (to track the current location in the original structure, useful for debugging or metadata).
*   **`if (Array.isArray(data)) { return this.chunkArray(data, path) }`**: If `data` is an array, it delegates the chunking to the `chunkArray` method.
*   **`if (typeof data === 'object' && data !== null) { return this.chunkObject(data, path) }`**: If `data` is an object (and not `null`), it delegates the chunking to the `chunkObject` method.
*   **`const content = JSON.stringify(data, null, 2)`**: If `data` is neither an array nor an object (i.e., it's a primitive like a string, number, boolean, or `null`), it converts this primitive value into a formatted JSON string (with 2-space indentation).
*   **`const tokenCount = getTokenCount(content)`**: Calculates the token count for this primitive content.
*   **`if (tokenCount >= this.minChunkSize) { ... }`**: If the token count of this primitive value meets or exceeds the `minChunkSize`, it's considered a valid chunk.
    *   **`chunks.push({ ... })`**: The primitive content, its token count, and basic metadata (start/end index of its string representation) are added as a `Chunk`.
*   **`return chunks`**: Returns the array of chunks generated from this `data` segment.

#### **7. Array Chunking (`chunkArray`)**

```typescript
  /**
   * Chunk an array intelligently
   */
  private chunkArray(arr: any[], path: string[]): Chunk[] {
    const chunks: Chunk[] = []
    let currentBatch: any[] = []
    let currentTokens = 0

    const contextHeader = path.length > 0 ? `// ${path.join('.')}\n` : ''

    for (let i = 0; i < arr.length; i++) {
      const item = arr[i]
      const itemStr = JSON.stringify(item, null, 2)
      const itemTokens = getTokenCount(itemStr)

      if (itemTokens > this.chunkSize) {
        // ... (handle currentBatch before processing large item)
        if (currentBatch.length > 0) {
          const batchContent = contextHeader + JSON.stringify(currentBatch, null, 2)
          chunks.push({
            text: batchContent,
            tokenCount: getTokenCount(batchContent),
            metadata: {
              startIndex: i - currentBatch.length,
              endIndex: i - 1,
            },
          })
          currentBatch = []
          currentTokens = 0
        }

        // Chunk the large item recursively or directly
        if (typeof item === 'object' && item !== null) {
          const subChunks = this.chunkStructuredData(item, [...path, `[${i}]`])
          chunks.push(...subChunks)
        } else {
          chunks.push({
            text: contextHeader + itemStr,
            tokenCount: itemTokens,
            metadata: {
              startIndex: i,
              endIndex: i,
            },
          })
        }
      } else if (currentTokens + itemTokens > this.chunkSize && currentBatch.length > 0) {
        // ... (finalize currentBatch if adding item exceeds chunkSize)
        const batchContent = contextHeader + JSON.stringify(currentBatch, null, 2)
        chunks.push({
          text: batchContent,
          tokenCount: getTokenCount(batchContent),
          metadata: {
            startIndex: i - currentBatch.length,
            endIndex: i - 1,
          },
        })
        currentBatch = [item]
        currentTokens = itemTokens
      } else {
        // ... (add item to currentBatch)
        currentBatch.push(item)
        currentTokens += itemTokens
      }
    }

    // ... (finalize any remaining currentBatch)
    if (currentBatch.length > 0) {
      const batchContent = contextHeader + JSON.stringify(currentBatch, null, 2)
      chunks.push({
        text: batchContent,
        tokenCount: getTokenCount(batchContent),
        metadata: {
          startIndex: arr.length - currentBatch.length,
          endIndex: arr.length - 1,
        },
      })
    }

    return chunks
  }
```

*   **`private chunkArray(arr: any[], path: string[]): Chunk[]`**: This method intelligently chunks an array `arr`.
*   **`const contextHeader = path.length > 0 ? `// ${path.join('.')}\n` : ''`**: Creates a comment header if a `path` is provided, showing the location in the original document (e.g., `// root.data.items`). This adds context to the chunk.
*   **`let currentBatch: any[] = []; let currentTokens = 0;`**: These variables are used to accumulate array items into a "batch" until their combined token count approaches the `chunkSize`.
*   **`for (let i = 0; i < arr.length; i++) { ... }`**: It iterates through each `item` in the input `arr`.
    *   **`const itemStr = JSON.stringify(item, null, 2); const itemTokens = getTokenCount(itemStr);`**: Converts the current `item` to a JSON string and gets its token count.
    *   **`if (itemTokens > this.chunkSize)`**: If a single `item` is already larger than `this.chunkSize`:
        *   **`if (currentBatch.length > 0) { ... }`**: If there's an existing `currentBatch` (from previous smaller items), that batch is first finalized as a `Chunk` before processing the oversized item. This ensures no data is lost.
        *   **`if (typeof item === 'object' && item !== null) { ... }`**: If the oversized `item` is an object or array, it's recursively passed back to `this.chunkStructuredData` to be broken down further. The `path` is updated to reflect the item's index (`[${i}]`).
        *   **`else { ... }`**: If the oversized `item` is a primitive, it's added as a single `Chunk` (even though it's over `chunkSize`, it can't be split further structurally).
    *   **`else if (currentTokens + itemTokens > this.chunkSize && currentBatch.length > 0)`**: If adding the current `item` to `currentBatch` would exceed `this.chunkSize`, and there's already content in `currentBatch`:
        *   The `currentBatch` is finalized as a `Chunk`.
        *   A *new* `currentBatch` is started with just the current `item`.
    *   **`else { ... }`**: Otherwise (if the `item` fits and doesn't exceed the limit), the `item` is added to `currentBatch`, and `currentTokens` is updated.
*   **`if (currentBatch.length > 0) { ... }`**: After the loop finishes, any remaining items in `currentBatch` are finalized as a last `Chunk`.
*   **`return chunks`**: Returns all generated chunks.

#### **8. Object Chunking (`chunkObject`)**

```typescript
  /**
   * Chunk an object intelligently
   */
  private chunkObject(obj: Record<string, any>, path: string[]): Chunk[] {
    const chunks: Chunk[] = []
    const entries = Object.entries(obj)

    const fullContent = JSON.stringify(obj, null, 2)
    const fullTokens = getTokenCount(fullContent)

    if (fullTokens <= this.chunkSize) {
      chunks.push({
        text: fullContent,
        tokenCount: fullTokens,
        metadata: {
          startIndex: 0,
          endIndex: fullContent.length,
        },
      })
      return chunks
    }

    let currentObj: Record<string, any> = {}
    let currentTokens = 0
    let currentKeys: string[] = [] // Not used explicitly in current code, but good for context

    for (const [key, value] of entries) {
      const valueStr = JSON.stringify({ [key]: value }, null, 2)
      const valueTokens = getTokenCount(valueStr)

      if (valueTokens > this.chunkSize) {
        // ... (handle currentObj before processing large value)
        if (Object.keys(currentObj).length > 0) {
          const objContent = JSON.stringify(currentObj, null, 2)
          chunks.push({
            text: objContent,
            tokenCount: getTokenCount(objContent),
            metadata: {
              startIndex: 0,
              endIndex: objContent.length,
            },
          })
          currentObj = {}
          currentTokens = 0
          currentKeys = []
        }

        // Chunk the large value recursively or directly
        if (typeof value === 'object' && value !== null) {
          const subChunks = this.chunkStructuredData(value, [...path, key])
          chunks.push(...subChunks)
        } else {
          chunks.push({
            text: valueStr,
            tokenCount: valueTokens,
            metadata: {
              startIndex: 0,
              endIndex: valueStr.length,
            },
          })
        }
      } else if (
        currentTokens + valueTokens > this.chunkSize &&
        Object.keys(currentObj).length > 0
      ) {
        // ... (finalize currentObj if adding value exceeds chunkSize)
        const objContent = JSON.stringify(currentObj, null, 2)
        chunks.push({
          text: objContent,
          tokenCount: getTokenCount(objContent),
          metadata: {
            startIndex: 0,
            endIndex: objContent.length,
          },
        })
        currentObj = { [key]: value }
        currentTokens = valueTokens
        currentKeys = [key]
      } else {
        // ... (add value to currentObj)
        currentObj[key] = value
        currentTokens += valueTokens
        currentKeys.push(key)
      }
    }

    // ... (finalize any remaining currentObj)
    if (Object.keys(currentObj).length > 0) {
      const objContent = JSON.stringify(currentObj, null, 2)
      chunks.push({
        text: objContent,
        tokenCount: getTokenCount(objContent),
        metadata: {
          startIndex: 0,
          endIndex: objContent.length,
        },
      })
    }

    return chunks
  }
```

*   **`private chunkObject(obj: Record<string, any>, path: string[]): Chunk[]`**: This method intelligently chunks an object `obj`.
*   **`const entries = Object.entries(obj)`**: Gets an array of `[key, value]` pairs from the object.
*   **`const fullContent = JSON.stringify(obj, null, 2); const fullTokens = getTokenCount(fullContent);`**: First, it checks if the *entire object* can fit into a single chunk.
*   **`if (fullTokens <= this.chunkSize) { ... }`**: If the whole object fits within `this.chunkSize`, it's added as a single `Chunk`, and the method returns immediately. This avoids unnecessary splitting.
*   **`let currentObj: Record<string, any> = {}; let currentTokens = 0;`**: Similar to `chunkArray`, these variables accumulate properties into a "batch" object.
*   **`for (const [key, value] of entries) { ... }`**: It iterates through each key-value pair in the object.
    *   **`const valueStr = JSON.stringify({ [key]: value }, null, 2); const valueTokens = getTokenCount(valueStr);`**: Converts the current key-value pair (as a minimal object `{[key]: value}`) to a JSON string and gets its token count.
    *   **`if (valueTokens > this.chunkSize)`**: If a single key-value pair (or rather, the value itself when represented as `{key: value}`) is larger than `this.chunkSize`:
        *   **`if (Object.keys(currentObj).length > 0) { ... }`**: If there's an existing `currentObj`, it's finalized as a `Chunk`.
        *   **`if (typeof value === 'object' && value !== null) { ... }`**: If the oversized `value` is an object or array, it's recursively chunked by `this.chunkStructuredData`. The `path` is updated with the `key`.
        *   **`else { ... }`**: If the oversized `value` is a primitive, it's added as a single `Chunk`.
    *   **`else if (currentTokens + valueTokens > this.chunkSize && Object.keys(currentObj).length > 0)`**: If adding the current key-value pair to `currentObj` would exceed `this.chunkSize`, and there's existing content in `currentObj`:
        *   The `currentObj` is finalized as a `Chunk`.
        *   A *new* `currentObj` is started with just the current key-value pair.
    *   **`else { ... }`**: Otherwise, the key-value pair is added to `currentObj`, and `currentTokens` is updated.
*   **`if (Object.keys(currentObj).length > 0) { ... }`**: After the loop, any remaining properties in `currentObj` are finalized as a last `Chunk`.
*   **`return chunks`**: Returns all generated chunks.

#### **9. Fallback Text Chunking (`chunkAsText`)**

```typescript
  /**
   * Fall back to text chunking if JSON parsing fails
   */
  private async chunkAsText(content: string): Promise<Chunk[]> {
    const chunks: Chunk[] = []
    const lines = content.split('\n')
    let currentChunk = ''
    let currentTokens = 0
    let startIndex = 0

    for (const line of lines) {
      const lineTokens = getTokenCount(line)

      if (currentTokens + lineTokens > this.chunkSize && currentChunk) {
        chunks.push({
          text: currentChunk,
          tokenCount: currentTokens,
          metadata: {
            startIndex,
            endIndex: startIndex + currentChunk.length,
          },
        })

        startIndex += currentChunk.length + 1 // +1 for the newline character
        currentChunk = line
        currentTokens = lineTokens
      } else {
        currentChunk = currentChunk ? `${currentChunk}\n${line}` : line
        currentTokens += lineTokens
      }
    }

    if (currentChunk && currentTokens >= this.minChunkSize) {
      chunks.push({
        text: currentChunk,
        tokenCount: currentTokens,
        metadata: {
          startIndex,
          endIndex: startIndex + currentChunk.length,
        },
      })
    }

    return chunks
  }
```

*   **`private async chunkAsText(content: string): Promise<Chunk[]>`**: This asynchronous method is used as a fallback if the input `content` cannot be parsed as JSON or YAML. It treats the input as plain text.
*   **`const lines = content.split('\n')`**: Splits the entire content into an array of individual lines.
*   **`let currentChunk = ''; let currentTokens = 0; let startIndex = 0;`**: Variables to accumulate lines into a `currentChunk`. `startIndex` tracks the character position of the current chunk in the original `content`.
*   **`for (const line of lines) { ... }`**: Iterates through each `line`.
    *   **`const lineTokens = getTokenCount(line)`**: Gets the token count for the current line.
    *   **`if (currentTokens + lineTokens > this.chunkSize && currentChunk)`**: If adding the current `line` to `currentChunk` would exceed `this.chunkSize`, and `currentChunk` already contains data:
        *   The `currentChunk` is finalized and pushed as a `Chunk`.
        *   `startIndex` is updated to reflect the start of the *next* chunk.
        *   `currentChunk` is reset to the current `line`, and `currentTokens` is reset to `lineTokens`.
    *   **`else { ... }`**: Otherwise, the `line` is appended to `currentChunk` (with a newline if `currentChunk` isn't empty), and `currentTokens` is updated.
*   **`if (currentChunk && currentTokens >= this.minChunkSize) { ... }`**: After the loop, any remaining `currentChunk` (if it meets the `minChunkSize` requirement) is finalized as the last `Chunk`.
*   **`return chunks`**: Returns the array of text chunks.

#### **10. Static Convenience Method**

```typescript
  /**
   * Static method for chunking JSON/YAML data with default options
   */
  static async chunkJsonYaml(content: string, options: ChunkerOptions = {}): Promise<Chunk[]> {
    const chunker = new JsonYamlChunker(options)
    return chunker.chunk(content)
  }
}
```

*   **`static async chunkJsonYaml(content: string, options: ChunkerOptions = {}): Promise<Chunk[]>`**: A static method that provides a convenient way to use the `JsonYamlChunker` without manually instantiating it.
*   **`const chunker = new JsonYamlChunker(options)`**: Creates a new instance of `JsonYamlChunker` with the provided or default `options`.
*   **`return chunker.chunk(content)`**: Calls the `chunk` method on the newly created instance and returns its result. This allows usage like `JsonYamlChunker.chunkJsonYaml(myContent)`, which is often more concise.

---

### **Simplified Complex Logic: The Chunking Strategy**

The core complexity lies in the `chunkStructuredData`, `chunkArray`, and `chunkObject` methods. Here's a simplified explanation:

1.  **Top-Down Parsing:** The `chunk` method first attempts to parse the entire input as either JSON or YAML. If successful, it gets a complete JavaScript data structure (an object, array, or primitive).
2.  **Structural Delegation:** The `chunkStructuredData` method then acts as a router:
    *   If it sees an **Array**, it passes it to `chunkArray`.
    *   If it sees an **Object**, it passes it to `chunkObject`.
    *   If it sees a **Primitive** (string, number, boolean, null), it converts it to a JSON string and checks if its token count meets the minimum size to become a chunk itself.
3.  **Batching Strategy (for Arrays and Objects):**
    *   Both `chunkArray` and `chunkObject` use a "batching" approach. They don't just split everything. Instead, they try to group smaller elements (array items or object properties) together into a `currentBatch` (or `currentObj`) until their combined token count approaches the `TARGET_CHUNK_SIZE`.
    *   When adding a new element would cause the `currentBatch` to exceed the `TARGET_CHUNK_SIZE`, the `currentBatch` is finalized into a `Chunk`, and a new batch is started.
4.  **Recursive Splitting of Large Elements:**
    *   If a *single* array item or object property is *itself* larger than `TARGET_CHUNK_SIZE` (e.g., a very long string, a huge nested object, or a massive array), the chunker doesn't just put it in a batch. Instead, it recursively calls `chunkStructuredData` on that specific large element. This ensures that even deeply nested complex structures are properly broken down.
5.  **Context Preservation:** By operating on actual data structures (objects and arrays) rather than raw text, the chunker attempts to keep related pieces of data together. For example, all properties of a small object might stay in one chunk, or a few related array items. This is crucial for AI models to understand the context of the data.
6.  **Fallback:** If the initial parsing fails (the input is neither valid JSON nor YAML), the `chunkAsText` method provides a basic line-by-line chunking, treating the content as plain text.

In essence, the `JsonYamlChunker` is a sophisticated tool for preparing structured data for AI, prioritizing semantic coherence and respecting token limits by intelligently traversing and splitting data based on its inherent structure.