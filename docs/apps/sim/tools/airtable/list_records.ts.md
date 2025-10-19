```typescript
import type { AirtableListParams, AirtableListResponse } from '@/tools/airtable/types'
import type { ToolConfig } from '@/tools/types'

export const airtableListRecordsTool: ToolConfig<AirtableListParams, AirtableListResponse> = {
  id: 'airtable_list_records',
  name: 'Airtable List Records',
  description: 'Read records from an Airtable table',
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
      description: 'ID of the table',
    },
    maxRecords: {
      type: 'number',
      required: false,
      visibility: 'user-only',
      description: 'Maximum number of records to return',
    },
    filterFormula: {
      type: 'string',
      required: false,
      visibility: 'user-or-llm',
      description: 'Formula to filter records (e.g., "({Field Name} = \'Value\')")',
    },
  },

  request: {
    url: (params) => {
      const url = `https://api.airtable.com/v0/${params.baseId}/${params.tableId}`
      const queryParams = new URLSearchParams()
      if (params.maxRecords) queryParams.append('maxRecords', params.maxRecords.toString())
      if (params.filterFormula) {
        // Airtable formulas often contain characters needing encoding,
        // but standard encodeURIComponent might over-encode.
        // Simple replacement for single quotes is often sufficient.
        // More complex formulas might need careful encoding.
        const encodedFormula = params.filterFormula.replace(/'/g, "'")
        queryParams.append('filterByFormula', encodedFormula)
      }
      const queryString = queryParams.toString()
      const finalUrl = queryString ? `${url}?${queryString}` : url
      return finalUrl
    },
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
        records: data.records || [],
        metadata: {
          offset: data.offset,
          totalRecords: (data.records || []).length,
        },
      },
    }
  },

  outputs: {
    records: {
      type: 'json',
      description: 'Array of retrieved Airtable records',
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
      description: 'Operation metadata including pagination offset and total records count',
    },
  },
}
```

### Purpose of this file

This TypeScript file defines a configuration object (`airtableListRecordsTool`) that describes how to interact with the Airtable API to list records from a specified table. This configuration is intended for use within a larger system or application that can utilize this `ToolConfig` to dynamically execute API calls to Airtable based on user inputs and/or other system logic. This "tool" allows to fetch data from airtable, and can be used by AI agents.

### Explanation

Let's break down the code step by step:

**1. Imports:**

```typescript
import type { AirtableListParams, AirtableListResponse } from '@/tools/airtable/types'
import type { ToolConfig } from '@/tools/types'
```

-   `import type { AirtableListParams, AirtableListResponse } from '@/tools/airtable/types'`:  This line imports type definitions for `AirtableListParams` and `AirtableListResponse` from a file located at `'@/tools/airtable/types'`. These types likely define the structure of the parameters expected by the tool and the structure of the data returned by the tool, respectively. Using `type` ensures that these are only used for type-checking and are not emitted as JavaScript.
-   `import type { ToolConfig } from '@/tools/types'`:  This imports the `ToolConfig` type from `'@/tools/types'`.  This type likely defines the overall structure that all tool configurations in the system must adhere to.

**2. Tool Configuration:**

```typescript
export const airtableListRecordsTool: ToolConfig<AirtableListParams, AirtableListResponse> = {
  // ... configuration details ...
}
```

-   `export const airtableListRecordsTool`:  This declares a constant variable named `airtableListRecordsTool`. This variable will hold the configuration object for the Airtable listing tool.  The `export` keyword makes this configuration available for use in other modules.
-   `: ToolConfig<AirtableListParams, AirtableListResponse>`:  This uses a TypeScript type annotation to specify that the `airtableListRecordsTool` variable must conform to the `ToolConfig` type. The generic types `AirtableListParams` and `AirtableListResponse` specify the type of parameters the tool expects and the type of the tool's response, respectively.  This provides strong type checking to ensure the configuration is valid.
-   `= { ... }`:  This assigns an object literal to the `airtableListRecordsTool` variable. This object contains all the configuration details for the tool.

**3. Core Tool Information:**

```typescript
  id: 'airtable_list_records',
  name: 'Airtable List Records',
  description: 'Read records from an Airtable table',
  version: '1.0.0',
```

-   `id`: A unique identifier for this tool within the system.
-   `name`: A human-readable name for the tool.
-   `description`:  A brief description of what the tool does.
-   `version`:  The version number of the tool.

**4. OAuth Configuration:**

```typescript
  oauth: {
    required: true,
    provider: 'airtable',
  },
```

-   `oauth`: Defines the OAuth authentication requirements for the tool.
-   `required: true`: Indicates that OAuth authentication is mandatory to use this tool.
-   `provider: 'airtable'`: Specifies that the OAuth provider to use is 'airtable'.  This likely refers to a configured OAuth integration within the larger system.

**5. Parameters Configuration:**

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
      description: 'ID of the table',
    },
    maxRecords: {
      type: 'number',
      required: false,
      visibility: 'user-only',
      description: 'Maximum number of records to return',
    },
    filterFormula: {
      type: 'string',
      required: false,
      visibility: 'user-or-llm',
      description: 'Formula to filter records (e.g., "({Field Name} = \'Value\')")',
    },
  },
```

