This file is a **Vitest test suite** for a collection of utility functions located in `@/lib/utils`. Its primary purpose is to ensure that these utility functions work correctly, handle various inputs as expected, and maintain their intended behavior over time.

Think of it as a quality assurance checklist for the `utils` library. Each `describe` block focuses on a specific utility function or a group of related functions, and within each block, individual `it` statements define specific scenarios and expected outcomes.

Here's a detailed, easy-to-read explanation:

---

### **Purpose of this File (`utils.test.ts`)**

This file serves as the **test bedrock** for your application's core utility functions. In software development, utility functions are small, reusable pieces of code that perform common tasks (like formatting dates, validating inputs, or handling encryption). This test file uses **Vitest**, a fast and modern testing framework, to:

1.  **Verify Correctness**: Ensure each utility function produces the expected output for various inputs.
2.  **Prevent Regressions**: Catch unintended side effects or bugs introduced when changes are made to the utility functions.
3.  **Document Behavior**: The tests themselves act as living documentation, showing how each function is supposed to be used and what it's designed to do.
4.  **Isolate Logic**: Use mocking to test functions in isolation, ensuring they work independently of external dependencies (like the `crypto` module or environment variables).

In essence, this file guarantees that the building blocks of your application are robust and reliable.

---

### **Detailed Explanation of Each Section**

```typescript
import { afterEach, describe, expect, it, vi } from 'vitest'
import {
  cn,
  convertScheduleOptionsToCron,
  decryptSecret,
  encryptSecret,
  formatDate,
  formatDateTime,
  formatDuration,
  formatTime,
  getInvalidCharacters,
  getTimezoneAbbreviation,
  isValidName,
  redactApiKeys,
  validateName,
} from '@/lib/utils'
```

*   **`import { afterEach, describe, expect, it, vi } from 'vitest'`**: These are imports from the `vitest` testing framework.
    *   `describe`: Used to group related tests together into a test suite.
    *   `it`: Defines an individual test case. `it.concurrent` means this test can run in parallel with others, potentially speeding up the test suite.
    *   `expect`: Used for making assertions (e.g., `expect(result).toBe('expected')`) that check if a value meets certain conditions.
    *   `vi`: The Vitest mocking utility, used to replace parts of your code with "mock" versions for testing purposes.
    *   `afterEach`: A hook that runs after each individual test within a `describe` block, typically used for cleanup.
*   **`import { ... } from '@/lib/utils'`**: This line imports all the specific utility functions that will be tested in this file. The `@/lib/utils` path indicates that these functions are part of a shared utility library within your project.

---

```typescript
vi.mock('crypto', () => ({
  createCipheriv: vi.fn().mockReturnValue({
    update: vi.fn().mockReturnValue('encrypted-data'),
    final: vi.fn().mockReturnValue('final-data'),
    getAuthTag: vi.fn().mockReturnValue({
      toString: vi.fn().mockReturnValue('auth-tag'),
    }),
  }),
  createDecipheriv: vi.fn().mockReturnValue({
    update: vi.fn().mockReturnValue('decrypted-data'),
    final: vi.fn().mockReturnValue('final-data'),
    setAuthTag: vi.fn(),
  }),
  randomBytes: vi.fn().mockReturnValue({
    toString: vi.fn().mockReturnValue('random-iv'),
  }),
}))
```

This block uses Vitest's `vi.mock` function to **mock** the built-in Node.js `crypto` module.
**Why mock `crypto`?**
*   **Determinism**: Actual cryptographic operations involve randomness and complex computations. For testing, we want predictable results. Mocking ensures that `encryptSecret` and `decryptSecret` always get the same "fake" encrypted/decrypted data, IVs, and auth tags, making tests reliable and repeatable.
*   **Isolation**: We want to test *our* `encryptSecret` and `decryptSecret` functions, not the underlying `crypto` library. Mocking allows us to isolate our code.
*   **Speed**: Real encryption can be computationally intensive; mocks are instant.

