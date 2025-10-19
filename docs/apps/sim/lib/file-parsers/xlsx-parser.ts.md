```typescript
import { existsSync } from 'fs'
import * as XLSX from 'xlsx'
import type { FileParseResult, FileParser } from '@/lib/file-parsers/types'
import { sanitizeTextForUTF8 } from '@/lib/file-parsers/utils'
import { createLogger } from '@/lib/logs/console/logger'

// Create a logger instance for this parser.  This allows us to track the parser's activity.
const logger = createLogger('XlsxParser')

// Configuration settings that control how the XLSX file is parsed.
const CONFIG = {
  MAX_PREVIEW_ROWS: 1000, // Only process the first 1000 rows of each sheet for the previewed content to avoid performance issues with large files.
  MAX_SAMPLE_ROWS: 100, //  Extract a sample of the first 100 rows for metadata (e.g., to infer column types).
  ROWS_PER_CHUNK: 50, //  Process rows in chunks of 50. This helps to prevent building up large strings in memory and improves responsiveness.
  MAX_CELL_LENGTH: 1000, // Truncate cell values to a maximum of 1000 characters to prevent very long cells from overwhelming the output.
  MAX_CONTENT_SIZE: 10 * 1024 * 1024, // Limit the total content size to 10MB. This prevents the application from crashing when processing extremely large files.
}

// Defines the XlsxParser class, which implements the FileParser interface. This interface likely defines a contract for parsing different file types.
export class XlsxParser implements FileParser {
  // Implementation of the `parseFile` method from the `FileParser` interface. This method takes a file path as input and returns a `Promise` that resolves to a `FileParseResult` object.
  async parseFile(filePath: string): Promise<FileParseResult> {
    try {
      // Basic validation to ensure a file path is provided.
      if (!filePath) {
        throw new Error('No file path provided')
      }

      // Check if the file exists at the given path.
      if (!existsSync(filePath)) {
        throw new Error(`File not found: ${filePath}`)
      }

      // Log that the parsing process has started.
      logger.info(`Parsing XLSX file: ${filePath}`)

      // Use the `xlsx` library to read the XLSX file. The `readFile` method parses the file and returns a workbook object.
      // `dense: true` optimizes memory usage, especially for large files with many empty cells.
      // `sheetStubs: false` prevents the creation of stub cells, which can also improve performance.
      const workbook = XLSX.readFile(filePath, {
        dense: true, // Use dense mode for better memory efficiency
        sheetStubs: false, // Don't create stub cells
      })

      // Call the `processWorkbook` method to extract and format the data from the workbook.
      return this.processWorkbook(workbook)
    } catch (error) {
      // Catch any errors that occur during the parsing process and log them.
      logger.error('XLSX file parsing error:', error)
      throw new Error(`Failed to parse XLSX file: ${(error as Error).message}`)
    }
  }

  // Implementation of the `parseBuffer` method. This method takes a `Buffer` object (representing the file content in memory) and returns a `Promise` resolving to a `FileParseResult`.
  async parseBuffer(buffer: Buffer): Promise<FileParseResult> {
    try {
      // Get the size of the buffer.
      const bufferSize = buffer.length
      logger.info(
        `Parsing XLSX buffer, size: ${bufferSize} bytes (${(bufferSize / 1024 / 1024).toFixed(2)} MB)`
      )

      // Check if the buffer is empty.
      if (!buffer || buffer.length === 0) {
        throw new Error('Empty buffer provided')
      }

      // Use the `xlsx` library to read the XLSX data from the buffer.
      //  `type: 'buffer'` tells the `xlsx` library that the input is a buffer.
      const workbook = XLSX.read(buffer, {
        type: 'buffer',
        dense: true, // Use dense mode for better memory efficiency
        sheetStubs: false, // Don't create stub cells
      })

      // Process the workbook to extract the data.
      return this.processWorkbook(workbook)
    } catch (error) {
      // Handle any errors that occur during buffer parsing.
      logger.error('XLSX buffer parsing error:', error)
      throw new Error(`Failed to parse XLSX buffer: ${(error as Error).message}`)
    }
  }

  // A private method that performs the core logic of processing the XLSX workbook.
  private processWorkbook(workbook: XLSX.WorkBook): FileParseResult {
    // Get the names of all the sheets in the workbook.
    const sheetNames = workbook.SheetNames
    // Initialize variables to store the extracted content, the total number of rows, a flag indicating if the content was truncated, the size of the content, and a sample of the data.
    let content = ''
    let totalRows = 0
    let truncated = false
    let contentSize = 0
    const sampledData: any[] = []

    // Iterate over each sheet in the workbook.
    for (const sheetName of sheetNames) {
      // Get the worksheet object for the current sheet.
      const worksheet = workbook.Sheets[sheetName]

      // Get sheet dimensions
      const range = XLSX.utils.decode_range(worksheet['!ref'] || 'A1')
      const rowCount = range.e.r - range.s.r + 1

      // Log the start of processing for the current sheet.
      logger.info(`Processing sheet: ${sheetName} with ${rowCount} rows`)

      // Convert the worksheet data to a JSON format.  `header: 1` specifies that the first row should be treated as headers. `defval: ''` sets the default value for empty cells to an empty string. `blankrows: false` skips blank rows.
      const sheetData = XLSX.utils.sheet_to_json(worksheet, {
        header: 1,
        defval: '', // Default value for empty cells
        blankrows: false, // Skip blank rows
      })

      const actualRowCount = sheetData.length
      totalRows += actualRowCount

      // Store limited sample for metadata
      if (sampledData.length < CONFIG.MAX_SAMPLE_ROWS) {
        const sampleSize = Math.min(CONFIG.MAX_SAMPLE_ROWS - sampledData.length, actualRowCount)
        sampledData.push(...sheetData.slice(0, sampleSize))
      }

      // Process only a limited number of rows for preview purposes.
      const rowsToProcess = Math.min(actualRowCount, CONFIG.MAX_PREVIEW_ROWS)
      // Sanitize the sheet name to ensure it's UTF-8 encoded and safe for output.
      const cleanSheetName = sanitizeTextForUTF8(sheetName)

      // Add a sheet header to the content.
      const sheetHeader = `\n=== Sheet: ${cleanSheetName} ===\n`
      content += sheetHeader
      contentSize += sheetHeader.length

      // Process the sheet data if there are rows to process.
      if (actualRowCount > 0) {
        // Get headers if available
        const headers = sheetData[0] as any[]
        if (headers && headers.length > 0) {
          const headerRow = headers.map((h) => this.truncateCell(h)).join('\t')
          content += `${headerRow}\n`
          content += `${'-'.repeat(Math.min(80, headerRow.length))}\n`
          contentSize += headerRow.length + 82
        }

        // Process data rows in chunks
        let chunkContent = ''
        let chunkRowCount = 0

        // Iterate over the rows to be processed.
        for (let i = 1; i < rowsToProcess; i++) {
          const row = sheetData[i] as any[]
          // Check if the row exists and is not empty.
          if (row && row.length > 0) {
            // Convert each cell in the row to a string, truncate it, and join the cells with a tab character.
            const rowString = row.map((cell) => this.truncateCell(cell)).join('\t')

            chunkContent += `${rowString}\n`
            chunkRowCount++

            // Add chunk separator every N rows for better readability
            if (chunkRowCount >= CONFIG.ROWS_PER_CHUNK) {
              content += chunkContent
              contentSize += chunkContent.length
              chunkContent = ''
              chunkRowCount = 0

              // Check content size limit
              if (contentSize > CONFIG.MAX_CONTENT_SIZE) {
                truncated = true
                break
              }
            }
          }
        }

        // Add remaining chunk content
        if (chunkContent && contentSize < CONFIG.MAX_CONTENT_SIZE) {
          content += chunkContent
          contentSize += chunkContent.length
        }

        // If the sheet has more rows than were processed, add a truncation notice.
        if (actualRowCount > rowsToProcess) {
          const notice = `\n[... ${actualRowCount.toLocaleString()} total rows, showing first ${rowsToProcess.toLocaleString()} ...]\n`
          content += notice
          truncated = true
        }
      } else {
        // If the sheet is empty, add a message indicating that.
        content += '[Empty sheet]\n'
      }

      // Stop processing if content is too large
      if (contentSize > CONFIG.MAX_CONTENT_SIZE) {
        content += '\n[... Content truncated due to size limits ...]\n'
        truncated = true
        break
      }
    }

    // Log the completion of the XLSX parsing.
    logger.info(
      `XLSX parsing completed: ${sheetNames.length} sheets, ${totalRows} total rows, truncated: ${truncated}`
    )

    // Sanitize the content to ensure it's UTF-8 encoded.
    const cleanContent = sanitizeTextForUTF8(content).trim()

    // Return the extracted content and metadata.
    return {
      content: cleanContent,
      metadata: {
        sheetCount: sheetNames.length,
        sheetNames: sheetNames,
        totalRows: totalRows,
        truncated: truncated,
        sampledData: sampledData.slice(0, CONFIG.MAX_SAMPLE_ROWS),
        contentSize: contentSize,
      },
    }
  }

  // A private method that truncates a cell value if it exceeds the maximum allowed length.
  private truncateCell(cell: any): string {
    // Handle null or undefined cells by returning an empty string.
    if (cell === null || cell === undefined) {
      return ''
    }

    // Convert the cell value to a string.
    let cellStr = String(cell)

    // Truncate very long cells
    if (cellStr.length > CONFIG.MAX_CELL_LENGTH) {
      cellStr = `${cellStr.substring(0, CONFIG.MAX_CELL_LENGTH)}...`
    }

    // Sanitize the cell string to ensure it's UTF-8 encoded.
    return sanitizeTextForUTF8(cellStr)
  }
}
```
**Explanation:**

