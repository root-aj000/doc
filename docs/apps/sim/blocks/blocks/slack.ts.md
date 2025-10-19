This TypeScript file defines a `BlockConfig` for a "Slack" integration within a larger workflow automation or application building platform. Think of it as a blueprint that tells the platform: "Here's how to display a Slack action in the UI, what inputs it needs, how to process those inputs, what backend tools to use, and what kind of data it produces."

It's a comprehensive configuration object that covers everything from UI representation to backend execution logic for interacting with Slack.

---

### Purpose of this File

The primary purpose of this file is to define the **"Slack Block"**. This block allows users of a platform (likely one that builds workflows or applications visually) to integrate various Slack functionalities into their processes.

Specifically, this `SlackBlock` configuration enables users to:
1.  **Send messages** to Slack channels.
2.  **Create Slack canvases**.
3.  **Read messages** from Slack channels.
4.  Optionally, **trigger workflows** based on Slack events (like messages in a channel).

It acts as a bridge, translating user selections and inputs in a graphical interface into concrete instructions and parameters for the underlying Slack API tools.

---

### Simplified Complex Logic

The most complex parts of this configuration are:

1.  **Conditional UI Fields (`subBlocks` with `condition`):** Instead of showing all possible input fields at once, the block dynamically displays fields based on user choices. For example, if you select "Send Message," only the "Message" input field appears. If you choose "Custom Bot" authentication, the "Bot Token" input shows up instead of the "Slack Account" selector.
2.  **Dynamic Tool Selection and Parameter Mapping (`tools.config`):** This section is the brain of the block's execution.
    *   It first determines *which* specific Slack operation (send, canvas, read) the user wants to perform based on their "Operation" dropdown selection.
    *   Then, it intelligently collects all the relevant user inputs (like channel, message text, authentication details) and formats them into a structured set of parameters that the chosen Slack API tool can understand and execute. This involves handling different authentication methods (OAuth vs. Bot Token) and ensuring all required data for the chosen operation is present.

In essence, this file defines a **smart, adaptive form** for Slack interactions, where the form's appearance and the way it processes data change based on what the user wants to achieve.

---

### Explanation of Each Line of Code

Let's break down the `SlackBlock` configuration object line by line and section by section.

#### Imports

```typescript
import { SlackIcon } from '@/components/icons'
import type { BlockConfig } from '@/blocks/types'
import { AuthMode } from '@/blocks/types'
import type { SlackResponse } from '@/tools/slack/types'
```

*   `import { SlackIcon } from '@/components/icons'`: Imports a visual icon component named `SlackIcon`. This is used to represent the Slack block visually in the platform's UI. The `@/components/icons` path suggests it's a shared UI component.
*   `import type { BlockConfig } from '@/blocks/types'`: Imports the `BlockConfig` type. This is a generic type that defines the expected structure for any block configuration in the system. The `type` keyword indicates it's only imported for type-checking purposes and won't be compiled into JavaScript.
*   `import { AuthMode } from '@/blocks/types'`: Imports the `AuthMode` enum. This enumeration likely defines different authentication methods supported by blocks, such as OAuth, API Key, etc.
*   `import type { SlackResponse } from '@/tools/slack/types'`: Imports the `SlackResponse` type. This type defines the expected structure of the data that the Slack block will output after successfully performing an operation.

#### Block Configuration Object (`SlackBlock`)

```typescript
export const SlackBlock: BlockConfig<SlackResponse> = {
```

*   `export const SlackBlock`: Declares and exports a constant variable named `SlackBlock`. This makes this configuration available for other parts of the application to import and use.
*   `: BlockConfig<SlackResponse>`: This is a TypeScript type annotation. It states that `SlackBlock` *must* conform to the `BlockConfig` interface, and specifically, this `BlockConfig` is configured to return a `SlackResponse` type when it completes its operation.

---

##### **General Block Metadata and Configuration**