*   **`vi.mock('crypto', () => ({ ... }))`**: Tells Vitest to replace the actual `crypto` module with this custom mock implementation whenever it's imported during tests.
*   **`createCipheriv: vi.fn().mockReturnValue(...)`**: Mocks the `createCipheriv` function (used for encryption). Instead of creating a real cipher, it returns a mock object.
    *   This mock object has `update`, `final`, and `getAuthTag` methods.
    *   `update: vi.fn().mockReturnValue('encrypted-data')`: The `update` method (which processes data during encryption) is mocked to always return the string `'encrypted-data'`.
    *   `final: vi.fn().mockReturnValue('final-data')`: The `final` method (which finishes the encryption process) is mocked to always return `'final-data'`.
    *   `getAuthTag: vi.fn().mockReturnValue({ toString: vi.fn().mockReturnValue('auth-tag') })`: The `getAuthTag` method (which provides an authentication tag for integrity) is mocked to return an object whose `toString` method returns `'auth-tag'`.
*   **`createDecipheriv: vi.fn().mockReturnValue(...)`**: Mocks the `createDecipheriv` function (used for decryption) similarly.
    *   `update: vi.fn().mockReturnValue('decrypted-data')`: Returns `'decrypted-data'` when called.
    *   `final: vi.fn().mockReturnValue('final-data')`: Returns `'final-data'` when called.
    *   `setAuthTag: vi.fn()`: The `setAuthTag` method is just mocked without a return value, as its purpose is to accept data, not return it.
*   **`randomBytes: vi.fn().mockReturnValue(...)`**: Mocks the `randomBytes` function (used to generate Initialization Vectors, or IVs).
    *   `toString: vi.fn().mockReturnValue('random-iv')`: It's mocked to return an object whose `toString` method always returns `'random-iv'`, ensuring a consistent IV for tests.

---

```typescript
vi.mock('@/lib/env', () => ({
  env: {
    ENCRYPTION_KEY: '0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef',
  },
}))
```

This block mocks the environment variables module (`@/lib/env`).
**Why mock environment variables?**
*   **Consistency**: Environment variables can change depending on the deployment environment. For tests, you need a fixed, known value.
*   **Security**: You should *never* expose real sensitive keys in your test files or source code. Mocking provides a dummy key.

*   **`vi.mock('@/lib/env', () => ({ ... }))`**: Replaces the actual `@/lib/env` module with this mock.
*   **`env: { ENCRYPTION_KEY: '...' }`**: Provides a fake `ENCRYPTION_KEY` that the encryption/decryption functions will use during tests. This key is a simple placeholder, not a real cryptographic key.

---

```typescript
afterEach(() => {
  vi.clearAllMocks()
})
```

*   **`afterEach(() => { ... })`**: This is a Vitest lifecycle hook. The function inside will run *after* every single `it` test case has completed.
*   **`vi.clearAllMocks()`**: This crucial call resets all mock functions (like `createCipheriv`, `randomBytes`, etc.) to their initial state before the next test runs. This ensures that the results or call counts from one test don't "leak" into and affect subsequent tests, promoting test isolation.

---

### `describe('cn (class name utility)')`

This section tests the `cn` utility function, which is commonly used in web development to conditionally combine CSS class names. It's similar to popular libraries like `clsx` or `classnames`.

```typescript
describe('cn (class name utility)', () => {
  it.concurrent('should merge class names correctly', () => {
    const result = cn('class1', 'class2')
    expect(result).toBe('class1 class2')
  })

  it.concurrent('should handle conditional classes', () => {
    const isActive = true
    const result = cn('base', isActive && 'active')
    expect(result).toBe('base active')
  })

  it.concurrent('should handle falsy values', () => {
    const result = cn('base', false && 'hidden', null, undefined, 0, '')
    expect(result).toBe('base')
  })

  it.concurrent('should handle arrays of class names', () => {
    const result = cn('base', ['class1', 'class2'])
    expect(result).toContain('base')
    expect(result).toContain('class1')
    expect(result).toContain('class2')
  })
})
```

*   **`describe('cn (class name utility)', () => { ... })`**: Groups all tests related to the `cn` function.
*   **`it.concurrent('should merge class names correctly', () => { ... })`**:
    *   **Purpose**: Tests basic merging of multiple string arguments.
    *   `cn('class1', 'class2')` should concatenate them with a space.
    *   `expect(result).toBe('class1 class2')`: Asserts that the output is exactly `'class1 class2'`.
