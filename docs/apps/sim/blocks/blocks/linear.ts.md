This TypeScript file defines a configuration object for a "Linear Block," which is a reusable component or step in a larger workflow automation system. Think of it like a specialized "Lego brick" that users can drag and drop into a workflow to interact with Linear, a popular issue tracking tool.

This configuration tells the system:
1.  **How the block should appear in the user interface:** Its name, description, icon, and background color.
2.  **What input fields it needs from the user:** Dropdowns, text inputs, account selectors, etc., along with their labels, types, and dependencies.
3.  **How to authenticate with Linear:** Using OAuth.
4.  **What operations it can perform:** Reading issues or creating issues.
5.  **How to translate user inputs into actual API calls:** Which backend "tool" (API endpoint) to use and what parameters to send.
6.  **What kind of data it accepts as inputs and produces as outputs** within a workflow.

---

### **Detailed Explanation**

Let's break down the code line by line.

#### **Imports**

```typescript
import { LinearIcon } from '@/components/icons'
import type { BlockConfig } from '@/blocks/types'
import { AuthMode } from '@/blocks/types'
import type { LinearResponse } from '@/tools/linear/types'
```

*   **`import { LinearIcon } from '@/components/icons'`**: This line imports a React component (or similar UI component) named `LinearIcon`. This component will be used to display a visual icon for the Linear block in the user interface. The `@/components/icons` path suggests it's coming from a central icon library within the application.
*   **`import type { BlockConfig } from '@/blocks/types'`**: This imports the `BlockConfig` type. The `type` keyword indicates that this is a type-only import, meaning it won't generate any JavaScript code at runtime, serving purely for static type checking during development. `BlockConfig` is a generic type that defines the overall structure for *any* block in the system. It's generic over the expected output type of the block.
*   **`import { AuthMode } from '@/blocks/types'`**: This imports the `AuthMode` enum. An `enum` (enumeration) is a set of named constants. `AuthMode` likely defines different ways a block can authenticate (e.g., API Key, OAuth).
*   **`import type { LinearResponse } from '@/tools/linear/types'`**: This imports the `LinearResponse` type, again as a type-only import. This type defines the expected structure of the data that the Linear backend tools will return after an operation (like reading or creating an issue).

#### **The `LinearBlock` Configuration Object**

```typescript
export const LinearBlock: BlockConfig<LinearResponse> = {
  // ... configuration details ...
}
```

*   **`export const LinearBlock: BlockConfig<LinearResponse> = { ... }`**: This declares a constant variable named `LinearBlock` and exports it, making it available for other parts of the application to import and use.
    *   `: BlockConfig<LinearResponse>`: This is a type annotation. It specifies that `LinearBlock` must conform to the `BlockConfig` interface, and it's parameterized with `LinearResponse`, indicating that this specific block will produce data of type `LinearResponse` as its output.

Now let's dive into the properties of this `LinearBlock` object:

*   **`type: 'linear'`**: A unique string identifier for this specific block type. This is used internally by the system to distinguish it from other blocks (e.g., a "GitHub Block" or a "Slack Block").
*   **`name: 'Linear'`**: The display name for the block, shown in the user interface (e.g., in a block palette or when selecting a block).
*   **`description: 'Read and create issues in Linear'`**: A short, concise summary of what the block does, displayed in the UI.
*   **`authMode: AuthMode.OAuth`**: Specifies the authentication method for interacting with Linear. `AuthMode.OAuth` means it will use the OAuth standard for secure delegated access.
*   **`longDescription: 'Integrate Linear into the workflow. Can read and create issues.'`**: A more detailed explanation of the block's capabilities, potentially shown in a tooltip or help section in the UI.
*   **`category: 'tools'`**: A categorization string, used to group similar blocks in the UI (e.g., "tools," "communication," "data").
*   **`icon: LinearIcon`**: Refers to the `LinearIcon` component imported earlier. This component will be rendered as the visual icon for the block.
*   **`bgColor: '#5E6AD2'`**: A hexadecimal color code that sets the background color for the block's representation in the UI, often for branding or visual distinction.

---

#### **`subBlocks` (User Interface Inputs)**

This is an array of objects, where each object defines a single input field or UI control that the user interacts with within this Linear block.

```typescript
  subBlocks: [
    // ... input field configurations ...
  ],
```

