```typescript
import type { ToolResponse } from '@/tools/types'

// Common types
export interface AirtableRecord {
  id: string
  createdTime: string
  fields: Record<string, any>
}

interface AirtableBaseParams {
  accessToken: string
  baseId: string
  tableId: string
}

// List Records Types
export interface AirtableListParams extends AirtableBaseParams {
  maxRecords?: number
  filterFormula?: string
}

export interface AirtableListResponse extends ToolResponse {
  output: {
    records: AirtableRecord[]
    metadata: {
      offset?: string
      totalRecords: number
    }
  }
}

// Get Record Types
export interface AirtableGetParams extends AirtableBaseParams {
  recordId: string
}

export interface AirtableGetResponse extends ToolResponse {
  output: {
    record: AirtableRecord
    metadata: {
      recordCount: 1
    }
  }
}

// Create Records Types
export interface AirtableCreateParams extends AirtableBaseParams {
  records: Array<{ fields: Record<string, any> }>
}

export interface AirtableCreateResponse extends ToolResponse {
  output: {
    records: AirtableRecord[]
    metadata: {
      recordCount: number
    }
  }
}

// Update Record Types (Single)
export interface AirtableUpdateParams extends AirtableBaseParams {
  recordId: string
  fields: Record<string, any>
}

export interface AirtableUpdateResponse extends ToolResponse {
  output: {
    record: AirtableRecord // Airtable returns the single updated record
    metadata: {
      recordCount: 1
      updatedFields: string[]
    }
  }
}

// Update Multiple Records Types
export interface AirtableUpdateMultipleParams extends AirtableBaseParams {
  records: Array<{ id: string; fields: Record<string, any> }>
}

export interface AirtableUpdateMultipleResponse extends ToolResponse {
  output: {
    records: AirtableRecord[] // Airtable returns the array of updated records
    metadata: {
      recordCount: number
      updatedRecordIds: string[]
    }
  }
}

export type AirtableResponse =
  | AirtableListResponse
  | AirtableGetResponse
  | AirtableCreateResponse
  | AirtableUpdateResponse
  | AirtableUpdateMultipleResponse
```

## Explanation of the Code: Airtable Types

This TypeScript file defines a comprehensive set of interfaces for interacting with the Airtable API. It covers common data structures, parameters for different operations (listing, getting, creating, and updating records), and the structure of the responses from Airtable.

**Purpose of this file:**

The primary goal of this file is to provide type safety and clarity when working with the Airtable API in a TypeScript project.  By defining these interfaces, developers can ensure that their code correctly handles Airtable data and that they are passing the correct parameters to the API.  This prevents runtime errors and improves the overall maintainability of the codebase.

**Breakdown of the Code:**

1.  **Import Statements:**

    ```typescript
    import type { ToolResponse } from '@/tools/types'
    ```

    *   This line imports the `ToolResponse` type from a local module `@/tools/types`.  It's likely that `ToolResponse` defines a standard wrapper or structure for responses from various "tools" within the application, potentially including error handling or status information alongside the actual data.  The `type` keyword signifies that we are importing a type, not a value.

2.  **Common Types:**

    ```typescript
    // Common types
    export interface AirtableRecord {
      id: string
      createdTime: string
      fields: Record<string, any>
    }

    interface AirtableBaseParams {
      accessToken: string
      baseId: string
      tableId: string
    }
    ```

    *   **`AirtableRecord`:** This interface represents a single record from an Airtable table.

        *   `id`:  The unique identifier of the record in Airtable (string).
        *   `createdTime`: A timestamp indicating when the record was created in Airtable (string).
        *   `fields`: A dictionary (Record) containing the data for each field in the record.
            *   `Record<string, any>` means that the keys are strings (field names), and the values can be of any type (`any`). This is a broad type and might be narrowed down in a real application for better type safety if the possible field types are known.

    *   **`AirtableBaseParams`:** This interface defines the common parameters required for most Airtable API operations.

        *   `accessToken`:  The API key used to authenticate with Airtable (string).
        *   `baseId`:  The ID of the Airtable base (analogous to a database) (string).
        *   `tableId`: The ID of the table within the Airtable base (string).

3.  **List Records Types:**

    ```typescript
    // List Records Types
    export interface AirtableListParams extends AirtableBaseParams {
      maxRecords?: number
      filterFormula?: string
    }

    export interface AirtableListResponse extends ToolResponse {
      output: {
        records: AirtableRecord[]
        metadata: {
          offset?: string
          totalRecords: number
        }
      }
    }
    ```

    *   **`AirtableListParams`:**  Specifies the parameters for listing records in an Airtable table. It *extends* `AirtableBaseParams`, meaning it inherits `accessToken`, `baseId`, and `tableId`.

        *   `maxRecords?`:  An optional parameter specifying the maximum number of records to retrieve in a single API call (number).  The `?` indicates that it's optional.
        *   `filterFormula?`: An optional parameter to filter the records based on an Airtable formula (string).

    *   **`AirtableListResponse`:** Defines the structure of the response returned when listing records.

        *   It extends `ToolResponse`, implying that the overall response includes metadata from the tool itself (e.g., status codes, error messages).
        *   `output`:  An object containing the actual data returned by Airtable.
            *   `records`: An array of `AirtableRecord` objects, representing the retrieved records.
            *   `metadata`: An object containing metadata about the list operation.
                *   `offset?`: An optional string that can be used to paginate through large result sets.  If present, it indicates that there are more records available.
                *   `totalRecords`: The total number of records in the table that match the filter (number).