*   **`it.concurrent('should handle conditional classes', () => { ... })`**:
    *   **Purpose**: Tests `cn`'s ability to include classes based on a boolean condition.
    *   `isActive && 'active'` evaluates to `'active'` because `isActive` is `true`.
    *   `cn('base', 'active')` should result in `'base active'`.
*   **`it.concurrent('should handle falsy values', () => { ... })`**:
    *   **Purpose**: Ensures `cn` ignores common "falsy" JavaScript values (`false`, `null`, `undefined`, `0`, `''`) when merging.
    *   The `false && 'hidden'` part evaluates to `false` and is ignored. The other falsy values are also ignored.
    *   `expect(result).toBe('base')`: Only `'base'` should remain.
*   **`it.concurrent('should handle arrays of class names', () => { ... })`**:
    *   **Purpose**: Tests that `cn` can process an array of class names.
    *   `cn('base', ['class1', 'class2'])` should combine `'base'` with the elements of the array.
    *   `expect(result).toContain('base')`, `expect(result).toContain('class1')`, `expect(result).toContain('class2')`: These assertions check that the resulting string *contains* each of the expected class names, allowing for flexibility in the order if `cn` normalizes spaces.

---

### `describe('encryption and decryption')`

This section tests the `encryptSecret` and `decryptSecret` functions, which rely on the mocked `crypto` module and `ENCRYPTION_KEY`.

```typescript
describe('encryption and decryption', () => {
  it.concurrent('should encrypt secrets correctly', async () => {
    const result = await encryptSecret('my-secret')
    expect(result).toHaveProperty('encrypted')
    expect(result).toHaveProperty('iv')
    expect(result.encrypted).toContain('random-iv')
    expect(result.encrypted).toContain('encrypted-data')
    expect(result.encrypted).toContain('final-data')
    expect(result.encrypted).toContain('auth-tag')
  })

  it.concurrent('should decrypt secrets correctly', async () => {
    const result = await decryptSecret('iv:encrypted:authTag')
    expect(result).toHaveProperty('decrypted')
    expect(result.decrypted).toBe('decrypted-datafinal-data')
  })

  it.concurrent('should throw error for invalid decrypt format', async () => {
    await expect(decryptSecret('invalid-format')).rejects.toThrow('Invalid encrypted value format')
  })
})
```

*   **`describe('encryption and decryption', () => { ... })`**: Groups tests for encryption and decryption.
*   **`it.concurrent('should encrypt secrets correctly', async () => { ... })`**:
    *   **Purpose**: Verifies that the `encryptSecret` function processes a secret and returns the expected structure and mocked data.
    *   `await encryptSecret('my-secret')`: Calls the encryption function with a dummy secret. Since `crypto` is mocked, it will use the mocked return values.
    *   `expect(result).toHaveProperty('encrypted')` and `expect(result).toHaveProperty('iv')`: Asserts that the returned object has `encrypted` and `iv` properties (which are crucial for decryption).
    *   `expect(result.encrypted).toContain(...)`: These checks verify that the `encrypted` string produced by `encryptSecret` contains the mocked parts (`'random-iv'`, `'encrypted-data'`, `'final-data'`, `'auth-tag'`). This confirms that `encryptSecret` is using the `crypto` functions in the expected sequence to construct the final encrypted string.
*   **`it.concurrent('should decrypt secrets correctly', async () => { ... })`**:
    *   **Purpose**: Verifies that the `decryptSecret` function can take an encrypted string (in a specific format) and produce the mocked decrypted data.
    *   `await decryptSecret('iv:encrypted:authTag')`: Calls the decryption function with a string that *mimics* the expected format (IV, encrypted data, auth tag, separated by colons).
    *   `expect(result).toHaveProperty('decrypted')`: Asserts that the result object has a `decrypted` property.
    *   `expect(result.decrypted).toBe('decrypted-datafinal-data')`: Asserts that the decrypted value matches the combination of the `update` and `final` mock returns from `createDecipheriv`.
*   **`it.concurrent('should throw error for invalid decrypt format', async () => { ... })`**:
    *   **Purpose**: Tests error handling for incorrectly formatted input to `decryptSecret`.
    *   `await expect(decryptSecret('invalid-format')).rejects.toThrow('Invalid encrypted value format')`: This assertion specifically checks that calling `decryptSecret` with a malformed string causes it to throw an error with the specified message. `rejects.toThrow` is used because `decryptSecret` is an `async` function that might reject its promise.

