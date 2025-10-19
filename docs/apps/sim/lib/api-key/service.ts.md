This TypeScript file, `api-key-service.ts`, serves as a comprehensive API key management and authentication service for the `@sim` application. It centralizes functionalities like authenticating incoming API keys, interacting with the database to retrieve or update key metadata, encrypting/decrypting API keys for secure storage, and generating new keys.

The file aims to provide a robust and flexible system for handling various API key scenarios, including different key types (personal, workspace), expiration, and supporting both legacy and modern encrypted key formats.

---

## Detailed Explanation

### Imports

This section brings in all the necessary modules and types from Node.js, the application's database, external libraries, and internal utilities.

```typescript
import { createCipheriv, createDecipheriv, randomBytes } from 'crypto'
// Imports Node.js built-in crypto module functions for encryption.
// - createCipheriv: Used to create an encryption cipher with a specific algorithm, key, and initialization vector (IV).
// - createDecipheriv: Used to create a decryption decipher.
// - randomBytes: Generates cryptographically strong pseudo-random data, crucial for IVs and keys.

import { db } from '@sim/db'
// Imports the Drizzle ORM database instance from the application's database module. This `db` object is used to execute all database queries.

import { apiKey as apiKeyTable, workspace } from '@sim/db/schema'
// Imports specific table schemas from the Drizzle ORM schema definition.
// - apiKeyTable: Represents the 'api_key' table in the database, containing API key details.
// - workspace: Represents the 'workspace' table, potentially used for linking workspace-specific API keys.

import { and, eq } from 'drizzle-orm'
// Imports helper functions from Drizzle ORM for constructing query conditions.
// - and: Combines multiple conditions with a logical AND operator.
// - eq: Creates an equality condition (e.g., column = value).

import { nanoid } from 'nanoid'
// Imports the `nanoid` library, a small, secure, URL-friendly unique string ID generator. Used for generating API key identifiers.

import { authenticateApiKey } from '@/lib/api-key/auth'
// Imports an external function responsible for the actual comparison and validation of an API key (e.g., hashing and matching). This function likely handles the core security logic for verifying a given API key string against a stored hashed key.

import { env } from '@/lib/env'
// Imports environment variables for the application, used here to retrieve the API encryption key.

import { createLogger } from '@/lib/logs/console/logger'
// Imports a utility to create a logger instance, used for structured logging within this service.
```

### Logger Initialization

```typescript
const logger = createLogger('ApiKeyService')
// Initializes a logger instance specifically for the 'ApiKeyService'. This allows logs from this file to be easily identified and filtered.
```

### Interfaces

These interfaces define the structure of data used for API key authentication options and results, improving type safety and readability.

```typescript
export interface ApiKeyAuthOptions {
  userId?: string // Optional user ID to filter API keys owned by a specific user.
  workspaceId?: string // Optional workspace ID to filter API keys associated with a specific workspace.
  keyTypes?: ('personal' | 'workspace')[] // Optional array of key types to filter by (e.g., 'personal' or 'workspace').
}

export interface ApiKeyAuthResult {
  success: boolean // Indicates if the authentication was successful.
  userId?: string // The ID of the user associated with the authenticated key (if successful).
  keyId?: string // The ID of the authenticated API key (if successful).
  keyType?: 'personal' | 'workspace' // The type of the authenticated key (if successful).
  workspaceId?: string // The ID of the workspace associated with the key (if successful, and applicable).
  error?: string // An error message if authentication failed.
}
```

### `authenticateApiKeyFromHeader` Function

This is the core function for authenticating an API key provided in an HTTP header, offering flexible filtering based on user, workspace, and key type.

