This TypeScript file is a configuration blueprint for a "Telegram" integration, likely used within a larger application that allows users to build workflows or automate tasks. It defines everything needed to represent and interact with Telegram services: its appearance in a user interface, the input fields it requires, the operations it can perform, and the data it can output.

Think of it as a detailed instruction manual for a "Telegram Block" component in a drag-and-drop workflow builder.

---

### **Purpose of this File**

The primary purpose of `telegram.ts` is to export a `BlockConfig` object named `TelegramBlock`. This object acts as a comprehensive schema that describes how a "Telegram" feature or tool should function within a system. Specifically, it defines:

1.  **Metadata:** Basic information like its name, description, category, and an icon.
2.  **User Interface (UI) Configuration:** What input fields a user needs to see and interact with to configure a Telegram action (e.g., "Send Message," "Delete Message"). This includes dropdowns, text inputs, placeholders, help text, and conditional visibility for fields.
3.  **Authentication:** How the block authenticates with Telegram (using a Bot Token).
4.  **Operational Logic:** How user inputs from the UI are translated into specific Telegram API calls and their required parameters. It includes validation checks to ensure necessary inputs are provided.
5.  **Workflow Integration:** What inputs the block expects when dynamically triggered in a workflow and what outputs it will produce upon completion.
6.  **Trigger Capabilities:** Whether this block can *start* a workflow (e.g., when a new message is received in Telegram).

In essence, this file empowers the application to render a sophisticated Telegram integration without hardcoding its details, making it extensible and configurable.

---

### **Detailed Explanation**

Let's break down the code line by line, simplifying complex logic as we go.

```typescript
import { TelegramIcon } from '@/components/icons'
import type { BlockConfig } from '@/blocks/types'
import { AuthMode } from '@/blocks/types'
import type { TelegramResponse } from '@/tools/telegram/types'
```
These lines import necessary components and types:
*   `import { TelegramIcon } from '@/components/icons'`: Imports a visual icon specifically for Telegram, likely used in the UI to represent this block.
*   `import type { BlockConfig } from '@/blocks/types'`: Imports the TypeScript type definition for `BlockConfig`. This is crucial as it tells TypeScript (and us) the expected structure and properties for any block configuration. The `type` keyword means it's only used for type checking and doesn't generate JavaScript code.
*   `import { AuthMode } from '@/blocks/types'`: Imports an enum (a set of named constants) called `AuthMode`. This enum defines different ways a block can authenticate.
*   `import type { TelegramResponse } from '@/tools/telegram/types'`: Imports the TypeScript type definition for `TelegramResponse`. This type describes the data structure expected as a result when a Telegram operation completes.

```typescript
export const TelegramBlock: BlockConfig<TelegramResponse> = {
```
This line declares and exports a constant variable named `TelegramBlock`.
*   `export`: Makes this variable available for other files to import and use.
*   `const TelegramBlock`: Defines the constant variable.
*   `: BlockConfig<TelegramResponse>`: This is a type annotation. It specifies that `TelegramBlock` *must* conform to the `BlockConfig` interface. The `<TelegramResponse>` part is a generic type argument, indicating that this specific `BlockConfig` will produce outputs that match the `TelegramResponse` type.

```typescript
  type: 'telegram',
  name: 'Telegram',
  description: 'Interact with Telegram',
  authMode: AuthMode.BotToken,
```
These properties provide basic identifying information for the block:
*   `type: 'telegram'`: A unique string identifier for this block type within the system.
*   `name: 'Telegram'`: The human-readable name displayed in the UI.
*   `description: 'Interact with Telegram'`: A short description shown to users.
*   `authMode: AuthMode.BotToken`: Specifies the authentication method for this block. `AuthMode.BotToken` indicates that a Telegram Bot Token is required for operations.

```typescript
  longDescription:
    'Integrate Telegram into the workflow. Can send and delete messages. Can be used in trigger mode to trigger a workflow when a message is sent to a chat.',
  docsLink: 'https://docs.sim.ai/tools/telegram',
  category: 'tools',
  bgColor: '#E0E0E0',
  icon: TelegramIcon,
  triggerAllowed: true,
```
More metadata and configuration:
*   `longDescription`: A more detailed explanation of what the Telegram integration can do, displayed in the UI (e.g., in a sidebar or pop-up). It highlights the send/delete message capabilities and its role as a workflow trigger.
*   `docsLink`: A URL pointing to the official documentation for this Telegram integration.
*   `category: 'tools'`: Classifies this block into a specific category within the application, helping users find it.
*   `bgColor: '#E0E0E0'`: A hexadecimal color code for the block's background, used for visual styling in the UI.
*   `icon: TelegramIcon`: Assigns the previously imported `TelegramIcon` component as the visual icon for this block.
*   `triggerAllowed: true`: A boolean flag indicating that this block *can* be used to initiate a workflow, meaning it can react to external events (like incoming Telegram messages).

