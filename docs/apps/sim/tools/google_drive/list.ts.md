This TypeScript file defines a configuration for a "Google Drive List Files" tool. This tool allows an application (likely an AI agent or an internal system) to programmatically list files and folders within a Google Drive. It specifies how to make the API request, what parameters it accepts, how to handle authentication, and how to transform the API response into a usable format.

---

### **Purpose of This File: Defining a Google Drive "List Files" Tool**

At its core, this file describes a "tool" designed to interact with the Google Drive API. Think of it as a blueprint for a specific action: "list files."

Here's what this blueprint covers:

1.  **Identity:** What is this tool called, what does it do, and what's its version?
2.  **Authentication:** How does it securely connect to Google Drive (using OAuth)?
3.  **Inputs (Parameters):** What information does it need from the user or an AI to perform its task (e.g., which folder to list, any search queries)?
4.  **Request Logic:** How does it construct the actual HTTP request to Google's servers (the URL, HTTP method, headers)?
5.  **Output (Response) Transformation:** Once Google Drive sends back data, how does the tool process it, check for errors, and format it into a clean, usable structure?
6.  **Outputs:** What specific data fields does the tool promise to return?

This configuration makes it easy for a larger system to integrate and use this Google Drive functionality without needing to know the low-level API details.

---

### **Overall Structure: `ToolConfig`**

The entire file exports a single constant, `listTool`, which is an object conforming to the `ToolConfig` type.

```typescript
export const listTool: ToolConfig<GoogleDriveToolParams, GoogleDriveListResponse> = {
  // ... tool configuration properties ...
}
```

*   **`export const listTool`**: This declares a constant named `listTool` and makes it available for other files to import and use.
*   **`: ToolConfig<GoogleDriveToolParams, GoogleDriveListResponse>`**: This is a type annotation. It tells TypeScript that `listTool` must adhere to the structure defined by `ToolConfig`.
    *   `ToolConfig` is a generic type, meaning it takes other types as arguments.
    *   `GoogleDriveToolParams`: This specifies the *input* parameters the tool expects.
    *   `GoogleDriveListResponse`: This specifies the *output* data structure the tool will return after processing the API response.

This strongly typed approach ensures consistency and helps catch errors during development.

---

### **Detailed Explanation (Section by Section)**

Let's break down each part of the `listTool` configuration.

#### **1. Imports**

```typescript
import type { GoogleDriveListResponse, GoogleDriveToolParams } from '@/tools/google_drive/types'
import type { ToolConfig } from '@/tools/types'
```

*   **`import type { GoogleDriveListResponse, GoogleDriveToolParams } from '@/tools/google_drive/types'`**: This line imports two type definitions from a local file.
    *   `GoogleDriveListResponse`: Describes the exact shape of the data that this tool will output (e.g., an array of files, a `nextPageToken`).
    *   `GoogleDriveToolParams`: Describes the exact shape of the input parameters that this tool expects to receive (e.g., `accessToken`, `folderId`).
    *   The `type` keyword indicates these are only imported for type-checking purposes and won't generate any JavaScript code at runtime.
*   **`import type { ToolConfig } from '@/tools/types'`**: This imports the `ToolConfig` type, which is the overall interface that all tool configurations must implement.

#### **2. Basic Tool Metadata**

```typescript
  id: 'google_drive_list',
  name: 'List Google Drive Files',
  description: 'List files and folders in Google Drive',
  version: '1.0',
```

These properties provide basic identification and description for the tool:

*   **`id: 'google_drive_list'`**: A unique identifier for this specific tool. Useful for programmatic referencing.
*   **`name: 'List Google Drive Files'`**: A human-readable name for the tool.
*   **`description: 'List files and folders in Google Drive'`**: A brief explanation of what the tool does.
*   **`version: '1.0'`**: The version of this tool configuration.

#### **3. OAuth Configuration (`oauth`)**

```typescript
  oauth: {
    required: true,
    provider: 'google-drive',
    additionalScopes: ['https://www.googleapis.com/auth/drive.file'],
  },
```

This section specifies how the tool handles OAuth (Open Authorization) for secure access to Google Drive:

