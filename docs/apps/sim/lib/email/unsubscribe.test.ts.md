```typescript
import { describe, expect, it, vi } from 'vitest'
import type { EmailType } from '@/lib/email/mailer'
import {
  generateUnsubscribeToken,
  isTransactionalEmail,
  verifyUnsubscribeToken,
} from '@/lib/email/unsubscribe'

vi.mock('@/lib/env', () => ({
  env: {
    BETTER_AUTH_SECRET: 'test-secret-key',
  },
  isTruthy: (value: string | boolean | number | undefined) =>
    typeof value === 'string' ? value === 'true' || value === '1' : Boolean(value),
  getEnv: (variable: string) => process.env[variable],
}))

describe('unsubscribe utilities', () => {
  const testEmail = 'test@example.com'
  const testEmailType = 'marketing'

  describe('generateUnsubscribeToken', () => {
    it.concurrent('should generate a token with salt:hash:emailType format', () => {
      const token = generateUnsubscribeToken(testEmail, testEmailType)
      const parts = token.split(':')

      expect(parts).toHaveLength(3)
      expect(parts[0]).toHaveLength(32) // Salt should be 32 chars (16 bytes hex)
      expect(parts[1]).toHaveLength(64) // SHA256 hash should be 64 chars
      expect(parts[2]).toBe(testEmailType)
    })

    it.concurrent(
      'should generate different tokens for the same email (due to random salt)',
      () => {
        const token1 = generateUnsubscribeToken(testEmail, testEmailType)
        const token2 = generateUnsubscribeToken(testEmail, testEmailType)

        expect(token1).not.toBe(token2)
      }
    )

    it.concurrent('should default to marketing email type', () => {
      const token = generateUnsubscribeToken(testEmail)
      const parts = token.split(':')

      expect(parts[2]).toBe('marketing')
    })

    it.concurrent('should generate different tokens for different email types', () => {
      const marketingToken = generateUnsubscribeToken(testEmail, 'marketing')
      const updatesToken = generateUnsubscribeToken(testEmail, 'updates')

      expect(marketingToken).not.toBe(updatesToken)
    })
  })

  describe('verifyUnsubscribeToken', () => {
    it.concurrent('should verify a valid token', () => {
      const token = generateUnsubscribeToken(testEmail, testEmailType)
      const result = verifyUnsubscribeToken(testEmail, token)

      expect(result.valid).toBe(true)
      expect(result.emailType).toBe(testEmailType)
    })

    it.concurrent('should reject an invalid token', () => {
      const invalidToken = 'invalid:token:format'
      const result = verifyUnsubscribeToken(testEmail, invalidToken)

      expect(result.valid).toBe(false)
      expect(result.emailType).toBe('format')
    })

    it.concurrent('should reject a token for wrong email', () => {
      const token = generateUnsubscribeToken(testEmail, testEmailType)
      const result = verifyUnsubscribeToken('wrong@example.com', token)

      expect(result.valid).toBe(false)
    })

    it.concurrent('should handle legacy tokens (2 parts) and default to marketing', () => {
      // Generate a real legacy token using the actual hashing logic to ensure backward compatibility
      const salt = 'abc123'
      const secret = 'test-secret-key'
      const { createHash } = require('crypto')
      const hash = createHash('sha256').update(`${testEmail}:${salt}:${secret}`).digest('hex')
      const legacyToken = `${salt}:${hash}`

      // This should return valid since we're using the actual legacy format properly
      const result = verifyUnsubscribeToken(testEmail, legacyToken)
      expect(result.valid).toBe(true)
      expect(result.emailType).toBe('marketing') // Should default to marketing for legacy tokens
    })

    it.concurrent('should reject malformed tokens', () => {
      const malformedTokens = ['', 'single-part', 'too:many:parts:here:invalid', ':empty:parts:']

      malformedTokens.forEach((token) => {
        const result = verifyUnsubscribeToken(testEmail, token)
        expect(result.valid).toBe(false)
      })
    })
  })

  describe('isTransactionalEmail', () => {
    it.concurrent('should identify transactional emails correctly', () => {
      expect(isTransactionalEmail('transactional')).toBe(true)
    })

    it.concurrent('should identify non-transactional emails correctly', () => {
      const nonTransactionalTypes: EmailType[] = ['marketing', 'updates', 'notifications']

      nonTransactionalTypes.forEach((type) => {
        expect(isTransactionalEmail(type)).toBe(false)
      })
    })
  })
})
```

### Purpose of this file

This file contains unit tests for unsubscribe functionality related to email handling in a TypeScript application. It specifically tests three functions: `generateUnsubscribeToken`, `verifyUnsubscribeToken`, and `isTransactionalEmail`. The tests ensure that unsubscribe tokens are generated correctly, can be verified, and that the application can distinguish between transactional and non-transactional emails. It also covers backward compatibility for legacy tokens.

### Explanation of Code

1.  **Imports**:

    ```typescript
    import { describe, expect, it, vi } from 'vitest'
    import type { EmailType } from '@/lib/email/mailer'
    import {
      generateUnsubscribeToken,
      isTransactionalEmail,
      verifyUnsubscribeToken,
    } from '@/lib/email/unsubscribe'
    ```

    *   `vitest`: Imports testing utilities such as `describe` for grouping tests, `it` for individual test cases, `expect` for assertions, and `vi` for creating mocks.
    *   `EmailType`: Imports a type definition from `@/lib/email/mailer`, likely an enum or type that represents different types of emails (e.g., 'marketing', 'transactional').
    *   `generateUnsubscribeToken`, `isTransactionalEmail`, `verifyUnsubscribeToken`: Imports the functions to be tested from `@/lib/email/unsubscribe`. These functions are responsible for generating unsubscribe tokens, verifying them, and determining if an email is transactional.

