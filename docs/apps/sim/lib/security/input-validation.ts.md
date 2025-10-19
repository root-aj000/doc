```typescript
import { createLogger } from '@/lib/logs/console/logger'

const logger = createLogger('InputValidation')
```

*   **Purpose:** This code imports a logger function and initializes a logger instance for use within the validation functions defined in this file.
*   **`import { createLogger } from '@/lib/logs/console/logger'`**: This line imports the `createLogger` function from a module located at `'@/lib/logs/console/logger'`.  It's assumed that this function is responsible for creating a logger instance that can be used to record messages (e.g., warnings, errors) during the validation process.  The `@/` likely refers to the project's root directory (using path aliases).
*   **`const logger = createLogger('InputValidation')`**: This line calls the `createLogger` function with the string `'InputValidation'` as an argument.  This string likely serves as a label or category for the logger, allowing you to easily filter or identify messages originating from this specific validation module.  The returned logger instance is then assigned to the `logger` constant. This `logger` object will be used to log suspicious or invalid input attempts.

```typescript
/**
 * Result type for validation functions
 */
export interface ValidationResult {
  isValid: boolean
  error?: string
  sanitized?: string
}
```

*   **Purpose:** This code defines a TypeScript interface named `ValidationResult`. This interface serves as a standard structure for the return value of all validation functions within the module.
*   **`export interface ValidationResult { ... }`**: This declares a TypeScript interface named `ValidationResult` and exports it, making it available for use in other modules. Interfaces define the shape of an object.
*   **`isValid: boolean`**: This property indicates whether the validation was successful.  If `true`, the input is considered valid; otherwise, it's considered invalid.
*   **`error?: string`**: This optional property holds an error message that provides more details about why the validation failed.  The `?` indicates that this property is optional; it will only be present if `isValid` is `false`.
*   **`sanitized?: string`**: This optional property holds a sanitized version of the input value.  Sanitization might involve removing potentially harmful characters or formatting the input in a specific way.  Like `error`, it's optional and may only be present when `isValid` is `true` and sanitization was performed.

```typescript
/**
 * Options for path segment validation
 */
export interface PathSegmentOptions {
  /** Name of the parameter for error messages */
  paramName?: string
  /** Maximum length allowed (default: 255) */
  maxLength?: number
  /** Allow hyphens (default: true) */
  allowHyphens?: boolean
  /** Allow underscores (default: true) */
  allowUnderscores?: boolean
  /** Allow dots (default: false, to prevent directory traversal) */
  allowDots?: boolean
  /** Custom regex pattern to match */
  customPattern?: RegExp
}
```

*   **Purpose:** This code defines a TypeScript interface named `PathSegmentOptions`. This interface is used to provide configuration options to the `validatePathSegment` function, allowing customization of the validation process.
*   **`export interface PathSegmentOptions { ... }`**: This declares a TypeScript interface named `PathSegmentOptions` and exports it. This interface will be used to type the `options` argument of the `validatePathSegment` function.
*   **`paramName?: string`**:  An optional string that represents the name of the parameter being validated.  This name is used in error messages to provide context to the user.  Defaults to 'path segment' if not provided.
*   **`maxLength?: number`**: An optional number that specifies the maximum allowed length of the path segment.  Defaults to 255 if not provided.
*   **`allowHyphens?: boolean`**:  An optional boolean that indicates whether hyphens (`-`) are allowed in the path segment.  Defaults to `true`.
*   **`allowUnderscores?: boolean`**: An optional boolean that indicates whether underscores (`_`) are allowed in the path segment.  Defaults to `true`.
*   **`allowDots?: boolean`**: An optional boolean that indicates whether dots (`.`) are allowed in the path segment.  Defaults to `false`.  Disallowing dots is crucial for preventing directory traversal attacks (e.g., `../`).
*   **`customPattern?: RegExp`**: An optional regular expression that can be used to define a custom validation pattern for the path segment.  If provided, this pattern will be used instead of the default alphanumeric pattern.

