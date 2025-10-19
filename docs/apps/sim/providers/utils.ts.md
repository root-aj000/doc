This TypeScript file acts as a central **Provider Utilities and Configuration Hub** for an application that interacts with various AI models from different providers (OpenAI, Anthropic, Google, etc.).

It consolidates model and provider definitions, offers a unified interface for model-related operations like fetching lists, determining pricing, managing API keys, and handling advanced features such as structured output and sequential tool usage. In essence, it abstracts away the complexities of dealing with multiple AI services, providing a consistent API for the rest of the application.

### Simplified Complex Logic:

The core complexity lies in:
1.  **Consolidating Provider Information:** It takes basic provider configurations and comprehensive model definitions from separate files (`@/providers/anthropic`, `@/providers/models`) and merges them into a single, enriched `providers` object that is easy to access.
2.  **Dynamic Model Discovery:** Not all models are explicitly listed; some providers (like OpenRouter) can host a vast array, and this file uses regular expressions (`modelPatterns`) to identify them on the fly.
3.  **Intelligent API Key Management:** It intelligently decides whether to use a user-provided API key, a server-side rotating key (for hosted environments and specific providers), or no key at all (for local models like Ollama).
4.  **Advanced Tool Usage Control:** The `prepareToolsWithUsageControl` and `trackForcedToolUsage` functions work together to implement a sophisticated mechanism for guiding AI models to use specific tools in a particular order, or preventing them from using certain tools, adapting to provider-specific API formats.
5.  **Robust JSON Parsing:** The `extractAndParseJSON` function is designed to handle "sloppy" JSON output that LLMs sometimes produce, cleaning it up before attempting to parse.

---

### Detailed Code Explanation

#### 1. Imports

This section brings in all the necessary components from other parts of the application.

```typescript
import { isHosted } from '@/lib/environment' // Utility to check if the application is running in a hosted (production) environment.
import { createLogger } from '@/lib/logs/console/logger' // A function to create a logger instance for debugging and error reporting.

// Individual provider configurations (these are base definitions).
import { anthropicProvider } from '@/providers/anthropic'
import { azureOpenAIProvider } from '@/providers/azure-openai'
import { cerebrasProvider } from '@/providers/cerebras'
import { deepseekProvider } from '@/providers/deepseek'
import { googleProvider } from '@/providers/google'
import { groqProvider } from '@/providers/groq'
import { mistralProvider } from '@/providers/mistral'

// Comprehensive model definitions and helper functions from a central models file.
import {
  getComputerUseModels, // Gets models specifically designed for "computer use" tasks.
  getEmbeddingModelPricing, // Retrieves pricing information for embedding models.
  getHostedModels as getHostedModelsFromDefinitions, // Gets models hosted by the platform (aliased to avoid name collision).
  getMaxTemperature as getMaxTempFromDefinitions, // Gets the maximum temperature setting for a model (aliased).
  getModelPricing as getModelPricingFromDefinitions, // Retrieves pricing information for general chat/completion models (aliased).
  getModelsWithReasoningEffort, // Gets models that support a "reasoning effort" parameter.
  getModelsWithTemperatureSupport, // Gets models that support a "temperature" parameter.
  getModelsWithTempRange01, // Gets models with temperature range 0-1.
  getModelsWithTempRange02, // Gets models with temperature range 0-2.
  getModelsWithVerbosity, // Gets models that support a "verbosity" parameter.
  getProviderModels as getProviderModelsFromDefinitions, // Gets a list of models for a specific provider (aliased).
  getProvidersWithToolUsageControl, // Gets providers that support fine-grained tool usage control.
  PROVIDER_DEFINITIONS, // The raw, comprehensive definitions of all providers.
  supportsTemperature as supportsTemperatureFromDefinitions, // Checks if a model supports temperature (aliased).
  supportsToolUsageControl as supportsToolUsageControlFromDefinitions, // Checks if a provider supports tool usage control (aliased).
  updateOllamaModels as updateOllamaModelsInDefinitions, // Function to update Ollama model definitions (aliased).
} from '@/providers/models'

import { ollamaProvider } from '@/providers/ollama' // Ollama provider base configuration.
import { openaiProvider } from '@/providers/openai' // OpenAI provider base configuration.
import { openRouterProvider } from '@/providers/openrouter' // OpenRouter provider base configuration.
import type { ProviderConfig, ProviderId, ProviderToolConfig } from '@/providers/types' // TypeScript type definitions for provider configurations and IDs.
import { xAIProvider } from '@/providers/xai' // xAI provider base configuration.

// Zustand stores for managing global state related to custom tools and provider settings.
import { useCustomToolsStore } from '@/stores/custom-tools/store'
import { useProvidersStore } from '@/stores/providers/store'
```

#### 2. Logger Initialization

```typescript
const logger = createLogger('ProviderUtils') // Creates a logger instance specifically for this utility file, making it easier to track logs originating from here.
```

#### 3. Exported Provider Configurations (`providers`)

This is the main, enriched configuration object that the rest of the application will use. It combines the base provider configurations with model lists and patterns from `PROVIDER_DEFINITIONS`.

