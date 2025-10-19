This TypeScript file is a configuration blueprint for a "Notion Block" within a larger platform or workflow automation system (like the one suggested by `sim.ai` in the `docsLink`).

Think of it as defining how an integration with Notion should appear in a user interface, what operations it supports, how it authenticates, what inputs it takes for each operation, and what outputs it provides. It acts as a declarative description of the Notion tool's capabilities and its user-facing configuration.

### Simplified Complex Logic: Dynamic User Interface and Data Transformation

The most sophisticated aspects of this configuration are:

1.  **Dynamic Input Fields (`subBlocks` with `condition`):** Instead of showing all possible input fields for every Notion operation, the configuration defines rules (`condition`) that control which fields are visible. For example, if you select "Read Page," only the "Page ID" field appears; "Database ID" remains hidden. This creates a clean and user-friendly experience.

2.  **Data Transformation (`tools.config.params`):** When a user enters data into the UI, it often comes as simple strings (e.g., a JSON string for filters). However, the backend Notion API might expect these as parsed JavaScript objects or specifically formatted JSON strings. The `params` function handles this critical transformation, including parsing JSON strings and adding error handling. It ensures the data provided by the user is correctly formatted before being sent to the Notion integration's backend.

---

### Detailed Code Explanation

Let's break down the code line by line.

#### Imports

```typescript
import { NotionIcon } from '@/components/icons'
import type { BlockConfig } from '@/blocks/types'
import { AuthMode } from '@/blocks/types'
import type { NotionResponse } from '@/tools/notion/types'
```

These lines import necessary types and components from other parts of the application:

*   `import { NotionIcon } from '@/components/icons'`: Imports a visual React component (`NotionIcon`) that will be used to represent the Notion block in the UI.
*   `import type { BlockConfig } from '@/blocks/types'`: Imports the TypeScript type definition for `BlockConfig`. This type defines the expected structure and properties of any block configuration object, ensuring consistency. The `type` keyword means it's only used for type checking and won't be bundled in the final JavaScript output.
*   `import { AuthMode } from '@/blocks/types'`: Imports an enum (`AuthMode`) that defines different authentication methods supported by blocks. Here, it's used to specify that Notion uses OAuth.
*   `import type { NotionResponse } from '@/tools/notion/types'`: Imports the TypeScript type definition for `NotionResponse`, which describes the structure of data expected to be returned when interacting with the Notion API.

#### The `NotionBlock` Configuration Object

```typescript
export const NotionBlock: BlockConfig<NotionResponse> = {
  // ... configuration details ...
}
```

This is the main declaration. It defines and exports a constant named `NotionBlock`. It's explicitly typed as `BlockConfig<NotionResponse>`, meaning it must conform to the `BlockConfig` structure and its operations will ultimately produce data of type `NotionResponse`.

---

#### Core Block Properties

```typescript
  type: 'notion',
  name: 'Notion',
  description: 'Manage Notion pages',
  authMode: AuthMode.OAuth,
  longDescription:
    'Integrate with Notion into the workflow. Can read page, read database, create page, create database, append content, query database, and search workspace.',
  docsLink: 'https://docs.sim.ai/tools/notion',
  category: 'tools',
  bgColor: '#181C1E',
  icon: NotionIcon,
```

These properties define the fundamental characteristics of the Notion block:

*   `type: 'notion'`: A unique identifier for this specific block type within the platform.
*   `name: 'Notion'`: The human-readable name displayed for the block in the UI.
*   `description: 'Manage Notion pages'`: A concise summary of what the block does.
*   `authMode: AuthMode.OAuth`: Specifies that this block uses OAuth (Open Authorization) for connecting to Notion, typically requiring users to grant permissions through a Notion authorization flow.
*   `longDescription: '...'`: A more detailed explanation of the block's capabilities, listing all the supported Notion operations (read, create, append, query, search).
*   `docsLink: 'https://docs.sim.ai/tools/notion'`: A URL pointing to external documentation for this Notion integration.
*   `category: 'tools'`: Categorizes this block under "tools," likely for organizational purposes in a block palette or library.
*   `bgColor: '#181C1E'`: Defines a background color for the block's visual representation in the UI.
*   `icon: NotionIcon`: Assigns the `NotionIcon` component imported earlier as the visual icon for the block.

---

#### `subBlocks`: Defining the User Interface

The `subBlocks` array is crucial for defining the interactive elements (input fields, dropdowns, etc.) that users will see and interact with when configuring the Notion block. Each object in this array represents a distinct UI control.

