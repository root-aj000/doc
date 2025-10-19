This file defines a "Microsoft Planner" block configuration for a system (likely a workflow or integration platform like Sim.ai) that allows users to interact with Microsoft Planner. It specifies how this block should appear in the user interface, what inputs it takes, how those inputs are transformed into calls to underlying tools, and what outputs it produces.

---

## 🚀 Purpose of this File

The primary purpose of `microsoft_planner.ts` is to **define a reusable, configurable block for integrating with Microsoft Planner within a larger application or workflow orchestration system.**

Think of it as a blueprint for a user-facing component. When a user drags and drops a "Microsoft Planner" block into their workflow, this configuration tells the system:

1.  **What it is**: Its name, description, category, and visual representation (icon, color).
2.  **How to authenticate**: That it uses OAuth with Microsoft.
3.  **How to render its UI**: What input fields to display, their labels, types (dropdown, text input, file selector), placeholders, and how they become visible or required based on user selections (e.g., "show task ID field only if 'Read Task' is selected").
4.  **How to execute**: Which specific internal "tools" (API wrappers) to call based on the user's chosen operation (e.g., `microsoft_planner_read_task` or `microsoft_planner_create_task`).
5.  **How to map UI inputs to tool parameters**: A crucial part that translates user-friendly input fields into the specific parameters required by the underlying Microsoft Planner API calls. This includes validation and defaulting.
6.  **What outputs it provides**: The structure of the data it will return after execution.

In essence, it bridges the gap between a user-friendly visual block and the complex backend logic required to interact with Microsoft Planner.

---

## 🧠 Simplified Complex Logic

The most "complex" parts of this file lie in two main areas:

1.  **Defining Dynamic UI (`subBlocks` array)**:
    *   Imagine you're building a form. Some fields only appear when you select certain options from a dropdown. The `subBlocks` array is like the definition of this dynamic form.
    *   It specifies all possible input fields (like 'Operation', 'Plan ID', 'Task Title', etc.).
    *   For each field, it includes `condition` and `dependsOn` properties.
        *   `condition`: Says "this field should only be visible if another field (`field`) has one of these `value`s." For example, 'Task Title' only shows up if 'Operation' is 'Create Task'.
        *   `dependsOn`: Says "this field needs the values of these other fields before it can even fetch its own data or be fully functional." For example, 'Task ID' depends on 'Credential' and 'Plan ID' because you can't select a task without knowing which account and plan to look in.
    *   It also handles two ways to input a 'Task ID': a `file-selector` (dropdown to pick from existing tasks) for `mode: 'basic'` and a `short-input` (manual text entry) for `mode: 'advanced'`. Both ultimately map to the same `canonicalParamId: 'taskId'`, meaning they both control the *same underlying parameter*.

2.  **Transforming Inputs to Tool Parameters (`tools.config.params` function)**:
    *   This is the "brains" of the block where user-friendly inputs are translated into the precise data structure expected by the underlying Microsoft Planner API calls.
    *   **Operation Selection**: It first checks whether the user wants to `read_task` or `create_task`.
    *   **Common Parameters**: It gathers basic parameters like `credential` that are always needed.
    *   **Handling Task ID**: It elegantly handles the `taskId` and `manualTaskId` inputs from the UI. Since both refer to the same logical "Task ID", it prioritizes the one that has a value and trims any whitespace. This `effectiveTaskId` is then used.
    *   **`read_task` Logic**:
        *   If an `effectiveTaskId` is provided, it means the user wants a *specific* task.
        *   If no `effectiveTaskId` but a `planId` is provided, it means the user wants all tasks *within that plan*.
        *   If neither, it implies getting all tasks accessible to the user.
    *   **`create_task` Logic**:
        *   It performs **validation**: checks if `planId` and `title` are present, throwing an error if not (because these are essential for creating a task).
        *   It then constructs the parameters for creating a task, including optional fields like `description`, `dueDateTime`, `assigneeUserId`, and `bucketId` only if they have values.

In short, the `subBlocks` define *what the user sees and interacts with*, while the `tools.config.params` defines *how those interactions translate into actions behind the scenes*.

---

## 📝 Line-by-Line Explanation

