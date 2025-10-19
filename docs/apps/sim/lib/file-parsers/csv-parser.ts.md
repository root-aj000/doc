```typescript
import { createReadStream, existsSync } from 'fs'
import { Readable } from 'stream'
import { type Options, parse } from 'csv-parse'
import type { FileParseResult, FileParser } from '@/lib/file-parsers/types'
import { sanitizeTextForUTF8 } from '@/lib/file-parsers/utils'
import { createLogger } from '@/lib/logs/console/logger'

const logger = createLogger('CsvParser')

const CONFIG = {
  MAX_PREVIEW_ROWS: 1000, // Only keep first 1000 rows for preview
  MAX_SAMPLE_ROWS: 100, // Sample for metadata
  MAX_ERRORS: 100, // Stop after 100 errors
  STREAM_CHUNK_SIZE: 16384, // 16KB chunks for streaming
}

export class CsvParser implements FileParser {
  async parseFile(filePath: string): Promise<FileParseResult> {
    if (!filePath) {
      throw new Error('No file path provided')
    }

    if (!existsSync(filePath)) {
      throw new Error(`File not found: ${filePath}`)
    }

    const stream = createReadStream(filePath, {
      highWaterMark: CONFIG.STREAM_CHUNK_SIZE,
    })

    return this.parseStream(stream)
  }

  async parseBuffer(buffer: Buffer): Promise<FileParseResult> {
    const bufferSize = buffer.length
    logger.info(
      `Parsing CSV buffer, size: ${bufferSize} bytes (${(bufferSize / 1024 / 1024).toFixed(2)} MB)`
    )

    const stream = Readable.from(buffer, {
      highWaterMark: CONFIG.STREAM_CHUNK_SIZE,
    })

    return this.parseStream(stream)
  }

  private parseStream(inputStream: NodeJS.ReadableStream): Promise<FileParseResult> {
    return new Promise((resolve, reject) => {
      let rowCount = 0
      let errorCount = 0
      let headers: string[] = []
      let processedContent = ''
      const sampledRows: any[] = []
      const errors: string[] = []
      let firstRowProcessed = false
      let aborted = false

      const parserOptions: Options = {
        columns: true, // Use first row as headers
        skip_empty_lines: true, // Skip empty lines
        trim: true, // Trim whitespace
        relax_column_count: true, // Allow variable column counts
        relax_quotes: true, // Be lenient with quotes
        skip_records_with_error: true, // Skip bad records
        raw: false,
        cast: false,
      }
      const parser = parse(parserOptions)

      parser.on('readable', () => {
        let record
        while ((record = parser.read()) !== null && !aborted) {
          rowCount++

          if (!firstRowProcessed && record) {
            headers = Object.keys(record).map((h) => sanitizeTextForUTF8(String(h)))
            processedContent = `${headers.join(', ')}\n`
            firstRowProcessed = true
          }

          if (rowCount <= CONFIG.MAX_PREVIEW_ROWS) {
            try {
              const cleanValues = Object.values(record).map((v: any) =>
                sanitizeTextForUTF8(String(v || ''))
              )
              processedContent += `${cleanValues.join(', ')}\n`

              if (rowCount <= CONFIG.MAX_SAMPLE_ROWS) {
                sampledRows.push(record)
              }
            } catch (err) {
              logger.warn(`Error processing row ${rowCount}:`, err)
            }
          }

          if (rowCount % 10000 === 0) {
            logger.info(`Processed ${rowCount} rows...`)
          }
        }
      })

      parser.on('skip', (err: any) => {
        errorCount++

        if (errorCount <= 5) {
          const errorMsg = `Row ${err.lines || rowCount}: ${err.message || 'Unknown error'}`
          errors.push(errorMsg)
          logger.warn('CSV skip:', errorMsg)
        }

        if (errorCount >= CONFIG.MAX_ERRORS) {
          aborted = true
          parser.destroy()
          reject(new Error(`Too many errors (${errorCount}). File may be corrupted.`))
        }
      })

      parser.on('error', (err: Error) => {
        logger.error('CSV parser error:', err)
        reject(new Error(`CSV parsing failed: ${err.message}`))
      })

      parser.on('end', () => {
        if (!aborted) {
          if (rowCount > CONFIG.MAX_PREVIEW_ROWS) {
            processedContent += `\n[... ${rowCount.toLocaleString()} total rows, showing first ${CONFIG.MAX_PREVIEW_ROWS} ...]\n`
          }

          logger.info(`CSV parsing complete: ${rowCount} rows, ${errorCount} errors`)

          resolve({
            content: sanitizeTextForUTF8(processedContent),
            metadata: {
              rowCount,
              headers,
              errorCount,
              errors: errors.slice(0, 10),
              truncated: rowCount > CONFIG.MAX_PREVIEW_ROWS,
              sampledData: sampledRows,
            },
          })
        }
      })

      inputStream.on('error', (err) => {
        logger.error('Input stream error:', err)
        parser.destroy()
        reject(new Error(`Stream error: ${err.message}`))
      })

      inputStream.pipe(parser)
    })
  }
}
```

