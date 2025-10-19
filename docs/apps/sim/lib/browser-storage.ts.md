This TypeScript file provides a robust and safe way to interact with `localStorage` in web applications, with special considerations for Server-Side Rendering (SSR) environments and enhanced error handling. It then builds a specific use case on top of these utilities for managing a "landing page prompt."

---

## Detailed Explanation

### Purpose of This File

The primary goal of this file is twofold:

1.  **Provide Safe `localStorage` Utilities (`BrowserStorage` class):** `localStorage` is a browser-only API. If you try to access it in a Node.js environment (like during Server-Side Rendering), it will throw an error. This file creates a `BrowserStorage` class that wraps standard `localStorage` operations (`getItem`, `setItem`, `removeItem`, `isAvailable`) with checks to ensure they only run in a browser environment (`window` object exists) and include comprehensive `try...catch` blocks to handle potential runtime errors (e.g., `localStorage` being full, security restrictions). It also handles JSON serialization/deserialization for storing complex data types.

2.  **Manage a Landing Page Prompt (`LandingPromptStorage` class):** Using the safe `BrowserStorage` utilities, this file implements a specific feature: storing and retrieving a "landing page prompt." This prompt can have an associated timestamp, allowing for expiration logic (e.g., a message that only appears for 24 hours). The `consume` method is designed to retrieve the prompt *and then immediately clear it*, ensuring it's shown only once.

In essence, it's a foundation for reliable browser storage and a practical example of its use for a common UI pattern.

### Simplifying Complex Logic

The most "complex" aspects here are:

1.  **SSR Safety (`typeof window === 'undefined'`):** The code repeatedly checks `if (typeof window === 'undefined')`. This is a straightforward way to determine if the code is running in a browser. If `window` is not defined, it means we are in a server-side (Node.js) environment, and `localStorage` is not available. In such cases, the methods gracefully return a default value or `false`, preventing server crashes.

2.  **Robust Error Handling (`try...catch`):** Every `localStorage` operation is wrapped in a `try...catch` block. This is crucial because `localStorage` can fail for various reasons (e.g., user has disabled cookies, browser storage quota exceeded, security policies). Instead of crashing the application, these utilities log a warning and return a sensible fallback (e.g., `defaultValue` for `getItem`, `false` for `setItem`/`removeItem`).

3.  **JSON Serialization/Deserialization:** `localStorage` can only store strings. When you want to store objects, arrays, numbers, or booleans, you need to convert them to strings (serialize) before storing and convert them back (deserialize) after retrieving. The `BrowserStorage` class handles this automatically using `JSON.stringify` and `JSON.parse`. There's even an inner `try...catch` in `getItem` to handle cases where stored data might be malformed JSON.

### Line-by-Line Explanation

```typescript
/**
 * Safe localStorage utilities with SSR support
 * Provides clean error handling and type safety for browser storage operations
 */
```
*   **Doc Comment:** This is a JSDoc comment explaining the overall purpose of the file: it offers safe utilities for `localStorage`, supports Server-Side Rendering (SSR) by avoiding browser-specific APIs on the server, includes robust error handling, and ensures type safety with TypeScript.

```typescript
import { createLogger } from '@/lib/logs/console/logger'
```
*   **Import Logger:** Imports a `createLogger` function from a specified path. This function is used to create a logging instance for reporting warnings or errors.

```typescript
const logger = createLogger('BrowserStorage')
```
*   **Initialize Logger:** Creates a logger instance named `BrowserStorage`. This logger will be used to output messages related to `BrowserStorage` operations, particularly warnings when `localStorage` operations fail.

```typescript
/**
 * Safe localStorage operations with fallbacks
 */
export class BrowserStorage {
```
*   **Doc Comment & Class Definition:** Declares an exported class named `BrowserStorage`. This class encapsulates all the safe `localStorage` interaction methods. `export` means it can be imported and used in other files.

```typescript
  /**
   * Safely gets an item from localStorage
   * @param key - The storage key
   * @param defaultValue - The default value to return if key doesn't exist or access fails
   * @returns The stored value or default value
   */
  static getItem<T = string>(key: string, defaultValue: T): T {
```
*   **Doc Comment & Method Signature (`getItem`):**
    *   `static`: This method belongs to the class itself, not to an instance of the class. You'd call it like `BrowserStorage.getItem(...)`.
    *   `<T = string>`: This is a generic type parameter. It allows you to specify the expected type of the value being retrieved. If not specified, it defaults to `string`. For example, `BrowserStorage.getItem<number>('myNumber', 0)`.
    *   `(key: string, defaultValue: T): T`: Takes a `key` (string) to identify the item, and a `defaultValue` of type `T`. It returns a value of type `T`.

