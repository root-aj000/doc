```typescript
import { beforeEach, describe, expect, it, vi } from 'vitest'

// Mock the env module
vi.mock('@/lib/env', () => ({
  env: {
    FROM_EMAIL_ADDRESS: undefined,
    EMAIL_DOMAIN: undefined,
  },
}))

// Mock the getEmailDomain function
vi.mock('@/lib/urls/utils', () => ({
  getEmailDomain: vi.fn().mockReturnValue('fallback.com'),
}))

describe('getFromEmailAddress', () => {
  beforeEach(() => {
    // Reset mocks before each test
    vi.resetModules()
  })

  it('should return FROM_EMAIL_ADDRESS when set', async () => {
    // Mock env with FROM_EMAIL_ADDRESS
    vi.doMock('@/lib/env', () => ({
      env: {
        FROM_EMAIL_ADDRESS: 'Sim <noreply@sim.ai>',
        EMAIL_DOMAIN: 'example.com',
      },
    }))

    const { getFromEmailAddress } = await import('./utils')
    const result = getFromEmailAddress()

    expect(result).toBe('Sim <noreply@sim.ai>')
  })

  it('should return simple email format when FROM_EMAIL_ADDRESS is set without display name', async () => {
    vi.doMock('@/lib/env', () => ({
      env: {
        FROM_EMAIL_ADDRESS: 'noreply@sim.ai',
        EMAIL_DOMAIN: 'example.com',
      },
    }))

    const { getFromEmailAddress } = await import('./utils')
    const result = getFromEmailAddress()

    expect(result).toBe('noreply@sim.ai')
  })

  it('should return Azure ACS format when FROM_EMAIL_ADDRESS is set', async () => {
    vi.doMock('@/lib/env', () => ({
      env: {
        FROM_EMAIL_ADDRESS: 'DoNotReply@customer.azurecomm.net',
        EMAIL_DOMAIN: 'example.com',
      },
    }))

    const { getFromEmailAddress } = await import('./utils')
    const result = getFromEmailAddress()

    expect(result).toBe('DoNotReply@customer.azurecomm.net')
  })

  it('should construct from EMAIL_DOMAIN when FROM_EMAIL_ADDRESS is not set', async () => {
    vi.doMock('@/lib/env', () => ({
      env: {
        FROM_EMAIL_ADDRESS: undefined,
        EMAIL_DOMAIN: 'example.com',
      },
    }))

    const { getFromEmailAddress } = await import('./utils')
    const result = getFromEmailAddress()

    expect(result).toBe('noreply@example.com')
  })

  it('should use getEmailDomain fallback when both FROM_EMAIL_ADDRESS and EMAIL_DOMAIN are not set', async () => {
    vi.doMock('@/lib/env', () => ({
      env: {
        FROM_EMAIL_ADDRESS: undefined,
        EMAIL_DOMAIN: undefined,
      },
    }))

    const mockGetEmailDomain = vi.fn().mockReturnValue('fallback.com')
    vi.doMock('@/lib/urls/utils', () => ({
      getEmailDomain: mockGetEmailDomain,
    }))

    const { getFromEmailAddress } = await import('./utils')
    const result = getFromEmailAddress()

    expect(result).toBe('noreply@fallback.com')
    expect(mockGetEmailDomain).toHaveBeenCalled()
  })

  it('should prioritize FROM_EMAIL_ADDRESS over EMAIL_DOMAIN when both are set', async () => {
    vi.doMock('@/lib/env', () => ({
      env: {
        FROM_EMAIL_ADDRESS: 'Custom <custom@custom.com>',
        EMAIL_DOMAIN: 'ignored.com',
      },
    }))

    const { getFromEmailAddress } = await import('./utils')
    const result = getFromEmailAddress()

    expect(result).toBe('Custom <custom@custom.com>')
  })

  it('should handle empty string FROM_EMAIL_ADDRESS by falling back to EMAIL_DOMAIN', async () => {
    vi.doMock('@/lib/env', () => ({
      env: {
        FROM_EMAIL_ADDRESS: '',
        EMAIL_DOMAIN: 'fallback.com',
      },
    }))

    const { getFromEmailAddress } = await import('./utils')
    const result = getFromEmailAddress()

    expect(result).toBe('noreply@fallback.com')
  })

  it('should handle whitespace-only FROM_EMAIL_ADDRESS by falling back to EMAIL_DOMAIN', async () => {
    vi.doMock('@/lib/env', () => ({
      env: {
        FROM_EMAIL_ADDRESS: '   ',
        EMAIL_DOMAIN: 'fallback.com',
      },
    }))

    const { getFromEmailAddress } = await import('./utils')
    const result = getFromEmailAddress()

    expect(result).toBe('noreply@fallback.com')
  })
})
```

