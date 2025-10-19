This file, `utils.ts`, acts as a central repository for a wide array of reusable helper functions within the application. It's designed to provide common utilities that don't belong to any specific feature or module but are needed across different parts of the codebase.

The functionalities covered in this file include:

*   **Security & Cryptography**: Encrypting and decrypting sensitive data using robust AES-256-GCM, generating secure passwords.
*   **UI & Styling**: Merging Tailwind CSS classes intelligently.
*   **Configuration Management**: Accessing environment variables and implementing API key rotation.
*   **Logging**: Creating a dedicated logger instance for this utility module.
*   **Scheduling**: Converting various human-readable schedule options into standard Cron expressions.
*   **Date & Time**: Formatting dates, times, and durations, and providing user-friendly timezone abbreviations.
*   **Data Manipulation**: Redacting sensitive API keys from objects and validating/sanitizing names.
*   **Networking**: Defining headers and encoding data for Server-Sent Events (SSE).
*   **General Helpers**: Generating request IDs and providing a no-operation function.

By centralizing these utilities, the file promotes code reusability, consistency, and easier maintenance across the project.

---

## Detailed Code Explanation

Let's break down each part of the `utils.ts` file.

### 1. Imports

This section brings in necessary modules and types from various libraries and internal project paths.

```typescript
import { createCipheriv, createDecipheriv, randomBytes } from 'crypto'
import { type ClassValue, clsx } from 'clsx'
import { twMerge } from 'tailwind-merge'
import { env } from '@/lib/env'
import { createLogger } from '@/lib/logs/console/logger'
```

*   `import { createCipheriv, createDecipheriv, randomBytes } from 'crypto'`: Imports specific functions from Node.js's built-in `crypto` module.
    *   `createCipheriv`: Used to create an encryption cipher (for symmetric encryption with an Initialization Vector).
    *   `createDecipheriv`: Used to create a decryption decipher.
    *   `randomBytes`: Generates cryptographically strong pseudo-random data, essential for secure IVs.
*   `import { type ClassValue, clsx } from 'clsx'`: Imports `clsx`, a utility for constructing `className` strings conditionally, and `ClassValue`, its associated type.
*   `import { twMerge } from 'tailwind-merge'`: Imports `twMerge`, a library that intelligently merges Tailwind CSS classes, resolving conflicts (e.g., `p-4` and `p-6` would resolve to `p-6`).
*   `import { env } from '@/lib/env'`: Imports the `env` object from an internal library (`@/lib/env`). This object is presumed to securely load and expose environment variables (like API keys, encryption keys, etc.) that are configured for the application.
*   `import { createLogger } from '@/lib/logs/console/logger'`: Imports a `createLogger` function from an internal logging utility (`@/lib/logs/console/logger`). This allows for consistent logging across the application.

### 2. Logger Initialization

```typescript
const logger = createLogger('Utils')
```

*   `const logger = createLogger('Utils')`: Initializes a logger instance specifically for this `Utils` file. This means any logs emitted from functions within this file will be tagged with "Utils," making it easier to trace log messages back to their source during debugging.

### 3. `cn` Function (CSS Class Merging)

This function combines `clsx` and `twMerge` to provide a robust utility for managing Tailwind CSS class names in React or similar UI frameworks.

```typescript
export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs))
}
```

*   `export function cn(...inputs: ClassValue[])`: Defines an exported function `cn` (short for "class names"). It accepts a rest parameter `inputs`, which is an array of `ClassValue` types. `ClassValue` (from `clsx`) is a flexible type that can represent strings, objects (for conditional classes), or arrays of other `ClassValue`s.
*   `return twMerge(clsx(inputs))`: This line does two main things:
    1.  `clsx(inputs)`: It first calls `clsx` with all the provided `inputs`. `clsx` processes these inputs, resolving any conditional class names (e.g., `{ 'bg-red-500': true }` becomes `bg-red-500`) and joining them into a single, space-separated string (e.g., `"flex p-4 bg-blue-500"`).
    2.  `twMerge(...)`: The resulting string from `clsx` is then passed to `twMerge`. `twMerge` intelligently analyzes this string, identifying and resolving any conflicting Tailwind CSS classes. For example, if `clsx` produced `"p-4 px-6"`, `twMerge` would understand that `px-6` (horizontal padding) overrides `p-4` (all-side padding) for the horizontal axis, resulting in `"p-4 px-6"` or simply `"px-6"` depending on its internal logic. This prevents unexpected styling and keeps your class lists clean.

### 4. `getEncryptionKey` Function (Private Key Retrieval)

This internal helper function retrieves and validates the application's encryption key from environment variables.

```typescript
function getEncryptionKey(): Buffer {
  const key = env.ENCRYPTION_KEY
  if (!key || key.length !== 64) {
    throw new Error('ENCRYPTION_KEY must be set to a 64-character hex string (32 bytes)')
  }
  return Buffer.from(key, 'hex')
}
```

*   `function getEncryptionKey(): Buffer`: Declares a private (not exported) function that returns a `Buffer` object, which is how Node.js's `crypto` module expects cryptographic keys.
*   `const key = env.ENCRYPTION_KEY`: Accesses the `ENCRYPTION_KEY` from the `env` object. This key is crucial for all encryption and decryption operations.
*   `if (!key || key.length !== 64)`: Performs a crucial validation step. For AES-256 (256-bit encryption), the key must be 32 bytes long. When represented as a hexadecimal string, 32 bytes translate to 64 characters (since each byte is two hex digits). This check ensures the key is present and has the correct length.
*   `throw new Error(...)`: If the key is missing or invalid, an error is thrown with a clear message, preventing the application from proceeding with insecure or impossible encryption/decryption.
*   `return Buffer.from(key, 'hex')`: Converts the hexadecimal string `key` into a `Buffer` object. The `crypto` module requires keys to be in `Buffer` format.

### 5. `encryptSecret` Function (AES-256-GCM Encryption)

This function encrypts a given string using the AES-256-GCM algorithm, providing both confidentiality and integrity.