```typescript
    if (typeof window === 'undefined') {
      return defaultValue
    }
```
*   **SSR Check:** Checks if the `window` object is `undefined`. If it is (meaning the code is running in a non-browser environment like Node.js during SSR), it immediately returns the `defaultValue` because `localStorage` is not available.

```typescript
    try {
```
*   **Outer `try` Block:** Starts a `try` block to catch any errors that might occur during `localStorage` access (e.g., security errors, storage full errors).

```typescript
      const item = window.localStorage.getItem(key)
```
*   **Get Item:** Calls the native `window.localStorage.getItem(key)` method to retrieve the item associated with the given `key`. The result is a string or `null`.

```typescript
      if (item === null) {
        return defaultValue
      }
```
*   **Check for Null:** If `localStorage.getItem` returns `null`, it means no item with that `key` was found. In this case, the `defaultValue` is returned.

```typescript
      try {
        return JSON.parse(item) as T
      } catch {
        return item as T
      }
```
*   **Inner `try...catch` for JSON Parsing:**
    *   `try { return JSON.parse(item) as T }`: Attempts to parse the retrieved string (`item`) as JSON. This is crucial for retrieving objects, arrays, numbers, or booleans that were stored using `JSON.stringify`. `as T` asserts the type.
    *   `catch { return item as T }`: If `JSON.parse` fails (e.g., the stored string is not valid JSON, or it was originally a plain string not intended for JSON parsing), this `catch` block executes. It then returns the `item` as a raw string (or whatever `T` is specified to be), effectively falling back to the original string value. This makes `getItem` more resilient to potentially malformed or non-JSON data.

```typescript
    } catch (error) {
      logger.warn(`Failed to get localStorage item "${key}":`, error)
      return defaultValue
    }
```
*   **Outer `catch` Block:** If any error occurs within the outer `try` block (e.g., `localStorage` is not accessible due to browser settings), this block catches it.
    *   `logger.warn(...)`: Logs a warning message using the `BrowserStorage` logger, including the key and the error object for debugging.
    *   `return defaultValue`: Returns the `defaultValue` as a safe fallback when an error prevents successful retrieval.

```typescript
  /**
   * Safely sets an item in localStorage
   * @param key - The storage key
   * @param value - The value to store
   * @returns True if successful, false otherwise
   */
  static setItem<T>(key: string, value: T): boolean {
```
*   **Doc Comment & Method Signature (`setItem`):**
    *   `static`: A static method of the `BrowserStorage` class.
    *   `<T>`: A generic type parameter allowing any type of `value` to be passed.
    *   `(key: string, value: T): boolean`: Takes a `key` (string) and a `value` of type `T`. It returns `true` if the operation was successful, `false` otherwise.

```typescript
    if (typeof window === 'undefined') {
      return false
    }
```
*   **SSR Check:** Similar to `getItem`, returns `false` immediately if not in a browser environment.

```typescript
    try {
```
*   **`try` Block:** Starts a `try` block to catch potential errors during `localStorage.setItem`.

```typescript
      const serializedValue = typeof value === 'string' ? value : JSON.stringify(value)
```
*   **Serialization Logic:**
    *   `typeof value === 'string' ? value : JSON.stringify(value)`: This is a conditional (ternary) operator.
        *   If the `value` is already a string, it uses the `value` directly.
        *   If the `value` is *not* a string (e.g., an object, array, number, boolean), it converts it into a JSON string using `JSON.stringify()`. This ensures that all types can be stored in `localStorage`, which only accepts strings.

```typescript
      window.localStorage.setItem(key, serializedValue)
```
*   **Set Item:** Calls the native `window.localStorage.setItem(key, serializedValue)` to store the (potentially serialized) value under the given `key`.

```typescript
      return true
    } catch (error) {
      logger.warn(`Failed to set localStorage item "${key}":`, error)
      return false
    }
```
*   **`catch` Block:** If an error occurs during `setItem` (e.g., storage quota exceeded), it catches the error.
    *   `logger.warn(...)`: Logs a warning with details about the failed operation.
    *   `return false`: Returns `false` to indicate that the operation failed.

```typescript
  /**
   * Safely removes an item from localStorage
   * @param key - The storage key to remove
   * @returns True if successful, false otherwise
   */
  static removeItem(key: string): boolean {
```
*   **Doc Comment & Method Signature (`removeItem`):**
    *   `static`: A static method.
    *   `(key: string): boolean`: Takes a `key` (string) and returns `true` on success, `false` on failure.

```typescript
    if (typeof window === 'undefined') {
      return false
    }
```
*   **SSR Check:** Returns `false` if not in a browser environment.