*   **1. Operation Selector**
    ```typescript
    {
      id: 'operation',
      title: 'Operation',
      type: 'dropdown',
      layout: 'full',
      options: [
        { label: 'Read Issues', id: 'read' },
        { label: 'Create Issue', id: 'write' },
      ],
      value: () => 'read',
    },
    ```
    *   **`id: 'operation'`**: A unique identifier for this input field.
    *   **`title: 'Operation'`**: The label displayed to the user for this field.
    *   **`type: 'dropdown'`**: Specifies that this is a dropdown (select) input.
    *   **`layout: 'full'`**: Dictates how the input should be laid out visually (e.g., taking full width).
    *   **`options`**: An array of objects, where each object defines a choice in the dropdown.
        *   `label`: The text shown to the user.
        *   `id`: The internal value associated with that choice.
    *   **`value: () => 'read'`**: A function that returns the default selected value for this dropdown (in this case, 'read').

*   **2. Linear Account Selector**
    ```typescript
    {
      id: 'credential',
      title: 'Linear Account',
      type: 'oauth-input',
      layout: 'full',
      provider: 'linear',
      serviceId: 'linear',
      requiredScopes: ['read', 'write'],
      placeholder: 'Select Linear account',
      required: true,
    },
    ```
    *   **`id: 'credential'`**: Identifier for this input, which will hold the selected Linear account credentials.
    *   **`title: 'Linear Account'`**: Label.
    *   **`type: 'oauth-input'`**: A specialized input type for selecting or connecting an OAuth account.
    *   **`provider: 'linear'`, `serviceId: 'linear'`**: These link this input to the specific Linear OAuth provider configured in the system.
    *   **`requiredScopes: ['read', 'write']`**: Specifies the necessary permissions (OAuth scopes) that the connected Linear account must have to perform operations within this block (both reading and writing issues).
    *   **`placeholder: 'Select Linear account'`**: Placeholder text displayed when no account is selected.
    *   **`required: true`**: Indicates that the user *must* provide a Linear account to use this block.

*   **3. Team Selector (Basic Mode)**
    ```typescript
    {
      id: 'teamId',
      title: 'Team',
      type: 'project-selector',
      layout: 'full',
      canonicalParamId: 'teamId',
      provider: 'linear',
      serviceId: 'linear',
      placeholder: 'Select a team',
      dependsOn: ['credential'],
      mode: 'basic',
    },
    ```
    *   **`id: 'teamId'`**: Identifier for this input.
    *   **`type: 'project-selector'`**: A custom input type designed to fetch and display a list of teams/projects from a service like Linear.
    *   **`canonicalParamId: 'teamId'`**: This is important. It specifies the *final parameter name* that this input's value will map to when passed to the backend tools. This allows multiple UI inputs (like a selector and a manual input) to ultimately provide the same logical parameter.
    *   **`provider: 'linear'`, `serviceId: 'linear'`**: Again, links to the Linear service to fetch data for the selector.
    *   **`dependsOn: ['credential']`**: This input will only become active or populate its options *after* the `credential` (Linear account) has been selected.
    *   **`mode: 'basic'`**: Indicates this is for the standard, user-friendly interface.

*   **4. Project Selector (Basic Mode)**
    ```typescript
    {
      id: 'projectId',
      title: 'Project',
      type: 'project-selector',
      layout: 'full',
      canonicalParamId: 'projectId',
      provider: 'linear',
      serviceId: 'linear',
      placeholder: 'Select a project',
      dependsOn: ['credential', 'teamId'],
      mode: 'basic',
    },
    ```
    *   Similar to the `teamId` selector, but for projects.
    *   **`canonicalParamId: 'projectId'`**: Maps to the logical `projectId` parameter.
    *   **`dependsOn: ['credential', 'teamId']`**: This input will only activate and populate *after* both a `credential` and a `teamId` have been selected, ensuring a logical flow.

*   **5. Manual Team ID Input (Advanced Mode)**
    ```typescript
    {
      id: 'manualTeamId',
      title: 'Team ID',
      type: 'short-input',
      layout: 'full',
      canonicalParamId: 'teamId', // Maps to the same logical 'teamId' as the selector
      placeholder: 'Enter Linear team ID',
      mode: 'advanced',
    },
    ```
    *   **`id: 'manualTeamId'`**: Identifier for this simple text input.
    *   **`type: 'short-input'`**: A standard single-line text input.
    *   **`canonicalParamId: 'teamId'`**: *Crucially*, this input maps to the *same* logical `teamId` parameter as the `teamId` project selector. This means the backend doesn't care if the ID came from a selector or manual entry.
    *   **`mode: 'advanced'`**: This input is likely hidden by default and only shown when the user switches to an "advanced" mode in the UI. It provides flexibility for users who prefer to manually enter IDs rather than use selectors.