```typescript
  subBlocks: [
```
This property is an array of objects, where each object defines a specific input field or configuration element that will be presented to the user in the UI to configure this Telegram block. These are the building blocks of the configuration form.

```typescript
    {
      id: 'operation',
      title: 'Operation',
      type: 'dropdown',
      layout: 'full',
      options: [
        { label: 'Send Message', id: 'telegram_message' },
        { label: 'Send Photo', id: 'telegram_send_photo' },
        { label: 'Send Video', id: 'telegram_send_video' },
        { label: 'Send Audio', id: 'telegram_send_audio' },
        { label: 'Send Animation', id: 'telegram_send_animation' },
        { label: 'Delete Message', id: 'telegram_delete_message' },
      ],
      value: () => 'telegram_message',
    },
```
This is the first `subBlock`, defining a dropdown menu for selecting the type of Telegram action:
*   `id: 'operation'`: A unique identifier for this input field, used internally to reference its value.
*   `title: 'Operation'`: The label displayed above the dropdown in the UI.
*   `type: 'dropdown'`: Specifies that this UI element is a dropdown selection.
*   `layout: 'full'`: Dictates that this field should occupy the full width of its container.
*   `options`: An array of objects, where each object represents an option in the dropdown.
    *   `label`: The human-readable text displayed in the dropdown.
    *   `id`: The internal value associated with that option when selected. These IDs (`telegram_message`, `telegram_send_photo`, etc.) are critical as they are used later to determine which specific Telegram API call to make.
*   `value: () => 'telegram_message'`: A function that returns the default selected value for this dropdown when the block is first added. Here, "Send Message" is the default.

```typescript
    {
      id: 'botToken',
      title: 'Bot Token',
      type: 'short-input',
      layout: 'full',
      placeholder: 'Enter your Telegram Bot Token',
      password: true,
      connectionDroppable: false,
      description: `Getting Bot Token:
1. If you haven't already, message "/newbot" to @BotFather
2. Choose a name for your bot
3. Copy the token it provides and paste it here`,
      required: true,
    },
```
This `subBlock` defines an input field for the Telegram Bot Token:
*   `id: 'botToken'`: Unique identifier.
*   `title: 'Bot Token'`: Label for the field.
*   `type: 'short-input'`: A single-line text input field.
*   `layout: 'full'`: Occupies full width.
*   `placeholder: 'Enter your Telegram Bot Token'`: Text displayed in the input field when it's empty.
*   `password: true`: This is a security feature. It tells the UI to treat this input as sensitive, likely obscuring the text (like asterisks) and preventing it from being accidentally logged or displayed.
*   `connectionDroppable: false`: Indicates that a "connection" (e.g., from a credentials manager) cannot be directly dropped into this field.
*   `description`: Provides detailed instructions on how to obtain a Telegram Bot Token using `@BotFather`. This is helpful user guidance.
*   `required: true`: Specifies that this field must be filled out by the user.

```typescript
    {
      id: 'chatId',
      title: 'Chat ID',
      type: 'short-input',
      layout: 'full',
      placeholder: 'Enter Telegram Chat ID',
      description: `Getting Chat ID:
1. Add your bot as a member to desired Telegram channel
2. Send any message to the channel (e.g. "I love Sim")
3. Visit https://api.telegram.org/bot<YOUR_BOT_TOKEN>/getUpdates
4. Look for the chat field in the JSON response at the very bottom where you'll find the chat ID`,
      required: true,
    },