```typescript
/**
 * Provider configurations - built from the comprehensive definitions
 */
export const providers: Record<
  ProviderId, // The key for this record is a ProviderId (e.g., 'openai', 'anthropic').
  ProviderConfig & { // Each value is a ProviderConfig (from its base definition)
    models: string[] // ...plus a dynamically populated array of model IDs.
    computerUseModels?: string[] // Optional array for models specifically for computer interaction.
    modelPatterns?: RegExp[] // Optional array of regular expressions to match dynamic model names.
  }
> = {
  openai: {
    ...openaiProvider, // Spreads all properties from the base openaiProvider config.
    models: getProviderModelsFromDefinitions('openai'), // Populates the 'models' array by calling a helper function from `models.ts`.
    computerUseModels: ['computer-use-preview'], // Specific computer-use models for OpenAI.
    modelPatterns: PROVIDER_DEFINITIONS.openai.modelPatterns, // Adds model patterns for OpenAI from the central definitions.
  },
  anthropic: {
    ...anthropicProvider,
    models: getProviderModelsFromDefinitions('anthropic'),
    computerUseModels: getComputerUseModels().filter((model) => // Filters general computer-use models to only include those available to Anthropic.
      getProviderModelsFromDefinitions('anthropic').includes(model)
    ),
    modelPatterns: PROVIDER_DEFINITIONS.anthropic.modelPatterns,
  },
  google: {
    ...googleProvider,
    models: getProviderModelsFromDefinitions('google'),
    modelPatterns: PROVIDER_DEFINITIONS.google.modelPatterns,
  },
  deepseek: {
    ...deepseekProvider,
    models: getProviderModelsFromDefinitions('deepseek'),
    modelPatterns: PROVIDER_DEFINITIONS.deepseek.modelPatterns,
  },
  xai: {
    ...xAIProvider,
    models: getProviderModelsFromDefinitions('xai'),
    modelPatterns: PROVIDER_DEFINITIONS.xai.modelPatterns,
  },
  cerebras: {
    ...cerebrasProvider,
    models: getProviderModelsFromDefinitions('cerebras'),
    modelPatterns: PROVIDER_DEFINITIONS.cerebras.modelPatterns,
  },
  groq: {
    ...groqProvider,
    models: getProviderModelsFromDefinitions('groq'),
    modelPatterns: PROVIDER_DEFINITIONS.groq.modelPatterns,
  },
  mistral: {
    ...mistralProvider,
    models: getProviderModelsFromDefinitions('mistral'),
    modelPatterns: PROVIDER_DEFINITIONS.mistral.modelPatterns,
  },
  'azure-openai': {
    ...azureOpenAIProvider,
    models: getProviderModelsFromDefinitions('azure-openai'),
    modelPatterns: PROVIDER_DEFINITIONS['azure-openai'].modelPatterns,
  },
  openrouter: {
    ...openRouterProvider,
    models: getProviderModelsFromDefinitions('openrouter'),
    modelPatterns: PROVIDER_DEFINITIONS.openrouter.modelPatterns,
  },
  ollama: {
    ...ollamaProvider,
    models: getProviderModelsFromDefinitions('ollama'),
    modelPatterns: PROVIDER_DEFINITIONS.ollama.modelPatterns,
  },
}
```

#### 4. Provider Initialization Loop

Some providers might require an asynchronous setup when the application starts (e.g., fetching a list of available models from an API). This loop handles that.

```typescript
Object.entries(providers).forEach(([id, provider]) => { // Iterates over each provider in the `providers` object.
  if (provider.initialize) { // Checks if the current provider has an `initialize` method defined.
    provider.initialize().catch((error) => { // Calls the `initialize` method, catching any potential errors during setup.
      logger.error(`Failed to initialize ${id} provider`, { // Logs an error if initialization fails.
        error: error instanceof Error ? error.message : 'Unknown error', // Provides a clear error message.
      })
    })
  }
})
```

#### 5. Model Update Functions

These functions allow dynamic updates to the lists of models for providers like Ollama and OpenRouter, whose available models might change at runtime (e.g., locally running Ollama models, or OpenRouter fetching new ones).

```typescript
export function updateOllamaProviderModels(models: string[]): void {
  updateOllamaModelsInDefinitions(models) // Updates the central Ollama model definitions in `models.ts`.
  providers.ollama.models = getProviderModelsFromDefinitions('ollama') // Re-reads the updated models and assigns them to the `providers.ollama` configuration in this file.
}

export async function updateOpenRouterProviderModels(models: string[]): Promise<void> {
  // Dynamically imports the `updateOpenRouterModels` function. This might be done to keep the initial bundle smaller or if it's conditionally needed.
  const { updateOpenRouterModels } = await import('@/providers/models')
  updateOpenRouterModels(models) // Updates the central OpenRouter model definitions.
  providers.openrouter.models = getProviderModelsFromDefinitions('openrouter') // Re-reads the updated models.
}
```

#### 6. Model-to-Provider Mapping Functions

These utilities help quickly determine which provider an AI model belongs to.

