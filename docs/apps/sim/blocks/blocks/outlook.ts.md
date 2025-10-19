This TypeScript file defines a configuration for an "Outlook Block" within a larger platform, likely a workflow automation tool or an AI orchestration engine. Think of it as a blueprint that tells the platform:

*   **What this Outlook integration is capable of.**
*   **How users should interact with it through a user interface (UI).**
*   **What underlying backend services (tools) it uses.**
*   **What data it expects as input and what data it can produce as output.**
*   **How it can act as a trigger for workflows.**

This approach is highly declarative: instead of writing procedural code for the UI or backend logic, you define a structured data object (`OutlookBlock`) that the platform interprets to render the UI, validate inputs, execute operations, and process results.

---

### Purpose of This File

The primary purpose of this file is to create a *reusable configuration object* for interacting with Microsoft Outlook. This `OutlookBlock` object will be consumed by two main parts of a platform:

1.  **Frontend/UI**: It provides all the necessary information to render a user-friendly form or interface for configuring Outlook operations (e.g., sending an email, reading emails). This includes field types, labels, placeholders, conditions for when fields should appear, and authentication requirements.
2.  **Backend/Execution Engine**: It specifies which backend "tools" or APIs should be invoked based on user selections, how to transform the user's input data into the format expected by these tools, and what kind of data to expect back.

In essence, this file makes the Outlook integration discoverable, configurable, and executable within the platform without needing to write custom UI code or complex backend routing logic for each new Outlook feature.

---

### Simplified Complex Logic

The most "complex" logic here revolves around two main concepts:

1.  **Dynamic UI (`subBlocks` with `condition` and `mode`)**: The UI fields a user sees are not static. They change based on the "Operation" selected (e.g., 'To' field appears for sending, 'Message ID' appears for forwarding). Some fields are also designated as "advanced" and might only show up when a user toggles an "Advanced Settings" option. This dynamic behavior is defined declaratively using `condition` objects within `subBlocks`.

2.  **Mapping UI to Backend (`tools.config`)**: The `tools.config` object contains functions that translate what the user configures in the UI into commands and parameters for the actual backend services.
    *   `tool(params)`: This function decides *which specific backend Outlook API* to call based on the user's chosen "Operation" (e.g., if the user selects "Send Email," it tells the backend to use the `outlook_send` tool).
    *   `params(params)`: This function massages the data provided by the user (from the UI) into the exact format expected by the chosen backend tool. A good example is how it handles the "Folder" input, allowing users to select from a dropdown or manually type a folder name, then consolidating these into a single `folder` parameter for the backend.

These two areas ensure a flexible user experience and a robust connection to backend services without hardcoding every possible permutation.

---

### Line-by-Line Explanation

Let's break down the code step by step:

```typescript
import { OutlookIcon } from '@/components/icons'
import type { BlockConfig } from '@/blocks/types'
import { AuthMode } from '@/blocks/types'
import type { OutlookResponse } from '@/tools/outlook/types'
```
These lines import necessary types and components:
*   `OutlookIcon`: A React component (or similar) used to display an icon for the Outlook block in the UI. The `@/components/icons` path suggests it's a common icon library within the project.
*   `BlockConfig`: A TypeScript type definition that outlines the expected structure for any block configuration object. This provides strong type-checking and ensures consistency across different blocks in the platform. It's generic, `BlockConfig<OutlookResponse>`, meaning this block, when executed, will produce an output conforming to the `OutlookResponse` type.
*   `AuthMode`: An enum (or similar) defining different authentication methods (e.g., OAuth, API Key). This is used to specify how this block authenticates with external services.
*   `OutlookResponse`: A TypeScript type definition describing the structure of the data expected as a response from Outlook operations.

---

