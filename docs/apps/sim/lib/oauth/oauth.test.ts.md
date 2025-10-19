```typescript
import { afterEach, beforeEach, describe, expect, it, type Mock, vi } from 'vitest'

vi.mock('@/lib/env', () => ({
  env: {
    GOOGLE_CLIENT_ID: 'google_client_id',
    GOOGLE_CLIENT_SECRET: 'google_client_secret',
    GITHUB_CLIENT_ID: 'github_client_id',
    GITHUB_CLIENT_SECRET: 'github_client_secret',
    X_CLIENT_ID: 'x_client_id',
    X_CLIENT_SECRET: 'x_client_secret',
    CONFLUENCE_CLIENT_ID: 'confluence_client_id',
    CONFLUENCE_CLIENT_SECRET: 'confluence_client_secret',
    JIRA_CLIENT_ID: 'jira_client_id',
    JIRA_CLIENT_SECRET: 'jira_client_secret',
    AIRTABLE_CLIENT_ID: 'airtable_client_id',
    AIRTABLE_CLIENT_SECRET: 'airtable_client_secret',
    SUPABASE_CLIENT_ID: 'supabase_client_id',
    SUPABASE_CLIENT_SECRET: 'supabase_client_secret',
    NOTION_CLIENT_ID: 'notion_client_id',
    NOTION_CLIENT_SECRET: 'notion_client_secret',
    DISCORD_CLIENT_ID: 'discord_client_id',
    DISCORD_CLIENT_SECRET: 'discord_client_secret',
    MICROSOFT_CLIENT_ID: 'microsoft_client_id',
    MICROSOFT_CLIENT_SECRET: 'microsoft_client_secret',
    LINEAR_CLIENT_ID: 'linear_client_id',
    LINEAR_CLIENT_SECRET: 'linear_client_secret',
    SLACK_CLIENT_ID: 'slack_client_id',
    SLACK_CLIENT_SECRET: 'slack_client_secret',
    REDDIT_CLIENT_ID: 'reddit_client_id',
    REDDIT_CLIENT_SECRET: 'reddit_client_secret',
  },
}))

vi.mock('@/lib/logs/console/logger', () => ({
  createLogger: vi.fn().mockReturnValue({
    info: vi.fn(),
    error: vi.fn(),
    warn: vi.fn(),
    debug: vi.fn(),
  }),
}))

const mockFetch = vi.fn()
global.fetch = mockFetch

import { refreshOAuthToken } from '@/lib/oauth/oauth'

describe('OAuth Token Refresh', () => {
  beforeEach(() => {
    vi.clearAllMocks()
    mockFetch.mockResolvedValue({
      ok: true,
      json: async () => ({
        access_token: 'new_access_token',
        expires_in: 3600,
        refresh_token: 'new_refresh_token',
      }),
    })
  })

  afterEach(() => {
    vi.clearAllMocks()
  })

  describe('Basic Auth Providers', () => {
    const basicAuthProviders = [
      {
        name: 'Airtable',
        providerId: 'airtable',
        endpoint: 'https://airtable.com/oauth2/v1/token',
      },
      { name: 'X (Twitter)', providerId: 'x', endpoint: 'https://api.x.com/2/oauth2/token' },
      {
        name: 'Confluence',
        providerId: 'confluence',
        endpoint: 'https://auth.atlassian.com/oauth/token',
      },
      { name: 'Jira', providerId: 'jira', endpoint: 'https://auth.atlassian.com/oauth/token' },
      {
        name: 'Discord',
        providerId: 'discord',
        endpoint: 'https://discord.com/api/v10/oauth2/token',
      },
      { name: 'Linear', providerId: 'linear', endpoint: 'https://api.linear.app/oauth/token' },
      {
        name: 'Reddit',
        providerId: 'reddit',
        endpoint: 'https://www.reddit.com/api/v1/access_token',
      },
    ]

    basicAuthProviders.forEach(({ name, providerId, endpoint }) => {
      it(`should send ${name} request with Basic Auth header and no credentials in body`, async () => {
        const refreshToken = 'test_refresh_token'

        await refreshOAuthToken(providerId, refreshToken)

        expect(mockFetch).toHaveBeenCalledWith(
          endpoint,
          expect.objectContaining({
            method: 'POST',
            headers: expect.objectContaining({
              'Content-Type': 'application/x-www-form-urlencoded',
              Authorization: expect.stringMatching(/^Basic /),
            }),
            body: expect.any(String),
          })
        )

        const [, requestOptions] = (mockFetch as Mock).mock.calls[0]

        // Verify Basic Auth header
        const authHeader = requestOptions.headers.Authorization
        expect(authHeader).toMatch(/^Basic /)

        // Decode and verify credentials
        const base64Credentials = authHeader.replace('Basic ', '')
        const credentials = Buffer.from(base64Credentials, 'base64').toString('utf-8')
        const [clientId, clientSecret] = credentials.split(':')

        expect(clientId).toBe(`${providerId}_client_id`)
        expect(clientSecret).toBe(`${providerId}_client_secret`)

        // Verify body contains only required parameters
        const bodyParams = new URLSearchParams(requestOptions.body)
        const bodyKeys = Array.from(bodyParams.keys())

        expect(bodyKeys).toEqual(['grant_type', 'refresh_token'])
        expect(bodyParams.get('grant_type')).toBe('refresh_token')
        expect(bodyParams.get('refresh_token')).toBe(refreshToken)

        // Verify client credentials are NOT in the body
        expect(bodyParams.get('client_id')).toBeNull()
        expect(bodyParams.get('client_secret')).toBeNull()
      })
    })
  })

  describe('Body Credential Providers', () => {
    const bodyCredentialProviders = [
      { name: 'Google', providerId: 'google', endpoint: 'https://oauth2.googleapis.com/token' },
      {
        name: 'GitHub',
        providerId: 'github',
        endpoint: 'https://github.com/login/oauth/access_token',
      },
      {
        name: 'Microsoft',
        providerId: 'microsoft',
        endpoint: 'https://login.microsoftonline.com/common/oauth2/v2.0/token',
      },
      {
        name: 'Outlook',
        providerId: 'outlook',
        endpoint: 'https://login.microsoftonline.com/common/oauth2/v2.0/token',
      },
      {
        name: 'Supabase',
        providerId: 'supabase',
        endpoint: 'https://api.supabase.com/v1/oauth/token',
      },
      { name: 'Notion', providerId: 'notion', endpoint: 'https://api.notion.com/v1/oauth/token' },
      { name: 'Slack', providerId: 'slack', endpoint: 'https://slack.com/api/oauth.v2.access' },
    ]

    bodyCredentialProviders.forEach(({ name, providerId, endpoint }) => {
      it(`should send ${name} request with credentials in body and no Basic Auth`, async () => {
        const refreshToken = 'test_refresh_token'

        await refreshOAuthToken(providerId, refreshToken)

        expect(mockFetch).toHaveBeenCalledWith(
          endpoint,
          expect.objectContaining({
            method: 'POST',
            headers: expect.objectContaining({
              'Content-Type': 'application/x-www-form-urlencoded',
            }),
            body: expect.any(String),
          })
        )

        const [, requestOptions] = (mockFetch as Mock).mock.calls[0]

        // Verify no Basic Auth header
        expect(requestOptions.headers.Authorization).toBeUndefined()

        // Verify body contains all required parameters
        const bodyParams = new URLSearchParams(requestOptions.body)
        const bodyKeys = Array.from(bodyParams.keys()).sort()

        expect(bodyKeys).toEqual(['client_id', 'client_secret', 'grant_type', 'refresh_token'])
        expect(bodyParams.get('grant_type')).toBe('refresh_token')
        expect(bodyParams.get('refresh_token')).toBe(refreshToken)

        // Verify client credentials are in the body
        const expectedClientId =
          providerId === 'outlook' ? 'microsoft_client_id' : `${providerId}_client_id`
        const expectedClientSecret =
          providerId === 'outlook' ? 'microsoft_client_secret' : `${providerId}_client_secret`

        expect(bodyParams.get('client_id')).toBe(expectedClientId)
        expect(bodyParams.get('client_secret')).toBe(expectedClientSecret)
      })
    })

    it('should include Accept header for GitHub requests', async () => {
      const refreshToken = 'test_refresh_token'

      await refreshOAuthToken('github', refreshToken)

      const [, requestOptions] = (mockFetch as Mock).mock.calls[0]
      expect(requestOptions.headers.Accept).toBe('application/json')
    })
  })

  describe('Error Handling', () => {
    it('should return null for unsupported provider', async () => {
      const refreshToken = 'test_refresh_token'

      const result = await refreshOAuthToken('unsupported', refreshToken)

      expect(result).toBeNull()
    })

    it('should return null for API error responses', async () => {
      const refreshToken = 'test_refresh_token'

      mockFetch.mockResolvedValueOnce({
        ok: false,
        status: 400,
        text: async () =>
          JSON.stringify({
            error: 'invalid_request',
            error_description: 'Invalid refresh token',
          }),
      })

      const result = await refreshOAuthToken('google', refreshToken)

      expect(result).toBeNull()
    })

    it('should return null for network errors', async () => {
      const refreshToken = 'test_refresh_token'

      mockFetch.mockRejectedValueOnce(new Error('Network error'))

      const result = await refreshOAuthToken('google', refreshToken)

      expect(result).toBeNull()
    })
  })

  describe('Token Response Handling', () => {
    it('should handle providers that return new refresh tokens', async () => {
      const refreshToken = 'old_refresh_token'
      const newRefreshToken = 'new_refresh_token'

      mockFetch.mockResolvedValueOnce({
        ok: true,
        json: async () => ({
          access_token: 'new_access_token',
          expires_in: 3600,
          refresh_token: newRefreshToken,
        }),
      })

      const result = await refreshOAuthToken('airtable', refreshToken)

      expect(result).toEqual({
        accessToken: 'new_access_token',
        expiresIn: 3600,
        refreshToken: newRefreshToken,
      })
    })

    it('should use original refresh token when new one is not provided', async () => {
      const refreshToken = 'original_refresh_token'

      mockFetch.mockResolvedValueOnce({
        ok: true,
        json: async () => ({
          access_token: 'new_access_token',
          expires_in: 3600,
          // No refresh_token in response
        }),
      })

      const result = await refreshOAuthToken('google', refreshToken)

      expect(result).toEqual({
        accessToken: 'new_access_token',
        expiresIn: 3600,
        refreshToken: refreshToken,
      })
    })

    it('should return null when access token is missing', async () => {
      const refreshToken = 'test_refresh_token'

      mockFetch.mockResolvedValueOnce({
        ok: true,
        json: async () => ({
          expires_in: 3600,
          // No access_token in response
        }),
      })

      const result = await refreshOAuthToken('google', refreshToken)

      expect(result).toBeNull()
    })

    it('should use default expiration when not provided', async () => {
      const refreshToken = 'test_refresh_token'

      mockFetch.mockResolvedValueOnce({
        ok: true,
        json: async () => ({
          access_token: 'new_access_token',
          // No expires_in in response
        }),
      })

      const result = await refreshOAuthToken('google', refreshToken)

      expect(result).toEqual({
        accessToken: 'new_access_token',
        expiresIn: 3600,
        refreshToken: refreshToken,
      })
    })
  })

  describe('Airtable Tests', () => {
    it('should not have duplicate client ID issue', async () => {
      const refreshToken = 'test_refresh_token'

      await refreshOAuthToken('airtable', refreshToken)

      const [, requestOptions] = (mockFetch as Mock).mock.calls[0]

      // Verify Authorization header is present and correct
      expect(requestOptions.headers.Authorization).toMatch(/^Basic /)

      // Parse body and verify client credentials are NOT present
      const bodyParams = new URLSearchParams(requestOptions.body)
      expect(bodyParams.get('client_id')).toBeNull()
      expect(bodyParams.get('client_secret')).toBeNull()

      // Verify only expected parameters are present
      const bodyKeys = Array.from(bodyParams.keys())
      expect(bodyKeys).toEqual(['grant_type', 'refresh_token'])
    })

    it('should handle Airtable refresh token rotation', async () => {
      const refreshToken = 'old_refresh_token'
      const newRefreshToken = 'rotated_refresh_token'

      mockFetch.mockResolvedValueOnce({
        ok: true,
        json: async () => ({
          access_token: 'new_access_token',
          expires_in: 3600,
          refresh_token: newRefreshToken,
        }),
      })

      const result = await refreshOAuthToken('airtable', refreshToken)

      expect(result?.refreshToken).toBe(newRefreshToken)
    })
  })
})
```

