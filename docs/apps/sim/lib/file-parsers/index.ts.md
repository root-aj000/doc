```typescript
import { existsSync } from 'fs'
import path from 'path'
import type { FileParseResult, FileParser, SupportedFileType } from '@/lib/file-parsers/types'
import { createLogger } from '@/lib/logs/console/logger'

// Creates a logger instance for this module, named 'FileParser'.  This allows for easy filtering
// of logs originating from this file.
const logger = createLogger('FileParser')

// This variable will hold the parser instances.  It's initialized to null and populated lazily
// the first time `getParserInstances` is called.  Using `null` here allows us to easily check
// if the parsers have been initialized yet.
let parserInstances: Record<string, FileParser> | null = null

/**
 * Get parser instances with lazy initialization
 *
 * This function implements a singleton pattern with lazy initialization. It ensures that
 * parser instances are only created once when they are actually needed, improving performance
 * by avoiding unnecessary initialization at application startup.
 * @returns {Record<string, FileParser>} - An object mapping file extensions (e.g., "pdf", "csv")
 * to their corresponding parser instances.
 */
function getParserInstances(): Record<string, FileParser> {
  // Check if the parser instances have already been created.
  if (parserInstances === null) {
    // If not, initialize an empty object to store the parsers.
    parserInstances = {}

    // Use a try-catch block to handle potential errors during parser loading. This prevents
    // the entire application from crashing if a specific parser fails to load.
    try {
      // The following try-catch blocks attempt to dynamically load parser modules.
      // Dynamic loading is used to avoid bundling all parsers into the application if they
      // are not needed, reducing the initial bundle size.  Each block attempts to load a
      // specific parser. If the loading is successful, a new instance of the parser is created
      // and added to the `parserInstances` object, using the file extension as the key (e.g., 'pdf', 'csv').

      // PDF Parser
      try {
        logger.info('Loading PDF parser...')
        // Dynamically import the PdfParser. The `require` statement is used here because the original code used `require`.
        const { PdfParser } = require('@/lib/file-parsers/pdf-parser')
        // Create a new instance of the PdfParser and store it in the parserInstances object.
        parserInstances.pdf = new PdfParser()
        logger.info('PDF parser loaded successfully')
      } catch (error) {
        // Log an error message if the PDF parser fails to load.
        logger.error('Failed to load PDF parser:', error)
      }

      // CSV Parser
      try {
        const { CsvParser } = require('@/lib/file-parsers/csv-parser')
        parserInstances.csv = new CsvParser()
        logger.info('Loaded streaming CSV parser with csv-parse library')
      } catch (error) {
        logger.error('Failed to load streaming CSV parser:', error)
      }

      // DOCX Parser
      try {
        const { DocxParser } = require('@/lib/file-parsers/docx-parser')
        parserInstances.docx = new DocxParser()
      } catch (error) {
        logger.error('Failed to load DOCX parser:', error)
      }

      // DOC Parser
      try {
        const { DocParser } = require('@/lib/file-parsers/doc-parser')
        parserInstances.doc = new DocParser()
      } catch (error) {
        logger.error('Failed to load DOC parser:', error)
      }

      // TXT Parser
      try {
        const { TxtParser } = require('@/lib/file-parsers/txt-parser')
        parserInstances.txt = new TxtParser()
      } catch (error) {
        logger.error('Failed to load TXT parser:', error)
      }

      // MD Parser
      try {
        const { MdParser } = require('@/lib/file-parsers/md-parser')
        parserInstances.md = new MdParser()
      } catch (error) {
        logger.error('Failed to load MD parser:', error)
      }

      // XLSX Parser
      try {
        const { XlsxParser } = require('@/lib/file-parsers/xlsx-parser')
        parserInstances.xlsx = new XlsxParser()
        parserInstances.xls = new XlsxParser()
        logger.info('Loaded XLSX parser')
      } catch (error) {
        logger.error('Failed to load XLSX parser:', error)
      }

      // PPTX Parser
      try {
        const { PptxParser } = require('@/lib/file-parsers/pptx-parser')
        parserInstances.pptx = new PptxParser()
        parserInstances.ppt = new PptxParser()
      } catch (error) {
        logger.error('Failed to load PPTX parser:', error)
      }

      // HTML Parser
      try {
        const { HtmlParser } = require('@/lib/file-parsers/html-parser')
        parserInstances.html = new HtmlParser()
        parserInstances.htm = new HtmlParser()
      } catch (error) {
        logger.error('Failed to load HTML parser:', error)
      }

      // JSON Parser
      try {
        // JSON parser uses a different structure as it exports function to parse file and buffer
        const { parseJSON, parseJSONBuffer } = require('@/lib/file-parsers/json-parser')
        parserInstances.json = {
          parseFile: parseJSON,
          parseBuffer: parseJSONBuffer,
        }
        logger.info('Loaded JSON parser')
      } catch (error) {
        logger.error('Failed to load JSON parser:', error)
      }

      // YAML Parser
      try {
        // YAML parser uses a different structure as it exports function to parse file and buffer
        const { parseYAML, parseYAMLBuffer } = require('@/lib/file-parsers/yaml-parser')
        parserInstances.yaml = {
          parseFile: parseYAML,
          parseBuffer: parseYAMLBuffer,
        }
        parserInstances.yml = {
          parseFile: parseYAML,
          parseBuffer: parseYAMLBuffer,
        }
        logger.info('Loaded YAML parser')
      } catch (error) {
        logger.error('Failed to load YAML parser:', error)
      }
    } catch (error) {
      // Log a general error message if any error occurs during parser loading.
      logger.error('Error loading file parsers:', error)
    }
  }

  // Return the parser instances.
  return parserInstances
}

/**
 * Parse a file based on its extension
 * @param filePath Path to the file
 * @returns Parsed content and metadata
 */
export async function parseFile(filePath: string): Promise<FileParseResult> {
  // Wrap the entire function in a try-catch block to handle any potential errors during file parsing.
  try {
    // Validate the file path.  Throw an error if it's missing.
    if (!filePath) {
      throw new Error('No file path provided')
    }

    // Check if the file exists.  Throw an error if it doesn't.
    if (!existsSync(filePath)) {
      throw new Error(`File not found: ${filePath}`)
    }

    // Extract the file extension from the file path, convert it to lowercase, and remove the leading dot.
    const extension = path.extname(filePath).toLowerCase().substring(1)
    logger.info('Attempting to parse file with extension:', extension)

    // Get the parser instances. This will initialize them if they haven't been already.
    const parsers = getParserInstances()

    // Check if a parser exists for the given file extension.
    if (!Object.keys(parsers).includes(extension)) {
      logger.info('No parser found for extension:', extension)
      // If no parser is found, throw an error indicating that the file type is not supported.
      throw new Error(
        `Unsupported file type: ${extension}. Supported types are: ${Object.keys(parsers).join(', ')}`
      )
    }

    // Log that the respective parser will be used.
    logger.info('Using parser for extension:', extension)
    // Get the appropriate parser from the `parsers` object using the file extension as the key.
    const parser = parsers[extension]
    // Call the `parseFile` method of the parser to parse the file and return the result.
    // The `await` keyword is used because the `parseFile` method is likely asynchronous.
    return await parser.parseFile(filePath)
  } catch (error) {
    // Log any errors that occur during file parsing.
    logger.error('File parsing error:', error)
    // Re-throw the error to allow the calling function to handle it.
    throw error
  }
}

/**
 * Parse a buffer based on file extension
 * @param buffer Buffer containing the file data
 * @param extension File extension without the dot (e.g., 'pdf', 'csv')
 * @returns Parsed content and metadata
 */
export async function parseBuffer(buffer: Buffer, extension: string): Promise<FileParseResult> {
  try {
    // Validate the buffer. Throw an error if it's empty or null.
    if (!buffer || buffer.length === 0) {
      throw new Error('Empty buffer provided')
    }

    // Validate the file extension. Throw an error if it's missing.
    if (!extension) {
      throw new Error('No file extension provided')
    }

    // Normalize the extension to lowercase for consistency.
    const normalizedExtension = extension.toLowerCase()
    logger.info('Attempting to parse buffer with extension:', normalizedExtension)

    // Get the parser instances.
    const parsers = getParserInstances()

    // Check if a parser exists for the given file extension.
    if (!Object.keys(parsers).includes(normalizedExtension)) {
      logger.info('No parser found for extension:', normalizedExtension)
      // If no parser is found, throw an error indicating that the file type is not supported.
      throw new Error(
        `Unsupported file type: ${normalizedExtension}. Supported types are: ${Object.keys(parsers).join(', ')}`
      )
    }

    // Log the parser that will be used.
    logger.info('Using parser for extension:', normalizedExtension)
    // Get the appropriate parser from the `parsers` object using the normalized file extension.
    const parser = parsers[normalizedExtension]

    // Check if the parser supports parsing from a buffer.
    if (parser.parseBuffer) {
      // If it does, call the `parseBuffer` method of the parser to parse the buffer and return the result.
      // The `await` keyword is used because the `parseBuffer` method is likely asynchronous.
      return await parser.parseBuffer(buffer)
    }
    // If the parser does not support buffer parsing, throw an error.
    throw new Error(`Parser for ${normalizedExtension} does not support buffer parsing`)
  } catch (error) {
    // Log any errors that occur during buffer parsing.
    logger.error('Buffer parsing error:', error)
    // Re-throw the error to allow the calling function to handle it.
    throw error
  }
}

/**
 * Check if a file type is supported
 * @param extension File extension without the dot
 * @returns true if supported, false otherwise
 */
export function isSupportedFileType(extension: string): extension is SupportedFileType {
  try {
    // Check if the given extension exists as a key in the `parserInstances` object.
    return Object.keys(getParserInstances()).includes(extension.toLowerCase())
  } catch (error) {
    // Log the error if one occurs
    logger.error('Error checking supported file type:', error)
    // if error occurs, return false
    return false
  }
}

// Re-export the types to allow them to be used in other modules. This is good practice as it avoids
// the need to import them from the specific file parser modules.
export type { FileParseResult, FileParser, SupportedFileType }
```

