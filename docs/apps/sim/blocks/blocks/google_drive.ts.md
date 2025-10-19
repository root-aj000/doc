This file defines the configuration for a "Google Drive" block within a larger system, likely an automation or workflow builder platform. Think of it as a blueprint that tells the platform how to display, interact with, and execute operations related to Google Drive.

---

### Purpose of This File

The primary purpose of this TypeScript file is to **declare a configuration object (`GoogleDriveBlock`)** that describes a Google Drive integration block. This configuration is used by a user interface (UI) to:

1.  **Render the Google Drive Block**: Define its name, description, icon, and visual appearance.
2.  **Specify Authentication**: Indicate that Google Drive operations require OAuth authentication.
3.  **Define User Inputs (Sub-Blocks)**: Lay out the various input fields (e.g., dropdowns, text inputs, file selectors) a user needs to fill out to perform Google Drive actions like creating a folder, uploading a file, or listing files. Crucially, these inputs are *conditional*, meaning only relevant fields are shown based on the user's selected "operation."
4.  **Map Inputs to Backend Tools**: Specify which backend "tools" (API calls or functions) should be invoked based on the user's chosen operation and how to transform the UI input values into the parameters required by these backend tools.
5.  **Declare Input and Output Schema**: Define the expected data types and descriptions for both the inputs consumed by the block and the outputs it will produce, enabling clear data flow within the platform.

In essence, this file acts as a declarative interface for the Google Drive integration, bridging the gap between user interaction and backend functionality.

---

### Simplified Complex Logic

The most "complex" aspects of this file are not about intricate algorithms, but rather about **declarative configuration patterns** and **conditional rendering/execution**:

1.  **Declarative UI with `subBlocks` and `condition`**:
    *   Instead of writing UI code directly, this file *describes* the UI using an array of `subBlocks`. Each `subBlock` is an input field.
    *   The `condition` property within each `subBlock` is key. It acts like an `if` statement for the UI: "If the 'operation' field is set to 'upload', then show *these* specific fields (fileName, content, mimeType, etc.)." This makes the block dynamic and user-friendly, only presenting relevant options.
    *   **Analogy**: Imagine a multi-page form where each page is an "operation." When you select an operation, the form dynamically shows only the questions relevant to that operation, hiding others. `subBlocks` and `condition` achieve this for a single-page block.

2.  **Mapping UI Inputs to Backend Actions with `tools.config`**:
    *   The platform needs to know *what* to do when a user configures this block. The `tools` object handles this.
    *   The `tools.config.tool` function acts as a **router**: based on the `operation` selected by the user, it returns the name of the specific backend "tool" (e.g., `google_drive_upload`, `google_drive_create_folder`) that should be executed.
    *   The `tools.config.params` function acts as a **transformer**: it takes all the raw input values from the UI (`params`) and reshapes them into the exact format and data types expected by the chosen backend tool. For example, it converts `pageSize` from a string to a number and intelligently picks between a user-selected folder (`folderSelector`) or a manually entered folder ID (`manualFolderId`).
    *   **Analogy**: You tell a chef (the platform) what dish you want (`operation`). The chef then picks the right recipe (`tool` name) and prepares the ingredients (`params` function) exactly as the recipe demands, even if you gave them some ingredients in a slightly different form.

3.  **Canonical Parameters with `canonicalParamId`**:
    *   Notice multiple `subBlocks` for "Upload File" and "Create Folder" operations have `id: 'fileName'`. This allows different UI labels ("File Name" vs "Folder Name") but maps to the *same* underlying data parameter.
    *   Similarly, `file-selector` and `short-input` fields for parent folders often use `canonicalParamId: 'folderId'`. This means whether the user *selects* a folder visually or *types* an ID manually, both ultimately contribute to a single `folderId` parameter that the backend tool expects.
    *   **Analogy**: You have two ways to tell someone your home address: either by pointing it on a map or by typing it in. Regardless of *how* you provide it, the end result is still your home address.

---

### Explanation of Each Line of Code

