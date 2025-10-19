This TypeScript file defines the configuration for a "Wealthbox" integration block within a larger application, likely a workflow automation or no-code/low-code platform. It acts as a blueprint, telling the platform how to:

1.  **Represent the Wealthbox integration in the UI:** Its name, description, icon, and categories.
2.  **Handle user interaction:** What input fields to show, how they behave, and when they are visible (e.g., "Note ID" only appears when "Read Note" is selected).
3.  **Manage authentication:** How users connect their Wealthbox accounts.
4.  **Execute operations:** How to translate user choices and inputs into actual calls to the underlying Wealthbox API (via internal "tools").
5.  **Define inputs and outputs:** What data this block expects and what data it can produce.

In essence, this file configures a UI component and its underlying logic to allow users to interact with Wealthbox (a CRM for financial advisors) by performing operations like reading/writing notes, contacts, or tasks.

---

### Detailed Explanation

Let's break down the code line by line and section by section.

#### Imports

```typescript
import { WealthboxIcon } from '@/components/icons'
import type { BlockConfig } from '@/blocks/types'
import { AuthMode } from '@/blocks/types'
import type { WealthboxResponse } from '@/tools/wealthbox/types'
```

These lines import necessary types and components from other parts of the application:

*   **`import { WealthboxIcon } from '@/components/icons'`**: Imports a specific React component or SVG data named `WealthboxIcon`. This will likely be used to display an icon for the Wealthbox block in the user interface, making it visually identifiable. The `@/` prefix usually indicates an alias for a base directory in the project (e.g., `src/`).
*   **`import type { BlockConfig } from '@/blocks/types'`**: Imports a TypeScript `type` called `BlockConfig`. This is a generic type that defines the expected structure for any "block" configuration in the system. It ensures that `WealthboxBlock` adheres to a standard interface, providing consistency across different integration blocks.
*   **`import { AuthMode } from '@/blocks/types'`**: Imports an `enum` or object `AuthMode`. This specifies different authentication mechanisms supported by blocks (e.g., OAuth, API Key). Here, it's used to declare that Wealthbox requires OAuth.
*   **`import type { WealthboxResponse } from '@/tools/wealthbox/types'`**: Imports a TypeScript `type` called `WealthboxResponse`. This type defines the expected shape of the data returned directly from the underlying Wealthbox tool calls. While `outputs` property defines user-facing outputs, this type might represent the raw data returned by the API client.

#### The `WealthboxBlock` Configuration Object

```typescript
export const WealthboxBlock: BlockConfig<WealthboxResponse> = {
  // ... configuration properties ...
}
```

This declares and exports a constant named `WealthboxBlock`. It's explicitly typed as `BlockConfig<WealthboxResponse>`, meaning it must conform to the `BlockConfig` structure, and its primary response type (for the overall block execution, potentially) is `WealthboxResponse`.

Let's go through its properties:

*   **`type: 'wealthbox'`**
    *   **Purpose:** A unique identifier for this specific block type.
    *   **Explanation:** When the platform needs to identify or render this block, it uses this `type` string.

*   **`name: 'Wealthbox'`**
    *   **Purpose:** The human-readable name displayed in the UI.
    *   **Explanation:** This is what users will see (e.g., in a list of available integrations).

*   **`description: 'Interact with Wealthbox'`**
    *   **Purpose:** A short summary of what the block does.
    *   **Explanation:** Provides a concise explanation to users.

*   **`authMode: AuthMode.OAuth`**
    *   **Purpose:** Specifies the authentication method required.
    *   **Explanation:** Indicates that this block requires users to authenticate with Wealthbox using the OAuth protocol (e.g., connecting their Wealthbox account through a secure flow).

*   **`longDescription: 'Integrate Wealthbox into the workflow. Can read and write notes, read and write contacts, and read and write tasks.'`**
    *   **Purpose:** A more detailed explanation for the user.
    *   **Explanation:** Gives users more context about the block's capabilities.