**Explanation:**

**Purpose of this file:**

This TypeScript file provides a unified interface for parsing different types of files (e.g., PDF, CSV, DOCX, etc.). It uses a strategy pattern to dynamically load and utilize the appropriate parser based on the file extension.  The main goals are:

1.  **Abstraction:** Hide the complexity of using different file parsing libraries.
2.  **Extensibility:** Easily add support for new file types by creating a new parser and registering it.
3.  **Lazy Loading:** Only load the necessary parsers, improving performance.
4.  **Error Handling:** Centralized error handling for file parsing operations.

**Simplified Logic & Key Improvements:**

*   **Lazy Initialization (Singleton):**  The `getParserInstances` function ensures that parser instances are only created once when first needed. This avoids unnecessary overhead during application startup. The `parserInstances` variable acts as a cache.
*   **Dynamic Loading:** Uses `require()` to load parser modules. This is a form of dynamic importing, which helps reduce the initial bundle size. Error handling is implemented within each try-catch block to prevent failures in one parser from crashing the entire parsing system.
*   **Centralized Error Handling:** Wraps the main functions (`parseFile`, `parseBuffer`, `isSupportedFileType`) in try-catch blocks to handle potential errors during file operations and parser usage.
*   **Type Safety:** Uses TypeScript's type system extensively to ensure type safety.  This is critical for maintainability and reducing runtime errors.
*   **Clear Logging:**  Uses a logger for informational and error messages, which can be invaluable for debugging.