```typescript
export function getBaseModelProviders(): Record<string, ProviderId> {
  // Creates a map of model names (lowercase) to their ProviderId, excluding 'ollama'.
  return Object.entries(providers)
    .filter(([providerId]) => providerId !== 'ollama') // Filters out Ollama models, likely because they behave differently or are local.
    .reduce(
      (map, [providerId, config]) => {
        config.models.forEach((model) => {
          map[model.toLowerCase()] = providerId as ProviderId // Adds each model to the map, using lowercase for case-insensitive lookup.
        })
        return map
      },
      {} as Record<string, ProviderId> // Initializes an empty object for the map.
    )
}

export function getAllModelProviders(): Record<string, ProviderId> {
  // Similar to `getBaseModelProviders`, but includes all providers, including Ollama.
  return Object.entries(providers).reduce(
    (map, [providerId, config]) => {
      config.models.forEach((model) => {
        map[model.toLowerCase()] = providerId as ProviderId
      })
      return map
    },
    {} as Record<string, ProviderId>
  )
}

export function getProviderFromModel(model: string): ProviderId {
  const normalizedModel = model.toLowerCase() // Normalizes the model name to lowercase for consistent lookup.

  if (normalizedModel in getAllModelProviders()) {
    return getAllModelProviders()[normalizedModel] // First, checks for an exact match in the full model-to-provider map.
  }

  for (const [providerId, config] of Object.entries(providers)) {
    if (config.modelPatterns) { // If a provider has model patterns (Regular Expressions)...
      for (const pattern of config.modelPatterns) { // ...iterate through each pattern.
        if (pattern.test(normalizedModel)) { // If the model name matches a pattern...
          return providerId as ProviderId // ...return that provider. This is crucial for dynamic model naming schemes (e.g., OpenRouter).
        }
      }
    }
  }

  logger.warn(`No provider found for model: ${model}, defaulting to ollama`) // If no provider is found after exact and pattern matching, log a warning.
  return 'ollama' // Defaults to 'ollama', perhaps as a safe fallback for unknown local models.
}

export function getProvider(id: string): ProviderConfig | undefined {
  // Retrieves a provider's configuration by its ID.
  // Handle both formats: 'openai' and 'openai/chat' - splits by '/' to get the base ID.
  const providerId = id.split('/')[0] as ProviderId
  return providers[providerId] // Returns the configuration from the `providers` object.
}

export function getProviderConfigFromModel(model: string): ProviderConfig | undefined {
  const providerId = getProviderFromModel(model) // First finds the provider ID using `getProviderFromModel`.
  return providers[providerId] // Then returns the corresponding provider configuration.
}
```

#### 7. General Provider & Model Accessors

Simple functions to get lists of models or provider IDs.

```typescript
export function getAllModels(): string[] {
  // Returns a flattened array of all model IDs across all providers.
  return Object.values(providers).flatMap((provider) => provider.models || [])
}

export function getAllProviderIds(): ProviderId[] {
  // Returns an array of all available ProviderIds.
  return Object.keys(providers) as ProviderId[]
}

export function getProviderModels(providerId: ProviderId): string[] {
  // Returns the list of models for a specific provider, directly from the definitions.
  return getProviderModelsFromDefinitions(providerId)
}

/**
 * Get provider icon for a given model
 */
export function getProviderIcon(model: string): React.ComponentType<{ className?: string }> | null {
  const providerId = getProviderFromModel(model) // Determines the provider for the given model.
  return PROVIDER_DEFINITIONS[providerId]?.icon || null // Retrieves the icon component from the central `PROVIDER_DEFINITIONS`.
}
```

#### 8. Structured Output & JSON Parsing

These functions help AI models produce structured data and robustly parse it from their responses.

```typescript
export function generateStructuredOutputInstructions(responseFormat: any): string {
  // Generates instructions for an LLM to produce output in a specific JSON format.
  if (!responseFormat) return '' // Handles null/undefined input.

  // If using the new JSON Schema format, don't add additional instructions
  // This is necessary because providers now handle the schema directly
  if (responseFormat.schema || (responseFormat.type === 'object' && responseFormat.properties)) {
    return '' // If the response format is already a full JSON Schema, no extra instructions are needed as the model understands it.
  }

  // Handle legacy format with fields array
  if (!responseFormat.fields) return '' // If it's not a new schema and has no legacy 'fields', return empty.

  function generateFieldStructure(field: any): string {
    // Helper function to build an example JSON structure for a given field.
    if (field.type === 'object' && field.properties) {
      // If the field is an object, recursively generate its properties.
      return `{
    ${Object.entries(field.properties)
      .map(([key, prop]: [string, any]) => `"${key}": ${prop.type === 'number' ? '0' : '"value"'}`) // Example values based on type.
      .join(',\n    ')}
  }`
    }
    // Handles basic types.
    return field.type === 'string'
      ? '"value"'
      : field.type === 'number'
        ? '0'
        : field.type === 'boolean'
          ? 'true/false'
          : '[]'
  }

  const exampleFormat = responseFormat.fields
    .map((field: any) => `  "${field.name}": ${generateFieldStructure(field)}`) // Creates the example JSON structure.
    .join(',\n')

  const fieldDescriptions = responseFormat.fields
    .map((field: any) => {
      let desc = `${field.name} (${field.type})` // Basic description: name (type).
      if (field.description) desc += `: ${field.description}` // Adds detailed description if available.
      if (field.type === 'object' && field.properties) {
        // If an object, lists its properties and their descriptions.
        desc += '\nProperties:'
        Object.entries(field.properties).forEach(([key, prop]: [string, any]) => {
          desc += `\n  - ${key} (${(prop as any).type}): ${(prop as any).description || ''}`
        })
      }
      return desc
    })
    .join('\n')

  // Returns the full instruction string, including example JSON, field descriptions, and a directive for valid JSON.
  return `
