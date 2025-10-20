This TypeScript code defines a configuration object for an integration tool, specifically designed to interact with the Google Vault API to create a data export within a specific "matter" (a case or investigation).

---

### **1. Purpose of this File**

This file's primary purpose is to define a **"tool" configuration** named `createMattersExportTool`. This configuration acts as a blueprint for a specific operation: **creating a new export in Google Vault related to a legal matter**.

Think of it as a pre-packaged instruction set for your application:
*   It tells your application *what* Google Vault API endpoint to call.
*   It specifies *how* to authenticate with Google Vault (using OAuth).
*   It outlines *what information* (parameters) is needed from the user or system to perform the export (e.g., matter ID, export name, data source).
*   It describes *how to construct* the HTTP request body and headers.
*   It provides instructions on *how to process* the response received from Google Vault.

Essentially, this file makes it easy for other parts of your application to trigger a Google Vault export without needing to manually handle the intricate details of the API request and response every time.

---

### **2. Simplified Complex Logic**

The code is structured as a `ToolConfig` object, which is a common pattern for defining integrations in many applications. Here's a simplification of its key components:

*   **Self-Contained Definition**: The `createMattersExportTool` object encapsulates *everything* needed for this specific Google Vault operation.
*   **Input Parameters (`params`)**: Instead of directly dealing with API fields, you define user-friendly input parameters. Some are hidden (like `accessToken` for security), others are user-facing (like `matterId`, `exportName`).
*   **OAuth for Security (`oauth`)**: This section explicitly states that authentication is required and what specific Google permissions (`ediscovery` scope) are needed. Your application's OAuth flow would handle getting this access.
*   **Request Building (`request`)**: This is where the magic happens. It dynamically creates the API call:
    *   **URL**: Uses the `matterId` to build the correct API endpoint.
    *   **Headers**: Adds the required `Authorization` token and `Content-Type`.
    *   **Body (the trickiest part)**: This function constructs the complex JSON payload Google Vault expects.
        *   It intelligently handles `accountEmails`, allowing them to be passed either as a single comma-separated string or an array, then cleans them up.
        *   It then decides whether the export should be scoped by specific `emails` or by an `orgUnitId`, constructing the correct JSON structure for the `query` based on which option is provided.
*   **Response Handling (`transformResponse`)**: This defines how to interpret what Google Vault sends back. It checks if the request was successful and either returns the parsed data or throws a descriptive error.

In essence, this `ToolConfig` takes simple inputs, handles all the complex API-specific logic (authentication, URL construction, JSON body formatting, error handling), and returns a standardized output.

---

### **3. Line-by-Line Explanation**

```typescript
import type { GoogleVaultCreateMattersExportParams } from '@/tools/google_vault/types'
import type { ToolConfig } from '@/tools/types'
```
*   **`import type { GoogleVaultCreateMattersExportParams } from '@/tools/google_vault/types'`**: This line imports a TypeScript *type* definition named `GoogleVaultCreateMattersExportParams`. This type likely specifies the expected structure and data types for all the parameters (inputs) that this tool will receive when an export is created. The `type` keyword ensures that only the type information is imported, not any executable JavaScript code.
*   **`import type { ToolConfig } from '@/tools/types'`**: This line imports another TypeScript *type* named `ToolConfig`. This is a generic type that likely defines the standardized structure for all "tool" configuration objects within your application. It helps enforce consistency across different integration tools.

```typescript
// matters.exports.create
// POST https://vault.googleapis.com/v1/matters/{matterId}/exports
```
*   These are **comments** that serve as documentation. They specify the corresponding Google Vault API method (`matters.exports.create`), the HTTP request method (`POST`), and the general pattern of the API endpoint URL (`https://vault.googleapis.com/v1/matters/{matterId}/exports`) this tool targets.

```typescript
export const createMattersExportTool: ToolConfig<GoogleVaultCreateMattersExportParams> = {
```
*   **`export const createMattersExportTool:`**: Declares a constant variable named `createMattersExportTool` and makes it accessible for import in other files (`export`).
*   **`: ToolConfig<GoogleVaultCreateMattersExportParams>`**: This is a **type annotation**. It tells TypeScript that the `createMattersExportTool` object *must* conform to the `ToolConfig` interface, using `GoogleVaultCreateMattersExportParams` as the type for its specific parameters. This ensures type safety and autocompletion.
*   **`= { ... }`**: Begins the definition of the `createMattersExportTool` object.

```typescript
  id: 'create_matters_export',
  name: 'Vault Create Export (by Matter)',
  description: 'Create an export in a matter',
  version: '1.0',
```
*   **`id: 'create_matters_export'`**: A unique, machine-readable identifier for this tool within your application.
*   **`name: 'Vault Create Export (by Matter)'`**: A human-readable name, useful for displaying in a user interface.
*   **`description: 'Create an export in a matter'`**: A short, helpful description of what the tool does.
*   **`version: '1.0'`**: The version number of this specific tool configuration.

