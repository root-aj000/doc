This TypeScript file is a comprehensive configuration for an "Agent" block within a workflow automation or AI orchestration platform. Think of it as defining a modular, configurable component that can leverage Large Language Models (LLMs) to perform tasks, similar to an AI assistant or a sophisticated chatbot.

## Purpose of this File

The primary purpose of this file is to define the **`AgentBlock`**, which is a core workflow block in a system that allows users to build and orchestrate AI applications. Specifically, this file:

1.  **Defines the Agent's Capabilities:** It specifies that the Agent is essentially a wrapper around an LLM, capable of taking prompts, calling LLM providers, making tool calls, and returning structured output.
2.  **Configures the User Interface (UI):** It dictates what input fields (e.g., text areas for prompts, dropdowns for models, sliders for temperature, code editors for JSON schema) the user will see when configuring an `AgentBlock` in the platform's UI.
3.  **Manages LLM Provider Integration:** It lists which LLM providers the `AgentBlock` can use internally (`openai_chat`, `anthropic_chat`, etc.) and how it dynamically selects and configures the chosen model (e.g., `gpt-4o`).
4.  **Handles Tool Management:** It describes how external tools (like an API call, web search, etc.) can be passed *into* the Agent, and how the Agent transforms and filters these tools before presenting them to the underlying LLM.
5.  **Specifies Inputs and Outputs:** It formally defines the expected data types and descriptions for what goes *into* the `AgentBlock` and what comes *out* of it.
6.  **Enables AI-Assisted Configuration:** It includes "wand configurations" for certain fields, implying AI-powered assistance for generating prompts or JSON schemas.

In essence, this file is the blueprint for how an "Agent" behaves, looks, and integrates into a larger AI workflow system.

---

## Simplified Complex Logic

Let's break down the more intricate parts:

