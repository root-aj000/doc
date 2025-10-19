This file is a Vitest test suite (`.spec.ts` or `.test.ts`), meaning its primary purpose is to **verify the correct behavior of several utility functions** defined in `@/tools/utils.ts`. It ensures that these functions handle various inputs, edge cases, and integrations (like environment variables or API calls) as expected.

The functions being tested are:
*   `transformTable`
*   `createCustomToolRequestBody`
*   `createParamSchema`
*   `executeRequest`
*   `formatRequestParams`
*   `getClientEnvVars`
*   `validateRequiredParametersAfterMerge`

Essentially, this file acts as a quality assurance check for core logic related to managing and executing "tools" within the application, which often involve making API requests, processing data, and handling user/LLM provided parameters.

---

### Simplifying Complex Logic

The complexity in this file arises from:

1.  **Testing Environment Setup:** Using Vitest for testing, which involves mocking external dependencies (`@/lib/logs/console/logger`, `@/stores/settings/environment/store`, `global.fetch`, `global.window`) to isolate the code under test and control its environment.
2.  **Tool Configuration Structure (`ToolConfig`):** The `ToolConfig` object (imported from `@/tools/types`) is central to many of these utilities. It's a rich object that describes a tool, including its parameters, how to construct an API request (`request.url`, `request.method`, `request.headers`, `request.body`), and how to process the response (`transformResponse`). Understanding this structure is key to understanding the tests.
3.  **Client vs. Server Environment:** Some functions (like `getClientEnvVars` and parts of `createCustomToolRequestBody`) behave differently depending on whether they are executing in a browser (client-side, `window` object exists) or a Node.js environment (server-side, `window` is undefined). The tests carefully simulate these environments.
4.  **Parameter Visibility (`visibility: 'user-only'` vs. `'user-or-llm'`):** The `validateRequiredParametersAfterMerge` function introduces the concept of parameter visibility. This is a subtle but important distinction:
    *   `'user-only'` parameters are expected to be provided by a human user (e.g., an API key they configure) and are typically validated *before* the tool is even considered for execution by an LLM.
    *   `'user-or-llm'` parameters are those the LLM (or a user) might provide as part of a specific tool call. This validation step focuses *only* on these, assuming `user-only` ones have been dealt with upstream.

In simple terms, this file is rigorously checking that different parts of a "tool execution pipeline" (from understanding tool definitions to making API calls and handling responses) work correctly and robustly under various conditions.

---

### Detailed Line-by-Line Explanation

```typescript
import { afterEach, beforeEach, describe, expect, it, vi } from 'vitest'
import type { ToolConfig } from '@/tools/types'
import {
  createCustomToolRequestBody,
  createParamSchema,
  executeRequest,
  formatRequestParams,
  getClientEnvVars,
  transformTable,
  validateRequiredParametersAfterMerge,
} from '@/tools/utils'
```
*   **`import { ..., vi } from 'vitest'`**: Imports testing utilities from the Vitest framework.
    *   `describe`: Groups related tests.
    *   `it` (or `test`): Defines an individual test case.
    *   `expect`: Used for making assertions (e.g., `expect(value).toEqual(expected)`).
    *   `beforeEach`: A hook that runs before each test in a `describe` block.
    *   `afterEach`: A hook that runs after each test in a `describe` block.
    *   `vi`: The Vitest mocking utility, similar to Jest's `jest`.
*   **`import type { ToolConfig } from '@/tools/types'`**: Imports the TypeScript type definition for `ToolConfig`. This is crucial for defining the structure of the tools being tested.
*   **`import { ... } from '@/tools/utils'`**: Imports the actual utility functions that are being tested in this file. These are the "units under test."

```typescript
vi.mock('@/lib/logs/console/logger', () => ({
  createLogger: vi.fn().mockReturnValue({
    debug: vi.fn(),
    info: vi.fn(),
    warn: vi.fn(),
    error: vi.fn(),
  }),
}))
```
*   **`vi.mock(...)`**: This tells Vitest to replace the actual module at `@/lib/logs/console/logger` with a mock implementation during these tests.
*   **`() => ({ ... })`**: The mock implementation.
*   **`createLogger: vi.fn().mockReturnValue(...)`**: Mocks the `createLogger` function. Instead of returning a real logger, it returns an object where each logging method (`debug`, `info`, `warn`, `error`) is also a Vitest mock function (`vi.fn()`).
    *   **Purpose:** This prevents actual logging output during tests, making test runs cleaner. It also allows tests to assert whether specific logging methods were called, and with what arguments, if needed (though not explicitly done in this snippet).

```typescript
vi.mock('@/stores/settings/environment/store', () => {
  const mockStore = {
    getAllVariables: vi.fn().mockReturnValue({
      API_KEY: { value: 'mock-api-key' },
      BASE_URL: { value: 'https://example.com' },
    }),
  }

  return {
    useEnvironmentStore: {
      getState: vi.fn().mockImplementation(() => mockStore),
    },
  }
})
```
*   **`vi.mock('@/stores/settings/environment/store', ...)`**: Mocks the environment store module.
*   **`const mockStore = { ... }`**: Defines a mock version of the store's internal state.
*   **`getAllVariables: vi.fn().mockReturnValue(...)`**: Mocks the `getAllVariables` method within the store. It's configured to return a predefined set of environment variables (`API_KEY` and `BASE_URL`) with mock values.
*   **`return { useEnvironmentStore: { getState: vi.fn().mockImplementation(() => mockStore) } }`**: This is the shape of the mock module. It mimics how the `useEnvironmentStore` hook would typically be used (e.g., `useEnvironmentStore.getState().getAllVariables()`). The `getState` method is mocked to consistently return the `mockStore` defined above.
    *   **Purpose:** This allows tests to control the environment variables that `getClientEnvVars` (and potentially other functions) will see, ensuring predictable behavior without relying on a real store or actual system environment variables.