```typescript
/**
 * Validates a path segment to prevent path traversal and SSRF attacks
 *
 * This function ensures that user-provided input used in URL paths or file paths
 * cannot be used for directory traversal attacks or SSRF.
 *
 * Default behavior:
 * - Allows: letters (a-z, A-Z), numbers (0-9), hyphens (-), underscores (_)
 * - Blocks: dots (.), slashes (/, \), null bytes, URL encoding, and special characters
 *
 * @param value - The path segment to validate
 * @param options - Validation options
 * @returns ValidationResult with isValid flag and optional error message
 *
 * @example
 * ```typescript
 * const result = validatePathSegment(itemId, { paramName: 'itemId' })
 * if (!result.isValid) {
 *   return NextResponse.json({ error: result.error }, { status: 400 })
 * }
 * ```
 */
export function validatePathSegment(
  value: string | null | undefined,
  options: PathSegmentOptions = {}
): ValidationResult {
  const {
    paramName = 'path segment',
    maxLength = 255,
    allowHyphens = true,
    allowUnderscores = true,
    allowDots = false,
    customPattern,
  } = options

  // Check for null/undefined
  if (value === null || value === undefined || value === '') {
    return {
      isValid: false,
      error: `${paramName} is required`,
    }
  }

  // Check length
  if (value.length > maxLength) {
    logger.warn('Path segment exceeds maximum length', {
      paramName,
      length: value.length,
      maxLength,
    })
    return {
      isValid: false,
      error: `${paramName} exceeds maximum length of ${maxLength} characters`,
    }
  }

  // Check for null bytes (potential for bypass attacks)
  if (value.includes('\0') || value.includes('%00')) {
    logger.warn('Path segment contains null bytes', { paramName })
    return {
      isValid: false,
      error: `${paramName} contains invalid characters`,
    }
  }

  // Check for path traversal patterns
  const pathTraversalPatterns = [
    '..',
    './',
    '.\\.', // Windows path traversal
    '%2e%2e', // URL encoded ..
    '%252e%252e', // Double URL encoded ..
    '..%2f',
    '..%5c',
    '%2e%2e%2f',
    '%2e%2e/',
    '..%252f',
  ]

  const lowerValue = value.toLowerCase()
  for (const pattern of pathTraversalPatterns) {
    if (lowerValue.includes(pattern.toLowerCase())) {
      logger.warn('Path traversal attempt detected', {
        paramName,
        pattern,
        value: value.substring(0, 100),
      })
      return {
        isValid: false,
        error: `${paramName} contains invalid path traversal sequences`,
      }
    }
  }

  // Check for directory separators
  if (value.includes('/') || value.includes('\\')) {
    logger.warn('Path segment contains directory separators', { paramName })
    return {
      isValid: false,
      error: `${paramName} cannot contain directory separators`,
    }
  }

  // Use custom pattern if provided
  if (customPattern) {
    if (!customPattern.test(value)) {
      logger.warn('Path segment failed custom pattern validation', {
        paramName,
        pattern: customPattern.toString(),
      })
      return {
        isValid: false,
        error: `${paramName} format is invalid`,
      }
    }
    return { isValid: true, sanitized: value }
  }

  // Build allowed character pattern
  let pattern = '^[a-zA-Z0-9'
  if (allowHyphens) pattern += '\\-'
  if (allowUnderscores) pattern += '_'
  if (allowDots) pattern += '\\.'
  pattern += ']+$'

  const regex = new RegExp(pattern)

  if (!regex.test(value)) {
    logger.warn('Path segment contains disallowed characters', {
      paramName,
      value: value.substring(0, 100),
    })
    return {
      isValid: false,
      error: `${paramName} can only contain alphanumeric characters${allowHyphens ? ', hyphens' : ''}${allowUnderscores ? ', underscores' : ''}${allowDots ? ', dots' : ''}`,
    }
  }

  return { isValid: true, sanitized: value }
}
```

*   **Purpose:** The `validatePathSegment` function validates a string to ensure it's safe for use as a path segment in a URL or file path, preventing directory traversal and SSRF vulnerabilities.

*   **`export function validatePathSegment(...)`**:  Defines and exports the main validation function. It accepts the input `value` (string, potentially null or undefined) and an optional `options` object of type `PathSegmentOptions`. It returns a `ValidationResult` object.