1.  **Dynamic UI Configuration (`subBlocks` with `options` and `condition`):**
    *   Imagine you're building a form. Some fields only appear based on what you select in another field. For example, if you pick "Azure OpenAI" as your model, you'll see fields for "Azure Endpoint" and "Azure API Version." This file uses `condition` properties to achieve this dynamic behavior, making the UI smarter and less cluttered.
    *   Similarly, the list of available models (`options`) for the "Model" dropdown is gathered dynamically from different sources (like local Ollama models, OpenRouter, and base models provided by the system).
    *   There are *two* "Temperature" sliders, but only one is ever shown. The system checks the selected model's maximum temperature (some models go up to 1.0, others to 2.0) and displays the appropriate slider.
    *   The "API Key" field is hidden if you're using a model that's already hosted by the platform or an Ollama model (which is usually self-hosted and doesn't need an external API key in the same way).

2.  **AI-Assisted Content Generation (`wandConfig`):**
    *   For fields like "System Prompt" and "Response Format," there's a special `wandConfig`. This tells the UI to offer an AI assistant (a "wand" feature) to help the user generate content for that field.
    *   It provides a very detailed "system prompt" *to the AI assistant itself*, instructing it on how to generate good system prompts for the user's agent, or how to generate valid JSON schemas. This is a "meta-prompting" concept.

3.  **Agent's Internal Tooling (`tools` object):**
    *   The `AgentBlock` itself has access to a list of underlying LLM providers (e.g., `openai_chat`, `anthropic_chat`). This is defined in `tools.access`.
    *   More complex is `tools.config`. This section defines how the `AgentBlock` processes the `model` and any *external* `tools` that a user might provide *to the agent*.
        *   `tools.config.tool`: This function selects the *actual* LLM provider object from the system based on the `model` chosen by the user in the UI.
        *   `tools.config.params`: This crucial function takes the raw parameters (including a list of external tools) from the user's configuration and transforms them.
            *   It iterates through the provided `tools` array.
            *   It *filters out* any tools that the user has explicitly set to `usageControl: 'none'`, meaning the LLM should not use them.
            *   For the remaining tools, it re-structures their information (like ID, name, description, parameters) into a format suitable for the underlying LLM provider. This ensures the LLM understands how to call the external tools correctly.

4.  **`getToolIdFromBlock` Helper:**
    *   This helper function is used when an external tool is actually another "block" in the system. It dynamically loads the block registry and finds the associated tool ID, preventing the need to hardcode tool IDs everywhere. This allows for modularity and extensibility.

---

## Line-by-Line Explanation

```typescript
import { AgentIcon } from '@/components/icons'
```
*   **`import { AgentIcon } from '@/components/icons'`**: Imports a React component named `AgentIcon` from a shared icons directory. This icon will visually represent the `AgentBlock` in the UI.

```typescript
import { isHosted } from '@/lib/environment'
```
*   **`import { isHosted } from '@/lib/environment'`**: Imports a boolean variable `isHosted` from an environment configuration library. This variable likely indicates whether the application is running in a cloud-hosted environment managed by the platform, or if it's a self-hosted instance. It's used to dynamically adjust UI fields, specifically for API key inputs.

```typescript
import { createLogger } from '@/lib/logs/console/logger'
```
*   **`import { createLogger } from '@/lib/logs/console/logger'`**: Imports a utility function `createLogger` from a logging library. This allows for structured logging of events and errors within this file.

```typescript
import type { BlockConfig } from '@/blocks/types'
```
*   **`import type { BlockConfig } from '@/blocks/types'`**: Imports the `BlockConfig` TypeScript *type*. This is a generic type that defines the overall structure for any configurable block within the system. It uses `type` for a type-only import, indicating it's not runtime code.

```typescript
import { AuthMode } from '@/blocks/types'
```
*   **`import { AuthMode } from '@/blocks/types'`**: Imports the `AuthMode` enum from the same types file. This enum specifies different authentication mechanisms for blocks (e.g., `ApiKey`, `OAuth`, `None`).

```typescript
import {
  getAllModelProviders,
  getBaseModelProviders,
  getHostedModels,
  getMaxTemperature,
  getProviderIcon,
  MODELS_WITH_REASONING_EFFORT,
  MODELS_WITH_VERBOSITY,
  providers,
  supportsTemperature,
} from '@/providers/utils'
```
*   **`import {...} from '@/providers/utils'`**: This is a multi-import from a utility file related to LLM providers.
    *   `getAllModelProviders`: A function that returns a map or list of all supported LLM providers.
    *   `getBaseModelProviders`: A function that returns a map or list of the core, foundational LLM providers.
    *   `getHostedModels`: A function that returns a list of models that are managed directly by the platform (i.e., don't require the user to provide their own API key).
    *   `getMaxTemperature`: A function that takes a model name and returns its maximum supported temperature value (e.g., 1.0 or 2.0).
    *   `getProviderIcon`: A function that takes a model name and returns an icon associated with its provider.
    *   `MODELS_WITH_REASONING_EFFORT`: A constant (likely an array of strings) listing model IDs that support a "reasoning effort" configuration.
    *   `MODELS_WITH_VERBOSITY`: A constant (likely an array of strings) listing model IDs that support a "verbosity" configuration.
    *   `providers`: A constant (likely an object) containing configuration details for various LLM providers, including their supported models.
    *   `supportsTemperature`: A function that takes a model name and returns `true` if that model supports a `temperature` parameter.

```typescript
const getCurrentOllamaModels = () => {
  return useProvidersStore.getState().providers.ollama.models
}
```
*   **`const getCurrentOllamaModels = () => { ... }`**: Defines a helper function that retrieves the currently available Ollama models from a global state store.
    *   `useProvidersStore.getState()`: Accesses the current state of a Zustand (or similar) store named `useProvidersStore`.
    *   `.providers.ollama.models`: Navigates through the state object to extract the array of Ollama models.

```typescript
import { useProvidersStore } from '@/stores/providers/store'
```
*   **`import { useProvidersStore } from '@/stores/providers/store'`**: Imports the `useProvidersStore` hook/object, which is likely a global state management store (e.g., powered by Zustand) that holds information about available LLM providers and their models.

```typescript
import type { ToolResponse } from '@/tools/types'
```
*   **`import type { ToolResponse } from '@/tools/types'`**: Imports the `ToolResponse` TypeScript *type*. This type defines the expected structure of a response when a tool is called.

```typescript
const logger = createLogger('AgentBlock')
```
*   **`const logger = createLogger('AgentBlock')`**: Initializes a logger instance specifically for the `AgentBlock`, making it easier to identify logs originating from this component.

```typescript
interface AgentResponse extends ToolResponse {
  output: {
    content: string
    model: string
    tokens?: {
      prompt?: number
      completion?: number
      total?: number
    }
    toolCalls?: {
      list: Array<{
        name: string
        arguments: Record<string, any>
      }>
      count: number
    }
  }
}
```
*   **`interface AgentResponse extends ToolResponse { ... }`**: Defines the `AgentResponse` interface, which describes the *specific shape of the data that the `AgentBlock` will output*.
    *   `extends ToolResponse`: It inherits all properties from the `ToolResponse` interface.
    *   `output`: This nested object contains the primary output data from the agent.
        *   `content: string`: The main textual response generated by the LLM.
        *   `model: string`: The name of the LLM model that was used to generate the response.
        *   `tokens?: { ... }`: An optional object containing token usage statistics (prompt tokens, completion tokens, total tokens). The `?` denotes optional properties.
        *   `toolCalls?: { ... }`: An optional object detailing any tool calls made by the agent.
            *   `list`: An array of tool call objects, each with a `name` and `arguments` (a record of key-value pairs).
            *   `count`: The total number of tool calls made.

```typescript
// Helper function to get the tool ID from a block type
const getToolIdFromBlock = (blockType: string): string | undefined => {
  try {
    const { getAllBlocks } = require('@/blocks/registry')
    const blocks = getAllBlocks()
    const block = blocks.find(
      (b: { type: string; tools?: { access?: string[] } }) => b.type === blockType
    )
    return block?.tools?.access?.[0]
  } catch (error) {
    logger.error('Error getting tool ID from block', { error })
    return undefined
  }
}
```
*   **`const getToolIdFromBlock = (blockType: string): string | undefined => { ... }`**: This is a helper function designed to extract the unique identifier (ID) of a tool from a given `blockType`. This is useful when one block needs to refer to another block as a "tool."
    *   `try { ... } catch (error) { ... }`: Includes error handling to gracefully manage cases where the block registry might not be available or an error occurs during lookup.
    *   `const { getAllBlocks } = require('@/blocks/registry')`: This is a CommonJS-style dynamic import. It retrieves the `getAllBlocks` function from the block registry module at runtime. This allows the system to discover available blocks dynamically.
    *   `const blocks = getAllBlocks()`: Calls the function to get a list of all registered blocks in the system.
    *   `const block = blocks.find(...)`: Searches this list of blocks.
        *   `(b: { type: string; tools?: { access?: string[] } }) => b.type === blockType`: The `find` predicate looks for a block whose `type` property matches the `blockType` passed into the function. It also types the block for clarity.
    *   `return block?.tools?.access?.[0]`: If a matching block is found, it attempts to safely access its `tools` property, then its `access` array, and returns the *first* element of that array. The `?.` (optional chaining) ensures that if any part of the path is `null` or `undefined`, the expression short-circuits and returns `undefined` instead of throwing an error.
    *   `logger.error(...)`: If an error occurs, it's logged with details.
    *   `return undefined`: Returns `undefined` if an error occurs or no tool ID can be found.

```typescript
export const AgentBlock: BlockConfig<AgentResponse> = {
  type: 'agent',
  name: 'Agent',
  description: 'Build an agent',
  authMode: AuthMode.ApiKey,
  longDescription:
    'The Agent block is a core workflow block that is a wrapper around an LLM. It takes in system/user prompts and calls an LLM provider. It can also make tool calls by directly containing tools inside of its tool input. It can additionally return structured output.',
  bestPractices: `
  - Cannot use core blocks like API, Webhook, Function, Workflow, Memory as tools. Only integrations or custom tools. 
  - Check custom tools examples for YAML syntax. Only construct these if there isn't an existing integration for that purpose.
  - Response Format should be a valid JSON Schema. This determines the output of the agent only if present. Fields can be accessed at root level by the following blocks: e.g. <agent1.field>. If response format is not present, the agent will return the standard outputs: content, model, tokens, toolCalls.
  `,
  docsLink: 'https://docs.sim.ai/blocks/agent',
  category: 'blocks',
  bgColor: 'var(--brand-primary-hex)',
  icon: AgentIcon,
  subBlocks: [
    // ... (details below)
  ],
  tools: {
    // ... (details below)
  },
  inputs: {
    // ... (details below)
  },
  outputs: {
    // ... (details below)
  },
}
```
*   **`export const AgentBlock: BlockConfig<AgentResponse> = { ... }`**: This is the main export of the file. It defines the `AgentBlock` configuration object.
    *   `export const AgentBlock`: Makes this object available for other parts of the application to import.
    *   `: BlockConfig<AgentResponse>`: Asserts that this object conforms to the `BlockConfig` type, and specifies that its outputs will conform to the `AgentResponse` interface.

    *   **`type: 'agent'`**: A unique string identifier for this block type within the system.
    *   **`name: 'Agent'`**: The user-friendly name displayed in the UI.
    *   **`description: 'Build an agent'`**: A short description for the UI.
    *   **`authMode: AuthMode.ApiKey`**: Specifies that this block typically requires an API key for authentication with LLM providers.
    *   **`longDescription: '...' `**: A more detailed explanation of what the `AgentBlock` does, shown in more comprehensive UI views.
    *   **`bestPractices: `...` `**: Provides guidelines and tips for users on how to effectively use this block, including limitations on what can be used as tools and how structured output works.
    *   **`docsLink: 'https://docs.sim.ai/blocks/agent'`**: A URL pointing to the official documentation for this block.
    *   **`category: 'blocks'`**: Categorizes this block, likely for UI organization (e.g., in a palette of available blocks).
    *   **`bgColor: 'var(--brand-primary-hex)'`**: Defines the background color for the block's visual representation in the UI, using a CSS variable.
    *   **`icon: AgentIcon`**: Assigns the imported `AgentIcon` component as the visual icon for this block.

### `subBlocks` Array (UI Input Fields)

This array defines the individual input fields and controls that appear in the UI for configuring an `AgentBlock`. Each object represents a single UI element.

```typescript
  subBlocks: [
    {
      id: 'systemPrompt',
      title: 'System Prompt',
      type: 'long-input',
      layout: 'full',
      placeholder: 'Enter system prompt...',
      rows: 5,
      wandConfig: {
        enabled: true,
        maintainHistory: true, // Enable conversation history for iterative improvements
        prompt: `You are an expert system prompt engineer. Create a system prompt based on the user's request. ... (detailed meta-prompt) ...`,
        placeholder: 'Describe the AI agent you want to create...',
        generationType: 'system-prompt',
      },
    },
    // ... other sub-blocks
  ],