*   **`required: true`**: Indicates that authentication is mandatory for this tool to function.
*   **`provider: 'google-drive'`**: Identifies the specific OAuth provider to use. The larger system would know how to interact with the 'google-drive' OAuth flow.
*   **`additionalScopes: ['https://www.googleapis.com/auth/drive.file']`**: This is crucial for Google OAuth. "Scopes" are permissions that the user must grant to the application.
    *   `https://www.googleapis.com/auth/drive.file`: This specific scope requests permission to view, edit, create, and delete *only the files that this application explicitly opens or creates*. It's a more restrictive (and safer) scope than full `drive` access.

#### **4. Input Parameters (`params`)**

This object defines all the inputs the tool can accept, along with their types, requirements, and how they should be presented.

```typescript
  params: {
    accessToken: {
      type: 'string',
      required: true,
      visibility: 'hidden',
      description: 'The access token for the Google Drive API',
    },
    folderSelector: {
      type: 'string',
      required: false,
      visibility: 'user-only',
      description: 'Select the folder to list files from',
    },
    folderId: {
      type: 'string',
      required: false,
      visibility: 'hidden',
      description: 'The ID of the folder to list files from (internal use)',
    },
    query: {
      type: 'string',
      required: false,
      visibility: 'user-or-llm',
      description: 'A query to filter the files',
    },
    pageSize: {
      type: 'number',
      required: false,
      visibility: 'user-only',
      description: 'The number of files to return',
    },
    pageToken: {
      type: 'string',
      required: false,
      visibility: 'hidden',
      description: 'The page token to use for pagination',
    },
  },
```

Each property within `params` describes an argument for the tool:

*   **`accessToken`**:
    *   `type: 'string'`, `required: true`: A string representing the OAuth access token. It's mandatory for API calls.
    *   `visibility: 'hidden'`: This parameter is handled internally by the system and should not be exposed to the end-user or directly prompted from an LLM.
    *   `description`: Explains its purpose.
*   **`folderSelector`**:
    *   `type: 'string'`, `required: false`: A string representing the folder.
    *   `visibility: 'user-only'`: This parameter is meant for direct user input (e.g., a dropdown or text field where a user can select/type a folder name/path). Not for an LLM to generate.
    *   `description`: Explains its purpose.
*   **`folderId`**:
    *   `type: 'string'`, `required: false`: The actual ID of a Google Drive folder.
    *   `visibility: 'hidden'`: Similar to `accessToken`, this is for internal use or pre-populated values, not direct user/LLM input. It's likely that `folderSelector` would be resolved internally into a `folderId` before calling the tool.
    *   `description`: Clarifies its internal use.
*   **`query`**:
    *   `type: 'string'`, `required: false`: A string to filter files by name.
    *   `visibility: 'user-or-llm'`: This parameter can be provided either directly by a user or generated by a Large Language Model (LLM) if the tool is used within an AI agent.
    *   `description`: Explains its role in filtering.
*   **`pageSize`**:
    *   `type: 'number'`, `required: false`: How many items to return per page.
    *   `visibility: 'user-only'`: Primarily for user control (e.g., "show 10 files per page").
    *   `description`: Explains its pagination role.
*   **`pageToken`**:
    *   `type: 'string'`, `required: false`: A token used to fetch the *next* set of results in a paginated list.
    *   `visibility: 'hidden'`: This is entirely internal for handling subsequent API calls for pagination, not for user or LLM input.
    *   `description`: Explains its role in pagination.

#### **5. Request Configuration (`request`)**

This section defines how the HTTP request to the Google Drive API should be constructed.

```typescript
  request: {
    url: (params) => {
      const url = new URL('https://www.googleapis.com/drive/v3/files')
      url.searchParams.append(
        'fields',
        'files(id,name,mimeType,webViewLink,webContentLink,size,createdTime,modifiedTime,parents),nextPageToken'
      )
      // Ensure shared drives support
      url.searchParams.append('supportsAllDrives', 'true')
      url.searchParams.append('includeItemsFromAllDrives', 'true')
      url.searchParams.append('spaces', 'drive')

      // Build the query conditions
      const conditions = ['trashed = false'] // Always exclude trashed files
      const folderId = params.folderId || params.folderSelector
      if (folderId) {
        conditions.push(`'${folderId}' in parents`)
      }

      // Combine all conditions with AND
      url.searchParams.append('q', conditions.join(' and '))

      if (params.query) {
        const existingQ = url.searchParams.get('q')
        const queryPart = `name contains '${params.query}'`
        url.searchParams.set('q', `${existingQ} and ${queryPart}`)
      }
      if (params.pageSize) {
        url.searchParams.append('pageSize', params.pageSize.toString())
      }
      if (params.pageToken) {
        url.searchParams.append('pageToken', params.pageToken)
      }

      return url.toString()
    },
    method: 'GET',
    headers: (params) => ({
      Authorization: `Bearer ${params.accessToken}`,
    }),
  },
```

