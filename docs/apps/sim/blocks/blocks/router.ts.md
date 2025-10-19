This TypeScript file is a core component for defining a "Router" block within a larger workflow or automation system. Its primary purpose is to enable intelligent routing of workflow execution based on an input request. It leverages an AI model to make routing decisions, directing the workflow to the most appropriate subsequent block based on the prompt provided by the user and the available target blocks in the workflow.

Think of it as a smart traffic controller for your automated processes. You give it a task, it looks at all the possible next steps (other blocks), and decides which one is the best fit, sending the workflow down that specific path.

---

### Purpose of this File

This file serves two main purposes:

1.  **Define the `RouterBlock` configuration**: It creates a comprehensive configuration object (`RouterBlock`) that describes all aspects of a "Router" block in a workflow system. This includes its type, name, description, how it interacts with users (via its configurable "sub-blocks"), how it interacts with AI models (`tools`), and what inputs it accepts and outputs it produces.
2.  **Generate AI prompts for routing**: It provides a function (`generateRouterPrompt`) responsible for dynamically constructing the system prompt that guides an AI model to act as a routing agent. This prompt includes the user's request and detailed information about the available destination blocks, enabling the AI to make an informed routing decision.

In essence, this file defines a reusable, configurable workflow component that can intelligently direct the flow of operations based on natural language input.

---

### Detailed Explanation

Let's break down the code section by section.

#### Imports

These lines bring in external functionalities and types needed for our Router block.

```typescript
import { ConnectIcon } from '@/components/icons'
```

*   **`ConnectIcon`**: This imports a specific icon component. This icon will visually represent the "Router" block in the user interface, making it easily identifiable.

```typescript
import { isHosted } from '@/lib/environment'
```

*   **`isHosted`**: This is a utility function that likely checks if the application is running in a hosted (e.g., cloud-based) environment versus a local or self-hosted setup. This is used for conditional logic, such as determining when an API key field should be displayed.

```typescript
import { AuthMode, type BlockConfig } from '@/blocks/types'
```

*   **`AuthMode`**: An enum (a set of named constant values) defining different authentication modes. The Router block will specify its authentication requirements using this.
*   **`BlockConfig`**: This is a TypeScript `type` (or interface) that defines the expected structure for any workflow block's configuration. Our `RouterBlock` object will conform to this `BlockConfig` type.

```typescript
import type { ProviderId } from '@/providers/types'
```

*   **`ProviderId`**: A TypeScript `type` representing a unique identifier for an AI model provider (e.g., 'openai', 'anthropic', 'ollama').

```typescript
import {
  getAllModelProviders,
  getBaseModelProviders,
  getHostedModels,
  getProviderIcon,
  providers,
} from '@/providers/utils'
```

*   **`getAllModelProviders`**: A utility function that returns a comprehensive list or map of all available AI model providers.
*   **`getBaseModelProviders`**: A utility function to get a list of "base" or standard AI model providers.
*   **`getHostedModels`**: A utility function to get a list of models that are typically available in a hosted environment (e.g., via a cloud provider API).
*   **`getProviderIcon`**: A utility function that, given a model name, returns the appropriate icon for its provider.
*   **`providers`**: An object containing configuration details for various AI model providers.

```typescript
import { useProvidersStore } from '@/stores/providers/store'
```

*   **`useProvidersStore`**: This is likely a custom hook or object from a Zustand state management library. It provides access to a global store (`providersStore`) that holds information about configured AI providers and their available models (e.g., Ollama models installed locally, OpenRouter models).

```typescript
import type { ToolResponse } from '@/tools/types'
```

*   **`ToolResponse`**: A TypeScript `type` defining the basic structure of a response from an AI "tool" (which, in this context, refers to an AI model interaction).

#### Helper Function: `getCurrentOllamaModels`

```typescript
const getCurrentOllamaModels = () => {
  return useProvidersStore.getState().providers.ollama.models
}
```

*   This is a simple helper function.
*   It accesses the current state of the `useProvidersStore`.
*   Specifically, it dives into `providers.ollama.models` to retrieve a list of all currently available Ollama models.
*   **Purpose**: This list is used later to conditionally hide the API key field for Ollama models, as Ollama models are typically run locally and don't require an external API key.

#### Interfaces

These define the shapes of data objects used within the file, enhancing type safety and readability.

```typescript
interface RouterResponse extends ToolResponse {
  output: {
    prompt: string
    model: string
    tokens?: {
      prompt?: number
      completion?: number
      total?: number
    }
    cost?: {
      input: number
      output: number
      total: number
    }
    selectedPath: {
      blockId: string
      blockType: string
      blockTitle: string
    }
  }
}
```