```typescript
  oauth: {
    required: true,
    provider: 'google-vault',
    additionalScopes: ['https://www.googleapis.com/auth/ediscovery'],
  },
```
*   **`oauth: { ... }`**: This object defines the OAuth (authentication) requirements for using this tool.
    *   **`required: true`**: Indicates that OAuth authentication is absolutely necessary to execute this tool.
    *   **`provider: 'google-vault'`**: Specifies the name of the OAuth provider, likely an internal identifier that maps to your Google OAuth client configuration.
    *   **`additionalScopes: ['https://www.googleapis.com/auth/ediscovery']`**: An array of specific OAuth scopes (permissions) that must be requested from the user. The `ediscovery` scope grants permission to manage eDiscovery-related operations in Google Vault.

```typescript
  params: {
    accessToken: { type: 'string', required: true, visibility: 'hidden' },
    matterId: { type: 'string', required: true, visibility: 'user-only' },
    exportName: { type: 'string', required: true, visibility: 'user-only' },
    corpus: {
      type: 'string',
      required: true,
      visibility: 'user-only',
      description: 'Data corpus to export (MAIL, DRIVE, GROUPS, HANGOUTS_CHAT, VOICE)',
    },
    accountEmails: {
      type: 'string',
      required: false,
      visibility: 'user-only',
      description: 'Comma-separated list of user emails to scope export',
    },
    orgUnitId: {
      type: 'string',
      required: false,
      visibility: 'user-only',
      description: 'Organization unit ID to scope export (alternative to emails)',
    },
  },
```
*   **`params: { ... }`**: This object defines the input parameters (arguments) that the tool expects when it's invoked. Each parameter is an object with these properties:
    *   **`accessToken: { type: 'string', required: true, visibility: 'hidden' }`**:
        *   `type: 'string'`: The parameter is expected to be a string.
        *   `required: true`: This parameter is mandatory.
        *   `visibility: 'hidden'`: This parameter is typically handled internally by the system (e.g., provided by the OAuth flow) and not directly exposed to or entered by a user.
    *   **`matterId: { type: 'string', required: true, visibility: 'user-only' }`**:
        *   `type: 'string'`, `required: true`: The unique ID of the Google Vault matter (case).
        *   `visibility: 'user-only'`: This parameter is expected to be provided by the end-user or an agent acting on their behalf.
    *   **`exportName: { type: 'string', required: true, visibility: 'user-only' }`**:
        *   `type: 'string'`, `required: true`: The desired name for the export job.
        *   `visibility: 'user-only'`: User-provided.
    *   **`corpus: { ... }`**:
        *   `type: 'string'`, `required: true`: Specifies the type of data to be exported (e.g., mail, drive files).
        *   `visibility: 'user-only'`: User-provided.
        *   `description: 'Data corpus to export (MAIL, DRIVE, GROUPS, HANGOUTS_CHAT, VOICE)'`: Provides a helpful explanation of the allowed values.
    *   **`accountEmails: { ... }`**:
        *   `type: 'string'`, `required: false`: An optional parameter, expected to be a string containing a comma-separated list of user email addresses.
        *   `visibility: 'user-only'`: User-provided.
        *   `description: 'Comma-separated list of user emails to scope export'`: Explains its purpose.
    *   **`orgUnitId: { ... }`**:
        *   `type: 'string'`, `required: false`: An optional parameter, expected to be a string representing an organizational unit ID.
        *   `visibility: 'user-only'`: User-provided.
        *   `description: 'Organization unit ID to scope export (alternative to emails)'`: Explains its purpose as an alternative to `accountEmails`.

```typescript
  request: {
    url: (params) => `https://vault.googleapis.com/v1/matters/${params.matterId}/exports`,
    method: 'POST',
    headers: (params) => ({
      Authorization: `Bearer ${params.accessToken}`,
      'Content-Type': 'application/json',
    }),
    body: (params) => {
      // Handle accountEmails - can be string (comma-separated) or array
      let emails: string[] = []
      if (params.accountEmails) {
        if (Array.isArray(params.accountEmails)) {
          emails = params.accountEmails
        } else if (typeof params.accountEmails === 'string') {
          emails = params.accountEmails
            .split(',')
            .map((e) => e.trim())
            .filter(Boolean)
        }
      }

      const scope =
        emails.length > 0
          ? { accountInfo: { emails } }
          : params.orgUnitId
            ? { orgUnitInfo: { orgUnitId: params.orgUnitId } }
            : {}

      const searchMethod = emails.length > 0 ? 'ACCOUNT' : params.orgUnitId ? 'ORG_UNIT' : undefined

      const query: any = {
        corpus: params.corpus,
        dataScope: 'ALL_DATA',
        searchMethod: searchMethod,
        terms: params.terms || undefined,
        startTime: params.startTime || undefined,
        endTime: params.endTime || undefined,
        timeZone: params.timeZone || undefined,
        ...scope,
      }

      return {
        name: params.exportName,
        query,
      }
    },
  },