```typescript
  type: 'slack',
  name: 'Slack',
  description: 'Send messages to Slack or trigger workflows from Slack events',
  authMode: AuthMode.OAuth,
  longDescription:
    'Integrate Slack into the workflow. Can send messages, create canvases, and read messages. Requires Bot Token instead of OAuth in advanced mode. Can be used in trigger mode to trigger a workflow when a message is sent to a channel.',
  docsLink: 'https://docs.sim.ai/tools/slack',
  category: 'tools',
  bgColor: '#611f69',
  icon: SlackIcon,
  triggerAllowed: true,
```

*   `type: 'slack'`: A unique identifier string for this type of block within the system.
*   `name: 'Slack'`: The user-friendly display name for this block in the UI.
*   `description: 'Send messages to Slack or trigger workflows from Slack events'`: A short summary of what the block does, shown in tooltips or lists.
*   `authMode: AuthMode.OAuth`: Specifies the default or primary authentication mode for this block. Here, it's set to `OAuth`, meaning it typically uses an OAuth flow to connect to Slack.
*   `longDescription: '...'`: A more detailed explanation of the block's capabilities, potentially shown in a detailed view or documentation. It highlights sending messages, creating canvases, reading messages, advanced bot token usage, and trigger capabilities.
*   `docsLink: 'https://docs.sim.ai/tools/slack'`: A URL pointing to the external documentation for this Slack integration.
*   `category: 'tools'`: Categorizes this block, likely for organization in a UI sidebar or search.
*   `bgColor: '#611f69'`: A hexadecimal color code used for the block's background, often for visual branding.
*   `icon: SlackIcon`: References the `SlackIcon` component imported earlier, providing the visual icon for the block.
*   `triggerAllowed: true`: A boolean flag indicating that this block can be used as a trigger (i.e., it can initiate a workflow based on an external event).

---

##### **`subBlocks` (User Interface Input Fields Configuration)**

This array defines the interactive elements (input fields, dropdowns, selectors) that users will see and interact with when configuring this Slack block in the platform's UI.

```typescript
  subBlocks: [
    {
      id: 'operation',
      title: 'Operation',
      type: 'dropdown',
      layout: 'full',
      options: [
        { label: 'Send Message', id: 'send' },
        { label: 'Create Canvas', id: 'canvas' },
        { label: 'Read Messages', id: 'read' },
      ],
      value: () => 'send',
    },
```

*   **Operation Selector:**
    *   `id: 'operation'`: Unique identifier for this input field.
    *   `title: 'Operation'`: The label displayed to the user.
    *   `type: 'dropdown'`: Specifies that this is a dropdown menu.
    *   `layout: 'full'`: Indicates the field should take up the full available width.
    *   `options: [...]`: An array of objects defining the choices in the dropdown. Each choice has a `label` (what the user sees) and an `id` (the internal value when selected).
        *   `{ label: 'Send Message', id: 'send' }`: Option to send a message.
        *   `{ label: 'Create Canvas', id: 'canvas' }`: Option to create a canvas.
        *   `{ label: 'Read Messages', id: 'read' }`: Option to read messages.
    *   `value: () => 'send'`: A function that returns the default selected value for this dropdown, which is 'send'.

```typescript
    {
      id: 'authMethod',
      title: 'Authentication Method',
      type: 'dropdown',
      layout: 'full',
      options: [
        { label: 'Sim Bot', id: 'oauth' },
        { label: 'Custom Bot', id: 'bot_token' },
      ],
      value: () => 'oauth',
      required: true,
    },
```

*   **Authentication Method Selector:**
    *   `id: 'authMethod'`: Identifier for authentication method selection.
    *   `title: 'Authentication Method'`: Label for the field.
    *   `type: 'dropdown'`: It's a dropdown menu.
    *   `layout: 'full'`: Full width.
    *   `options: [...]`:
        *   `{ label: 'Sim Bot', id: 'oauth' }`: Option to use the platform's built-in (OAuth) authentication.
        *   `{ label: 'Custom Bot', id: 'bot_token' }`: Option for users to provide their own Slack bot token.
    *   `value: () => 'oauth'`: Default to 'Sim Bot' (OAuth).
    *   `required: true`: This field must be selected by the user.