```typescript
const originalWindow = global.window
beforeEach(() => {
  global.window = {} as any
})

afterEach(() => {
  global.window = originalWindow

  vi.clearAllMocks()
})
```
*   **`const originalWindow = global.window`**: Before any tests run, this line saves a reference to the actual `window` object (if it exists, e.g., in a browser-like JSDOM environment, or `undefined` in Node.js).
*   **`beforeEach(() => { global.window = {} as any })`**: This hook runs before *each* test. It overrides `global.window` with an empty object.
    *   **Purpose:** This simulates a Node.js (server-side) environment where `window` is typically not present, for tests that need to verify server-specific logic or ensure client-side code doesn't break in a server context. It's type-cast `as any` because `global.window` is normally `Window | undefined`.
*   **`afterEach(() => { ... })`**: This hook runs after *each* test.
    *   **`global.window = originalWindow`**: Restores the `global.window` object to its original state, ensuring that subsequent tests don't inherit the mocked `window` from previous tests.
    *   **`vi.clearAllMocks()`**: Resets the call history and state of all mock functions created with `vi.fn()` within the current test file.
        *   **Purpose:** Crucial for test isolation, preventing side effects from one test influencing another. For example, if a mock function was called in `test A`, `vi.clearAllMocks()` ensures its `toHaveBeenCalled()` count is reset to 0 before `test B` runs.

---

### `describe('transformTable', ...)`

This block tests the `transformTable` utility function, which is designed to convert a specific table-like data structure into a plain JavaScript object.

```typescript
describe('transformTable', () => {
  // ... tests for transformTable
})
```
*   **`describe('transformTable', () => { ... })`**: Groups all tests related to the `transformTable` function.

```typescript
  it.concurrent('should return empty object for null input', () => {
    const result = transformTable(null)
    expect(result).toEqual({})
  })
```
*   **`it.concurrent(...)`**: Defines an individual test case. `concurrent` allows Vitest to run this test in parallel with other `concurrent` tests in the same `describe` block (if configured).
*   **`const result = transformTable(null)`**: Calls the `transformTable` function with `null` as input.
*   **`expect(result).toEqual({})`**: Asserts that the function returns an empty object (`{}`) when given `null`, demonstrating robust error handling for invalid input.

```typescript
  it.concurrent('should transform table rows to key-value pairs', () => {
    const table = [
      { id: '1', cells: { Key: 'name', Value: 'John Doe' } },
      { id: '2', cells: { Key: 'age', Value: 30 } },
      { id: '3', cells: { Key: 'isActive', Value: true } },
      { id: '4', cells: { Key: 'data', Value: { foo: 'bar' } } },
    ]

    const result = transformTable(table)

    expect(result).toEqual({
      name: 'John Doe',
      age: 30,
      isActive: true,
      data: { foo: 'bar' },
    })
  })
```
*   **`const table = [...]`**: Defines a sample input array representing table rows. Each row has an `id` and a `cells` object, which contains `Key` and `Value` properties.
*   **`const result = transformTable(table)`**: Calls the function with the sample table.
*   **`expect(result).toEqual(...)`**: Asserts that the output is an object where the `Key` from each row becomes the object property name, and `Value` becomes its corresponding value. This tests the core functionality of the transformation.

```typescript
  it.concurrent('should skip rows without Key or Value properties', () => {
    const table: any = [
      { id: '1', cells: { Key: 'name', Value: 'John Doe' } },
      { id: '2', cells: { Key: 'age' } }, // Missing Value
      { id: '3', cells: { Value: true } }, // Missing Key
      { id: '4', cells: {} }, // Empty cells
    ]

    const result = transformTable(table)

    expect(result).toEqual({
      name: 'John Doe',
    })
  })
```
*   **`const table: any = [...]`**: Defines a table with some malformed rows (missing `Key` or `Value` in `cells`). The `any` type assertion is used to allow defining these incomplete objects for testing purposes, even if the `ToolConfig` type might expect them to be fully formed.
*   **`expect(result).toEqual({ name: 'John Doe' })`**: Asserts that only the well-formed row (`name: 'John Doe'`) is included in the output, demonstrating that the function correctly skips rows that lack both `Key` and `Value`.

```typescript
  it.concurrent('should handle Value=0 and Value=false correctly', () => {
    const table = [
      { id: '1', cells: { Key: 'count', Value: 0 } },
      { id: '2', cells: { Key: 'enabled', Value: false } },
    ]

    const result = transformTable(table)

    expect(result).toEqual({
      count: 0,
      enabled: false,
    })
  })
```
*   **`const table = [...]`**: Defines a table where `Value` properties are `0` and `false`.
*   **`expect(result).toEqual({ count: 0, enabled: false })`**: Asserts that these "falsy" values are correctly preserved in the output object, not mistakenly skipped or converted.

---

### `describe('formatRequestParams', ...)`

This block tests the `formatRequestParams` function, responsible for taking a `ToolConfig` and user-provided parameters, then translating them into a `RequestInit` object suitable for making an HTTP request (like `fetch`).