### Purpose of this file

This file contains unit tests for a function called `getFromEmailAddress`. This function, which presumably resides in a file named `utils.ts` in the same directory, is responsible for determining the "from" email address to use when sending emails. The tests verify that `getFromEmailAddress` correctly handles various scenarios, including:

- When a specific "from" email address is configured in environment variables (`FROM_EMAIL_ADDRESS`).
- When only the domain is configured (`EMAIL_DOMAIN`).
- When neither is configured, falling back to a default domain.
- Handling empty or whitespace-only `FROM_EMAIL_ADDRESS` values.

### Explanation

Let's break down the code line by line:

**Imports:**

```typescript
import { beforeEach, describe, expect, it, vi } from 'vitest'
```

- `vitest` is a testing framework, similar to Jest. This line imports several functions from it:
  - `describe`:  Used to group related tests together in a suite.
  - `it`: Defines an individual test case.  It's an alias for `test`.
  - `expect`:  Used to make assertions about the results of the code being tested.
  - `beforeEach`:  A hook that runs before each test case in the suite.  Useful for setup.
  - `vi`: The vitest object, which contains utilities for mocking dependencies and controlling the test environment.

**Mocks:**

```typescript
vi.mock('@/lib/env', () => ({
  env: {
    FROM_EMAIL_ADDRESS: undefined,
    EMAIL_DOMAIN: undefined,
  },
}))

vi.mock('@/lib/urls/utils', () => ({
  getEmailDomain: vi.fn().mockReturnValue('fallback.com'),
}))
```

- These lines are crucial for isolating the function under test. They use `vi.mock` to replace the actual implementations of two modules with mock versions:
  - `@/lib/env`:  This module likely provides access to environment variables. The mock provides an `env` object with `FROM_EMAIL_ADDRESS` and `EMAIL_DOMAIN` initially set to `undefined`. This ensures a clean slate before each test.
  - `@/lib/urls/utils`: This module is mocked to control the behavior of the `getEmailDomain` function.  `vi.fn().mockReturnValue('fallback.com')` creates a mock function that *always* returns `'fallback.com'`. This allows tests to verify fallback behavior when no environment variables are set, and to make sure that the function is being called correctly.

**Test Suite:**

```typescript
describe('getFromEmailAddress', () => { ... })
```

- `describe('getFromEmailAddress', () => { ... })` defines a test suite specifically for the `getFromEmailAddress` function. All the tests within this block are related to this function.

**`beforeEach` Hook:**

```typescript
beforeEach(() => {
  // Reset mocks before each test
  vi.resetModules()
})
```

- `beforeEach(() => { vi.resetModules() })` sets up a fresh environment before each individual test case. `vi.resetModules()` effectively clears all previously defined mocks and module imports. This is important to prevent state from one test from affecting subsequent tests, ensuring tests are isolated and repeatable.

**Test Cases (`it` blocks):**

Each `it` block represents a single test case. Let's examine one closely:

```typescript
it('should return FROM_EMAIL_ADDRESS when set', async () => {
  // Mock env with FROM_EMAIL_ADDRESS
  vi.doMock('@/lib/env', () => ({
    env: {
      FROM_EMAIL_ADDRESS: 'Sim <noreply@sim.ai>',
      EMAIL_DOMAIN: 'example.com',
    },
  }))

  const { getFromEmailAddress } = await import('./utils')
  const result = getFromEmailAddress()

  expect(result).toBe('Sim <noreply@sim.ai>')
})
```

1. **Test Description:** `it('should return FROM_EMAIL_ADDRESS when set', async () => { ... })` gives a descriptive name to the test, explaining what it's supposed to verify.

2. **Mocking (within the test):**
   ```typescript
   vi.doMock('@/lib/env', () => ({
     env: {
       FROM_EMAIL_ADDRESS: 'Sim <noreply@sim.ai>',
       EMAIL_DOMAIN: 'example.com',
     },
   }))
   ```
   - `vi.doMock` *overrides* the initial mock defined at the top of the file. This is how each test sets up its *specific* environment.
   - In this case, it mocks the `@/lib/env` module to provide a specific value for `FROM_EMAIL_ADDRESS`. The value `'Sim <noreply@sim.ai>'`  is configured and `EMAIL_DOMAIN` is also configured to 'example.com' although this test expects the `FROM_EMAIL_ADDRESS` value to be used, so this value is irrelevant.

3. **Importing the function under test:**
   ```typescript
   const { getFromEmailAddress } = await import('./utils')
   ```
   -  This line dynamically imports the `getFromEmailAddress` function from the `utils.ts` file in the same directory.  The `await` keyword is necessary because dynamic imports are asynchronous. Note that because of the `vi.doMock` above, when the `@/lib/env` module is imported *within* the `utils.ts` file, it will receive the *mocked* version, not the real one.

4. **Calling the function:**
   ```typescript
   const result = getFromEmailAddress()
   ```
   - This line calls the function being tested and stores the result in the `result` variable.

5. **Assertion:**
   ```typescript
   expect(result).toBe('Sim <noreply@sim.ai>')
   ```
   - `expect(result).toBe('Sim <noreply@sim.ai>')` is the assertion that verifies the behavior of the function.  It checks if the value returned by `getFromEmailAddress` is equal to the expected value `'Sim <noreply@sim.ai>'`. If the values don't match, the test fails.

**Other Test Cases:**

The other `it` blocks follow a similar pattern:

- They set up specific mock environments using `vi.doMock`.
- They import the `getFromEmailAddress` function.
- They call the function.
- They use `expect` to assert that the returned value is correct for the given environment.

**Key Concepts and Simplifications:**

- **Mocking:** Mocking is essential for unit testing. It allows you to isolate the code you're testing by replacing its dependencies with controlled substitutes (mocks).  This makes tests more predictable and prevents external factors (like network requests or database connections) from influencing the test results.  `vi.mock` and `vi.doMock` are key to this.
- **`vi.fn()`:** Creates a mock function.  This is useful when you need to track whether a function was called, how many times it was called, and with what arguments.
- **`vi.mockReturnValue()`:** Configures a mock function to return a specific value.
- **Dynamic Imports:**  `await import('./utils')` is used to import the `utils` module *after* the mocks have been set up.  This is crucial because the mocks need to be in place *before* the module is loaded.
- **Test Isolation:**  The `beforeEach` hook and the use of `vi.doMock` ensure that each test case runs in a clean, isolated environment.  This is crucial for reliable and repeatable test results.
- **Asynchronous Testing:** The `async` keyword and `await` keyword are used because the module import is asynchronous. This is necessary to ensure that the module is fully loaded before the test runs.

In summary, this file provides a comprehensive suite of unit tests for the `getFromEmailAddress` function. It uses mocking extensively to isolate the function under test and verify its behavior in various scenarios. The tests are well-structured and easy to understand, making it easy to maintain and extend them as needed.