## Explanation of the CSV Parser Code

This TypeScript code defines a `CsvParser` class responsible for parsing CSV (Comma Separated Values) files. It provides methods to parse CSV data from a file path or a buffer and returns structured data containing content and metadata.

**1. Imports:**

*   `fs`: This module is part of the Node.js standard library and provides methods for interacting with the file system.
    *   `createReadStream`:  Used to create a readable stream from a file. This is crucial for handling large files efficiently, as it processes the file in chunks rather than loading the entire file into memory at once.
    *   `existsSync`:  A synchronous method that checks if a file exists at the given path.

*   `stream`: This module is also part of the Node.js standard library and provides an abstraction for working with streaming data.
    *   `Readable`: Used to create a readable stream from a buffer.

*   `csv-parse`: This external library (`csv-parse`) is the core of the CSV parsing functionality. It provides the `parse` function, which takes a readable stream and parsing options as input and converts the CSV data into JavaScript objects.
    *   `Options`:  A type definition for the options that can be passed to the `parse` function.
    *   `parse`:  The main function from the `csv-parse` library that performs the actual CSV parsing.

*   `@/lib/file-parsers/types`: This imports type definitions related to file parsing, specifically `FileParseResult` (the structure of the data returned after parsing) and `FileParser` (an interface defining the `parseFile` method). This indicates that the `CsvParser` class implements a more general file parsing interface.

*   `@/lib/file-parsers/utils`:  Imports utility functions, in this case, `sanitizeTextForUTF8`, which likely cleans text data to ensure it's valid UTF-8. This is important for handling CSV files with potentially problematic character encodings.

*   `@/lib/logs/console/logger`: Imports a logging utility (`createLogger`) to handle logging messages. This is helpful for debugging and monitoring the parsing process.

**2. Logger Initialization:**

```typescript
const logger = createLogger('CsvParser')
```

*   Creates a logger instance named `CsvParser`.  This logger is used throughout the class to record information, warnings, and errors. Using a logger makes debugging and monitoring easier.

**3. Configuration Constants:**

```typescript
const CONFIG = {
  MAX_PREVIEW_ROWS: 1000, // Only keep first 1000 rows for preview
  MAX_SAMPLE_ROWS: 100, // Sample for metadata
  MAX_ERRORS: 100, // Stop after 100 errors
  STREAM_CHUNK_SIZE: 16384, // 16KB chunks for streaming
}
```

*   Defines constants for configuring the parser's behavior.
    *   `MAX_PREVIEW_ROWS`:  Limits the number of rows included in the `content` of the result.  This is useful for preventing the parser from generating extremely large strings when dealing with very large files.  The preview is the main content returned, meant for quick viewing.
    *   `MAX_SAMPLE_ROWS`: Limits the number of rows stored in the `sampledData` array in the result's `metadata`. This provides a representative sample of the data without storing the entire dataset in memory.  This is used for creating previews and summaries.
    *   `MAX_ERRORS`:  Sets a maximum number of errors that the parser will tolerate before aborting.  This prevents the parser from getting stuck in an infinite loop when processing a corrupted file.
    *   `STREAM_CHUNK_SIZE`:  Specifies the size of the chunks that the `createReadStream` function will use when reading the file. This determines how much data is read from the file system at a time, balancing memory usage and performance.

**4. `CsvParser` Class:**

```typescript
export class CsvParser implements FileParser {
  // ... methods ...
}
```

*   Declares the `CsvParser` class, which is the main component of this code. It implements the `FileParser` interface.
*   The `export` keyword makes this class available for use in other modules.

**5. `parseFile` Method:**

```typescript
async parseFile(filePath: string): Promise<FileParseResult> {
  if (!filePath) {
    throw new Error('No file path provided')
  }

  if (!existsSync(filePath)) {
    throw new Error(`File not found: ${filePath}`)
  }

  const stream = createReadStream(filePath, {
    highWaterMark: CONFIG.STREAM_CHUNK_SIZE,
  })

  return this.parseStream(stream)
}
```

