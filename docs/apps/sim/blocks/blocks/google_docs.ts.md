This TypeScript code defines a configuration for a "Google Docs Block" within a larger platform, likely a workflow automation or AI tool. Think of it as a blueprint for a user interface component that allows users to interact with Google Docs (read, write, or create documents) from within the platform.

The file essentially describes:
1.  **What the block is:** Its name, description, icon, and basic properties.
2.  **How it looks and behaves in the UI:** A list of interactive fields (dropdowns, text inputs, file selectors) that the user will see and fill out.
3.  **How it authenticates:** Specifies that Google OAuth is required.
4.  **How it translates user input into actions:** It maps the user's selections and inputs to specific Google Docs API calls (tools) and formats the data appropriately for those calls.
5.  **What data it expects as input and what it produces as output:** For integration with other parts of the workflow.

---

### Simplified Complex Logic:

The most complex parts are:

*   **`subBlocks` Array:** This array defines all the interactive elements the user sees. Many of these elements are shown or hidden conditionally based on what the user selects in the "Operation" dropdown (`read`, `write`, or `create`). For example, if you select "Create Document", fields for "Document ID" will disappear, and fields for "Document Title" and "Parent Folder" will appear. Also, for document/folder selection, there's often a "basic" (file selector UI) and "advanced" (manual ID input) mode, both mapping to the same underlying data parameter (`canonicalParamId`).
*   **`tools.config` Object:** This section dictates how the block actually *executes* its function.
    *   The `tool` function dynamically decides *which* Google Docs operation to perform (`read`, `write`, or `create`) based on the user's selection.
    *   The `params` function acts as a data adapter. It takes all the raw inputs from the UI fields, cleans them up, combines values from "basic" and "advanced" mode fields (e.g., `documentId` and `manualDocumentId`), and formats them into a single, clean object that the chosen Google Docs tool expects.

---

### Line-by-Line Explanation:

```typescript
import { GoogleDocsIcon } from '@/components/icons'
```
*   **Purpose:** Imports a React component named `GoogleDocsIcon`.
*   **Explanation:** This line brings in an icon that will be used to visually represent the Google Docs block in the user interface. The `@` symbol usually indicates an alias for a specific directory in the project (e.g., `src`).

```typescript
import type { BlockConfig } from '@/blocks/types'
import { AuthMode } from '@/blocks/types'
import type { GoogleDocsResponse } from '@/tools/google_docs/types'
```
*   **Purpose:** Imports TypeScript type definitions and an enum.
*   **Explanation:**
    *   `import type { BlockConfig } from '@/blocks/types'`: Imports the `BlockConfig` type. This type defines the expected structure for any block configuration object in the system. The `type` keyword ensures this import is only for type-checking and generates no JavaScript code.
    *   `import { AuthMode } from '@/blocks/types'`: Imports the `AuthMode` enum. This enum likely defines different authentication methods (e.g., API Key, OAuth).
    *   `import type { GoogleDocsResponse } from '@/tools/google_docs/types'`: Imports the `GoogleDocsResponse` type. This type specifies the structure of the data expected back from the Google Docs tools after an operation.

```typescript
export const GoogleDocsBlock: BlockConfig<GoogleDocsResponse> = {
```
*   **Purpose:** Declares and exports the main configuration object for the Google Docs block.
*   **Explanation:** This line defines a constant variable named `GoogleDocsBlock` and exports it so it can be used by other parts of the application. It explicitly types `GoogleDocsBlock` as `BlockConfig<GoogleDocsResponse>`, meaning it must conform to the `BlockConfig` structure and its `outputs` property will ultimately produce data of type `GoogleDocsResponse`.

```typescript
  type: 'google_docs',
```
*   **Purpose:** Specifies the internal identifier for this block type.
*   **Explanation:** This is a unique string identifier for this particular block. The system uses this `type` to recognize and register the block.

```typescript
  name: 'Google Docs',
```
*   **Purpose:** Sets the user-friendly name of the block.
*   **Explanation:** This is the display name that users will see in the UI (e.g., in a block selector or sidebar).