```typescript
    {
      id: 'credential',
      title: 'Slack Account',
      type: 'oauth-input',
      layout: 'full',
      provider: 'slack',
      serviceId: 'slack',
      requiredScopes: [
        'channels:read', 'channels:history', 'groups:read', 'groups:history',
        'chat:write', 'chat:write.public', 'users:read', 'files:write', 'canvases:write',
      ],
      placeholder: 'Select Slack workspace',
      condition: {
        field: 'authMethod',
        value: 'oauth',
      },
    },
```

*   **Slack Account (OAuth) Selector:**
    *   `id: 'credential'`: Identifier for the OAuth credential input.
    *   `title: 'Slack Account'`: Label.
    *   `type: 'oauth-input'`: A specialized input type for handling OAuth credentials, likely displaying a list of connected accounts or prompting for a new connection.
    *   `layout: 'full'`: Full width.
    *   `provider: 'slack'`, `serviceId: 'slack'`: Specifies the external service this OAuth input connects to.
    *   `requiredScopes: [...]`: An array of Slack API scopes (permissions) that this block will require from the user's connected Slack workspace for its operations (e.g., reading channels, writing messages, creating canvases).
    *   `placeholder: 'Select Slack workspace'`: Hint text for the input field.
    *   `condition: { field: 'authMethod', value: 'oauth' }`: **This is a key piece of conditional logic.** This field will *only* be shown to the user if the `authMethod` dropdown (defined above) has a value of `'oauth'` (i.e., 'Sim Bot' is selected).

```typescript
    {
      id: 'botToken',
      title: 'Bot Token',
      type: 'short-input',
      layout: 'full',
      placeholder: 'Enter your Slack bot token (xoxb-...)',
      password: true,
      condition: {
        field: 'authMethod',
        value: 'bot_token',
      },
    },
```

*   **Bot Token Input:**
    *   `id: 'botToken'`: Identifier for the bot token input.
    *   `title: 'Bot Token'`: Label.
    *   `type: 'short-input'`: A standard single-line text input.
    *   `layout: 'full'`: Full width.
    *   `placeholder: 'Enter your Slack bot token (xoxb-...)'`: Hint text.
    *   `password: true`: This causes the input to mask characters (like a password field) for security.
    *   `condition: { field: 'authMethod', value: 'bot_token' }`: **Conditional logic.** This field is *only* shown if 'Custom Bot' is selected in the `authMethod` dropdown.

```typescript
    {
      id: 'channel',
      title: 'Channel',
      type: 'channel-selector',
      layout: 'full',
      canonicalParamId: 'channel',
      provider: 'slack',
      placeholder: 'Select Slack channel',
      mode: 'basic',
      dependsOn: ['credential', 'authMethod'],
    },
```

*   **Channel Selector (Basic Mode):**
    *   `id: 'channel'`: Identifier for the channel selection.
    *   `title: 'Channel'`: Label.
    *   `type: 'channel-selector'`: A specialized input type that allows users to search and select a Slack channel, typically pre-populated from the connected Slack workspace.
    *   `layout: 'full'`: Full width.
    *   `canonicalParamId: 'channel'`: This is important. It means this UI field (and potentially another, like `manualChannel`) ultimately maps to a single logical parameter named `channel` in the backend.
    *   `provider: 'slack'`: Specifies the service for channel lookup.
    *   `placeholder: 'Select Slack channel'`: Hint text.
    *   `mode: 'basic'`: Indicates this is part of the "basic" user interface, perhaps contrasted with an "advanced" mode.
    *   `dependsOn: ['credential', 'authMethod']`: This field will only become active or populate its options *after* a `credential` (Slack account) and an `authMethod` have been selected, as it needs authentication to fetch channels.