```typescript
  subBlocks: [
    {
      id: 'operation',
      title: 'Operation',
      type: 'dropdown',
      layout: 'full',
      options: [
        { label: 'Read Page', id: 'notion_read' },
        // ... other operation options ...
      ],
      value: () => 'notion_read',
    },
    // ... other subBlocks ...
  ],
```

*   **Common Properties for `subBlocks`:**
    *   `id`: A unique identifier for this specific UI element.
    *   `title`: The label displayed to the user for this element.
    *   `type`: The type of UI control (e.g., `dropdown`, `short-input`, `long-input`, `oauth-input`).
    *   `layout`: Defines how the element should be arranged in the UI (e.g., `full` width).
    *   `placeholder`: Text displayed inside an input field when it's empty, providing a hint.
    *   `required`: A boolean indicating if the field must be filled by the user.
    *   `condition`: **(Crucial!)** This property is an object that specifies when this `subBlock` should be visible. It typically checks the value of another field. For example, `{ field: 'operation', value: 'notion_read' }` means this element will only appear if the `operation` dropdown is set to "Read Page."

Let's go through the key `subBlocks`:

1.  **Operation Dropdown:**
    ```typescript
    {
      id: 'operation',
      title: 'Operation',
      type: 'dropdown',
      layout: 'full',
      options: [
        { label: 'Read Page', id: 'notion_read' },
        { label: 'Read Database', id: 'notion_read_database' },
        { label: 'Create Page', id: 'notion_create_page' },
        { label: 'Create Database', id: 'notion_create_database' },
        { label: 'Append Content', id: 'notion_write' },
        { label: 'Query Database', id: 'notion_query_database' },
        { label: 'Search Workspace', id: 'notion_search' },
      ],
      value: () => 'notion_read',
    },
    ```
    This `dropdown` allows users to select which specific Notion action they want to perform.
    *   `options`: An array of objects, each representing a choice in the dropdown with a `label` for display and an `id` for internal identification.
    *   `value: () => 'notion_read'`: Sets the default selected operation to "Read Page."

2.  **Credential Input:**
    ```typescript
    {
      id: 'credential',
      title: 'Notion Account',
      type: 'oauth-input',
      layout: 'full',
      provider: 'notion',
      serviceId: 'notion',
      requiredScopes: ['workspace.content', 'workspace.name', 'page.read', 'page.write'],
      placeholder: 'Select Notion account',
      required: true,
    },
    ```
    This `oauth-input` is for connecting or selecting an authenticated Notion account.
    *   `provider` and `serviceId`: Identify the OAuth service being used.
    *   `requiredScopes`: An array of permissions (`scopes`) that the Notion integration needs from the user's Notion workspace (e.g., read/write pages, access workspace name).