**Line-by-Line Explanation:**

1.  `import { existsSync } from 'fs'`
    *   Imports the `existsSync` function from the `fs` (file system) module.  `existsSync` synchronously checks if a file exists at the given path.

2.  `import path from 'path'`
    *   Imports the `path` module, which provides utilities for working with file and directory paths.

3.  `import type { FileParseResult, FileParser, SupportedFileType } from '@/lib/file-parsers/types'`
    *   Imports TypeScript *types* (not values) from a module related to file parsing.  These types likely define:
        *   `FileParseResult`: The structure of the object returned after a successful file parse (e.g., content, metadata).
        *   `FileParser`: An interface or type defining the expected methods for a file parser (e.g., `parseFile`, `parseBuffer`).
        *   `SupportedFileType`: A type representing the valid file extensions that are supported by the system. The `type` import indicates that these are only used for compile-time type checking and won't be included in the runtime code.

4.  `import { createLogger } from '@/lib/logs/console/logger'`
    *   Imports the `createLogger` function, presumably from a custom logging module.  This function is used to create a logger instance specific to this file.

5.  `const logger = createLogger('FileParser')`
    *   Creates a logger instance named 'FileParser'.  This allows you to filter logs specifically from this file when debugging.

6.  `let parserInstances: Record<string, FileParser> | null = null`
    *   Declares a variable `parserInstances`.
        *   `Record<string, FileParser>`:  A TypeScript type that represents an object where the keys are strings (file extensions like "pdf", "csv") and the values are `FileParser` objects (instances of the parser classes).
        *   `| null`:  Indicates that `parserInstances` can be either a `Record<string, FileParser>` or `null`.
        *   `= null`:  Initializes `parserInstances` to `null`. This is the key to lazy loading.