```typescript
    // Manual channel ID input (advanced mode)
    {
      id: 'manualChannel',
      title: 'Channel ID',
      type: 'short-input',
      layout: 'full',
      canonicalParamId: 'channel',
      placeholder: 'Enter Slack channel ID (e.g., C1234567890)',
      mode: 'advanced',
    },
```

*   **Manual Channel ID Input (Advanced Mode):**
    *   `id: 'manualChannel'`: Identifier for manual channel input.
    *   `title: 'Channel ID'`: Label.
    *   `type: 'short-input'`: Standard text input.
    *   `layout: 'full'`: Full width.
    *   `canonicalParamId: 'channel'`: Also maps to the same logical `channel` parameter as the `channel-selector`. The backend logic will decide which one to use if both are present.
    *   `placeholder: 'Enter Slack channel ID (e.g., C1234567890)'`: Hint for entering a raw Slack channel ID.
    *   `mode: 'advanced'`: This field is likely only visible when the user enables an "advanced" mode setting in the UI.

```typescript
    {
      id: 'text',
      title: 'Message',
      type: 'long-input',
      layout: 'full',
      placeholder: 'Enter your message (supports Slack mrkdwn)',
      condition: {
        field: 'operation',
        value: 'send',
      },
      required: true,
    },
```

*   **Message Text Input:**
    *   `id: 'text'`: Identifier for the message content.
    *   `title: 'Message'`: Label.
    *   `type: 'long-input'`: A multi-line text area, suitable for longer messages.
    *   `layout: 'full'`: Full width.
    *   `placeholder: 'Enter your message (supports Slack mrkdwn)'`: Hint, noting Slack's Markdown support.
    *   `condition: { field: 'operation', value: 'send' }`: **Conditional logic.** This field is *only* shown if the `operation` dropdown is set to 'Send Message'.
    *   `required: true`: This field must be filled when visible.

```typescript
    // Canvas specific fields
    {
      id: 'title',
      title: 'Canvas Title',
      type: 'short-input',
      layout: 'full',
      placeholder: 'Enter canvas title',
      condition: {
        field: 'operation',
        value: 'canvas',
      },
      required: true,
    },
    {
      id: 'content',
      title: 'Canvas Content',
      type: 'long-input',
      layout: 'full',
      placeholder: 'Enter canvas content (markdown supported)',
      condition: {
        field: 'operation',
        value: 'canvas',
      },
      required: true,
    },
```

*   **Canvas Specific Fields:**
    *   `id: 'title'`, `title: 'Canvas Title'`, `type: 'short-input'`: Input for the canvas title.
        *   `condition: { field: 'operation', value: 'canvas' }`: Only shown if `operation` is 'Create Canvas'.
        *   `required: true`: Mandatory.
    *   `id: 'content'`, `title: 'Canvas Content'`, `type: 'long-input'`: Input for the canvas content.
        *   `condition: { field: 'operation', value: 'canvas' }`: Only shown if `operation` is 'Create Canvas'.
        *   `required: true`: Mandatory.

```typescript
    // Message Reader specific fields
    {
      id: 'limit',
      title: 'Message Limit',
      type: 'short-input',
      layout: 'half',
      placeholder: '15',
      condition: {
        field: 'operation',
        value: 'read',
      },
    },
    {
      id: 'oldest',
      title: 'Oldest Timestamp',
      type: 'short-input',
      layout: 'half',
      placeholder: 'ISO 8601 timestamp',
      condition: {
        field: 'operation',
        value: 'read',
      },
    },
```

*   **Message Reader Specific Fields:**
    *   `id: 'limit'`, `title: 'Message Limit'`, `type: 'short-input'`: Input for the maximum number of messages to read.
        *   `layout: 'half'`: This field takes up half the width, allowing another `half` layout field to sit next to it.
        *   `condition: { field: 'operation', value: 'read' }`: Only shown if `operation` is 'Read Messages'.
    *   `id: 'oldest'`, `title: 'Oldest Timestamp'`, `type: 'short-input'`: Input for the earliest message timestamp to retrieve.
        *   `layout: 'half'`: Half width.
        *   `condition: { field: 'operation', value: 'read' }`: Only shown if `operation` is 'Read Messages'.