Please provide your response in the following JSON format:
{
${exampleFormat}
}

Field descriptions:
${fieldDescriptions}

Your response MUST be valid JSON and include all the specified fields with their correct types.
Each metric should be an object containing 'score' (number) and 'reasoning' (string).`
}

export function extractAndParseJSON(content: string): any {
  // Robustly extracts and parses JSON from a string, designed to handle imperfect LLM output.
  const trimmed = content.trim() // Removes leading/trailing whitespace.

  const firstBrace = trimmed.indexOf('{') // Finds the first opening curly brace.
  const lastBrace = trimmed.lastIndexOf('}') // Finds the last closing curly brace.

  if (firstBrace === -1 || lastBrace === -1) {
    throw new Error('No JSON object found in content') // If no braces are found, it's not JSON.
  }

  const jsonStr = trimmed.slice(firstBrace, lastBrace + 1) // Extracts the substring between the first '{' and last '}'.

  try {
    return JSON.parse(jsonStr) // First attempt to parse the extracted JSON.
  } catch (_error) {
    // If the first parse fails, try to clean up common issues in LLM output.
    const cleaned = jsonStr
      .replace(/\n/g, ' ') // Replaces newlines with spaces.
      .replace(/\s+/g, ' ') // Normalizes all whitespace to single spaces.
      .replace(/,\s*([}\]])/g, '$1') // Removes trailing commas before '}' or ']' (a very common LLM error).

    try {
      return JSON.parse(cleaned) // Second attempt to parse after cleanup.
    } catch (innerError) {
      logger.error('Failed to parse JSON response', { // If it still fails, log detailed context.
        contentLength: content.length,
        extractedLength: jsonStr.length,
        cleanedLength: cleaned.length,
        error: innerError instanceof Error ? innerError.message : 'Unknown error',
      })

      throw new Error(
        `Failed to parse JSON after cleanup: ${innerError instanceof Error ? innerError.message : 'Unknown error'}`
      ) // Re-throws with a descriptive error.
    }
  }
}
```

#### 9. Tool Management Functions

These functions handle custom tools defined by users and "block" tools used in the application's UI, converting them into a format (`ProviderToolConfig`) that AI models can understand for function calling.

```typescript
/**
 * Transforms a custom tool schema into a provider tool config
 */
export function transformCustomTool(customTool: any): ProviderToolConfig {
  const schema = customTool.schema // Gets the schema definition from the custom tool.

  if (!schema || !schema.function) {
    throw new Error('Invalid custom tool schema') // Ensures the tool has a valid function schema.
  }

  return {
    id: `custom_${customTool.id}`, // Prefixes the ID to clearly identify it as a custom tool.
    name: schema.function.name, // Extracts the tool's name.
    description: schema.function.description || '', // Extracts the tool's description.
    params: {}, // Placeholder; actual params are handled later during execution.
    parameters: {
      // Extracts the JSON Schema parameters for the tool's function.
      type: schema.function.parameters.type,
      properties: schema.function.parameters.properties,
      required: schema.function.parameters.required || [],
    },
  }
}

/**
 * Gets all available custom tools as provider tool configs
 */
export function getCustomTools(): ProviderToolConfig[] {
  const customTools = useCustomToolsStore.getState().getAllTools() // Retrieves all custom tools from the global Zustand store.
  return customTools.map(transformCustomTool) // Transforms each raw custom tool into a `ProviderToolConfig`.
}

/**
 * Transforms a block tool into a provider tool config with operation selection
 *
 * @param block The block to transform
 * @param options Additional options including dependencies and selected operation
 * @returns The provider tool config or null if transform fails
 */
export async function transformBlockTool(
  block: any,
  options: {
    selectedOperation?: string // The specific operation selected for blocks with multiple tools.
    getAllBlocks: () => any[] // Helper function to get all block definitions.
    getTool: (toolId: string) => any // Helper function to get a tool by ID synchronously.
    getToolAsync?: (toolId: string) => Promise<any> // Optional helper for asynchronous tool fetching (e.g., custom tools).
  }
): Promise<ProviderToolConfig | null> {
  const { selectedOperation, getAllBlocks, getTool, getToolAsync } = options

  const blockDef = getAllBlocks().find((b: any) => b.type === block.type) // Finds the definition for the given block type.
  if (!blockDef) {
    logger.warn(`Block definition not found for type: ${block.type}`)
    return null
  }

  let toolId: string | null = null // Variable to store the ID of the specific tool to be used.

  if ((blockDef.tools?.access?.length || 0) > 1) {
    // If the block definition has multiple tools (operations).
    if (selectedOperation && blockDef.tools?.config?.tool) {
      // If an operation is selected and the block has a tool selection function.
      try {
        toolId = blockDef.tools.config.tool({ // Uses the block's own logic to determine the tool ID based on parameters and selected operation.
          ...block.params,
          operation: selectedOperation,
        })
      } catch (error) {
        logger.error('Error selecting tool for block', { blockType: block.type, operation: selectedOperation, error })
        return null
      }
    } else {
      toolId = blockDef.tools.access[0] // Defaults to the first tool if no specific operation is selected.
    }
  } else {
    toolId = blockDef.tools?.access?.[0] || null // For blocks with a single tool, directly get its ID.
  }

  if (!toolId) {
    logger.warn(`No tool ID found for block: ${block.type}`)
    return null
  }

  let toolConfig: any
  if (toolId.startsWith('custom_') && getToolAsync) {
    // If it's a custom tool, use the asynchronous getter.
    toolConfig = await getToolAsync(toolId)
  } else {
    toolConfig = getTool(toolId) // Otherwise, use the synchronous getter for built-in tools.
  }

  if (!toolConfig) {
    logger.warn(`Tool config not found for ID: ${toolId}`)
    return null
  }

  const { createLLMToolSchema } = await import('@/tools/params') // Dynamically imports a utility to create an LLM tool schema.

  const userProvidedParams = block.params || {} // Gets parameters provided by the user in the block's configuration.

  // Creates the LLM-specific schema, intelligently excluding parameters already provided by the user.
  // This means the LLM won't try to ask for parameters the user has already specified.
  const llmSchema = createLLMToolSchema(toolConfig, userProvidedParams)

  return {
    id: toolConfig.id,
    name: toolConfig.name,
    description: toolConfig.description,
    params: userProvidedParams, // Stores user-provided parameters directly.
    parameters: llmSchema, // The LLM-ready schema.
  }
}
```