```typescript
  description: 'Read, write, and create documents',
```
*   **Purpose:** Provides a brief description of the block's capabilities.
*   **Explanation:** A short summary of what the block does, often shown as a tooltip or subtitle.

```typescript
  authMode: AuthMode.OAuth,
```
*   **Purpose:** Defines the authentication method required.
*   **Explanation:** Specifies that this block uses `OAuth` (Open Authorization) for authentication, which is typical for Google services. `AuthMode.OAuth` comes from the imported enum.

```typescript
  longDescription:
    'Integrate Google Docs into the workflow. Can read, write, and create documents.',
```
*   **Purpose:** Offers a more detailed description of the block.
*   **Explanation:** A more extensive explanation of the block's functionality, potentially shown in a detailed view or help section.

```typescript
  docsLink: 'https://docs.sim.ai/tools/google_docs',
```
*   **Purpose:** Provides a link to external documentation.
*   **Explanation:** If users need more information, this URL will direct them to the relevant documentation page for the Google Docs integration.

```typescript
  category: 'tools',
```
*   **Purpose:** Categorizes the block for organization in the UI.
*   **Explanation:** This helps group similar blocks together (e.g., "AI", "Integrations", "Tools") in a user interface.

```typescript
  bgColor: '#E0E0E0',
```
*   **Purpose:** Sets a background color for the block's visual representation.
*   **Explanation:** This is a hexadecimal color code that might be used for the block's card, icon background, or other UI elements to visually distinguish it.

```typescript
  icon: GoogleDocsIcon,
```
*   **Purpose:** Assigns the visual icon component for the block.
*   **Explanation:** This uses the `GoogleDocsIcon` component imported earlier, providing a visual representation for the block in the UI.