*   **`const { ... } = options`**: This destructures the `options` object, providing default values if certain options are not provided in the `options` argument.
    *   `paramName = 'path segment'` Defaults the parameter name to "path segment".
    *   `maxLength = 255` Defaults the maximum length to 255 characters.
    *   `allowHyphens = true` Defaults to allowing hyphens.
    *   `allowUnderscores = true` Defaults to allowing underscores.
    *   `allowDots = false` Defaults to *not* allowing dots (important for security).
    *   `customPattern`  If a custom regex pattern is supplied, it will be used instead of the default.

*   **`if (value === null || value === undefined || value === '')`**: Checks if the input `value` is null, undefined, or an empty string. If so, it's considered invalid, and an appropriate error message is returned.

*   **`if (value.length > maxLength)`**: Checks if the length of the input `value` exceeds the configured `maxLength`. If so, it logs a warning using the `logger` and returns an error message.

*   **`if (value.includes('\0') || value.includes('%00'))`**: This check looks for null bytes (represented as `\0` or URL-encoded `%00`) within the input. Null bytes can be used to truncate strings and potentially bypass security checks.  If found, a warning is logged, and the function returns an error.

*   **`const pathTraversalPatterns = [...]`**: Defines an array of strings representing common path traversal patterns (e.g., `../`, `./`, URL-encoded variations).

*   **`const lowerValue = value.toLowerCase()`**: Converts the input to lowercase for case-insensitive matching of path traversal patterns.

*   **`for (const pattern of pathTraversalPatterns)`**: Iterates through the `pathTraversalPatterns` array and checks if any of these patterns are present in the lowercase version of the input.

*   **`if (lowerValue.includes(pattern.toLowerCase()))`**: Performs the case-insensitive check for path traversal patterns. If a pattern is found, a warning is logged with information about the detected attempt and then returns an error.

*   **`if (value.includes('/') || value.includes('\\'))`**: Checks if the input contains forward slashes (`/`) or backslashes (`\`), which are directory separators. Including those characters would allow the user to specify a different path. If found, a warning is logged, and an error message is returned.

*   **`if (customPattern)`**: Checks if a `customPattern` regular expression was provided in the options.
    *   **`if (!customPattern.test(value))`**: If a custom pattern is provided, it tests the input `value` against the pattern.  If the input *doesn't* match the pattern (i.e., `test` returns `false`), a warning is logged, and an error message is returned.
    *   **`return { isValid: true, sanitized: value }`**: If the input *does* match the custom pattern, the function considers the input valid and returns a `ValidationResult` with `isValid: true` and the original `value` as the `sanitized` value.

*   **`let pattern = '^[a-zA-Z0-9'`**:  If no `customPattern` is provided, this part dynamically builds a regular expression pattern based on the `allowHyphens`, `allowUnderscores`, and `allowDots` options. It starts with a base pattern that allows alphanumeric characters.

*   **`if (allowHyphens) pattern += '\\-'`**: Appends `\\-` to the pattern if `allowHyphens` is true, allowing hyphens in the input. The backslash is necessary to escape the hyphen, which has a special meaning in regular expressions.

*   **`if (allowUnderscores) pattern += '_'`**: Appends `_` to the pattern if `allowUnderscores` is true, allowing underscores in the input.

*   **`if (allowDots) pattern += '\\.'`**: Appends `\\.` to the pattern if `allowDots` is true, allowing dots in the input. The backslash is necessary to escape the dot, which has a special meaning in regular expressions.

*   **`pattern += ']+$'`**: Completes the regular expression pattern. The `]+` means "one or more of the preceding characters," and the `$` means "end of the string."  The `^` at the beginning means "start of the string."  Together, `^[...]` means that the *entire* input must match the allowed characters.

*   **`const regex = new RegExp(pattern)`**: Creates a new `RegExp` object from the dynamically built pattern string.

*   **`if (!regex.test(value))`**: Tests the input `value` against the generated regular expression.  If the input *doesn't* match the pattern, a warning is logged and an error message is returned.

*   **`return { isValid: true, sanitized: value }`**: If all checks pass (no null bytes, no path traversal attempts, no directory separators, and matches the allowed character pattern), the function considers the input valid and returns a `ValidationResult` with `isValid: true` and the original `value` as the `sanitized` value.

```typescript
/**
 * Validates a UUID (v4 format)
 *
 * @param value - The UUID to validate
 * @param paramName - Name of the parameter for error messages
 * @returns ValidationResult
 *
 * @example
 * ```typescript
 * const result = validateUUID(workflowId, 'workflowId')
 * if (!result.isValid) {
 *   return NextResponse.json({ error: result.error }, { status: 400 })
 * }
 * ```
 */
