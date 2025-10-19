This TypeScript file defines a configuration for a "MongoDB Block" within a larger application, likely a workflow builder or a UI component library. Its primary purpose is to:

1.  **Describe a User Interface:** It specifies all the input fields a user needs to configure a MongoDB operation (like connection details, operation type, query, update, etc.). This includes labels, placeholders, dropdown options, and conditional visibility rules.
2.  **Integrate with Backend Tools:** It defines how the user's input from the UI translates into parameters for actual backend MongoDB API calls (e.g., `mongodb_query`, `mongodb_insert`).
3.  **Provide AI Assistance:** It embeds detailed prompts and guidelines for an AI assistant (`wandConfig`) to help users generate complex MongoDB queries, aggregation pipelines, and update operations in JSON format.
4.  **Define a Data Contract:** It explicitly lists the expected inputs the block accepts and the outputs it will produce, making it predictable for integration into workflows.

In essence, this file acts as a blueprint for a versatile MongoDB interaction component, allowing users to perform various database operations directly from a user interface, with smart assistance.

---

### Simplified Complex Logic

The most "complex" parts of this file involve:

1.  **Conditional UI Logic (`subBlocks`):** Many fields only appear based on the user's selection in the "Operation" dropdown. For example, "Query Filter" only shows if "Find Documents" is selected. This is handled by the `condition` property.
2.  **AI Integration (`wandConfig`):** For fields like `query`, `pipeline`, `sort`, `documents`, `filter`, and `update`, a `wandConfig` object is present. This embeds a comprehensive prompt that guides an AI model to generate correct MongoDB JSON syntax based on user descriptions. The prompt includes specific instructions, examples, security, and performance tips.
3.  **Parameter Transformation (`tools.config.params`):** The `params` function is crucial. It takes all the raw input from the UI (which can be strings, numbers, or even strings that need to be parsed as JSON) and transforms them into a clean, structured JavaScript object suitable for a backend MongoDB API call. This involves:
    *   Parsing string-based JSON inputs (`query`, `documents`, `pipeline`, `update`, `filter`, `sort`).
    *   Converting string numbers (`port`, `limit`) to actual numbers.
    *   Converting string booleans (`upsert`, `multi`) to actual booleans.
    *   Setting default values (`port`, `limit`, `ssl`, `operation`).
    *   Grouping connection details.

---

### Line-by-Line Explanation

```typescript
import { MongoDBIcon } from '@/components/icons'
```

*   **`import { MongoDBIcon } from '@/components/icons'`**: This line imports a visual component named `MongoDBIcon`. This icon will likely be used to represent the MongoDB block in the user interface (e.g., in a toolbox or a workflow diagram). The `@/components/icons` path suggests it's a relative import from a root `src` directory.

```typescript
import type { BlockConfig } from '@/blocks/types'
import type { MongoDBResponse } from '@/tools/mongodb/types'
```

*   **`import type { BlockConfig } from '@/blocks/types'`**: This imports the `BlockConfig` type definition. This type defines the overall structure and expected properties for any "block" configuration in the system, ensuring consistency across different types of blocks. The `type` keyword indicates that this is only used for type checking at compile time and is removed during compilation to JavaScript.
*   **`import type { MongoDBResponse } from '@/tools/mongodb/types'`**: This imports the `MongoDBResponse` type, which specifies the expected structure of the data returned after a MongoDB operation. This type is used to infer the structure of the block's `outputs`.

```typescript
export const MongoDBBlock: BlockConfig<MongoDBResponse> = {
```

*   **`export const MongoDBBlock: BlockConfig<MongoDBResponse> = {`**: This declares and exports a constant variable named `MongoDBBlock`. It's explicitly typed as `BlockConfig<MongoDBResponse>`, meaning it's a configuration for a block that, when executed, will produce data conforming to the `MongoDBResponse` type. This is the main definition of our MongoDB block.

    Let's break down the properties within this `MongoDBBlock` object:

```typescript
  type: 'mongodb',
  name: 'MongoDB',
  description: 'Connect to MongoDB database',
  longDescription:
    'Integrate MongoDB into the workflow. Can find, insert, update, delete, and aggregate data.',
  docsLink: 'https://docs.sim.ai/tools/mongodb',
  category: 'tools',
  bgColor: '#E0E0E0',
  icon: MongoDBIcon,
```

