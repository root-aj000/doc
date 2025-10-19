This TypeScript file defines `ConfluenceBlock`, a configuration object that describes a reusable component or "block" for interacting with Confluence within a larger application or workflow orchestration platform.

Think of it as a blueprint for a visual component that allows users to either read or update a Confluence page. This blueprint specifies everything from how the component looks and what inputs it takes, to how it authenticates and what backend tools it uses.

---

### **Detailed Explanation**

#### **Purpose of this file**

This file defines a `BlockConfig` object named `ConfluenceBlock`. Its primary purpose is to:

1.  **Define a UI Component:** Specify the layout, inputs, and visual representation of a "Confluence" integration block in a user interface. This includes fields for selecting an operation (read/update), entering a Confluence domain, choosing an account, selecting a page, and providing new content/title for updates.
2.  **Configure Backend Interaction:** Map user inputs from the UI to specific backend "tools" (API calls) for Confluence, ensuring correct parameters are passed and authentication is handled.
3.  **Establish Data Contracts:** Define the expected input and output data structures for this block, allowing it to integrate seamlessly into a broader workflow.

In essence, it's a declarative configuration that bridges the gap between a user-facing interface and underlying API functionality for Confluence.

---

#### **Line-by-Line Explanation**

```typescript
import { ConfluenceIcon } from '@/components/icons'
```
*   **`import { ConfluenceIcon } from '@/components/icons'`**: This line imports a React component named `ConfluenceIcon`. This component will be used to display a visual icon for the Confluence block in the user interface, typically for branding or easy identification. The `@` symbol usually denotes an alias for a specific path in the project, like `src/`.

```typescript
import type { BlockConfig } from '@/blocks/types'
import { AuthMode } from '@/blocks/types'
```
*   **`import type { BlockConfig } from '@/blocks/types'`**: This imports the TypeScript *type* `BlockConfig`. It's used solely for type checking during development, ensuring that `ConfluenceBlock` adheres to the expected structure for all blocks in the system. The `type` keyword indicates it's not importing any runnable code.
*   **`import { AuthMode } from '@/blocks/types'`**: This imports the `AuthMode` *enum* (a set of named constants) from the same types file. This enum defines the different ways a block can authenticate with an external service.

```typescript
import type { ConfluenceResponse } from '@/tools/confluence/types'
```
*   **`import type { ConfluenceResponse } from '@/tools/confluence/types'`**: This imports the TypeScript *type* `ConfluenceResponse`. This type defines the expected data structure of the response when the Confluence block successfully completes an operation. `BlockConfig` is generic and takes this type, meaning this particular Confluence block is configured to produce data matching `ConfluenceResponse`.

```typescript
export const ConfluenceBlock: BlockConfig<ConfluenceResponse> = {
```
*   **`export const ConfluenceBlock: BlockConfig<ConfluenceResponse> = {`**: This line declares and exports a constant variable named `ConfluenceBlock`.
    *   **`export`**: Makes `ConfluenceBlock` available for use in other files in the project.
    *   **`const`**: Declares a constant, meaning its value cannot be reassigned after initialization.
    *   **`ConfluenceBlock`**: The name of our configuration object.
    *   **`: BlockConfig<ConfluenceResponse>`**: This is a TypeScript type annotation. It tells the TypeScript compiler that `ConfluenceBlock` must conform to the `BlockConfig` interface, and that the generic type parameter `T` for `BlockConfig` is `ConfluenceResponse`. This ensures the structure is correct and that the `outputs` property (among others) will align with `ConfluenceResponse`.
    *   **`=`**: Assigns the object literal that follows to `ConfluenceBlock`.

#### **Top-Level Block Configuration**

```typescript
  type: 'confluence',
```
*   **`type: 'confluence'`**: A unique string identifier for this specific block type. This is often used internally by the application to identify and register different types of blocks.

```typescript
  name: 'Confluence',
```
*   **`name: 'Confluence'`**: The human-readable name of the block, displayed in the user interface (e.g., in a toolbox or block selection menu).

```typescript
  description: 'Interact with Confluence',
```
*   **`description: 'Interact with Confluence'`**: A brief, concise description of what the block does, often used for tooltips or short summaries in the UI.

```typescript
  authMode: AuthMode.OAuth,
```
*   **`authMode: AuthMode.OAuth`**: Specifies the authentication method required for this block. Here, `AuthMode.OAuth` indicates that the block uses OAuth (Open Authorization) for secure delegation of access to Confluence, rather than, for example, API keys or basic authentication.