**Simplified Logic**: Imagine you have a secret message. This function acts like a super-secure digital lockbox.
1.  It gets a unique, one-time "code" (the IV) to make sure every time you encrypt the same message, the result looks different.
2.  It uses a powerful master "key" (the encryption key) to scramble your message.
3.  It also generates a "tamper-proof seal" (the authentication tag). If anyone tries to change even a tiny part of your scrambled message or the seal, the lockbox won't open correctly, and you'll know it was tampered with.
4.  Finally, it gives you the scrambled message, the one-time code, and the tamper-proof seal, all packaged together so you can open it later.

```typescript
/**
 * Encrypts a secret using AES-256-GCM
 * @param secret - The secret to encrypt
 * @returns A promise that resolves to an object containing the encrypted secret and IV
 */
export async function encryptSecret(secret: string): Promise<{ encrypted: string; iv: string }> {
  const iv = randomBytes(16)
  const key = getEncryptionKey()

  const cipher = createCipheriv('aes-256-gcm', key, iv)
  let encrypted = cipher.update(secret, 'utf8', 'hex')
  encrypted += cipher.final('hex')

  const authTag = cipher.getAuthTag()

  // Format: iv:encrypted:authTag
  return {
    encrypted: `${iv.toString('hex')}:${encrypted}:${authTag.toString('hex')}`,
    iv: iv.toString('hex'),
  }
}
```

