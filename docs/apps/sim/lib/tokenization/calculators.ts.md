Okay, let's break down this TypeScript code.

**Purpose of this file:**

This file, `cost.ts`, contains functions specifically designed to calculate the cost associated with tokenizing text, primarily for use with Language Model (LLM) APIs.  It provides functions to estimate the cost of using different models based on the number of tokens used for both input (prompt) and output (completion). It uses tokenization to estimate the cost of both streaming and non-streaming LLM executions.

**Simplifying Complex Logic:**

The code takes a complex problem (estimating LLM costs) and breaks it down into smaller, manageable steps:

1.  **Input Validation:** Ensures the provided data is valid before proceeding.
2.  **Provider Identification:** Determines which LLM provider is being used based on the model name.
3.  **Token Estimation:**  Estimates the number of tokens required for both the input prompt and the output completion.  This involves considering various input sources like system prompts, context, user messages, and the main input text.
4.  **Cost Calculation:**  Uses the estimated token counts and the provider's pricing model to calculate the cost.
5.  **Result Formatting:** Structures the results into a well-defined `StreamingCostResult` object, including token usage, cost breakdown, model information, and the method used for calculation.
6.  **Error Handling:** Gracefully handles errors during the calculation process, logging them and potentially throwing custom `TokenizationError` exceptions.

**Line-by-line Explanation:**

```typescript
/**
 * Cost calculation functions for tokenization
 */
```

*   This is a JSDoc comment that describes the purpose of the file.

```typescript
import { createLogger } from '@/lib/logs/console/logger'
import { createTokenizationError } from '@/lib/tokenization/errors'
import {
  estimateInputTokens,
  estimateOutputTokens,
  estimateTokenCount,
} from '@/lib/tokenization/estimators'
import type {
  CostBreakdown,
  StreamingCostResult,
  TokenizationInput,
  TokenUsage,
} from '@/lib/tokenization/types'
import {
  getProviderForTokenization,
  logTokenizationDetails,
  validateTokenizationInput,
} from '@/lib/tokenization/utils'
import { calculateCost } from '@/providers/utils'
```

*   These lines import necessary modules and types from other files within the project.  Let's break them down:
    *   `createLogger`:  A function to create a logger instance for logging information and errors.  It's likely configured to output to the console.
    *   `createTokenizationError`:  A function to create a custom error object specifically for tokenization-related issues.  This helps in providing more informative error messages.
    *   `estimateInputTokens`, `estimateOutputTokens`, `estimateTokenCount`: Functions that are responsible for estimating how many tokens a given input or output string will be converted to.  These are likely using different tokenization algorithms depending on the model being used.
    *   `CostBreakdown`, `StreamingCostResult`, `TokenizationInput`, `TokenUsage`:  Type definitions that define the structure of the data used in the cost calculation process.  Using types ensures consistency and helps prevent errors.
        *   `CostBreakdown` likely describes the cost split into input cost, output cost, and the total cost.
        *   `StreamingCostResult` is the main result type, containing token usage, cost, model, provider, and the calculation method.
        *   `TokenizationInput` defines the structure of the input object passed to `calculateTokenizationCost`.
        *   `TokenUsage` holds the token counts for prompt, completion, and total tokens.
    *   `getProviderForTokenization`, `logTokenizationDetails`, `validateTokenizationInput`: Utility functions related to tokenization.
        *   `getProviderForTokenization` determines the LLM provider (e.g., OpenAI, Cohere) based on the model name.
        *   `logTokenizationDetails` logs detailed information about the tokenization process, which is helpful for debugging and monitoring.
        *   `validateTokenizationInput` performs input validation, likely checking for missing or invalid data.
    *   `calculateCost`: A function that calculates the cost based on the model, input tokens, and output tokens. It likely uses the provider's specific pricing rules.

```typescript
const logger = createLogger('TokenizationCalculators')
```

*   Creates a logger instance using the `createLogger` function, tagged with the name 'TokenizationCalculators'. This allows you to filter logs specifically related to these cost calculation functions.