*   This method takes a file path as input and returns a `Promise` that resolves to a `FileParseResult` object.
*   **Input Validation:** It first checks if the `filePath` is provided and if the file exists using `existsSync`. If either check fails, it throws an error.
*   **Creating a Readable Stream:** It creates a readable stream from the file using `createReadStream`. The `highWaterMark` option sets the buffer size for the stream.
*   **Calling `parseStream`:** It then calls the `parseStream` method, passing the created stream to handle the actual parsing logic.

**6. `parseBuffer` Method:**

```typescript
async parseBuffer(buffer: Buffer): Promise<FileParseResult> {
  const bufferSize = buffer.length
  logger.info(
    `Parsing CSV buffer, size: ${bufferSize} bytes (${(bufferSize / 1024 / 1024).toFixed(2)} MB)`
  )

  const stream = Readable.from(buffer, {
    highWaterMark: CONFIG.STREAM_CHUNK_SIZE,
  })

  return this.parseStream(stream)
}
```

*   This method takes a `Buffer` object (containing the CSV data) as input and returns a `Promise` that resolves to a `FileParseResult`.
*   **Logging Buffer Size:** Logs the size of the buffer being parsed for informational purposes.
*   **Creating a Readable Stream:** It creates a readable stream from the buffer using `Readable.from`. The `highWaterMark` option sets the buffer size for the stream.
*   **Calling `parseStream`:** It then calls the `parseStream` method, passing the created stream.

**7. `parseStream` Method:**

```typescript
private parseStream(inputStream: NodeJS.ReadableStream): Promise<FileParseResult> {
  return new Promise((resolve, reject) => {
    // ... parsing logic ...
  })
}
```

*   This is the core method that performs the CSV parsing.  It takes a `ReadableStream` as input and returns a `Promise` containing the `FileParseResult`. This method is `private` because it's intended to be used only within the `CsvParser` class.
*   **Promise Wrapper:** The entire parsing logic is wrapped in a `Promise` constructor. This allows the method to handle asynchronous operations and return a result (or an error) when the parsing is complete.
*   **Variable Initialization:** Several variables are initialized to store parsing state:
    *   `rowCount`: Keeps track of the number of rows processed.
    *   `errorCount`: Keeps track of the number of errors encountered.
    *   `headers`: Stores the column headers extracted from the first row.
    *   `processedContent`: Accumulates the processed CSV content (limited to `MAX_PREVIEW_ROWS`).
    *   `sampledRows`: An array to store a sample of rows for metadata.
    *   `errors`: An array to store error messages.
    *   `firstRowProcessed`: A boolean flag to indicate whether the first row (containing headers) has been processed.
    *   `aborted`: A boolean flag to indicate whether the parsing has been aborted due to too many errors.
*   **CSV Parser Options:**

```typescript
const parserOptions: Options = {
    columns: true, // Use first row as headers
    skip_empty_lines: true, // Skip empty lines
    trim: true, // Trim whitespace
    relax_column_count: true, // Allow variable column counts
    relax_quotes: true, // Be lenient with quotes
    skip_records_with_error: true, // Skip bad records
    raw: false,
    cast: false,
  }
  const parser = parse(parserOptions)
```

    *   An object `parserOptions` defines the configuration for the `csv-parse` library. These options control how the CSV data is interpreted:
        *   `columns: true`:  Specifies that the first row of the CSV file should be used as column headers.  The parser will then return each subsequent row as an object where the keys are the column headers.
        *   `skip_empty_lines: true`:  Instructs the parser to ignore empty lines in the CSV file.
        *   `trim: true`:  Tells the parser to trim whitespace from the beginning and end of each field.
        *   `relax_column_count: true`:  Allows rows to have different numbers of columns.  If a row has fewer columns than the header row, the missing values will be treated as `undefined`. If a row has more columns than the header row, the extra columns will be added.
        *   `relax_quotes: true`:  Makes the parser more lenient in handling quotes.  This is useful for CSV files that may have inconsistent or malformed quoting.
        *   `skip_records_with_error: true`: Skips rows that cause parsing errors.
        *   `raw: false`:  This option controls whether the parser returns the raw, unprocessed data. When set to `false`, the parser applies the `cast` function to transform the data.
        *   `cast: false`:  This option disables type casting. By default, `csv-parse` attempts to automatically convert strings to numbers, dates, or booleans. Setting `cast` to `false` prevents this behavior, ensuring that all data is returned as strings.
    *   The `parse(parserOptions)` function creates a CSV parser instance configured with the specified options.