```typescript
describe('formatRequestParams', () => {
  let mockTool: ToolConfig

  beforeEach(() => {
    mockTool = {
      id: 'test-tool',
      name: 'Test Tool',
      description: 'A test tool',
      version: '1.0.0',
      params: {},
      request: {
        url: 'https://api.example.com',
        method: 'GET',
        headers: vi.fn().mockReturnValue({
          'Content-Type': 'application/json',
        }),
        body: vi.fn().mockReturnValue({ data: 'test-data' }),
      },
    }
  })
  // ... tests for formatRequestParams
})
```
*   **`let mockTool: ToolConfig`**: Declares a variable to hold a mock `ToolConfig` object.
*   **`beforeEach(() => { ... })`**: This hook runs before each test in this `describe` block.
    *   **`mockTool = { ... }`**: Initializes `mockTool` with a basic configuration.
        *   Crucially, `request.headers` and `request.body` are defined as `vi.fn().mockReturnValue(...)`. This means they are mock functions that, when called, will return the specified values. This setup allows tests to check if these functions were called with the correct arguments.
        *   **Purpose:** Ensures each test starts with a clean, consistent `mockTool` object, isolating test cases.

```typescript
  it.concurrent('should format request with static URL', () => {
    const params = { foo: 'bar' }
    const result = formatRequestParams(mockTool, params)

    expect(result).toEqual({
      url: 'https://api.example.com',
      method: 'GET',
      headers: { 'Content-Type': 'application/json' },
      body: undefined, // No body for GET
    })

    expect(mockTool.request.headers).toHaveBeenCalledWith(params)
  })
```
*   **`const params = { foo: 'bar' }`**: Defines some sample input parameters.
*   **`const result = formatRequestParams(mockTool, params)`**: Calls the function being tested.
*   **`expect(result).toEqual(...)`**: Asserts that the formatted request object matches the expected structure, using the static URL and method from `mockTool`. For a `GET` request, the `body` should be `undefined`.
*   **`expect(mockTool.request.headers).toHaveBeenCalledWith(params)`**: Asserts that the `headers` function defined in `mockTool.request` was called once, and it received the `params` as its argument. This verifies that dynamic header generation works.

```typescript
  it.concurrent('should format request with dynamic URL function', () => {
    mockTool.request.url = (params) => `https://api.example.com/${params.id}`
    const params = { id: '123' }

    const result = formatRequestParams(mockTool, params)

    expect(result).toEqual({
      url: 'https://api.example.com/123',
      method: 'GET',
      headers: { 'Content-Type': 'application/json' },
      body: undefined,
    })
  })
```
*   **`mockTool.request.url = (params) => ...`**: Overrides the `url` property of `mockTool.request` to be a function that dynamically constructs the URL based on `params`.
*   **`const params = { id: '123' }`**: Provides parameters that the dynamic URL function expects.
*   **`expect(result).toEqual(...)`**: Asserts that the `url` in the result is correctly constructed by calling the dynamic URL function with `params`.

```typescript
  it.concurrent('should use method from params over tool default', () => {
    const params = { method: 'POST' }
    const result = formatRequestParams(mockTool, params)

    expect(result.method).toBe('POST')
    expect(result.body).toBe(JSON.stringify({ data: 'test-data' }))
    expect(mockTool.request.body).toHaveBeenCalledWith(params)
  })
```
*   **`const params = { method: 'POST' }`**: Provides parameters that include a `method` property, overriding the `GET` method defined in `mockTool`.
*   **`expect(result.method).toBe('POST')`**: Asserts that the method from `params` takes precedence.
*   **`expect(result.body).toBe(JSON.stringify({ data: 'test-data' }))`**: Asserts that for a `POST` request (or any method that allows a body), the `body` property from `mockTool.request.body()` is used and JSON-stringified by default.
*   **`expect(mockTool.request.body).toHaveBeenCalledWith(params)`**: Asserts that the `body` function from `mockTool` was called with the provided `params`.

```typescript
  it.concurrent('should handle preformatted content types', () => {
    // Set Content-Type to a preformatted type
    mockTool.request.headers = vi.fn().mockReturnValue({
      'Content-Type': 'application/x-www-form-urlencoded',
    })

    // Return a preformatted body
    mockTool.request.body = vi.fn().mockReturnValue('key1=value1&key2=value2')

    const params = { method: 'POST' }
    const result = formatRequestParams(mockTool, params)

    expect(result.body).toBe('key1=value1&key2=value2')
  })
```
*   **`mockTool.request.headers = vi.fn().mockReturnValue(...)`**: Overrides the headers to specify `application/x-www-form-urlencoded`.
*   **`mockTool.request.body = vi.fn().mockReturnValue(...)`**: Overrides the body to return an already-formatted string.
*   **`expect(result.body).toBe('key1=value1&key2=value2')`**: Asserts that if the `Content-Type` is a "preformatted" type (like form URL-encoded data), the `body` is used directly as a string, without further JSON stringification.

```typescript
  it.concurrent('should handle NDJSON content type', () => {
    // Set Content-Type to NDJSON
    mockTool.request.headers = vi.fn().mockReturnValue({
      'Content-Type': 'application/x-ndjson',
    })

    // Return a preformatted body for NDJSON
    mockTool.request.body = vi.fn().mockReturnValue('{"prompt": "Hello"}\n{"prompt": "World"}')

    const params = { method: 'POST' }
    const result = formatRequestParams(mockTool, params)

    expect(result.body).toBe('{"prompt": "Hello"}\n{"prompt": "World"}')
  })