```typescript
    // TRIGGER MODE: Trigger configuration (only shown when trigger mode is active)
    {
      id: 'triggerConfig',
      title: 'Trigger Configuration',
      type: 'trigger-config',
      layout: 'full',
      triggerProvider: 'slack',
      availableTriggers: ['slack_webhook'],
    },
  ],
```

*   **Trigger Configuration Field:**
    *   `id: 'triggerConfig'`: Identifier for trigger settings.
    *   `title: 'Trigger Configuration'`: Label.
    *   `type: 'trigger-config'`: A specialized input type that configures how this block acts as a workflow trigger.
    *   `layout: 'full'`: Full width.
    *   `triggerProvider: 'slack'`: Specifies that Slack is the source of the trigger events.
    *   `availableTriggers: ['slack_webhook']`: Lists the specific types of Slack triggers this block can handle (in this case, a Slack webhook).

---

##### **`tools` (Backend Execution Logic)**

This object defines how the block interacts with underlying backend "tools" or APIs to perform its actions.

```typescript
  tools: {
    access: ['slack_message', 'slack_canvas', 'slack_message_reader'],
    config: {
      tool: (params) => {
        switch (params.operation) {
          case 'send':
            return 'slack_message'
          case 'canvas':
            return 'slack_canvas'
          case 'read':
            return 'slack_message_reader'
          default:
            throw new Error(`Invalid Slack operation: ${params.operation}`)
        }
      },
      params: (params) => {
        const {
          credential,
          authMethod,
          botToken,
          operation,
          channel,
          manualChannel,
          title,
          content,
          limit,
          oldest,
          ...rest
        } = params
```

*   `tools: { ... }`: Container for tool-related configurations.
    *   `access: ['slack_message', 'slack_canvas', 'slack_message_reader']`: A list of the actual backend "tool" identifiers that this block is authorized to use. These represent specific functions or APIs for sending messages, creating canvases, and reading messages.
    *   `config: { ... }`: Contains functions to dynamically configure which tool to use and with what parameters.
        *   `tool: (params) => { ... }`: This function is responsible for determining *which* backend Slack tool to invoke based on the user's `operation` choice.
            *   `switch (params.operation)`: It takes the `operation` parameter (selected by the user in the 'Operation' dropdown) and uses a switch statement.
            *   `case 'send': return 'slack_message'`: If 'send' is selected, it returns the ID of the Slack message sending tool.
            *   `case 'canvas': return 'slack_canvas'`: If 'canvas' is selected, it returns the ID of the Slack canvas creation tool.
            *   `case 'read': return 'slack_message_reader'`: If 'read' is selected, it returns the ID of the Slack message reading tool.
            *   `default: throw new Error(...)`: If an unknown operation is provided, it throws an error.
        *   `params: (params) => { ... }`: This is the most critical function. It takes all the raw inputs (`params`) provided by the user (from `subBlocks`) and transforms them into a structured object containing only the necessary and properly formatted parameters for the selected backend tool.
            *   `const { ... } = params`: Uses object destructuring to extract specific input values from the `params` object.
                *   `credential`, `authMethod`, `botToken`, `operation`, `channel`, `manualChannel`, `title`, `content`, `limit`, `oldest`: These correspond directly to the `id`s of the `subBlocks` defined earlier.
                *   `...rest`: Gathers any other parameters not explicitly destructured into a `rest` object. This is good practice to ensure unused parameters don't cause issues.

```typescript
        // Handle both selector and manual channel input
        const effectiveChannel = (channel || manualChannel || '').trim()

        if (!effectiveChannel) {
          throw new Error('Channel is required.')
        }

        const baseParams: Record<string, any> = {
          channel: effectiveChannel,
        }
```

