This TypeScript file, `api-key.ts`, serves as a comprehensive utility for managing and authenticating API keys within an application. Its primary goal is to provide a robust system that can gracefully handle a migration from older, plain-text API keys to newer, more secure encrypted API keys, all without breaking existing functionality for users still relying on the legacy format.

In essence, this file empowers the application to:
1.  **Generate** new API keys.
2.  **Encrypt** API keys for secure storage.
3.  **Decrypt** stored API keys when needed (e.g., for authentication or display).
4.  **Authenticate** an incoming API key against a stored key, intelligently determining if the stored key is encrypted or plain-text.
5.  **Format** API keys for display, showing a user-friendly masked version.
6.  Perform basic **validation** on API key formats.

This dual-format support is crucial for applications that need to upgrade their security practices incrementally.

---

### Code Explanation

Let's break down the code section by section.

#### Imports

```typescript
import {
  decryptApiKey,
  encryptApiKey,
  generateApiKey,
  generateEncryptedApiKey,
  isEncryptedApiKeyFormat,
  isLegacyApiKeyFormat,
} from '@/lib/api-key/service'
import { env } from '@/lib/env'
import { createLogger } from '@/lib/logs/console/logger'
```

*   **`import { ... } from '@/lib/api-key/service'`**: This line imports several core functions related to API key operations from a separate service file. These functions are likely low-level cryptographic or key generation utilities:
    *   `decryptApiKey`: A function to decrypt an encrypted API key.
    *   `encryptApiKey`: A function to encrypt a plain-text API key.
    *   `generateApiKey`: Generates a generic plain-text API key.
    *   `generateEncryptedApiKey`: Generates an API key specifically formatted to *look like* a new, encrypted key (even though it's still plain-text at this stage, it will later be truly encrypted for storage).
    *   `isEncryptedApiKeyFormat`: Checks if a given API key string *follows the format* of a new, encrypted key (e.g., starts with `sk-sim-`).
    *   `isLegacyApiKeyFormat`: Checks if a given API key string *follows the format* of an older, legacy plain-text key (e.g., starts with `sim_`).
*   **`import { env } from '@/lib/env'`**: Imports the `env` object, which provides access to environment variables (like `API_ENCRYPTION_KEY`). This is crucial for configuration, especially for enabling or disabling encryption features.
*   **`import { createLogger } from '@/lib/logs/console/logger'`**: Imports a utility to create a logger instance. This allows the file to emit log messages for debugging and monitoring purposes.

#### Logger Setup

```typescript
const logger = createLogger('ApiKeyAuth')
```

*   **`const logger = createLogger('ApiKeyAuth')`**: Initializes a logger instance specifically named 'ApiKeyAuth'. This makes it easy to identify log messages originating from this file in the application's overall log output.

#### Module Description

```typescript
/**
 * API key authentication utilities supporting both legacy plain text keys
 * and modern encrypted keys for gradual migration without breaking existing keys
 */
```

*   This JSDoc comment clearly states the purpose of the file: to provide utilities for API key authentication, with a key focus on supporting both old (plain text) and new (encrypted) formats for a smooth migration.

#### `isEncryptedKey` Function

```typescript
/**
 * Checks if a stored key is in the new encrypted format
 * @param storedKey - The key stored in the database
 * @returns true if the key is encrypted, false if it's plain text
 */
export function isEncryptedKey(storedKey: string): boolean {
  // Check if it follows the encrypted format: iv:encrypted:authTag
  return storedKey.includes(':') && storedKey.split(':').length === 3
}
```

*   **Purpose**: This helper function determines if a key, as it's currently stored in the database, is in the format of an encrypted key. Encrypted keys are typically represented as a concatenated string of an Initialization Vector (IV), the actual encrypted payload, and an authentication tag, all separated by colons.
*   **`export function isEncryptedKey(storedKey: string): boolean { ... }`**: Defines an exported function named `isEncryptedKey` that takes one argument, `storedKey` (a string), and returns a boolean.
*   **`return storedKey.includes(':') && storedKey.split(':').length === 3`**: This is the core logic.
    *   `storedKey.includes(':')`: Checks if the `storedKey` string contains at least one colon. If not, it can't be an encrypted key in the expected format.
    *   `storedKey.split(':')`: Splits the `storedKey` string into an array of substrings, using the colon as a delimiter.
    *   `.length === 3`: Checks if the resulting array has exactly three parts. This verifies the `iv:encrypted:authTag` structure.
    *   The `&&` (logical AND) ensures both conditions must be true for the key to be considered encrypted.

#### `authenticateApiKey` Function

```typescript
/**
 * Authenticates an API key against a stored key, supporting both legacy and new encrypted formats
 * @param inputKey - The API key provided by the client
 * @param storedKey - The key stored in the database (may be plain text or encrypted)
 * @returns Promise<boolean> - true if the key is valid
 */
export async function authenticateApiKey(inputKey: string, storedKey: string): Promise<boolean> {
  try {
    // If input key has new encrypted prefix (sk-sim-), only check against encrypted storage
    if (isEncryptedApiKeyFormat(inputKey)) {
      if (isEncryptedKey(storedKey)) {
        try {
          const { decrypted } = await decryptApiKey(storedKey)
          return inputKey === decrypted
        } catch (decryptError) {
          logger.error('Failed to decrypt stored API key:', { error: decryptError })
          return false
        }
      }
      // New format keys should never match against plain text storage
      return false
    }

    // If input key has legacy prefix (sim_), check both encrypted and plain text
    if (isLegacyApiKeyFormat(inputKey)) {
      if (isEncryptedKey(storedKey)) {
        try {
          const { decrypted } = await decryptApiKey(storedKey)
          return inputKey === decrypted
        } catch (decryptError) {
          logger.error('Failed to decrypt stored API key:', { error: decryptError })
          // Fall through to plain text comparison if decryption fails
        }
      }
      // Legacy format can match against plain text storage
      return inputKey === storedKey
    }

    // If no recognized prefix, fall back to original behavior
    if (isEncryptedKey(storedKey)) {
      try {
        const { decrypted } = await decryptApiKey(storedKey)
        return inputKey === decrypted
      } catch (decryptError) {
        logger.error('Failed to decrypt stored API key:', { error: decryptError })
      }
    }

    return inputKey === storedKey
  } catch (error) {
    logger.error('API key authentication error:', { error })
    return false
  }
}
```

*   **Purpose**: This is the most complex function, responsible for securely authenticating an API key provided by a client (`inputKey`) against a key stored in the database (`storedKey`). It intelligently handles all four combinations:
    1.  Client provides new-format key, database has encrypted key.
    2.  Client provides new-format key, database has plain-text key.
    3.  Client provides legacy-format key, database has encrypted key.
    4.  Client provides legacy-format key, database has plain-text key.
    5.  Client provides unknown-format key, database has either.
*   **Simplifying Complex Logic**: The function uses a series of `if` statements to categorize the `inputKey` first, then branches based on the `storedKey` format. The key takeaway is the priority: New format keys are stricter, while legacy keys are more flexible to allow for migration.
*   **`export async function authenticateApiKey(...) { ... }`**: Defines an exported asynchronous function. It takes `inputKey` (from the client) and `storedKey` (from the database) as strings and returns a Promise that resolves to a boolean (`true` for success, `false` for failure).
*   **`try { ... } catch (error) { ... }`**: An outer `try...catch` block wraps the entire authentication logic to catch any unexpected errors during the process and log them.
*   **`if (isEncryptedApiKeyFormat(inputKey)) { ... }`**: This block executes if the `inputKey` provided by the client has the prefix of the *new*, encrypted key format (e.g., `sk-sim-`).
    *   **`if (isEncryptedKey(storedKey)) { ... }`**: If the `storedKey` from the database is also in the encrypted format:
        *   **`try { const { decrypted } = await decryptApiKey(storedKey); return inputKey === decrypted; }`**: It attempts to decrypt the `storedKey` using `decryptApiKey`. If successful, it compares the original `inputKey` (which is plain-text, but formatted as if it were encrypted) with the `decrypted` stored key. If they match, authentication succeeds.
        *   **`catch (decryptError) { logger.error(...); return false; }`**: If decryption fails (e.g., corrupt key, wrong encryption key), it logs an error and returns `false`.
    *   **`return false`**: If the `inputKey` is in the new format, but the `storedKey` is *not* encrypted (i.e., it's plain text), authentication *fails*. New-format keys are expected to only match against encrypted storage.
*   **`if (isLegacyApiKeyFormat(inputKey)) { ... }`**: This block executes if the `inputKey` provided by the client has the prefix of the *legacy* key format (e.g., `sim_`).
    *   **`if (isEncryptedKey(storedKey)) { ... }`**: If the `storedKey` from the database is in the encrypted format:
        *   **`try { const { decrypted } = await decryptApiKey(storedKey); return inputKey === decrypted; }`**: It attempts to decrypt the `storedKey`. If successful, it compares the legacy `inputKey` with the `decrypted` stored key. If they match, authentication succeeds.
        *   **`catch (decryptError) { logger.error(...); }`**: If decryption fails, it logs an error but *does not immediately return `false`*. This is a crucial part of the migration strategy: if a legacy key is provided and the stored key *looks* encrypted but fails to decrypt, the system *falls through* to the next check, assuming the stored key might still be a plain-text legacy key. This prevents breaking old clients if a migration attempt for a specific key failed or is incomplete.
    *   **`return inputKey === storedKey`**: This line is reached if the `storedKey` was not encrypted *or* if decryption of an encrypted `storedKey` failed. It performs a direct plain-text comparison between the `inputKey` and `storedKey`. This handles cases where both keys are legacy plain text, or where a legacy `inputKey` is being compared to a `storedKey` that *should* have been encrypted but wasn't (or failed decryption).
*   **`if (isEncryptedKey(storedKey)) { ... }`**: This block is a fallback for when the `inputKey` doesn't match either the new or legacy specific formats. It first checks if the `storedKey` is encrypted.
    *   **`try { const { decrypted } = await decryptApiKey(storedKey); return inputKey === decrypted; }`**: Attempts to decrypt the `storedKey` and compare it to the `inputKey`.
    *   **`catch (decryptError) { logger.error(...); }`**: Logs an error if decryption fails, but again, *falls through* to the next comparison.
*   **`return inputKey === storedKey`**: This is the final fallback. If none of the above conditions led to a successful authentication, it performs a direct plain-text comparison. This covers situations where keys have no specific prefix or if all previous decryption attempts failed.

#### `encryptApiKeyForStorage` Function

```typescript
/**
 * Encrypts an API key for secure storage
 * @param apiKey - The plain text API key to encrypt
 * @returns Promise<string> - The encrypted key
 */
export async function encryptApiKeyForStorage(apiKey: string): Promise<string> {
  try {
    const { encrypted } = await encryptApiKey(apiKey)
    return encrypted
  } catch (error) {
    logger.error('API key encryption error:', { error })
    throw new Error('Failed to encrypt API key')
  }
}
```

*   **Purpose**: Encrypts a given plain-text API key, preparing it for secure storage in a database or other persistent storage.
*   **`export async function encryptApiKeyForStorage(apiKey: string): Promise<string> { ... }`**: Defines an exported asynchronous function. It takes a plain `apiKey` string and returns a Promise resolving to the encrypted string.
*   **`try { ... } catch (error) { ... }`**: A `try...catch` block to handle potential errors during encryption.
*   **`const { encrypted } = await encryptApiKey(apiKey)`**: Calls the imported `encryptApiKey` service function, which performs the actual cryptographic encryption. It likely returns an object, and we destructure the `encrypted` property from it.
*   **`return encrypted`**: Returns the resulting encrypted string.
*   **`logger.error(...)`**: Logs any encryption errors.
*   **`throw new Error('Failed to encrypt API key')`**: Re-throws a more generic error, signaling that the encryption operation failed to the caller.

#### `createApiKey` Function

```typescript
/**
 * Creates a new API key
 * @param useStorage - Whether to encrypt the key before storage (default: true)
 * @returns Promise<{key: string, encryptedKey?: string}> - The plain key and optionally encrypted version
 */
export async function createApiKey(useStorage = true): Promise<{
  key: string
  encryptedKey?: string
}> {
  try {
    const hasEncryptionKey = env.API_ENCRYPTION_KEY !== undefined

    const plainKey = hasEncryptionKey ? generateEncryptedApiKey() : generateApiKey()

    if (useStorage) {
      const encryptedKey = await encryptApiKeyForStorage(plainKey)
      return { key: plainKey, encryptedKey }
    }

    return { key: plainKey }
  } catch (error) {
    logger.error('API key creation error:', { error })
    throw new Error('Failed to create API key')
  }
}
```

*   **Purpose**: Generates a brand new API key. It can optionally encrypt this key immediately if it's intended for storage.
*   **Simplifying Complex Logic**: It first decides which type of plain key to generate (one that looks like the new format or a generic one), then conditionally encrypts it based on the `useStorage` flag.
*   **`export async function createApiKey(useStorage = true): Promise<{ key: string; encryptedKey?: string }> { ... }`**: Defines an exported asynchronous function. It has an optional parameter `useStorage` (defaulting to `true`). It returns a Promise resolving to an object containing the plain `key` and an optional `encryptedKey` if encryption was performed.
*   **`try { ... } catch (error) { ... }`**: A `try...catch` block for error handling during key creation.
*   **`const hasEncryptionKey = env.API_ENCRYPTION_KEY !== undefined`**: Checks if an encryption key is configured in the environment variables. This determines if the application *can* use the new encrypted key features.
*   **`const plainKey = hasEncryptionKey ? generateEncryptedApiKey() : generateApiKey()`**: This line conditionally generates the plain-text API key:
    *   If `hasEncryptionKey` is `true`, it uses `generateEncryptedApiKey()`, which creates a key formatted like the new `sk-sim-` keys (even though it's still plain text at this stage).
    *   Otherwise, it uses `generateApiKey()`, which creates a more generic API key.
*   **`if (useStorage) { ... }`**: If `useStorage` is `true` (meaning the key needs to be stored securely):
    *   **`const encryptedKey = await encryptApiKeyForStorage(plainKey)`**: The generated `plainKey` is encrypted using the `encryptApiKeyForStorage` function we just explained.
    *   **`return { key: plainKey, encryptedKey }`**: Returns an object containing both the original `plainKey` and its `encryptedKey` version.
*   **`return { key: plainKey }`**: If `useStorage` is `false`, only the `plainKey` is returned.
*   **`logger.error(...)`** and **`throw new Error(...)`**: Handle errors during key creation.

#### `decryptApiKeyFromStorage` Function

```typescript
/**
 * Decrypts an API key from storage for display purposes
 * @param encryptedKey - The encrypted API key from the database
 * @returns Promise<string> - The decrypted API key
 */
export async function decryptApiKeyFromStorage(encryptedKey: string): Promise<string> {
  try {
    const { decrypted } = await decryptApiKey(encryptedKey)
    return decrypted
  } catch (error) {
    logger.error('API key decryption error:', { error })
    throw new Error('Failed to decrypt API key')
  }
}
```

*   **Purpose**: Decrypts an API key that was previously retrieved from storage, typically for displaying to a user or for authentication (though `authenticateApiKey` handles authentication internally).
*   **`export async function decryptApiKeyFromStorage(encryptedKey: string): Promise<string> { ... }`**: Defines an exported asynchronous function. It takes an `encryptedKey` string and returns a Promise resolving to the plain-text decrypted string.
*   **`try { ... } catch (error) { ... }`**: A `try...catch` block for error handling.
*   **`const { decrypted } = await decryptApiKey(encryptedKey)`**: Calls the imported `decryptApiKey` service function, which performs the actual cryptographic decryption. It expects an object and destructures the `decrypted` property.
*   **`return decrypted`**: Returns the resulting plain-text decrypted string.
*   **`logger.error(...)`** and **`throw new Error(...)`**: Handle errors during decryption.

#### `getApiKeyLast4` Function

```typescript
/**
 * Gets the last 4 characters of an API key for display purposes
 * @param apiKey - The API key (plain text)
 * @returns string - The last 4 characters
 */
export function getApiKeyLast4(apiKey: string): string {
  return apiKey.slice(-4)
}
```

*   **Purpose**: A simple utility to extract the last four characters of a *plain-text* API key. This is commonly used for display purposes, allowing users to identify their keys without revealing the full secret.
*   **`export function getApiKeyLast4(apiKey: string): string { ... }`**: Defines an exported function. It takes a `apiKey` string and returns a string.
*   **`return apiKey.slice(-4)`**: Uses the `slice()` string method. Passing a negative number to `slice()` makes it count from the end of the string. So, `-4` means "get the last 4 characters".

#### `getApiKeyDisplayFormat` Function

```typescript
/**
 * Gets the display format for an API key showing prefix and last 4 characters
 * @param encryptedKey - The encrypted API key from the database
 * @returns Promise<string> - The display format like "sk-sim-...r6AA"
 */
export async function getApiKeyDisplayFormat(encryptedKey: string): Promise<string> {
  try {
    if (isEncryptedKey(encryptedKey)) {
      const decryptedKey = await decryptApiKeyFromStorage(encryptedKey)
      return formatApiKeyForDisplay(decryptedKey)
    }
    // For plain text keys (legacy), format directly
    return formatApiKeyForDisplay(encryptedKey)
  } catch (error) {
    logger.error('Failed to format API key for display:', { error })
    return '****'
  }
}
```

*   **Purpose**: Provides a user-friendly, masked representation of an API key (e.g., "sk-sim-...r6AA" or "sim\_...r6AA"). It smartly handles whether the input `encryptedKey` parameter is actually encrypted or a legacy plain-text key.
*   **`export async function getApiKeyDisplayFormat(encryptedKey: string): Promise<string> { ... }`**: Defines an exported asynchronous function. It takes an `encryptedKey` string (though it might be plain text) and returns a Promise resolving to the formatted display string.
*   **`try { ... } catch (error) { ... }`**: A `try...catch` block for error handling. On failure, it returns a generic `****`.
*   **`if (isEncryptedKey(encryptedKey)) { ... }`**: Checks if the provided key is in the encrypted format (using the `isEncryptedKey` helper from earlier).
    *   **`const decryptedKey = await decryptApiKeyFromStorage(encryptedKey)`**: If it is encrypted, it decrypts the key to get its plain-text value.
    *   **`return formatApiKeyForDisplay(decryptedKey)`**: The decrypted (plain-text) key is then passed to `formatApiKeyForDisplay` for actual formatting.
*   **`return formatApiKeyForDisplay(encryptedKey)`**: If the provided `encryptedKey` was *not* in the encrypted format (meaning it's likely a legacy plain-text key), it's passed directly to `formatApiKeyForDisplay` without decryption.

#### `formatApiKeyForDisplay` Function

```typescript
/**
 * Formats an API key for display showing prefix and last 4 characters
 * @param apiKey - The API key (plain text)
 * @returns string - The display format like "sk-sim-...r6AA" or "sim_...r6AA"
 */
export function formatApiKeyForDisplay(apiKey: string): string {
  if (isEncryptedApiKeyFormat(apiKey)) {
    // For sk-sim- format: "sk-sim-...r6AA"
    const last4 = getApiKeyLast4(apiKey)
    return `sk-sim-...${last4}`
  }
  if (isLegacyApiKeyFormat(apiKey)) {
    // For sim_ format: "sim_...r6AA"
    const last4 = getApiKeyLast4(apiKey)
    return `sim_...${last4}`
  }
  // Unknown format, just show last 4
  const last4 = getApiKeyLast4(apiKey)
  return `...${last4}`
}
```

*   **Purpose**: This function takes a *plain-text* API key and formats it into a user-friendly display string, including its prefix (if recognized) and the last four characters, like `sk-sim-...r6AA`.
*   **`export function formatApiKeyForDisplay(apiKey: string): string { ... }`**: Defines an exported function that takes a plain `apiKey` string and returns a formatted string.
*   **`if (isEncryptedApiKeyFormat(apiKey)) { ... }`**: Checks if the `apiKey` starts with the prefix for the new encrypted format (e.g., `sk-sim-`).
    *   **`const last4 = getApiKeyLast4(apiKey)`**: Gets the last four characters.
    *   **`return `sk-sim-...${last4}``**: Constructs the display string using a template literal.
*   **`if (isLegacyApiKeyFormat(apiKey)) { ... }`**: Checks if the `apiKey` starts with the prefix for the legacy format (e.g., `sim_`).
    *   **`const last4 = getApiKeyLast4(apiKey)`**: Gets the last four characters.
    *   **`return `sim_...${last4}``**: Constructs the display string.
*   **`const last4 = getApiKeyLast4(apiKey); return `...${last4}``**: This is a fallback. If the key doesn't match either a new or legacy prefix, it still displays the last four characters prefixed with `...` (e.g., `...r6AA`).

#### `getEncryptedApiKeyLast4` Function

```typescript
/**
 * Gets the last 4 characters of an encrypted API key by decrypting it first
 * @param encryptedKey - The encrypted API key from the database
 * @returns Promise<string> - The last 4 characters
 */
export async function getEncryptedApiKeyLast4(encryptedKey: string): Promise<string> {
  try {
    if (isEncryptedKey(encryptedKey)) {
      const decryptedKey = await decryptApiKeyFromStorage(encryptedKey)
      return getApiKeyLast4(decryptedKey)
    }
    // For plain text keys (legacy), return last 4 directly
    return getApiKeyLast4(encryptedKey)
  } catch (error) {
    logger.error('Failed to get last 4 characters of API key:', { error })
    return '****'
  }
}
```

*   **Purpose**: Similar to `getApiKeyLast4`, but specifically designed to get the last four characters of a key that *might be encrypted* in storage. It ensures decryption happens first if necessary.
*   **`export async function getEncryptedApiKeyLast4(encryptedKey: string): Promise<string> { ... }`**: Defines an exported asynchronous function. It takes an `encryptedKey` string and returns a Promise resolving to the last four characters.
*   **`try { ... } catch (error) { ... }`**: A `try...catch` block for error handling. On failure, it returns `****`.
*   **`if (isEncryptedKey(encryptedKey)) { ... }`**: Checks if the provided key is in the encrypted format.
    *   **`const decryptedKey = await decryptApiKeyFromStorage(encryptedKey)`**: If encrypted, it decrypts the key.
    *   **`return getApiKeyLast4(decryptedKey)`**: Returns the last four characters of the now plain-text key.
*   **`return getApiKeyLast4(encryptedKey)`**: If the provided `encryptedKey` was *not* encrypted (i.e., it was a plain-text legacy key), its last four characters are retrieved directly.

#### `isValidApiKeyFormat` Function

```typescript
/**
 * Validates API key format (basic validation)
 * @param apiKey - The API key to validate
 * @returns boolean - true if the format appears valid
 */
export function isValidApiKeyFormat(apiKey: string): boolean {
  return typeof apiKey === 'string' && apiKey.length > 10 && apiKey.length < 200
}
```

*   **Purpose**: Provides a very basic, preliminary check to ensure an API key string has a reasonable format (is a string and falls within expected length boundaries). This is a sanity check, not a cryptographic validation.
*   **`export function isValidApiKeyFormat(apiKey: string): boolean { ... }`**: Defines an exported function that takes an `apiKey` string and returns a boolean.
*   **`return typeof apiKey === 'string' && apiKey.length > 10 && apiKey.length < 200`**: This line performs three checks:
    *   `typeof apiKey === 'string'`: Ensures the input is actually a string.
    *   `apiKey.length > 10`: Ensures the key is not too short (a common sign of an invalid or incomplete key).
    *   `apiKey.length < 200`: Ensures the key is not excessively long (which might indicate malformed data or an attack attempt).
    *   All three conditions must be true for the key's format to be considered "valid" by this basic check.

---

This file encapsulates critical security and migration logic for API keys, making it a central point for all API key operations in the application, from creation to authentication and display. Its careful handling of both legacy and modern formats ensures a smooth transition to enhanced security without disrupting existing users.