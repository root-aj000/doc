This TypeScript code defines a "tool" that allows a larger system (like an AI agent, automation platform, or specialized application) to interact with the Google Vault API. Specifically, this tool is designed to **list "holds" (legal holds) associated with a specific "matter" (a case or investigation) in Google Vault.**

It acts as a configuration blueprint, detailing everything needed to make this specific API call: what parameters it takes, how to authenticate, how to construct the request, and how to process the response.

---

### Purpose of this File

This file serves as a **declarative configuration for a "Vault List Holds (by Matter)" tool.**

Imagine you have a system that can perform various actions (tools). This file defines one such action. It describes:
1.  **What it does:** Lists legal holds in Google Vault.
2.  **How to find it:** Unique `id` and human-readable `name`.
3.  **What information it needs:** `matterId`, `accessToken`, `pageSize`, etc.
4.  **How to talk to Google Vault:** The `url`, `method`, and `headers` for the API call.
5.  **How to handle the results:** How to process the API response (`transformResponse`).
6.  **Authentication requirements:** It needs OAuth with a specific scope.

In essence, it's a reusable module that encapsulates the complexity of interacting with a specific Google Vault API endpoint, making it easy for other parts of the application to "call" this functionality without knowing the low-level API details.

---

### Simplified Complex Logic

The most "complex" parts are how the **URL is dynamically constructed** and how the **response is handled**. Let's simplify:

1.  **Constructing the Request URL (`request.url`):**
    *   This part is smart enough to handle two scenarios for listing holds:
        *   **Scenario A: Fetching a *specific* hold:** If you provide a `holdId`, it builds a URL like `/matters/{matterId}/holds/{holdId}` to get details about *just that one* hold.
        *   **Scenario B: Listing *all* holds for a matter:** If you *don't* provide a `holdId`, it builds a URL like `/matters/{matterId}/holds` and then optionally adds parameters for pagination (like `pageSize` to control how many results per page, and `pageToken` to get the next page of results). It also ensures `pageSize` is a valid number.
    *   The goal is to create the correct API endpoint URL based on the user's input.

2.  **Transforming the API Response (`transformResponse`):**
    *   After the API call is made, Google Vault sends back a response. This function's job is to:
        *   **Check for errors:** Was the API call successful (e.g., HTTP status 200-299)? If not, it tries to extract an error message from Google's response and throws a clear error.
        *   **Extract data:** If successful, it takes the actual data (the list of holds) from Google's response.
        *   **Standardize output:** It then wraps this data in a consistent format (`{ success: true, output: data }`) so that the calling system always knows what to expect.

---

### Explanation of Each Line of Code