```typescript
export const OutlookBlock: BlockConfig<OutlookResponse> = {
  type: 'outlook',
  name: 'Outlook',
  description: 'Access Outlook',
  authMode: AuthMode.OAuth,
  longDescription:
    'Integrate Outlook into the workflow. Can read, draft, and send email messages. Can be used in trigger mode to trigger a workflow when a new email is received.',
  docsLink: 'https://docs.sim.ai/tools/outlook',
  category: 'tools',
  triggerAllowed: true,
  bgColor: '#E0E0E0',
  icon: OutlookIcon,
```
This is the declaration of the `OutlookBlock` constant, typed as `BlockConfig<OutlookResponse>`. It's an object literal containing the top-level configuration for the Outlook integration.

*   `type: 'outlook'`: A unique identifier for this block type within the platform.
*   `name: 'Outlook'`: The human-readable name displayed in the UI.
*   `description: 'Access Outlook'`: A short summary of what the block does, often shown in tooltips or search results.
*   `authMode: AuthMode.OAuth`: Specifies that this block uses OAuth (Open Authorization) for authentication, which is common for services like Microsoft Outlook.
*   `longDescription`: A more detailed explanation of the block's capabilities, visible when a user learns more about it. It highlights key actions (read, draft, send, trigger) that can be performed.
*   `docsLink: 'https://docs.sim.ai/tools/outlook'`: A URL pointing to more extensive documentation for this block.
*   `category: 'tools'`: Classifies this block into a category (e.g., 'tools', 'logic', 'data'). This helps organize blocks in a UI.
*   `triggerAllowed: true`: Indicates that this block can be configured to act as a trigger, meaning a new Outlook event (like receiving an email) can start a workflow.
*   `bgColor: '#E0E0E0'`: A hexadecimal color code, likely used for the block's background color in the UI, providing visual distinction.
*   `icon: OutlookIcon`: The imported React component for the block's visual icon.

---

```typescript
  subBlocks: [
    {
      id: 'operation',
      title: 'Operation',
      type: 'dropdown',
      layout: 'full',
      options: [
        { label: 'Send Email', id: 'send_outlook' },
        { label: 'Draft Email', id: 'draft_outlook' },
        { label: 'Read Email', id: 'read_outlook' },
        { label: 'Forward Email', id: 'forward_outlook' },
      ],
      value: () => 'send_outlook',
    },
```
The `subBlocks` array defines the input fields and UI elements that will be rendered when a user configures this Outlook block. Each object in this array represents a distinct UI control.

This first `subBlock` defines a dropdown menu:
*   `id: 'operation'`: A unique identifier for this input field. Its value will be crucial for conditionally displaying other fields and for determining the backend tool.
*   `title: 'Operation'`: The label displayed above the dropdown in the UI.
*   `type: 'dropdown'`: Specifies that this is a dropdown UI component.
*   `layout: 'full'`: Dictates that this field should take up the full width available in its layout container.
*   `options`: An array of objects, where each object represents a selectable option in the dropdown.
    *   `label`: The human-readable text displayed to the user.
    *   `id`: The internal value associated with the option, which will be passed to the backend.
*   `value: () => 'send_outlook'`: A function that provides the default selected value for this dropdown (in this case, 'Send Email').

---

```typescript
    {
      id: 'credential',
      title: 'Microsoft Account',
      type: 'oauth-input',
      layout: 'full',
      provider: 'outlook',
      serviceId: 'outlook',
      requiredScopes: [
        'Mail.ReadWrite',
        'Mail.ReadBasic',
        'Mail.Read',
        'Mail.Send',
        'offline_access',
        'openid',
        'profile',
        'email',
      ],
      placeholder: 'Select Microsoft account',
      required: true,
    },
```
This `subBlock` configures an OAuth credential input field:
*   `id: 'credential'`: Identifier for the input.
*   `title: 'Microsoft Account'`: Label for the input.
*   `type: 'oauth-input'`: A specialized UI component designed to handle OAuth credentials (e.g., allowing a user to link their Microsoft account).
*   `layout: 'full'`: Full-width layout.
*   `provider: 'outlook'`: Specifies the OAuth provider (e.g., tells the system which OAuth integration to use).
*   `serviceId: 'outlook'`: Another identifier for the specific service.
*   `requiredScopes`: An array of strings defining the permissions (scopes) this block needs from the user's Microsoft account. These are crucial for OAuth to function correctly (e.g., `Mail.ReadWrite` allows reading and writing emails, `Mail.Send` allows sending, `offline_access` allows refreshing tokens).
*   `placeholder: 'Select Microsoft account'`: Hint text displayed when no account is selected.
*   `required: true`: Marks this field as mandatory for the block to be configured.