```typescript
/**
 * Calculates cost estimate for streaming execution using token estimation
 */
export function calculateStreamingCost(
  model: string,
  inputText: string,
  outputText: string,
  systemPrompt?: string,
  context?: string,
  messages?: Array<{ role: string; content: string }>
): StreamingCostResult {
```

*   This is the main function for calculating the cost of a streaming LLM execution.  It's an `export`ed function, meaning it can be used by other modules.
    *   It takes the following parameters:
        *   `model`: The name of the LLM model being used (e.g., "gpt-3.5-turbo", "claude-v1").
        *   `inputText`: The main input text that the LLM will process.
        *   `outputText`: The output text generated by the LLM.
        *   `systemPrompt` (optional): A system-level prompt that sets the behavior of the LLM.
        *   `context` (optional):  Additional context that might be relevant to the LLM's response.
        *   `messages` (optional): An array of messages representing a conversation history. Each message has a `role` (e.g., "user", "assistant") and `content`.
    *   It returns a `StreamingCostResult` object, which contains the cost breakdown, token usage, and other relevant information.

```typescript
  try {
    // Validate inputs
    validateTokenizationInput(model, inputText, outputText)

    const providerId = getProviderForTokenization(model)

    // Estimate input tokens (combine all input sources)
    const inputEstimate = estimateInputTokens(systemPrompt, context, messages, providerId)

    // Add the main input text to the estimation
    const mainInputEstimate = estimateTokenCount(inputText, providerId)
    const totalPromptTokens = inputEstimate.count + mainInputEstimate.count

    // Estimate output tokens
    const outputEstimate = estimateOutputTokens(outputText, providerId)
    const completionTokens = outputEstimate.count

    // Calculate total tokens
    const totalTokens = totalPromptTokens + completionTokens

    // Create token usage object
    const tokens: TokenUsage = {
      prompt: totalPromptTokens,
      completion: completionTokens,
      total: totalTokens,
    }

    // Calculate cost using provider pricing
    const costResult = calculateCost(model, totalPromptTokens, completionTokens, false)

    const cost: CostBreakdown = {
      input: costResult.input,
      output: costResult.output,
      total: costResult.total,
    }

    const result: StreamingCostResult = {
      tokens,
      cost,
      model,
      provider: providerId,
      method: 'tokenization',
    }

    logTokenizationDetails('Streaming cost calculation completed', {
      model,
      provider: providerId,
      inputLength: inputText.length,
      outputLength: outputText.length,
      tokens,
      cost,
      method: 'tokenization',
    })

    return result
  } catch (error) {
    logger.error('Streaming cost calculation failed', {
      model,
      inputLength: inputText?.length || 0,
      outputLength: outputText?.length || 0,
      error: error instanceof Error ? error.message : String(error),
    })

    if (error instanceof Error && error.name === 'TokenizationError') {
      throw error
    }

    throw createTokenizationError(
      'CALCULATION_FAILED',
      `Failed to calculate streaming cost: ${error instanceof Error ? error.message : String(error)}`,
      { model, inputLength: inputText?.length || 0, outputLength: outputText?.length || 0 }
    )
  }
}
```