*   **`docsLink: 'https://docs.sim.ai/tools/wealthbox'`**
    *   **Purpose:** Provides a link to external documentation.
    *   **Explanation:** Allows users to easily access more in-depth information about using the Wealthbox integration.

*   **`category: 'tools'`**
    *   **Purpose:** Categorizes the block for organization in the UI.
    *   **Explanation:** Helps users find blocks by grouping similar integrations (e.g., "CRM tools", "Communication tools").

*   **`bgColor: '#E0E0E0'`**
    *   **Purpose:** Defines a background color for the block in the UI.
    *   **Explanation:** Used for visual styling, often to differentiate blocks or indicate their associated service.

*   **`icon: WealthboxIcon`**
    *   **Purpose:** Assigns the icon component for visual representation.
    *   **Explanation:** The `WealthboxIcon` imported earlier is used here to display an icon next to the block's name.

#### `subBlocks`: Defining the User Interface

```typescript
  subBlocks: [
    // ... array of input field configurations ...
  ],
```

This is a crucial array that defines the interactive input fields and controls that users will see and interact with when configuring this Wealthbox block in the workflow builder. Each object in this array represents a distinct UI element.

Let's go through some example `subBlocks` to understand their structure:

1.  **Operation Dropdown**
    ```typescript
    {
      id: 'operation',
      title: 'Operation',
      type: 'dropdown',
      layout: 'full',
      options: [
        { label: 'Read Note', id: 'read_note' },
        { label: 'Write Note', id: 'write_note' },
        // ... other options ...
      ],
      value: () => 'read_note',
    },
    ```
    *   **`id: 'operation'`**: A unique programmatic identifier for this input field.
    *   **`title: 'Operation'`**: The label displayed to the user.
    *   **`type: 'dropdown'`**: Specifies that this UI element should be a dropdown menu.
    *   **`layout: 'full'`**: Dictates how wide the component should be in the UI (e.g., take full available width).
    *   **`options: [...]`**: An array of objects, each defining an option in the dropdown.
        *   **`label`**: What the user sees in the dropdown.
        *   **`id`**: The value that will be passed internally when this option is selected.
    *   **`value: () => 'read_note'`**: A function that returns the default selected value for this dropdown. Here, it defaults to 'Read Note'.

2.  **Credential Input (OAuth)**
    ```typescript
    {
      id: 'credential',
      title: 'Wealthbox Account',
      type: 'oauth-input',
      layout: 'full',
      provider: 'wealthbox',
      serviceId: 'wealthbox',
      requiredScopes: ['login', 'data'],
      placeholder: 'Select Wealthbox account',
      required: true,
    },
    ```
    *   **`id: 'credential'`**: Identifier for the account selection input.
    *   **`title: 'Wealthbox Account'`**: Label for the input.
    *   **`type: 'oauth-input'`**: A special input type for selecting or connecting OAuth accounts.
    *   **`provider: 'wealthbox'`**: Identifies the OAuth provider.
    *   **`serviceId: 'wealthbox'`**: Further specifies the service if a provider offers multiple services.
    *   **`requiredScopes: ['login', 'data']`**: The OAuth scopes (permissions) required from the user's Wealthbox account for this block to function.
    *   **`placeholder: 'Select Wealthbox account'`**: Text displayed when no account is selected.
    *   **`required: true`**: Indicates that the user *must* select an account.

3.  **Note ID Input (Conditional)**
    ```typescript
    {
      id: 'noteId',
      title: 'Note ID',
      type: 'short-input',
      layout: 'full',
      placeholder: 'Enter Note ID (optional)',
      condition: { field: 'operation', value: ['read_note'] },
    },
    ```
    *   **`id: 'noteId'`**: Identifier for the note ID input.
    *   **`title: 'Note ID'`**: Label for the input.
    *   **`type: 'short-input'`**: A single-line text input field.
    *   **`placeholder: 'Enter Note ID (optional)'`**: Hint text.
    *   **`condition: { field: 'operation', value: ['read_note'] }`**: This is key for dynamic UI. This field will *only* be visible if the `operation` dropdown (defined earlier) has a value of `'read_note'`.

