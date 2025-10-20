This TypeScript file sets up a Next.js API route designed to interact with Large Language Models (LLMs) from either OpenAI or Azure OpenAI, allowing your application to generate text content. It's a "Wand generation API" that handles user prompts, manages conversation history, supports both standard (single response) and streaming (real-time chunks) modes, and meticulously tracks token usage for billing purposes.

Think of it as the backend brains behind a feature where users can type a request and get AI-generated text back, all while ensuring proper resource management and cost tracking.

---

### **Simplified Complex Logic**

1.  **Smart AI Client Setup**: The code intelligently checks your environment variables to decide if it should use Microsoft Azure's OpenAI service or the standard OpenAI service. It then sets up the correct client to talk to the chosen AI. If neither is configured, it logs a warning and gracefully degrades.

2.  **Flexible Content Generation**:
    *   **Request Input**: It receives a user's `prompt`, optional `systemPrompt` (to guide the AI's personality), any `history` of previous messages (for context), and a `workflowId` (for billing). It also checks if the user wants the AI's response to `stream` (come back in real-time chunks) or as a single, complete `response`.
    *   **Message Construction**: It carefully organizes all these inputs into a standard message format that AI models understand, ensuring the AI gets the full context.

3.  **Real-time vs. Full Response**:
    *   **Streaming (for live updates)**: If `stream` is requested, it makes a special HTTP `fetch` call to the AI service. As the AI generates text, the code receives tiny pieces of text (chunks), processes them, and immediately sends them back to the user's browser, making the experience feel real-time. It also looks for "usage" information that often comes at the very end of the stream to record costs.
    *   **Non-Streaming (for one-off requests)**: If streaming isn't needed, it uses the AI client object to send the prompt, waits for the AI to finish generating *all* content, and then sends the complete response back.

4.  **Automated Billing & Usage Tracking**:
    *   After the AI generates content, the code extracts the number of "tokens" (words/parts of words) used for both the input and output.
    *   It then calculates the cost based on the AI model's pricing and applies any environment-specific cost multipliers.
    *   Finally, it updates the user's total token usage and cost in a database. It even triggers a check to see if the user has gone over any predefined spending limits, and if so, initiates a billing action.

5.  **Robust Error Handling**: The API is designed to catch various problems, from missing user input to issues communicating with the AI service. It logs detailed errors for developers and provides user-friendly error messages to the client.

---

### **Detailed Line-by-Line Explanation**

#### **Imports and Setup**

```typescript
import { db } from '@sim/db'
```
*   **Explanation**: This line imports the pre-configured database client, named `db`. This client is what allows our application to interact with the database (e.g., PostgreSQL, MySQL) using Drizzle ORM. The `@sim/db` path suggests it's part of a monorepo setup where `sim` is a core library.

```typescript
import { userStats, workflow } from '@sim/db/schema'
```
*   **Explanation**: This imports type-safe schema definitions for two database tables (or collections): `userStats` (likely stores user usage data and costs) and `workflow` (likely stores information about user workflows, including who owns them). These definitions come from `drizzle-orm` and help ensure correct data manipulation.

```typescript
import { eq, sql } from 'drizzle-orm'
```
*   **Explanation**: From the `drizzle-orm` library, we import two essential functions:
    *   `eq`: Used to create equality conditions in database queries (e.g., `WHERE id = '...'`).
    *   `sql`: A special function (called a "template literal tag") that allows you to write raw SQL expressions directly within Drizzle queries. This is useful for performing atomic operations like incrementing a value (`column = column + value`) directly in the database.

```typescript
import { type NextRequest, NextResponse } from 'next/server'
```
*   **Explanation**: These imports are specific to Next.js API routes:
    *   `NextRequest`: A type definition for the incoming HTTP request object, providing access to things like headers, body, and URL.
    *   `NextResponse`: A utility class for constructing and sending HTTP responses, allowing us to easily set status codes, headers, and JSON bodies.

```typescript
import OpenAI, { AzureOpenAI } from 'openai'
```
*   **Explanation**: This brings in the official OpenAI SDK client classes:
    *   `OpenAI`: Used to interact with the standard OpenAI API (e.g., `api.openai.com`).
    *   `AzureOpenAI`: A specialized client for connecting to OpenAI models hosted and managed within a Microsoft Azure environment.