**Purpose of the File:**

The TypeScript file `XlsxParser.ts` defines a class, `XlsxParser`, responsible for parsing and extracting data from XLSX (Excel) files. It handles both reading from a file path and reading from a buffer in memory.  The parser extracts the content of the XLSX file in a structured textual format suitable for preview or further processing. It also collects metadata about the file, like the number of sheets, total rows, and whether the content was truncated due to size limitations.  The primary goal is to provide a robust and memory-efficient way to handle potentially large XLSX files, extracting a usable representation of the data while preventing the application from crashing or becoming unresponsive.

**Simplifying Complex Logic:**

1.  **Configuration:** The use of the `CONFIG` object centralizes all configurable parameters (e.g., `MAX_PREVIEW_ROWS`, `MAX_CONTENT_SIZE`).  This makes it easy to adjust the parser's behavior without modifying the core logic.

2.  **Chunking:** The code processes rows in chunks (`ROWS_PER_CHUNK`) to prevent large string concatenations that could lead to memory issues.

3.  **Truncation:** The code truncates both long cell values (`MAX_CELL_LENGTH`) and the overall content (`MAX_CONTENT_SIZE`).

4.  **Separation of Concerns:** The `processWorkbook` function is separated from the `parseFile` and `parseBuffer` functions in order to improve code organization and testability.