## Explanation of the Code

This TypeScript file contains unit tests for the `refreshOAuthToken` function, which is responsible for refreshing OAuth tokens for various providers like Google, GitHub, Airtable, etc. The tests are written using Vitest, a modern testing framework.

**1. Imports and Mocking:**

```typescript
import { afterEach, beforeEach, describe, expect, it, type Mock, vi } from 'vitest'

vi.mock('@/lib/env', () => ({
  env: {
    GOOGLE_CLIENT_ID: 'google_client_id',
    GOOGLE_CLIENT_SECRET: 'google_client_secret',
    GITHUB_CLIENT_ID: 'github_client_id',
    GITHUB_CLIENT_SECRET: 'github_client_secret',
    X_CLIENT_ID: 'x_client_id',
    X_CLIENT_SECRET: 'x_client_secret',
    CONFLUENCE_CLIENT_ID: 'confluence_client_id',
    CONFLUENCE_CLIENT_SECRET: 'confluence_client_secret',
    JIRA_CLIENT_ID: 'jira_client_id',
    JIRA_CLIENT_SECRET: 'jira_client_secret',
    AIRTABLE_CLIENT_ID: 'airtable_client_id',
    AIRTABLE_CLIENT_SECRET: 'airtable_client_secret',
    SUPABASE_CLIENT_ID: 'supabase_client_id',
    SUPABASE_CLIENT_SECRET: 'supabase_client_secret',
    NOTION_CLIENT_ID: 'notion_client_id',
    NOTION_CLIENT_SECRET: 'notion_client_secret',
    DISCORD_CLIENT_ID: 'discord_client_id',
    DISCORD_CLIENT_SECRET: 'discord_client_secret',
    MICROSOFT_CLIENT_ID: 'microsoft_client_id',
    MICROSOFT_CLIENT_SECRET: 'microsoft_client_secret',
    LINEAR_CLIENT_ID: 'linear_client_id',
    LINEAR_CLIENT_SECRET: 'linear_client_secret',
    SLACK_CLIENT_ID: 'slack_client_id',
    SLACK_CLIENT_SECRET: 'slack_client_secret',
    REDDIT_CLIENT_ID: 'reddit_client_id',
    REDDIT_CLIENT_SECRET: 'reddit_client_secret',
  },
}))

vi.mock('@/lib/logs/console/logger', () => ({
  createLogger: vi.fn().mockReturnValue({
    info: vi.fn(),
    error: vi.fn(),
    warn: vi.fn(),
    debug: vi.fn(),
  }),
}))

const mockFetch = vi.fn()
global.fetch = mockFetch

import { refreshOAuthToken } from '@/lib/oauth/oauth'
```

