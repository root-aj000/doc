```typescript
import type { FileParseResult } from './types'

/**
 * Parse JSON files
 */
export async function parseJSON(filePath: string): Promise<FileParseResult> {
  // Dynamically import the 'fs/promises' module. This is done asynchronously, which is good for performance,
  // as it only loads the file system module when needed. The fs/promises module provides a promise-based API for interacting with the file system.
  const fs = await import('fs/promises')
  // Read the file at the specified filePath.  'utf-8' specifies the character encoding to use when reading the file.
  // This returns the entire file content as a string.
  const content = await fs.readFile(filePath, 'utf-8')

  try {
    // Attempt to parse the file content as JSON.  This is where the validation of the JSON structure occurs.
    const jsonData = JSON.parse(content)

    // Convert the JSON data back into a string, but this time with indentation for better readability.
    //  `null` as the second argument is a replacer function (or null to use all properties of the object)
    //  `2` as the third argument specifies two spaces for indentation.
    const formattedContent = JSON.stringify(jsonData, null, 2)

    // Extract metadata about the JSON structure to provide additional information about the parsed JSON.
    const metadata = {
      type: 'json', // Indicate that the parsed content is of type JSON.
      isArray: Array.isArray(jsonData), // Determine if the JSON data is an array.
      keys: Array.isArray(jsonData) ? [] : Object.keys(jsonData), // If it is not array, get keys of JSON object
      itemCount: Array.isArray(jsonData) ? jsonData.length : undefined, // If it is an array, get number of items in array.
      depth: getJsonDepth(jsonData), // Calculate the depth of the JSON structure.
    }

    // Return an object containing the formatted JSON content and the extracted metadata.  This is the result of the successful parsing operation.
    return {
      content: formattedContent,
      metadata,
    }
  } catch (error) {
    // If an error occurs during JSON parsing (e.g., invalid JSON syntax), catch the error.
    // Construct a new error message that indicates the JSON is invalid and includes the original error message (if available).
    // Throw the new error to signal that the parsing operation failed.
    throw new Error(`Invalid JSON: ${error instanceof Error ? error.message : 'Unknown error'}`)
  }
}

/**
 * Parse JSON from buffer
 */
export async function parseJSONBuffer(buffer: Buffer): Promise<FileParseResult> {
  // Convert the buffer to a UTF-8 string.
  const content = buffer.toString('utf-8')

  try {
    // Parse the content as JSON.
    const jsonData = JSON.parse(content)
    // Format the JSON with indentation.
    const formattedContent = JSON.stringify(jsonData, null, 2)

    // Extract metadata about the JSON structure.
    const metadata = {
      type: 'json',
      isArray: Array.isArray(jsonData),
      keys: Array.isArray(jsonData) ? [] : Object.keys(jsonData),
      itemCount: Array.isArray(jsonData) ? jsonData.length : undefined,
      depth: getJsonDepth(jsonData),
    }

    // Return the formatted content and metadata.
    return {
      content: formattedContent,
      metadata,
    }
  } catch (error) {
    // Handle JSON parsing errors.
    throw new Error(`Invalid JSON: ${error instanceof Error ? error.message : 'Unknown error'}`)
  }
}

/**
 * Calculate the depth of a JSON object
 */
function getJsonDepth(obj: any): number {
  // Base case: If the object is null or not an object, the depth is 0.
  if (obj === null || typeof obj !== 'object') return 0

  // If the object is an array
  if (Array.isArray(obj)) {
    // If the array is empty, the depth is 1. Otherwise, recursively calculate the depth of each element and return 1 plus the maximum depth.
    return obj.length > 0 ? 1 + Math.max(...obj.map(getJsonDepth)) : 1
  }

  // If the object is a regular object, recursively calculate the depth of each value and return 1 plus the maximum depth.
  const depths = Object.values(obj).map(getJsonDepth)
  return depths.length > 0 ? 1 + Math.max(...depths) : 1
}
```

### Purpose of this file

This file provides functions for parsing JSON data from different sources: a file path (`parseJSON`) and a buffer (`parseJSONBuffer`).  It also includes a helper function (`getJsonDepth`) to determine the nested depth of a JSON structure. The main purpose is to read JSON data, validate it, format it for readability, and extract useful metadata about the JSON structure.

### Simplification of Complex Logic