*   **Event Handlers:** The code then defines several event handlers for the `parser` object.  These handlers respond to different events during the parsing process:
    *   `parser.on('readable', ...)`: This event is emitted when the parser has a new record available.
        *   The `while` loop reads records from the parser until there are no more records or the parsing has been aborted.
        *   `rowCount` is incremented for each record processed.
        *   **Header Extraction:** If it's the first row (`!firstRowProcessed`), the code extracts the headers (column names) from the `record` object and stores them in the `headers` array. The `sanitizeTextForUTF8` function is used to clean the header names.  The `processedContent` string is initialized with the headers.
        *   **Content Processing:** If the number of rows processed is less than or equal to `MAX_PREVIEW_ROWS`, the code extracts the values from the record, cleans them using `sanitizeTextForUTF8`, and appends them to the `processedContent` string.
        *   **Sampling:** If the number of rows processed is less than or equal to `MAX_SAMPLE_ROWS`, the code adds the record to the `sampledRows` array.
        *   **Logging Progress:** Every 10,000 rows, the code logs a message indicating the progress of the parsing.
    *   `parser.on('skip', ...)`: This event is emitted when the parser skips a record due to an error.
        *   `errorCount` is incremented.
        *   An error message is created and added to the `errors` array.
        *   If the number of errors exceeds `MAX_ERRORS`, the parsing is aborted.
    *   `parser.on('error', ...)`: This event is emitted when a fatal error occurs during parsing.
        *   The error is logged.
        *   The promise is rejected with an error message.
    *   `parser.on('end', ...)`: This event is emitted when the parser has finished processing the input stream.
        *   If the parsing was not aborted, the code constructs the final `FileParseResult` object.
        *   If the number of rows exceeds `MAX_PREVIEW_ROWS`, a message is added to the `processedContent` string indicating that the output has been truncated.
        *   The `content`, `metadata` (including `rowCount`, `headers`, `errorCount`, `errors`, `truncated`, and `sampledData`), are used to resolve the promise.
    *   `inputStream.on('error', ...)`: This handles errors from the input stream (e.g., file reading errors).
        *   The error is logged.
        *   The parser is destroyed.
        *   The promise is rejected.
*   **Piping the Stream:** Finally, the `inputStream.pipe(parser)` line connects the input stream to the CSV parser. This starts the parsing process. Data flows from the input stream to the parser, which emits events as it processes the data.

**8. `FileParseResult` Structure:**

The `parseStream` method resolves the promise with an object of type `FileParseResult`.  Based on the provided code, this object likely has the following structure:

```typescript
interface FileParseResult {
  content: string;       // The parsed CSV data, limited to MAX_PREVIEW_ROWS
  metadata: {
    rowCount: number;    // The total number of rows in the CSV file
    headers: string[];    // An array of column headers
    errorCount: number;  // The number of errors encountered during parsing
    errors: string[];    // An array of error messages (limited to 10)
    truncated: boolean;  // True if the content was truncated due to MAX_PREVIEW_ROWS
    sampledData: any[];   // An array of sample rows (limited to MAX_SAMPLE_ROWS)
  };
}
```

**Purpose of the File:**

This file defines a robust and configurable CSV parser. It handles parsing CSV data from files or buffers, manages errors, provides metadata about the parsed data, and limits the amount of data processed to prevent memory issues.  It's designed to be used as part of a larger system for processing files, as indicated by the `FileParser` interface and the import paths (e.g., `@/lib/file-parsers/types`).

**Simplifying Complex Logic:**

*   **Configuration:** The use of the `CONFIG` object centralizes and simplifies configuration.  Changing parsing limits or stream chunk sizes only requires modifying this object.
*   **Stream-based Processing:** Using streams (`createReadStream`, `Readable.from`) allows the parser to handle large files efficiently, processing them in chunks and avoiding loading the entire file into memory.
*   **`csv-parse` Library:**  Leveraging the `csv-parse` library abstracts away the complexities of CSV parsing, handling quoting, delimiters, and other CSV-specific details.
*   **Event-Driven Architecture:** The use of event handlers (`parser.on('readable'`, `parser.on('error'`, etc.) makes the parsing logic more modular and easier to understand.
*   **Error Handling:**  The error handling logic prevents the parser from crashing or getting stuck in infinite loops when encountering corrupted files. The `MAX_ERRORS` limit and the `skip_records_with_error` option are key to this.
*   **Abstraction:** The `parseStream` method encapsulates the core parsing logic, making the `parseFile` and `parseBuffer` methods cleaner and easier to read.

In summary, this code provides a well-structured, efficient, and robust CSV parsing solution using streams, a dedicated CSV parsing library, and careful error handling.  It's designed to be reusable and configurable within a larger file processing system.