#### 10. Cost Calculation & Formatting

Functions for calculating and displaying the cost of AI model usage.

```typescript
/**
 * Calculate cost for token usage based on model pricing
 */
export function calculateCost(
  model: string,
  promptTokens = 0,
  completionTokens = 0,
  useCachedInput = false,
  inputMultiplier?: number,
  outputMultiplier?: number
) {
  let pricing = getEmbeddingModelPricing(model) // First, try to get pricing for embedding models.

  if (!pricing) {
    pricing = getModelPricingFromDefinitions(model) // If not an embedding model, get pricing for chat/completion models.
  }

  if (!pricing) {
    // If no pricing is found, return a default zero-cost result with default pricing info.
    const defaultPricing = { input: 1.0, cachedInput: 0.5, output: 5.0, updatedAt: '2025-03-21' }
    return { input: 0, output: 0, total: 0, pricing: defaultPricing }
  }

  // Calculate costs in USD. Pricing values are "per million tokens", so divide by 1,000,000.
  const inputCost =
    promptTokens *
    (useCachedInput && pricing.cachedInput // If using cached input and cached pricing exists, use it.
      ? pricing.cachedInput / 1_000_000
      : pricing.input / 1_000_000)

  const outputCost = completionTokens * (pricing.output / 1_000_000) // Calculate output token cost.
  const finalInputCost = inputCost * (inputMultiplier ?? 1) // Apply optional input cost multiplier.
  const finalOutputCost = outputCost * (outputMultiplier ?? 1) // Apply optional output cost multiplier.
  const finalTotalCost = finalInputCost + finalOutputCost // Total cost.

  return {
    input: Number.parseFloat(finalInputCost.toFixed(8)), // Format costs to 8 decimal places for precision, especially with very small numbers.
    output: Number.parseFloat(finalOutputCost.toFixed(8)),
    total: Number.parseFloat(finalTotalCost.toFixed(8)),
    pricing, // Include the pricing information used.
  }
}

/**
 * Get pricing information for a specific model (including embedding models)
 */
export function getModelPricing(modelId: string): any {
  const embeddingPricing = getEmbeddingModelPricing(modelId) // Checks for embedding model pricing first.
  if (embeddingPricing) {
    return embeddingPricing
  }
  return getModelPricingFromDefinitions(modelId) // Otherwise, returns general model pricing.
}

/**
 * Format cost as a currency string
 */
export function formatCost(cost: number): string {
  if (cost === undefined || cost === null) return '—' // Handles null/undefined costs.

  // Formats the cost with varying decimal places based on its magnitude for better readability.
  if (cost >= 1) {
    return `$${cost.toFixed(2)}`
  }
  if (cost >= 0.01) {
    return `$${cost.toFixed(3)}`
  }
  if (cost >= 0.001) {
    return `$${cost.toFixed(4)}`
  }
  if (cost > 0) {
    const places = Math.max(4, Math.abs(Math.floor(Math.log10(cost))) + 3) // Dynamically adjusts decimal places for very small numbers.
    return `$${cost.toFixed(places)}`
  }
  return '$0' // If cost is 0, display '$0'.
}
```

#### 11. API Key and Environment Management

Manages how API keys are retrieved and whether models are platform-hosted or user-billed.