```
This `subBlock` defines an input field for the Telegram Chat ID:
*   `id: 'chatId'`: Unique identifier.
*   `title: 'Chat ID'`: Label for the field.
*   `type: 'short-input'`: Single-line text input.
*   `layout: 'full'`: Full width.
*   `placeholder: 'Enter Telegram Chat ID'`: Placeholder text.
*   `description`: Provides detailed, step-by-step instructions on how to find the `chatId` for a Telegram chat/channel, which often involves using the Telegram API's `getUpdates` endpoint.
*   `required: true`: This field must be provided.

The following `subBlocks` (`text`, `photo`, `video`, `audio`, `animation`, `caption`, `messageId`) all follow a similar structure. Their key differentiating feature is the `condition` property.

**Simplifying Complex Logic: Conditional Fields (`condition` property)**

The `condition` property is crucial for making the UI user-friendly. Instead of showing all possible input fields at once (which would be overwhelming), `condition` dictates when a particular field should be visible. It works by checking the value of another field, specifically the `operation` dropdown we defined earlier.

Example: The `text` input field for messages will *only* appear if the `operation` dropdown is set to `'telegram_message'`. If `operation` is set to `'telegram_send_photo'`, the `text` field will be hidden, and the `photo` field will appear instead.

Let's look at the `text` field:
```typescript
    {
      id: 'text',
      title: 'Message',
      type: 'long-input',
      layout: 'full',
      placeholder: 'Enter the message to send',
      required: true,
      condition: { field: 'operation', value: 'telegram_message' },
    },
```
*   `id: 'text'`, `title: 'Message'`, `type: 'long-input'` (a multi-line text area), `layout: 'full'`, `placeholder`, `required: true`: Standard input field properties.
*   `condition: { field: 'operation', value: 'telegram_message' }`: **This is the critical part.** This object states: "This input field (`text`) should only be visible when the `operation` field has a selected `value` equal to `'telegram_message'`."

The same logic applies to `photo`, `video`, `audio`, `animation`, and `messageId`, each appearing only for their respective operations.

```typescript
    {
      id: 'photo',
      title: 'Photo',
      type: 'short-input',
      layout: 'full',
      placeholder: 'Enter photo URL or file_id',
      description: 'Photo to send. Pass a file_id or HTTP URL',
      required: true,
      condition: { field: 'operation', value: 'telegram_send_photo' },
    },
    {
      id: 'video',
      title: 'Video',
      type: 'short-input',
      layout: 'full',
      placeholder: 'Enter video URL or file_id',
      description: 'Video to send. Pass a file_id or HTTP URL',
      required: true,
      condition: { field: 'operation', value: 'telegram_send_video' },
    },
    {
      id: 'audio',
      title: 'Audio',
      type: 'short-input',
      layout: 'full',
      placeholder: 'Enter audio URL or file_id',
      description: 'Audio file to send. Pass a file_id or HTTP URL',
      required: true,
      condition: { field: 'operation', value: 'telegram_send_audio' },
    },
    {
      id: 'animation',
      title: 'Animation',
      type: 'short-input',
      layout: 'full',
      placeholder: 'Enter animation URL or file_id',
      description: 'Animation (GIF) to send. Pass a file_id or HTTP URL',
      required: true,
      condition: { field: 'operation', value: 'telegram_send_animation' },
    },
```
These are inputs for various media types, each conditionally displayed based on the selected `operation`. They require a URL or a `file_id` (a Telegram-specific identifier for uploaded files).

```typescript
    {
      id: 'caption',
      title: 'Caption',
      type: 'long-input',
      layout: 'full',
      placeholder: 'Enter optional caption',
      description: 'Media caption (optional)',
      condition: {
        field: 'operation',
        value: [
          'telegram_send_photo',
          'telegram_send_video',
          'telegram_send_audio',
          'telegram_send_animation',
        ],
      },
    },
```
This `subBlock` defines an optional `caption` field for media.
*   `condition: { field: 'operation', value: [...] }`: This condition is slightly different. The `value` is an array, meaning this `caption` field will be visible if the `operation` is *any one* of the listed media sending operations (photo, video, audio, animation).

```typescript
    {
      id: 'messageId',
      title: 'Message ID',
      type: 'short-input',
      layout: 'full',
      placeholder: 'Enter the message ID to delete',
      description: 'The unique identifier of the message you want to delete',
      required: true,
      condition: { field: 'operation', value: 'telegram_delete_message' },
    },
```
This `subBlock` is for entering the ID of a message to be deleted, conditionally shown only when the "Delete Message" operation is selected.

```typescript
    // TRIGGER MODE: Trigger configuration (only shown when trigger mode is active)
    {
      id: 'triggerConfig',
      title: 'Trigger Configuration',
      type: 'trigger-config',
      layout: 'full',
      triggerProvider: 'telegram',
      availableTriggers: ['telegram_webhook'],
    },
  ],