1.  **Asynchronous File Reading:** The `parseJSON` function uses `fs/promises` for asynchronous file reading, improving responsiveness.
2.  **Error Handling:** Both `parseJSON` and `parseJSONBuffer` use a `try...catch` block to gracefully handle potential JSON parsing errors, providing a user-friendly error message.
3.  **Metadata Extraction:** The functions extract relevant metadata, such as whether the JSON data is an array, its keys (if it's an object), the number of items (if it's an array), and the depth of the JSON structure.
4.  **JSON Formatting:** The `JSON.stringify` method is used with indentation to make the JSON output more readable.
5.  **Depth Calculation:** `getJsonDepth` handles null, primitive types, arrays and objects. Recursion is used to get the maximum depth of each nested object/array.

### Explanation of each line of code

**1. `import type { FileParseResult } from './types'`**

*   **Purpose:** Imports the `FileParseResult` type definition from a local file named `types.ts` (or `types.js`, etc.).  This type likely defines the structure of the object returned by the `parseJSON` and `parseJSONBuffer` functions, enforcing type safety.
*   **`type` keyword:** Ensures that only the *type* `FileParseResult` is imported, not any actual values.  This can help with performance in some build systems by avoiding unnecessary code being bundled.

**2. `/**\n * Parse JSON files\n */\nexport async function parseJSON(filePath: string): Promise<FileParseResult> {`**

*   **Purpose:** Defines an asynchronous function named `parseJSON` that takes a file path (`filePath`) as input and returns a promise that resolves to a `FileParseResult` object.
*   **`/** ... */`:** A JSDoc-style comment that describes the function's purpose.
*   **`export`:** Makes the function available for use in other modules.
*   **`async`:** Indicates that the function is asynchronous and uses the `await` keyword.
*   **`filePath: string`:** Specifies that the `filePath` argument must be a string.
*   **`Promise<FileParseResult>`:** Specifies that the function returns a promise that will resolve to an object of type `FileParseResult`.

**3. `const fs = await import('fs/promises')`**

*   **Purpose:** Asynchronously imports the `fs/promises` module, which provides a promise-based API for interacting with the file system.
*   **`await`:** Pauses the execution of the function until the `fs/promises` module is successfully imported.  This ensures that the `fs` variable is properly initialized before being used.
*   **`fs`:** A constant variable that will hold the imported file system module.
*   **Dynamic import:** Loads the `fs` module only when this function is called, which can improve the application startup time if this function isn't immediately needed.

**4. `const content = await fs.readFile(filePath, 'utf-8')`**

*   **Purpose:** Reads the contents of the file specified by `filePath` asynchronously.
*   **`await`:** Pauses execution until the file is completely read.
*   **`fs.readFile(filePath, 'utf-8')`:**  Calls the `readFile` function from the `fs/promises` module.
    *   `filePath`: The path to the file to read.
    *   `'utf-8'`:  The character encoding to use when reading the file.  UTF-8 is a common encoding for text files.
*   **`content`:** A constant variable that stores the file's content as a string.

**5. `try { ... } catch (error) { ... }`**

*   **Purpose:** A `try...catch` block is used for error handling.  The code within the `try` block is executed. If any error occurs during the execution of the `try` block, the code within the `catch` block is executed. This prevents the program from crashing and allows you to handle errors gracefully.

**6. `const jsonData = JSON.parse(content)`**

*   **Purpose:** Parses the `content` (which is a string read from the file) as JSON data.
*   **`JSON.parse(content)`:** A built-in JavaScript function that converts a JSON string into a JavaScript object.
*   **`jsonData`:** A constant variable that stores the resulting JavaScript object.  If `content` is not valid JSON, this line will throw an error, which will be caught by the `catch` block.

**7. `const formattedContent = JSON.stringify(jsonData, null, 2)`**

*   **Purpose:** Converts the `jsonData` (JavaScript object) back into a JSON string, but this time formats it with indentation for better readability.
*   **`JSON.stringify(jsonData, null, 2)`:** A built-in JavaScript function that converts a JavaScript object into a JSON string.
    *   `jsonData`: The JavaScript object to convert.
    *   `null`: A replacer function (or null to use all properties of the object).
    *   `2`:  Specifies the number of spaces to use for indentation.
*   **`formattedContent`:** A constant variable that stores the formatted JSON string.

**8. `const metadata = { ... }`**

*   **Purpose:** Creates an object called `metadata` that contains information *about* the parsed JSON data.  This is helpful for understanding the structure of the JSON.
*   **`type: 'json'`:**  Indicates that the parsed content is of type JSON.
*   **`isArray: Array.isArray(jsonData)`:** Checks if `jsonData` is an array and stores the boolean result.
*   **`keys: Array.isArray(jsonData) ? [] : Object.keys(jsonData)`:** If `jsonData` is an array, sets `keys` to an empty array. Otherwise, gets an array of the keys (property names) of the `jsonData` object.
*   **`itemCount: Array.isArray(jsonData) ? jsonData.length : undefined`:** If `jsonData` is an array, sets `itemCount` to the number of elements in the array. Otherwise, sets `itemCount` to `undefined`.
*   **`depth: getJsonDepth(jsonData)`:** Calls the `getJsonDepth` function (defined later) to calculate the nested depth of the JSON structure and stores the result.