```
*   **`id: 'systemPrompt'`**: A unique identifier for this input field.
*   **`title: 'System Prompt'`**: The label displayed to the user for this field.
*   **`type: 'long-input'`**: Specifies the UI component type (a multi-line text area).
*   **`layout: 'full'`**: Indicates it should take up the full available width.
*   **`placeholder: 'Enter system prompt...'`**: Text displayed in the input field when it's empty.
*   **`rows: 5`**: Sets the initial height of the text area to 5 rows.
*   **`wandConfig`**: This block provides AI-assisted content generation.
    *   `enabled: true`: Activates the AI assistant feature for this field.
    *   `maintainHistory: true`: The AI assistant will remember previous interactions for iterative improvements.
    *   `prompt: `...` `: This is the "meta-prompt" given to the AI *that generates the system prompt*. It instructs the AI on how to act as an expert system prompt engineer, provides context, instructions, core principles, structure, tool integration guidelines, and examples for crafting effective system prompts. This is a powerful feature for guiding users in creating high-quality prompts.
    *   `placeholder: 'Describe the AI agent you want to create...'`: A specific placeholder for the AI assistant input.
    *   `generationType: 'system-prompt'`: Categorizes the type of content being generated.

```typescript
    {
      id: 'userPrompt',
      title: 'User Prompt',
      type: 'long-input',
      layout: 'full',
      placeholder: 'Enter context or user message...',
      rows: 3,
    },