2.  **Mocking the Environment**:

    ```typescript
    vi.mock('@/lib/env', () => ({
      env: {
        BETTER_AUTH_SECRET: 'test-secret-key',
      },
      isTruthy: (value: string | boolean | number | undefined) =>
        typeof value === 'string' ? value === 'true' || value === '1' : Boolean(value),
      getEnv: (variable: string) => process.env[variable],
    }))
    ```

    *   `vi.mock()`:  This is a Vitest function that mocks the `@/lib/env` module. Mocking allows you to isolate the code being tested and control its dependencies, ensuring consistent test results.
    *   The mock implementation defines a mock `env` object containing `BETTER_AUTH_SECRET`, which is essential for generating and verifying unsubscribe tokens. By mocking this, the tests don't rely on the actual environment variables being set.
    *   The mock implementation also provides `isTruthy` and `getEnv` functions. These are used within the test environment to simulate environment variable retrieval, which helps keep tests predictable and prevents them from depending on the actual environment.  The `isTruthy` function converts various types of inputs into a boolean.

3.  **Test Suite: `unsubscribe utilities`**:

    ```typescript
    describe('unsubscribe utilities', () => {
      // ... tests ...
    })
    ```

    *   `describe()`:  Defines a test suite for the unsubscribe utilities. It groups related tests together. The string argument provides a descriptive name for the suite.

4.  **Test Variables**:

    ```typescript
    const testEmail = 'test@example.com'
    const testEmailType = 'marketing'
    ```

    *   Defines constant variables to be used in multiple test cases.  This improves readability and reduces redundancy.  `testEmail` is a sample email address and `testEmailType` is set to `marketing`.

5.  **Test Suite: `generateUnsubscribeToken`**:

    ```typescript
    describe('generateUnsubscribeToken', () => {
      // ... tests ...
    })
    ```

    *   `describe()`: Defines a nested test suite specifically for the `generateUnsubscribeToken` function.

    *   `it.concurrent('should generate a token with salt:hash:emailType format', () => { ... })`: Tests that the generated token has the expected format `salt:hash:emailType`. It splits the token by the colon (`:`) and asserts that there are three parts, the first part (salt) has a length of 32 characters, the second part (hash) has a length of 64 characters, and the third part is the email type.

    *   `it.concurrent('should generate different tokens for the same email (due to random salt)', () => { ... })`: Tests that two tokens generated for the same email address and email type are different. This verifies that a random salt is used in the token generation process.

    *   `it.concurrent('should default to marketing email type', () => { ... })`: Tests that when no email type is provided, the function defaults to `marketing`.

    *   `it.concurrent('should generate different tokens for different email types', () => { ... })`: Tests that the tokens generated for the same email but with different email types are different.

6.  **Test Suite: `verifyUnsubscribeToken`**:

    ```typescript
    describe('verifyUnsubscribeToken', () => {
      // ... tests ...
    })
    ```

    *   `describe()`: Defines a test suite for the `verifyUnsubscribeToken` function.

    *   `it.concurrent('should verify a valid token', () => { ... })`: Tests that a valid token generated using `generateUnsubscribeToken` is correctly verified.  It asserts that the `valid` property of the result is `true` and the `emailType` matches the original email type.

    *   `it.concurrent('should reject an invalid token', () => { ... })`: Tests that an invalid token is rejected. It asserts that the `valid` property is `false`.  The `emailType` is checked to see if its returning the correct emailType from the malformed token.

    *   `it.concurrent('should reject a token for wrong email', () => { ... })`: Tests that a token generated for one email address is rejected when verified against a different email address.

    *   `it.concurrent('should handle legacy tokens (2 parts) and default to marketing', () => { ... })`: This test is crucial for maintaining backward compatibility. It simulates a legacy token (with only salt and hash) and verifies that it's still accepted, defaulting the email type to 'marketing'. This involves manually constructing a legacy token using the SHA256 hashing algorithm.

    *   `it.concurrent('should reject malformed tokens', () => { ... })`: Tests that various malformed tokens are rejected.  It iterates through an array of invalid tokens and asserts that `verifyUnsubscribeToken` returns `false` for each one.

7.  **Test Suite: `isTransactionalEmail`**:

    ```typescript
    describe('isTransactionalEmail', () => {
      // ... tests ...
    })
    ```

    *   `describe()`: Defines a test suite for the `isTransactionalEmail` function.

    *   `it.concurrent('should identify transactional emails correctly', () => { ... })`: Tests that `isTransactionalEmail` returns `true` for the 'transactional' email type.

    *   `it.concurrent('should identify non-transactional emails correctly', () => { ... })`: Tests that `isTransactionalEmail` returns `false` for non-transactional email types ('marketing', 'updates', 'notifications'). It iterates through an array of non-transactional types and asserts that the function returns `false` for each one.

### Summary

This file provides a comprehensive set of unit tests for email unsubscribe functionality. It covers token generation, verification, and email type identification. The tests ensure that the unsubscribe process is secure, reliable, and backward-compatible. The mocking of the environment and the clear, focused test cases make this a well-structured and maintainable test suite.
