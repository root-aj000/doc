```typescript
import type { AirtableGetParams, AirtableGetResponse } from '@/tools/airtable/types'
import type { ToolConfig } from '@/tools/types'

export const airtableGetRecordTool: ToolConfig<AirtableGetParams, AirtableGetResponse> = {
  id: 'airtable_get_record',
  name: 'Airtable Get Record',
  description: 'Retrieve a single record from an Airtable table by its ID',
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
    recordId: {
      type: 'string',
      required: true,
      visibility: 'user-only',
      description: 'ID of the record to retrieve',
    },
  },

  request: {
    url: (params) =>
      `https://api.airtable.com/v0/${params.baseId}/${params.tableId}/${params.recordId}`,
    method: 'GET',
    headers: (params) => ({
      Authorization: `Bearer ${params.accessToken}`,
      'Content-Type': 'application/json',
    }),
  },

  transformResponse: async (response) => {
    const data = await response.json()
    return {
      success: true,
      output: {
        record: data, // API returns the single record object
        metadata: {
          recordCount: 1,
        },
      },
    }
  },

  outputs: {
    record: {
      type: 'json',
      description: 'Retrieved Airtable record with id, createdTime, and fields',
    },
    metadata: {
      type: 'json',
      description: 'Operation metadata including record count',
    },
  },
}
```

## Explanation of the Code

This TypeScript code defines a tool configuration for retrieving a single record from Airtable using its API.  This tool configuration is designed to be used within a larger system, likely one that orchestrates different tools and services. Let's break down each part:

**1. Imports:**

```typescript
import type { AirtableGetParams, AirtableGetResponse } from '@/tools/airtable/types'
import type { ToolConfig } from '@/tools/types'
```

*   `import type { AirtableGetParams, AirtableGetResponse } from '@/tools/airtable/types'`: This line imports type definitions related to Airtable interaction.
    *   `AirtableGetParams`:  A TypeScript type that likely defines the structure of the parameters required to make the Airtable API request for getting a record. It will likely include properties for things like `baseId`, `tableId`, `recordId`, and `accessToken`.  The `type` keyword signifies it's a type import, not a value import, improving performance.
    *   `AirtableGetResponse`: A TypeScript type that likely defines the structure of the expected response from the Airtable API when getting a record. This might include the record data itself, along with metadata.  Again, `type` indicates a type import.
    *   `@/tools/airtable/types`:  This is a path alias (configured in the TypeScript or build system configuration) pointing to the file containing these specific Airtable type definitions.

*   `import type { ToolConfig } from '@/tools/types'`: This line imports the `ToolConfig` type definition.
    *   `ToolConfig`:  A TypeScript type that defines the structure of a tool configuration object.  This likely includes properties for the tool's ID, name, description, parameters, request details, response transformation, and outputs.
    *   `@/tools/types`:  A path alias (similar to above) pointing to the file containing the general `ToolConfig` type definition.

**2. `airtableGetRecordTool` Definition:**

```typescript
export const airtableGetRecordTool: ToolConfig<AirtableGetParams, AirtableGetResponse> = { ... }
```

*   `export const airtableGetRecordTool`:  This declares a constant variable named `airtableGetRecordTool` and exports it.  This makes the tool configuration available for use in other parts of the application.
*   `: ToolConfig<AirtableGetParams, AirtableGetResponse>`:  This specifies the type of the `airtableGetRecordTool` variable. It's a `ToolConfig` object, parameterized with two type arguments:
    *   `AirtableGetParams`:  Specifies the type of the parameters the tool expects as input.
    *   `AirtableGetResponse`: Specifies the type of the response the tool produces.
*   `= { ... }`:  This assigns an object literal to the `airtableGetRecordTool` variable. This object literal contains the configuration details for the tool.

**3. Tool Configuration Object:**

The object literal assigned to `airtableGetRecordTool` contains the following properties:

*   **`id`**:

    ```typescript
    id: 'airtable_get_record',
    ```

    *   A unique identifier for the tool. This is used to distinguish it from other tools in the system.
*   **`name`**:

    ```typescript
    name: 'Airtable Get Record',
    ```

    *   A human-readable name for the tool. This is likely used in the user interface.
*   **`description`**:

    ```typescript
    description: 'Retrieve a single record from an Airtable table by its ID',
    ```

    *   A brief explanation of what the tool does. This helps users understand its purpose.
*   **`version`**:

    ```typescript
    version: '1.0.0',
    ```

    *   The version number of the tool. This helps track changes and updates.
*   **`oauth`**:

    ```typescript
    oauth: {
        required: true,
        provider: 'airtable',
    },
    ```

    *   Configures the OAuth authentication required for this tool.
        *   `required: true`:  Indicates that OAuth authentication is mandatory to use this tool.
        *   `provider: 'airtable'`: Specifies that the OAuth provider is Airtable. This likely tells the system which OAuth flow to use.
*   **`params`**:

    ```typescript
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
        recordId: {
            type: 'string',
            required: true,
            visibility: 'user-only',
            description: 'ID of the record to retrieve',
        },
    },
    ```

    *   Defines the parameters that the tool accepts as input. Each parameter has the following properties:
        *   `type`:  The data type of the parameter (e.g., 'string', 'number', 'boolean').
        *   `required`:  A boolean indicating whether the parameter is required.
        *   `visibility`:  Controls the parameter's visibility in the UI (e.g., 'user-only', 'hidden').  `user-only` likely means the user must provide the value directly. `hidden` means the system will manage the value (like the access token).
        *   `description`:  A human-readable description of the parameter.
        *   `accessToken`:  The Airtable OAuth access token. Marked as `hidden` because the system manages it.
        *   `baseId`:  The ID of the Airtable base (workspace). User must provide.
        *   `tableId`:  The ID or name of the table. User must provide.
        *   `recordId`:  The ID of the specific record to retrieve. User must provide.
*   **`request`**:

    ```typescript
    request: {
        url: (params) =>
            `https://api.airtable.com/v0/${params.baseId}/${params.tableId}/${params.recordId}`,
        method: 'GET',
        headers: (params) => ({
            Authorization: `Bearer ${params.accessToken}`,
            'Content-Type': 'application/json',
        }),
    },
    ```

    *   Configures how the tool makes the API request to Airtable.
        *   `url`:  A function that dynamically constructs the API endpoint URL using the provided parameters (`baseId`, `tableId`, and `recordId`).  This uses template literals for easy string concatenation.
        *   `method`:  The HTTP method to use for the request (in this case, 'GET').
        *   `headers`:  A function that returns an object containing the HTTP headers to include in the request.  It includes:
            *   `Authorization`:  The `Bearer` token for authenticating with the Airtable API.  It uses the `accessToken` from the parameters.
            *   `Content-Type`:  Specifies that the request body is in JSON format (though a GET request doesn't typically have a body).
*   **`transformResponse`**:

    ```typescript
    transformResponse: async (response) => {
        const data = await response.json()
        return {
            success: true,
            output: {
                record: data, // API returns the single record object
                metadata: {
                    recordCount: 1,
                },
            },
        }
    },
    ```

    *   An asynchronous function that transforms the raw API response from Airtable into a structured output.
        *   `const data = await response.json()`: Parses the response body as JSON.  The `await` keyword ensures the parsing is complete before proceeding.
        *   The `return` statement constructs an object with the following properties:
            *   `success`:  A boolean indicating whether the request was successful (assumed to be true if the response was received and parsed).
            *   `output`:  An object containing the transformed data.
                *   `record`: The raw Airtable record object returned by the API.
                *   `metadata`:  An object containing metadata about the operation.  In this case, it includes `recordCount: 1` since only one record is retrieved.
*   **`outputs`**:

    ```typescript
    outputs: {
        record: {
            type: 'json',
            description: 'Retrieved Airtable record with id, createdTime, and fields',
        },
        metadata: {
            type: 'json',
            description: 'Operation metadata including record count',
        },
    },
    ```

    *   Defines the structure of the data that the tool outputs.  This helps the larger system understand the format of the data and how to use it.
        *   `record`: Describes the `record` output from the `transformResponse`.
            *   `type`: 'json', indicating it is a JSON object
            *   `description`: Describes the record, which has properties like `id`, `createdTime`, and `fields`.
        *   `metadata`: Describes the `metadata` output.
            *   `type`: 'json'
            *   `description`: Describes that it contains operation metadata, including the record count.

**Purpose of this file:**

The purpose of this file is to define a reusable configuration for an "Airtable Get Record" tool. This configuration encapsulates all the necessary information for interacting with the Airtable API to retrieve a single record. This includes authentication details, request parameters, API endpoint information, and response transformation logic.  By defining this as a `ToolConfig`, the code promotes modularity, reusability, and maintainability.

**Simplifying Complex Logic:**

While this code is already fairly straightforward, here are some ways complex logic could be simplified (though they're not necessarily needed here):

*   **Centralized Error Handling:**  The `transformResponse` function currently assumes success.  In a real-world scenario, you'd want to add error handling to check the HTTP status code of the response and handle errors appropriately (e.g., setting `success: false` and providing an error message in the output). This could be centralized in a separate utility function for handling Airtable API errors.
*   **Configuration Abstraction:**  If certain parts of the configuration (e.g., the Airtable API base URL) are likely to change or be used in multiple tools, you could extract them into a separate configuration file or environment variables.
*   **Parameter Validation:**  Adding validation to the `params` object could ensure that the input parameters are in the correct format and meet certain criteria before making the API request.  This can prevent errors and improve the robustness of the tool.

In summary, this code provides a well-structured and reusable configuration for an Airtable Get Record tool. It clearly defines the inputs, outputs, and behavior of the tool, making it easy to integrate into a larger system.