```typescript
import { checkAndBillOverageThreshold } from '@/lib/billing/threshold-billing'
```
*   **Explanation**: This imports a utility function responsible for checking if a user's accumulated cost has exceeded a predefined limit and, if so, triggering an incremental billing process for the overage.

```typescript
import { env } from '@/lib/env'
```
*   **Explanation**: This imports an `env` object, which likely provides a type-safe and validated way to access environment variables (e.g., `process.env.OPENAI_API_KEY`) configured for the application.

```typescript
import { getCostMultiplier, isBillingEnabled } from '@/lib/environment'
```
*   **Explanation**: These imports provide configuration related to the application's environment:
    *   `getCostMultiplier`: A function that returns a numerical factor to adjust the base cost of AI usage (e.g., for different pricing tiers or profit margins).
    *   `isBillingEnabled`: A boolean flag that indicates whether the billing system is active or disabled for the current environment.

```typescript
import { createLogger } from '@/lib/logs/console/logger'
```
*   **Explanation**: This imports a function to create logger instances. This allows for structured and categorized logging, making it easier to monitor and debug the application's behavior.

```typescript
import { generateRequestId } from '@/lib/utils'
```
*   **Explanation**: This imports a utility function that generates a unique identifier for each incoming request. This `requestId` is crucial for tracing specific requests through logs and debugging.

```typescript
import { getModelPricing } from '@/providers/utils'
```
*   **Explanation**: This imports a function that retrieves the cost per token for various AI models. This pricing information is vital for accurately calculating user costs based on their AI usage.

---

#### **Next.js Route Configuration**

```typescript
export const dynamic = 'force-dynamic'
```
*   **Explanation**: This Next.js export ensures that this API route is treated as fully dynamic. This means it will not be cached, and every request will execute the `POST` function from scratch, which is essential for an API handling real-time AI interactions and billing.

```typescript
export const runtime = 'nodejs'
```
*   **Explanation**: This specifies that the API route should run in the Node.js runtime environment. This is often necessary when using Node.js-specific features or SDKs, like Drizzle ORM and the OpenAI SDKs, which are built for Node.js.

```typescript
export const maxDuration = 60
```
*   **Explanation**: This sets the maximum allowed execution time for this API route to 60 seconds. AI generation, especially for complex prompts or streaming scenarios, can sometimes take longer, so this extends the default timeout.

---

#### **Logger and AI Client Initialization**

```typescript
const logger = createLogger('WandGenerateAPI')
```
*   **Explanation**: A logger instance is created specifically for this file, named `WandGenerateAPI`. All log messages from this file will be associated with this name, aiding in log filtering and analysis.

```typescript
const azureApiKey = env.AZURE_OPENAI_API_KEY
const azureEndpoint = env.AZURE_OPENAI_ENDPOINT
const azureApiVersion = env.AZURE_OPENAI_API_VERSION
const wandModelName = env.WAND_OPENAI_MODEL_NAME || 'gpt-4o'
const openaiApiKey = env.OPENAI_API_KEY
```
*   **Explanation**: These lines retrieve critical configuration values from environment variables:
    *   `azureApiKey`, `azureEndpoint`, `azureApiVersion`: Credentials and deployment details for Azure OpenAI.
    *   `wandModelName`: The specific AI model to use (e.g., "gpt-4o"). It defaults to `'gpt-4o'` if the environment variable isn't set.
    *   `openaiApiKey`: The API key for standard OpenAI.

```typescript
const useWandAzure = azureApiKey && azureEndpoint && azureApiVersion
```
*   **Explanation**: This boolean variable `useWandAzure` becomes `true` only if all three required Azure OpenAI environment variables (`azureApiKey`, `azureEndpoint`, `azureApiVersion`) are present. This determines whether to use the Azure OpenAI service.