export function validateUUID(
  value: string | null | undefined,
  paramName = 'UUID'
): ValidationResult {
  if (value === null || value === undefined || value === '') {
    return {
      isValid: false,
      error: `${paramName} is required`,
    }
  }

  // UUID v4 pattern
  const uuidPattern = /^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/i

  if (!uuidPattern.test(value)) {
    logger.warn('Invalid UUID format', { paramName, value: value.substring(0, 50) })
    return {
      isValid: false,
      error: `${paramName} must be a valid UUID`,
    }
  }

  return { isValid: true, sanitized: value.toLowerCase() }
}
```

*   **Purpose:** The `validateUUID` function validates a string to ensure it conforms to the UUID v4 format.

*   **`export function validateUUID(...)`**:  Defines and exports the validation function. It accepts the input `value` (string, potentially null or undefined) and an optional `paramName` (string) for use in error messages. It returns a `ValidationResult` object.  The `paramName` defaults to 'UUID'.

*   **`if (value === null || value === undefined || value === '')`**: Checks for null, undefined, or empty input, returning an error if found.

*   **`const uuidPattern = /^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/i`**: Defines a regular expression (`uuidPattern`) that matches the structure of a UUID v4. Let's break down this regex:
    *   `^[0-9a-f]{8}`: Starts with 8 hexadecimal characters (0-9 and a-f).
    *   `-`: Followed by a hyphen.
    *   `[0-9a-f]{4}`: Followed by 4 hexadecimal characters.
    *   `-4[0-9a-f]{3}`: Followed by a hyphen, the number 4, and 3 hexadecimal characters (this is specific to UUID v4).
    *   `-[89ab][0-9a-f]{3}`: Followed by a hyphen, one of the characters 8, 9, a, or b, and 3 hexadecimal characters.
    *   `-[0-9a-f]{12}$`: Followed by a hyphen and 12 hexadecimal characters, and then the end of the string.
    *   `/i`: The `i` flag at the end makes the regex case-insensitive.

*   **`if (!uuidPattern.test(value))`**: Tests the input `value` against the `uuidPattern` regular expression. If it doesn't match, the function logs a warning and returns an error.

*   **`return { isValid: true, sanitized: value.toLowerCase() }`**: If the input is a valid UUID v4, the function returns a `ValidationResult` with `isValid: true` and the `sanitized` value as the lowercase version of the input. Converting to lowercase is a common sanitization practice for UUIDs.

```typescript
/**
 * Validates an alphanumeric ID (letters, numbers, hyphens, underscores only)
 *
 * @param value - The ID to validate
 * @param paramName - Name of the parameter for error messages
 * @param maxLength - Maximum length (default: 100)
 * @returns ValidationResult
 *
 * @example
 * ```typescript
 * const result = validateAlphanumericId(userId, 'userId')
 * if (!result.isValid) {
 *   return NextResponse.json({ error: result.error }, { status: 400 })
 * }
 * ```
 */