*   **Imports:** This section imports necessary functions and types from Vitest. `describe`, `it`, `expect`, `beforeEach`, `afterEach`, `vi`, and `Mock` are all part of the Vitest API.
*   **`vi.mock('@/lib/env', ...)`:** This line mocks the `@/lib/env` module.  This is crucial because the `refreshOAuthToken` function likely uses environment variables (like client IDs and secrets) to make the API calls.  Instead of relying on actual environment variables during testing (which could be inconsistent or unavailable), this mock provides fixed values. The function defines a mock `env` object with the required client ID and secret keys for all providers being tested.  The keys are in the format `<PROVIDER>_CLIENT_ID` and `<PROVIDER>_CLIENT_SECRET`.
*   **`vi.mock('@/lib/logs/console/logger', ...)`:** This line mocks the logging module.  The `createLogger` function is replaced with a mock that returns an object with empty `info`, `error`, `warn`, and `debug` functions. This prevents the tests from generating console output.
*   **`const mockFetch = vi.fn()`:** This line creates a mock function for the global `fetch` API. The `fetch` API is used to make network requests. By mocking it, the tests can control the responses and verify that the correct requests are being made.
*   **`global.fetch = mockFetch`:** This line replaces the actual `fetch` API with the mocked version. This ensures that the `refreshOAuthToken` function uses the mock `fetch` during testing.
*   **`import { refreshOAuthToken } from '@/lib/oauth/oauth'`:**  This imports the function that is being tested.