4.  **Contact ID Inputs (Selector & Manual, Conditional)**
    Notice there are two `subBlocks` for `contactId`:
    *   One with `type: 'file-selector'` and `mode: 'basic'`. This likely presents a UI component that can search for contacts within Wealthbox and allow the user to select one.
    *   Another with `type: 'short-input'` and `mode: 'advanced'`. This allows users to manually type in a contact ID.
    *   Both share `canonicalParamId: 'contactId'`, indicating they contribute to the same logical parameter.
    *   Both are conditioned on `operation` being `read_contact`, `write_task`, or `write_note`.

5.  **Task ID Inputs (Selector & Manual, Conditional)**
    Similar to Contact ID, there are two `subBlocks` for `taskId` (one for basic mode/selector, one for advanced mode/manual input), both conditioned on `operation` being `read_task`.

6.  **Other Inputs (`title`, `dueDate`, `firstName`, `lastName`, `emailAddress`, `content`, `backgroundInformation`)**
    These are all `short-input` or `long-input` fields with specific `title`s, `placeholder`s, and `condition`s. They typically become visible and/or required when the `operation` dropdown is set to a relevant 'write' action (e.g., `firstName` and `lastName` are required for `write_contact`).

#### `tools`: Connecting to the Backend API

```typescript
  tools: {
    // ... tool configuration ...
  },
```

This section defines how the block interacts with the actual backend "tools" or API wrappers.

*   **`access: [...]`**
    ```typescript
    access: [
      'wealthbox_read_note',
      'wealthbox_write_note',
      'wealthbox_read_contact',
      'wealthbox_write_contact',
      'wealthbox_read_task',
      'wealthbox_write_task',
    ],
    ```
    *   **Purpose:** Lists all the specific backend tools (API functions) that this block is allowed to invoke.
    *   **Explanation:** This array acts as a whitelist, declaring which low-level operations within the Wealthbox integration this block might trigger. This is important for security and permission management.

*   **`config: { ... }`**
    This object contains functions that dynamically determine which specific tool to call and how to format the parameters for it, based on the user's selections in the UI.

    *   **`tool: (params) => { ... }`**
        ```typescript
        tool: (params) => {
          switch (params.operation) {
            case 'read_note':
              return 'wealthbox_read_note'
            // ... other cases ...
            default:
              throw new Error(`Unknown operation: ${params.operation}`)
          }
        },
        ```
        *   **Purpose:** Maps the user's chosen `operation` (from the dropdown `subBlock`) to the correct internal `tool` identifier from the `access` list.
        *   **Explanation:** This function receives all the current parameters from the UI (`params`). It uses a `switch` statement on `params.operation` to return the appropriate string identifier for the backend tool to execute. For example, if the user selected 'Read Note', it returns `'wealthbox_read_note'`. If an unknown operation is somehow selected, it throws an error.

    *   **`params: (params) => { ... }`**
        ```typescript
        params: (params) => {
          const { credential, operation, contactId, manualContactId, taskId, manualTaskId, ...rest } =
            params

          // Handle both selector and manual inputs
          const effectiveContactId = (contactId || manualContactId || '').trim()
          const effectiveTaskId = (taskId || manualTaskId || '').trim()

          const baseParams = {
            ...rest,
            credential,
          }

          if (operation === 'read_note' || operation === 'write_note') {
            return {
              ...baseParams,
              noteId: params.noteId,
              contactId: effectiveContactId,
            }
          }
          // ... other conditional returns ...
          return baseParams
        },
        ```
        *   **Purpose:** Transforms the raw inputs from the UI (`params`) into the specific format and structure expected by the chosen backend tool. It also handles validation and merging of related inputs.
        *   **Explanation:** This is the most complex part of the `tools` section.
            *   **`const { credential, operation, contactId, manualContactId, taskId, manualTaskId, ...rest } = params`**: It destructures the incoming `params` object. It specifically extracts `credential`, `operation`, and both selector-based (`contactId`, `taskId`) and manual (`manualContactId`, `manualTaskId`) IDs. `...rest` collects all other parameters that don't need special handling initially.
            *   **`const effectiveContactId = (contactId || manualContactId || '').trim()`**: This clever line determines the actual `contactId` to use. If `contactId` (from the selector) is present, it uses that. Otherwise, if `manualContactId` is present, it uses that. If neither, it defaults to an empty string. `.trim()` removes leading/trailing whitespace. A similar logic applies to `effectiveTaskId`. This handles the `basic` and `advanced` mode inputs for the same logical ID.
            *   **`const baseParams = { ...rest, credential }`**: Creates a base set of parameters, including all the `rest` parameters and the `credential`.
            *   **Conditional Returns**: The code then uses `if` statements to check the `operation`. Based on the operation, it constructs and returns a specific parameter object.
                *   For `read_note` or `write_note`, it includes `noteId` and the `effectiveContactId`.
                *   For `read_contact`, it validates that `effectiveContactId` exists (throwing an error if not) and then returns it.
                *   For `read_task`, it validates `taskId` (which means either `taskId` or `manualTaskId` must have been populated via `effectiveTaskId`) and returns it.
                *   For `write_task` or `write_note`, it validates that `contactId` (from the selector) is present and returns it.
                *   Finally, if none of the specific conditions are met, it returns `baseParams`.
            *   **Error Handling**: Note the `throw new Error(...)` calls, which provide immediate feedback to the user if required fields are missing for a chosen operation.