```typescript
// Imports necessary components and types from other parts of the application.
import { MicrosoftPlannerIcon } from '@/components/icons' // Imports the SVG icon component for Microsoft Planner.
import type { BlockConfig } from '@/blocks/types' // Imports the base type definition for a block configuration.
import { AuthMode } from '@/blocks/types' // Imports an enum (AuthMode) for specifying authentication methods.
import type { MicrosoftPlannerResponse } from '@/tools/microsoft_planner/types' // Imports the type definition for the response structure from Microsoft Planner tools.

// Defines an interface for the parameters that the Microsoft Planner block will accept
// when interacting with the underlying Microsoft Planner tools.
interface MicrosoftPlannerBlockParams {
  credential: string // The ID of the Microsoft account credential to use.
  accessToken?: string // Optional access token (might be used internally or for specific scenarios).
  planId?: string // Optional ID of the Planner plan.
  taskId?: string // Optional ID of a specific task.
  title?: string // Optional title for a new task.
  description?: string // Optional description for a new task.
  dueDateTime?: string // Optional due date and time for a task (ISO 8601 format).
  assigneeUserId?: string // Optional user ID to assign a task to.
  bucketId?: string // Optional ID of the bucket to put the task in.
  [key: string]: string | number | boolean | undefined // An index signature allowing for additional, unspecified string/number/boolean/undefined properties. This makes the interface more flexible for other potential parameters.
}

// Defines the main configuration object for the Microsoft Planner block.
// It's typed as BlockConfig, parameterized with MicrosoftPlannerResponse,
// meaning this block will output data of type MicrosoftPlannerResponse.
export const MicrosoftPlannerBlock: BlockConfig<MicrosoftPlannerResponse> = {
  type: 'microsoft_planner', // A unique identifier for this block type within the system.
  name: 'Microsoft Planner', // The user-friendly name displayed in the UI.
  description: 'Read and create tasks in Microsoft Planner', // A short description for the block.
  authMode: AuthMode.OAuth, // Specifies that this block uses OAuth for authentication.
  longDescription: 'Integrate Microsoft Planner into the workflow. Can read and create tasks.', // A more detailed description.
  docsLink: 'https://docs.sim.ai/tools/microsoft_planner', // Link to relevant documentation for this block.
  category: 'tools', // Categorizes the block (e.g., 'tools', 'communication', 'data').
  bgColor: '#E0E0E0', // Background color for the block's UI representation.
  icon: MicrosoftPlannerIcon, // The React component used to render the block's icon.

  // Defines the input fields and controls that will be displayed in the block's UI.
  // Each object in this array represents a single UI element.
  subBlocks: [
    {
      id: 'operation', // Unique identifier for this input field.
      title: 'Operation', // User-friendly label for the input.
      type: 'dropdown', // The type of UI control (a select/dropdown menu).
      layout: 'full', // Specifies the layout of the input (e.g., takes full width).
      options: [
        { label: 'Read Task', id: 'read_task' }, // Option for reading an existing task.
        { label: 'Create Task', id: 'create_task' }, // Option for creating a new task.
      ],
    },
    {
      id: 'credential', // ID for the Microsoft account selection field.
      title: 'Microsoft Account', // Label for the field.
      type: 'oauth-input', // A custom input type for selecting an OAuth credential.
      layout: 'full',
      provider: 'microsoft-planner', // The OAuth provider this credential belongs to.
      serviceId: 'microsoft-planner', // The specific service ID for this provider.
      requiredScopes: [
        'openid', // Standard OpenID Connect scope for identity information.
        'profile', // Standard scope for user profile information.
        'email', // Standard scope for user email address.
        'Group.ReadWrite.All', // Allows reading and writing to all groups (needed for Planner access).
        'Group.Read.All', // Allows reading all groups.
        'Tasks.ReadWrite', // Allows reading and writing user tasks in Planner.
        'offline_access', // Allows the application to get refresh tokens, enabling long-term access.
      ],
      placeholder: 'Select Microsoft account', // Placeholder text when no account is selected.
    },
    {
      id: 'planId', // ID for the Plan ID input.
      title: 'Plan ID', // Label for the field.
      type: 'short-input', // A single-line text input.
      layout: 'full',
      placeholder: 'Enter the plan ID', // Placeholder text.
      condition: { field: 'operation', value: ['create_task', 'read_task'] }, // This field is only visible if 'operation' is 'create_task' or 'read_task'.
      dependsOn: ['credential'], // This field's functionality might depend on 'credential' being selected first (e.g., to fetch available plans).
    },
    {
      id: 'taskId', // ID for selecting a Task ID (basic mode).
      title: 'Task ID', // Label.
      type: 'file-selector', // A specialized selector that might list available tasks.
      layout: 'full',
      placeholder: 'Select a task', // Placeholder text.
      provider: 'microsoft-planner', // Specifies the provider for the selector.
      condition: { field: 'operation', value: ['read_task'] }, // Visible only if 'operation' is 'read_task'.
      dependsOn: ['credential', 'planId'], // Depends on credential and planId to fetch tasks.
      mode: 'basic', // This is for the 'basic' UI mode.
      canonicalParamId: 'taskId', // Maps this UI field to the underlying 'taskId' parameter.
    },

    // Advanced mode for manual Task ID entry.
    {
      id: 'manualTaskId', // ID for manual Task ID input.
      title: 'Manual Task ID', // Label.
      type: 'short-input', // Single-line text input.
      layout: 'full',
      placeholder: 'Enter the task ID', // Placeholder text.
      condition: { field: 'operation', value: ['read_task'] }, // Visible only if 'operation' is 'read_task'.
      dependsOn: ['credential', 'planId'], // Depends on credential and planId.
      mode: 'advanced', // This is for the 'advanced' UI mode.
      canonicalParamId: 'taskId', // Also maps to the same 'taskId' underlying parameter, allowing flexibility.
    },
    {
      id: 'title', // ID for the Task Title input.
      title: 'Task Title', // Label.
      type: 'short-input', // Single-line text input.
      layout: 'full',
      placeholder: 'Enter the task title', // Placeholder text.
      condition: { field: 'operation', value: ['create_task'] }, // Visible only if 'operation' is 'create_task'.
    },
    {
      id: 'description', // ID for the Description input.
      title: 'Description', // Label.
      type: 'long-input', // A multi-line text area.
      layout: 'full',
      placeholder: 'Enter task description (optional)', // Placeholder text.
      condition: { field: 'operation', value: ['create_task'] }, // Visible only if 'operation' is 'create_task'.
    },
    {
      id: 'dueDateTime', // ID for the Due Date input.
      title: 'Due Date', // Label.
      type: 'short-input', // Single-line text input.
      layout: 'full',
      placeholder: 'Enter due date in ISO 8601 format (e.g., 2024-12-31T23:59:59Z)', // Placeholder with format hint.
      condition: { field: 'operation', value: ['create_task'] }, // Visible only if 'operation' is 'create_task'.
    },
    {
      id: 'assigneeUserId', // ID for the Assignee User ID input.
      title: 'Assignee User ID', // Label.
      type: 'short-input', // Single-line text input.
      layout: 'full',
      placeholder: 'Enter the user ID to assign this task to (optional)', // Placeholder.
      condition: { field: 'operation', value: ['create_task'] }, // Visible only if 'operation' is 'create_task'.
    },
    {
      id: 'bucketId', // ID for the Bucket ID input.
      title: 'Bucket ID', // Label.
      type: 'short-input', // Single-line text input.
      layout: 'full',
      placeholder: 'Enter the bucket ID to organize the task (optional)', // Placeholder.
      condition: { field: 'operation', value: ['create_task'] }, // Visible only if 'operation' is 'create_task'.
    },
  ],

  // Defines how this block interacts with underlying "tools" (API wrappers/functions).
  tools: {
    access: ['microsoft_planner_read_task', 'microsoft_planner_create_task'], // Lists the specific tools this block is allowed to use.
    config: {
      // The `tool` function determines which specific underlying tool to call based on block inputs.
      tool: (params) => {
        switch (params.operation) {
          case 'read_task': // If the 'operation' input is 'read_task'...
            return 'microsoft_planner_read_task' // ...then call the 'microsoft_planner_read_task' tool.
          case 'create_task': // If the 'operation' input is 'create_task'...
            return 'microsoft_planner_create_task' // ...then call the 'microsoft_planner_create_task' tool.
          default:
            // If an invalid or missing operation is provided, throw an error.
            throw new Error(`Invalid Microsoft Planner operation: ${params.operation}`)
        }
      },
      // The `params` function transforms the raw input values from the UI into the
      // structured parameters expected by the selected underlying tool.
      params: (params) => {
        // Destructures the input `params` object, extracting specific fields and
        // gathering any remaining fields into `rest`.
        const {
          credential,
          operation,
          planId,
          taskId,
          manualTaskId,
          title,
          description,
          dueDateTime,
          assigneeUserId,
          bucketId,
          ...rest
        } = params

        // Creates a base set of parameters, including any 'rest' parameters
        // and the mandatory 'credential'.
        const baseParams = {
          ...rest,
          credential,
        }

        // Determines the effective task ID, prioritizing `taskId` (from selector)
        // then `manualTaskId` (from manual input), then an empty string, and trimming whitespace.
        const effectiveTaskId = (taskId || manualTaskId || '').trim()

        // Logic for 'read_task' operation.
        if (operation === 'read_task') {
          // Creates a new object for read-specific parameters, starting with baseParams.
          const readParams: MicrosoftPlannerBlockParams = { ...baseParams }

          // If an effective task ID is available, add it to the read parameters.
          // This typically means fetching a specific task.
          if (effectiveTaskId) {
            readParams.taskId = effectiveTaskId
          }
          // If no specific task ID, but a plan ID is provided, add the plan ID.
          // This typically means fetching all tasks within a given plan.
          else if (planId?.trim()) {
            readParams.planId = planId.trim()
          }
          // If neither a task ID nor a plan ID, baseParams will be returned,
          // implying a request for all user tasks or a default behavior.

          return readParams
        }

        // Logic for 'create_task' operation.
        if (operation === 'create_task') {
          // Validation: Plan ID is required to create a task.
          if (!planId?.trim()) {
            throw new Error('Plan ID is required to create a task.')
          }
          // Validation: Task title is required to create a task.
          if (!title?.trim()) {
            throw new Error('Task title is required to create a task.')
          }

          // Creates a new object for create-specific parameters,
          // including baseParams, the required plan ID, and title.
          const createParams: MicrosoftPlannerBlockParams = {
            ...baseParams,
            planId: planId.trim(),
            title: title.trim(),
          }

          // Adds optional fields only if they have non-empty values.
          if (description?.trim()) {
            createParams.description = description.trim()
          }

          if (dueDateTime?.trim()) {
            createParams.dueDateTime = dueDateTime.trim()
          }

          if (assigneeUserId?.trim()) {
            createParams.assigneeUserId = assigneeUserId.trim()
          }

          if (bucketId?.trim()) {
            createParams.bucketId = bucketId.trim()
          }

          return createParams
        }

        // If no specific operation matches, return the base parameters.
        return baseParams
      },
    },
  },

  // Defines the expected *programmatic* inputs for the block. These are not directly tied
  // to the UI subBlocks but describe what data the block expects to receive if called
  // programmatically (e.g., from another block's output or an API call).
  inputs: {
    operation: { type: 'string', description: 'Operation to perform' },
    credential: { type: 'string', description: 'Microsoft account credential' },
    planId: { type: 'string', description: 'Plan ID' },
    taskId: { type: 'string', description: 'Task ID' },
    manualTaskId: { type: 'string', description: 'Manual Task ID' },
    title: { type: 'string', description: 'Task title' },
    description: { type: 'string', description: 'Task description' },
    dueDateTime: { type: 'string', description: 'Due date' },
    assigneeUserId: { type: 'string', description: 'Assignee user ID' },
    bucketId: { type: 'string', description: 'Bucket ID' },
  },

  // Defines the outputs this block will produce after successful execution.
  // These outputs can then be used as inputs for subsequent blocks in a workflow.
  outputs: {
    task: {
      type: 'json', // The output will be a JSON object.
      description:
        'The Microsoft Planner task object, including details such as id, title, description, status, due date, and assignees.', // Description of the output.
    },
    metadata: {
      type: 'json', // The output will be a JSON object.
      description:
        'Additional metadata about the operation, such as timestamps, request status, or other relevant information.', // Description of the output.
    },
  },
}
```