*   **Channel Selection Logic:**
    *   `const effectiveChannel = (channel || manualChannel || '').trim()`: This line intelligently selects the channel. It tries to use the `channel` value (from the channel selector), if that's empty or undefined, it then tries `manualChannel` (from the manual input). If both are empty, it defaults to an empty string. `.trim()` removes leading/trailing whitespace.
    *   `if (!effectiveChannel) { ... }`: Checks if an effective channel was determined. If not, it throws an error, indicating that a channel is mandatory.
    *   `const baseParams: Record<string, any> = { channel: effectiveChannel, }`: Initializes a `baseParams` object. This object will be built up with all the necessary parameters for the chosen Slack operation. The `channel` is added first.

```typescript
        // Handle authentication based on method
        if (authMethod === 'bot_token') {
          if (!botToken) {
            throw new Error('Bot token is required when using bot token authentication')
          }
          baseParams.accessToken = botToken
        } else {
          // Default to OAuth
          if (!credential) {
            throw new Error('Slack account credential is required when using Sim Bot')
          }
          baseParams.credential = credential
        }
```

*   **Authentication Logic:**
    *   `if (authMethod === 'bot_token')`: Checks if the user selected 'Custom Bot' authentication.
        *   `if (!botToken) { ... }`: If `bot_token` is selected but no `botToken` value was provided, an error is thrown.
        *   `baseParams.accessToken = botToken`: If a bot token is provided, it's added to `baseParams` under the `accessToken` key.
    *   `else`: If `authMethod` is not `bot_token` (meaning it's 'oauth' by default).
        *   `if (!credential) { ... }`: If OAuth is chosen but no `credential` (Slack account) was selected, an error is thrown.
        *   `baseParams.credential = credential`: The OAuth `credential` is added to `baseParams`.

```typescript
        // Handle operation-specific params
        switch (operation) {
          case 'send':
            if (!rest.text) {
              throw new Error('Message text is required for send operation')
            }
            baseParams.text = rest.text
            break

          case 'canvas':
            if (!title || !content) {
              throw new Error('Title and content are required for canvas operation')
            }
            baseParams.title = title
            baseParams.content = content
            break

          case 'read':
            if (limit) {
              const parsedLimit = Number.parseInt(limit, 10)
              baseParams.limit = !Number.isNaN(parsedLimit) ? parsedLimit : 10
            } else {
              baseParams.limit = 10
            }
            if (oldest) {
              baseParams.oldest = oldest
            }
            break
        }

        return baseParams
      },
    },
  },
```

*   **Operation-Specific Parameter Handling:** This `switch` statement adds parameters required by each specific operation.
    *   `case 'send'`:
        *   `if (!rest.text) { ... }`: Checks if the message `text` is provided (from the `rest` object, as `text` was not explicitly destructured earlier). If not, throws an error.
        *   `baseParams.text = rest.text`: Adds the message text to `baseParams`.
    *   `case 'canvas'`:
        *   `if (!title || !content) { ... }`: Checks if both `title` and `content` for the canvas are provided. If not, throws an error.
        *   `baseParams.title = title`: Adds the canvas title.
        *   `baseParams.content = content`: Adds the canvas content.
    *   `case 'read'`:
        *   `if (limit) { ... }`: If a `limit` for messages is provided:
            *   `const parsedLimit = Number.parseInt(limit, 10)`: Tries to parse the `limit` (which comes as a string from the input field) into an integer.
            *   `baseParams.limit = !Number.isNaN(parsedLimit) ? parsedLimit : 10`: If `parsedLimit` is a valid number, use it; otherwise, default to `10`.
        *   `else { baseParams.limit = 10 }`: If no `limit` was provided, default to `10` messages.
        *   `if (oldest) { baseParams.oldest = oldest }`: If an `oldest` timestamp is provided, add it to `baseParams`.
    *   `return baseParams`: Finally, the fully constructed `baseParams` object is returned, ready to be used by the selected backend Slack tool.

---

##### **`inputs` (Workflow Input Schema)**

This object defines the formal schema of inputs this block expects from the *workflow system*. These often mirror the UI fields but define the *type* of data the block internally processes, not just how it's presented to the user.

```typescript
  inputs: {
    operation: { type: 'string', description: 'Operation to perform' },
    authMethod: { type: 'string', description: 'Authentication method' },
    credential: { type: 'string', description: 'Slack access token' },
    botToken: { type: 'string', description: 'Bot token' },
    channel: { type: 'string', description: 'Channel identifier' },
    manualChannel: { type: 'string', description: 'Manual channel identifier' },
    text: { type: 'string', description: 'Message text' },
    title: { type: 'string', description: 'Canvas title' },
    content: { type: 'string', description: 'Canvas content' },
    limit: { type: 'string', description: 'Message limit' },
    oldest: { type: 'string', description: 'Oldest timestamp' },
  },
```

*   For each key (e.g., `operation`, `credential`):
    *   `type: 'string'`: Defines the expected data type of the input. Most are strings here because they come from text input fields or dropdowns.
    *   `description: '...'`: A description of what the input represents.

---

##### **`outputs` (Workflow Output Schema)**

This object defines the formal schema of data that this block will *produce* and make available to subsequent blocks in a workflow. The output fields vary depending on the operation performed.

```typescript
  outputs: {
    // slack_message outputs
    ts: { type: 'string', description: 'Message timestamp returned by Slack API' },
    channel: { type: 'string', description: 'Channel identifier where message was sent' },

    // slack_canvas outputs
    canvas_id: { type: 'string', description: 'Canvas identifier for created canvases' },
    title: { type: 'string', description: 'Canvas title' },

    // slack_message_reader outputs
    messages: {
      type: 'json',
      description: 'Array of message objects',
    },

    // Trigger outputs (when used as webhook trigger)
    event_type: { type: 'string', description: 'Type of Slack event that triggered the workflow' },
    channel_name: { type: 'string', description: 'Human-readable channel name' },
    user_name: { type: 'string', description: 'Username who triggered the event' },
    team_id: { type: 'string', description: 'Slack workspace/team ID' },
    event_id: { type: 'string', description: 'Unique event identifier for the trigger' },
  },
```

*   **Outputs for `slack_message` (Send Message):**
    *   `ts: { type: 'string', description: '...' }`: The timestamp of the sent message.
    *   `channel: { type: 'string', description: '...' }`: The ID of the channel where the message was sent.
*   **Outputs for `slack_canvas` (Create Canvas):**
    *   `canvas_id: { type: 'string', description: '...' }`: The ID of the created canvas.
    *   `title: { type: 'string', description: '...' }`: The title of the created canvas.
*   **Outputs for `slack_message_reader` (Read Messages):**
    *   `messages: { type: 'json', description: '...' }`: An array of message objects (Slack messages can be complex, so `json` indicates a structured data output).
*   **Outputs for Trigger Mode:** These outputs are available when the block is configured to *start* a workflow from a Slack event.
    *   `event_type`, `channel_name`, `user_name`, `team_id`, `event_id`: Various details about the Slack event that triggered the workflow.

---

##### **`triggers` (Trigger Capabilities)**

This object explicitly declares that this block can act as a trigger.

```typescript
  // New: Trigger capabilities
  triggers: {
    enabled: true,
    available: ['slack_webhook'],
  },
}
```

*   `triggers: { ... }`: Container for trigger-specific configurations.
    *   `enabled: true`: A boolean flag indicating that this block is indeed capable of functioning as a trigger.
    *   `available: ['slack_webhook']`: A list of the specific types of triggers that this block supports (in this case, triggering a workflow when a Slack webhook event is received).

---

This `SlackBlock` configuration provides a powerful and flexible way to integrate Slack functionality into a workflow platform, handling UI presentation, dynamic input validation, authentication, and backend API orchestration all within a single, well-structured definition.