```typescript
  subBlocks: [
```
*   **Purpose:** Defines the interactive input fields and UI components for the block.
*   **Explanation:** This is an array of objects, where each object describes a specific input field, dropdown, or other UI element that the user will interact with to configure this Google Docs block.

    *   **Operation selector**
    ```typescript
    {
      id: 'operation',
      title: 'Operation',
      type: 'dropdown',
      layout: 'full',
      options: [
        { label: 'Read Document', id: 'read' },
        { label: 'Write to Document', id: 'write' },
        { label: 'Create Document', id: 'create' },
      ],
      value: () => 'read',
    },
    ```
    *   **Explanation:** This defines a dropdown menu where the user selects the desired Google Docs action.
        *   `id: 'operation'`: Unique identifier for this input field.
        *   `title: 'Operation'`: The label displayed to the user.
        *   `type: 'dropdown'`: Specifies it's a dropdown UI component.
        *   `layout: 'full'`: Indicates it should take up the full width available.
        *   `options`: An array of objects defining the choices in the dropdown. Each option has a `label` (what the user sees) and an `id` (the value used internally).
        *   `value: () => 'read'`: Sets the default selected value to 'read' when the block is first added.

    *   **Google Docs Credentials**
    ```typescript
    {
      id: 'credential',
      title: 'Google Account',
      type: 'oauth-input',
      layout: 'full',
      required: true,
      provider: 'google-docs',
      serviceId: 'google-docs',
      requiredScopes: ['https://www.googleapis.com/auth/drive.file'],
      placeholder: 'Select Google account',
    },
    ```
    *   **Explanation:** This configures an input field specifically for selecting or setting up Google OAuth credentials.
        *   `id: 'credential'`: Identifier for the credential field.
        *   `title: 'Google Account'`: Label for the field.
        *   `type: 'oauth-input'`: Custom type for OAuth integration.
        *   `required: true`: This field must be filled out by the user.
        *   `provider: 'google-docs'`, `serviceId: 'google-docs'`: Identifiers for the specific OAuth provider/service.
        *   `requiredScopes: ['https://www.googleapis.com/auth/drive.file']`: Specifies the necessary Google API permissions (OAuth scopes) the user must grant. `drive.file` grants access to files created or opened by the application, and files explicitly shared with it.
        *   `placeholder: 'Select Google account'`: Hint text shown in the input field.

    *   **Document selector (basic mode)**
    ```typescript
    {
      id: 'documentId',
      title: 'Select Document',
      type: 'file-selector',
      layout: 'full',
      canonicalParamId: 'documentId',
      provider: 'google-drive',
      serviceId: 'google-drive',
      requiredScopes: [], // Scopes handled by parent credential
      mimeType: 'application/vnd.google-apps.document',
      placeholder: 'Select a document',
      dependsOn: ['credential'],
      mode: 'basic',
      condition: { field: 'operation', value: ['read', 'write'] },
    },
    ```
    *   **Explanation:** This defines a UI component that allows users to browse and select a Google Docs document. This is the "basic" user-friendly mode.
        *   `id: 'documentId'`: Identifier for this specific UI component.
        *   `canonicalParamId: 'documentId'`: **Crucial.** This indicates that the value from *this field* (and potentially `manualDocumentId`) should ultimately be mapped to a single parameter called `documentId` for the underlying tool.
        *   `type: 'file-selector'`: A custom UI component that opens a file browsing dialog.
        *   `provider: 'google-drive'`, `serviceId: 'google-drive'`: Specifies it's for Google Drive, which hosts Google Docs files.
        *   `mimeType: 'application/vnd.google-apps.document'`: Filters the file selector to only show Google Docs documents.
        *   `dependsOn: ['credential']`: This field only becomes active/visible once a `credential` has been selected.
        *   `mode: 'basic'`: Marks this as part of a basic UI mode (implying an advanced mode might also exist).
        *   `condition: { field: 'operation', value: ['read', 'write'] }`: This field is only displayed if the `operation` dropdown is set to 'read' or 'write'.

    *   **Manual document ID input (advanced mode)**
    ```typescript
    {
      id: 'manualDocumentId',
      title: 'Document ID',
      type: 'short-input',
      layout: 'full',
      canonicalParamId: 'documentId',
      placeholder: 'Enter document ID',
      dependsOn: ['credential'],
      mode: 'advanced',
      condition: { field: 'operation', value: ['read', 'write'] },
    },
    ```
    *   **Explanation:** This provides a simple text input for users to manually type in a Google Docs ID. This is the "advanced" mode alternative to the file selector.
        *   `id: 'manualDocumentId'`: Identifier for this input.
        *   `canonicalParamId: 'documentId'`: Again, this input also maps to the same `documentId` parameter for the tool, allowing users to choose between the file selector or manual input.
        *   `type: 'short-input'`: A single-line text input field.
        *   `mode: 'advanced'`: Marks this as part of an advanced UI mode.
        *   `condition: { field: 'operation', value: ['read', 'write'] }`: Like the file selector, it's only shown for 'read' or 'write' operations.

    *   **Create-specific Fields**
    ```typescript
    {
      id: 'title',
      title: 'Document Title',
      type: 'short-input',
      layout: 'full',
      placeholder: 'Enter title for the new document',
      condition: { field: 'operation', value: 'create' },
      required: true,
    },
    ```
    *   **Explanation:** This field is for entering the title of a *new* document.
        *   `condition: { field: 'operation', value: 'create' }`: Only visible when the operation is 'create'.
        *   `required: true`: A title is mandatory for creating a document.

    *   **Folder selector (basic mode)**
    ```typescript
    {
      id: 'folderSelector',
      title: 'Select Parent Folder',
      type: 'file-selector',
      layout: 'full',
      canonicalParamId: 'folderId',
      provider: 'google-drive',
      serviceId: 'google-drive',
      requiredScopes: [],
      mimeType: 'application/vnd.google-apps.folder',
      placeholder: 'Select a parent folder',
      dependsOn: ['credential'],
      mode: 'basic',
      condition: { field: 'operation', value: 'create' },
    },
    ```
    *   **Explanation:** Similar to the document selector, but for selecting a parent folder.
        *   `canonicalParamId: 'folderId'`: Maps to the `folderId` parameter.
        *   `mimeType: 'application/vnd.google-apps.folder'`: Filters for folders only.
        *   `condition: { field: 'operation', value: 'create' }`: Only visible when creating a document.

    *   **Manual folder ID input (advanced mode)**
    ```typescript
    {
      id: 'folderId',
      title: 'Parent Folder ID',
      type: 'short-input',
      layout: 'full',
      canonicalParamId: 'folderId',
      placeholder: 'Enter parent folder ID (leave empty for root folder)',
      dependsOn: ['credential'],
      mode: 'advanced',
      condition: { field: 'operation', value: 'create' },
    },
    ```
    *   **Explanation:** Provides a manual input for a folder ID, for advanced users.
        *   `canonicalParamId: 'folderId'`: Also maps to the `folderId` parameter.
        *   `condition: { field: 'operation', value: 'create' }`: Only visible when creating a document.

    *   **Content Field for write operation**
    ```typescript
    {
      id: 'content',
      title: 'Content',
      type: 'long-input',
      layout: 'full',
      placeholder: 'Enter document content',
      condition: { field: 'operation', value: 'write' },
      required: true,
    },
    ```
    *   **Explanation:** A multi-line text input field for the content when *writing* to an existing document.
        *   `type: 'long-input'`: A multi-line text area.
        *   `condition: { field: 'operation', value: 'write' }`: Only shown for the 'write' operation.
        *   `required: true`: Content is mandatory for writing.

    *   **Content Field for create operation**
    ```typescript
    {
      id: 'content',
      title: 'Content',
      type: 'long-input',
      layout: 'full',
      placeholder: 'Enter document content',
      condition: { field: 'operation', value: 'create' },
    },
    ```
    *   **Explanation:** Another multi-line text input for content, this time when *creating* a document. It's similar to the write operation's content field but appears separately in the configuration.
        *   `condition: { field: 'operation', value: 'create' }`: Only shown for the 'create' operation. Note that `required` is `false` here, meaning the user can create an empty document.

