This file is a **Vitest test suite** for a collection of utility functions related to managing AI models, providers, and their capabilities within a TypeScript application. It ensures that these core functions work as expected under various conditions, covering API key retrieval, model features (like temperature support and tool usage), cost calculation, provider management, JSON parsing, structured output generation, and tool preparation.

By systematically testing each utility function, the developers gain confidence that changes to the underlying model configurations or logic won't introduce regressions, and that the application interacts correctly with different AI providers.

---

## Detailed Explanation of the Code

### Imports

```typescript
import { afterEach, beforeEach, describe, expect, it, vi } from 'vitest'
import * as environmentModule from '@/lib/environment'
import {
  calculateCost,
  extractAndParseJSON,
  formatCost,
  generateStructuredOutputInstructions,
  getAllModelProviders,
  getAllModels,
  getAllProviderIds,
  getApiKey,
  getBaseModelProviders,
  getCustomTools,
  getHostedModels,
  getMaxTemperature,
  getProvider,
  getProviderConfigFromModel,
  getProviderFromModel,
  getProviderModels,
  MODELS_TEMP_RANGE_0_1,
  MODELS_TEMP_RANGE_0_2,
  MODELS_WITH_REASONING_EFFORT,
  MODELS_WITH_TEMPERATURE_SUPPORT,
  MODELS_WITH_VERBOSITY,
  PROVIDERS_WITH_TOOL_USAGE_CONTROL,
  prepareToolsWithUsageControl,
  supportsTemperature,
  supportsToolUsageControl,
  transformCustomTool,
  updateOllamaProviderModels,
} from '@/providers/utils'
```

*   **`vitest` imports:** These are the standard testing utilities from the Vitest framework:
    *   `afterEach`, `beforeEach`: Functions to run code before and after each test.
    *   `describe`: Groups related tests together.
    *   `expect`: Used to make assertions about values.
    *   `it`: Defines an individual test case.
    *   `vi`: Vitest's mocking utility (similar to Jest's `jest`).
*   **`environmentModule`:** Imports all exports from the `environment.ts` file, likely containing functions to determine the application's environment (e.g., `isHosted`).
*   **`@/providers/utils` imports:** This is a long list of functions and constants being imported from the `utils.ts` file located in the `providers` directory. These are the core utility functions that this test file is designed to validate. They cover various aspects like:
    *   Model capabilities (`supportsTemperature`, `getMaxTemperature`, `MODELS_TEMP_RANGE_0_1`, etc.)
    *   Provider management (`getProvider`, `getProviderFromModel`, `getAllModels`, etc.)
    *   API key handling (`getApiKey`)
    *   Cost calculation (`calculateCost`, `formatCost`)
    *   JSON and structured output (`extractAndParseJSON`, `generateStructuredOutputInstructions`)
    *   Tool management (`transformCustomTool`, `prepareToolsWithUsageControl`)

---

### Global Mocks and Setup

```typescript
const isHostedSpy = vi.spyOn(environmentModule, 'isHosted', 'get')
const mockGetRotatingApiKey = vi.fn().mockReturnValue('rotating-server-key')
const originalRequire = module.require
```

*   **`isHostedSpy`**:
    *   `vi.spyOn(environmentModule, 'isHosted', 'get')` creates a spy on the `isHosted` property (which is likely a getter function or a variable) within the `environmentModule`.
    *   A spy allows us to monitor calls to `isHosted` and, more importantly, `mock` its return value without changing its original implementation for other parts of the application. This is crucial for testing different environment scenarios (hosted vs. non-hosted).
*   **`mockGetRotatingApiKey`**:
    *   `vi.fn()` creates a mock function.
    *   `.mockReturnValue('rotating-server-key')` configures this mock function to always return the string `'rotating-server-key'` when called. This simulates a secure key rotation service.
*   **`originalRequire`**:
    *   `module.require` refers to Node.js's native `require` function.
    *   It's being saved to `originalRequire` so it can be restored after tests, preventing side effects from mocking `module.require`. This is a common pattern when testing code that might dynamically `require` modules.

