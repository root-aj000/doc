This TypeScript file, `ResponseFormatUtils.ts`, is a collection of utility functions designed to help a system process, extract, and format structured data, often in the context of handling responses from various components or blocks (e.g., in a workflow, AI agent, or data pipeline).

Its primary responsibilities include:
*   **Schema Interpretation**: Understanding different ways data structures (schemas) can be defined and extracting their fields.
*   **Safe Parsing**: Robustly converting stringified JSON data into JavaScript objects, with error handling.
*   **Data Extraction**: Navigating complex nested objects to pull out specific pieces of data based on user-defined "paths."
*   **Output Formatting**: Presenting extracted data in a human-readable string format.
*   **Identifier Management**: Parsing special identifiers (`outputId`) to determine which block they belong to and which specific data path they refer to.

In essence, this file provides the foundational tools for working with dynamic, structured data, making it easier to select, retrieve, and display relevant information within a larger application.

---

### Detailed Code Explanation

Let's break down the code section by section.

#### Logger Initialization

```typescript
import { createLogger } from '@/lib/logs/console/logger'

const logger = createLogger('ResponseFormatUtils')
```

*   **`import { createLogger } from '@/lib/logs/console/logger'`**: This line imports a function called `createLogger` from a specific path within your project. This function is used to set up a logging utility.
*   **`const logger = createLogger('ResponseFormatUtils')`**: This initializes a logger instance specifically for this file, naming it `ResponseFormatUtils`. This allows you to output messages (like warnings or errors) to the console, often with a prefix that indicates where the message originated, making debugging easier.

#### Field Interface

```typescript
export interface Field {
  name: string
  type: string
  description?: string
}
```

*   **`export interface Field`**: This defines a TypeScript interface named `Field`. Interfaces are blueprints that describe the shape of an object. The `export` keyword makes this interface available for use in other files.
*   **`name: string`**: Every `Field` object must have a `name` property, which is a string. This typically represents the name of a data field (e.g., "user_id", "product_name").
*   **`type: string`**: Every `Field` object must have a `type` property, also a string. This describes the data type of the field (e.g., "string", "number", "array", "object").
*   **`description?: string`**: This is an optional property (`?` makes it optional). If present, it's a string providing a brief explanation of the field.

This interface ensures consistency when dealing with data fields extracted from schemas.

#### `extractFieldsFromSchema` Function

```typescript
/**
 * Helper function to extract fields from JSON Schema
 * Handles both legacy format with fields array and new JSON Schema format
 */
export function extractFieldsFromSchema(schema: any): Field[] {
  if (!schema || typeof schema !== 'object') {
    return []
  }

  // Handle legacy format with fields array
  if (Array.isArray(schema.fields)) {
    return schema.fields
  }

  // Handle new JSON Schema format
  const schemaObj = schema.schema || schema
  if (!schemaObj || !schemaObj.properties || typeof schemaObj.properties !== 'object') {
    return []
  }

  // Extract fields from schema properties
  return Object.entries(schemaObj.properties).map(([name, prop]: [string, any]) => {
    // Handle array format like ['string', 'array']
    if (Array.isArray(prop)) {
      return {
        name,
        type: prop.includes('array') ? 'array' : prop[0] || 'string',
        description: undefined,
      }
    }

    // Handle object format like { type: 'string', description: '...' }
    return {
      name,
      type: prop.type || 'string',
      description: prop.description,
    }
  })
}
```

This function is crucial for making sense of different ways data schemas might be structured. It can extract a list of `Field` objects from two distinct formats: a "legacy" format and a "new JSON Schema" format.