4.  **Get Record Types:**

    ```typescript
    // Get Record Types
    export interface AirtableGetParams extends AirtableBaseParams {
      recordId: string
    }

    export interface AirtableGetResponse extends ToolResponse {
      output: {
        record: AirtableRecord
        metadata: {
          recordCount: 1
        }
      }
    }
    ```

    *   **`AirtableGetParams`:** Defines parameters for retrieving a *single* record from Airtable by its ID. It extends `AirtableBaseParams`.

        *   `recordId`:  The ID of the record to retrieve (string).

    *   **`AirtableGetResponse`:** Defines the structure of the response when getting a single record.

        *   Extends `ToolResponse`.
        *   `output`:
            *   `record`: A single `AirtableRecord` object.
            *   `metadata`:
                *   `recordCount`: Always 1, indicating that a single record was returned.

5.  **Create Records Types:**

    ```typescript
    // Create Records Types
    export interface AirtableCreateParams extends AirtableBaseParams {
      records: Array<{ fields: Record<string, any> }>
    }

    export interface AirtableCreateResponse extends ToolResponse {
      output: {
        records: AirtableRecord[]
        metadata: {
          recordCount: number
        }
      }
    }
    ```

    *   **`AirtableCreateParams`:** Defines the parameters for creating new records in Airtable. It extends `AirtableBaseParams`.

        *   `records`: An array of objects. Each object represents a record to be created and contains a `fields` property.
            *   `fields`:  A dictionary of field names and their corresponding values for the new record.

    *   **`AirtableCreateResponse`:** Defines the response structure after creating records.

        *   Extends `ToolResponse`.
        *   `output`:
            *   `records`: An array of `AirtableRecord` objects, representing the newly created records (including their IDs and `createdTime`).
            *   `metadata`:
                *   `recordCount`:  The number of records that were successfully created.

6.  **Update Record Types (Single):**

    ```typescript
    // Update Record Types (Single)
    export interface AirtableUpdateParams extends AirtableBaseParams {
      recordId: string
      fields: Record<string, any>
    }

    export interface AirtableUpdateResponse extends ToolResponse {
      output: {
        record: AirtableRecord // Airtable returns the single updated record
        metadata: {
          recordCount: 1
          updatedFields: string[]
        }
      }
    }
    ```

    *   **`AirtableUpdateParams`:** Defines parameters for updating a *single* record in Airtable. It extends `AirtableBaseParams`.

        *   `recordId`: The ID of the record to update (string).
        *   `fields`: A dictionary containing the field names and their *new* values for the update.

    *   **`AirtableUpdateResponse`:** Defines the response structure after updating a single record.

        *   Extends `ToolResponse`.
        *   `output`:
            *   `record`: The updated `AirtableRecord` object.
            *   `metadata`:
                *   `recordCount`: Always 1, indicating that one record was updated.
                *   `updatedFields`: An array of strings, listing the names of the fields that were actually modified during the update.

7.  **Update Multiple Records Types:**

    ```typescript
    // Update Multiple Records Types
    export interface AirtableUpdateMultipleParams extends AirtableBaseParams {
      records: Array<{ id: string; fields: Record<string, any> }>
    }

    export interface AirtableUpdateMultipleResponse extends ToolResponse {
      output: {
        records: AirtableRecord[] // Airtable returns the array of updated records
        metadata: {
          recordCount: number
          updatedRecordIds: string[]
        }
      }
    }
    ```

    *   **`AirtableUpdateMultipleParams`:** Defines parameters for updating *multiple* records in Airtable. It extends `AirtableBaseParams`.

        *   `records`: An array of objects. Each object represents a record to be updated and must include both `id` and `fields`.
            *   `id`: The ID of the record to update.
            *   `fields`: The fields to update for that specific record.

    *   **`AirtableUpdateMultipleResponse`:** Defines the response structure after updating multiple records.

        *   Extends `ToolResponse`.
        *   `output`:
            *   `records`: An array of `AirtableRecord` objects, representing the updated records.
            *   `metadata`:
                *   `recordCount`: The number of records that were successfully updated.
                *   `updatedRecordIds`: An array of strings, listing the IDs of the records that were updated.

8.  **Union Type for Responses:**

    ```typescript
    export type AirtableResponse =
      | AirtableListResponse
      | AirtableGetResponse
      | AirtableCreateResponse
      | AirtableUpdateResponse
      | AirtableUpdateMultipleResponse
    ```

    *   **`AirtableResponse`:** This is a *union type*.  It means that a variable of type `AirtableResponse` can be *any* of the response types defined above: `AirtableListResponse`, `AirtableGetResponse`, `AirtableCreateResponse`, `AirtableUpdateResponse`, or `AirtableUpdateMultipleResponse`.  This is useful for functions that can return different types of Airtable responses, depending on the operation performed.

**In Summary:**

This file provides a robust and type-safe foundation for interacting with the Airtable API in a TypeScript environment. It clearly defines the structure of requests and responses for various operations, promoting code clarity, reducing errors, and improving maintainability.  The use of interfaces and the union type `AirtableResponse` ensures that developers can confidently work with Airtable data while leveraging the benefits of TypeScript's type system.