**2. Test Suite Setup:**

```typescript
describe('OAuth Token Refresh', () => {
  beforeEach(() => {
    vi.clearAllMocks()
    mockFetch.mockResolvedValue({
      ok: true,
      json: async () => ({
        access_token: 'new_access_token',
        expires_in: 3600,
        refresh_token: 'new_refresh_token',
      }),
    })
  })

  afterEach(() => {
    vi.clearAllMocks()
  })

  // ... rest of the tests
})
```

*   **`describe('OAuth Token Refresh', ...)`:** This defines a test suite for the OAuth token refresh functionality.  A test suite groups related tests together.
*   **`beforeEach(() => { ... })`:** This function is executed before each test within the `describe` block.  It performs the following actions:
    *   **`vi.clearAllMocks()`:** This resets all mock function calls, ensuring that each test starts with a clean slate.  It clears the history of calls to mocked functions.
    *   **`mockFetch.mockResolvedValue(...)`:** This sets up a default successful response for the `mockFetch` function.  By default, every call to `fetch` will return a promise that resolves to a successful response (`ok: true`) with a JSON body containing a new access token, expiration time, and refresh token. This simplifies the tests by providing a default response that can be overridden if needed.
*   **`afterEach(() => { ... })`:** This function is executed after each test.
    *   **`vi.clearAllMocks()`:**  This resets all mock function calls again, cleaning up after each test and preventing interference between tests.