---

### `describe('convertScheduleOptionsToCron')`

This section tests `convertScheduleOptionsToCron`, a function that converts human-readable scheduling options (like "every 5 minutes" or "daily at 9 AM") into a Cron expression string. Cron expressions are a standard way to define scheduled tasks on Unix-like systems.

```typescript
describe('convertScheduleOptionsToCron', () => {
  it.concurrent('should convert minutes schedule to cron', () => {
    const result = convertScheduleOptionsToCron('minutes', { minutesInterval: '5' })
    expect(result).toBe('*/5 * * * *')
  })

  it.concurrent('should convert hourly schedule to cron', () => {
    const result = convertScheduleOptionsToCron('hourly', { hourlyMinute: '30' })
    expect(result).toBe('30 * * * *')
  })

  // ... (other schedule types)

  it.concurrent('should throw error for unsupported schedule type', () => {
    expect(() => convertScheduleOptionsToCron('invalid', {})).toThrow('Unsupported schedule type')
  })

  it.concurrent('should use default values when options are not provided', () => {
    const result = convertScheduleOptionsToCron('daily', {})
    expect(result).toBe('00 09 * * *')
  })
})
```

*   **`describe('convertScheduleOptionsToCron', () => { ... })`**: Groups tests for Cron conversion.
*   **`it.concurrent('should convert minutes schedule to cron', () => { ... })`**:
    *   **Purpose**: Tests converting a "run every N minutes" setting.
    *   `'minutes'` type with `minutesInterval: '5'`.
    *   `expect(result).toBe('*/5 * * * *')`: `*/5` means "every 5th minute". The rest are wildcards (`*`), meaning "every hour, every day of month, every month, every day of week".
*   **`it.concurrent('should convert hourly schedule to cron', () => { ... })`**:
    *   **Purpose**: Tests converting a "run at minute N of every hour" setting.
    *   `'hourly'` type with `hourlyMinute: '30'`.
    *   `expect(result).toBe('30 * * * *')`: `30` means "at minute 30".
*   **`it.concurrent('should convert daily schedule to cron', () => { ... })`**:
    *   **Purpose**: Tests converting a "run at a specific time every day" setting.
    *   `'daily'` type with `dailyTime: '15:30'`.
    *   `expect(result).toBe('15 30 * * *')`: `15` is the minute, `30` is the hour (24-hour format).
*   **`it.concurrent('should convert weekly schedule to cron', () => { ... })`**:
    *   **Purpose**: Tests converting a "run at a specific time on a specific day of the week" setting.
    *   `'weekly'` type with `weeklyDay: 'MON'`, `weeklyDayTime: '09:30'`.
    *   `expect(result).toBe('09 30 * * 1')`: `09` is the minute, `30` is the hour. The `1` represents Monday (where Sunday is usually 0 or 7, Monday is 1).
*   **`it.concurrent('should convert monthly schedule to cron', () => { ... })`**:
    *   **Purpose**: Tests converting a "run at a specific time on a specific day of the month" setting.
    *   `'monthly'` type with `monthlyDay: '15'`, `monthlyTime: '12:00'`.
    *   `expect(result).toBe('12 00 15 * *')`: `12` is the minute, `00` is the hour, `15` is the day of the month.
*   **`it.concurrent('should use custom cron expression directly', () => { ... })`**:
    *   **Purpose**: Tests that a "custom" schedule type simply returns the provided Cron expression without modification.
    *   `'custom'` type with `cronExpression: '*/15 9-17 * * 1-5'`.
    *   `expect(result).toBe(customCron)`: Ensures the custom string is passed through directly.
*   **`it.concurrent('should throw error for unsupported schedule type', () => { ... })`**:
    *   **Purpose**: Tests error handling for an unrecognized schedule type.
    *   `expect(() => convertScheduleOptionsToCron('invalid', {})).toThrow('Unsupported schedule type')`: Checks that an error with the correct message is thrown.