##### **`url: (params) => { ... }` (Dynamic URL Construction)**

This is a function that dynamically constructs the request URL based on the input `params`.

*   **`const url = new URL('https://www.googleapis.com/drive/v3/files')`**:
    *   Initializes a `URL` object with the base endpoint for listing files in Google Drive v3 API. This object makes it easy to manipulate URL parameters.
*   **`url.searchParams.append('fields', '...')`**:
    *   Appends the `fields` query parameter. This tells the Google Drive API *exactly* which fields of each file object it should return, optimizing the response size and speed.
    *   `files(...)`: Requests an array of file objects.
    *   `(id,name,mimeType,webViewLink,webContentLink,size,createdTime,modifiedTime,parents)`: Specifies the particular fields to retrieve for each file. These include unique ID, name, type, various links, size, timestamps, and parent folder IDs.
    *   `,nextPageToken`: Also requests the `nextPageToken`, which is crucial for handling pagination.
*   **`// Ensure shared drives support`**: These next three lines are specific to Google Drive and ensure that files from "Shared Drives" (formerly Team Drives) are included in the results.
    *   `url.searchParams.append('supportsAllDrives', 'true')`: Indicates the requesting application supports shared drives.
    *   `url.searchParams.append('includeItemsFromAllDrives', 'true')`: Tells the API to include items from all accessible shared drives.
    *   `url.searchParams.append('spaces', 'drive')`: Filters results to only include items from the 'drive' space (which includes My Drive and Shared Drives).
*   **`// Build the query conditions`**: This block constructs the `q` query parameter, which is Google Drive's way of filtering results using a custom syntax.
    *   `const conditions = ['trashed = false']`: Initializes an array of conditions. The first condition `trashed = false` is always included to ensure that deleted (trashed) files are not returned.
    *   `const folderId = params.folderId || params.folderSelector`: Prioritizes `folderId` if provided, otherwise uses `folderSelector`. This allows flexibility in how the folder is specified.
    *   `if (folderId) { conditions.push(`'${folderId}' in parents`) }`: If a `folderId` (or resolved `folderSelector`) is present, it adds a condition to the array: `'[folderId]' in parents`. This filters files to only include those whose `parents` array contains the specified `folderId`, effectively listing files within that folder.
    *   `url.searchParams.append('q', conditions.join(' and '))`: Joins all current conditions with ` and ` (e.g., `trashed = false and '123' in parents`) and appends it as the `q` parameter.
*   **`if (params.query) { ... }`**: If an additional search `query` is provided by the user/LLM:
    *   `const existingQ = url.searchParams.get('q')`: Retrieves the `q` parameter built so far.
    *   `const queryPart = `name contains '${params.query}'``: Creates a new query part to filter files by name.
    *   `url.searchParams.set('q', `${existingQ} and ${queryPart}`)`: Updates the `q` parameter by appending the new name-based query using ` and `.
*   **`if (params.pageSize) { ... }`**: If `pageSize` is provided, it's converted to a string and added as a query parameter.
*   **`if (params.pageToken) { ... }`**: If `pageToken` is provided (for pagination), it's added as a query parameter.
*   **`return url.toString()`**: Returns the complete, constructed URL as a string.

##### **`method: 'GET'`**

*   Specifies that the HTTP request should use the `GET` method, which is standard for retrieving resources.

##### **`headers: (params) => ({ ... })` (Dynamic Headers)**

This is a function that constructs the HTTP headers based on the input `params`.

*   **`Authorization: `Bearer ${params.accessToken}``**:
    *   Sets the `Authorization` header. This is the standard way to send an OAuth 2.0 access token.
    *   `Bearer`: Indicates the type of token.
    *   `${params.accessToken}`: Inserts the actual access token provided in the tool's parameters. This token proves the user has granted permission for this action.