```
*   This test is similar to the previous one, but specifically for `application/x-ndjson` (Newline Delimited JSON). It verifies that `formatRequestParams` also treats NDJSON bodies as preformatted strings, not attempting to JSON.stringify them.

---

### `describe('validateRequiredParametersAfterMerge', ...)`

This block tests the `validateRequiredParametersAfterMerge` function, which checks if all *required* tool parameters (specifically those visible to users or LLMs) have been provided with non-empty values after any merging of parameters has occurred.

```typescript
describe('validateRequiredParametersAfterMerge', () => {
  let mockTool: ToolConfig

  beforeEach(() => {
    mockTool = {
      id: 'test-tool',
      name: 'Test Tool',
      description: 'A test tool',
      version: '1.0.0',
      params: {
        required1: {
          type: 'string',
          required: true,
          visibility: 'user-or-llm',
        },
        required2: {
          type: 'number',
          required: true,
          visibility: 'user-or-llm',
        },
        optional: {
          type: 'boolean',
        },
      },
      request: {
        url: 'https://api.example.com',
        method: 'GET',
        headers: () => ({}),
      },
    }
  })
  // ... tests for validateRequiredParametersAfterMerge
})
```
*   **`let mockTool: ToolConfig`**: Declares a variable for a mock `ToolConfig`.
*   **`beforeEach(() => { ... })`**: Initializes `mockTool` before each test in this block.
    *   **`params: { ... }`**: Defines several parameters:
        *   `required1`, `required2`: Marked as `required: true` and `visibility: 'user-or-llm'`. These are the focus of this validation function.
        *   `optional`: Has no `required` property (defaults to `false`), so it shouldn't be validated here.
    *   **Purpose:** Provides a consistent base `ToolConfig` for testing validation rules.

```typescript
  it.concurrent('should throw error for missing tool', () => {
    expect(() => {
      validateRequiredParametersAfterMerge('missing-tool', undefined, {})
    }).toThrow('Tool not found: missing-tool')
  })
```
*   **`expect(() => { ... }).toThrow(...)`**: This is how you test that a function throws an error. The function call is wrapped in an arrow function.
*   **`validateRequiredParametersAfterMerge('missing-tool', undefined, {})`**: Calls the function with an invalid tool ID and `undefined` for the tool object.
*   **`toThrow('Tool not found: missing-tool')`**: Asserts that a specific error message is thrown, confirming the function handles the case where the tool itself is not found.

```typescript
  it.concurrent('should throw error for missing required parameters', () => {
    expect(() => {
      validateRequiredParametersAfterMerge('test-tool', mockTool, {
        required1: 'value',
        // required2 is missing
      })
    }).toThrow('"Required2" is required for Test Tool')
  })
```
*   Calls the function with `mockTool` but intentionally omits `required2` from the provided parameters.
*   **`toThrow('"Required2" is required for Test Tool')`**: Asserts that an error is thrown, specifically mentioning the missing `Required2` parameter and the tool name.

```typescript
  it.concurrent('should not throw error when all required parameters are provided', () => {
    expect(() => {
      validateRequiredParametersAfterMerge('test-tool', mockTool, {
        required1: 'value',
        required2: 42,
      })
    }).not.toThrow()
  })
```
*   Calls the function with all required parameters (`required1`, `required2`) present.
*   **`.not.toThrow()`**: Asserts that no error is thrown, confirming successful validation.

```typescript
  it.concurrent('should not require optional parameters', () => {
    expect(() => {
      validateRequiredParametersAfterMerge('test-tool', mockTool, {
        required1: 'value',
        required2: 42,
        // optional parameter not provided
      })
    }).not.toThrow()
  })
```
*   Calls the function, providing only required parameters, and omitting the `optional` parameter.
*   **`.not.toThrow()`**: Asserts that no error is thrown, confirming that optional parameters are indeed not validated as required.

```typescript
  it.concurrent('should handle null and empty string values as missing', () => {
    expect(() => {
      validateRequiredParametersAfterMerge('test-tool', mockTool, {
        required1: null,
        required2: '',
      })
    }).toThrow('"Required1" is required for Test Tool')
  })
```
*   Calls the function with `null` and an empty string (`''`) for required parameters.
*   **`toThrow('"Required1" is required...')`**: Asserts that these are treated as missing values, and an error is thrown for the first one (`Required1`).

```typescript
  it.concurrent(
    'should not validate user-only parameters (they should be validated earlier)',
    () => {
      const toolWithUserOnlyParam = {
        ...mockTool,
        params: {
          ...mockTool.params,
          apiKey: {
            type: 'string' as const,
            required: true,
            visibility: 'user-only' as const, // This should NOT be validated here
          },
        },
      }

      // Should NOT throw for missing user-only params - they're validated at serialization
      expect(() => {
        validateRequiredParametersAfterMerge('test-tool', toolWithUserOnlyParam, {
          required1: 'value',
          required2: 42,
          // apiKey missing but it's user-only, so not validated here
        })
      }).not.toThrow()
    }
  )
