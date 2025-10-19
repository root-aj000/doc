```typescript
/**
 * Utility functions for file parsing
 */

/**
 * Clean text content to ensure it's safe for UTF-8 storage in PostgreSQL
 * Removes null bytes and control characters that can cause encoding errors
 */
export function sanitizeTextForUTF8(text: string): string {
  if (!text || typeof text !== 'string') {
    return ''
  }

  return text
    .replace(/\0/g, '') // Remove null bytes (0x00)
    .replace(/[\x00-\x08\x0B\x0C\x0E-\x1F\x7F]/g, '') // Remove control characters except \t(0x09), \n(0x0A), \r(0x0D)
    .replace(/\uFFFD/g, '') // Remove Unicode replacement character
    .replace(/[\uD800-\uDFFF]/g, '') // Remove unpaired surrogate characters
}

/**
 * Sanitize an array of strings
 */
export function sanitizeTextArray(texts: string[]): string[] {
  return texts.map((text) => sanitizeTextForUTF8(text))
}

/**
 * Check if a string contains problematic characters for UTF-8 storage
 */
export function hasInvalidUTF8Characters(text: string): boolean {
  if (!text || typeof text !== 'string') {
    return false
  }

  // Check for null bytes and control characters
  return (
    /[\x00-\x08\x0B\x0C\x0E-\x1F\x7F]/.test(text) ||
    /\uFFFD/.test(text) ||
    /[\uD800-\uDFFF]/.test(text)
  )
}
```

## Explanation of the `file-parsing-utils.ts` file

This TypeScript file, aptly named based on its utilities, provides a set of functions designed to clean and validate text data, specifically focusing on ensuring compatibility and safety when storing it as UTF-8, particularly in a PostgreSQL database.  UTF-8 is a common character encoding, but certain characters can cause issues during storage or retrieval. This file helps address these problems.

Let's break down each function individually:

### 1. `sanitizeTextForUTF8(text: string): string`

**Purpose:**

This function is the core of the file, responsible for cleaning a given string and removing characters that could cause issues when stored as UTF-8.  It's a defensive measure to prevent encoding errors or data corruption.

**Code Breakdown:**

```typescript
export function sanitizeTextForUTF8(text: string): string {
  if (!text || typeof text !== 'string') {
    return ''
  }

  return text
    .replace(/\0/g, '') // Remove null bytes (0x00)
    .replace(/[\x00-\x08\x0B\x0C\x0E-\x1F\x7F]/g, '') // Remove control characters except \t(0x09), \n(0x0A), \r(0x0D)
    .replace(/\uFFFD/g, '') // Remove Unicode replacement character
    .replace(/[\uD800-\uDFFF]/g, '') // Remove unpaired surrogate characters
}
```

1. **`export function sanitizeTextForUTF8(text: string): string {`**:
   - This line declares a public (exported) function named `sanitizeTextForUTF8`.
   - It takes a single argument, `text`, which is expected to be a string.
   - The function is defined to return a string (the sanitized version of the input text).

2. **`if (!text || typeof text !== 'string') {`**:
   - This is a crucial input validation step.  It checks if the provided `text` is either:
     - `!text`:  Falsy (e.g., `null`, `undefined`, empty string).  An empty string will be returned in such cases.
     - `typeof text !== 'string'`: Not a string.  This prevents errors if a non-string value is accidentally passed to the function.

3. **`return ''`**:
   - If the `if` condition is true (invalid input), the function immediately returns an empty string (`''`).  This provides a safe default behavior.

4. **`return text...`**:
   - If the input is a valid string, the code proceeds to sanitize it using a chain of `replace` methods.  Each `replace` method uses a regular expression to find and remove specific characters.

5. **`.replace(/\0/g, '') // Remove null bytes (0x00)`**:
   - This line removes null bytes (represented as `\0`). Null bytes can cause problems in many systems, including databases and string handling functions. The `g` flag in the regular expression ensures that *all* occurrences of null bytes are removed, not just the first one.

6. **`.replace(/[\x00-\x08\x0B\x0C\x0E-\x1F\x7F]/g, '') // Remove control characters except \t(0x09), \n(0x0A), \r(0x0D)`**:
   - This is a more complex regular expression that removes a range of control characters.  Control characters are non-printing characters that can affect text formatting or have special meanings.
   - `\x00-\x08`: Represents characters with hexadecimal codes 0x00 to 0x08 (inclusive).
   - `\x0B`: Represents the character with hexadecimal code 0x0B (Vertical Tab).
   - `\x0C`: Represents the character with hexadecimal code 0x0C (Form Feed).
   - `\x0E-\x1F`: Represents characters with hexadecimal codes 0x0E to 0x1F (inclusive).
   - `\x7F`: Represents the Delete character (DEL).
   - **Important:**  The characters Tab (`\t` or 0x09), Newline (`\n` or 0x0A), and Carriage Return (`\r` or 0x0D) are *intentionally excluded* from the removal.  These are common whitespace characters that are usually safe and important for text formatting.

