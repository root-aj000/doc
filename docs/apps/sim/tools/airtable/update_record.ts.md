```typescript
import type { AirtableUpdateParams, AirtableUpdateResponse } from '@/tools/airtable/types'
import type { ToolConfig } from '@/tools/types'

export const airtableUpdateRecordTool: ToolConfig<AirtableUpdateParams, AirtableUpdateResponse> = {
  id: 'airtable_update_record',
  name: 'Airtable Update Record',
  description: 'Update an existing record in an Airtable table by ID',
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
      description: 'ID of the record to update',
    },
    fields: {
      type: 'json',
      required: true,
      visibility: 'user-or-llm',
      description: 'An object containing the field names and their new values',
    },
  },

  request: {
    // The API endpoint uses PATCH for single record updates
    url: (params) =>
      `https://api.airtable.com/v0/${params.baseId}/${params.tableId}/${params.recordId}`,
    method: 'PATCH',
    headers: (params) => ({
      Authorization: `Bearer ${params.accessToken}`,
      'Content-Type': 'application/json',
    }),
    body: (params) => ({ fields: params.fields }),
  },

  transformResponse: async (response) => {
    const data = await response.json()
    return {
      success: true,
      output: {
        record: data, // API returns the single updated record object
        metadata: {
          recordCount: 1,
          updatedFields: Object.keys(data.fields || {}),
        },
      },
    }
  },

  outputs: {
    record: {
      type: 'json',
      description: 'Updated Airtable record with id, createdTime, and fields',
    },
    metadata: {
      type: 'json',
      description: 'Operation metadata including record count and updated field names',
    },
  },
}
```

### Purpose of this file

This file defines a "tool" configuration for updating records in Airtable.  It encapsulates all the necessary information for an application to interact with the Airtable API to update a specific record within a given base and table. This includes authentication details (OAuth), input parameters, the API request specification (URL, method, headers, body), how to transform the API response, and the structure of the tool's output. The `ToolConfig` type provides a standardized way to define and use such tools within a larger system.

### Explanation of each line of code

1.  **`import type { AirtableUpdateParams, AirtableUpdateResponse } from '@/tools/airtable/types'`**

    *   This line imports type definitions from a file located at `@/tools/airtable/types`.
    *   `import type` indicates that these are only type definitions and not actual values to be imported, optimizing the bundle size.
    *   `AirtableUpdateParams` likely defines the structure of the parameters required for the Airtable update operation (e.g., `baseId`, `tableId`, `recordId`, `fields`).
    *   `AirtableUpdateResponse` likely defines the structure of the expected response from the Airtable API after a successful update.
    *   The `@` symbol is often used as an alias that maps to the project's source directory (e.g., `src`).

2.  **`import type { ToolConfig } from '@/tools/types'`**

    *   This line imports the `ToolConfig` type definition from `@/tools/types`.
    *   `ToolConfig` is a generic type that represents the overall structure for configuring a tool, defining its input parameters, how to make the API request, and how to process the response. It uses generics to enforce type safety of parameters and responses.

3.  **`export const airtableUpdateRecordTool: ToolConfig<AirtableUpdateParams, AirtableUpdateResponse> = { ... }`**

    *   This line declares a constant variable named `airtableUpdateRecordTool`.
    *   `export` makes this constant available for use in other modules.
    *   The `: ToolConfig<AirtableUpdateParams, AirtableUpdateResponse>` part specifies the type of `airtableUpdateRecordTool`.  It enforces that the assigned object conforms to the `ToolConfig` interface, with `AirtableUpdateParams` as the type for input parameters and `AirtableUpdateResponse` as the expected response type.
    *   The `{ ... }` is an object literal, representing the configuration details for the Airtable update tool.

4.  **`id: 'airtable_update_record',`**

    *   A unique identifier for this tool. This is often used internally for referencing the tool.

5.  **`name: 'Airtable Update Record',`**

    *   A user-friendly name for the tool, which may be displayed in a user interface.

6.  **`description: 'Update an existing record in an Airtable table by ID',`**

    *   A brief description of what the tool does, helpful for users understanding its functionality.

7.  **`version: '1.0.0',`**

    *   The version number of the tool. This is useful for tracking changes and ensuring compatibility.

8.  **`oauth: { required: true, provider: 'airtable', },`**

    *   Specifies that OAuth authentication is required to use this tool.
    *   `required: true` indicates that a valid OAuth token is mandatory.
    *   `provider: 'airtable'` identifies Airtable as the OAuth provider.  This signals which OAuth flow and credentials to use.

9.  **`params: { ... },`**

    *   Defines the input parameters that the tool accepts. Each parameter has its own configuration.

10. **`accessToken: { type: 'string', required: true, visibility: 'hidden', description: 'OAuth access token', },`**

    *   Defines the `accessToken` parameter.
    *   `type: 'string'` specifies that the value must be a string.
    *   `required: true` indicates that this parameter is mandatory.
    *   `visibility: 'hidden'` suggests that this parameter should not be directly exposed to the user (e.g., it will be handled internally by the application).
    *   `description: 'OAuth access token'` provides a helpful description of the parameter.

11. **`baseId: { type: 'string', required: true, visibility: 'user-only', description: 'ID of the Airtable base', },`**

    *   Defines the `baseId` parameter.
    *   `type: 'string'` specifies that the value must be a string.
    *   `required: true` indicates that this parameter is mandatory.
    *   `visibility: 'user-only'` indicates that this parameter should be configured by the user.
    *   `description: 'ID of the Airtable base'` provides a helpful description of the parameter.

12. **`tableId: { type: 'string', required: true, visibility: 'user-only', description: 'ID or name of the table', },`**

    *   Defines the `tableId` parameter.
    *   `type: 'string'` specifies that the value must be a string.
    *   `required: true` indicates that this parameter is mandatory.
    *   `visibility: 'user-only'` indicates that this parameter should be configured by the user.
    *   `description: 'ID or name of the table'` provides a helpful description of the parameter.

13. **`recordId: { type: 'string', required: true, visibility: 'user-only', description: 'ID of the record to update', },`**

    *   Defines the `recordId` parameter.
    *   `type: 'string'` specifies that the value must be a string.
    *   `required: true` indicates that this parameter is mandatory.
    *   `visibility: 'user-only'` indicates that this parameter should be configured by the user.
    *   `description: 'ID of the record to update'` provides a helpful description of the parameter.

14. **`fields: { type: 'json', required: true, visibility: 'user-or-llm', description: 'An object containing the field names and their new values', },`**

    *   Defines the `fields` parameter.
    *   `type: 'json'` specifies that the value must be a JSON object.
    *   `required: true` indicates that this parameter is mandatory.
    *   `visibility: 'user-or-llm'` indicates that the tool can automatically determine the fields or the user can define them.
    *   `description: 'An object containing the field names and their new values'` provides a helpful description of the parameter.

15. **`request: { ... },`**

    *   Defines how to make the API request to Airtable.

16. **`url: (params) => \`https://api.airtable.com/v0/${params.baseId}/${params.tableId}/${params.recordId}\`,`**

    *   A function that dynamically constructs the URL for the Airtable API endpoint.
    *   It uses template literals to insert the `baseId`, `tableId`, and `recordId` from the `params` object into the URL.
    *   The API version is `v0`.