```
*   **`const toolWithUserOnlyParam = { ...mockTool, params: { ..., apiKey: { ..., visibility: 'user-only' } } }`**: Creates a new tool configuration that includes an `apiKey` parameter marked as `required: true` but with `visibility: 'user-only'`.
*   **`expect(() => { ... }).not.toThrow()`**: Calls the function, omitting the `apiKey`. It asserts that *no error is thrown* for the missing `apiKey`.
    *   **Purpose:** This is a crucial test case. It confirms that `validateRequiredParametersAfterMerge` specifically ignores parameters with `visibility: 'user-only'`, as these are assumed to be handled (e.g., provided by a user in settings) at an earlier stage of the application workflow.

```typescript
  it.concurrent('should validate mixed user-or-llm and user-only parameters correctly', () => {
    const toolWithMixedParams = {
      ...mockTool,
      params: {
        userOrLlmParam: { /* ... */ visibility: 'user-or-llm' as const, required: true },
        userOnlyParam: { /* ... */ visibility: 'user-only' as const, required: true },
        optionalParam: { /* ... */ required: false, visibility: 'user-or-llm' as const },
      },
    }

    // Should throw for missing user-or-llm param, but not user-only param
    expect(() => {
      validateRequiredParametersAfterMerge('test-tool', toolWithMixedParams, {
        // userOrLlmParam missing - should cause error
        // userOnlyParam missing - should NOT cause error (validated earlier)
      })
    }).toThrow('"User Or Llm Param" is required for')
  })
```
*   Similar to the previous test, but with a mix of parameter visibilities.
*   It asserts that a missing `userOrLlmParam` (which is `required: true` and `visibility: 'user-or-llm'`) *does* cause an error, while a missing `userOnlyParam` (also `required: true` but `visibility: 'user-only'`) does not. This re-confirms the logic based on `visibility`.

```typescript
  it.concurrent('should use parameter description in error messages when available', () => {
    const toolWithDescriptions = {
      ...mockTool,
      params: {
        subreddit: { /* ... */ required: true, visibility: 'user-or-llm' as const, description: 'Subreddit name (without r/ prefix)' },
      },
    }

    expect(() => {
      validateRequiredParametersAfterMerge('test-tool', toolWithDescriptions, {})
    }).toThrow('"Subreddit" is required for Test Tool')
  })
```
*   Adds a `description` property to a required parameter.
*   Asserts that the error message uses the parameter's `description` (or rather, the parameter name inferred from the key, which might be derived from the description if it were a title) in the error message for better user clarity. *Correction:* The test actually asserts the *parameter name* ("Subreddit") is used, not the description text itself. The utility function likely uses a title-casing or human-readable version of the parameter key.

```typescript
  it.concurrent('should fall back to parameter name when no description available', () => {
    const toolWithoutDescription = {
      ...mockTool,
      params: {
        subreddit: { /* ... */ required: true, visibility: 'user-or-llm' as const, /* No description provided */ },
      },
    }

    expect(() => {
      validateRequiredParametersAfterMerge('test-tool', toolWithoutDescription, {})
    }).toThrow('"Subreddit" is required for Test Tool')
  })
```
*   Similar to the previous test, but explicitly without a `description`.
*   Asserts that the error message still correctly uses the parameter's key (`"Subreddit"`) when no `description` is available.

```typescript
  it.concurrent('should handle undefined values as missing', () => {
    expect(() => {
      validateRequiredParametersAfterMerge('test-tool', mockTool, {
        required1: 'value',
        required2: undefined, // Explicitly undefined
      })
    }).toThrow('"Required2" is required for Test Tool')
  })
```
*   Tests that an `undefined` value for a required parameter is treated the same as a missing parameter.

```typescript
  it.concurrent('should validate all missing parameters at once', () => {
    const toolWithMultipleRequired = { /* ... */ params: { param1: { /* ... */ required: true }, param2: { /* ... */ required: true } } }

    // Should throw for the first missing parameter it encounters
    expect(() => {
      validateRequiredParametersAfterMerge('test-tool', toolWithMultipleRequired, {})
    }).toThrow('"Param1" is required for Test Tool')
  })
```
*   Sets up a tool with two required parameters (`param1`, `param2`) and provides none.
*   Asserts that an error is thrown for the *first* missing parameter it encounters (`"Param1"`), demonstrating that the validation stops at the first failure.

---

### `describe('executeRequest', ...)`

This block tests the `executeRequest` function, which is responsible for making an actual HTTP request (using `fetch`) and processing its response, including error handling and custom response transformations.

```typescript
describe('executeRequest', () => {
  let mockTool: ToolConfig
  let mockFetch: any

  beforeEach(() => {
    mockFetch = vi.fn()
    global.fetch = mockFetch

    mockTool = {
      id: 'test-tool',
      name: 'Test Tool',
      description: 'A test tool',
      version: '1.0.0',
      params: {},
      request: {
        url: 'https://api.example.com',
        method: 'GET',
        headers: () => ({ 'Content-Type': 'application/json' }),
      },
      transformResponse: vi.fn(async (response) => ({
        success: true,
        output: await response.json(),
      })),
    }
  })

  afterEach(() => {
    vi.resetAllMocks()
  })
  // ... tests for executeRequest
})
```
*   **`let mockTool: ToolConfig`**, **`let mockFetch: any`**: Declares variables for the mock tool and the mocked `fetch` function.
*   **`beforeEach(() => { ... })`**: Setup before each test:
    *   **`mockFetch = vi.fn()`**: Creates a Vitest mock function for `fetch`.
    *   **`global.fetch = mockFetch`**: Overrides the global `fetch` function with the mock. This is essential for preventing actual network requests during tests.
    *   **`mockTool = { ... }`**: Initializes a mock `ToolConfig`.
        *   **`transformResponse: vi.fn(async (response) => (...))`**: The `transformResponse` property is also mocked. It's an async function that, by default, tries to parse the response as JSON and wraps it in a `{ success: true, output: ... }` structure.
    *   **Purpose:** Prepares a controlled environment for testing API calls by intercepting `fetch` and providing a typical `ToolConfig`.
*   **`afterEach(() => { vi.resetAllMocks() })`**: Resets all mock functions after each test, ensuring a clean slate. `resetAllMocks` specifically clears mock history and internal state without destroying the mock itself (like `clearAllMocks` does).

```typescript
  it('should handle successful requests', async () => {
    mockFetch.mockResolvedValueOnce({
      ok: true,
      status: 200,
      json: async () => ({ result: 'success' }),
    })

    const result = await executeRequest('test-tool', mockTool, {
      url: 'https://api.example.com',
      method: 'GET',
      headers: {},
    })

    expect(mockFetch).toHaveBeenCalledWith('https://api.example.com', {
      method: 'GET',
      headers: {},
      body: undefined,
    })
    expect(mockTool.transformResponse).toHaveBeenCalled()
    expect(result).toEqual({
      success: true,
      output: { result: 'success' },
    })
  })