**9. `return { content: formattedContent, metadata }`**

*   **Purpose:** Returns an object containing the formatted JSON content and the extracted metadata.  This is the successful result of the parsing operation.

**10. `catch (error) { ... }`**

*   **Purpose:** This block is executed if any error occurs within the `try` block.  It handles potential errors during JSON parsing.
*   **`throw new Error(\`Invalid JSON: ${error instanceof Error ? error.message : 'Unknown error'}\`)`:** Creates a new `Error` object with a descriptive message indicating that the JSON is invalid and includes the original error message, if available. The `throw` keyword re-throws the error, which will stop the execution of the function and propagate the error to the caller.

**11. `/**\n * Parse JSON from buffer\n */\nexport async function parseJSONBuffer(buffer: Buffer): Promise<FileParseResult> {`**

*   **Purpose:** Defines an asynchronous function named `parseJSONBuffer` that takes a `Buffer` object as input and returns a promise that resolves to a `FileParseResult` object. This function is similar to `parseJSON`, but it parses JSON data directly from a buffer instead of a file path.  A buffer is a region of memory that holds binary data.

**12. `const content = buffer.toString('utf-8')`**

*   **Purpose:** Converts the buffer to a UTF-8 string.
*   **`buffer.toString('utf-8')`:**  Converts the contents of the buffer into a string, using UTF-8 encoding.

**13-36. Logic inside `parseJSONBuffer`**

*   Almost identical to `parseJSON`, and the explanation is covered in sections 6-10.

**37. `/**\n * Calculate the depth of a JSON object\n */\nfunction getJsonDepth(obj: any): number {`**

*   **Purpose:** Defines a recursive function named `getJsonDepth` that calculates the nested depth of a JSON object (or array).
*   **`obj: any`:** Specifies that the `obj` argument can be of any type.  This is necessary because JSON structures can contain various data types.
*   **`number`:** Specifies that the function returns a number, representing the depth of the JSON structure.

**38. `if (obj === null || typeof obj !== 'object') return 0`**

*   **Purpose:** Base case for the recursion. If the object is `null` or not an object (i.e., it's a primitive like a string, number, or boolean), the depth is 0. This stops the recursion when it reaches the "leaves" of the JSON structure.

**39. `if (Array.isArray(obj)) { ... }`**

*   **Purpose:** Handles the case where the object is an array.

**40. `return obj.length > 0 ? 1 + Math.max(...obj.map(getJsonDepth)) : 1`**

*   **Purpose:** If the array is not empty, it recursively calculates the depth of each element in the array using `obj.map(getJsonDepth)`, finds the maximum depth among those elements using `Math.max`, and adds 1 to it (to account for the array itself). If the array is empty, the depth is simply 1 (an empty array has a depth of 1).
*   **`obj.length > 0 ? ... : 1`:** A ternary operator that checks if the array has any elements.
*   **`obj.map(getJsonDepth)`:** Applies the `getJsonDepth` function to each element in the array, creating a new array of depths.
*   **`Math.max(...obj.map(getJsonDepth))`:** Finds the maximum value in the array of depths.  The spread operator (`...`) expands the array into individual arguments for `Math.max`.
*   **`1 + Math.max(...)`:** Adds 1 to the maximum depth to account for the current array level.

**41. `const depths = Object.values(obj).map(getJsonDepth)`**

*   **Purpose:** Gets the values of the object, maps them to their depths using recursion.
*   **`Object.values(obj)`:** Returns an array containing all the values of the object.
*   **`.map(getJsonDepth)`:**  Applies `getJsonDepth` to each value in the array of values, creating a new array containing the depths of each value.

**42. `return depths.length > 0 ? 1 + Math.max(...depths) : 1`**

*   **Purpose:** If the object has values then returns 1 (for the current level) + the maximum depth of its values. If it has no values (empty object), it returns 1.  Similar to array, but applies to objects.
*   **`depths.length > 0 ? ... : 1`:** A ternary operator that checks if the object has any values.
*   **`Math.max(...depths)`:** Finds the maximum depth among the values.
*   **`1 + Math.max(...)`:** Adds 1 to the maximum depth.