*   **6. Manual Project ID Input (Advanced Mode)**
    ```typescript
    {
      id: 'manualProjectId',
      title: 'Project ID',
      type: 'short-input',
      layout: 'full',
      canonicalParamId: 'projectId', // Maps to the same logical 'projectId' as the selector
      placeholder: 'Enter Linear project ID',
      mode: 'advanced',
    },
    ```
    *   Similar to `manualTeamId`, but for `projectId`. Also in `advanced` mode and maps to the same logical `projectId` parameter as the selector.

*   **7. Title Input (Conditional)**
    ```typescript
    {
      id: 'title',
      title: 'Title',
      type: 'short-input',
      layout: 'full',
      condition: { field: 'operation', value: ['write'] },
      required: true,
    },
    ```
    *   **`id: 'title'`**: Identifier for the issue title input.
    *   **`condition: { field: 'operation', value: ['write'] }`**: This is a powerful feature. This input field will **only be shown** in the UI if the `operation` dropdown (`field: 'operation'`) has a value of `'write'`. This ensures a cleaner UI by only showing relevant fields.
    *   **`required: true`**: If this field is visible (i.e., when `operation` is 'write'), it must be filled out by the user.

*   **8. Description Input (Conditional)**
    ```typescript
    {
      id: 'description',
      title: 'Description',
      type: 'long-input',
      layout: 'full',
      condition: { field: 'operation', value: ['write'] },
    },
    ```
    *   Similar to the `title` input, but for the issue description.
    *   **`type: 'long-input'`**: A multi-line text area input.
    *   **`condition`**: Also only visible when the `operation` is set to 'write'.

---

#### **`tools` (Backend Integration Logic)**

This section defines how the block interacts with the backend "tools" (API wrappers or services) to perform its actual work.

```typescript
  tools: {
    access: ['linear_read_issues', 'linear_create_issue'],
    config: {
      // ... tool selection and parameter mapping ...
    },
  },
```

*   **`access: ['linear_read_issues', 'linear_create_issue']`**: This array lists the specific backend tools (often corresponding to API functions) that this block is authorized to call. This can be used for security checks or to inform the backend about available functionalities.
    *   `linear_read_issues`: A tool to read Linear issues.
    *   `linear_create_issue`: A tool to create a new Linear issue.