---

### `describe('getApiKey', ...)`

This block contains tests for the `getApiKey` function, which is responsible for retrieving the appropriate API key for a given provider and model, taking into account the application's hosting environment and user-provided keys.

```typescript
describe('getApiKey', () => {
  const originalEnv = { ...process.env } // Save original env and reset between tests

  beforeEach(() => {
    vi.clearAllMocks() // Clear any previous mock calls/returns
    isHostedSpy.mockReturnValue(false) // Default to non-hosted environment for most tests
    module.require = vi.fn(() => ({ // Mock module.require to simulate a dynamic import for rotating keys
      getRotatingApiKey: mockGetRotatingApiKey,
    }))
  })

  afterEach(() => {
    process.env = { ...originalEnv } // Restore original environment variables
    module.require = originalRequire // Restore original module.require
  })
  // ... tests below ...
})
```

*   **`originalEnv`**: Stores a copy of `process.env` before any tests run, so it can be restored later. This prevents tests from polluting the environment for subsequent tests.
*   **`beforeEach`**:
    *   `vi.clearAllMocks()`: Resets all mock functions (including `isHostedSpy` and `mockGetRotatingApiKey`) before each test, ensuring a clean state.
    *   `isHostedSpy.mockReturnValue(false)`: Sets the default environment for tests within this block to "not hosted."
    *   `module.require = vi.fn(() => ...)`: Overrides Node.js's `require` for the duration of the test. It mocks a scenario where `getApiKey` might try to `require` a module that exposes `getRotatingApiKey`. This mock ensures `getApiKey` can interact with the `mockGetRotatingApiKey` we defined earlier.
*   **`afterEach`**:
    *   `process.env = { ...originalEnv }`: Restores the `process.env` to its state before this test block ran.
    *   `module.require = originalRequire`: Restores the original `module.require` function.

#### Test Cases for `getApiKey`

*   **`it('should return user-provided key when not in hosted environment', ...)`**:
    *   Verifies that if the application is not hosted (`isHostedSpy.mockReturnValue(false)`) and a user provides an API key, that key is returned directly for any provider (OpenAI, Anthropic in this case).
*   **`it('should throw error if no key provided in non-hosted environment', ...)`**:
    *   Ensures that in a non-hosted environment, if `getApiKey` is called without a user-provided key, it throws an error indicating that a key is required.
*   **`it('should fall back to user key in hosted environment if rotation fails', ...)`**:
    *   Simulates a hosted environment (`isHostedSpy.mockReturnValue(true)`).
    *   Mocks `module.require` to *throw an error* when trying to get a rotating key, simulating a failure in the rotating key service.
    *   Verifies that if a user-provided key *is* available, it is used as a fallback.
*   **`it('should throw error in hosted environment if rotation fails and no user key', ...)`**:
    *   Similar to the above, simulates a hosted environment and a rotating key failure.
    *   Verifies that if *no* user-provided key is available, an error is thrown because no key can be retrieved.
*   **`it('should require user key for non-OpenAI/Anthropic providers even in hosted environment', ...)`**:
    *   Tests a special rule: even in a hosted environment, providers other than OpenAI or Anthropic still *require* a user-provided key. This suggests the rotating key service is only for specific providers.
    *   It checks both successful retrieval with a user key and an error when no user key is provided.

---

### `describe('Model Capabilities', ...)`

This block groups tests related to what specific AI models can do.

#### `describe('supportsTemperature', ...)`

Tests the `supportsTemperature` function, which determines if a given model allows the `temperature` parameter (controlling randomness).

*   **`it.concurrent('should return true for models that support temperature', ...)`**:
    *   Iterates through a list of `supportedModels` and asserts that `supportsTemperature` returns `true` for each.
*   **`it.concurrent('should return false for models that do not support temperature', ...)`**:
    *   Iterates through `unsupportedModels` and asserts that `supportsTemperature` returns `false` for each, including models that are explicitly known not to support it and some generic cases.
*   **`it.concurrent('should be case insensitive', ...)`**:
    *   Checks that the function works regardless of the input model's casing (e.g., `'GPT-4O'` vs `'gpt-4o'`).