```typescript
/**
 * Get the list of models that are hosted by the platform (don't require user API keys)
 * These are the models for which we hide the API key field in the hosted environment
 */
export function getHostedModels(): string[] {
  return getHostedModelsFromDefinitions() // Retrieves the list of hosted models from the central definitions.
}

/**
 * Determine if model usage should be billed to the user
 */
export function shouldBillModelUsage(model: string): boolean {
  const hostedModels = getHostedModels()
  // Returns true if the model is in the hosted list, implying the platform pays (or handles billing internally).
  return hostedModels.includes(model)
}

/**
 * Get an API key for a specific provider, handling rotation and fallbacks
 * For use server-side only
 */
export function getApiKey(provider: string, model: string, userProvidedKey?: string): string {
  const hasUserKey = !!userProvidedKey // Checks if a user-provided key exists.

  // Ollama models don't require API keys - they run locally
  const isOllamaModel =
    provider === 'ollama' || useProvidersStore.getState().providers.ollama.models.includes(model)
  if (isOllamaModel) {
    return 'empty' // Ollama uses 'empty' as a placeholder API key.
  }

  // Use server key rotation for all OpenAI models and Anthropic's Claude models on the hosted platform
  const isOpenAIModel = provider === 'openai'
  const isClaudeModel = provider === 'anthropic'

  if (isHosted && (isOpenAIModel || isClaudeModel)) {
    // If in a hosted environment and it's an OpenAI or Claude model.
    try {
      // Import the key rotation function - `require` suggests this is a server-side-only import to prevent bundling client-side.
      const { getRotatingApiKey } = require('@/lib/utils')
      const serverKey = getRotatingApiKey(provider) // Gets a rotating API key from the server-side utility.
      return serverKey
    } catch (_error) {
      if (hasUserKey) {
        return userProvidedKey! // If server key rotation fails, fallback to the user-provided key.
      }
      throw new Error(`No API key available for ${provider} ${model}`) // Otherwise, throw an error.
    }
  }

  // For all other cases, require user-provided key
  if (!hasUserKey) {
    throw new Error(`API key is required for ${provider} ${model}`) // If not hosted/rotating, a user key is mandatory.
  }

  return userProvidedKey! // Returns the user-provided key if all other conditions don't apply.
}
```

#### 12. Advanced Tool Usage Control (Orchestration)

These functions work together to control how LLMs use tools, including filtering, forcing specific tools, and sequential execution.