```typescript
/**
 * Authenticate an API key from header with flexible filtering options
 */
export async function authenticateApiKeyFromHeader(
  apiKeyHeader: string, // The raw API key string received from the request header.
  options: ApiKeyAuthOptions = {} // Optional filtering criteria. Defaults to an empty object if not provided.
): Promise<ApiKeyAuthResult> {
  // If no API key is provided in the header, return an immediate failure.
  if (!apiKeyHeader) {
    return { success: false, error: 'API key required' }
  }

  try {
    // Build query based on options
    // Start building a Drizzle ORM select query to retrieve necessary fields from the apiKeyTable.
    let query = db
      .select({
        id: apiKeyTable.id, // Select the API key's unique ID.
        userId: apiKeyTable.userId, // Select the user ID associated with the key.
        workspaceId: apiKeyTable.workspaceId, // Select the workspace ID associated with the key.
        type: apiKeyTable.type, // Select the type of the key ('personal' or 'workspace').
        key: apiKeyTable.key, // Select the *stored* key value (likely hashed or encrypted).
        expiresAt: apiKeyTable.expiresAt, // Select the expiration timestamp of the key.
      })
      .from(apiKeyTable) // Specify the table to query from.

    // Add workspace join if needed for workspace keys
    // If a workspaceId filter is provided, or if 'workspace' key type is requested,
    // perform a LEFT JOIN with the `workspace` table. This allows filtering on workspace properties or ensures workspace-specific keys are correctly retrieved.
    if (options.workspaceId || options.keyTypes?.includes('workspace')) {
      query = query.leftJoin(workspace, eq(apiKeyTable.workspaceId, workspace.id)) as any
      // `as any` is used here to bypass potential Drizzle ORM type inference issues when dynamically adding joins.
    }

    // Apply filters
    const conditions = [] // Initialize an array to store Drizzle WHERE conditions.

    // If a userId is specified in options, add an equality condition for userId.
    if (options.userId) {
      conditions.push(eq(apiKeyTable.userId, options.userId))
    }

    // If a workspaceId is specified in options, add an equality condition for workspaceId.
    if (options.workspaceId) {
      conditions.push(eq(apiKeyTable.workspaceId, options.workspaceId))
    }

    // If keyTypes are specified in options:
    if (options.keyTypes?.length) {
      if (options.keyTypes.length === 1) {
        // If only one key type is specified, add a direct equality condition.
        conditions.push(eq(apiKeyTable.type, options.keyTypes[0]))
      } else {
        // If multiple key types are specified, Drizzle's `inArray` can be complex for dynamic queries.
        // The strategy here is to fetch all records matching other criteria and filter by key type in memory later.
        // This comment explicitly notes the complexity and the chosen approach.
      }
    }

    // If any conditions were added, apply them to the Drizzle query using an `AND` operator.
    if (conditions.length > 0) {
      query = query.where(and(...conditions)) as any
      // `as any` is used here again for similar reasons as the join.
    }

    const keyRecords = await query // Execute the constructed Drizzle query and await the results.

    // Filter by keyTypes in memory if multiple types specified
    // If multiple key types were specified in the options, apply an in-memory filter to the fetched records.
    const filteredRecords =
      options.keyTypes?.length && options.keyTypes.length > 1
        ? keyRecords.filter((record) => options.keyTypes!.includes(record.type as any))
        : keyRecords
    // This handles the case where `drizzle-orm` might not easily support `IN (...)` clauses for dynamically built queries, opting for client-side filtering.

    // Authenticate each key
    // Iterate through the potentially matching API key records from the database.
    for (const storedKey of filteredRecords) {
      // Skip expired keys
      // Check if the key has an expiration date and if it's in the past. If so, skip this key.
      if (storedKey.expiresAt && storedKey.expiresAt < new Date()) {
        continue // Move to the next key.
      }

      try {
        // Call the external `authenticateApiKey` function to compare the incoming API key with the stored key.
        // This function handles the actual security logic (e.g., comparing hashed values).
        const isValid = await authenticateApiKey(apiKeyHeader, storedKey.key)
        if (isValid) {
          // If the key is valid, return a successful authentication result with details.
          return {
            success: true,
            userId: storedKey.userId,
            keyId: storedKey.id,
            keyType: storedKey.type as 'personal' | 'workspace', // Cast type for strictness.
            workspaceId: storedKey.workspaceId || undefined, // Return workspaceId or undefined if null.
          }
        }
      } catch (error) {
        // Log any errors that occur during the `authenticateApiKey` call for a specific key.
        logger.error('Error authenticating API key:', error)
      }
    }

    // If the loop completes without finding a valid, unexpired key, return an invalid API key error.
    return { success: false, error: 'Invalid API key' }
  } catch (error) {
    // Catch any broader errors that occur during the database query or overall process.
    logger.error('API key authentication error:', error)
    return { success: false, error: 'Authentication failed' } // Return a generic authentication failed message.
  }
}
```

### `updateApiKeyLastUsed` Function

This function updates the `lastUsed` timestamp for a given API key in the database.

```typescript
/**
 * Update the last used timestamp for an API key
 */
export async function updateApiKeyLastUsed(keyId: string): Promise<void> {
  try {
    // Updates the `apiKeyTable` by setting the `lastUsed` column to the current date and time.
    // The update is applied to the row where the `id` matches the provided `keyId`.
    await db.update(apiKeyTable).set({ lastUsed: new Date() }).where(eq(apiKeyTable.id, keyId))
  } catch (error) {
    // Logs any errors encountered during the database update.
    logger.error('Error updating API key last used:', error)
  }
}
```