---

```typescript
    {
      id: 'to',
      title: 'To',
      type: 'short-input',
      layout: 'full',
      placeholder: 'Recipient email address',
      condition: {
        field: 'operation',
        value: ['send_outlook', 'draft_outlook', 'forward_outlook'],
      },
      required: true,
    },
```
This is a standard text input field for the recipient's email address:
*   `id: 'to'`: Identifier for the input.
*   `title: 'To'`: Label for the input.
*   `type: 'short-input'`: A single-line text input field.
*   `layout: 'full'`: Full-width layout.
*   `placeholder: 'Recipient email address'`: Hint text.
*   `condition`: This is key. It's an object that defines when this field should be visible.
    *   `field: 'operation'`: The `condition` checks the value of the `operation` field (the dropdown defined earlier).
    *   `value: ['send_outlook', 'draft_outlook', 'forward_outlook']`: The 'To' field will only be displayed if the `operation` dropdown is set to 'Send Email', 'Draft Email', or 'Forward Email'.
*   `required: true`: This field is mandatory when displayed.

---

```typescript
    {
      id: 'messageId',
      title: 'Message ID',
      type: 'short-input',
      layout: 'full',
      placeholder: 'Message ID to forward',
      condition: { field: 'operation', value: ['forward_outlook'] },
      required: true,
    },
    {
      id: 'comment',
      title: 'Comment',
      type: 'long-input',
      layout: 'full',
      placeholder: 'Optional comment to include when forwarding',
      condition: { field: 'operation', value: ['forward_outlook'] },
      required: false,
    },
```
These fields are specific to the "Forward Email" operation:
*   `id: 'messageId'`, `title: 'Message ID'`: A short input for the ID of the email to be forwarded.
*   `condition: { field: 'operation', value: ['forward_outlook'] }`: Only shown when the operation is 'Forward Email'. It's `required: true`.
*   `id: 'comment'`, `title: 'Comment'`: A `long-input` (multi-line text area) for an optional comment.
*   `condition: { field: 'operation', value: ['forward_outlook'] }`: Also only shown for 'Forward Email'. It's `required: false` (optional).

---

```typescript
    {
      id: 'subject',
      title: 'Subject',
      type: 'short-input',
      layout: 'full',
      placeholder: 'Email subject',
      condition: { field: 'operation', value: ['send_outlook', 'draft_outlook'] },
      required: true,
    },
    {
      id: 'body',
      title: 'Body',
      type: 'long-input',
      layout: 'full',
      placeholder: 'Email content',
      condition: { field: 'operation', value: ['send_outlook', 'draft_outlook'] },
      required: true,
    },
```
These fields are for the email `subject` and `body`:
*   Both are `short-input` for subject and `long-input` for body respectively.
*   Both have a `condition` that makes them appear only for 'Send Email' or 'Draft Email' operations.
*   Both are `required: true`.

---

```typescript
    // Advanced Settings - Threading
    {
      id: 'replyToMessageId',
      title: 'Reply to Message ID',
      type: 'short-input',
      layout: 'full',
      placeholder: 'Message ID to reply to (for threading)',
      condition: { field: 'operation', value: ['send_outlook'] },
      mode: 'advanced',
      required: false,
    },
    {
      id: 'conversationId',
      title: 'Conversation ID',
      type: 'short-input',
      layout: 'full',
      placeholder: 'Conversation ID for threading',
      condition: { field: 'operation', value: ['send_outlook'] },
      mode: 'advanced',
      required: false,
    },
```
These fields are for advanced email threading features:
*   `id: 'replyToMessageId'` and `id: 'conversationId'` are `short-input` fields.
*   They are only shown for the 'Send Email' `operation`.
*   `mode: 'advanced'`: This crucial property indicates these fields should only be visible when the UI is in an "advanced" display mode (e.g., after clicking an "Advanced Settings" toggle).
*   Both are `required: false` (optional).