*   **`it.concurrent('should use default values when options are not provided', () => { ... })`**:
    *   **Purpose**: Tests that if specific options for a schedule type (like `dailyTime`) are missing, the function falls back to sensible default values.
    *   `convertScheduleOptionsToCron('daily', {})` with no `dailyTime`.
    *   `expect(result).toBe('00 09 * * *')`: Assumes the default daily time is 09:00 (9 AM).

---

### `describe('date formatting functions')`

This section tests functions responsible for formatting `Date` objects into human-readable strings.

```typescript
describe('date formatting functions', () => {
  it.concurrent('should format datetime correctly', () => {
    const date = new Date('2023-05-15T14:30:00')
    const result = formatDateTime(date)
    expect(result).toMatch(/May 15, 2023/)
    expect(result).toMatch(/2:30 PM|14:30/)
  })

  it.concurrent('should format date correctly', () => {
    const date = new Date('2023-05-15T14:30:00')
    const result = formatDate(date)
    expect(result).toMatch(/May 15, 2023/)
    expect(result).not.toMatch(/2:30|14:30/)
  })

  it.concurrent('should format time correctly', () => {
    const date = new Date('2023-05-15T14:30:00')
    const result = formatTime(date)
    expect(result).toMatch(/2:30 PM|14:30/)
    expect(result).not.toMatch(/2023|May/)
  })
})
```

*   **`describe('date formatting functions', () => { ... })`**: Groups tests for date and time formatting.
*   **`it.concurrent('should format datetime correctly', () => { ... })`**:
    *   **Purpose**: Tests `formatDateTime`, which should include both date and time.
    *   `const date = new Date('2023-05-15T14:30:00')`: Creates a `Date` object for May 15, 2023, at 2:30 PM (14:30).
    *   `expect(result).toMatch(/May 15, 2023/)`: Uses a regular expression `toMatch` because the exact output string (e.g., "May 15, 2023" vs. "15 May 2023") might vary slightly based on the testing environment's locale settings. This checks for the key date components.
    *   `expect(result).toMatch(/2:30 PM|14:30/)`: Checks for the time part, accounting for both 12-hour (PM) and 24-hour formats.
*   **`it.concurrent('should format date correctly', () => { ... })`**:
    *   **Purpose**: Tests `formatDate`, which should only include the date part.
    *   `expect(result).toMatch(/May 15, 2023/)`: Checks for the date.
    *   `expect(result).not.toMatch(/2:30|14:30/)`: Crucially, asserts that the time component is *not* present in the output.
*   **`it.concurrent('should format time correctly', () => { ... })`**:
    *   **Purpose**: Tests `formatTime`, which should only include the time part.
    *   `expect(result).toMatch(/2:30 PM|14:30/)`: Checks for the time.
    *   `expect(result).not.toMatch(/2023|May/)`: Asserts that date components are *not* present.

---

### `describe('formatDuration')`

This section tests `formatDuration`, a function that converts a duration in milliseconds into a more human-readable string (e.g., "1h 2m 5s").

```typescript
describe('formatDuration', () => {
  it.concurrent('should format milliseconds correctly', () => {
    const result = formatDuration(500)
    expect(result).toBe('500ms')
  })

  it.concurrent('should format seconds correctly', () => {
    const result = formatDuration(5000)
    expect(result).toBe('5s')
  })

  it.concurrent('should format minutes and seconds correctly', () => {
    const result = formatDuration(125000) // 2m 5s
    expect(result).toBe('2m 5s')
  })

  it.concurrent('should format hours, minutes correctly', () => {
    const result = formatDuration(3725000) // 1h 2m 5s
    expect(result).toBe('1h 2m')
  })
})
```

*   **`describe('formatDuration', () => { ... })`**: Groups tests for duration formatting.
*   Each `it.concurrent` block tests a different magnitude of duration:
    *   **`500` ms**: Should display as `'500ms'`.
    *   **`5000` ms (5 seconds)**: Should display as `'5s'`.
    *   **`125000` ms (2 minutes, 5 seconds)**: Should display as `'2m 5s'`.
    *   **`3725000` ms (1 hour, 2 minutes, 5 seconds)**: This test expects `'1h 2m'`, implying the function might round up or only show the largest two significant units, or truncate milliseconds/seconds if they are smaller than a certain threshold or if hours/minutes are present. This highlights an important behavior of the `formatDuration` function.

---

### `describe('getTimezoneAbbreviation')`