```typescript
  ], // End of subBlocks array
```

```typescript
  tools: {
```
*   **Purpose:** Defines how the block interacts with the underlying backend tools/APIs.
*   **Explanation:** This object configures the "backend" logic of the block, mapping user inputs to specific tool calls.

```typescript
    access: ['google_docs_read', 'google_docs_write', 'google_docs_create'],
```
*   **Purpose:** Lists all the specific tool identifiers that this block *might* use.
*   **Explanation:** This array tells the system which underlying Google Docs tools (`google_docs_read`, `google_docs_write`, `google_docs_create`) this block has the potential to access and execute. This can be used for permissions or resource management.

```typescript
    config: {
```
*   **Purpose:** Contains functions to dynamically determine the tool and its parameters.
*   **Explanation:** This sub-object holds the core logic for selecting and configuring the actual tool call.

```typescript
      tool: (params) => {
        switch (params.operation) {
          case 'read':
            return 'google_docs_read'
          case 'write':
            return 'google_docs_write'
          case 'create':
            return 'google_docs_create'
          default:
            throw new Error(`Invalid Google Docs operation: ${params.operation}`)
        }
      },
```
*   **Purpose:** A function that selects the correct Google Docs tool based on the user's chosen operation.
*   **Explanation:**
    *   This function takes all the raw parameters (`params`) that the user has provided through the `subBlocks`.
    *   It uses a `switch` statement to check the value of `params.operation` (which comes from the 'Operation' dropdown).
    *   If `operation` is 'read', it returns the string `'google_docs_read'`, indicating the tool for reading documents should be used.
    *   Similarly for 'write' and 'create'.
    *   If an unexpected `operation` value is encountered, it throws an error.

```typescript
      params: (params) => {
        const { credential, documentId, manualDocumentId, folderSelector, folderId, ...rest } =
          params
```
*   **Purpose:** A function that transforms the raw input parameters from the UI into the format expected by the chosen Google Docs tool.
*   **Explanation:**
    *   This function also receives all user-provided `params`.
    *   `const { credential, documentId, manualDocumentId, folderSelector, folderId, ...rest } = params`: This line uses object destructuring to extract specific parameters (`credential`, `documentId`, `manualDocumentId`, `folderSelector`, `folderId`) and collects all *other* parameters into a new object called `rest`. This is a common pattern to process specific fields and then forward the rest.