```typescript
const client = useWandAzure
  ? new AzureOpenAI({
      apiKey: azureApiKey,
      apiVersion: azureApiVersion,
      endpoint: azureEndpoint,
    })
  : openaiApiKey
    ? new OpenAI({
        apiKey: openaiApiKey,
      })
    : null
```
*   **Explanation**: This is a conditional statement that initializes the AI client:
    *   If `useWandAzure` is `true`, an `AzureOpenAI` client is created using the Azure credentials.
    *   Else (if Azure isn't configured), it checks if `openaiApiKey` is available. If so, a standard `OpenAI` client is created.
    *   If neither Azure nor standard OpenAI API keys are found, `client` is set to `null`.

```typescript
if (!useWandAzure && !openaiApiKey) {
  logger.warn(
    'Neither Azure OpenAI nor OpenAI API key found. Wand generation API will not function.'
  )
} else {
  logger.info(`Using ${useWandAzure ? 'Azure OpenAI' : 'OpenAI'} for wand generation`)
}
```
*   **Explanation**: This block logs the outcome of the AI client initialization. It warns if no API keys were found (meaning the AI generation won't work) or informs which AI service (Azure or standard OpenAI) is being used.

---

#### **Interfaces for Type Safety**

```typescript
interface ChatMessage {
  role: 'user' | 'assistant' | 'system'
  content: string
}
```
*   **Explanation**: This TypeScript interface defines the expected structure of a single message in a chat conversation. It has two properties:
    *   `role`: Specifies who sent the message (`'user'`, `'assistant'`, or `'system'`).
    *   `content`: The actual text of the message.

```typescript
interface RequestBody {
  prompt: string
  systemPrompt?: string
  stream?: boolean
  history?: ChatMessage[]
  workflowId?: string
}
```
*   **Explanation**: This interface defines the expected structure of the JSON body for incoming `POST` requests to this API route:
    *   `prompt`: The mandatory user input for AI generation.
    *   `systemPrompt?`: An optional initial instruction for the AI (e.g., "Act as a helpful assistant").
    *   `stream?`: An optional boolean indicating if the response should be streamed (defaulting to `false`).
    *   `history?`: An optional array of `ChatMessage` objects providing previous conversation context.
    *   `workflowId?`: An optional ID to link the generation request to a specific workflow for billing.

---

#### **Helper Functions**

```typescript
function safeStringify(value: unknown): string {
  try {
    return JSON.stringify(value)
  } catch {
    return '[unserializable]'
  }
}
```
*   **Explanation**: This utility function attempts to convert any JavaScript `value` into a JSON string. It's wrapped in a `try...catch` block to gracefully handle situations where the `value` cannot be serialized to JSON (e.g., circular references), returning `'[unserializable]'` instead of throwing an error. This is useful for robust logging.

---

```typescript
async function updateUserStatsForWand(
  workflowId: string,
  usage: {
    prompt_tokens?: number
    completion_tokens?: number
    total_tokens?: number
  },
  requestId: string
): Promise<void> {
```
*   **Explanation**: This asynchronous function is responsible for calculating and updating a user's token usage and associated costs in the database. It takes the `workflowId` (to identify the user), an object `usage` containing token counts from the AI response, and the `requestId` for logging context.

```typescript
  if (!isBillingEnabled) {
    logger.debug(`[${requestId}] Billing is disabled, skipping wand usage cost update`)
    return
  }
```
*   **Explanation**: An early exit condition: if the `isBillingEnabled` flag (from `getCostMultiplier`) is `false`, it logs that billing is skipped and stops the function, preventing unnecessary database operations.

```typescript
  if (!usage.total_tokens || usage.total_tokens <= 0) {
    logger.debug(`[${requestId}] No tokens to update in user stats`)
    return
  }
```
*   **Explanation**: Another early exit: if the `usage` object doesn't contain `total_tokens` or if the count is zero or negative, it logs that no tokens need to be updated and stops.

```typescript
  try {
    const [workflowRecord] = await db
      .select({ userId: workflow.userId })
      .from(workflow)
      .where(eq(workflow.id, workflowId))
      .limit(1)
```
*   **Explanation**: This Drizzle ORM query fetches the `userId` from the `workflow` table, matching the provided `workflowId`. `.limit(1)` is used for efficiency as we only expect one workflow record. The `[workflowRecord]` syntax destructures the first (and only) element of the result array.

```typescript
    if (!workflowRecord?.userId) {
      logger.warn(
        `[${requestId}] No user found for workflow ${workflowId}, cannot update user stats`
      )
      return
    }
```
*   **Explanation**: If the query above fails to find a workflow record or the record doesn't have a `userId`, it logs a warning and exits, as we cannot attribute the usage to a specific user.

```typescript
    const userId = workflowRecord.userId
    const totalTokens = usage.total_tokens || 0
    const promptTokens = usage.prompt_tokens || 0
    const completionTokens = usage.completion_tokens || 0
```
*   **Explanation**: These lines extract the `userId` and token counts (`totalTokens`, `promptTokens`, `completionTokens`) from the retrieved data and the `usage` object. The `|| 0` ensures that any missing values default to zero.

```typescript
    const modelName = useWandAzure ? wandModelName : 'gpt-4o'
    const pricing = getModelPricing(modelName)
```
*   **Explanation**: It determines the `modelName` based on whether Azure is being used. Then, it calls `getModelPricing` to fetch the specific input and output costs for that model.

```typescript
    const costMultiplier = getCostMultiplier()
    let modelCost = 0
```
*   **Explanation**: The `costMultiplier` (e.g., for profit margins) is fetched. `modelCost` is initialized to `0` before calculation.

```typescript
    if (pricing) {
      const inputCost = (promptTokens / 1000000) * pricing.input
      const outputCost = (completionTokens / 1000000) * pricing.output
      modelCost = inputCost + outputCost
    } else {
      modelCost = (promptTokens / 1000000) * 0.005 + (completionTokens / 1000000) * 0.015
    }
```
*   **Explanation**: This block calculates the base cost:
    *   If `pricing` is available, it calculates `inputCost` and `outputCost` based on tokens (converted to millions, as pricing is often per million) and the `pricing` rates, then sums them.
    *   If `pricing` is *not* available (e.g., `getModelPricing` failed), it uses fallback hardcoded default rates for a generic model.

```typescript
    const costToStore = modelCost * costMultiplier
```
*   **Explanation**: The calculated `modelCost` is adjusted by the `costMultiplier` to get the final `costToStore` that will be saved in the database.

```typescript
    await db
      .update(userStats)
      .set({
        totalTokensUsed: sql`total_tokens_used + ${totalTokens}`,
        totalCost: sql`total_cost + ${costToStore}`,
        currentPeriodCost: sql`current_period_cost + ${costToStore}`,
        lastActive: new Date(),
      })
      .where(eq(userStats.userId, userId))
```
*   **Explanation**: This Drizzle ORM query updates the `userStats` record for the given `userId`:
    *   `totalTokensUsed`: Increments the stored total tokens by `totalTokens` using a raw SQL increment (`sql` function for atomic operation).
    *   `totalCost`: Increments the overall total cost by `costToStore`.
    *   `currentPeriodCost`: Increments the cost for the current billing period by `costToStore`.
    *   `lastActive`: Updates the timestamp to the current date and time.
    *   `.where(eq(userStats.userId, userId))`: Ensures only the correct user's stats are updated.

```typescript
    logger.debug(`[${requestId}] Updated user stats for wand usage`, {
      userId,
      tokensUsed: totalTokens,
      costAdded: costToStore,
    })
```
*   **Explanation**: A debug log confirms the successful update of user statistics, including the `userId`, `tokensUsed`, and `costAdded`.

```typescript
    // Check if user has hit overage threshold and bill incrementally
    await checkAndBillOverageThreshold(userId)
```
*   **Explanation**: After updating the user's spending, this function is called to determine if the user has crossed any configured billing thresholds and, if so, to initiate an overage charge.

```typescript
  } catch (error) {
    logger.error(`[${requestId}] Failed to update user stats for wand usage`, error)
  }
}
```
*   **Explanation**: This `catch` block handles any errors that occur during the database interaction or cost calculation within this function, logging them without stopping the main request flow.

---

#### **Main API Route Handler: `POST`**

```typescript
export async function POST(req: NextRequest) {
```
*   **Explanation**: This defines the main asynchronous function that will be executed when an HTTP `POST` request is made to this API route. It receives the `NextRequest` object (`req`) containing details about the incoming request.

```typescript
  const requestId = generateRequestId()
  logger.info(`[${requestId}] Received wand generation request`)
```
*   **Explanation**: A unique `requestId` is generated for this specific incoming request, and an informational log message marks the beginning of its processing.

```typescript
  if (!client) {
    logger.error(`[${requestId}] AI client not initialized. Missing API key.`)
    return NextResponse.json(
      { success: false, error: 'Wand generation service is not configured.' },
      { status: 503 }
    )
  }
```
*   **Explanation**: This is a critical check. If the `client` object (initialized earlier) is `null` (meaning no AI API key was provided), it logs an error and returns a `503 Service Unavailable` response to the client, as the core functionality cannot proceed.

```typescript
  try {
    const body = (await req.json()) as RequestBody
```
*   **Explanation**: It asynchronously parses the JSON body of the incoming request and casts it to the `RequestBody` interface for type safety.

```typescript
    const { prompt, systemPrompt, stream = false, history = [], workflowId } = body
```
*   **Explanation**: This line destructures the properties from the `body` object. It provides default values for `stream` (`false`) and `history` (`[]`) if they are not present in the request.

```typescript
    if (!prompt) {
      logger.warn(`[${requestId}] Invalid request: Missing prompt.`)
      return NextResponse.json(
        { success: false, error: 'Missing required field: prompt.' },
        { status: 400 }
      )
    }
```
*   **Explanation**: It validates that the `prompt` field, which is mandatory, is present in the request. If missing, it logs a warning and sends a `400 Bad Request` error response.

```typescript
    const finalSystemPrompt =
      systemPrompt ||
      'You are a helpful AI assistant. Generate content exactly as requested by the user.'
```
*   **Explanation**: It sets the `finalSystemPrompt`. If a `systemPrompt` was provided in the request, it's used; otherwise, a default, general helpful assistant prompt is used.

```typescript
    const messages: ChatMessage[] = [{ role: 'system', content: finalSystemPrompt }]
```
*   **Explanation**: An array `messages` is initialized with the `finalSystemPrompt` as the first system message. This array will hold the entire conversation context for the AI.

```typescript
    messages.push(...history.filter((msg) => msg.role !== 'system'))
```
*   **Explanation**: Any historical chat messages provided in the `history` array from the request body are added to the `messages` array. It filters out any previous system messages from the history, ensuring our `finalSystemPrompt` is the primary system instruction.

```typescript
    messages.push({ role: 'user', content: prompt })
```
*   **Explanation**: The current `prompt` from the user is added to the `messages` array with the `role` set to `'user'`, completing the conversation context.

```typescript
    logger.debug(
      `[${requestId}] Calling ${useWandAzure ? 'Azure OpenAI' : 'OpenAI'} API for wand generation`,
      {
        stream,
        historyLength: history.length,
        endpoint: useWandAzure ? azureEndpoint : 'api.openai.com',
        model: useWandAzure ? wandModelName : 'gpt-4o',
        apiVersion: useWandAzure ? azureApiVersion : 'N/A',
      }
    )
```
*   **Explanation**: A detailed debug log provides context about the upcoming AI API call, including the chosen AI service, streaming status, history length, and model details.

---

#### **Streaming Logic**

```typescript
    if (stream) {
      try {
        logger.debug(
          `[${requestId}] Starting streaming request to ${useWandAzure ? 'Azure OpenAI' : 'OpenAI'}`
        )

        logger.info(
          `[${requestId}] About to create stream with model: ${useWandAzure ? wandModelName : 'gpt-4o'}`
        )
```
*   **Explanation**: This `if` block executes if the client requested `stream: true`. It logs that a streaming request is starting and specifies the model being used.

```typescript
        const apiUrl = useWandAzure
          ? `${azureEndpoint}/openai/deployments/${wandModelName}/chat/completions?api-version=${azureApiVersion}`
          : 'https://api.openai.com/v1/chat/completions'
```
*   **Explanation**: Dynamically constructs the target API URL for the AI service. It uses an Azure-specific format if `useWandAzure` is true, otherwise, it uses the standard OpenAI endpoint.

```typescript
        const headers: Record<string, string> = {
          'Content-Type': 'application/json',
        }

        if (useWandAzure) {
          headers['api-key'] = azureApiKey!
        } else {
          headers.Authorization = `Bearer ${openaiApiKey}`
        }
```
*   **Explanation**: Sets up the HTTP headers for the `fetch` request. It always includes `Content-Type: application/json`. For Azure, it adds `api-key`. For standard OpenAI, it adds `Authorization: Bearer <API_KEY>`. The `!` after `azureApiKey` is a non-null assertion, safe because `useWandAzure` ensures it's present.

```typescript
        logger.debug(`[${requestId}] Making streaming request to: ${apiUrl}`)

        const response = await fetch(apiUrl, {
          method: 'POST',
          headers,
          body: JSON.stringify({
            model: useWandAzure ? wandModelName : 'gpt-4o',
            messages: messages,
            temperature: 0.2,
            max_tokens: 10000,
            stream: true,
            stream_options: { include_usage: true },
          }),
        })
```
*   **Explanation**: This is the actual network call using the browser's `fetch` API. It sends a `POST` request to the AI API with the constructed URL, headers, and a JSON body containing:
    *   `model`: The AI model.
    *   `messages`: The conversation context.
    *   `temperature`: Controls AI creativity (0.2 means fairly deterministic).
    *   `max_tokens`: Maximum output length.
    *   `stream: true`: Crucially tells the AI API to send a streaming response.
    *   `stream_options: { include_usage: true }`: Requests that token usage data be included in the stream (usually at the end).

```typescript
        if (!response.ok) {
          const errorText = await response.text()
          logger.error(`[${requestId}] API request failed`, {
            status: response.status,
            statusText: response.statusText,
            error: errorText,
          })
          throw new Error(`API request failed: ${response.status} ${response.statusText}`)
        }
```
*   **Explanation**: If the `fetch` request receives an HTTP status code indicating an error (e.g., 4xx or 5xx), it reads the error body, logs the details, and throws an error to be caught by the outer `try...catch`.

```typescript
        logger.info(`[${requestId}] Stream response received, starting processing`)

        const encoder = new TextEncoder()
        const decoder = new TextDecoder()
```
*   **Explanation**: An info log confirms the stream is received. `TextEncoder` and `TextDecoder` are initialized to convert between JavaScript strings and raw byte data, necessary for processing the incoming stream and preparing it for the client.

```typescript
        const readable = new ReadableStream({
          async start(controller) {
            const reader = response.body?.getReader()
            if (!reader) {
              controller.close()
              return
            }
```
*   **Explanation**: This creates a `ReadableStream`, which is a powerful Web API for processing chunks of data. The `start` method is called when the stream begins. It attempts to get a `reader` from the `response.body` (the raw stream from the AI API). If no reader is available, it closes the stream.

```typescript
            try {
              let buffer = ''
              let chunkCount = 0
              let finalUsage: any = null
```
*   **Explanation**: Inside the stream processing logic, these variables are initialized:
    *   `buffer`: Stores partial lines of data received from the AI API.
    *   `chunkCount`: Counts how many content chunks have been processed.
    *   `finalUsage`: Will store the final token usage statistics reported by the AI.

```typescript
              while (true) {
                const { done, value } = await reader.read()

                if (done) {
                  logger.info(`[${requestId}] Stream completed. Total chunks: ${chunkCount}`)
                  controller.enqueue(encoder.encode(`data: ${JSON.stringify({ done: true })}\n\n`))
                  controller.close()
                  break
                }
```
*   **Explanation**: This `while` loop continuously reads chunks from the AI API stream.
    *   `reader.read()` fetches the next `value` (a `Uint8Array` of bytes) and a `done` flag.
    *   If `done` is `true`, the upstream AI stream has finished. It logs, sends a `done: true` message to the client, closes its own stream `controller`, and breaks the loop.

```typescript
                buffer += decoder.decode(value, { stream: true })

                const lines = buffer.split('\n')
                buffer = lines.pop() || ''
```
*   **Explanation**:
    *   The raw `value` (bytes) is decoded into a string and appended to `buffer`.
    *   `buffer` is split into `lines` by newline characters.
    *   The last (potentially incomplete) line is kept in `buffer` for the next iteration, while complete `lines` are processed.

```typescript
                for (const line of lines) {
                  if (line.startsWith('data: ')) {
                    const data = line.slice(6).trim()

                    if (data === '[DONE]') {
                      logger.info(`[${requestId}] Received [DONE] signal`)
                      controller.enqueue(
                        encoder.encode(`data: ${JSON.stringify({ done: true })}\n\n`)
                      )
                      controller.close()
                      return
                    }
```
*   **Explanation**: This loop processes each complete line. It checks if the line starts with `data: ` (the SSE format). If the extracted `data` is `[DONE]`, it's a special signal from OpenAI, indicating content completion. It logs, sends a `done: true` message, closes the controller, and exits the stream handler.

```typescript
                    try {
                      const parsed = JSON.parse(data)
                      const content = parsed.choices?.[0]?.delta?.content

                      if (content) {
                        chunkCount++
                        if (chunkCount === 1) {
                          logger.info(`[${requestId}] Received first content chunk`)
                        }

                        controller.enqueue(
                          encoder.encode(`data: ${JSON.stringify({ chunk: content })}\n\n`)
                        )
                      }

                      if (parsed.usage) {
                        finalUsage = parsed.usage
                        logger.info(
                          `[${requestId}] Received usage data: ${JSON.stringify(parsed.usage)}`
                        )
                      }

                      if (chunkCount % 10 === 0) {
                        logger.debug(`[${requestId}] Processed ${chunkCount} chunks`)
                      }
                    } catch (parseError) {
                      logger.debug(
                        `[${requestId}] Skipped non-JSON line: ${data.substring(0, 100)}`
                      )
                    }
                  }
                }
              }
```
*   **Explanation**: This `try...catch` block attempts to parse the `data` as JSON.
    *   If `content` is found (`parsed.choices?.[0]?.delta?.content`), it's an AI-generated text chunk. It increments `chunkCount`, logs if it's the first chunk, and `enqueues` it as an SSE message (`data: {"chunk": "..."}\n\n`) to be sent to the client.
    *   If `parsed.usage` is found, it stores this token usage data in `finalUsage` and logs it. This is typically sent as the final piece of information from the AI stream.
    *   A debug log shows progress every 10 chunks.
    *   The `catch` block handles non-JSON lines, logging them as debug messages and skipping them.

```typescript
              logger.info(`[${requestId}] Wand generation streaming completed successfully`)

              if (finalUsage && workflowId) {
                await updateUserStatsForWand(workflowId, finalUsage, requestId)
              }
            } catch (streamError: any) {
              logger.error(`[${requestId}] Streaming error`, {
                name: streamError?.name,
                message: streamError?.message || 'Unknown error',
                stack: streamError?.stack,
              })

              const errorData = `data: ${JSON.stringify({ error: 'Streaming failed', done: true })}\n\n`
              controller.enqueue(encoder.encode(errorData))
              controller.close()
            } finally {
              reader.releaseLock()
            }
          },
        })
```
*   **Explanation**:
    *   After the stream finishes, a success log is recorded.
    *   If `finalUsage` was collected and a `workflowId` exists, `updateUserStatsForWand` is called to update billing.
    *   The inner `catch (streamError)` handles errors that occur specifically *during* the stream processing (e.g., network interruptions while reading chunks). It logs the error, sends an error SSE message to the client, and closes the stream.
    *   The `finally` block ensures `reader.releaseLock()` is called, releasing the lock on the response body resources.

```typescript
        return new Response(readable, {
          headers: {
            'Content-Type': 'text/event-stream',
            'Cache-Control': 'no-cache, no-transform',
            Connection: 'keep-alive',
            'X-Accel-Buffering': 'no',
          },
        })
```
*   **Explanation**: This returns a new `Response` object containing the `readable` stream to the client. The `headers` are crucial for Server-Sent Events (SSE): `Content-Type: text/event-stream` tells the client how to interpret the response, and the other headers manage caching and connection behavior for a continuous stream.

```typescript
      } catch (error: any) {
        logger.error(`[${requestId}] Failed to create stream`, {
          name: error?.name,
          message: error?.message || 'Unknown error',
          code: error?.code,
          status: error?.status,
          responseStatus: error?.response?.status,
          responseData: error?.response?.data ? safeStringify(error.response.data) : undefined,
          stack: error?.stack,
          useWandAzure,
          model: useWandAzure ? wandModelName : 'gpt-4o',
          endpoint: useWandAzure ? azureEndpoint : 'api.openai.com',
          apiVersion: useWandAzure ? azureApiVersion : 'N/A',
        })

        return NextResponse.json(
          { success: false, error: 'An error occurred during wand generation streaming.' },
          { status: 500 }
        )
      }
    }
```
*   **Explanation**: This `catch` block specifically handles errors that occur *during the setup* of the streaming request (e.g., issues with the initial `fetch` call before the `ReadableStream` takes over). It logs detailed error information and sends a generic `500 Internal Server Error` JSON response to the client.

---

#### **Non-Streaming (Standard) Logic**

```typescript
    const completion = await client.chat.completions.create({
      model: useWandAzure ? wandModelName : 'gpt-4o',
      messages: messages,
      temperature: 0.3,
      max_tokens: 10000,
    })
```
*   **Explanation**: This block executes if `stream` was `false`. It uses the initialized AI `client` (OpenAI or AzureOpenAI SDK) to make a standard `chat.completions.create` API call. The parameters are similar to the streaming call, but `stream: true` is omitted. The `temperature` is slightly higher (0.3), allowing for a bit more creativity.

```typescript
    const generatedContent = completion.choices[0]?.message?.content?.trim()
```
*   **Explanation**: It safely extracts the generated text content from the AI's `completion` response. `?.` (optional chaining) is used to prevent errors if any part of the path is `undefined`, and `.trim()` removes extra whitespace.

```typescript
    if (!generatedContent) {
      logger.error(
        `[${requestId}] ${useWandAzure ? 'Azure OpenAI' : 'OpenAI'} response was empty or invalid.`
      )
      return NextResponse.json(
        { success: false, error: 'Failed to generate content. AI response was empty.' },
        { status: 500 }
      )
    }
```
*   **Explanation**: If `generatedContent` is empty or `null`, it logs an error and returns a `500 Internal Server Error` response, indicating that the AI failed to produce content.

```typescript
    logger.info(`[${requestId}] Wand generation successful`)

    if (completion.usage && workflowId) {
      await updateUserStatsForWand(workflowId, completion.usage, requestId)
    }
```
*   **Explanation**: If content was successfully generated, an info log confirms it. If the `completion` object contains `usage` data (token counts) and a `workflowId` was provided, `updateUserStatsForWand` is called to update billing.

```typescript
    return NextResponse.json({ success: true, content: generatedContent })
```
*   **Explanation**: Finally, a successful JSON response is sent to the client, containing `success: true` and the `generatedContent`.

---

#### **General Error Handling**

```typescript
  } catch (error: any) {
    logger.error(`[${requestId}] Wand generation failed`, {
      name: error?.name,
      message: error?.message || 'Unknown error',
      code: error?.code,
      status: error?.status,
      responseStatus: error instanceof OpenAI.APIError ? error.status : error?.response?.status,
      responseData: (error as any)?.response?.data
        ? safeStringify((error as any).response.data)
        : undefined,
      stack: error?.stack,
      useWandAzure,
      model: useWandAzure ? wandModelName : 'gpt-4o',
      endpoint: useWandAzure ? azureEndpoint : 'api.openai.com',
      apiVersion: useWandAzure ? azureApiVersion : 'N/A',
    })
```
*   **Explanation**: This is the top-level `catch` block for the entire `POST` function, catching any unhandled errors during non-streaming AI generation or other parts of the request. It logs a very detailed error object, including specific properties like `name`, `message`, various `status` codes, and safely stringified response data, to assist in debugging.

```typescript
    let clientErrorMessage = 'Wand generation failed. Please try again later.'
    let status = 500
```
*   **Explanation**: Default error message and HTTP status code are initialized for the client response.

```typescript
    if (error instanceof OpenAI.APIError) {
      status = error.status || 500
      logger.error(
        `[${requestId}] ${useWandAzure ? 'Azure OpenAI' : 'OpenAI'} API Error: ${status} - ${error.message}`
      )

      if (status === 401) {
        clientErrorMessage = 'Authentication failed. Please check your API key configuration.'
      } else if (status === 429) {
        clientErrorMessage = 'Rate limit exceeded. Please try again later.'
      } else if (status >= 500) {
        clientErrorMessage =
          'The wand generation service is currently unavailable. Please try again later.'
      }
    } else if (useWandAzure && error.message?.includes('DeploymentNotFound')) {
      clientErrorMessage =
        'Azure OpenAI deployment not found. Please check your model deployment configuration.'
      status = 404
    }
```
*   **Explanation**: This block refines the `clientErrorMessage` and `status` based on the type of error:
    *   If it's an `OpenAI.APIError` (which covers both OpenAI and Azure SDK errors), it extracts the HTTP status and provides specific messages for `401` (Unauthorized), `429` (Rate Limit Exceeded), and `5xx` (Server Errors).
    *   If it's an Azure-specific error indicating `DeploymentNotFound`, it sets a specialized message and a `404 Not Found` status.

```typescript
    return NextResponse.json(
      {
        success: false,
        error: clientErrorMessage,
      },
      { status }
    )
  }
}
```
*   **Explanation**: Finally, it returns a `NextResponse` with `success: false`, the determined `clientErrorMessage`, and the appropriate HTTP `status` code, ensuring a consistent error format for the API consumer.