**3. Test Groups: Basic Auth Providers and Body Credential Providers**

The tests are structured into two main groups, based on how the OAuth provider expects the client ID and secret to be sent:

*   **Basic Auth Providers:** These providers expect the client ID and secret to be encoded in the `Authorization` header using Basic Authentication.
*   **Body Credential Providers:** These providers expect the client ID and secret to be included in the request body, along with the other parameters.

**4. Testing Basic Auth Providers:**

```typescript
 describe('Basic Auth Providers', () => {
    const basicAuthProviders = [
      {
        name: 'Airtable',
        providerId: 'airtable',
        endpoint: 'https://airtable.com/oauth2/v1/token',
      },
      { name: 'X (Twitter)', providerId: 'x', endpoint: 'https://api.x.com/2/oauth2/token' },
      {
        name: 'Confluence',
        providerId: 'confluence',
        endpoint: 'https://auth.atlassian.com/oauth/token',
      },
      { name: 'Jira', providerId: 'jira', endpoint: 'https://auth.atlassian.com/oauth/token' },
      {
        name: 'Discord',
        providerId: 'discord',
        endpoint: 'https://discord.com/api/v10/oauth2/token',
      },
      { name: 'Linear', providerId: 'linear', endpoint: 'https://api.linear.app/oauth/token' },
      {
        name: 'Reddit',
        providerId: 'reddit',
        endpoint: 'https://www.reddit.com/api/v1/access_token',
      },
    ]

    basicAuthProviders.forEach(({ name, providerId, endpoint }) => {
      it(`should send ${name} request with Basic Auth header and no credentials in body`, async () => {
        const refreshToken = 'test_refresh_token'

        await refreshOAuthToken(providerId, refreshToken)

        expect(mockFetch).toHaveBeenCalledWith(
          endpoint,
          expect.objectContaining({
            method: 'POST',
            headers: expect.objectContaining({
              'Content-Type': 'application/x-www-form-urlencoded',
              Authorization: expect.stringMatching(/^Basic /),
            }),
            body: expect.any(String),
          })
        )

        const [, requestOptions] = (mockFetch as Mock).mock.calls[0]

        // Verify Basic Auth header
        const authHeader = requestOptions.headers.Authorization
        expect(authHeader).toMatch(/^Basic /)

        // Decode and verify credentials
        const base64Credentials = authHeader.replace('Basic ', '')
        const credentials = Buffer.from(base64Credentials, 'base64').toString('utf-8')
        const [clientId, clientSecret] = credentials.split(':')

        expect(clientId).toBe(`${providerId}_client_id`)
        expect(clientSecret).toBe(`${providerId}_client_secret`)

        // Verify body contains only required parameters
        const bodyParams = new URLSearchParams(requestOptions.body)
        const bodyKeys = Array.from(bodyParams.keys())

        expect(bodyKeys).toEqual(['grant_type', 'refresh_token'])
        expect(bodyParams.get('grant_type')).toBe('refresh_token')
        expect(bodyParams.get('refresh_token')).toBe(refreshToken)

        // Verify client credentials are NOT in the body
        expect(bodyParams.get('client_id')).toBeNull()
        expect(bodyParams.get('client_secret')).toBeNull()
      })
    })
  })
```