This section tests `getTimezoneAbbreviation`, a function that provides the short, common abbreviation for a given IANA timezone string (e.g., "America/Los_Angeles" -> "PST" or "PDT").

```typescript
describe('getTimezoneAbbreviation', () => {
  it.concurrent('should return UTC for UTC timezone', () => {
    const result = getTimezoneAbbreviation('UTC')
    expect(result).toBe('UTC')
  })

  it.concurrent('should return PST/PDT for Los Angeles timezone', () => {
    const winterDate = new Date('2023-01-15') // Standard time
    const summerDate = new Date('2023-07-15') // Daylight time

    const winterResult = getTimezoneAbbreviation('America/Los_Angeles', winterDate)
    const summerResult = getTimezoneAbbreviation('America/Los_Angeles', summerDate)

    expect(['PST', 'PDT']).toContain(winterResult)
    expect(['PST', 'PDT']).toContain(summerResult)
  })

  it.concurrent('should return JST for Tokyo timezone (no DST)', () => {
    const winterDate = new Date('2023-01-15')
    const summerDate = new Date('2023-07-15')

    const winterResult = getTimezoneAbbreviation('Asia/Tokyo', winterDate)
    const summerResult = getTimezoneAbzoneAbbreviation('Asia/Tokyo', summerDate)

    expect(winterResult).toBe('JST')
    expect(summerResult).toBe('JST')
  })

  it.concurrent('should return full timezone name for unknown timezones', () => {
    const result = getTimezoneAbbreviation('Unknown/Timezone')
    expect(result).toBe('Unknown/Timezone')
  })
})
```

*   **`describe('getTimezoneAbbreviation', () => { ... })`**: Groups tests for timezone abbreviation.
*   **`it.concurrent('should return UTC for UTC timezone', () => { ... })`**:
    *   **Purpose**: Verifies the simplest case, "UTC".
*   **`it.concurrent('should return PST/PDT for Los Angeles timezone', () => { ... })`**:
    *   **Purpose**: Demonstrates handling of Daylight Saving Time (DST). The abbreviation for "America/Los_Angeles" changes between Pacific Standard Time (PST) in winter and Pacific Daylight Time (PDT) in summer.
    *   `winterDate` and `summerDate` are used to simulate different times of the year.
    *   `expect(['PST', 'PDT']).toContain(winterResult)` and `expect(['PST', 'PDT']).toContain(summerResult)`: These flexible assertions check if the result is *either* PST or PDT, which is appropriate because the exact abbreviation can depend on the execution environment's locale and underlying system timezone data.
*   **`it.concurrent('should return JST for Tokyo timezone (no DST)', () => { ... })`**:
    *   **Purpose**: Tests a timezone that does not observe DST.
    *   `expect(winterResult).toBe('JST')` and `expect(summerResult).toBe('JST')`: Both winter and summer dates should yield "JST" (Japan Standard Time).
*   **`it.concurrent('should return full timezone name for unknown timezones', () => { ... })`**:
    *   **Purpose**: Tests how the function handles an invalid or unrecognized timezone string.
    *   `expect(result).toBe('Unknown/Timezone')`: Asserts that for an unknown timezone, it simply returns the input string itself (a common fallback behavior).

---

### `describe('redactApiKeys')`

This section tests `redactApiKeys`, a security-focused function that takes an object (or array) and replaces values associated with sensitive keys (like "apiKey", "password") with a placeholder like `***REDACTED***`. This is often used before logging data to prevent sensitive information from appearing in logs.

```typescript
describe('redactApiKeys', () => {
  it.concurrent('should redact API keys in objects', () => {
    const obj = { /* ... */ }
    const result = redactApiKeys(obj)
    expect(result.apiKey).toBe('***REDACTED***')
    expect(result.normalField).toBe('normal-value')
  })

  it.concurrent('should redact API keys in nested objects', () => {
    const obj = { config: { apiKey: 'secret-key', normalField: 'normal-value' } }
    const result = redactApiKeys(obj)
    expect(result.config.apiKey).toBe('***REDACTED***')
  })

  it.concurrent('should redact API keys in arrays', () => {
    const arr = [{ apiKey: 'secret-key-1' }, { apiKey: 'secret-key-2' }]
    const result = redactApiKeys(arr)
    expect(result[0].apiKey).toBe('***REDACTED***')
  })

  it.concurrent('should handle primitive values', () => {
    expect(redactApiKeys('string')).toBe('string')
    expect(redactApiKeys(null)).toBe(null)
  })

  it.concurrent('should handle complex nested structures', () => {
    const obj = { /* ... */ }
    const result = redactApiKeys(obj)
    expect(result.users[0].credentials.apiKey).toBe('***REDACTED***')
    expect(result.config.database.password).toBe('***REDACTED***')
  })
})
```