```typescript
    try {
      window.localStorage.removeItem(key)
      return true
    } catch (error) {
      logger.warn(`Failed to remove localStorage item "${key}":`, error)
      return false
    }
```
*   **`try...catch` Block:**
    *   `window.localStorage.removeItem(key)`: Calls the native method to remove the item.
    *   `return true`: Returns `true` if successful.
    *   `catch`: If an error occurs, logs a warning and returns `false`.

```typescript
  /**
   * Check if localStorage is available
   * @returns True if localStorage is available and accessible
   */
  static isAvailable(): boolean {
```
*   **Doc Comment & Method Signature (`isAvailable`):**
    *   `static`: A static method.
    *   `(): boolean`: Returns `true` if `localStorage` is available and accessible, `false` otherwise.

```typescript
    if (typeof window === 'undefined') {
      return false
    }
```
*   **SSR Check:** Returns `false` if not in a browser environment.

```typescript
    try {
      const testKey = '__test_localStorage_availability__'
      window.localStorage.setItem(testKey, 'test')
      window.localStorage.removeItem(testKey)
      return true
    } catch {
      return false
    }
  }
}
```
*   **`try...catch` Block for Availability Check:** This is the standard, reliable way to check `localStorage` availability.
    *   `const testKey = '__test_localStorage_availability__'`: Defines a temporary key.
    *   `window.localStorage.setItem(testKey, 'test')`: Tries to set a dummy item. If this throws an error, `localStorage` is not available/accessible.
    *   `window.localStorage.removeItem(testKey)`: If setting was successful, removes the dummy item to clean up.
    *   `return true`: If both operations succeed without error, `localStorage` is available.
    *   `catch`: If any of the above operations throw an error (e.g., `SecurityError` in Safari private mode, or quota exceeded), this block catches it, and `return false` indicates `localStorage` is not available.

```typescript
/**
 * Constants for localStorage keys to avoid typos and provide centralized management
 */
export const STORAGE_KEYS = {
  LANDING_PAGE_PROMPT: 'sim_landing_page_prompt',
} as const
```
*   **Doc Comment & `STORAGE_KEYS` Object:**
    *   `export const STORAGE_KEYS`: Exports a constant object to hold all `localStorage` keys used in the application. This is a best practice to prevent typos and make key management easier.
    *   `LANDING_PAGE_PROMPT: 'sim_landing_page_prompt'`: Defines a specific key for the landing page prompt.
    *   `as const`: This is a TypeScript feature ("const assertion"). It tells TypeScript to infer the narrowest possible type for the object (i.e., its properties are literal string types, and the object itself is immutable). This provides stronger type safety, as you can't accidentally assign a new value to `STORAGE_KEYS.LANDING_PAGE_PROMPT` or use it as a non-literal string.

```typescript
/**
 * Specialized utility for managing the landing page prompt
 */
export class LandingPromptStorage {
```
*   **Doc Comment & Class Definition:** Declares an exported class `LandingPromptStorage`. This class provides a higher-level abstraction specifically for handling the landing page prompt, building upon the generic `BrowserStorage` class.

```typescript
  private static readonly KEY = STORAGE_KEYS.LANDING_PAGE_PROMPT
```
*   **Internal Key Definition:**
    *   `private static readonly KEY`: Declares a static, read-only property named `KEY` within this class.
    *   `= STORAGE_KEYS.LANDING_PAGE_PROMPT`: Assigns the centralized key from `STORAGE_KEYS` to this internal constant. `private` ensures it's only accessible within `LandingPromptStorage`.

```typescript
  /**
   * Store a prompt from the landing page
   * @param prompt - The prompt text to store
   * @returns True if successful, false otherwise
   */
  static store(prompt: string): boolean {
```
*   **Doc Comment & Method Signature (`store`):**
    *   `static`: A static method.
    *   `(prompt: string): boolean`: Takes the `prompt` text as a string and returns `true` on success, `false` on failure.

```typescript
    if (!prompt || prompt.trim().length === 0) {
      return false
    }
```
*   **Input Validation:** Checks if the `prompt` is empty, `null`, or consists only of whitespace. If so, it returns `false` as there's nothing meaningful to store.

```typescript
    const data = {
      prompt: prompt.trim(),
      timestamp: Date.now(),
    }
```
*   **Prepare Data:** Creates an object `data` to store in `localStorage`.
    *   `prompt: prompt.trim()`: Stores the trimmed prompt text.
    *   `timestamp: Date.now()`: Stores the current timestamp (milliseconds since epoch). This is essential for implementing prompt expiration.

```typescript
    return BrowserStorage.setItem(LandingPromptStorage.KEY, data)
  }
```
*   **Store Data:** Uses the generic `BrowserStorage.setItem` method to store the `data` object. The `BrowserStorage` class handles the JSON serialization. It returns the boolean result of `setItem`.

