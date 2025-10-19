```typescript
import * as yaml from 'js-yaml'
import type { FileParseResult } from './types'

/**
 * Parse YAML files
 */
export async function parseYAML(filePath: string): Promise<FileParseResult> {
  // Dynamically import the 'fs/promises' module. This allows the code to run in environments where file system access may not be immediately available (e.g., browsers).
  const fs = await import('fs/promises')
  // Read the file content asynchronously from the given file path, using UTF-8 encoding.  The 'await' keyword ensures that the file is fully read before proceeding.
  const content = await fs.readFile(filePath, 'utf-8')

  try {
    // Parse YAML to validate and extract structure
    // Use the 'js-yaml' library to parse the YAML content into a JavaScript object.  This step also validates the YAML syntax.
    const yamlData = yaml.load(content)

    // Convert to JSON for consistent processing
    // Convert the parsed YAML data into a JSON string.  'JSON.stringify' with 'null, 2' arguments formats the JSON with 2 spaces for indentation, improving readability.  This ensures a consistent data format for later processing, regardless of the input format (YAML).
    const jsonContent = JSON.stringify(yamlData, null, 2)

    // Extract metadata about the YAML structure
    // Create an object to store metadata about the parsed YAML data.  This metadata can be used for various purposes, such as determining the structure and complexity of the data.
    const metadata = {
      // Indicates that the original file was YAML.
      type: 'yaml',
      // Checks if the parsed YAML data is an array.
      isArray: Array.isArray(yamlData),
      // If 'yamlData' is an array, 'keys' will be an empty array.  Otherwise, it will be an array of the keys in the 'yamlData' object.  The '|| {}' handles the case where 'yamlData' is null or undefined.
      keys: Array.isArray(yamlData) ? [] : Object.keys(yamlData || {}),
      // If 'yamlData' is an array, 'itemCount' will be the length of the array.  Otherwise, it will be undefined.
      itemCount: Array.isArray(yamlData) ? yamlData.length : undefined,
      // Calculate the depth (nesting level) of the YAML data structure.
      depth: getYamlDepth(yamlData),
    }

    // Return the parsed content (in JSON format) and the extracted metadata.
    return {
      content: jsonContent,
      metadata,
    }
  } catch (error) {
    // If any error occurs during the YAML parsing or JSON conversion, throw a new error with a descriptive message.  The 'error instanceof Error ? error.message : 'Unknown error'' part safely extracts the error message if the error object is an instance of the Error class.
    throw new Error(`Invalid YAML: ${error instanceof Error ? error.message : 'Unknown error'}`)
  }
}

/**
 * Parse YAML from buffer
 */
export async function parseYAMLBuffer(buffer: Buffer): Promise<FileParseResult> {
  // Convert the Buffer to a UTF-8 string.  This assumes the YAML content is encoded in UTF-8.
  const content = buffer.toString('utf-8')

  try {
    // Use the 'js-yaml' library to parse the YAML content into a JavaScript object.  This step also validates the YAML syntax.
    const yamlData = yaml.load(content)
    // Convert the parsed YAML data into a JSON string.  'JSON.stringify' with 'null, 2' arguments formats the JSON with 2 spaces for indentation, improving readability.  This ensures a consistent data format for later processing, regardless of the input format (YAML).
    const jsonContent = JSON.stringify(yamlData, null, 2)

    // Create an object to store metadata about the parsed YAML data.  This metadata can be used for various purposes, such as determining the structure and complexity of the data.
    const metadata = {
      // Indicates that the original file was YAML.
      type: 'yaml',
      // Checks if the parsed YAML data is an array.
      isArray: Array.isArray(yamlData),
      // If 'yamlData' is an array, 'keys' will be an empty array.  Otherwise, it will be an array of the keys in the 'yamlData' object.  The '|| {}' handles the case where 'yamlData' is null or undefined.
      keys: Array.isArray(yamlData) ? [] : Object.keys(yamlData || {}),
      // If 'yamlData' is an array, 'itemCount' will be the length of the array.  Otherwise, it will be undefined.
      itemCount: Array.isArray(yamlData) ? yamlData.length : undefined,
      // Calculate the depth (nesting level) of the YAML data structure.
      depth: getYamlDepth(yamlData),
    }

    // Return the parsed content (in JSON format) and the extracted metadata.
    return {
      content: jsonContent,
      metadata,
    }
  } catch (error) {
    // If any error occurs during the YAML parsing or JSON conversion, throw a new error with a descriptive message.  The 'error instanceof Error ? error.message : 'Unknown error'' part safely extracts the error message if the error object is an instance of the Error class.
    throw new Error(`Invalid YAML: ${error instanceof Error ? error.message : 'Unknown error'}`)
  }
}

/**
 * Calculate the depth of a YAML/JSON object
 */
function getYamlDepth(obj: any): number {
  // Base case: If the object is null or not an object (e.g., a primitive value), the depth is 0.
  if (obj === null || typeof obj !== 'object') return 0

  // If the object is an array:
  if (Array.isArray(obj)) {
    // If the array is empty, the depth is 1.
    // Otherwise, the depth is 1 plus the maximum depth of any of its elements.  This uses recursion to calculate the depth of each element.
    return obj.length > 0 ? 1 + Math.max(...obj.map(getYamlDepth)) : 1
  }

  // If the object is a regular object:
  // Get an array of the values of the object.
  const depths = Object.values(obj).map(getYamlDepth)
  // If the object is empty, the depth is 1.
  // Otherwise, the depth is 1 plus the maximum depth of any of its values.  This uses recursion to calculate the depth of each value.
  return depths.length > 0 ? 1 + Math.max(...depths) : 1
}
```