3.  **Operation-Specific Input Fields:**

    Many `subBlocks` define input fields (`short-input`, `long-input`) that only appear based on the `operation` selected. This is the dynamic UI in action.

    *   **Page ID for Read/Write:**
        ```typescript
        {
          id: 'pageId',
          title: 'Page ID',
          type: 'short-input',
          layout: 'full',
          placeholder: 'Enter Notion page ID',
          condition: { field: 'operation', value: 'notion_read' }, // Only show for "Read Page"
          required: true,
        },
        // ... (similar block for notion_write operation) ...
        ```
        These blocks prompt for a Notion Page ID, but critically, only when the `operation` field is set to `notion_read` or `notion_write`. Note that `pageId` is used for both operations but with different conditions, ensuring the field appears contextually.

    *   **Database ID for Read/Query:**
        ```typescript
        {
          id: 'databaseId',
          title: 'Database ID',
          type: 'short-input',
          layout: 'full',
          placeholder: 'Enter Notion database ID',
          condition: { field: 'operation', value: 'notion_read_database' }, // Only show for "Read Database"
          required: true,
        },
        // ... (similar block for notion_query_database operation) ...
        ```
        Similar to `pageId`, `databaseId` appears for "Read Database" and "Query Database" operations.

    *   **Create Page Fields (`parentId`, `title`, `content`):**
        ```typescript
        {
          id: 'parentId',
          title: 'Parent Page ID',
          type: 'short-input',
          layout: 'full',
          placeholder: 'ID of parent page',
          condition: { field: 'operation', value: 'notion_create_page' },
          required: true,
        },
        // ... title and content fields also appear for notion_create_page ...
        ```
        These fields are shown when the user wants to `notion_create_page`, asking for where the new page should live (`parentId`), its `title`, and initial `content`.

    *   **Content Input for Write/Create:**
        ```typescript
        {
          id: 'content',
          title: 'Content',
          type: 'long-input', // A multi-line text input
          layout: 'full',
          placeholder: 'Enter content to add to the page',
          condition: { field: 'operation', value: 'notion_write' },
          required: true,
        },
        // ... (similar block for notion_create_page) ...
        ```
        A `long-input` field for the actual content, appearing for "Append Content" (`notion_write`) and "Create Page" (`notion_create_page`).

    *   **Query Database Fields (`filter`, `sorts`, `pageSize`):**
        ```typescript
        {
          id: 'filter',
          title: 'Filter (JSON)',
          type: 'long-input',
          layout: 'full',
          placeholder: 'Enter filter conditions as JSON (optional)',
          condition: { field: 'operation', value: 'notion_query_database' },
          required: true, // Note: required true, but placeholder says optional, potential UX inconsistency.
        },
        // ... sorts and pageSize fields also for notion_query_database ...
        ```
        These fields allow users to specify advanced query parameters for Notion databases. `filter` and `sorts` are expected as JSON strings.

    *   **Search Fields (`query`, `filterType`):**
        ```typescript
        {
          id: 'query',
          title: 'Search Query',
          type: 'short-input',
          layout: 'full',
          placeholder: 'Enter search terms (leave empty for all pages)',
          condition: { field: 'operation', value: 'notion_search' },
        },
        // ... filterType dropdown also for notion_search ...
        ```
        These fields appear for the "Search Workspace" operation, allowing users to enter a search term and optionally filter by type (pages or databases).

    *   **Create Database Fields (`parentId`, `title`, `properties`):**
        ```typescript
        {
          id: 'parentId',
          title: 'Parent Page ID',
          type: 'short-input',
          layout: 'full',
          placeholder: 'ID of parent page where database will be created',
          condition: { field: 'operation', value: 'notion_create_database' },
          required: true,
        },
        // ... title and properties fields also for notion_create_database ...
        ```
        These fields gather the necessary information to create a new Notion database, including its location (`parentId`), `title`, and `properties` (expected as a JSON object string).

---

#### `tools`: Backend Integration Configuration

```typescript
  tools: {
    access: [
      'notion_read',
      // ... all supported tool IDs ...
    ],
    config: {
      tool: (params) => { /* ... */ },
      params: (params) => { /* ... */ },
    },
  },
```

This section describes how the block integrates with the backend "tools" (API wrappers or functions) that perform the actual Notion operations.

*   `access: [...]`: This array lists all the internal tool IDs that this `NotionBlock` is allowed to invoke. These IDs correspond to the `id` values defined in the `operation` dropdown.

