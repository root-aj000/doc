```typescript
import type { AirtableCreateParams, AirtableCreateResponse } from '@/tools/airtable/types'
import type { ToolConfig } from '@/tools/types'

export const airtableCreateRecordsTool: ToolConfig<AirtableCreateParams, AirtableCreateResponse> = {
  id: 'airtable_create_records',
  name: 'Airtable Create Records',
  description: 'Write new records to an Airtable table',
  version: '1.0.0',

  oauth: {
    required: true,
    provider: 'airtable',
  },

  params: {
    accessToken: {
      type: 'string',
      required: true,
      visibility: 'hidden',
      description: 'OAuth access token',
    },
    baseId: {
      type: 'string',
      required: true,
      visibility: 'user-only',
      description: 'ID of the Airtable base',
    },
    tableId: {
      type: 'string',
      required: true,
      visibility: 'user-only',
      description: 'ID or name of the table',
    },
    records: {
      type: 'json',
      required: true,
      visibility: 'user-or-llm',
      description: 'Array of records to create, each with a `fields` object',
      // Example: [{ fields: { "Field 1": "Value1", "Field 2": "Value2" } }]
    },
  },

  request: {
    url: (params) => `https://api.airtable.com/v0/${params.baseId}/${params.tableId}`,
    method: 'POST',
    headers: (params) => ({
      Authorization: `Bearer ${params.accessToken}`,
      'Content-Type': 'application/json',
    }),
    body: (params) => ({ records: params.records }),
  },

  transformResponse: async (response) => {
    const data = await response.json()
    return {
      success: true,
      output: {
        records: data.records || [],
        metadata: {
          recordCount: (data.records || []).length,
        },
      },
    }
  },

  outputs: {
    records: {
      type: 'json',
      description: 'Array of created Airtable records',
      items: {
        type: 'object',
        properties: {
          id: { type: 'string' },
          createdTime: { type: 'string' },
          fields: { type: 'object' },
        },
      },
    },
    metadata: {
      type: 'json',
      description: 'Operation metadata',
    },
  },
}
```

## Explanation of the Code:

This TypeScript code defines a configuration object for a tool that creates records in an Airtable database.  It leverages a specific structure (`ToolConfig`) to describe the tool's properties, how to interact with it (parameters, request details), and how to process its responses.  Let's break down each part:

**1. Imports:**

```typescript
import type { AirtableCreateParams, AirtableCreateResponse } from '@/tools/airtable/types'
import type { ToolConfig } from '@/tools/types'
```

*   `import type { AirtableCreateParams, AirtableCreateResponse } from '@/tools/airtable/types'`: This line imports type definitions related to Airtable interaction.  `AirtableCreateParams` likely defines the structure of the parameters required to *create* records in Airtable (e.g., base ID, table ID, records data).  `AirtableCreateResponse` likely defines the structure of the response received after creating records in Airtable. Using `type` ensures these are only used for type checking and aren't emitted into the JavaScript output.  The `@/` alias suggests these types are defined within the project's `src` directory, specifically in a `tools/airtable/types.ts` file (or similar).
*   `import type { ToolConfig } from '@/tools/types'`: This imports the `ToolConfig` type, which is a generic type that defines the structure for configuring any tool within the application. It likely specifies the expected properties like `id`, `name`, `description`, `params`, `request`, `transformResponse`, and `outputs`. The `@/` alias suggests these types are defined within the project's `src` directory, specifically in a `tools/types.ts` file (or similar).

**2. `airtableCreateRecordsTool` Definition:**

```typescript
export const airtableCreateRecordsTool: ToolConfig<AirtableCreateParams, AirtableCreateResponse> = { ... }
```

*   `export const airtableCreateRecordsTool`:  This declares a constant variable named `airtableCreateRecordsTool`.  It is `export`ed, meaning it can be used by other modules in the application.
*   `: ToolConfig<AirtableCreateParams, AirtableCreateResponse>`: This is a TypeScript type annotation.  It specifies that `airtableCreateRecordsTool` must conform to the `ToolConfig` interface. Critically, the generic type arguments `<AirtableCreateParams, AirtableCreateResponse>` tell `ToolConfig` that this specific tool uses `AirtableCreateParams` for its input parameters and produces `AirtableCreateResponse` as its output response type. This provides strong typing and ensures that the tool configuration is consistent.

**3. Tool Configuration Object (inside the `{ ... }`):**

This is where the actual configuration of the Airtable create records tool is defined.  Let's go through each property:

*   **`id: 'airtable_create_records'`**: A unique identifier for this tool.  This is used internally to reference the tool.
*   **`name: 'Airtable Create Records'`**: A human-readable name for the tool, displayed in the user interface.
*   **`description: 'Write new records to an Airtable table'`**: A short description of what the tool does, helping users understand its purpose.
*   **`version: '1.0.0'`**: The version number of the tool, useful for tracking updates and compatibility.
*   **`oauth: { required: true, provider: 'airtable' }`**:  This specifies that OAuth authentication is required to use this tool and that the OAuth provider is 'airtable'. This indicates that the tool will use an OAuth flow to obtain permission to access the user's Airtable account.

*   **`params: { ... }`**:  This section defines the parameters that the tool accepts.  Each parameter has a type, requirement status, visibility, and description:

    *   `accessToken`:
        *   `type: 'string'`: The parameter is expected to be a string.
        *   `required: true`: The parameter must be provided.
        *   `visibility: 'hidden'`: The parameter is hidden from the user, likely because it's handled automatically via OAuth.
        *   `description: 'OAuth access token'`: Explains the purpose of the parameter.
    *   `baseId`:
        *   `type: 'string'`:  The parameter is expected to be a string.
        *   `required: true`: The parameter must be provided.
        *   `visibility: 'user-only'`:  The parameter is visible to the user, but not to LLMs.
        *   `description: 'ID of the Airtable base'`: Explains the purpose of the parameter.
    *   `tableId`:
        *   `type: 'string'`: The parameter is expected to be a string.
        *   `required: true`: The parameter must be provided.
        *   `visibility: 'user-only'`: The parameter is visible to the user, but not to LLMs.
        *   `description: 'ID or name of the table'`: Explains the purpose of the parameter.
    *   `records`:
        *   `type: 'json'`: The parameter is expected to be a JSON object.
        *   `required: true`: The parameter must be provided.
        *   `visibility: 'user-or-llm'`: The parameter is visible to the user or to LLMs.
        *   `description: 'Array of records to create, each with a `fields` object'`:  Explains the purpose of the parameter and its expected structure. The comment provides a helpful example.

*   **`request: { ... }`**: This section describes how to make the HTTP request to the Airtable API.

    *   `url: (params) => \`https://api.airtable.com/v0/\${params.baseId}/\${params.tableId}\``:  A function that dynamically constructs the Airtable API URL using the `baseId` and `tableId` parameters.  This is an interpolated string (template literal) in JavaScript/TypeScript.
    *   `method: 'POST'`: Specifies that the HTTP request should use the POST method, which is typically used for creating new resources.
    *   `headers: (params) => ({ ... })`: A function that returns an object containing the HTTP headers.
        *   `Authorization: \`Bearer \${params.accessToken}\``: Sets the `Authorization` header with the `accessToken` to authenticate with the Airtable API.  This uses the "Bearer" authentication scheme.
        *   `'Content-Type': 'application/json'`:  Sets the `Content-Type` header to `application/json`, indicating that the request body is in JSON format.
    *   `body: (params) => ({ records: params.records })`: A function that creates the request body.  It wraps the `records` parameter in a JSON object with the key `records`, as required by the Airtable API.