**Purpose of this file:**

This TypeScript file provides functions for parsing YAML (Yet Another Markup Language) data. It includes functionalities to:

1.  **Parse YAML from a file:** Reads a YAML file from a specified path, parses its content, converts it into a JSON string, and extracts metadata about the YAML structure.
2.  **Parse YAML from a buffer:**  Parses YAML content directly from a `Buffer` object (which might come from a network request or other source).  It performs the same parsing, JSON conversion, and metadata extraction as the file-based function.
3.  **Calculate YAML depth:** Determines the maximum nesting level (depth) of a YAML/JSON object.

The parsed content is converted to JSON to provide a consistent data format.  Metadata extracted includes the data type, whether it's an array, keys (if it's an object), number of items (if it's an array), and its depth. This is helpful for understanding the structure of the YAML data.

**Simplifying Complex Logic:**

*   **Dynamic Import:** The `fs/promises` module is imported dynamically using `await import('fs/promises')`. This makes the code more flexible, as it only loads the file system module when needed, and it can be used in environments where the file system may not be immediately available (like browsers with bundlers).
*   **Error Handling:**  The `try...catch` blocks provide robust error handling. Specifically, the `error instanceof Error ? error.message : 'Unknown error'` ensures that a meaningful error message is provided even if the caught error is not an instance of the `Error` class.
*   **Metadata Extraction:**  The metadata extraction logic is clearly separated, making it easy to understand what properties are being gathered about the YAML data. The ternary operators make the code concise and readable.
*   **Recursive Depth Calculation:** The `getYamlDepth` function effectively uses recursion to calculate the nesting depth of the YAML structure. It handles both arrays and objects.

**Explanation of each line of code:**

**Imports:**

*   `import * as yaml from 'js-yaml'`:  Imports the `js-yaml` library, which is used for parsing YAML data.  The `* as yaml` syntax imports all exports from the library into a namespace called `yaml`.
*   `import type { FileParseResult } from './types'`: Imports the `FileParseResult` type definition from a local file named `types.ts`. This type likely defines the structure of the object returned by the parsing functions (i.e., it defines the structure of the `{ content: string; metadata: ... }` object).

**`parseYAML` function:**

*   `export async function parseYAML(filePath: string): Promise<FileParseResult>`: Defines an asynchronous function named `parseYAML` that takes a file path (`filePath`) as input (a string) and returns a `Promise` that resolves to a `FileParseResult` object.  The `export` keyword makes this function available for use in other modules.
*   `const fs = await import('fs/promises')`: Dynamically imports the `fs/promises` module, which provides asynchronous file system operations.  The `await` keyword ensures that the module is fully loaded before proceeding.
*   `const content = await fs.readFile(filePath, 'utf-8')`: Reads the content of the file specified by `filePath` asynchronously.  The `'utf-8'` argument specifies that the file should be read using UTF-8 encoding. The `await` keyword makes sure the file is completely read before execution continues.
*   `try { ... } catch (error) { ... }`: A `try...catch` block is used for error handling.  The code within the `try` block is executed, and if any error occurs, the code within the `catch` block is executed.
*   `const yamlData = yaml.load(content)`: Parses the YAML content into a JavaScript object using the `yaml.load()` function from the `js-yaml` library.
*   `const jsonContent = JSON.stringify(yamlData, null, 2)`: Converts the parsed YAML data into a JSON string using `JSON.stringify()`.  The `null` argument specifies that no replacer function should be used, and the `2` argument specifies that the JSON should be formatted with 2 spaces for indentation.
*   `const metadata = { ... }`: Creates an object named `metadata` that contains information about the YAML data.
    *   `type: 'yaml'`: Sets the `type` property to `'yaml'`, indicating that the data was originally in YAML format.
    *   `isArray: Array.isArray(yamlData)`: Checks if the parsed YAML data is an array.
    *   `keys: Array.isArray(yamlData) ? [] : Object.keys(yamlData || {})`: If the `yamlData` is an array, the `keys` property is set to an empty array. Otherwise, it gets the keys of the `yamlData` object using `Object.keys()`. `yamlData || {}` handles potential null/undefined values of `yamlData` safely, ensuring that `Object.keys()` is called on an object.
    *   `itemCount: Array.isArray(yamlData) ? yamlData.length : undefined`: If `yamlData` is an array, the `itemCount` is set to the length of the array; otherwise, it's `undefined`.
    *   `depth: getYamlDepth(yamlData)`: Calls the `getYamlDepth()` function to calculate the nesting depth of the YAML data.
*   `return { content: jsonContent, metadata }`: Returns an object containing the JSON string representation of the YAML data and the metadata.
*   `throw new Error(\`Invalid YAML: ${error instanceof Error ? error.message : 'Unknown error'}\`)`: If an error occurs during parsing or conversion, this line throws an error with a descriptive message. The `error instanceof Error ? error.message : 'Unknown error'` part ensures a safe error message extraction, guarding against non-Error objects.

**`parseYAMLBuffer` function:**

This function is very similar to `parseYAML`, but it takes a `Buffer` as input instead of a file path.  The key difference is this line:

*   `const content = buffer.toString('utf-8')`: Converts the `Buffer` object to a string using UTF-8 encoding.

The rest of the function's logic, including parsing, conversion, metadata extraction, and error handling, is identical to the `parseYAML` function.

**`getYamlDepth` function:**

*   `function getYamlDepth(obj: any): number`: Defines a recursive function named `getYamlDepth` that takes any type of value (`any`) as input and returns a number representing the depth of the object.
*   `if (obj === null || typeof obj !== 'object') return 0`: Base case for the recursion. If the object is `null` or not an object (e.g., a number, string, or boolean), the depth is 0.
*   `if (Array.isArray(obj)) { ... }`: If the object is an array, this block calculates the depth.
    *   `return obj.length > 0 ? 1 + Math.max(...obj.map(getYamlDepth)) : 1`: If the array is not empty, it recursively calculates the depth of each element using `obj.map(getYamlDepth)` and then takes the maximum depth among all elements and adds 1 (for the current level of the array). If the array is empty, the depth is considered to be 1.
*   `const depths = Object.values(obj).map(getYamlDepth)`: Gets an array of the values of the object using `Object.values()` and recursively calculates the depth of each value using `map`.
*   `return depths.length > 0 ? 1 + Math.max(...depths) : 1`: If the object has values (i.e., `depths` is not empty), it calculates the maximum depth among the values and adds 1 (for the current level of the object).  If the object has no values, the depth is considered 1.

In summary, this file provides essential tools for working with YAML data in a TypeScript environment, offering parsing from both files and buffers, conversion to a consistent JSON format, and extraction of key metadata for understanding data structure. The code is well-structured, handles errors gracefully, and uses asynchronous operations for efficient file handling.