*   `export async function encryptSecret(secret: string): Promise<{ encrypted: string; iv: string }>`: Defines an exported asynchronous function named `encryptSecret`. It takes a `secret` string to encrypt and returns a `Promise` that resolves to an object containing the encrypted string and the Initialization Vector (IV) string.
*   `const iv = randomBytes(16)`: Generates 16 cryptographically strong random bytes. This is the **Initialization Vector (IV)**. The IV must be unique for each encryption operation with the same key to ensure security (preventing identical plaintexts from yielding identical ciphertexts). For `aes-256-gcm` in Node.js, a 16-byte IV is typically expected.
*   `const key = getEncryptionKey()`: Retrieves the application's secret encryption key using the helper function.
*   `const cipher = createCipheriv('aes-256-gcm', key, iv)`: Creates a `Cipher` object.
    *   `'aes-256-gcm'`: Specifies the encryption algorithm: AES (Advanced Encryption Standard) with a 256-bit key in GCM (Galois/Counter Mode). GCM provides authenticated encryption, meaning it ensures both data confidentiality (the secret is unreadable) and data integrity/authenticity (the secret hasn't been tampered with).
    *   `key`: The 32-byte encryption key.
    *   `iv`: The 16-byte unique Initialization Vector.
*   `let encrypted = cipher.update(secret, 'utf8', 'hex')`: Encrypts the main part of the `secret`.
    *   `secret`: The plaintext string to be encrypted.
    *   `'utf8'`: Specifies that the input `secret` is in UTF-8 encoding.
    *   `'hex'`: Specifies that the output `encrypted` data should be represented as a hexadecimal string.
*   `encrypted += cipher.final('hex')`: Finalizes the encryption process. This method processes any remaining buffered data and ensures all data is encrypted. It also outputs the last part in hexadecimal.
*   `const authTag = cipher.getAuthTag()`: Retrieves the **authentication tag**. In GCM mode, this tag is generated during encryption and is essential for verifying data integrity during decryption. If the encrypted data or the tag is altered, decryption will fail, alerting to tampering.
*   `return { encrypted: `${iv.toString('hex')}:${encrypted}:${authTag.toString('hex')}`, iv: iv.toString('hex'), }`: Returns an object containing:
    *   `encrypted`: A string that combines the hexadecimal IV, the hexadecimal encrypted data, and the hexadecimal authentication tag, separated by colons. This single string conveniently packages all necessary components for later decryption.
    *   `iv`: The IV, also in hexadecimal format. While redundant (as it's part of the `encrypted` string), it's provided separately for explicit access if needed.

### 6. `decryptSecret` Function (AES-256-GCM Decryption)

This function decrypts an encrypted secret and verifies its integrity using the provided authentication tag.

**Simplified Logic**: Using the lockbox analogy:
1.  You get the combined package (scrambled message, one-time code, tamper-proof seal).
2.  You carefully separate the one-time code, the scrambled message, and the seal.
3.  You use the same master "key" as before, along with the one-time code, to try and open the lockbox.
4.  Crucially, as you open it, you present the "tamper-proof seal." If the seal doesn't match the one the lockbox expects (which it computes from the unscrambled message and other data), the decryption fails, and you know the message was compromised.
5.  If the seal is good, you get your original, unscrambled message back.

```typescript
/**
 * Decrypts an encrypted secret
 * @param encryptedValue - The encrypted value in format "iv:encrypted:authTag"
 * @returns A promise that resolves to an object containing the decrypted secret
 */
export async function decryptSecret(encryptedValue: string): Promise<{ decrypted: string }> {
  const parts = encryptedValue.split(':')
  const ivHex = parts[0]
  const authTagHex = parts[parts.length - 1]
  const encrypted = parts.slice(1, -1).join(':') // Handle potential colons in encrypted data

  if (!ivHex || !encrypted || !authTagHex) {
    throw new Error('Invalid encrypted value format. Expected "iv:encrypted:authTag"')
  }

  const key = getEncryptionKey()
  const iv = Buffer.from(ivHex, 'hex')
  const authTag = Buffer.from(authTagHex, 'hex')

  try {
    const decipher = createDecipheriv('aes-256-gcm', key, iv)
    decipher.setAuthTag(authTag) // Set the authentication tag for verification

    let decrypted = decipher.update(encrypted, 'hex', 'utf8')
    decrypted += decipher.final('utf8') // Finalize decryption and verify auth tag

    return { decrypted }
  } catch (error: any) {
    logger.error('Decryption error:', { error: error.message })
    throw error // Re-throw the error for upstream handling
  }
}
```

*   `export async function decryptSecret(encryptedValue: string): Promise<{ decrypted: string }>`: Defines an exported asynchronous function `decryptSecret`. It takes `encryptedValue` (expected in the `iv:encrypted:authTag` format) and returns a `Promise` resolving to an object with the `decrypted` string.
*   `const parts = encryptedValue.split(':')`: Splits the input string by the colon delimiter to separate its three components.
*   `const ivHex = parts[0]`: Extracts the hexadecimal IV string.
*   `const authTagHex = parts[parts.length - 1]`: Extracts the hexadecimal authentication tag string.
*   `const encrypted = parts.slice(1, -1).join(':')`: This is a careful way to extract the encrypted data. It takes all parts *between* the first (IV) and the last (auth tag) and joins them back with colons. This handles cases where the *actual encrypted data itself* might contain colon characters, preventing incorrect parsing.
*   `if (!ivHex || !encrypted || !authTagHex)`: Validates that all three crucial components were successfully extracted. If any are missing, the format is considered invalid.
*   `throw new Error(...)`: Throws an error if the format is incorrect.
*   `const key = getEncryptionKey()`: Retrieves the encryption key, which is also needed for decryption.
*   `const iv = Buffer.from(ivHex, 'hex')`: Converts the hexadecimal IV string back into a `Buffer`.
*   `const authTag = Buffer.from(authTagHex, 'hex')`: Converts the hexadecimal authentication tag string back into a `Buffer`.
*   `try { ... } catch (error: any) { ... }`: The decryption process is wrapped in a `try...catch` block. This is critical because decryption can fail if:
    *   The encryption key is incorrect.
    *   The IV is incorrect.
    *   The encrypted data has been tampered with.
    *   The authentication tag does not match (indicating tampering).
*   `const decipher = createDecipheriv('aes-256-gcm', key, iv)`: Creates a `Decipher` object using the same algorithm, key, and IV that were used for encryption.
*   `decipher.setAuthTag(authTag)`: This is the crucial step for integrity verification. The decipher object is provided with the expected authentication tag. During the `final()` step, the decipher will compute its own authentication tag from the decrypted data and compare it against this `authTag`. If they don't match, an error will be thrown.
*   `let decrypted = decipher.update(encrypted, 'hex', 'utf8')`: Decrypts the main part of the encrypted data. It expects hexadecimal input and converts it back to a UTF-8 string.
*   `decrypted += decipher.final('utf8')`: Finalizes the decryption process. This step is where the authentication tag is verified. If the tag is invalid, an error will be thrown here. The remaining decrypted data is appended.
*   `return { decrypted }`: If decryption is successful and the authentication tag matches, the original `decrypted` string is returned.
*   `logger.error('Decryption error:', { error: error.message })`: If an error occurs during decryption (e.g., due to tampering or incorrect key/IV), it's logged for debugging.
*   `throw error`: The error is re-thrown, allowing upstream calling code to handle the decryption failure appropriately.

### 7. `convertScheduleOptionsToCron` Function (Cron Expression Generation)

This function translates various human-readable schedule types and options into standard Cron expressions, which are used to define recurring tasks.

**Simplified Logic**: Cron expressions are a compact, standardized way to tell a system "when" to do something (e.g., "every 15 minutes," "at 9 AM on Mondays"). This function acts as a translator, taking user-friendly descriptions like "daily at 9:00 AM" and turning them into the specific Cron syntax.

```typescript
export function convertScheduleOptionsToCron(
  scheduleType: string,
  options: Record<string, string>
): string {
  switch (scheduleType) {
    case 'minutes': {
      const interval = options.minutesInterval || '15'
      // For example, if options.minutesStartingAt is provided, use that as the start minute.
      return `*/${interval} * * * *` // E.g., "*/15 * * * *" for every 15 minutes
    }
    case 'hourly': {
      // When scheduling hourly, take the specified minute offset
      return `${options.hourlyMinute || '00'} * * * *` // E.g., "00 * * * *" for every hour at minute 00
    }
    case 'daily': {
      // Expected dailyTime in HH:MM
      const [minute, hour] = (options.dailyTime || '00:09').split(':')
      return `${minute || '00'} ${hour || '09'} * * *` // E.g., "00 09 * * *" for every day at 09:00
    }
    case 'weekly': {
      // Expected weeklyDay as MON, TUE, etc. and weeklyDayTime in HH:MM
      const dayMap: Record<string, number> = {
        MON: 1,
        TUE: 2,
        WED: 3,
        THU: 4,
        FRI: 5,
        SAT: 6,
        SUN: 0, // Sunday is 0 or 7 in Cron
      }
      const day = dayMap[options.weeklyDay || 'MON']
      const [minute, hour] = (options.weeklyDayTime || '00:09').split(':')
      return `${minute || '00'} ${hour || '09'} * * ${day}` // E.g., "00 09 * * 1" for every Monday at 09:00
    }
    case 'monthly': {
      // Expected monthlyDay and monthlyTime in HH:MM
      const day = options.monthlyDay || '1'
      const [minute, hour] = (options.monthlyTime || '00:09').split(':')
      return `${minute || '00'} ${hour || '09'} ${day} * *` // E.g., "00 09 1 * *" for the 1st of every month at 09:00
    }
    case 'custom': {
      // Use the provided cron expression directly
      return options.cronExpression
    }
    default:
      throw new Error('Unsupported schedule type')
  }
}
```

*   `export function convertScheduleOptionsToCron(scheduleType: string, options: Record<string, string>): string`: Defines an exported function that takes `scheduleType` (a string describing the type of schedule) and `options` (an object containing detailed parameters for that schedule type) and returns a Cron expression string.
*   The function uses a `switch` statement based on `scheduleType` to handle different scheduling patterns.
*   **Cron Expression Format**: A Cron string typically has five fields: `minute hour day-of-month month day-of-week`.
    *   `*`: A wildcard, meaning "every" or "any."
    *   `*/N`: Means "every N units" (e.g., `*/15` in the minute field means "every 15 minutes").
*   **Case `minutes`**:
    *   `const interval = options.minutesInterval || '15'`: Retrieves the desired minute interval from `options`, defaulting to `15` if not provided.
    *   `return `*/${interval} * * * *``: Generates a Cron string that runs every `interval` minutes.
*   **Case `hourly`**:
    *   `return `${options.hourlyMinute || '00'} * * * *``: Generates a Cron string that runs at a specific minute past every hour (e.g., "00 * * * *" for the top of every hour).
*   **Case `daily`**:
    *   `const [minute, hour] = (options.dailyTime || '00:09').split(':')`: Extracts the minute and hour from a `dailyTime` string (expected format "HH:MM"), defaulting to "00:09" (9 AM) if not provided.
    *   `return `${minute || '00'} ${hour || '09'} * * *``: Generates a Cron string that runs daily at the specified time.
*   **Case `weekly`**:
    *   `const dayMap`: A mapping from common three-letter day abbreviations (MON, TUE, etc.) to their numerical representation in Cron (Sunday is 0, Monday is 1, ..., Saturday is 6).
    *   `const day = dayMap[options.weeklyDay || 'MON']`: Retrieves the numerical day of the week, defaulting to Monday (1).
    *   `return `${minute || '00'} ${hour || '09'} * * ${day}``: Generates a Cron string that runs weekly on the specified day and time.
*   **Case `monthly`**:
    *   `const day = options.monthlyDay || '1'`: Retrieves the day of the month, defaulting to the 1st.
    *   `return `${minute || '00'} ${hour || '09'} ${day} * *``: Generates a Cron string that runs monthly on the specified day and time.
*   **Case `custom`**:
    *   `return options.cronExpression`: Directly uses a full Cron expression provided in the `options`.
*   **`default`**:
    *   `throw new Error('Unsupported schedule type')`: If an unrecognized `scheduleType` is provided, an error is thrown.

### 8. `getTimezoneAbbreviation` Function (Timezone Abbreviation)

This function provides a user-friendly, abbreviated name for a given IANA timezone string, correctly accounting for Daylight Saving Time (DST).

**Simplified Logic**: You give it a full timezone name (like "America/Los_Angeles") and a date. It checks if there's a common short name (like "PST" or "PDT"). If there is, it then cleverly figures out if Daylight Saving Time is active on your given date for that timezone, so it can return the *correct* abbreviation (e.g., "PDT" if DST is active, "PST" otherwise).

```typescript
/**
 * Get a user-friendly timezone abbreviation
 * @param timezone - IANA timezone string
 * @param date - Date to check for DST
 * @returns A simplified timezone string (e.g., "PST" instead of "America/Los_Angeles")
 */
export function getTimezoneAbbreviation(timezone: string, date: Date = new Date()): string {
  if (timezone === 'UTC') return 'UTC'

  // Common timezone mappings
  const timezoneMap: Record<string, { standard: string; daylight: string }> = {
    'America/Los_Angeles': { standard: 'PST', daylight: 'PDT' },
    'America/Denver': { standard: 'MST', daylight: 'MDT' },
    'America/Chicago': { standard: 'CST', daylight: 'CDT' },
    'America/New_York': { standard: 'EST', daylight: 'EDT' },
    'Europe/London': { standard: 'GMT', daylight: 'BST' },
    'Europe/Paris': { standard: 'CET', daylight: 'CEST' },
    'Asia/Tokyo': { standard: 'JST', daylight: 'JST' }, // Japan doesn't use DST
    'Australia/Sydney': { standard: 'AEST', daylight: 'AEDT' },
    'Asia/Singapore': { standard: 'SGT', daylight: 'SGT' }, // Singapore doesn't use DST
  }

  // If we have a mapping for this timezone
  if (timezone in timezoneMap) {
    // January 1 is guaranteed to be standard time in northern hemisphere
    // July 1 is guaranteed to be daylight time in northern hemisphere (if observed)
    const januaryDate = new Date(date.getFullYear(), 0, 1) // Month is 0-indexed for Jan
    const julyDate = new Date(date.getFullYear(), 6, 1)    // Month is 0-indexed for Jul

    // Get offset in January (standard time)
    const januaryFormatter = new Intl.DateTimeFormat('en-US', {
      timeZone: timezone,
      timeZoneName: 'short', // Request the short timezone name (e.g., PST)
    })

    // Get offset in July (likely daylight time, if observed)
    const julyFormatter = new Intl.DateTimeFormat('en-US', {
      timeZone: timezone,
      timeZoneName: 'short',
    })

    // If offsets are different, timezone observes DST
    const isDSTObserved = januaryFormatter.format(januaryDate) !== julyFormatter.format(julyDate)

    // If DST is observed, check if current date is in DST by comparing its offset
    // with January's offset (standard time)
    if (isDSTObserved) {
      const currentFormatter = new Intl.DateTimeFormat('en-US', {
        timeZone: timezone,
        timeZoneName: 'short',
      })

      // Compare current date's timezone name with the standard (January) one
      const isDST = currentFormatter.format(date) !== januaryFormatter.format(januaryDate)
      return isDST ? timezoneMap[timezone].daylight : timezoneMap[timezone].standard
    }

    // If DST is not observed, always use standard abbreviation
    return timezoneMap[timezone].standard
  }

  // For unknown timezones, use full IANA name as a fallback
  return timezone
}
```

*   `export function getTimezoneAbbreviation(timezone: string, date: Date = new Date()): string`: Defines an exported function. It takes an IANA `timezone` string (e.g., "America/Los_Angeles") and an optional `date` object (defaulting to the current date) to determine if DST is in effect. It returns a string abbreviation.
*   `if (timezone === 'UTC') return 'UTC'`: Handles the special case of Coordinated Universal Time.
*   `const timezoneMap`: An object mapping specific IANA timezone strings to custom `standard` and `daylight` abbreviations. This is a curated list for common timezones.
*   `if (timezone in timezoneMap)`: Checks if the provided `timezone` has a predefined mapping in `timezoneMap`.
*   `const januaryDate = new Date(date.getFullYear(), 0, 1)`: Creates a `Date` object for January 1st of the current year. January 1st is typically in standard time for most Northern Hemisphere timezones.
*   `const julyDate = new Date(date.getFullYear(), 6, 1)`: Creates a `Date` object for July 1st of the current year. July 1st is typically in daylight saving time for most Northern Hemisphere timezones that observe it.
*   `const januaryFormatter = new Intl.DateTimeFormat('en-US', { timeZone: timezone, timeZoneName: 'short' })`: Creates an `Intl.DateTimeFormat` object, configured for the target `timezone` and requesting the `short` timezone name (e.g., "PST").
*   `const julyFormatter = new Intl.DateTimeFormat('en-US', { timeZone: timezone, timeZoneName: 'short' })`: Similar formatter for July.
*   `const isDSTObserved = januaryFormatter.format(januaryDate) !== julyFormatter.format(julyDate)`: This is a clever trick to detect if the timezone observes DST at all. If the short timezone name for January 1st is different from July 1st (e.g., "PST" vs. "PDT"), it implies that DST is observed.
*   `if (isDSTObserved)`: If the timezone observes DST:
    *   `const currentFormatter = new Intl.DateTimeFormat('en-US', { timeZone: timezone, timeZoneName: 'short' })`: Creates a formatter for the *actual* `date` passed to the function.
    *   `const isDST = currentFormatter.format(date) !== januaryFormatter.format(januaryDate)`: Compares the short timezone name for the *current `date`* with the *standard time* (January 1st) abbreviation. If they differ, then the `current date` falls within DST.
    *   `return isDST ? timezoneMap[timezone].daylight : timezoneMap[timezone].standard`: Returns the appropriate abbreviation based on whether `isDST` is true or false.
*   `return timezoneMap[timezone].standard`: If `isDSTObserved` is false (meaning the timezone doesn't observe DST, like JST or SGT), it always returns the standard abbreviation.
*   `return timezone`: If the provided `timezone` is not found in the `timezoneMap`, the original IANA timezone string is returned as a fallback.

### 9. `formatDateTime`, `formatDate`, `formatTime` Functions (Date & Time Formatting)

These functions provide different human-readable formatting options for `Date` objects, leveraging the browser's internationalization API (`toLocaleString`).

```typescript
/**
 * Format a date into a human-readable format
 * @param date - The date to format
 * @param timezone - Optional IANA timezone string (e.g., 'America/Los_Angeles', 'UTC')
 * @returns A formatted date string in the format "MMM D, YYYY h:mm A"
 */
export function formatDateTime(date: Date, timezone?: string): string {
  const formattedDate = date.toLocaleString('en-US', {
    month: 'short',    // e.g., "Jan"
    day: 'numeric',    // e.g., "1"
    year: 'numeric',   // e.g., "2023"
    hour: 'numeric',   // e.g., "1"
    minute: '2-digit', // e.g., "05"
    hour12: true,      // Use 12-hour format with AM/PM
    timeZone: timezone || undefined, // Apply timezone if provided
  })

  // If timezone is provided, add a friendly timezone abbreviation
  if (timezone) {
    const tzAbbr = getTimezoneAbbreviation(timezone, date)
    return `${formattedDate} ${tzAbbr}`
  }

  return formattedDate
}

/**
 * Format a date into a short format
 * @param date - The date to format
 * @returns A formatted date string in the format "MMM D, YYYY"
 */
export function formatDate(date: Date): string {
  return date.toLocaleString('en-US', {
    month: 'short',
    day: 'numeric',
    year: 'numeric',
  })
}

/**
 * Format a time into a short format
 * @param date - The date to format
 * @returns A formatted time string in the format "h:mm A"
 */
export function formatTime(date: Date): string {
  return date.toLocaleString('en-US', {
    hour: 'numeric',
    minute: '2-digit',
    hour12: true,
  })
}
```

*   **`formatDateTime`**:
    *   `export function formatDateTime(date: Date, timezone?: string): string`: Takes a `Date` object and an optional `timezone` string.
    *   `date.toLocaleString('en-US', { ... })`: Formats the `date` using the `en-US` locale and a set of options:
        *   `month: 'short'`: Short month name (e.g., "Jan").
        *   `day: 'numeric'`: Day of the month (e.g., "1").
        *   `year: 'numeric'`: Full year (e.g., "2023").
        *   `hour: 'numeric'`, `minute: '2-digit'`, `hour12: true`: Formats the time as "h:mm AM/PM" (e.g., "1:05 PM").
        *   `timeZone: timezone || undefined`: Applies the specified `timezone` if provided; otherwise, it uses the local timezone of the environment where the code runs.
    *   `if (timezone) { ... }`: If a `timezone` was provided, it calls `getTimezoneAbbreviation` to get the friendly short name (e.g., "PST" or "PDT") and appends it to the formatted date string.
*   **`formatDate`**:
    *   `export function formatDate(date: Date): string`: Takes a `Date` object.
    *   `return date.toLocaleString('en-US', { month: 'short', day: 'numeric', year: 'numeric' })`: Formats the date as "MMM D, YYYY" (e.g., "Jan 1, 2023").
*   **`formatTime`**:
    *   `export function formatTime(date: Date): string`: Takes a `Date` object.
    *   `return date.toLocaleString('en-US', { hour: 'numeric', minute: '2-digit', hour12: true })`: Formats the time as "h:mm A" (e.g., "1:05 PM").

### 10. `formatDuration` Function (Duration Formatting)

This function converts a duration expressed in milliseconds into a human-readable string (e.g., "150ms", "10s", "2m 30s", "1h 5m").

```typescript
/**
 * Format a duration in milliseconds to a human-readable format
 * @param durationMs - The duration in milliseconds
 * @returns A formatted duration string
 */
export function formatDuration(durationMs: number): string {
  if (durationMs < 1000) {
    return `${durationMs}ms` // Less than 1 second, display in milliseconds
  }

  const seconds = Math.floor(durationMs / 1000)
  if (seconds < 60) {
    return `${seconds}s` // Less than 1 minute, display in seconds
  }

  const minutes = Math.floor(seconds / 60)
  const remainingSeconds = seconds % 60
  if (minutes < 60) {
    // Less than 1 hour, display in minutes and seconds
    return `${minutes}m ${remainingSeconds}s`
  }

  const hours = Math.floor(minutes / 60)
  const remainingMinutes = minutes % 60
  // 1 hour or more, display in hours and minutes
  return `${hours}h ${remainingMinutes}m`
}
```

*   `export function formatDuration(durationMs: number): string`: Defines an exported function that takes a duration in `durationMs` (milliseconds) and returns a formatted string.
*   `if (durationMs < 1000)`: If the duration is less than 1 second, it's displayed directly in milliseconds (e.g., "150ms").
*   `const seconds = Math.floor(durationMs / 1000)`: Converts milliseconds to whole seconds.
*   `if (seconds < 60)`: If the duration is less than 1 minute, it's displayed in seconds (e.g., "10s").
*   `const minutes = Math.floor(seconds / 60)`, `const remainingSeconds = seconds % 60`: Calculates the number of whole minutes and any remaining seconds.
*   `if (minutes < 60)`: If the duration is less than 1 hour, it's displayed in minutes and seconds (e.g., "2m 30s").
*   `const hours = Math.floor(minutes / 60)`, `const remainingMinutes = minutes % 60`: Calculates the number of whole hours and any remaining minutes.
*   `return `${hours}h ${remainingMinutes}m``: For durations of an hour or more, it's displayed in hours and minutes (e.g., "1h 5m").

### 11. `generatePassword` Function (Password Generation)

This function generates a random password of a specified length using a predefined set of characters.

```typescript
/**
 * Generates a secure random password
 * @param length - The length of the password (default: 24)
 * @returns A new secure password string
 */
export function generatePassword(length = 24): string {
  const chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789!@#$%^&*()_-+='
  let result = ''

  for (let i = 0; i < length; i++) {
    result += chars.charAt(Math.floor(Math.random() * chars.length))
  }

  return result
}
```

*   `export function generatePassword(length = 24): string`: Defines an exported function that generates a password. It takes an optional `length` parameter (defaulting to 24) and returns the generated password as a string.
*   `const chars = '...'`: Defines a string containing all possible characters for the password, including uppercase letters, lowercase letters, digits, and various symbols.
*   `let result = ''`: Initializes an empty string to build the password.
*   `for (let i = 0; i < length; i++)`: A loop that runs `length` times, once for each character of the password.
*   `result += chars.charAt(Math.floor(Math.random() * chars.length))`: In each iteration:
    *   `Math.random() * chars.length`: Generates a random floating-point number between 0 (inclusive) and the total number of available characters (exclusive).
    *   `Math.floor(...)`: Rounds this number down to the nearest whole integer, giving a valid index within the `chars` string.
    *   `chars.charAt(...)`: Retrieves the character at that random index.
    *   `result += ...`: Appends the chosen character to the `result` string.
*   `return result`: Returns the fully constructed random password.
    *   **Note**: While `Math.random()` provides randomness, for truly *cryptographically secure* passwords where the highest level of unpredictability is required, using `crypto.randomBytes` to generate random numbers and then mapping them to characters would be preferred, similar to how the `iv` is generated in `encryptSecret`.

### 12. `getRotatingApiKey` Function (API Key Rotation)

This function provides a simple mechanism to rotate through multiple API keys for a given provider, helping to distribute load or mitigate rate limits.

**Simplified Logic**: If you have a few API keys for a service (e.g., OpenAI Key 1, Key 2, Key 3), this function helps you pick a different one each minute. It uses the current minute to decide which key to return in a round-robin fashion, ensuring that requests are spread out across your available keys without needing complex state management.

```typescript
/**
 * Rotates through available API keys for a provider
 * @param provider - The provider to get a key for (e.g., 'openai')
 * @returns The selected API key
 * @throws Error if no API keys are configured for rotation
 */
export function getRotatingApiKey(provider: string): string {
  if (provider !== 'openai' && provider !== 'anthropic') {
    throw new Error(`No rotation implemented for provider: ${provider}`)
  }

  const keys = []

  if (provider === 'openai') {
    if (env.OPENAI_API_KEY_1) keys.push(env.OPENAI_API_KEY_1)
    if (env.OPENAI_API_KEY_2) keys.push(env.OPENAI_API_KEY_2)
    if (env.OPENAI_API_KEY_3) keys.push(env.OPENAI_API_KEY_3)
  } else if (provider === 'anthropic') {
    if (env.ANTHROPIC_API_KEY_1) keys.push(env.ANTHROPIC_API_KEY_1)
    if (env.ANTHROPIC_API_KEY_2) keys.push(env.ANTHROPIC_API_KEY_2)
    if (env.ANTHROPIC_API_KEY_3) keys.push(env.ANTHROPIC_API_KEY_3)
  }

  if (keys.length === 0) {
    throw new Error(
      `No API keys configured for rotation. Please configure ${provider.toUpperCase()}_API_KEY_1, ${provider.toUpperCase()}_API_KEY_2, or ${provider.toUpperCase()}_API_KEY_3.`
    )
  }

  // Simple round-robin rotation based on current minute
  // This distributes load across keys and is stateless
  const currentMinute = new Date().getMinutes()
  const keyIndex = currentMinute % keys.length // Use modulo to cycle through keys

  return keys[keyIndex]
}
```

*   `export function getRotatingApiKey(provider: string): string`: Defines an exported function that takes a `provider` string (e.g., "openai", "anthropic") and returns a chosen API key string.
*   `if (provider !== 'openai' && provider !== 'anthropic')`: Checks if the provided `provider` is one for which key rotation logic is implemented. If not, it throws an error.
*   `const keys = []`: Initializes an empty array to store the available API keys for the given provider.
*   `if (provider === 'openai') { ... } else if (provider === 'anthropic') { ... }`: Conditionally checks for environment variables for each provider (e.g., `OPENAI_API_KEY_1`, `OPENAI_API_KEY_2`, `OPENAI_API_KEY_3`) and pushes any found keys into the `keys` array. This allows for up to three keys per provider.
*   `if (keys.length === 0)`: After checking environment variables, if no keys were found for the provider, it means rotation cannot occur.
*   `throw new Error(...)`: An error is thrown, instructing the user to configure the necessary environment variables.
*   `const currentMinute = new Date().getMinutes()`: Gets the current minute (0-59) from the system clock.
*   `const keyIndex = currentMinute % keys.length`: This is the core of the rotation logic. The modulo operator (`%`) calculates the remainder when `currentMinute` is divided by the number of `keys`. This ensures that `keyIndex` will always be a valid index within the `keys` array (0, 1, ..., `keys.length - 1`). As the minute changes, the `keyIndex` will cycle through the available keys in a round-robin fashion.
*   `return keys[keyIndex]`: Returns the API key located at the calculated `keyIndex`.

### 13. `redactApiKeys` Function (Security Redaction)

This function recursively traverses an object and redacts (replaces with a placeholder) the values of keys that are identified as sensitive (e.g., API keys, secrets, passwords).

**Simplified Logic**: This function is like a censor for your data. You give it any object, and it goes through every part of it. If it finds a field named "apiKey," "password," or anything similar, it automatically replaces the actual value with `***REDACTED***` to prevent sensitive information from accidentally being exposed in logs or displays. It does this even if the sensitive data is inside other objects or arrays.

```typescript
/**
 * Recursively redacts API keys in an object
 * @param obj The object to redact API keys from
 * @returns A new object with API keys redacted
 */
export const redactApiKeys = (obj: any): any => {
  if (!obj || typeof obj !== 'object') {
    return obj // Base case: if not an object or null, return as is
  }

  if (Array.isArray(obj)) {
    return obj.map(redactApiKeys) // If it's an array, recursively redact each element
  }

  const result: Record<string, any> = {}

  for (const [key, value] of Object.entries(obj)) {
    // Check if the key name suggests sensitive information (case-insensitive)
    if (
      key.toLowerCase() === 'apikey' ||
      key.toLowerCase() === 'api_key' ||
      key.toLowerCase() === 'access_token' ||
      /\bsecret\b/i.test(key.toLowerCase()) || // Uses regex for 'secret' as a word boundary
      /\bpassword\b/i.test(key.toLowerCase())  // Uses regex for 'password' as a word boundary
    ) {
      result[key] = '***REDACTED***' // Redact the value
    } else if (typeof value === 'object' && value !== null) {
      result[key] = redactApiKeys(value) // Recursively redact nested objects
    } else {
      result[key] = value // Copy non-sensitive, non-object values as is
    }
  }

  return result // Return the new, redacted object
}
```

*   `export const redactApiKeys = (obj: any): any => { ... }`: Defines an exported arrow function `redactApiKeys` that takes any `obj` and returns a potentially modified `any` type (a new object with redactions).
*   `if (!obj || typeof obj !== 'object') { return obj }`: This is the base case for the recursion. If the input `obj` is `null`, `undefined`, or not an object (e.g., a string, number, boolean), it's returned as is because there's nothing to redact.
*   `if (Array.isArray(obj)) { return obj.map(redactApiKeys) }`: If the `obj` is an array, it uses `map` to create a new array where each element is recursively processed by `redactApiKeys`.
*   `const result: Record<string, any> = {}`: Initializes an empty object that will hold the redacted (or copied) properties. This ensures the original object is not mutated.
*   `for (const [key, value] of Object.entries(obj))`: Iterates over each key-value pair in the input `obj`.
*   `if (key.toLowerCase() === 'apikey' || ...)`: This `if` condition checks if the current `key` (converted to lowercase for case-insensitivity) matches any of the predefined sensitive key names: `apikey`, `api_key`, `access_token`. It also uses regular expressions (`/\bsecret\b/i` and `/\bpassword\b/i`) to detect if the key contains the words "secret" or "password" with word boundaries (`\b`) ensuring it matches whole words, not just substrings (e.g., "supersecret" would match, but "secretive" would not for `secret`).
*   `result[key] = '***REDACTED***'`: If the key is identified as sensitive, its value in the `result` object is replaced with the `***REDACTED***` placeholder string.
*   `else if (typeof value === 'object' && value !== null)`: If the `value` of the current key is itself an object (and not `null`), it means there might be nested sensitive data.
*   `result[key] = redactApiKeys(value)`: In this case, `redactApiKeys` is called recursively on the nested object, and its redacted result is assigned to the `key` in `result`.
*   `else { result[key] = value }`: If the value is not sensitive and not an object (e.g., a string, number), it's copied directly to the `result` object.
*   `return result`: After iterating through all properties, the newly created and redacted `result` object is returned.

### 14. `validateName`, `isValidName`, `getInvalidCharacters` Functions (Name Validation)

These functions provide utilities for sanitizing and validating names or identifiers, often used for user inputs or variable names to prevent issues.

```typescript
/**
 * Validates a name by removing any characters that could cause issues
 * with variable references or node naming.
 *
 * @param name - The name to validate
 * @returns The validated name with invalid characters removed, trimmed, and collapsed whitespace
 */
export function validateName(name: string): string {
  return name
    .replace(/[^a-zA-Z0-9_\s]/g, '') // Remove invalid characters (anything not letter, number, underscore, or space)
    .replace(/\s+/g, ' ') // Collapse multiple spaces into single spaces
    .trim() // Remove leading/trailing whitespace
}

/**
 * Checks if a name contains invalid characters
 *
 * @param name - The name to check
 * @returns True if the name is valid, false otherwise
 */
export function isValidName(name: string): boolean {
  return /^[a-zA-Z0-9_\s]*$/.test(name) // Test if the name consists ONLY of valid characters
}

/**
 * Gets a list of invalid characters in a name
 *
 * @param name - The name to check
 * @returns Array of invalid characters found
 */
export function getInvalidCharacters(name: string): string[] {
  const invalidChars = name.match(/[^a-zA-Z0-9_\s]/g) // Find all characters NOT in the valid set
  return invalidChars ? [...new Set(invalidChars)] : [] // Return unique invalid characters, or an empty array
}
```

*   **`validateName`**:
    *   `export function validateName(name: string): string`: Takes a `name` string and returns a cleaned, validated string.
    *   `.replace(/[^a-zA-Z0-9_\s]/g, '')`: This regular expression finds any character that is *not* (`^` inside `[]`) an uppercase letter (`A-Z`), lowercase letter (`a-z`), digit (`0-9`), underscore (`_`), or whitespace character (`\s`). All such invalid characters are replaced with an empty string, effectively removing them.
    *   `.replace(/\s+/g, ' ')`: This regular expression finds one or more (`+`) whitespace characters (`\s`) and replaces them with a single space. This collapses multiple spaces into single spaces (e.g., "hello   world" becomes "hello world").
    *   `.trim()`: Removes any leading or trailing whitespace from the resulting string.
*   **`isValidName`**:
    *   `export function isValidName(name: string): boolean`: Takes a `name` string and returns `true` if it contains only valid characters, `false` otherwise.
    *   `return /^[a-zA-Z0-9_\s]*$/.test(name)`: This regular expression checks if the entire string (`^` to `$` anchors) consists *only* of the allowed characters (letters, numbers, underscore, space). The `*` quantifier means zero or more of these characters.
*   **`getInvalidCharacters`**:
    *   `export function getInvalidCharacters(name: string): string[]`: Takes a `name` string and returns an array of unique invalid characters found in it.
    *   `const invalidChars = name.match(/[^a-zA-Z0-9_\s]/g)`: This uses the same regex pattern as `validateName` to *find* all characters that are not allowed. The `match` method returns an array of all matches, or `null` if no matches are found.
    *   `return invalidChars ? [...new Set(invalidChars)] : []`: If `invalidChars` is not `null` (meaning invalid characters were found), it creates a `Set` from the array to get only unique invalid characters, then spreads it into a new array. If `invalidChars` is `null`, it returns an empty array.

### 15. `SSE_HEADERS` Constant (Server-Sent Events Headers)

This constant defines the standard HTTP headers required when implementing a Server-Sent Events (SSE) stream.

```typescript
export const SSE_HEADERS = {
  'Content-Type': 'text/event-stream',
  'Cache-Control': 'no-cache',
  Connection: 'keep-alive',
  'X-Accel-Buffering': 'no',
} as const
```

*   `export const SSE_HEADERS = { ... } as const`: Exports a constant object `SSE_HEADERS`. The `as const` assertion tells TypeScript that this object and all its properties are read-only and immutable.
*   `'Content-Type': 'text/event-stream'`: This is the most critical header for SSE. It tells the client (typically a browser) that the server is sending a stream of events, not a single response.
*   `'Cache-Control': 'no-cache'`: Instructs proxies and browsers not to cache the event stream, ensuring that clients always receive the latest events.
*   `Connection: 'keep-alive'`: Requests that the connection remains open after the initial response, allowing the server to continuously send new events.
*   `'X-Accel-Buffering': 'no'`: This header is specifically for use with reverse proxies like Nginx. It tells the proxy not to buffer the response, ensuring that events are sent to the client immediately as they are generated by the server, rather than being held back.

### 16. `encodeSSE` Function (SSE Encoding)

This function formats arbitrary data into a Server-Sent Events (SSE) message format and encodes it as a `Uint8Array` for efficient streaming.

**Simplified Logic**: When you want to send data over an SSE connection, it needs to be packaged in a very specific way: `data: [your JSON message]\n\n`. This function takes your raw data, converts it to JSON, adds that special `data:` prefix and `\n\n` suffix, and then turns that string into a sequence of bytes ready to be streamed.

```typescript
/**
 * Encodes data as a Server-Sent Events (SSE) message.
 * Formats the data as a JSON string prefixed with "data:" and suffixed with two newlines,
 * then encodes it as a Uint8Array for streaming.
 *
 * @param data - The data to encode and send via SSE
 * @returns The encoded SSE message as a Uint8Array
 */
export function encodeSSE(data: any): Uint8Array {
  return new TextEncoder().encode(`data: ${JSON.stringify(data)}\n\n`)
}
```

*   `export function encodeSSE(data: any): Uint8Array`: Defines an exported function that takes `data` of any type and returns a `Uint8Array` (an array of 8-bit unsigned integers, which is the standard way to represent raw byte data in JavaScript/TypeScript).
*   `new TextEncoder()`: Creates an instance of `TextEncoder`, a Web API interface that encodes a string into a `Uint8Array` using UTF-8 encoding.
*   `.encode(...)`: Calls the `encode` method on the `TextEncoder` instance with a template literal string:
    *   ``data: ${JSON.stringify(data)}\n\n``: This is the crucial SSE formatting.
        *   `JSON.stringify(data)`: Converts the input `data` (which can be an object, array, string, etc.) into its JSON string representation.
        *   `data: `: This prefix is a standard part of the SSE protocol, indicating that the following content is the event data.
        *   `\n\n`: Two newline characters at the end signify the termination of an SSE event message.

### 17. `generateRequestId` Function (Request ID Generation)

This function generates a short, unique identifier, typically used for correlating requests across different services or for logging.

```typescript
/**
 * Generate a short request ID for correlation
 */
export function generateRequestId(): string {
  return crypto.randomUUID().slice(0, 8)
}
```

*   `export function generateRequestId(): string`: Defines an exported function that returns a string representing a request ID.
*   `crypto.randomUUID()`: This function, available in modern JavaScript environments (Node.js, browsers), generates a cryptographically strong Universally Unique Identifier (UUID) conforming to RFC 4122 (e.g., "a1b2c3d4-e5f6-7a8b-9c0d-1e2f3a4b5c6d").
*   `.slice(0, 8)`: Takes only the first 8 characters of the generated UUID. This creates a shorter, more manageable ID that is still highly unique for practical purposes like request correlation and logging without being excessively long.

### 18. `noop` Constant (No-Operation Function)

This constant provides a simple function that performs no operation, useful as a placeholder or default callback.

```typescript
/**
 * No-operation function for use as default callback
 */
export const noop = () => {}
```

*   `export const noop = () => {}`: Defines an exported constant named `noop` (short for "no operation"). It's an arrow function that takes no arguments and has an empty function body, meaning it does absolutely nothing when called. This is commonly used as a default value for optional callback props or parameters to avoid `undefined` function calls.