*   **`config`**: This object contains the logic to dynamically select which tool to use and how to map the user's UI inputs to the parameters required by that tool.
    *   **`tool: (params) => ...`**: This is a function that takes all the user's input `params` (from the `subBlocks`) and returns the *name* of the specific backend tool to be executed.
        ```typescript
        tool: (params) =>
          params.operation === 'write' ? 'linear_create_issue' : 'linear_read_issues',
        ```
        *   **Simplified Logic**: If the user selected 'write' for the `operation` dropdown, the `linear_create_issue` tool will be called. Otherwise (if 'read' was selected), the `linear_read_issues` tool will be called.

    *   **`params: (params) => { ... }`**: This is a crucial function. It takes all the user's input `params` and transforms them into a structured object containing *only* the parameters required by the selected backend tool, along with any necessary validation.
        ```typescript
        params: (params) => {
          // Handle both selector and manual inputs
          const effectiveTeamId = (params.teamId || params.manualTeamId || '').trim()
          const effectiveProjectId = (params.projectId || params.manualProjectId || '').trim()

          if (!effectiveTeamId) {
            throw new Error('Team ID is required.')
          }
          if (!effectiveProjectId) {
            throw new Error('Project ID is required.')
          }

          if (params.operation === 'write') {
            if (!params.title?.trim()) {
              throw new Error('Title is required for creating issues.')
            }
            if (!params.description?.trim()) {
              throw new Error('Description is required for creating issues.')
            }
            return {
              credential: params.credential,
              teamId: effectiveTeamId,
              projectId: effectiveProjectId,
              title: params.title,
              description: params.description,
            }
          }
          return {
            credential: params.credential,
            teamId: effectiveTeamId,
            projectId: effectiveProjectId,
          }
        },
        ```
        *   **Simplifying `effectiveTeamId` and `effectiveProjectId`**: This logic combines the basic and advanced inputs for Team ID and Project ID.
            *   `(params.teamId || params.manualTeamId || '')` means "use `params.teamId` if it exists and is truthy, otherwise use `params.manualTeamId` if it exists and is truthy, otherwise use an empty string." This gracefully handles whether the user provided an ID via the selector or manually.
            *   `.trim()` removes any leading or trailing whitespace from the resulting ID.
        *   **Validation**:
            *   `if (!effectiveTeamId) { throw new Error('Team ID is required.') }`: Checks if an effective team ID was provided after consolidation.
            *   `if (!effectiveProjectId) { throw new Error('Project ID is required.') }`: Checks for a project ID.
        *   **Conditional Parameters for 'write'**:
            *   `if (params.operation === 'write')`: If the user wants to create an issue, additional validation is performed for `title` and `description`.
            *   If validation passes, it returns an object with all the necessary parameters for creating an issue: `credential`, `teamId`, `projectId`, `title`, and `description`.
        *   **Parameters for 'read'**:
            *   If `params.operation` is not 'write' (meaning it's 'read'), it returns an object with only `credential`, `teamId`, and `projectId`, as title and description are not needed for reading.

---

#### **`inputs` (Input Schema for Workflow)**

This object defines the *expected data structure* for inputs that this block might receive from *other* blocks in a workflow. This is a schema, not the actual UI inputs.

```typescript
  inputs: {
    operation: { type: 'string', description: 'Operation to perform' },
    credential: { type: 'string', description: 'Linear access token' },
    teamId: { type: 'string', description: 'Linear team identifier' },
    projectId: { type: 'string', description: 'Linear project identifier' },
    manualTeamId: { type: 'string', description: 'Manual team identifier' },
    manualProjectId: { type: 'string', description: 'Manual project identifier' },
    title: { type: 'string', description: 'Issue title' },
    description: { type: 'string', description: 'Issue description' },
  },
```

*   Each property here corresponds to a potential input data point.
*   **`type: 'string'`**: Defines the expected data type.
*   **`description`**: Provides a human-readable explanation of what this input represents.
*   Notice how `teamId`, `projectId`, `manualTeamId`, and `manualProjectId` are all listed. While `tools.config.params` handles their consolidation for the *backend call*, here they represent the *potential raw inputs* this block might process if values are passed in from elsewhere in a workflow (e.g., from a preceding block).

#### **`outputs` (Output Schema for Workflow)**

This object defines the *expected data structure* for the output that this block will produce and make available to subsequent blocks in a workflow.

```typescript
  outputs: {
    issues: { type: 'json', description: 'Issues list' },
    issue: { type: 'json', description: 'Single issue data' },
  },
}
```

*   **`issues: { type: 'json', description: 'Issues list' }`**: If the block performs a "read" operation, it's expected to output a JSON array representing a list of Linear issues.
*   **`issue: { type: 'json', description: 'Single issue data' }`**: If the block performs a "write" (create) operation, it's expected to output a JSON object representing the newly created Linear issue.
*   **`type: 'json'`**: Indicates the output will be a JSON-formatted object or array.
*   **`description`**: Explains what the output represents.

---

### **Summary of Complex Logic**

The most sophisticated parts of this configuration are:

1.  **Conditional UI (`subBlocks[].condition`)**: Inputs like `title` and `description` only appear when the `operation` is set to 'write', making the UI dynamic and less cluttered.
2.  **Input Consolidation and Validation (`tools.config.params`)**: The `params` function intelligently combines inputs from both basic (`teamId`, `projectId`) and advanced (`manualTeamId`, `manualProjectId`) UI fields. It then performs necessary validation (e.g., ensuring IDs, titles, and descriptions are present when required) before constructing the final payload for the backend tool.
3.  **Dynamic Tool Selection (`tools.config.tool`)**: The `tool` function dynamically chooses which backend API (`linear_read_issues` or `linear_create_issue`) to call based on the user's `operation` selection.

This `LinearBlock` configuration provides a comprehensive, declarative way to define a powerful, user-friendly, and robust integration with Linear within a workflow automation platform.