```typescript
/**
 * Prepares tool configuration for provider requests with consistent tool usage control behavior
 *
 * @param tools Array of tools in provider-specific format (e.g., function definitions for OpenAI)
 * @param providerTools Original tool configurations with usage control settings (e.g., 'none', 'force')
 * @param logger Logger instance to use for logging
 * @param provider Optional provider ID to adjust format for specific providers (e.g., Google, Anthropic)
 * @returns Object with prepared tools and tool_choice settings
 */
export function prepareToolsWithUsageControl(
  tools: any[] | undefined,
  providerTools: any[] | undefined,
  logger: any,
  provider?: string
): {
  tools: any[] | undefined // The filtered list of tools to send to the LLM.
  toolChoice: // The specific `tool_choice` parameter for the LLM API.
    | 'auto'
    | 'none'
    | { type: 'function'; function: { name: string } }
    | { type: 'tool'; name: string }
    | { type: 'any'; any: { model: string; name: string } }
    | undefined
  toolConfig?: { // Specific tool configuration format for Google (Gemini).
    functionCallingConfig: {
      mode: 'AUTO' | 'ANY' | 'NONE'
      allowedFunctionNames?: string[]
    }
  }
  hasFilteredTools: boolean // Indicates if any tools were filtered out.
  forcedTools: string[] // List of tool IDs that were marked as 'force'.
} {
  if (!tools || tools.length === 0) {
    return { tools: undefined, toolChoice: undefined, hasFilteredTools: false, forcedTools: [] } // If no tools, return early.
  }

  // Filter out tools marked with usageControl='none'
  const filteredTools = tools.filter((tool) => {
    const toolId = tool.function?.name || tool.name // Get the tool's identifier.
    const toolConfig = providerTools?.find((t) => t.id === toolId) // Find its corresponding config with usage control.
    return toolConfig?.usageControl !== 'none' // Keep tools not explicitly set to 'none'.
  })

  const hasFilteredTools = filteredTools.length < tools.length // Check if any tools were removed.
  if (hasFilteredTools) {
    logger.info(`Filtered out ${tools.length - filteredTools.length} tools with usageControl='none'`)
  }

  if (filteredTools.length === 0) {
    logger.info('All tools were filtered out due to usageControl="none"')
    return { tools: undefined, toolChoice: undefined, hasFilteredTools: true, forcedTools: [] } // If all tools are filtered, return empty.
  }

  // Get all tools that should be forced
  const forcedTools = providerTools?.filter((tool) => tool.usageControl === 'force') || []
  const forcedToolIds = forcedTools.map((tool) => tool.id)

  let toolChoice: any = 'auto' // Default tool_choice to 'auto'.
  let toolConfig: any // For Google's specific format.

  if (forcedTools.length > 0) {
    // If there are tools marked for 'force' usage.
    const forcedTool = forcedTools[0] // Select the first one to force.

    // Adjust `toolChoice` (or `toolConfig`) format based on the provider.
    if (provider === 'anthropic') {
      toolChoice = { type: 'tool', name: forcedTool.id }
    } else if (provider === 'google') {
      toolConfig = {
        functionCallingConfig: {
          mode: 'ANY', // Google's `ANY` mode with `allowedFunctionNames` allows specifying specific tools.
          allowedFunctionNames:
            forcedTools.length === 1 // If only one tool to force, specify just that one.
              ? [forcedTool.id]
              : forcedToolIds, // If multiple, list all for Google's API to consider.
        },
      }
      toolChoice = 'auto' // For Google, `toolChoice` remains 'auto' while `toolConfig` handles the forcing.
    } else {
      toolChoice = { type: 'function', function: { name: forcedTool.id } } // Default OpenAI format.
    }

    logger.info(`Forcing use of tool: ${forcedTool.id}`)
    if (forcedTools.length > 1) {
      logger.info(
        `Multiple tools set to 'force' mode (${forcedToolIds.join(', ')}). Will cycle through them sequentially.`
      )
    }
  } else {
    // No tools are forced, so allow the model to choose automatically.
    toolChoice = 'auto'
    if (provider === 'google') {
      toolConfig = { functionCallingConfig: { mode: 'AUTO' } } // Google's auto mode.
    }
    logger.info('Setting tool_choice to auto - letting model decide which tools to use')
  }

  return { tools: filteredTools, toolChoice, toolConfig, hasFilteredTools, forcedTools: forcedToolIds }
}

/**
 * Checks if a forced tool has been used in a response and manages the tool_choice accordingly
 * This function enables *sequential forcing* of multiple tools.
 *
 * @param toolCallsResponse Array of tool calls in the response from the LLM
 * @param originalToolChoice The `tool_choice` setting originally sent in the request
 * @param logger Logger instance to use for logging
 * @param provider Optional provider ID
 * @param forcedTools Array of all tool IDs that should be forced in sequence
 * @param usedForcedTools Array of tool IDs that have already been used in previous turns
 * @returns Object containing tracking information and next tool choice
 */
export function trackForcedToolUsage(
  toolCallsResponse: any[] | undefined,
  originalToolChoice: any,
  logger: any,
  provider?: string,
  forcedTools: string[] = [],
  usedForcedTools: string[] = []
): {
  hasUsedForcedTool: boolean // True if a forced tool was used in this response.
  usedForcedTools: string[] // All unique forced tools used so far.
  nextToolChoice?: // The adjusted tool_choice for the next request.
    | 'auto'
    | { type: 'function'; function: { name: string } }
    | { type: 'tool'; name: string }
    | { type: 'any'; any: { model: string; name: string } }
    | null
  nextToolConfig?: { // The adjusted toolConfig for Google.
    functionCallingConfig: {
      mode: 'AUTO' | 'ANY' | 'NONE'
      allowedFunctionNames?: string[]
    }
  }
} {
  let hasUsedForcedTool = false
  let nextToolChoice = originalToolChoice // By default, keep the original choice.
  let nextToolConfig: any // For Google.

  const updatedUsedForcedTools = [...usedForcedTools] // Copy previous used tools.
  const isGoogleFormat = provider === 'google' // Check for Google's specific format.

  let forcedToolNames: string[] = []
  if (isGoogleFormat && originalToolChoice?.functionCallingConfig?.allowedFunctionNames) {
    forcedToolNames = originalToolChoice.functionCallingConfig.allowedFunctionNames // Get forced tool names from Google's format.
  } else if (
    typeof originalToolChoice === 'object' &&
    (originalToolChoice?.function?.name ||
      (originalToolChoice?.type === 'tool' && originalToolChoice?.name) ||
      (originalToolChoice?.type === 'any' && originalToolChoice?.any?.name))
  ) {
    // Get forced tool names from other providers' formats.
    forcedToolNames = [
      originalToolChoice?.function?.name || originalToolChoice?.name || originalToolChoice?.any?.name,
    ].filter(Boolean)
  }

  // If we were forcing tools and the LLM actually made tool calls.
  if (forcedToolNames.length > 0 && toolCallsResponse && toolCallsResponse.length > 0) {
    const toolNames = toolCallsResponse.map((tc) => tc.function?.name || tc.name || tc.id) // Get names of tools called by the LLM.

    const usedTools = forcedToolNames.filter((toolName) => toolNames.includes(toolName)) // Check which forced tools were actually used.

    if (usedTools.length > 0) {
      // If at least one forced tool was used.
      hasUsedForcedTool = true
      updatedUsedForcedTools.push(...usedTools) // Add them to the list of used forced tools.

      // Find the next tools in the sequence that haven't been used yet.
      const remainingTools = forcedTools.filter((tool) => !updatedUsedForcedTools.includes(tool))

      if (remainingTools.length > 0) {
        // If there are still tools left to force in the sequence.
        const nextToolToForce = remainingTools[0] // Force the next one.

        // Adjust nextToolChoice/nextToolConfig format based on provider.
        if (provider === 'anthropic') {
          nextToolChoice = { type: 'tool', name: nextToolToForce }
        } else if (provider === 'google') {
          nextToolConfig = {
            functionCallingConfig: {
              mode: 'ANY',
              allowedFunctionNames: remainingTools.length === 1 ? [nextToolToForce] : remainingTools,
            },
          }
        } else {
          nextToolChoice = { type: 'function', function: { name: nextToolToForce } }
        }

        logger.info(
          `Forced tool(s) ${usedTools.join(', ')} used, switching to next forced tool(s): ${remainingTools.join(', ')}`
        )
      } else {
        // All forced tools have been used in the sequence.
        if (provider === 'anthropic') {
          nextToolChoice = null // Anthropic needs `null` to remove the parameter.
        } else if (provider === 'google') {
          nextToolConfig = { functionCallingConfig: { mode: 'AUTO' } } // Google switches to 'AUTO'.
        } else {
          nextToolChoice = 'auto' // Others switch to 'auto'.
        }
        logger.info('All forced tools have been used, switching to auto mode for future iterations')
      }
    }
  }

  // Return the results. If no forced tool was used, keep the original `toolChoice`.
  return {
    hasUsedForcedTool,
    usedForcedTools: updatedUsedForcedTools,
    nextToolChoice: hasUsedForcedTool ? nextToolChoice : originalToolChoice,
    nextToolConfig: isGoogleFormat ? (hasUsedForcedTool ? nextToolConfig : originalToolChoice) : undefined,
  }
}
```