```
*   **`it('should handle successful requests', async () => { ... })`**: An async test case, as `executeRequest` performs an async operation.
*   **`mockFetch.mockResolvedValueOnce(...)`**: Configures the `mockFetch` to return a successful `Response` object once. This mock response includes `ok: true`, `status: 200`, and a `json` method that returns `({ result: 'success' })`.
*   **`const result = await executeRequest(...)`**: Calls the function with a dummy request configuration.
*   **`expect(mockFetch).toHaveBeenCalledWith(...)`**: Asserts that `fetch` was called with the correct URL and request options.
*   **`expect(mockTool.transformResponse).toHaveBeenCalled()`**: Asserts that the `transformResponse` function was called (after `fetch` returned successfully).
*   **`expect(result).toEqual(...)`**: Asserts that the final result, after `transformResponse` has processed the data, matches the expected success output.

```typescript
  it.concurrent('should use default transform response if not provided', async () => {
    mockTool.transformResponse = undefined // Unset transformResponse

    mockFetch.mockResolvedValueOnce({
      ok: true,
      status: 200,
      json: async () => ({ result: 'success' }),
    })

    const result = await executeRequest('test-tool', mockTool, {
      url: 'https://api.example.com',
      method: 'GET',
      headers: {},
    })

    expect(result).toEqual({
      success: true,
      output: { result: 'success' },
    })
  })
```
*   **`mockTool.transformResponse = undefined`**: Explicitly removes the custom `transformResponse` from `mockTool`.
*   **`expect(result).toEqual(...)`**: Asserts that even without a custom `transformResponse`, the request is successful and the default handling (likely `response.json()`) is applied, returning the data wrapped in a success object.

```typescript
  it('should handle error responses', async () => {
    mockFetch.mockResolvedValueOnce({
      ok: false,
      status: 400,
      statusText: 'Bad Request',
      json: async () => ({ message: 'Invalid input' }),
    })

    const result = await executeRequest('test-tool', mockTool, {
      url: 'https://api.example.com',
      method: 'GET',
      headers: {},
    })

    expect(result).toEqual({
      success: false,
      output: {},
      error: 'Invalid input',
    })
  })
```
*   **`mockFetch.mockResolvedValueOnce({ ok: false, status: 400, ... })`**: Configures `mockFetch` to return an error response (e.g., HTTP 400).
*   **`expect(result).toEqual({ success: false, output: {}, error: 'Invalid input' })`**: Asserts that the `executeRequest` function correctly captures the error message from the JSON body of the error response and returns it in a `{ success: false, error: ... }` format.

```typescript
  it.concurrent('should handle network errors', async () => {
    const networkError = new Error('Network error')
    mockFetch.mockRejectedValueOnce(networkError)

    const result = await executeRequest('test-tool', mockTool, {
      url: 'https://api.example.com',
      method: 'GET',
      headers: {},
    })

    expect(result).toEqual({
      success: false,
      output: {},
      error: 'Network error',
    })
  })
```
*   **`mockFetch.mockRejectedValueOnce(networkError)`**: Configures `mockFetch` to *reject* with an error, simulating a network failure (e.g., no internet connection).
*   **`expect(result).toEqual({ success: false, output: {}, error: 'Network error' })`**: Asserts that the network error message is correctly captured and returned in the error object.

```typescript
  it('should handle JSON parse errors in error response', async () => {
    mockFetch.mockResolvedValueOnce({
      ok: false,
      status: 500,
      statusText: 'Server Error',
      json: async () => {
        throw new Error('Invalid JSON')
      },
    })

    const result = await executeRequest('test-tool', mockTool, {
      url: 'https://api.example.com',
      method: 'GET',
      headers: {},
    })

    expect(result).toEqual({
      success: false,
      output: {},
      error: 'Server Error', // Should use statusText in the error message
    })
  })