*   `config: { ... }`: Contains functions that determine which tool to call and how to format the parameters for that tool.

    *   `tool: (params) => { ... }`:
        ```typescript
        tool: (params) => {
          switch (params.operation) {
            case 'notion_read':
              return 'notion_read'
            // ... other cases ...
            default:
              return 'notion_read'
          }
        },
        ```
        This function takes all the user-provided parameters (`params`) and, based on the `operation` chosen by the user, returns the specific tool ID (e.g., `'notion_read'`) that the backend should execute. It acts as a router for operations. A `default` case ensures an operation is always returned, falling back to `'notion_read'`.

    *   `params: (params) => { ... }`: **(Most Complex Part)**
        ```typescript
        params: (params) => {
          const { credential, operation, properties, filter, sorts, ...rest } = params

          let parsedProperties
          if (
            (operation === 'notion_create_page' || operation === 'notion_create_database') &&
            properties
          ) {
            try {
              parsedProperties = JSON.parse(properties)
            } catch (error) {
              throw new Error(
                `Invalid JSON for properties: ${error instanceof Error ? error.message : String(error)}`
              )
            }
          }

          let parsedFilter
          if (operation === 'notion_query_database' && filter) {
            try {
              parsedFilter = JSON.parse(filter)
            } catch (error) {
              throw new Error(
                `Invalid JSON for filter: ${error instanceof Error ? error.message : String(error)}`
              )
            }
          }

          let parsedSorts
          if (operation === 'notion_query_database' && sorts) {
            try {
              parsedSorts = JSON.parse(sorts)
            } catch (error) {
              throw new Error(
                `Invalid JSON for sorts: ${error instanceof Error ? error.message : String(error)}`
              )
            }
          }

          return {
            ...rest,
            credential,
            ...(parsedProperties ? { properties: parsedProperties } : {}),
            ...(parsedFilter ? { filter: JSON.stringify(parsedFilter) } : {}),
            ...(parsedSorts ? { sorts: JSON.stringify(parsedSorts) } : {}),
          }
        },
        ```
        This function is responsible for transforming the raw input `params` (from the UI) into the exact format expected by the underlying Notion tool function.

        1.  `const { credential, operation, properties, filter, sorts, ...rest } = params`: It uses object destructuring to extract specific fields (`credential`, `operation`, `properties`, `filter`, `sorts`) that need special handling, while `...rest` collects all other parameters into a single object.
        2.  **JSON Parsing and Error Handling:**
            *   For `properties` (used in `notion_create_page` and `notion_create_database`), `filter` (for `notion_query_database`), and `sorts` (for `notion_query_database`), the UI typically provides these as *JSON strings*. The backend tool, however, might expect them as *JavaScript objects* or validated JSON strings.
            *   Each block of `if` statements checks if the current `operation` requires one of these fields and if the field has a value.
            *   `try { parsed... = JSON.parse(...) } catch (error) { throw new Error(...) }`: It attempts to parse the JSON string into a JavaScript object. If parsing fails (meaning the user entered invalid JSON), it catches the error and throws a new, more descriptive error message.
        3.  `return { ... }`: Finally, it constructs the object of parameters to be sent to the backend tool:
            *   `...rest`: Includes all the parameters that didn't require special parsing or transformation.
            *   `credential`: Passes the Notion account credential.
            *   `...(parsedProperties ? { properties: parsedProperties } : {})`: This is a conditional spread. If `parsedProperties` exists (i.e., it was successfully parsed), it adds a `properties` field with the parsed object. Otherwise, it adds nothing.
            *   `...(parsedFilter ? { filter: JSON.stringify(parsedFilter) } : {})`: For `filter`, it parses it, and then *stringifies it back*. This pattern suggests that while the parsing happens here for validation or potential intermediate manipulation, the ultimate backend Notion tool function expects the `filter` parameter to be a *JSON string* (not a raw JavaScript object).
            *   `...(parsedSorts ? { sorts: JSON.stringify(parsedSorts) } : {})`: The same logic applies to `sorts` as to `filter`.

---

#### `inputs`: Block's Input Schema

```typescript
  inputs: {
    operation: { type: 'string', description: 'Operation to perform' },
    credential: { type: 'string', description: 'Notion access token' },
    pageId: { type: 'string', description: 'Page identifier' },
    content: { type: 'string', description: 'Page content' },
    // ... other inputs ...
  },
```

This object defines the formal input contract for the Notion block. It lists all possible parameters the block *can* receive (from user input or other connected blocks in a workflow), along with their expected `type` and a `description`. This is useful for documentation, validation, and programmatic interaction with the block.

---

#### `outputs`: Block's Output Schema

```typescript
  outputs: {
    content: {
      type: 'string',
      description: 'Page content, search results, or confirmation messages',
    },
    metadata: {
      type: 'json',
      description:
        'Metadata containing operation-specific details including page/database info, results, and pagination data',
    },
  },
```

This object defines the formal output contract for the Notion block. It specifies what data the block will produce after an operation, including its `type` and `description`.

*   `content`: A general output field, often a string, for primary data like page content or search results.
*   `metadata`: A more structured `json` output, providing richer details about the operation's outcome, such as page/database properties, query results, and pagination information.

---

### In Summary

This `NotionBlock` configuration file is a comprehensive definition for integrating Notion into an automated platform. It meticulously describes:

*   **Metadata:** Basic information about the Notion integration (name, description, icon).
*   **User Interface:** How users interact with the block through dynamic input fields that appear based on the selected operation (`subBlocks` with `condition`).
*   **Authentication:** Specifies OAuth as the method and lists required permissions.
*   **Backend Integration:** How user inputs are mapped to specific backend Notion API calls and transformed into the correct data formats (`tools` section, especially the `params` function with its JSON parsing/stringifying logic).
*   **API Contract:** The expected inputs and outputs for the block, enabling other parts of the system to interact with it predictably (`inputs` and `outputs`).

It serves as a single source of truth for the Notion integration's capabilities, UI, and backend interaction.