export function validateAlphanumericId(
  value: string | null | undefined,
  paramName = 'ID',
  maxLength = 100
): ValidationResult {
  return validatePathSegment(value, {
    paramName,
    maxLength,
    allowHyphens: true,
    allowUnderscores: true,
    allowDots: false,
  })
}
```

*   **Purpose:** The `validateAlphanumericId` function validates a string to ensure it contains only alphanumeric characters, hyphens, and underscores. It reuses the `validatePathSegment` function with specific options configured for this purpose.

*   **`export function validateAlphanumericId(...)`**: Defines and exports the validation function.  It accepts the input `value` (string, potentially null or undefined), an optional `paramName` (string) for use in error messages (defaults to 'ID'), and an optional `maxLength` (number) for the maximum allowed length (defaults to 100).  It returns a `ValidationResult` object.

*   **`return validatePathSegment(value, { ... })`**:  This is the key part. It calls the `validatePathSegment` function, passing the input `value` and an options object. The options object configures `validatePathSegment` to allow only alphanumeric characters, hyphens, and underscores, effectively implementing the alphanumeric ID validation.
    *   `paramName`: Passed directly from the `validateAlphanumericId` function's argument.
    *   `maxLength`: Passed directly from the `validateAlphanumericId` function's argument.
    *   `allowHyphens: true`: Explicitly allows hyphens.
    *   `allowUnderscores: true`: Explicitly allows underscores.
    *   `allowDots: false`: Explicitly *disallows* dots.

This function demonstrates code reuse and abstraction. Instead of duplicating the validation logic, it leverages the existing `validatePathSegment` function and customizes its behavior with specific options.

```typescript
/**
 * Validates a numeric ID
 *
 * @param value - The ID to validate
 * @param paramName - Name of the parameter for error messages
 * @param options - Additional options (min, max)
 * @returns ValidationResult with sanitized number as string
 *
 * @example
 * ```typescript
 * const result = validateNumericId(pageNumber, 'pageNumber', { min: 1, max: 1000 })
 * if (!result.isValid) {
 *   return NextResponse.json({ error: result.error }, { status: 400 })
 * }
 * ```
 */
export function validateNumericId(
  value: string | number | null | undefined,
  paramName = 'ID',
  options: { min?: number; max?: number } = {}
): ValidationResult {
  if (value === null || value === undefined || value === '') {
    return {
      isValid: false,
      error: `${paramName} is required`,
    }
  }

  const num = typeof value === 'number' ? value : Number(value)

  if (Number.isNaN(num) || !Number.isFinite(num)) {
    logger.warn('Invalid numeric ID', { paramName, value })
    return {
      isValid: false,
      error: `${paramName} must be a valid number`,
    }
  }

  if (options.min !== undefined && num < options.min) {
    return {
      isValid: false,
      error: `${paramName} must be at least ${options.min}`,
    }
  }

  if (options.max !== undefined && num > options.max) {
    return {
      isValid: false,
      error: `${paramName} must be at most ${options.max}`,
    }
  }

  return { isValid: true, sanitized: num.toString() }
}
```

*   **Purpose:** The `validateNumericId` function validates a value to ensure it represents a valid number, optionally within a specified range (minimum and maximum).

*   **`export function validateNumericId(...)`**: Defines and exports the function. It takes the input `value` (string, number, null, or undefined), an optional `paramName` (string) for error messages (defaults to 'ID'), and an optional `options` object that can contain `min` and `max` properties (both numbers) to define the acceptable range.  Returns a `ValidationResult` object.

*   **`if (value === null || value === undefined || value === '')`**: Checks if the input `value` is null, undefined, or an empty string. If so, it returns an error, as a numeric ID is required.

*   **`const num = typeof value === 'number' ? value : Number(value)`**: This line attempts to convert the input `value` to a number.
    *   `typeof value === 'number' ? value : ...`: This is a ternary operator. If the `value` is already a number, it's assigned directly to `num`.
    *   `Number(value)`: If the `value` is *not* a number (likely a string), it attempts to convert it to a number using the `Number()` constructor.

*   **`if (Number.isNaN(num) || !Number.isFinite(num))`**: This checks if the result of the conversion is a valid number.
    *   `Number.isNaN(num)`: Checks if the `num` is `NaN` (Not a Number). This occurs if the string cannot be parsed as a number (e.g., `Number("abc")` returns `NaN`).
    *   `!Number.isFinite(num)`: Checks if `num` is *not* a finite number (e.g., `Infinity`, `-Infinity`).

*   **`if (options.min !== undefined && num < options.min)`**: Checks if a minimum value (`options.min`) is defined and if the converted number `num` is less than that minimum. If so, an error is returned.

*   **`if (options.max !== undefined && num > options.max)`**: Checks if a maximum value (`options.max`) is defined and if the converted number `num` is greater than that maximum. If so, an error is returned.

*   **`return { isValid: true, sanitized: num.toString() }`**: If all checks pass, the function returns a `ValidationResult` with `isValid: true` and the `sanitized` value as the number converted back to a string.

```typescript
/**
 * Validates that a value is in an allowed list (enum validation)
 *
 * @param value - The value to validate
 * @param allowedValues - Array of allowed values
 * @param paramName - Name of the parameter for error messages
 * @returns ValidationResult
 *
 * @example
 * ```typescript
 * const result = validateEnum(type, ['note', 'contact', 'task'], 'type')
 * if (!result.isValid) {
 *   return NextResponse.json({ error: result.error }, { status: 400 })
 * }
 * ```
 */