```
This special `subBlock` defines how the block can act as a workflow trigger:
*   `id: 'triggerConfig'`, `title: 'Trigger Configuration'`: Standard identifiers.
*   `type: 'trigger-config'`: A special UI type for configuring triggers. This implies a more complex UI component is rendered here that might involve setting up webhooks or polling intervals.
*   `triggerProvider: 'telegram'`: Specifies that the trigger mechanism is provided by the Telegram service.
*   `availableTriggers: ['telegram_webhook']`: Lists the specific types of triggers this block supports. In this case, it's a `telegram_webhook`, meaning Telegram will send a notification (via webhook) to the application when an event occurs (e.g., a new message).

```typescript
  tools: {
```
The `tools` object defines how this block interacts with the underlying "tooling" or API layer of the application. It acts as an intermediary, translating the user's configuration into a format the Telegram API (or the application's wrapper around it) can understand.

```typescript
    access: [
      'telegram_message',
      'telegram_delete_message',
      'telegram_send_photo',
      'telegram_send_video',
      'telegram_send_audio',
      'telegram_send_animation',
    ],
```
*   `access`: An array listing all the specific Telegram API actions (or "tools") that this block is authorized to call. This defines the scope of capabilities for the integration.

```typescript
    config: {
```
The `config` object within `tools` contains functions that dynamically determine which tool to use and what parameters to pass based on the user's `subBlock` inputs.

```typescript
      tool: (params) => {
        switch (params.operation) {
          case 'telegram_message':
            return 'telegram_message'
          case 'telegram_delete_message':
            return 'telegram_delete_message'
          case 'telegram_send_photo':
            return 'telegram_send_photo'
          case 'telegram_send_video':
            return 'telegram_send_video'
          case 'telegram_send_audio':
            return 'telegram_send_audio'
          case 'telegram_send_animation':
            return 'telegram_send_animation'
          default:
            return 'telegram_message'
        }
      },
```
*   `tool: (params) => { ... }`: This function takes the *raw user inputs* (`params`, which would contain `operation`, `botToken`, `chatId`, etc., from the `subBlocks`) and returns the specific *internal tool identifier* that corresponds to the selected operation.
    *   It uses a `switch` statement on `params.operation` (the value selected in the "Operation" dropdown).
    *   For each case (e.g., `'telegram_message'`), it returns a matching string, which is the exact tool name to be invoked by the underlying system.
    *   `default: return 'telegram_message'`: If for some reason `params.operation` doesn't match any known case, it defaults to 'telegram_message' to provide a fallback.

```typescript
      params: (params) => {
        if (!params.botToken) throw new Error('Bot token required for this operation')

        const chatId = (params.chatId || '').trim()
        if (!chatId) {
          throw new Error('Chat ID is required.')
        }

        const commonParams = {
          botToken: params.botToken,
          chatId,
        }
```
*   `params: (params) => { ... }`: This is a critical function that takes the raw user inputs (`params`) and transforms them into the *exact payload* required by the specific Telegram API call. It also performs crucial input validation.
    *   `if (!params.botToken) throw new Error(...)`: Performs a basic validation check. If `botToken` is missing, it throws an error, preventing the operation from proceeding.
    *   `const chatId = (params.chatId || '').trim()`: Retrieves the `chatId` from the inputs, handling cases where it might be `null` or `undefined` by defaulting to an empty string, and then removes leading/trailing whitespace using `.trim()`.
    *   `if (!chatId) { throw new Error(...) }`: Validates that the `chatId` is not empty after trimming.
    *   `const commonParams = { botToken: params.botToken, chatId, }`: Creates an object `commonParams` containing parameters (botToken, chatId) that are common to most Telegram operations, reducing code repetition.

```typescript
        switch (params.operation) {
          case 'telegram_message':
            if (!params.text) {
              throw new Error('Message text is required.')
            }
            return {
              ...commonParams,
              text: params.text,
            }
          case 'telegram_delete_message':
            if (!params.messageId) {
              throw new Error('Message ID is required for delete operation.')
            }
            return {
              ...commonParams,
              messageId: params.messageId,
            }
          // ... (similar cases for photo, video, audio, animation with their respective validations)
          default:
            return {
              ...commonParams,
              text: params.text,
            }
        }
      },
    },
  },