### `getApiKeyOwnerUserId` Function

This function retrieves the `userId` associated with a specific API key ID.

```typescript
/**
 * Given a pinned API key ID, resolve the owning userId (actor).
 * Returns null if not found.
 */
export async function getApiKeyOwnerUserId(
  pinnedApiKeyId: string | null | undefined // The ID of the API key to look up. Can be null or undefined.
): Promise<string | null> {
  // If no API key ID is provided, return null immediately.
  if (!pinnedApiKeyId) return null
  try {
    // Selects the `userId` from the `apiKeyTable` where the `id` matches `pinnedApiKeyId`.
    // `.limit(1)` ensures only one record is fetched, as IDs are unique.
    const rows = await db
      .select({ userId: apiKeyTable.userId })
      .from(apiKeyTable)
      .where(eq(apiKeyTable.id, pinnedApiKeyId))
      .limit(1)
    // Returns the `userId` of the first (and only) row if found, otherwise returns null.
    return rows[0]?.userId ?? null
  } catch (error) {
    // Logs any errors encountered during the database query.
    logger.error('Error resolving API key owner', { error, pinnedApiKeyId })
    return null // Return null on error.
  }
}
```

### `getApiEncryptionKey` Function

This utility function retrieves and validates the encryption key from environment variables.

```typescript
/**
 * Get the API encryption key from the environment
 * @returns The API encryption key as a Buffer, or null if not set.
 */
function getApiEncryptionKey(): Buffer | null {
  const key = env.API_ENCRYPTION_KEY // Reads the API_ENCRYPTION_KEY from environment variables.
  if (!key) {
    // If the key is not set, log a warning indicating that API keys will be stored in plain text.
    logger.warn(
      'API_ENCRYPTION_KEY not set - API keys will be stored in plain text. Consider setting this for better security.'
    )
    return null // Returns null, indicating no encryption key is available.
  }
  // Validate the key length. For AES-256, a 32-byte key is required, which translates to a 64-character hex string.
  if (key.length !== 64) {
    throw new Error('API_ENCRYPTION_KEY must be a 64-character hex string (32 bytes)')
  }
  // Converts the hex string key from the environment into a Buffer, as required by the crypto module.
  return Buffer.from(key, 'hex')
}
```

### `encryptApiKey` Function

This function encrypts an API key using AES-256-GCM.

```typescript
/**
 * Encrypts an API key using the dedicated API encryption key
 * @param apiKey - The API key string to encrypt
 * @returns A promise that resolves to an object containing the encrypted API key and IV.
 *          The encrypted key is formatted as "iv:encrypted:authTag".
 */
export async function encryptApiKey(apiKey: string): Promise<{ encrypted: string; iv: string }> {
  const key = getApiEncryptionKey() // Retrieve the API encryption key.

  // If no API encryption key is set (e.g., `API_ENCRYPTION_KEY` is missing in `.env`),
  // return the API key as-is without encryption. This provides backward compatibility.
  if (!key) {
    return { encrypted: apiKey, iv: '' }
  }

  const iv = randomBytes(16) // Generate a random 16-byte (128-bit) Initialization Vector (IV). The IV must be unique for each encryption operation with the same key.
  const cipher = createCipheriv('aes-256-gcm', key, iv) // Create a cipher using AES-256 in Galois/Counter Mode (GCM), with the encryption key and IV. GCM provides both confidentiality and authenticity.
  let encrypted = cipher.update(apiKey, 'utf8', 'hex') // Encrypt the API key data. The input is UTF-8, and the output is a hex string.
  encrypted += cipher.final('hex') // Finalize the encryption process.

  const authTag = cipher.getAuthTag() // Retrieve the authentication tag from the GCM cipher. This tag is used to verify data integrity during decryption.

  // Return the encrypted value. The format "iv:encrypted:authTag" is used for easy parsing during decryption.
  // The IV is also returned separately, though it's part of the `encrypted` string.
  return {
    encrypted: `${iv.toString('hex')}:${encrypted}:${authTag.toString('hex')}`,
    iv: iv.toString('hex'),
  }
}
```

### `decryptApiKey` Function

This function decrypts an API key that was encrypted using `encryptApiKey`.