```typescript
  longDescription: 'Integrate Confluence into the workflow. Can read and update a page.',
```
*   **`longDescription: 'Integrate Confluence into the workflow. Can read and update a page.'`**: A more detailed explanation of the block's capabilities, potentially shown in a "learn more" section or detailed view.

```typescript
  docsLink: 'https://docs.sim.ai/tools/confluence',
```
*   **`docsLink: 'https://docs.sim.ai/tools/confluence'`**: A URL pointing to external documentation for this Confluence block, providing users with more information on how to use it.

```typescript
  category: 'tools',
```
*   **`category: 'tools'`**: A categorization string that helps organize blocks in the UI (e.g., grouping "tools" blocks together).

```typescript
  bgColor: '#E0E0E0',
```
*   **`bgColor: '#E0E0E0'`**: A hexadecimal color code specifying the background color for the block's representation in the UI, often used for visual branding or distinction.

```typescript
  icon: ConfluenceIcon,
```
*   **`icon: ConfluenceIcon`**: References the `ConfluenceIcon` React component imported earlier. This component will be rendered as the visual icon for the block in the UI.

#### **`subBlocks` Configuration (User Interface Inputs)**

This array defines the individual input fields or interactive elements that will be displayed to the user within the Confluence block's configuration panel. Each object in the array represents one such input.

```typescript
  subBlocks: [
```
*   **`subBlocks: [`**: This property is an array that contains definitions for all the interactive input fields or configuration options that the user will see and interact with for this Confluence block.

##### **1. Operation Dropdown**

```typescript
    {
      id: 'operation',
      title: 'Operation',
      type: 'dropdown',
      layout: 'full',
      options: [
        { label: 'Read Page', id: 'read' },
        { label: 'Update Page', id: 'update' },
      ],
      value: () => 'read',
    },
```
*   **`id: 'operation'`**: A unique identifier for this specific input field, used to reference its value.
*   **`title: 'Operation'`**: The label displayed next to this input in the UI.
*   **`type: 'dropdown'`**: Specifies that this input should be rendered as a dropdown menu.
*   **`layout: 'full'`**: Dictates that this input should take up the full available width in its layout container.
*   **`options: [...]`**: An array defining the selectable items in the dropdown.
    *   `{ label: 'Read Page', id: 'read' }`: An option with the user-facing text "Read Page" and an internal identifier `'read'`.
    *   `{ label: 'Update Page', id: 'update' }`: Another option for "Update Page" with the internal identifier `'update'`.
*   **`value: () => 'read'`**: A function that returns the default selected value for this dropdown. In this case, it defaults to `'read'`, meaning "Read Page" will be pre-selected when the block is added.

##### **2. Domain Input**

```typescript
    {
      id: 'domain',
      title: 'Domain',
      type: 'short-input',
      layout: 'full',
      placeholder: 'Enter Confluence domain (e.g., simstudio.atlassian.net)',
      required: true,
    },
```
*   **`id: 'domain'`**: Identifier for the Confluence domain input.
*   **`title: 'Domain'`**: Label for the input field.
*   **`type: 'short-input'`**: Specifies a single-line text input field.
*   **`layout: 'full'`**: Full-width layout.
*   **`placeholder: 'Enter Confluence domain (...)'`**: Text displayed inside the input field when it's empty, guiding the user.
*   **`required: true`**: Indicates that this field *must* be filled out by the user.

##### **3. Credential/Account Selector**

```typescript
    {
      id: 'credential',
      title: 'Confluence Account',
      type: 'oauth-input',
      layout: 'full',
      provider: 'confluence',
      serviceId: 'confluence',
      requiredScopes: [
        'read:page:confluence',
        'write:page:confluence',
        'read:me',
        'offline_access',
      ],
      placeholder: 'Select Confluence account',
      required: true,
    },
```
*   **`id: 'credential'`**: Identifier for the authentication credential selector.
*   **`title: 'Confluence Account'`**: Label for the input.
*   **`type: 'oauth-input'`**: A special input type designed for managing OAuth connections. It typically allows users to select an existing linked account or initiate a new OAuth flow.
*   **`layout: 'full'`**: Full-width layout.
*   **`provider: 'confluence'`**: Specifies the external service provider for OAuth (e.g., which OAuth integration to use).
*   **`serviceId: 'confluence'`**: A more specific identifier for the service within the provider, if multiple Confluence instances were supported.
*   **`requiredScopes: [...]`**: An array of OAuth scopes (permissions) that the application requests from Confluence when establishing or verifying the connection.
    *   `'read:page:confluence'`: Permission to read Confluence pages.
    *   `'write:page:confluence'`: Permission to write (update) Confluence pages.
    *   `'read:me'`: Permission to read user profile information (often used for validating the connection).
    *   `'offline_access'`: Permission to access Confluence even when the user is not actively present (e.g., for long-running workflows using refresh tokens).