---

```typescript
    // Advanced Settings - Additional Recipients
    {
      id: 'cc',
      title: 'CC',
      type: 'short-input',
      layout: 'full',
      placeholder: 'CC recipients (comma-separated)',
      condition: { field: 'operation', value: ['send_outlook', 'draft_outlook'] },
      mode: 'advanced',
      required: false,
    },
    {
      id: 'bcc',
      title: 'BCC',
      type: 'short-input',
      layout: 'full',
      placeholder: 'BCC recipients (comma-separated)',
      condition: { field: 'operation', value: ['send_outlook', 'draft_outlook'] },
      mode: 'advanced',
      required: false,
    },
```
These fields allow adding CC and BCC recipients:
*   `id: 'cc'` and `id: 'bcc'` are `short-input` fields.
*   They appear for 'Send Email' or 'Draft Email' operations.
*   `mode: 'advanced'`: Also part of advanced settings.
*   Both are `required: false` (optional).

---

```typescript
    // Read Email Fields - Add folder selector (basic mode)
    {
      id: 'folder',
      title: 'Folder',
      type: 'folder-selector',
      layout: 'full',
      canonicalParamId: 'folder',
      provider: 'outlook',
      serviceId: 'outlook',
      requiredScopes: ['Mail.ReadWrite', 'Mail.ReadBasic', 'Mail.Read'],
      placeholder: 'Select Outlook folder',
      dependsOn: ['credential'],
      mode: 'basic',
      condition: { field: 'operation', value: 'read_outlook' },
    },
    // Manual folder input (advanced mode)
    {
      id: 'manualFolder',
      title: 'Folder',
      type: 'short-input',
      layout: 'full',
      canonicalParamId: 'folder',
      placeholder: 'Enter Outlook folder name (e.g., INBOX, SENT, or custom folder)',
      mode: 'advanced',
      condition: { field: 'operation', value: 'read_outlook' },
    },
```
These two `subBlocks` handle selecting an email folder, demonstrating both a user-friendly selector and a manual input for advanced users:
*   **Folder Selector (`id: 'folder'`)**:
    *   `type: 'folder-selector'`: A specialized UI component that likely fetches and displays a list of available Outlook folders from the user's account.
    *   `canonicalParamId: 'folder'`: This is important. It tells the system that even though this UI field has `id: 'folder'`, its value should ultimately be mapped to a backend parameter also named `'folder'`. (This will become clearer when we look at the `tools.config.params` function).
    *   `provider`, `serviceId`, `requiredScopes`: Similar to the `credential` field, these ensure the folder selector has the necessary permissions to fetch folders.
    *   `dependsOn: ['credential']`: This field will only become active and fetch folders once a `credential` (Microsoft Account) has been selected.
    *   `mode: 'basic'`: This is the default, simpler way for users to select a folder.
    *   `condition: { field: 'operation', value: 'read_outlook' }`: Only visible for the 'Read Email' operation.
*   **Manual Folder Input (`id: 'manualFolder'`)**:
    *   `type: 'short-input'`: A simple text box.
    *   `canonicalParamId: 'folder'`: Again, its value maps to the same backend `'folder'` parameter. This allows the backend to abstract away whether the folder came from a selector or manual input.
    *   `mode: 'advanced'`: Only visible in advanced mode.
    *   `condition: { field: 'operation', value: 'read_outlook' }`: Only visible for the 'Read Email' operation.

---

```typescript
    {
      id: 'maxResults',
      title: 'Number of Emails',
      type: 'short-input',
      layout: 'full',
      placeholder: 'Number of emails to retrieve (default: 1, max: 10)',
      condition: { field: 'operation', value: 'read_outlook' },
    },
    {
      id: 'includeAttachments',
      title: 'Include Attachments',
      type: 'switch',
      layout: 'full',
      condition: { field: 'operation', value: 'read_outlook' },
    },
```
These fields control how emails are read:
*   `id: 'maxResults'`, `title: 'Number of Emails'`: A `short-input` for specifying how many emails to retrieve.
*   `id: 'includeAttachments'`, `title: 'Include Attachments'`: A `switch` (toggle) component to decide whether to fetch email attachments.
*   Both are only shown for the 'Read Email' operation.