*   **`export function extractFieldsFromSchema(schema: any): Field[]`**: Declares an exported function that takes a `schema` (which can be of any type, hence `any`) and is expected to return an array of `Field` objects.
*   **`if (!schema || typeof schema !== 'object') { return [] }`**: This is a basic validation check. If the provided `schema` is `null`, `undefined`, or not an object, it cannot contain field definitions, so an empty array is returned immediately.
*   **`if (Array.isArray(schema.fields)) { return schema.fields }`**: This handles the "legacy" format. If the `schema` object directly has a property named `fields` and that property is an array, it assumes this array already contains the `Field` objects and returns them directly.
*   **`const schemaObj = schema.schema || schema`**: This line prepares for the "new JSON Schema" format.
    *   It tries to access `schema.schema`. This means the actual schema definition might be nested one level deeper (e.g., `schema = { someOtherData, schema: { properties: ... } }`).
    *   If `schema.schema` is `null` or `undefined`, it falls back to using the `schema` object itself. This makes the function flexible to schemas that are either directly the JSON Schema or wrapped within another object.
*   **`if (!schemaObj || !schemaObj.properties || typeof schemaObj.properties !== 'object') { return [] }`**: This checks if `schemaObj` (the potentially unwrapped schema) is valid, if it has a `properties` key, and if `properties` is an object. In standard JSON Schema, `properties` is an object where keys are field names and values are their definitions. If these conditions aren't met, an empty array is returned.
*   **`return Object.entries(schemaObj.properties).map(...)`**: This is where the core extraction for JSON Schema happens.
    *   `Object.entries(schemaObj.properties)`: This takes the `properties` object and converts it into an array of `[key, value]` pairs. For example, if `properties` is `{ "name": { type: "string" }, "age": { type: "number" } }`, `Object.entries` will produce `[["name", { type: "string" }], ["age", { type: "number" }]]`.
    *   `.map(([name, prop]: [string, any]) => { ... })`: It then iterates over each of these `[key, value]` pairs.
        *   `name`: The field name (e.g., "name", "age").
        *   `prop`: The property definition object (e.g., `{ type: "string" }`, `['string', 'array']`).
*   **`if (Array.isArray(prop)) { ... }`**: This handles a less common, but possible, schema property definition where the type might be an array of strings (e.g., `["string", "null"]` or `["string", "array"]`).
    *   `prop.includes('array') ? 'array' : prop[0] || 'string'`: If the array `prop` contains `'array'`, the type is set to `'array'`. Otherwise, it takes the first element of the array as the type (e.g., `'string'` from `['string', 'null']`), defaulting to `'string'` if `prop[0]` is also falsy.
    *   `description: undefined`: No description is available in this array format.
*   **`return { name, type: prop.type || 'string', description: prop.description }`**: This handles the standard object-based property definition (e.g., `{ type: "string", description: "User's full name" }`).
    *   It extracts `name`, `type` (defaulting to `'string'` if `prop.type` is missing), and `description` directly from the `prop` object.

#### `parseResponseFormatSafely` Function

```typescript
/**
 * Helper function to safely parse response format
 * Handles both string and object formats
 */
export function parseResponseFormatSafely(responseFormatValue: any, blockId: string): any {
  if (!responseFormatValue) {
    return null
  }

  try {
    if (typeof responseFormatValue === 'string') {
      return JSON.parse(responseFormatValue)
    }
    return responseFormatValue
  } catch (error) {
    logger.warn(`Failed to parse response format for block ${blockId}:`, error)
    return null
  }
}
```

This function ensures that a `responseFormatValue` is always returned as a JavaScript object, handling cases where it might be a JSON string.