7. **`.replace(/\uFFFD/g, '') // Remove Unicode replacement character`**:
   - This line removes the Unicode replacement character (`\uFFFD`). This character () is often used to indicate that a character could not be decoded properly. Removing it can prevent the display of these strange symbols.

8. **`.replace(/[\uD800-\uDFFF]/g, '') // Remove unpaired surrogate characters`**:
   - This line removes unpaired surrogate characters.  Surrogate characters are used in UTF-16 to represent characters outside the Basic Multilingual Plane (BMP).  Unpaired surrogates can occur when a UTF-16 string is corrupted or incomplete.  These unpaired surrogates can cause encoding errors. The range `\uD800-\uDFFF` covers both high and low surrogate code points.

**In Summary:**  `sanitizeTextForUTF8` meticulously cleans a string by removing potentially problematic characters to ensure it's safe for UTF-8 storage, especially in a database context like PostgreSQL.  It handles null bytes, control characters (excluding common whitespace), Unicode replacement characters, and unpaired surrogate characters.

### 2. `sanitizeTextArray(texts: string[]): string[]`

**Purpose:**

This function applies the `sanitizeTextForUTF8` function to each element of an array of strings, effectively sanitizing the entire array.

**Code Breakdown:**

```typescript
export function sanitizeTextArray(texts: string[]): string[] {
  return texts.map((text) => sanitizeTextForUTF8(text))
}
```

1. **`export function sanitizeTextArray(texts: string[]): string[] {`**:
   - Declares a public function named `sanitizeTextArray`.
   - It takes a single argument, `texts`, which is an array of strings (`string[]`).
   - The function returns a new array of strings (`string[]`), containing the sanitized versions of the input strings.

2. **`return texts.map((text) => sanitizeTextForUTF8(text))`**:
   - This line uses the `map` method of the `Array` object.
   - `map` iterates over each element in the `texts` array.
   - For each `text` element, it calls the `sanitizeTextForUTF8(text)` function.
   - The `map` function then collects the results (the sanitized strings) into a new array.
   - Finally, the function returns this new array of sanitized strings.

**In Summary:**  `sanitizeTextArray` provides a convenient way to sanitize an entire array of strings using the core sanitization logic provided by `sanitizeTextForUTF8`.

### 3. `hasInvalidUTF8Characters(text: string): boolean`

**Purpose:**

This function checks if a string contains any characters that are considered problematic for UTF-8 storage, *without modifying the string*.  It provides a way to validate the string before attempting to store or process it.

**Code Breakdown:**

```typescript
export function hasInvalidUTF8Characters(text: string): boolean {
  if (!text || typeof text !== 'string') {
    return false
  }

  // Check for null bytes and control characters
  return (
    /[\x00-\x08\x0B\x0C\x0E-\x1F\x7F]/.test(text) ||
    /\uFFFD/.test(text) ||
    /[\uD800-\uDFFF]/.test(text)
  )
}
```

1. **`export function hasInvalidUTF8Characters(text: string): boolean {`**:
   - Declares a public function named `hasInvalidUTF8Characters`.
   - It takes a single argument, `text`, which is a string.
   - The function returns a boolean value (`boolean`), indicating whether the string contains invalid UTF-8 characters.

2. **`if (!text || typeof text !== 'string') {`**:
   - Similar to `sanitizeTextForUTF8`, this performs input validation.
   - If `text` is falsy or not a string, the function immediately returns `false` (meaning the string is considered *not* to have invalid characters, as it is an empty or invalid string).

3. **`return (...)`**:
   - This line returns the result of a boolean expression.  The expression uses the `||` (OR) operator to check for multiple conditions. If any of the conditions are true, the entire expression is true.

4. **`/[\x00-\x08\x0B\x0C\x0E-\x1F\x7F]/.test(text) ||`**:
   - This part checks for null bytes and control characters (excluding tab, newline, and carriage return), using the same regular expression as in `sanitizeTextForUTF8`.
   - The `.test(text)` method returns `true` if the regular expression finds a match in the `text`, and `false` otherwise.

5. **`/\uFFFD/.test(text) ||`**:
   - This part checks for the Unicode replacement character.  It returns `true` if the replacement character is found in the `text`.

6. **`/[\uD800-\uDFFF]/.test(text)`**:
   - This part checks for unpaired surrogate characters.  It returns `true` if any unpaired surrogate characters are found in the `text`.

**In Summary:**  `hasInvalidUTF8Characters` provides a way to *detect* the presence of problematic UTF-8 characters in a string without modifying the string. It uses regular expressions to check for null bytes, control characters, the Unicode replacement character, and unpaired surrogate characters. This allows you to validate data before attempting to store it, potentially preventing errors or data corruption.

### Overall Summary of the File

This file provides a robust set of utilities for handling text data and ensuring its compatibility with UTF-8 encoding, especially in the context of PostgreSQL databases. The functions allow you to both sanitize (clean) strings and validate strings for potentially problematic characters, providing a comprehensive approach to data integrity.