export function validateEnum<T extends string>(
  value: string | null | undefined,
  allowedValues: readonly T[],
  paramName = 'value'
): ValidationResult {
  if (value === null || value === undefined || value === '') {
    return {
      isValid: false,
      error: `${paramName} is required`,
    }
  }

  if (!allowedValues.includes(value as T)) {
    logger.warn('Value not in allowed list', {
      paramName,
      value,
      allowedValues,
    })
    return {
      isValid: false,
      error: `${paramName} must be one of: ${allowedValues.join(', ')}`,
    }
  }

  return { isValid: true, sanitized: value }
}
```

*   **Purpose:** The `validateEnum` function validates that a given value is present within a predefined list of allowed values, effectively performing enum validation.

*   **`export function validateEnum<T extends string>(...)`**: Defines and exports the function. It uses a generic type `T` which extends `string`, ensuring that the allowed values are strings.  It takes the input `value` (string, null, or undefined), `allowedValues` (a read-only array of type `T`), and an optional `paramName` (string) for error messages (defaults to 'value').  Returns a `ValidationResult` object.  The use of `readonly T[]` ensures that the `allowedValues` array cannot be modified within the function.

*   **`if (value === null || value === undefined || value === '')`**: Checks for null, undefined, or empty input, returning an error if found.

*   **`if (!allowedValues.includes(value as T))`**: This is the core validation logic.
    *   `allowedValues.includes(value as T)`:  Checks if the `allowedValues` array includes the input `value`. The `value as T` is a type assertion that tells TypeScript to treat the `value` as being of type `T`.  This is necessary because TypeScript needs to be sure that the `value` is of the same type as the elements in the `allowedValues` array.
    *   `!allowedValues.includes(...)`: The `!` negates the result of `includes()`.  So, if the `value` is *not* found in the `allowedValues` array, the condition is true.  In that case, a warning is logged, and an error message is returned, indicating the allowed values.

*   **`return { isValid: true, sanitized: value }`**: If the `value` is found in the `allowedValues` array, the function returns a `ValidationResult` with `isValid: true` and the original `value` as the `sanitized` value.

```typescript
/**
 * Validates a hostname to prevent SSRF attacks
 *
 * This function checks that a hostname is not a private IP, localhost, or other reserved address.
 * It complements the validateProxyUrl function by providing hostname-specific validation.
 *
 * @param hostname - The hostname to validate
 * @param paramName - Name of the parameter for error messages
 * @returns ValidationResult
 *
 * @example
 * ```typescript
 * const result = validateHostname(webhookDomain, 'webhook domain')
 * if (!result.isValid) {
 *   return NextResponse.json({ error: result.error }, { status: 400 })
 * }
 * ```
 */