```
*   **`request: { ... }`**: This object defines how to construct the actual HTTP request that will be sent to the Google Vault API.
    *   **`url: (params) => `https://vault.googleapis.com/v1/matters/${params.matterId}/exports```**:
        *   A **function** that takes the input `params` and returns the full URL for the API endpoint. It dynamically injects the `matterId` from the parameters into the URL.
    *   **`method: 'POST'`**: Specifies that the HTTP request method will be `POST`, as indicated by the API documentation.
    *   **`headers: (params) => ({ ... })`**:
        *   A **function** that takes the input `params` and returns an object of HTTP headers for the request.
        *   **`Authorization: `Bearer ${params.accessToken}```**: Sets the `Authorization` header, including the `accessToken` obtained via OAuth, prefixed with "Bearer " as per common token standards. This authenticates the request.
        *   **`'Content-Type': 'application/json'`**: Informs the server that the request body will be in JSON format.
    *   **`body: (params) => { ... }`**:
        *   A **function** that takes the input `params` and returns a JavaScript object. This object will be serialized into the JSON body of the `POST` request. This is the most complex part of the tool.
        *   **`let emails: string[] = []`**: Initializes an empty array to store cleaned email addresses.
        *   **`if (params.accountEmails) { ... }`**: Checks if the `accountEmails` parameter was provided.
            *   **`if (Array.isArray(params.accountEmails)) { emails = params.accountEmails }`**: If `accountEmails` is already an array, it's used directly.
            *   **`else if (typeof params.accountEmails === 'string') { ... }`**: If `accountEmails` is a string (as defined in the `params` section, typically comma-separated).
                *   **`.split(',')`**: Splits the string by commas into an array of substrings.
                *   **`.map((e) => e.trim())`**: Trims whitespace from each resulting email string.
                *   **`.filter(Boolean)`**: Removes any empty strings from the array (e.g., if there were redundant commas).
        *   **`const scope = ...`**: This line constructs the `scope` object for the export query based on whether account emails or an organizational unit ID are provided.
            *   **`emails.length > 0 ? { accountInfo: { emails } } : ...`**: If the `emails` array is not empty, the `scope` is set to include `accountInfo` with the processed `emails`.
            *   **`params.orgUnitId ? { orgUnitInfo: { orgUnitId: params.orgUnitId } } : {}`**: Otherwise (if no emails), if `orgUnitId` is provided, the `scope` is set to include `orgUnitInfo`. If neither is provided, `scope` becomes an empty object.
        *   **`const searchMethod = ...`**: This line determines the `searchMethod` string for the API request based on the chosen scoping method.
            *   **`emails.length > 0 ? 'ACCOUNT' : ...`**: If `emails` are present, `searchMethod` is `'ACCOUNT'`.
            *   **`params.orgUnitId ? 'ORG_UNIT' : undefined`**: Otherwise (if no emails), if `orgUnitId` is present, `searchMethod` is `'ORG_UNIT'`. If neither, it's `undefined`.
        *   **`const query: any = { ... }`**: This builds the main `query` object, which is part of the request body. `any` is used as a fallback type here for flexibility.
            *   **`corpus: params.corpus`**: Sets the data corpus from the input parameters.
            *   **`dataScope: 'ALL_DATA'`**: A hardcoded value indicating that all data within the specified scope should be exported.
            *   **`searchMethod: searchMethod`**: Uses the `searchMethod` determined earlier.
            *   **`terms: params.terms || undefined`**: Includes optional search terms if provided (from the `GoogleVaultCreateMattersExportParams` type, even if not explicitly listed in the `params` section above), otherwise `undefined`.
            *   **`startTime: params.startTime || undefined`**, **`endTime: params.endTime || undefined`**, **`timeZone: params.timeZone || undefined`**: Similar to `terms`, these are optional parameters for the query.
            *   **`...scope`**: The **spread operator** merges the properties of the dynamically created `scope` object (either `accountInfo` or `orgUnitInfo`) directly into the `query` object.
        *   **`return { name: params.exportName, query, }`**: Returns the final object that will be converted to JSON and sent as the request body. It includes the `exportName` and the constructed `query` object.

```typescript
  transformResponse: async (response: Response) => {
    const data = await response.json()
    if (!response.ok) {
      throw new Error(data.error?.message || 'Failed to create export')
    }
    return { success: true, output: data }
  },
}
```
*   **`transformResponse: async (response: Response) => { ... }`**: This is an asynchronous function responsible for processing the HTTP `response` received from the Google Vault API.
    *   **`const data = await response.json()`**: Asynchronously parses the body of the HTTP response as JSON.
    *   **`if (!response.ok) { ... }`**: Checks if the `response.ok` property is `false`. `response.ok` is `true` for successful HTTP status codes (200-299) and `false` otherwise.
        *   **`throw new Error(data.error?.message || 'Failed to create export')`**: If the response was *not* successful, it throws a new `Error`. It attempts to extract a specific error message from the API's JSON response (`data.error?.message`) or uses a generic fallback message.
    *   **`return { success: true, output: data }`**: If the response was successful, it returns a standardized object indicating `success: true` and includes the parsed JSON `data` from the API response in the `output` property.