---

```typescript
    // TRIGGER MODE: Trigger configuration (only shown when trigger mode is active)
    {
      id: 'triggerConfig',
      title: 'Trigger Configuration',
      type: 'trigger-config',
      layout: 'full',
      triggerProvider: 'outlook',
      availableTriggers: ['outlook_poller'],
    },
  ], // End of subBlocks array
```
This special `subBlock` configures the trigger functionality:
*   `id: 'triggerConfig'`, `title: 'Trigger Configuration'`: Identifiers and label.
*   `type: 'trigger-config'`: A specialized UI component for setting up workflow triggers. This component is likely only displayed when the block is being configured in "trigger mode" by the platform.
*   `triggerProvider: 'outlook'`: Specifies the provider for the trigger.
*   `availableTriggers: ['outlook_poller']`: Lists the specific trigger types available for Outlook (in this case, an `outlook_poller` which likely periodically checks for new emails).

---

```typescript
  tools: {
    access: ['outlook_send', 'outlook_draft', 'outlook_read', 'outlook_forward'],
    config: {
      tool: (params) => {
        switch (params.operation) {
          case 'send_outlook':
            return 'outlook_send'
          case 'read_outlook':
            return 'outlook_read'
          case 'draft_outlook':
            return 'outlook_draft'
          case 'forward_outlook':
            return 'outlook_forward'
          default:
            throw new Error(`Invalid Outlook operation: ${params.operation}`)
        }
      },
      params: (params) => {
        const { credential, folder, manualFolder, ...rest } = params

        // Handle both selector and manual folder input
        const effectiveFolder = (folder || manualFolder || '').trim()

        if (rest.operation === 'read_outlook') {
          rest.folder = effectiveFolder || 'INBOX'
        }

        return {
          ...rest,
          credential, // Keep the credential parameter
        }
      },
    },
  },
```
The `tools` object defines how the block interacts with backend services.

*   `access`: An array listing the backend "tools" (or API endpoints/functions) that this block is authorized to call. This acts as a whitelist.
*   `config`: Contains functions that dynamically configure which tool to use and how to prepare its parameters.
    *   `tool: (params) => { ... }`: This function takes the UI input `params` (an object containing all the `id` values from `subBlocks` and their current values) and returns the specific backend tool ID to execute.
        *   It uses a `switch` statement on `params.operation` to map the user's chosen operation (e.g., `'send_outlook'`) to the corresponding backend tool (e.g., `'outlook_send'`).
        *   If an unknown `operation` is provided, it throws an error.
    *   `params: (params) => { ... }`: This function is critical for transforming the UI inputs into the format the backend tool expects.
        *   `const { credential, folder, manualFolder, ...rest } = params`: It destructures the incoming `params` object. It specifically extracts `credential`, `folder` (from the selector), `manualFolder` (from the text input), and gathers all other parameters into `rest`.
        *   `const effectiveFolder = (folder || manualFolder || '').trim()`: This is the core logic for handling the dual folder input. It checks if `folder` (from the selector) has a value. If not, it checks `manualFolder`. If neither, it's an empty string. `.trim()` removes whitespace. This ensures only one "folder" value is considered.
        *   `if (rest.operation === 'read_outlook') { rest.folder = effectiveFolder || 'INBOX' }`: If the operation is "Read Email", it assigns the `effectiveFolder` to `rest.folder`. If `effectiveFolder` is still empty (meaning neither selector nor manual input provided a folder), it defaults to `'INBOX'`. This sets the folder parameter for the backend call.
        *   `return { ...rest, credential, }`: It returns a new object containing all the original `rest` parameters (potentially with the `folder` updated) and explicitly re-includes the `credential` parameter, ensuring it's always passed to the backend tool.

---