*   **`export function parseResponseFormatSafely(responseFormatValue: any, blockId: string): any`**: Defines an exported function that takes a `responseFormatValue` (any type) and a `blockId` (string, used for logging) and returns any type.
*   **`if (!responseFormatValue) { return null }`**: If the input is null or undefined, there's nothing to parse, so it returns `null`.
*   **`try { ... } catch (error) { ... }`**: This `try-catch` block is crucial for "safely" parsing. It attempts the parsing logic and, if an error occurs (like malformed JSON), it gracefully handles it.
*   **`if (typeof responseFormatValue === 'string') { return JSON.parse(responseFormatValue) }`**: Inside the `try` block, if `responseFormatValue` is a string, it attempts to parse it as JSON using `JSON.parse()`.
*   **`return responseFormatValue`**: If `responseFormatValue` is *not* a string (meaning it's likely already an object or another primitive type), it's returned as is.
*   **`logger.warn(...)`**: If `JSON.parse()` fails (e.g., the string is not valid JSON), the `catch` block executes. It logs a warning message using the `logger` instance, including the `blockId` for context, and then returns `null` to indicate the parsing failure.

#### `extractFieldValues` Function

```typescript
/**
 * Extract field values from a parsed JSON object based on selected output paths
 * Used for both workspace and chat client field extraction
 */
export function extractFieldValues(
  parsedContent: any,
  selectedOutputs: string[],
  blockId: string
): Record<string, any> {
  const extractedValues: Record<string, any> = {}

  for (const outputId of selectedOutputs) {
    const blockIdForOutput = extractBlockIdFromOutputId(outputId)

    if (blockIdForOutput !== blockId) {
      continue
    }

    const path = extractPathFromOutputId(outputId, blockIdForOutput)

    if (path) {
      const current = traverseObjectPathInternal(parsedContent, path)
      if (current !== undefined) {
        extractedValues[path] = current
      }
    }
  }

  return extractedValues
}
```

This function is designed to pick out specific data points from a larger parsed object based on a list of `selectedOutputs`.

*   **`export function extractFieldValues(...)`**: Defines an exported function that takes:
    *   `parsedContent`: The object from which to extract values (e.g., a parsed JSON response).
    *   `selectedOutputs`: An array of strings, where each string (`outputId`) represents a selected field, often in the format `blockId_path` or `blockId.path`.
    *   `blockId`: The ID of the specific block whose outputs are being considered.
    *   It returns a `Record<string, any>`, which is an object where keys are field paths and values are the extracted data.
*   **`const extractedValues: Record<string, any> = {}`**: Initializes an empty object to store the results.
*   **`for (const outputId of selectedOutputs) { ... }`**: It iterates through each `outputId` in the `selectedOutputs` array.
*   **`const blockIdForOutput = extractBlockIdFromOutputId(outputId)`**: For each `outputId`, it calls `extractBlockIdFromOutputId` (explained below) to get the block ID part of the `outputId`.
*   **`if (blockIdForOutput !== blockId) { continue }`**: This filters `outputId`s. It only processes `outputId`s that belong to the `blockId` currently being processed. If an `outputId` belongs to a different block, it skips to the next iteration.
*   **`const path = extractPathFromOutputId(outputId, blockIdForOutput)`**: If the `outputId` matches the current `blockId`, it then calls `extractPathFromOutputId` (explained below) to get the actual data path (e.g., "data.items[0].value").
*   **`if (path) { ... }`**: If a valid `path` was extracted (meaning it's not an `outputId` that's just a `blockId` without a specific path), it proceeds.
*   **`const current = traverseObjectPathInternal(parsedContent, path)`**: It uses the internal helper `traverseObjectPathInternal` (explained below) to safely navigate the `parsedContent` object using the `path` and retrieve the value.
*   **`if (current !== undefined) { extractedValues[path] = current }`**: If a value was found at the given `path` (i.e., `current` is not `undefined`), it stores that value in `extractedValues` with the `path` as the key.
*   **`return extractedValues`**: Finally, it returns the object containing all successfully extracted field values.

#### `formatFieldValues` Function

```typescript
/**
 * Format extracted field values for display
 * Returns formatted string representation of field values
 */
export function formatFieldValues(extractedValues: Record<string, any>): string {
  const formattedValues: string[] = []

  for (const [fieldName, value] of Object.entries(extractedValues)) {
    const formattedValue = typeof value === 'string' ? value : JSON.stringify(value)
    formattedValues.push(formattedValue)
  }

  return formattedValues.join('\n')
}
```

This function takes the object of extracted values and formats them into a single, human-readable string.

*   **`export function formatFieldValues(extractedValues: Record<string, any>): string`**: Defines an exported function that takes the `extractedValues` object and returns a single string.
*   **`const formattedValues: string[] = []`**: Initializes an empty array to hold the formatted string representation of each value.
*   **`for (const [fieldName, value] of Object.entries(extractedValues)) { ... }`**: It iterates over each `[key, value]` pair in the `extractedValues` object. `fieldName` will be the path (e.g., "data.item"), and `value` will be the extracted data.
*   **`const formattedValue = typeof value === 'string' ? value : JSON.stringify(value)`**: This line determines how to format each `value`:
    *   If `value` is already a string, it's used directly.
    *   If `value` is not a string (e.g., a number, boolean, array, or object), `JSON.stringify(value)` is used to convert it into a string representation. This ensures that complex data types are also readable.
*   **`formattedValues.push(formattedValue)`**: The formatted string for the current value is added to the `formattedValues` array.
*   **`return formattedValues.join('\n')`**: After processing all values, the elements in the `formattedValues` array are joined together into a single string, with each formatted value separated by a newline character (`\n`), making it easy to read when displayed.

#### `extractBlockIdFromOutputId` Function

```typescript
/**
 * Extract block ID from output ID
 * Handles both formats: "blockId" and "blockId_path" or "blockId.path"
 */
export function extractBlockIdFromOutputId(outputId: string): string {
  return outputId.includes('_') ? outputId.split('_')[0] : outputId.split('.')[0]
}
```

This function parses an `outputId` string to get the `blockId` part.

*   **`export function extractBlockIdFromOutputId(outputId: string): string`**: Defines an exported function that takes an `outputId` string and returns a string (the block ID).
*   **`outputId.includes('_') ? outputId.split('_')[0] : outputId.split('.')[0]`**: This is a conditional (ternary) operator:
    *   It checks if the `outputId` string contains an underscore (`_`).
    *   If it does (`outputId.includes('_')` is true), it splits the string by `_` and returns the first part (`[0]`). This handles `blockId_path` format.
    *   If it doesn't contain `_`, it assumes the separator is a dot (`.`), splits by `.` and returns the first part. This handles `blockId.path` or just `blockId` (where `blockId.path` would mean `outputId.split('.')[0]` is just `blockId`).

#### `extractPathFromOutputId` Function

```typescript
/**
 * Extract path from output ID after the block ID
 */
export function extractPathFromOutputId(outputId: string, blockId: string): string {
  return outputId.substring(blockId.length + 1)
}
```

This function gets the "path" part of an `outputId` after the `blockId`.

*   **`export function extractPathFromOutputId(outputId: string, blockId: string): string`**: Defines an exported function that takes the full `outputId` and the already extracted `blockId`, returning the path string.
*   **`return outputId.substring(blockId.length + 1)`**:
    *   `blockId.length`: Gets the length of the `blockId` string.
    *   `+ 1`: Adds 1 to account for the separator character (`_` or `.`) immediately following the `blockId`.
    *   `outputId.substring(...)`: Extracts a substring from `outputId` starting from the character *after* the `blockId` and its separator. This gives you the path part (e.g., if `outputId` is "block1_data.name" and `blockId` is "block1", it returns "data.name").

#### `parseOutputContentSafely` Function

```typescript
/**
 * Parse JSON content from output safely
 * Handles both string and object formats with proper error handling
 */
export function parseOutputContentSafely(output: any): any {
  if (!output?.content) {
    return output
  }

  if (typeof output.content === 'string') {
    try {
      return JSON.parse(output.content)
    } catch (e) {
      // Fallback to original structure if parsing fails
      return output
    }
  }

  return output
}
```

This function safely parses the `content` property of an `output` object, if it's a stringified JSON.

*   **`export function parseOutputContentSafely(output: any): any`**: Defines an exported function that takes an `output` object (any type) and returns its content (parsed or original).
*   **`if (!output?.content) { return output }`**: This checks if the `output` object exists and if it has a `content` property. The `?.` is optional chaining. If `output` is null/undefined or `output.content` is null/undefined, it means there's no content to parse, so the original `output` object is returned as is.
*   **`if (typeof output.content === 'string') { ... }`**: If `output.content` is a string, it proceeds to attempt JSON parsing.
    *   **`try { return JSON.parse(output.content) } catch (e) { ... }`**: A `try-catch` block is used for safe parsing. It attempts `JSON.parse()`.
    *   **`return output`**: If parsing fails (the `catch` block is executed), it means `output.content` was a malformed JSON string. In this case, instead of throwing an error, it gracefully falls back and returns the *original* `output` object (which still contains the unparsed string content). This prevents crashes and allows the system to continue with the raw content if parsing isn't strictly necessary or if the content is not JSON.
*   **`return output`**: If `output.content` was not a string (meaning it was already an object or another primitive type), it means there's no parsing needed, so the original `output` object is returned directly.

#### `hasResponseFormatSelection` Function

```typescript
/**
 * Check if a set of output IDs contains response format selections for a specific block
 */
export function hasResponseFormatSelection(selectedOutputs: string[], blockId: string): boolean {
  return selectedOutputs.some((outputId) => {
    const blockIdForOutput = extractBlockIdFromOutputId(outputId)
    return blockIdForOutput === blockId && outputId.includes('_')
  })
}
```

This function determines if any specific field selections have been made for a given `blockId`.

*   **`export function hasResponseFormatSelection(...)`**: Defines an exported function that takes `selectedOutputs` (array of strings) and `blockId` (string), and returns a boolean.
*   **`return selectedOutputs.some((outputId) => { ... })`**: The `some()` method iterates over the `selectedOutputs` array. It returns `true` as soon as the provided callback function returns `true` for *any* element, otherwise it returns `false`.
*   **`const blockIdForOutput = extractBlockIdFromOutputId(outputId)`**: Extracts the `blockId` from the current `outputId`.
*   **`return blockIdForOutput === blockId && outputId.includes('_')`**: The condition for `some()` to return true:
    *   `blockIdForOutput === blockId`: Ensures the `outputId` belongs to the `blockId` we are interested in.
    *   `outputId.includes('_')`: Checks if the `outputId` includes an underscore. This is a heuristic to determine if it's a *specific field selection* (`blockId_path`) rather than just a `blockId` alone (which might imply selecting the whole output). If it contains an underscore, it implies a path selection has been made.

#### `getSelectedFieldNames` Function

```typescript
/**
 * Get selected field names for a specific block from output IDs
 */
export function getSelectedFieldNames(selectedOutputs: string[], blockId: string): string[] {
  return selectedOutputs
    .filter((outputId) => {
      const blockIdForOutput = extractBlockIdFromOutputId(outputId)
      return blockIdForOutput === blockId && outputId.includes('_')
    })
    .map((outputId) => extractPathFromOutputId(outputId, blockId))
}
```

This function extracts the actual *paths* (field names) that have been selected for a particular block.

*   **`export function getSelectedFieldNames(...)`**: Defines an exported function that takes `selectedOutputs` and `blockId`, returning an array of strings (the field paths).
*   **`.filter((outputId) => { ... })`**: First, it filters the `selectedOutputs` array. The filter condition is the same as in `hasResponseFormatSelection`: it keeps only those `outputId`s that belong to the current `blockId` AND represent a specific field selection (contain `_`).
*   **`.map((outputId) => extractPathFromOutputId(outputId, blockId))`**: After filtering, for each remaining `outputId` (which now definitively represents a selected field for the current `blockId`), it uses `extractPathFromOutputId` to get just the path part. The result is an array of these paths.

#### `traverseObjectPathInternal` Function

```typescript
/**
 * Internal helper to traverse an object path without parsing
 * @param obj The object to traverse
 * @param path The dot-separated path (e.g., "result.data.value")
 * @returns The value at the path, or undefined if path doesn't exist
 */
function traverseObjectPathInternal(obj: any, path: string): any {
  if (!path) return obj

  let current = obj
  const parts = path.split('.')

  for (const part of parts) {
    if (current?.[part] !== undefined) {
      current = current[part]
    } else {
      return undefined
    }
  }

  return current
}
```

This is a private helper function (not exported) for navigating deeply nested objects using a dot-separated path string.

*   **`function traverseObjectPathInternal(obj: any, path: string): any`**: Defines a non-exported function that takes an object `obj` and a `path` string, returning the value found at the path or `undefined`.
*   **`if (!path) return obj`**: If the `path` is empty or null, it means no specific nested element is requested, so the original `obj` itself is returned.
*   **`let current = obj`**: Initializes a `current` variable to the starting `obj`. This `current` variable will be updated as we traverse deeper.
*   **`const parts = path.split('.')`**: The `path` string (e.g., "result.data.value") is split into an array of individual property names (e.g., `["result", "data", "value"]`).
*   **`for (const part of parts) { ... }`**: It iterates through each `part` (property name) in the `parts` array.
*   **`if (current?.[part] !== undefined) { current = current[part] }`**:
    *   `current?.[part]`: This uses optional chaining (`?.`) to safely access the property `part` on the `current` object. If `current` is `null` or `undefined` at any point, `current?.[part]` will evaluate to `undefined` without throwing an error.
    *   `!== undefined`: It checks if the property exists and is not `undefined`. This differentiates between a property that genuinely doesn't exist versus one that exists but holds an `undefined` value.
    *   `current = current[part]`: If the property exists, `current` is updated to be the value of that property, effectively moving deeper into the object.
*   **`else { return undefined }`**: If `current?.[part]` is `undefined` (meaning the property `part` does not exist on `current` or `current` itself was `null`/`undefined`), it immediately returns `undefined`, as the path cannot be fully traversed.
*   **`return current`**: If the loop completes successfully, `current` holds the value at the deepest part of the path, which is then returned.

#### `traverseObjectPath` Function

```typescript
/**
 * Traverses an object path safely, returning undefined if any part doesn't exist
 * Automatically handles parsing of output content if needed
 * @param obj The object to traverse (may contain unparsed content)
 * @param path The dot-separated path (e.g., "result.data.value")
 * @returns The value at the path, or undefined if path doesn't exist
 */
export function traverseObjectPath(obj: any, path: string): any {
  const parsed = parseOutputContentSafely(obj)
  return traverseObjectPathInternal(parsed, path)
}
```

This is the public-facing version of the object traversal function. It adds an important pre-processing step: safely parsing the object's content if necessary.

*   **`export function traverseObjectPath(obj: any, path: string): any`**: Defines an exported function that takes an object `obj` (potentially with unparsed string content) and a `path`, returning the value or `undefined`.
*   **`const parsed = parseOutputContentSafely(obj)`**: This is the key difference. Before attempting to traverse, it calls `parseOutputContentSafely(obj)`. This ensures that if `obj` is an `output` object with a stringified JSON `content` property, that content is first parsed into an actual JavaScript object. If parsing fails, `parseOutputContentSafely` returns the original `obj`.
*   **`return traverseObjectPathInternal(parsed, path)`**: After the content is safely parsed (or the original object is retained if no parsing was needed/possible), the `traverseObjectPathInternal` function is called to perform the actual path traversal on the (now potentially parsed) `obj`. This combines safe parsing with safe traversal.

---

This `ResponseFormatUtils.ts` file demonstrates robust and flexible utilities for handling structured data, particularly useful in scenarios where data sources or user selections can vary in format and depth. It emphasizes safety through error handling and consistent data structures via interfaces, making the application more resilient and maintainable.