*   **`placeholder: 'Select Confluence account'`**: Placeholder text.
*   **`required: true`**: This field is mandatory.

##### **4. Page Selector (UI-driven)**

```typescript
    {
      id: 'pageId',
      title: 'Select Page',
      type: 'file-selector',
      layout: 'full',
      canonicalParamId: 'pageId',
      provider: 'confluence',
      serviceId: 'confluence',
      placeholder: 'Select Confluence page',
      dependsOn: ['credential', 'domain'],
      mode: 'basic',
    },
```
*   **`id: 'pageId'`**: Identifier for this page selection input.
*   **`title: 'Select Page'`**: Label.
*   **`type: 'file-selector'`**: A specialized UI component that allows users to browse and select a file (or in this case, a Confluence page) from the connected service.
*   **`layout: 'full'`**: Full-width layout.
*   **`canonicalParamId: 'pageId'`**: This indicates that the value selected from this field should conceptually map to the `pageId` parameter in the backend tool's configuration, even if there are multiple ways to specify a page ID (like `manualPageId` below).
*   **`provider: 'confluence'`**: Specifies the service for the file selector.
*   **`serviceId: 'confluence'`**: Specific service identifier.
*   **`placeholder: 'Select Confluence page'`**: Placeholder text.
*   **`dependsOn: ['credential', 'domain']`**: This is a crucial property for dynamic UI. It means this `file-selector` will only become active or enable its functionality *after* the `credential` (Confluence account) and `domain` fields have valid values. This is logical because you need to know *which* Confluence instance and *which* account to browse pages from.
*   **`mode: 'basic'`**: Indicates that this field is part of the "basic" or simpler configuration view. This suggests there might be an "advanced" mode as well.

##### **5. Manual Page ID Input (Advanced)**

```typescript
    {
      id: 'manualPageId',
      title: 'Page ID',
      type: 'short-input',
      layout: 'full',
      canonicalParamId: 'pageId',
      placeholder: 'Enter Confluence page ID',
      mode: 'advanced',
    },
```
*   **`id: 'manualPageId'`**: Identifier for a manual input field for the page ID.
*   **`title: 'Page ID'`**: Label.
*   **`type: 'short-input'`**: Single-line text input.
*   **`layout: 'full'`**: Full-width.
*   **`canonicalParamId: 'pageId'`**: Like `pageId` above, this also maps to the conceptual `pageId` parameter. This is important for the `tools.config.params` function later, which needs to decide which `pageId` source to use.
*   **`placeholder: 'Enter Confluence page ID'`**: Placeholder text.
*   **`mode: 'advanced'`**: This field is only shown in the "advanced" configuration view, providing an alternative to the graphical `file-selector`.

##### **6. New Title Input (Conditional)**

```typescript
    {
      id: 'title',
      title: 'New Title',
      type: 'short-input',
      layout: 'full',
      placeholder: 'Enter new title for the page',
      condition: { field: 'operation', value: 'update' },
    },
```
*   **`id: 'title'`**: Identifier for the new page title input.
*   **`title: 'New Title'`**: Label.
*   **`type: 'short-input'`**: Single-line text input.
*   **`layout: 'full'`**: Full-width.
*   **`placeholder: 'Enter new title for the page'`**: Placeholder text.
*   **`condition: { field: 'operation', value: 'update' }`**: This is a powerful conditional rendering rule. This input field will *only be displayed* in the UI when the `operation` dropdown (defined earlier) has its value set to `'update'`. This ensures the field is relevant to the selected operation.

##### **7. New Content Input (Conditional)**