*   **`basicAuthProviders` array:** This array defines the OAuth providers that use Basic Authentication. Each object in the array contains the provider's `name`, `providerId`, and token `endpoint`.
*   **`basicAuthProviders.forEach(...)`:** This loop iterates through each provider in the `basicAuthProviders` array and creates a test case for each one.
*   **`it(...)`:** This defines an individual test case.  The description clearly states the purpose of the test: `should send ${name} request with Basic Auth header and no credentials in body`.
*   **`await refreshOAuthToken(providerId, refreshToken)`:** This calls the function being tested, passing the `providerId` and a mock `refreshToken`.
*   **`expect(mockFetch).toHaveBeenCalledWith(...)`:** This is a key assertion.  It verifies that the `mockFetch` function was called with the correct arguments. Specifically, it checks:
    *   The correct `endpoint` URL.
    *   The request `method` is `POST`.
    *   The `headers` include:
        *   `Content-Type: application/x-www-form-urlencoded` (the standard content type for OAuth token requests).
        *   `Authorization: expect.stringMatching(/^Basic /)`  (that the Authorization header is present and starts with "Basic ").
    *   The `body` is some string (URL-encoded parameters).
*   **`const [, requestOptions] = (mockFetch as Mock).mock.calls[0]`:** This line retrieves the request options that were passed to `mockFetch`.  `mockFetch.mock.calls` is an array of arrays, where each inner array contains the arguments passed to `mockFetch` on each call. `[0]` gets the first call, and the destructuring `[, requestOptions]` gets the second argument (the request options). Type assertion `as Mock` is used here to tell TypeScript that `mockFetch` is a mock function, so it has a `mock` property.
*   **`// Verify Basic Auth header...`:** The following lines extract and verify the contents of the `Authorization` header:
    *   It confirms that the `Authorization` header exists and starts with "Basic ".
    *   It extracts the Base64-encoded credentials from the header.
    *   It decodes the Base64 string to retrieve the `clientId` and `clientSecret`.
    *   It asserts that the `clientId` and `clientSecret` match the expected values, which are constructed from the `providerId` and the mock environment variables.
