Okay, let's break down this TypeScript code, focusing on its purpose, simplifying the complex logic, and explaining each line in detail.

**Purpose of this File**

This file provides utility functions for generating embeddings using either OpenAI or Azure OpenAI's embedding APIs.  Embeddings are numerical representations of text that capture its semantic meaning, which is crucial for tasks like semantic search, document similarity analysis, and knowledge retrieval.

The file handles:

*   Configuration: Determining whether to use OpenAI or Azure OpenAI based on environment variables.
*   API Calls: Making requests to the chosen embedding API.
*   Error Handling: Managing potential errors during API calls, including retries.
*   Token Batching: Dividing large texts into smaller batches to respect API token limits.
*   Logging: Recording important information and errors during the embedding generation process.

**Simplifying Complex Logic**

The core complexity lies in:

1.  **Conditional Configuration:**  The code dynamically configures the API endpoint and headers based on whether Azure OpenAI is being used. This adds conditional branches that can be confusing.

2.  **Token Batching:**  The `batchByTokenLimit` function (imported from `'@/lib/tokenization'`) handles the complexity of splitting the input text into batches that respect the token limits of the embedding API.  We won't dive into the implementation of `batchByTokenLimit` in this explanation, but understand it's crucial for handling long documents.

3.  **Retry Logic:** The `retryWithExponentialBackoff` function (imported from `'@/lib/knowledge/documents/utils'`) implements a retry mechanism for handling transient API errors.  This makes the code more robust but adds complexity.

**Detailed Explanation**