7.  `/** ... */`
    *   This is a JSDoc comment block, providing documentation for the `getParserInstances` function.

8.  `function getParserInstances(): Record<string, FileParser> {`
    *   Defines a function called `getParserInstances`.
        *   `(): Record<string, FileParser>`:  The function takes no arguments and returns a `Record<string, FileParser>` object.

9.  `if (parserInstances === null) {`
    *   Checks if `parserInstances` is currently `null`. This is the lazy initialization check.  If it's `null`, it means the parsers haven't been loaded yet.

10. `parserInstances = {}`
    *   If `parserInstances` is `null`, this line initializes it to an empty object.  This will store the parser instances.

11. `try { ... } catch (error) { ... }`
    *   This is a main `try...catch` block that handles any errors that might occur during the parser loading process. This provides a general error handling.

12. `try { ... } catch (error) { ... }` (nested)
    *   Each of these nested `try...catch` blocks attempts to load a specific parser module. This prevents a failure in loading one parser from crashing the whole file parsing system.

13. `logger.info('Loading PDF parser...')`
    *   Logs an informational message indicating that the PDF parser is about to be loaded.

14. `const { PdfParser } = require('@/lib/file-parsers/pdf-parser')`
    *   Dynamically imports the `PdfParser` class from the `@/lib/file-parsers/pdf-parser` module using `require`.
    *   `require()` is a Node.js function used to load modules. The `const { PdfParser }` syntax is destructuring, extracting the `PdfParser` export from the required module.
    *   **Important:** Using `require` instead of `import` allows for dynamic loading, which is important for lazy initialization. This avoids bundling all parsers upfront.

15. `parserInstances.pdf = new PdfParser()`
    *   Creates a new instance of the `PdfParser` class and assigns it to the `pdf` property of the `parserInstances` object.  This associates the "pdf" extension with the PDF parser.

16. `logger.info('PDF parser loaded successfully')`
    *   Logs an informational message indicating that the PDF parser was loaded successfully.

17. `logger.error('Failed to load PDF parser:', error)`
    *   Logs an error message if the PDF parser fails to load, including the error object for debugging.

18.  `const { CsvParser } = require('@/lib/file-parsers/csv-parser')`
    *   Similar to the PDF parser, this line dynamically imports the `CsvParser` class from the `@/lib/file-parsers/csv-parser` module.

19.  `parserInstances.csv = new CsvParser()`
    *   Creates a new instance of the `CsvParser` class and assigns it to the `csv` property of the `parserInstances` object.

20.  `const { parseJSON, parseJSONBuffer } = require('@/lib/file-parsers/json-parser')`
    *  JSON parser exports functions rather than a class. This line imports the `parseJSON` and `parseJSONBuffer` functions from the `@/lib/file-parsers/json-parser` module

21.   `parserInstances.json = {
          parseFile: parseJSON,
          parseBuffer: parseJSONBuffer,
        }`
    *  This creates `json` entry within `parserInstances` that contains two functions, `parseFile` and `parseBuffer` to be consistent with the `FileParser` interface

22. `return parserInstances`
    *   Returns the `parserInstances` object.  If this is the first time the function is called, it will return the newly created object with all the loaded parsers.  Subsequent calls will return the same object, ensuring that the parsers are only loaded once.