```
*   **`id: 'userPrompt'`, `title: 'User Prompt'`, `type: 'long-input'`, etc.**: Similar to `systemPrompt`, this defines a text area for the user's input or context.

```typescript
    {
      id: 'memories',
      title: 'Memories',
      type: 'short-input',
      layout: 'full',
      placeholder: 'Connect memory block output...',
      mode: 'advanced',
    },
```
*   **`id: 'memories'`, `title: 'Memories'`, `type: 'short-input'`, etc.**: A single-line input field, likely for connecting the output of another "Memory" block, used in an "advanced" mode.

```typescript
    {
      id: 'model',
      title: 'Model',
      type: 'combobox',
      layout: 'half',
      placeholder: 'Type or select a model...',
      required: true,
      options: () => {
        const providersState = useProvidersStore.getState()
        const ollamaModels = providersState.providers.ollama.models
        const openrouterModels = providersState.providers.openrouter.models
        const baseModels = Object.keys(getBaseModelProviders())
        const allModels = Array.from(new Set([...baseModels, ...ollamaModels, ...openrouterModels]))

        return allModels.map((model) => {
          const icon = getProviderIcon(model)
          return { label: model, id: model, ...(icon && { icon }) }
        })
      },
    },
```
*   **`id: 'model'`, `title: 'Model'`, `type: 'combobox'`**: A dropdown (combobox) for selecting the LLM model.
    *   `layout: 'half'`**: Takes up half the available width.
    *   `required: true`: This field must be filled.
    *   **`options: () => { ... }`**: A function that dynamically generates the list of available model options.
        *   `const providersState = useProvidersStore.getState()`: Gets the current state from the global providers store.
        *   `const ollamaModels = providersState.providers.ollama.models`: Retrieves Ollama models.
        *   `const openrouterModels = providersState.providers.openrouter.models`: Retrieves OpenRouter models.
        *   `const baseModels = Object.keys(getBaseModelProviders())`: Gets the names of base models.
        *   `const allModels = Array.from(new Set([...]))`: Combines all these model lists, uses a `Set` to remove duplicates, and converts back to an array.
        *   `return allModels.map((model) => { ... })`: Transforms each model name into an object with `label`, `id`, and an optional `icon` (fetched using `getProviderIcon`).

```typescript
    {
      id: 'temperature',
      title: 'Temperature',
      type: 'slider',
      layout: 'half',
      min: 0,
      max: 1,
      defaultValue: 0.5,
      condition: () => ({
        field: 'model',
        value: (() => {
          const allModels = Object.keys(getAllModelProviders())
          return allModels.filter(
            (model) => supportsTemperature(model) && getMaxTemperature(model) === 1
          )
        })(),
      }),
    },
    {
      id: 'temperature', // Same ID as above, but different max value and condition
      title: 'Temperature',
      type: 'slider',
      layout: 'half',
      min: 0,
      max: 2,
      defaultValue: 1,
      condition: () => ({
        field: 'model',
        value: (() => {
          const allModels = Object.keys(getAllModelProviders())
          return allModels.filter(
            (model) => supportsTemperature(model) && getMaxTemperature(model) === 2
          )
        })(),
      }),
    },