*   **`RouterResponse`**: This interface describes the expected *output* structure when the Router block successfully completes its operation.
    *   It `extends ToolResponse`, meaning it inherits all properties from `ToolResponse` and adds its own.
    *   `output`: This object contains the specific results of the routing.
        *   `prompt`: The actual prompt that was sent to the AI model for routing.
        *   `model`: The AI model that was used for the routing decision.
        *   `tokens?`: An optional object detailing token usage (prompt tokens, completion tokens, total).
        *   `cost?`: An optional object detailing the financial cost of the AI interaction.
        *   `selectedPath`: The most crucial output—an object identifying the block chosen by the router.
            *   `blockId`: The unique ID of the selected destination block.
            *   `blockType`: The type of the selected block.
            *   `blockTitle`: The title/name of the selected block.
    *   **Purpose**: Ensures consistency in how routing results are structured and consumed by subsequent workflow blocks.

```typescript
interface TargetBlock {
  id: string
  type?: string
  title?: string
  description?: string
  category?: string
  subBlocks?: Record<string, any>
  currentState?: any
}
```

*   **`TargetBlock`**: This interface defines the structure of information provided for each *potential destination block* that the Router can choose from.
    *   `id`: The unique identifier of the target block.
    *   `type?`: Optional type of the block (e.g., 'text-generation', 'image-processor').
    *   `title?`: Optional human-readable title of the block.
    *   `description?`: Optional brief description of what the block does.
    *   `category?`: Optional category the block belongs to.
    *   `subBlocks?`: Optional configuration details of the target block (e.g., its specific settings or parameters). This is important because the AI can analyze these settings to make a better routing decision.
    *   `currentState?`: Optional current operational state of the block, which might influence routing (e.g., if a block is busy or has specific data loaded).
    *   **Purpose**: To provide the AI model with comprehensive context about each available destination block, enabling it to make an intelligent routing choice.

#### `generateRouterPrompt` Function

This function is critical for how the Router block "thinks." It constructs the detailed instructions and context given to the AI model to perform the routing task.

```typescript
export const generateRouterPrompt = (prompt: string, targetBlocks?: TargetBlock[]): string => {
  const basePrompt = `You are an intelligent routing agent responsible for directing workflow requests to the most appropriate block. Your task is to analyze the input and determine the single most suitable destination based on the request.

Key Instructions:
1. You MUST choose exactly ONE destination from the IDs of the blocks in the workflow. The destination must be a valid block id.

2. Analysis Framework:
   - Carefully evaluate the intent and requirements of the request
   - Consider the primary action needed
   - Match the core functionality with the most appropriate destination`
```

*   **`generateRouterPrompt`**: This function takes the user's `prompt` (the routing request) and an optional array of `targetBlocks` (potential destinations) and returns a complete string that will be sent to the AI model.
*   **`basePrompt`**: This defines the core role and instructions for the AI.
    *   It clearly states the AI's identity ("intelligent routing agent").
    *   Its task ("analyze the input and determine the single most suitable destination").
    *   Key rules: "MUST choose exactly ONE destination" and "destination must be a valid block id."
    *   A high-level "Analysis Framework" for how the AI should approach the decision (intent, primary action, core functionality match).
    *   **Purpose**: Sets the stage and fundamental rules for the AI's behavior.

```typescript
  // If we have target blocks, add their information to the prompt
  const targetBlocksInfo = targetBlocks
    ? `

Available Target Blocks:
${targetBlocks
  .map(
    (block) => `
ID: ${block.id}
Type: ${block.type}
Title: ${block.title}
Description: ${block.description}
System Prompt: ${JSON.stringify(block.subBlocks?.systemPrompt || '')}
Configuration: ${JSON.stringify(block.subBlocks, null, 2)}
${block.currentState ? `Current State: ${JSON.stringify(block.currentState, null, 2)}` : ''}
---`
  )
  .join('\n')}

Routing Instructions:
1. Analyze the input request carefully against each block's:
   - Primary purpose (from title, description, and system prompt)
   - Look for keywords in the system prompt that match the user's request
   - Configuration settings
   - Current state (if available)
   - Processing capabilities

2. Selection Criteria:
   - Choose the block that best matches the input's requirements
   - Consider the block's specific functionality and constraints
   - Factor in any relevant current state or configuration
   - Prioritize blocks that can handle the input most effectively`
    : ''
```