17. **`method: 'PATCH',`**

    *   Specifies the HTTP method to use for the request, which is `PATCH`. This is the correct method to update only certain fields on an Airtable record.

18. **`headers: (params) => ({ Authorization: \`Bearer ${params.accessToken}\`, 'Content-Type': 'application/json', }),`**

    *   A function that returns an object containing the headers for the request.
    *   `Authorization: \`Bearer ${params.accessToken}\`` sets the authorization header with the OAuth access token.  This is essential for authenticating with the Airtable API.
    *   `'Content-Type': 'application/json'` indicates that the request body will be in JSON format.

19. **`body: (params) => ({ fields: params.fields }),`**

    *   A function that constructs the request body.
    *   It includes the `fields` parameter in the body, which contains the field names and their new values.

20. **`transformResponse: async (response) => { ... },`**

    *   An asynchronous function that transforms the raw API response into a more usable format.

21. **`const data = await response.json()`**

    *   Parses the response body as JSON and assigns it to the `data` variable.

22. **`return { success: true, output: { record: data, metadata: { recordCount: 1, updatedFields: Object.keys(data.fields || {}), }, }, };`**

    *   Returns an object indicating the success of the operation and providing structured output.
    *   `success: true` indicates that the update was successful.
    *   `output` contains the processed data from the API response.
        *   `record: data` includes the complete, updated record received from the API.
        *   `metadata` provides additional information about the operation:
            *   `recordCount: 1` indicates that one record was updated.
            *   `updatedFields: Object.keys(data.fields || {})` extracts the names of the fields that were updated from the response data.  The `|| {}` part is a safeguard to prevent errors if the `data.fields` property is not present.

23. **`outputs: { ... },`**

    *   Defines the structure and descriptions of the output properties.

24. **`record: { type: 'json', description: 'Updated Airtable record with id, createdTime, and fields', },`**

    *   Describes the `record` output property.
    *   `type: 'json'` indicates that it is a JSON object.
    *   Provides a description of the property.

25. **`metadata: { type: 'json', description: 'Operation metadata including record count and updated field names', },`**

    *   Describes the `metadata` output property.
    *   `type: 'json'` indicates that it is a JSON object.
    *   Provides a description of the property.

### Summary

In essence, this code defines a reusable and well-structured configuration for updating records in Airtable. The `ToolConfig` provides a clear contract for defining the tool's inputs, how it interacts with the Airtable API, and how it presents the results. The use of types ensures that the configuration is valid and that the tool is used correctly.  The consistent structure makes it easier to integrate this tool into a larger system.