5.  **Error Handling:** The code includes comprehensive error handling with informative error messages.

6.  **Sanitization:** All textual content is sanitized using `sanitizeTextForUTF8` to ensure UTF-8 compatibility.

**Line-by-Line Explanation:**

*   **`import { existsSync } from 'fs'`:** Imports the `existsSync` function from the `fs` (file system) module.  `existsSync` is used to check if a file exists at a given path.
*   **`import * as XLSX from 'xlsx'`:** Imports the entire `xlsx` library, aliasing it as `XLSX`.  This library provides functions for reading and writing XLSX files.
*   **`import type { FileParseResult, FileParser } from '@/lib/file-parsers/types'`:** Imports the types `FileParseResult` and `FileParser` from a custom module.  These types likely define the structure of the parser's output and the interface for file parsers, respectively.
*   **`import { sanitizeTextForUTF8 } from '@/lib/file-parsers/utils'`:** Imports the `sanitizeTextForUTF8` function from a custom utility module. This function likely cleans up text data to ensure it's properly encoded in UTF-8, preventing encoding issues.
*   **`import { createLogger } from '@/lib/logs/console/logger'`:** Imports the `createLogger` function from a custom logging module. This allows the parser to log messages for debugging and monitoring.
*   **`const logger = createLogger('XlsxParser')`:** Creates a logger instance specifically for the `XlsxParser` class.  The logger will tag its messages with "XlsxParser", making it easy to filter logs.
*   **`const CONFIG = { ... }`:** Defines a configuration object `CONFIG` to store various settings for the parser.
    *   **`MAX_PREVIEW_ROWS: 1000`:** Limits the number of rows processed for the preview to 1000.
    *   **`MAX_SAMPLE_ROWS: 100`:** Limits the number of rows used for creating metadata samples to 100.
    *   **`ROWS_PER_CHUNK: 50`:** Specifies that rows should be processed in chunks of 50 to prevent memory issues.
    *   **`MAX_CELL_LENGTH: 1000`:** Limits the length of cell values to 1000 characters.
    *   **`MAX_CONTENT_SIZE: 10 * 1024 * 1024`:** Sets the maximum content size to 10MB.