*   **`it.concurrent('should inherit temperature support from provider for dynamically fetched models', ...)`**:
    *   Tests a scenario where a model (like those from OpenRouter) might not be explicitly listed but its temperature support is inferred from its base provider (e.g., `openrouter/anthropic/claude-3.5-sonnet` should support temperature because Anthropic models generally do).

#### `describe('getMaxTemperature', ...)`

Tests the `getMaxTemperature` function, which returns the maximum allowed value for the `temperature` parameter for a given model.

*   **`it.concurrent('should return 2 for models with temperature range 0-2', ...)`**:
    *   Verifies that models known to have a 0-2 temperature range correctly return `2`.
*   **`it.concurrent('should return 1 for models with temperature range 0-1', ...)`**:
    *   Verifies that models known to have a 0-1 temperature range correctly return `1`.
*   **`it.concurrent('should return undefined for models that do not support temperature', ...)`**:
    *   Asserts that for models that don't support temperature at all, `getMaxTemperature` returns `undefined`.
*   **`it.concurrent('should be case insensitive', ...)`**:
    *   Checks for case insensitivity in model names.
*   **`it.concurrent('should inherit max temperature from provider for dynamically fetched models', ...)`**:
    *   Similar to `supportsTemperature`, this verifies that for dynamically fetched models, the max temperature is correctly inherited or inferred from the provider.

#### `describe('supportsToolUsageControl', ...)`

Tests the `supportsToolUsageControl` function, which checks if a given provider allows fine-grained control over how tools (functions) are used by the model.

*   **`it.concurrent('should return true for providers that support tool usage control', ...)`**:
    *   Asserts that known providers supporting this feature return `true`.
*   **`it.concurrent('should return false for providers that do not support tool usage control', ...)`**:
    *   Asserts that providers not supporting this feature (or non-existent ones) return `false`.

#### `describe('Model Constants', ...)`

This block directly tests the contents and consistency of various constant arrays (sets of model IDs) defined in `providers/utils.ts`.

*   **`it.concurrent('should have correct models in MODELS_TEMP_RANGE_0_2', ...)`**:
    *   Checks if specific models are correctly included or excluded from the `MODELS_TEMP_RANGE_0_2` constant.
*   **`it.concurrent('should have correct models in MODELS_TEMP_RANGE_0_1', ...)`**:
    *   Checks inclusions/exclusions for `MODELS_TEMP_RANGE_0_1`.
*   **`it.concurrent('should have correct providers in PROVIDERS_WITH_TOOL_USAGE_CONTROL', ...)`**:
    *   Checks inclusions/exclusions for `PROVIDERS_WITH_TOOL_USAGE_CONTROL`.
*   **`it.concurrent('should combine both temperature ranges in MODELS_WITH_TEMPERATURE_SUPPORT', ...)`**:
    *   Ensures that the `MODELS_WITH_TEMPERATURE_SUPPORT` constant correctly combines all models from both 0-1 and 0-2 temperature ranges and has the expected total count.
*   **`it.concurrent('should have correct models in MODELS_WITH_REASONING_EFFORT', ...)`**:
    *   Verifies which models (specifically certain GPT-5 variants) are designated as supporting "reasoning effort." It also checks for exclusions.
*   **`it.concurrent('should have correct models in MODELS_WITH_VERBOSITY', ...)`**:
    *   Similar to the above, verifies which models support "verbosity."
*   **`it.concurrent('should have same models in both reasoning effort and verbosity arrays', ...)`**:
    *   Asserts that the sets of models supporting reasoning effort and verbosity are identical, implying these two capabilities go hand-in-hand for the designated models.

---

### `describe('Cost Calculation', ...)`

This block tests functions related to calculating and formatting costs associated with model usage.

#### `describe('calculateCost', ...)`

Tests the `calculateCost` function, which determines the cost based on the model used, input/output token counts, and whether cached pricing applies.