*   This is the main logic of the `calculateStreamingCost` function. Let's break it down step-by-step within the `try...catch` block:
    1.  `validateTokenizationInput(model, inputText, outputText)`: Validates the input parameters. This likely checks if `model`, `inputText`, and `outputText` are provided and are of the correct type.
    2.  `const providerId = getProviderForTokenization(model)`: Determines the LLM provider based on the `model` name.  For example, if `model` is "gpt-4", `providerId` might be "openai".
    3.  `const inputEstimate = estimateInputTokens(systemPrompt, context, messages, providerId)`: Estimates the number of tokens required for the input prompt. It considers the `systemPrompt`, `context`, and `messages` (conversation history) in the calculation.  The `providerId` is used because different providers might have slightly different tokenization rules.
    4.  `const mainInputEstimate = estimateTokenCount(inputText, providerId)`: Estimates the number of tokens for the main `inputText`.
    5.  `const totalPromptTokens = inputEstimate.count + mainInputEstimate.count`: Calculates the total number of tokens in the prompt by adding the tokens from the system prompt, context, messages, and the main input text.
    6.  `const outputEstimate = estimateOutputTokens(outputText, providerId)`: Estimates the number of tokens in the output text (`outputText`).
    7.  `const completionTokens = outputEstimate.count`: Assigns the output estimate to a more descriptive `completionTokens` variable.
    8.  `const totalTokens = totalPromptTokens + completionTokens`: Calculates the total number of tokens used (prompt + completion).
    9.  `const tokens: TokenUsage = { prompt: totalPromptTokens, completion: completionTokens, total: totalTokens }`: Creates a `TokenUsage` object to store the token counts for prompt, completion, and total tokens.
    10. `const costResult = calculateCost(model, totalPromptTokens, completionTokens, false)`: Calculates the cost using the `calculateCost` function. It takes the model, total prompt tokens, and completion tokens as input. The `false` argument likely indicates that this is not a fine-tuning operation (fine-tuning often has different pricing).
    11. `const cost: CostBreakdown = { input: costResult.input, output: costResult.output, total: costResult.total }`: Creates a `CostBreakdown` object from the `costResult`.
    12. `const result: StreamingCostResult = { tokens, cost, model, provider: providerId, method: 'tokenization' }`: Creates the final `StreamingCostResult` object, which contains all the calculated information.  The `method` field is set to 'tokenization' to indicate that the cost was estimated using tokenization.
    13. `logTokenizationDetails(...)`: Logs detailed information about the cost calculation process.
    14. `return result`: Returns the `StreamingCostResult` object.
    15. `catch (error)`: Handles any errors that occur during the process.
    16. `logger.error(...)`: Logs the error to the console.
    17.  `if (error instanceof Error && error.name === 'TokenizationError') { throw error; }`: If the error is a `TokenizationError`, re-throws it. This allows the calling function to handle specific tokenization errors differently.
    18. `throw createTokenizationError(...)`: If the error is not a `TokenizationError`, creates a new `TokenizationError` with a generic "CALCULATION\_FAILED" code and re-throws it. This provides a consistent error format.

```typescript
/**
 * Calculates cost for tokenization input object
 */
export function calculateTokenizationCost(input: TokenizationInput): StreamingCostResult {
  return calculateStreamingCost(
    input.model,
    input.inputText,
    input.outputText,
    input.systemPrompt,
    input.context,
    input.messages
  )
}
```

*   This function, `calculateTokenizationCost`, is a convenience function that takes a `TokenizationInput` object as input and calls `calculateStreamingCost` with the individual properties of the `TokenizationInput` object. This provides a more structured way to pass the input parameters.

```typescript
/**
 * Creates a streaming cost result from existing provider response data
 */
export function createCostResultFromProviderData(
  model: string,
  providerTokens: TokenUsage,
  providerCost: CostBreakdown
): StreamingCostResult {
  const providerId = getProviderForTokenization(model)

  return {
    tokens: providerTokens,
    cost: providerCost,
    model,
    provider: providerId,
    method: 'provider_response',
  }
}
```

*   This function, `createCostResultFromProviderData`, is used when the token usage and cost information are already provided by the LLM provider's API response.  It constructs a `StreamingCostResult` object using this data.  This is useful when you don't need to estimate the tokens yourself but want to standardize the cost result format. The `method` is set to `provider_response` to indicate that the cost information came directly from the provider.

**In Summary:**

This code provides a robust and well-structured way to estimate and calculate the cost of using LLMs. It incorporates input validation, token estimation, cost calculation, error handling, and structured result formatting. The functions are designed to be flexible and can be used with different LLM providers. The use of types and logging enhances the maintainability and debuggability of the code.