*   **`transformResponse: async (response) => { ... }`**: This function is responsible for processing the response received from the Airtable API.

    *   `const data = await response.json()`: Parses the JSON response from the API.
    *   `return { success: true, output: { ... } }`:  Returns an object indicating whether the request was successful and provides structured output.
        *   `success: true`:  Indicates that the response was successfully processed (this doesn't necessarily mean the Airtable API call itself was successful, just that the response was handled).  Error handling within the `transformResponse` could set this to `false` if necessary.
        *   `output: { ... }`:  Contains the processed data from the Airtable API response.
            *   `records: data.records || []`: Extracts the `records` array from the API response. If `data.records` is undefined or null, it defaults to an empty array (`[]`). This prevents errors if the response doesn't contain any records.
            *   `metadata: { recordCount: (data.records || []).length }`:  Creates a `metadata` object containing information about the operation, specifically the number of records created. It uses the same nullish coalescing operator (`|| []`) to ensure that `length` is accessed on an array even if `data.records` is null or undefined.

*   **`outputs: { ... }`**: This section defines the structure of the output data that the tool returns.  It describes the data that is made available to other parts of the application after the tool has been executed.

    *   `records`:
        *   `type: 'json'`: Specifies that the `records` output is a JSON array.
        *   `description: 'Array of created Airtable records'`: Describes the contents of the `records` output.
        *   `items: { ... }`: Defines the structure of each item (record) within the `records` array.
            *   `type: 'object'`: Each item is an object.
            *   `properties: { ... }`:  Defines the properties of each record object.
                *   `id: { type: 'string' }`: The `id` property is a string.
                *   `createdTime: { type: 'string' }`: The `createdTime` property is a string.
                *   `fields: { type: 'object' }`: The `fields` property is an object.
    *   `metadata`:
        *   `type: 'json'`: Specifies that the `metadata` output is a JSON object.
        *   `description: 'Operation metadata'`: Describes the contents of the `metadata` output.

**In Summary:**

This code defines a configurable tool for creating records in Airtable. It specifies the parameters needed (Airtable base ID, table ID, records data, and an OAuth token), describes how to make the API request to Airtable, and defines how to process the response and structure the output data. The use of TypeScript types ensures that the configuration is consistent and that the tool interacts correctly with the Airtable API. The `visibility` property on the `params` controls whether a parameter is visible to the user interface, to a Large Language Model (LLM), or both.  This is crucial for security and usability in applications that might use LLMs to automate tasks.