*   **`targetBlocksInfo`**: This is a conditional string.
    *   `targetBlocks ? ... : ''`: If `targetBlocks` are provided, it constructs a detailed section; otherwise, it's an empty string.
    *   `Available Target Blocks:`: Introduces the list of blocks.
    *   `targetBlocks.map(...)`: It iterates over each `block` in the `targetBlocks` array.
    *   For each block, it formats its `ID`, `Type`, `Title`, `Description`, `System Prompt` (if available in `subBlocks`), `Configuration` (the full `subBlocks` object, prettified with `JSON.stringify(..., null, 2)`), and `Current State` (if `block.currentState` exists).
        *   `JSON.stringify(block.subBlocks?.systemPrompt || '')`: Safely extracts and stringifies a system prompt from the block's sub-configurations.
        *   `JSON.stringify(block.subBlocks, null, 2)`: Dumps the entire sub-block configuration in a readable JSON format.
    *   `.join('\n')`: Combines all the individual block descriptions into a single string, separated by newlines.
    *   `Routing Instructions`: Provides specific, detailed guidance to the AI on *how* to evaluate each target block. It tells the AI to look at purpose, keywords, configuration, and current state.
    *   `Selection Criteria`: Further refines how the AI should prioritize its choice, emphasizing best match, specific functionality, and effective handling.
    *   **Purpose**: This section provides the AI with the necessary data and a clear framework to analyze the available options and make an informed routing decision. Without this, the AI would be guessing.

```typescript
  return `${basePrompt}${targetBlocksInfo}

Routing Request: ${prompt}

Response Format:
Return ONLY the destination id as a single word, lowercase, no punctuation or explanation.
Example: "2acd9007-27e8-4510-a487-73d3b825e7c1"

Remember: Your response must be ONLY the block ID - no additional text, formatting, or explanation.`
}
```

*   **Return Statement**: This concatenates all parts of the prompt.
    *   `${basePrompt}${targetBlocksInfo}`: Combines the general instructions with the detailed block information.
    *   `Routing Request: ${prompt}`: Inserts the actual user-provided routing request.
    *   `Response Format:`: Crucially, this section dictates the exact output format the AI *must* adhere to. It instructs the AI to return *only* the block ID, ensuring that the system can easily parse the AI's response.
    *   `Remember: Your response must be ONLY the block ID...`: Reiterates the strict output format for clarity.
    *   **Purpose**: Assembles the complete prompt string ready to be sent to an AI model. The strict response format is essential for programmatic parsing of the AI's decision.

#### `RouterBlock` Configuration Object

This is the main definition of our Router workflow block, conforming to the `BlockConfig` interface.

```typescript
export const RouterBlock: BlockConfig<RouterResponse> = {
  type: 'router',
  name: 'Router',
  description: 'Route workflow',
  authMode: AuthMode.ApiKey,
  longDescription:
    'This is a core workflow block. Intelligently direct workflow execution to different paths based on input analysis. Use natural language to instruct the router to route to certain blocks based on the input.',
  bestPractices: `
  - For the prompt, make it almost programmatic. Use the system prompt to define the routing criteria. Should be very specific with no ambiguity.
  - Use the target block *names* to define the routing criteria.
  `,
  category: 'blocks',
  bgColor: '#28C43F',
  icon: ConnectIcon,
```

*   **`type`**: A unique identifier for this block type within the system ('router').
*   **`name`**: The human-readable name displayed in the UI ('Router').
*   **`description`**: A short description for the UI ('Route workflow').
*   **`authMode`**: Specifies that this block requires an API key for authentication (`AuthMode.ApiKey`).
*   **`longDescription`**: A more detailed explanation of the block's functionality, helpful for users.
*   **`bestPractices`**: Provides guidance to users on how to best configure and use this block for optimal results.
*   **`category`**: Groups this block under a specific category in the UI ('blocks').
*   **`bgColor`**: The background color for the block's representation in the UI.
*   **`icon`**: The imported `ConnectIcon` to visually represent the block.
*   **Purpose**: These are metadata and UI-related properties that define the Router block's identity and presentation.

```typescript
  subBlocks: [
    {
      id: 'prompt',
      title: 'Prompt',
      type: 'long-input',
      layout: 'full',
      placeholder: 'Route to the correct block based on the input...',
      required: true,
    },
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
    {
      id: 'temperature',
      title: 'Temperature',
      type: 'slider',
      layout: 'half',
      hidden: true,
      min: 0,
      max: 2,
    },
    {
      id: 'systemPrompt',
      title: 'System Prompt',
      type: 'code',
      layout: 'full',
      hidden: true,
      value: (params: Record<string, any>) => {
        return generateRouterPrompt(params.prompt || '')
      },
    },
  ],
```

