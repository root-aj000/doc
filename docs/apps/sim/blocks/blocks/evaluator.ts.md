This TypeScript file is a comprehensive definition of an "Evaluator" block, designed to be integrated into a larger workflow or AI orchestration platform.

## Purpose of this file

The primary purpose of this file is to **define a reusable "Evaluator" component (or "block")** within a system. This block allows users to:

1.  **Define custom evaluation metrics:** Specify criteria (e.g., "Clarity," "Accuracy," "Fluency") with associated score ranges and descriptions.
2.  **Provide content for evaluation:** Supply text or data that needs to be assessed.
3.  **Select an AI model:** Choose a large language model (LLM) to perform the actual evaluation.
4.  **Get structured evaluation results:** Receive a JSON object containing scores for each defined metric, along with model usage details.

Essentially, this file sets up everything needed for the system to:
*   Display the "Evaluator" block in a user interface.
*   Configure its inputs (metrics, content, model, API keys).
*   Generate the necessary instructions (prompts and response formats) for an AI model to perform the evaluation.
*   Handle the communication with various AI providers.
*   Define the output structure of the evaluation.

It acts as a blueprint, allowing developers to easily integrate a powerful, configurable evaluation step into their AI workflows without having to rebuild the logic every time.

---

## Simplified Complex Logic

The most complex parts of this file revolve around dynamically generating the instructions for an AI model:

1.  **Crafting the AI Prompt (`generateEvaluatorPrompt`):**
    *   **Problem:** LLMs need very clear, unambiguous instructions to produce structured output.
    *   **Solution:** This function takes the user-defined metrics and content and constructs a highly detailed prompt. It explicitly tells the AI:
        *   Its role ("objective evaluation agent").
        *   What to evaluate (the `content`).
        *   *How* to evaluate (against `metrics` and their `ranges`).
        *   The *exact format* for its response (a JSON object with lowercase metric names as keys and numeric scores as values).
        *   It even provides a concrete example of the expected JSON output to eliminate ambiguity.
        *   It also intelligently tries to pretty-print any JSON content provided, making it easier for the AI to parse.