*   **`// Verify body contains only required parameters...`:** The following lines verify the contents of the request body:
    *   It creates a `URLSearchParams` object from the request body.
    *   It asserts that the body contains only the `grant_type` and `refresh_token` parameters.
    *   It asserts that the `grant_type` is `refresh_token`.
    *   It asserts that the `refresh_token` matches the mock `refreshToken`.
    *   It asserts that the `client_id` and `client_secret` are NOT present in the body.

**5. Testing Body Credential Providers:**

```typescript
 describe('Body Credential Providers', () => {
    const bodyCredentialProviders = [
      { name: 'Google', providerId: 'google', endpoint: 'https://oauth2.googleapis.com/token' },
      {
        name: 'GitHub',
        providerId: 'github',
        endpoint: 'https://github.com/login/oauth/access_token',
      },
      {
        name: 'Microsoft',
        providerId: 'microsoft',
        endpoint: 'https://login.microsoftonline.com/common/oauth2/v2.0/token',
      },
      {
        name: 'Outlook',
        providerId: 'outlook',
        endpoint: 'https://login.microsoftonline.com/common/oauth2/v2.0/token',
      },
      {
        name: 'Supabase',
        providerId: 'supabase',
        endpoint: 'https://api.supabase.com/v1/oauth/token',
      },
      { name: 'Notion', providerId: 'notion', endpoint: 'https://api.notion.com/v1/oauth/token' },
      { name: 'Slack', providerId: 'slack', endpoint: 'https://slack.com/api/oauth.v2.access' },
    ]

    bodyCredentialProviders.forEach(({ name, providerId, endpoint }) => {
      it(`should send ${name} request with credentials in body and no Basic Auth`, async () => {
        const refreshToken = 'test_refresh_token'

        await refreshOAuthToken(providerId, refreshToken)

        expect(mockFetch).toHaveBeenCalledWith(
          endpoint,
          expect.objectContaining({
            method: 'POST',
            headers: expect.objectContaining({
              'Content-Type': 'application/x-www-form-urlencoded',
            }),
            body: expect.any(String),
          })
        )

        const [, requestOptions] = (mockFetch as Mock).mock.calls[0]

        // Verify no Basic Auth header
        expect(requestOptions.headers.Authorization).toBeUndefined()

        // Verify body contains all required parameters
        const bodyParams = new URLSearchParams(requestOptions.body)
        const bodyKeys = Array.from(bodyParams.keys()).sort()

        expect(bodyKeys).toEqual(['client_id', 'client_secret', 'grant_type', 'refresh_token'])
        expect(bodyParams.get('grant_type')).toBe('refresh_token')
        expect(bodyParams.get('refresh_token')).toBe(refreshToken)

        // Verify client credentials are in the body
        const expectedClientId =
          providerId === 'outlook' ? 'microsoft_client_id' : `${providerId}_client_id`
        const expectedClientSecret =
          providerId === 'outlook' ? 'microsoft_client_secret' : `${providerId}_client_secret`

        expect(bodyParams.get('client_id')).toBe(expectedClientId)
        expect(bodyParams.get('client_secret')).toBe(expectedClientSecret)
      })
    })

    it('should include Accept header for GitHub requests', async () => {
      const refreshToken = 'test_refresh_token'

      await refreshOAuthToken('github', refreshToken)

      const [, requestOptions] = (mockFetch as Mock).mock.calls[0]
      expect(requestOptions.headers.Accept).toBe('application/json')
    })
  })
```