*   **`export class XlsxParser implements FileParser { ... }`:** Defines the `XlsxParser` class, which implements the `FileParser` interface.  This makes this class a concrete implementation of a more general file parser interface.
*   **`async parseFile(filePath: string): Promise<FileParseResult> { ... }`:** Defines the `parseFile` method, which takes a file path as input and returns a `Promise` that resolves to a `FileParseResult` object.  This method reads an XLSX file from the file system.
    *   **`if (!filePath) { throw new Error('No file path provided') }`:** Checks if a file path is provided.  If not, it throws an error.
    *   **`if (!existsSync(filePath)) { throw new Error(\`File not found: ${filePath}\`)}`:** Checks if the file exists. If not, it throws an error.
    *   **`logger.info(\`Parsing XLSX file: ${filePath}\`)`:** Logs a message indicating that the parsing process has started.
    *   **`const workbook = XLSX.readFile(filePath, { dense: true, sheetStubs: false })`:** Reads the XLSX file using the `XLSX.readFile` method, which is a function from the `xlsx` library. Options such as `dense: true` (for efficient memory usage) and `sheetStubs: false` (to avoid creating stub cells) are passed to optimize performance.
    *   **`return this.processWorkbook(workbook)`:** Calls the `processWorkbook` method to extract and format the data from the workbook.
    *   **`catch (error) { ... }`:** Catches any errors that occur during the parsing process.
*   **`async parseBuffer(buffer: Buffer): Promise<FileParseResult> { ... }`:** Defines the `parseBuffer` method, which takes a `Buffer` object (representing the file content in memory) as input and returns a `Promise` that resolves to a `FileParseResult` object.
    *   The logic is almost identical to `parseFile`, but instead of reading from a file path, it reads from a buffer in memory using `XLSX.read(buffer, { type: 'buffer', dense: true, sheetStubs: false })`.
*   **`private processWorkbook(workbook: XLSX.WorkBook): FileParseResult { ... }`:** Defines the `processWorkbook` method, which performs the core logic of processing the XLSX workbook.
    *   **`const sheetNames = workbook.SheetNames`:** Gets the names of all the sheets in the workbook.
    *   **`let content = ''; let totalRows = 0; let truncated = false; let contentSize = 0; const sampledData: any[] = []`:** Initializes variables to store the extracted content, the total number of rows, a flag indicating if the content was truncated, the size of the content, and a sample of the data.
    *   The code iterates through each sheet in the workbook (`for (const sheetName of sheetNames) { ... }`).
    *   **`const worksheet = workbook.Sheets[sheetName]`:** Gets the worksheet object for the current sheet.
    *   **`const range = XLSX.utils.decode_range(worksheet['!ref'] || 'A1')`**: Determines the dimensions (row and column count) of the worksheet.
    *   **`const rowCount = range.e.r - range.s.r + 1`**: Calculate the total number of rows in the work sheet.
    *   **`const sheetData = XLSX.utils.sheet_to_json(worksheet, { header: 1, defval: '', blankrows: false })`:** Converts the worksheet data to a JSON format using the `XLSX.utils.sheet_to_json` method.  `header: 1` specifies that the first row should be treated as headers, `defval: ''` sets the default value for empty cells to an empty string and `blankrows: false` ignores blank rows.
    *   The code then appends the sheet name as a header in content, and adds a header row to content if available. Processes each row and truncates the cell to `MAX_CELL_LENGTH`. The `contentSize` is updated accordingly.
    *   It limits the total content processed, and truncates if the total size is greater than `MAX_CONTENT_SIZE`. Also provides messages like  `[... Content truncated due to size limits ...]` if truncation occurs.
    *   `const cleanContent = sanitizeTextForUTF8(content).trim()`: calls the imported function to replace any non UTF-8 characters with UTF-8 characters, and also trim any leading or trailing spaces.
    *   Finally it returns the extracted `content` and `metadata`. The metadata object includes the following:
        *   `sheetCount`: The number of sheets in the workbook.
        *   `sheetNames`: An array of sheet names.
        *   `totalRows`: The total number of rows across all sheets.
        *   `truncated`: A boolean indicating whether the content was truncated.
        *   `sampledData`: A sample of the data from the first few rows.
        *   `contentSize`: The size of the extracted content.

*   **`private truncateCell(cell: any): string { ... }`:** Defines the `truncateCell` method, which truncates a cell value if it exceeds the maximum allowed length (`CONFIG.MAX_CELL_LENGTH`). It handles `null` and `undefined` values, converts the cell value to a string, truncates it if necessary, and sanitizes the result using `sanitizeTextForUTF8`.

**In Summary:**

This `XlsxParser` class provides a well-structured and robust solution for parsing XLSX files in a memory-efficient and controlled manner. It includes comprehensive error handling, configuration options, and data sanitization to ensure reliable and safe data extraction. The code is designed to handle large files gracefully by limiting the amount of data processed and truncating where necessary. It is well-documented, making it easy to understand and maintain.
