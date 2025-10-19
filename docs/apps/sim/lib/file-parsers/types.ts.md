```typescript
// Purpose: This file defines interfaces and types related to parsing various file types.
// It provides a contract for file parsers and specifies the structure of the parsing results,
// along with a list of supported file types.  This promotes code reusability and maintainability
// by establishing clear expectations for how file parsing should be handled within the application.

// Interface: FileParseResult
// Defines the structure of the object returned after parsing a file.
export interface FileParseResult {
  // content: string
  // The extracted textual content from the file.  This is the main output of the parsing process.
  content: string

  // metadata?: Record<string, any>
  // Optional metadata associated with the file. This could include information like author,
  // creation date, page count, etc.  It's a record (object) where keys are strings and values can be of any type.
  // The `?` makes this property optional, meaning a parser might not always provide metadata.
  metadata?: Record<string, any>
}

// Interface: FileParser
// Defines the contract for any class that will handle file parsing.  It specifies the methods
// that a file parser must implement.
export interface FileParser {
  // parseFile(filePath: string): Promise<FileParseResult>
  // This method is required. It takes the file path as input and returns a Promise that resolves
  // to a FileParseResult object.  The `Promise` indicates that the parsing operation is asynchronous.
  parseFile(filePath: string): Promise<FileParseResult>

  // parseBuffer?(buffer: Buffer): Promise<FileParseResult>
  // This method is optional. It takes a Buffer (representing the file content in memory) as input
  // and returns a Promise that resolves to a FileParseResult object.  This allows parsers to work
  // directly with file data already loaded into memory, avoiding the need to read from a file path.
  // The `?` makes this method optional, meaning a parser might not implement it.
  parseBuffer?(buffer: Buffer): Promise<FileParseResult>
}

// Type: SupportedFileType
// Defines a type alias for the string literals representing the supported file types.
// This provides type safety and autocompletion when working with file types.
export type SupportedFileType =
  | 'pdf'
  | 'csv'
  | 'doc'
  | 'docx'
  | 'txt'
  | 'md'
  | 'xlsx'
  | 'xls'
  | 'html'
  | 'htm'
  | 'pptx'
  | 'ppt'
```

**Explanation Breakdown:**

1. **`FileParseResult` Interface:**

   - This interface acts as a blueprint for the data returned by a file parser.  It's crucial for ensuring consistency across different file parser implementations.
   - `content: string`:  The most important part - the extracted text from the file.  All parsers *must* provide this.
   - `metadata?: Record<string, any>`: An optional dictionary of key-value pairs.  Think of it as "extra information" about the file.  The `?` means this can be missing. Using `Record<string, any>` allows for flexible metadata structures without being overly restrictive.

2. **`FileParser` Interface:**

   - This interface defines what a file parser *must* be able to do.  Any class implementing this interface guarantees that it has at least a `parseFile` method.
   - `parseFile(filePath: string): Promise<FileParseResult>`:  This *required* method takes a file path as input (a string like `/path/to/my/file.pdf`).  It *must* return a `Promise` that resolves to a `FileParseResult` object.  The `Promise` is important because file parsing can be slow, and using a Promise allows the application to continue doing other things while the parsing happens in the background.  It ensures asynchronous execution, preventing the UI from freezing.
   - `parseBuffer?(buffer: Buffer): Promise<FileParseResult>`:  This *optional* method is useful when you already have the file content in memory as a `Buffer`.  Instead of forcing the parser to read from disk, you can pass the `Buffer` directly.  The `?` makes it optional. This is helpful in scenarios where the file is uploaded or already in memory for some other reason. Buffers represent raw binary data.

3. **`SupportedFileType` Type:**

   - This is a *type alias* that creates a more specific type named `SupportedFileType`. It's essentially a list of allowed string values.
   - The `|` symbol represents a *union type*. This means a variable of type `SupportedFileType` can only hold one of the specified string values (e.g., 'pdf', 'csv', 'doc', etc.).
   - This type is very helpful for:
     - **Type safety:**  The TypeScript compiler will catch errors if you try to use a file type that's not in the list.
     - **Autocompletion:**  When you're writing code that uses `SupportedFileType`, your IDE will suggest the valid file types.
     - **Readability:**  It makes the code easier to understand because it clearly defines the allowed values.

**In summary:**

This file provides a set of building blocks for handling file parsing in a type-safe and organized way.  It defines:

- The expected structure of the parsing results (`FileParseResult`).
- The contract that file parsers must adhere to (`FileParser`).
- A list of supported file types (`SupportedFileType`).

These definitions help to ensure that different file parsing components can work together seamlessly and that the application can handle a variety of file formats in a consistent manner.  The use of interfaces and type aliases promotes code reusability, maintainability, and type safety.