2.  **Defining the AI Response Schema (`generateResponseFormat`):**
    *   **Problem:** Modern LLM APIs (like OpenAI's Function Calling or tool usage) can be guided by a JSON schema to ensure they return data in a specific, machine-readable format.
    *   **Solution:** This function takes the same user-defined metrics and converts them into a formal JSON Schema structure. This schema tells the AI API exactly what properties (the metrics), their types (`number`), descriptions, and whether they are required. This enforces strict adherence to the desired output format, making parsing the AI's response much more reliable.

3.  **Dynamic UI and API Key Handling (`EvaluatorBlock.subBlocks`):**
    *   **Problem:** Different AI models or deployment environments (e.g., hosted vs. local) might require different authentication methods or configurations.
    *   **Solution:** The `subBlocks` configuration uses `condition` properties to dynamically show or hide fields like `apiKey` or `azureEndpoint`. For example, if a user selects a model that's already managed by the hosted platform, they don't need to provide an API key. If they select an Azure OpenAI model, the Azure-specific endpoint and API version fields appear. This creates a flexible and user-friendly interface.

---

## Explanation of Each Line of Code

Let's break down the file section by section.

### Imports

These lines bring in necessary components, types, and utilities from other parts of the application.

```typescript
import { ChartBarIcon } from '@/components/icons'
```
*   **`import { ChartBarIcon } from '@/components/icons'`**: Imports a specific icon component (`ChartBarIcon`) from the application's icon library. This icon will likely be used to visually represent the "Evaluator" block in the user interface.

```typescript
import { isHosted } from '@/lib/environment'
```
*   **`import { isHosted } from '@/lib/environment'`**: Imports a utility function `isHosted` from the environment library. This function likely checks if the application is running in a hosted (cloud) environment or locally, which can affect how certain features (like API key inputs) behave.

```typescript
import { createLogger } from '@/lib/logs/console/logger'
```
*   **`import { createLogger } from '@/lib/logs/console/logger'`**: Imports a utility function `createLogger` for creating logging instances. This is used for debugging and reporting warnings/errors.

```typescript
import type { BlockConfig, ParamType } from '@/blocks/types'
```
*   **`import type { BlockConfig, ParamType } from '@/blocks/types'`**: Imports TypeScript type definitions.
    *   `BlockConfig`: A generic type that defines the overall structure for configuring any block in the system.
    *   `ParamType`: A type used to specify the data type of a parameter (e.g., 'string', 'number', 'json').

```typescript
import type { ProviderId } from '@/providers/types'
```
*   **`import type { ProviderId } from '@/providers/types'`**: Imports a TypeScript type `ProviderId`, which likely represents a unique identifier for an AI model provider (e.g., 'openai', 'anthropic', 'ollama').

```typescript
import {
  getAllModelProviders,
  getBaseModelProviders,
  getHostedModels,
  getProviderIcon,
  providers,
} from '@/providers/utils'
```
*   **`import { ... } from '@/providers/utils'`**: Imports several utility functions and an object related to AI model providers.
    *   `getAllModelProviders`: A function to get a map of all available AI model providers.
    *   `getBaseModelProviders`: A function to get providers that are considered "base" or standard.
    *   `getHostedModels`: A function to get models that are managed directly by the hosted platform (e.g., don't require user API keys).
    *   `getProviderIcon`: A function to retrieve an icon associated with a specific AI provider.
    *   `providers`: An object likely containing static information or configurations for various providers.

```typescript
import { useProvidersStore } from '@/stores/providers/store'
```
*   **`import { useProvidersStore } from '@/stores/providers/store'`**: Imports a Zustand or similar global state store for managing AI provider information, particularly the available models.

```typescript
import type { ToolResponse } from '@/tools/types'
```
*   **`import type { ToolResponse } from '@/tools/types'`**: Imports a TypeScript type `ToolResponse`, which likely defines the base structure for responses received from "tools" (in this context, AI models acting as tools).

### Logger Initialization

```typescript
const logger = createLogger('EvaluatorBlock')
```
*   **`const logger = createLogger('EvaluatorBlock')`**: Creates a logger instance specifically named 'EvaluatorBlock'. This allows logs originating from this file to be easily identified in the console.

### Helper Function: `getCurrentOllamaModels`

```typescript
const getCurrentOllamaModels = () => {
  return useProvidersStore.getState().providers.ollama.models
}
```
*   **`const getCurrentOllamaModels = () => { ... }`**: Defines a simple arrow function to retrieve the currently configured Ollama models from the global `providersStore`. This is a quick way to access this specific list of models, likely for conditional UI logic later.

### Interfaces for Data Structures

These define the expected shapes of data that this block will work with.

```typescript
interface Metric {
  name: string
  description: string
  range: {
    min: number
    max: number
  }
}
```
*   **`interface Metric { ... }`**: Defines the structure for a single evaluation metric.
    *   **`name: string`**: The name of the metric (e.g., "Clarity").
    *   **`description: string`**: A detailed explanation of what the metric measures.
    *   **`range: { min: number; max: number }`**: An object specifying the minimum and maximum possible scores for this metric.

```typescript
interface EvaluatorResponse extends ToolResponse {
  output: {
    content: string
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
    [metricName: string]: any // Allow dynamic metric fields
  }
}
```
*   **`interface EvaluatorResponse extends ToolResponse { ... }`**: Defines the expected structure of the final evaluation response.
    *   **`extends ToolResponse`**: It inherits properties from the `ToolResponse` interface, meaning it will include any base fields common to all tool responses.
    *   **`output: { ... }`**: An object containing the core evaluation results.
        *   **`content: string`**: The primary output content, which will be the AI's structured evaluation (typically stringified JSON).
        *   **`model: string`**: The name of the AI model that performed the evaluation.
        *   **`tokens?: { ... }`**: An optional object detailing token usage (prompt, completion, total). The `?` makes it optional.
        *   **`cost?: { ... }`**: An optional object detailing the cost of the AI call (input, output, total).
        *   **`[metricName: string]: any`**: This is an index signature. It allows `output` to have any number of additional properties whose names are strings (e.g., "clarity", "accuracy") and whose values can be of any type (`any`, though expected to be `number` based on the prompt). This is how the dynamic metric scores will be stored.

### Function: `generateEvaluatorPrompt`

This crucial function constructs the detailed textual prompt that will be sent to the AI model.

```typescript
export const generateEvaluatorPrompt = (metrics: Metric[], content: string): string => {
```
*   **`export const generateEvaluatorPrompt = (metrics: Metric[], content: string): string => {`**: Defines an exported function `generateEvaluatorPrompt`. It takes an array of `Metric` objects and the `content` to be evaluated (as a string) and returns a single string, which is the complete prompt for the AI.

```typescript
  // Filter out invalid/incomplete metrics first
  const validMetrics = metrics.filter((m) => m?.name && m.range)
```
*   **`const validMetrics = metrics.filter((m) => m?.name && m.range)`**: Filters the input `metrics` array to ensure only metrics with both a `name` and a `range` defined are used. This prevents errors if incomplete metric data is provided.

```typescript
  // Create a clear metrics description with name, range, and description
  const metricsDescription = validMetrics
    .map(
      (metric) =>
        `"${metric.name}" (${metric.range.min}-${metric.range.max}): ${metric.description || ''}` // Handle potentially missing description
    )
    .join('\n')
```
*   **`const metricsDescription = validMetrics.map(...).join('\n')`**: This block generates a human-readable list of metrics for the AI.
    *   `.map(...)`: Iterates over `validMetrics`. For each metric, it creates a string like: `"MetricName" (min-max): Metric Description`.
    *   `metric.description || ''`: Handles cases where a description might be missing, defaulting to an empty string.
    *   `.join('\n')`: Joins all the generated metric strings together, separating them with a newline character, creating a clear list.

```typescript
  // Format the content properly - try to detect and format JSON
  let formattedContent = content
  try {
    // If content looks like JSON (starts with { or [)
    if (
      typeof content === 'string' &&
      (content.trim().startsWith('{') || content.trim().startsWith('['))
    ) {
      // Try to parse and pretty-print
      const parsedContent = JSON.parse(content)
      formattedContent = JSON.stringify(parsedContent, null, 2)
    }
    // If it's already an object (shouldn't happen here but just in case)
    else if (typeof content === 'object') {
      formattedContent = JSON.stringify(content, null, 2)
    }
  } catch (e) {
    logger.warn('Warning: Content may not be valid JSON, using as-is', { e })
    formattedContent = content
  }
```
*   **`let formattedContent = content; try { ... } catch (e) { ... }`**: This section attempts to format the input `content` for better AI readability, especially if it's JSON.
    *   `formattedContent = content`: Initializes `formattedContent` with the original content.
    *   `try { ... } catch (e) { ... }`: A `try-catch` block is used to gracefully handle potential errors during JSON parsing.
    *   `if (typeof content === 'string' && (content.trim().startsWith('{') || content.trim().startsWith('[')))`: Checks if the content is a string and appears to be JSON (starts with `{` for objects or `[` for arrays, after trimming whitespace).
    *   `const parsedContent = JSON.parse(content)`: If it looks like JSON, it attempts to parse the string into a JavaScript object.
    *   `formattedContent = JSON.stringify(parsedContent, null, 2)`: If parsing succeeds, it converts the object back into a nicely formatted (pretty-printed) JSON string with 2-space indentation. This is easier for LLMs to read.
    *   `else if (typeof content === 'object')`: A fallback for if the content was *already* an object (though the function signature expects a string, defensive coding).
    *   `logger.warn(...)`: If parsing fails, a warning is logged, and the original `content` is used as is.

```typescript
  // Generate an example of the expected output format using only valid metrics
  const exampleOutput = validMetrics.reduce(
    (acc, metric) => {
      // Ensure metric and name are valid before using them
      if (metric?.name) {
        acc[metric.name.toLowerCase()] = Math.floor((metric.range.min + metric.range.max) / 2) // Use middle of range as example
      } else {
        logger.warn('Skipping invalid metric during example generation:', metric)
      }
      return acc
    },
    {} as Record<string, number>
  )
```
*   **`const exampleOutput = validMetrics.reduce(...)`**: This block generates a concrete example of the expected JSON output format for the AI. This is extremely helpful for guiding LLMs to produce the correct structure.
    *   `validMetrics.reduce((acc, metric) => { ... }, {} as Record<string, number>)`: The `reduce` method iterates through `validMetrics` and builds up an accumulator object (`acc`).
    *   `if (metric?.name)`: Ensures the metric has a name.
    *   `acc[metric.name.toLowerCase()] = Math.floor((metric.range.min + metric.range.max) / 2)`: For each valid metric, it adds a property to `acc`. The key is the lowercase version of the metric's name (as specified in the prompt instructions), and the value is the midpoint of its score range (rounded down) as an example score.
    *   `{} as Record<string, number>`: The initial value for the accumulator, an empty object type-hinted to store string keys and number values.

```typescript
  return `You are an objective evaluation agent. Analyze the content against the provided metrics and provide detailed scoring.

Evaluation Instructions:
- You MUST evaluate the content against each metric
- For each metric, provide a numeric score within the specified range
- Your response MUST be a valid JSON object with each metric name as a key and a numeric score as the value
- IMPORTANT: Use lowercase versions of the metric names as keys in your JSON response
- Follow the exact schema of the response format provided to you
- Do not include explanations in the JSON - only numeric scores
- Do not add any additional fields not specified in the schema
- Do not include ANY text before or after the JSON object

Metrics to evaluate:
${metricsDescription}

Content to evaluate:
${formattedContent}

Example of expected response format (with different scores):
${JSON.stringify(exampleOutput, null, 2)}

Remember: Your response MUST be a valid JSON object containing only the lowercase metric names as keys with their numeric scores as values. No text explanations.`
}
```
*   **`return ``...```**: This is a template literal that constructs the final prompt string. It's carefully designed to give the AI explicit instructions and context.
    *   **"You are an objective evaluation agent."**: Sets the persona for the AI.
    *   **"Evaluation Instructions:"**: A clear list of rules for the AI, emphasizing `MUST` for critical requirements.
        *   `MUST evaluate content against each metric`: Ensures all metrics are covered.
        *   `numeric score within the specified range`: Adheres to metric definitions.
        *   `response MUST be a valid JSON object`: Crucial for machine readability.
        *   `IMPORTANT: Use lowercase versions... as keys`: Enforces a consistent key format.
        *   `Follow the exact schema...`: Reinforces adherence to structured output.
        *   `Do not include explanations...`: Prevents extraneous text inside the JSON.
        *   `Do not add any additional fields...`: Enforces strictness.
        *   `Do not include ANY text before or after the JSON object`: Guarantees a pure JSON response, critical for automated parsing.
    *   **"Metrics to evaluate:"**: Inserts the `metricsDescription` generated earlier.
    *   **"Content to evaluate:"**: Inserts the `formattedContent`.
    *   **"Example of expected response format:"**: Inserts the `exampleOutput` object, stringified and pretty-printed into JSON.
    *   **"Remember: Your response MUST be..."**: A final, strong reiteration of the most critical instruction to ensure compliance.

### Function: `generateResponseFormat`

This function creates a JSON Schema definition that describes the *expected output structure* from the AI. This is often used by LLM APIs for "tool calling" or "function calling" to enforce the output format.

```typescript
const generateResponseFormat = (metrics: Metric[]) => {
```
*   **`const generateResponseFormat = (metrics: Metric[]) => {`**: Defines a function `generateResponseFormat` that takes an array of `Metric` objects and returns an object representing the JSON Schema.

```typescript
  // Filter out invalid/incomplete metrics first
  const validMetrics = metrics.filter((m) => m?.name)
```
*   **`const validMetrics = metrics.filter((m) => m?.name)`**: Filters the input metrics, ensuring only those with a valid `name` are considered for the schema.

```typescript
  // Create properties for each metric
  const properties: Record<string, any> = {}
```
*   **`const properties: Record<string, any> = {}`**: Initializes an empty object `properties`. This object will hold the schema definitions for each individual metric.

```typescript
  // Add each metric as a property
  validMetrics.forEach((metric) => {
    // We've already filtered, but double-check just in case
    if (metric?.name) {
      properties[metric.name.toLowerCase()] = {
        type: 'number',
        description: `${metric.description || ''} (Score between ${metric.range?.min ?? 0}-${metric.range?.max ?? 'N/A'})`, // Safely access range
      }
    } else {
      logger.warn('Skipping invalid metric during response format property generation:', metric)
    }
  })
```
*   **`validMetrics.forEach((metric) => { ... })`**: Iterates through each `validMetric`.
    *   `if (metric?.name)`: A final defensive check for a metric name.
    *   `properties[metric.name.toLowerCase()] = { ... }`: Adds a new property to the `properties` object. The key is the lowercase metric name.
    *   **`type: 'number'`**: Specifies that the value for this metric in the AI's response should be a number.
    *   **`description: \`...\``**: Provides a description for the property in the schema, including the metric's original description and its allowed score range.
        *   `metric.range?.min ?? 0`: Safely accesses `min`, defaulting to `0` if `range` or `min` is undefined.
        *   `metric.range?.max ?? 'N/A'`: Safely accesses `max`, defaulting to `'N/A'` if `range` or `max` is undefined.
    *   `logger.warn(...)`: Logs a warning if a metric without a name is somehow encountered here.

```typescript
  // Return a proper JSON Schema format
  return {
    name: 'evaluation_response',
    schema: {
      type: 'object',
      properties,
      // Use only valid, lowercase metric names for the required array
      required: validMetrics
        .filter((metric) => metric?.name)
        .map((metric) => metric.name.toLowerCase()),
      additionalProperties: false,
    },
    strict: true,
  }
}
```
*   **`return { ... }`**: Returns the complete JSON Schema definition object.
    *   **`name: 'evaluation_response'`**: A name for this specific response schema.
    *   **`schema: { ... }`**: The core JSON Schema definition.
        *   **`type: 'object'`**: States that the overall response should be a JSON object.
        *   **`properties`**: The `properties` object created above, containing definitions for each metric.
        *   **`required: validMetrics.filter(...).map(...)`**: An array listing the names of all required properties (metrics) in the AI's response. It ensures only valid, lowercase metric names are included.
        *   **`additionalProperties: false`**: This is a critical setting. It tells the AI (or API parsing its response) that *no other properties* beyond those explicitly listed in `properties` are allowed in the response. This enforces strict adherence to the schema.
    *   **`strict: true`**: A custom property (likely for the platform's internal schema validation) further emphasizing strictness.

### `EvaluatorBlock` Configuration

This is the main exported object that defines the "Evaluator" block's behavior, appearance, and configuration options within the larger system.

```typescript
export const EvaluatorBlock: BlockConfig<EvaluatorResponse> = {
```
*   **`export const EvaluatorBlock: BlockConfig<EvaluatorResponse> = {`**: Declares and exports a constant `EvaluatorBlock`. It's typed as `BlockConfig<EvaluatorResponse>`, meaning it's a block configuration that is expected to produce an `EvaluatorResponse` as its output.

```typescript
  type: 'evaluator',
  name: 'Evaluator',
  description: 'Evaluate content',
  longDescription:
    'This is a core workflow block. Assess content quality using customizable evaluation metrics and scoring criteria. Create objective evaluation frameworks with numeric scoring to measure performance across multiple dimensions.',
  docsLink: 'https://docs.sim.ai/blocks/evaluator',
  category: 'tools',
  bgColor: '#4D5FFF',
  icon: ChartBarIcon,
```
*   **`type: 'evaluator'`**: A unique identifier for this block type within the system.
*   **`name: 'Evaluator'`**: The display name of the block in the UI.
*   **`description: 'Evaluate content'`**: A short description of the block's function.
*   **`longDescription: '...'`**: A more detailed explanation, helpful for users understanding its capabilities.
*   **`docsLink: 'https://docs.sim.ai/blocks/evaluator'`**: A link to relevant documentation.
*   **`category: 'tools'`**: Categorizes the block, likely for organization in a palette or menu.
*   **`bgColor: '#4D5FFF'`**: Defines the background color for the block's representation in the UI.
*   **`icon: ChartBarIcon`**: Specifies the icon component to be displayed with the block, imported earlier.

```typescript
  subBlocks: [
    {
      id: 'metrics',
      title: 'Evaluation Metrics',
      type: 'eval-input',
      layout: 'full',
      required: true,
    },
    {
      id: 'content',
      title: 'Content',
      type: 'short-input',
      layout: 'full',
      placeholder: 'Enter the content to evaluate',
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
      min: 0,
      max: 2,
      value: () => '0.1',
      hidden: true,
    },
    {
      id: 'systemPrompt',
      title: 'System Prompt',
      type: 'code',
      layout: 'full',
      hidden: true,
      value: (params: Record<string, any>) => {
        try {
          const metrics = params.metrics || []

          // Process content safely
          let processedContent = ''
          if (typeof params.content === 'object') {
            processedContent = JSON.stringify(params.content, null, 2)
          } else {
            processedContent = String(params.content || '')
          }

          // Generate prompt and response format directly
          const promptText = generateEvaluatorPrompt(metrics, processedContent)
          const responseFormatObj = generateResponseFormat(metrics)

          // Create a clean, simple JSON object
          const result = {
            systemPrompt: promptText,
            responseFormat: responseFormatObj,
          }

          return JSON.stringify(result)
        } catch (e) {
          logger.error('Error in systemPrompt value function:', { e })
          // Return a minimal valid JSON as fallback
          return JSON.stringify({
            systemPrompt: 'Evaluate the content and return a JSON with metric scores.',
            responseFormat: {
              schema: {
                type: 'object',
                properties: {},
                additionalProperties: true,
              },
            },
          })
        }
      },
    },
  ],
```
*   **`subBlocks: [...]`**: This array defines the individual input fields and configurations that will appear in the UI for this block. Each object within the array represents one input field.
    *   **`id`**: A unique identifier for the input field.
    *   **`title`**: The label displayed to the user.
    *   **`type`**: The type of UI component (e.g., `eval-input`, `short-input`, `combobox`, `slider`, `code`).
    *   **`layout`**: Specifies how the component should be arranged (e.g., `full` width, `half` width).
    *   **`placeholder`**: Text displayed when the input is empty.
    *   **`required`**: Boolean indicating if the field is mandatory.
    *   **`password`**: If `true`, input is masked (for API keys).
    *   **`connectionDroppable`**: If `false`, prevents connecting workflow lines to this input.
    *   **`hidden`**: If `true`, the field is not displayed in the UI.
    *   **`condition`**: A powerful property that determines *when* an input field should be shown or hidden based on the values of other fields.
        *   It can be an object with `field`, `value`, and `not` properties (e.g., show if `field`'s value is *not* one of the `value`s).
        *   It can also be a function that dynamically returns such an object.
    *   **`options`**: For `combobox` types, a function that returns the available selection options (e.g., list of models).
    *   **`value`**: For inputs that have a default or computed value, a function that returns that value.

    Let's look at specific `subBlocks`:
    *   **`metrics`**: Custom input for defining evaluation metrics.
    *   **`content`**: Text input for the content to be evaluated.
    *   **`model`**: A combobox for selecting an AI model.
        *   Its `options` function dynamically gathers models from Ollama, OpenRouter, and base providers, ensuring a comprehensive list. It also retrieves and adds provider icons.
    *   **`apiKey`**: Input for an API key.
        *   **`condition`**: This is dynamic:
            *   If `isHosted` (the app is in the cloud): The `apiKey` field is shown if the selected `model` is *not* one of the `getHostedModels()`. This means hosted models don't need a user-provided key.
            *   If *not* `isHosted` (app is local): The `apiKey` field is shown if the selected `model` is *not* one of the `getCurrentOllamaModels()` (Ollama models typically run locally and don't need external API keys).
    *   **`azureEndpoint`, `azureApiVersion`**: Inputs specific to Azure OpenAI.
        *   **`condition`**: Both are shown only if the selected `model` is one of the Azure OpenAI models listed in `providers['azure-openai'].models`.
    *   **`temperature`**: A slider for model temperature, `hidden: true` suggests it's internally controlled or less relevant for evaluation.
    *   **`systemPrompt`**: This is the most crucial `subBlock` for the AI interaction.
        *   **`hidden: true`**: This field is not visible to the user; its value is generated programmatically.
        *   **`value: (params: Record<string, any>) => { ... }`**: The core logic where the AI prompt and response format are generated.
            *   It receives `params` (the current values of other `subBlocks`).
            *   It retrieves `metrics` and `content`.
            *   It safely processes `content` by stringifying it if it's an object.
            *   It calls `generateEvaluatorPrompt` and `generateResponseFormat` (the functions explained earlier) to get the AI instructions.
            *   It creates an object `{ systemPrompt: promptText, responseFormat: responseFormatObj }`. This is a common pattern for passing both the textual prompt and the expected JSON schema to an LLM API.
            *   **`return JSON.stringify(result)`**: It then stringifies this entire object into a JSON string, as the `type: 'code'` field likely expects a string representation of its value.
            *   `try-catch` block: Provides robust error handling, returning a generic fallback prompt and schema if an error occurs during generation.

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
*   **`tools: { ... }`**: Defines which AI "tools" (models/APIs) this block can access and how to configure them.
    *   **`access: [...]`**: An array listing the IDs of specific AI tools/integrations that this block is allowed to use.
    *   **`config: { tool: (params) => { ... } }`**: A function that determines which specific tool to invoke based on the block's parameters.
        *   `const model = params.model || 'gpt-4o'`: Gets the selected model ID from the block's parameters, defaulting to 'gpt-4o' if none is selected.
        *   `if (!model) throw new Error(...)`: Ensures a model is selected.
        *   `const tool = getAllModelProviders()[model as ProviderId]`: Looks up the actual tool configuration object using the selected `model` ID.
        *   `if (!tool) throw new Error(...)`: Throws an error if the selected model is not found or supported.
        *   `return tool`: Returns the configuration object for the chosen AI tool.

```typescript
  inputs: {
    metrics: {
      type: 'json' as ParamType,
      description: 'Evaluation metrics configuration',
      schema: {
        type: 'array',
        properties: {},
        items: {
          type: 'object',
          properties: {
            name: {
              type: 'string',
              description: 'Name of the metric',
            },
            description: {
              type: 'string',
              description: 'Description of what this metric measures',
            },
            range: {
              type: 'object',
              properties: {
                min: {
                  type: 'number',
                  description: 'Minimum possible score',
                },
                max: {
                  type: 'number',
                  description: 'Maximum possible score',
                },
              },
              required: ['min', 'max'],
            },
          },
          required: ['name', 'description', 'range'],
        },
      },
    },
    model: { type: 'string' as ParamType, description: 'AI model to use' },
    apiKey: { type: 'string' as ParamType, description: 'Provider API key' },
    azureEndpoint: { type: 'string' as ParamType, description: 'Azure OpenAI endpoint URL' },
    azureApiVersion: { type: 'string' as ParamType, description: 'Azure API version' },
    temperature: {
      type: 'number' as ParamType,
      description: 'Response randomness level (low for consistent evaluation)',
    },
    content: { type: 'string' as ParamType, description: 'Content to evaluate' },
  },
```
*   **`inputs: { ... }`**: Defines the *expected data inputs* for the block at runtime, along with their types and descriptions. This is more about the data contract than the UI fields.
    *   **`metrics: { type: 'json' as ParamType, description: '...', schema: { ... } }`**: Defines the `metrics` input.
        *   `type: 'json'`: Specifies that this input expects a JSON object/array.
        *   `description`: A helpful explanation.
        *   `schema: { ... }`: A detailed JSON Schema validating the structure of the `metrics` array, ensuring each `item` (metric) is an object with `name`, `description`, and a `range` (with `min` and `max`). This schema corresponds directly to the `Metric` interface defined earlier.
    *   **`model`, `apiKey`, `azureEndpoint`, `azureApiVersion`, `temperature`, `content`**: These define the other expected inputs, mirroring the `subBlocks` for configuration but focusing on their data types (`string`, `number`).

```typescript
  outputs: {
    content: { type: 'string', description: 'Evaluation results' },
    model: { type: 'string', description: 'Model used' },
    tokens: { type: 'json', description: 'Token usage' },
    cost: { type: 'json', description: 'Cost information' },
  } as any,
}
```
*   **`outputs: { ... } as any`**: Defines the *data outputs* that this block will produce. These match the structure of the `EvaluatorResponse` interface.
    *   **`content: { type: 'string', description: 'Evaluation results' }`**: The main evaluation result, which will be a string (likely stringified JSON).
    *   **`model: { type: 'string', description: 'Model used' }`**: The name of the model used.
    *   **`tokens: { type: 'json', description: 'Token usage' }`**: JSON data about token usage.
    *   **`cost: { type: 'json', description: 'Cost information' }`**: JSON data about the cost of the operation.
    *   **`as any`**: This type assertion is used to bypass strict type checking for the `outputs` object. It implies that the actual output structure might be more dynamic than what TypeScript can easily capture statically (e.g., due to the `[metricName: string]: any` in `EvaluatorResponse.output`).

---

This file, in essence, is a highly configurable and robust definition for an AI-powered evaluation step within a larger application, ensuring both a good user experience and precise control over the AI's behavior.