```
*   **`json: async () => { throw new Error('Invalid JSON') }`**: Simulates a scenario where the server returns an error, but its body is not valid JSON, causing `response.json()` to throw an error.
*   **`expect(result).toEqual({ ..., error: 'Server Error' })`**: Asserts that in such a case, the `statusText` from the HTTP response (`'Server Error'`) is used as the error message, providing a fallback when the JSON body cannot be parsed.

```typescript
  it('should handle transformResponse with non-JSON response', async () => {
    const toolWithTransform = {
      ...mockTool,
      transformResponse: async (response: Response) => {
        const xmlText = await response.text()
        return {
          success: true,
          output: {
            parsedData: 'mocked xml parsing result',
            originalXml: xmlText,
          },
        }
      },
    }

    mockFetch.mockResolvedValueOnce({
      ok: true,
      status: 200,
      statusText: 'OK',
      text: async () => '<xml><test>Mock XML response</test></xml>',
    })

    const result = await executeRequest('test-tool', toolWithTransform, {
      url: 'https://api.example.com',
      method: 'GET',
      headers: {},
    })

    expect(result).toEqual({
      success: true,
      output: {
        parsedData: 'mocked xml parsing result',
        originalXml: '<xml><test>Mock XML response</test></xml>',
      },
    })
  })
```
*   **`const toolWithTransform = { ..., transformResponse: async (response: Response) => { const xmlText = await response.text(); ... } }`**: Creates a tool with a `transformResponse` that explicitly reads the response body as plain text (`response.text()`), simulating handling of non-JSON responses like XML.
*   **`mockFetch.mockResolvedValueOnce({ ..., text: async () => '<xml>...</xml>' })`**: Configures `mockFetch` to return a response where the `text()` method provides an XML string.
*   **`expect(result).toEqual(...)`**: Asserts that the custom `transformResponse` function correctly processed the XML text, demonstrating flexibility beyond just JSON.

---

### `describe('createParamSchema', ...)`

This block tests the `createParamSchema` function, which converts a tool's internal schema (often derived from a custom tool definition, possibly in JSON schema format) into the `ToolConfig['params']` structure used by the application for validation and display.

```typescript
describe('createParamSchema', () => {
  // ... tests for createParamSchema
})
```
*   **`describe('createParamSchema', () => { ... })`**: Groups tests for `createParamSchema`.

```typescript
  it.concurrent('should create parameter schema from custom tool schema', () => {
    const customTool = {
      id: 'test-tool',
      title: 'Test Tool',
      schema: {
        function: {
          name: 'testFunc',
          description: 'A test function',
          parameters: {
            type: 'object',
            properties: {
              required1: { type: 'string', description: 'Required param' },
              optional1: { type: 'number', description: 'Optional param' },
            },
            required: ['required1'],
          },
        },
      },
    }

    const result = createParamSchema(customTool)

    expect(result).toEqual({
      required1: {
        type: 'string',
        required: true,
        visibility: 'user-or-llm',
        description: 'Required param',
      },
      optional1: {
        type: 'number',
        required: false,
        visibility: 'user-only', // Default visibility for optional params
        description: 'Optional param',
      },
    })
  })
```
*   **`const customTool = { ... }`**: Defines a mock "custom tool" object. It contains a `schema` property, which has a `function` property with `parameters` in a JSON Schema-like format (`properties`, `required` array).
*   **`const result = createParamSchema(customTool)`**: Calls the function being tested.
*   **`expect(result).toEqual(...)`**: Asserts that the output `params` object is correctly structured:
    *   `required1` has `required: true` because it's listed in `customTool.schema.function.parameters.required`.
    *   `optional1` has `required: false` because it's *not* in the `required` array.
    *   Both get their `type` and `description` from the `properties`.
    *   A default `visibility` is assigned: `user-or-llm` for required, `user-only` for optional. This might reflect a design decision where LLMs are only concerned with truly required parameters, while optional ones are more for direct user configuration.

```typescript
  it.concurrent('should handle empty or missing schema gracefully', () => {
    const emptyTool = {
      id: 'empty-tool',
      title: 'Empty Tool',
      schema: {},
    }

    const result = createParamSchema(emptyTool)

    expect(result).toEqual({})

    const missingPropsTool = {
      id: 'missing-props',
      title: 'Missing Props',
      schema: { function: { parameters: {} } },
    }

    const result2 = createParamSchema(missingPropsTool)
    expect(result2).toEqual({})
  })
```
*   These tests check robustness:
    *   `emptyTool`: Provides a tool with an empty `schema` object. Expects an empty result object.
    *   `missingPropsTool`: Provides a tool where the `parameters` object is empty. Expects an empty result object.
*   **Purpose:** Ensures the function doesn't crash and gracefully handles incomplete or empty schema definitions, returning an empty `params` object.

---

### `describe('getClientEnvVars', ...)`

This block tests the `getClientEnvVars` function, which retrieves environment variables *specifically for client-side execution* and depends on the presence of the `window` object.

```typescript
describe('getClientEnvVars', () => {
  // ... tests for getClientEnvVars
})
```
*   **`describe('getClientEnvVars', () => { ... })`**: Groups tests for `getClientEnvVars`.

```typescript
  it.concurrent('should return environment variables from store in browser environment', () => {
    // Note: global.window is set to {} in beforeEach, simulating a browser environment
    // where window exists but is empty.
    // The environment store is mocked globally.

    const mockStoreGetter = () => ({
      getAllVariables: () => ({
        API_KEY: { value: 'mock-api-key' },
        BASE_URL: { value: 'https://example.com' },
      }),
    })

    const result = getClientEnvVars(mockStoreGetter)

    expect(result).toEqual({
      API_KEY: 'mock-api-key',
      BASE_URL: 'https://example.com',
    })
  })