#### 13. Model Capability Constants & Functions

Pre-computed lists and functions to quickly check model capabilities.

```typescript
export const MODELS_TEMP_RANGE_0_2 = getModelsWithTempRange02() // List of models supporting temperature from 0 to 2.
export const MODELS_TEMP_RANGE_0_1 = getModelsWithTempRange01() // List of models supporting temperature from 0 to 1.
export const MODELS_WITH_TEMPERATURE_SUPPORT = getModelsWithTemperatureSupport() // List of all models supporting temperature.
export const MODELS_WITH_REASONING_EFFORT = getModelsWithReasoningEffort() // List of models supporting reasoning effort.
export const MODELS_WITH_VERBOSITY = getModelsWithVerbosity() // List of models supporting verbosity.
export const PROVIDERS_WITH_TOOL_USAGE_CONTROL = getProvidersWithToolUsageControl() // List of providers supporting tool usage control.

/**
 * Check if a model supports temperature parameter
 */
export function supportsTemperature(model: string): boolean {
  return supportsTemperatureFromDefinitions(model) // Delegates to the central definition.
}

/**
 * Get the maximum temperature value for a model
 */
export function getMaxTemperature(model: string): number | undefined {
  return getMaxTempFromDefinitions(model) // Delegates to the central definition.
}

/**
 * Check if a provider supports tool usage control
 */
export function supportsToolUsageControl(provider: string): boolean {
  return supportsToolUsageControlFromDefinitions(provider) // Delegates to the central definition.
}
```

#### 14. Tool Execution Preparation

Prepares the arguments for actually executing a tool, separating tool-specific parameters from broader system context parameters.

```typescript
/**
 * Prepare tool execution parameters, separating tool parameters from system parameters
 */
export function prepareToolExecution(
  tool: { params?: Record<string, any> }, // The tool object, potentially with pre-defined parameters.
  llmArgs: Record<string, any>, // Arguments provided by the LLM call for the tool.
  request: { // System-level request context.
    workflowId?: string
    workspaceId?: string
    chatId?: string
    userId?: string
    environmentVariables?: Record<string, any>
    workflowVariables?: Record<string, any>
    blockData?: Record<string, any>
    blockNameMapping?: Record<string, string>
  }
): {
  toolParams: Record<string, any> // Parameters directly for the tool's function.
  executionParams: Record<string, any> // All parameters, including system context, for the tool execution environment.
} {
  // `toolParams` combines parameters from the tool definition and those suggested by the LLM.
  const toolParams = {
    ...tool.params,
    ...llmArgs,
  }

  // `executionParams` adds system-level context parameters, often prefixed with `_context` or `envVars`,
  // which might be needed by the tool's underlying implementation for logging, authorization, or data access.
  const executionParams = {
    ...toolParams, // Start with the tool-specific parameters.
    ...(request.workflowId // Conditionally add workflow context.
      ? {
          _context: {
            workflowId: request.workflowId,
            ...(request.workspaceId ? { workspaceId: request.workspaceId } : {}),
            ...(request.chatId ? { chatId: request.chatId } : {}),
            ...(request.userId ? { userId: request.userId } : {}),
          },
        }
      : {}),
    ...(request.environmentVariables ? { envVars: request.environmentVariables } : {}), // Add environment variables.
    ...(request.workflowVariables ? { workflowVariables: request.workflowVariables } : {}), // Add workflow-specific variables.
    ...(request.blockData ? { blockData: request.blockData } : {}), // Add data associated with blocks.
    ...(request.blockNameMapping ? { blockNameMapping: request.blockNameMapping } : {}), // Add block name mapping.
  }

  return { toolParams, executionParams } // Returns both sets of parameters.
}
```