```
*   **`id: 'temperature'`, `title: 'Temperature'`, `type: 'slider'`**: There are two definitions for the "Temperature" slider. This is a common pattern to show different ranges or default values based on other selections.
    *   `min: 0`, `max: 1`, `defaultValue: 0.5`: The first slider configured for models with a max temperature of 1.0.
    *   `min: 0`, `max: 2`, `defaultValue: 1`: The second slider configured for models with a max temperature of 2.0.
    *   **`condition: () => ({ field: 'model', value: [...] })`**: This is the key. Each slider has a condition that checks the currently selected `model`.
        *   It filters `allModels` to find those that `supportsTemperature` and have a `getMaxTemperature` equal to `1` (for the first slider) or `2` (for the second slider).
        *   Only one of these sliders will be visible at a time, based on the selected model's capabilities.

```typescript
    {
      id: 'reasoningEffort',
      title: 'Reasoning Effort',
      type: 'dropdown',
      layout: 'half',
      placeholder: 'Select reasoning effort...',
      options: [ /* ... */ ],
      value: () => 'medium',
      condition: {
        field: 'model',
        value: MODELS_WITH_REASONING_EFFORT,
      },
    },
    {
      id: 'verbosity',
      title: 'Verbosity',
      type: 'dropdown',
      layout: 'half',
      placeholder: 'Select verbosity...',
      options: [ /* ... */ ],
      value: () => 'medium',
      condition: {
        field: 'model',
        value: MODELS_WITH_VERBOSITY,
      },
    },