```
*   **`// Note: global.window is set to {} in beforeEach, simulating a browser environment`**: This comment reminds us that the `beforeEach` hook at the top of the file sets `global.window = {}`, which makes the `getClientEnvVars` function believe it's running in a browser environment.
*   **`const mockStoreGetter = () => ({ ... })`**: Provides an explicit mock for the store getter. While the store is globally mocked, passing this explicitly allows for fine-grained control if needed, but in this case, it essentially reflects the global mock.
*   **`const result = getClientEnvVars(mockStoreGetter)`**: Calls the function.
*   **`expect(result).toEqual(...)`**: Asserts that the function successfully retrieves the environment variables from the mocked store when `window` is present.

```typescript
  it.concurrent('should return empty object in server environment', () => {
    global.window = undefined as any // Explicitly set window to undefined for this test

    const result = getClientEnvVars()

    expect(result).toEqual({})
  })
```
*   **`global.window = undefined as any`**: *Overrides* the `beforeEach` hook's setting *just for this test*, explicitly setting `global.window` to `undefined`.
    *   **Purpose:** This simulates a Node.js (server-side) environment where the `window` object does not exist.
*   **`const result = getClientEnvVars()`**: Calls the function.
*   **`expect(result).toEqual({})`**: Asserts that in a server environment (where `window` is `undefined`), the function correctly returns an empty object, as client-side environment variables are not applicable or accessible there.

---

### `describe('createCustomToolRequestBody', ...)`

This block tests the `createCustomToolRequestBody` function, which generates a function that constructs the request body for executing a "custom tool." This body varies depending on whether the tool is executed client-side or server-side.

```typescript
describe('createCustomToolRequestBody', () => {
  // ... tests for createCustomToolRequestBody
})
```
*   **`describe('createCustomToolRequestBody', () => { ... })`**: Groups tests for `createCustomToolRequestBody`.

```typescript
  it.concurrent('should create request body function for client-side execution', () => {
    const customTool = {
      code: 'return a + b',
      schema: {
        function: {
          parameters: { type: 'object', properties: {} },
        },
      },
    }

    const mockStoreGetter = () => ({
      getAllVariables: () => ({
        API_KEY: { value: 'mock-api-key' },
        BASE_URL: { value: 'https://example.com' },
      }),
    })

    const bodyFn = createCustomToolRequestBody(customTool, true, undefined, mockStoreGetter)
    const result = bodyFn({ a: 5, b: 3 })

    expect(result).toEqual({
      code: 'return a + b',
      params: { a: 5, b: 3 },
      schema: { type: 'object', properties: {} },
      envVars: {
        API_KEY: 'mock-api-key',
        BASE_URL: 'https://example.com',
      },
      workflowId: undefined,
      workflowVariables: {},
      blockData: {},
      blockNameMapping: {},
      isCustomTool: true,
    })
  })
```
*   **`const customTool = { ... }`**: Defines a basic `customTool` with some `code` and a minimal `schema`.
*   **`const mockStoreGetter = () => ({ ... })`**: Provides a local mock for the environment store getter, ensuring controlled environment variables.
*   **`const bodyFn = createCustomToolRequestBody(customTool, true, undefined, mockStoreGetter)`**: Calls `createCustomToolRequestBody`.
    *   `customTool`: The tool definition.
    *   `true`: Indicates client-side execution.
    *   `undefined`: No `workflowId` for client-side.
    *   `mockStoreGetter`: The function to get client environment variables.
*   **`const result = bodyFn({ a: 5, b: 3 })`**: The function `createCustomToolRequestBody` returns *another function* (`bodyFn`). This line calls `bodyFn` with specific parameters (`{ a: 5, b: 3 }`) to generate the actual request body.
*   **`expect(result).toEqual(...)`**: Asserts that the generated request body includes:
    *   The `code` and `schema` from `customTool`.
    *   The `params` passed to `bodyFn`.
    *   `envVars` obtained from the `mockStoreGetter` (because `isClient` was `true`).
    *   `workflowId` as `undefined`.
    *   Default empty objects for other workflow-related properties.
    *   `isCustomTool: true`.
    *   **Purpose:** Verifies that the client-side request body correctly bundles the tool's details along with client-specific environment variables.

```typescript
  it.concurrent('should create request body function for server-side execution', () => {
    const customTool = {
      code: 'return a + b',
      schema: {
        function: {
          parameters: { type: 'object', properties: {} },
        },
      },
    }

    const workflowId = 'test-workflow-123'
    const bodyFn = createCustomToolRequestBody(customTool, false, workflowId)
    const result = bodyFn({ a: 5, b: 3 })

    expect(result).toEqual({
      code: 'return a + b',
      params: { a: 5, b: 3 },
      schema: { type: 'object', properties: {} },
      envVars: {}, // Empty for server-side
      workflowId: 'test-workflow-123',
      workflowVariables: {},
      blockData: {},
      blockNameMapping: {},
      isCustomTool: true,
    })
  })
```
*   **`const workflowId = 'test-workflow-123'`**: Defines a sample `workflowId`.
*   **`const bodyFn = createCustomToolRequestBody(customTool, false, workflowId)`**: Calls `createCustomToolRequestBody`.
    *   `false`: Indicates server-side execution.
    *   `workflowId`: Provided for server-side context.
*   **`const result = bodyFn({ a: 5, b: 3 })`**: Calls the generated `bodyFn`.
*   **`expect(result).toEqual(...)`**: Asserts that the generated request body:
    *   Includes `workflowId`.
    *   Has an empty `envVars` object (because `isClient` was `false`).
    *   Other properties are similar to the client-side test.
    *   **Purpose:** Verifies that the server-side request body correctly bundles tool details and workflow context, but omits client-specific environment variables.