```typescript
    {
      id: 'content',
      title: 'New Content',
      type: 'long-input',
      layout: 'full',
      placeholder: 'Enter new content for the page',
      condition: { field: 'operation', value: 'update' },
    },
  ],
```
*   **`id: 'content'`**: Identifier for the new page content input.
*   **`title: 'New Content'`**: Label.
*   **`type: 'long-input'`**: A multi-line text area input field (e.g., for body text).
*   **`layout: 'full'`**: Full-width.
*   **`placeholder: 'Enter new content for the page'`**: Placeholder text.
*   **`condition: { field: 'operation', value: 'update' }`**: Similar to the `title` field, this `content` input will only be displayed when the `operation` dropdown is set to `'update'`. This makes sense as you only provide new content when updating a page.

#### **`tools` Configuration (Backend Interaction Logic)**

This section defines how the block translates user inputs into calls to specific backend "tools" or APIs.

```typescript
  tools: {
```
*   **`tools: {`**: This object defines the integration with backend tools or services.

```typescript
    access: ['confluence_retrieve', 'confluence_update'],
```
*   **`access: ['confluence_retrieve', 'confluence_update']`**: This array lists the names of the backend tools that this block *might* need to access. These names correspond to actual functions or APIs on the backend that perform operations like retrieving or updating Confluence data. This helps the system pre-authorize or manage permissions for the block.

```typescript
    config: {
```
*   **`config: {`**: This nested object contains functions that dynamically configure which tool to use and what parameters to send to it, based on the user's inputs.

##### **`tool` Function**