```typescript
  /**
   * Retrieve and consume the stored prompt
   * @param maxAge - Maximum age of the prompt in milliseconds (default: 24 hours)
   * @returns The stored prompt or null if not found/expired
   */
  static consume(maxAge: number = 24 * 60 * 60 * 1000): string | null {
```
*   **Doc Comment & Method Signature (`consume`):**
    *   `static`: A static method.
    *   `(maxAge: number = 24 * 60 * 60 * 1000)`: Takes an optional `maxAge` parameter (in milliseconds). Its default value is 24 hours (24 hours * 60 minutes * 60 seconds * 1000 milliseconds).
    *   `: string | null`: Returns the prompt string if valid and not expired, otherwise returns `null`. This method also *consumes* the prompt (removes it after retrieval).

```typescript
    const data = BrowserStorage.getItem<{ prompt: string; timestamp: number } | null>(
      LandingPromptStorage.KEY,
      null
    )
```
*   **Retrieve Data:** Uses `BrowserStorage.getItem` to retrieve the stored data.
    *   `<{ prompt: string; timestamp: number } | null>`: Explicitly tells TypeScript the expected type of the retrieved item: an object with `prompt` (string) and `timestamp` (number), or `null`.
    *   `null`: Provides `null` as the `defaultValue` if the item isn't found or an error occurs.

```typescript
    if (!data || !data.prompt || !data.timestamp) {
      return null
    }
```
*   **Data Validation:** Checks if `data` is `null`, or if it's missing the `prompt` or `timestamp` properties. If any are missing, the stored data is considered invalid, and `null` is returned.

```typescript
    const age = Date.now() - data.timestamp
    if (age > maxAge) {
      LandingPromptStorage.clear()
      return null
    }
```
*   **Expiration Check:**
    *   `const age = Date.now() - data.timestamp`: Calculates how old the stored prompt is in milliseconds.
    *   `if (age > maxAge)`: If the prompt's age exceeds the `maxAge`, it's considered expired.
        *   `LandingPromptStorage.clear()`: The expired prompt is immediately removed from `localStorage`.
        *   `return null`: `null` is returned because the prompt is no longer valid.

```typescript
    LandingPromptStorage.clear()
    return data.prompt
  }
```
*   **Consume and Return:**
    *   `LandingPromptStorage.clear()`: If the prompt is found and not expired, it is *consumed* by immediately clearing it from `localStorage`. This ensures the prompt is only presented once per `consume` call.
    *   `return data.prompt`: Returns the actual prompt text.

```typescript
  /**
   * Check if there's a stored prompt without consuming it
   * @param maxAge - Maximum age of the prompt in milliseconds (default: 24 hours)
   * @returns True if there's a valid prompt, false otherwise
   */
  static hasPrompt(maxAge: number = 24 * 60 * 60 * 1000): boolean {
```
*   **Doc Comment & Method Signature (`hasPrompt`):**
    *   `static`: A static method.
    *   `(maxAge: number = 24 * 60 * 60 * 1000)`: Same optional `maxAge` parameter with a default of 24 hours.
    *   `: boolean`: Returns `true` if a valid, unexpired prompt exists, `false` otherwise. This method *does not* clear the prompt.

```typescript
    const data = BrowserStorage.getItem<{ prompt: string; timestamp: number } | null>(
      LandingPromptStorage.KEY,
      null
    )

    if (!data || !data.prompt || !data.timestamp) {
      return false
    }

    const age = Date.now() - data.timestamp
    if (age > maxAge) {
      LandingPromptStorage.clear() // Clear expired prompt even if just checking
      return false
    }

    return true
  }
```
*   **Logic (Similar to `consume` but without immediate clearing):**
    *   Retrieves the data using `BrowserStorage.getItem`.
    *   Performs data validation (`!data || !data.prompt || !data.timestamp`).
    *   Calculates `age` and checks for expiration (`if (age > maxAge)`).
    *   **Important difference:** If the prompt is expired, it calls `LandingPromptStorage.clear()` to clean up the expired item, then returns `false`.
    *   If the prompt is valid and not expired, it simply `return true` *without* clearing the prompt, allowing it to be checked multiple times or consumed later.

```typescript
  /**
   * Clear the stored prompt
   * @returns True if successful, false otherwise
   */
  static clear(): boolean {
```
*   **Doc Comment & Method Signature (`clear`):**
    *   `static`: A static method.
    *   `(): boolean`: Returns `true` on successful removal, `false` on failure.

```typescript
    return BrowserStorage.removeItem(LandingPromptStorage.KEY)
  }
}
```
*   **Clear Logic:** Directly calls the generic `BrowserStorage.removeItem` method with the specific `LandingPromptStorage.KEY` to remove the prompt from `localStorage`. It returns the boolean result of `removeItem`.