-   `params`: This section defines the input parameters that the tool accepts.  Each parameter is an object with the following properties:
    -   `type`: The data type of the parameter (e.g., 'string', 'number').
    -   `required`: A boolean indicating whether the parameter is required.
    -   `visibility`:  Controls who or what can see/set the parameter.
        -   `hidden`: The parameter is not visible to the user and is likely managed internally.  This is typical for sensitive information like access tokens.
        -   `user-only`: The parameter is visible and can be set by the user.
        -   `user-or-llm`:  The parameter can be set by either the user or an Language Model (LLM).
    -   `description`:  A human-readable description of the parameter.

    The parameters include:

    -   `accessToken`: The OAuth access token for authenticating with Airtable.  It's hidden from the user for security.
    -   `baseId`: The ID of the Airtable base to access.  User-configurable.
    -   `tableId`: The ID of the table within the Airtable base to access. User-configurable.
    -   `maxRecords`: The maximum number of records to retrieve. User-configurable and optional.
    -   `filterFormula`:  A formula used to filter the records retrieved from Airtable. Can be set by the user or an LLM.

**6. Request Configuration:**

```typescript
  request: {
    url: (params) => {
      const url = `https://api.airtable.com/v0/${params.baseId}/${params.tableId}`
      const queryParams = new URLSearchParams()
      if (params.maxRecords) queryParams.append('maxRecords', params.maxRecords.toString())
      if (params.filterFormula) {
        // Airtable formulas often contain characters needing encoding,
        // but standard encodeURIComponent might over-encode.
        // Simple replacement for single quotes is often sufficient.
        // More complex formulas might need careful encoding.
        const encodedFormula = params.filterFormula.replace(/'/g, "'")
        queryParams.append('filterByFormula', encodedFormula)
      }
      const queryString = queryParams.toString()
      const finalUrl = queryString ? `${url}?${queryString}` : url
      return finalUrl
    },
    method: 'GET',
    headers: (params) => ({
      Authorization: `Bearer ${params.accessToken}`,
      'Content-Type': 'application/json',
    }),
  },
```

-   `request`:  Defines how to construct and send the HTTP request to the Airtable API.
    -   `url`: A function that dynamically generates the API endpoint URL based on the input parameters.
        -   It constructs the base URL using the `baseId` and `tableId` parameters.
        -   It uses `URLSearchParams` to build the query string for optional parameters (`maxRecords`, `filterFormula`).
        -   **Important Note on Encoding:**  The code includes a comment explaining a potential issue with encoding the `filterFormula`.  Standard URL encoding might over-encode characters that are valid within Airtable formulas.  The code uses a simple replacement of single quotes with single quotes, which may be sufficient for many cases, but a more robust encoding strategy might be needed for complex formulas.
    -   `method`:  Specifies the HTTP method to use (in this case, `GET`).
    -   `headers`:  A function that returns an object containing the HTTP headers to include in the request.
        -   `Authorization`:  Sets the `Authorization` header with the `Bearer` token (OAuth access token).
        -   `Content-Type`: Sets the `Content-Type` header to `application/json`.

**7. Response Transformation:**

```typescript
  transformResponse: async (response) => {
    const data = await response.json()
    return {
      success: true,
      output: {
        records: data.records || [],
        metadata: {
          offset: data.offset,
          totalRecords: (data.records || []).length,
        },
      },
    }
  },
```

-   `transformResponse`:  An asynchronous function that processes the raw response from the Airtable API.
    -   It parses the response body as JSON using `response.json()`.
    -   It constructs a standardized output object with a `success` flag and an `output` property.
        -   `records`: Extracts the `records` array from the Airtable response.  It uses a default empty array (`data.records || []`) to prevent errors if the response doesn't contain a `records` property.
        -   `metadata`:  Creates a metadata object containing:
            -   `offset`:  The pagination offset returned by Airtable, if any.
            -   `totalRecords`: The number of records returned in the current response.

**8. Output Definition:**

```typescript
  outputs: {
    records: {
      type: 'json',
      description: 'Array of retrieved Airtable records',
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
      description: 'Operation metadata including pagination offset and total records count',
    },
  },
```

-   `outputs`: This section defines the structure and types of the data returned by the tool.  This is used to validate and describe the output of the `transformResponse` function.
    -   `records`: Describes the structure of the `records` array.
        -   `type: 'json'`: Specifies that this output is a JSON array.
        -   `description`: Provides a description of the output.
        -   `items`: Defines the structure of each item (record) in the `records` array.  Each record is expected to be an object with `id`, `createdTime`, and `fields` properties, all with their respective types.
    -   `metadata`: Describes the structure of the `metadata` object.
        -   `type: 'json'`: Specifies that this output is a JSON object.
        -   `description`: Provides a description of the metadata.

### Summary

In essence, this code defines a configurable tool for retrieving records from Airtable. It specifies the parameters needed for the API request, how to construct the request URL and headers, how to transform the response data into a standardized format, and the structure of the output data.  The use of TypeScript types ensures that the configuration is valid and that the tool is used correctly within the larger system. The `visibility` properties on the parameters also help manage how the tool is exposed to users and/or automated systems. The comment about the Airtable formula encoding is crucial and highlights a potential issue that needs careful consideration when using this tool with complex Airtable formulas.