```
*   **`id: 'reasoningEffort'`, `title: 'Reasoning Effort'`, `type: 'dropdown'`**: A dropdown for selecting the reasoning effort level.
    *   `options`: Defines the available options (minimal, low, medium, high).
    *   `value: () => 'medium'`: Sets the default selected value to 'medium'.
    *   **`condition: { field: 'model', value: MODELS_WITH_REASONING_EFFORT }`**: This dropdown will only appear if the selected `model` is one of the models listed in the `MODELS_WITH_REASONING_EFFORT` constant.
*   **`id: 'verbosity'`, `title: 'Verbosity'`, `type: 'dropdown'`**: Similar to `reasoningEffort`, for selecting verbosity. It's conditionally displayed based on `MODELS_WITH_VERBOSITY`.

```typescript
    {
      id: 'apiKey',
      title: 'API Key',
      type: 'short-input',
      layout: 'full',
      placeholder: 'Enter your API key',
      password: true,
      connectionDroppable: false,
      required: true,
      // Hide API key for hosted models and Ollama models
      condition: isHosted
        ? {
            field: 'model',
            value: getHostedModels(),
            not: true, // Show for all models EXCEPT those listed
          }
        : () => ({
            field: 'model',
            value: getCurrentOllamaModels(),
            not: true, // Show for all models EXCEPT Ollama models
          }),
    },
```
*   **`id: 'apiKey'`, `title: 'API Key'`, `type: 'short-input'`**: An input field for the API key.
    *   `password: true`: Hides the input characters (like a password field).
    *   `connectionDroppable: false`: Prevents other blocks from being "connected" to this input, as it expects a direct string.
    *   `required: true`: This field is mandatory.
    *   **`condition: isHosted ? { ... } : () => ({ ... })`**: This is a conditional display logic based on whether the application is `isHosted`.
        *   **If `isHosted` is `true`**: The condition is `{ field: 'model', value: getHostedModels(), not: true }`. This means: "Show this API key field for any model *except* those that are `getHostedModels()` (i.e., models managed by the platform)."
        *   **If `isHosted` is `false` (self-hosted)**: The condition is `() => ({ field: 'model', value: getCurrentOllamaModels(), not: true })`. This means: "Show this API key field for any model *except* those that are `getCurrentOllamaModels()` (i.e., local Ollama models)."
        *   This intelligently hides the API key field when it's not needed, improving user experience.

```typescript
    {
      id: 'azureEndpoint',
      title: 'Azure OpenAI Endpoint',
      type: 'short-input',
      layout: 'full',
      password: true,
      placeholder: 'https://your-resource.openai.azure.com',
      connectionDroppable: false,
      condition: {
        field: 'model',
        value: providers['azure-openai'].models,
      },
    },
    {
      id: 'azureApiVersion',
      title: 'Azure API Version',
      type: 'short-input',
      layout: 'full',
      placeholder: '2024-07-01-preview',
      connectionDroppable: false,
      condition: {
        field: 'model',
        value: providers['azure-openai'].models,
      },
    },
```
*   **`id: 'azureEndpoint'`, `title: 'Azure OpenAI Endpoint'`, `type: 'short-input'`, etc.**: Input fields for Azure-specific configuration.
    *   **`condition: { field: 'model', value: providers['azure-openai'].models }`**: These fields are only shown if the selected `model` is one of the models provided by `azure-openai`.

```typescript
    {
      id: 'tools',
      title: 'Tools',
      type: 'tool-input',
      layout: 'full',
      defaultValue: [],
    },
```
*   **`id: 'tools'`, `title: 'Tools'`, `type: 'tool-input'`**: An input field specifically designed to allow users to select or define external tools that the agent can use (e.g., calling an external API, a web search tool).
    *   `defaultValue: []`: The default value is an empty array, meaning no tools are selected initially.

```typescript
    {
      id: 'responseFormat',
      title: 'Response Format',
      type: 'code',
      layout: 'full',
      placeholder: 'Enter JSON schema...',
      language: 'json',
      wandConfig: {
        enabled: true,
        maintainHistory: true,
        prompt: `You are an expert programmer specializing in creating JSON schemas according to a specific format. ... (detailed meta-prompt) ...`,
        placeholder: 'Describe the JSON schema structure you need...',
        generationType: 'json-schema',
      },
    },