```typescript
// Imports necessary components and types from other parts of the application.
import { GoogleDriveIcon } from '@/components/icons' // Imports the visual icon for the Google Drive block.
import type { BlockConfig } from '@/blocks/types' // Imports the TypeScript type definition for a block configuration. This ensures our object adheres to the expected structure.
import { AuthMode } from '@/blocks/types' // Imports an enum (enumerated type) that defines different authentication modes, like OAuth.
import type { GoogleDriveResponse } from '@/tools/google_drive/types' // Imports the TypeScript type definition for the expected response structure from Google Drive operations.

// Exports a constant variable named `GoogleDriveBlock`.
// This variable holds the entire configuration object for the Google Drive integration block.
// It's typed as `BlockConfig<GoogleDriveResponse>`, meaning it's a block configuration
// that will ultimately produce a `GoogleDriveResponse` type as its output.
export const GoogleDriveBlock: BlockConfig<GoogleDriveResponse> = {
  // Defines the unique identifier for this type of block. Used internally by the platform.
  type: 'google_drive',
  // The user-friendly name displayed in the UI.
  name: 'Google Drive',
  // A short description shown in the UI, often in a tooltip or block picker.
  description: 'Create, upload, and list files',
  // Specifies the authentication method required for this block. Here, it's OAuth.
  authMode: AuthMode.OAuth,
  // A more detailed description, possibly shown in a block's details panel.
  longDescription: 'Integrate Google Drive into the workflow. Can create, upload, and list files.',
  // A link to external documentation for this specific block.
  docsLink: 'https://docs.sim.ai/tools/google_drive',
  // Categorizes the block, helping users find it in a library (e.g., under "tools" or "integrations").
  category: 'tools',
  // The background color for the block's visual representation in the UI.
  bgColor: '#E0E0E0',
  // The icon component to be displayed with the block.
  icon: GoogleDriveIcon,
  // An array defining the individual input fields and UI elements that make up this block.
  // These are often referred to as "sub-blocks" or "parameters."
  subBlocks: [
    // --- Operation selector ---
    {
      // Unique identifier for this input field within the block.
      id: 'operation',
      // Title displayed next to the input field in the UI.
      title: 'Operation',
      // Type of UI component: a dropdown menu.
      type: 'dropdown',
      // Layout specification: 'full' typically means it takes the full available width.
      layout: 'full',
      // Array of options for the dropdown. Each option has a user-friendly `label` and an internal `id` (value).
      options: [
        { label: 'Create Folder', id: 'create_folder' },
        { label: 'Upload File', id: 'upload' },
        { label: 'List Files', id: 'list' },
      ],
      // A function that returns the default selected value for this dropdown.
      // Here, 'create_folder' is pre-selected when the block is added.
      value: () => 'create_folder',
    },
    // --- Google Drive Credentials ---
    {
      id: 'credential',
      title: 'Google Drive Account',
      // Type of UI component: an input specifically designed for OAuth credentials.
      type: 'oauth-input',
      layout: 'full',
      // Indicates that this field must be filled out.
      required: true,
      // Specifies the OAuth provider this input is for.
      provider: 'google-drive',
      // Specifies the service ID for the OAuth integration.
      serviceId: 'google-drive',
      // An array of Google API scopes required for the credentials to function correctly.
      // 'drive.file' grants per-file access to files created or opened by the app.
      requiredScopes: ['https://www.googleapis.com/auth/drive.file'],
      // Placeholder text shown when the input is empty.
      placeholder: 'Select Google Drive account',
    },
    // --- Upload Fields (only visible when 'operation' is 'upload') ---
    {
      id: 'fileName',
      title: 'File Name',
      // Type of UI component: a short single-line text input.
      type: 'short-input',
      layout: 'full',
      placeholder: 'Name of the file',
      // Conditional rendering: this field is only shown if the 'operation' field's value is 'upload'.
      condition: { field: 'operation', value: 'upload' },
      required: true,
    },
    {
      id: 'content',
      title: 'Content',
      // Type of UI component: a multi-line text area.
      type: 'long-input',
      layout: 'full',
      placeholder: 'Content to upload to the file',
      condition: { field: 'operation', value: 'upload' },
      required: true,
    },
    {
      id: 'mimeType',
      title: 'MIME Type',
      type: 'dropdown',
      layout: 'full',
      options: [
        { label: 'Google Doc', id: 'application/vnd.google-apps.document' },
        { label: 'Google Sheet', id: 'application/vnd.google-apps.spreadsheet' },
        { label: 'Google Slides', id: 'application/vnd.google-apps.presentation' },
        { label: 'PDF (application/pdf)', id: 'application/pdf' },
      ],
      placeholder: 'Select a file type',
      condition: { field: 'operation', value: 'upload' },
    },
    {
      id: 'folderSelector',
      title: 'Select Parent Folder',
      // Type of UI component: a file/folder selector, likely with a browsing interface.
      type: 'file-selector',
      layout: 'full',
      // Specifies the canonical (standardized) parameter ID this field contributes to.
      // This allows different UI fields (e.g., a visual selector and a manual input)
      // to map to the same underlying parameter for the backend tool.
      canonicalParamId: 'folderId',
      provider: 'google-drive',
      serviceId: 'google-drive',
      requiredScopes: ['https://www.googleapis.com/auth/drive.file'],
      // Specifies the MIME type of items this selector should display (here, only folders).
      mimeType: 'application/vnd.google-apps.folder',
      placeholder: 'Select a parent folder',
      // Display mode: 'basic' typically means a user-friendly selection.
      mode: 'basic',
      // Indicates that this field depends on the 'credential' field being filled.
      // It won't be usable or might be disabled until credentials are provided.
      dependsOn: ['credential'],
      condition: { field: 'operation', value: 'upload' },
    },
    {
      id: 'manualFolderId',
      title: 'Parent Folder ID',
      type: 'short-input',
      layout: 'full',
      canonicalParamId: 'folderId', // Also maps to 'folderId'
      placeholder: 'Enter parent folder ID (leave empty for root folder)',
      // Display mode: 'advanced' often means it's an alternative, less user-friendly input.
      mode: 'advanced',
      condition: { field: 'operation', value: 'upload' },
    },
    // Commented-out sections (// ...) indicate features that are either planned,
    // temporarily disabled, or alternate ways of handling certain operations (e.g., 'get_content').
    // They are not active in the current configuration.

    // --- Create Folder Fields (only visible when 'operation' is 'create_folder') ---
    {
      id: 'fileName', // Reuses the 'fileName' ID, but with a different title and condition.
      title: 'Folder Name',
      type: 'short-input',
      layout: 'full',
      placeholder: 'Name for the new folder',
      condition: { field: 'operation', value: 'create_folder' },
      required: true,
    },
    {
      id: 'folderSelector', // Reuses the 'folderSelector' ID.
      title: 'Select Parent Folder',
      type: 'file-selector',
      layout: 'full',
      canonicalParamId: 'folderId',
      provider: 'google-drive',
      serviceId: 'google-drive',
      requiredScopes: ['https://www.googleapis.com/auth/drive.file'],
      mimeType: 'application/vnd.google-apps.folder',
      placeholder: 'Select a parent folder',
      mode: 'basic',
      dependsOn: ['credential'],
      condition: { field: 'operation', value: 'create_folder' },
    },
    {
      id: 'manualFolderId', // Reuses the 'manualFolderId' ID.
      title: 'Parent Folder ID',
      type: 'short-input',
      layout: 'full',
      canonicalParamId: 'folderId',
      placeholder: 'Enter parent folder ID (leave empty for root folder)',
      mode: 'advanced',
      condition: { field: 'operation', value: 'create_folder' },
    },
    // --- List Fields (only visible when 'operation' is 'list') ---
    {
      id: 'folderSelector', // Reuses the 'folderSelector' ID.
      title: 'Select Folder',
      type: 'file-selector',
      layout: 'full',
      canonicalParamId: 'folderId',
      provider: 'google-drive',
      serviceId: 'google-drive',
      requiredScopes: ['https://www.googleapis.com/auth/drive.file'],
      mimeType: 'application/vnd.google-apps.folder',
      placeholder: 'Select a folder to list files from',
      mode: 'basic',
      dependsOn: ['credential'],
      condition: { field: 'operation', value: 'list' },
    },
    {
      id: 'manualFolderId', // Reuses the 'manualFolderId' ID.
      title: 'Folder ID',
      type: 'short-input',
      layout: 'full',
      canonicalParamId: 'folderId',
      placeholder: 'Enter folder ID (leave empty for root folder)',
      mode: 'advanced',
      condition: { field: 'operation', value: 'list' },
    },
    {
      id: 'query',
      title: 'Search Query',
      type: 'short-input',
      layout: 'full',
      placeholder: 'Search for specific files (e.g., name contains "report")',
      condition: { field: 'operation', value: 'list' },
    },
    {
      id: 'pageSize',
      title: 'Results Per Page',
      type: 'short-input',
      layout: 'full',
      placeholder: 'Number of results (default: 100, max: 1000)',
      condition: { field: 'operation', value: 'list' },
    },
  ],
  // Defines how this block interacts with backend "tools" (API calls, functions).
  tools: {
    // An array of backend tool identifiers that this block is allowed to access/invoke.
    access: ['google_drive_upload', 'google_drive_create_folder', 'google_drive_list'],
    // Configuration for how to select and prepare parameters for the backend tools.
    config: {
      // A function that determines which backend tool to call based on the user's input.
      // `params` is an object containing all the raw input values from the subBlocks.
      tool: (params) => {
        // Uses a switch statement to return the appropriate tool ID based on the `operation` parameter.
        switch (params.operation) {
          case 'upload':
            return 'google_drive_upload'
          case 'create_folder':
            return 'google_drive_create_folder'
          case 'list':
            return 'google_drive_list'
          default:
            // Throws an error if an unknown operation is encountered, ensuring robust error handling.
            throw new Error(`Invalid Google Drive operation: ${params.operation}`)
        }
      },
      // A function that transforms the raw user input parameters into the format expected by the backend tool.
      params: (params) => {
        // Destructures relevant parameters, separating those that need special handling from the rest.
        const { credential, folderSelector, manualFolderId, mimeType, ...rest } = params

        // Determines the effective folder ID:
        // Prioritizes `folderSelector` (visual selection), then `manualFolderId` (manual input).
        // If neither is provided, it defaults to an empty string. `.trim()` removes leading/trailing whitespace.
        const effectiveFolderId = (folderSelector || manualFolderId || '').trim()

        // Returns the transformed parameters object.
        return {
          credential, // Pass credential directly.
          // Sets `folderId` to the `effectiveFolderId` if it has a value, otherwise `undefined`.
          // `undefined` is often used to omit a parameter if it's not present or optional.
          folderId: effectiveFolderId || undefined,
          // Parses `pageSize` from string to integer if it exists, otherwise `undefined`.
          // `Number.parseInt(..., 10)` ensures parsing as a base-10 integer.
          pageSize: rest.pageSize ? Number.parseInt(rest.pageSize as string, 10) : undefined,
          // Passes mimeType directly.
          mimeType: mimeType,
          // Spreads any remaining parameters from `rest` (e.g., fileName, content, query).
          ...rest,
        }
      },
    },
  },
  // Defines the expected inputs that this block consumes, along with their types and descriptions.
  // This acts as a schema for other parts of the platform that might connect to this block.
  inputs: {
    operation: { type: 'string', description: 'Operation to perform' },
    credential: { type: 'string', description: 'Google Drive access token' },
    // Upload and Create Folder operation inputs
    fileName: { type: 'string', description: 'File or folder name' },
    content: { type: 'string', description: 'File content' },
    mimeType: { type: 'string', description: 'File MIME type' },
    // List operation inputs
    folderSelector: { type: 'string', description: 'Selected folder' },
    manualFolderId: { type: 'string', description: 'Manual folder identifier' },
    query: { type: 'string', description: 'Search query' },
    pageSize: { type: 'number', description: 'Results per page' },
  },
  // Defines the expected outputs that this block produces.
  // This allows other blocks to understand what data they can receive from a Google Drive operation.
  outputs: {
    file: { type: 'json', description: 'File data' }, // For operations returning a single file (e.g., upload, create_folder).
    files: { type: 'json', description: 'Files list' }, // For operations returning a list of files (e.g., list).
  },
}
```