```
This `switch` statement dynamically constructs the parameters object based on the selected `operation`:
*   For each `case` (e.g., `'telegram_message'`):
    *   It performs specific validation for that operation (e.g., `if (!params.text) ...` for sending a message).
    *   It then returns an object containing `commonParams` (bot token, chat ID) spread (`...commonParams`) and the operation-specific parameters (e.g., `text: params.text`). This structured object is the actual payload sent to the Telegram API via the underlying tooling layer.
*   The `default` case acts as a fallback, returning common parameters along with any available `text`.

```typescript
  inputs: {
    operation: { type: 'string', description: 'Operation to perform' },
    botToken: { type: 'string', description: 'Telegram bot token' },
    chatId: { type: 'string', description: 'Chat identifier' },
    text: { type: 'string', description: 'Message text' },
    photo: { type: 'string', description: 'Photo URL or file_id' },
    video: { type: 'string', description: 'Video URL or file_id' },
    audio: { type: 'string', description: 'Audio URL or file_id' },
    animation: { type: 'string', description: 'Animation URL or file_id' },
    caption: { type: 'string', description: 'Caption for media' },
    messageId: { type: 'string', description: 'Message ID to delete' },
  },
```
The `inputs` object defines the data structure that this `TelegramBlock` expects *if it were to receive inputs from a preceding block in a workflow*.
*   Each property (e.g., `operation`, `botToken`) specifies an input key.
*   `type: 'string'` (or other types like `'number'`, `'boolean'`, `'json'`) defines the expected data type.
*   `description`: Provides a brief explanation of what that input represents.
This is distinct from `subBlocks` (which configure the UI) because `inputs` describe the *data contract* for the block in a programmatic workflow context.

```typescript
  outputs: {
    // Send message operation outputs
    ok: { type: 'boolean', description: 'API response success status' },
    result: {
      type: 'json',
      description: 'Complete message result object from Telegram API',
    },
    message: { type: 'string', description: 'Success or error message' },
    data: { type: 'json', description: 'Response data' },
    // Specific result fields
    messageId: { type: 'number', description: 'Sent message ID' },
    chatId: { type: 'number', description: 'Chat ID where message was sent' },
    chatType: {
      type: 'string',
      description: 'Type of chat (private, group, supergroup, channel)',
    },
    username: { type: 'string', description: 'Chat username (if available)' },
    messageDate: {
      type: 'number',
      description: 'Unix timestamp of sent message',
    },
    messageText: {
      type: 'string',
      description: 'Text content of sent message',
    },
    // Delete message outputs
    deleted: {
      type: 'boolean',
      description: 'Whether the message was successfully deleted',
    },
    // Webhook trigger outputs (incoming messages)
    update_id: {
      type: 'number',
      description: 'Unique identifier for the update',
    },
    // ... (many more fields for webhook/incoming message data)
    entities: {
      type: 'json',
      description: 'Special entities in the message (mentions, hashtags, etc.)',
    },
  },
```
The `outputs` object defines the data structure that this `TelegramBlock` will produce *after it has successfully executed*. This data can then be used by subsequent blocks in a workflow.
*   Each property (e.g., `ok`, `result`, `messageId`) represents a potential output field.
*   `type`: Specifies the data type (boolean, string, number, json).
*   `description`: Explains what the output field represents.
The outputs are categorized into:
    *   **General API Response:** Fields like `ok`, `result`, `message`, `data` that might be common to many API calls.
    *   **Send Message Specific:** Fields like `messageId`, `chatId`, `messageText` that are relevant when a message is successfully sent.
    *   **Delete Message Specific:** A `deleted` boolean indicating success of a delete operation.
    *   **Webhook Trigger Outputs:** A comprehensive list of fields (`update_id`, `message_id`, `from_id`, `text`, `date`, `entities`, etc.) that would be available when this block is configured as a trigger and receives an incoming Telegram message via a webhook. These fields provide detailed information about the received message and its sender/chat.

```typescript
  triggers: {
    enabled: true,
    available: ['telegram_webhook'],
  },
}
```
The `triggers` object specifies the capabilities of this block when it acts as a workflow trigger:
*   `enabled: true`: Confirms that this block can indeed function as a trigger. This corresponds to the `triggerAllowed: true` metadata field.
*   `available: ['telegram_webhook']`: Lists the specific types of triggers that this block supports. `telegram_webhook` confirms it can listen for incoming events from Telegram by registering a webhook.
*   `}`: Closes the `TelegramBlock` object.

---

In summary, this `telegram.ts` file provides a complete, self-contained definition for integrating Telegram actions into a workflow system. It handles everything from UI presentation and user guidance to input validation, API parameter mapping, and defining the expected inputs and outputs for seamless workflow automation.