export function validateHostname(
  hostname: string | null | undefined,
  paramName = 'hostname'
): ValidationResult {
  if (hostname === null || hostname === undefined || hostname === '') {
    return {
      isValid: false,
      error: `${paramName} is required`,
    }
  }

  // Import the blocked IP ranges from url-validation
  const BLOCKED_IP_RANGES = [
    // Private IPv4 ranges (RFC 1918)
    /^10\./,
    /^172\.(1[6-9]|2[0-9]|3[01])\./,
    /^192\.168\./,

    // Loopback addresses
    /^127\./,
    /^localhost$/i,

    // Link-local addresses (RFC 3927)
    /^169\.254\./,

    // Cloud metadata endpoints
    /^169\.254\.169\.254$/,

    // Broadcast and other reserved ranges
    /^0\./,
    /^224\./,
    /^240\./,
    /^255\./,

    // IPv6 loopback and link-local
    /^::1$/,
    /^fe80:/i,
    /^::ffff:127\./i,
    /^::ffff:10\./i,
    /^::ffff:172\.(1[6-9]|2[0-9]|3[01])\./i,
    /^::ffff:192\.168\./i,
  ]

  const lowerHostname = hostname.toLowerCase()

  for (const pattern of BLOCKED_IP_RANGES) {
    if (pattern.test(lowerHostname)) {
      logger.warn('Hostname matches blocked IP range', {
        paramName,
        hostname: hostname.substring(0, 100),
      })
      return {
        isValid: false,
        error: `${paramName} cannot be a private IP address or localhost`,
      }
    }
  }

  // Basic hostname format validation
  const hostnamePattern =
    /^[a-z0-9]([a-z0-9-]{0,61}[a-z0-9])?(\.[a-z0-9]([a-z0-9-]{0,61}[a-z0-9])?)*$/i

  if (!hostnamePattern.test(hostname)) {
    logger.warn('Invalid hostname format', {
      paramName,
      hostname: hostname.substring(0, 100),
    })
    return {
      isValid: false,
      error: `${paramName} is not a valid hostname`,
    }
  }

  return { isValid: true, sanitized: hostname }
}
```

*   **Purpose:** The `validateHostname` function validates a hostname to prevent Server-Side Request Forgery (SSRF) attacks. It checks if the hostname is a private IP address, localhost, or other reserved address.

*   **`export function validateHostname(...)`**: Defines and exports the function. It accepts the `hostname` (string, null, or undefined) and an optional `paramName` (string) for error messages (defaults to 'hostname'). Returns a `ValidationResult` object.

*   **`if (hostname === null || hostname === undefined || hostname === '')`**: Checks if the `hostname` is null, undefined, or an empty string, returning an error if it is.

*   **`const BLOCKED_IP_RANGES = [...]`**: This is a crucial part of the SSRF protection. It defines an array of regular expressions that match various private IP address ranges, loopback addresses (localhost), link-local addresses, cloud metadata endpoints, and other reserved ranges.  These addresses are generally not safe to access from a public server, as they might expose internal services or resources.
    *   Each entry in the array is a regular expression.  For example, `/^10\./` matches any string that *starts with* "10." (the beginning of a Class A private IP address range).  The `^` anchors the match to the start of the string, and the `.` matches a literal dot.

*   **`const lowerHostname = hostname.toLowerCase()`**: Converts the `hostname` to lowercase for case-insensitive matching.

*   **`for (const pattern of BLOCKED_IP_RANGES)`**: Iterates through the `BLOCKED_IP_RANGES` array.

*   **`if (pattern.test(lowerHostname))`**: Checks if the lowercase `hostname` matches any of the blocked IP address patterns using the `test()` method of the regular expression. If a match is found, it means the hostname is pointing to a private or reserved address, which is a potential SSRF vulnerability.  A warning is logged and returns an error.

*   **`const hostnamePattern = /^[a-z0-9]([a-z0-9-]{0,61}[a-z0-9])?(\.[a-z0-9]([a-z0-9-]{0,61}[a-z0-9])?)*$/i`**: Defines a regular expression (`hostnamePattern`) to validate the general format of a hostname.  This regex is complex, but here's a simplified explanation:
    *   `^[a-z0-9]`: The hostname must start with a lowercase letter or number.
    *   `([a-z0-9-]{0,61}[a-z0-9])?`:  An optional group that allows for 0 to 61 lowercase letters, numbers, or hyphens, and it *must* end with a letter or a number.  This ensures that a hyphen doesn't appear at the beginning or end of a hostname segment.
    *   `(\.[a-z0-9]([a-z0-9-]{0,61}[a-z0-9])?)*`: An optional group that