*   **`it.concurrent('should calculate cost correctly for known models', ...)`**:
    *   Calls `calculateCost` with a known model (`'gpt-4o'`) and token counts.
    *   Asserts that the calculated input, output, and total costs are positive, the total is the sum of input and output, and the `pricing` object is defined with the expected input cost per million tokens.
*   **`it.concurrent('should handle cached input pricing when enabled', ...)`**:
    *   Compares the cost calculation with `isCached = false` vs. `isCached = true`.
    *   Asserts that cached input cost is less than regular input cost, while output cost remains the same.
*   **`it.concurrent('should return default pricing for unknown models', ...)`**:
    *   Tests with an `'unknown-model'`.
    *   Asserts that input, output, and total costs are `0`, and the `pricing` object returns a default input cost (likely a placeholder indicating it's unknown).
*   **`it.concurrent('should handle zero tokens', ...)`**:
    *   Verifies that if both input and output tokens are `0`, the calculated costs are also `0`.

#### `describe('formatCost', ...)`

Tests the `formatCost` function, which takes a numerical cost and formats it into a user-friendly string (e.g., "$1.23", "$0.0023").

*   **`it.concurrent('should format costs >= $1 with two decimal places', ...)`**:
    *   Tests rounding and formatting for costs greater than or equal to one dollar.
*   **`it.concurrent('should format costs between 1¢ and $1 with three decimal places', ...)`**:
    *   Tests for costs between $0.01 and $1, formatting to three decimal places.
*   **`it.concurrent('should format costs between 0.1¢ and 1¢ with four decimal places', ...)`**:
    *   Tests for very small costs, formatting to four decimal places.
*   **`it.concurrent('should format very small costs with appropriate precision', ...)`**:
    *   Covers even smaller costs, ensuring appropriate (more dynamic) precision.
*   **`it.concurrent('should handle zero cost', ...)`**:
    *   Asserts that `0` is formatted as `'$0'`.
*   **`it.concurrent('should handle undefined/null costs', ...)`**:
    *   Asserts that `undefined` or `null` costs are formatted as `'—'` (an em dash), indicating unknown or unavailable cost.

---

### `describe('getHostedModels', ...)`

This block tests the `getHostedModels` function, which returns a list of AI models specifically designated as "hosted" by the application (implying they might be managed directly or have specific internal integrations).

*   **`it.concurrent('should return OpenAI and Anthropic models as hosted', ...)`**:
    *   Asserts that specific OpenAI and Anthropic models are included in the list of hosted models, and models from other providers are *not*.
*   **`it.concurrent('should return an array of strings', ...)`**:
    *   Verifies that the function returns an array, it's not empty, and all its elements are strings.

---

### `describe('Provider Management', ...)`

This block groups tests for functions that manage and retrieve information about AI providers and their models.

#### `describe('getProviderFromModel', ...)`

Tests the `getProviderFromModel` function, which infers the provider ID from a given model ID.

*   **`it.concurrent('should return correct provider for known models', ...)`**:
    *   Checks if the function correctly identifies the provider for well-known models from different providers (OpenAI, Anthropic, Google, Azure).
*   **`it.concurrent('should use model patterns for pattern matching', ...)`**:
    *   Verifies that the function can identify providers based on patterns in the model name (e.g., any model starting with `gpt` is OpenAI, any starting with `claude` is Anthropic).
*   **`it.concurrent('should default to ollama for unknown models', ...)`**:
    *   Asserts that if a model is not recognized, the default provider assigned is `'ollama'`.
*   **`it.concurrent('should be case insensitive', ...)`**:
    *   Checks that model name casing doesn't affect provider identification.

#### `describe('getProvider', ...)`

Tests the `getProvider` function, which retrieves the full configuration object for a given provider ID.

*   **`it.concurrent('should return provider config for valid provider IDs', ...)`**:
    *   Asserts that `getProvider` returns a defined configuration object for known provider IDs (`'openai'`, `'anthropic'`) and that the `id` and `name` properties are correct.
*   **`it.concurrent('should handle provider/service format', ...)`**:
    *   Tests if the function can correctly parse provider IDs that might include a service suffix (e.g., `'openai/chat'`), still returning the base provider config.
*   **`it.concurrent('should return undefined for invalid provider IDs', ...)`**:
    *   Asserts that an unknown provider ID returns `undefined`.

#### `describe('getProviderConfigFromModel', ...)`

Tests the `getProviderConfigFromModel` function, which directly gets the provider configuration associated with a specific model ID.

*   **`it.concurrent('should return provider config for model', ...)`**:
    *   Verifies that calling this function with a model ID (e.g., `'gpt-4o'`, `'claude-sonnet-4-0'`) returns the correct provider configuration object.

#### `describe('getAllModels', ...)`

Tests the `getAllModels` function, which returns a comprehensive list of all known model IDs across all configured providers.

*   **`it.concurrent('should return all models from all providers', ...)`**:
    *   Asserts that the result is an array, has content, and includes examples of models from different providers.

#### `describe('getAllProviderIds', ...)`

Tests the `getAllProviderIds` function, which returns a list of all configured provider IDs.

*   **`it.concurrent('should return all provider IDs', ...)`**:
    *   Asserts that the result is an array and contains a representative set of provider IDs.

#### `describe('getProviderModels', ...)`

Tests the `getProviderModels` function, which returns a list of models specifically associated with a given provider.

*   **`it.concurrent('should return models for specific providers', ...)`**:
    *   Asserts that for `'openai'` and `'anthropic'`, the returned array contains their respective models.
*   **`it.concurrent('should return empty array for unknown providers', ...)`**:
    *   Verifies that querying models for an unknown provider returns an empty array.

#### `describe('getBaseModelProviders and getAllModelProviders', ...)`

Tests functions that provide mappings from model IDs to their corresponding provider IDs.

*   **`it.concurrent('should return model to provider mapping', ...)`**:
    *   Checks that `getAllModelProviders` returns an object mapping models to providers (e.g., `'gpt-4o'` to `'openai'`).
    *   Also checks `getBaseModelProviders`, noting that it should exclude specific models (like Ollama models, which might be handled differently).

#### `describe('updateOllamaProviderModels', ...)`

Tests the `updateOllamaProviderModels` function, which dynamically updates the list of models available for the `ollama` provider. This is likely used for locally hosted models that can change.

*   **`it.concurrent('should update ollama models', ...)`**:
    *   Calls the function with a mock list of Ollama models.
    *   Asserts that the `getProviderModels('ollama')` then returns this updated list, proving the dynamic update works.

---

### `describe('JSON and Structured Output', ...)`

This block contains tests for functions related to processing JSON data and generating structured output instructions.

#### `describe('extractAndParseJSON', ...)`

Tests the `extractAndParseJSON` function, which attempts to extract and parse a JSON object from a potentially larger string, often handling common formatting issues.

*   **`it.concurrent('should extract and parse valid JSON', ...)`**:
    *   Tests extracting JSON embedded within a markdown code block (` ```json... ` ` `).
*   **`it.concurrent('should extract JSON without code blocks', ...)`**:
    *   Tests extracting bare JSON that is not enclosed in markdown code blocks.
*   **`it.concurrent('should handle nested objects', ...)`**:
    *   Verifies correct parsing of JSON with nested structures.
*   **`it.concurrent('should clean up common JSON issues', ...)`**:
    *   Tests the function's ability to fix common JSON errors, specifically a trailing comma in this case, before parsing.
*   **`it.concurrent('should throw error for content without JSON', ...)`**:
    *   Asserts that if no valid JSON object can be found in the string, an error is thrown.
*   **`it.concurrent('should throw error for invalid JSON', ...)`**:
    *   Asserts that if the extracted text is malformed JSON and cannot be parsed even after cleanup, an error is thrown.

#### `describe('generateStructuredOutputInstructions', ...)`

Tests the `generateStructuredOutputInstructions` function, which creates human-readable instructions for an AI model on how to format its output based on a provided schema or field definition.

*   **`it.concurrent('should return empty string for JSON Schema format', ...)`**:
    *   If the input format is a full JSON Schema (which models might understand natively), no additional instructions are needed, so it should return an empty string.
*   **`it.concurrent('should return empty string for object type with properties', ...)`**:
    *   Similar to above, for a simple object type definition, it returns an empty string.
*   **`it.concurrent('should generate instructions for legacy fields format', ...)`**:
    *   Tests an older, "fields-based" format. It asserts that the function generates a descriptive string that explains the expected JSON structure, including field names and descriptions.
*   **`it.concurrent('should handle object fields with properties', ...)`**:
    *   Tests generating instructions for a field that is itself an object with its own properties.
*   **`it.concurrent('should return empty string for missing fields', ...)`**:
    *   Verifies that if the input format is empty, `null`, or lacks the `fields` property, an empty string is returned, as no instructions can be generated.

---

### `describe('Tool Management', ...)`

This block contains tests for functions related to managing and preparing AI tools (also known as functions or function calling).

#### `describe('transformCustomTool', ...)`

Tests the `transformCustomTool` function, which takes a custom tool definition and transforms it into a standardized format compatible with AI model APIs.

*   **`it.concurrent('should transform valid custom tool schema', ...)`**:
    *   Provides a valid `customTool` object with a `schema.function` definition.
    *   Asserts that the `result` has the correct `id` (prefixed with `custom_`), `name`, `description`, and correctly parsed `parameters` from the schema.
*   **`it.concurrent('should throw error for invalid schema', ...)`**:
    *   Tests with `null` schema or a schema missing the `function` property.
    *   Asserts that these invalid inputs correctly throw an error.

#### `describe('getCustomTools', ...)`

Tests the `getCustomTools` function, which retrieves and transforms all configured custom tools.

*   **`it.concurrent('should return array of transformed custom tools', ...)`**:
    *   Simply checks that the function returns an array, indicating it successfully processed and transformed any custom tools.

#### `describe('prepareToolsWithUsageControl', ...)`

Tests the `prepareToolsWithUsageControl` function, which is critical for dynamic tool management. It takes a list of tools and provider-specific tool usage controls, then filters the tools and determines the appropriate `toolChoice` or `toolConfig` to send to the AI model API.

*   **`mockLogger`**: A mock object with `info`, `warn`, `error` methods. This allows the test to verify if the function logs messages correctly, which is useful for debugging tool filtering.
*   **`beforeEach`**: Clears the mock logger's call history before each test.

*   **`it.concurrent('should return early for no tools', ...)`**:
    *   If no tools or provider tools are provided, the function should return early with `undefined` tools and toolChoice, and `false` for `hasFilteredTools`.
*   **`it.concurrent('should filter out tools with usageControl="none"', ...)`**:
    *   Provides a set of tools and `providerTools` configurations, one of which has `usageControl: 'none'`.
    *   Asserts that the tool with `usageControl: 'none'` is removed from the `tools` array, `hasFilteredTools` is `true`, the correct forced tool is identified, and a logging message is recorded.
*   **`it.concurrent('should set toolChoice for forced tools (OpenAI format)', ...)`**:
    *   If a tool has `usageControl: 'force'`, the `toolChoice` parameter for the OpenAI API should be set to force that specific tool.
*   **`it.concurrent('should set toolChoice for forced tools (Anthropic format)', ...)`**:
    *   Similar to OpenAI, but verifies the specific `toolChoice` format required by Anthropic's API (`type: 'tool'`, `name: '...'`).
*   **`it.concurrent('should set toolConfig for Google format', ...)`**:
    *   Verifies the `toolConfig` format for Google's API, which uses `functionCallingConfig` to specify allowed function names when forcing a tool.
*   **`it.concurrent('should return empty when all tools are filtered', ...)`**:
    *   If all provided tools are marked with `usageControl: 'none'`, the function should return with no tools or `toolChoice`, and `hasFilteredTools` should be `true`.
*   **`it.concurrent('should default to auto when no forced tools', ...)`**:
    *   If there are tools but none are explicitly `force`d, the `toolChoice` should default to `'auto'`, allowing the model to decide whether to use tools.

---