```typescript
  inputs: {
    operation: { type: 'string', description: 'Operation to perform' },
    credential: { type: 'string', description: 'Outlook access token' },
    // Send operation inputs
    to: { type: 'string', description: 'Recipient email address' },
    subject: { type: 'string', description: 'Email subject' },
    body: { type: 'string', description: 'Email content' },
    // Forward operation inputs
    messageId: { type: 'string', description: 'Message ID to forward' },
    comment: { type: 'string', description: 'Optional comment for forwarding' },
    // Read operation inputs
    folder: { type: 'string', description: 'Email folder' },
    manualFolder: { type: 'string', description: 'Manual folder name' },
    maxResults: { type: 'number', description: 'Maximum emails' },
    includeAttachments: { type: 'boolean', description: 'Include email attachments' },
  },
```
The `inputs` object defines the *schema* of the data that the backend expects to *receive* from this block. This is used for validation and documentation.
*   Each key (e.g., `operation`, `credential`, `to`) corresponds to an input parameter.
*   `type`: The expected data type (e.g., `string`, `number`, `boolean`).
*   `description`: A human-readable explanation of the parameter.
Notice how `folder` and `manualFolder` are both listed here as potential inputs, even though the `tools.config.params` function consolidates them for the *actual* backend call. This allows the system to validate inputs from both UI fields.

---

```typescript
  outputs: {
    // Common outputs
    message: { type: 'string', description: 'Response message' },
    results: { type: 'json', description: 'Operation results' },
    // Send operation specific outputs
    status: { type: 'string', description: 'Email send status (sent)' },
    timestamp: { type: 'string', description: 'Operation timestamp' },
    // Draft operation specific outputs
    messageId: { type: 'string', description: 'Draft message ID' },
    subject: { type: 'string', description: 'Draft email subject' },
    // Read operation specific outputs
    emailCount: { type: 'number', description: 'Number of emails retrieved' },
    emails: { type: 'json', description: 'Array of email objects' },
    emailId: { type: 'string', description: 'Individual email ID' },
    emailSubject: { type: 'string', description: 'Individual email subject' },
    bodyPreview: { type: 'string', description: 'Email body preview' },
    bodyContent: { type: 'string', description: 'Full email body content' },
    sender: { type: 'json', description: 'Email sender information' },
    from: { type: 'json', description: 'Email from information' },
    recipients: { type: 'json', description: 'Email recipients' },
    receivedDateTime: { type: 'string', description: 'Email received timestamp' },
    sentDateTime: { type: 'string', description: 'Email sent timestamp' },
    hasAttachments: { type: 'boolean', description: 'Whether email has attachments' },
    attachments: {
      type: 'json',
      description: 'Email attachments (if includeAttachments is enabled)',
    },
    isRead: { type: 'boolean', description: 'Whether email is read' },
    importance: { type: 'string', description: 'Email importance level' },
    // Trigger outputs
    email: { type: 'json', description: 'Email data from trigger' },
    rawEmail: { type: 'json', description: 'Complete raw email data from Microsoft Graph API' },
  },
```
The `outputs` object defines the *schema* of the data that this block can *produce* after execution. This is crucial for connecting this block's output to other blocks' inputs in a workflow.
*   It lists various possible output parameters, categorized by operation (common, send, draft, read, trigger).
*   Each output has a `type` (e.g., `string`, `number`, `boolean`, `json`) and a `description`.
*   `type: 'json'` typically indicates that the output is a complex object or array that can be parsed as JSON.
This comprehensive list allows downstream blocks to know what data they can expect to receive from the Outlook block, enabling robust workflow building.

---

```typescript
  triggers: {
    enabled: true,
    available: ['outlook_poller'],
  },
} // End of OutlookBlock object
```
The `triggers` object provides configuration specifically for when this block acts as a workflow trigger:
*   `enabled: true`: Confirms that this block can indeed be used as a trigger.
*   `available: ['outlook_poller']`: Specifies which specific trigger types are implemented for Outlook. This likely refers to a backend service that periodically checks for new emails or other Outlook events.

---

This detailed configuration provides a complete, self-describing blueprint for integrating Microsoft Outlook capabilities into a larger system, covering both user interaction and backend execution.