*   **`type: 'mongodb'`**: A unique identifier for this type of block. Used internally by the system to recognize and handle MongoDB operations.
*   **`name: 'MongoDB'`**: The human-readable name of the block, displayed in the UI.
*   **`description: 'Connect to MongoDB database'`**: A short, concise description shown in the UI.
*   **`longDescription: 'Integrate MongoDB into the workflow. Can find, insert, update, delete, and aggregate data.'`**: A more detailed description, possibly shown in a tooltip or details panel.
*   **`docsLink: 'https://docs.sim.ai/tools/mongodb'`**: A URL pointing to external documentation for this block.
*   **`category: 'tools'`**: Categorizes the block, useful for organizing blocks in a UI sidebar or search.
*   **`bgColor: '#E0E0E0'`**: Defines a background color for the block, likely for visual distinction in a workflow canvas.
*   **`icon: MongoDBIcon`**: Assigns the imported `MongoDBIcon` component as the visual representation for this block.

```typescript
  subBlocks: [
```

*   **`subBlocks: [`**: This property defines an array of "sub-blocks" or input fields that will appear within the main MongoDB block in the UI. These are the user-configurable elements.

    Each object in this array represents a single input field:

    ---

    **Operation Dropdown:**
    ```typescript
    {
      id: 'operation',
      title: 'Operation',
      type: 'dropdown',
      layout: 'full',
      options: [
        { label: 'Find Documents', id: 'query' },
        { label: 'Insert Documents', id: 'insert' },
        { label: 'Update Documents', id: 'update' },
        { label: 'Delete Documents', id: 'delete' },
        { label: 'Aggregate Pipeline', id: 'execute' },
      ],
      value: () => 'query',
    },
    ```
    *   **`id: 'operation'`**: A unique identifier for this input field.
    *   **`title: 'Operation'`**: The label displayed next to the dropdown in the UI.
    *   **`type: 'dropdown'`**: Specifies that this is a dropdown select input.
    *   **`layout: 'full'`**: Dictates how the input field should render, usually spanning the full width of its container.
    *   **`options: [...]`**: An array of objects, where each object defines a selectable option in the dropdown. `label` is what the user sees, `id` is the internal value associated with that selection.
        *   `{ label: 'Find Documents', id: 'query' }`
        *   `{ label: 'Insert Documents', id: 'insert' }`
        *   `{ label: 'Update Documents', id: 'update' }`
        *   `{ label: 'Delete Documents', id: 'delete' }`
        *   `{ label: 'Aggregate Pipeline', id: 'execute' }`
    *   **`value: () => 'query'`**: A function that returns the default selected value for this dropdown, which is `'query'` (Find Documents).

    ---

    **Host Input:**
    ```typescript
    {
      id: 'host',
      title: 'Host',
      type: 'short-input',
      layout: 'full',
      placeholder: 'localhost or your.mongodb.host',
      required: true,
    },
    ```
    *   **`id: 'host'`**: Identifier for the MongoDB host input.
    *   **`title: 'Host'`**: Label.
    *   **`type: 'short-input'`**: A standard single-line text input field.
    *   **`layout: 'full'`**: Full width layout.
    *   **`placeholder: 'localhost or your.mongodb.host'`**: Text displayed when the input is empty.
    *   **`required: true`**: Indicates that this field must be filled out by the user.

    ---

    **Port Input:**
    ```typescript
    {
      id: 'port',
      title: 'Port',
      type: 'short-input',
      layout: 'full',
      placeholder: '27017',
      value: () => '27017',
      required: true,
    },
    ```
    *   **`id: 'port'`**: Identifier for the MongoDB port input.
    *   **`title: 'Port'`**: Label.
    *   **`type: 'short-input'`**: Short text input.
    *   **`layout: 'full'`**: Full width.
    *   **`placeholder: '27017'`**: Placeholder text.
    *   **`value: () => '27017'`**: Default value for the port.
    *   **`required: true`**: This field is mandatory.

    ---

    **Database Name Input:**
    ```typescript
    {
      id: 'database',
      title: 'Database Name',
      type: 'short-input',
      layout: 'full',
      placeholder: 'your_database',
      required: true,
    },
    ```
    *   **`id: 'database'`**: Identifier for the database name.
    *   **`title: 'Database Name'`**: Label.
    *   **`type: 'short-input'`**: Short text input.
    *   **`layout: 'full'`**: Full width.
    *   **`placeholder: 'your_database'`**: Placeholder.
    *   **`required: true`**: Mandatory.

    ---

    **Username Input:**
    ```typescript
    {
      id: 'username',
      title: 'Username',
      type: 'short-input',
      layout: 'full',
      placeholder: 'mongodb_user',
      required: true,
    },
    ```
    *   **`id: 'username'`**: Identifier for the username.
    *   **`title: 'Username'`**: Label.
    *   **`type: 'short-input'`**: Short text input.
    *   **`layout: 'full'`**: Full width.
    *   **`placeholder: 'mongodb_user'`**: Placeholder.
    *   **`required: true`**: Mandatory.

    ---

    **Password Input:**
    ```typescript
    {
      id: 'password',
      title: 'Password',
      type: 'short-input',
      layout: 'full',
      password: true, // Key property for password fields
      placeholder: 'Your database password',
      required: true,
    },
    ```
    *   **`id: 'password'`**: Identifier for the password.
    *   **`title: 'Password'`**: Label.
    *   **`type: 'short-input'`**: Short text input.
    *   **`layout: 'full'`**: Full width.
    *   **`password: true`**: This crucial property indicates that the input should be masked (e.g., with asterisks or dots) for security when typed.
    *   **`placeholder: 'Your database password'`**: Placeholder.
    *   **`required: true`**: Mandatory.

    ---

    **Auth Source Input:**
    ```typescript
    {
      id: 'authSource',
      title: 'Auth Source',
      type: 'short-input',
      layout: 'full',
      placeholder: 'admin',
    },
    ```
    *   **`id: 'authSource'`**: Identifier for the authentication source database.
    *   **`title: 'Auth Source'`**: Label.
    *   **`type: 'short-input'`**: Short text input.
    *   **`layout: 'full'`**: Full width.
    *   **`placeholder: 'admin'`**: Placeholder.
    *   **`required` is absent**: This field is optional.

    ---

    **SSL Mode Dropdown:**
    ```typescript
    {
      id: 'ssl',
      title: 'SSL Mode',
      type: 'dropdown',
      layout: 'full',
      options: [
        { label: 'Disabled', id: 'disabled' },
        { label: 'Required', id: 'required' },
        { label: 'Preferred', id: 'preferred' },
      ],
      value: () => 'preferred',
    },
    ```
    *   **`id: 'ssl'`**: Identifier for SSL mode selection.
    *   **`title: 'SSL Mode'`**: Label.
    *   **`type: 'dropdown'`**: Dropdown input.
    *   **`layout: 'full'`**: Full width.
    *   **`options: [...]`**: Defines the SSL options: `Disabled`, `Required`, `Preferred`.
    *   **`value: () => 'preferred'`**: Default SSL mode is 'preferred'.

    ---

    **Collection Name Input:**
    ```typescript
    {
      id: 'collection',
      title: 'Collection Name',
      type: 'short-input',
      layout: 'full',
      placeholder: 'users',
      required: true,
    },
    ```
    *   **`id: 'collection'`**: Identifier for the collection name.
    *   **`title: 'Collection Name'`**: Label.
    *   **`type: 'short-input'`**: Short text input.
    *   **`layout: 'full'`**: Full width.
    *   **`placeholder: 'users'`**: Placeholder.
    *   **`required: true`**: Mandatory.

    ---

    **Query Filter (JSON) Input (Conditional):**
    ```typescript
    {
      id: 'query',
      title: 'Query Filter (JSON)',
      type: 'code', // This indicates a code editor/text area expecting code
      layout: 'full',
      placeholder: '{"status": "active"}',
      condition: { field: 'operation', value: 'query' }, // ONLY show if 'operation' is 'query'
      wandConfig: { // Configuration for AI assistance
        enabled: true,
        maintainHistory: true,
        prompt: `You are an expert MongoDB developer. Generate MongoDB query filters as JSON objects based on the user's request. ...`,
        placeholder: 'Describe the documents you want to find...',
        generationType: 'mongodb-filter',
      },
    },
    ```
    *   **`id: 'query'`**: Identifier for the query filter input.
    *   **`title: 'Query Filter (JSON)'`**: Label, indicating JSON format is expected.
    *   **`type: 'code'`**: Specifies a specialized input component, likely a multi-line text area with syntax highlighting, for entering code/JSON.
    *   **`layout: 'full'`**: Full width.
    *   **`placeholder: '{"status": "active"}'`**: Example JSON.
    *   **`condition: { field: 'operation', value: 'query' }`**: **Crucial for UI logic.** This field will *only* be visible if the `operation` dropdown (defined earlier) has its value set to `'query'` (i.e., "Find Documents").
    *   **`wandConfig: { ... }`**: This object configures an AI "wand" or assistant feature.
        *   **`enabled: true`**: Turns on the AI assistance for this field.
        *   **`maintainHistory: true`**: Indicates the AI should remember previous interactions/context.
        *   **`prompt: `...` `**: This is a multi-line string containing the detailed instructions for the AI. It guides the AI to act as a MongoDB expert, generate JSON query filters, provides context placeholders (`{context}`), critical output format instructions, query guidelines, a comprehensive list of MongoDB operators, various examples, security, and performance tips.
        *   **`placeholder: 'Describe the documents you want to find...' `**: A specific placeholder for the AI input box.
        *   **`generationType: 'mongodb-filter'`**: A specific identifier used by the AI service to determine which model or logic to apply for generating this type of content.

    ---

    **Aggregation Pipeline (JSON Array) Input (Conditional):**
    ```typescript
    {
      id: 'pipeline',
      title: 'Aggregation Pipeline (JSON Array)',
      type: 'code',
      layout: 'full',
      placeholder: '[{"$group": {"_id": "$status", "count": {"$sum": 1}}}]',
      condition: { field: 'operation', value: 'execute' }, // ONLY show if 'operation' is 'execute'
      required: true,
      wandConfig: { // AI configuration for aggregation pipelines
        enabled: true,
        maintainHistory: true,
        prompt: `You are an expert MongoDB aggregation developer. Create MongoDB aggregation pipelines based on the user's request. ...`,
        placeholder: 'Describe the aggregation you want to perform...',
        generationType: 'mongodb-pipeline',
      },
    },
    ```
    *   Similar to `query`, but for MongoDB aggregation pipelines.
    *   **`condition: { field: 'operation', value: 'execute' }`**: Only visible when the `operation` is set to `'execute'` (i.e., "Aggregate Pipeline").
    *   **`required: true`**: Mandatory if visible.
    *   The `wandConfig.prompt` is tailored specifically for generating MongoDB aggregation pipelines, with different guidelines, stages, and examples.
    *   **`generationType: 'mongodb-pipeline'`**: Specific AI generation type.

    ---

    **Limit Input (Conditional):**
    ```typescript
    {
      id: 'limit',
      title: 'Limit',
      type: 'short-input',
      layout: 'full',
      placeholder: '100',
      condition: { field: 'operation', value: 'query' }, // ONLY show if 'operation' is 'query'
    },
    ```
    *   **`id: 'limit'`**: Identifier for the result limit.
    *   **`condition: { field: 'operation', value: 'query' }`**: Only visible when the `operation` is set to `'query'` ("Find Documents").

    ---

    **Sort (JSON) Input (Conditional):**
    ```typescript
    {
      id: 'sort',
      title: 'Sort (JSON)',
      type: 'code',
      layout: 'full',
      placeholder: '{"createdAt": -1}',
      condition: { field: 'operation', value: 'query' }, // ONLY show if 'operation' is 'query'
      wandConfig: { // AI configuration for sort criteria
        enabled: true,
        maintainHistory: true,
        prompt: `Write MongoDB sort criteria as JSON. ...`,
        placeholder: 'Describe how you want to sort the results...',
        generationType: 'mongodb-sort',
      },
    },
    ```
    *   **`id: 'sort'`**: Identifier for sort criteria.
    *   **`condition: { field: 'operation', value: 'query' }`**: Only visible when the `operation` is set to `'query'` ("Find Documents").
    *   The `wandConfig.prompt` is tailored for MongoDB sort criteria.
    *   **`generationType: 'mongodb-sort'`**: Specific AI generation type.

    ---

    **Documents (JSON Array) Input (Conditional):**
    ```typescript
    {
      id: 'documents',
      title: 'Documents (JSON Array)',
      type: 'code',
      layout: 'full',
      placeholder: '[{"name": "John Doe", "email": "john@example.com", "status": "active"}]',
      condition: { field: 'operation', value: 'insert' }, // ONLY show if 'operation' is 'insert'
      required: true,
      wandConfig: { // AI configuration for documents to insert
        enabled: true,
        maintainHistory: true,
        prompt: `Write MongoDB documents as JSON array. ...`,
        placeholder: 'Describe the documents you want to insert...',
        generationType: 'mongodb-documents',
      },
    },
    ```
    *   **`id: 'documents'`**: Identifier for documents to insert.
    *   **`condition: { field: 'operation', value: 'insert' }`**: Only visible when the `operation` is set to `'insert'` ("Insert Documents").
    *   **`required: true`**: Mandatory if visible.
    *   The `wandConfig.prompt` is tailored for generating JSON arrays of documents.
    *   **`generationType: 'mongodb-documents'`**: Specific AI generation type.

    ---

    **Filter (JSON) Input (Conditional, for Update):**
    ```typescript
    {
      id: 'filter',
      title: 'Filter (JSON)',
      type: 'code',
      layout: 'full',
      placeholder: '{"name": "Alice Test"}',
      condition: { field: 'operation', value: 'update' }, // ONLY show if 'operation' is 'update'
      required: true,
      wandConfig: { // AI configuration for update filters
        enabled: true,
        maintainHistory: true,
        prompt: `You are an expert MongoDB developer. Generate MongoDB query filters as JSON objects to target specific documents for UPDATE operations. ...`,
        placeholder: 'Describe which documents to update...',
        generationType: 'mongodb-filter',
      },
    },
    ```
    *   **`id: 'filter'`**: Identifier for the filter used in update operations.
    *   **`condition: { field: 'operation', value: 'update' }`**: Only visible when the `operation` is set to `'update'` ("Update Documents").
    *   **`required: true`**: Mandatory if visible.
    *   The `wandConfig.prompt` is specifically designed for generating filters for *update* operations, with an emphasis on precision and safety.
    *   **`generationType: 'mongodb-filter'`**: Reuses the 'mongodb-filter' generation type, implying the core AI logic for filters is shared, but the prompt guides it for update-specific use cases.

    ---

    **Update (JSON) Input (Conditional):**
    ```typescript
    {
      id: 'update',
      title: 'Update (JSON)',
      type: 'code',
      layout: 'full',
      placeholder: '{"$set": {"name": "Jane Doe", "email": "jane@example.com"}}',
      condition: { field: 'operation', value: 'update' }, // ONLY show if 'operation' is 'update'
      required: true,
      wandConfig: { // AI configuration for update operations
        enabled: true,
        maintainHistory: true,
        prompt: `You are an expert MongoDB developer. Generate ONLY the raw JSON update operation based on the user's request. ...`,
        placeholder: 'Describe what you want to update...',
        generationType: 'mongodb-update',
      },
    },
    ```
    *   **`id: 'update'`**: Identifier for the update operation definition.
    *   **`condition: { field: 'operation', value: 'update' }`**: Only visible when the `operation` is set to `'update'` ("Update Documents").
    *   **`required: true`**: Mandatory if visible.
    *   The `wandConfig.prompt` is extensively detailed for generating MongoDB update operators and their syntax.
    *   **`generationType: 'mongodb-update'`**: Specific AI generation type.

    ---

    **Upsert Dropdown (Conditional):**
    ```typescript
    {
      id: 'upsert',
      title: 'Upsert',
      type: 'dropdown',
      layout: 'full',
      options: [
        { label: 'False', id: 'false' },
        { label: 'True', id: 'true' },
      ],
      value: () => 'false',
      condition: { field: 'operation', value: 'update' }, // ONLY show if 'operation' is 'update'
    },
    ```
    *   **`id: 'upsert'`**: Identifier for the upsert option.
    *   **`condition: { field: 'operation', value: 'update' }`**: Only visible when the `operation` is set to `'update'` ("Update Documents").
    *   **`value: () => 'false'`**: Default value is 'False'.

    ---

    **Update Multiple Dropdown (Conditional):**
    ```typescript
    {
      id: 'multi',
      title: 'Update Multiple',
      type: 'dropdown',
      layout: 'full',
      options: [
        { label: 'False', id: 'false' },
        { label: 'True', id: 'true' },
      ],
      value: () => 'false',
      condition: { field: 'operation', value: 'update' }, // ONLY show if 'operation' is 'update'
    },
    ```
    *   **`id: 'multi'`**: Identifier for the "update multiple" option.
    *   **`condition: { field: 'operation', value: 'update' }`**: Only visible when the `operation` is set to `'update'` ("Update Documents").
    *   **`value: () => 'false'`**: Default value is 'False'.

    ---

    **Filter (JSON) Input (Conditional, for Delete):**
    ```typescript
    {
      id: 'filter', // Reused ID, but distinct condition and wandConfig
      title: 'Filter (JSON)',
      type: 'code',
      layout: 'full',
      placeholder: '{"status": "inactive"}',
      condition: { field: 'operation', value: 'delete' }, // ONLY show if 'operation' is 'delete'
      required: true,
      wandConfig: { // AI configuration for delete filters
        enabled: true,
        maintainHistory: true,
        prompt: `You are an expert MongoDB developer. Generate MongoDB query filters as JSON objects to target specific documents for DELETION operations. ...`,
        placeholder: 'Describe which documents to delete...',
        generationType: 'mongodb-filter',
      },
    },
    ```
    *   **`id: 'filter'`**: This `id` is reused, but its visibility (`condition`) and AI prompt (`wandConfig`) are specific to delete operations. This means the UI system can handle multiple fields with the same `id` as long as their conditions prevent them from being active simultaneously.
    *   **`condition: { field: 'operation', value: 'delete' }`**: Only visible when the `operation` is set to `'delete'` ("Delete Documents").
    *   **`required: true`**: Mandatory if visible.
    *   The `wandConfig.prompt` is explicitly tailored for *delete* operations, emphasizing safety and warnings about permanence.
    *   **`generationType: 'mongodb-filter'`**: Reuses the 'mongodb-filter' generation type.

    ---

    **Delete Multiple Dropdown (Conditional):**
    ```typescript
    {
      id: 'multi', // Reused ID
      title: 'Delete Multiple',
      type: 'dropdown',
      layout: 'full',
      options: [
        { label: 'False', id: 'false' },
        { label: 'True', id: 'true' },
      ],
      value: () => 'false',
      condition: { field: 'operation', value: 'delete' }, // ONLY show if 'operation' is 'delete'
    },
  ], // End of subBlocks array
    ```
    *   **`id: 'multi'`**: This `id` is also reused, similar to `filter`.
    *   **`condition: { field: 'operation', value: 'delete' }`**: Only visible when the `operation` is set to `'delete'` ("Delete Documents").
    *   **`value: () => 'false'`**: Default value is 'False'.

```typescript
  tools: {
```

*   **`tools: {`**: This section defines how the block interacts with backend "tools" or APIs. It's the bridge between the UI configuration and actual execution.

```typescript
    access: [
      'mongodb_query',
      'mongodb_insert',
      'mongodb_update',
      'mongodb_delete',
      'mongodb_execute',
    ],
```

*   **`access: [...]`**: An array listing the specific backend tool names (or permissions) that this block might invoke. This is likely used for authorization or to inform the system what capabilities this block requires.

```typescript
    config: {
```

*   **`config: {`**: This object holds the logic for configuring the backend tool call based on user input.

```typescript
      tool: (params) => {
        switch (params.operation) {
          case 'query':
            return 'mongodb_query'
          case 'insert':
            return 'mongodb_insert'
          case 'update':
            return 'mongodb_update'
          case 'delete':
            return 'mongodb_delete'
          case 'execute':
            return 'mongodb_execute'
          default:
            throw new Error(`Invalid MongoDB operation: ${params.operation}`)
        }
      },
```

*   **`tool: (params) => { ... }`**: This is a function that determines *which specific backend MongoDB tool* to call, based on the `operation` selected by the user.
    *   **`params`**: An object containing all the input values from the `subBlocks`.
    *   **`switch (params.operation)`**: It uses a switch statement to check the value of the `operation` input.
    *   **`case 'query': return 'mongodb_query'`**: If the user selected 'Find Documents' (id 'query'), it returns the tool name `mongodb_query`.
    *   Similarly for `insert`, `update`, `delete`, and `execute` (aggregate).
    *   **`default: throw new Error(...)`**: If an unknown operation is encountered (which shouldn't happen with dropdowns, but good for robustness), it throws an error.

```typescript
      params: (params) => {
        const { operation, documents, ...rest } = params
```

*   **`params: (params) => { ... }`**: This is another function that takes all the raw input `params` from the UI and transforms them into a structured object that the selected backend MongoDB tool expects. This is where most of the input sanitization and structuring happens.
    *   **`const { operation, documents, ...rest } = params`**: Uses object destructuring to extract `operation` and `documents` (which need special handling) from the `params` object, putting all other parameters into a `rest` object.

```typescript
        let parsedDocuments
        if (documents && typeof documents === 'string' && documents.trim()) {
          try {
            parsedDocuments = JSON.parse(documents)
          } catch (parseError) {
            const errorMsg = parseError instanceof Error ? parseError.message : 'Unknown JSON error'
            throw new Error(
              `Invalid JSON documents format: ${errorMsg}. Please check your JSON syntax.`
            )
          }
        } else if (documents && typeof documents === 'object') {
          parsedDocuments = documents
        }
```

*   **`let parsedDocuments`**: Declares a variable to hold the parsed `documents`.
*   **`if (documents && typeof documents === 'string' && documents.trim()) { ... }`**: Checks if `documents` exists, is a non-empty string (meaning it came from a code input field).
    *   **`try { parsedDocuments = JSON.parse(documents) } catch (parseError) { ... }`**: Attempts to parse the `documents` string as JSON. If it fails, it catches the `parseError` and throws a more descriptive error message to the user.
*   **`else if (documents && typeof documents === 'object') { parsedDocuments = documents }`**: If `documents` already exists and is an object (it might come from a prior block's output, for instance), it's used directly without parsing.

```typescript
        const connectionConfig = {
          host: rest.host,
          port: typeof rest.port === 'string' ? Number.parseInt(rest.port, 10) : rest.port || 27017,
          database: rest.database,
          username: rest.username,
          password: rest.password,
          authSource: rest.authSource,
          ssl: rest.ssl || 'preferred',
        }
```

*   **`const connectionConfig = { ... }`**: Creates an object to hold all the MongoDB connection parameters.
    *   **`host: rest.host`**: Assigns the host.
    *   **`port: typeof rest.port === 'string' ? Number.parseInt(rest.port, 10) : rest.port || 27017`**: This line handles the port input:
        *   If `rest.port` is a string (from a text input), it parses it into an integer using `Number.parseInt(..., 10)`.
        *   Otherwise (if it's already a number or undefined), it uses `rest.port` itself.
        *   `|| 27017`: Provides a default value of `27017` if `rest.port` is `null`, `undefined`, or `0`.
    *   **`database: rest.database`**: Assigns the database name.
    *   **`username: rest.username`**: Assigns the username.
    *   **`password: rest.password`**: Assigns the password.
    *   **`authSource: rest.authSource`**: Assigns the authentication source.
    *   **`ssl: rest.ssl || 'preferred'`**: Assigns the SSL mode, defaulting to `'preferred'` if not explicitly provided.

```typescript
        const result: any = { ...connectionConfig }
```

*   **`const result: any = { ...connectionConfig }`**: Initializes the final `result` object (which will be sent to the backend tool). It starts by spreading all properties from `connectionConfig` into `result`. `any` is used because the exact shape of `result` depends dynamically on the `operation` and other inputs.

```typescript
        if (rest.collection) result.collection = rest.collection
        if (rest.query) {
          result.query = typeof rest.query === 'string' ? rest.query : JSON.stringify(rest.query)
        }
        if (rest.limit && rest.limit !== '') {
          result.limit =
            typeof rest.limit === 'string' ? Number.parseInt(rest.limit, 10) : rest.limit
        } else {
          result.limit = 100 // Default to 100 if not provided
        }
        if (rest.sort) {
          result.sort = typeof rest.sort === 'string' ? rest.sort : JSON.stringify(rest.sort)
        }
        if (rest.filter) {
          result.filter =
            typeof rest.filter === 'string' ? rest.filter : JSON.stringify(rest.filter)
        }
        if (rest.update) {
          result.update =
            typeof rest.update === 'string' ? rest.update : JSON.stringify(rest.update)
        }
        if (rest.pipeline) {
          result.pipeline =
            typeof rest.pipeline === 'string' ? rest.pipeline : JSON.stringify(rest.pipeline)
        }
        if (rest.upsert) result.upsert = rest.upsert === 'true' || rest.upsert === true
        if (rest.multi) result.multi = rest.multi === 'true' || rest.multi === true
        if (parsedDocuments !== undefined) result.documents = parsedDocuments
```

*   These lines conditionally add other parameters to the `result` object based on whether they were provided by the user.
    *   **`if (rest.collection) result.collection = rest.collection`**: Adds the `collection` name if it exists.
    *   **`if (rest.query) { result.query = typeof rest.query === 'string' ? rest.query : JSON.stringify(rest.query) }`**: Handles the `query` parameter. If it's a string, it's used directly. If it's an object (perhaps from a previous block's output), it's converted to a JSON string.
    *   **`if (rest.limit && rest.limit !== '') { ... } else { result.limit = 100 }`**: Handles the `limit` parameter, parsing it to an integer if it's a string, and setting a default of `100` if it's not provided or empty.
    *   **`if (rest.sort) { ... }`**: Similar string/object handling as `query` for the `sort` parameter.
    *   **`if (rest.filter) { ... }`**: Similar handling for the `filter` parameter.
    *   **`if (rest.update) { ... }`**: Similar handling for the `update` parameter.
    *   **`if (rest.pipeline) { ... }`**: Similar handling for the `pipeline` parameter.
    *   **`if (rest.upsert) result.upsert = rest.upsert === 'true' || rest.upsert === true`**: Converts the `upsert` input (which could be a string `'true'`/`'false'` from a dropdown or a boolean from another source) into a proper boolean.
    *   **`if (rest.multi) result.multi = rest.multi === 'true' || rest.multi === true`**: Same conversion for the `multi` parameter.
    *   **`if (parsedDocuments !== undefined) result.documents = parsedDocuments`**: Adds the `parsedDocuments` (from the earlier parsing logic) if they were present.

```typescript
        return result
      },
    },
  },
```

*   **`return result`**: The function returns the fully constructed `result` object, ready to be sent to the backend MongoDB tool.
*   **`}, }, },`**: Closes the `params` function, `config` object, and `tools` object.

```typescript
  inputs: {
```

*   **`inputs: {`**: This object formally declares the inputs that this block *accepts*. This is useful for external systems (like a workflow orchestrator) to understand the data contract. Each property corresponds to an `id` from `subBlocks` or other dynamic inputs.

```typescript
    operation: { type: 'string', description: 'Database operation to perform' },
    host: { type: 'string', description: 'MongoDB host' },
    port: { type: 'string', description: 'MongoDB port' },
    database: { type: 'string', description: 'Database name' },
    username: { type: 'string', description: 'MongoDB username' },
    password: { type: 'string', description: 'MongoDB password' },
    authSource: { type: 'string', description: 'Authentication database' },
    ssl: { type: 'string', description: 'SSL mode' },
    collection: { type: 'string', description: 'Collection name' },
    query: { type: 'string', description: 'Query filter as JSON string' },
    limit: { type: 'number', description: 'Limit number of documents' },
    sort: { type: 'string', description: 'Sort criteria as JSON string' },
    documents: { type: 'json', description: 'Documents to insert' },
    filter: { type: 'string', description: 'Filter criteria as JSON string' },
    update: { type: 'string', description: 'Update operations as JSON string' },
    pipeline: { type: 'string', description: 'Aggregation pipeline as JSON string' },
    upsert: { type: 'boolean', description: 'Create document if not found' },
    multi: { type: 'boolean', description: 'Operate on multiple documents' },
  },
```

*   Each entry defines an input:
    *   **`operation: { type: 'string', description: 'Database operation to perform' }`**: The type is `string` and a description.
    *   **`query: { type: 'string', description: 'Query filter as JSON string' }`**: Even though the UI field is `type: 'code'`, the *input expected by the workflow* is a `string` (because it's JSON *stringified*).
    *   **`limit: { type: 'number', description: 'Limit number of documents' }`**: Explicitly typed as `number` (after parsing by the `params` function).
    *   **`documents: { type: 'json', description: 'Documents to insert' }`**: Declares `documents` as `json` type, which means it expects a JSON object or array (parsed from string if necessary).
    *   **`upsert: { type: 'boolean', description: 'Create document if not found' }`**: Declares `upsert` as a boolean.
    *   The remaining inputs are similarly defined with their respective `type` and `description`.

```typescript
  outputs: {
```

*   **`outputs: {`**: This object formally declares the possible outputs that this block will produce after execution. This corresponds to the `MongoDBResponse` type imported at the top.

```typescript
    message: {
      type: 'string',
      description: 'Success or error message describing the operation outcome',
    },
    documents: {
      type: 'array',
      description: 'Array of documents returned from the operation',
    },
    documentCount: {
      type: 'number',
      description: 'Number of documents affected by the operation',
    },
    insertedId: {
      type: 'string',
      description: 'ID of the inserted document (single insert)',
    },
    insertedIds: {
      type: 'array',
      description: 'Array of IDs for inserted documents (multiple insert)',
    },
    modifiedCount: {
      type: 'number',
      description: 'Number of documents modified (update operations)',
    },
    deletedCount: {
      type: 'number',
      description: 'Number of documents deleted (delete operations)',
    },
    matchedCount: {
      type: 'number',
      description: 'Number of documents matched (update operations)',
    },
  },
}
```

*   Each entry defines a possible output:
    *   **`message: { type: 'string', description: '...' }`**: A string message.
    *   **`documents: { type: 'array', description: '...' }`**: An array of documents (e.g., from a find or aggregate operation).
    *   **`documentCount: { type: 'number', description: '...' }`**: A number indicating affected documents.
    *   **`insertedId: { type: 'string', description: '...' }`**: String ID for a single insert.
    *   **`insertedIds: { type: 'array', description: '...' }`**: Array of IDs for multiple inserts.
    *   **`modifiedCount: { type: 'number', description: '...' }`**: Number of modified documents.
    *   **`deletedCount: { type: 'number', description: '...' }`**: Number of deleted documents.
    *   **`matchedCount: { type: 'number', description: '...' }`**: Number of matched documents (for updates).
*   **`}, }`**: Closes the `outputs` object and the entire `MongoDBBlock` configuration object.

This detailed breakdown covers every aspect of the `mongodb.ts` file, from its overall purpose and simplified logic to a line-by-line explanation of its properties and functions.