#### **6. Response Transformation (`transformResponse`)**

This asynchronous function takes the raw HTTP `Response` from the Google Drive API and transforms it into the desired output format, including error handling.

```typescript
  transformResponse: async (response: Response) => {
    const data = await response.json()

    if (!response.ok) {
      throw new Error(data.error?.message || 'Failed to list Google Drive files')
    }

    return {
      success: true,
      output: {
        files: data.files.map((file: any) => ({
          id: file.id,
          name: file.name,
          mimeType: file.mimeType,
          webViewLink: file.webViewLink,
          webContentLink: file.webContentLink,
          size: file.size,
          createdTime: file.createdTime,
          modifiedTime: file.modifiedTime,
          parents: file.parents,
        })),
        nextPageToken: data.nextPageToken,
      },
    }
  },
```

*   **`async (response: Response) => { ... }`**: Declares an asynchronous function that takes a standard `Response` object (from a `fetch` call) as input.
*   **`const data = await response.json()`**:
    *   Parses the `Response` body as JSON. Since `response.json()` is asynchronous, `await` is used. The `data` variable will hold the parsed JSON object from the API.
*   **`if (!response.ok) { ... }`**:
    *   Checks if the HTTP response status code indicates success (i.e., `response.ok` is true for 2xx status codes).
    *   If `!response.ok` (meaning an error occurred, like 4xx or 5xx), it throws an `Error`.
    *   `data.error?.message || 'Failed to list Google Drive files'`: It tries to extract an error message from the `data` object (if the API provided one) or falls back to a generic message.
*   **`return { success: true, output: { ... } }`**: If the response was successful, it returns an object with a `success: true` flag and the processed `output`.
    *   **`files: data.files.map((file: any) => ({ ... }))`**:
        *   `data.files`: Accesses the `files` array from the raw API response.
        *   `.map((file: any) => ({ ... }))`: Iterates over each raw `file` object returned by Google Drive and transforms it into a cleaner, more specific format. This ensures that only the relevant and expected fields are passed on. `file: any` is used here because the incoming `file` structure is dynamically known only at runtime, and `map` then shapes it into a well-typed output.
        *   The inner object `({ id: file.id, name: file.name, ... })` explicitly picks and maps the desired properties, creating a consistent output structure.
    *   **`nextPageToken: data.nextPageToken`**: Extracts the `nextPageToken` directly from the raw response, allowing for subsequent paginated requests.

#### **7. Outputs Definition (`outputs`)**

This section formally describes the structure of the data that the tool will return when successful.

```typescript
  outputs: {
    files: {
      type: 'json',
      description: 'Array of file metadata objects from the specified folder',
    },
  },
```

*   **`files`**: This defines a top-level output field named `files`.
    *   **`type: 'json'`**: Indicates that the content of `files` will be a JSON-like structure (specifically, an array of objects, as defined by `GoogleDriveListResponse`).
    *   **`description: 'Array of file metadata objects from the specified folder'`**: A clear description of what this output field contains.

---

### **Summary of Complex Logic Simplification**

The most complex parts are the `request.url` and `transformResponse` functions.

1.  **Request URL Construction:**
    *   Instead of manually concatenating strings, it uses the `URL` and `URLSearchParams` objects, which are much safer and easier to manage for building query parameters.
    *   The `q` parameter (Google's custom query language) is built incrementally: starting with `trashed = false`, then adding a `folderId` filter, and finally appending any user-defined `query`. This modular approach makes it robust and readable.
    *   Explicitly selecting `fields` (like `id, name, mimeType`) ensures only necessary data is fetched, improving performance.
    *   Dedicated lines for `supportsAllDrives`, `includeItemsFromAllDrives`, and `spaces` clearly indicate shared drive support.

2.  **Response Transformation:**
    *   It clearly separates error handling (`if (!response.ok)`) from successful data processing.
    *   The `.map()` function is used to selectively pick and rename (if necessary) properties from the raw Google Drive file objects. This ensures the output is consistent, predictable, and only contains data relevant to the tool's defined `GoogleDriveListResponse` type, regardless of how many extraneous fields Google's API might send.
    *   The final output structure (`{ success: true, output: { files: [...], nextPageToken: ... } }`) is standardized for easy consumption by the calling system.