*   **`describe('redactApiKeys', () => { ... })`**: Groups tests for API key redaction.
*   **`it.concurrent('should redact API keys in objects', () => { ... })`**:
    *   **Purpose**: Tests redaction in a flat object.
    *   The `obj` includes various common sensitive key names (`apiKey`, `api_key`, `access_token`, `secret`, `password`).
    *   Assertions check that these sensitive fields are redacted, while `normalField` remains unchanged.
*   **`it.concurrent('should redact API keys in nested objects', () => { ... })`**:
    *   **Purpose**: Tests that the function works recursively for nested objects.
    *   `obj.config.apiKey` is expected to be redacted.
*   **`it.concurrent('should redact API keys in arrays', () => { ... })`**:
    *   **Purpose**: Tests redaction within an array of objects.
    *   Each object in the `arr` array is processed, and its `apiKey` is redacted.
*   **`it.concurrent('should handle primitive values', () => { ... })`**:
    *   **Purpose**: Ensures the function correctly handles non-object/non-array inputs by returning them as-is.
    *   Strings, numbers, `null`, and `undefined` should pass through without modification.
*   **`it.concurrent('should handle complex nested structures', () => { ... })`**:
    *   **Purpose**: Combines multiple scenarios to test deep recursion and mixed data types.
    *   Includes arrays of objects and deeply nested objects.
    *   Assertions confirm redaction in `users[0].credentials.apiKey` and `config.database.password`, and that other fields like `username` and `host` are left intact.

---

### `describe('validateName')`

This section tests `validateName`, a function designed to clean up input strings to create valid "names" by removing invalid characters and normalizing spaces.

```typescript
describe('validateName', () => {
  it.concurrent('should remove invalid characters', () => {
    const result = validateName('test@#$%name')
    expect(result).toBe('testname')
  })

  // ... (other cases)

  it.concurrent('should collapse multiple spaces into single spaces', () => {
    const result = validateName('test    multiple     spaces')
    expect(result).toBe('test multiple spaces')
  })

  it.concurrent('should handle mixed whitespace and invalid characters', () => {
    const result = validateName('test@#$  name')
    expect(result).toBe('test name')
  })
})
```

*   **`describe('validateName', () => { ... })`**: Groups tests for name validation.
*   **`it.concurrent('should remove invalid characters', () => { ... })`**:
    *   **Purpose**: Tests basic removal of characters deemed invalid.
    *   `'test@#$%name'` should become `'testname'`.
*   **`it.concurrent('should keep valid characters', () => { ... })`**:
    *   **Purpose**: Ensures valid characters (alphanumeric, underscore) are preserved.
    *   `'test_name_123'` should remain `'test_name_123'`.
*   **`it.concurrent('should keep spaces', () => { ... })`**:
    *   **Purpose**: Confirms that spaces are considered valid and kept.
    *   `'test name'` should remain `'test name'`.
*   **`it.concurrent('should handle empty string', () => { ... })`**:
    *   **Purpose**: Edge case: an empty input should return an empty string.
*   **`it.concurrent('should handle string with only invalid characters', () => { ... })`**:
    *   **Purpose**: Edge case: if only invalid characters are present, the result should be an empty string.
*   **`it.concurrent('should handle mixed valid and invalid characters', () => { ... })`**:
    *   **Purpose**: Tests a combination of valid and invalid characters.
    *   `'my-workflow@2023!'` should become `'myworkflow2023'` (dashes, '@', and '!' are removed). This implies that only alphanumeric and spaces/underscores are allowed.
*   **`it.concurrent('should collapse multiple spaces into single spaces', () => { ... })`**:
    *   **Purpose**: Tests normalization of whitespace.
    *   Multiple consecutive spaces should be replaced by a single space.