```typescript
// Imports necessary type definitions from other files.
// These types help TypeScript ensure that the data structures used are correct.
import type { GoogleVaultListMattersHoldsParams } from '@/tools/google_vault/types'
import type { ToolConfig } from '@/tools/types'

// Exports a constant variable named 'listMattersHoldsTool'.
// This constant holds the entire configuration for our Google Vault tool.
// It's typed as 'ToolConfig', parameterized with 'GoogleVaultListMattersHoldsParams',
// which means it's a tool configuration specifically for listing Google Vault holds.
export const listMattersHoldsTool: ToolConfig<GoogleVaultListMattersHoldsParams> = {

  // A unique identifier for this specific tool.
  // This is how the system internally refers to this action.
  id: 'list_matters_holds',

  // A human-readable name for the tool, which might be displayed in a UI.
  name: 'Vault List Holds (by Matter)',

  // A brief description explaining what the tool does.
  description: 'List holds for a matter',

  // The version of this tool configuration. Useful for tracking changes.
  version: '1.0',

  // Configuration for OAuth authentication, indicating how to get authorized.
  oauth: {
    // 'required: true' means this tool cannot be used without proper OAuth authentication.
    required: true,
    // Specifies that the authentication provider is 'google-vault'.
    provider: 'google-vault',
    // Defines the specific Google API scope required for this operation.
    // This scope grants permission to access eDiscovery data in Google Vault.
    additionalScopes: ['https://www.googleapis.com/auth/ediscovery'],
  },

  // Defines the input parameters that this tool expects.
  // Each parameter has a type, a 'required' status, and 'visibility' for UI hints.
  params: {
    // 'accessToken': The OAuth access token needed to authenticate the API request.
    // 'type: 'string'': It's a string.
    // 'required: true'': It must always be provided.
    // 'visibility: 'hidden'': It should not be shown to a user in a UI, as it's a technical detail.
    accessToken: { type: 'string', required: true, visibility: 'hidden' },

    // 'matterId': The unique identifier of the legal matter for which to list holds.
    // 'type: 'string'': It's a string.
    // 'required: true'': It must always be provided.
    // 'visibility: 'user-only'': This is an important input that a user would specify.
    matterId: { type: 'string', required: true, visibility: 'user-only' },

    // 'pageSize': Optional parameter to specify the maximum number of holds to return per page.
    // 'type: 'number'': It's a number.
    // 'required: false'': It's optional. If not provided, the API will use its default page size.
    // 'visibility: 'user-only'': A user might want to control this.
    pageSize: { type: 'number', required: false, visibility: 'user-only' },

    // 'pageToken': Optional parameter for pagination; used to retrieve the next page of results.
    // 'type: 'string'': It's a string token.
    // 'required: false'': It's optional.
    // 'visibility: 'hidden'': Typically handled internally by the system, not directly by a user.
    pageToken: { type: 'string', required: false, visibility: 'hidden' },

    // 'holdId': Optional parameter to fetch details of a specific hold instead of listing all holds.
    // 'type: 'string'': It's a string.
    // 'required: false'': It's optional. If provided, it overrides the "list all" behavior.
    // 'visibility: 'user-only'': A user might specify a particular hold to look up.
    holdId: { type: 'string', required: false, visibility: 'user-only' },
  },

  // Configuration for how to construct and make the actual HTTP request to Google Vault.
  request: {
    // 'url': A function that dynamically generates the full API endpoint URL based on the input parameters.
    url: (params) => {
      // Check if a specific 'holdId' was provided.
      if (params.holdId) {
        // If 'holdId' is present, construct a URL to fetch that specific hold.
        // Example: /v1/matters/123/holds/abc
        return `https://vault.googleapis.com/v1/matters/${params.matterId}/holds/${params.holdId}`
      }
      // If 'holdId' is NOT present, construct a URL to list all holds for the matter.
      // Start with the base URL for listing holds.
      const url = new URL(`https://vault.googleapis.com/v1/matters/${params.matterId}/holds`)
      
      // Handle 'pageSize' parameter for pagination.
      // Check if 'pageSize' was provided and is not null or undefined.
      if (params.pageSize !== undefined && params.pageSize !== null) {
        // Convert the 'pageSize' parameter to a number (it might come in as a string from some inputs).
        const pageSize = Number(params.pageSize)
        // Validate if the converted 'pageSize' is a finite number greater than 0.
        if (Number.isFinite(pageSize) && pageSize > 0) {
          // If valid, add it as a query parameter to the URL (e.g., ?pageSize=10).
          url.searchParams.set('pageSize', String(pageSize))
        }
      }
      // Handle 'pageToken' parameter for pagination.
      // If 'pageToken' was provided, add it as a query parameter to the URL (e.g., ?pageToken=xyz).
      if (params.pageToken) url.searchParams.set('pageToken', params.pageToken)
      
      // The Google Vault API documentation states that omitting the 'view' parameter
      // implicitly requests the 'BASIC_HOLD' view, which is sufficient here.
      // Convert the URL object back to a string and return it.
      return url.toString()
    },
    // 'method': The HTTP method to use for the request. For listing resources, 'GET' is standard.
    method: 'GET',
    // 'headers': A function that generates the HTTP headers for the request.
    headers: (params) => ({
      // Sets the 'Authorization' header using the provided 'accessToken' as a Bearer token.
      Authorization: `Bearer ${params.accessToken}`
    }),
  },

  // 'transformResponse': An asynchronous function to process the API response received from Google Vault.
  transformResponse: async (response: Response) => {
    // Parse the JSON body of the HTTP response. This line waits for the JSON to be fully parsed.
    const data = await response.json()
    // Check if the HTTP response indicates an error (e.g., status code 4xx or 5xx).
    if (!response.ok) {
      // If there was an error, throw a new Error.
      // It tries to use Google's specific error message if available (data.error?.message),
      // otherwise, it provides a generic "Failed to list holds" message.
      throw new Error(data.error?.message || 'Failed to list holds')
    }
    // If the response was successful, return a standardized success object.
    // 'success: true' indicates the operation completed without an API-level error.
    // 'output: data' contains the actual data (the list of holds) from Google Vault.
    return { success: true, output: data }
  },
}
```