```typescript
/**
 * Decrypts an API key using the dedicated API encryption key
 * @param encryptedValue - The encrypted value in format "iv:encrypted:authTag" or a plain text key.
 * @returns A promise that resolves to an object containing the decrypted API key.
 */
export async function decryptApiKey(encryptedValue: string): Promise<{ decrypted: string }> {
  // Check if this is actually encrypted (contains colons)
  // If the value does not contain colons, or doesn't have exactly 3 parts (iv, encrypted, authTag),
  // it's treated as a plain text key (e.g., legacy key or stored without encryption).
  if (!encryptedValue.includes(':') || encryptedValue.split(':').length !== 3) {
    // This is a plain text key, return as-is.
    return { decrypted: encryptedValue }
  }

  const key = getApiEncryptionKey() // Retrieve the API encryption key.

  // If no API encryption key is set, assume the value is plain text and return it as-is.
  // This is a safety net for environments where encryption was never configured, even if a colon is present (though unlikely to happen with the specific format).
  if (!key) {
    return { decrypted: encryptedValue }
  }

  const parts = encryptedValue.split(':') // Split the encrypted value into its components: IV, encrypted data, and authentication tag.
  const ivHex = parts[0] // Extract the IV (hex string).
  const authTagHex = parts[parts.length - 1] // Extract the authentication tag (hex string).
  const encrypted = parts.slice(1, -1).join(':') // Extract the actual encrypted data. `join(':')` handles cases where encrypted data might contain colons itself (unlikely with hex output, but robust).

  // Basic validation to ensure all parts were successfully extracted.
  if (!ivHex || !encrypted || !authTagHex) {
    throw new Error('Invalid encrypted API key format. Expected "iv:encrypted:authTag"')
  }

  const iv = Buffer.from(ivHex, 'hex') // Convert the IV hex string back to a Buffer.
  const authTag = Buffer.from(authTagHex, 'hex') // Convert the authentication tag hex string back to a Buffer.

  try {
    const decipher = createDecipheriv('aes-256-gcm', key, iv) // Create a decipher using AES-256-GCM, the key, and the IV.
    decipher.setAuthTag(authTag) // Set the authentication tag. This is crucial for GCM to verify the data's integrity and authenticity during decryption. If the data or tag has been tampered with, decryption will fail.

    let decrypted = decipher.update(encrypted, 'hex', 'utf8') // Decrypt the main data. Input is hex, output is UTF-8.
    decrypted += decipher.final('utf8') // Finalize the decryption process.

    return { decrypted } // Return the successfully decrypted API key.
  } catch (error: unknown) {
    // Catch any errors during decryption (e.g., incorrect key, tampered data, bad auth tag).
    logger.error('API key decryption error:', {
      error: error instanceof Error ? error.message : 'Unknown error',
    })
    throw error // Re-throw the error after logging, as decryption failure is a critical issue.
  }
}
```

### `generateApiKey` Function

Generates a legacy-formatted API key.

```typescript
/**
 * Generates a standardized API key with the 'sim_' prefix (legacy format)
 * @returns A new API key string
 */
export function generateApiKey(): string {
  // Uses `nanoid` to generate a 32-character random string and prepends 'sim_'.
  return `sim_${nanoid(32)}`
}
```

### `generateEncryptedApiKey` Function

Generates a new API key string with a specific prefix indicating it's intended for encrypted storage.

```typescript
/**
 * Generates a new encrypted API key with the 'sk-sim-' prefix
 * @returns A new encrypted API key string
 */
export function generateEncryptedApiKey(): string {
  // Uses `nanoid` to generate a 32-character random string and prepends 'sk-sim-'.
  // This prefix indicates that the key *should* be encrypted before storage.
  return `sk-sim-${nanoid(32)}`
}
```

### `isEncryptedApiKeyFormat` Function

Checks if an API key adheres to the format used for encrypted keys.

```typescript
/**
 * Determines if an API key uses the new encrypted format based on prefix
 * @param apiKey - The API key string to check
 * @returns true if the key uses the new encrypted format (sk-sim- prefix)
 */
export function isEncryptedApiKeyFormat(apiKey: string): boolean {
  return apiKey.startsWith('sk-sim-') // Checks if the API key starts with the 'sk-sim-' prefix.
}
```

### `isLegacyApiKeyFormat` Function

Checks if an API key adheres to the legacy format.

```typescript
/**
 * Determines if an API key uses the legacy format based on prefix
 * @param apiKey - The API key string to check
 * @returns true if the key uses the legacy format (sim_ prefix)
 */
export function isLegacyApiKeyFormat(apiKey: string): boolean {
  // Checks if the API key starts with 'sim_' AND does NOT start with 'sk-sim-'.
  // The second condition is important to distinguish true legacy keys from potentially new keys that might accidentally start with 'sim_' but are meant for encryption.
  return apiKey.startsWith('sim_') && !apiKey.startsWith('sk-sim-')
}
```