*   **`it.concurrent('should handle mixed whitespace and invalid characters', () => { ... })`**:
    *   **Purpose**: Tests a combination of invalid character removal and space normalization.
    *   `'test@#$ name'` should remove `@#$` and collapse the spaces to a single one, resulting in `'test name'`.

---

### `describe('isValidName')`

This section tests `isValidName`, a function that checks if a given string consists *only* of allowed characters (alphanumeric, underscore, space). It returns `true` or `false`.

```typescript
describe('isValidName', () => {
  it.concurrent('should return true for valid names', () => {
    expect(isValidName('test_name')).toBe(true)
    expect(isValidName('test123')).toBe(true)
    expect(isValidName('test name')).toBe(true)
    expect(isValidName('TestName')).toBe(true)
    expect(isValidName('')).toBe(true)
  })

  it.concurrent('should return false for invalid names', () => {
    expect(isValidName('test@name')).toBe(false)
    expect(isValidName('test-name')).toBe(false)
    expect(isValidName('test#name')).toBe(false)
    expect(isValidName('test$name')).toBe(false)
    expect(isValidName('test%name')).toBe(false)
  })
})
```

*   **`describe('isValidName', () => { ... })`**: Groups tests for `isValidName`.
*   **`it.concurrent('should return true for valid names', () => { ... })`**:
    *   **Purpose**: Tests various inputs that should be considered valid.
    *   Includes alphanumeric, underscores, spaces, mixed cases, and even an empty string (which is often considered valid for a "name" input that might be optional).
*   **`it.concurrent('should return false for invalid names', () => { ... })`**:
    *   **Purpose**: Tests inputs containing characters that are explicitly *not* allowed.
    *   Examples include common symbols like `@`, `-`, `#`, `$`, `%`.

---

### `describe('getInvalidCharacters')`

This section tests `getInvalidCharacters`, a function that, given a string, identifies and returns an array of the unique characters that are *not* allowed based on the validation rules. This is useful for providing specific feedback to users about why their input is invalid.

```typescript
describe('getInvalidCharacters', () => {
  it.concurrent('should return empty array for valid names', () => {
    const result = getInvalidCharacters('test_name_123')
    expect(result).toEqual([])
  })

  it.concurrent('should return invalid characters', () => {
    const result = getInvalidCharacters('test@#$name')
    expect(result).toEqual(['@', '#', '$'])
  })

  it.concurrent('should return unique invalid characters', () => {
    const result = getInvalidCharacters('test@@##name')
    expect(result).toEqual(['@', '#'])
  })

  it.concurrent('should handle empty string', () => {
    const result = getInvalidCharacters('')
    expect(result).toEqual([])
  })

  it.concurrent('should handle string with only invalid characters', () => {
    const result = getInvalidCharacters('@#$%')
    expect(result).toEqual(['@', '#', '$', '%'])
  })
})
```

*   **`describe('getInvalidCharacters', () => { ... })`**: Groups tests for `getInvalidCharacters`.
*   **`it.concurrent('should return empty array for valid names', () => { ... })`**:
    *   **Purpose**: For a completely valid name, there should be no invalid characters.
    *   `expect(result).toEqual([])`: Asserts an empty array is returned.
*   **`it.concurrent('should return invalid characters', () => { ... })`**:
    *   **Purpose**: Identifies specific invalid characters.
    *   `'test@#$name'` should yield `['@', '#', '$']`.
*   **`it.concurrent('should return unique invalid characters', () => { ... })`**:
    *   **Purpose**: Checks that the function returns *unique* invalid characters, even if they appear multiple times in the input.
    *   `'test@@##name'` should yield `['@', '#']`, not `['@', '@', '#', '#']`.
*   **`it.concurrent('should handle empty string', () => { ... })`**:
    *   **Purpose**: Edge case: an empty string contains no characters, so no invalid ones either.
*   **`it.concurrent('should handle string with only invalid characters', () => { ... })`**:
    *   **Purpose**: Tests an input consisting entirely of invalid characters.
    *   `'@#$%'` should return all of them in the array.

---

This comprehensive test suite thoroughly covers the expected behavior of each utility function, providing confidence in their reliability and correctness across various scenarios and edge cases.