23. `export async function parseFile(filePath: string): Promise<FileParseResult> {`
    *   Defines an asynchronous function called `parseFile`.
        *   `filePath: string`:  The function takes a single argument, `filePath`, which is a string representing the path to the file.
        *   `: Promise<FileParseResult>`:  The function returns a `Promise` that resolves to a `FileParseResult` object.

24. `if (!filePath) { ... }`
    *   Checks if the `filePath` argument is empty or null.  If it is, it throws an error.

25. `if (!existsSync(filePath)) { ... }`
    *   Uses `existsSync` to check if the file at the given `filePath` actually exists.  If it doesn't, it throws an error.

26. `const extension = path.extname(filePath).toLowerCase().substring(1)`
    *   Extracts the file extension from the `filePath`.
        *   `path.extname(filePath)`:  Gets the file extension (including the leading dot, e.g., ".pdf").
        *   `.toLowerCase()`:  Converts the extension to lowercase for case-insensitive matching.
        *   `.substring(1)`:  Removes the leading dot from the extension (e.g., "pdf").

27. `const parsers = getParserInstances()`
    *   Calls the `getParserInstances` function to get the object containing the parser instances.  This ensures that the parsers are loaded (if they haven't been already).

28. `if (!Object.keys(parsers).includes(extension)) { ... }`
    *   Checks if a parser exists for the extracted `extension`.
        *   `Object.keys(parsers)`:  Gets an array of the keys (file extensions) in the `parsers` object.
        *   `.includes(extension)`:  Checks if the `extension` is present in the array of keys.
        *   If the extension is not found, it means there's no registered parser for that file type, so it throws an error.

29. `const parser = parsers[extension]`
    *   Retrieves the appropriate parser from the `parsers` object using the `extension` as the key.

30. `return await parser.parseFile(filePath)`
    *   Calls the `parseFile` method of the selected `parser` instance, passing the `filePath` as an argument.
    *   `await`:  Since `parseFile` is likely an asynchronous function, `await` is used to wait for the promise to resolve and get the result.
    *   This returns the `FileParseResult` from the parser

31. `export async function parseBuffer(buffer: Buffer, extension: string): Promise<FileParseResult> {`
    *   Defines an asynchronous function called `parseBuffer` that parses a file from a buffer.  This is useful when the file data is already in memory.  It has similar logic to `parseFile`, but operates on a buffer instead of a file path.

32. `if (parser.parseBuffer) { ... }`
    *   Before calling `parser.parseBuffer`, this checks if the parser actually *has* a `parseBuffer` method. Not all parsers might support parsing directly from a buffer.

33. `return await parser.parseBuffer(buffer)`
    *   If the parser has a `parseBuffer` method, this line calls it with the `buffer` and returns the result.

34. `throw new Error(\`Parser for ${normalizedExtension} does not support buffer parsing\`)`
    *   If the selected parser does not implement `parseBuffer`, an error is thrown.

35. `export function isSupportedFileType(extension: string): extension is SupportedFileType {`
    *   Defines a function that checks if a given file extension is supported by the system.
        *   `extension: string`:  The file extension to check.
        *   `: extension is SupportedFileType`:  This is a TypeScript *type predicate*.  It means that if the function returns `true`, TypeScript will treat the `extension` variable as being of type `SupportedFileType` within the scope where the function was called. This is a powerful way to narrow the type of a variable based on a runtime check.

36. `return Object.keys(getParserInstances()).includes(extension.toLowerCase())`
    *   Gets the list of supported extensions (from the keys of the `parserInstances` object) and checks if the given `extension` (converted to lowercase) is in that list.

37. `export type { FileParseResult, FileParser, SupportedFileType }`
    *   Re-exports the `FileParseResult`, `FileParser`, and `SupportedFileType` types.  This allows other modules to import these types directly from this file, rather than having to import them from the `@/lib/file-parsers/types` module.  This can make the code cleaner and easier to understand.

**In Summary:**

This file provides a robust and flexible system for parsing various file types. Its use of lazy loading, dynamic imports, and comprehensive error handling ensures that it's both performant and reliable. The type safety provided by TypeScript further enhances maintainability and reduces the risk of runtime errors. This module uses a `require` statements to dynamically import file parsers, allowing lazy loading the parser. Because this codebase uses `require` statements instead of `import` statements, it indicates that the project is likely using CommonJS module system.