*   **`subBlocks`**: This is an array defining the individual configuration fields or inputs that the user sees and interacts with when setting up this Router block in the UI.
    *   **`prompt`**:
        *   `id: 'prompt'`, `title: 'Prompt'`, `type: 'long-input'`: A large text area where the user specifies their routing instructions or request.
        *   `required: true`: This field must be filled out.
    *   **`model`**:
        *   `id: 'model'`, `title: 'Model'`, `type: 'combobox'`: A dropdown/searchable input for selecting the AI model to be used for routing.
        *   `options: () => { ... }`: This is a function that dynamically generates the list of available models.
            *   It retrieves models from `useProvidersStore` (Ollama, OpenRouter) and from `getBaseModelProviders()`.
            *   `Array.from(new Set([...]))`: Combines all model lists and removes duplicates to ensure each model appears only once.
            *   It then maps these model names to an array of objects, each with a `label`, `id`, and optionally an `icon` (fetched via `getProviderIcon`).
            *   **Simplified Logic**: This ensures the user sees all models they have configured or that are generally available, allowing them to choose the best AI for routing.
    *   **`apiKey`**:
        *   `id: 'apiKey'`, `title: 'API Key'`, `type: 'short-input'`, `password: true`: A text input for the API key, masked for security.
        *   `condition: isHosted ? { ... } : () => ({ ... })`: This is a **complex conditional logic** for showing/hiding the API key field.
            *   If `isHosted` is `true` (running in a hosted environment), it hides the API key field for models listed by `getHostedModels()`. This implies hosted environments might automatically manage API keys for certain providers.
            *   If `isHosted` is `false` (not hosted, e.g., local), it hides the API key field specifically for models returned by `getCurrentOllamaModels()`. Ollama models typically run locally and don't need external API keys.
            *   `not: true`: In both cases, the condition is set to `not: true`, meaning the field is shown for *all models EXCEPT* those specified in the `value` array.
            *   **Simplified Logic**: This intelligently shows the API key field *only* when it's genuinely needed for the selected AI model, improving user experience.
    *   **`azureEndpoint`, `azureApiVersion`**:
        *   These are specific input fields for Azure OpenAI models.
        *   `condition: { field: 'model', value: providers['azure-openai'].models }`: These fields are only displayed if the currently selected `model` is one of the models associated with 'azure-openai' in the `providers` configuration.
        *   **Simplified Logic**: These fields are contextually shown, only appearing when an Azure OpenAI model is selected, keeping the UI clean.
    *   **`temperature`**:
        *   `id: 'temperature'`, `type: 'slider'`, `hidden: true`: A slider to control the AI's "creativity" or randomness, but it's *hidden* by default. For routing, a low temperature (less randomness) is generally preferred to ensure consistent decisions.
    *   **`systemPrompt`**:
        *   `id: 'systemPrompt'`, `type: 'code'`, `hidden: true`: This field holds the actual prompt sent to the AI, but it's hidden from the user.
        *   `value: (params: Record<string, any>) => { return generateRouterPrompt(params.prompt || '') }`: This is a function that dynamically generates the `systemPrompt`'s content. It calls our `generateRouterPrompt` function, passing the user's input `prompt` (from the `prompt` sub-block) and any available `targetBlocks` (though `targetBlocks` are not explicitly passed here, they would typically be passed from the workflow context).
        *   **Simplified Logic**: The user doesn't directly edit the complex AI system prompt. Instead, their simple routing `prompt` input (from the 'prompt' sub-block) is combined with the system's detailed instructions (from `generateRouterPrompt`) to form the complete AI prompt.

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
        const tool = getAllModelProviders()[model as ProviderId]
        if (!tool) {
          throw new Error(`Invalid model selected: ${model}`)
        }
        return tool
      },
    },
  },