```typescript
      tool: (params) => {
        switch (params.operation) {
          case 'read':
            return 'confluence_retrieve'
          case 'update':
            return 'confluence_update'
          default:
            return 'confluence_retrieve'
        }
      },
```
*   **`tool: (params) => { ... }`**: This is a function that takes `params` (an object containing all the user's inputs from `subBlocks`) and returns the *name* of the specific backend tool to be executed.
    *   **`switch (params.operation)`**: It checks the value of the `operation` input (from the dropdown).
    *   **`case 'read': return 'confluence_retrieve'`**: If `operation` is `'read'`, it tells the system to use the backend tool named `'confluence_retrieve'` (likely an API call to read a Confluence page).
    *   **`case 'update': return 'confluence_update'`**: If `operation` is `'update'`, it uses the `'confluence_update'` tool (likely an API call to modify a Confluence page).
    *   **`default: return 'confluence_retrieve'`**: As a fallback, if `operation` is anything else (or undefined), it defaults to `'confluence_retrieve'`.

##### **`params` Function**

```typescript
      params: (params) => {
        const { credential, pageId, manualPageId, ...rest } = params
```
*   **`params: (params) => { ... }`**: This is another function that takes the raw `params` (all user inputs) and transforms them into the *final set of parameters* that will be sent to the chosen backend tool.
    *   **`const { credential, pageId, manualPageId, ...rest } = params`**: This uses object destructuring to extract specific input values (`credential`, `pageId`, `manualPageId`) from the `params` object, and collects all *other* parameters into a new object called `rest`. This is a clean way to handle parameters you want to modify or prioritize.

```typescript
        const effectivePageId = (pageId || manualPageId || '').trim()
```
*   **`const effectivePageId = (pageId || manualPageId || '').trim()`**: This is a critical piece of logic. It determines the *actual* page ID to be used for the backend operation:
    *   It first tries to use `pageId` (from the `file-selector`).
    *   If `pageId` is empty or `null`/`undefined`, it then tries to use `manualPageId` (from the manual input field).
    *   If both are empty, it defaults to an empty string (`''`).
    *   `.trim()` removes any leading or trailing whitespace from the resulting ID. This handles cases where a user might accidentally enter spaces.

```typescript
        if (!effectivePageId) {
          throw new Error('Page ID is required. Please select a page or enter a page ID manually.')
        }
```
*   **`if (!effectivePageId) { ... }`**: This is an input validation step. If, after the prioritization logic, `effectivePageId` is still an empty string (meaning neither `pageId` nor `manualPageId` was provided), it throws an `Error`. This prevents the backend tool from being called with missing mandatory data and provides a user-friendly error message.

```typescript
        return {
          credential,
          pageId: effectivePageId,
          ...rest,
        }
      },
    },
  },
```
*   **`return { ... }`**: This returns the final object of parameters that will be passed to the backend tool.
    *   **`credential`**: The Confluence account credential (OAuth token or ID).
    *   **`pageId: effectivePageId`**: The determined page ID, prioritizing the selector over manual input.
    *   **`...rest`**: This uses the spread operator to include all other parameters that were not explicitly extracted (`domain`, `operation`, `title`, `content`, etc.) in the final parameters object. This ensures all relevant user inputs are passed along.

#### **`inputs` Configuration (Input Data Schema)**

This object defines the expected data types and descriptions for all possible input parameters that this block *could* receive or output, acting as a schema.

```typescript
  inputs: {
    operation: { type: 'string', description: 'Operation to perform' },
    domain: { type: 'string', description: 'Confluence domain' },
    credential: { type: 'string', description: 'Confluence access token' },
    pageId: { type: 'string', description: 'Page identifier' },
    manualPageId: { type: 'string', description: 'Manual page identifier' },
    title: { type: 'string', description: 'New page title' },
    content: { type: 'string', description: 'New page content' },
  },
```
*   **`inputs: { ... }`**: Defines the schema for inputs this block expects. Each property here corresponds to a user input from `subBlocks` or a value that could be passed *into* this block from another block in a workflow.
    *   **`operation: { type: 'string', description: 'Operation to perform' }`**: Defines the `operation` input as a string with a description.
    *   **`domain: { type: 'string', description: 'Confluence domain' }`**: Defines the `domain` input as a string.
    *   **`credential: { type: 'string', description: 'Confluence access token' }`**: Defines `credential` as a string (representing the token or ID).
    *   **`pageId: { type: 'string', description: 'Page identifier' }`**: Defines `pageId` as a string.
    *   **`manualPageId: { type: 'string', description: 'Manual page identifier' }`**: Defines `manualPageId` as a string.
    *   **`title: { type: 'string', description: 'New page title' }`**: Defines `title` as a string.
    *   **`content: { type: 'string', description: 'New page content' }`**: Defines `content` as a string.

#### **`outputs` Configuration (Output Data Schema)**

This object defines the expected data types and descriptions for the output parameters that this block will produce upon successful execution.

```typescript
  outputs: {
    ts: { type: 'string', description: 'Timestamp' },
    pageId: { type: 'string', description: 'Page identifier' },
    content: { type: 'string', description: 'Page content' },
    title: { type: 'string', description: 'Page title' },
    success: { type: 'boolean', description: 'Operation success status' },
  },
}
```
*   **`outputs: { ... }`**: Defines the schema for data that this block will *output* to subsequent blocks in a workflow, or as the final result. This aligns with the `ConfluenceResponse` type specified in the `BlockConfig` generic.
    *   **`ts: { type: 'string', description: 'Timestamp' }`**: The `ts` (timestamp) output is a string.
    *   **`pageId: { type: 'string', description: 'Page identifier' }`**: The `pageId` of the page that was read or updated is a string.
    *   **`content: { type: 'string', description: 'Page content' }`**: The content of the page (especially for 'read' operations, or the new content for 'update') is a string.
    *   **`title: { type: 'string', description: 'Page title' }`**: The title of the page is a string.
    *   **`success: { type: 'boolean', description: 'Operation success status' }`**: A boolean indicating whether the Confluence operation was successful or not.

---

### **Simplifying Complex Logic**

The most "complex" parts of this file are:

1.  **`subBlocks` with `condition` and `dependsOn`**: These properties allow the UI to be dynamic.
    *   `dependsOn`: An input like `Select Page` (the `file-selector`) will only become active or load data *after* its dependencies (`credential` and `domain`) have valid values. This makes sense: you can't pick a page until you know *where* (domain) and *who* (account) to look with.
    *   `condition`: Inputs like `New Title` and `New Content` only *appear* when the `Operation` dropdown is set to `Update Page`. This prevents clutter and ensures users only see relevant fields for their chosen task.

2.  **`tools.config.tool` and `tools.config.params` functions**: These are the core "translation layers" between the user interface and the backend APIs.
    *   **`tools.config.tool`**: This function is like a switchboard operator. It looks at the `operation` (read or update) chosen by the user and directs the request to the appropriate backend worker: `confluence_retrieve` for reading or `confluence_update` for updating.
    *   **`tools.config.params`**: This function is a parameter packer. It takes all the user's inputs, does some crucial validation (like making sure a page ID is actually provided, either by selecting it or typing it), combines `pageId` and `manualPageId` intelligently, and then organizes all the necessary information into a neat package that the chosen backend worker (`confluence_retrieve` or `confluence_update`) expects. The key here is the `effectivePageId` logic which prioritizes the user-friendly selector but allows for manual override, and then throws an error if neither is provided.

These mechanisms make the Confluence block both user-friendly (dynamic UI, clear choices) and robust (correct tool selection, input validation) without requiring a lot of imperative, repetitive code.