```typescript
        const effectiveDocumentId = (documentId || manualDocumentId || '').trim()
        const effectiveFolderId = (folderSelector || folderId || '').trim()
```
*   **Purpose:** Resolves which `documentId` and `folderId` to use, consolidating input from basic and advanced UI modes.
*   **Explanation:**
    *   `effectiveDocumentId`: This line determines the final `documentId` to be sent to the tool. It uses the logical OR (`||`) operator. It checks `documentId` (from the file selector) first. If that's empty, it checks `manualDocumentId` (from the text input). If both are empty, it defaults to an empty string. `.trim()` removes any leading/trailing whitespace. This handles the `canonicalParamId` logic by choosing the first available ID from either the 'basic' or 'advanced' mode input.
    *   `effectiveFolderId`: Performs the same consolidation logic for `folderId`, checking `folderSelector` (file selector) then `folderId` (manual input).

```typescript
        return {
          ...rest,
          documentId: effectiveDocumentId || undefined,
          folderId: effectiveFolderId || undefined,
          credential,
        }
      },
    }, // End of config
  }, // End of tools
```
*   **Purpose:** Returns the final, cleaned, and formatted parameters for the selected tool.
*   **Explanation:**
    *   `return { ... }`: This returns a new object containing the parameters for the tool.
    *   `...rest`: Includes all the parameters that were not explicitly handled (like `operation`, `title`, `content`).
    *   `documentId: effectiveDocumentId || undefined`: Sets the `documentId` property. If `effectiveDocumentId` is an empty string (meaning neither basic nor advanced mode provided an ID), it sets `documentId` to `undefined`. This is often preferred for optional parameters in API calls rather than sending empty strings.
    *   `folderId: effectiveFolderId || undefined`: Does the same for `folderId`.
    *   `credential`: Includes the `credential` as it's directly needed by the tool.

```typescript
  inputs: {
```
*   **Purpose:** Defines the expected input parameters if this block were to receive data from another part of the workflow (not just from user UI).
*   **Explanation:** This object describes the external inputs that *this block* expects to receive. These are the "public" parameters.

```typescript
    operation: { type: 'string', description: 'Operation to perform' },
    credential: { type: 'string', description: 'Google Docs access token' },
    documentId: { type: 'string', description: 'Document identifier' },
    manualDocumentId: { type: 'string', description: 'Manual document identifier' },
    title: { type: 'string', description: 'Document title' },
    folderSelector: { type: 'string', description: 'Selected folder' },
    folderId: { type: 'string', description: 'Folder identifier' },
    content: { type: 'string', description: 'Document content' },
  },
```
*   **Explanation:** Each line specifies an input parameter by its name, data `type` (e.g., 'string', 'number', 'json'), and a `description`. Note that it lists both `documentId` and `manualDocumentId`, and `folderSelector` and `folderId` as potential external inputs, even though the `params` function consolidates them internally. This `inputs` definition is for the *interface* of the block, not its internal processing.

```typescript
  outputs: {
```
*   **Purpose:** Defines the data that this block will produce as output after its execution.
*   **Explanation:** This object describes the external outputs that *this block* will generate and pass on to subsequent blocks in a workflow.

```typescript
    content: { type: 'string', description: 'Document content' },
    metadata: { type: 'json', description: 'Document metadata' },
    updatedContent: { type: 'boolean', description: 'Content update status' },
  },
} // End of GoogleDocsBlock object
```
*   **Explanation:** Each line specifies an output parameter with its name, data `type`, and `description`.
    *   `content`: The text content of a read document, or the content that was written/created.
    *   `metadata`: Any additional structured data about the document (e.g., author, creation date).
    *   `updatedContent`: A boolean indicating whether the content was successfully updated (relevant for 'write' operation).

---

In summary, this `GoogleDocsBlock` file is a comprehensive definition for a modular component that allows users to seamlessly integrate Google Docs functionality into a larger application, handling everything from UI presentation and user interaction to authentication and dynamic backend tool invocation.