```

*   **`tools`**: This section defines how the Router block interacts with AI models (referred to as "tools" in this context).
    *   **`access`**: An array listing the `ProviderId`s of all AI chat models that this Router block is permitted to use (e.g., OpenAI, Anthropic, Google).
    *   **`config.tool`**: This is a function that, given the current block's parameters (`params`), determines *which specific AI tool (provider configuration)* should be used.
        *   `const model = params.model || 'gpt-4o'`: Retrieves the selected model ID from the block's parameters (defaulting to 'gpt-4o' if none is selected).
        *   `getAllModelProviders()[model as ProviderId]`: It uses the selected `model` ID to look up the actual tool configuration from the list of all available model providers.
        *   Error handling: Throws an error if no model is selected or if the selected model is invalid.
        *   `return tool`: Returns the configuration for the chosen AI tool.
    *   **Simplified Logic**: This dynamically connects the user's `model` selection in the UI (from `subBlocks.model`) to the actual backend AI service that will process the routing request.

```typescript
  inputs: {
    prompt: { type: 'string', description: 'Routing prompt content' },
    model: { type: 'string', description: 'AI model to use' },
    apiKey: { type: 'string', description: 'Provider API key' },
    azureEndpoint: { type: 'string', description: 'Azure OpenAI endpoint URL' },
    azureApiVersion: { type: 'string', description: 'Azure API version' },
    temperature: {
      type: 'number',
      description: 'Response randomness level (low for consistent routing)',
    },
  },
```

*   **`inputs`**: This object defines the expected input data structure *for this block*. These are the parameters that can be passed *into* the Router block from previous blocks in a workflow.
    *   Each property (e.g., `prompt`, `model`) specifies its `type` and a `description`.
    *   **Purpose**: This helps the workflow system understand what data the Router block needs to operate and ensures type consistency when chaining blocks.

```typescript
  outputs: {
    prompt: { type: 'string', description: 'Routing prompt used' },
    model: { type: 'string', description: 'Model used' },
    tokens: { type: 'json', description: 'Token usage' },
    cost: { type: 'json', description: 'Cost information' },
    selectedPath: { type: 'json', description: 'Selected routing path' },
  },
}
```

*   **`outputs`**: This object defines the expected output data structure *produced by this block*. These are the results that can be passed *out* of the Router block to subsequent blocks.
    *   The structure directly mirrors the `output` property of the `RouterResponse` interface we defined earlier.
    *   `type: 'json'` is used for `tokens`, `cost`, and `selectedPath` because their internal structure is more complex than a simple string or number.
    *   **Purpose**: Clearly documents what information the Router block provides, allowing other blocks in the workflow to consume its results effectively, especially the `selectedPath` which dictates the next step.

---

### Simplifying Complex Logic

1.  **Dynamic Model Selection & API Key Visibility**:
    *   **Complexity**: The `model` combobox dynamically gathers available models from various sources (base, Ollama, OpenRouter). The `apiKey` field's visibility is conditional: it hides if the selected model is one of the hosted ones (in a hosted environment) or an Ollama model (locally run).
    *   **Simplification**: The system is smart enough to know which models you have available and which ones require an API key. You only see the API key field when you've selected a model that actually needs one, keeping the interface clean and relevant.

2.  **`generateRouterPrompt` Function**:
    *   **Complexity**: This function constructs a long, multi-part prompt for the AI, including static instructions, detailed information about all target blocks, and strict output format rules.
    *   **Simplification**: Instead of you, the user, having to craft a complex prompt for the AI every time, this function does it automatically. You just provide your simple routing instruction, and the system intelligently adds all the necessary context about the available workflow steps for the AI to make a good decision. It also ensures the AI's answer is in a format the system can understand (just the block ID).

3.  **Connecting UI Selection to AI Tool**:
    *   **Complexity**: The `subBlocks.model` allows a user to pick a model from a dropdown, and the `tools.config.tool` function then needs to ensure the correct AI provider is invoked based on that selection.
    *   **Simplification**: When you choose an AI model in the Router block's configuration, the system automatically knows which underlying AI service (e.g., OpenAI, Anthropic) to call and how to configure it. You don't have to manually link your model choice to a specific API call.

4.  **User Input to System Prompt**:
    *   **Complexity**: The `systemPrompt` sub-block is hidden and its `value` is derived by calling `generateRouterPrompt` with the user's `prompt`.
    *   **Simplification**: You, as the workflow builder, only write a straightforward prompt (`subBlocks.prompt`) explaining your routing logic. The system then takes your simple prompt, combines it with the detailed, prescriptive instructions generated by `generateRouterPrompt`, and sends that complete, powerful instruction set to the AI. This way, you get the benefit of a highly intelligent AI router without needing to understand the complex prompt engineering required behind the scenes.

---

In summary, this file defines a powerful and flexible "Router" block that acts as an AI-powered decision maker within a workflow. It simplifies complex AI interactions by providing a user-friendly interface for configuration and automating the detailed prompt engineering required for intelligent routing.