```
*   **`id: 'responseFormat'`, `title: 'Response Format'`, `type: 'code'`**: A code editor input specifically for defining the desired JSON schema for the agent's output.
    *   `language: 'json'`: Configures the editor for JSON syntax highlighting.
    *   **`wandConfig`**: Similar to `systemPrompt`, this provides an AI assistant to help users generate valid JSON schemas.
        *   The `prompt` here is a highly detailed instruction for an AI to act as a JSON schema expert, specifying the exact top-level properties and internal structure expected for the schema, along with multiple examples.

### `tools` Object (Agent's Internal Tooling and Configuration)

This `tools` property defines how the `AgentBlock` interacts with LLM providers and how it processes the tools passed to it.

```typescript
  tools: {
    access: [
      'openai_chat',
      'anthropic_chat',
      'google_chat',
      'xai_chat',
      'deepseek_chat',
      'deepseek_reasoner',
    ],
    config: {
      tool: (params: Record<string, any>) => {
        const model = params.model || 'gpt-4o'
        if (!model) {
          throw new Error('No model selected')
        }
        const tool = getAllModelProviders()[model]
        if (!tool) {
          throw new Error(`Invalid model selected: ${model}`)
        }
        return tool
      },
      params: (params: Record<string, any>) => {
        // If tools array is provided, handle tool usage control
        if (params.tools && Array.isArray(params.tools)) {
          // Transform tools to include usageControl
          const transformedTools = params.tools
            // Filter out tools set to 'none' - they should never be passed to the provider
            .filter((tool: any) => {
              const usageControl = tool.usageControl || 'auto'
              return usageControl !== 'none'
            })
            .map((tool: any) => {
              const toolConfig = {
                id:
                  tool.type === 'custom-tool'
                    ? tool.schema?.function?.name
                    : tool.operation || getToolIdFromBlock(tool.type),
                name: tool.title,
                description: tool.type === 'custom-tool' ? tool.schema?.function?.description : '',
                params: tool.params || {},
                parameters: tool.type === 'custom-tool' ? tool.schema?.function?.parameters : {},
                usageControl: tool.usageControl || 'auto',
                type: tool.type,
              }
              return toolConfig
            })

          // Log which tools are being passed and which are filtered out
          const filteredOutTools = params.tools
            .filter((tool: any) => (tool.usageControl || 'auto') === 'none')
            .map((tool: any) => tool.title)

          if (filteredOutTools.length > 0) {
            logger.info('Filtered out tools set to none', { tools: filteredOutTools.join(', ') })
          }

          return { ...params, tools: transformedTools }
        }
        return params
      },
    },
  },