```typescript
import { env } from '@/lib/env'
import { isRetryableError, retryWithExponentialBackoff } from '@/lib/knowledge/documents/utils'
import { createLogger } from '@/lib/logs/console/logger'
import { batchByTokenLimit, getTotalTokenCount } from '@/lib/tokenization'

// Creates a logger instance for this module, named 'EmbeddingUtils'.  This allows for organized logging.
const logger = createLogger('EmbeddingUtils')

// Defines the maximum number of tokens allowed per API request.  This is a crucial constraint due to API limitations.
const MAX_TOKENS_PER_REQUEST = 8000

// Defines a custom error class for handling embedding API errors.
export class EmbeddingAPIError extends Error {
  public status: number // Holds the HTTP status code of the error.

  constructor(message: string, status: number) {
    super(message) // Calls the constructor of the parent `Error` class.
    this.name = 'EmbeddingAPIError' // Sets the name of the error for easier identification.
    this.status = status // Assigns the HTTP status code to the `status` property.
  }
}

// Defines an interface for the embedding configuration.
interface EmbeddingConfig {
  useAzure: boolean // Indicates whether to use Azure OpenAI.
  apiUrl: string // The API endpoint URL.
  headers: Record<string, string> // The headers to include in the API request.
  modelName: string // The name of the embedding model to use.
}

// Determines the embedding configuration based on environment variables.
function getEmbeddingConfig(embeddingModel = 'text-embedding-3-small'): EmbeddingConfig {
  const azureApiKey = env.AZURE_OPENAI_API_KEY // Retrieves the Azure OpenAI API key from environment variables.
  const azureEndpoint = env.AZURE_OPENAI_ENDPOINT // Retrieves the Azure OpenAI endpoint from environment variables.
  const azureApiVersion = env.AZURE_OPENAI_API_VERSION // Retrieves the Azure OpenAI API version from environment variables.
  const kbModelName = env.KB_OPENAI_MODEL_NAME || embeddingModel // Retrieves the knowledge base model name, defaulting to the provided `embeddingModel`.
  const openaiApiKey = env.OPENAI_API_KEY // Retrieves the OpenAI API key from environment variables.

  const useAzure = !!(azureApiKey && azureEndpoint) // Determines if Azure OpenAI should be used based on the presence of the API key and endpoint.

  // Checks if either OpenAI API key or Azure OpenAI configuration is provided.
  if (!useAzure && !openaiApiKey) {
    throw new Error(
      'Either OPENAI_API_KEY or Azure OpenAI configuration (AZURE_OPENAI_API_KEY + AZURE_OPENAI_ENDPOINT) must be configured'
    )
  }

  // Constructs the API URL based on whether Azure OpenAI is being used.
  const apiUrl = useAzure
    ? `${azureEndpoint}/openai/deployments/${kbModelName}/embeddings?api-version=${azureApiVersion}`
    : 'https://api.openai.com/v1/embeddings'

  // Constructs the headers based on whether Azure OpenAI is being used.
  const headers: Record<string, string> = useAzure
    ? {
        'api-key': azureApiKey!,
        'Content-Type': 'application/json',
      }
    : {
        Authorization: `Bearer ${openaiApiKey!}`,
        'Content-Type': 'application/json',
      }

  // Returns the embedding configuration object.
  return {
    useAzure,
    apiUrl,
    headers,
    modelName: useAzure ? kbModelName : embeddingModel,
  }
}

// Calls the embedding API to generate embeddings for the given inputs.
async function callEmbeddingAPI(inputs: string[], config: EmbeddingConfig): Promise<number[][]> {
  // Uses retryWithExponentialBackoff to retry the API call in case of errors.
  return retryWithExponentialBackoff(
    async () => {
      // Constructs the request body based on whether Azure OpenAI is being used.
      const requestBody = config.useAzure
        ? {
            input: inputs,
            encoding_format: 'float',
          }
        : {
            input: inputs,
            model: config.modelName,
            encoding_format: 'float',
          }

      // Makes the API request using fetch.
      const response = await fetch(config.apiUrl, {
        method: 'POST',
        headers: config.headers,
        body: JSON.stringify(requestBody),
      })

      // Checks if the response was successful.
      if (!response.ok) {
        const errorText = await response.text()
        throw new EmbeddingAPIError(
          `Embedding API failed: ${response.status} ${response.statusText} - ${errorText}`,
          response.status
        )
      }

      // Parses the response and extracts the embeddings.
      const data = await response.json()
      return data.data.map((item: any) => item.embedding)
    },
    {
      maxRetries: 3, // Sets the maximum number of retries.
      initialDelayMs: 1000, // Sets the initial delay between retries (in milliseconds).
      maxDelayMs: 10000, // Sets the maximum delay between retries (in milliseconds).
      retryCondition: (error: any) => {
        // Defines the condition for retrying the API call.
        if (error instanceof EmbeddingAPIError) {
          // Retries if the error is an EmbeddingAPIError with a status code of 429 (Too Many Requests) or 500 or higher (Server Error).
          return error.status === 429 || error.status >= 500
        }
        // Otherwise, it uses the isRetryableError utility function to determine if the error is retryable.
        return isRetryableError(error)
      },
    }
  )
}

/**
 * Generate embeddings for multiple texts with token-aware batching
 * Uses tiktoken for token counting
 */
// Generates embeddings for multiple texts, handling token limits by batching the texts.
export async function generateEmbeddings(
  texts: string[],
  embeddingModel = 'text-embedding-3-small'
): Promise<number[][]> {
  const config = getEmbeddingConfig(embeddingModel) // Gets the embedding configuration.

  logger.info(
    `Using ${config.useAzure ? 'Azure OpenAI' : 'OpenAI'} for embeddings generation (${texts.length} texts)`
  )

  // Splits the texts into batches based on the token limit.  This is where the `batchByTokenLimit` function is used.
  const batches = batchByTokenLimit(texts, MAX_TOKENS_PER_REQUEST, embeddingModel)

  logger.info(
    `Split ${texts.length} texts into ${batches.length} batches (max ${MAX_TOKENS_PER_REQUEST} tokens per batch)`
  )

  const allEmbeddings: number[][] = [] // Initializes an array to store all embeddings.

  // Iterates over the batches.
  for (let i = 0; i < batches.length; i++) {
    const batch = batches[i] // Gets the current batch.
    const batchTokenCount = getTotalTokenCount(batch, embeddingModel) // Calculates the total token count for the batch.

    logger.info(
      `Processing batch ${i + 1}/${batches.length}: ${batch.length} texts, ${batchTokenCount} tokens`
    )

    try {
      // Calls the embedding API to generate embeddings for the batch.
      const batchEmbeddings = await callEmbeddingAPI(batch, config)
      allEmbeddings.push(...batchEmbeddings) // Adds the batch embeddings to the overall array.

      logger.info(
        `Generated ${batchEmbeddings.length} embeddings for batch ${i + 1}/${batches.length}`
      )
    } catch (error) {
      // Logs an error message if the embedding generation fails.
      logger.error(`Failed to generate embeddings for batch ${i + 1}:`, error)
      throw error // Re-throws the error to propagate it up the call stack.
    }

    // Adds a small delay between batches to avoid rate limiting.
    if (i + 1 < batches.length) {
      await new Promise((resolve) => setTimeout(resolve, 100))
    }
  }

  logger.info(`Successfully generated ${allEmbeddings.length} embeddings total`)

  return allEmbeddings // Returns the array of all embeddings.
}

/**
 * Generate embedding for a single search query
 */
// Generates an embedding for a single search query.
export async function generateSearchEmbedding(
  query: string,
  embeddingModel = 'text-embedding-3-small'
): Promise<number[]> {
  const config = getEmbeddingConfig(embeddingModel) // Gets the embedding configuration.

  logger.info(
    `Using ${config.useAzure ? 'Azure OpenAI' : 'OpenAI'} for search embedding generation`
  )

  const embeddings = await callEmbeddingAPI([query], config) // Calls the embedding API with the single query.
  return embeddings[0] // Returns the first (and only) embedding.
}
```

**Key Takeaways:**

*   This code encapsulates the process of generating embeddings from text using OpenAI or Azure OpenAI, abstracting away the complexities of API configuration, error handling, and token management.
*   The `generateEmbeddings` function is the workhorse, handling batching and API calls for multiple texts.  The `generateSearchEmbedding` function is a simplified version for single queries.
*   The use of a custom `EmbeddingAPIError` class and `retryWithExponentialBackoff` demonstrate robust error handling practices.
*   The code leverages environment variables for configuration, making it adaptable to different environments.
*   The use of logging provides valuable insights into the embedding generation process, which is helpful for debugging and monitoring.

This comprehensive explanation should give you a strong understanding of the code's functionality and design.  Let me know if you have any more questions.