*   **`bodyCredentialProviders` array:** This array defines the OAuth providers that expect credentials in the request body.
*   The structure and logic of the tests within this `describe` block are similar to the Basic Auth Providers section, but with the following key differences:
    *   **`expect(requestOptions.headers.Authorization).toBeUndefined()`:** This asserts that the `Authorization` header is NOT present.
    *   **`expect(bodyKeys).toEqual(['client_id', 'client_secret', 'grant_type', 'refresh_token'])`:** This asserts that the request body contains `client_id`, `client_secret`, `grant_type`, and `refresh_token` parameters.
    *   It verifies that the `client_id` and `client_secret` in the body match the expected values based on the `providerId` (and a special case for "outlook").
*   **`it('should include Accept header for GitHub requests', ...)`:** This is a specific test case for GitHub that verifies that the `Accept: application/json` header is included in the request.  GitHub requires this header for the token endpoint to return JSON.

**6. Error Handling Tests:**

```typescript
  describe('Error Handling', () => {
    it('should return null for unsupported provider', async () => {
      const refreshToken = 'test_refresh_token'

      const result = await refreshOAuthToken('unsupported', refreshToken)

      expect(result).toBeNull()
    })

    it('should return null for API error responses', async () => {
      const refreshToken = 'test_refresh_token'

      mockFetch.mockResolvedValueOnce({
        ok: false,
        status: 400,
        text: async () =>
          JSON.stringify({
            error: 'invalid_request',
            error_description: 'Invalid refresh token',
          }),
      })

      const result = await refreshOAuthToken('google', refreshToken)

      expect(result).toBeNull()
    })

    it('should return null for network errors', async () => {
      const refreshToken = 'test_refresh_token'

      mockFetch.mockRejectedValueOnce(new Error('Network error'))

      const result = await refreshOAuthToken('google', refreshToken)

      expect(result).toBeNull()
    })
  })
```

*   This `describe` block tests how `refreshOAuthToken` handles errors.
*   **`it('should return null for unsupported provider', ...)`:** This tests what happens when an invalid `providerId` is passed to the function.
*   **`it('should return null for API error responses', ...)`:** This tests how the function handles errors returned by the OAuth provider's API (e.g., invalid refresh token). It mocks a failed `fetch` response (`ok: false`) with a 400 status code and a JSON body containing error information.
*   **`it('should return null for network errors', ...)`:** This tests how the function handles network errors (e.g., the API is unavailable). It mocks `fetch` to reject with an error.

**7. Token Response Handling Tests:**

```typescript
  describe('Token Response Handling', () => {
    it('should handle providers that return new refresh tokens', async () => {
      const refreshToken = 'old_refresh_token'
      const newRefreshToken = 'new_refresh_token'

      mockFetch.mockResolvedValueOnce({
        ok: true,
        json: async () => ({
          access_token: 'new_access_token',
          expires_in: 3600,
          refresh_token: newRefreshToken,
        }),
      })

      const result = await refreshOAuthToken('airtable', refreshToken)

      expect(result).toEqual({
        accessToken: 'new_access_token',
        expiresIn: 3600,
        refreshToken: newRefreshToken,
      })
    })

    it('should use original refresh token when new one is not provided', async () => {
      const refreshToken = 'original_refresh_token'

      mockFetch.mockResolvedValueOnce({
        ok: true,
        json: async () => ({
          access_token: 'new_access_token',
          expires_in: 3600,
          // No refresh_token in response
        }),
      })

      const result = await refreshOAuthToken('google', refreshToken)

      expect(result).toEqual({
        accessToken: 'new_access_token',
        expiresIn: 3600,
        refreshToken: refreshToken,
      })
    })

    it('should return null when access token is missing', async () => {
      const refreshToken = 'test_refresh_token'

      mockFetch.mockResolvedValueOnce({
        ok: true,
        json: async () => ({
          expires_