```
*   **`access: [ ... ]`**: This array specifies the *internal* LLM provider IDs that the `AgentBlock` is configured to use as its underlying "brain." These are the services it can directly call for generation.
*   **`config: { ... }`**: This object defines how the `AgentBlock` configures its chosen LLM and the tools it will use.
    *   **`tool: (params: Record<string, any>) => { ... }`**: This function is responsible for selecting the actual LLM provider implementation based on the user's `model` selection.
        *   `const model = params.model || 'gpt-4o'`: Gets the `model` parameter, defaulting to 'gpt-4o' if not provided.
        *   `if (!model) { ... }`: Throws an error if no model is selected.
        *   `const tool = getAllModelProviders()[model]`: Retrieves the specific LLM provider object from the system using the `model` name as a key.
        *   `if (!tool) { ... }`: Throws an error if the model is invalid.
        *   `return tool`: Returns the selected LLM provider object.
    *   **`params: (params: Record<string, any>) => { ... }`**: This function transforms the input parameters (especially the `tools` array provided *to* the agent) into a format suitable for the chosen LLM provider.
        *   `if (params.tools && Array.isArray(params.tools))`: Checks if an array of tools was provided to the agent.
        *   `const transformedTools = params.tools ... .filter(...) .map(...)`: This is the core transformation logic for external tools.
            *   `.filter((tool: any) => { ... })`: It first filters the tools. Any tool with `usageControl` explicitly set to `'none'` is removed, as it should not be passed to the LLM. Defaults to `'auto'` if not specified.
            *   `.map((tool: any) => { ... })`: For each remaining tool, it creates a `toolConfig` object with standardized properties:
                *   `id`: The unique identifier for the tool. If it's a 'custom-tool', it gets the name from its schema. Otherwise, it uses `tool.operation` or `getToolIdFromBlock` for other block types.
                *   `name`: The user-friendly title of the tool.
                *   `description`: The tool's description.
                *   `params`: Any parameters explicitly defined for the tool.
                *   `parameters`: The JSON schema for the tool's parameters (specifically for 'custom-tool' types).
                *   `usageControl`: How the LLM should handle this tool (e.g., 'auto', 'required', 'none').
                *   `type`: The original type of the tool.
        *   `const filteredOutTools = params.tools ... .filter(...) .map(...)`: This separate block identifies and logs any tools that were filtered out, which is useful for debugging or monitoring.
        *   `if (filteredOutTools.length > 0) { logger.info(...) }`: Logs a message if any tools were explicitly filtered out.
        *   `return { ...params, tools: transformedTools }`: Returns a new parameter object with the original parameters (`...params`) but with the `tools` array replaced by the `transformedTools` array.
        *   `return params`: If no `tools` array was provided, the original parameters are returned unchanged.

### `inputs` Object (Expected Inputs to the Block)

This object formally describes the data structure and types of inputs that the `AgentBlock` expects to receive from other parts of the workflow or from the UI.

```typescript
  inputs: {
    systemPrompt: { type: 'string', description: 'Initial system instructions' },
    userPrompt: { type: 'string', description: 'User message or context' },
    memories: { type: 'json', description: 'Agent memory data' },
    model: { type: 'string', description: 'AI model to use' },
    apiKey: { type: 'string', description: 'Provider API key' },
    azureEndpoint: { type: 'string', description: 'Azure OpenAI endpoint URL' },
    azureApiVersion: { type: 'string', description: 'Azure API version' },
    responseFormat: {
      type: 'json',
      description: 'JSON response format schema',
      schema: {
        type: 'object',
        properties: { /* ... nested schema ... */ },
        required: ['schema'],
      },
    },
    temperature: { type: 'number', description: 'Response randomness level' },
    reasoningEffort: { type: 'string', description: 'Reasoning effort level for GPT-5 models' },
    verbosity: { type: 'string', description: 'Verbosity level for GPT-5 models' },
    tools: { type: 'json', description: 'Available tools configuration' },
  },
```
*   Each property here corresponds to an `id` from the `subBlocks` array (or internal parameters not directly exposed as sub-blocks).
*   `type`: Specifies the expected data type (`string`, `json`, `number`).
*   `description`: A short explanation of the input's purpose.
*   **`responseFormat: { ... schema: { ... } ... }`**: For `responseFormat`, it provides a detailed JSON Schema that describes the *expected structure of the JSON schema itself* that the user provides. This ensures that the user's `responseFormat` input is valid.
    *   It expects an `object` with `name`, `schema` (which is another object defining the actual JSON schema with `type`, `properties`, `required`, `additionalProperties`), and `strict`.
    *   `required: ['schema']`: The `schema` property is mandatory within the `responseFormat` input.

### `outputs` Object (Expected Outputs from the Block)

This object formally describes the data structure and types of outputs that the `AgentBlock` will produce, making them available for other blocks in the workflow.

```typescript
  outputs: {
    content: { type: 'string', description: 'Generated response content' },
    model: { type: 'string', description: 'Model used for generation' },
    tokens: { type: 'any', description: 'Token usage statistics' },
    toolCalls: { type: 'any', description: 'Tool calls made' },
  },
```
*   These properties align with the `AgentResponse.output` interface defined earlier.
*   `type`: Specifies the data type (`string`, `any`).
*   `description`: A short explanation of the output's purpose.
*   `tokens` and `toolCalls` are typed as `any` here because their internal structure is already detailed in the `AgentResponse` interface. This often happens when the detailed type is defined once at the interface level and then `any` is used for brevity in the `outputs` configuration, assuming the `BlockConfig<AgentResponse>` generic handles the stricter typing.