#### `inputs`: Documenting Expected Inputs

```typescript
  inputs: {
    operation: { type: 'string', description: 'Operation to perform' },
    credential: { type: 'string', description: 'Wealthbox access token' },
    // ... other inputs ...
  },
```

*   **Purpose:** This object formally defines all the *possible* input parameters this block can receive from the workflow (either directly from previous steps or via the UI).
*   **Explanation:** It's a structured schema that declares each input's `type` and `description`. This is used for documentation, schema validation, and potentially for features like auto-completion in a UI. It lists all potential parameters, even if they are conditionally displayed in the `subBlocks` UI.

#### `outputs`: Documenting Possible Outputs

```typescript
  outputs: {
    note: {
      type: 'json',
      description: 'Single note object with ID, content, creator, and linked contacts',
    },
    notes: { type: 'json', description: 'Array of note objects from bulk read operations' },
    // ... other outputs ...
    success: {
      type: 'boolean',
      description: 'Boolean indicating whether the operation completed successfully',
    },
  },
```

*   **Purpose:** This object defines all the *possible* output data that this block can produce once it has executed.
*   **Explanation:** Similar to `inputs`, this is a schema that declares each potential output field, its `type`, and a `description`. For example, after a 'Read Note' operation, the `note` output field might be populated. After a 'Read Contacts' operation, the `contacts` field would hold an array. `metadata` and `success` are common outputs for operational status. This informs users what data they can expect to use in subsequent steps of their workflow.

---

### Simplified Complex Logic Summary

*   **`subBlocks` are your UI forms:** They define all the buttons, dropdowns, and text fields a user sees. The `condition` property is like a smart switch that hides or shows fields based on other selections.
*   **`tools.access` is the permission slip:** It lists exactly which backend functions this block is allowed to use.
*   **`tools.config.tool` picks the right action:** Based on what the user selects in the "Operation" dropdown, this function tells the system which specific Wealthbox action (like `read_note` or `write_contact`) to prepare for.
*   **`tools.config.params` prepares the ingredients:** This function takes all the raw values the user typed or selected in the UI and tidies them up. It combines similar inputs (like a contact ID from a search or a manually typed one) and makes sure all the necessary pieces are there before sending them off to the Wealthbox API. If something is missing, it raises an error.
*   **`inputs` and `outputs` are the contract:** They clearly state what information the block expects to receive to do its job, and what information it promises to deliver once it's done.

This `WealthboxBlock` configuration is a complete, self-contained definition for integrating Wealthbox functionalities into a user-friendly, dynamic workflow system.